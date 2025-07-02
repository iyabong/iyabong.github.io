---
title: "Oracle SQL 실습 (1) - 부품 교체 이력 및 정비 이력 조회"
date: 2025-06-14
categories: [Oracle, SQL]
tags: [SQL 실습, 부품 교체, 차량 정비 이력, Oracle]
---

## 🧾 시나리오 개요

차량에 장착된 **부품의 교체 이력**과, **해당 시점 전후의 정비 작업**을 함께 조회하는 실습입니다.  
아래 조건을 만족하는 SQL을 Oracle 스타일 전통 조인 방식으로 작성합니다.

---

## 🧩 주요 테이블 구성

| 테이블명                 | 역할                  |
| ------------------------ | --------------------- |
| `CAR`                    | 차량 기준 정보        |
| `PART`                   | 부품 기준 정보        |
| `CAR_PART_CHANGE_HIST`   | 부품 교체 이력 (메인) |
| `CAR_PART_CHANGE_DETAIL` | 부품 탈거/장착 시점   |
| `CAR_MAINTAIN_HIST`      | 차량 정비 작업 이력   |

---

## 🎯 조회 조건

- `CAR_PART_CHANGE_HIST.CREATED_TIME`을 기준으로
  - **교체 직전 점검 작업 종료 시각**
  - **교체 직후 점검 작업 시작 시각** 을 함께 조회
- 이전/현재 부품의 SN 및 탈거/장착 시각도 표시
- Oracle 전통 조인 문법 사용

---

## ✅ 조회 SQL - 스칼라 서브쿼리 방식

```sql
SELECT 
  A.ID AS "부품교체이력ID",
  A.CAR_ID AS "차ID",
  C.NAME AS "차명",
  A.PART_ID AS "부품ID", 
  P.NAME AS "부품명",
  A.UNMOUNT_PART_SN AS "탈거부품SN", 
  A.MOUNT_PART_SN AS "장착부품SN", 
  A.REMARK AS "비고",
  TO_CHAR(A.CREATED_TIME, 'YYYY-MM-DD HH24:MI:SS') AS "(교체완료에 따른)이력생성시각",
  TO_CHAR(UNMOUNT.UNMOUNT_TIME, 'YYYY-MM-DD HH24:MI:SS') AS "(기존부품)탈거시각",
  TO_CHAR(MOUNT.MOUNT_TIME, 'YYYY-MM-DD HH24:MI:SS') AS "(새부품으로 교체)장착시각",

  -- 교체 직전 점검 종료 시각
  (SELECT TO_CHAR(MAX(MH.END_TIME), 'YYYY-MM-DD HH24:MI:SS')
     FROM CAR_MAINTAIN_HIST MH
    WHERE MH.CAR_ID = A.CAR_ID
      AND MH.END_TIME < A.CREATED_TIME
  ) AS "(부품 교체 전) 차량작업 완료시각",

  -- 교체 직후 점검 시작 시각
  (SELECT TO_CHAR(MIN(MH.START_TIME), 'YYYY-MM-DD HH24:MI:SS')
     FROM CAR_MAINTAIN_HIST MH
    WHERE MH.CAR_ID = A.CAR_ID
      AND MH.START_TIME > A.CREATED_TIME
  ) AS "(부품 교체 후) 차량작업 시작시각"

FROM 
  CAR_PART_CHANGE_HIST A,
  CAR C,
  PART P,
  CAR_PART_CHANGE_DETAIL UNMOUNT,
  CAR_PART_CHANGE_DETAIL MOUNT

WHERE 
  A.CAR_ID = C.ID
  AND A.PART_ID = P.ID
  AND A.UNMOUNT_PART_SN = UNMOUNT.PART_SN
  AND A.MOUNT_PART_SN = MOUNT.PART_SN

ORDER BY 
  A.CREATED_TIME;
```

---

## 🛠️ (보완) 인라인뷰 + 윈도우 함수 방식

스칼라 서브쿼리는 행마다 반복적으로 정비 이력 테이블을 조회하기 때문에,
정비 이력의 데이터 양이 많아질수록 성능 저하가 발생할 수 있습니다.

이를 개선하기 위해 `ROW_NUMBER()` 윈도우 함수와 인라인뷰를 활용해
**각 교체이력 건당 정확히 1건의 직전/직후 정비 이력을 사전에 추출**하고 조인하는 방식으로 변경하여 대량 데이터 조회시의 성능을 고려하여 보완하였습니다.

