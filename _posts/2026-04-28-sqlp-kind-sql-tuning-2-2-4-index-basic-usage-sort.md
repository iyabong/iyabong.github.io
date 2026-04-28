---
title: 친절한 SQL 튜닝 - 2.2.4 인덱스를 이용한 소트 연산 생략
date: 2026-04-28
categories: [SQLP]
tags: [SQLP, 친절한 SQL 튜닝, 인덱스, INDEX, SORT]
---

# 친절한 SQL 튜닝
2 인덱스 기본
  2.2 인덱스 기본 사용법
    2.2.4 인덱스를 이용한 소트 연산 생략
[친절한 SQL 튜닝](https://product.kyobobook.co.kr/detail/S000001975837)
---

### 0. 실습 준비
```sql
-- 쇼핑몰 주문 변경 이력 테이블 
-- PK 구성: CUSTOMER_ID + CHANGE_DATE + CHANGE_SEQ
CREATE TABLE ORDER_CHANGE_LOG (
    CUSTOMER_ID  VARCHAR2(10),
    CHANGE_DATE  VARCHAR2(8),
    CHANGE_SEQ   VARCHAR2(6),
    CONSTRAINT ORDER_CHANGE_LOG_PK PRIMARY KEY (CUSTOMER_ID, CHANGE_DATE, CHANGE_SEQ)
);

-- 데이터 입력
INSERT INTO ORDER_CHANGE_LOG (CUSTOMER_ID,CHANGE_DATE,CHANGE_SEQ) VALUES ('A001','20240101','000001');
INSERT INTO ORDER_CHANGE_LOG (CUSTOMER_ID,CHANGE_DATE,CHANGE_SEQ) VALUES ('A001','20240101','000002');
INSERT INTO ORDER_CHANGE_LOG (CUSTOMER_ID,CHANGE_DATE,CHANGE_SEQ) VALUES ('A001','20240101','000005');
INSERT INTO ORDER_CHANGE_LOG (CUSTOMER_ID,CHANGE_DATE,CHANGE_SEQ) VALUES ('A001','20240315','000001');
INSERT INTO ORDER_CHANGE_LOG (CUSTOMER_ID,CHANGE_DATE,CHANGE_SEQ) VALUES ('B002','20240210','000001');
INSERT INTO ORDER_CHANGE_LOG (CUSTOMER_ID,CHANGE_DATE,CHANGE_SEQ) VALUES ('B002','20240210','000003');
INSERT INTO ORDER_CHANGE_LOG (CUSTOMER_ID,CHANGE_DATE,CHANGE_SEQ) VALUES ('C003','20240401','000001');
INSERT INTO ORDER_CHANGE_LOG (CUSTOMER_ID,CHANGE_DATE,CHANGE_SEQ) VALUES ('D001','20240101','000001');
INSERT INTO ORDER_CHANGE_LOG (CUSTOMER_ID,CHANGE_DATE,CHANGE_SEQ) VALUES ('D001','20240101','000002');
INSERT INTO ORDER_CHANGE_LOG (CUSTOMER_ID,CHANGE_DATE,CHANGE_SEQ) VALUES ('D001','20240101','000003');
INSERT INTO ORDER_CHANGE_LOG (CUSTOMER_ID,CHANGE_DATE,CHANGE_SEQ) VALUES ('D001','20240101','000004');
INSERT INTO ORDER_CHANGE_LOG (CUSTOMER_ID,CHANGE_DATE,CHANGE_SEQ) VALUES ('D001','20240101','000005');
INSERT INTO ORDER_CHANGE_LOG (CUSTOMER_ID,CHANGE_DATE,CHANGE_SEQ) VALUES ('D001','20240101','000006');
```

### 1. 기본 Range Scan + 자동 정렬
PK 인덱스가 이미 CHANGE_SEQ 순으로 정렬되어 있어 별도 정렬 연산 불필요
```sql
SELECT *
FROM ORDER_CHANGE_LOG
WHERE CUSTOMER_ID = 'A001'
  AND CHANGE_DATE = '20240101';

CUSTOMER_ID|CHANGE_DATE|CHANGE_SEQ|
-----------+-----------+----------+
A001       |20240101   |000001    |
A001       |20240101   |000002    |
A001       |20240101   |000005    |


----------------------------------------------------------------------------------------|
| Id  | Operation        | Name                | Rows  | Bytes | Cost (%CPU)| Time     ||
----------------------------------------------------------------------------------------|
|   0 | SELECT STATEMENT |                     |     2 |    42 |     1   (0)| 00:00:01 ||
|*  1 |  INDEX RANGE SCAN| ORDER_CHANGE_LOG_PK |     2 |    42 |     1   (0)| 00:00:01 ||
----------------------------------------------------------------------------------------|
                                                                                        |
Predicate Information (identified by operation id):                                     |
---------------------------------------------------                                     |
                                                                                        |
   1 - access("CUSTOMER_ID"='A001' AND "CHANGE_DATE"='20240101')                        |
```

### 2. DESC 정렬
인덱스를 뒤에서부터 역방향으로 읽어 DESC 정렬도 추가 연산 없이 처리
```sql
SELECT *
FROM ORDER_CHANGE_LOG
WHERE CUSTOMER_ID = 'A001'
  AND CHANGE_DATE = '20240101'
ORDER BY CHANGE_SEQ DESC

CUSTOMER_ID|CHANGE_DATE|CHANGE_SEQ|
-----------+-----------+----------+
A001       |20240101   |000005    |
A001       |20240101   |000002    |
A001       |20240101   |000001    |


---------------------------------------------------------------------------------------------------|
| Id  | Operation                   | Name                | Rows  | Bytes | Cost (%CPU)| Time     ||
---------------------------------------------------------------------------------------------------|
|   0 | SELECT STATEMENT            |                     |     2 |    42 |     1   (0)| 00:00:01 ||
|*  1 |  INDEX RANGE SCAN DESCENDING| ORDER_CHANGE_LOG_PK |     2 |    42 |     1   (0)| 00:00:01 ||
---------------------------------------------------------------------------------------------------|
                                                                                                   |
Predicate Information (identified by operation id):                                                |
---------------------------------------------------                                                |
                                                                                                   |
   1 - access("CUSTOMER_ID"='A001' AND "CHANGE_DATE"='20240101')                                   |
```

### 3. 컬럼 가공 시 정렬 속성 파괴
컬럼을 가공하면 인덱스의 정렬 속성을 활용 불가 → 별도 정렬 연산 발생
WHERE절뿐 아니라 ORDER BY절에서도 동일하게 적용
```sql
SELECT *
FROM ORDER_CHANGE_LOG
WHERE CUSTOMER_ID = 'A001'
  AND CHANGE_DATE = '20240101'
ORDER BY 1 || CHANGE_SEQ DESC

CUSTOMER_ID|CHANGE_DATE|CHANGE_SEQ|
-----------+-----------+----------+
A001       |20240101   |000005    |
A001       |20240101   |000002    |
A001       |20240101   |000001    |


-----------------------------------------------------------------------------------------|
| Id  | Operation         | Name                | Rows  | Bytes | Cost (%CPU)| Time     ||
-----------------------------------------------------------------------------------------|
|   0 | SELECT STATEMENT  |                     |     2 |    42 |     2  (50)| 00:00:01 ||
|   1 |  SORT ORDER BY    |                     |     2 |    42 |     2  (50)| 00:00:01 ||
|*  2 |   INDEX RANGE SCAN| ORDER_CHANGE_LOG_PK |     2 |    42 |     1   (0)| 00:00:01 ||
-----------------------------------------------------------------------------------------|
                                                                                         |
Predicate Information (identified by operation id):                                      |
---------------------------------------------------                                      |
                                                                                         |
   2 - access("CUSTOMER_ID"='A001' AND "CHANGE_DATE"='20240101')                         |
```

### 4. MIN() / MAX() 최적화
인덱스의 첫(또는 마지막) 레코드 단 1건만 읽고 종료
D001의 데이터가 6건이어도 실제 액세스는 1번
```sql
SELECT MIN(CHANGE_SEQ)
FROM ORDER_CHANGE_LOG
WHERE CUSTOMER_ID = 'D001'
  AND CHANGE_DATE = '20240101';

MIN(CHANGE_SEQ)|
---------------+
000001         |


----------------------------------------------------------------------------------------------------|
| Id  | Operation                    | Name                | Rows  | Bytes | Cost (%CPU)| Time     ||
----------------------------------------------------------------------------------------------------|
|   0 | SELECT STATEMENT             |                     |     1 |    21 |     1   (0)| 00:00:01 ||
|   1 |  SORT AGGREGATE              |                     |     1 |    21 |            |          ||
|   2 |   FIRST ROW                  |                     |     1 |    21 |     1   (0)| 00:00:01 ||
|*  3 |    INDEX RANGE SCAN (MIN/MAX)| ORDER_CHANGE_LOG_PK |     1 |    21 |     1   (0)| 00:00:01 ||
----------------------------------------------------------------------------------------------------|
                                                                                                    |
Predicate Information (identified by operation id):                                                 |
---------------------------------------------------                                                 |
                                                                                                    |
   3 - access("CUSTOMER_ID"='D001' AND "CHANGE_DATE"='20240101')                                    |
```