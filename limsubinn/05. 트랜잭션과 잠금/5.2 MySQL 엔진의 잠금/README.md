## 5.2 MySQL 엔진의 잠금

MySQL에서 사용되는 잠금은 크게 **스토리지 엔진 레벨**과 **MySQL 엔진 레벨**로 나눌 수 있다. <br>
MySQL 엔진 레벨의 잠금은 모든 스토리지 엔진에 영향을 미치지만, 스토리지 엔진 레벨의 잠금은 스토리지 엔진 간 상호 영향을 미치지 않는다.

<br>

### 5.2.1 글로벌 락
`FLUSH TABLES WITH READ LOCK` 명령으로 획득할 수 있으며, MySQL에서 제공하는 잠금 가운데 가장 범위가 크다. <br>
한 세션에서 글로벌 락을 획득하면 다른 세션에서 SELECT를 제외한 대부분의 DDL이나 DML 문장을 실행하는 경우 글로벌 락이 해제될 때까지 해당 문장이 대기 상태로 남는다. <br>

글로벌 락이 영향을 미치는 범위는 MySQL 서버 전체이며, 작업 대상 테이블이나 데이터베이스가 달라도 동일한 영향을 미친다. <br>
여러 데이터베이스에 존재하는 MyISAM이나 MEMORY 테이블에 대해 mysqldump로 일관된 백업을 받아야 할 때는 글로벌 락을 사용한다. <br>

글로벌 락은 MySQL 서버의 모든 변경 작업을 멈춘다. <br>
하지만 InnoDB 스토리지 엔진의 사용이 일반화되면서 일관된 데이터 상태를 위해 모든 데이터 변경 작업을 멈출 필요가 없어졌다. (트랜잭션 지원) <br>
또한, MySQL 8.0부터 InnoDB가 기본 스토리지 엔진으로 채택되면서 조금 더 가벼운 글로벌 락인 **백업 락**이 도입되었다. 

```sql
mysql> LOCK INSTANCE FOR BACKUP;
--- 백업 실행
mysql> UNLOCK INSTANCE;
```
특정 세션에서 백업 락을 획득하면 모든 세션에서 테이블의 스키마나 사용자의 인증 관련 정보를 변경할 수 없게 된다.
- 데이터베이스 및 테이블 등 모든 객체 생성 및 변경, 삭제
- REPAIR TABLE 과 OPTIMIZE TABLE 명령
- 사용자 관리 및 비밀번호 변경

하지만 백업 락은 일반적인 테이블의 데이터 변경은 허용된다. <br>
백업 락은 정상적으로 복제는 실행되지만 백업의 실패를 막기 위해 DDL 명령이 실행되면 복제를 일시 중지하는 역할을 한다.

<br>

### 5.2.2 테이블 락
개별 테이블 단위로 설정되는 잠금이다.

<br>

**1️⃣ 명시적 테이블 락**
<br>

`LOCK TABLES table_name [ READ | WRITE ]` 명령으로 락을 획득하고, <br>
`UNLOCK TABLES table_name` 명령으로 락을 반납(해제)한다. <br>

명시적으로 테이블을 잠그는 작업은 글로벌 락과 동일하게 온라인 작업에 상당한 영향을 미치기 때문에 <br>
특별한 상황이 아니면 애플리케이션에서 사용할 필요가 거의 없다.

<br>

**2️⃣ 묵시적 테이블 락**
<br>

MyISAM이나 MEMORY 테이블에 데이터를 변경하는 쿼리를 실행하면 발생한다. <br>
MySQL 서버가 데이터가 변경되는 테이블에 잠금을 설정하고 데이터를 변경한 후 즉시 잠금을 해제하는 형태로 사용된다. <br>
즉, 쿼리가 실행되는 동안 자동으로 락이 획득됐다가 쿼리가 완료된 후 자동 해제된다. <br>

InnoDB 테이블의 경우 스토리지 엔진 차원에서 레코드 기반의 잠금을 제공하기 때문에 <br>
단순 데이터 변경 쿼리로 인해 묵시적인 테이블 락이 설정되지는 않는다. <br>
(대부분의 DML 쿼리에서는 무시되고 DDL의 경우에만 영향을 미친다.)

<br>

### 5.2.3 네임드 락
`GET_LOCK()` 함수를 이용해 임의의 문자열에 대해 잠금을 설정할 수 있다. <br>
이 잠금의 특징은 대상이 테이블이나 레코드 또는 AUTO_INCREMENT와 같은 데이터베이스 객체가 아니라는 것이다. <br>
단순히 사용자가 지정한 문자열(String)에 대해 획득하고 반납하는 잠금이다. <br>
네임드 락은 자주 사용되지는 않는다. <br>

예를 들어, 여러 클라이언트가 상호 동기화를 처리해야 할 때 네임드 락을 이용하면 쉽게 해결할 수 있다. (동시성 제어)
```sql
# "mylock"이라는 문자열에 대해 잠금을 획득한다.
# 이미 잠금을 사용 중이면 2초 동안만 대기한다. (2초 이후 자동 잠금 해제)
mysql> SELECT GET_LOCK('mylock', 2);

# "mylock"이라는 문자열에 대해 잠금이 설정되어 있는지 확인한다.
mysql> SELECT IS_FREE_LOCK('mylock');

# "mylock"이라는 문자열에 대해 획득했던 잠금을 반납(해제)한다.
mysql> SELECT RELEASE_LOCK('mylock');

# 모두 정상적으로 락을 획득하거나 해제한 경우 1, 아니면 NULL이나 0을 반환한다.
```

