## 9.2 기본 데이터 처리

### 9.2.1 풀 테이블 스캔과 풀 인덱스 스캔
풀 테이블 스캔은 인덱스를 사용하지 않고 테이블의 데이터를 처음부터 끝까지 읽어서 요청된 작업을 처리하는 작업을 의미한다.
- 테이블의 레코드 건수가 너무 작아 인덱스를 통해 읽는 것보다 풀 테이블 스캔을 하는 편이 더 빠른 경우
- WHERE 절이나 ON 절에 인덱스를 이용할 수 있는 적절한 조건이 없는 경우
- 인덱스 레인지 스캔을 사용할 수 있는 쿼리라고 하더라도 옵티마이저가 판단한 조건 일치 레코드 건수가 너무 많은 경우

<br>

일반적으로 테이블의 전체 크기는 인덱스보다 훨씬 크기 때문에 테이블을 처음부터 끝까지 읽는 작업은 상당히 많은 디스크 읽기가 필요하다. <br>
그래서 대부분 DBMS는 풀 테이블 스캔을 실행할 때 한꺼번에 여러 개의 블록이나 페이지를 읽어오는 기능을 내장하고 있다. <br>

MySQL에서 MyISAM 스토리지 엔진은 풀 테이블 스캔을 실행할 때 디스크로부터 페이지를 하나씩 읽어오지만, InnoDB 스토리지 엔진은 아니다. <br>
InnoDB 스토리지 엔진은 특정 테이블의 연속된 데이터 페이지가 읽히면 백그라운드 스레드에 의해 Read ahead 작업이 자동으로 시작된다. <br>
이는 어떤 영역의 데이터가 앞으로 필요해질 것이라고 예측해서 요청이 오기 전에 미리 디스크에서 읽어 InnoDB의 버퍼 풀에 가져다 두는 것이다. <br>
즉, 풀 테이블 스캔이 실행되면 처음 몇 개의 데이터 페이지는 포그라운드 스레드가 페이지 읽기를 실행하지만, <br>
특정 시점부터는 읽기 작업을 백그라운드 스레드로 넘기고, 이 시점부터는 한 번에 4개 또는 8개씩 페이지를 읽으면서 계속 그 수를 증가시킨다. <br>
이때 한 번에 최대 64개의 데이터 페이지까지 읽어서 버퍼 풀에 저장해 둔다. <br>
포그라운드 스레드는 미리 버퍼 풀에 준비된 데이터를 가져다 사용하기만 하면 되므로 쿼리가 상당히 빨리 처리되는 것이다. <br>

MySQL 서버에서는 `innodb_read_ahead_threshold` 시스템 변수를 이용해 Read ahead를 언제 시작할지 임계값을 설정할 수 있다. <br>
포그라운드 스레드에 의해 시스템 변수에 설정된 개수만큼의 연속된 데이터 페이지가 읽히면 <br>
InnoDB 스토리지 엔진은 백그라운드 스레드를 이용해 대량으로 그 다음 페이지들을 읽어서 버퍼 풀로 적재한다. <br>
일반적으로 디폴트 설정으로도 충분하지만 데이터 웨어하우스용으로 MySQL을 사용한다면 이 옵션을 더 낮은 값으로 설정하는 것도 좋다. <br>

Read ahead는 풀 인덱스 스캔에서도 동일하게 사용된다. <br>
풀 인덱스 스캔은 인덱스를 처음부터 끝까지 스캔하는 것을 의미한다. <br>

<br>

### 9.2.2 병렬 처리
MySQL 8.0부터 하나의 쿼리를 여러 스레드가 작업을 나누어 동시에 처리하는 것이 가능해졌다. <br>
`innodb_parallel_read_threads` 시스템 변수를 이용해 하나의 쿼리를 최대 몇 개의 스레드를 이용해서 처리할지 설정할 수 있다. <br>

아직 MySQL 서버에서는 쿼리를 여러 개의 스레드를 이용해 병렬로 처리하게 하는 힌트나 옵션은 없다. <br>
아무런 WHERE 조건 없이 단순히 테이블의 전체 건수를 가져오는 쿼리만 병렬로 처리할 수 있다. <br>

