# 6.1 기본 DML 튜닝

DML(INSERT/UPDATE/DELETE)

# 6.1.1 DML성능에 영향을 미치는요소

- 인덱스
- 무결성 제약
- 조건절
- 서브쿼리
- Redo로깅
- Undo로깅
- Lock
- 커밋

### 인덱스와 DML성능

INSERT

- 테이블 한 건 삽입 시, **관련 인덱스 개수만큼** 인덱스 리프 블록에 키 삽입.
- 삽입 위치 탐색(루트→브랜치→리프) + **리프 분할(split)** 가능성 + Redo/Undo 발생.
- 인덱스가 **1개 → N개**로 늘면, **키 삽입 작업이 N배수로 증가**(정확히 선형은 아니지만 경향은 선형에 가까움).

DELETE

- 테이블에서 행 삭제 + **모든 인덱스에서 해당 키 삭제**.
- 삭제 마크/실제 삭제/리프 재조정 등 부가 비용.

 UPDATE

- 변경 컬럼이 **인덱스에 포함**되면 = **삭제 1회 + 삽입 1회**(인덱스 입장에선 키 변경).
- 그림 6-2처럼 한 컬럼 바꿨을 뿐인데, **해당 컬럼을 포함한 모든 인덱스마다** 삭제/삽입이 반복됩니다.

```sql
-- 1) 소스 100만건 생성
create table source as
select /*+ materialize */
       rownum as no, a.*
from   (select * from emp where rownum <= 10) a,
       (select level as lvl from dual connect by level <= 100000);

-- 2) 타깃 생성 (인덱스: PK 1개)
create table target as select * from source where 1=2;
alter table target add constraint target_pk primary key(no);

-- 3) 시간 측정 후 적재
-- set timing on;  -- 클라이언트에 따라 다름
insert /*+ append */ into target select * from source;
commit;

-- 4) 타깃 비우고 보조 인덱스 2개 추가
truncate table target;
create index target_x1 on target(ename);
create index target_x2 on target(deptno, mgr);

-- 5) 동일 적재 반복 → 시간 비교
insert /*+ append */ into target select * from source;
commit;

```

1. **SOURCE 테이블**: 100만 건 데이터 준비
2. **TARGET 테이블**:
    - A단계: PK 하나만 생성한 상태에서 `INSERT INTO TARGET SELECT * FROM SOURCE`
    - B단계: 여기에 **보조 인덱스 2개 추가** 후, 다시 동일한 100만 건 로드
- 결과 해석
    - **A단계(인덱스: PK 1개)**: 약 **4.95초**
    - **B단계(인덱스: PK 1개 + 추가 2개 = 총 3개)**: 약 **38.98초**

즉, 인덱스 2개 추가만으로 **약 7~8배 이상 느려짐**을 확인할 수 있다.

왜 이렇게 커질까?

- 100만 건마다 **추가 인덱스 2개 × (키 탐색 + 리프 삽입 + 분할 가능성)**
- 인덱스 리프 블록 경합(핫 블록), 로그 발생량 증가, 버퍼/체인지 관리 오버헤드가 누적

### 무결성 제약과 DML성능

- 데이터 무결성 규칙
    - 개체 무결성
        - PK/UK로 행의 유일성을 보장
        
        → **유일성 검사**를 위해 보통 **(고유) 인덱스**가 필요하고, DML 때마다 인덱스 유지·검사가 수행됨.
        
    - 참조 무결성
        - FK로 부모–자식 관계를 보장
        
        → 자식 INSERT 시 부모 존재 여부 검사, 부모 DELETE/UPDATE 시 자식 존재 검사
        
        → 검사 과정에서 인덱스가 없으면 풀스캔/락 경합이 커지고, 있어도 조회 + 인덱스 유지 비용이 듦
        
    - 도메인 무결성
        - CHECK/NOT NULL 등 값의 범위·형식 제약
        
        → **행 당 불리언 평가**가 전부(인덱스가 필수는 아님)라서 **PK/FK보다 상대적으로 가벼움**
        
    - 사용자 정의 무결성
        - 트리거/애플리케이션 로직 등으로 보강

