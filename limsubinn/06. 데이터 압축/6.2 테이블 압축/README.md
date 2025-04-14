## 6.2 테이블 압축
운영체제나 하드웨어에 대한 제약 없이 사용할 수 있어 일반적으로 활용도가 더 높다. <br>

디스크의 데이터 파일 크기를 줄일 수 있는 장점이 있지만, <br>
버퍼 풀 공간 활용률이 낮다, 쿼리 처리 성능이 낮다, 빈번한 데이터 변경 시 압축률이 떨어진다는 단점이 있다. <br>

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

