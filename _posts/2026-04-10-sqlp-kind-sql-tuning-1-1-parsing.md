---
title: 친절한 SQL 튜닝
date: 2026-04-10
categories: [SQLP]
tags: [SQLP, 친절한 SQL 튜닝, SQL 파싱과 최적화]
---

## 친절한 SQL 튜닝
1. SQL 처리 과정과 I/O
1.1 SQL 파싱과 최적화

[친절한 SQL 튜닝](https://product.kyobobook.co.kr/detail/S000001975837)
---

### 인덱스, 옵티마이저 힌트 실습 준비

```sql
-- 테스트용 테이블 생성
SELECT TABLE T
AS
SELECT D.NO, E.*
  FROM SELECT EMP E,
       (SELECT ROWNUM NO FROM DUAL CONNECT BY LEVEL <= 1000) D;

-- 인덱스 생성
CREATE INDEX T_X01 ON T(DEPTNO, NO);
CREATE INDEX T_X02 ON T(DEPTNO, JOB, NO);
```

### 1. NO HINT

```sql
EXPLAIN PLAN FOR
SELECT *
FROM T
WHERE DEPTNO = 10
 AND NO =1;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

PLAN_TABLE_OUTPUT                                                                            |
---------------------------------------------------------------------------------------------+
Plan hash value: 2369825647                                                                  |
                                                                                             |
---------------------------------------------------------------------------------------------|
| Id  | Operation                           | Name  | Rows  | Bytes | Cost (%CPU)| Time     ||
---------------------------------------------------------------------------------------------|
|   0 | SELECT STATEMENT                    |       |     5 |   210 |     2   (0)| 00:00:01 ||
|   1 |  TABLE ACCESS BY INDEX ROWID BATCHED| T     |     5 |   210 |     2   (0)| 00:00:01 ||
|*  2 |   INDEX RANGE SCAN                  | T_X01 |     5 |       |     1   (0)| 00:00:01 ||
---------------------------------------------------------------------------------------------|
                                                                                             |
Predicate Information (identified by operation id):                                          |
---------------------------------------------------                                          |
                                                                                             |
   2 - access("DEPTNO"=10 AND "NO"=1)                                                        |
```

### 2. HINT사용 - INDEX

```sql
EXPLAIN PLAN FOR
SELECT /*+ INDEX(T T_X02) */ *
  FROM T
 WHERE DEPTNO = 10
   AND NO = 1;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

PLAN_TABLE_OUTPUT                                                                            |
---------------------------------------------------------------------------------------------+
Plan hash value: 1588293685                                                                  |
                                                                                             |
---------------------------------------------------------------------------------------------|
| Id  | Operation                           | Name  | Rows  | Bytes | Cost (%CPU)| Time     ||
---------------------------------------------------------------------------------------------|
|   0 | SELECT STATEMENT                    |       |     5 |   210 |     4   (0)| 00:00:01 ||
|   1 |  TABLE ACCESS BY INDEX ROWID BATCHED| T     |     5 |   210 |     4   (0)| 00:00:01 ||
|*  2 |   INDEX SKIP SCAN                   | T_X02 |     5 |       |     3   (0)| 00:00:01 ||
---------------------------------------------------------------------------------------------|
                                                                                             |
Predicate Information (identified by operation id):                                          |
---------------------------------------------------                                          |
                                                                                             |
   2 - access("DEPTNO"=10 AND "NO"=1)                                                        |
       filter("NO"=1)                                                                        |
```

### 3. HINT 사용 - FULL
```sql
EXPLAIN PLAN FOR
SELECT /*+ FULL(T) */ *
  FROM T
 WHERE DEPTNO = 10
   AND NO = 1;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

PLAN_TABLE_OUTPUT                                                         |
--------------------------------------------------------------------------+
Plan hash value: 1601196873                                               |
                                                                          |
--------------------------------------------------------------------------|
| Id  | Operation         | Name | Rows  | Bytes | Cost (%CPU)| Time     ||
--------------------------------------------------------------------------|
|   0 | SELECT STATEMENT  |      |     5 |   210 |    29   (0)| 00:00:01 ||
|*  1 |  TABLE ACCESS FULL| T    |     5 |   210 |    29   (0)| 00:00:01 ||
--------------------------------------------------------------------------|
                                                                          |
Predicate Information (identified by operation id):                       |
---------------------------------------------------                       |
                                                                          |
   1 - filter("NO"=1 AND "DEPTNO"=10)                                     |
```