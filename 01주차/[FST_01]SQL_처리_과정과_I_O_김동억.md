# 1장. SQL 처리 과정과 I/O

## 1.1 SQL 파싱과 최적화

### 1.1.1 구조적,집합적,선언적 질의 언어

- 데이터 준비

```
-- 부서 테이블
CREATE TABLE dept (
    deptno INT PRIMARY KEY,
    dname VARCHAR(20),
    loc VARCHAR(20)
);

-- 사원 테이블
CREATE TABLE emp (
    empno INT PRIMARY KEY,
    ename VARCHAR(20),
    job VARCHAR(20),
    deptno INT,
    FOREIGN KEY (deptno) REFERENCES dept(deptno)
);

-- 부서 데이터 입력
INSERT INTO dept (deptno, dname, loc) VALUES
(10, 'ACCOUNTING', 'NEW YORK'),
(20, 'RESEARCH', 'DALLAS'),
(30, 'SALES', 'CHICAGO'),
(40, 'OPERATIONS', 'BOSTON');

-- 사원 데이터 입력
INSERT INTO emp (empno, ename, job, deptno) VALUES
(7369, 'SMITH',  'CLERK',     20),
(7499, 'ALLEN',  'SALESMAN',  30),
(7521, 'WARD',   'SALESMAN',  30),
(7566, 'JONES',  'MANAGER',   20),
(7654, 'MARTIN', 'SALESMAN',  30),
(7698, 'BLAKE',  'MANAGER',   30),
(7782, 'CLARK',  'MANAGER',   10),
(7788, 'SCOTT',  'ANALYST',   20),
(7839, 'KING',   'PRESIDENT', 10),
(7844, 'TURNER', 'SALESMAN',  30),
(7876, 'ADAMS',  'CLERK',     20),
(7900, 'JAMES',  'CLERK',     30),
(7902, 'FORD',   'ANALYST',   20),
(7934, 'MILLER', 'CLERK',     10);
```

#### SQL(Structured Query Language)

- 구조적 질의 언어
- 특징
  - structured
  - set-based
  - declarative

#### SQL 옵티마이저

- 프로시저를 만들어 내는 DBMS 내부 엔진
- 사용자 --SQL--> 옵티마이저 --실행계획--> 프로시저

### 1.1.2 SQL 최적화

- 세부 과정

* SQL 파싱
  - SQL 파서 담당
  - 파싱 트리 생성 -> Syntax 체크 -> Semantic 체크
* SQL 최적화
  - 옵티마이저 담당
  - 미리 수집한 시스템 및 오브젝트 통계정보 보유
  - 다양한 실행경로를 생성해서 가장 효율적인 경로 선택
* 로우 소스 생성
  - 로우 소스 생성기
  - 실행 가능한 코드 또는 프로시저 형태로 포맷팅

### 1.1.3 SQL 옵티마이저

- 최적화 단계
  - 실행계회 후보군 -> 오브젝트/시스템 통계정보를 이용해 예상비용 산정 -> 최저 비용 실행계획 선택

### 1.1.4 실행계획과 비용

- 실행계획 테스트 준비 및 설정

```
CREATE TABLE t
AS
SELECT d.no, e.*
FROM emp e
    , (SELECT rownum no from dual connect by level <= 1000) d;

create index t_x01 on t(deptno, no);
create index t_x02 on t(deptno, job, no);

exec dbms_stats.gather_table_stats(user,'t');

set autotrace traceonly exp;

# 결과 확인 - index t_x01
select * from t
where deptno = 10
and no = 1;

# 위의 결과와 비교 - index t_x02
select /*+ index(t t_x02) */ * from t
where deptno = 10
and no = 1;

# Table Full Scan 으로 변경
select /*+full(t) */ * from t
where depno = 10
and no = 1;
```

### 1.1.5 옵티마이저 힌트

#### 주석 기호에 '+' 붙이기

```
SELECT /*+ INDEX(A 고객_PK) */
    고객명, ~~
FROM 고객 A
WHERE 고객ID = '0008'
```

#### 주의사항

- 힌트 안에 인자를 나열할때 ','를 사용할 수 있지만 힌트와 힌트 사이에 사용하면 안 됨

```
/*+ INDEX(A A_X01) INDEX(B, B_X03) */  -> 모두 유효
/*+ INDEX(C), FULL(D) */  -> 첫번째 힌트만 유효
```

- 테이블을 지정할 때 스키마명까지 명시하면 안 됨

```
SELECT /*+ FULL(SCOTT.EMP) */ -> 무효
  FROM EMP
```

- FROM 절 테이블 옆에 ALIAS를 지정했다면 힌트에도 반드시 ALIAS를 사용해야 함

```
SELECT /*+ FULL(EMP) */ -> 무효
  FROM EMP E
```

#### 자율 vs 강제

- SQL 옵티마이저에게 일부만 지시 가능