다시 말하자면, **PK/UK/FK**는 대개 **인덱스 유지 + 검사**때문에 DML 비용이 크고,**CHECK/NOT NULL**은 단순 평가라 비교적 가볍다.

```sql
-- 준비: SOURCE 100만 건 생성
DROP TABLE source PURGE;

CREATE TABLE source AS
SELECT
       ROWNUM AS no,
       e.empno,
       e.ename,
       e.job,
       e.mgr,
       e.hiredate,
       e.sal,
       e.comm,
       e.deptno
FROM (SELECT * FROM emp) e,
     (SELECT LEVEL AS lv FROM dual CONNECT BY LEVEL <= 100000);

-- TARGET 생성(빈 테이블) + PK 1개
DROP TABLE target PURGE;

CREATE TABLE target AS SELECT * FROM source WHERE 1=2;
ALTER TABLE target ADD CONSTRAINT target_pk PRIMARY KEY(no);

-- ===========================================
-- A단계: PK 1개 상태
-- ===========================================
SET TIMING ON;

INSERT INTO target
SELECT * FROM source;
COMMIT;
/* 예상: ~5초 내외 (환경차 있음) */

-- ===========================================
-- B단계: 보조 인덱스 2개 추가(총 3개) 후 재적재
-- ===========================================
TRUNCATE TABLE target;

CREATE INDEX target_x1 ON target(ename);
CREATE INDEX target_x2 ON target(deptno, mgr);

INSERT INTO target
SELECT * FROM source;
COMMIT;
/* 예상: ~39초 내외 (A 대비 7~8배) */

-- ===========================================
-- C단계: 인덱스/PK 모두 제거 후 재적재(가장 빠름)
-- ===========================================
DROP INDEX target_x1;
DROP INDEX target_x2;
ALTER TABLE target DROP PRIMARY KEY;

TRUNCATE TABLE target;

INSERT INTO target
SELECT * FROM source;
COMMIT;
/* 예상: ~1.3초 내외 */
```

| PK 제약 / 인덱스 | 일반 인덱스(2개) | 소요시간 |
| --- | --- | --- |
| O | O | 38.98초 |
| O | X | 4.95초 |
| X | X | 1.32초 |

 **→ 인덱스 개수**와 **PK 유무**가 대량 적재 성능에 **압도적인 영향**을 준다.

     인덱스/PK가 없으면 테이블에**데이터만 쓰면 끝**이므로 가장 빠르다는 것을 알 수 있다.

### 조건절과 DML성능

```sql
SQL> set autotrace traceonly exp;

SQL> update emp set sal = sal * 1.1 where deptno = 40;
```

- `where deptno = 40` 조건을 만족하는 행을 찾기 위해 **EMP_X01 인덱스(Deptno 컬럼 기반)** 을 **Range Scan**으로 사용하게 됩니다.  즉, **UPDATE 실행 전 “SELECT WHERE” 단계가 선행**
- DML의 성능 중 상당 부분은 **조건절을 통해 데이터를 찾는 비용**이 차지

### 서브쿼리와 DML성능

```sql
UPDATE emp e
   SET sal = sal * 1.1
 WHERE EXISTS (
       SELECT 'x'
         FROM dept
        WHERE deptno = e.deptno
          AND loc = 'CHICAGO');
```

- **EMP 테이블을 외부 루프(Nested Loop Outer)** 로 돌며, 각 행마다 **DEPT 테이블에 EXISTS 조건 검사**를 합니다.
- 즉, EMP 한 행마다 DEPT 조회 1번 → 5행이라면 총 5번
- **조건절에 인덱스(DEPT_X01)** 가 있으면 빠르지만 없으면 DEPT는 매번 Full Scan 하게 됩니다.

### Redo로깅과 DML성능

