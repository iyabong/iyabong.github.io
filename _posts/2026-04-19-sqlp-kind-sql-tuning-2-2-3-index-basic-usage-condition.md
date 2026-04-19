---
title: 친절한 SQL 튜닝 - 2.2.3 더 중요한 인덱스 사용 조건
date: 2026-04-19
categories: [SQLP]
tags: [SQLP, 친절한 SQL 튜닝, 인덱스, INDEX]
---

# 친절한 SQL 튜닝
2 인덱스 기본
  2.2 인덱스 기본 사용법
    2.2.3 더 중요한 인덱스 사용 조건
[친절한 SQL 튜닝](https://product.kyobobook.co.kr/detail/S000001975837)
---

## 1. 인덱스에서 선두 컬럼 없이 조회하는 실습

### 1-0. 실습 데이터 생성
```sql
-- 사원 테이블 생성
CREATE TABLE 사원 (
    사원번호 NUMBER,
    사원명   VARCHAR2(20),
    소속팀   VARCHAR2(20),
    연령     NUMBER,
    입사일자 DATE,
    전화번호 VARCHAR2(20)
);

-- 데이터 입력
INSERT INTO 사원 VALUES (1, '홍길동', '생산', 23, DATE '2020-01-01', '010-1111-1111');
INSERT INTO 사원 VALUES (2, '홍길동', '영업', 36, DATE '2018-03-01', '010-2222-2222');
INSERT INTO 사원 VALUES (3, '홍길동', '연구개발', 49, DATE '2015-06-01', '010-3333-3333');
INSERT INTO 사원 VALUES (4, '김소현', '연구개발', 36, DATE '2019-01-01', '010-4444-4444');
INSERT INTO 사원 VALUES (5, '이해정', '마케팅', 41, DATE '2017-05-01', '010-5555-5555');
INSERT INTO 사원 VALUES (6, '홍길동', '인사', 36, DATE '2016-01-01', '010-6666-6666');
INSERT INTO 사원 VALUES (7, '홍길동', '마케팅', 34, DATE '2021-01-01', '010-7777-7777');

-- 대량 데이터 추가
INSERT INTO 사원
SELECT ROWNUM + 10,
       DBMS_RANDOM.STRING('A', 4),
       CASE MOD(ROWNUM, 6)
           WHEN 0 THEN '생산'
           WHEN 1 THEN '영업'
           WHEN 2 THEN '마케팅'
           WHEN 3 THEN '인사'
           WHEN 4 THEN '연구개발'
           ELSE '회계'
       END,
       MOD(ROWNUM, 40) + 20,
       SYSDATE - MOD(ROWNUM, 3650),
       '010-0000-' || LPAD(ROWNUM, 4, '0')
FROM DUAL CONNECT BY ROWNUM <= 10000;
COMMIT;

-- 인덱스 생성 (소속팀 + 사원명 + 연령)
CREATE INDEX IDX_사원 ON 사원(소속팀, 사원명, 연령);
```

### 1-1. 인덱스 선두컬럼 없이 사원명만 조건
사원명만 조건 → 선두(소속팀) 없음
 → 리프 블록 전 구간에 흩어져 있음
 → Range Scan 불가
```sql
EXPLAIN PLAN FOR
SELECT 사원번호, 소속팀, 연령, 입사일자, 전화번호
  FROM 사원
 WHERE 사원명 = '홍길동';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

PLAN_TABLE_OUTPUT                                                                             |
----------------------------------------------------------------------------------------------+
Plan hash value: 53218996                                                                     |
                                                                                              |
----------------------------------------------------------------------------------------------|
| Id  | Operation                           | Name   | Rows  | Bytes | Cost (%CPU)| Time     ||
----------------------------------------------------------------------------------------------|
|   0 | SELECT STATEMENT                    |        |     1 |    43 |     8   (0)| 00:00:01 ||
|   1 |  TABLE ACCESS BY INDEX ROWID BATCHED| 사원   |     1 |    43 |     8   (0)| 00:00:01 ||
|*  2 |   INDEX SKIP SCAN                   | IDX_사 |     1 |       |     7   (0)| 00:00:01 ||
----------------------------------------------------------------------------------------------|
                                                                                              |
Predicate Information (identified by operation id):                                           |
---------------------------------------------------                                           |
                                                                                              |
   2 - access("사원명"='홍길동')                                                              |
       filter("사원명"='홍길동')                                                              |
```

### 1-2. 인덱스 선두컬럼 조건 포함
소속팀 + 사원명 조건 → 선두 있음
→ 연속된 구간 탐색 가능
```sql
EXPLAIN PLAN FOR
SELECT 사원번호, 소속팀, 연령, 입사일자, 전화번호
  FROM 사원
 WHERE 소속팀 = '생산' AND 사원명 = '홍길동';

PLAN_TABLE_OUTPUT                                                                             |
----------------------------------------------------------------------------------------------+
Plan hash value: 321857020                                                                    |
                                                                                              |
----------------------------------------------------------------------------------------------|
| Id  | Operation                           | Name   | Rows  | Bytes | Cost (%CPU)| Time     ||
----------------------------------------------------------------------------------------------|
|   0 | SELECT STATEMENT                    |        |     1 |    43 |     3   (0)| 00:00:01 ||
|   1 |  TABLE ACCESS BY INDEX ROWID BATCHED| 사원   |     1 |    43 |     3   (0)| 00:00:01 ||
|*  2 |   INDEX RANGE SCAN                  | IDX_사 |     1 |       |     2   (0)| 00:00:01 ||
----------------------------------------------------------------------------------------------|
                                                                                              |
Predicate Information (identified by operation id):                                           |
---------------------------------------------------                                           |
                                                                                              |
   2 - access("소속팀"='생산' AND "사원명"='홍길동')                                          |
```

## 2. 인덱스 컬럼 가공 실습

### 2-0. 실습 데이터 생성
```sql
-- 테이블 생성
CREATE TABLE TAX (
    기준연도        VARCHAR2(4),
    과세구분코드    VARCHAR2(8),
    보고회차        NUMBER,
    실명확인번호    VARCHAR2(10)
);

-- 인덱스 생성
CREATE INDEX IDX_TAX ON TAX(기준연도, 과세구분코드, 보고회차, 실명확인번호);

-- 데이터 입력
INSERT INTO TAX
SELECT LPAD(2010 + MOD(ROWNUM, 14), 4),
       LPAD(MOD(ROWNUM, 10), 8, '0'),
       MOD(ROWNUM, 5) + 1,
       LPAD(ROWNUM, 10, '0')
FROM DUAL CONNECT BY ROWNUM <= 10000;

INSERT INTO SQLP.TAX (기준연도,과세구분코드,보고회차,실명확인번호) VALUES ('2024','00000004',4,'0000009999');
INSERT INTO SQLP.TAX (기준연도,과세구분코드,보고회차,실명확인번호) VALUES ('2024','00000003',3,'0000009998');
INSERT INTO SQLP.TAX (기준연도,과세구분코드,보고회차,실명확인번호) VALUES ('2024','00000002',2,'0000009997');
INSERT INTO SQLP.TAX (기준연도,과세구분코드,보고회차,실명확인번호) VALUES ('2024','00000001',1,'0000009996');

COMMIT;
```

## 2-1. 인덱스 선두 컬럼 가공하여 조회
 → 인덱스 무력화
```sql
EXPLAIN PLAN FOR
SELECT * 
 FROM TAX
WHERE SUBSTR(기준연도, 3,2) = '24'
  AND SUBSTR(과세구분코드, 8, 1) = '4'
  AND 보고회차 = 1;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);  

PLAN_TABLE_OUTPUT                                                         |
--------------------------------------------------------------------------+
Plan hash value: 1630028917                                               |
                                                                          |
--------------------------------------------------------------------------|
| Id  | Operation         | Name | Rows  | Bytes | Cost (%CPU)| Time     ||
--------------------------------------------------------------------------|
|   0 | SELECT STATEMENT  |      |     1 |    28 |    17   (0)| 00:00:01 ||
|*  1 |  TABLE ACCESS FULL| TAX  |     1 |    28 |    17   (0)| 00:00:01 ||
--------------------------------------------------------------------------|
                                                                          |
Predicate Information (identified by operation id):                       |
---------------------------------------------------                       |
                                                                          |
   1 - filter("보고회차"=1 AND SUBSTR("기준연도",3,2)='24' AND            |
              SUBSTR("과세구분코드",8,1)='4')                             |
```              

### 2-2. 인덱스 선두 외 컬럼 가공하여 조회
 → 인덱스 사용 가능
```sql
EXPLAIN PLAN FOR
SELECT * 
  FROM TAX
 WHERE 기준연도 = '2024'
   AND SUBSTR(과세구분코드, 8, 1) = '4'
   AND 보고회차 = 1

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

PLAN_TABLE_OUTPUT                                                           |
----------------------------------------------------------------------------+
Plan hash value: 211306740                                                  |
                                                                            |
----------------------------------------------------------------------------|
| Id  | Operation        | Name    | Rows  | Bytes | Cost (%CPU)| Time     ||
----------------------------------------------------------------------------|
|   0 | SELECT STATEMENT |         |     1 |    28 |     2   (0)| 00:00:01 ||
|*  1 |  INDEX RANGE SCAN| IDX_TAX |     1 |    28 |     2   (0)| 00:00:01 ||
----------------------------------------------------------------------------|
                                                                            |
Predicate Information (identified by operation id):                         |
---------------------------------------------------                         |
                                                                            |
   1 - access("기준연도"='2024' AND "보고회차"=1)                           |
       filter("보고회차"=1 AND SUBSTR("과세구분코드",8,1)='4')              |
```