## 9.3 고급 최적화
MySQL 서버의 옵티마이저가 실행 계획을 수립할 때 통계 정보와 옵티마이저 옵션을 결합해서 최적의 실행 계획을 수립하게 된다. <br>
옵티마이저 옵션은 크게 조인 관련된 옵티마이저 옵션과 옵티마이저 스위치로 구분할 수 있다. <br>

<br>

### 9.3.1 옵티마이저 스위치 옵션
`optimizer_switch` 시스템 변수를 이용해서 제어하는데, 여러 개의 옵션을 세트로 묶어서 설정하는 방식으로 사용한다. <br>
| 옵티마이저 스위치 이름 | 기본값 | 설명 |
|:---|:---|:---|
| batched_key_access | off | BKA 조인 알고리즘을 사용할지 여부 설정 |
| block_nested_loop | on | Block Nested Loop 조인 알고리즘을 사용할지 여부 설정 |
| engine_condition_pushdown | on | Engine Condition Pushdown 기능을 사용할지 여부 설정 |
| index_condition_pushdown | on | Index Condition Pushdown 기능을 사용할지 여부 설정 |
| use_index_extensions | on | Index Extension 최적화를 사용할지 여부 설정 |
| index_merge | on | Index Merge 최적화를 사용할지 여부 설정 |
| index_merge_intersection | on | Index Merge Intersection 최적화를 사용할지 여부 설정 |
| index_merge_sort_union | on | Index Merge Sort Union 최적화를 사용할지 여부 설정 |
| index_merge_union | on | Index Merge Union 최적화를 사용할지 여부 설정 |
| mrr | on | MRR 최적화를 사용할지 여부 설정 |
| mrr_cost_baed | on | 비용 기반의 MRR 최적화를 사용할지 여부 설정 |
| semijoin | on | 세미 조인 최적화를 사용할지 여부 설정 |
| firstmatch | on | FirstMatch 세미 조인 최적화를 사용할지 여부 설정 |
| loosescan | on | Loosescan 세미 조인 최적화를 사용할지 여부 설정 |
| materialization | on | Materialization 최적화를 사용할지 여부 설정 <br> (Materialization 세미 조인 최적화 포함) |
| subquery_materialization_cost_based | on | 비용 기반의 Materialization 최적화를 사용할지 여부 설정 |

<br>

옵티마이저 스위치 옵션은 글로벌과 세션별 모두 설정할 수 있는 시스템 변수이므로 <br>
MySQL 서버 전체적으로 또는 현재 커넥션에 대해서만 다음과 같이 설정할 수 있다. <br>
```sql
# MySQL 서버 전체적으로 옵티마이저 스위치 설정
mysql> SET GLOBAL optimizer_switch='index_merge=on,index_merge_union=on,...';

# 현재 커넥션의 옵티마이저 스위치만 설정
mysql> SET SESSION optimizer_switch='index_merge=on,index_merge_union=on,...';
```
또한, 다음과 같이 "SET_VAR" 옵티마이저 힌트를 이용해 현재 쿼리에만 설정할 수도 있다.
```sql
mysql> SELECT /*+ SET_VAR(optimizer_switch='condition_fanout_filter=off') */
       ...
       FROM ...
```

<br>

#### 9.3.1.1 MRR과 배치 키 액세스(mrr & batched_key_access)
MRR은 "Multi-Range Read"를 줄여서 부르는 이름인데, 매뉴얼에서는 DS-MRR(Disk Sweep Multi-Range Read)라고도 한다. <br>
MySQL 서버에서 지금까지 지원하던 조인 방식은 <br>
드라이빙 테이블의 레코드를 한 건 읽어 드리븐 테이블의 일치하는 레코드를 찾아 조인을 수행하는 Nested Loop Join이었다. <br>
MySQL 서버의 내부 구조상 조인 처리는 MySQL 엔진이 처리하지만, 실제 레코드를 검색하고 읽는 부분은 스토리지 엔진이 담당한다. <br>
이때 드라이빙 테이블의 레코드 건별로 드리븐 테이블의 레코드를 찾으면 레코드를 찾고 읽는 스토리지 엔진에서는 아무런 최적화를 수행할 수 없다. <br>

이러한 단점을 보완하기 위해 MySQL 서버는 조인 대상 테이블 중 하나로부터 레코드를 읽어 조인 버퍼에 버퍼링한다. <br>
즉, 드라이빙 테이블의 레코드를 읽어 드리븐 테이블과의 조인을 즉시 실행하지 않고 조인 대상을 버퍼링하는 것이다. <br>
조인 버퍼에 레코드가 가득 차면 MySQL 엔진은 버퍼링된 레코드를 스토리지 엔진으로 한 번에 요청한다. <br>
이렇게 함으로써 스토리지 엔진은 읽어야 할 레코드들을 데이터 페이지에 정렬된 순서로 접근해서 <br>
디스크의 데이터 페이지 읽기를 최소화할 수 있는 것이다. <br>
물론 데이터 페이지가 메모리(InnoDB 버퍼 풀)에 있다고 하더라도 버퍼 풀의 접근을 최소화할 수 있는 것이다. <br>

이러한 읽기 방식을 MRR이라고 하며, MRR을 응용해서 실행되는 조인 방식을 BKA(Batched Key Access) 조인이라고 한다. <br>
BKA 조인을 사용하게 되면 부가적인 정렬 작업이 필요해지면서 오히려 성능에 안 좋은 영향을 미치는 경우도 있기 때문에 <br>
BKA 조인 최적화를 기본적으로 비활성화되어 있다. <br>

<br>

#### 9.3.1.2 블록 네스티드 루프 조인(block_nested_loop)
MySQL 서버에서 사용되는 대부분의 조인은 Nested Loop Join인데, 조인의 연결 조건이 되는 컬럼에 모두 인덱스가 있는 경우 사용된다. <br>

