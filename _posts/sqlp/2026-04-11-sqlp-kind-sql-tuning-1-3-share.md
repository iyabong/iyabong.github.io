---
title: 친절한 SQL 튜닝 - 1.3 SQL 공유 및 재사용
date: 2026-04-11
categories: [SQLP]
tags: [SQLP, 친절한 SQL 튜닝, SQL 공유 및 재사용]
---

## 친절한 SQL 튜닝
1. SQL 처리 과정과 I/O
  1.3 SQL 공유 및 재사용

[친절한 SQL 튜닝](https://product.kyobobook.co.kr/detail/S000001975837)
---

### 1. 종류별 테이블 생성

```sql
-- 일반 테이블
CREATE TABLE T_NORMAL (
    ID   NUMBER,
    NAME VARCHAR2(100)
);

-- 파티션 테이블 (RANGE)
CREATE TABLE T_PARTITION (
    ID          NUMBER,
    NAME        VARCHAR2(100),
    CREATED_DT  DATE
)
PARTITION BY RANGE (CREATED_DT) (
    PARTITION P2024 VALUES LESS THAN (DATE '2025-01-01'),
    PARTITION P2025 VALUES LESS THAN (DATE '2026-01-01'),
    PARTITION P2026 VALUES LESS THAN (DATE '2027-01-01')
);
```

### 2. 데이터 INSERT
  SEGMENT CREATION DEFERRED 라서

 ```sql
INSERT INTO T_PARTITION VALUES (1, 'TEST', DATE '2025-06-01');
INSERT INTO T_PARTITION VALUES (2, 'TEST', DATE '2026-06-01');

```

### 3. 파티션별 세그먼트 확인

```sql
SELECT SEGMENT_NAME, PARTITION_NAME, SEGMENT_TYPE, BYTES/1024 AS KB
FROM USER_SEGMENTS
WHERE SEGMENT_NAME IN ('T_NORMAL', 'T_PARTITION')
ORDER BY SEGMENT_NAME, PARTITION_NAME;

SEGMENT_NAME|PARTITION_NAME|SEGMENT_TYPE   |KB  |
------------+--------------+---------------+----+
T_NORMAL    |              |TABLE          |  64|
T_PARTITION |P2025         |TABLE PARTITION|8192|
T_PARTITION |P2026         |TABLE PARTITION|8192|
```

### 4. 익스텐트 조회

```sql
SELECT SEGMENT_TYPE, TABLESPACE_NAME, EXTENT_ID, FILE_ID, BLOCK_ID, BLOCKS
  FROM DBA_EXTENTS
 WHERE OWNER = 'SQLP'
   AND SEGMENT_NAME = 'T'
ORDER BY EXTENT_ID;

SEGMENT_TYPE|TABLESPACE_NAME|EXTENT_ID|FILE_ID|BLOCK_ID|BLOCKS|
------------+---------------+---------+-------+--------+------+
TABLE       |DATA           |        0|  18904|   53144|     8|
TABLE       |DATA           |        1|  18904|   53152|     8|
TABLE       |DATA           |        2|  18904|   53160|     8|
TABLE       |DATA           |        3|  18904|   53168|     8|
TABLE       |DATA           |        4|  18904|   53176|     8|
TABLE       |DATA           |        5|  18904|   53184|     8|
TABLE       |DATA           |        6|  18904|   53192|     8|
TABLE       |DATA           |        7|  18904|   53200|     8|
TABLE       |DATA           |        8|  18904|   53208|     8|
TABLE       |DATA           |        9|  18904|   53216|     8|
TABLE       |DATA           |       10|  18904|   53224|     8|
TABLE       |DATA           |       11|  18904|   53232|     8|
TABLE       |DATA           |       12|  18904|   53240|     8|
```