- Redo역할
    - **무엇?** 데이터파일과 컨트롤파일에 일어난 **모든 변경 사실**을 **로그(redo)** 에 기록
    - **왜?** 장애가 나면 **트랜잭션을 재현(roll-forward)** 해서 **일관된 상태**로 복구하기 위해
    - **언제?** **모든 DML 때마다**(INSERT/UPDATE/DELETE/DDL 일부도) 기록됨
- INSERT가 특히 느려지는 이유
    - 인덱스가 많으면 **인덱스 변경 + redo**가 기하급수적으로 증가
    - 대량 적재 시에는 **redo 최소화 전략**(예: direct-path insert, nologging, 병렬) 등을 검토해야함

### Undo로깅과 DML성능

- Undo의 역할
    - **롤백/일관성 읽기** 용도
    - **Undo = 현재를 과거로 되돌릴 정보**, **Redo = 과거를 현재로 되돌릴 정보**
    - 오라클은 **MVCC** 모델: DML 할 때마다 Undo 세그먼트에 **이전 값**을 기록
        - 다른 세션이 예전 SCN 시점으로 **일관된 조회**를 할 수 있음
        - 장시간 트랜잭션/대량 변경이 이어지면 **Undo 공간**과 **읽기 일관성 비용**이 커짐
- 성능에 미치는 영향
    - DML 1건마다 **Undo도 생성** → 공간/I/O 부담
    - 오래 잡고 있는 트랜잭션은 **Undo를 오래 점유** → 경합/ORA-01555(Snapshot too old) 위험

### Lock과 DML성능

- 동시성 제어를 위해 락은 **필수**지만, **경합이 심해지면 대기(Blocking)** 로 성능 급락
- **격리수준**(Read Committed/Serializable 등)이 올라갈수록 **락 유지 시간/범위**가 커짐

### 커밋과 DML성능

- 커밋의 내부 매커니즘을 이해해야함
    1. DB 버퍼캐시
    2. Redo로그 버퍼
    3. 트랜잭션 데이터 저장과정

→ 트랜잭션 종료 + 영속성 보장하기 위해선 LGWR(Database Writer)가 redo를 디스크에 Batch작업으로 기록헤야 끝

- 엑셀파일의 저장버튼을 커밋으로 비유
    - 문서 작성에서 **저장을 자주 누르면 안전하지만 느리다**.
    - DB도 **Commit Too Often**은 redo 동기 기록 빈도를 높여 **지연 증가**
    - 반대로 **너무 늦게 커밋**하면 Undo/락 보유 시간 증가, 장애 시 롤백 비용↑

## 6.1.2 데이터베이스 Call과 성능

SQL은 세 단계로 나누어 실행된다.

- Parse Call
    - SQL 파싱과 최적화를 수행하는 단계다. SQL과 실행계획을 라이브러리 캐시에서 찾으면, 최적화 단계는 생략가능
- Execute Call
    - 말 그대로 SQL을 실행하는 단계다. DML은 이 단계에서 모든 과정이 끝나지만, SELECT문은 Fetch 단계를 거친다.
- Fetch Call
    - 데이터를 읽어서 사용자에게 결과집합을 전송하는 과정으로 SELECT문에서만 나타난다. 전송할 데이터가 많을 때는 Fetch Call이 여러번 발생한다.

Call이 어디서 발생하느냐에 따라 User Call과 Recursive Call로 나눌 수 있다.

- User Call
    - 네트워크를 경유해 DBMS 외부로부터 인입되는 Call
- Recursive Call
    - DBMS 내부에서 발생하는 Call

두개 모두 위에 세단계를 거친다.

### 절차적 루프처리

오라클에서 SQL을 수행할 때는 내부적으로 **Call**(호출)이 발생한다

- **한 번의 SQL**로 **많은 데이터를 처리**하면 → **Call 횟수가 적다 → 빠름**
- **PL/SQL 반복문으로 한 행씩 처리**하면 → **Call 횟수가 매우 많아진다 → 느림**

