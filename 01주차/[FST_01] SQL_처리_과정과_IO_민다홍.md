## 1. SQL 파싱과 최적화

### 1.1 SQL의 특성

**SQL(Structured Query Language)**은 관계형 데이터베이스를 위한 **구조적, 집합적, 선언적 질의 언어**

> * SQL is designed for a specific purpose: to query data contained in a relational database. 
> * SQL is a set-based, declarative programming language, not an imperative programming language like C or BASIC.

#### SQL의 특징

- **집합 기반 선언적 언어** (명령형 언어가 아님)
- 원하는 결과집합을 구조적으로 선언
- 결과를 만드는 과정은 절차적일 수밖에 없음

#### SQL 처리 흐름

>사용자(SQL) → SQL 옵티마이저(실행 계획 생성) → 실행 가능한 프로시저
> **SQL 최적화**: DBMS 내부에서 프로시저를 작성하고, 컴파일해서 실행 가능한 상태로 만드는 전 과정

---

### SQL 최적화

#### 1단계: SQL 파싱 [SQL 파서]
1. **파싱 트리 생성**: SQL 문을 구성요소별로 분석하여 트리 생성
2. **Syntax 체크**: 문법적 오류 확인
   - 사용할 수 없는 키워드 사용
   - 잘못된 구문 순서
3. **Semantic 체크**: 의미상 오류 확인
   - 존재하지 않는 테이블/컬럼 사용
   - 객체에 대한 권한 부족

#### 2단계: SQL 최적화 [SQL 옵티마이저]
- **시스템 및 객체 통계 정보** 기반으로 다양한 실행경로 생성
- 각 경로를 비교하여 **가장 효율적인 실행계획** 선택
- **데이터베이스 성능을 결정하는 가장 핵심적인 엔진**

#### 3단계: 로우 소스 생성 [로우 소스 생성기]
- 옵티마이저가 선택한 실행경로를 **실제 실행 가능한 코드/프로시저**로 변환

---
### SQL 옵티마이저 동작 원리

#### 옵티마이저의 3가지 역할
1. **후보 실행계획 탐색**: 쿼리 수행을 위한 모든 가능한 실행계획 도출
2. **비용 산정**: 데이터 딕셔너리의 통계 정보를 활용하여 각 실행계획의 예상 비용 계산
3. **최적 계획 선택**: 최저 비용을 나타내는 실행계획 선택

#### 실행계획과 비용
- **실행계획**: 옵티마이저가 생성한 처리 절차를 트리 구조로 표현
- **비용**: 각 실행계획의 예상 자원 소모량

#### 옵티마이저 힌트
- 옵티마이저의 선택이 항상 최선이 아닐 때 사용
- **개발자가 직접 더 나은 액세스 경로를 지정** 가능

---

##  SQL 공유 및 재사용

###  소프트 파싱 vs 하드 파싱

### 소프트 파싱
- **과정**: 사용자 → SQL 파싱 → 캐시에 존재 → 실행

### 하드 파싱
- **과정**: 사용자 → SQL 파싱 → 캐시에 존재하지 않음 → 최적화 → 로우 소스 생성 → 실행

#### 라이브러리 캐시 (Library Cache)
- **옵티마이저가 생성한 내부 프로시저를 재사용하기 위한 캐싱 공간**
- **SGA(System Global Area)** 의 구성요소
- 서버 프로세스와 백그라운드 프로세스가 공통으로 액세스

#### 하드 파싱이 무거운 이유
- 무수히 많은 **후보 실행경로 도출**
- 짧은 시간 내 **딕셔너리와 통계 정보 읽기**
- 각 경로의 **효율성 판단 과정**이 복잡

---

### 바인드 변수의 중요성

#### 이름 없는 SQL의 문제점
**SQL 자체가 이름**이므로, 텍스트의 작은 차이라도 서로 다른 객체로 인식

```sql
-- 모두 다른 SQL로 인식됨
SELECT * FROM EMP WHERE EMPNO = 7900;
SELECT * FROM EMP WHERE SCOTT.EMPNO = 7900;
SELECT /* comment */ FROM EMP WHERE SCOTT.EMPNO = 7900;
```

#### 바인드 변수 미사용 시 문제점
```sql
-- 로그인 할 때마다, 내부 프로시저를 만들어서 라이브캐시를 적재하는 셈 
SELECT * FROM CUSTOMER WHERE LOGIN_ID = '" + login_id + "'";
```
- **저장 공간 낭비**
- **SQL 탐색 속도 저하**
- **매번 하드 파싱 발생**


#### 바인드 변수 사용 
```sql
-- 바인드 변수 사용
SELECT * FROM CUSTOMER WHERE LOGIN_ID = ?;
SELECT * FROM CUSTOMER WHERE LOGIN_ID = : 1

```
- **1번의 하드 파싱**만 발생
- **모든 고객이 동일한 SQL 공유** 및 재사용
- 성능  향상

