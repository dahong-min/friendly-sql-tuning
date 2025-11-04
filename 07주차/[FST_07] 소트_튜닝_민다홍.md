### 소트 튜닝

#### 소트 수행 과정

- default : PGA에 할당한  Sort Area (메모리)
- Sort Area 부족 시 : **Sort Area** 초과 시 디스크의 Temp 테이블스페이스 사용

##### Sort Area 에서 작업을 완료 할 수 있는 지에 따른 소트 유형

1. 메모리 소트 (In-Memory Sort / Internal Sort)

- 전체 정렬 작업을 메모리 내에서 완료

2. 디스크 소트 (To-Disk Sort / External Sort)

- Sort Area 내에서 정렬을 완료하지 못해 디스크 공간까지 사용한다.
- 부분 처리를 불가능하게 함으로써 OLTP 환경에서 성능 저하의 주요 원인이다.

1) 소트할 대상 집합을 SGA 버퍼캐시를 통해 읽음. Sort Area에서 정렬 시도
2) 양이 많다면 중간 단계 집합 Sort Run을  Temp 테이블 스페이스에 임시 세그먼트를 만들어서 저장한다.
3) 정렬된 최종 집합을 얻기위해서 Merge 한다.
> 소트가 발생하지 않게 SQL을 작성하거나,불가피하다면 메모내에서 수행을 완료 할 수 있도록 해야 한다.


#### 소트오퍼레이션
##### 1. Sort Aggregate
   * 전체 로우 대상 집계 수행 시 나타지만, 실제로 데이터를 정렬하지는 않는다.
   * Sort Area를 사용한다

```declarative
SELECT SUM(SAL), MAX(SAL), MIN(SAL), AVG(SAL) FROM EMP;
```
**[절차]**
1. Sort Area에 SUM, MAX, MIN, COUNT 변수를 각각 할당한다.
2. 레코드를 하나씩 SAL 값을 읽으면서 SUM, MAX, MIN 변수에 저장, COUNT 변수에는 1을 저장한다.
3. SUM 값을 누적하고,MAX, MIN 큰 값이나 작은 값이 나타나면 값을 대체한다.


##### 2. Sort Order By
  * 데이터를 정렬할 때 발생
```declarative
SELECT * FROM EMP ORDER BY SAL;
```
##### 3. Sort Order By
* 소팅 알고리즘을 사용해 그룹별 집계 수행
```declarative
SELECT DEPTNO, SUM(SAL), MAX(SAL), MIN(SAL), AVG(SAL)
FROM EMP
GROUP BY DEPTNO
ORDER BY DEPTNO;
```
**[예시]**
```declarative

부서 번호 10, 20, 30, 40 → 메모지 4장 준비
1. 각 메모지에는 아래의 내용을적을수록 입력단을 두고 부서 번호 순으로 정렬 준비
SUM / MAX / MIN / COUNT
2. EMP 테이블 을 한 줄씩 읽으면서  해당 사원의 부서번호에 맞는 메모지를 찾고
SUM, MAX, MIN, COUNT 값을 갱신한다.(Sort Aggregate 방식과 동일)
```


Hash Group By (10gR2 이후)

- GROUP BY 절 뒤에 ORDER BY를 명시하지 않으면 대부분 Hash Group By 사용한다.
- 소트 알고리즘 대신 해싱 알고리즘 사용한다.
- 그룹 개수(부서)가 많지 않으면 Temp 테이블스페이스를 거의 사용하지 않는다.


>Sort Group By는 정렬이 보장되는 것이 아니기때문에 정렬된 그룹핑 결과를 얻고자 할때는 Order By를 명시해야한다.


##### 4. Sort Unique
* 메인쿼리와 조인하기 전 중복 레코드 제거 시 발생
* PK/Unique 인덱스를 통해 Unnesting 된 서브쿼리의 유일성이 보장된다면 Sort Unique 생략된다.
```declarative

SELECT /*+ ordered use_nl(dept) */ * FROM DEPT
WHERE DEPTNO IN (SELECT  /*+ unnest */ DEPTNO FROM EMP WHERE JOB = 'CLERK');
```

* 집합 연산자 사용 시: Union, Minus, Intersect

```declarative

SELECT * JOB, MGR FROM EMP WHERE DEPTNO = 10
    UNION
SELECT * JOB, MGR FROM EMP WHERE DEPTNO = 20
```
* Distinct 연산자 사용 시

