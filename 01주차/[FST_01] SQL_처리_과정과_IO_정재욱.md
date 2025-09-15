# SQL 파싱과 최적화

## 구조적, 집합적, 선언적 질의 언어

RDBMS는 SQL 최적화를 수행

## SQL 최적화

SQL을 실행하기 전 최적화 과정을 세분화하면 아래와 같다.

1. SQL 파싱
    - 파싱 트리 생성: SQL문을 이루는 개별 구성요소를 분석해서 파싱 트리 생성
    - Syntax 체크: 문법적 오류가 없는지 확인. 예를 들어, 사용할 수 없는 키워드를 사용했거나 순서가 바르지 않거나 누락된 키워드가 있는지 확인
    - Semantic 체크: 의미상 오류가 없는지 확인. 예를 들어, 존재하지 않는 테이블 또는 컬럼을 사용했는지, 사용한 오브젝트에 대한 권한이 있는지 확인
2. SQL 최적화: optimizer는 미리 수집한 시스템 및 오브젝트 통계 정보를 바탕으로 다양한 실행 경로를 생성 및 비교한 후 가장 효율적인 것을 선택
3. Row Source 생성: optimizer가 선택한 실행 경로를 실제 실행 가능한 코드 또는 프로시저 형태로 포매팅

## SQL Optimizer

SQL optimizer는 작업에 대해 최적의 데이터 access 경로를 선택해주는 엔진이다. Optimizer의 최적화 단계를 요약하면 다음과 같다:

1. 쿼리를 수행하는 데 후보군이 될만한 실행 계획들을 찾는다
2. Data dictionary에 미리 수집해 둔 오브젝트 통계 및 시스템 통계 정보를 이용해 각 실행 계획의 예상 비용을 산정한다
3. 최저 비용을 나타내는 실행 계획을 선택한다

## 실행계획과 비용

Execution plan은 SQL optimizer가 생성한 처리 절차를 사용자가 확인할 수 있게 다음과 같이 트리 구조로 표현한 것을 말한다:

```
Execution Plan
----------------------------------------------------------------------
0     SELECT STATEMENT Optimizer=ALL_ROWS (Cost=209 Card=5 Bytes=175)
1  0    TABLE ACCESS (BY INDEX ROWID) OF 'EMP' (Cost=2 Card=5 Bytes=85)
2  1      NESTED LOOPS (Cost=209 Card=5 Bytes=175)
3  2        TABLE ACCESS (BY INDEX ROWID) OF 'DEPT' (Cost=207 Card=1 Bytes=18)
4  3          INDEX (RANGE SCAN) OF 'DEPT_LOC_IDX' (NON-UNIQUE) (Cost=7 Card=1)
5  4        INDEX (RANGE SCAN) OF 'EMP_DEPTNO_IDX' (NON-UNIQUE) (Cost=1 Card=5)
```

이 기능을 통해 작선한 쿼리가 테이블을 스캔하는지 인덱스를 스캔하는지, 인덱스를 스캔한다면 어떤 인덱스인지를 확인할 수 있고, 예상과 다른 방식으로 처리된다면 실행 경로를 변경할 수 있다.

Optimizer가 index를 선택하는 근거는 비용이다. 비용(Cost)은 쿼리를 수행할 동안 발생할 것으로 예상하는 I/O 횟수 또는 예상 소요시간을 표현한 값이다.

SQL 실행계획에 표시되는 Cost는 어디까지나 예상치다. 실행경로를 선택하기 위해 optimizer가 여러 통계정보를 활용해서 계산해 낸 값이다. 실측치가 아니므로 실제 수행할 때의 결과와 차이가 난다.

---

# SQL 공유 및 재사용

## 소프트 파싱 vs 하드 파싱

사용자가 SQL 문을 전달하면 DBMS는 SQL을 파싱한 후 해당 SQL이 library cache에 존재하는지 확인한다. Cache에서 찾으면 곧바로 실행 단계로 넘어가지만, 찾지 못하면 최적화 단계를 거친다. SQL을 캐시에서 찾아 곧바로 실행 단계로 넘어가는 것을 Soft Parsing이라 하고, 찾는 데 실패해 최적화 및 row source 생성 단계까지 모두 거치는 것을 Hard Parsing이라 한다.

## Bind 변수의 중요성

Custom 함수/프로시저, 트리거, 패키지 등은 생성할 때부터 이름을 갖는다. 실행할 때 library cache에 적재함으로써 여러 사용자가 공유하면서 재사용한다. 반면, SQL은 이름이 따로 없다. 전체 SQL 텍스트가 이름 역할을 한다. 딕셔너리에 저장하지도 않는다. 최초 실행 시 최적화 과정을 거쳐 동적으로 생성한 내부 프로시저를 library cache에 적재함으로써 여러 사용자가 공유하면서 재사용한다. 캐시 공간이 부족하면 버려졌다가 다음에 다시 실행할 때 똑같은 최적화 과정을 거쳐 캐시에 적재된다.

