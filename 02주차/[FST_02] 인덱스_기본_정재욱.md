# 인덱스 구조 및 탐색

## 미리 보는 인덱스 튜닝

### 데이터를 찾는 두 가지 방법

DB table에서 데이터를 찾는 방법은 다음 두 가지이다:

- Table Full Scan
- Index를 이용한다

### 인덱스 튜닝의 두 가지 핵심요소

인덱스 튜닝의 핵심요소는 크게 두 가지로 나뉨:

1. Index scan 효율화 튜닝: 인덱스 스캔 과정에서 발생하는 비효율을 줄이는 것
    - ~~e.g., Index의 순서에 대한 고려 `(이름, 시력)` vs `(시력, 이름)`~~
2. Random access 최소화 튜닝: table access 횟수를 줄이는 것
    - e.g., cardinality가 높은, selectivity가 낮은 항목에 대한 인덱스를 사용해야 함 (`이름` vs `시력`)

두 가지 사항을 모두 고려하여 인덱스 튜닝을 진행해야하나, 둘 중 더 중요한 하나를 고른다면 random access 최소화 튜닝이다. 성능에 미치는 영향이 더 크기 때문이다.

```
그림 2-1
```

### SQL 튜닝은 Random I/O와의 전쟁

DB 성능이 느린 이유는 디스크 I/O 때문이다. 읽어야 할 데이터량이 많고, 그 과정에 디스크 I/O가 많이 발생할 때 느리다. 인덱스를 많이 사용하는 서비스/시스템이라면 디스크 I/O 중에서도 랜덤 I/O가 성능적으로 특히 중요하다.

성능을 위해 DBMS가 제공하는 많은 기능(IOT, 클러스터, 파티션, Prefetch, Batch I/O, Sort merge join, Hash join 등)이 랜덤 I/O를 줄이고 극복하기 위해 개발됐다.

## 인덱스 구조

인덱스를 이용하면 테이블 데이터를 일부만 읽고 멈출 수 있다. 즉, Range Scan이 가능하다. Range scan이 가능한 이유는 인덱스가 정렬돼 있기 때문이다.

DBMS는 일반적으로 B*Tree 인덱스를 사용한다(MySQL InnoDB는 B+Tree 사용). 고객 테이블에 고객명 컬럼 기준으로 만든 B*Tree 인덱스 구조는 다음과 같다:

<aside>
💁‍♂️

B+Tree 구조

![image.png](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdna%2FlmNml%2FbtrhSZR4BuI%2FAAAAAAAAAAAAAAAAAAAAAJ52OYh7p9Ib3J7Xd4KxFH5FdYJSzlqf7CXU7JDFSYjX%2Fimg.png%3Fcredential%3DyqXZFxpELC7KVnFOS48ylbz2pIh7yKj8%26expires%3D1759244399%26allow_ip%3D%26allow_referer%3D%26signature%3DuvshUJjwBo9zRExrjAV9gHwklOk%253D)

</aside>

```
그림 2-3
```

그림에는 나와있지 않지만, 루트와 브랜치 블록에 있는 각 레코드는 하위 블록에 대한 주소값을 갖는다. 키 값은 하위 블록에 저장된 키 값의 범위를 나타낸다.

루트와 브랜치 블록에는 키 값을 갖지 않는 **LMC(Leafmost Child)**라는 레코드를 갖는다. LMC는 자식 노드 중 가장 왼쪽 끝에 위치한 블록을 가리킨다. LMC가 가리키는 주소로 찾아간 블록에는 키 값을 가진 첫 번째 레코드(그림의 루트에서 ‘서’, 왼쪽 브랜치에서 ‘강덕승’, 오른쪽 브랜치에서 ‘송재훈’)보다 작거나 같은 레코드가 저장되어 있다.

리프 블록에 저장된 각 레코드는 키 순서대로 정렬돼 있을 뿐만 아니라 테이블 레코드를 가리키는 주소값, 즉 ROWID를 갖는다. 인덱스 키 값이 같은 경우에는 ROWID 순으로 정렬된다. ROWID는 아래와 같이 DBA(Data Block Address)와 row 번호로 구성되므로 이 값을 알면 테이블 레코드를 찾아갈 수 있다.

- ROWID = 테이블 블록 주소 + row 번호
- Data Block Address = 데이터 파일 번호 + 블록 번호
- 블록 번호: 데이터파일 내에서 부여한 상대적 순번
- Row 번호: 블록 내 순번

인덱스 탐색 과정은 수직적 탐색과 수평적 탐색으로 나눌 수 있다:

- 수직적 탐색: 인덱스 스캔 시작지점을 찾는 과정
- 수평적 탐색: 데이터를 찾는 과정

## 인덱스 수직적 탐색

정렬된 인덱스 레코드 중 조건을 만족하는 첫 번째 레코드를 찾는 과정, 즉 **인덱스 스캔 시작지점을 찾는 과정**이다.

인덱스 수직적 탐색은 루트 블록에서부터 시작한다. 루트를 포함해 브랜치 블록에 저장된 각 인덱스 레코드는 하위 블록에 대한 주소값을 갖는다. 

