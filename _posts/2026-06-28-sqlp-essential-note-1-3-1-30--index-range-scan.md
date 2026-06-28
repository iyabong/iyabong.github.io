---
title: 국가공인 SQLP 자격검정 핵심노트 1 - 30. Index Range Scan 유도(4)
date: 2026-06-28
categories: [SQLP]
tags: [SQLP, 인덱스 튜닝, 인덱스 기본 원리, Index Range Scan]
---

[2024 국가공인 SQLP 자격검정 핵심노트 1](https://product.kyobobook.co.kr/detail/S000213913597)

- 인덱스 튜닝
  - 인덱스 기본 원리
    - **30. Index Range Scan 유도(4)**

```sql
-- 실행계획 수집
EXPLAIN PLAN FOR

-- 실행계획 출력
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY());
```

#### 0. 실습 준비
```sql
-- 일별지수업종별거래
CREATE TABLE DAILY_IDX_SECTOR_TRD (
    IDX_TYPE_CD   VARCHAR2(1)    NOT NULL,   -- 지수구분코드
    SECTOR_CD     VARCHAR2(3)    NOT NULL,   -- 지수업종코드
    TRD_DT        VARCHAR2(8)    NOT NULL,   -- 거래일자 (YYYYMMDD)
    CLOSE_VAL     NUMBER(15, 2),             -- 지수종가
    ACC_TRD_VOL   NUMBER(18)                 -- 누적거래량
);

-- PK : IDX_TYPE_CD + SECTOR_CD + TRD_DT
ALTER TABLE DAILY_IDX_SECTOR_TRD
    ADD CONSTRAINT DAILY_IDX_SECTOR_TRD_PK
    PRIMARY KEY (IDX_TYPE_CD, SECTOR_CD, TRD_DT);

-- X1 : TRD_DT  (일별지수업종별거래_X1)
CREATE INDEX DAILY_IDX_SECTOR_TRD_X1
    ON DAILY_IDX_SECTOR_TRD (TRD_DT);

INSERT INTO DAILY_IDX_SECTOR_TRD VALUES ('1', '001', '20240101', 2650.30,  5000000);
INSERT INTO DAILY_IDX_SECTOR_TRD VALUES ('1', '001', '20240102', 2660.50,  5200000);
INSERT INTO DAILY_IDX_SECTOR_TRD VALUES ('1', '001', '20240103', 2645.80,  4800000);
INSERT INTO DAILY_IDX_SECTOR_TRD VALUES ('1', '001', '20240104', 2670.10,  5100000);
INSERT INTO DAILY_IDX_SECTOR_TRD VALUES ('1', '001', '20240105', 2680.00,  5300000);

INSERT INTO DAILY_IDX_SECTOR_TRD VALUES ('2', '003', '20240101',  850.20,  3000000);
INSERT INTO DAILY_IDX_SECTOR_TRD VALUES ('2', '003', '20240102',  855.70,  3100000);
INSERT INTO DAILY_IDX_SECTOR_TRD VALUES ('2', '003', '20240103',  848.30,  2900000);
INSERT INTO DAILY_IDX_SECTOR_TRD VALUES ('2', '003', '20240104',  860.00,  3200000);
INSERT INTO DAILY_IDX_SECTOR_TRD VALUES ('2', '003', '20240105',  865.50,  3400000);

INSERT INTO DAILY_IDX_SECTOR_TRD VALUES ('1', '002', '20240101', 1200.00,  1000000);
INSERT INTO DAILY_IDX_SECTOR_TRD VALUES ('1', '002', '20240102', 1210.00,  1100000);
INSERT INTO DAILY_IDX_SECTOR_TRD VALUES ('3', '005', '20240101',  500.00,   800000);

COMMIT;
```

#### 튜닝 전
→ PK 선두컬럼을 가공(문자열 연결) 하였으므로, 옵티마이저가 PK 인덱스를 사용하지 않음.
```sql
-- startDt: '20240101'
-- endDt  : '20240105'
SELECT TRD_DT
     , SUM(DECODE(IDX_TYPE_CD, '1', CLOSE_VAL,   0)) KOSPI200_IDX
     , SUM(DECODE(IDX_TYPE_CD, '1', ACC_TRD_VOL, 0)) KOSPI200_IDX_TRDVOL
     , SUM(DECODE(IDX_TYPE_CD, '2', CLOSE_VAL,   0)) KOSDAQ_IDX
     , SUM(DECODE(IDX_TYPE_CD, '2', ACC_TRD_VOL, 0)) KOSDAQ_IDX_TRDVOL
FROM   DAILY_IDX_SECTOR_TRD A
WHERE  TRD_DT BETWEEN :startDt AND :endDt
  AND  IDX_TYPE_CD || SECTOR_CD IN ('1001', '2003')
GROUP BY TRD_DT
ORDER BY TRD_DT;
--------------------------------------------------------------------------------------------------------
| Id  | Operation                    | Name                    | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |                         |     1 |    23 |     2   (0)| 00:00:01 |
|   1 |  SORT GROUP BY NOSORT        |                         |     1 |    23 |     2   (0)| 00:00:01 |
|*  2 |   TABLE ACCESS BY INDEX ROWID| DAILY_IDX_SECTOR_TRD    |     1 |    23 |     2   (0)| 00:00:01 |
|*  3 |    INDEX RANGE SCAN          | DAILY_IDX_SECTOR_TRD_X1 |    13 |       |     1   (0)| 00:00:01 |
--------------------------------------------------------------------------------------------------------
```

#### 튜닝 1 - INLIST ITERATOR
1. PK 선두컬럼을 가공하는 대신, IN 조건으로 변경
2. 옵티마이저 HINT에 PK INDEX 사용 명시
  → PK INDEX RANGE SCAN 및 INLIST ITERATOR 유도 성공
```sql
-- startDt: '20240101'
-- endDt  : '20240105'
SELECT /*+ INDEX(A DAILY_IDX_SECTOR_TRD_PK) */ TRD_DT 
     , SUM(DECODE(IDX_TYPE_CD, '1', CLOSE_VAL,   0)) KOSPI200_IDX
     , SUM(DECODE(IDX_TYPE_CD, '1', ACC_TRD_VOL, 0)) KOSPI200_IDX_TRDVOL
     , SUM(DECODE(IDX_TYPE_CD, '2', CLOSE_VAL,   0)) KOSDAQ_IDX
     , SUM(DECODE(IDX_TYPE_CD, '2', ACC_TRD_VOL, 0)) KOSDAQ_IDX_TRDVOL
FROM   DAILY_IDX_SECTOR_TRD A
WHERE  TRD_DT BETWEEN :startDt AND :endDt
  AND (IDX_TYPE_CD,  SECTOR_CD)  IN ( ('1', '001'), ('2', '003') )
GROUP BY TRD_DT
ORDER BY TRD_DT;

-----------------------------------------------------------------------------------------------------------------
| Id  | Operation                             | Name                    | Rows  | Bytes | Cost (%CPU)| Time     |
-----------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                      |                         |     3 |    69 |     3  (34)| 00:00:01 |
|   1 |  SORT GROUP BY                        |                         |     3 |    69 |     3  (34)| 00:00:01 |
|   2 |   INLIST ITERATOR                     |                         |       |       |            |          |
|   3 |    TABLE ACCESS BY INDEX ROWID BATCHED| DAILY_IDX_SECTOR_TRD    |     5 |   115 |     2   (0)| 00:00:01 |
|*  4 |     INDEX RANGE SCAN                  | DAILY_IDX_SECTOR_TRD_PK |     5 |       |     1   (0)| 00:00:01 |
-----------------------------------------------------------------------------------------------------------------
   4 - access(("IDX_TYPE_CD"='1' AND "SECTOR_CD"='001' OR "IDX_TYPE_CD"='2' AND "SECTOR_CD"='003') AND 
              "TRD_DT">='20240101' AND "TRD_DT"<='20240105')
```

#### 튜닝 2 - UNION ALL
1. UNION ALL 명시적으로 구현
2. UNION ALL 후에, GROUP BY와 ORDER BY 처리
  → PK INDEX RANGE SCAN 및 UNION-ALL 유도 성공
```sql
-- startDt: '20240101'
-- endDt  : '20240105'
SELECT TRD_DT
	,SUM(KOSPI200_IDX) AS  KOSPI200_IDX
	,SUM(KOSPI200_IDX_TRDVOL) AS KOSPI200_IDX_TRDVOL
	,SUM(KOSDAQ_IDX) AS KOSDAQ_IDX
	,SUM(KOSDAQ_IDX_TRDVOL) AS KOSDAQ_IDX_TRDVOL
FROM
	(
	SELECT TRD_DT 
	     , CLOSE_VAL AS  KOSPI200_IDX
	     , ACC_TRD_VOL AS  KOSPI200_IDX_TRDVOL
	     , 0 AS KOSDAQ_IDX
	     , 0 AS KOSDAQ_IDX_TRDVOL
	FROM   DAILY_IDX_SECTOR_TRD A
	WHERE  TRD_DT BETWEEN :startDt AND :endDt
	  AND (IDX_TYPE_CD,  SECTOR_CD)  IN ( ('1', '001') )
	UNION ALL
	SELECT TRD_DT 
	     , 0 AS KOSPI200_IDX
	     , 0 AS KOSPI200_IDX_TRDVOL
	     , CLOSE_VAL AS KOSDAQ_IDX
	     , ACC_TRD_VOL AS  KOSDAQ_IDX_TRDVOL
	FROM   DAILY_IDX_SECTOR_TRD A
	WHERE  TRD_DT BETWEEN :startDt AND :endDt
	  AND (IDX_TYPE_CD,  SECTOR_CD)  IN (('2', '003') )
	)
GROUP BY TRD_DT
ORDER BY TRD_DT;

-----------------------------------------------------------------------------------------------------------|
| Id  | Operation                       | Name                    | Rows  | Bytes | Cost (%CPU)| Time     ||
-----------------------------------------------------------------------------------------------------------|
|   0 | SELECT STATEMENT                |                         |     3 |   183 |     5  (20)| 00:00:01 ||
|   1 |  SORT GROUP BY                  |                         |     3 |   183 |     5  (20)| 00:00:01 ||
|   2 |   VIEW                          |                         |     4 |   244 |     4   (0)| 00:00:01 ||
|   3 |    UNION-ALL                    |                         |     4 |    92 |     4   (0)| 00:00:01 ||
|   4 |     SORT GROUP BY NOSORT        |                         |     2 |    46 |     2   (0)| 00:00:01 ||
|   5 |      TABLE ACCESS BY INDEX ROWID| DAILY_IDX_SECTOR_TRD    |     3 |    69 |     2   (0)| 00:00:01 ||
|*  6 |       INDEX RANGE SCAN          | DAILY_IDX_SECTOR_TRD_PK |     3 |       |     1   (0)| 00:00:01 ||
|   7 |     SORT GROUP BY NOSORT        |                         |     2 |    46 |     2   (0)| 00:00:01 ||
|   8 |      TABLE ACCESS BY INDEX ROWID| DAILY_IDX_SECTOR_TRD    |     2 |    46 |     2   (0)| 00:00:01 ||
|*  9 |       INDEX RANGE SCAN          | DAILY_IDX_SECTOR_TRD_PK |     2 |       |     1   (0)| 00:00:01 ||
-----------------------------------------------------------------------------------------------------------|
   6 - access("IDX_TYPE_CD"='1' AND "SECTOR_CD"='001' AND "TRD_DT">='20240101' AND                         |
              "TRD_DT"<='20240105')                                                                        |
   9 - access("IDX_TYPE_CD"='2' AND "SECTOR_CD"='003' AND "TRD_DT">='20240101' AND                         |
              "TRD_DT"<='20240105')                                                                        |
```