# Table Access 최소화

이전에 살펴봤듯, SQL 튜닝은 느린 랜덤 I/O를 줄이기 위한 작업이다. 우선, 테이블 랜덤 액세스가 성능에 미치는 영향을 정리해보자.

## Table Random Access

### 인덱스 ROWID는 물리적 주소? 논리적 주소?

아래는 인덱스를 이용해 테이블을 액세스하는 SQL 실행계획이다:

```
SQL> SELECT * FROM 고객 WHERE 지역 = '서울';

Execution Plan
----------------------------------------------------
0     SELECT STATEMENT Optimizer=ALL_ROWS
1  0    TABLE ACCESS BY INDEX ROWID OF '고객' (TABLE)
2  1      INDEX RANGE SCAN OF '고객_지역_IDX' (INDEX)
```

커버링 인덱스 등을 사용하는 경우가 아니라면, 인덱스를 스캔한 후에 반드시 테이블 액세스한다. 실행 계획에서 ‘TABLE ACCESS BY INDEX ROWID’라고 표시된 부분이 여기에 해당한다.

그림으로 표현하면 아래와 같다:

```
[그림 3-1]
```

인덱스를 스캔하는 이유는, 조건을 만족하는 소량의 데이터에 대한 ROWID를 얻기 위함이다. ROWID는 데이터파일 번호, 오브젝트 번호, 블록 번호 같은 물리적 요소로 구성되어 있기에 물리적 주소라고 할 수 있다. 하지만 ROWID는 물리적 주소보다는 논리적 주소에 더 가깝다. 물리적으로 직접 연결되는 것이 아닌, 레코드를 찾아내기 위한 논리적 주소 정보를 담고 있기 때문이다.

```
[그림 3-2]
```

### 메인 메모리 DB와 비교

오라클과 같은 RDBMS는 Disk(Storage)를 주로 사용하는 DB이다. 이와 다르게 메모리를 주로 사용하는 MMDB(Main Memory DB)도 존재한다. 그렇다면, RDMBS에서 버퍼 캐시 히트가 발생하여 메모리에서 레코드를 읽는 것과 MMDB에서 레코드를 읽는 것의 성능은 동일할까?

MMDB에서는 디스크에 저장된 데이터를 버퍼 캐시로 로딩하고 인덱스를 생성한다. 이때, 인덱스는 메모리상의 주소 정보, 즉 포인터를 갖는다. 따라서 인덱스 ROWID를 경유해 테이블을 액세스하는 오라클과 비교했을 때, MMDB가 액세스 비용이 훨씬 낮다.

여기서 기억해야 할 것은 일반적인 RDBMS에서 인덱스 ROWID를 이용한 테이블 액세스가 수행되고, 실제 주소값이 아니기에 생각보다 빠르지 않다는 점이다.

### I/O 메커니즘 복습

DBA(데이터파일번호 + 블록번호)는 디스크 상에서 블록을 찾기 위한 주소 정보다. 블록을 읽을 때는 디스크로 가기 전에 버퍼캐시부터 찾아본다. 

같은 블록이더라도 매번 다른 위치에 캐싱되는데, 실제 저장된 메모리 주소값을 버퍼 헤더가 갖고 있다. 해싱 알고리즘으로 버퍼 헤더를 우선 찾고, 헤더에서 얻은 포인터로 메모리 상의 버퍼 블록을 찾아간다.

인덱스로 테이블 블록에 액세스할 때에는 리프 블록에서 얻은 ROWID를 분해해서 DBA를 얻고, Table Full Scan 할 때는 인스텐트 맵을 통해 읽을 블록들의 DBA 정보를 얻는다.

```
[그림 3-3]
```

모든 데이터가 버퍼 캐시에 캐싱돼 있다고 하더라도 테이블 레코드를 찾기 위해 매번 DBA 해싱과 래치 획득 과정을 반복해야 한다. 동시 액세스가 심할 때는 캐시버퍼 체인 래치와 버퍼 Lock에 대한 경합도 발생한다. 이처럼 인덱스 ROWID를 이용한 테이블 액세스는 생각보다 비용이 많이 드는 구조다.

## 인덱스 클러스터링 팩터

클래스터링 팩터(Clustering Factor, ‘CF’)는 특정 컬럼을 기준으로 같은 값을 갖는 데이터가 서로 모여있는 정도를 의미한다. CF가 좋은 컬럼에 생성한 인덱스는 검색 효율이 매우 좋다.