수직적 탐색 과정에서는 찾고자 하는 값보다 크거나 같은 값을 만나면, 바로 직전 레코드가 가리키는 하위 블록으로 이동한다. 

예를 들어, 그림 2-3에서 ‘이재희’를 찾아보자. 루트 블록에는 ‘이재희’보다 크거나 같은 값이 없다. 그럴 때는 맨 마지막 ‘서’ 레코드가 가리키는 하위 블록으로 이동하면 된다. 브랜치 블록에서는 ‘이재희’보다 큰 레코드인 ‘정재우’를 찾았다. 그러면 바로 직전 레코드(’이재룡’)가 가리키는 하위 블록으로 이동하면 된다. 이렇게 진행하다가 리프 블록에 도달하면 거기서 조건을 만족하는(`고객명=’이재희’`) 첫 번째 레코드를 찾는다.

이번에는 ‘강덕승’을 찾아보자. 루트 블록에 ‘강덕승’보다 큰 값(’서’)이 있으므로 바로 직전 레코드(LMC)가 가리키는 하위 블록으로 이동한다. 이동한 브랜치 블록에는 찾고자하는 값과 정확히 일치하는 레코드가 있다. 그러면 일치하는 레코드의 바로 직전 레코드(LMC)가 가리키는 하위 블록으로 이동한다. 이렇게 해야 첫 번째 리프 블록 맨 마지막에 저장된 ‘강덕승’ 레코드를 빠뜨리지 않는다.

**수직적 탐색은 ‘조건을 만족하는 레코드’를 찾는 과정이 아니라 ‘조건을 만족하는 첫 번째 레코드’를 찾는 과정임을 기억하자**.

## 인덱스 수평적 탐색

수직적 탐색을 통해 스캔 시작점을 찾았으면, 찾고자 하는 데이터가 더 이상 나타나지 않을 때까지 인덱스 리프 블록을 수평적으로 스캔한다. 이 과정이 본격적으로 **데이터를 찾는 과정**이다. 인덱스 리프 블록끼리는 서로 앞뒤 블록에 대한 주소값을 갖는 양방향(double) linked list 구조이므로, 수평적 탐색이 가능하다.

인덱스를 수평적으로 탐색하는 이유는 다음과 같다:

1. 조건절을 만족하는 데이터를 모두 찾기 위해
2. ROWID를 얻기 위해

## 결합(복합) 인덱스 구조와 탐색

고객 테이블에 성별과 고객명 기준으로 만든 인덱스 구조는 아래와 같다:

```
그림 2-5
```

위 그림에서도 이전 예시에서 설명한대로 인덱스 탐색이 진행된다. 남자인 ‘이재희’ 고객을 찾는 경우, 루트 블록에서 찾고자 하는 값보다 큰 첫 레코드를 만나게 된다. 그러면 그 직전 레코드(LMC)가 가리키는 하위 블록인 왼쪽 브랜치 블록으로 이동한다. 브랜치 블록을 스캔하다보면 찾고자 하는 값보다 큰 첫 번째 레코드(’남 & 정재우’)를 만나는데, 이 레코드의 직전 레코드(’남 & 이재룡’)가 가리키는 하위 블록으로 이동한다. 그렇게 이동한 리프 블록에서 찾고자 하는 데이터를 찾으면 된다.

이번에는 인덱스 컬럼 순서를 바꿔보자. 다음은 고객명과 성별 순으로 구성한 복합 인덱스를 도식화한 것이다.

```
그림 2-6
```

주목할 것은, **인덱스를 `(고객명, 성별)`로 구성하든, `(성별, 고객명)`으로 구성하든 읽는 인덱스 블록 개수가 똑같다는 사실이다**. **인덱스 선두 컬럼을 모두 `=` 조건으로 검색할 때는 어느 컬럼을 인덱스 앞쪽에 두든 블록 I/O 개수가 같으므로 성능도 똑같다**.

<aside>
💁‍♂️

**B-Tree Balanced의 의미**

‘Balanced’는 어떤 값으로 탐색하더라도 인덱스 루트에서 리프 블록에 도달하기까지 읽는 블록 수가 같음을 의미한다. 즉, 루트로부터 모든 리프 블록까지의 높이(height)는 항상 같다.

</aside>

# 인덱스 기본 사용법

## 인덱스를 사용한다는 것

인덱스 컬럼(선두 컬럼)을 가공하지 않아야 인덱스를 정상적으로 사용할 수 있다. ‘인덱스를 정상적으로 사용한다’는 표현은 리프 블록에서 스캔 시작점을 찾아 스캔하다가 중간에 멈추는 것을 의미한다. 즉, 리프 블록 일부만 스캔하는 Index Range Scan을 의미한다.

## Index Range Scan 할 수 없는 이유

> 인덱스 컬럼을 가공하면 인덱스를 정상적으로 사용(Range Scan) 할 수 없다.
> 