Hash Unique (10gR2 이후)

Distinct 연산에도 Hash Unique 방식 사용 가능 (ORDER BY 생략 시)

```declarative

SELECT DISTINCT DEPTNO  FROM EMP;
```

##### 5. Sort Join
* 소트 머지 조인 수행 시 발생
##### 6. Window Sort
* 윈도우 함수(분석 함수) 수행 시 발생

#### 소트가 발생하지 않도록 SQL 작성


##### 1. UNION vs UNION ALL

| 구분       | UNION                                      | UNION ALL                                      |
|------------|--------------------------------------------|------------------------------------------------|
| **중복 처리** | 두 집합 간 **중복 제거** (DISTINCT 동작)     | **중복 확인 없이** 단순 결합                   |
| **소트 작업** | 중복 제거를 위해 **소트(SORT) 수행**         | **소트 없음**

```declarative
--  불필요한 Sort 발생 (상호배타적 조건)
-- 결제수단코드가 'M'과 'C'로 서로 다른 값
-- 두 집합 간 인스턴스 중복 가능성 없다.

SELECT 결제번호, 주문번호, 결제금액, 주문일자
FROM 결제
WHERE 결제수단코드 = 'M' AND 결제일자 = '20180316'

UNION

SELECT 결제번호, 주문번호, 결제금액, 주문일자
FROM 결제
WHERE 결제수단코드 = 'C' AND 결제일자 = '20180316'
```

```declarative
--  UNION > UNION ALL 변경시 중복가능성이 있을때

SELECT 결제번호, 결제수단코드, 주문번호, 결제금액, 결제일자, 주문일자
FROM 결제
WHERE 결제일자 = '20180316'

UNION ALL

SELECT 결제번호, 결제수단코드, 주문번호, 결제금액, 결제일자, 주문일자
FROM 결제
WHERE 주문일자 = '20180316'
  AND 결제일자 <> '20180316'  -- 데이터 중복 방지를 위해서
```

결제일자가 Null 허용 컬럼인 경우

```declarative

AND (결제일자 <> '20180316' OR 결제일자 IS NULL)

AND LNNVL(결제일자 = '20180316')

```

#####  2. Exists 활용

```declarative
SELECT DISTINCT P.상품번호, P.상품명, P.상품가격, ...
FROM 상품 P, 계약 C
WHERE P.상품유형코드 = :pclscd
  AND C.상품번호 = P.상품번호
  AND C.계약일자 BETWEEN :DT1 AND :DT2
  AND C.계약구분코드 = :CTPCD


* 계약_X2 인덱스: 상품번호 + 계약일자
* 상품유형코드 조건절에 해당하는 계약 데이터를 읽은 후 중복 제거해야한다.
-> 상품 수는 적고, 상품별 계약 건수가 많을수록 비효율 심화한다.

```

Distinct의 문제점
1.  중복 제거를 위해 조건에 해당하는 데이터를 모두 읽어야 한다.
2. 부분범위 처리 불가능 하다.
3. 모든 데이터를 대량 I/O 발생한다.


```declarative
SELECT P.상품번호, P.상품명, P.상품가격, ...
FROM 상품 P
WHERE P.상품유형코드 = :pclscd
  AND EXISTS ( SELECT 'X' 
               FROM 계약 C
               WHERE C.상품번호 = P.상품번호
                 AND C.계약일자 BETWEEN :DT1 AND :DT2
                 AND C.계약구분코드 = :CTPCD )

* 각 상품에 대해 조건을 만족하는 계약이 한 건이라도 존재하는지만 확인
* 존재하면 즉시 다음 상품으로 이동

```
EXISTS의 장점
1. 데이터 존재 여부만 확인 (조건 만족 데이터를 모두 읽지 않음)
2. 한 건만 찾으면 즉시 중단
3. Distinct 연산자 불필요 → 소트 연산 제거
4. 상품 테이블에 대한 부분범위 처리 가능

#### 인덱스를 이용한 소트 연산 생략

##### Sort Order By 생략

```
-- 인덱스 : 종목코드 + 거래일시

SELECT 거래일시, 체결건수, 체결수량, 거래대금
  FROM 종목거래
 WHERE 종목코드 = ‘KR123456’
 ORDER BY 거래일시

```
* 인덱스가 [종목 코드 + 거래일시] 경우, 이미 인덱스 거래일시 순서대로 정렬되어 옵티마이저는 Sort Order By 오퍼레이션을 생략한다.
* 전체 레코드(종목코드 = ‘KR123456’)를 읽을 필요가 없으므로, 소트 대상이 많다면 성능 개선 효과가 크다.