---

## 데이터 저장 구조 및 I/O 메커니즘

### SQL이 느린 이유
**SQL이 느린 주된 이유는 디스크 I/O** 때문이다. I/O 처리 중 프로세스가 대기해야 하므로 SQL 실행 속도가 느려질 수 있음

###  데이터베이스 저장 구조

- **테이블스페이스**: 세그먼트를 담는 컨테이너
- **세그먼트**: 저장공간이 필요한 객체 (테이블, 인덱스, 파티션, LOB 등)
- **익스텐트**: 공간을 확장하는 단위, 연속된 블록 집합
- **블록**: 데이터를 읽고 쓰는 최소 단위
- **데이터 파일**: 디스크 상의 물리적 OS 파일

---

### 3.3 데이터 액세스 방식

#### 시퀀셜 액세스 (Sequential Access)
- **논리적/물리적으로 연결된 순서**에 따라 차례대로 블록을 읽는 방식
- **인덱스 리프 블록**: 앞뒤 주소값을 통해 논리적으로 연결
- **테이블 Full Scan**: 세그먼트 헤더의 익스텐트 맵을 활용하여 순차 읽기

#### 랜덤 액세스 (Random Access)
- **논리적/물리적 순서를 따르지 않고** 레코드 하나를 위해 한 블록씩 접근
- **ROWID**를 통한 직접 접근

---

### DB 버퍼 캐시

#### 버퍼 캐시의 역할
- **데이터 캐시**라고도 함
- **디스크에서 읽은 데이터 블록을 캐싱**
- **반복적인 I/O Call 최소화**가 목적

#### I/O 개념
- **논리적 I/O**: SQL 처리 과정에서 발생한 총 블록 I/O
- **물리적 I/O**: 디스크에서 발생한 실제 블록 I/O

#### [예시] 
```
논리적 일량: 바퀴를 7,500번 회전시켜야 갈 수 있는 거리 (고정값)
물리적 일량: 페달을 750번 밟아야 갈 수 있는 거리 (가변값)
```

#### 버퍼 캐시 히트율 [BCHR]

```
BCHR = (캐시에서 곧바로 찾은 블록 수 / 총 읽은 블록 수) × 100
물리적 I/O = 논리적 I/O × (100 - BCHR)
```

> **핵심**: SQL 성능 향상을 위해서는 **논리적 I/O를 줄이는 것**이 중요하다.

---

###  I/O 처리 방식

#### Single Block I/O
- **한 번에 한 블록씩** 요청하여 메모리에 적재
- **인덱스**에서 주로 사용
- **소량 데이터** 처리에 적합

#### Multiblock I/O
- **한 번에 여러 블록씩** 요청하여 메모리에 적재
- **테이블 전체 스캔**에서 사용
- **대량 데이터** 처리에 적합
- **인접한 블록들을 한꺼번에** 읽어와 캐시에 미리 적재

#### Multiblock I/O의 효율성
- **단위가 클수록 더 효율적** (한 번의 I/O 작업으로 많은 데이터 확보)
- **OS 단위**: 보통 1MB 단위로 I/O 수행
- **Oracle**: 8KB × 128 = 1MB (최대 128개 블록)
- **제한사항**: 익스텐트 경계를 넘지 못함

---

###  테이블 액세스 방식 비교

#### Table Full Scan
- **테이블 전체**를 읽어 원하는 데이터를 찾는 방식
- **시퀀셜 액세스 + Multiblock I/O** 사용
- **대량 데이터** 처리에 유리

#### Index Range Scan
- **인덱스**에서 일정량을 스캔하여 얻은 **ROWID로 테이블 레코드** 접근
- **랜덤 액세스 + Single Block I/O** 사용
- **소량 데이터** 처리에 유리

#### 성능 특성 비교

| 구분 | Table Full Scan | Index Range Scan |
|------|-----------------|------------------|
| **액세스 방식** | 시퀀셜 액세스 | 랜덤 액세스 |
| **I/O 방식** | Multiblock I/O | Single Block I/O |
| **적합한 상황** | 대량 데이터 처리 | 소량 데이터 처리 |
| **장점** | 한 번에 많은 블록 처리 | 정확한 데이터만 처리 |
| **단점** | 불필요한 데이터도 읽음 | 매번 I/O 대기 발생 |


**"인덱스는 만능이 아니다. 읽을 데이터가 일정량을 넘으면 Table Full Scan이 유리할 수 있다."**

---

####  캐시 탐색 메커니즘 

- Direct Path I/O 를 제외한 모든 블록 I/O 는 메모리 버퍼캐시를 경유한다.

1. 인덱스 루트 블록을 읽을때
2. 인덱스 루트 블록에서 얻은 주소 정보로 브랜치 블록을 읽을때
3. 인덱스 브랜치 블록에서 얻은 주소 정보로 리프 블록을 읽을 때
4. 인덱스 리프 블록에서 얻은 주소 정보로 테이블 블록을 읽을 때
5. 테이블 블록 Full Scan 할 때