**인덱스 컬럼을 가공했을 때, 인덱스를 정상적으로 사용할 수 없는 이유는 인덱스 스캔 시작점을 찾을 수 없기 때문**이다. Index Range Scan에서 ‘Range’는 범위를 의미하므로, 일정 범위를 스캔하려면 ‘시작지점’과 ‘끝지점’이 있어야 한다.

예를 들어, 2007년 1월에 태어난 학생을 찾으려면, 우선 2007년 1월 1일(또는 그 이후)에 태어난 첫 번째 학생을 찾는다. 그리고 순서대로 스캔하다가 2007년 2월 1일(또는 그 이후)에 태어난 첫 번째 학생을 만나는 순간 멈추면 된다. 이처럼 명확한 스캔 시작지점과 종료지점이 있다. 이 경우에는 Index Range Scan을 사용할 수 있다.

```sql
WHERE 생년월일 BETWEEN '20070101' and '20070131'
```

이번에는 년도와 상관없이 5월에 태어난 학생을 찾아보자. 스캔 시작점을 찾을 수 있는가? 스캔 시작지점과 종료지점을 알 수 없다. 즉, 전교생을 전부 스캔해야 한다.

```sql
WHERE SUBSTR(생년월일, 5, 2) = '05'
```

아래 조건절도 마찬가지다, 인덱스 스캔 시작지점을 찾을 수 없고, 인덱스를 정상적으로 사용할 수 없다(Index Range Scan 할 수 없다).

```sql
WHERE NVL(주문수량, 0) < 100
```

`LIKE`로 중간 값을 검색할 때도 마찬가지다. ‘대한’으로 시작하는 값은 특정 구간에 모여 있으므로 range scan이 가능하지만, ‘대한’을 포함하는 값은 전체 구간에 걸쳐 흩어져 있어 range scan이 불가능하다.

```sql
-- Index Range Scan 가능
WHERE 업체명 LIKE '대한%'

-- Index Range Scan 불가능
WHERE 업체명 LIKE '%대한%'
```

`OR` 조건으로 검색하는 경우에는 어떨까?

```sql
WHERE (전화번호 = :tel_no OR 고객명 = :cust_nm)
```

수직적 탐색을 통해 전화번호가 ‘01012345678’이거나 고객명이 ‘홍길동’인 어느 한 시작지점을 명확하게 찾을 수 없다. 따라서 인덱스를 어떤 방식으로 구성하더라도 range scan 할 수 없다.

<aside>
💁‍♂️

**OR Epansion**

```sql
SELECT *
FROM 고객
WHERE 고객명 = :cust_nm  -- 고객명이 선두 컬럼인 인덱스를 range scan
UNION ALL
SELECT *
FROM 고객
WHERE 전화번호 = :tel_no  -- 전화번호가 선두 컬럼인 인덱스를 range scan
AND (고객명 <> : cust_nm OR 고객명 IS NULL)
```

`OR` 조건식을 SQL optimizer가 위와 같은 형태로 변환하기도 한다. 이를 **OR Expansion**이라고 한다. 이렇게 하면 index range scan이 작동할 수 있다.

</aside>

`IN` 조건절은 어떨까?

```sql
WHERE 전화번호 IN (:tel_no1, :tel_no2)
```

수직적 탐색을 통해 전화번호가 ‘01012345678’이거나 ‘01098765432’인 어느 한 지점을 바로 찾을 수 있을까? 불가능하다. `IN` 조건은 `OR` 조건을 표현하는 다른 방식일 뿐이다.

하지만, SQL을 아래와 같이 `UNION ALL` 방식으로 처리하면, index range scan이 가능하다.

```sql
SELECT *
FROM 고객
WHERE 전화번호 = :tel_no1
UNION ALL
SELECT *
FROM 고객
WHERE 전화번호 = :tel_no2
```

그래서 `IN` 조건절에 대해서는 SQL optimizer가 IN-List Iterator 방식을 사용한다. IN-List 개수만큼 index range scan을 반복하는 것이다. 이를 통해 SQL을 `UNION ALL` 방식으로 변환한 것과 같은 효과를 얻을 수 있다.

## 더 중요한 인덱스 사용 조건

인덱스를 아래 그림처럼 `(소속팀, 사원명, 연령)`순으로 구성했다고 가정하자.

```sql
그림 2-13
```

아래 조건절에 대해 index range scan이 가능할까?

```sql
SELECT 사원번호, 소속팀, 연령, 입사일자, 전화번호
FROM 사원
WHERE 사원명 = '홍길동'
```

Index range scan이 불가능하다. 이름이 같은 사원이더라도 소속팀이 다르면 서로 멀리 떨어지게 된다. 예를 들어, `사원명 = '홍길동'` 조건을 만족하는 데이터는 그림 2-13에서 보듯 리프 블록 전 구간에 흩어져있다.

**Index range scan하기 위한 가장 첫 번째 조건은 인덱스 선두 컬럼이 조건절에 가공되지 않은 상태로 있어야 한다는 사실이다.**

### 인덱스 잘 타니까 튜닝 끝?