#### 부분범위 처리를 활용한 튜닝 기법

1. 부분범위 처리란?

>쿼리 결과 전체를 한 번에 가져오지 않고, 앞쪽 일부만 우선 전송하고, 남은 데이터는 클라이언트 요청 할때마다 나눠서 전송하는 방식


* 과거(2-Tier) 환경에서의 활용

클라이언트가 DB 서버에 직접 접속하던 시절에는
한 세션이 DB 커넥션을 독점했기 때문에
부분범위 처리로 결과를 나눠서 전송(Fetch Call)하는 튜닝이 효과적이었다.

3️⃣ 현재(3-Tier) 환경에서의 한계

3-Tier(웹 서버 / WAS / DB) 구조에서는
DB 커넥션을 다수의 클라이언트가 공유한다.

요청이 끝나면 커넥션을 커넥션 풀에 즉시 반환해야 하므로
➡️ 쿼리 결과를 부분 전송하는 구조는 불가능.

즉, DB 커서를 유지한 채 조금씩 Fetch 하는 방식은 현실적으로 불가.

4️⃣ 그럼에도 “부분범위 처리 원리”가 여전히 유효한 이유

Top-N 쿼리 및 페이징 처리(Pagination) 에서
“일부만 읽고 멈추는” Stopkey 알고리즘 형태로 여전히 활용된다.

##### Top-N 쿼리

전체 결과 집합 중 상위 N개 레코드만 선택하는 쿼리


| DBMS           | 문법 예시                            |
| -------------- | -------------------------------- |
| **SQL Server** | `SELECT TOP 10 ... ORDER BY ...` |
| **DB2**        | `FETCH FIRST 10 ROWS ONLY`       |
| **Oracle**     | 인라인 뷰 + `ROWNUM <= 10`           |

```
-- 인덱스 : 종목코드 + 거래일시

SELECT *
  FROM ( SELECT 거래일시, 체결건수, 체결수량, 거래대금
           FROM 종목거래
          WHERE 종목코드 = 'KR123456'
            AND 거래일시 >= '20180304'
          ORDER BY 거래일시 )
 WHERE ROWNUM <= 10;

```

1. 인덱스가 [종목 코드 + 거래일시] 경우, 이미 인덱스 거래일시 순서대로 정렬되어 옵티마이저는 Sort Order By 오퍼레이션을 생략한다.

2. 옵티마이저는 인덱스 순서대로 스캔하다가 ROWNUM이 10개 채워지는 순간 즉시 멈춘다.
```
0  SELECT STATEMENT  Optimizer=ALL_ROWS
1 0  COUNT(STOPKEY)
2 1   VIEW
3 2    TABLE ACCESS (BY INDEX ROWID) OF '종목거래'
4 3     INDEX (RANGE SCAN) OF '종목거래_PK'

실행계획에서 COUNT(STOPKEY) 로 확인 가능

```
##### 페이징 처리

```
SELECT *
  FROM (
        SELECT ROWNUM NO, A.*
          FROM (
                 .......
               ) A
         WHERE ROWNUM <= (:page * 10)
       )
 WHERE NO >= (:page - 1) * 10 + 1;

```

* 3-Tier 에서 부분범위 처리를 활용하기 위해 해야 할 일

1.부분범위 처리 가능하도록 SQL 작성한다.
1-1.인덱스 사용 가능하도록 조건절을 작성
1-2.조인은 NL 조인 위주로 처리(룩업을 위한 작은 테이블을 해시 조인 Build Input 으로 처리해도 됨)
1-3.Order By 절이 있어도 소트 연산을 생략할 수 있도록 인덱스 구성
2. 작성한 SQL 문을 페이징 처리용 표준 패턴 SQL Body 부분에 붙여 넣는다.

##### 페이징 처리 시 Anti-Pattern

```
WHERE ROWNUM <= (:page * 10)

WHERE NO BETWEEN (:PAGE - 1)*10 +1 AND (:PAGE * 10)
```

ROWNUM 조건절을 제거하면 실행계획에서 COUNT(STOPKEY) 가 COUNT 로 변경된다.
변경 되면 전체 범위를 처리하게 된다.


