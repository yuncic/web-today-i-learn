# DB 실습 정답

---

## DDL 실습

### 문제 1: 테이블 생성하기 (CREATE TABLE)

**중복 컬럼**: `crew_id`와 `nickname`은 같은 크루가 출석할 때마다 반복 저장되므로 중복이 발생한다.

**crew 테이블 구성**: crew_id(PK), nickname 두 컬럼으로 구성한다.

```sql
-- 크루 정보 추출 (SELECT)
SELECT DISTINCT crew_id, nickname
FROM attendance
ORDER BY crew_id;

-- crew 테이블 생성 (CREATE)
CREATE TABLE crew (
  crew_id   INT          NOT NULL,
  nickname  VARCHAR(50)  NOT NULL,
  PRIMARY KEY (crew_id)
);

-- 크루 정보 삽입 (INSERT)
INSERT INTO crew (crew_id, nickname)
SELECT DISTINCT crew_id, nickname
FROM attendance
ORDER BY crew_id;
```

---

### 문제 2: 테이블 컬럼 삭제하기 (ALTER TABLE)

**불필요해지는 컬럼**: `nickname`. 크루 정보는 crew 테이블에서 조회할 수 있으므로 attendance에서 중복 보관할 이유가 없다.

```sql
ALTER TABLE attendance
DROP COLUMN nickname;
```

---

### 문제 3: 외래키 설정하기

attendance.crew_id가 crew.crew_id를 참조하도록 외래키 제약 조건을 건다.
존재하지 않는 crew_id로의 출석 기록 삽입·수정이 DB 레벨에서 차단된다.

```sql
ALTER TABLE attendance
ADD CONSTRAINT fk_attendance_crew
  FOREIGN KEY (crew_id) REFERENCES crew (crew_id);
```

---

### 문제 4: 유니크 키 설정

```sql
ALTER TABLE crew
ADD CONSTRAINT uq_crew_nickname
  UNIQUE (nickname);
```

---

## DML(CRUD) 실습

### 문제 5: 크루 닉네임 검색하기 (LIKE)

```sql
SELECT *
FROM crew
WHERE nickname LIKE '디%';
```

---

### 문제 6: 출석 기록 확인하기 (SELECT + WHERE)

```sql
SELECT *
FROM attendance
WHERE crew_id = (SELECT crew_id FROM crew WHERE nickname = '어셔')
  AND attendance_date = '2025-03-06';
```

---

### 문제 7: 누락된 출석 기록 추가 (INSERT)

crew 테이블에 어셔가 없다면 먼저 추가한 후 출석 기록을 삽입한다.

```sql
-- crew 테이블에 어셔 추가 (없는 경우)
INSERT INTO crew (crew_id, nickname)
VALUES (13, '어셔');

-- 출석 기록 삽입
INSERT INTO attendance (crew_id, attendance_date, start_time, end_time)
VALUES (13, '2025-03-06', '09:31', '18:01');
```

---

### 문제 8: 잘못된 출석 기록 수정 (UPDATE)

```sql
UPDATE attendance
SET start_time = '10:00'
WHERE crew_id  = (SELECT crew_id FROM crew WHERE nickname = '주니')
  AND attendance_date = '2025-03-12';
```

---

### 문제 9: 허위 출석 기록 삭제 (DELETE)

```sql
DELETE FROM attendance
WHERE crew_id  = (SELECT crew_id FROM crew WHERE nickname = '아론')
  AND attendance_date = '2025-03-12';
```

---

### 문제 10: 출석 정보 조회하기 (JOIN)

```sql
SELECT c.nickname, a.attendance_date, a.start_time, a.end_time
FROM attendance a
JOIN crew c ON a.crew_id = c.crew_id
ORDER BY c.crew_id, a.attendance_date;
```

---

### 문제 11: nickname으로 쿼리 처리하기 (서브쿼리)

