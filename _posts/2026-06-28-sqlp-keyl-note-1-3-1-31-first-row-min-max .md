---
title: 국가공인 SQLP 자격검정 핵심노트 1 - 31. FIRST ROW(MIN/MAX)
date: 2026-06-28
categories: [SQLP]
tags: [SQLP, 인덱스 튜닝, 인덱스 기본 원리, Index Range Scan]
---

[2024 국가공인 SQLP 자격검정 핵심노트 1](https://product.kyobobook.co.kr/detail/S000213913597)

- 인덱스 튜닝
  - 인덱스 기본 원리
    - **31. FIRST ROW(MIN/MAX)**

```sql
-- 실행계획 수집
EXPLAIN PLAN FOR

-- 실행계획 출력
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY());
```

#### 0. 실습 준비
```sql
-- 주문 테이블 생성
CREATE TABLE ORD
(
	ORD_DT VARCHAR2(8),	
	ORD_NO NUMBER,
	ITEM_CD VARCHAR2(20),
	ORD_CNT NUMBER
);

-- PK : ORD_DT + ORD_NO
ALTER TABLE ORD
ADD CONSTRAINT ORD_PK PRIMARY KEY (ORD_DT,ORD_NO) ENABLE;

-- 데이터 적재
INSERT INTO ORD
SELECT '20260628' AS ORD_DT,
       LEVEL AS ORD_NO,
       'ITEM' || LPAD(TO_CHAR(LEVEL), 4, 0) AS ITEM_CD,
       LEVEL * 10 AS ORD_CNT 
FROM DUAL
CONNECT BY LEVEL <= 100;

COMMIT;
```

#### 튜닝 전
인덱스 컬럼인 ORD_NO를 가공(+1)
→ FIRST ROW(MIN/MAX) 방식 미작동
```sql
-- ordDt: '20260628'
SELECT NVL(MAX(ORD_NO+1), 1) AS ORD_NO
FROM ORD
WHERE ORD_DT = :ordDt;
----------------------------------------------------------------------------
| Id  | Operation         | Name   | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |        |     1 |    19 |     1   (0)| 00:00:01 |
|   1 |  SORT AGGREGATE   |        |     1 |    19 |            |          |
|*  2 |   INDEX RANGE SCAN| ORD_PK |     1 |    19 |     1   (0)| 00:00:01 |
----------------------------------------------------------------------------
```

#### 튜닝 - FIRST ROW(MIN/MAX) 유도
1. PK컬럼 가공 없이 MAX(ORD_NO) 
2. MAX(ORD_NO) 후에 +1 및 NVL 적용
  → FIRST ROW(MIN/MAX) 유도 성공

```sql
-- ordDt: '20260628'
SELECT NVL(MAX(ORD_NO) +1, 1) AS ORD_NO
FROM ORD
WHERE ORD_DT = :ordDt;
---------------------------------------------------------------------------------------
| Id  | Operation                    | Name   | Rows  | Bytes | Cost (%CPU)| Time     |
---------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |        |     1 |    12 |     1   (0)| 00:00:01 |
|   1 |  SORT AGGREGATE              |        |     1 |    12 |            |          |
|   2 |   FIRST ROW                  |        |     1 |    12 |     1   (0)| 00:00:01 |
|*  3 |    INDEX RANGE SCAN (MIN/MAX)| ORD_PK |     1 |    12 |     1   (0)| 00:00:01 |
---------------------------------------------------------------------------------------
```