`(주문일자, 상품번호)`로 구성된 `주문상품_N1` 인덱스가 존재한다고 가정해보자.

```
Execution Plan
------------------------------------------------------------
  0     SELECT STATEMENT Optimizer=ALL_ROWS
  1  0    TABLE ACCESS (BY INDEX ROWID) OF '주문상품' (TABLE)
  2  1      INDEX(RANGE SCAN) OF '주문상품_N1' (INDEX)
```

위 쿼리는 인덱스를 잘 타는 것을 확인할 수 있다. 그렇다면, 이 쿼리는 정말 효율적인 쿼리인걸까? 정말 인덱스를 잘 타는지는 인덱스 리프 블록에서 스캔하는 양을 따져봐야 알 수 있다.

이 테이블에 쌓이는 데이터량이 하루에 100만건이라고 가정하자.

```sql
SELECT *
FROM 주문상품
WHERE 주문일자 = :ord_dt
AND 상품번호 LIKE '%PING%';

SELECT *
FROM 주문상품
WHERE 주문일자 = :ord_dt
AND SUBSTR(상품번호, 1, 4) = 'PING';
```

위 SQL에서 상품번호는 스캔 범위를 줄이는 데 아무런 역할도 하지 않는다. 따라서 조건을 만족하는 스캔 데이터량은 100만건이다. 

이러한 문제는 3장 3절 ‘인덱스 스캔 효율화’에서 자세히 살펴볼 것이다.

## 인덱스를 이용한 소트 연산 생략

인덱스는 정렬돼 있다. 인덱스가 정렬돼 있기 때문에 range scan이 가능하고, 지금부터 살펴보고자 하는 소트 연산 생략 효과도 얻게 된다.

PK를 아래 그림처럼 `(장비번호, 변경일자, 변경순번)` 순으로 구성한 상태변경이력 테이블이 있다고 하자.

```sql
그림 2-14
```

이 테이블에서 레코드는 장비번호, 변경일자, 변경순번 순으로 정렬돼 있다. 그렇기에 아래와 같이 검색할 때, 결과는 변경순번 순으로 출력된다.

```sql
SELECT *
FROM 상태변경이력
WHERE 장비번호 = 'C'
AND 변경일자 = '20180316'
```

```
Execution Plan
------------------------------------------------------------
  0     SELECT STATEMENT Optimizer=ALL_ROWS (Cost=85 Card=81 Bytes=5K)
  1  0    TABLE ACCESS (BY INDEX ROWID) OF '상태변경이력' (TABLE) (Cost=85 ...)
  2  1      INDEX (RANGE SCAN) OF '상태변경이력_PK' (INDEX (UNIQUE)) (Cost=3 ...)
```

Optimizer는 이런 특성을 활용해 아래와 같이 `ORDER BY`가 있어도 정렬 연산을 따로 수행하지 않는다. 이미 결과 집합이 정렬되어 있기 때문이다.

```sql
SELECT *
FROM 상태변경이력
WHERE 장비번호 = 'C'
AND 변경일자 = '20180316'
ORDER BY 변경순번
```

```
Execution Plan
------------------------------------------------------------
  0     SELECT STATEMENT Optimizer=ALL_ROWS (Cost=85 Card=81 Bytes=5K)
  1  0    TABLE ACCESS (BY INDEX ROWID) OF '상태변경이력' (TABLE) (Cost=85 ...)
  2  1      INDEX (RANGE SCAN) OF '상태변경이력_PK' (INDEX (UNIQUE)) (Cost=3 ...)
```

만약 정렬 연산을 생략할 수 있게 인덱스가 구성돼 있지 않다면, 아래와 같이 SORT ORDER BY 연산 단계가 추가될 것이다.

```
Execution Plan
------------------------------------------------------------
  0     SELECT STATEMENT Optimizer=ALL_ROWS (Cost=85 Card=81 Bytes=5K)
  1  0    SORT (ORDER BY) (Cost=86 Card=81 Bytes=5K)
  2  1      TABLE ACCESS (BY INDEX ROWID) OF '상태변경이력' (TABLE) (Cost=85 ...)
  3  2        INDEX (RANGE SCAN) OF '상태변경이력_PK' (INDEX (UNIQUE)) (Cost=3 ...)
```

인덱스 리프 블록은 double linked list 구조이기에 내림차순 정렬일 때에도 마찬가지로 인덱스를 활용할 수 있다. 

## ORDER BY 절에서 컬럼 가공

앞서 “인덱스 컬럼을 가공하면 인덱스를 정상적으로 사용할 수 없다”고 했다. 마찬가지로 ORDER BY 또는 SELECT-LIST에서 컬럼을 가공함으로 인해 인덱스를 정상적으로 사용할 수 없는 경우도 있다.

그림 2-14처럼 PK 인덱스를 구성했다면, 아래 SQL도 정렬 연산을 생략할 수 있다.

```sql
SELECT *
FROM 상태변경이력
WHERE 장비번호 = 'C'
ORDER BY 변경일자, 변경순번
```

