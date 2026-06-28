---
title: "[Oracle 튜닝] 자전거 대여 및 유지보수 스키마 구성"
date: 2025-07-13
categories: [Oracle, SQL 튜닝]
tags: [SQL 실습, 튜닝, 대용량, Oracle]
---

## 🧾 실습 개요

대용량 자전거 대여 및 부품 교체 이력을 다루는 `BIKER` 스키마를 기반으로,
SQL 성능 분석 및 튜닝 실습을 진행합니다.

본 포스팅에서는 실습 환경 구성을 소개합니다.

---

## 📂 스키마명

- `BIKER`

---

## 🧩 주요 테이블 및 컬럼 구성

```text
📌 BIKE
  - ID (PK)
  - MODEL_ID
  - NAME

📌 BIKE_MODEL
  - ID (PK)
  - NAME

📌 BIKE_PART
  - ID (PK)
  - NAME

📌 BIKE_PART_CHANGE_RECORD
  - ID (PK)
  - BIKE_ID
  - PART_ID
  - ATTACH_TIME
  - DETACH_TIME
  - REMARK

📌 BIKE_PART_CHANGE_LOG
  - ID (PK)
  - CREATED_TIME
  - WORKER_ID
  - ATTACH_PART_RECORD_ID
  - DETACH_PART_RECORD_ID  

📌 BIKE_LOG
  - ID (PK)
  - BIKE_ID
  - SERVICE_TYPE  -- 'RENT' 또는 'MAINTENANCE'
  - START_TIME
  - END_TIME
```

---

---

## 📦 BIKER 실습 데이터 생성 순서

1. `BIKE_MODEL` (모델 20종)
2. `BIKE` (자전거 2000대)
3. `BIKE_PART` (부품 100종)
4. `BIKE_PART_CHANGE_RECORD` (교체이력 2만건)
5. `BIKE_PART_CHANGE_LOG` (탈거+장착 로그 총 4만건)
6. `BIKE_LOG` (100만건 + MAINTENANCE 1만건)