### 공유 가능 SQL

Library cache에서 SQL을 찾기 위해 사용하는 key value가 ‘SQL문 그 자체’이므로 아래는 모두 다른 SQL이다:

```sql
SELECT * FROM emp WHERE empno = 7900;
select * from EMP where EMPNO = 7900;
select * from emp where empno = 7900;
select * from emp where empno = 7900 ;
select * from emp where empno = 7900  ;
select * from scott.emp where empno = 7900;
select /* comment */ * from emp where empno = 7900;
select /*+ first_rows */ * from emp where empno = 7900;
```

즉, 쿼리마다 DBMS 내부 프로시저를 하나씩 만들어서 library cache에 적재하고 있는 것이다.

그렇다면 로그인 ID를 파라미터로 받는 프로시저 하나를 공유하면서 재사용하는 것이 효율적일 것이다.

```sql
String stmt = "SELECT * FROM CUSTOMER WHERE LOGIN_ID = ?";
```

이처럼 parameter driven 방식으로 SQL을 작성하는 방법이 제공되는데, bind 변수가 바로 그것이다.

이렇게 하면, SQL에 대한 hard parsing은 최초 한 번만 일어나고, 캐싱된 SQL을 100만 고객이 공유하면서 재사용한다.

---

# 데이터 저장 구조 및 I/O 메커니즘

## SQL이 느린 이유

SQL이 느린 이유는 디스크 I/O 때문이다.

프로세스는 디스크에서 데이터를 읽어야 할 때, CPU를 OS에 반환하고 잠시 waiting 상태에서 I/O가 완료되기를 기다린다. OS system call을 호출하고 CPU를 반환한 채 알람 설정한 후 wait queue에서 잠을 자는 것이다. 그러니 I/O가 많으면 느릴 수 밖에 없다.

## 데이터베이스 저장 구조 (MySQL 기준)

MySQL InnoDB에서는 **시스템 테이블스페이스**(ibdata1)와 **개별 테이블스페이스**(.ibd 파일) 두 가지 방식이 있다. `innodb_file_per_table=ON` 설정 시 각 테이블마다 별도의 .ibd 파일이 생성된다.

**세그먼트**는 테이블의 데이터나 인덱스처럼 저장공간이 필요한 오브젝트다. InnoDB에서 하나의 테이블은 여러 세그먼트로 구성된다:

- 클러스터드 인덱스 세그먼트 (데이터)
- 보조 인덱스 세그먼트들
- 롤백 세그먼트

각 파티션도 별도의 세그먼트가 되며, TEXT/BLOB 같은 대용량 컬럼은 별도 페이지에 저장된다.

**익스텐트**는 공간을 확장하는 단위다. InnoDB에서 익스텐트는 **64개의 연속된 페이지**(64 × 16KB = 1MB)로 구성된다. 테이블이나 인덱스에 데이터가 추가되어 공간이 부족하면 새로운 익스텐트를 할당받는다.

실제 데이터가 저장되는 단위는 **페이지**다. MySQL InnoDB는 기본적으로 **16KB 페이지**를 사용한다. 한 페이지는 하나의 테이블이 독점하며, 같은 페이지 내 모든 레코드는 동일한 테이블에 속한다. 하나의 익스텐트도 하나의 세그먼트가 독점한다.

**파일 구조**:

- **System Tablespace** (ibdata1): 시스템 정보, 언두 로그 등
- **File-per-table** (.ibd): 개별 테이블 데이터와 인덱스
- **Redo Log** (ib_logfile): 트랜잭션 로그
- **Binary Log**: 복제용 로그

익스텐트 내 페이지들은 물리적으로 연속된 공간이지만, 서로 다른 익스텐트들은 연속되지 않을 수 있다. 특히 file-per-table 방식에서는 각 테이블이 별도 .ibd 파일을 가지므로 물리적 분리가 더 명확하다.

**MySQL 저장 구조 정리:**

- **페이지**: 데이터를 읽고 쓰는 단위 (16KB 기본)
- **익스텐트**: 공간 확장 단위, 64개 연속 페이지 (1MB)
- **세그먼트**: 테이블의 데이터/인덱스 저장 공간
- **테이블스페이스**: 세그먼트들을 담는 논리적 컨테이너
- **물리적 파일**: .ibd, ibdata1 등 실제 OS 파일

## 블록 단위 I/O

DBMS는 블록(또는 페이지) 단위로 데이터를 읽고 쓴다. 그러므로 레코드 하나를 읽고 싶어도 해당 블록을 통째로 읽는다.