```sql
mysql> EXPLAIN
        SELECT *
        FROM employees e
          INNER JOIN salaries s ON s.emp_no = e.emp_no
                     AND s.from_date <= NOW()
                     AND s.to_date >= NOW()
        WHERE e.first_name = 'Amor';
```
<img width="1047" height="82" alt="image" src="https://github.com/user-attachments/assets/e18a619a-a7b7-4884-b254-43762aebb2d7" /> <br>

이러한 형태의 조인은 중첩된 반복 명령을 사용하는 것처럼 작동한다고 해서 Nested Loop Join이라고 한다. <br>
```
for (row1 IN employees) {
  for (row2 IN salaries) {
    if (condition_matched) return (row1, ro2);
  }
}
```

네스티드 루프 조인과 블록 네스티드 루프 조인의 가장 큰 차이는 <br>
조인 버퍼가 사용되는지 여부와 조인에서 드라이빙 테이블과 드리븐 테이블이 어떤 순서로 조인되느냐다. <br>
조인 알고리즘에서 "Block"이라는 단어가 사용되면 조인용으로 별도의 버퍼가 사용됐다는 것을 의미하는데, <br>
조인 쿼리의 실행 계획에서 Extra 컬럼에 "Using Join buffer" 문구가 표시되면 그 실행 계획은 조인 버퍼를 사용한다는 것을 의미한다. <br>

조인은 드라이빙 테이블에서 일치하는 레코드의 건수만큼 드리븐 테이블을 검색하면서 처리된다. <br>
즉, 드라이빙 테이블은 한 번에 쭉 읽지만, 드리븐 테이블은 여러 번 읽는다는 것을 의미한다. <br>
예를 들어, 드라이빙 테이블에서 일치하는 레코드가 1,000건이었는데, 드리븐 테이블의 조인 조건이 인덱스를 이용할 수 없었다면 <br>
드리븐 테이블에서 연결되는 레코드를 찾기 위해 1,000번의 풀 테이블 스캔을 해야 한다. <br>
그래서 드리븐 테이블을 검색할 때 인덱스를 사용할 수 없는 쿼리는 상당히 느려지며, <br>
옵티마이저는 최대한 드리븐 테이블의 검색이 인덱스를 사용할 수 있게 실행 계획을 수립한다. <br>

그런데 어떤 방식으로든 드리븐 테이블의 풀 테이블 스캔이나 인덱스 풀 스캔을 피할 수 없다면 <br>
옵티마이저는 드라이빙 테이블에서 읽은 레코드를 메모리에 캐시한 후 드리븐 테이블과 이 메모리 캐시를 조인하는 형태로 처리한다. <br>
이때 사용되는 메모리의 캐시를 조인 버퍼라고 한다. <br>
조인 버퍼는 `join_buffer_size` 시스템 변수로 크기를 제한할 수 있으며, 조인이 완료되면 조인 버퍼는 바로 해제된다. <br>

두 테이블이 조인되는 다음 예제 쿼리에서 각 테이블에 대한 조건은 WHERE 절에 있지만, <br>
두 테이블 간의 연결 고리 역할을 하는 조인 조건은 없다. <br>
그래서 dept_emp 테이블에서 from_date > '2000-01-01' 인 레코드와 <br>
employees 테이블에서 emp_no < 109004 조건을 만족하는 레코드는 카테시안 조인을 수행한다. <br>
```sql
mysql> SELECT *
       FROM dept_emp de, employees e
       WHERE de.from_date > '1995-01-01' AND e.emp_no < 109004;
```

이 쿼리의 실행 계획을 살펴보면 dept_emp 테이블이 드라이빙 테이블이며, <br>
employees 테이블을 읽을 때는 조인 버퍼를 이용해 블록 네스티드 루프 조인을 한다는 것을 알 수 있다. <br>

*****

but .. hash join 이라고 뜸.

<img width="1103" height="84" alt="image" src="https://github.com/user-attachments/assets/a1b687ec-0fe1-4453-8674-eb28b5e43681" />

| 항목       | Block Nested Loop                             | Hash Join                                         |
| :-------- | :-------------------------------------------- | :------------------------------------------------ |
| 설명       | 외부 루프의 각 행에 대해 내부 테이블을 반복 검색함                 | 한 테이블의 데이터를 메모리에 해시 테이블로 저장하고, 다른 테이블과 해시 기반으로 매칭 |
| 성능       | 인덱스가 잘 있으면 효율적이나, 내부 테이블을 반복 스캔하므로 비효율적일 수 있음 | 일반적으로 대용량 조인에서 빠름, 메모리를 많이 사용                     |
| 사용 조건    | 보통 인덱스가 있고, 데이터 양이 많지 않을 때                    | MySQL 8.0 이상에서 옵티마이저가 판단하여 선택 (e.g. 인덱스 미사용, 대용량)  |
| MySQL 적용 | MySQL 5.x까지는 주로 Nested Loop                   | MySQL 8.0 이상부터는 Hash Join 도입됨                     |


그렇다고 한다 ..

*****

다시 돌아와서.. 

단계별로 잘라서 조인 버퍼가 어떻게 사용되는지 실행 내역을 한 번 살펴보자. 

1. dept_emp 테이블의 ix_fromdate 인덱스를 이용해 (from_date > '1995-01-01') 조건을 만족하는 레코드 검색
2. 조인에 필요한 나머지 컬럼을 모두 dept_emp 테이블로부터 읽어 조인 버퍼에 저장
3. employees 테이블의 프라이머리 키를 이용해 (emp_no < 109004) 조건을 만족하는 레코드 검색
4. 3번에서 검색된 결과(employees)에 2번의 캐시된 조인 버퍼의 레코드(dept_emp)를 결합해서 반환

<img width="770" height="478" alt="image" src="https://github.com/user-attachments/assets/1dd4b7e8-4e59-4edc-9dac-dbffc5bea8e4" />

