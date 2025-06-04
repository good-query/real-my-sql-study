## 8.7 멀티 밸류 인덱스
전문 검색 인덱스를 제외한 모든 인덱스는 레코드 1건이 1개의 인덱스 키 값을 가진다. <br>
멀티 밸류 인덱스는 하나의 데이터 레코드가 여러 개의 키 값을 가질 수 있는 형태의 인덱스다. <br>
최근 RDBMS들이 JSON 데이터 타입을 지원하면서 JSON의 배열 타입의 필드에 저장된 원소들에 대한 인덱스 요건이 발생한 것이다. <br>

다음과 같이 신용 정보 점수를 배열로 JSON 타입 컬럼에 저장하는 테이블을 가정해보자.
```sql
mysql> CREATE TABLE user (
          user_id BIGINT AUTO_INCREMENT PRIMARY KEY,
          first_name VARCHAR(10),
          last_name VARCHAR(10),
          credit_info JSON,
          INDEX mx_creditscores((CAST(credit_info->'$.credit_scores' AS UNSIGNED ARRAY)))
       );

mysql> INSERT INTO user VALUES (1, 'Matt', 'Lee', '{"credit_scores":[360, 353, 351]}');
```
<br>

멀티 밸류 인덱스를 활용하기 위해서는 일반적인 조건 방식을 사용하면 안 되고, <br>
반드시 다음 함수들을 이용해서 검색해야 옵티마이저가 인덱스를 활용한 실행 계획을 수립한다. <br>
| 함수                                             | 용도                                                | 입력 형태                | 반환값                | 멀티밸류 인덱스 활용 여부                               |
| :---------------------------------------------- | :------------------------------------------------- | :-------------------- | :------------------ | :-------------------------------------------- |
| `MEMBER OF(<스칼라>, <JSON_배열>)`                  | JSON 배열 내부에 특정 스칼라(문자/숫자/불린)가 있나?                 | 스칼라 값 × JSON 배열      | BOOL (1/0)         | 가능: 배열을 스캔하지 않고 인덱스으로 빠르게 매치                 |
| `JSON_CONTAINS(<target>, <candidate>[, path])` | JSON 문서(배열/객체) 안에 특정 부분(JSON 조각 혹은 스칼라)이 포함되어 있나? | JSON × JSON (또는 스칼라) | BOOL (1/0)         | 가능: 멀티밸류 인덱스로 부분 검색 (경로(path) 지정 시에도 인덱스 활용) |
| `JSON_OVERLAPS(<json1>, <json2>)`              | 두 JSON(배열 또는 객체) 간 교집합이 하나라도 있나?                  | JSON × JSON          | BOOL (1/0 or NULL) | 가능: 두 멀티밸류 인덱스 간의 교차 인덱스 스캔으로 빠른 교집합 검사      |
<br>

이제 신용 점수를 검색하는 쿼리를 한번 살펴보자.
```sql
mysql> SELECT * FROM user WHERE 360 MEMBER OF(credit_info->'$.credit_scores');
```
![image](https://github.com/user-attachments/assets/649c93b1-a798-407f-83e8-dc0a4185b60a)
```sql
mysql> EXPLAIN SELECT * FROM user WHERE 360 MEMBER OF(credit_info->'$.credit_scores');
```
![image](https://github.com/user-attachments/assets/dd7fabe6-6da5-45de-a04c-4e926596dc53)