오라클은 기본적으로 8KB 크기의 블록을 사용하므로 1Byte 데이터를 읽기 위해서도 8KB를 읽게 된다.

테이블뿐만 아니라 인덱스도 블록 단위로 데이터를 읽고 쓴다.

## Sequential access vs Random access

테이블 또는 인덱스 블록을 access하는 방식으로는 sequential access와 random access 두 가지가 있다.

- Sequential access: 논리적 또는 물리적으로 연결된 순서에 따라 차례대로 블록을 읽는 방식
- Random access: 논리적, 물리적인 순서를 따르지 않고, 레코드 하나를 읽기 위해 한 block씩 접근하는 방식

## Logical I/O vs Physical I/O

논리적 블록 I/O는 SQL을 처리하는 과정에서 발생한 총 블록 I/O를 말한다.

물리적 블록 I/O는 디스크에서 발생한 총 블록 I/O를 말한다.

메모리 I/O는 전기적 신호인 데 반해, 디스크 I/O는 access arm을 통해 물리적 작용이 일어나므로 메모리 I/O에 비해 상당히 느리다.

### Buffer cache hit rate (BCHR)

```
BCHR = (캐시에서 곧바로 찾은 블록 수 / 총 읽은 블록 수) * 100
     = ((논리적 I/O - 물리적 I/O) / 논리적 I/O) * 100
     = (1 - (물리적 I/O) / (논리적 I/O)) * 100
```

BCHR 공식을 다음과 같이 변형하면: 

```
물리적 I/O = 논리적 I/O * (100% - BCHR)
```

물리적 I/O가 성능을 결정하지만, 실제 SQL 성능을 향상하려면 물리적 I/O가 아닌 논리적 I/O를 줄여야 한다는 사실을 알 수 있다.

물리적 I/O는 결국 시스템 상황에 의해 결정되는 통제 불가능한 외생변수이므로, SQL 성능을 높이기 위해 할 수 있는 일은 논리적 I/O를 줄이는 일뿐이다.

그럼 논리적 I/O는 어떻게 줄일 수 있을까? SQL을 튜닝해서 읽는 총 블록 개수를 줄이면 된다. 논리적 I/O는 SQL 튜닝을 통해 줄일 수 있는 통제 가능한 내생변수다. **논리적 I/O를 줄임으로써 물리적 I/O를 줄이는 것이 곧 SQL 튜닝이다**.

BCHR에는 주의해야 할 함정이 있다. BCHR이 SQL 성능을 좌우하지만, BCHR이 높다고 해서 효율적인 SQL을 의미하는 것은 아니라는 것이다. SQL이 효율적이든 비효율적이든 같은 블록을 반복해서 읽기만 하면 BCHR이 높아진다. 이에 유의하자.

## Single Block I/O vs Multiblock I/O

I/O call을 통해 디스크에서 DB buffer cache로 블록을 적재할 때, 한 번에 한 블록씩 요청하기도 하고, 여러 블록씩 요청하기도 한다.

- Single Block I/O: 한 번에 한 블록씩 요청해서 메모리에 적재하는 방식
- Multiblock I/O: 한 번에 여러 블록씩 요청해서 메모리에 적재하는 방식

인덱스를 이용할 때는 기본적으로 인덱스와 테이블 블록 모두 Single Block I/O 방식을 사용한다. 구체적으로 다음 목록이 Single Block I/O 대상 operation이다:

- 인덱스 루트 블록을 읽을 때
- 인덱스 루트 블록에서 얻은 주소 정보로 브랜치 블록을 읽을 때
- 인덱스 브랜치 블록에서 얻은 주소 정보로 리프 블록을 읽을 때
- 인덱스 리프 블록에서 얻은 주소 정보로 테이블 블록을 읽을 때

인덱스는 소량 데이터를 읽을 때 주로 사용하므로 이 방식이 효율적이다.

Multiblock I/O는 특정 블록을 디스크에서 읽으려고 I/O call 할 때, 디스크 상에 그 블록과 ‘인접한’ 블록들을 한꺼번에 읽어 캐시에 미리 적재하는 기능이다. 그러므로, 많은 데이터 블록을 읽을 때는 Multiblock I/O 방식이 효율적이다. 

## Table Full Scan vs Index Range Scan

- Table Full Scan: 테이블에 속한 블록 ‘전체’를 읽어서 사용자가 원하는 데이터를 찾는 방식
- Index Range Scan: 인덱스에서 ‘일정량’을 스캔하면서 얻은 ROWID로 테이블 레코드를 찾아가는 방식 (ROWID는 테이블 레코드가 디스크 상에 어디 저장됐는지를 가리키는 위치 정보)

Table Full Scan은 sequential access와 Multiblock I/O 방식으로 디스크 블록을 읽는다. 한 블록에 속한 모든 레코드를 한 번에 읽어들이고, 한 번의 디스크 I/O call을 통해 수십~수백개의 블록을 한꺼번에 I/O한다.