중요한 점은 조인 버퍼가 사용되는 쿼리에서는 조인의 순서가 거꾸로인 것처럼 실행된다는 것이다. <br>
위에서 설명한 절차의 4번 단계가 employees 테이블의 결과를 기준으로 dept_emp 테이블의 결과를 결합한다는 것을 의미한다. <br>
실제 이 쿼리의 실행 계획상으로는 dept_emp 테이블이 드라이빙 테이블이 되고, employees 테이블이 드리븐 테이블이 된다. <br>
하지만 실제 드라이빙 테이블의 결과는 조인 버퍼에 담아두고, 드리븐 테이블을 먼저 읽고 조인 버퍼에서 일치하는 레코드를 찾는 방식으로 처리된다. <br>
일반적으로 조인이 수행된 후 가져오는 결과는 드라이빙 테이블의 순서에 의해 결정되지만, <br>
조인 버퍼가 사용되는 조인에서는 결과의 정렬 순서가 흐트러질 수 있음을 기억해야 한다. <br>

*****

MySQL 8.0.18 버전부터 해시 조인 알고리즘이 도입됐으며, <br>
MySQL 8.0.20 버전부터는 더이상 블록 네스티드 루프 조인은 사용되지 않고, 해시 조인 알고리즘이 대체되어 사용된다고 한다.

*****

#### ⭐️ 추가 내용: Hash Join
많은 양의 데이터를 조인해야 하는 경우에 주로 사용된다. <br>
비용 기반 옵티마이저를 사용할 때만 사용될 수 있는 조인 방식이며, `=` 조건을 사용하는 `INNER JOIN`에서만 사용될 수 있다. <br>

두 테이블을 조인할 때, 한쪽 테이블의 데이터를 해시 테이블로 만들어 놓고, 다른 테이블을 순회하면서 해시로 빠르게 매칭시키는 조인 방식이다. <br>
1. 작은 테이블(드리븐 테이블)을 메모리에 올린다.
2. 조인 키를 기준으로 해시 테이블을 만든다.
3. 다른 테이블(드라이빙 테이블)을 한 row씩 읽으면서 해시 테이블에서 매칭되는 row를 찾는다.

✅ 조인 조건에 인덱스가 없어도 빠른 조인이 가능하다. <br>
✅ 해시 테이블 생성으로 인해 메모리 사용량이 크다. <br>
✅ 범위 조건(`<`, `>`, `BETWEEN`)에서는 사용할 수 없다. <br>

<br>

#### 9.3.1.3 인덱스 컨디션 푸시다운(index_condition_pushdown)
MySQL 5.6부터 인덱스 컨디션 푸시다운 기능이 도입되었다. <br>

<br>

**1️⃣ 인덱스 컨디션 푸시다운이 작동하지 않을 때** <br>

```sql
# 인덱스 컨디션 푸시다운 기능 비활성화
mysql> SET optimizer_switch='index_condition_pushdown=off';
```

다음 쿼리에서 `last_name='Action'` 조건은 인덱스를 레인지 스캔으로 사용할 수 있다. <br>
하지만 `first_name LIKE '%sal'` 조건은 인덱스 레인지 스캔으로 검색해야 할 인덱스의 범위를 좁힐 수 없다. <br>
그래서 `last_name` 조건은 `ix_lastname_firstname` 인덱스의 특정 범위만 조회할 수 있는 조건이며, <br>
`first_name LIKE '%sal'` 조건은 데이터를 모두 읽은 후 사용자가 원하는 결과인지 하나씩 비교해보는 조건(체크 조건 or 필터링 조건)으로만 사용된다. <br>

```sql
mysql> ALTER TABLE employees
       ADD INDEX is_lastname_firstname (last_name, first_name);

mysql> SELECT * FROM employees
       WHERE last_name='Action' AND first_name like '%sal';
```

위 쿼리의 실행 계획을 확인해 보면 Extra 컬럼에 "Using where"가 표시된 것을 확인할 수 있다. <br>
이는 InnoDB 스토리지 엔진이 읽어서 반환해준 레코드가 인덱스를 사용할 수 없는 WHERE 조건에 일치하는지 검사하는 과정을 의미한다. <br>

<img width="1056" height="69" alt="image" src="https://github.com/user-attachments/assets/e7daf8aa-672d-400c-b72f-6d8bf783928b" /> <br>

그림은 `last_name='Action'` 조건으로 인덱스 레인지 스캔을 하고 테이블의 레코드를 읽은 후, <br>
`first_name LIKE '%sal'` 조건에 부합되는지 여부를 비교하는 과정이다. <br>
실제 테이블을 읽어서 3건의 레코드를 가져왔지만 그중 단 1건만 `first_name LIKE '%sal'` 조건에 일치했다. <br>
만약 `last_name='Action'` 조건에 일치하는 레코드가 10만 건인데, <br>
그중 단 1건만 `first_name LIKE '%sal'` 조건에 일치했다면 99,999건의 레코드 읽기가 불필요한 작업이 되어버린다. <br>

<img width="519" height="231" alt="image" src="https://github.com/user-attachments/assets/06a4f970-ea00-45e2-9572-7c1642d6c2bd" />

인덱스를 비교하는 작업은 실제 InnoDB 스토리지 엔진이 수행하지만 <br>
테이블의 레코드에서 first_name 조건을 비교하는 작업은 MySQL 엔진이 수행하는 작업이다. <br>
그런데 MySQL 5.5까지는 인덱스를 범위 제한 조건으로 사용하지 못하는 first_name 조건은 MySQL 엔진이 스토리지 엔진으로 아예 전달해주지 않았다. <br>
그래서 스토리지 엔진에서는 불필요한 2건의 테이블 읽기를 수행할 수밖에 없었다. <br>

<br>

**2️⃣ 인덱스 컨디션 푸시다운이 작동할 때** <br>

```sql
# 인덱스 컨디션 푸시다운 기능 활성화
mysql> SET optimizer_switch='index_condition_pushdown=on';
```

