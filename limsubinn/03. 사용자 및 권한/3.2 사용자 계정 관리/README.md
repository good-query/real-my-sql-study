## 3.2 사용자 계정 관리

### 3.2.1 시스템 계정과 일반 계정

MySQL 8.0부터 계정은 `SYSTEM_USER` 권한을 가지고 있느냐에 따라 시스템 계정과 일반 계정으로 구분된다.

- 시스템 계정
    - 데이터베이스 서버 관리자를 위한 계정
    - 시스템 계정과 일반 계정을 관리(생성, 삭제, 변경 및 권한 부여, 제거)
    - 다른 세션(Connection) 또는 그 세션에서 실행 중인 쿼리를 강제 종료
    - Stored 프로그램 생성 시 DEFINER를 타 사용자로 설정
- 일반 계정
    - 응용 프로그램이나 개발자를 위한 계정

<br>

MySQL 서버에는 다음과 같이 내장된 계정들이 있다. 이 계정들은 처음부터 잠겨있는 상태이다. (account_locked = ‘Y’)

- ‘mysql.sys’@’localhost’ : MySQL 8.0부터 기본으로 내장된 sys 스키마의 객체(뷰, 함수, 프로시저)들의 DEFINER로 사용되는 계정
- ‘mysql.session’@’localhost’ : MySQL 플러그인이 서버로 접근할 때 사용되는 계정
- ‘mysql.infoschema’@’localhost’ : information_schema에 정의된 뷰의 DEFINER로 사용되는 계정

<br>

### 3.2.2 계정 생성

MySQL 5.7까지는 `GRANT` 명령으로 권한의 부여와 동시에 계정 생성이 가능했다. <br>
하지만 MySQL 8.0부터는 계정의 생성은 `CREATE USER` 명령으로, 권한 부여는 `GRANT` 명령으로 구분해서 실행하도록 바뀌었다.

<br>

**1️⃣ IDENTIFIED WITH**

> 사용자의 인증 방식과 비밀번호를 설정한다.

- IDENTIFIED WITH 뒤에는 반드시 인증 방식(인증 플러그인의 이름)을 명시해야 하는데, 기본 인증 방식을 사용하고자 한다면 'password'로 명시해주면 된다.

- MySQL 5.7까지는 `Native Authentication`이 기본 인증 방식이었다.
  - 비밀번호에 대한 해시(SHA-1 알고리즘) 값을 저장해두고, 클라이언트가 보낸 값과 해시값이 일치하는지 비교하는 인증 방식이다.

- MySQL 8.0부터는 `Caching SHA-2 Authentication`이 기본 인증 방식이다.
  - 암호화 해시값 생성을 위해 SHA-2(256비트) 알고리즘을 사용하기 때문에 동일한 키 값에 대해서도 결과가 달라진다.
  - 이처럼 해시값을 계산하는 방식은 시간 소모적이라 성능이 떨어지는데, 이를 보완하기 위해 MySQL 서버는 해시 결과값을 메모리에 캐시해서 사용한다.
  - 이 인증 방식을 사용하려면 SSL/TLS 또는 RSA 키페어를 반드시 사용해야 한다.

- 만약 MySQL 8.0에서도 Native Authentication을 기본 인증 방식으로 설정하고자 한다면 다음과 같이 MySQL 설정을 변경하거나 my.cnf 설정 파일에 추가하면 된다.
  ```sql
  mysql> SET GLOBAL default_authentication_plugin="mysql_native_password"
  ```

<br>

**2️⃣ REQUIRE**

> MySQL 서버에 접속할 때 암호화된 SSL/TLS 채널을 사용할지 여부를 설정한다.

- 별도로 설정하지 않으면 비암호화 채널로 연결한다.

- REQUIRE 옵션을 SSL로 설정하지 않았다고 하더라도 Caching SHA-2 Authentication 인증 방식을 사용하면 암호화된 채널만으로 MySQL 서버에 접속할 수 있다.

<br>

**3️⃣ PASSWORD EXPIRE**

> 비밀번호의 유효기간을 설정한다.

- 설정 가능한 옵션은 다음과 같다.
  - `PASSWORD EXPIRE`: 계정 생성과 동시에 비밀번호의 만료 처리
  - `PASSWORD EXPIRE NEVER`: 없음
  - `PASSWORD EXPIRE DEFAULT`: default_password_lifetime 시스템 변수에 저장된 기간으로 설정 (설정하지 않을 시 기본값)
  - `PASSWORD EXPIRE INTERVAL n DAY`: 오늘부터 n일자로 설정

<br>

**4️⃣ PASSWORD HISTORY**

> 한 번 사용했던 비밀번호를 재사용하지 못하게 설정한다.

- mysql DB의 `password_history` 테이블을 사용하여 이전에 사용했던 비밀번호를 기억한다.

- 설정 가능한 옵션은 다음과 같다.
  - `PASSWORD HISTORY DEFAULT`: password_history 시스템 변수에 저장된 개수만큼 비밀번호의 이력을 저장하며, 저장된 이력에 남아있는 비밀번호는 재사용할 수 없다.
  - `PASSWORD HISTORY n DAY`: 비밀번호의 이력을 최곤 n개까지만 저장하며, 저장된 이력에 남아있는 비밀번호는 재사용할 수 없다.

<br>

**5️⃣ PASSWORD REUSE INTERVAL**

> 한 번 사용했던 비밀번호의 재사용 금지 기간을 설정한다.

- 설정 가능한 옵션은 다음과 같다.
  - `PASSWORD REUSE INTERVAL DEFAULT`: password_reuse_interval 변수에 저장된 기간으로 설정
  - `PASSWORD REUSE INTERVAL n DAY`: n일자 이후에 비밀번호를 재사용할 수 있게 설정

<br>

**6️⃣ PASSWORD REQUIRE**

> 비밀번호가 만료되어 새로운 비밀번호를 변경할 때 현재 비밀번호를 필요로 할지 여부를 설정한다.

- 설정 가능한 옵션은 다음과 같다.
  - `PASSWORD REQUIRE CURRENT`: 비밀번호를 변경할 때 현재 비밀번호를 먼저 입력하도록 설정
  - `PASSWORD REQUIRE OPTIONAL`: 비밀번호를 변경할 때 현재 비밀번호를 입력하지 않아도 되도록 설정
  - `PASSWORD REQUIRE DEFAULT`: password_require_current 시스템 변수의 값으로 설정

<br>

**7️⃣ ACCOUNT LOCK / UNLOCK**

> 계정 생성 또는 계정 정보 변경 시 계정을 사용하지 못하게 잠글지 여부를 설정한다.