#### 버퍼캐시
- 해시 구조로 관리된다.
- 해시 알고리즘으로 버퍼 헤더를 찾고, 얻은 포인터로 버퍼 블록을 액세스한다.

```
예) 20번 블록을 찾고 싶은 경우
- 블록 번호 20을 5로 나누면 나머지가 0
- 캐싱되어 있다면 버퍼 헤더가 첫 번째 해시 체인(해시값: 0)에 연결
- 항상 첫 번째 해시 체인만 탐색하면 됨
```

#### 해시 구조의 특징

1. 같은 입력값은 항상 동일한 해시 체인에 연결됨
2. 다른 입력값이 동일한 해시 체인에 연결될 수 있음 (해시 충돌)
3. 해시 체인 내에서는 정렬이 보장되지 않음

#### 동시성 문제
- 버퍼캐시는 SGA 구성요소
- 캐싱된 버퍼 블록은 공유자원
- 두 개 이상의 프로세스가 동시 접근하려고 할 때 동시성 문제 발생
- 한 프로세스씩 순차적 접근이 가능하도록 직렬화 메커니즘 필요

#### 래치(Latch)
버퍼 캐시에서 해시 체인을 탐색하면서 대량의 데이터를 읽는 동안
그 사이에 체인이 수정되지 않도록 보호
해시 체인 래치를 획득한 프로세스만이 진입 가능

* 특징
- SGA를 구성하는 서브 캐시마다 별도의 래치 존재
- 래치에 의한 경합이 높으면 성능 저하 발생
- 버퍼 블록에도 직렬화 메커니즘이 존재: 버퍼 Lock

결국 SQL 튜닝을 통해 쿼리 일량(논리적I/O)를 줄여야 한다




1장.
> 논리적I/O를 줄여야한다.
위의 문장이 1장에서 강조하고 있는 주제 인 것같다.
쿼리가 느리면 인덱스 생성을 먼저 생각했던 접근 방식을 반성하게 됐다.
간단하게 테스트를 진행해봤는데 시간이 있다면 조금 더 다르게 테스트를 조금 해보고 싶다.

```
-- 1. 샘플 테이블 생성

DROP TABLE IF EXISTS LARGE_TABLE;
CREATE TABLE LARGE_TABLE (
    ID BIGINT PRIMARY KEY AUTO_INCREMENT,
    STATUS ENUM('ACTIVE','INACTIVE'),
    DATA VARCHAR(100)
);

-- 2. 프로시저 생성

CREATE PROCEDURE populate_large_table()
BEGIN
    DECLARE i INT DEFAULT 1;
    WHILE i <= 100000 DO
        INSERT INTO LARGE_TABLE (STATUS, DATA)
        VALUES (
            CASE WHEN RAND() < 0.5 THEN 'ACTIVE' ELSE 'INACTIVE' END,
            CONCAT('Sample data ', i)
        );
        SET i = i + 1;
    END WHILE;
END;

-- 3. 프로시저 호출 : 샘플 데이터 삽입 (총 100,000건, ACTIVE 50%)

call populate_large_table();


-- 1. 변수 초기화
SET @start_time = 0;
SET @end_time = 0;

-- 2. Full Table Scan 실행 계획 확인
EXPLAIN
SELECT STATUS FROM LARGE_TABLE WHERE STATUS = 'ACTIVE';

-- 3. Full Table Scan 시간 측정
SET @start_time = UNIX_TIMESTAMP(NOW(6));
SELECT STATUS FROM LARGE_TABLE WHERE STATUS = 'ACTIVE';
SET @end_time = UNIX_TIMESTAMP(NOW(6));
SELECT CONCAT('Full Scan Time: ', ROUND(@end_time - @start_time, 3), ' seconds') AS full_scan_time;

Full Scan Time (STATUS only): 0.106 seconds

-- 1. INDEX 생성
CREATE INDEX  IDX_STATUS ON LARGE_TABLE(STATUS);

-- 2. Index Range Scan 실행 계획 확인
EXPLAIN
SELECT STATUS FROM LARGE_TABLE WHERE STATUS = 'ACTIVE';

-- 3. Index Range Scan 시간 측정
SET @start_time = UNIX_TIMESTAMP(NOW(6));
SELECT STATUS FROM LARGE_TABLE WHERE STATUS = 'ACTIVE';
SET @end_time = UNIX_TIMESTAMP(NOW(6));
SELECT CONCAT('Index Scan Time: ', ROUND(@end_time - @start_time, 3), ' seconds') AS index_scan_time;

-- Index Scan Time: 0.084 seconds seconds


SELECT * FROM LARGE_TABLE WHERE STATUS = 'ACTIVE'; 
Full Scan 이 Index Scan 보다 빠름 > 커버링 인덱스를 타지 않아서 차이 남 


```