```sql
-- 1. BIKE_MODEL: 모델 20종
BEGIN
  FOR i IN 1..20 LOOP
    INSERT INTO BIKE_MODEL (ID, NAME)
    VALUES ('M' || LPAD(i, 2, '0'), 'Model_' || i);
  END LOOP;
  COMMIT;
END;
/

-- 2. BIKE: 자전거 2000대
BEGIN
  FOR i IN 1..2000 LOOP
    INSERT INTO BIKE (ID, MODEL_ID, NAME)
    VALUES (
      'B' || LPAD(i, 4, '0'),
      'M' || LPAD(TRUNC(DBMS_RANDOM.VALUE(1, 21)), 2, '0'),
      'Bike_' || i
    );
    IF MOD(i, 500) = 0 THEN COMMIT; END IF;
  END LOOP;
  COMMIT;
END;
/

-- 3. BIKE_PART: 부품 100종
BEGIN
  FOR i IN 1..100 LOOP
    INSERT INTO BIKE_PART (ID, NAME)
    VALUES ('P' || LPAD(i, 3, '0'), 'Part_' || i);
  END LOOP;
  COMMIT;
END;
/

-- 4. BIKE_PART_CHANGE_RECORD - 교체 이력 2만 건 (7/7일)
BEGIN
  FOR i IN 1..20000 LOOP
    DECLARE
      v_attach_time DATE := TO_DATE('2025-07-07 08:00:00', 'YYYY-MM-DD HH24:MI:SS') + DBMS_RANDOM.VALUE * (8/24);
      v_detach_time DATE := v_attach_time + DBMS_RANDOM.VALUE * (1/24);
    BEGIN
      INSERT INTO BIKE_PART_CHANGE_RECORD (
        ID, BIKE_ID, PART_ID, ATTACH_TIME, DETACH_TIME, CREATE_TIME, UPDATE_TIME, REMARK
      ) VALUES (
        'PCR' || LPAD(i, 6, '0'),
        'B' || LPAD(TRUNC(DBMS_RANDOM.VALUE(1, 2001)), 4, '0'),
        'P' || LPAD(TRUNC(DBMS_RANDOM.VALUE(1, 101)), 3, '0'),
        v_attach_time,
        v_detach_time,
        v_attach_time,
        v_detach_time,
        'CHANGE_FINISHED'
      );
    END;
  END LOOP;
  COMMIT;
END;
/

-- 5. BIKE_PART_CHANGE_LOG - 교체 로그 4만 건 (2건/RECORD)
BEGIN
  FOR rec IN (
    SELECT ID, BIKE_ID, PART_ID, ATTACH_TIME, DETACH_TIME
    FROM BIKE_PART_CHANGE_RECORD
    WHERE TRUNC(ATTACH_TIME) = TO_DATE('2025-07-07', 'YYYY-MM-DD')
  ) LOOP
    -- 5.1. 탈거 로그
    INSERT INTO BIKE_PART_CHANGE_LOG (
      ID, BIKE_ID, PART_ID, DETACH_PART_RECORD_ID, CREATED_TIME, WORKER_ID, PREV_USED_KM, USED_KM, REMARK
    ) VALUES (
      SEQ_CHANGE_LOG_ID.NEXTVAL,
      rec.BIKE_ID,
      rec.PART_ID,
      rec.ID,
      rec.DETACH_TIME,
      'WORKER' || LPAD(TRUNC(DBMS_RANDOM.VALUE(1, 11)), 2, '0'),
      TRUNC(DBMS_RANDOM.VALUE(500, 5000)),
      TRUNC(DBMS_RANDOM.VALUE(100, 1000)),
      '자동생성: 탈거'
    );

    -- 5.2. 장착 로그
    INSERT INTO BIKE_PART_CHANGE_LOG (
      ID, BIKE_ID, PART_ID, ATTACH_PART_RECORD_ID, CREATED_TIME, WORKER_ID, USED_KM, REMARK
    ) VALUES (
      SEQ_CHANGE_LOG_ID.NEXTVAL,
      rec.BIKE_ID,
      rec.PART_ID,
      rec.ID,
      rec.ATTACH_TIME,
      'WORKER' || LPAD(TRUNC(DBMS_RANDOM.VALUE(1, 11)), 2, '0'),
      0,
      '자동생성: 장착'
    );
  END LOOP;
  COMMIT;
END;
/

-- 6. BIKE_LOG - 자전거 운영 로그 100만 건

-- 6.1. 시퀀스 생성
CREATE SEQUENCE SEQ_BIKE_LOG START WITH 1 INCREMENT BY 1;

-- 6.2. 100만 건 INSERT (PL/SQL 루프)
BEGIN
  FOR i IN 1..1000000 LOOP
    INSERT INTO BIKE_LOG (
      ID,
      CREATE_TIME
    ) VALUES (
      SEQ_BIKE_LOG.NEXTVAL,
      TRUNC(SYSDATE - DBMS_RANDOM.VALUE(0, 365))  -- 최근 1년 내 날짜
    );

    -- 1만건마다 커밋
    IF MOD(i, 10000) = 0 THEN
      COMMIT;
    END IF;
  END LOOP;
  COMMIT;
END;
/

-- 6.3. SERVICE_TYPE 랜덤 분포 설정 (RENT 95%, MAINTENANCE 5%)
-- 6.3.1. 임시 컬럼 생성
ALTER TABLE BIKE_LOG ADD TEMP_RND NUMBER;

-- 6.3.2. 무작위 값 부여
UPDATE BIKE_LOG
SET TEMP_RND = DBMS_RANDOM.VALUE;

-- 6.3.3. 서비스유형 분기
UPDATE BIKE_LOG
SET SERVICE_TYPE = CASE
  WHEN TEMP_RND <= 0.05 THEN 'MAINTENANCE'
  ELSE 'RENT'
END;

-- 6.3.4. 임시 컬럼 제거
ALTER TABLE BIKE_LOG DROP COLUMN TEMP_RND;

COMMIT;
```

---

## 🔹 BIKE_MODEL 샘플

| ID  | NAME     | TYPE       |
| --- | -------- | ---------- |
| M01 | Model_1  | 하이브리드 |
| M02 | Model_2  | 스포츠     |
| M03 | Model_3  | 전기       |
| ... | ...      | ...        |
| M20 | Model_20 | 스포츠     |

---

## 🔹 BIKE 샘플

