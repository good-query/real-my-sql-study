## 6.2 테이블 압축
운영체제나 하드웨어에 대한 제약 없이 사용할 수 있어 일반적으로 활용도가 더 높다. <br>

디스크의 데이터 파일 크기를 줄일 수 있는 장점이 있지만, <br>
버퍼 풀 공간 활용률이 낮다, 쿼리 처리 성능이 낮다, 빈번한 데이터 변경 시 압축률이 떨어진다는 단점이 있다. 

<br>

### 6.2.1 압축 테이블 생성
```sql
mysql> SET GLOBAL innodb_file_per_table=ON;

mysql> CREATE TABLE compressed_table (
         c1 INT PRIMARY KEY
       )
       ROW_FORMAT=COMPRESSED
       KEY_BLOCK_SIZE=8;
```

- 압축을 사용하려는 테이블이 별도의 테이블 스페이스를 사용해야 한다.
  - 이를 위해서는 `innodb_file_per_table` 시스템 변수가 `ON`으로 설정된 상태에서 테이블이 생성되어야 한다.
- 테이블을 생성할 때 `ROW_FORMAT=COMPRESSED` 옵션을 명시해야 한다.
- `KEY_BLOCK_SIZE` 옵션을 이용해 압축된 페이지의 목표 크기를 설정한다. (2*n, n은 2 이상)
  - InnoDB 스토리지 엔진의 페이지 크기가 16KB라면 `KEY_BLOCK_SIZE`는 4KB 또는 8KB만 설정할 수 있다.
  - 32KB 또는 64KB인 경우에는 테이블 압축을 적용할 수 없다.
- `ROW_FORMAT` 옵션이 생략되고 `KEY_BLOCK_SIZE` 옵션만 명시하면 자동으로 `ROW_FORMAT=COMPRESSED` 옵션이 추가되어 생성된다.

<br>

현재 InnoDB 스토리지 엔진의 데이터 페이지(블록) 크기가 16KB, KEY_BLOCK_SIZE가 8로 설정됐다고 가정해보자. <br>
InnoDB 스토리지 엔진이 압축을 적용하는 방법은 다음과 같다.
```
1. 16KB의 데이터 페이지를 압축
   1.1 압축된 결과가 8KB 이하이면 그대로 디스크에 저장 (압축 완료)
   1.2 압축된 결과가 8KB를 초과하면 원본 페이지를 split해서 2개의 페이지에 8KB씩 저장
2. 나뉜 페이지 각각에 대해 "1"번 단계를 반복 실행
```
<img width="500" alt="image" src="https://github.com/user-attachments/assets/758acd08-7ff5-4600-a9a6-ae9231adadb4"> <br>

<br>

테이블 압축 방식에서 가장 중요한 것은 원본 데이터 페이지의 압축 결과가 목표 크기보다 작거나 같을 때까지 반복해서 페이지를 스플릿하는 것이다. <br>
따라서 목표 크기가 잘못 설정되면 MySQL 서버의 처리 성능이 급격히 떨어질 수 있으니 주의하자.

<br>

### 6.2.2 KEY_BLOCK_SIZE 결정
테이블 압축에서 가장 중요한 부분은 압축된 결과가 어느 정도가 될지를 예측해서 `KEY_BLOCK_SIZE`를 결정하는 것이다.
- 테이블 압축을 적용하기 전에 먼저 KEY_BLOCK_SIZE를 4KB 또는 8KB로 테이블을 생성해서 샘플 데이터를 저장해보고 적절한지 판단하는 것이 좋다.
- 샘플 데이터는 많으면 많을수록 더 정확한 테스트가 가능하다. 최소한 테이블의 데이터 페이지가 10개 정도는 생성되도록 테스트 데이터를 INSERT하는 것이 좋다.

[예제] 데이터베이스의 employees 테이블을 이용해 간단히 KEY_BLOCK_SIZE를 선택해보자.
```sql
mysql> USE employees;

# employees 테이블과 동일한 구조로 테이블 압축을 사용하는 예제 테이블 생성
mysql> CREATE TABLE employees_comp4k (
         emp_no int NOT NULL,
         birth_date date NOT NULL,
         first_name varchar(14) NOT NULL,
         last_name varchar(16) NOT NULL,
         gender enum('M', 'F') NOT NULL,
         hire_date date NOT NULL,
         PRIMARY KEY (emp_no),
         KEY ix_firstname (first_name),
         KEY ix_hiredate (hire_date)
       ) ROW_FORMAT=COMPRESSED KEY_BLOCK_SIZE=4;

# 테스트 실행 전 innodb_cmp_per_index_enabled 시스템 변수를 ON으로 변경 => 인덱스별로 압축 실행 횟수와 성공 횟수가 기록된다.
mysql> SET GLOBAL innodb_cmp_per_index_enabled=ON;

# employees 테이블의 데이터를 그대로 압축 테이블로 저장
mysql> INSERT INTO employees_comp4k SELECT * FROM employees;

# 인덱스별로 압축 횟수와 성공 횟수, 압축 실패율을 조회
mysql> SELECT
         table_name, index_name, compress_ops, compress_ops_ok,
         (compress_ops-compress_ops_ok)/compress_ops * 100 as compression_failure_pct
       FROM information_schema.INNODB_CMP_PER_INDEX;
```
압축 실패율은 3~5% 미만으로 유지할 수 있게 `KEY_BLOCK_SIZE`를 선택하는 것이 좋다. <br>