```sql
SELECT 
  A.ID AS "부품교체이력ID",
  A.CAR_ID AS "차ID",
  C.NAME AS "차명",
  A.PART_ID AS "부품ID", 
  P.NAME AS "부품명",
  A.UNMOUNT_PART_SN AS "탈거부품SN", 
  A.MOUNT_PART_SN AS "장착부품SN", 
  A.REMARK AS "비고",
  TO_CHAR(A.CREATED_TIME, 'YYYY-MM-DD HH24:MI:SS') AS "(부품교체 완료되어) 이력생성시각",
  TO_CHAR(UNMOUNT.UNMOUNT_TIME, 'YYYY-MM-DD HH24:MI:SS') AS "(기존부품)탈거시각" ,
  TO_CHAR(MOUNT.MOUNT_TIME, 'YYYY-MM-DD HH24:MI:SS') AS "(새부품으로 교체)장착시각",
  TO_CHAR(PREV.END_TIME, 'YYYY-MM-DD HH24:MI:SS') AS "(부품 교체 전) 차량작업 완료시각",
  TO_CHAR(NEXT.START_TIME, 'YYYY-MM-DD HH24:MI:SS') AS "(부품 교체 후) 차량작업 시작시각"
FROM 
  CAR_PART_CHANGE_HIST A,
  CAR C,
  PART P,
  CAR_PART_CHANGE_DETAIL UNMOUNT,
  CAR_PART_CHANGE_DETAIL MOUNT,
  (
    SELECT CHANGE_ID, CAR_ID, END_TIME, CREATED_TIME
    FROM (
      SELECT PH.ID AS CHANGE_ID,
             MH.CAR_ID,
             MH.END_TIME,
             PH.CREATED_TIME,
             ROW_NUMBER() OVER(PARTITION BY PH.ID ORDER BY MH.END_TIME DESC) AS RN
      FROM CAR_MAINTAIN_HIST MH,
           CAR_PART_CHANGE_HIST PH
      WHERE MH.CAR_ID = PH.CAR_ID
        AND MH.END_TIME < PH.CREATED_TIME
    )
    WHERE RN = 1
  ) PREV,
  (
    SELECT CHANGE_ID, CAR_ID, START_TIME, CREATED_TIME
    FROM (
      SELECT PH.ID AS CHANGE_ID,
             MH.CAR_ID,
             MH.START_TIME,
             PH.CREATED_TIME,
             ROW_NUMBER() OVER(PARTITION BY PH.ID ORDER BY MH.START_TIME ASC) AS RN
      FROM CAR_MAINTAIN_HIST MH,
           CAR_PART_CHANGE_HIST PH
      WHERE MH.CAR_ID = PH.CAR_ID
        AND MH.START_TIME > PH.CREATED_TIME
    )
    WHERE RN = 1
  ) NEXT
WHERE 
  A.CAR_ID = C.ID
  AND A.PART_ID = P.ID
  AND A.UNMOUNT_PART_SN = UNMOUNT.PART_SN
  AND A.MOUNT_PART_SN = MOUNT.PART_SN
  AND A.ID = PREV.CHANGE_ID(+)
  AND A.ID = NEXT.CHANGE_ID(+)
ORDER BY 
  A.CREATED_TIME;
```

---

## 📊 결과 예시

| 부품교체이력ID | 차명   | 부품명       | 탈거SN | 장착SN | 이력생성시각        | (부품 교체 전) 차량작업 완료시각 | (부품 교체 후) 차량작업 시작시각 |
| -------------- | ------ | ------------ | ------ | ------ | ------------------- | -------------------------------- | -------------------------------- |
| CH01           | 아반떼 | 브레이크패드 | SN001  | SN002  | 2025-01-15 10:00:00 | 2025-01-10 09:00:00              | 2025-01-20 14:00:00              |
| CH02           | 그랜저 | 엔진오일     | SN003  | SN004  | 2025-02-20 14:00:00 | 2025-02-18 11:00:00              | 2025-02-21 09:00:00              |

---


## 📂 테이블별 더미 데이터

### 🚗 CAR (차량 기준 정보)

| ID    | NAME   |
| ----- | ------ |
| CAR01 | 아반떼 |
| CAR02 | 그랜저 |

### 🔩 PART (부품 기준 정보)

| ID  | NAME         |
| --- | ------------ |
| P01 | 브레이크패드 |
| P02 | 엔진오일     |

### 📝 CAR_PART_CHANGE_HIST (부품 교체 이력)

| ID   | PART_ID | CAR_ID | UNMOUNT_PART_SN | MOUNT_PART_SN | REMARK       | CREATED_TIME        |
| ---- | ------- | ------ | --------------- | ------------- | ------------ | ------------------- |
| CH01 | P01     | CAR01  | SN001           | SN002         | 정기교체     | 2025-01-15 10:00:00 |
| CH02 | P02     | CAR02  | SN003           | SN004         | 점검 중 교체 | 2025-02-20 14:00:00 |

### 📦 CAR_PART_CHANGE_DETAIL (부품 탈거/장착 이력)

| ID  | PART_ID | PART_SN | UNMOUNT_TIME        | MOUNT_TIME          |
| --- | ------- | ------- | ------------------- | ------------------- |
| D01 | P01     | SN001   | 2024-12-01 09:00:00 |                     |
| D02 | P01     | SN002   |                     | 2025-01-15 10:00:00 |
| D03 | P02     | SN003   | 2025-02-10 08:00:00 |                     |
| D04 | P02     | SN004   |                     | 2025-02-20 14:00:00 |

### 🛠️ CAR_MAINTAIN_HIST (차량 정비 이력)

| ID  | CAR_ID | WORK_TYPE        | START_TIME          | END_TIME            | WORKER |
| --- | ------ | ---------------- | ------------------- | ------------------- | ------ |
| M01 | CAR01  | 일상점검         | 2025-01-10 08:00:00 | 2025-01-10 09:00:00 | 김정비 |
| M02 | CAR01  | 부품검사         | 2025-01-20 14:00:00 | 2025-01-20 15:00:00 | 박정비 |
| M03 | CAR02  | 상시점검         | 2025-02-18 10:00:00 | 2025-02-18 11:00:00 | 이정비 |
| M04 | CAR02  | 부품교환 후 점검 | 2025-02-21 09:00:00 | 2025-02-21 10:00:00 | 김정비 |

