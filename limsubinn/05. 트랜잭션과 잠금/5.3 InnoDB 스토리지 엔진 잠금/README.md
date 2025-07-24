## 5.3 InnoDB 스토리지 엔진 잠금
InnoDB 스토리지 엔진은 레코드 기반의 잠금 방식을 탑재하고 있다. <br>
덕분에 MyISAM보다 훨씬 뛰어난 동시성 처리를 제공한다. <br>

MySQL 최근 버전에서는 MySQL 서버의 `information_schema` 데이터베이스에 존재하는 <br>
`INNODB_TRX`, `INNODB_LOCKS`, `INNODB_LOCK_WAITS` 테이블을 조인해서 조회하면 <br>
현재 어떤 트랜잭션이 어떤 잠금을 대기하고 있고 해당 잠금을 어느 트랜잭션이 가지고 있는지 확인할 수 있으며, <br>
장시간 잠금을 가지고 있는 클라이언트를 찾아서 종료시킬 수도 있다. <br>
또한, `performance_schema`를 이용해 InnoDB 스토리지 엔진의 내부 잠금(세마포어)에 대한 모니터링 방법도 추가되었다. 

<br>

### 5.3.1 InnoDB 스토리지 엔진의 잠금

**1️⃣ 레코드 락**
<br>

레코드 자체만을 잠그는 것을 레코드 락이라고 한다. <br>
다른 상용 DBMS의 레코드 락과 동일한 역할을 하지만, InnoDB 스토리지 엔진은 레코드 자체가 아니라 인덱스의 레코드를 잠근다. <br>
인덱스가 없는 테이블이더라도 내부적으로 자동 생성된 클러스터 인덱스를 이용해 잠금을 설정한다. 

<br>

**2️⃣ 갭 락**
<br>

레코드와 바로 인접한 레코드 사이의 간격만을 잠그는 것을 갭 락이라고 한다. <br>
레코드와 레코드 사이의 간격에 새로운 레코드가 생성되는 것을 제어한다.

<br>

**3️⃣ 넥스트 키 락**
<br>

레코드 락과 갭 락을 합쳐 놓은 형태의 잠금을 넥스트 키 락이라고 한다. <br>
InnoDB의 갭 락이나 넥스트 키 락은 바이너리 로그에 기록되는 쿼리가 레플리카 서버에서 실행될 때 <br>
소스 서버에서 만들어 낸 결과와 동일한 결과를 만들어내도록 보장하는 것이 주목적이다. <br>

하지만 넥스트 키 락과 갭 락으로 인해 데드락이 발생하거나 다른 트랜잭션을 기다리게 만드는 일이 자주 발생한다. <br>
따라서 바이너리 로그 포맷을 ROW 형태로 바꿔서 해당 락을 줄이는 것이 좋다.

<br>

**4️⃣ 자동 증가 락**
<br>

MySQL에서는 `AUTO_INCREMENT`라는 컬럼 속성을 제공하는데, <br>
해당 컬럼이 사용된 테이블에 동시에 여러 레코드가 INSERT 되는 경우 <br>
저장되는 각 레코드는 중복되지 않고 저장된 순서대로 증가하는 일련번호 값을 가져야 한다. <br>
이를 위해 InnoDB 스토리지 엔진에서 내부적으로 AUTO_INCREMENT 락이라는 테이블 수준의 잠금을 사용한다. <br>

INSERT와 REPLACE 쿼리 문장과 같이 새로운 레코드를 저장하는 쿼리에서만 필요하며, UPDATE나 DELETE 등의 쿼리에서는 걸리지 않는다. <br>
또한, 트랜잭션과 관계없이 INSERT나 REPLACE 문장에서 AUTO_INCREMENT 값을 가져오는 순간만 락이 걸렸다가 즉시 해제된다. <br>

AUTO_INCREMENT 락은 테이블에 단 하나만 존재하기 때문에 두 개의 INSERT 쿼리가 동시에 실행되는 경우 <br>
하나의 쿼리가 AUTO_INCREMENT 락을 걸면 나머지 쿼리는 락을 기다려야 한다. <br>
또한, 명시적으로 값을 설정하더라도 자동 증가 락을 걸게 된다. <br>
해당 락을 명시적으로 획득하고 해제하는 방법은 없다. <br>