```sql
set timing on;

begin
  for s in (select * from source)
  loop
    insert into target values
      (s.no, s.empno, s.ename, s.job, s.mgr,
       s.hiredate, s.sal, s.comm, s.deptno);
  end loop;

  commit;
end;
/

```

약 29초가 나옴

루프를 돌 때마다 **INSERT 1번 → DB Call 1번**

100만 건이면 → DB Call **100만 번**

하지만 여기서 중요한 포인트:

> 이 실험은 네트워크 왕복(Network Round Trip) 이 없음
> 
> 
> → 즉, **Recursive Call(내부 호출)** 형태라 그나마 빠른 편
> 

만약 이걸 **JDBC, Python, Node 등 애플리케이션에서 루프 처리** 했다?

→ 네트워크 왕복까지 생김 → 29초가 아니라 5~20배 더 느려질 것으로 예상

- 페이지 412P 코드 참고

JAVA로 실행 → 218초 소요

### One SQL의 중요성

DB는 한번에 묶어서 처리할 때가 가장빠름

```sql
insert into target
select * from source;
```

단 한번의 Call로 처리하니 1.46초만에 수행 마침

## 6.1.3 Array Processing 활용

앞서 절차적으로 100만건을 한 건씩 INSERT 했을 때 약 29초걸린 것을 확인할 수 있었다.

**Array Processing(배열 처리)** 를 이용해서 성능을 개선할 수 있다. 

- Array Processing
    - 대량 데이터를 다룰 때 한 건씩 DB에 SQL Call을 보내지 않고, 여러 건을 배열로 묶어서 한 번에 처리하는 방법
- 소스 415P참고 → 수행결과) 29.31초 → 3.99초 수행완료

## 6.1.4 인덱스 및 제약 해제를 통한 대량 DML 튜닝

대용량 적재 시엔 인덱스/제약을 사용하지 않고, 적재 후 한번에 재생성/재빌드 하면 훨씬 빠르다. 

### 1. 인덱스/PK **그대로** 두고 적재

```sql
SET TIMING ON;

INSERT /*+ APPEND */ INTO target
SELECT * FROM source;

COMMIT;

# 결과: 약 1분 19초
# 이유: PK 고유성 검사 + PK 인덱스 유지 + 일반 인덱스 유지 + Redo/Undo 모두 발생.
```

### 2. PK 제약과 인덱스 해제1 - PK 제약에 Unique 인덱스를 사용한 경우

PK 제약을 비활성화하면서 인덱스는 Drop, 일반 인덱스는 UNUSABLE로 바꾼 뒤 적재한다.

```sql
-- 2-1) 타깃 비우기
TRUNCATE TABLE target;

-- 2-2) PK 비활성 + PK 인덱스 삭제(drop index)
ALTER TABLE target MODIFY CONSTRAINT target_pk DISABLE DROP INDEX;

-- 2-3) 일반 인덱스는 UNUSABLE 처리
ALTER INDEX target_x1 UNUSABLE;

-- 2-4) UNUSABLE 인덱스는 DML 시 건드리지 않도록
ALTER SESSION SET skip_unusable_indexes = TRUE;

-- 2-5) 대량 적재
INSERT /*+ APPEND */ INTO target
SELECT * FROM source;

COMMIT;

# 결과: 약 5.84초 (엄청 빨라짐)
# → 인덱스 유지 및 고유성 검사가 일시적으로 사라졌기 때문
```

이후 PK 재활성화 + 일반 인덱스 재빌드

```sql
-- 2-6) PK 재활성화 (기존 데이터 검증을 생략해 속도를 올리려면 NOVALIDATE)
ALTER TABLE target MODIFY CONSTRAINT target_pk ENABLE NOVALIDATE;

-- 2-7) 일반 인덱스 재빌드
ALTER INDEX target_x1 REBUILD;

# 결과: 8.3초
# 기존 1분 19초보다 훨씬 빠름
```

