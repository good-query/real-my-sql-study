## 3.3 비밀번호 관리

### 3.3.1 고수준 비밀번호

`validate_password` 컴포넌트를 이용하면 MySQL 서버에서 비밀번호의 유효성 체크 규칙을 적용할 수 있다. <br>
(MySQL 5.7까지는 플러그인 형태로 제공됐지만 MySQL 8.0부터는 컴포넌트 형태로 제공된다.)

```sql
# 컴포넌트 설치
mysql> INSTALL COMPONENT 'file://component_validate_password';

# 설치된 컴포넌트 확인
mysql> SELECT * FROM mysql.component;

# 컴포넌트에서 제공하는 시스템 변수 확인
mysql> SHOW GLOBAL VARIABLES LIKE 'validate_password%';
```

<img width="350" alt="image" src="https://github.com/user-attachments/assets/cfe79f71-0b3d-4db0-a1e3-39edd22f0824" />

- 정책 (`validate_password.policy`)
  - `LOW`: 비밀번호의 길이
  - `MEDIUM`: + 숫자, 대소문자, 특수문자의 배합 (기본값)
  - `STRONG`: + 금칙어 포함 여부

- 길이
  - `validate_password.length`: 전체
  - `validate_password.number_count`: 숫자
  - `validate_password.mixed_case_count`: 대소문자
  - `validate_password.special_char_count`: 특수문자

- 금칙어
  - `validate_password.dictionary_file` 시스템 변수에 금칙어들이 저장된 사전 파일 등록 <br>
    (⚠️ 사전 파일은 MySQL 서버가 접근할 수 있는 디렉터리에 저장해야 하며, MySQL이 해당 파일을 읽을 수 있도록 적절한 읽기 권한을 설정해야 한다.)
  - 금칙어들을 한 줄에 하나씩 기록해서 저장한 텍스트 파일
  - e.g)
    ```sql
    mysql> SET GLOBAL validate_password.dictionary_file='prohibitive_word.data';
    mysql> SET GLOBAL validate_password.policy='STRONG';
    ```

<br>

### 3.3.2 이중 비밀번호

하나의 계정에 대해 2개의 비밀번호를 동시에 설정할 수 있는데, 최근에 설정한 비밀번호가 Primary, 이전 비밀번호는 Secondary이다. <br>
이중 비밀번호를 사용하려면 기존 비밀번호 변경 구문에 `RETAIN CRUUENT PASSWORD` 옵션을 추가하면 된다.

```sql
# 새로운 비밀번호를 설정하면서 기존 비밀번호를 세컨더리로 설정
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'new_password' RETAIN CURRENT PASSWORD;

# 세컨더리 비밀번호 삭제
mysql> ALTER USER 'root'@'localhost' DISCARD OLD PASSWORD;
```