```sql
SELECT *
FROM attendance
WHERE crew_id = (
  SELECT crew_id
  FROM crew
  WHERE nickname = '검프'
)
ORDER BY attendance_date;
```

---

### 문제 12: 가장 늦게 하교한 크루 찾기

```sql
SELECT c.nickname, a.end_time
FROM attendance a
JOIN crew c ON a.crew_id = c.crew_id
WHERE a.attendance_date = '2025-03-05'
ORDER BY a.end_time DESC
LIMIT 1;
```

---

## 집계 함수 실습

### 문제 13: 크루별로 '기록된' 날짜 수 조회

```sql
SELECT c.nickname, COUNT(*) AS record_count
FROM attendance a
JOIN crew c ON a.crew_id = c.crew_id
GROUP BY a.crew_id
ORDER BY a.crew_id;
```

---

### 문제 14: 크루별로 등교 기록이 있는 날짜 수 조회

```sql
SELECT c.nickname, COUNT(a.start_time) AS attended_count
FROM attendance a
JOIN crew c ON a.crew_id = c.crew_id
WHERE a.start_time IS NOT NULL
GROUP BY a.crew_id
ORDER BY a.crew_id;
```

---

### 문제 15: 날짜별로 등교한 크루 수 조회

```sql
SELECT attendance_date, COUNT(*) AS crew_count
FROM attendance
WHERE start_time IS NOT NULL
GROUP BY attendance_date
ORDER BY attendance_date;
```

---

### 문제 16: 크루별 가장 빠른 등교 시각(MIN)과 가장 늦은 등교 시각(MAX)

```sql
SELECT c.nickname,
       MIN(a.start_time) AS earliest_start,
       MAX(a.start_time) AS latest_start
FROM attendance a
JOIN crew c ON a.crew_id = c.crew_id
GROUP BY a.crew_id
ORDER BY a.crew_id;
```

---

## 생각해 보기

### 기본키란 무엇이고 왜 필요한가?

기본키(Primary Key)는 테이블 내 각 레코드를 고유하게 식별하는 컬럼(또는 컬럼 조합)이다. NOT NULL과 UNIQUE 제약이 함께 적용된다.

기본키가 없으면 특정 레코드를 정확히 지목할 수단이 없다. UPDATE나 DELETE 시 의도하지 않은 여러 행이 영향을 받을 수 있고, 외래키 참조의 대상이 될 수도 없다. attendance 테이블에서 검프의 3월 4일 기록만 수정하려 할 때 attendance_id가 없다면 crew_id + attendance_date 조합으로 찾아야 하는데, 같은 날 같은 크루의 기록이 중복 삽입된 경우 구분이 불가능해진다.

### MySQL에서 AUTO_INCREMENT는 왜 필요한가?

INSERT할 때마다 현재 최댓값을 조회해 +1한 값을 직접 지정하는 방식은 번거롭고, 동시 요청 환경에서는 두 세션이 같은 값을 읽어 중복 ID를 삽입할 수 있다. AUTO_INCREMENT는 DB 엔진이 원자적으로 다음 값을 채번하므로 중복 없이 고유 ID를 보장한다.

### NULL 값을 처리할 때 주의할 점

NULL은 "알 수 없음"이지 0이나 빈 문자열이 아니다. `NULL = NULL`은 TRUE가 아니라 NULL을 반환하므로 비교 시 반드시 `IS NULL` / `IS NOT NULL`을 써야 한다. COUNT(컬럼)은 NULL을 제외하고 세지만 COUNT(\*)는 NULL 행도 포함한다. 프론트엔드에서는 end_time이 NULL인 경우 "하교 미기록" 등으로 별도 표시하는 처리가 필요하다.

### crew와 attendance 테이블의 ER 다이어그램 / 일상 비유

crew(1) ─── attendance(N). crew 한 명이 여러 날 출석 기록을 가지는 1:N 관계다.