### 3. PK제약과 인덱스 해제 2 - PK제약에 Non-Unique 인덱스를 사용한 경우

Unique 인덱스가 UNUSABLE이면 적재가 막히지만, Non-Unique 인덱스를 PK가 참조하도록 해두면 제약을 DISABLE만 해도 인덱스를 DROP하지 않고 UNUSABLE로 둔 채 적재할 수 있다.

3-1) PK가 Non-Unique 인덱스를 사용하도록 전환

```sql
SET TIMING OFF;
TRUNCATE TABLE target;

-- 기존 PK 인덱스 제거(드롭) + Non-Unique 인덱스 생성
ALTER TABLE target DROP PRIMARY KEY DROP INDEX;

CREATE INDEX target_pk ON target(no, empno);   -- Non-Unique 인덱스

-- 같은 인덱스를 PK가 사용하도록 지정(keep index 효과)
ALTER TABLE target
  ADD CONSTRAINT target_pk PRIMARY KEY (no, empno)
  USING INDEX target_pk;  -- Non-Unique 인덱스를 PK에 연결
```

3-2) PK 비활성화 + 인덱스 UNUSABLE 후 적재

```sql
ALTER TABLE target MODIFY CONSTRAINT target_pk DISABLE KEEP INDEX;

ALTER INDEX target_pk UNUSABLE;  -- PK가 참조하는 인덱스
ALTER INDEX target_x1 UNUSABLE;

ALTER SESSION SET skip_unusable_indexes = TRUE;

SET TIMING ON;

INSERT /*+ APPEND */ INTO target
SELECT * FROM source;

COMMIT;
```

3-3) 인덱스 재생성 + PK 제약 활성화

```sql
ALTER INDEX target_x1 REBUILD;
ALTER INDEX target_pk REBUILD;
ALTER TABLE target MODIFY CONSTRAINT target_pk ENABLE NOVALIDATE;
```

총 소요시간 18초 > 기존 1분 19초 대비 **3~4배 이상** 빠름

- 빨라진 이유
    - **인덱스 유지 제거**
        - 적재 중 인덱스 키 삽입/분할(split)/Redo/Undo가 발생하지 않음.
    - **고유성 검사 제거**
        - PK/UK의 **유일성 검증** 비용이 없어짐
    - **Direct-Path Insert(/**+ APPEND */)*
        - 세그먼트에 **바로 할당**하는 경로 → **버퍼캐시/일부 로깅 부담 감소**

## 6.1.5 수정가능 조인 뷰

```sql
update 고객 c
set  최종거래일시 = (select max(거래일시) from 거래 where 고객번호 = c.고객번호)
   , 최근거래횟수 = (select count(*) from 거래 where 고객번호 = c.고객번호)
   , 최근거래금액 = (select sum(거래금액) from 거래 where 고객번호 = c.고객번호);
```

기존 방식은 다음과 같이 각 항목마다 서브쿼리를 여러 번 호출하여 UPDATE함

**❗ 문제점**

- 고객 한 명마다 거래 테이블을 여러 번 스캔
- 고객 수가 많아질수록 성능이 기하급수적으로 느려짐

**성능개선 1차 - 서브쿼리 한번으로 묶기**

```sql
update 고객 c
set (최종거래일시, 최근거래횟수, 최근거래금액) =
    (select max(거래일시), count(*), sum(거래금액)
     from 거래
     where 고객번호 = c.고객번호)
where exists (
     select 1
     from 거래
     where 고객번호 = c.고객번호);
```

• 거래를 한 번만 스캔 → 성능 대폭 개선

**성능 개선 2차 — GROUP BY 서브쿼리 먼저 수행** 

```sql
update 고객 c
set (최종거래일시, 최근거래횟수, 최근거래금액) =
(
  select max(거래일시), count(*), sum(거래금액)
  from 거래
  where 고객번호 = c.고객번호
  and 거래일시 >= trunc(add_months(sysdate, -1))
)
where exists (
  select 1 from 거래
  where 고객번호 = c.고객번호
  and 거래일시 >= trunc(add_months(sysdate, -1))
);
```

