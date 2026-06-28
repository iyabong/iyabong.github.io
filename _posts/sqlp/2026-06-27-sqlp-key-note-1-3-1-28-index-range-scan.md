---
title: 국가공인 SQLP 자격검정 핵심노트 1 - 28. Index Range Scan 유도(2)
date: 2026-06-27
categories: [SQLP, 국가공인 SQLP 자격검정 핵심노트 1]
tags: [SQLP, 인덱스 튜닝, 인덱스 기본 원리, Index Range Scan]
---

[2024 국가공인 SQLP 자격검정 핵심노트 1](https://product.kyobobook.co.kr/detail/S000213913597)

- 인덱스 튜닝
  - 인덱스 기본 원리
    - **28. Index Range Scan 유도(2)**

```sql
-- 실행계획 수집
EXPLAIN PLAN FOR

-- 실행계획 출력
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY());
```

#### 0. 실습 준비
```sql
 -- 1. 주문상태 코드 테이블 생성
CREATE TABLE ORDER_STATUS (
    STATUS_CODE NUMBER PRIMARY KEY,
    STATUS_NAME VARCHAR2(50) NOT NULL
);

-- 2. 주문 테이블 생성
CREATE TABLE ORDERS (
    ORDER_NO     NUMBER NOT NULL,
    ORDER_DT     VARCHAR2(8) NOT NULL,
    CUST_ID      VARCHAR2(20),
    TOTAL_AMT    NUMBER,
    STATUS_CODE  NUMBER NOT NULL
);

-- 3. 인덱스 생성
CREATE INDEX ORDERS_IDX_01 ON ORDERS(STATUS_CODE);

-- 4. 데이터 적재
INSERT INTO ORDER_STATUS VALUES ('0', 'Payment Pending');
INSERT INTO ORDER_STATUS VALUES ('1', 'Preparing Item');
INSERT INTO ORDER_STATUS VALUES ('2', 'Shipping');
INSERT INTO ORDER_STATUS VALUES ('3', 'Delivery Completed'); -- 99.93% 비중
INSERT INTO ORDER_STATUS VALUES ('4', 'Out of Stock');
INSERT INTO ORDER_STATUS VALUES ('5', 'Order Cancelled');

 INSERT INTO ORDERS
 SELECT 
    LEVEL AS ORDER_NO,
    '202605' ||  LPAD(MOD(LEVEL-1, 31)+1, 2, 0) AS ORDER_DT,
    'CUST_' || LPAD(MOD(LEVEL, 100), 4, '0') AS CUST_ID,
    TRUNC(DBMS_RANDOM.VALUE(10000, 500000), -3) AS TOTAL_AMT,
    3 AS STATUS_CODE -- 기본적으로 모두 배송완료(3)로 세팅
FROM DUAL
CONNECT BY LEVEL <= 10000;


-- 극소수의 데이터만 다른 상태 코드로 업데이트
-- 0.01%
UPDATE ORDERS SET STATUS_CODE = 0 WHERE ORDER_NO = 100;
UPDATE ORDERS SET STATUS_CODE = 4 WHERE ORDER_NO = 200;
UPDATE ORDERS SET STATUS_CODE = 5 WHERE ORDER_NO = 300;

-- 0.02%
UPDATE ORDERS SET STATUS_CODE = 1 WHERE ORDER_NO = 400;
UPDATE ORDERS SET STATUS_CODE = 1 WHERE ORDER_NO = 500;
UPDATE ORDERS SET STATUS_CODE = 2 WHERE ORDER_NO = 600;
UPDATE ORDERS SET STATUS_CODE = 2 WHERE ORDER_NO = 700;
COMMIT;

-- 데이터 분포 현황
SELECT STATUS_CODE, COUNT(*)
FROM ORDERS
GROUP BY ROLLUP(STATUS_CODE)
ORDER BY STATUS_CODE;

STATUS_CODE|COUNT(*)|
-----------+--------+
          0|       1|
          1|       2|
          2|       2|
          3|    9993|
          4|       1|
          5|       1|
           |   10000|

-- 오라클 통계정보 갱신
BEGIN
    DBMS_STATS.GATHER_TABLE_STATS(USER, 'ORDERS');
END;
```

#### 튜닝 전
```sql
SELECT ORDER_NO, ORDER_DT, CUST_ID, TOTAL_AMT, STATUS_CODE
FROM ORDERS
WHERE STATUS_CODE <> 3
 AND ORDER_DT BETWEEN :dt1 AND :dt2 -- '20260511' '20260516'
----------------------------------------------------------------------------
| Id  | Operation         | Name   | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |        |     1 |    30 |    15   (0)| 00:00:01 |
|*  1 |  TABLE ACCESS FULL| ORDERS |     1 |    30 |    15   (0)| 00:00:01 |
----------------------------------------------------------------------------
```

#### 튜닝 1 - INLIST ITERATOR
```sql
SELECT ORDER_NO, ORDER_DT, CUST_ID, TOTAL_AMT, STATUS_CODE
FROM ORDERS
WHERE STATUS_CODE IN (0,1,2,4,5)
 AND ORDER_DT BETWEEN :dt1 AND :dt2
------------------------------------------------------------------------------------------------------|
| Id  | Operation                            | Name          | Rows  | Bytes | Cost (%CPU)| Time     ||
------------------------------------------------------------------------------------------------------|
|   0 | SELECT STATEMENT                     |               |     1 |    30 |     5   (0)| 00:00:01 ||
|   1 |  INLIST ITERATOR                     |               |       |       |            |          ||
|*  2 |   TABLE ACCESS BY INDEX ROWID BATCHED| ORDERS        |     1 |    30 |     5   (0)| 00:00:01 ||
|*  3 |    INDEX RANGE SCAN                  | ORDERS_IDX_01 |     7 |       |     4   (0)| 00:00:01 ||
------------------------------------------------------------------------------------------------------|
   2 - filter("ORDER_DT"<='20260516' AND "ORDER_DT">='20260511')                                      |
   3 - access("STATUS_CODE"=0 OR "STATUS_CODE"=1 OR "STATUS_CODE"=2 OR "STATUS_CODE"=4 OR             |
              "STATUS_CODE"=5)                                                                        |
```

#### 튜닝 2 - QUERY BLOCK
```sql
SELECT /*+ UNNEST(@S1) LEADING(ORDER_STATUS@S1) USE_NL(ORDERS) */ 
       ORDER_NO, ORDER_DT, CUST_ID, TOTAL_AMT, STATUS_CODE
FROM ORDERS
WHERE STATUS_CODE IN (SELECT /*+ QB_NAME(S1) */ STATUS_CODE
                        FROM ORDER_STATUS
                       WHERE STATUS_CODE <> 3)
AND ORDER_DT BETWEEN :dt1 AND :dt2;
----------------------------------------------------------------------------------------------
| Id  | Operation                    | Name          | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |               |  1615 | 53295 |    66   (0)| 00:00:01 |
|   1 |  NESTED LOOPS                |               |  1615 | 53295 |    66   (0)| 00:00:01 |
|   2 |   NESTED LOOPS               |               |  8335 | 53295 |    66   (0)| 00:00:01 |
|*  3 |    INDEX FULL SCAN           | SYS_C0061097  |     5 |    15 |     1   (0)| 00:00:01 |
|*  4 |    INDEX RANGE SCAN          | ORDERS_IDX_01 |  1667 |       |     3   (0)| 00:00:01 |
|*  5 |   TABLE ACCESS BY INDEX ROWID| ORDERS        |   323 |  9690 |    13   (0)| 00:00:01 |
----------------------------------------------------------------------------------------------
```

#### 튜닝 3 - INLINE VIEW MERGING
```sql
SELECT /*+ ORDERED USE_NL(O) */ 
       O.ORDER_NO, O.ORDER_DT, O.CUST_ID, O.TOTAL_AMT, S.STATUS_CODE
FROM (SELECT STATUS_CODE
        FROM ORDER_STATUS
       WHERE STATUS_CODE <> 3) S, ORDERS O
WHERE S.STATUS_CODE = O.STATUS_CODE  
  AND O.ORDER_DT BETWEEN :dt1 AND :dt2;
----------------------------------------------------------------------------------------------
| Id  | Operation                    | Name          | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |               |  1615 | 53295 |    66   (0)| 00:00:01 |
|   1 |  NESTED LOOPS                |               |  1615 | 53295 |    66   (0)| 00:00:01 |
|   2 |   NESTED LOOPS               |               |  8335 | 53295 |    66   (0)| 00:00:01 |
|*  3 |    INDEX FULL SCAN           | SYS_C0061097  |     5 |    15 |     1   (0)| 00:00:01 |
|*  4 |    INDEX RANGE SCAN          | ORDERS_IDX_01 |  1667 |       |     3   (0)| 00:00:01 |
|*  5 |   TABLE ACCESS BY INDEX ROWID| ORDERS        |   323 |  9690 |    13   (0)| 00:00:01 |
----------------------------------------------------------------------------------------------
```

#### 튜닝 4 - VW_ORE(View Or Rewrite Expansion)
```sql
SELECT /*+ INDEX(O) */
	 O.ORDER_NO, O.ORDER_DT, O.CUST_ID, O.TOTAL_AMT, O.STATUS_CODE
 FROM ORDERS O
WHERE (O.STATUS_CODE < 3 OR STATUS_CODE > 3) 
  AND O.ORDER_DT BETWEEN :dt1 AND :dt2;
---------------------------------------------------------------------------------------------------------
| Id  | Operation                             | Name            | Rows  | Bytes | Cost (%CPU)| Time     |
---------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                      |                 |     2 |   114 |     6   (0)| 00:00:01 |
|   1 |  VIEW                                 | VW_ORE_72AE2D8F |     2 |   114 |     6   (0)| 00:00:01 |
|   2 |   UNION-ALL                           |                 |     2 |    60 |     6   (0)| 00:00:01 |
|*  3 |    TABLE ACCESS BY INDEX ROWID BATCHED| ORDERS          |     1 |    30 |     3   (0)| 00:00:01 |
|*  4 |     INDEX RANGE SCAN                  | ORDERS_IDX_01   |     6 |       |     2   (0)| 00:00:01 |
|*  5 |    TABLE ACCESS BY INDEX ROWID BATCHED| ORDERS          |     1 |    30 |     3   (0)| 00:00:01 |
|*  6 |     INDEX RANGE SCAN                  | ORDERS_IDX_01   |     1 |       |     2   (0)| 00:00:01 |
---------------------------------------------------------------------------------------------------------
```

#### 튜닝 5 - UNION ALL
```sql
EXPLAIN PLAN FOR
SELECT O.ORDER_NO, O.ORDER_DT, O.CUST_ID, O.TOTAL_AMT, O.STATUS_CODE
 FROM ORDERS O
WHERE O.STATUS_CODE < 3
  AND O.ORDER_DT BETWEEN :dt1 AND :dt2
UNION ALL
SELECT O.ORDER_NO, O.ORDER_DT, O.CUST_ID, O.TOTAL_AMT, O.STATUS_CODE
 FROM ORDERS O
WHERE O.STATUS_CODE > 3
  AND O.ORDER_DT BETWEEN :dt1 AND :dt2
------------------------------------------------------------------------------------------------------
| Id  | Operation                            | Name          | Rows  | Bytes | Cost (%CPU)| Time     |
------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                     |               |     2 |    60 |     6   (0)| 00:00:01 |
|   1 |  UNION-ALL                           |               |     2 |    60 |     6   (0)| 00:00:01 |
|*  2 |   TABLE ACCESS BY INDEX ROWID BATCHED| ORDERS        |     1 |    30 |     3   (0)| 00:00:01 |
|*  3 |    INDEX RANGE SCAN                  | ORDERS_IDX_01 |     6 |       |     2   (0)| 00:00:01 |
|*  4 |   TABLE ACCESS BY INDEX ROWID BATCHED| ORDERS        |     1 |    30 |     3   (0)| 00:00:01 |
|*  5 |    INDEX RANGE SCAN                  | ORDERS_IDX_01 |     1 |       |     2   (0)| 00:00:01 |
------------------------------------------------------------------------------------------------------
```