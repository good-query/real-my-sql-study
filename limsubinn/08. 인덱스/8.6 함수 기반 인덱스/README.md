## 8.6 함수 기반 인덱스
일반적인 인덱스는 컬럼의 값 앞부분 또는 전체에 대해서만 인덱스 생성이 허용된다. <br>
때로는 컬럼의 값을 변형해서 만들어진 값에 대해 인덱스를 구축해야 할 때도 있는데, 이러한 경우 함수 기반의 인덱스를 활용하면 된다. <br>

MySQL 서버의 함수 기반 인덱스는 인덱싱할 값을 계산하는 과정의 차이만 있을 뿐, <br>
실제 인덱스의 내부적인 구조 및 유지관리 방법은 B-Tree 인덱스와 동일하다.

<br>

### 8.6.1 가상 컬럼을 이용한 인덱스
다음과 같이 사용자의 정보를 저장하는 테이블이 있다고 해보자.
```sql
mysql> CREATE TABLE user (
          user_id BIGINT,
          first_name VARCHAR(10),
          last_name VARCHAR(10),
          PRIMARY KEY (user_id)
       );
```

이때 `first_name`과 `last_name`을 합쳐서 검색해야 하는 요건이 생겼다면 <br>
MySQL 8.0부터 다음과 같이 가상 컬럼을 추가하고 그 가상 컬럼에 인덱스를 생성할 수 있다.
```sql
mysql> ALTER TABLE user
          ADD full_name VARCHAR(30) AS (CONCAT(first_name, ' ', last_name)) VIRTUAL,
          ADD INDEX ix_fullname (full_name);
```
- `VIRTUAL`(기본값): 값을 물리적으로 저장하지 않고, **읽을 때마다 계산**한다. 디스크 공간을 사용하지 않는다.
- `STORED`: **계산된 값**을 테이블에 실제로 저장한다. 디스크 공간을 사용하지만, 읽기 속도가 빠르다.

이제부터는 `full_name` 컬럼에 대한 검색도 새로 만들어진 `ix_fullname` 인덱스를 이용해 실행 계획이 만들어진 것을 확인할 수 있다.
```sql
mysql> EXPLAIN SELECT * FROM user WHERE full_name = 'Matt Lee';
```
![image](https://github.com/user-attachments/assets/a1ce97da-b04f-47e7-ac02-62f529485a96)

가상 컬럼은 테이블에 새로운 컬럼을 추가하는 것과 같은 효과를 내기 때문에 실제 테이블의 구조가 변경된다는 단점이 있다. 

<br>

### 8.6.2 함수를 이용한 인덱스
MySQL 8.0부터 테이블의 구조를 변경하지 않고, 함수를 직접 사용하는 인덱스를 생성할 수 있다.
```sql
mysql> CREATE TABLE user (
          user_id BIGINT,
          first_name VARCHAR(10),
          last_name VARCHAR(10),
          PRIMARY KEY (user_id),
          INDEX ix_fullname ((CONCAT(first_name, ' ', last_name)))
       );
```
함수를 직접 사용하는 인덱스는 테이블의 구조는 변경하지 않고, 계산된 결괏값의 검색을 빠르게 만들어준다. <br>
함수 기반 인덱스를 제대로 활용하려면 반드시 조건절에 함수 기반 인덱스에 명시된 표현식이 그대로 사용되어야 한다.
```sql
mysql> EXPLAIN SELECT * FROM user WHERE CONCAT(first_name, ' ', last_name) = 'Matt Lee';
```
![image](https://github.com/user-attachments/assets/33f14a20-c0fa-41fc-896e-e47b5b2f2925) 

<br>

❗️ 옵티마이저가 표시하는 실행 계획이 `ix_fullname` 인덱스를 사용하지 않는 것으로 표시되는 경우 <br>
- CONCAT 함수에 사용된 공백 문자 리터럴 때문일 가능성이 높다.
- `collation_connection`, `collation_database`, `collation_server` 시스템 변수의 값을 동일 콜레이션으로 일치시키자. <br>
  ![image](https://github.com/user-attachments/assets/73405d2a-89db-497f-aa5b-9aa26f9be211)
