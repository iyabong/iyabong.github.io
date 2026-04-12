---
title: 친절한 SQL 튜닝 - 2.1 인덱스 구조 및 탐색
date: 2026-04-12
categories: [SQLP]
tags: [SQLP, 친절한 SQL 튜닝, 인덱스 구조 및 탐색]
---

## 친절한 SQL 튜닝
2 인덱스 기본
  2.1 인덱스 구조 및 탐색

[친절한 SQL 튜닝](https://product.kyobobook.co.kr/detail/S000001975837)
---

### 1. 실습 인덱스 만들기

```sql
-- 고객 테이블
CREATE TABLE CUSTOMER (
    CUST_ID   NUMBER,
    CUST_NAME VARCHAR2(50),
    GENDER    VARCHAR2(1),   -- M/F
    AGE       NUMBER,
    REGION    VARCHAR2(20)
);

-- 주문 테이블
CREATE TABLE ORDERS (
    ORDER_ID   NUMBER,
    CUST_ID    NUMBER,
    ORDER_DT   DATE,
    PRODUCT    VARCHAR2(50),
    AMOUNT     NUMBER,
    STATUS     VARCHAR2(10)  -- COMPLETE/CANCEL/PENDING
);

-- 데이터 넣기
INSERT INTO CUSTOMER
SELECT ROWNUM,
       DBMS_RANDOM.STRING('A', 5),
       CASE MOD(ROWNUM, 2) WHEN 0 THEN 'M' ELSE 'F' END,
       MOD(ROWNUM, 50) + 20,
       CASE MOD(ROWNUM, 5)
           WHEN 0 THEN '서울'
           WHEN 1 THEN '부산'
           WHEN 2 THEN '대구'
           WHEN 3 THEN '인천'
           ELSE '광주'
       END
FROM DUAL CONNECT BY ROWNUM <= 10000;

INSERT INTO ORDERS
SELECT ROWNUM,
       MOD(ROWNUM, 10000) + 1,
       SYSDATE - MOD(ROWNUM, 365),
       DBMS_RANDOM.STRING('A', 10),
       MOD(ROWNUM, 1000) * 100,
       CASE MOD(ROWNUM, 3)
           WHEN 0 THEN 'COMPLETE'
           WHEN 1 THEN 'CANCEL'
           ELSE 'PENDING'
       END
FROM DUAL CONNECT BY ROWNUM <= 100000;

COMMIT;

-- 결합 인덱스 생성
CREATE INDEX IDX_CUST_GENDER_NAME ON CUSTOMER(GENDER, CUST_NAME);
```

### 2-1. WHERE절에 선두 컬럼 있음
```sql
EXPLAIN PLAN FOR
SELECT *
  FROM CUSTOMER 
 WHERE GENDER = 'M'; -- 선두컬럼
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

PLAN_TABLE_OUTPUT                                                             |
------------------------------------------------------------------------------+
Plan hash value: 2844954298                                                   |
                                                                              |
------------------------------------------------------------------------------|
| Id  | Operation         | Name     | Rows  | Bytes | Cost (%CPU)| Time     ||
------------------------------------------------------------------------------|
|   0 | SELECT STATEMENT  |          |  5000 |   107K|    13   (0)| 00:00:01 ||
|*  1 |  TABLE ACCESS FULL| CUSTOMER |  5000 |   107K|    13   (0)| 00:00:01 ||
------------------------------------------------------------------------------|
                                                                              |
Predicate Information (identified by operation id):                           |
---------------------------------------------------                           |
                                                                              |
   1 - filter("GENDER"='M')                                                   |
```
- FULL SCAN 이유는 선택도(50%)를 옵티마이저가 고려한 것으로 보임.


### 2-2. WHERE절에 선두 컬럼 없는 경우

```sql
EXPLAIN PLAN FOR
SELECT * 
  FROM CUSTOMER
 WHERE CUST_NAME = 'ABCDE'; -- 선두(GENDER) 없이 후행(CUST_NAME)만 조건
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

PLAN_TABLE_OUTPUT                                                                                           |
------------------------------------------------------------------------------------------------------------+
Plan hash value: 3142539142                                                                                 |
                                                                                                            |
------------------------------------------------------------------------------------------------------------|
| Id  | Operation                           | Name                 | Rows  | Bytes | Cost (%CPU)| Time     ||
------------------------------------------------------------------------------------------------------------|
|   0 | SELECT STATEMENT                    |                      |     1 |    22 |     4   (0)| 00:00:01 ||
|   1 |  TABLE ACCESS BY INDEX ROWID BATCHED| CUSTOMER             |     1 |    22 |     4   (0)| 00:00:01 ||
|*  2 |   INDEX SKIP SCAN                   | IDX_CUST_GENDER_NAME |     1 |       |     3   (0)| 00:00:01 ||
------------------------------------------------------------------------------------------------------------|
                                                                                                            |
Predicate Information (identified by operation id):                                                         |
---------------------------------------------------                                                         |
                                                                                                            |
   2 - access("CUST_NAME"='ABCDE')                                                                          |
       filter("CUST_NAME"='ABCDE')                                                                          |
```
인덱스: (GENDER, CUST_NAME)

[INDEX SKIP SCAN]
선두 컬럼 없이 후행 컬럼만 조건으로 줬을 때

GENDER = 'F' 구간에서 CUST_NAME = 'ABCDE' 찾기
GENDER = 'M' 구간에서 CUST_NAME = 'ABCDE' 찾기

※ 선두 컬럼의 값 종류가 적을 때(M/F 2개) 옵티마이저가 SKIP SCAN을 선택

[Predicate]
access("CUST_NAME"='ABCDE')   ← 인덱스로 접근
filter("CUST_NAME"='ABCDE')   ← 추가로 필터링

※ SKIP SCAN 특징: access + filter 둘 다 있음.


### 2-3. WHERE절에 선두, 후행 컬럼 모두 있는 경우
```sql
EXPLAIN PLAN FOR
SELECT * 
  FROM CUSTOMER
 WHERE GENDER = 'M'         -- 선행 컬럼
   AND CUST_NAME = 'ABCDE'; -- 후행 컬럼
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

PLAN_TABLE_OUTPUT                                                                                           |
------------------------------------------------------------------------------------------------------------+
Plan hash value: 4085076058                                                                                 |
                                                                                                            |
------------------------------------------------------------------------------------------------------------|
| Id  | Operation                           | Name                 | Rows  | Bytes | Cost (%CPU)| Time     ||
------------------------------------------------------------------------------------------------------------|
|   0 | SELECT STATEMENT                    |                      |     1 |    22 |     2   (0)| 00:00:01 ||
|   1 |  TABLE ACCESS BY INDEX ROWID BATCHED| CUSTOMER             |     1 |    22 |     2   (0)| 00:00:01 ||
|*  2 |   INDEX RANGE SCAN                  | IDX_CUST_GENDER_NAME |     1 |       |     1   (0)| 00:00:01 ||
------------------------------------------------------------------------------------------------------------|
                                                                                                            |
Predicate Information (identified by operation id):                                                         |
---------------------------------------------------                                                         |
                                                                                                            |
   2 - access("GENDER"='M' AND "CUST_NAME"='ABCDE')                                                         |
```
GENDER + NAME  → 1건 (0.001%)

※ 두 컬럼 조합으로 선택도가 급격히 높아지면 옵티마이저가 인덱스 선택


### 3. PK(UNIQUE 인덱스) 실습
```sql
-- PK 생성 (UNIQUE 인덱스 자동 생성)
ALTER TABLE CUSTOMER ADD CONSTRAINT PK_CUSTOMER PRIMARY KEY (CUST_ID);

-- UNIQUE 인덱스 확인
SELECT INDEX_NAME, INDEX_TYPE, UNIQUENESS
FROM USER_INDEXES
WHERE TABLE_NAME = 'CUSTOMER';

INDEX_NAME          |INDEX_TYPE|UNIQUENESS|
--------------------+----------+----------+
IDX_CUST_GENDER_NAME|NORMAL    |NONUNIQUE |
PK_CUSTOMER         |NORMAL    |UNIQUE    |


-- 1. UNIQUE SCAN (단건 = 조건)
EXPLAIN PLAN FOR
SELECT * FROM CUSTOMER WHERE CUST_ID = 500;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

PLAN_TABLE_OUTPUT                                                                          |
-------------------------------------------------------------------------------------------+
Plan hash value: 3929111785                                                                |
                                                                                           |
-------------------------------------------------------------------------------------------|
| Id  | Operation                   | Name        | Rows  | Bytes | Cost (%CPU)| Time     ||
-------------------------------------------------------------------------------------------|
|   0 | SELECT STATEMENT            |             |     1 |    22 |     2   (0)| 00:00:01 ||
|   1 |  TABLE ACCESS BY INDEX ROWID| CUSTOMER    |     1 |    22 |     2   (0)| 00:00:01 ||
|*  2 |   INDEX UNIQUE SCAN         | PK_CUSTOMER |     1 |       |     1   (0)| 00:00:01 ||
-------------------------------------------------------------------------------------------|
                                                                                           |
Predicate Information (identified by operation id):                                        |
---------------------------------------------------                                        |
                                                                                           |
   2 - access("CUST_ID"=500)                                                               |


-- 2. FULL SCAN (UNIQUE 인덱스인데 넓은 범위 조건)
-- 95%의 데이터를 읽어야 해서 FULL SCAN
EXPLAIN PLAN FOR
SELECT * FROM CUSTOMER WHERE CUST_ID > 500;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

PLAN_TABLE_OUTPUT                                                             |
------------------------------------------------------------------------------+
Plan hash value: 2844954298                                                   |
                                                                              |
------------------------------------------------------------------------------|
| Id  | Operation         | Name     | Rows  | Bytes | Cost (%CPU)| Time     ||
------------------------------------------------------------------------------|
|   0 | SELECT STATEMENT  |          |  9500 |   204K|    13   (0)| 00:00:01 ||
|*  1 |  TABLE ACCESS FULL| CUSTOMER |  9500 |   204K|    13   (0)| 00:00:01 ||
------------------------------------------------------------------------------|
                                                                              |
Predicate Information (identified by operation id):                           |
---------------------------------------------------                           |
                                                                              |
   1 - filter("CUST_ID">500)                                                  |


-- 3. INDEX RANGE SCAN (UNIQUE 인덱스인데 좁은 범위 조건)
EXPLAIN PLAN FOR
SELECT * FROM CUSTOMER WHERE CUST_ID > 9900;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

PLAN_TABLE_OUTPUT                                                                                  |
---------------------------------------------------------------------------------------------------+
Plan hash value: 1181449693                                                                        |
                                                                                                   |
---------------------------------------------------------------------------------------------------|
| Id  | Operation                           | Name        | Rows  | Bytes | Cost (%CPU)| Time     ||
---------------------------------------------------------------------------------------------------|
|   0 | SELECT STATEMENT                    |             |   100 |  2200 |     3   (0)| 00:00:01 ||
|   1 |  TABLE ACCESS BY INDEX ROWID BATCHED| CUSTOMER    |   100 |  2200 |     3   (0)| 00:00:01 ||
|*  2 |   INDEX RANGE SCAN                  | PK_CUSTOMER |   100 |       |     2   (0)| 00:00:01 ||
---------------------------------------------------------------------------------------------------|
                                                                                                   |
Predicate Information (identified by operation id):                                                |
---------------------------------------------------                                                |
                                                                                                   |
   2 - access("CUST_ID">9900)                                                                      |
```


## SCAN 종류 정리
- FULL SCAN
→ WHERE절에 인덱스 선두 컬럼이 없거나,
  있어도 선택도가 낮을 때

- INDEX SKIP SCAN
→ WHERE절에 인덱스 선두 컬럼이 없지만,
  선두 컬럼의 값 종류가 적어 구간별 탐색이 가능할 때

- INDEX RANGE SCAN
→ WHERE절에 인덱스 선두 컬럼이 있고,
  선택도가 높아 인덱스 탐색이 효율적일 때

- INDEX UNIQUE SCAN
→ WHERE절에 PK/UNIQUE 인덱스 컬럼이 = 조건으로 있을 때

UNIQUE 인덱스 + = 조건   → INDEX UNIQUE SCAN (무조건)
UNIQUE 인덱스 + 범위조건  → 선택도에 따라 결정
                           적은 건수: INDEX RANGE SCAN
                           많은 건수: TABLE ACCESS FULL