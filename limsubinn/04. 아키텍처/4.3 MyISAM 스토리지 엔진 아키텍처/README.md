## 4.3 MyISAM 스토리지 엔진 아키텍처

<img width="400" alt="image" src="https://github.com/user-attachments/assets/5c52a1a7-e671-47b2-b483-fa915de0e7f9"> <br>

<br>

### 4.3.1 키 캐시
InnoDB의 버퍼 풀과 비슷한 역할을 한다. <br>
하지만 이름 그대로 MyISAM 키 캐시는 인덱스만을 대상으로 작동하며, 인덱스의 쓰기 작업에 대해서만 부분적으로 버퍼링 역할을 한다.
```
키 캐시 히트율(Hit rate) = 100 - (Key_reads / Key_read_requests * 100)
```

- ```sql
  mysql> SHOW GLOBAL STATUS LIKE 'Key%';
  ```
  ![image](https://github.com/user-attachments/assets/d48e5bde-a12e-414d-971f-4006f00c126c)
  - `Key_blocks_not_flushed`: 아직 디스크에 플러시되지 않은 키 블록 수 → 변경되었지만 기록되지 않은 블록 수
  - `Key_blocks_unused`: 키 캐시에서 사용되지 않은 블록 수 → 현재 사용 가능한 빈 블록 수
  - `Key_blocks_used`: 캐시에서 현재 사용 중인 블록 수
  - `Key_read_requests`: 키 캐시로부터 인덱스를 읽은 횟수 (논리적 읽기)
  - `Key_reads`: 인덱스를 디스크에서 읽어 들인 횟수 (물리적 읽기) → 캐시 미스 시 디스크에서 읽는다.
  - `Key_write_requests`: 키 캐시에 쓰기를 시도한 횟수 (논리적 쓰기)
  - `Key_writes`: 디스크에 기록된 키 블록 수 (물리적 쓰기)



<br>

키 캐시를 이용한 쿼리의 비율(Hit rate)을 99% 이상으로 유지하는 것을 권장한다. <br> 
히트율이 99% 미만이라면 키 캐시를 조금 더 크게 설정하는 것이 좋다. <br>
32비트 운영체제에서는 하나의 키 캐시에 4GB 이상의 메모리 공간을 설정할 수 없고, <br>
64비트 운영체제에서는 `OS_PER_PROCESS_LIMIT` 값에 설정된 크기만큼의 메모리를 할당할 수 있다. <br>
제한 값 이상의 키 캐시를 할당하고 싶다면 기본(Default) 키 캐시 이외에 별도의 이름이 붙은 키 캐시 공간을 설정해야 한다. <br>
기본 키 캐시 공간을 설정하는 파라미터는 `key_buffer_size` 이다.

```
e.g.

key_buffer_size = 4GB
kbuf_board.key_buffer_size = 2GB
kbuf_comment.key_buffer_size = 2GB

=> 기본 키 캐시 4GB, kbuf_board 라는 이름의 키 캐시 2GB, kbuf_comment 라는 이름의 키 캐시 2GB가 설정된다.
```

<br>

기본 키 캐시 이외의 명명된 키 캐시 영역은 아무런 설정을 하지 않으면 메모리 할당만 해두고 사용하지 않는다. <br>
따라서 명명된 추가 키 캐시는 어떤 인덱스를 캐시할지 설정해야 한다.

```sql
mysql> CACHE INDEX db1.board, db2.board IN kbuf_board;
mysql> CACHE INDEX db1.comment, db2.comment IN kbuf_comment;
```

<br>

### 4.3.2 운영체제의 캐시 및 버퍼
MyISAM 테이블의 데이터에 대한 읽기나 쓰기 작업은 항상 운영체제의 작업으로 요청된다. <br>
운영체제의 캐시 공간은 남는 메모리를 사용한다. <br>
따라서 데이터베이스 MyISAM 테이블을 주로 사용한다면 운영체제가 사용할 수 있는 캐시 공간을 위해 충분한 메모리를 비워둬야 한다. <br>
MyISAM이 주로 사용되는 MySQL에서 일반적으로 키 캐시는 최대 물리 메모리의 40% 이상을 넘지 않게 설정하고, <br>
나머지 메모리 공간은 운영체제가 자체적인 파일 시스템을 위한 캐시 공간을 마련할 수 있게 해주는 것이 좋다.

<br>

### 4.3.3 데이터 파일과 프라이머리 키(인덱스) 구조
MyISAM 테이블은 프라이머리 키에 의한 클러스터링 없이 데이터 파일이 힙(Heap) 공간처럼 활용된다. <br>
따라서 레코드는 INSERT 되는 순서대로 데이터 파일에 저장된다. <br>
그리고 MyISAM 테이블에 저장되는 레코드는 모두 `ROWID` 라는 물리적인 주솟값을 가지는데, <br>
프라이머리 키와 세컨더리 인덱스는 모두 데이터 파일에 저장된 레코드의 ROWID 값을 포인터로 가진다.

- 고정 길이 ROWID
  - MyISAM 테이블을 생성할 때 `MAX_ROWS` 옵션을 명시하면 가질 수 있는 레코드의 개수가 한정된다.
  - ROWID 값으로 4바이트 정수를 사용한다.
  - 레코드가 INSERT된 순번이 ROWID로 사용된다.
- 가변 길이 ROWID
  - MyISAM 테이블을 생성할 때 `MAX_ROWS` 옵션을 설정하지 않으면 MyISAM 테이블의 ROWID는 최대 `myisam_data_pointer_size` 시스템 변수에 설정된 바이트 수만큼의 공간을 사용할 수 있다.
  - 기본값은 7이므로 MyISAM 테이블의 ROWID는 2바이트부터 7바이트까지 가변적인 ROWID를 갖게 되는데, 그 중 첫 번째 바이트는 ROWID의 길이를 저장하는 용도로 사용하고 나머지 공간은 실제 ROWID를 저장하는 데 사용된다.
  - 레코드의 위치(offset)가 ROWID로 사용된다.