MySQL 5.6부터는 인덱스를 범위 제한 조건으로 사용하지 못한다고 하더라도 <br>
인덱스에 포함된 컬럼의 조건이 있다면 모두 같이 모아서 스토리지 엔진으로 전달할 수 있게 핸들러 API가 개선되었다. <br>
그래서 인덱스를 이용해 최대한 필터링까지 완료해서 꼭 필요한 레코드 1건에 대해서만 테이블 읽기를 수행할 수 있게 되었다. <br>

<img width="523" height="228" alt="image" src="https://github.com/user-attachments/assets/e0f34b8e-ce0f-4dd0-830f-34df3ccd0891" /> <br>

<img width="1126" height="71" alt="image" src="https://github.com/user-attachments/assets/180dfc43-624c-438d-ad74-3511463e0ae7" /> <br>

<br>

#### 9.3.1.4 인덱스 확장(use_index_extensions)
InnoDB 스토리지 엔진을 사용하는 테이블에서 세컨더리 인덱스에 자동으로 추가된 프라이머리 키를 활용할 수 있게 할지를 결정하는 옵션이다. <br>

InnoDB 스토리지 엔진은 프라이머리 키를 클러스터링 키로 생성한다. <br>
그래서 모든 세컨더리 인덱스는 리프 노드에 프라이머리 키 값을 가진다. <br>

다음과 같이 프라이머리 키와 세컨더리 인덱스를 가진 테이블을 가정해보자. <br>

```sql
mysql> CREATE TABLE dept_emp (
         emp_no INT NOT NULL,
         dept_no CHAR(4) NOT NULL,
         from_date DATE NOT NULL,
         to_date DATE NOT NULL,
         PRIMARY KEY (dept_no, emp_no),
         KEY ix_fromdate (from_date)
       ) ENGINE=InnoDB;
```

dept_emp 테이블에서 프라이머리 키는 (dept_no, emp_no) 이며, 세컨더리 인덱스는 from_date 컬럼만 포함한다. <br>
그런데 세컨더리 인덱스는 데이터 레코드를 찾아가기 위해 프라이머리 키인 dept_no와 emp_no 컬럼을 순서대로 포함한다. <br>
그래서 최종적으로 ix_fromdate 인덱스는 (from_date, dept_no, emp_no) 조합으로 인덱스를 생성한 것과 흡사하게 작동할 수 있게 된다. <br>

예전 MySQL 버전에서는 다음과 같은 쿼리가 세컨더리 인덱스의 마지막에 자동 추가되는 프라이머리 키를 제대로 활용하지 못했지만 <br>
옵티마이저는 ix_fromdate 인덱스의 마지막에 (dept_no, emp_no) 컬럼이 숨어있다는 것을 인지하고 실행 계획을 수립하도록 개선됐다. <br>

```sql
mysql> EXPLAIN SELECT COUNT(*) FROM dept_emp WHERE from_date='1987-07-25' AND dept_no='d001';
```
<img width="1133" height="68" alt="image" src="https://github.com/user-attachments/assets/173c0037-ba83-4989-afe4-c366e5a91bb8" /> <br>

실행 계획의 key_len 컬럼은 이 쿼리가 인덱스를 구성하는 컬럼 중 어느 부분까지 사용했는지를 바이트 수로 보여주는데, <br>
19바이트가 표시된 것을 보면 from_date(3바이트), dept_no(16바이트) 까지 사용했다는 것을 알 수 있다. <br>

"dept_no='d001'" 조건을 제거한 쿼리의 실행 계획에서는 key_len 컬럼에서 from_date 컬럼을 위한 3바이트만 표시된 것을 확인할 수 있다. <br>

```sql
mysql> EXPLAIN SELECT COUNT(*) FROM dept_emp WHERE from_date='1987-07-25';
```
<img width="1033" height="68" alt="image" src="https://github.com/user-attachments/assets/c6434d2e-8a8e-4ac8-97fe-21604ff6758b" /> <br>

InnoDB의 프라이머리 키가 세컨더리 인덱스에 포함되어 있으므로 정렬 작업도 인덱스를 활용해 처리되는 장점도 있다. <br>
Extra 컬럼에 "Using filesort"가 표시되지 않았다는 것은 <br>
별도의 정렬 작업 없이 인덱스 순서대로 레코드를 읽기만 함으로써 ORDER BY 조건을 만족했다는 것을 의미한다. <br>

```sql
mysql> EXPLAIN SELECT COUNT(*) FROM dept_emp WHERE from_date='1987-07-25' ORDER BY dept_no;
```
<img width="1034" height="69" alt="image" src="https://github.com/user-attachments/assets/754e90fd-f2e3-4e8e-9900-9b00ce5d2f92" /> <br>

<br>

#### 9.3.1.5 인덱스 머지(index_merge)
인덱스를 이용해 쿼리를 실행하는 경우, 대부분 옵티마이저는 테이블별로 하나의 인덱스만 사용하도록 실행 계획을 수립한다. <br>
하지만 인덱스 머지 실행 계획을 사용하면 하나의 테이블에 대해 2개 이상의 인덱스를 이용해 쿼리를 처리한다. <br>

쿼리에서 한 테이블에 대한 WHERE 조건이 여러 개 있더라도 하나의 인덱스에 포함된 컬럼에 대한 조건만으로 인덱스를 검색하고 <br>
나머지 조건은 읽어온 레코드에 대해서 체크하는 형태로만 사용되는 것이 일반적이다. <br>
이처럼 하나의 인덱스만 사용해서 작업 범위를 충분히 줄일 수 있는 경우라면 테이블별로 하나의 인덱스만 활용하는 것이 효율적이다. <br>

하지만 쿼리에 사용된 각각의 조건이 서로 다른 인덱스를 사용할 수 있고 그 조건을 만족하는 레코드 건수가 많을 것으로 예상될 때 <br>
MySQL 서버는 인덱스 머지 실행 계획을 선택한다. <br>

- `index_merge_intersection`
- `index_merge_sort_union`
- `index_merge_union`