→ 최근 한 달 거래만 보는 조건을 앞에 넣어 처리 범위를 줄임

**수정가능 조인 뷰**

- 조인뷰

FROM절에 두 개 이상 테이블을 가진 뷰를 가리킴

- 수정가능 조인뷰

입력, 수정, 삭제가 허용되는 조인뷰

- 조인 뷰에서 수정(UPDATE/DELETE/INSERT) 이 가능하려면조인 결과에서 한 테이블(row)의 중복 없이 유일하게 매핑될 수 있어야 함
    - 이를 만족하는 테이블 = Key-Preserved Table (키 보존 테이블)

| 테이블 | 조건 | 의미 |
| --- | --- | --- |
| Key-Preserved Table | PK 또는 Unique 기준으로 조인후에도 중복 X | 수정 가능 |
| Non Key-Preserved Table | 조인 후 중복 발생 | 수정 불가 > ORA-017779 발생 |

예시

```sql
create or replace view EMP_DEPT_VIEW as
select e.rowid emp_rid, e.*, d.rowid dept_rid, d.dname, d.loc
from emp e, dept d
where e.deptno = d.deptno;
```

EMP는 행이 유일하지만,

DEPT는 조인 결과에서 중복이 생김 → Non Key-Preserved Table

```sql
update EMP_DEPT_VIEW set loc = 'SEOUL' where job = 'CLERK';
```

따라서 아래 UPDATE는 오류 발생: ORA-01779

**해결방안**

조인 대상 중 하나에 PK 또는 Unique 인덱스 생성

```sql
alter table dept add constraint dept_pk primary key(deptno);
```

이제 dept는 Key-Preserved Table

→ VIEW에서 UPDATE, DELETE 가능해짐

**ORA-01779오류 회피**

| 버전 | 해결 방안 |
| --- | --- |
| 10g 이하 | bypass_uivc 힌트로 우회가능 |
| 11g | 힌트는 유지, 수정 가능 조인뷰 사용가는 |
| 12c 이후 | 힌트 없이도 조인뷰 업데이트 가능하도록 개선 |

## 6.1.6 MERGE 문 활용

MERGE 문 활용(= UPSERT) 을 주제로, DW 적재나 동기화 작업에서 신규는 INSERT, 기존은 UPDATE를 한 번의 SQL로 처리하는 방법이다.

### **왜 Merge가 필요한가?**

- 기간계 → DW 동기화: 전일/당일 변경분을 뽑아(Source: customer_delta) → 타깃 DW(customer)에 있으면 UPDATE, 없으면 INSERT
- 여러 SQL(SELECT→IF→UPDATE/INSERT)로 나누지 말고, 한 문장으로 원자적 처리가 목적

예시

```sql
MERGE INTO customer t
USING customer_delta s
   ON (t.cust_id = s.cust_id)
WHEN MATCHED THEN
     UPDATE SET
       t.cust_nm = s.cust_nm,
       t.email  = s.email,
       t.tel_no = s.tel_no,
       t.region = s.region,
       t.addr   = s.addr,
       t.reg_dt = s.reg_dt
WHEN NOT MATCHED THEN
     INSERT (cust_id, cust_nm, email, tel_no, region, addr, reg_dt)
     VALUES (s.cust_id, s.cust_nm, s.email, s.tel_no, s.region, s.addr, s.reg_dt);
```

- matched: 조인 성공 → UPDATE
- not matched: 조인 실패 → INSERT
- 그림(Left Outer Join 비유)처럼 소스 기준으로 타깃을 갱신하는 구조

### **2) Optional/Conditional Operations**

 **1. UPDATE/INSERT에 추가 조건 부여**