하지만 아래 SQL은 정렬 연산을 생략할 수 없다. 가공한 값 기준으로 정렬을 요청했기 때문이다.

```sql
SELECT *
FROM 상태변경이력
WHERE 장비번호 = 'C'
ORDER BY 변경일자 | 변경순번
```

## SELECT-LIST에서 컬럼 가공

인덱스를 `(장비번호, 변경일자, 변경순번)` 순으로 구성하면, 아래와 같은 SQL에서도 optimizer는 정렬 연산을 따로 수행하지 않는다. 수직적 탐색을 통해 조건을 만족하는 가장 왼쪽 지점으로 내려가 읽는 첫 번째 레코드가 최솟값이기 때문이다.

```sql
SELECT MIN(변경순번)
FROM 상태변경이력
WHERE 장비번호 = 'C'
AND 변경일자 = '20180316'
```

아래와 같은 SQL도 마찬가지로 정렬 연산을 수행하지 않는다. 수직적 탐색을 통해 조건을 만족하는 가장 오른쪽 지점으로 내려가서 첫 번째 읽는 레코드가 바로 최댓값이다.

```sql
SELECT MAX(변경순번)
FROM 상태변경이력
WHERE 장비번호 = 'C'
AND 변경일자 = '20180316'
```

인덱스를 이용해 이처럼 정렬 연산 없이 최소 또는 최대값을 빠르게 찾을 때는 아래와 같은 실행 계획이 나타난다. 인덱스 리프 블록의 왼쪽(MIN) 또는 오른쪽(MAX)에서 레코드 하나만 읽고 멈춘다.

```
ROWS  ROW Source Operation
----  ---------------------------------
   0  STATEMENT
   1    SORT AGGREGATE (cr=6, pr=0, pw=0, time=81 us)
   1      FIRST ROW (cr=6 pr=0 pw=0 time=59 us)
   1        INDEX RANGE SCAN (MIN/MAX) 상태변경이력_PK (cr=6 pr=0 pw=0 ...)
```

## 자동 형변환

고객 테이블에 생년월일이 선두 컬럼인 인덱스가 있다고 하자. 아래 SQL은 컬럼을 조건절에서 가공하지 않았음에도 Full Table Scan이 발생했다.

```
SELECT * FROM 고객
WHERE 생년월일 = 19821225
```

이유는 실행 계획을 보면 알 수 있다.

```
Execution Plan
--------------------------------------------------------------------
0     SELECT STATEMENT Optimizer=ALL_ROWS (Cost=3 Card=1 Bytes=38)
1  0    TABLE ACCESS (FULL) OF '고객' (TABLE) (Cost=3 Card=1 Bytes=38)

Predicate information (identified by operation id):
--------------------------------------------------------------------
1 - filter(TO_NUMBER("생년월일")=19821225
```

Optimizer가 SQL을 아래와 같이 변환했고, 그로 인해 인덱스 컬럼이 가공됐기 때문에 인덱스를 range scan 할 수 없게 된 것이다.

```
SELECT * FROM 고객
WHERE TO_NUMBER(생년월일) = 19821225
```

이는 고객 테이블 생년월일 컬럼이 문자형인데 조건절 비교값을 숫자형으로 표현했기 때문에 나타난 현상이다. 오라클은 이런 경우 자동으로 형변환을 한다.

---

# 인덱스 확장기능 사용법

## Index Range Scan

Index Range Scan은 B*Tree 인덱스의 가장 일반적이고 정상적인 형태의 액세스 방식이다. 아래 그림처럼 인덱스 루트에서 리프 블록까지 수직적으로 탐색한 후에 ‘필요한 범위(Range)만’ 스캔한다.

```
그림 2-15
```

실행계획은 아래와 같다.

```sql
SET AUTOTRACE TRACEONLY exp

SELECT * FROM emp WHERE deptno = 20;
```

```
Execution Plan
-------------------------------------------------------------
0     SELECT STATEMENT Optimizer=ALL_ROWS
1  0    TABLE ACCESS (BY INDEX ROWID) OF 'EMP' (TABLE)
2  1      INDEX (RANGE SCAN) OF 'EMP_DEPTNO_IDX' (INDEX)
```

이전 내용에서도 살펴봤듯이, 인덱스를 range scan하려면 선두 컬럼을 가공하지 않은 상태로 조건절에 사용해야 한다. 

주의해야 할 점이 하나 있는데, 실행계획을 보고 ‘인덱스 잘 타니까 성능도 좋겠지’라고 생각하면 안 된다는 것이다. 성능은 인덱스 스캔 범위, 테이블 액세스 횟수를 얼마나 줄일 수 있느냐로 결정된다.

## Index Full Scan

Index Full Scan은 아래 그림처럼 수직적 탐색 없이 인덱스 리프 블록을 처음부터 끝까지 수평적으로 탐색하는 방식이다.

```
그림 2-16
```

실행 계획은 아래와 같다.

