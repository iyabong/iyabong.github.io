---
title: "[Oracle 튜닝] 실행계획 비교 - Nested Loop vs Hash Join"
date: 2025-07-02
categories: [Oracle, SQL 튜닝]
tags: [oracle, sql, tuning, execution plan, join, nested loop, hash join]
---

## 🔍 실습 목표

- `EMP` 와 `DEPT` 테이블을 조인할 때,
- **Nested Loop Join**과 **Hash Join**의 실행계획 차이를 비교한다.
- 인덱스 유무, 힌트 사용, 병렬 처리(PX) 영향도 함께 확인

---

## ✅ 실습 환경

- Oracle ADB (Autonomous Transaction Processing)
- SQL Developer
- 계정: `SCOTT`
- 기본 테이블: `EMP`, `DEPT`

---

## 🛠️ 1. 인덱스 존재 여부 확인 및 생성

```sql
-- 1-1. 인덱스 존재 여부 확인
SELECT INDEX_NAME, TABLE_NAME, COLUMN_NAME
FROM USER_IND_COLUMNS
WHERE TABLE_NAME = 'EMP';

-- 1-2. EMP.DEPTNO에 인덱스 생성
CREATE INDEX IDX_EMP_DEPTNO ON EMP(DEPTNO);
```

---

## 📊 2. 실행계획 비교

### 🔹 (1) 기본 실행계획

```sql
EXPLAIN PLAN FOR
SELECT E.ENAME, E.JOB, D.DNAME
FROM EMP E
JOIN DEPT D ON E.DEPTNO = D.DEPTNO;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

- 조인 방식: `NESTED LOOPS`
- DEPT: `TABLE ACCESS FULL`
- EMP: `INDEX RANGE SCAN (IDX_EMP_DEPTNO)` + `TABLE ACCESS BY INDEX ROWID`
- PX 병렬 처리 자동 적용됨

---

### 🔹 (2) Hash Join 힌트 적용

```sql
EXPLAIN PLAN FOR
SELECT /*+ USE_HASH(E D) */ E.ENAME, E.JOB, D.DNAME
FROM EMP E
JOIN DEPT D ON E.DEPTNO = D.DEPTNO;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

- 조인 방식: `HASH JOIN`
- EMP, DEPT 모두: `TABLE ACCESS FULL`
- PX 병렬 처리 유지
- 인덱스 무시하고 전체 테이블 스캔

---

## 📌 실행계획 비교 요약

| 항목      | 기본 실행계획  | 힌트 적용 (USE_HASH) |
| --------- | -------------- | -------------------- |
| 조인 방식 | `Nested Loops` | `Hash Join`          |
| EMP 접근  | 인덱스 사용    | Full Scan            |
| DEPT 접근 | Full Scan      | Full Scan            |
| 병렬 처리 | O              | O                    |

---

## 🧠 튜닝 포인트 정리

| 조건                         | 추천 Join 방식     |
| ---------------------------- | ------------------ |
| 소량 데이터 + 인덱스 있음    | `Nested Loop Join` |
| 대량 데이터 + 병렬 처리 고려 | `Hash Join`        |

> 옵티마이저는 데이터 양과 인덱스 유무, 병렬 환경 등을 고려하여 Join 전략을 자동 선택  
> → 힌트를 통해 강제로 제어하며 성능 실험 가능