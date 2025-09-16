# Oracle을 Docker로 빠르게 실행해보기

## 1. Oracle Docker 컨테이너 실행

```
docker run -d --name oracle -p 1521:1521 -e ORACLE_PASSWORD=oracle container-registry.oracle.com/database/free:latest
docker exec -it oracle bash

sqlplus sys/oracle@localhost:1521/FREEPDB1 as sysdba
GRANT SELECT ON DBA_EXTENTS TO study01;
GRANT SELECT ON DBA_SEGMENTS TO study01;
GRANT SELECT ON DBA_TABLESPACES TO study01;
```

## 2. CDB 연결 (system 사용자)

- Host: localhost
- Port: 1521
- Connection type: Service Name
- Service name: FREE
- Username: system
- Password: oracle

## 3. 사용자 생성 (system으로 접속 후)

```sql
ALTER SESSION SET CONTAINER=FREEPDB1;
CREATE USER study01 IDENTIFIED BY "study01!";
GRANT CONNECT, RESOURCE TO study01;
GRANT UNLIMITED TABLESPACE TO study01;
GRANT SELECT ON DBA_EXTENTS TO study01;
GRANT SELECT ON DBA_SEGMENTS TO study01;
GRANT SELECT ON DBA_TABLESPACES TO study01;
GRANT SELECT ON SYS.V_$PARAMETER TO study01;
```

## 4. PDB 연결 (일반 사용자)

- Host: localhost
- Port: 1521
- Connection type: Service Name
- Service name: FREEPDB1
- Username: study01
- Password: study01!
