## 8.10 외래키
MySQL에서 외래키는 InnoDB 스토리지 엔진에서만 생성할 수 있으며, <br>
외래키 제약시 설정되면 자동으로 연관되는 테이블의 컬럼에 인덱스까지 생성된다. <br>
외래키가 제거되지 않은 상태에서는 자동으로 생성된 인덱스를 삭제할 수 없다. <br>

<br>

InnoDB의 외래키 관리에는 중요한 두 가지 특징이 있다.
1. 테이블의 변경(쓰기 잠금)이 발생하는 경우에만 잠금 경합(잠금 대기)이 발생한다.
2. 외래키와 연관되지 않은 컬럼의 변경은 최대한 잠금 경합(잠금 대기)을 발생시키지 않는다.

<br>

```sql
mysql> CREATE TABLE tb_parent (
          id INT NOT NULL,
          fd VARCHAR(100) NOT NULL, PRIMARY KEY (id)
       ) ENGINE=InnoDB;

mysql> CREATE TABLE tb_child (
          id INT NOT NULL,
          pid INT DEFAULT NULL,
          fd VARCHAR(100) DEFAULT NULL,
          PRIMARY KEY (id),
          KEY ix_parentid (pid),
          CONSTRAINT child_ibfk_1 FOREIGN KEY (pid) REFERENCES tb_parent (id) ON DELETE CASCADE
       ) ENGINE=InnoDB;

mysql> INSERT INTO tb_parent VALUES (1, 'parent-1'), (2, 'parent-2');
mysql> INSERT INTO tb_child VALUES (100,  1, 'child-100');
```

위와 같은 테이블에서 언제 자식 테이블의 변경이 잠금 대기를 하고, 언제 부모 테이블의 변경이 잠금 대기를 하는지 예제로 살펴보자.

<br>

### 8.10.1 자식 테이블의 변경이 대기하는 경우
자식 테이블의 외래키 컬럼의 변경(`INSERT`, `UPDATE`)은 부모 테이블의 확인이 필요한데, <br>
이 상태에서 부모 테이블의 해당 레코드가 쓰기 잠금이 걸려 있으면 해당 쓰기 잠금이 해제될 때까지 기다리게 된다. <br>
이것이 InnoDB의 외래키 관리의 첫 번째 특징에 해당한다. <br>

자식 테이블의 외래키가 아닌 컬럼의 변경은 외래키로 인한 잠금 확장이 발생하지 않는다. <br>
이것이 InnoDB의 외래키의 두 번째 특징에 해당한다. <br>

![image](https://github.com/user-attachments/assets/81296f88-93dc-4ff3-a5ef-481625d62204) <br>

이 작업에서는 1번 커넥션에서 먼저 트랜잭션을 시작하고 부모 테이블에서 id가 2인 레코드에 UPDATE를 실행한다. <br>
이 과정에서 1번 커넥션이 부모 테이블에서 id가 2인 레코드에 대해 쓰기 잠금을 획득한다. <br>
그리고 2번 커넥션에서 자식 테이블의 외래키 컬럼인 pid를 2로 변경하는 쿼리를 실행하면 <br>
이 쿼리는 부모 테이블의 변경 작업이 완료될 때까지 대기한다. <br>
다시 1번 커넥션에서 ROLLBACK이나 COMMIT으로 트랜잭션을 종료하면 2번 커넥션의 대기 중이던 작업이 즉시 처리된다. <br>

<br>

### 8.10.2 부모 테이블의 변경 작업이 대기하는 경우
![image](https://github.com/user-attachments/assets/c667a0ec-dba2-4e4a-ad30-0d1cc902c10d) <br>

이 작업에서는 1번 커넥션에서 부모 키 "1"을 참조하는 자식 테이블의 레코드를 변경하면 <br>
자식 테이블의 레코드에 대해 쓰기 잠금을 획득한다. <br>
이 상태에서 2번 커넥션이 부모 테이블에서 id가 1인 레코드를 삭제하는 경우 <br>
이 쿼리는 자식 테이블의 레코드에 대한 쓰기 잠금이 해제될 때까지 기다려야 한다. <br>
이는 자식 테이블이 생성될 때 정의된 외래키의 특성(`ON DELETE CASCADE`) 때문에 <br>
부모 레코드가 삭제되면 자식 레코드도 동시에 삭제되는 식으로 작동하기 때문이다.