세 가지 실행 계획 모두 여러 개의 인덱스를 통해 결과를 가져온다는 것은 동일하지만 <br>
각각의 결과를 어떤 방식으로 병합할지에 따라 구분된다. <br>
index_merge 옵티마이저 옵션은 3개의 최적화 옵션을 한 번에 모두 제어할 수 있는 옵션이다. <br>

<br>

#### 9.3.1.6 인덱스 머지 - 교집합(index_merge_intersection)
다음 쿼리는 2개의 WHERE 조건을 가지고 있는데, <br>
employees 테이블의 first_name 컬럼과 emp_no 컬럼 모두 각각의 인덱스(ix_firstname, PRIMARY)를 가지고 있다. <br>
즉, 2개 중 어떤 조건을 사용하더라도 인덱스를 사용할 수 있다. <br>
그에 따라 옵티마이저는 ix_firstname과 PRIMARY 키를 모두 사용해서 쿼리를 처리하기로 결정한다. <br>

```sql
mysql> EXPLAIN SELECT *
       FROM EMPLOYEES
       WHERE first_name='Georgi' AND emp_no BETWEEN 10000 AND 20000;
```

<img width="1356" height="69" alt="image" src="https://github.com/user-attachments/assets/5bad4376-1fc5-4833-87bc-fed1164a015b" /> <br>

실행 계획의 Extra 컬럼에 "Using intersect"라고 표시된 것은 이 쿼리가 여러 개의 인덱스를 각각 검색해서 그 결과의 교집합만 반환했다는 것을 의미한다. <br>

first_name 컬럼의 조건과 emp_no 컬럼의 조건 중 하나라도 충분히 효율적으로 쿼리를 처리할 수 있었다면 <br>
옵티마이저는 2개의 인덱스를 모두 사용하는 실행 계획을 사용하지 않았을 것이다. <br>
즉, 옵티마이저가 각각의 조건에 일치하는 레코드 건수를 예측해 본 결과, 두 조건 모두 상대적으로 많은 레코드를 가져와야 한다는 것을 알게 된 것이다. <br>

```sql
# 결과 -> 53건
mysql> SELECT COUNT(*) FROM employees WHERE first_name='Georgi';

# 결과 -> 10000건
mysql> SELECT COUNT(*) FROM employees WHERE emp_no BETWEEN 10000 AND 20000;
```

인덱스 머지 실행 계획이 아니었다면 다음 2가지 방식으로 처리해야 했을 것이다. <br>

1. "first_name='Georgi'" 조건만 인덱스를 사용했다면 일치하는 레코드 253건을 검색한 다음 데이터 페이지에서 레코드를 찾고
   emp_no 컬럼의 조건에 일치하는 레코드들만 반환하는 형태로 처리
2. "emp_no BETWEEN 10000 AND 20000" 조건만 인덱스를 사용했다면 프라이머리 키를 이용해 10,000건을 읽어와서
   "first_name='Georgi'" 조건에 일치하는 레코드만 반환하는 형태로 처리

```sql
# 결과 -> 14건
mysql> SELECT COUNT(*) FROM employees
       WHERE first_name='Georgi' AND emp_no BETWEEN 10000 AND 20000;
```

실제 두 조건을 모두 만족하는 레코드 건수는 14건밖에 안 된다. <br>
따라서 하나의 인덱스만 사용하는 작업은 비효율이 매우 큰 상황이어서 옵티마이저는 각 인덱스를 검색해 두 결과의 교집합만 찾아서 반환한 것이다. <br>

그런데 ix_firstname 인덱스는 프라이머리 키인 emp_no 컬럼을 자동으로 포함하고 있기 때문에 <br>
그냥 ix_firstname 인덱스만 사용하는 것이 더 성능이 좋을 것으로 생각할 수도 있다. <br>
그렇다면 index_merge_intersection 최적화를 비활성화하면 된다. <br>

```sql
# MySQL 서버 전체적으로 index_merge_intersection 최적화 비활성화
mysql> SET GLOBAL optimizer_switch='index_merge_intersection=off';

# 현재 커넥션에 대해 index_merge_intersection 최적화 비활성화
mysql> SET SESSION optimizer_switch='index_merge_intersection=off';

# 현재 쿼리에서만 index_merge_intersection 최적화 비활성화
mysql> EXPLAIN
       SELECT /*+ SET_VAR(optimizer_switch='index_merge_intersection=off') */ *
       FROM employees
       WHERE first_name='Georgi' AND emp_no BETWEEN 10000 AND 20000;
```

<img width="1059" height="69" alt="image" src="https://github.com/user-attachments/assets/a472135e-1b1b-4326-b313-b9e4ef9077c5" /> <br>

#### 9.3.1.7 인덱스 머지 - 합집합(index_merge_union)
인덱스 머지의 'Using union'은 WHERE 절에 사용된 2개 이상의 조건이 <br>
각각의 인덱스를 사용하되 OR 연산자로 연결된 경우에 사용되는 최적화다. <br>

```sql
mysql> SELECT *
       FROM employees
       WHERE first_name='Matt' OR hire_date='1987-03-31';
```

위의 예제 쿼리는 2개의 조건이 OR로 연결되어 있다. <br>
employees 테이블에는 first_name 컬럼과 hire_date 컬럼에 각각 ix_firstname 인덱스와 ix_hiredate 인덱스가 있다. <br>
그래서 first_name='Matt'인 조건과 hire_date='1987-03-01' 조건이 각각의 인덱스를 사용할 수 있다. <br>
이 쿼리의 실행 계획은 다음과 같이 'Using union' 최적화를 사용한다. <br>

<img width="1416" height="71" alt="image" src="https://github.com/user-attachments/assets/0fdc52ae-476b-4816-9504-9af6427c2b4a" /> <br>

쿼리의 실행 계획에서 Extra 컬럼에 "Using union(ix_firstname, ix_hiredate)"라고 표시됐는데, <br>
이는 인덱스 머지 최적화가 ix_firstname 인덱스의 검색 결과와 ix_hiredate 인덱스 검색 결과를 'Union' 알고리즘으로 병합했다는 것을 의미한다. <br>