```
# 주문일자 컬럼이 선두인 인덱스 사용
# 조인 방식과 순서, 고객 테이블 액세스 방식은 옵티마이저 판단
SELECT /*+ INDEX(A (주문일자)) */
    A.주문번호, A.주문금액, B.고객명, B.주소
  FROM 주문 A, 고객 B
 WHERE A.주문일자 = :ORD_DT
   AND A.고객ID = B.고객ID
```

- 빈틈없는 지정

```
SELECT /*+ LEADING(A) USE_NL(B) INDEX(A (주문일자)) INDEX(B 고객_PK) */
    A.주문번호, A.주문금액, B.고객명, B.주소
  FROM 주문 A, 고객 B
 WHERE A.주문일자 = :ORD_DT
   AND A.고객ID = B.고객ID
```

- 힌트를 사용할 예정이면 빈틈없이 기술
  아래는 이미지에 있는 Oracle SQL 힌트 목록을 마크다운 테이블로 정리한 내용입니다.[1][2]

### 자주 사용하는 Oracle SQL 힌트 목록

| 분류            | 힌트             | 설명                                                                   |
| --------------- | ---------------- | ---------------------------------------------------------------------- |
| 최적화 목표     | ALL_ROWS         | 전체 처리속도 최적화                                                   |
|                 | FIRST_ROWS(N)    | 최초 N건 응답속도 최적화                                               |
| 액세스 방식     | FULL             | Table Full Scan으로 유도                                               |
|                 | INDEX            | Index Scan으로 유도                                                    |
|                 | INDEX_DESC       | Index를 역순으로 스캔하도록 유도                                       |
|                 | INDEX_FFS        | Index Fast Full Scan으로 유도                                          |
|                 | INDEX_SS         | Index Skip Scan으로 유도                                               |
| 조인순서        | ORDERED          | FROM 절에 나열된 순서대로 조인                                         |
|                 | LEADING          | LEADING 힌트 끝에 기술한 순서대로 조인 (예: LEADING(T1 T2))            |
|                 | SWAP_JOIN_INPUTS | 해시 조인 시, BUILD INPUT을 명시적으로 선택 (예: SWAP_JOIN_INPUTS(T1)) |
| 조인방식        | USE_NL           | NL 조인으로 유도                                                       |
|                 | USE_MERGE        | 소트 머지 조인으로 유도                                                |
|                 | USE_HASH         | 해시 조인으로 유도                                                     |
|                 | NL_SJ            | NL 세미조인으로 유도                                                   |
|                 | MERGE_SJ         | 소트 머지 세미조인으로 유도                                            |
|                 | HASH_SJ          | 해시 세미조인으로 유도                                                 |
| 서브쿼리 팩토링 | MATERIALIZE      | WITH문 정의 집합을 물리적으로 생성                                     |
|                 | INLINE           | WITH문 정의 집합을 물리적으로 생성하지 않고 INLINE 처리하도록 유도     |
| 쿼리 변환       | MERGE            | 뷰 머지 유도                                                           |
|                 | NO_MERGE         | 뷰 머지 억지                                                           |
|                 | UNNEST           | 서브쿼리 Unnesting 유도                                                |
|                 | NO_UNNEST        | 서브쿼리 Unnesting 억제                                                |
|                 | PUSH_PRED        | 조인조건 Pushdown 유도                                                 |
|                 | NO_PUSH_PRED     | 조인조건 Pushdown 억제                                                 |
|                 | USE_CONCAT       | OR 또는 IN-List 조건을 OR-Expansion으로 확장 유도                      |
|                 | NO_EXPAND        | OR 또는 IN-List 조건을 OR-Expansion으로 확장 금지                      |
| 병렬 처리       | PARALLEL         | 테이블 스캔 또는 DML을 병렬적으로 처리하도록 유도 (예: PARALLEL(T1 2)) |
|                 | PARALLEL_INDEX   | 인덱스 스캔을 병렬적으로 처리하도록 유도                               |
|                 | PQ_DISTRIBUTE    | 병렬 수행 시 데이터 분할 방식 (예: PQ_DISTRIBUTE(HASH HASH))           |
| 기타            | APPEND           | Direct-Path Insert                                                     |
|                 | DRIVING_SITE     | DB link Remote 쿼리에 대한 최적화 실행장(Local 또는 Remote)            |
|                 | PUSH_SUBQ        | 서브쿼리를 푸쉬 처리되도록 유도                                        |
|                 | NO_PUSH_SUBQ     | 서브쿼리를 푸쉬되지 않도록 유도                                        |

## 1.2 SQL 공유 및 재사용

### 1.2.1 소프트 파싱 vs. 하드 파싱

### 1.2.2 바인드 변수의 중요성

## 1.3 데이터 저장 구조 및 I/O 메커니즘

### 1.3.1 SQL이 느린 이유

### 1.3.2 데이터베이스 저장 구조

### 1.3.3 블록 단위/IO

### 1.3.4 시퀀셜 액세스 vs. 랜덤 액세스

### 1.3.5 논리적 I/O vs. 물리적 I/O

### 1.3.6 Single Block I/O vs. Multiblock I/O

### 1.3.7 Table Full Scan vs. Index Range Scan

### 1.3.8 캐시 탐색 메커니즘