```sql
CREATE INDEX emp_ename_sal_idx ON emp (ename, sal);

SET AUTOTRACE TRACEONLY exp

SELECT * FROM exp
WHERE sal > 2000
ORDER BY ename;
```

```
Execution Plan
-------------------------------------------------------------
0     SELECT STATEMENT Optimizer=ALL_ROWS
1  0    TABLE ACCESS (BY INDEX ROWID) OF 'EMP' (TABLE)
2  1      INDEX (FULL SCAN) OF 'EMP_ENAME_SAL_IDX' (INDEX)
```

Index Full Scan은 대개 데이터 검색을 위한 최적의 인덱스가 없을 때 차선으로 선택된다. 위 SQL에서 인덱스 선두 컬럼인 `ename`이 조건절에 옶으므로 Index Range Scan은 불가능하다. 뒤쪽이긴 하지만 `sal` 컬럼이 인덱스에 있으므로 Index Full Scan을 통해 `sal`이 2000보다 큰 레코드를 찾을 수 있다.

### Index Full Scan의 효용성

위 SQL처럼 인덱스 선두 컬럼이 조건절에 없으면 optimizer는 우선 Table Full Scan을 고려한다. 그런데 대용량 테이블이어서 Tabler Full Scan에 대한 부담이 크다면, optimizer는 index를 활용하는 것을 다시 고려해본다. 그럴 때 optimizer는 Index Full Scan 방식을 선택할 수 있다. 물론, 이 방식은 Index Range Scan의 차선책으로 선택한 것이므로, 수행 빈도가 높은 SQL이라면 `sal` 컬럼이 선두인 인덱스를 생성해주는 것이 좋다.

### 인덱스를 이용한 소트 연산 생략

Index Full Scan하면 Range Scan과 마찬가지로 결과 집합을 정렬된 상태로 얻을 수 있다. 따라서 Sort Order By 연산을 생략할 목적으로 사용될 수도 있다. 이는 Range Scan의 차선책이 아닌 optimizer가 전략적으로 선택한 결과일 수 있다.

## Index Unique Scan

Index Unique Scan은 아래 그림처럼 수직적 탐색만으로 데이터를 찾는 스캔 방식으로서, Unique 인덱스를 ‘=’ 조건으로 탐색하는 경우에 작동한다.

```
그림 2-19
```

실행계획은 아래와 같다.

```sql
CREATE UNIQUE INDEX pk_emp ON emp(empno);

ALTER TABLE emp ADD CONSTRAINT pk_emp PRIMARY KEY(empno) USING INDEX pk_emp;

SET AUTOTRACE TRACEONLY EXPLAIN;

SELECT empno, ename, FROM emp WHERE empno = 7788;
```

```
Execution Plan
---------------------------------------------
0     SELECT STATEMENT Optimizer=ALL_ROWS
1  0    TABLE ACCESS (BY INDEX ROWID) OF 'EMP'
2  1      INDEX (UNIQUE SCAN) OF 'PK_EMP' (UNIQUE)
```

Unique index가 존재하는 컬럼은 중복되지 않으므로 조건에 일치하는 데이터를 한 건 찾는 순간 더 이상 탐색할 필요가 없다.

Unique index라고 해도 범위 검색 조건(`BETWEEN`, `>`, `<`, `LIKE` 등)으로 검색할 때는 Index Range Scan으로 처리된다. 수직적 탐색만으로는 조건에 해당하는 레코드를 모두 찾을 수 없기 때문이다.

## Index Skip Scan

인덱스 선두 컬럼을 조건절에 사용하지 않으면 optimizer는 기본적으로 Table Full Scan 또는 Index Full Scan을 사용한다. 그러나 인덱스 선두 컬럼이 조건절에 없더라도 인덱스를 활용하는 경우도 있다.

Index Skip Scan은 **조건절에 빠진 인덱스 선두 컬럼의 Dictinct Value 개수가 적고, 후행 컬럼의 Distinct Value 개수가 많을 때 유용**하게 사용될 수 있다.

아래 그림은 `(성별, 연봉)`으로 구성된 결합 인덱스가 적용된, 인덱스 루트 블록과 리프 블록이다.

```
그림 2-20
```

아래와 같이 성별과 연봉 컬럼에 대한 조건을 모두 사용했을 때는 당연히 Index Range Scan이 정상적으로 동작할 것이다.

```sql
SELECT * FROM 사원 WHERE 성별 = '남' AND 연봉 BETWEEN 2000 AND 4000
```

이제 인덱스 선두 컬럼인 성별 조건을 뺀 SQL을 통해 Index Skip Scan의 원리를 살펴보자. 이 스캔 방식을 유도하거나 방지하고자 할 때에는 `index_ss`, `no_index_ss` 힌트를 사용한다.

```sql
SELECT /*+ index_ss(사원 사원_IDX) */ *
FROM 사원
WHERE 연봉 BETWEEN 2000 AND 4000;
```

```
Execution Plan
-------------------------------------------------------------
0     SELECT STATEMENT Optimizer=ALL_ROWS
1  0    TABLE ACCESS (BY INDEX ROWID) OF '사원' (TABLE)
2  1      INDEX (SKIP SCAN) OF '사원_IDX' (INDEX)
```

