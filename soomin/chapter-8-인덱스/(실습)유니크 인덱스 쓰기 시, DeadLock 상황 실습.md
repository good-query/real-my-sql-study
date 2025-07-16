---
tags:
  - 데이터베이스
  - cs
parent: "[[유니크 인덱스 vs 非 유니크 인덱스 성능 비교]]"
related: 
created: 2025-06-18
---

# 0. 학습 목적
- 유니크 인덱스 사용 시, 데드락 상황이 빈번하게 발생한다고 하는데, 어떤 경우에 그렇게 되는지 확인한다.

# 1. 실습
1. 유니크 인덱스가 존재하는 테이블 생성
2. 해당 테이블에 하나의 행을 삽입
3. 두 개의 세션에서 트랜잭션을 시작하여, 모두 2번에서 삽입한 행을 삭제하고 새로 쓰는 작업을 진행. 

## (0) 세팅
```sql
use test_db;
drop table if exists t;

CREATE TABLE t (
a INT AUTO_INCREMENT,
b INT,
PRIMARY KEY (a),
UNIQUE KEY (b)
);

insert into t(a,b) values (100, 8);
```

## (1) Session 1
```sql
-- 트랜잭션 시작
START TRANSACTION;
-- Deadlock을 유발하는 REPLACE 실행 (a=10, b=8)
REPLACE INTO t(a, b) VALUES (10, 8);

-- (커밋은 하지 말고 대기, 또는 커밋 시도)
rollback;
```

## (2) Session 2
```sql
use test_db;
-- 트랜잭션 시작
START TRANSACTION;
-- Deadlock을 유발하는 REPLACE 실행 (a=11, b=8)
REPLACE INTO t(a, b) VALUES (11, 8);

-- (여기서 Deadlock 에러 발생 가능)
```

## (3) 결과
![image.png](https://raw.githubusercontent.com/dalcheonroadhead/img-cloud/main/2025-06/20250618154936.png)

deadLock 전용 에러는 안 났지만, 첫 번째 세션이 락을 잡고 있어서 두 번째 세션의 트랜잭션이 락 획득을 못한 채 시간초과 에러가 났다.

# 2. 핵심 요약
- 제대로 실습을 한 것인지 의문이다. 

---

# Metadata

### A. 모르는 단어 정리 
- `replace table(column, column2) into (a,b)`
	1. (a,b)의 새로운 행을 삽입하려고 시도
	2. a나 b가 PK나 유니크 인덱스로 쓰이는 행인지 확인, 맞다면 a,b로 이미 값이 존재하는지 확인
		1. 존재한다면, 기존에 존재하던 행 삭제 후 신규 삽입 
		2. 없으면 그대로 insert

###  B. 참고 문서

### C. 관련 글

```dataview
TABLE without id file.inlinks AS "BackLink"
WHERE file.path = this.file.path
```