예제로 살펴본 쿼리에서 first_name='Matt'이면서 hire_date='1987-03-01'인 사원이 있었다면 <br>
그 사원의 정보는 ix_firstname 인덱스를 검색한 결과와 ix_hiredate 인덱스를 검색한 결과에 모두 포함되어 있었을 것이다. <br>
하지만 이 쿼리의 결과에서는 그 사원의 정보가 두 번 출력되지는 않는다. <br>
그렇다면 MySQL 서버는 어떻게 중복 제거를 수행했을까? <br>

<img width="500" alt="image" src="https://github.com/user-attachments/assets/4f28b8b8-5fa4-40c1-825b-3fd5360006a6" /> <br>

MySQL 서버는 first_name 조건을 검색한 결과와 hire_date 컬럼을 검색한 결과가 프라이머리 키로 이미 각각 정렬되어 있다는 것을 이미 알고 있다. <br>
그래서 두 집합에서 하나씩 가져와서 서로 비교하면서 프라이머리 키인 emp_no 컬럼의 값이 중복된 레코드들을 정렬 없이 걸러낼 수 있다. <br>
이렇게 정렬된 두 집합의 결과를 하나씩 가져와 중복 제거를 수행할 때 사용된 알고리즘을 우선순위 큐라고 한다. <br>

2개의 조건이 AND로 연결된 경우에는 두 조건 중 하나라도 인덱스를 사용할 수 있으면 인덱스 레인지 스캔으로 쿼리가 실행된다. <br>
하지만 OR 연산자로 연결된 경우에는 둘 중 하나라도 제대로 인덱스를 사용하지 못하면 항상 풀 테이블 스캔으로밖에 처리하지 못한다. <br>

<br>

#### 9.3.1.8 인덱스 머지 - 정렬 후 합집합(index_merge_sort_union)
인덱스 머지 작업을 하는 도중에 결과의 정렬이 필요한 경우 MySQL 서버는 인덱스 머지 최적화의 'Sort union' 알고리즘을 사용한다. <br>

```sql
mysql> EXPLAIN
         SELECT * FROM employees
         WHERE first_name='Matt'
               OR hire_date BETWEEN '1987-03-01' AND '1987-03-31';
```

위의 예제 쿼리를 다음과 같이 2개의 쿼리로 분리해서 생각해보자. <br>

```sql
mysql> SELECT * FROM employees WHERE first_name='Matt';
mysql> SELECT * FROM employees WHERE hire_date BETWEEN '1987-03-01' AND '1987-03-31';
```

첫 번째 쿼리는 결과가 emp_no로 정렬되어 출력되지만, <br>
두 번째 쿼리는 범위 조건이기 때문에 인덱스 스캔 시 같은 hire_date 값을 가진 여러 row가 있을 수 있고, 그 안에서의 emp_no 순서는 보장되지 않는다. <br>
즉, 위 예제의 결과에서는 중복을 제거하기 위해 우선순위 큐를 사용하는 것이 불가능하다. <br>
그래서 MySQL 서버는 두 집합의 결과에서 중복을 제거하기 위해 각 집합을 emp_no 컬럼으로 정렬한 다음 중복 제거를 수행한다. <br>

<img width="1448" height="70" alt="image" src="https://github.com/user-attachments/assets/8291ddbc-1861-4ba6-9ce4-61ab631f8378" /> <br>

이처럼 인덱스 머지 최적화에서 중복 제거를 위해 강제로 정렬을 수행해야 하는 경우에는 <br>
실행 계획의 Extra 컬럼에 "Using sort_union" 문구가 표시된다. <br>

<br>

#### 9.3.1.9 세미 조인(semijoin)
다들 테이블과 실제 조인을 수행하지는 않고, 단지 다른 테이블에서 조건에 일치하는 레코드가 있는지 없는지만 체크하는 형태의 쿼리이다. <br>

<br>

다음 쿼리는 세미 조인의 대표적인 예시이다. <br>

```sql
mysql> SELECT *
       FROM employees e
       WHERE e.emp_no IN
             (SELECT de.emp_no FROM dept_emp de WHERE de.from_date='1995-01-01');
```

MySQL 서버에서 세미 조인 최적화 기능이 없었을 때는 실행 계획이 다음과 같았다. <br>

<img width="1028" height="82" alt="image" src="https://github.com/user-attachments/assets/e75b2295-488f-4974-845e-c2f13686e8fd" /> <br>

employees 테이블을 풀 스캔하면서 한 건 한 건 서브쿼리의 조건에 일치하는지 비교했다. <br>
대략 57건만 읽으면 될 쿼리를 30만 건 넘게 읽어 처리되었다. <br>

<br>

`= (subquery)` 형태와 `IN (subquery)` 형태의 **세미 조인 쿼리**에 대해 3가지 최적화 방법을 적용할 수 있다. <br>
- 세미 조인 최적화
- IN-to-EXISTS 최적화
- MATERIALIZATION 최적화

<br>

`<> (subquery)` 형태와 `NOT IN (subquery)` 형태의 **안티 세미 조인 쿼리**에 대해 2가지 최적화 방법이 있다.
- IN-to-EXISTS 최적화
- MATERIALIZATION 최적화

<br>

MySQL 서버 8.0부터 세미 조인 쿼리의 성능을 개선하기 위해 다음의 최적화 전략이 있다. <br>
- Table Pull-out
- Dulicate Weed-out
- First Match
- Loose Scan
- Materialzation

<br>

쿼리에 사용되는 테이블과 조인 조건의 특성에 따라 MySQL 옵티마이저는 사용 가능한 전략들을 선별적으로 사용한다. <br>