MySQL 5.1부터는 `innodb_autoinc_lock_mode` 시스템 변수를 이용해 자동 증가 락의 작동 방식을 변경할 수 있다.
- `0`: MySQL 5.0과 동일한 잠금 방식으로 모든 INSERT 문장은 자동 증가 락을 사용한다.
- `1`: 단순히 한 건 또는 여러 건의 레코드를 INSERT하는 SQL 중에서 MySQL 서버가 INSERT되는 레코드의 건수를 정확히 예측할 수 있을 때는 자동 증가 락을 사용하지 않고, 훨씬 가볍고 빠른 래치(뮤텍스)를 이용해 처리한다.
  아주 짧은 시간 동안만 잠금을 걸고 필요한 자동 증가 값을 가져오면 즉시 잠금이 해제 된다.
  이 설정에서는 최소한 하나의 INSERT 문장으로 INSERT 되는 레코드는 연속된 자동 증가 값을 가지게 되기 때문에 이 설정 모드를 연속 모드(Consecutive mode)라고도 한다.
- `2`: 절대 자동 증가 락을 걸지 않고 경량화된 래치(뮤텍스)를 사용한다.
  하지만 이 설정에서는 하나의 INSERT 문장으로 레코드라고 하더라도 연속된 자동 증가 값을 보장하지는 않기 때문에 이 설정 모드를 인터리빙 모드(Interleaved mode)라고도 한다.
  이 설정 모드에서는 대량 INSERT 문장이 실행되는 중에도 다른 커넥션에서 INSERT를 실행할 수 있으므로 동시 처리 성능이 높아진다.
  하지만 이 설정에서 작동하는 자동 증가 기능은 유니크한 값이 생성되는 것만 보장하기 때문에, STATEMENT 포맷의 바이너리 로그를 사용하는 복제에서는 소스 서버와 레플리카 서버의 자동 증가 값이 달라질 수도 있다.

MySQL 8.0부터는 `innodb_autoinc_lock_mode` 의 기본값이 2이다. (바이너리 로그 포맷의 기본값이 ROW 포맷이기 때문) <br>
STATEMENT 포맷의 바이너리 로그를 사용한다면 1로 변경하자.

자동 증가 값이 한 번 증가하면 절대 줄어들지 않는 이유는 AUTO_INCREMENT 잠금을 최소화하기 위해서이다. <br>
INSERT 쿼리가 실패했더라도 한 번 증가된 AUTO_INCREMENT 값은 다시 줄어들지 않고 그대로 남는다.

<br>

### 5.3.2 인덱스와 잠금
InnoDB의 잠금은 인덱스를 잠그는 방식으로 처리되기 때문에, 변경해야 할 레코드를 찾기 위해 검색한 인덱스의 레코드를 모두 락을 걸어야 한다. <br>

e.g.
```sql
# employees 테이블에 first_name 컬럼만 인덱스 설정되었다고 가정한다.

# 아래 실행 결과가 253건
mysql> SELECT COUNT(*) FROM employees WHERE first_name='Georgi';

# 아래 실행 결과가 1건
mysql> SELECT COUNT(*) FROM employees WHERE first_name='Georgi' AND last_name='Klassen';

# 변경 쿼리 실행
mysql> UPDATE employees SET hire_date=NOW() WHERE first_name='Georgi' AND last_name='Klassen';
```
해당 UPDATE 문장이 실행되면 1건의 레코드가 업데이트될 것이다. <br>
하지만 1건의 업데이트를 위해 253건의 레코드가 모두 잠긴다. (인덱스를 사용할 수 있는 조건이 first_name='Georgi') <br>

테이블에 인덱스가 하나도 없다면 테이블을 풀 스캔하면서 UPDATE 작업을 하는데, 이 과정에서 테이블에 있는 모든 레코드를 잠그게 된다. <br>
따라서 MySQL의 InnoDB에서 인덱스 설계가 매우 중요하다.

<br>

### 5.3.3 레코드 수준의 잠금 확인 및 해제
MySQL 5.1부터 레코드 잠금과 잠금 대기에 대한 조회가 가능하다. <br>
MySQL 8.0 버전부터는 `performance_schema`의 `data_locks`와 `data_lock_waits` 테이블을 조인해서 잠금 대기 순서를 조회할 수 있다.
```sql
mysql> SELECT
         r.trx_id wating_trx_id,
         r.trx_mysql_thread_id waiting_thread,
         r.trx_query waiting_query,
         b.trx_id blocking_trx_id,
         b.trx_mysql_thread_id blocking_thread,
         b.trx_query blocking_query
         FROM performance_schema.data_lock_waits w
         INNER JOIN information_schema.innodb_trx b
           ON b.trx_id = w.blocking_engine_transaction_id
         INNER JOIN information_schema.innodb_trx r
           ON r.trx_id = w.requesting_engine_transaction_id;
```
특정 스레드가 어떤 잠금을 가지고 있는지 더 상세히 확인하고 싶다면 `performance_schema`의 `data_locks` 테이블이 가진 컬럼을 모두 살펴보면 된다.
```sql
mysql> SELECT * FROM performance_schema.data_locks\G
```
특정 스레드를 강제 종료하면 잠금이 해제된다.
```sql
mysql KILL thread_no;
```