다음은 인덱스 클러스터링 팩터가 가장 좋은 상태를 도식화한 것으로서, 인덱스 레코드 정렬 순서와 테이블 레코드 정렬 순서가 100% 일치하는 경우이다:

```
그림 3-4
```

반면, 아래는 인덱스 클러스터링 팩터가 가장 안 좋은 상태를 도식화한 것으로, 인덱스 레코드 정렬 순서와 테이블 레코드 정렬 순서가 전혀 일치하지 않는 경우이다:

```
그림 3-5
```

<aside>
💁‍♂️

**인덱스 클러스터링 팩터 효과**

CF가 좋은 컬럼에 생성한 인덱스는 검색 효율이 좋다고 했는데, 이는 테이블 액세스량에 비해 블록 I/O가 적게 발생함을 의미한다. 그런데 인덱스 레코드마다 테이블 레코드를 각각 블록 I/O하기에, CF가 달라도 블록 I/O 횟수에는 차이가 없어야 하는 것이 아닐까?

인덱스 ROWID로 테이블을 액세스할 때, 오라클은 래치 획득과 해시 체인 스캔 과정을 거쳐 어렵게 찾아간 테이블 블록에 대한 포인터(memory address)를 바로 해제하지 않고 일단 유지한다. 이를 ‘Buffer Pinning’이라고 한다.

이 상태에서, 다음 레코드를 읽었는데, 마침 ‘직전과 같은’ 테이블 블록을 가리킨다면, 래치 획득과 해시 체인 스캔 과정을 생략하고 바로 테이블 블록으로 접근할 수 있다.

아래 그림은 CF가 좋은 인덱스를 사용할 때, 테이블 액세스 횟수에 비해 블록 I/O가 적게 발생하는 이유를 도식화한 것이다:

```
그림 3-6
```

굵은 실선은 실제 블록 I/O가 발생하는 경우를, 가는 점선은 포인터로 바로 액세스하는 경우다.

</aside>

## 인덱스 손익분기점

```
그림 3-7
```

Index Range Scan에 의한 테이블 액세스가 Table Full Scan보다 느려지는 지점을 ‘인덱스 손익분기점’이라 부른다.

Table Full Scan은 1000만 건 중 한 건을 조회하든, 1000만 건을 다 조회하든 차이가 거의 없이 성능이 일정하다. 반면에 인덱스를 사용해 테이블 액세스 할 때는 몇 건을 추출하느냐에 따라 성능이 크게 달라진다. 당연히 추출 건수가 많을수록 느려진다. 이는 테이블 랜덤 액세스 때문이다.

인덱스를 이용한 테이블 액세스가 Table Full Scan보다 느릴 수 있는 핵심적인 두 가지 요인은 다음과 같다:

- Table Full Scan은 sequential access인 반면, 인덱스 ROWID를 이용한 테이블 액세스는 random access 방식이다.
- Table Full Scan은 Multiblock I/O인 반면, 인덱스 ROWID를 이용한 테이블 액세스는 Single Block I/O 방식이다.

물론, 이는 인덱스가 항상 좋지 않을 수 있다는 이야기일 뿐, 테이블 스캔이 항상 나쁜 것이 아니며, 인덱스 스캔이 항상 좋은 것도 아니라는 것을 염두에 두자.

## 인덱스 컬럼 추가

EMP 테이블에 PK 이외에 `(DEPTNO, JOB)` 순으로 구성한 `EMP_X01` 인덱스 하나만 있다고 가정하자:

```sql
SELECT /*+ index(emp emp_x01) */ *
FROM emp
WHERE deptno = 30
AND sal >= 2000
```

위 쿼리에 대해, 아래 그림을 보면 조건을 만족하는 사원이 단 한 명인데, 이를 찾기 위해 여섯 번의 테이블 액세스가 필요하다:

```sql
그림 3-9
```

인덱스 구성을 `(DEPTNO, SAL)`로 변경하면 좋겠지만, 실 운영 환경에서 이러한 방법은 적용하기 어렵다. 기존 인덱스를 사용하는 SQL도 있을 것이고, 과도한 인덱스 생성으로 이어질 수 있기 때문이다.

이럴 때, 아래 그림처럼 기존 인덱스에 `SAL` 컬럼을 추가하는 것만으로도 큰 효과를 얻을 수 있다. 인덱스 스캔량은 동일하지만, 테이블 랜덤 액세스 횟수가 줄어든다:

```sql
그림 3-10
```

## 인덱스만 읽고 처리

```sql
SELECT 부서번호, SUM(수량)
FROM 판매집계
WHERE 부서번호 LIKE '12%'
GROUP BY 부서번호;
```

위와 같은 쿼리에 대해서는 쿼리에 사용된 컬럼(`부서번호`, `수량`)을 모두 인덱스에 추가해서 테이블 액세스가 아예 발생하지 않게끔 하는 방법을 고려해 볼 수 있다. 이러한 쿼리를 ‘Covered 쿼리’라고 부르며, 그 쿼리에 사용한 인덱스를 ‘Covered(또는 Covering) 인덱스’라고 부른다.

## 인덱스 구조 테이블

랜덤 액세스가 아예 발생하지 않도록 테이블을 인덱스 구조로 생성하는 것도 가능하다. 오라클에서는 IOT(Index-Organized Table)라고 부른다. 참고로, MS-SQL Server는 ‘Clustered 인덱스’라고 부른다.

테이블을 찾아가기 위한 ROWID를 갖는 일반적인 인덱스와 달리 IOT는 그 자리에 테이블 데이터를 갖는다. 즉, 테이블 블록에 있어야 할 데이터를 리프 블록에 모두 저장한다.

```sql
그림 3-11
```

테이블을 인덱스 구조로 만드는 구문은 아래와 같다.

```sql
CREATE TABLE index_org_t (
    a NUMBER,
    b VARCHAR(10),
    CONSTRAINT index_org_t_pk PRIMARY KEY (a)
)
ORGANIZATION INDEX;
```

일반 테이블은 ‘힙 구조 테이블’이라고 부르는데, 이 테이블에 데이터를 입력할 때는 랜덤 방식을 사용한다. 반면, IOT는 인덱스 구조 테이블이므로 (PK를 기준으로) 정렬 상태를 유지하며 데이터가 입력된다.

IOT는 인위적으로 CF(Clustering Factor)를 좋게 만드는 방법 중 하나다. 같은 값을 가진 레코드들이 정렬된 상태로 모여 있으므로 랜덤 액세스가 아닌 시퀀셜 방식으로 데이터를 액세스한다. 이 때문에 `BETWEEN`이나 부등호 조건으로 넓은 범위를 읽을 때 유리하다.

## 클러스터 테이블

### 인덱스 클러스터 테이블

인덱스 클러스터 테이블은 아래 그림처럼 클러스터 키(여기에서의 `deptno`) 값이 같은 레코드를 한 블록에 모아서 저장하는 구조다. 한 블록에 모두 담을 수 없을 때는 새로운 블록을 할당해서 클러스터 체인으로 연결한다.

```sql
그림 3-12
```

오라클에서 클러스터는 키 값이 같은 데이터를 같은 공간(블록)에 저장해 둘 뿐, IOT나 SQL Server의 Clustered Index처럼 정렬하지는 않는다.

인덱스 클러스터 테이블을 구성하려면, 우선 아래와 같이 클러스터를 생성해야 한다:

```sql
CREATE CLUSTER c_dept# ( deptno NUMBER(2) ) INDEX;
```

그리고 클러스터에 테이블을 담기 전에 아래와 같이 클러스터 인덱스를 반드시 정의해야 한다. 클러스터 인덱스는 데이터 검색 뿐만 아니라 데이터 저장을 위한 위치를 찾는 경우에도 사용된다:

```sql
CREATE INDEX c_dept#_idx ON CLUSTER c_dept#;
```

클러스터 인덱스를 만들었다면 아래와 같이 클러스터 테이블을 생성한다:

```sql
CREATE TABLE dept (
	deptno NUMBER(2) NOT NULL,
	dname VARCHAR2(14) NOT NULL,
	loc VARCHAR2(13)
) CLUSTER c_dept#( deptno );
```

클러스터 인덱스도 일반 B*Tree 인덱스 구조를 사용하지만, 테이블 레코드를 일일이 가리키지 않고 해당 키 값을 저장하는 첫 번째 데이터 블록을 가리킨다는 점이 다르다. 아래 그림에서 보듯이 인덱스 레코드는 테이블 레코드와 1:M 관계를 갖는다.

```sql
그림 3-13
```