Table Pull-out 최적화 전략은 사용 가능하면 항상 세민 조인보다 좋은 성능을 내기 때문에 별도로 제어하는 옵티마이저 옵션을 제공하지 않는다. <br>
그리고 firstmatch(First Match), loosescan(Loose Scan) 옵티마이저 옵션으로 사용 여부를 결정할 수 있고, <br>
materialization(Duplicate Weed-out, Materialization) 옵티마이저 스위치로 사용 여부를 선택할 수 있다. <br>

`optimizer_switch` 시스템 변수의 `semijoin` 옵티마이저 옵션은 firstmatch, lossescan, materialization 옵션을 한 번에 활성화/비활성화할 때 사용한다. <br>

<br>

#### 9.3.1.10 테이블 풀-아웃(Table Pull-out)
세미 조인의 서브쿼리에 사용된 테이블을 아우터 쿼리로 끄집어낸 후에 쿼리를 조인 쿼리로 재작성하는 형태의 최적화다. <br>

```sql
mysql> EXPLAIN
       SELECT * FROM employees e
       WHERE e.emp_no IN (SELECT de.emp_no FROM dept_emp de WHERE de.dept_no='d009');
```

<img width="1081" height="85" alt="image" src="https://github.com/user-attachments/assets/64d9d274-f34e-4dd0-822d-5e590dadf09f" /> <br>

해당 실행 계획에서 dept_emp 테이블과 employees 테이블이 순서대로 표시되어 있는데, 가장 중요한 것은 id 컬럼의 값이 모두 1이라는 것이다. <br>
Table pullout 최적화는 별도로 실행 계획의 Extra 컬럼에 "Using table pullout"과 같은 문구가 출력되지 않는다. <br>
그래서 실행 계획에서 해당 테이블들의 id 컬럼 값이 같은지 다른지를 비교해보는 것이 Table pullout 최적화가 사용됐는지를 확인하는 가장 간단한 방법이다. <br>
더 정확하게 확인하는 방법은 `EXPLAIN` 명령을 실행한 직후 `SHOW WARNINGS` 명령으로 MySQL 옵티마이저가 재작성한 쿼리를 살펴보는 것이다. <br>

<img width="785" height="144" alt="image" src="https://github.com/user-attachments/assets/5e59cddf-bb3b-4703-87ce-7f6a5a6f6959" /> <br>

이 쿼리를 보면 IN(subquery) 형태는 사라지고, JOIN으로 쿼리가 재작성된 것을 확인할 수 있다. <br>

Table pullout 최적화의 몇 가지 제한 사항과 특성이 있다. <br>
- 세미 조인 서브쿼리에서만 사용 가능하다.
- 서브쿼리 부분이 UNIQUE 인덱스나 프라이머리 키 룩업으로 결과가 1건인 경우에만 사용할 수 있다.
- Table pullout이 적용된다고 하더라도 기존 쿼리에서 가능했던 최적화 방법이 사용 불가능한 것은 아니므로
  MySQL에서는 가능하다면 Table pullout 최적화를 최대한 적용한다.
- 서브쿼리의 테이블을 아우터 쿼리로 가져와서 조인으로 풀어쓰는 최적화를 수행하는데,
  만약 서브쿼리의 모든 테이블이 아우터 쿼리로 끄집어 낼 수 있다면 서브쿼리 자체는 없어진다.
- MySQL에서는 "최대한 서브쿼리를 조인으로 풀어서 사용해라"라는 튜닝 가이드가 많은데,
  Table pullout 최적화는 이 가이드를 그대로 실행하는 것이다.

<br>

#### 9.3.1.11 퍼스트 매치(firstmatch)
IN(subquery) 형태의 세미 조인을 EXISTS(subquery) 형태로 튜닝한 것과 비슷한 방법으로 실행된다. <br>

```sql
mysql> EXPLAIN SELECT *
       FROM employees e WHERE e.first_name='Matt'
            AND e.emp_no IN (SELECT t.emp_no FROM titles t WHERE t.from_date BETWEEN '1995-01-01' AND '1995-01-30');
```

<img width="1245" height="86" alt="image" src="https://github.com/user-attachments/assets/f66b31fa-d2f3-41b8-ae0d-f5666c70d765" /> <br>

id 컬럼의 값이 모두 1로 표시된 것으로 봐서 titles 테이블이 서브쿼리 패턴으로 실행되지 않고, 조인으로 처리됐다는 것을 알 수 있다. <br>
또한, Extra 컬럼의 "FirstMatch(e)" 문구는 employees 테이블의 레코드에 대해 titles 테이블에 일치하는 레코드 1건만 찾으면 <br>
더이상의 titles 테이블 검색을 하지 않는다는 것을 의미한다. EXISTS(subquery)와 동일하게 처리된 것이다. <br>
하지만 FirstMatch는 서브쿼리가 아니라 조인으로 풀어서 실행하면서 일치하는 첫 번째 레코드만 검색하는 최적화를 실행한 것이다. <br>

<img width="800" alt="image" src="https://github.com/user-attachments/assets/448fe8d3-eb6e-4c8c-9882-9f132e8089c3" /> <br>

1. employees 테이블에서 first_name 컬럼의 값이 'Matt'인 사원의 정보를 ix_firstname 인덱스를 이용해 레인지 스캔으로 읽은 결과가 위의 그림 왼쪽에 있는 employees 테이블이다.
2. first_name이 'Matt'이고 사원 번호가 12302인 레코드를 titles 테이블과 조인해서 titles 테이블의 from_date가 "t.from_date BETWEEN '1995-01-01' AND '1995-01-30'" 조건을 만족하는 레코드를 찾아본다.
3. 12302번 사원은 from_date 조건을 만족하는 레코드가 없으므로 사용자에게 반환되는 결과는 없다.
4. 243075번 사원의 레코드를 읽어 titles 테이블과 조인하고 조인된 titles 레코드 중 from_date 조건을 만족하는지 검사한다.
5. 이때 일치하는 첫 번째 레코드를 찾았기 때문에 243075번 사원에 대해서는 더이상 titles 테이블을 검색하지 않고 즉시 사원 번호가 243075인 레코드를 최종 결과로 반환한다.

<br>

