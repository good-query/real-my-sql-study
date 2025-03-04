## 3.5 역할(Role)

MySQL 8.0부터 권한을 묶어서 역할(Role)을 사용할 수 있다.

<br>

✅ 역할의 사용 예시

1. `CREATE ROLE` 명령을 이용하여 역할 정의 (호스트를 명시하지 않은 경우 `%`가 자동으로 추가된다.)
   ```sql
   mysql> CREATE ROLE role_emp_read, role_emp_write;
   ```

2. `GRANT` 명령으로 각 역할에 대해 실질적인 권한을 부여
   ```sql
   mysql> GRANT SELECT ON employees.* TO role_emp_read;
   mysql> GRANT INSERT, UPDATE, DELETE ON employees.* TO role_emp_write;
   ```

3. 역할은 계정에 부여해야 하므로 `CREATE USER` 명령으로 계정을 생성하고, `GRANT` 명령으로 계정에 역할을 부여
   ```sql
   # 계정 생성
   mysql> CREATE USER reader@'127.0.0.1' IDENTIFIED BY 'qwerty';
   mysql> CREATE USER writer@'127.0.0.1' IDENTIFIED BY 'qwerty';

   # 역할 부여
   mysql> GRANT role_emp_read TO reader@'127.0.0.1';
   mysql> GRANT role_emp_read, role_emp_write TO writer@'127.0.0.1';
   ```

4. `SET ROLE` 명령으로 해당 역할을 활성화 => 계정이 로그아웃됐다가 다시 로그인하면 역할이 활성화되지 않은 상태로 초기화된다.
   ```sql
   mysql> SET ROLE 'role_emp_read';
   mysql> SET ROLE 'role_emp_write`;
   ```

5. `activate_all_roles_on_login` 시스템 변수를 설정하여 사용자가 MySQL 서버에 로그인할 때 역할을 자동으로 활성화할지 여부를 설정할 수 있다.
   ```sql
   mysql> SET GLOBAL activate_all_roles_on_login=ON;
   ```

<br>

✅ MySQL 서버 내부적으로 역할과 계정은 동일한 객체로 취급된다. 

- 그렇다면 MySQL 서버는 계정과 권한을 어떻게 구분할까?
  하나의 계정에 다른 계정의 권한을 병합하기만 하면 되므로 구분할 필요가 없다.

<br>

✅ 역할은 account_locked 컬럼의 값이 'Y'로 설정되어 있기 때문에 로그인 용도로 사용할 수 없다.

- CREATE USER 명령에 대해서는 권한이 없지만 CREATE ROLE 명령을 실행 가능한 사용자는 역할을 생성할 수 있고,
  이렇게 생성된 역할은 계정과 동일한 객체를 생성하지만 로그인 용도로 사용할 수 없기 때문에 보안을 강화하는 용도로 사용될 수 있다.

<br>

✅ mysql DB의 권한 관련 테이블

- `mysql.default_roles`: 계정별 기본 역할
  
- `mysql.role_edges`: 역할에 부여된 역할 관계 그래프
   