```sql
MERGE INTO customer t
USING customer_delta s
   ON (t.cust_id = s.cust_id)
WHEN MATCHED THEN
     UPDATE SET ...
     WHERE s.reg_dt >= DATE '2000-01-01'               -- UPDATE에 조건
WHEN NOT MATCHED THEN
     INSERT (...) VALUES (...)
     WHERE s.reg_dt < TRUNC(SYSDATE);                  -- INSERT에 조건
```

• ON 조건 외에 행동(UPDATE/INSERT) 자체를 제한할 수 있음

1. **DELETE절까지 사용**

```sql
MERGE INTO customer t
USING customer_delta s
   ON (t.cust_id = s.cust_id)
WHEN MATCHED THEN
     UPDATE SET ...
     DELETE WHERE t.withdraw_dt IS NOT NULL;  -- 매치된(기존) 행 중 조건 충족 시 삭제
WHEN NOT MATCHED THEN
     INSERT (... ) VALUES (...);
```

- 주의: DELETE는 “matched된 행”만 삭제합니다.
소스에 “삭제되었다”는 사실만 있고 조인이 실패하는 경우엔 MERGE만으로는 지울 수 없다!그런 케이스는 별도 반대조인/ANTI JOIN로 삭제 처리를 추가해야함

### 3) 전통적 IF+UPDATE/INSERT vs MERGE

기존 평소 패턴(SELECT로 존재 여부 확인 후 UPDATE/INSERT)은 보통 **최대 두 번**의 SQL이 필요

```sql
-- 존재 여부 확인
SELECT COUNT(*) INTO :cnt FROM dept WHERE deptno = :val1;

IF :cnt = 0 THEN
  INSERT INTO dept(deptno, dname, loc) VALUES(:val1, :val2, :val3);
ELSE
  UPDATE dept SET dname = :val2, loc = :val3 WHERE deptno = :val1;
END IF;
```

MERGE를 쓰면 **한 번에 가능하다.**

```sql
MERGE INTO dept a
USING (SELECT :val1 deptno, :val2 dname, :val3 loc FROM dual) b
   ON (b.deptno = a.deptno)
WHEN MATCHED THEN
     UPDATE SET a.dname = b.dname, a.loc = b.loc
WHEN NOT MATCHED THEN
     INSERT (a.deptno, a.dname, a.loc) VALUES (b.deptno, b.dname, b.loc);

```

### 수정가능 조인 뷰 UPDATE VS MERGE

- 조인 후 특정 조건만 바꾸는 단순 갱신이라면 **수정가능 조인 뷰 UPDATE**가 더 간결하고 때로는 빠를 수 있다.
- 반대로 **INSERT까지 필요한 upsert**라면 MERGE가 적합
- Merge
    
    ```sql
    MERGE INTO emp t
    USING emp_src s
       ON (t.empno = s.empno)
    WHEN MATCHED THEN UPDATE SET t.ename = s.ename
    WHERE t.ename <> s.ename;      -- 변경 필요시에만
    ```
    
- 수정가능 조인 뷰 UPDATE
    
    ```sql
    UPDATE (
      SELECT t.ename AS t_ename, s.ename AS s_ename
      FROM   emp t JOIN emp_src s ON s.empno = t.empno
      WHERE  t.ename <> s.ename
    )
    SET t_ename = s_ename;
    ```
    
- **INSERT까지 필요한 동기화/업서트** → **MERGE**가 정석.
- **갱신만 필요** & 조인 뷰가 수정 가능 → **수정가능 조인 뷰 UPDATE**가 더 단순·가벼울 수 있음.
- **소스 키가 유일하지 않거나 전처리가 복잡** → `USING` 서브쿼리에서 **집계/중복제거 후 MERGE**.

# 6.2 Direct Path I/O 활용

## 6.2.1 Direct Path I/O

### 일반(버퍼캐시) I/O

- 읽기/쓰기 전에 **DB Buffer Cache**를 거침
- 자주 읽는 **OLTP**엔 유리(캐시 히트↑)하지만, **대량 Full Scan/적재**에선찾을 수 없는 블록을 캐시에서 헤매고, 또 캐시에 적재/배출하는 오버헤드가 큼

