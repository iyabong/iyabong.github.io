---
title: 국가공인 SQLP 자격검정 핵심노트 1 - 해시 조인 36
date: 2026-07-05
categories: [SQLP, 국가공인 SQLP 자격검정 핵심노트 1]
tags: [SQLP, 인덱스 튜닝, 조인, 힌트]
---

[2024 국가공인 SQLP 자격검정 핵심노트 1](https://product.kyobobook.co.kr/detail/S000213913597)

- 조인 튜닝
  - 해시 조인
    - **36. 실행계획 힌트**

#### 실습 준비
```sql
-- 실행계획 작성
EXPLAIN PLAN FOR

-- 실행계획 출력
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY());

```

#### 실습 스크립트
```sql
-- 1. 계약 테이블 (a)
CREATE TABLE 계약 (
    계약번호 VARCHAR2(20),
    계약명   VARCHAR2(100),
    계약일자 VARCHAR2(8),
    CONSTRAINT 계약_PK PRIMARY KEY (계약번호)
);

CREATE INDEX 계약_X1 ON 계약 (계약일자);


-- 2. 가입상품 테이블 (b)
CREATE TABLE 가입상품 (
    계약번호 VARCHAR2(20),
    상품코드 VARCHAR2(20),
    가입일자 VARCHAR2(8),
    할인률   NUMBER,
    CONSTRAINT 가입상품_PK PRIMARY KEY (계약번호, 상품코드)
);

CREATE INDEX 가입상품_X1 ON 가입상품 (가입일자);


-- 3. 가입부가상품 테이블 (c)
CREATE TABLE 가입부가상품 (
    계약번호   VARCHAR2(20),
    상품코드   VARCHAR2(20),
    부가상품코드 VARCHAR2(20),
    CONSTRAINT 가입부가상품_PK PRIMARY KEY (계약번호, 상품코드, 부가상품코드)
);


-- 4. 상품 테이블 (d)
CREATE TABLE 상품 (
    상품코드 VARCHAR2(20),
    상품명   VARCHAR2(100),
    CONSTRAINT 상품_PK PRIMARY KEY (상품코드)
);

DECLARE
    v_date_str VARCHAR2(8);
BEGIN
    -- 1. 상품 테이블 데이터 생성 (총 100건)
    -- 부가상품 조건인 'A%' 계열과 일반 상품을 섞어서 생성
    FOR i IN 1..100 LOOP
        IF i <= 30 THEN
            INSERT INTO 상품 (상품코드, 상품명) VALUES ('A' || LPAD(i, 3, '0'), '부가특약상품_' || i);
        ELSE
            INSERT INTO 상품 (상품코드, 상품명) VALUES ('PROD' || LPAD(i, 3, '0'), '일반기본상품_' || i);
        END IF;
    END LOOP;

    -- 2. 계약, 가입상품, 가입부가상품 데이터 생성 (총 5,000건의 계약)
    FOR i IN 1..5000 LOOP
        -- 날짜를 20260701 ~ 20260710 사이로 분산 시킴
        v_date_str := TO_CHAR(TO_DATE('20260701', 'YYYYMMDD') + MOD(i, 10), 'YYYYMMDD');
        
        -- 계약 테이블 생성
        INSERT INTO 계약 (계약번호, 계약명, 계약일자) 
        VALUES ('CON' || LPAD(i, 6, '0'), '고객계약_' || i, v_date_str);
        
        -- 하나의 계약당 1~2개의 상품에 가입하도록 구성
        FOR j IN 1..2 LOOP
            INSERT INTO 가입상품 (계약번호, 상품코드, 가입일자, 할인률)
            VALUES (
                'CON' || LPAD(i, 6, '0'), 
                'PROD' || LPAD(MOD(i + j, 70) + 31, 3, '0'), -- 31번 이후 일반상품 매칭
                v_date_str, -- 계약일자와 동일하게 가입일자 세팅 (또는 +1일 등 다양화 가능)
                MOD(i, 15) -- 0% ~ 14% 사이의 할인율
            );
            
            -- 특정 확률(약 40%)로 'A'로 시작하는 부가상품에도 가입되도록 유도 (문제 조건 만족용)
            IF MOD(i + j, 5) IN (0, 1) THEN
                INSERT INTO 가입부가상품 (계약번호, 상품코드, 부가상품코드)
                VALUES (
                    'CON' || LPAD(i, 6, '0'),
                    'PROD' || LPAD(MOD(i + j, 70) + 31, 3, '0'),
                    'A' || LPAD(MOD(i + j, 30) + 1, 3, '0') -- A001 ~ A030 사이의 부가상품
                );
            -- 그 외에는 B, C 계열 부가상품을 넣어 조건에서 걸러지게 만듦
            ELSIF MOD(i + j, 5) = 2 THEN
                INSERT INTO 가입부가상품 (계약번호, 상품코드, 부가상품코드)
                VALUES (
                    'CON' || LPAD(i, 6, '0'),
                    'PROD' || LPAD(MOD(i + j, 70) + 31, 3, '0'),
                    'B' || LPAD(MOD(i + j, 30) + 1, 3, '0')
                );
            END IF;
            
        END LOOP;
        
        -- 1000건마다 커밋
        IF MOD(i, 1000) = 0 THEN
            COMMIT;
        END IF;
    END LOOP;
    
    COMMIT;
END;

-- 통계 데이터 최신화
BEGIN DBMS_STATS.GATHER_TABLE_STATS(USER, '상품'); END;
BEGIN DBMS_STATS.GATHER_TABLE_STATS(USER, '가입부가상품'); END;
BEGIN DBMS_STATS.GATHER_TABLE_STATS(USER, '가입상품'); END;
BEGIN DBMS_STATS.GATHER_TABLE_STATS(USER, '계약'); END;
```

#### 힌트 없이 쿼리 수행
```sql
SELECT a.계약번호, a.계약명, b.상품코드, b.가입일자, b.할인률, c.부가상품코드, d.상품명
from   계약 a, 가입상품 b, 가입부가상품 c, 상품 d
where  a.계약일자 = :cntr_dt
and    b.계약번호 = a.계약번호
and    b.가입일자 = :ent_dt
and    c.계약번호 = b.계약번호
and    c.상품코드 = b.상품코드
and    c.부가상품코드 like 'a%'
and    d.상품코드 = c.부가상품코드;
 
---------------------------------------------------------------------------------------------------
| Id  | Operation                               | Name    | Rows  | Bytes | Cost (%CPU)| Time     |
---------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                        |         |     1 |   119 |    17   (0)| 00:00:01 |
|   1 |  NESTED LOOPS                           |         |     1 |   119 |    17   (0)| 00:00:01 |
|   2 |   NESTED LOOPS                          |         |     1 |   119 |    17   (0)| 00:00:01 |
|   3 |    NESTED LOOPS                         |         |     1 |    82 |    16   (0)| 00:00:01 |
|*  4 |     HASH JOIN                           |         |     5 |   260 |    11   (0)| 00:00:01 |
|   5 |      TABLE ACCESS BY INDEX ROWID BATCHED| 상품    |     1 |    29 |     2   (0)| 00:00:01 |
|*  6 |       INDEX RANGE SCAN                  | 상품_PK |     1 |       |     1   (0)| 00:00:01 |
|*  7 |      TABLE ACCESS FULL                  | 가입부가|   166 |  3818 |     9   (0)| 00:00:01 |
|*  8 |     TABLE ACCESS BY INDEX ROWID         | 가입상품|     1 |    30 |     1   (0)| 00:00:01 |
|*  9 |      INDEX UNIQUE SCAN                  | 가입상품|     1 |       |     0   (0)| 00:00:01 |
|* 10 |    INDEX UNIQUE SCAN                    | 계약_PK |     1 |       |     0   (0)| 00:00:01 |
|* 11 |   TABLE ACCESS BY INDEX ROWID           | 계약    |     1 |    37 |     1   (0)| 00:00:01 |
---------------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   4 - access("D"."상품코드"="C"."부가상품코드")
   6 - access("D"."상품코드" LIKE 'a%')
       filter("D"."상품코드" LIKE 'a%')
   7 - filter("C"."부가상품코드" LIKE 'a%')
   8 - filter("B"."가입일자"='20260705')
   9 - access("C"."계약번호"="B"."계약번호" AND "C"."상품코드"="B"."상품코드")
  10 - access("B"."계약번호"="A"."계약번호")
  11 - filter("A"."계약일자"='20260705')
```

#### 힌트로 실행계획 유도
```sql
select /*+ leading(b a c d ) use_hash(a) use_nl(c) use_hash(d) 
           index(b 가입상품_X1) index(a 계약_X1) swap_join_inputs(d) */
a.계약번호, a.계약명, b.상품코드, b.가입일자, b.할인률, c.부가상품코드, d.상품명
from   계약 a, 가입상품 b, 가입부가상품 c, 상품 d
where  a.계약일자 = :cntr_dt
and    b.계약번호 = a.계약번호
and    b.가입일자 = :ent_dt
and    c.계약번호 = b.계약번호
and    c.상품코드 = b.상품코드
and    c.부가상품코드 like 'a%'
and    d.상품코드 = c.부가상품코드;
 
----------------------------------------------------------------------------------------------------
| Id  | Operation                              | Name      | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                       |           |     2 |   238 |   615   (0)| 00:00:01 |
|*  1 |  HASH JOIN                             |           |     2 |   238 |   615   (0)| 00:00:01 |
|   2 |   TABLE ACCESS BY INDEX ROWID BATCHED  | 상품      |     1 |    29 |     2   (0)| 00:00:01 |
|*  3 |    INDEX RANGE SCAN                    | 상품_PK   |     1 |       |     1   (0)| 00:00:01 |
|   4 |   NESTED LOOPS                         |           |    87 |  7830 |   613   (0)| 00:00:01 |
|*  5 |    HASH JOIN                           |           |   526 | 35242 |    87   (0)| 00:00:01 |
|   6 |     TABLE ACCESS BY INDEX ROWID BATCHED| 가입상품  |  1000 | 30000 |    55   (0)| 00:00:01 |
|*  7 |      INDEX RANGE SCAN                  | 가입상품_X|  1000 |       |     5   (0)| 00:00:01 |
|   8 |     TABLE ACCESS BY INDEX ROWID BATCHED| 계약      |   500 | 18500 |    32   (0)| 00:00:01 |
|*  9 |      INDEX RANGE SCAN                  | 계약_X1   |   500 |       |     2   (0)| 00:00:01 |
|* 10 |    INDEX RANGE SCAN                    | 가입부가상|     1 |    23 |     1   (0)| 00:00:01 |
----------------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   1 - access("D"."상품코드"="C"."부가상품코드")
   3 - access("D"."상품코드" LIKE 'a%')
       filter("D"."상품코드" LIKE 'a%')
   5 - access("B"."계약번호"="A"."계약번호")
   7 - access("B"."가입일자"='20260705')
   9 - access("A"."계약일자"='20260705')
  10 - access("C"."계약번호"="B"."계약번호" AND "C"."상품코드"="B"."상품코드" AND "C"."부가상품코드" LIKE 'a%')
       filter("C"."부가상품코드" LIKE 'a%')
```