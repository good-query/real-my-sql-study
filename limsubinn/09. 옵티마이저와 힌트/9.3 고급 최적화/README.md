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
