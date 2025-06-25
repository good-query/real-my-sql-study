## 7.2 keyring_file 플러그인 설치
TDE의 암호화 키 관리는 플러그인 방식으로 제공된다. <br>
그 중 MySQL 커뮤니티 에디션에서 사용 가능한 `keyring_file` 플러그인을 설치해보자.
<br>

keyring_file 플러그인은 테이블스페이스 키를 암호화하기 위한 마스터 키를 디스크의 파일로 관리하는데, **평문**으로 저장된다. <br>
따라서 외부에 파일이 노출되지 않도록 주의하자.
<br>

TDE 플러그인의 경우, MySQL 서버가 시작되는 단계에서 가장 빨리 초기화되어야 한다. <br>
MySQL 서버의 설정 파일(`my.cnf`)에서 keyring_file 플러그인을 위한 라이브러리와 마스터 키를 저장할 키링 파일의 경로를 명시해주자. <br>
`keyring_file_data` 설정 경로는 오직 하나의 MySQL 서버만 참조해야 한다.
```
early-plugin-load = keyring_file.so
keyring_file_data = /path/to/tde_master.key
```
<br>

MySQL 서버의 설정 파일이 준비되면 MySQL 서버 재시작 시 자동으로 keyring_file 플러그인이 초기화된다. <br>
이는 `SHOW_PLUGINS` 명령으로 확인 가능하다. 
<br>

keyring_file 플러그인이 초기화되면 keyring_file_data 시스템 변수의 경로에 빈 파일이 생성된다. <br>
이때 데이터 암호화 기능을 사용하는 테이블을 생성하거나 마스터 로테이션을 실행하면 키링 파일의 마스터 키가 초기화된다.

<br>

### 실습

![image](https://github.com/user-attachments/assets/fef2211b-5265-4251-86f3-609c4e364aad) <br>
나는 homebrew로 MySQL을 설치했기 때문에 `/opt/homebrew/etc/my.cnf`에 파일이 존재했다. <br>
이때 파일 내용을 보면 `/opt/homebrew/etc/my.cnf.d` 디렉토리를 포함하고 있다고 적혀있다.
<br>

![image](https://github.com/user-attachments/assets/6ba36d52-b511-484f-beff-9d019da7a2cc) <br>
따라서 `/opt/homebrew/etc/my.cnf.d/keyring.cnf` 파일을 생성해서 키링 파일 라이브러리와 경로를 설정해주었다. <br>
my.cnf 파일에 바로 작성해도 상관은 없다.

<br>

![image](https://github.com/user-attachments/assets/9242dbd2-ce06-4de4-ad9b-66ea8bee173f) <br>
이후 MySQL 서버를 재시작한다. <br>
나는 MariaDB로 설치되어 있었고 homebrew를 이용하여 설치했기 때문에 `brew services restart mariadb` 명령어를 사용하였다.

<br>

![image](https://github.com/user-attachments/assets/4ca16a4d-a630-4750-bace-96c919847510) <br>
하지만 눈을 씻고 찾아봐도 `keyring_file` 은 보이지 않는다... 왤까? <br>
검색해보니 homebrew로 mariadb를 설치하면 keyring_file.so 플러그인이 포함되지 않는다는 얘기가 있어서 . . <br>
`find /opt/homebrew -name 'keyring_file.so'` 명령어로 확인해보니 정말 아무것도 뜨지 않았다 . . . <br>
그래서 mariadb 제거하고, mysql로 다시 설치했다. 

<br>

![image](https://github.com/user-attachments/assets/c5f2ab98-5e17-4bdb-92e7-cd3b546b82ee) <br>
재설치 완. 다시 시작해보자. 하지만 여전히 없다. <br>
자세히 보니 `mysql --version`으로 확인했을 때의 버전과 실제 mysql을 실행했을 때 뜨는 버전이 달랐다. <br>
알고 보니 homebrew와 dmg 파일로 이중 설치가 되어 있었고 ^^ <br>
keyring은 homebrew에 설정하고, 서버는 dmg로 설치된 것을 실행하고 있었던 것.. <br>
8.x.x 버전 맞추기 위해 homebrew로 설치한 것을 모두 제거했다.

<br>

![image](https://github.com/user-attachments/assets/2eb0b4ea-3235-4a30-8842-8c539ba8f762) <br>
그리고 다시 써줬다. `user = root`는 mysql 서버가 자꾸 구동되지 않아서 지피티랑 싸우고 싸우다가 추가함..

<br>

![image](https://github.com/user-attachments/assets/21c860e4-ca74-4879-8155-56062a3194bb) <br>
한 시간 반 만에 드디어 떴다. 으아아아악

<br>

![image](https://github.com/user-attachments/assets/ac659470-1ab7-4435-9f74-5330b99a2fff) <br>
플러그인이 초기화되면서 빈 파일도 생성되었다. 

<br>

![image](https://github.com/user-attachments/assets/eb8998d6-a792-4089-a874-89e2c18dfdab) <br>
마스터 로테이션을 실행하고,

<br>

![image](https://github.com/user-attachments/assets/f16e583a-9162-45dd-b8c3-6921db7a4d2c) <br>
다시 파일을 확인해보면 키링 파일의 마스터 키가 초기화되면서 파일의 내용이 채워진 걸 볼 수 있다!