이러한 구조적 특성 때문에 인덱스를 스캔하며 값을 찾을 때는 랜덤 액세스가 값 하나당 한 번씩 밖에 발생하지 않는다. 클러스터에 도달해서는 시퀀셜 방식으로 스캔하기에 넓은 범위를 읽더라도 비효율이 없다.

클러스터 인덱스로 조회할 때, 실행 계획은 다음과 같다.

```
SQL> SELECT * FROM dept WHERE deptno = :deptno;

Execution Plan
----------------------------------------------------
0     SELECT STATEMENT Optimizer=ALL_ROWS
1  0    TABLE ACCESS (CLUSTER) OF 'DEPT' (CLUSTER)
2  1      INDEX (UNIQUE SCAN) OF 'C_DEPT#_IDX' (INDEX (CLUSTER))
```

### 해시 클러스터 테이블

해시 클러스터는 인덱스를 사용하지 않고, 아래 그림처럼 해시 알고리즘을 사용해 클러스터를 찾아간다는 점만 다르다.

```
그림 3-14
```

해시 클러스터 테이블을 구성하려면 우선 아래와 같이 클러스터를 생성한다:

```sql
CREATE cluster c_dept# ( deptno NUMBER(2) ) HASHKEYS 4;
```

그리고 클러스터 테이블을 생성한다:

```sql
CREATE TABLE dept(
	deptno NUMBER(2) NOT NULL,
	dname VARCHAR2(14) NOT NULL,
	loc VARCHAR2(13)
) CLUSTER c_dept#( deptno );
```

해시 클러스터를 조회할 때, 실행계획은 아래와 같다.

```
SQL> SELECT * FROM dept WHERE deptno = :deptno;

Execution Plan
----------------------------------------------------
0     SELECT STATEMENT Optimizer=ALL_ROWS
1  0    TABLE ACCESS (HASH) OF 'DEPT' (CLUSTER (HASH))
```

---

# 부분범위 처리 활용

## 부분범위 처리

```java
private void execute(Connection con) throws Exception {
		Statement stmt = con.createStatement();
		ResultSet rs = stmt.executeQuery("SELECT name FROM big_table");
	
		for (int i = 0; i < 100; i++) {
				if (rs.next()) {
						System.out.println(rs.getString(1));
				}
		}
		
		rs.close();
		stmt.close();
}
```

위와 같은 코드에 대해, SQL문에 사용한 `BIG_TABLE`이 1억 건에 이르는 대용량 테이블이어도 실행 결과는 버튼을 클릭하자마자 곧바로 화면에 출력된다.

이것이 가능한 이유는, DBMS가 데이터를 모두 읽어 한 번에 전송하지 않고, 먼저 읽는 데이터부터 일정량(Array Size)을 전송하고 멈추기 때문이다. 데이터를 전송하고 프로세스는 CPU를 OS에 반환하고 대기 큐에서 잠을 잔다. 다음 Fetch Call을 받으면 대기 큐에서 나와 그 다음 데이터부터 일정량을 읽어 전송하고 또 다시 잠을 잔다.

이처럼 전체 쿼리 결과 집합을 Fetch Call이 있을 때마다 일정량씩 나누어 전송하는 것을 ‘부분범위 처리’라 한다.

Array Size는 클라이언트 프로그램에서 설정하며, Array Size가 10인 상태에서 위 Java 프로그램이 데이터를 읽는 과정은 아래와 같다.

1. 최초 `rs.next()` 호출 시, Fetch Call을 통해 DB 서버로부터 전송받은 데이터 10건을 클라이언트 캐시에 저장한다.
2. 이후 `rs.next()`를 호출할 때는 Fetch Call을 발생시키지 않고 캐시에서 데이터를 읽는다.
3. 캐시에 저장한 데이터를 모두 소진하면, 이후 `rs.next()` 호출 시 추가 Fetch Call을 통해 10건을 전송받는다.
4. 100건을 다 읽을 때까지 2~3번 과정을 반복한다.

쿼리 수행 시, 결과 집합을 버퍼 캐시에 모두 적재하는 것이 아니다. Fetch Call이 발생할 때마다 일정량씩 전송하여 캐싱하는 것이다.

### 정렬 조건이 있을 때 부분범위 처리

```java
Statement stmt = con.createStatement();
ResultSet rs = stmt.executeQuery("SELECT name FROM big_table ORDER BY created");
```