또한, 많은 레코드에 대해 복잡한 요건으로 레코드를 변경하는 트랜잭션에 유용하게 사용할 수 있다. <br>
배치 프로그램처럼 한꺼번에 많은 레코드를 변경하는 쿼리는 자주 데드락의 원인이 된다. <br>
이때 동일 데이터를 변경하거나 참조하느나 프로그램끼리 분류해서 네임드 락을 걸고 쿼리를 실행하면 해결할 수 있다. <br>

MySQL 8.0부터는 네임드 락을 중첩해서 사용할 수 있고, 현재 세션에서 획득한 네임드 락을 한 번에 모두 해제할 수도 있다.
```sql
mysql> SELECT GET_LOCK('mylock_1', 10);
--- mylock_1에 대한 작업 실행
mysql> SELECT GET_LOCK('mylock_2', 10);
--- mylock_2에 대한 작업 실행

mysql> SELECT RELEASE_LOCK('mylock_2');
mysql> SELECT RELEASE_LOCK('mylock_1');

# 동시에 모두 해제
mysql> SELECT REALEASE_ALL_LOCKS();
```

<br>

### 5.2.4 메타데이터 락
데이터베이스 객체(테이블, 뷰 등)의 이름이나 구조를 변경하는 경우 획득하는 잠금이다. (묵시적) <br>

`RENAME TABLE` 명령의 경우 원본 이름과 변경될 이름 모두 한꺼번에 잠금을 설정한다. <br>
```sql
mysql> RENAME TABLE rank TO rank_backup, rank_new TO rank;
```
위와 같이 하나의 명령문에 두 개의 RENAME 작업을 한꺼번에 실행하면 실제 애플리케이션에서 오류 없이 적용이 가능하지만,
```sql
mysql> RENAME TABLE rank TO rank_backup;
mysql> RENAME TABLE rank_new TO rank;
```
위와 같이 2개의 문장으로 나눠서 실행하면 아주 짧은 시간이지만 rank 테이블이 존재하지 않는 순간이 생긴다. <br>
그 순간에 실행되는 쿼리는 "Table not found 'rank'" 오류를 발생시킬 것이다. <br>

메타데이터 잠금과 InnoDB의 트랜잭션을 동시에 사용해야 하는 경우도 있다. <br>
예를 들어, INSERT만 실행되는 로그 테이블을 가정해보자. 이 테이블은 UPDATE와 DELETE가 없다. <br>
그런데 어느 날 이 테이블의 구조를 변경해야 할 요건이 발생했다. <br>
물론 DDL을 이용해서 변경할 수 있지만 MySQL 서버의 DDL은 단일 스레드로 작동하기 때문에 상당히 많은 시간이 소요될 것이고, <br>
시간이 너무 오래 걸리는 경우라면 언두 로그 증가, 누적된 DDL 버퍼의 크기 등 고민해야 할 문제가 많다. <br>
따라서 이때는 새로운 구조의 테이블을 먼저 생성하고 최근의 데이터까지는 id 값을 범위별로 나눠 여러 개의 스레드로 빠르게 복사한다.
```sql
# 기존 테이블
mysql> CREATE TABLE access_log (
         id BIGINT NOT NULL AUTO_INCREMENT,
         client_ip INT UNSIGNED,
         access_dttm TIMESTAMP,
         ...
         PRIMARY KEY(id)
       );

# 신규 테이블 생성
mysql> CREATE TABLE access_log_new (
         id BIGINT NOT NULL AUTO_INCREMENT,
         client_ip INT UNSIGNED,
         access_dttm TIMESTAMP,
         ...
         PRIMARY KEY(id)
       ) KEY_BLOCK_SIZE=4;

# 4개의 스레드를 이용해 id 범위별로 레코드를 신규 테이블로 복사
mysql_thread1> INSERT INTO access_log_new SELECT * FROM access_log WHERE id>=0 AND id<10000;
mysql_thread2> INSERT INTO access_log_new SELECT * FROM access_log WHERE id>=10000 AND id<20000;
mysql_thread3> INSERT INTO access_log_new SELECT * FROM access_log WHERE id>=20000 AND id<30000;
mysql_thread4> INSERT INTO access_log_new SELECT * FROM access_log WHERE id>=30000 AND id<40000;

# 트랜잭션을 autocommit 비활성화하여 실행 (BEGIN이나 START TRANSACTION으로 실행하면 안 됨)
mysql> SET autocommit = 0;

# 작업 대상 테이블 2개에 대해 테이블 쓰기 락을 획득
mysql> LOCK TABLES access_log WRITE, access_log_new WRITE;

# 남은 데이터 복사
# 이 시간 동안은 테이블의 잠금으로 인해 INSERT가 불가능하기 때문에 가능하면 미리 아주 최근 데이터까지 복사해두자.
mysql> SELECT MAX(id) as @MAX_ID FROM access_log_new;
mysql> INSERT INTO access_log_new SELECT * FROM access_log WHERE pk>@MAX_ID;
mysql> COMMIT;

# 새로운 테이블로 데이터 복사가 완료되면 RENAME 명령으로 새로운 테이블을 서비스로 투입
mysql> RENAME TABLE access_log TO access_log_old, access_log_new TO access_log;
mysql> UNLOCK TABLES;

# 불필요한 테이블 삭제
mysql> DROP TABLE access_log_old;
```
