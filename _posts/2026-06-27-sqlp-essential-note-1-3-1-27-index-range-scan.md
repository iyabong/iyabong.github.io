---
title: 국가공인 SQLP 자격검정 핵심노트 1
date: 2026-06-27
categories: [SQLP]
tags: [SQLP, 인덱스 튜닝, 인덱스 기본 원리, Index Range Scan]
---

[2024 국가공인 SQLP 자격검정 핵심노트 1](https://product.kyobobook.co.kr/detail/S000213913597)

- 인덱스 튜닝
  - 인덱스 기본 원리
    - ** 27. Index Range Scan 유도**

```sql
-- 실행계획 수집
EXPLAIN PLAN FOR

-- 실행계획 출력
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY())
```

#### 0. 실습 준비
```sql
-- COMPANY
CREATE TABLE COMPANY (
    COMP_CODE VARCHAR2(10) PRIMARY KEY,
    COMP_NAME VARCHAR2(100) NOT NULL,
    TEL_NO    VARCHAR2(20)
);
CREATE INDEX IX_COMPANY_NAME ON COMPANY(COMP_NAME);
INSERT INTO COMPANY VALUES ('C001', 'Daehan Logistics', '02-123-4567');
INSERT INTO COMPANY VALUES ('C002', 'Korean Air', '02-987-6543');
INSERT INTO COMPANY VALUES ('C003', 'Minkuk Corp', '031-111-2222');
INSERT INTO COMPANY VALUES ('C004', 'Hanjin Shipping', '02-333-4444');

-- EMPLOYEE
CREATE TABLE EMPLOYEE (
    EMP_NO         NUMBER PRIMARY KEY,
    EMP_NAME       VARCHAR2(50) NOT NULL,
    MONTHLY_SALARY NUMBER NOT NULL,
    DEPT_CODE      VARCHAR2(10)
);
CREATE INDEX IX_EMP_SALARY ON EMPLOYEE(MONTHLY_SALARY);
INSERT INTO EMPLOYEE VALUES (1001, 'Kim', 2500000, 'D01');
INSERT INTO EMPLOYEE VALUES (1002, 'Lee', 3000000, 'D02');
INSERT INTO EMPLOYEE VALUES (1003, 'Park', 4000000, 'D01');
INSERT INTO EMPLOYEE VALUES (1004, 'Choi', 2000000, 'D03');

-- ORDERS
CREATE TABLE ORDERS (
    ORDER_NO   NUMBER PRIMARY KEY,
    ORDER_QTY  NUMBER,
    ORDER_DT   DATE NOT NULL,
    DC_RATE    NUMBER(5,2)
);
CREATE INDEX IX_ORDERS_QTY ON ORDERS(ORDER_QTY);
CREATE INDEX IX_ORDERS_DT  ON ORDERS(ORDER_DT);
CREATE INDEX IX_ORDERS_DC  ON ORDERS(DC_RATE);
INSERT INTO ORDERS VALUES (1, 150,  TO_DATE('2026-06-27 10:00:00', 'YYYY-MM-DD HH24:MI:SS'), 2.5);
INSERT INTO ORDERS VALUES (2, NULL, TO_DATE('2026-06-27 11:30:00', 'YYYY-MM-DD HH24:MI:SS'), 1.2);
INSERT INTO ORDERS VALUES (3, 80,   TO_DATE('2026-06-27 15:00:00', 'YYYY-MM-DD HH24:MI:SS'), 3.0);
INSERT INTO ORDERS VALUES (4, 200,  TO_DATE('2026-06-28 09:15:00', 'YYYY-MM-DD HH24:MI:SS'), 0.5);
```

#### 실습 1
```sql
-- 튜닝 전
SELECT *
FROM COMPANY
WHERE SUBSTR(COMP_NAME,1,6) = 'Daehan'
-----------------------------------------------------------------------------
| Id  | Operation         | Name    | Rows  | Bytes | Cost (%CPU)| Time     |
-----------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |         |     1 |    71 |     3   (0)| 00:00:01 |
|*  1 |  TABLE ACCESS FULL| COMPANY |     1 |    71 |     3   (0)| 00:00:01 |
-----------------------------------------------------------------------------

-- 튜닝 후
SELECT /*+ INDEX(COMPANY) */ *
FROM COMPANY
WHERE COMP_NAME LIKE 'Daehan%'
-------------------------------------------------------------------------------------------------------
| Id  | Operation                           | Name            | Rows  | Bytes | Cost (%CPU)| Time     |
-------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                    |                 |     1 |    71 |    12   (0)| 00:00:01 |
|   1 |  TABLE ACCESS BY INDEX ROWID BATCHED| COMPANY         |     1 |    71 |    12   (0)| 00:00:01 |
|*  2 |   INDEX RANGE SCAN                  | IX_COMPANY_NAME |     1 |       |     2   (0)| 00:00:01 |
-------------------------------------------------------------------------------------------------------
```

#### 실습 2
```sql
-- 튜닝 전
SELECT *
FROM EMPLOYEE
WHERE MONTHLY_SALARY
  AND MONTHLY_SALARY * 12 >= 36000000;
------------------------------------------------------------------------------
| Id  | Operation         | Name     | Rows  | Bytes | Cost (%CPU)| Time     |
------------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |          |     1 |    16 |     3   (0)| 00:00:01 |
|*  1 |  TABLE ACCESS FULL| EMPLOYEE |     1 |    16 |     3   (0)| 00:00:01 |
------------------------------------------------------------------------------

-- 튜닝 후
EXPLAIN PLAN FOR
SELECT *
FROM EMPLOYEE
WHERE MONTHLY_SALARY
  AND MONTHLY_SALARY >= 36000000 / 12;
-----------------------------------------------------------------------------------------------------
| Id  | Operation                           | Name          | Rows  | Bytes | Cost (%CPU)| Time     |
-----------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                    |               |     1 |    16 |     2   (0)| 00:00:01 |
|   1 |  TABLE ACCESS BY INDEX ROWID BATCHED| EMPLOYEE      |     1 |    16 |     2   (0)| 00:00:01 |
|*  2 |   INDEX RANGE SCAN                  | IX_EMP_SALARY |     1 |       |     1   (0)| 00:00:01 |
-----------------------------------------------------------------------------------------------------
```

### 실습 3
```sql
-- 튜닝 전
SELECT *
FROM ORDERS
WHERE NVL(ORDER_QTY, 0) >= 100;
----------------------------------------------------------------------------
| Id  | Operation         | Name   | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |        |     2 |    36 |     3   (0)| 00:00:01 |
|*  1 |  TABLE ACCESS FULL| ORDERS |     2 |    36 |     3   (0)| 00:00:01 |
----------------------------------------------------------------------------

-- 튜닝 후
SELECT *
FROM ORDERS
WHERE ORDER_QTY >= 100;
-----------------------------------------------------------------------------------------------------
| Id  | Operation                           | Name          | Rows  | Bytes | Cost (%CPU)| Time     |
-----------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                    |               |     2 |    36 |     2   (0)| 00:00:01 |
|   1 |  TABLE ACCESS BY INDEX ROWID BATCHED| ORDERS        |     2 |    36 |     2   (0)| 00:00:01 |
|*  2 |   INDEX RANGE SCAN                  | IX_ORDERS_QTY |     2 |       |     1   (0)| 00:00:01 |
-----------------------------------------------------------------------------------------------------
```

### 실습 4
```sql
-- 튜닝 전
SELECT *
FROM ORDERS
WHERE TO_CHAR(ORDER_DT, 'YYYYMMDD') = :dt -- :dt ← '20260627'
----------------------------------------------------------------------------
| Id  | Operation         | Name   | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |        |     1 |    18 |     3   (0)| 00:00:01 |
|*  1 |  TABLE ACCESS FULL| ORDERS |     1 |    18 |     3   (0)| 00:00:01 |
----------------------------------------------------------------------------

-- 튜닝 후
SELECT *
FROM ORDERS
WHERE ORDER_DT >= TRUNC(TO_DATE(:dt, 'YYYYMMDD'))		  -- :dt ← '20260627'
  AND ORDER_DT <  TRUNC(TO_DATE(:dt, 'YYYYMMDD')) + 1
----------------------------------------------------------------------------------------------------
| Id  | Operation                           | Name         | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                    |              |     3 |    54 |     2   (0)| 00:00:01 |
|   1 |  TABLE ACCESS BY INDEX ROWID BATCHED| ORDERS       |     3 |    54 |     2   (0)| 00:00:01 |
|*  2 |   INDEX RANGE SCAN                  | IX_ORDERS_DT |     3 |       |     1   (0)| 00:00:01 |
----------------------------------------------------------------------------------------------------
```

### 실습 5
```sql
-- 튜닝 전
SELECT DC_RATE, FLOOR(DC_RATE), CEIL(DC_RATE)
FROM ORDERS
WHERE FLOOR(DC_RATE) < :dcrt -- :dcrt ← 1.1
----------------------------------------------------------------------------
| Id  | Operation         | Name   | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |        |     1 |     4 |     3   (0)| 00:00:01 |
|*  1 |  TABLE ACCESS FULL| ORDERS |     1 |     4 |     3   (0)| 00:00:01 |
----------------------------------------------------------------------------

-- 튜닝 후
SELECT DC_RATE, FLOOR(DC_RATE), CEIL(DC_RATE)
FROM ORDERS
WHERE DC_RATE < CEIL(:dcrt) -- :dcrt ← 1.1
---------------------------------------------------------------------------------
| Id  | Operation        | Name         | Rows  | Bytes | Cost (%CPU)| Time     |
---------------------------------------------------------------------------------
|   0 | SELECT STATEMENT |              |     3 |    12 |     1   (0)| 00:00:01 |
|*  1 |  INDEX RANGE SCAN| IX_ORDERS_DC |     3 |    12 |     1   (0)| 00:00:01 |
---------------------------------------------------------------------------------
```