![image](https://github.com/user-attachments/assets/8b661dfb-a710-4f71-afa0-b26689aeafda) <br>

병렬 처리용 스레드 개수가 늘어날수록 쿼리 처리에 걸리는 시간이 줄어드는 것을 확인할 수 있다. <br>
하지만 서버에 장착된 CPU의 코어 개수를 넘어서는 경우에는 오히려 성능이 떨어질 수도 있으니 주의하자. <br>

<br>

### 9.2.3 ORDER BY 처리(Using filesort)
레코드 1~2건을 가져오는 쿼리를 제외하면 대부분의 SELECT 쿼리에서 정렬은 필수적으로 사용된다. <br>
정렬을 처리하는 방법은 인덱스를 이용하는 방법과 쿼리가 실행될 때 "Filesort"라는 별도의 처리를 이용하는 방법으로 나눌 수 있다. <br>

<table>
  <thead>
    <tr>
      <th></th>
      <th>장점</th>
      <th>단점</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>인덱스</td>
      <td>
        <ul>
          <li>읽기 속도가 빠르다.</li>
        </ul>
      </td>
      <td>
        <ul>
          <li>INSERT, UPDATE, DELETE 작업 시 부가적인 인덱스 추가/삭제 작업이 필요하므로 느리다.</li>
          <li>인덱스 때문에 디스크 공간이 더 많이 필요하다.</li>
          <li>인덱스의 개수가 늘어날수록 InnoDB의 버퍼 풀을 위한 메모리가 많이 필요하다.</li>
        </ul>
      </td>
    </tr>
    <tr>
      <td>Filesort</td>
      <td>
        <ul>
          <li>인덱스를 이용할 때의 단점이 장점으로 바뀐다.</li>
          <li>정렬해야 할 레코드가 많지 않으면 메모리에서 Filesort가 처리되므로 충분히 빠르다.</li>
        </ul>
      </td>
      <td>
        <ul>
          <li>정렬 작업이 쿼리 실행 시 처리되므로 레코드 대상 건수가 많아질수록 쿼리의 응답 속도가 느리다.</li>
        </ul>
      </td>
    </tr>
  </tbody>
</table>

<br>

다음과 같은 이유로 모든 정렬을 인덱스를 이용하도록 튜닝하기란 거의 불가능하다.
- 정렬 기준이 너무 많아 요건별로 모두 인덱스를 생성하는 것이 불가능한 경우
- GROUP BY의 결과 또는 DISTINCT 같은 처리의 결과를 정렬해야 하는 경우
- UNION의 결과와 같이 임시 테이블의 결과를 다시 정렬해야 하는 경우
- 랜덤하게 결과 레코드를 가져와야 하는 경우

<br>

MySQL 서버에서 인덱스를 이용하지 않고 별도의 정렬 처리를 수행했는지는 실행 계획의 **Extra** 컬럼에 "**Using filesort**" 메시지가 표시되는지 확인하면 된다.

<br>

#### 1️⃣ 소트 버퍼
MySQL은 정렬을 수행하기 위해 별도의 메모리 공간을 할당 받아서 사용하는데, 이 메모리 공간을 소트 버퍼라고 한다. <br>
정렬이 필요한 경우에만 할당되며, 버퍼의 크기는 정렬해야 할 레코드의 크기에 따라 가변적으로 증가하지만 <br>
최대 사용 가능한 소트 버퍼의 공간은 `sort_buffer_size` 시스템 변수로 설정할 수 있다. <br>
소트 버퍼를 위한 메모리 공간은 쿼리의 실행이 완료되면 즉시 시스템으로 반납된다. <br>

정렬해야 할 레코드의 건수가 소트 버퍼로 할당된 공간보다 크면 MySQL은 정렬해야 할 레코드를 여러 조각으로 나눠서 처리하는데, <br>
이 과정에서 임시 저장을 위해 디스크를 사용한다. <br>
메모리의 소트 버퍼에서 정렬을 수행하고, 그 결과를 임시로 디스크에 기록해 둔다. <br>
그리고 다음 레코드를 가져와서 다시 정렬해서 반복적으로 디스크에 임시 저장한다. <br>
이처럼 각 버퍼 크기만큼 정렬된 레코드를 다시 병합하면서 정렬을 수행해야 한다. <br>
이 병합 작업을 Multi-merge라고 표현하며, 수행된 멀티 머지 횟수는 `Sort_merge_passes` 상태 변수에 누적해서 집계된다. <br>

이 작업들은 모두 디스크의 쓰기와 읽기를 유발하며, 레코드 건수가 많을수록 이 반복 작업의 횟수가 많아진다. <br>
하지만 `sort_buffer_size` 시스템 변수의 설정값이 크다고 해서 속도가 무조건 빨라지는 것은 아니다. <br>
어떤 데이터를 정렬하는지, 서버의 메모리나 디스크의 특성에 따라 결과가 달라질 수 있다. <br>
또한, 리눅스 계열의 운영체제에서는 너무 큰 `sort_buffer_size`를 사용하는 경우, 큰 메모리 공간 할당 때문에 성능이 훨씬 떨어질 수도 있다. <br>

일반적인 트랜잭션 처리용 MySQL 서버의 소트 버퍼 크기는 56KB ~ 1MB 이 적절해 보인다. <br>
정렬을 위해 할당하는 소트 버퍼는 MySQL의 세션 메모리 영역에 해당하기 때문에 여러 클라이언트가 공유해서 사용할 수 있는 영역이 아니다. <br>
커넥션이 많을수록, 정렬 작업이 많을수록 소트 버퍼로 소비되는 메모리 공간이 커진다. <br>
소트 버퍼를 크게 설정해서 빠른 성능을 얻을 수는 없지만 디스크의 읽기와 쓰기 사용량은 줄일 수 있다. <br>

<br>

#### 2️⃣ 정렬 알고리즘
레코드를 정렬할 때 레코드 전체를 소트 버퍼에 담을지 또는 정렬 기준 컬럼만 소트 버퍼에 담을지에 따라 <br>
"Single-pass"와 "Two-pass" 두 가지 정렬 모드로 나눌 수 있다. <br>

MySQL 서버의 정렬 방식은 3가지가 있다.
1. `<sort_key, rowid>`: 정렬 키와 레코드의 ROW-ID만 가져와서 정렬하는 방식
2. `<sort_key, additional_fields>`: 정렬 키와 레코드 전체를 가져와서 정렬하는 방식. 레코드의 컬럼들은 고정 사이즈로 메모리 저장
3. `<sort_key, packed_additional_fields>`: 정렬 키와 레코드 전체를 가져와서 정렬하는 방식. 레코드의 컬럼들은 가변 사이즈로 메모리 저장

<br>

여기서 첫 번째 방식을 "투 패스" 정렬 방식, 두 번째와 세 번째 방식을 "싱글 패스" 정렬 방식이라고 하겠다. <br>
MySQL 5.7부터 세 번째 방식이 도입됐는데, 이는 정렬을 위한 메모리 공간의 효율적인 사용을 위해 추가로 도입된 방식이다. <br>

정렬을 수행하는 쿼리가 어떤 정렬 모드를 사용하는지는 옵티마이저 트레이스 기능으로 확인할 수 있다. <br>
```sql
# 옵티마이저 트레이스 활성화
mysql> SET OPTIMIZER_TRACE="enabled=on", END_MARKERS_IN_JSON=on;
mysql> SET OPTIMIZER_TRACE_MAX_MEM_SIZE=1000000;

# 쿼리 실행
mysql> SELECT * FROM employees ORDER BY last_name LIMIT 100000,1;

# 트레이스 내용 확인
mysql> SELECT * FROM INFORMATION_SCHEMA.OPTIMIZER_TRACE \G
```

![image](https://github.com/user-attachments/assets/1733500c-5907-4882-bd02-1eeabf88ac68) <br><br>
출력된 내용에서 "filesort_summary" 섹션의 "sort_algorithm" 필드에 정렬 알고리즘이 표시되고, <br>
"sort_mode" 필드에는 "<fixed_sort_key, packed_additional_fileds>"가 표시된 것을 확인할 수 있다. <br>

<br>

**1. 싱글 패스 정렬 방식**
<br>

소트 버퍼에 정렬 기준 컬럼을 포함해 SELECT 대상이 되는 컬럼 전부를 담아 정렬을 수행한다. <br>

e.g.
```sql
mysql> SELECT emp_no, first_name, last_name
       FROM employees
       ORDER BY first_name;
```

<img width="600" alt="image" src="https://github.com/user-attachments/assets/5f184a8c-b5b9-4ada-8b3a-df78f300465c"> <br>

처음 employees 테이블을 읽을 때 정렬에 필요하지 않은 last_name 컬럼까지 전부 읽어 소트 버퍼에 담고 정렬을 수행한다. <br>
그리고 정렬이 완료되면 정렬 버퍼의 내용을 그대로 클라이언트로 넘겨준다. <br>

<br>

**2. 투 패스 정렬 방식**
<br>

정렬 대상 컬럼과 프라이머리 키 값만 소트 버퍼에 담아 정렬을 수행하고, 정렬된 순서대로 다시 프라이머리 키로 테이블을 읽어 SELECT할 컬럼을 가져온다. <br>

<img width="600" alt="image" src="https://github.com/user-attachments/assets/945b60b7-ea8f-44b5-ba51-dd61b8393ac1"> <br>

처음 employees 테이블을 읽을 때는 정렬에 필요한 first_name 컬럼과 프라이머리 키인 emp_no만 읽어서 정렬을 수행한다. <br>
정렬이 완료되면 그 결과 순서대로 employees 테이블을 한 번 더 읽어 last_name을 가져오고, 최종적으로 그 결과를 클라이언트에 넘긴다. <br>

MySQL의 예전 정렬 방식인 투 패스 방식은 테이블을 두 번 읽어야 하지만, 새로운 정렬 방식인 싱글 패스는 그렇지 않다. <br>
하지만 싱글 패스 정렬 방식은 더 많은 소트 버퍼 공간이 필요하다. <br>

최신 버전에서는 일반적으로 싱글 패스 정렬 방식을 주로 사용한다. <br>
하지만 레코드의 크기가 `max_length_for_sort_data` 시스템 변수에 설정된 값보다 클 때, <br>
BLOB이나 TEXT 타입의 컬럼이 SELECT 대상에 포함될 때는 투 패스 정렬 방식을 사용한다. <br>

싱글 패스 방식은 정렬 대상 레코드의 크기나 건수가 작은 경우 빠른 성능을 보이며, <br>
투 패스 방식은 정렬 대상 레코드의 크기나 건수가 상당히 많은 경우 효율적이라고 볼 수 있다. <br>

<br>

💡 참고 <br>
> SELECT 쿼리에서 모든 컬럼을 가져오도록 개발하면 정렬 버퍼를 비효율적으로 사용할 가능성이 크다. <br>
> 따라서 꼭 필요한 컬럼만 조회하도록 쿼리를 작성하는 것이 좋다.

<br>

#### 3️⃣ 정렬 처리 방법
쿼리에 `ORDER BY`가 사용되면 반드시 다음 3가지 처리 방법 중 하나로 정렬이 처리된다. 일반적으로 아래로 갈수록 처리 속도는 떨어진다.
| 정렬 처리 방법 | 실행 계획의 Extra 컬럼 내용 |
| :-- | :-- |
| 인덱스 | 별도 표기 없음 |
| 조인에서 드라이빙 테이블만 정렬 | "Using filesort" |
| 조인에서 조인 결과를 임시 테이블로 저장 후 정렬 | "Using temporary; Using filesort" |

<br>

먼저 옵티마이저는 정렬 처리를 위해 인덱스를 이용할 수 있을지 검토하고, <br>
인덱스를 이용할 수 있다면 별도의 "Filesort" 과정 없이 인덱스를 순서대로 읽어 결과를 반환한다. <br>

인덱스를 이용할 수 없다면 WHERE 조건에 일치하는 레코드를 검색해 정렬 버퍼에 저장하면서 정렬을 처리(Filesort)한다. <br>
이때 MySQL 옵티마이저는 정렬 대상 레코드를 최소화하기 위해 다음 2가지 방법 중 하나를 선택한다. <br>
- 조인의 드라이빙 테이블만 정렬한 다음 조인을 수행
- 조인이 끝나고 일치하는 레코드를 모두 가져온 후 정렬을 수행

<br>

일반적으로 조인이 수행되면서 레코드 건수와 크기는 거의 배수로 불어나기 때문에 <br>
가능하다면 드라이빙 테이블만 정렬한 다음 조인을 수행하는 방법이 효율적이다. <br>

<br>

**1. 인덱스를 이용한 정렬**
<br>

반드시 `ORDER BY`에 명시된 컬럼이 제일 먼저 읽는 테이블에 속하고, `ORDER BY`의 순서대로 생성된 인덱스가 있어야 한다. <br>
`WHERE` 절에 첫 번째로 읽는 테이블의 컬럼에 대한 조건이 있다면 그 조건과 `ORDER BY`는 같은 인덱스를 사용할 수 있어야 한다. <br>
B-Tree 계열의 인덱스가 아닌 해시 인덱스, 전문 검색 인덱스 등에서는 사용할 수 없다. R-Tree도 사용할 수 없다. <br>
여러 테이블이 조인되는 경우에는 Nested-loop 방식의 조인에서만 사용할 수 있다. <br>

인덱스를 이용해 정렬이 처리되는 경우에는 실제 인덱스의 값이 정렬되어 있기 때문에 인덱스의 순서대로 읽기만 하면 된다. <br>
이는 B-Tree 인덱스가 키 값으로 정렬되어 있기 때문이다. <br>
또한, 조인이 Nested-loop 방식으로 실행되기 때문에 조인 때문에 드라이빙 테이블의 인덱스 읽기 순서가 흐트러지지 않는다. <br>
하지만 조인이 사용된 쿼리의 실행 계획에 조인 버퍼가 사용되면 순서가 흐트러질 수 있기 때문에 주의해야 한다. <br>

<br>

💡 참고 <br>
> Nested-loop: 2개 이상의 테이블에서 하나의 집합을 기준으로 순차적으로 상대방 Row를 결합하여 원하는 결과를 조합하는 조인 방식

<br>

**2. 조인의 드라이빙 테이블만 정렬**
<br>

조인을 실행하기 전에 첫 번째 테이블의 레코드를 먼저 정렬한 다음 조인을 실행한다. <br>
이 방법으로 처리되려면 첫 번째로 읽히는 테이블의 컬럼만으로 `ORDER BY` 절을 작성해야 한다. <br>

e.g.
```sql
mysql> SELECT *
       FROM employees e, salaries s
       WHERE s.emp_no = e.emp_no AND e.emp_no BETWEEN 100002 AND 100010
       ORDER BY e.last_name;
```

WHERE 절이 다음 2가지 조건을 갖추고 있기 때문에 옵티마이저는 employees 테이블을 드라이빙 테이블로 선택할 것이다.
- WHERE 절의 검색 조건은 employees 테이블의 프라이머리 키를 이용해 검색하면 작업량을 줄일 수 있다.
- 드리븐 테이블(salaries)의 조인 컬럼인 emp_no 컬럼에 인덱스가 있다.

<br>

검색은 인덱스 레인지 스캔으로 처리할 수 있지만, <br>
ORDER BY 절에 명시된 컬럼은 employees 테이블의 프라이머리 키와 전혀 연관이 없으므로 인덱스를 이용한 정렬은 불가능하다. <br>
그런데 ORDER BY 절의 정렬 기준 컬럼이 드라이빙 테이블(employees)에 포함된 컬럼임을 알 수 있다. <br>
옵티마이저는 드라이빙 테이블만 검색해서 정렬을 먼저 수행하고, 그 결과와 salaries 테이블을 조인한 것이다. <br>

<img width="600" alt="image" src="https://github.com/user-attachments/assets/aef99113-0cf4-416c-b968-914131266566"> <br>
1. 인덱스를 이용해 `emp_no BETWEEN 100002 AND 100010` 조건을 만족하는 9건을 검색
2. 검색 결과를 last_name 컬럼으로 정렬 수행 (Filesort)
3. 정렬된 결과를 순서대로 읽으면서 salaries 테이블과 조인을 수행해 86건의 최종 결과를 가져옴

<br>

**3. 임시 테이블을 이용한 정렬**
<br>

2개 이상의 테이블을 조인해서 그 결과를 정렬해야 한다면 임시 테이블이 필요할 수도 있다. <br>
"조인의 드라이빙 테이블만 정렬" 외 패턴의 쿼리에서는 항상 조인의 결과를 임시 테이블에 저장하고, 그 결과를 다시 정렬한다. <br>
이 방법은 3가지 정렬 방법 중 정렬해야 할 레코드 건수가 가장 많기 때문에 가장 느리다. <br>

e.g.
```sql
mysql> SELECT *
       FROM employees e, salaries s
       WHERE s.emp_no = e.emp_no AND e.emp_no BETWEEN 100002 AND 100010
       ORDER BY s.salary;
```

ORDER BY 절의 정렬 기준 컬럼이 드라이빙 테이블이 아닌 드리븐 테이블에 있는 컬럼이다. <br>
즉, 정렬이 수행되기 전에 salaries 테이블을 읽어야 하므로 이 쿼리는 조인된 데이터를 가지고 정렬해야 한다. <br>

<img width="600" alt="image" src="https://github.com/user-attachments/assets/1f83224d-ed20-4e50-a84e-4061085d33f4"> <br>
조인된 결과를 임시 테이블에 저장하고, 그 결과를 다시 정렬 처리한다. <br>

<br>

**4. 정렬 처리 방법의 성능 비교**
<br>

일반적으로 `LIMIT`은 테이블이나 처리 결과의 일부만 가져오기 때문에 MySQL 서버가 처리해야 할 작업량을 줄이는 역할을 한다. <br>
그런데 `ORDER BY`나 `GROUP BY`와 같은 작업은 `WHERE` 조건을 만족하는 레코드를 `LIMIT` 건수만큼만 가져와서는 처리할 수 없다. <br>
조건을 만족하는 레코드를 모두 가져와 정렬이나 그루핑 작업을 실행해야만 건수를 제한할 수 있다. <br>
`WHERE` 조건이 아무리 인덱스를 잘 활용하도록 튜닝해도 잘못된 `ORDER BY`나 `GROUP BY` 때문에 쿼리가 느려지는 경우가 자주 발생한다. <br>

<br>

1. 스트리밍 방식
   - 서버 쪽에서 처리할 데이터가 얼마인지에 관계없이 조건에 일치하는 레코드가 검색될 때마다 바로 클라이언트로 전송해주는 방식
   - 클라이언트는 쿼리를 요청하고 곧바로 원했던 첫 번째 레코드를 전달받는다.
   - 쿼리가 스트리밍 방식으로 처리될 수 있다면 클라이언트는 MySQL 서버가 일치하는 레코드를 찾는 즉시 전달받기 때문에 동시에 데이터의 가공 작업을 시작할 수 있다. 따라서 빠른 응답 시간을 보장한다.
   - 또한, 스트리밍 방식으로 처리되는 쿼리에서 `LIMIT`처럼 결과 건수를 제한하는 조건들은 쿼리의 전체 실행 시간을 줄여준다. 풀 테이블 스캔의 결과가 아무런 버퍼링 처리나 필터링 과정 없이 바로 클라이언트로 스트리밍되기 때문이다.
2. 버퍼링 방식
   - `ORDER BY`나 `GROUP BY` 같은 처리는 쿼리의 결과가 스트리밍되는 것을 불가능하게 한다. 조건에 일치하는 모든 레코드를 가져온 후 정렬하거나 그루핑해서 차례대로 보내야 하기 때문이다.
   - MySQL 서버에서는 모든 레코드를 검색하고 정렬 작업을 하는 동안 클라이언트는 아무것도 하지 않고 기다려야 하기 때문에 응답 속도가 느려진다.
   - 결과를 먼저 모아 MySQL 서버에서 일괄 가공해야 하므로 모든 결과를 스토리지 엔진으로부터 가져올 때까지 기다려야 한다.
   - 따라서 버퍼링 방식으로 처리되는 쿼리는 `LIMIT`처럼 결과 건수를 제한하는 조건이 있어도 성능 향상에 별로 도움이 되지 않는다.

<br>

`ORDER BY`의 3가지 처리 방법 중 인덱스를 사용한 정렬 방식만 스트리밍 형태의 처리이며, 나머지는 모두 버퍼링된 후에 정렬된다. <br>
조인과 함께 `ORDER BY` 절과 `LIMIT` 절이 사용될 경우 정렬 처리 방법 별로 어떤 차이가 있는지 비교해보자. <br>

<br>

```sql
mysql> SELECT *
       FROM tb_test1 t1, tb_test2 t2
       WHERE t1.col1 = t2.col1
       ORDER BY t1.col2
       LIMIT 10;
```

tb_test1 테이블의 레코드가 100건이고, tb_test21 테이블의 레코드가 1000건(tb_test1의 레코드 1건 당 tb_test2의 레코드가 10개씩 존재)이며, <br>
두 테이블의 조인 결과는 전체 1000건이라고 가정한다. <br>

<br>

<tb_test1이 드라이빙되는 경우> <br>
| 정렬 방법 | 읽어야 할 건수 | 조인 횟수 | 정렬해야 할 대상 건수 |
| :-- | :-- | :-- | :-- |
| 인덱스 사용 | tb_test1: 1건 <br> tb_test2: 10건 | 1번 | 0건 |
| 조인의 드라이빙 테이블만 정렬 | tb_test1: 100건 <br> tb_test2: 10건 | 1번 | 100건 |
| 임시 테이블 사용 후 정렬 | tb_test1: 100건 <br> tb_test2: 1000건 | 100번 | 1000건 |

<br>

<tb_test2가 드라이빙되는 경우> <br>
| 정렬 방법 | 읽어야 할 건수 | 조인 횟수 | 정렬해야 할 대상 건수 |
| :-- | :-- | :-- | :-- |
| 인덱스 사용 | tb_test2: 10건 <br> tb_test1: 10건 | 10번 | 0건 |
| 조인의 드라이빙 테이블만 정렬 | tb_test2: 1000건 <br> tb_test1: 10건 | 10번 | 1000건 |
| 임시 테이블 사용 후 정렬 | tb_test2: 1000건 <br> tb_test1: 100건 | 1000번 | 1000건 |

<br>

#### 4️⃣ 정렬 관련 상태 변수
MySQL 서버는 처리하는 주요 작업에 대해 해당 작업의 실행 횟수를 상태 변수로 저장한다. <br>
정렬과 관련해 지금까지 몇 건의 레코드나 정렬 처리를 수행했는지, 소트 버퍼 간의 병합 작업(멀티 머지)은 몇 번이나 발생했는지 등을 확인해볼 수 있다. <br>
```sql
mysql> FLUSH STATUS;
mysql> SHOW STATUS LIKE 'Sort%';
```

각 상태값은 다음과 같은 의미가 있다.
- `Sort_merge_passes`: 멀티 머지 처리 횟수
- `Sort_range`: 인덱스 레인지 스캔을 통해 검색된 결과에 대한 정렬 작업 횟수
- `Sort_scan`: 풀 테이블 스캔을 통해 검색된 결과에 대한 정렬 작업 횟수
- `Sort_rows`: 지금까지 정렬한 전체 레코드 건수

<br>

### 9.2.4 GROUP BY 처리
`GROUP BY` 처리는 쿼리가 스트리밍 처리를 할 수 없게 한다. <br>
`GROUP BY` 절이 있는 쿼리에서는 `HAVING` 절을 사용할 수 있는데, 이는 GROUP BY 결과에 대해 필터링 역할을 수행한다. <br>
`GROUP BY`에 사용된 조건은 인덱스를 사용해서 처리될 수 없으므로 `HAVING` 절을 튜닝하려고 인덱스를 생성하거나 다른 방법을 고민할 필요는 없다. <br>

`GROUP BY` 작업도 인덱스를 사용하는 경우와 그렇지 못한 경우로 나눌 수 있다. <br>
인덱스를 이용할 때는 인덱스를 차례대로 읽는 **인덱스 스캔 방법**과 인덱스를 건너뛰면서 읽는 **루스 인덱스 스캔 방법**으로 나뉜다. <br>
인덱스를 사용하지 못하는 쿼리에서는 **임시 테이블**을 사용한다. <br>

<br>

#### 1️⃣ 인덱스 스캔을 이용하는 GROUP BY(타이트 인덱스 스캔)
조인의 드라이빙 테이블에 속한 컬럼만 이용해 그루핑할 때 GROUP BY 컬럼으로 이미 인덱스가 있다면 <br>
그 인덱스를 차례대로 읽으면서 그루핑 작업을 수행하고 그 결과로 조인을 처리한다. <br>
GROUP BY가 인덱스를 사용해서 처리된다 하더라도 그룹 함수 등의 그룹값을 처리해야 해서 임시 테이블이 필요할 때도 있다. <br>
GROUP BY가 인덱스를 통해 처리되는 쿼리는 이미 정렬된 인덱스를 읽는 것이므로 쿼리 실행 시점에 추가적인 정렬 작업이나 내부 임시 테이블은 필요하지 않다. <br>
이러한 그루핑 방식을 사용하는 쿼리의 실행 계획에서는 Extra 컬럼에 별도로 GROUP BY 관련 코멘트("Using index for group-by")나 <br>
임시 테이블 사용 또는 정렬 관련 코멘트("Using temporary, Using filesort")가 표시되지 않는다. <br>

<br>

#### 2️⃣ 루스 인덱스 스캔을 이용하는 GROUP BY
루스 인덱스 스캔 방식은 인덱스의 레코드를 건너뛰면서 필요한 부분만 읽어서 가져오는 것을 의미한다. <br>
옵티마이저가 루스 인덱스 스캔을 사용할 때는 실행 계획의 Extra 컬럼에 **"Using index for group-by"** 코멘트가 표시된다. <br>

e.g.
```sql
mysql> EXPLAIN
         SELECT emp_no
         FROM salaries
         WHERE from_date='1985-03-01'
         GROUP BY emp_no;
```

salaries 테이블의 인덱스는 (emp_no, from_date)로 생성되어 있으므로 <br>
위의 쿼리 문장에서 WHERE 조건은 인덱스 레인지 스캔 접근 방식으로 이용할 수 없는 쿼리다. <br>
하지만 이 쿼리의 실행 계획은 다음과 같이 인덱스 레인지 스캔(range 타입)을 이용했으며, <br>
Extra 컬럼의 메시지를 보면 GROUP BY 처리까지 인덱스를 사용했다는 것을 알 수 있다. <br>

![image](https://github.com/user-attachments/assets/0698c9dd-2380-4599-854c-264e648b056f) <br>
<br>

1. (emp_no, from_date) 인덱스를 차례대로 스캔하면서 emp_no의 첫 번째 유일한 값(그룹 키) '10001'을 찾아낸다. <br>
2. (emp_no, from_date) 인덱스에서 emp_no가 '10001'인 것 중 from_date 값이 '1985-03-01'인 레코드만 가져온다.
   이 검색 방법은 1번 단계에서 알아낸 '10001' 값과 쿼리의 WHERE 절에 사용된 "from_date='1985-03-01'" 조건을 합쳐서
   "emp_no=10001 AND from_date='1985-03-01'" 조건으로 (emp_no, from_date) 인덱스를 검색하는 것과 거의 흡사하다.
3. (emp_no, from_date) 인덱스에서 emp_no의 그 다음 유니크한(그룹 키) 값을 가져온다.
4. 3번 단계에서 결과가 더 없으면 처리를 종료하고, 결과가 있다면 2번 과정으로 돌아가서 반복 수행한다.
<br>

MySQL의 루스 인덱스 스캔 방식은 단일 테이블에 대해 수행되는 GROUP BY 처리에서만 사용할 수 있다. <br>
또한, Prefix index(컬럼값의 앞쪽 일부만으로 생성된 인덱스)는 루스 인덱스 스캔을 사용할 수 없다. <br>
인덱스 레인지 스캔에서는 유니크한 값의 수가 많을수록 성능이 향상되는 반면 루스 인덱스 스캔에서는 인덱스의 유니크한 수가 적을수록 성능이 향상된다. <br>
즉, 루스 인덱스 스캔은 분포도가 좋지 않은 인덱스일수록 더 빠른 결과를 만들어낸다. <br>
루스 인덱스 스캔으로 처리되는 쿼리에서는 별도의 임시 테이블이 필요하지 않다. <br>
<br>

(col1, col2, col3) 컬럼으로 생성된 tb_test 테이블을 가정했을 때, 다음의 쿼리들은 루스 인덱스 스캔을 사용할 수 있는 쿼리다. <br>
```sql
SELECT col1, col2 FROM tb_test GROUP BY col1, col2;
SELECT DISTINCT col1, col2 FROM tb_test;
SELECT col1, MIN(col2) FROM tb_test GROUP BY col1;
SELECT col1, col2 FROM tb_test WHERE col1 < const GROUP BY col1, col2;
SELECT MAX(col3), MIN(col3), col1, col2 FROM tb_test WHERE col2 > const GROUP BY col1, col2;
SELECT col2 FROM tb_test WHERE col1 < const GROUP BY col1, col2;
SELECT col1, col2 FROM tb_test WHERE col3 = const GROUP BY col1, col2;
```
<br>

다음 쿼리는 루스 인덱스 스캔을 사용할 수 없는 쿼리 패턴이다.
```sql
# MIN()과 MAX() 이외의 집합 함수가 사용됐기 때문에 사용 불가
SELECT col1, SUM(col2) FROM tb_test GROUP BY col1;

# GROUP BY에 사용된 컬럼이 인덱스 구성 컬럼의 왼쪽부터 일치하지 않기 때문에 사용 불가
SELECT col1, col2 FROM tb_test GROUP BY col2, col3;

# SELECT 절의 컬럼이 GROUP BY와 일치하지 않기 때문에 사용 불가
SELECT col1, col3 FROM tb_test GROUP BY col1, col2;
```

<br>

#### 3️⃣ 임시 테이블을 사용하는 GROUP BY
GROUP BY에서 인덱스를 전혀 사용하지 못할 때는 이 방식으로 처리된다. <br>

e.g.
```sql
mysql> EXPLAIN
         SELECT e.last_name, AVG(s.salary)
         FROM employees e, salaries s
         WHERE s.emp_no=e.emp_no
         GROUP BY e.last_name;
```

이 쿼리의 실행 계획에서는 Extra 컬럼에 "Using temporary" 메시지가 표시됐다. <br>

![image](https://github.com/user-attachments/assets/f5f5b159-6808-4661-b0c6-9d8aa64d9fdc) <br>

실행 계획의 Extra 컬럼에 "Using filesort"는 표시되지 않고 "Using temporary"만 표시됐다. <br>
MySQL 8.0 이전까지는 GROUP BY가 사용된 쿼리는 그루핑되는 컬럼을 기준으로 묵시적인 정렬도 함께 수행됐다. <br>
하지만 MySQL 8.0부터는 이 같은 묵시적인 정렬은 더 이상 실행되지 않게 바뀌었다. <br>

MySQL 8.0에서는 GROUP BY가 필요한 경우 <br>
내부적으로 GROUP BY 절의 컬럼들로 구성된 유니크 인덱스를 가진 임시 테이블을 만들어서 중복 제거와 집합 함수 연산을 수행한다. <br>

```sql
CREATE TEMPORARY TABLE ... (
  last_name VARCHAR(16),
  salary INT,
  UNIQUE INDEX us_lastname (last_name)
);
```

그리고 조인의 결과를 한 건씩 가져와 임시 테이블에서 중복 체크를 하면서 INSERT 또는 UPDATE를 실행한다. <br>
즉, 별도의 정렬 작업 없이 GROUP BY가 처리된다. <br>
<br>

MySQL 8.0에서도 GROUP BY와 ORDER BY가 같이 사용되면 명시적으로 정렬 작업을 실행한다. <br>

```sql
mysql> EXPLAIN
         SELECT e.last_name, AVG(s.salary)
         FROM employees e, salaries s
         WHERE s.emp_no=e.emp_no
         GROUP BY e.last_name
         ORDER BY e.last_name;
```

![image](https://github.com/user-attachments/assets/ce8e3bd1-b94f-4515-9e7e-9982308de94a) <br>

<br>
