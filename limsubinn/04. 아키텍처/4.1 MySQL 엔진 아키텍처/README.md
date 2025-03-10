## 4.1 MySQL 엔진 아키텍처

### 4.1.1 MySQL의 전체 구조
<img width="600" alt="image" src="https://github.com/user-attachments/assets/cfe7ae41-e781-4dfc-a0e1-15b036e36ea6"> <br>
- MySQL은 대부분의 프로그래밍 언어로부터 접근 방법을 모두 지원한다. (e.g. JDBC, ODBC)
- MySQL 서버는 **MySQL 엔진**과 **스토리지 엔진**으로 구분할 수 있다.

<br>

**1️⃣ MySQL 엔진**
- 커넥션 핸들러(클라이언트로부터의 접속 및 쿼리 요청을 처리), SQL 파서 및 전처리기, 옵티마이저
- 표준 문법(ANSI SQL)에 따라 작성된 쿼리는 타 DBMS와 호환되어 실행될 수 있다.
- 요청된 SQL 문장을 분석하거나 최적화하는 등 DBMS의 두뇌에 해당하는 처리를 수행한다.
- MySQL 서버에서 MySQL 엔진은 하나이다.
  
<br>

**2️⃣ 스토리지 엔진**
- 실제 데이터를 디스크 스토리지에 저장하거나 디스크 스토리지로부터 데이터를 읽어온다.
- MySQL 서버에서 스토리지 엔진은 여러 개를 동시에 사용할 수 있다.
  - 테이블이 사용할 스토리지 엔진을 지정하면 이후 해당 테이블의 모든 읽기 작업이나 변경 작업은 정의된 스토리지 엔진이 처리한다.
    ```sql
    mysql> CREATE TABLE test_table (fd1 INT, fd2 INT) ENGINE=INNODB;
    ```
- 각 스토리지 엔진은 성능 향상을 위해 키 캐시(MyISAM 스토리지 엔진)나 InnoDB 버퍼 풀(InnoDB 스토리지 엔진)과 같은 기능을 내장하고 있다.

<br>

**3️⃣ 핸들러 API**
- MySQL 엔진의 쿼리 실행기에서 데이터를 쓰거나 읽어야 할 때 각 스토리지 엔진에 쓰기 또는 읽기를 요청한다. 여기서 사용되는 API를 핸들러 API라고 한다.
- 얼마나 많은 데이터 작업이 있었는지 확인할 수 있다.
  ```sql
  mysql> SHOW GLOBAL STATUS LIKE 'Handler%';
  ```
  ![image](https://github.com/user-attachments/assets/a9b4544a-b43e-47fc-9191-03340403a169)