Index Range Scan을 통한 테이블 액세스는 random access와 Single Block I/O 방식으로 디스크 블록을 읽는다. 따라서 많은 데이터를 읽을 때는 Table Full Scan보다 불리하다. 게다가 이 방식은 읽었던 블록을 반복해서 읽는 비효율이 존재한다. 물리적 블록 I/O 뿐만 아니라 논리적 블록 I/O 측면에서도 불리하다는 얘기다. 한 블록에 평균 500개 레코드가 있으면, 같은 블록을 최대 500번 읽을 수도 있다. 각 블록을 단 한 번만 읽는 Table Full Scan보다 훨씬 불리하다.

물론, sequential access와 Multiblock I/O가 아무리 좋아도, 큰 테이블에서 소량 데이터를 검색할 때는 반드시 인덱스를 사용해야 한다.

## 캐시 탐색 메커니즘

아래 operation은 모두 buffer cache 탐색 과정을 거친다:

- 인덱스 루트 블록을 읽을 때
- 인덱스 루트 블록에서 얻은 주소 정보로 브랜치 블록을 읽을 때
- 인덱스 브랜치 블록에서 얻은 주소 정보로 리프 블록을 읽을 때
- 인덱스 리프 블록에서 얻은 주소 정보로 테이블 블록을 읽을 때
- 테이블 블록을 Full Scan 할 때

Buffer cache에서 블록을 찾을 때, 이처럼 해시 알고리즘으로 buffer header를 찾고, 거기서 얻은 pointer로 buffer block을 액세스하는 방식을 사용한다. 해시 구조의 특징을 정리하면 다음과 같다:

- 같은 입력은 항상 동일한 hash chain에 연결됨
- 다른 입력이 동일한 hash chain에 연결될 수 있음
- Hash chain 내에서는 정렬이 보장되지 않음

### 메모리 공유자원에 대한 Access Serialization

Buffer cache는 SGA 구성요소이므로 buffer cache에 캐싱된 buffer block은 모두 공유자원이다. 그러므로 동시성 문제가 생길 수 있다.

따라서 자원을 공유하는 것처럼 보여도 내부에선 한 프로세스씩 순차 접근하도록 구현하고 있으며, 이를 위한 메커니즘이 serialization이다. 간단하게 ‘줄 세우기’인데, 이러한 줄 세우기가 가능하도록 지원하는 메커니즘이 Latch이다.

SGA를 구성하는 sub cache마다 별도의 latch가 존재하는데, buffer cache에는 cache buffer chain latch, cache buffer LRU chain latch 등이 작동한다. 또한, buffer block 자체에도 ‘Buffer Lock’이라는 serialization 메커니즘이 존재한다.

이러한 serialization 메커니즘으로 인해 경합이 발생하고, 이는 성능 저하로 이어질 수 있다. 결국 이러한 serialization 메커니즘에 의한 캐시 경합을 줄이려면, SQL 튜닝을 통해 논리적 I/O 자체를 줄여야 한다.

---

# 알게 된 점 / 감상

이번 내용은 크게 두 가지를 말하고 있는 것으로 이해했다.

1. SQL과 DBMS의 전반적인 개념(구조, 정의 등)
2. SQL 튜닝을 왜 해야하고, SQL 튜능으로 얻고자 하는 목표가 무엇인지 => 결국 논리적 Block I/O를 줄이자는 이야기

우리의 SQL이 왜 느리고, 느린 쿼리는 어떻게 개선해야 하는지에 대한 내용을 OS 수준의 디스크 I/O와 SGA buffer cache 개념을 통해 알 수 있었다. 원래도 막연하게 알고 있던 내용이지만, 책을 통해 내용을 정리하면서 명확하게 정리할 수 있었다.

전반적으로 내용이 Oracle RDBMS 기준으로 작성되어 있기에 기존 지식과 충돌하는 부분이 있어 혼동되기도 하였다. 그 때문에, '데이터 저장 구조' 내용은 MySQL 기준으로 살펴보기도 하였다. 

이렇게 내용을 비교해보니 MySQL(InnoDB)과 Oracle이 다른 부분도 있지만 전반적인 개념은 공유한다는 것을 알게 되었다. 예를 들어 레코드에 존재하는 크기가 큰 데이터를 저장하기 위해 MySQL은 off-page라는 개념을 사용하지만 Oracle은 Lob segment라는 개념을 사용한다고 한다. 하지만, 결국 큰 용량의 데이터는 별도로 저장하고, 그에 대한 pointer를 레코드에서 관리한다는 개념은 똑같은 것처럼 서로 다른 RDMBS더라도 전반적인/핵심적인 개념은 공유하는 것으로 보인다.