FirstMatch 최적화는 MySQL 5.5에서 수행했던 IN-to-EXISTS 변환과 비슷한 처리 로직을 수행하는데, 그에 비해 다음과 같은 장점이 있다. <br>
- 가끔 여러 테이블이 조인되는 경우 원래 쿼리에는 없던 동등 조건을 옵티마이저가 자동으로 추가하는 형태의 최적화가 실행되기도 한다.
- 서브쿼리의 모든 테이블에 대해 FirstMatch 최적화를 수행할지 혹은 일부 테이블에 대해서만 수행할지 취사선택할 수 있다.

<br>

FirstMatch 최적화의 몇 가지 제한 사항과 특성이 있다. <br>
- 서브쿼리에서 하나의 레코드만 검색되면 더이상의 검색을 멈추는 단축 실행 경로이기 때문에, FirstMatch 최적화에서 서브쿼리는 그 서브쿼리가 참조하는 모든 아우터 테이블이 먼저 조회된 이후 실행된다.
- 실행 계획의 Extra 컬럼에 "FirstMatch(table-N)" 문구가 표시된다.
- 상관 서브쿼리(Correlated subquery)에서도 사용될 수 있다.
- GROUP BY나 집합 함수가 사용된 서브쿼리의 최적화에 사용될 수 없다.

<br>

#### 9.3.1.12 루스 스캔(loosescan)
"Using index for group-by"의 루스 인덱스 스캔과 비슷한 읽기 방식을 사용한다. <br>

```sql
mysql> EXPLAIN
       SELECT * FROM departments d WHERE d.dept_no IN (SELECT de.dept_no FROM dept_emp de);
```

departments 테이블의 레코드 건수는 9건밖에 되지 않지만 dept_emp 테이블의 레코드 건수는 33만 건 가까이 저장되어 있다. <br>
그런데 dept_emp 테이블에는 (dept_no + emp_no) 컬럼의 조합으로 프라이머리 키 인덱스가 만들어져 있다. <br>
그리고 이 프라이머리 키는 전체 레코드 수는 33만 건 정도 있지만 dept_no 만으로 그루핑해서 보면 결국 9건밖에 없다는 것을 알 수 있다. <br>
그렇다면 dept_emp 테이블의 프라이머리 키를 루스 인덱스 스캔으로 유니크한 dept_no만 읽으면 아주 효율적으로 서브쿼리 부분을 실행할 수 있다. <br>

<img width="600" alt="image" src="https://github.com/user-attachments/assets/fab8b1eb-720b-4b74-8a36-9f7d0aade541" /> <br>

서브쿼리에 사용된 dept_emp 테이블이 드라이빙 테이블로 실행되며, dept_emp 테이블의 프라이머리 키를 dept_no 부분에서 유니크하게 한 건씩만 읽고 있다. <br>
루스 인덱스 스캔의 "Using index for group-by"도 위 그림에 표현된 dept_emp 테이블의 프라이머리 키를 읽는 방식과 동일하게 작동한다. <br>

<img width="1104" height="81" alt="image" src="https://github.com/user-attachments/assets/cff1de22-605f-4ff0-9a41-b2aca652ee6a" /> <br>

LooseScan 최적화는 다음과 같은 특성을 가진다. <br>
- 루스 인덱스 스캔으로 서브쿼리 테이블을 읽고, 그 다음으로 아우터 테이블을 드리븐으로 사용해서 조인을 수행한다.
  그래서 서브쿼리 부분이 루스 인덱스 스캔을 사용할 수 있는 조건이 갖춰져야 사용할 수 있는 최적화다.
  다음과 같은 형태의 서브쿼리들에서 사용할 수 있다.
  ```sql
  SELECT .. FROM .. WHERE expr IN (SELECT keypart1 FROM tab WHERE ...)
  SELECT .. FROM .. WHERE expr IN (SELECT keypart2 FROM tab WHERE keypart1='상수' ...)
  ```

<br>

#### 9.3.1.13 구체화(Materialization)
세미 조인에 사용된 서브쿼리를 통째로 구체화해서 쿼리를 최적화한다. 쉽게 표현하면 내부 임시 테이블을 생성한다는 것을 의미한다. <br>

```sql
mysql> EXPLAIN
       SELECT * FROM employees e
       WHERE e.emp_no IN (SELECT de.emp_no FROM dept_emp de WHERE de.from_date='1995-01-01');
```

<img width="1164" height="94" alt="image" src="https://github.com/user-attachments/assets/618d4aa9-ea11-4aac-b413-a8c59d20e2b1" /> <br>

마지막 라인의 selected_type 컬럼에 "MATERIALIZED"라고 표시됐다. <br>
이 쿼리에서 사용하는 테이블은 2개인데 실행 계획은 3개 라인이 출력된 것으로 보아 임시 테이블이 생성됐다는 것을 짐작할 수 있다. <br>
dept_emp 테이블을 읽는 서브쿼리가 먼저 실행되어 그 결과로 임시 테이블(<subquery2>)이 만들어졌다. <br>
그리고 최종적으로 서브쿼리가 구체화된 임시 테이블과 employees 테이블을 조인해서 결과를 반환한다. <br>

Materialization 최적화는 다른 서브쿼리 최적화와는 달리, 서브쿼리 내에 GROUP BY 절이 있어도 사용할 수 있다. <br>
```sql
mysql> EXPLAIN
       SELECT * FROM employees e
       WHERE e.emp_no IN (SELECT de.emp_no FROM dept_emp de WHERE de.from_date='1995-01-01' GROUP BY de.dept_no);
```

Materialization 최적화의 몇 가지 제한 사항과 특성이 있다. <br>
- IN(subquery)에서 서브쿼리는 상관 서브쿼리(Correlated subquery)가 아니어야 한다.
- 서브쿼리는 GROUP BY나 집합 함수들이 사용되어도 구체화를 사용할 수 있다.
- 구체화가 사용된 경우 내부 임시 테이블이 사용된다.