Index Skip Scan은 루트 또는 브랜치 블록에서 읽은 컬럼 값 정보를 이용해 조건절에 부합하는 레코드를 포함할 ‘가능성이 있는’ 리프 블록만 골라서 액세스하는 스캔 방식이다.

그림 2-20을 보면서 이해해보자. 인덱스 루트 블록에서 첫 번째 레코드가 가리키는 리프 블록은 ‘남 & 800 이하’인 레코드를 담고 있다. 이 블록은 액세스하지 않아도 될 것 같지만, ‘남’보다 작은 성별이 있다면, 그 사원에 대한 인덱스 레코드는 모두 1번 리프 블록에 저장되므로 액세스해야 한다. 성별에 ‘남’과 ‘여’ 두 개 값만 존재한다는 사실을 optimizer는 모른다.

두 번째 레코드가 가리키는 리프 블록은 ‘남 & 800’ 이상이면서 ‘남 & 1500’ 이하인 레코드를 담고 있다. 조건에 맞는 레코드가 존재할 가능성이 없으므로 Skip한다.

세 번째 레코드가 가리키는 리프 블록은 조건에 부합하는 레코드를 담고 있을 것이므로 액세스한다.

네 번째 레코드가 가리키는 리프 블록은 조건에 맞는 레코드가 존재할 가능성이 없으므로 Skip한다. 같은 이유로 다섯 번째 리프 블록도 Skip한다.

여섯 번째 레코드가 가리키는 리프 블록은 ‘남 & 10000’ 이상이므로 조건 범위를 초과한다. 하지만, 액세스해야 한다. 여자 중에서 ‘연봉 < 3000’이거나 ‘남’과 ‘여’ 사이에 다른 성별이 존재한다면 이 리프 블록에 저장되고, 연봉 = 3000인 여자 직원도 뒤쪽에 일부 저장될 수 있기 때문이다.

일곱 번째 레코드가 가리키는 리프 블록은 액세스하고, 여덟 번째와 아홉 번째 레코드가 가리키는 리프 블록은 Skip 해도 된다.

마지막으로 열 번째 리프 블록은 ‘여 & 10000’ 이상이므로 조건 구간을 초과하지만, ‘여’보다 값이 큰 성별이 존재한다면 여기에 저장될 것이므로 액세스해야만 한다.

지금까지 살펴본 Index Skip Scan 과정을 그림으로 표현해보면 아래와 같다.

### Index Skip Scan이 작동하기 위한 조건

앞서 Index Skip Scan은 Distinct Value 개수가 적은 선두 컬럼이 조건절에 없고, 후행 컬럼의 Distinct Value 개수가 많을 때 효과적이라는 것을 살펴봤다. 하지만, 인덱스 선두 컬럼이 조건절에 없을 때만 Index Skip Scan이 작동하는 것은 아니다.

예를 들어, 인덱스 구성이 다음과 같다고 하자.

```
일별업종별거래_PK : 업종유형코드 + 업종코드 + 기준일자
```

이때, 아래 SQL처럼 선두 컬럼에 대한 조건절은 있지만, 중간 컬럼에 대한 조건절이 없는 경우에도 Skip Scan을 사용할 수 있다.

```sql
SELECT /*+ INDEX_SS(A 일별업종별거래_PK) */
       기준일자, 업종코드, 체결건수, 체결수량, 거래대금
FROM 일별업종별거래 A
WHERE 업종유형코드 = '01'
AND 기준일자 BETWEEN '20080501' AND '20080531'
```

```
Execution Plan
-------------------------------------------------------------
0     SELECT STATEMENT Optimizer=ALL_ROWS (Cost=91 Card=7 Bytes=245)
1  0    TABLE ACCESS (BY INDEX ROWID) OF '일별업종별거래' (TABLE) (Cost=91 ...)
2  1      INDEX (SKIP SCAN) OF '일별업종별거래_PK' (INDEX (UNIQUE)) (Cost=102 ...)
```

마찬가지로 아래와 같이 Distinct Value가 적은 두 개의 선두 컬럼이 모두 조건절에 없는 경우에도 사용할 수 있다.

```sql
SELECT /*+ INDEX_SS(A 일별업종별거래_PK) */
       기준일자, 업종코드, 체결건수, 체결수량, 거래대금
FROM 일별업종별거래 A
WHERE 업종유형코드 = '01'
AND 기준일자 BETWEEN '20080501' AND '20080531'
```

```
Execution Plan
-------------------------------------------------------------
0     SELECT STATEMENT Optimizer=ALL_ROWS (Cost=91 Card=37 Bytes=1K)
1  0    TABLE ACCESS (BY INDEX ROWID) OF '일별업종별거래' (TABLE) (Cost=91 ...)
2  1      INDEX (SKIP SCAN) OF '일별업종별거래_PK' (INDEX (UNIQUE)) (Cost=90 Card=1)
```