##### 최소값/최대값 구하기
* 인덱스를 이용해 최소/최대값 구하기 위한 조건

전체 데이터를 읽지 않고 인덱스를 이용해 최소 또는 최대값을 구하려면,
1.조건절 컬럼과
2.MIN/MAX 함수 인자 컬럼
모두 인덱스에 포함돼 있어야 한다

```
CREATE INDEX EMP_X1 ON EMP(DEPTNO, SAL, MGR);
 
SELECT MAX(SAL) FROM EMP WHERE DEPTNO = 30 AND MGR = 7698;

```

DEPTNO = 30 조건을 만족하는 범위 가장 오른쪽으로 내려가면 가장 큰 SAL 값을 읽게된다. -> 엑세스조건

거기서 스캔을 시작해 MGR = 7698 조건을 만족하는 레코드를 1개 찾을 떄 멈춘다 -> 필터조건


```

CREATE INDEX EMP_X1 ON EMP(DEPTNO, SAL)
 
SELECT MAX(SAL) FROM EMP WHERE DEPTNO = 30 AND MGR = 7698;


```

DEPTNO = 30 조건을 만족하는 전체 레코드를 읽어 MGR =7698 조건을 필터링한 후 MAX(SAL) 값을 구한다.
First Row Stopkey 알고리즘이 작동하지 않는다.  -> MGR 누락 → 전체 읽고 비교 (Stopkey 작동 X)

2. Top-N 쿼리를 이용해 최소/최댓값 구하기

```
CREATE INDEX EMP_X1 ON EMP(DEPTNO, SAL);

SELECT *
  FROM (
        SELECT SAL
          FROM EMP
         WHERE DEPTNO = 30
           AND MGR = 7698
         ORDER BY SAL DESC
       )
 WHERE ROWNUM <= 1;

```

DEPTNO = 30 조건을 만족하는 가장 오른쪽부터 역순으로 스캔하면서 MGR=7698 조건을 만족하는 레코드 1개를 찾을 때 바로 멈춘다.

#### 이력조회

First Row Stopkey 또는 Top N Stopkey 알고리즘이 작동하게 구현하는 것이 필요하다.

##### 단순한 이력 조회

```

-- 인덱스 : 장비번호 + 변경일자 + 변경순

SELECT 장비번호, 장비명, 상태코드
     , ( SELECT MAX(변경일자)
           FROM 상태변경이력
           WHERE 장비번호 = P.장비번호 ) 최종변경일자
  FROM 장비 P
 WHERE 장비구분코드 = ‘A001’

```
MAX(변경일자) : 인덱스 구조(장비번호 + 변경일자 + 변경순) → First Row Stopkey 동작했다.

##### 점점 복잡해지는 이력조회
```



SELECT 장비번호
     , 장비명
     , 상태코드
     , SUBSTR(최종이력, 1, 8) AS 최종변경일자
     , TO_NUMBER(SUBSTR(최종이력, 9, 4)) AS 최종변경순번
  FROM (
        SELECT 장비번호
             , 장비명
             , 상태코드
             , (SELECT MAX(H.변경일자 || LPAD(H.변경순번, 4))
                  FROM 상태변경이력 H
                 WHERE H.장비번호 = P.장비번호
               ) AS 최종이력
          FROM 장비 P
         WHERE 장비구분코드 = 'A001'
       )

→ 인덱스를 가공했기 떄문에 First Row Stopkey 동작하지 않는다.


SELECT 장비번호
     , 장비명
     , 상태코드
     , (SELECT MAX(H.변경일자)
          FROM 상태변경이력 H
         WHERE H.장비번호 = P.장비번호
       ) AS 최종변경일자
     , (SELECT MAX(H.변경순번)
          FROM 상태변경이력 H
         WHERE H.장비번호 = P.장비번호
           AND H.변경일자 = (SELECT MAX(H2.변경일자)
                              FROM 상태변경이력 H2
                             WHERE H2.장비번호 = P.장비번호)
       ) AS 최종변경순번
  FROM 장비 P
 WHERE 장비구분코드 = 'A001'
 
 →  상태변경이력을 3번 조회하지만 First Row Stopkey가 동작하기 때문에 위와 같이 쿼리를 작성하는 것이 낫다. 

```

읽어야 할 컬럼이 더 많아진다면 어떻게 해야할까?

##### 1. INDEX_DESC 힌트 활용



