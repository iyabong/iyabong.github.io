---
title: 친절한 SQL 튜닝 - 8.4 DBMS_XPLAN 패키지
date: 2026-05-03
categories: [SQLP]
tags: [SQLP, 친절한 SQL 튜닝, 부록, SQL 분석 도구, 실행계획]
---

# 실습개요
- 예상 실행계획과 실제 실행계획 비교
- 실제 실행계획 확인방법
- 실제 실행계획 분석


# 친절한 SQL 튜닝
8 SQL 분석 도구
  4 DBMS_XPLAN 패키지
[친절한 SQL 튜닝](https://product.kyobobook.co.kr/detail/S000001975837)
---

### 0. 실습 준비
```sql
-- 1. 대용량 테이블 생성
CREATE TABLE EMP_BIG (
    EMPNO   NUMBER(10),
    ENAME   VARCHAR2(20),
    JOB     VARCHAR2(10),
    DEPTNO  NUMBER(5)
);

-- 2. 대량 데이터 삽입 (10만건)
INSERT INTO EMP_BIG (EMPNO, ENAME, JOB, DEPTNO)
SELECT 
    ROWNUM,
    'EMP' || ROWNUM,
    CASE MOD(ROWNUM, 3) 
        WHEN 0 THEN 'CLERK'
        WHEN 1 THEN 'MANAGER'
        ELSE 'ANALYST'
    END,
    CASE MOD(ROWNUM, 3)
        WHEN 0 THEN 20
        WHEN 1 THEN 30
        ELSE 20
    END
FROM DUAL
CONNECT BY LEVEL <= 100000;

-- 3. 인덱스 생성
CREATE INDEX EMP_BIG_X01 ON EMP_BIG(DEPTNO, JOB, EMPNO);

-- 4. ADMIN 계정으로 접속 후, 실습 계정(SQLP)에 권한 부여
GRANT SELECT ON V$SQL_PLAN_STATISTICS_ALL TO SQLP;
GRANT SELECT ON V$SQL TO SQLP;
GRANT SELECT ON V$SESSION TO SQLP;
GRANT SELECT ON V$PARAMETER TO SQLP;
```

### 1. 예상 실행계획
실제 수행 결과가 아닌 예상 값

```sql
-- 1. 실행계획 수집
EXPLAIN PLAN FOR
SELECT /*+ INDEX(E EMP_BIG_X01) */ *
FROM EMP_BIG E
WHERE DEPTNO = 20
  AND JOB = 'CLERK'
ORDER BY EMPNO;

-- 2. 실행계획 확인
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

PLAN_TABLE_OUTPUT                                                                          |
-------------------------------------------------------------------------------------------+
Plan hash value: 2894996259                                                                |
                                                                                           |
-------------------------------------------------------------------------------------------|
| Id  | Operation                   | Name        | Rows  | Bytes | Cost (%CPU)| Time     ||
-------------------------------------------------------------------------------------------|
|   0 | SELECT STATEMENT            |             | 22222 |   520K|   364   (0)| 00:00:01 ||
|   1 |  TABLE ACCESS BY INDEX ROWID| EMP_BIG     | 22222 |   520K|   364   (0)| 00:00:01 ||
|*  2 |   INDEX RANGE SCAN          | EMP_BIG_X01 | 22222 |       |    89   (0)| 00:00:01 ||
-------------------------------------------------------------------------------------------|
                                                                                           |
Predicate Information (identified by operation id):                                        |
---------------------------------------------------                                        |
                                                                                           |
   2 - access("DEPTNO"=20 AND "JOB"='CLERK')                                               |
```

### 2. 실제 실행계획
예상 값과 실제 수행 결과 데이터 확인 가능
- E-Rows: 예상 행 수
- A-Rows: 실제 행 수
- A-Time: 실제 수행 시간
- Buffers: 읽은 블록 수(논리적I/O)

```sql
-- 1. 실행계획 수집
SELECT /*+ INDEX(E EMP_BIG_X01) GATHER_PLAN_STATISTICS */ *
FROM EMP_BIG E
WHERE DEPTNO = 20
  AND JOB = 'CLERK'
ORDER BY EMPNO;

-- 2. 실행계획 수집된 SQL_ID 조회
SELECT SQL_ID
FROM V$SQL 
WHERE SQL_TEXT LIKE 'SELECT /*+ INDEX(E EMP_BIG_X01) GATHER_PLAN_STATISTICS */%'
AND SQL_TEXT NOT LIKE '%V$SQL%'

SQL_ID       |
-------------+
bmxnx98v0j9ar|

-- 3. 실행계획 확인
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR('bmxnx98v0j9ar',NULL,'ALLSTATS LAST'));

PLAN_TABLE_OUTPUT                                                                                    |
-----------------------------------------------------------------------------------------------------+
SQL_ID  bmxnx98v0j9ar, child number 0                                                                |
-------------------------------------                                                                |
SELECT /*+ INDEX(E EMP_BIG_X01) GATHER_PLAN_STATISTICS */ * FROM                                     |
EMP_BIG E WHERE DEPTNO = 20   AND JOB = 'CLERK' ORDER BY EMPNO                                       |
                                                                                                     |
Plan hash value: 2894996259                                                                          |
                                                                                                     |
-----------------------------------------------------------------------------------------------------|
| Id  | Operation                   | Name        | Starts | E-Rows | A-Rows |   A-Time   | Buffers ||
-----------------------------------------------------------------------------------------------------|
|   0 | SELECT STATEMENT            |             |      1 |        |  33333 |00:00:00.04 |    7159 ||
|   1 |  TABLE ACCESS BY INDEX ROWID| EMP_BIG     |      1 |  22222 |  33333 |00:00:00.04 |    7159 ||
|*  2 |   INDEX RANGE SCAN          | EMP_BIG_X01 |      1 |  22222 |  33333 |00:00:00.02 |    3457 ||
-----------------------------------------------------------------------------------------------------|
                                                                                                     |
Predicate Information (identified by operation id):                                                  |
---------------------------------------------------                                                  |
                                                                                                     |
   2 - access("DEPTNO"=20 AND "JOB"='CLERK')                                                         |
                                                                                                     |
```