선두 컬럼이 부등호, `BETWEEN`, `LIKE` 같은 범위검색 조건일 때도 Index Skip Scan을 사용할 수 있다.

예를 들어, 일별업종별거래 테이블에 아래와 같은 인덱스가 있다고 하자.

```
일별업종별거래_X01 : 기준일자 + 업종유형코드
```

SQL은 아래와 같다.

```sql
SELECT /*+ INDEX_SS(A 일별업종별거래_X01) */
       기준일자, 업종코드, 체결건수, 체결수량, 거래대금
FROM 일별업종별거래 A
WHERE 기준일자 BETWEEN '20080501' AND '20080531'
AND 업무유형코드 = '01'
```

위 SQL에 Index Range Scan을 사용한다면, 기준일자 `BETWEEN` 조건을 만족하는 인덱스 구간을 ‘모두’ 스캔해야 한다. Index Skip Scan을 사용한다면, 기준일자 `BETWEEN` 조건을 만족하는 인덱스 구간에서 `업종유형코드 = '01'`인 레코드를 ‘포함할 가능성이 있는 리프 블록만’ 골라서 액세스할 수 있다.

이처럼 Index Range Scan이 불가능하거나 효율적이지 못한 상황에서 Index Skip Scan이 도움이 될 수 있다.

하지만, **인덱스는 기본적으로 최적의 Index Range Scan을 목표로 설계해야 하며, 수행 횟수가 적은 SQL을 위해 인덱스를 추가하는 것이 비효율적일 때 이들 스캔 방식을 차선책으로 활용하는 전략이 바람직하다**.

## Index Fast Full Scan

Index Fast Full Scan은 논리적인 인덱스 트리 구조를 무시하고 인덱스 세그먼트 전체를 Multiblock I/O 방식으로 스캔하기에 Index Full Scan보다 빠르다. 관련 힌트는 `index_ffs`와 `no_index_ffs`이다.

아래 그림은 인덱스의 논리적인 연결 구조를 표시한 것이다.

```sql
그림 2-22
```

아래 그림은 그림 2-22에 논리적으로 배치한 블록들을 물리적 순서로 배치했지만, 논리적 순서를 화살표로 표시한 것이다.

```sql
그림 2-23
```

Index Full Scan은 인덱스의 논리적 구조를 따라 블록을 읽어들인다. 반면에, Index Fast Full Scan은 물리적으로 디스크에 저장된 순서대로 인덱스 리프 블록들을 읽어들인다. Multiblock I/O 방식으로 그림 2-23의 왼쪽 익스텐트에서 1 → 2 → 10 → 3 → 9번 순으로 읽고, 그 다음 오른쪽 익스텐트에서 8 → 7 → 4 → 5 → 6번 순으로 읽는다.

Index Fast Full Scan은 Multiblock I/O 방식을 사용하므로 디스크로부터 대량의 인덱스 블록을 읽어야 할 때 유용하다. 속도는 빠르지만, 논리적 연결 구조를 무시한 채 데이터를 읽기 때문에 결과 집합이 인덱스 키 순서대로 정렬되지 않는다. 또한, 쿼리에 사용한 컬럼이 모두 인덱스에 포함돼 있을 때(covering index)만 사용할 수 있다는 점도 특징이다.

## Index Range Scan Descending

Index Range Scan과 기본적으로 동일하나, 인덱스를 뒤에서부터 앞쪽으로 스캔하기 때문에 내림차순으로 정렬된 결과 집합을 얻을 수 있다.

```sql
그림 2-24
```

아래처럼 EMP 테이블을 EMPNO 기준으로 내림차순 정렬하고자 하고, EMPNO 컬럼에 대한 인덱스가 있으면 optimizer가 알아서 인덱스를 거꾸로 읽는 실행 계획을 수립한다.

```sql
SELECT * FROM EMP
WHERE EMPNO > 0
ORDER BY EMPNO DESC
```

```
Execution Plan
-------------------------------------------------------------
0     SELECT STATEMENT Optimizer=ALL_ROWS
1  0    TABLE ACCESS (BY INDEX ROWID) OF 'EMP' (TABLE)
2  1      INDEX (RANGE SCAN DESCENDING) OF 'PK_EMP' (INDEX (UNIQUE))
```

아래처럼 MAX 값을 구하고자 할 때도 해당 컬럼에 인덱스가 있으면 인덱스를 뒤에서부터 한 건만 읽고 멈추는 실행계획이 자동으로 수립된다.

```sql
CREATE INDEX emp_x02 ON emp(deptno, sal);

SELECT deptno, dname, loc, (SELECT MAX(sal) FROM emp WHERE deptno = d.deptno)
FROM dept d
```

```
Execution Plan
-------------------------------------------------------------
0     SELECT STATEMENT Optimizer=ALL_ROWS
1  0    SORT (AGGREGATE)
2  1      FIRST ROW
3  2        INDEX (RANGE SCAN (MIN/MAX)) OF 'EMP_X02' (INDEX)
4  0    TABLE ACCESS (FULL) OF 'DEPT' (TABLE)
```