일상 비유: 도서관 회원(crew)과 대출 기록(attendance). 한 회원이 여러 권을 빌릴 수 있지만, 각 대출 기록은 반드시 한 명의 회원에게 귀속된다.

### 동시에 100명이 등교 버튼을 누른다면?

100개의 INSERT가 거의 동시에 발생한다. 트랜잭션 없이 처리하면 AUTO_INCREMENT 중복, 부분 삽입 후 오류로 인한 불완전 상태 등이 생길 수 있다.

ACID 관점에서 원자성(Atomicity)은 각 INSERT가 완전히 성공하거나 아예 반영되지 않아야 함을 보장한다. 격리성(Isolation)은 다른 트랜잭션이 아직 커밋되지 않은 삽입을 읽지 않도록 막는다. InnoDB의 행 잠금(row-level lock)이 이를 지원하므로 100개의 INSERT는 서로 독립적으로 안전하게 처리된다.

### 출석 데이터를 파일(CSV)이 아닌 DB에 저장하는 이유

파일 시스템으로 출석을 관리하면 다음 문제가 생긴다.

- 동시 접근: 두 코치가 동시에 CSV를 열고 저장하면 한쪽 변경이 덮어씌워진다.
- 일관성: 파일 저장 도중 프로세스가 죽으면 절반만 쓰인 파일이 남는다.
- 중복: 같은 데이터를 여러 파일에 흩어 저장하면 싱크가 틀어진다.
- 보안·권한 제어가 어렵다.

DB는 트랜잭션, 제약 조건, 인덱스, 접근 제어를 기본으로 제공하므로 이 문제를 구조적으로 해결한다.

### NoSQL(MongoDB)로 저장한다면?

```json
{
  "_id": ObjectId("..."),
  "crew_id": 1,
  "nickname": "검프",
  "attendance_date": "2025-03-04",
  "start_time": "09:45",
  "end_time": "18:10"
}
```

정규화 없이 하나의 도큐먼트에 모든 정보를 담는 구조가 된다. 크루 정보 변경 시(닉네임 수정 등) 모든 출석 도큐먼트를 갱신해야 한다는 단점이 있다. 반면 JOIN 없이 단일 도큐먼트로 조회가 완결되므로 읽기 성능이 단순한 경우에 유리하다. 출석처럼 스키마가 고정된 데이터는 RDBMS가 적합하고, 크루 프로필처럼 항목이 크루마다 달라질 수 있다면 NoSQL이 유연성 면에서 유리하다.

---

## 더 생각해 보기 (심화)

### 왜 nickname을 기본키로 하지 않았나? attendance_id가 존재하는 이유는?

nickname은 자연키(natural key)다. 자연키는 미래에 변경될 가능성이 있다. 우테코에서 닉네임 변경 정책이 생기거나 오타를 수정하는 경우, nickname을 기본키로 쓰면 crew 테이블뿐 아니라 attendance의 외래키, 인덱스까지 연쇄 갱신이 필요하다. crew_id처럼 업무적 의미가 없는 대리키(surrogate key)는 변경될 일이 없으므로 안정적이다.

attendance_id도 같은 이유다. 같은 크루가 같은 날 기록을 두 번 남기는 경우 crew_id + attendance_date 조합으로는 구분이 안 된다. 대리키가 있으면 레코드를 항상 정확히 하나 지목할 수 있다.

### RESTRICT와 CASCADE

외래키에서 참조 대상 레코드를 삭제·수정할 때의 동작 전략이다.

- RESTRICT (기본값): 참조 중인 레코드가 있으면 부모 레코드의 삭제·수정을 거부한다. crew에서 crew_id=1인 검프를 삭제하려 하면, attendance에 해당 crew_id가 남아 있는 한 오류가 발생한다.
- CASCADE: 부모가 삭제되면 자식 레코드도 자동으로 함께 삭제된다. 사용자 탈퇴 시 그 사용자의 게시글도 함께 지워야 하는 서비스라면 CASCADE가 적합하다. 반면 주문 기록처럼 사용자가 탈퇴해도 기록을 보존해야 하는 경우는 RESTRICT 또는 SET NULL을 쓴다.

