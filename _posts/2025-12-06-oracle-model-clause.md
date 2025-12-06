---
title: "Oracle 실습 — MODEL 절"
date: 2-25-12-06
categories: [Oracle, SQL]
tags: [Oracle, SQL, MODEL]
---


## 💡 오라클 MODEL 절 소개
- SQL 쿼리 내에서 엑셀처럼 **배열 형태 게산**을 수행할 수 있는 기능
- 각 행을 배열의 좌표처럼 인식하여 복잡한 수식이나 이전 행의 값을 참조하는 계산을 직관적으로 처리 가능

> LAG()나 LEAD()보다 훨씬 복잡한 로직(재귀 계산, 예측 모델링 등)에 적합

### 1️⃣ MODEL 개념 : 엑셀 시트로 가정
 - PARTITION BY (시트): 데이터를 그룹으로 나눈다. (예: 부서별, 국가별)
 - DIMENSION BY (행/열 좌표): 특정 셀(Cell)을 찾기 위한 좌표 (예: 날짜, 제품ID) ※ 배열의 인덱스(Key) 역할
 - MEASURES (값): 실제 계산하고 조작할 데이터 값 (예: 매출액)  ※ 엑셀 셀 안의 값
 - RULES (수식): 값을 어떻게 계산할지 정의하는 공식

### 2️⃣ MODEL 문법 구조
 - PARTITION BY (시트): 데이터를 그룹으로 나눈다. (예: 부서별, 국가별)
 - DIMENSION BY (행/열 좌표): 특정 셀(Cell)을 찾기 위한 좌표 (예: 날짜, 제품ID) ※ 배열의 인덱스(Key) 역할
 - MEASURES (값): 실제 계산하고 조작할 데이터 값 (예: 매출액)  ※ 엑셀 셀 안의 값
 - RULES (수식): 값을 어떻게 계산할지 정의하는 공식

```sql
SELECT *
FROM 테이블명
MODEL
  PARTITION BY (그룹컬럼)       -- 1. 데이터를 나눌 기준
  DIMENSION BY (키컬럼)         -- 2. 각 행을 식별할 좌표
  MEASURES (값컬럼)             -- 3. 계산 대상이 되는 값
  RULES (
    -- 4. 계산 규칙 정의
    값컬럼[좌표값] = 계산식
  )
```

### 3️⃣ MODEL 실습

**3-1.실습 데이터 - 연도별 제품 판매량**

```sql
WITH SALES_DATA AS
(
SELECT  '2024' AS YEAR,
         'P01'  AS PRODUCT,
          140  AS AMOUNT 
UNION ALL
SELECT  '2024' AS YEAR,
         'P02'  AS PRODUCT,
          240  AS AMOUNT 
UNION ALL          
SELECT  '2025' AS YEAR,
         'P01'  AS PRODUCT,
          150  AS AMOUNT 
UNION ALL
SELECT  '2025' AS YEAR,
         'P02'  AS PRODUCT,
          250  AS AMOUNT           
)

/* 조회결과 */
YEAR|PRODUCT|AMOUNT|
----+-------+------+
2024|P01    |   140|
2024|P02    |   240|
2025|P01    |   150|
2025|P02    |   250|          
```

**3-2. MODEL절 적용**

```sql
WITH SALES_DATA AS
(
/* 3-1 실습데이터 참조 */      
)
SELECT YEAR, PRODUCT, AMOUNT
FROM SALES_DATA
MODEL
  PARTITION BY (YEAR)    -- 연도별로 나눠서 계산
  DIMENSION BY (PRODUCT)  -- PRODUCT를 좌표로 사용
  MEASURES (AMOUNT)        -- 매출액 계산
  RULES (
    -- P03 데이터가 없으면 새로 만들고(UPSERT), P02 값 * 1.1로 적용
    AMOUNT['P03'] = AMOUNT['P02'] * 1.1
  )
ORDER BY YEAR, PRODUCT;

/* MODEL절 적용결과 */
YEAR|PRODUCT|AMOUNT|
----+-------+------+
2024|P01    |   140|
2024|P02    |   240|
2024|P03    |   264|  -- P03 데이터가 없으면 새로 만들고(UPSERT), P02 값 * 1.1로 적용
2025|P01    |   150|
2025|P02    |   250|
2025|P03    |   275|  -- P03 데이터가 없으면 새로 만들고(UPSERT), P02 값 * 1.1로 적용
```