위처럼 정렬 조건을 추가하면 DB 서버는 ‘모든’ 데이터를 읽어 정렬을 마친 후 클라이언트에게 전송을 시작한다. 즉, 부분범위 처리가 작동하지 않는다. Sort Area와 Temp 테이블스페이스까지 이용해 데이터 정렬을 마치고 나면 그때부터 일정량씩 나눠 클라이언트에게 데이터를 전송한다.

만약 `created` 컬럼이 선두인 인덱스가 있다면, 부분범위 처리가 가능하다. 인덱스는 항상 정렬된 상태를 유지하기 때문이다.

### Array Size 조정을 통한 Fetch Call 최소화

대량 데이터를 필요로 하는 경우라면 Array Size를 크게 하는 것이 Fetch Call 횟수를 줄일 수 있어 좋을 것이다. 반대로, 앞쪽 일부 데이터만 필요한 경우라면 Array Size를 작게 설정하는 것이 유리하다.

## 부분범위 처리 구현

아래는 부분범위 처리를 활용하지 않은 예시다.

```java
public class AllRange {
		public static void execute(Connection con) throws Exception {
				int arraySize = 10;
				String sql = "SELECT object_id, object_name FROM all_objects";
				Statement stmt = con.createStatement();
				stmt.setFetchSize(arraySize);
				ResultSet rs = stmt.executeQuery(sql);
				while (rs.next()) {
						System.out.println(rs.getLong(1) + " : " + rs.getString(2));
				}
				rs.close();
				stmt.close();
		}
		
		public void main(String[] args) throws Exception {
				Connection con = getConnection();
				execute(con);
				releaseConnection(con);
		}
}
```

아래는 부분범위 처리를 활용한 예시이다. 출력 레코드 수가 Array Size에 도달하면 멈췄다가 사용자 요청이 있을 때 다시 데이터를 Fetch하는 부분이 핵심이다.

```java
import java.io.*;
import java.lang.*;
import java.util.*;
import java.net.*;
import java.sql.*;
import oracle.jdbc.driver.*;

public class PartialRange {
		public static int fetch(ResultSet rs, int arraySize) throws Exception {
				int i = 0;
				while (rs.next()) {
						System.out.println(rs.getLong(1) + " : " + rs.getString(2));
						if (++i >= arraySize) {
								return i;
						}
				}
				return i;
		}
		
		public static void execute(Connection con) throws Exception {
				int arraySize = 10;
				String sql = "SELECT object_id, object_name from all_objects";
				Statement stmt = con.createStatement();
				stmt.setFetchSize(arraySize);
				ResultSet rs = stmt.executeQuery(sql);
				while (true) {
						int r = fetch(rs, arraySize);
						if (r < arraySize)
								break
						System.out.println("Enter to Continue ... (Q)uit? ");
						BufferedReader in = new BufferedReader(new InputStreamReader(System.in));
						String input = in.readLine();
						if (input.equals("Q"))
								break;
				}
				rs.close();
				stmt.close();
		} 
			
		public void main(String[] args) throws Exception {
				Connection con = getConnection();
				execute(con);
				releaseConnection(con);
		}
}
```

## OLTP 환경에서 부분범위 처리에 의한 성능개선 원리

OLTP(Online Transaction Processing) 환경에서는 일반적으로 소량 데이터를 읽고 갱신한다. 은행 계좌 입출금 내역 조회, 게시판 조회 등의 경우 많은 데이터 중 일부 데이터만을 확인한다. 그럴 때, 항상 정렬 상태를 유지하는 인덱스를 이용하면, 정렬 작업을 생략하고 앞쪽 일부 데이터를 아주 빠르게 보여줄 수 있다.

```sql
SELECT 게시글ID, 제목, 작성자, 등록일시
FROM 게시판
WHERE 게시판구분코드 = 'A'
ORDER BY 등록일시 DESC
```

위 쿼리에 대해 인덱스를 `(게시판구분코드, 등록일시)` 순으로 구성하면 Sort Order By 연산을 생략할 수 있다.

```
Execution Plan
----------------------------------------------------
0     SELECT STATEMENT
1  0    TABLE ACCESS BY INDEX ROWID
2  1      INDEX RANGE SCAN DESCENDING
```

이렇듯 정렬 상태를 유지하는 인덱스의 특성을 이용하면, `게시판구분코드 = 'A'` 조건을 만족하는 전체 row를 읽지 않고도 바로 결과집합 출력을 시작할 수 있다.
