---
title: 국가공인 SQLP 자격검정 핵심노트 1 - 29. Index Range Scan 유도(3)
date: 2026-06-28
categories: [SQLP]
tags: [SQLP, 인덱스 튜닝, 인덱스 기본 원리, Index Range Scan]
---

[2024 국가공인 SQLP 자격검정 핵심노트 1](https://product.kyobobook.co.kr/detail/S000213913597)

- 인덱스 튜닝
  - 인덱스 기본 원리
    - **29. Index Range Scan 유도(3)**

```sql
-- 실행계획 수집
EXPLAIN PLAN FOR

-- 실행계획 출력
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY());
```

#### 0. 실습 준비
```sql
-- 1. 월별계좌상태
CREATE TABLE MONTHLY_ACCOUNT_STATUS (
    ACCOUNT_NO     VARCHAR2(10),
    ACCOUNT_SEQ    NUMBER(5),
    BASE_YM        VARCHAR2(6),
    STATUS_CODE    VARCHAR2(2),
    CONSTRAINT MONTHLY_ACCOUNT_STATUS_PK PRIMARY KEY (ACCOUNT_NO, ACCOUNT_SEQ, BASE_YM)
);
CREATE INDEX MONTHLY_ACCOUNT_STATUS_X1 ON MONTHLY_ACCOUNT_STATUS (BASE_YM, STATUS_CODE);

-- 2. 계좌원장
CREATE TABLE ACCOUNT_MASTER (
    ACCOUNT_NO     VARCHAR2(10),
    ACCOUNT_SEQ    NUMBER(5),
    OPEN_DATE      VARCHAR2(8),
    CONSTRAINT ACCOUNT_MASTER_PK PRIMARY KEY (ACCOUNT_NO, ACCOUNT_SEQ)
);

-- 계좌원장 데이터
INSERT INTO ACCOUNT_MASTER VALUES ('A0001', 1, '20260115');
INSERT INTO ACCOUNT_MASTER VALUES ('A0002', 1, '20260120');
INSERT INTO ACCOUNT_MASTER VALUES ('A0003', 1, '20260211');
INSERT INTO ACCOUNT_MASTER VALUES ('A0004', 1, '20260305');
INSERT INTO ACCOUNT_MASTER VALUES ('A0005', 1, '20260520');

-- 월별계좌상태 데이터
INSERT INTO MONTHLY_ACCOUNT_STATUS VALUES ('A0001', 1, '202606', '01');
INSERT INTO MONTHLY_ACCOUNT_STATUS VALUES ('A0002', 1, '202606', '02');
INSERT INTO MONTHLY_ACCOUNT_STATUS VALUES ('A0003', 1, '202606', '01');
INSERT INTO MONTHLY_ACCOUNT_STATUS VALUES ('A0004', 1, '202606', '03');
INSERT INTO MONTHLY_ACCOUNT_STATUS VALUES ('A0005', 1, '202606', '05');

COMMIT;
```

#### 튜닝 전
```sql
-- :BASE_DT = '202606'
-- :STD_YM  = '202601'
UPDATE MONTHLY_ACCOUNT_STATUS -- 월별계좌상태
SET    STATUS_CODE = '07'
WHERE  STATUS_CODE <> '01'
AND    BASE_YM = :BASE_DT
AND    ACCOUNT_NO || ACCOUNT_SEQ IN (
           SELECT ACCOUNT_NO || ACCOUNT_SEQ
           FROM   ACCOUNT_MASTER -- 계좌원장
           WHERE  OPEN_DATE LIKE :STD_YM || '%'
       );
-------------------------------------------------------------------------------------------------------------------
| Id  | Operation                             | Name                      | Rows  | Bytes | Cost (%CPU)| Time     |
-------------------------------------------------------------------------------------------------------------------
|   0 | UPDATE STATEMENT                      |                           |     1 |    37 |     5   (0)| 00:00:01 |
|   1 |  UPDATE                               | MONTHLY_ACCOUNT_STATUS    |       |       |            |          |
|*  2 |   HASH JOIN SEMI                      |                           |     1 |    37 |     5   (0)| 00:00:01 |
|   3 |    TABLE ACCESS BY INDEX ROWID BATCHED| MONTHLY_ACCOUNT_STATUS    |     3 |    57 |     2   (0)| 00:00:01 |
|*  4 |     INDEX RANGE SCAN                  | MONTHLY_ACCOUNT_STATUS_X1 |     3 |       |     1   (0)| 00:00:01 |
|*  5 |    TABLE ACCESS FULL                  | ACCOUNT_MASTER            |     3 |    54 |     3   (0)| 00:00:01 |
-------------------------------------------------------------------------------------------------------------------
```

#### 튜닝 1 - || to IN
- PK Index Skip Scan 유도: 성공
- PK Index Range Scan 유도: 실패
```sql
-- :BASE_DT = '202606'
-- :STD_YM  = '202601'
UPDATE  /*+ INDEX(T MONTHLY_ACCOUNT_STATUS_PK) */ MONTHLY_ACCOUNT_STATUS T -- 월별계좌상태
SET    STATUS_CODE = '07'
WHERE  STATUS_CODE > '01'
AND    BASE_YM = :BASE_DT
AND    (ACCOUNT_NO, ACCOUNT_SEQ) IN 
       (SELECT ACCOUNT_NO, ACCOUNT_SEQ
          FROM  ACCOUNT_MASTER -- 계좌원장
         WHERE  OPEN_DATE LIKE :STD_YM || '%' );
--------------------------------------------------------------------------------------------------------------------
| Id  | Operation                              | Name                      | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------------------------------------------------
|   0 | UPDATE STATEMENT                       |                           |     2 |    74 |     5  (20)| 00:00:01 |
|   1 |  UPDATE                                | MONTHLY_ACCOUNT_STATUS    |       |       |            |          |
|   2 |   MERGE JOIN                           |                           |     2 |    74 |     5  (20)| 00:00:01 |
|*  3 |    TABLE ACCESS BY INDEX ROWID         | ACCOUNT_MASTER            |     3 |    54 |     2   (0)| 00:00:01 |
|   4 |     INDEX FULL SCAN                    | ACCOUNT_MASTER_PK         |     5 |       |     1   (0)| 00:00:01 |
|*  5 |    SORT JOIN                           |                           |     3 |    57 |     3  (34)| 00:00:01 |
|*  6 |     TABLE ACCESS BY INDEX ROWID BATCHED| MONTHLY_ACCOUNT_STATUS    |     3 |    57 |     2   (0)| 00:00:01 |
|*  7 |      INDEX SKIP SCAN                   | MONTHLY_ACCOUNT_STATUS_PK |     5 |       |     1   (0)| 00:00:01 |
--------------------------------------------------------------------------------------------------------------------
```