```
SELECT 장비번호
     , 장비명
     , 상태코드
     , SUBSTR(최종이력, 1, 8) AS 최종변경일자
     , TO_NUMBER(SUBSTR(최종이력, 9, 4)) AS 최종변경순번
  FROM (
        SELECT /*+ INDEX_DESC(X 상태변경이력_PK) */
             , (SELECT H.변경일자 || LPAD(H.변경순번, 4)) || 상태코드
                  FROM 상태변경이력 H
                 WHERE H.장비번호 = P.장비번호
             AND ROWNUM <=1
               ) AS 최종이력
          FROM 장비 P
         WHERE 장비구분코드 = 'A001'
       )

```
* 인덱스를 역순 스캔 후 ROWNUM <= 1로 즉시 멈춤

* 인덱스 구성 변경 시 결과집합에 문제가 생길 수도 있음



##### Sort Group By 생략

```
-- 인덱스 : REGION

SELECT REGION, AVG(AGE), COUNT(*)
  FROM CUSTOMER
 GROUP BY REGION;
```
GROUP BY 도 인덱스를 이용하면 정렬(Sort Group By) 생략 가능하다.

실행계획에 SORT GROUP BY NOSORT 표시된다.

---


#### Sort Area 를 적게 사용하도록 SQL 작성

##### 소트 데이터 줄이기

```

SELECT LPAD(상품번호, 30) || LPAD(상품명, 30) || LPAD(고객ID, 30) || LPAD(고객명, 30) || TO_CHAR(주문일시, ‘yyyymmdd hh24:mi:ss’)
  FROM 주문상품
 WHERE 주문일시 BETWEEN :START AND :END
 ORDER BY 상품번호


SELECT LPAD(상품번호, 30) || LPAD(상품명, 30) || LPAD(고객ID, 30) || LPAD(고객명, 30) || TO_CHAR(주문일시, ‘yyyymmdd hh24:mi:ss’)
  FROM (
         SELECT 상품번호, 상품명, 고객ID, 고객명, 주문일시 
           FROM 주문상품
          WHERE 주문일시 BETWEEN :START AND :END
         ORDER BY 상품번호
        )

```
|       | 설명                                     |   |                                  |
| -------- | -------------------------------------- | - | -------------------------------- |
| **1번** | `SELECT LPAD(상품번호,30)                  |   | ... ORDER BY 상품번호` → 가공 데이터까지 정렬 |
| **2번**  | 가공 전 데이터(`상품번호`)만 정렬 → Sort Area 적게 사용 |   |                                  |


```
SELECT *
  FROM 예수금원장
 ORDER BY 총예수금 DESC
 
 SELECT 계좌번호, 총예수금
  FROM 예수금원장
 ORDER BY 총예수금 DESC
 
-- 2번째 쿼리는 Full Scan 해도, 소트한 데이터량이 다르게 되므로 성능차이가 발생한다.

```


##### Top N 쿼리의 소트 부하 경감 원리

예시) 전교생 1000명 중 키가 가장 큰 10명 선별하기

1. 전교생 집합
2. 맨 앞줄 맨 왼쪽에 있는 학생 열 명을 가장 큰 사람부터 작은 사람 순으로 줄 세운다.
3. 나머지 990명을 Top10 이 있는 학생들과 비교하고, 더 큰 학생이 나타나면 학생을 교체 한다.
4. 새로 배치된 학생 키에 맞춰 재배치

* Top N 소트 알고리즘이 소트 연산 횟수와 Sort Area 사용량을 줄여주는 원리다.
* 대상 집합이 아무리 커도 많은 메모리 공간이 필요 않다. (정렬 메모리 사용량은 “상위 N건” 크기만큼만 필요하다.)
* 전체 레코드를 다 정렬하지 않고도 오름차순으로 최소값을 갖는 열개의 레코드를 정확히 찾아낼 수 있다.

##### Top-N 조건이 없을 때 발생하는 소트 부하

WHERE ROWNUM 조건 제거 시

→ Top-N 알고리즘 작동 하지 않는다.

→ 전체 정렬 수행 하고, 이 과정에서 Temp 테이블 스페스를 사용하게 될 수 있다.

##### 분석함수에서의 Top N 소트

RANK(), ROW_NUMBER() 같은 윈도우 함수는 MAX()보다 Sort 부하가 적다.

→ 내부적으로 Top-N Sort 알고리즘 사용하기때문이다.
 
 