### 서브쿼리 vs JOIN 성능 비교

```sql
-- 쿼리 1: 서브쿼리
SELECT * FROM attendance
WHERE crew_id IN (SELECT crew_id FROM crew WHERE nickname LIKE '네%');

-- 쿼리 2: JOIN
SELECT a.* FROM attendance a
JOIN crew c ON a.crew_id = c.crew_id
WHERE c.nickname LIKE '네%';
```

MySQL 옵티마이저는 대부분의 경우 두 쿼리를 동일한 실행 계획으로 변환한다. 다만 IN 서브쿼리는 서브쿼리 결과를 임시 집합으로 만든 뒤 비교하는 방식이라, 서브쿼리 결과 집합이 클수록 메모리 부담이 커질 수 있다. JOIN은 옵티마이저가 인덱스를 더 직접적으로 활용하기 쉬운 구조다. 소규모 데이터에서는 차이가 미미하지만, crew 테이블이 수만 건 이상이 된다면 JOIN이 일반적으로 유리하다.

### 정규화의 장점 vs 비정규화의 성능 이점

정규화하면 데이터 중복이 없어 갱신 이상(update anomaly)이 사라지고 저장 공간이 줄어든다. 닉네임 변경 시 crew 테이블 한 곳만 수정하면 된다.

비정규화는 JOIN을 줄여 읽기 쿼리 속도를 높인다. 출석 기록을 조회할 때마다 nickname이 필요하다면, attendance에 nickname을 중복 저장해두면 JOIN 없이 조회가 가능하다. 쓰기가 적고 읽기가 많은 집계용 테이블(OLAP)에서 비정규화를 의도적으로 적용하기도 한다.

### 연결 풀링(Connection Pooling)이란?

DB 연결(connection)을 맺고 끊는 과정은 네트워크 핸드셰이크, 인증 등을 포함해 비용이 크다. 수백 명이 동시 접근하면 매 요청마다 새 연결을 생성·소멸하는 것은 비효율적이다.

연결 풀(connection pool)은 미리 일정 수의 연결을 만들어 두고 재사용하는 캐시다. 요청이 들어오면 풀에서 유휴 연결을 꺼내 쓰고, 요청이 끝나면 닫지 않고 풀에 반납한다. 연결 생성 오버헤드가 없어 응답 시간이 줄고, 동시 연결 수를 제어해 DB 과부하를 방지한다.

### INSERT, UPDATE, DELETE를 하나의 트랜잭션으로 묶기

```sql
START TRANSACTION;

-- 문제 7: 어셔 출석 추가
INSERT INTO attendance (crew_id, attendance_date, start_time, end_time)
VALUES (13, '2025-03-06', '09:31', '18:01');

-- 문제 8: 주니 start_time 수정
UPDATE attendance
SET start_time = '10:00'
WHERE crew_id = (SELECT crew_id FROM crew WHERE nickname = '주니')
  AND attendance_date = '2025-03-12';

-- 문제 9: 아론 허위 기록 삭제
DELETE FROM attendance
WHERE crew_id = (SELECT crew_id FROM crew WHERE nickname = '아론')
  AND attendance_date = '2025-03-12';

COMMIT;
```

DELETE 도중 오류가 발생하면 앞의 INSERT와 UPDATE도 반영되지 않아야 한다. 원자성(Atomicity) 덕분에 트랜잭션 내 어느 단계에서든 오류가 발생하면 `ROLLBACK`이 수행되어 세 작업 모두 없던 일이 된다. 반대로 세 작업이 모두 성공해야 비로소 `COMMIT`으로 확정된다.
