---
title: 친절한 SQL 튜닝 - 2.2 인덱스 기본 사용법
date: 2026-04-18
categories: [SQLP, 친절한 SQL 튜닝]
tags: [SQLP, 친절한 SQL 튜닝, 인덱스 기본 사용법]
---

# 친절한 SQL 튜닝
2 인덱스 기본
  2.2 인덱스 기본 사용법

[친절한 SQL 튜닝](https://product.kyobobook.co.kr/detail/S000001975837)
---

## 0. 인덱스 생성
```sql
CREATE INDEX IDX_ORDERS_CUST ON ORDERS(CUST_ID);
CREATE INDEX IDX_ORDERS_STATUS ON ORDERS(STATUS);
```

## 1. INLIST ITERATOR — 같은 컬럼 OR/IN
 → OR와 IN은 내부적으로 동일 (PLAN HASH VALUE 같음)

### 1-1. OR 조건
 → 결과: INLIST ITERATOR → INDEX RANGE SCAN 반복
```sql
SELECT * 
  FROM ORDERS 
 WHERE CUST_ID = 100 OR CUST_ID = 200;

--------------------------------------------------------------------------------------------------------+
Plan hash value: 3500756426                                                                             |
--------------------------------------------------------------------------------------------------------|
| Id  | Operation                            | Name            | Rows  | Bytes | Cost (%CPU)| Time     ||
--------------------------------------------------------------------------------------------------------|
|   0 | SELECT STATEMENT                     |                 |    20 |   800 |    23   (0)| 00:00:01 ||
|   1 |  INLIST ITERATOR                     |                 |       |       |            |          ||
|   2 |   TABLE ACCESS BY INDEX ROWID BATCHED| ORDERS          |    20 |   800 |    23   (0)| 00:00:01 ||
|*  3 |    INDEX RANGE SCAN                  | IDX_ORDERS_CUST |    20 |       |     3   (0)| 00:00:01 ||
--------------------------------------------------------------------------------------------------------|
                                                                                                        |
Predicate Information (identified by operation id):                                                     |
---------------------------------------------------                                                     |
   3 - access("CUST_ID"=100 OR "CUST_ID"=200)                                                           |
```

### 1-2. IN 조건
 → 결과: INLIST ITERATOR (OR와 완전히 동일)
```sql
SELECT *
  FROM ORDERS
 WHERE CUST_ID IN (100, 200);

--------------------------------------------------------------------------------------------------------+
Plan hash value: 3500756426                                                                             |
--------------------------------------------------------------------------------------------------------|
| Id  | Operation                            | Name            | Rows  | Bytes | Cost (%CPU)| Time     ||
--------------------------------------------------------------------------------------------------------|
|   0 | SELECT STATEMENT                     |                 |    20 |   800 |    23   (0)| 00:00:01 ||
|   1 |  INLIST ITERATOR                     |                 |       |       |            |          ||
|   2 |   TABLE ACCESS BY INDEX ROWID BATCHED| ORDERS          |    20 |   800 |    23   (0)| 00:00:01 ||
|*  3 |    INDEX RANGE SCAN                  | IDX_ORDERS_CUST |    20 |       |     3   (0)| 00:00:01 ||
--------------------------------------------------------------------------------------------------------|
                                                                                                        |
Predicate Information (identified by operation id):                                                     |
---------------------------------------------------                                                     |
   3 - access("CUST_ID"=100 OR "CUST_ID"=200)                                                           |
```

## 2. OR Expansion — 다른 컬럼 OR

### 2-1. 옵티마이저 자동 선택: FULL SCAN (선택도 낮아서)
```sql
SELECT * 
  FROM ORDERS
 WHERE CUST_ID = 100 OR STATUS = 'COMPLETE';

----------------------------------------------------------------------------+
Plan hash value: 1275100350                                                 |
----------------------------------------------------------------------------|
| Id  | Operation         | Name   | Rows  | Bytes | Cost (%CPU)| Time     ||
----------------------------------------------------------------------------|
|   0 | SELECT STATEMENT  |        | 33340 |  1302K|   205   (1)| 00:00:01 ||
|*  1 |  TABLE ACCESS FULL| ORDERS | 33340 |  1302K|   205   (1)| 00:00:01 ||
----------------------------------------------------------------------------|
                                                                            |
Predicate Information (identified by operation id):                         |
---------------------------------------------------                         |
   1 - filter("STATUS"='COMPLETE' OR "CUST_ID"=100)                         |
```

### 2-2. USE_CONCAT 힌트로 OR Expansion 강제
```sql
SELECT /*+ USE_CONCAT */ *
  FROM ORDERS
 WHERE CUST_ID = 100 OR STATUS = 'COMPLETE';

-- 결과: CONCATENATION
--       ├── INDEX RANGE SCAN (cust_id=100, 10건)
--       └── TABLE ACCESS FULL (status=COMPLETE, 33,330건)
--       + LNNVL로 중복 제거 자동 추가
-- Cost: 216 → 옵티마이저의 FULL SCAN(205)이 더 효율적
--------------------------------------------------------------------------------------------------------+
Plan hash value: 3079636013                                                                             |
--------------------------------------------------------------------------------------------------------|
| Id  | Operation                            | Name            | Rows  | Bytes | Cost (%CPU)| Time     ||
--------------------------------------------------------------------------------------------------------|
|   0 | SELECT STATEMENT                     |                 | 33340 |  1302K|   216   (1)| 00:00:01 ||
|   1 |  CONCATENATION                       |                 |       |       |            |          ||
|   2 |   TABLE ACCESS BY INDEX ROWID BATCHED| ORDERS          |    10 |   400 |    11   (0)| 00:00:01 ||
|*  3 |    INDEX RANGE SCAN                  | IDX_ORDERS_CUST |    10 |       |     1   (0)| 00:00:01 ||
|*  4 |   TABLE ACCESS FULL                  | ORDERS          | 33330 |  1301K|   205   (1)| 00:00:01 ||
--------------------------------------------------------------------------------------------------------|
                                                                                                        |
Predicate Information (identified by operation id):                                                     |
---------------------------------------------------                                                     |
   3 - access("CUST_ID"=100)                                                                            |
   4 - filter("STATUS"='COMPLETE' AND LNNVL("CUST_ID"=100))                                             |
```