| ID    | MODEL_ID | STATUS   |
| ----- | -------- | -------- |
| B0001 | M13      | ACTIVE   |
| B0011 | M17      | INACTIVE |
| B0021 | M02      | ACTIVE   |
| ...   | ...      | ...      |

---

## 🔹 BIKE_PART 샘플

| ID   | NAME    | MAX_USAGE_KM | MAX_USAGE_DAY |
| ---- | ------- | ------------ | ------------- |
| P001 | Part_1  | 516          | 250           |
| P002 | Part_2  | 628          | 98            |
| ...  | ...     | ...          | ...           |
| P020 | Part_20 | 938          | 218           |

---

## 🔹 BIKE_PART_CHANGE_RECORD 샘플

| ID        | BIKE_ID | PART_ID | ATTACH_TIME         | DETACH_TIME         | REMARK          |
| --------- | ------- | ------- | ------------------- | ------------------- | --------------- |
| PCR000187 | B0237   | P088    | 2025-07-07 11:56:06 | 2025-07-07 12:53:20 | CHANGE_FINISHED |
| PCR000188 | B0236   | P011    | 2025-07-07 15:59:51 | 2025-07-07 16:24:53 | CHANGE_FINISHED |
| ...       | ...     | ...     | ...                 | ...                 | ...             |

---

## 🔹 BIKE_PART_CHANGE_LOG 샘플

| ID  | PART_ID | BIKE_ID | ATTACH_PART_RECORD_ID | DETACH_PART_RECORD_ID | CREATED_TIME        | WORKER_ID | USED_KM | REMARK |
| --- | ------- | ------- | --------------------- | --------------------- | ------------------- | --------- | ------- | ------ |
| 193 | P044    | B0430   | PCR000282             | NULL                  | 2025-07-07 09:28:28 | WORKER07  | 0       | 장착   |
| 194 | P036    | B1641   | NULL                  | PCR000283             | 2025-07-07 13:02:23 | WORKER01  | 183     | 탈거   |
| ... | ...     | ...     | ...                   | ...                   | ...                 | ...       | ...     | ...    |

---

## 🔹 BIKE_LOG 샘플

| ID  | BIKE_ID | USER_ID | START_TIME          | END_TIME            | DISTANCE_KM | CREATE_TIME         | SERVICE_TYPE |
| --- | ------- | ------- | ------------------- | ------------------- | ----------- | ------------------- | ------------ |
| 359 | B1251   | U395    | 2025-07-07 11:17:42 | 2025-07-07 11:42:42 | 0.68        | 2025-07-07 11:42:42 | RENT         |
| 376 | B1232   | U272    | 2025-07-07 09:18:19 | 2025-07-07 09:38:19 | 1.65        | 2025-07-07 09:38:19 | MAINTENANCE  |

---

- `BIKE_PART_CHANGE_RECORD`은 모두 CHANGE_FINISHED 상태
- `BIKE_PART_CHANGE_LOG`는 각 RECORD당 2건 생성됨 (장착 + 탈거)
- SERVICE_TYPE은 'RENT', 'MAINTENANCE' 2종으로 구분됨

---

## ✅ 조회 실습 SQL

```sql
SELECT  
  R.ID,
  R.BIKE_ID,
  R.PART_ID,
  R.ATTACH_TIME,
  R.DETACH_TIME, 
  R.REMARK,
  D.WORKER_ID AS DETACH_WORKER,
  (SELECT MAX(BL.END_TIME)
     FROM BIKE_LOG BL
    WHERE BL.BIKE_ID = D.BIKE_ID
      AND BL.END_TIME < D.CREATED_TIME
  ) AS PREV_RENT_END_TIME
FROM
  BIKE_PART_CHANGE_RECORD R,
  BIKE_PART_CHANGE_LOG D,
  BIKE_PART_CHANGE_LOG A
WHERE 1=1
  AND D.DETACH_PART_RECORD_ID = R.ID
  AND D.DETACH_PART_RECORD_ID = A.ATTACH_PART_RECORD_ID 
  AND D.CREATED_TIME BETWEEN TO_DATE('20250707110000', 'YYYYMMDDHH24MISS')
                        AND TO_DATE('20250707110959', 'YYYYMMDDHH24MISS');
```