### Direct Path I/O

- **버퍼캐시를 우회**하고 **디스크 블록에 직행**(read/write)
- 캐시 탐색, 적재/배출 비용이 사라져 **대량 처리에 빠름**

### Direct Path I/O 기능이 작동하는 경우

1. **병렬 쿼리**(Full Scan)
2. **병렬 DML** 시 (Direct Path Read/Insert)
3. **Direct Path Insert**(APPEND 등)
4. TEMP 세그먼트 블록 읽기
5. direct 옵션을 지정하고 export를 수행할 때
6. LOB 읽기 + `nocache`

## 6.2.2 Direct Path Insert

### 일반 INSERT가 느린 이유

1. **Freelist**(여유블록 목록)에서 쓸 블록 탐색
    1. Freelist? 테이블 HWM 아래 쪽에 있는 블록 중 데이터 입력이 가능한(여유 공간이 있는)블록을 목록으로 관리하는 데 , 이를 Freelist라고 합니다. 
2. 버퍼캐시에 블록 적재/수정
3. **Undo/Redo 로깅** 기록

### Direct Path Insert가 빠른 이유

- HWM(High Water Mark) 뒤의 신규 영역에 순차 기록
- 버퍼캐시 탐색/적재를 생략하고 파일에 직접 기록
- Undo는 최소화
- Redo는 **줄이거나(NOLOGGING)** 생략적용 가능(복구 정책 주의)

### 쓰는 방법

```sql
ALTER TABLE t NOLOGGING;

```

### 주의점 2가지 (그림 6-9)

1. **TM Lock이 Exclusive**로 잡힘 →
    
    진행 중엔 **다른 트랜잭션이 해당 테이블에 DML 못하게 됨**
    
    동시에 접근해야하는 상황에는 사용을 안함 / 배치에서만 사용
    
2. **HWM 뒤로만 채움** → 프리블록 재사용 안 함
    
    삭제가 많은 테이블에 남발하면 **세그먼트가 계속 커짐**
    
    즉, 테이블 내에 비어있는 공간이 있더라도 그걸 사용하지 않고 계속 새로운 공간을 확장해 나가는 형태가 되기 때문에, 테이블 사이즈가 점점 커질 수 있다. 그래서 불필요하게 테이블이 커지지 않도록 가끔 정리 작업이나 파티션 관리 등을 고려
    
    → 파티션이면 **파티션 DROP/TRUNCATE**, 비파티션이면 **주기적 재구성** 필요
    

## 6.2.3 병렬 DML

### 활성화 방법

- 병렬 SELECT는 기본 활성, **병렬 DML은 비활성** → 세션에서 켜야 함

```sql
ALTER SESSION ENABLE PARALLEL DML;   
```

### Direct Path Insert와의 관계

- **병렬 INSERT는 APPEND 힌트를 주지 않아도** 보통 Direct Path Insert 사용
- 헷갈리면 **APPEND도 같이** 주는 걸 추천

### 두 단계 전략

병렬 DML을 할 때 한번에 모든 걸 병렬로 처리하는 게 아니라, 두 단꼐로 나눠서 처리하는 방식

- **1단계(찾기 단계)**: Consistent 읽기 모드로 대상 레코드 찾기(병렬 SELECT)
- **2단계(변경 단계)**: Current 모드로 추가/변경/삭제(병렬 DML)
    
    → 전체적으로 성능과 일관성을 모두 챙길 수 잇음
    

### 잘 동작하는지 확인 (실행계획을 확인)

- UPDATE/DELETE/INSERT 실행계획에
    - 상단이 **PX COORDINATOR** 이고, 그 아래 단계가 **PX**(병렬) 오퍼레이터로 내려가면 정상동작 중
    - 만약 전부 QC(코디네이터) 혼자 처리하면 **병렬 미동작**