압축 실패율이 높다고 해서 압축을 사용하지 말아야 한다는 것을 의미하지는 않는다. <br>
압축 시도가 실패해서 페이지 스플릿 후 재압축한다고 하더라도 전체적으로 데이터 파일의 크기가 큰 폭으로 줄어든다면 큰 손해는 아닐 것이다. <br>

반면에 압축 실패율이 높지 않더라도 테이블의 데이터가 매우 빈번하게 조회되고 변경된다면 압축은 고려하지 않는 것이 좋다. <br>
테이블 압축은 [zlib](https://namu.wiki/w/zlib)을 이용해 압축을 실행하는데, 이는 많은 CPU 자원을 소모한다. 

<br>

### 6.2.3 압축된 페이지의 버퍼 풀 적재 및 사용
InnoDB 스토리지 엔진은 압축된 테이블의 데이터 페이지를 버퍼 풀에 적재하면 압축된 상태와 압축이 해제된 상태 2개 버전을 관리한다.
따라서 디스크에서 읽은 상태 그대로의 데이터 페이지 목록을 관리하는 **LRU 리스트**와 압축된 페이지들의 압축 해제 버전인 **Unzip_LRU 리스트**를 별도로 관리한다. <br>

MySQL 서버에는 압축된 테이블과 압축되지 않은 테이블이 공존하므로 결국 LRU 리스트는 압축된 페이지와 압축되지 않은 페이지를 모두 가질 수 있다.
  - 압축이 적용되지 않은 테이블의 데이터 페이지
  - 압축이 적용된 테이블의 압축된 데이터 페이지

Unzip_LRU 리스트는 압축이 적용되지 않은 테이블의 데이터 페이지는 가지지 않으며, 압축이 적용된 테이블에서 읽은 데이터 페이지만 관리한다.

<br>

결국 InnoDB 스토리지 엔진은 압축된 테이블에 대해 버퍼 풀의 공간을 이중으로 사용함으로써 메모리를 낭비하는 효과를 가진다. <br>
또한, 압축된 페이지에서 데이터를 읽거나 변경하려면 압축을 해제해야 하는데, 압축 및 압축 해제 작업은 CPU를 상대적으로 많이 소모하는 작업이다. <br>
따라서 Unzip_LRU 리스트를 별도로 관리하고 있다가 MySQL 서버로 유입되는 요청 패턴에 따라 적절히 다음과 같은 처리를 수행한다.
- InnoDB 버퍼 풀의 공간이 필요한 경우에는 LRU 리스트에서 원본 데이터 페이지(압축된 형태)로 유지하고, Unzip_LRU 리스트에서 압축 해제된 버전은 제거해서 버퍼 풀의 공간을 확보한다.
- 압축된 데이터 페이지가 자주 사용되는 경우에는 Unzip_LRU 리스트에 압축 해제된 페이지를 계속 유지하면서 압축 및 압축 해제 작업을 최소화한다.
- 압축된 데이터 페이지가 사용되지 않아 LRU 리스트에서 제거되는 경우에는 Unzip_LRU 리스트에서도 함께 제거된다.

<br>

InnoDB 스토리지 엔진은 버퍼 풀에서 압축 해제된 버전의 데이터를 적절한 수준으로 유지하기 위해 다음과 같은 어댑티브 알고리즘을 사용한다.
- CPU 사용량이 높은 서버에서는 가능하면 압축과 압축 해제를 피하기 위해 Unzip_LRU의 비율을 높여서 유지
- Disk IO 사용량이 높은 서버에서는 가능하면 Unzip_LRU 리스트에 비율을 낮춰 InnoDB 버퍼 풀의 공간을 더 확보

<br>

### 6.2.4 테이블 압축 관련 설정
- `innodb_cmp_per_index_enabled`: 테이블 압축이 사용된 테이블의 모든 인덱스별로 압축 성공 및 압축 실행 횟수를 사용하도록 설정
  - 비활성화되면 테이블 단위의 압축 성공 및 압축 실행 횟수만 수집
  - 테이블 단위로 수집된 정보는 `information_schema.INNODB_CMP_PER_INDEX` 테이블에 기록된다.
- `innodb_compression_level`: zlib 압축 알고리즘의 압축률 설정 (0~9, 기본값 6)
  - 값이 작을수록 압축 속도는 빨라지지만 저장 공간은 커질 수 있다.
  - 값이 커질수록 압축 숙도는 느려질 수 있지만 압축률은 높아진다.
  - 압축 속도가 빨라진다는 것은 CPU 자원을 그만큼 적게 사용한다는 의미이다.
- `innodb_compression_failure_threshold_pct`: 테이블 단위로 압축 실패율이 해당 설정값보다 커지면 압축을 실행하기 전 원본 데이터 페이지의 끝에 의도적으로 일정 크기의 빈 공간(Padding)을 추가한다.
  추가된 빈 공간은 압축률을 높여서 압축 결과가 `KEY_BLOCK_SIZE`보다 작아지게 만드는 효과를 낸다.
- `innodb_compression_pad_pct_max`: 패딩 공간은 압축 실패율이 높아질수록 계속 증가된 크기를 가지는데, 추가할 수 있는 패딩 공간의 최대 크기를 설정한다.
  %값을 설정하는데, 전체 페이지 크기 대비 패딩 공간의 비율을 의미한다.
- `innodb_log_compressed_pages`: MySQL 서버가 비정상적으로 종료됐다가 다시 시작되는 경우
- 압축 알고리즘의 버전 차이가 있더라도 복구 과정이 실패하지 않도록 InnoDB 스토리지 엔진은 압축된 데이터 페이지를 그대로 리두 로그에 기록한다.
