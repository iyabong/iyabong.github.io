---
title: "[Oracle 튜닝] 실행계획(결과) 분석 흐름 정리"
date: 2025-07-06
categories: [Oracle, SQL 튜닝]
tags: [oracle, sql, tuning, execution plan, dbms_xplan]
---

Oracle ADB 환경에서 **실제 실행계획을 확인하고 분석하는 전체 흐름**을 정리한다.

---

## ✅ 실습 목표

- 실행된 SQL에 대해 V$SQL에서 SQL_ID를 조회하고,
- DBMS_XPLAN.DISPLAY_CURSOR로 **실제 실행 통계(ACTUAL STATS)** 를 확인하는 실무 흐름 구성

---

## ✅ 전체 흐름 요약

```text
[1] SCOTT 권한 부여 (V$SQL, V$SQL_PLAN 등)
        ↓
[2] 쿼리 작성: 힌트 + 식별자 포함
        ↓
[3] SQL 실행 → V$SQL에 등록됨
        ↓
[4] V$SQL에서 SQL_TEXT LIKE '%쿼리 식별자%'로 SQL_ID 검색
        ↓
[5] DBMS_XPLAN.DISPLAY_CURSOR(SQL_ID, NULL, 'ALLSTATS LAST +MEMSTATS +IOSTATS') 실행
```

---

## 🔐 [1] SCOTT 권한 부여 (관리자 계정에서 실행)

```sql
-- 성능 뷰 접근을 위한 권한 부여
GRANT SELECT ON V$SQL TO SCOTT;
GRANT SELECT ON V$SQL_PLAN TO SCOTT;
GRANT SELECT ON V$SQL_PLAN_STATISTICS_ALL TO SCOTT;

-- 또는 아래와 같이 통합 권한 부여
GRANT SELECT_CATALOG_ROLE TO SCOTT;
```

> ADB 환경: ADMIN 계정에서 실행

---

## 🧾 [2] 쿼리 작성 (힌트 + 식별자 주석 포함)

```sql
SELECT /*+ GATHER_PLAN_STATISTICS USE_HASH(E D) FULL(E) FULL(D) */
       /* ID:empDeptJoin */
       E.ENAME, E.JOB, D.DNAME
FROM EMP E
JOIN DEPT D ON E.DEPTNO = D.DEPTNO;
```

- `GATHER_PLAN_STATISTICS` → 실제 실행계획 통계 수집
- `USE_HASH`, `FULL` → 조인 방식 강제
- `/* ID:empDeptJoin */` → V$SQL 조회를 위한 식별자 주석
   
   ※ (ADB) `SELECT`가 식별자 주석보다 선행하도록 작성하여야 V$SQL에서 조회됨.

---

## ⚙️ [3] SQL 실행 → 자동으로 V$SQL에 등록됨

쿼리를 실행하면 해당 SQL은 Shared Pool의 커서 캐시에 등록되고,
`V$SQL`에 SQL_TEXT와 함께 저장됨.

---

## 🔍 [4] V$SQL에서 SQL_ID 조회

```sql
SELECT SQL_ID, SQL_TEXT
FROM V$SQL
WHERE SQL_TEXT LIKE '%ID:empDeptJoin%'
  AND SQL_TEXT NOT LIKE '%FROM V$SQL%';  -- 자기 자신 쿼리는 제외
ORDER BY LAST_ACTIVE_TIME DESC
```
- 실행 직후에 커서 캐시에서 조회해야만 SQL_ID를 확인할 수 있음
![SQL_ID 조회](/assets/img/oracle-sql-tuning/select_sqlid_sqltext_from_vsql.png)

---

## 📊 [5] 실행계획 + 통계 조회

```sql
SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(
  'SQL_ID_HERE',                        -- SQL_ID
  NULL,                                 -- child_number (NULL이면 최근 것)
  'ALLSTATS LAST +MEMSTATS +IOSTATS'    -- 통계 옵션
));
```
- `A-Rows`, `Buffers`, `Elapsed`, `OMem`, `Temp` 등 실제 실행 통계 확인 가능
- `+MEMSTATS`	메모리 사용량 (PGA, workarea 등)
- `+IOSTATS`	I/O 통계 (physical read/write)  
  
```sql
SQL_ID  [8zd5ps0hx7gbc], child number 0
-------------------------------------
SELECT /*+ GATHER_PLAN_STATISTICS USE_HASH(E D) FULL(E) FULL(D) */      
   /* ID:selectEmpDept */ E.ENAME, E.JOB, D.DNAME FROM EMP E JOIN DEPT 
D ON E.DEPTNO = D.DEPTNO
 
Plan hash value: 615168685
 
----------------------------------------------------------------------------------------------------------------
| Id  | Operation          | Name | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
----------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT   |      |      1 |        |     13 |00:00:00.01 |      70 |       |       |          |
|*  1 |  HASH JOIN         |      |      1 |     13 |     13 |00:00:00.01 |      70 |  1695K|  1695K| 1095K (0)|
|   2 |   TABLE ACCESS FULL| DEPT |      1 |      4 |      4 |00:00:00.01 |       6 |       |       |          |
|   3 |   TABLE ACCESS FULL| EMP  |      1 |     13 |     13 |00:00:00.01 |       6 |       |       |          |
----------------------------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   1 - access("E"."DEPTNO"="D"."DEPTNO")
 
Note
-----
   - This is SQL Plan Management Test Plan
 
```

---

## ⚖️ EXPLAIN PLAN vs. GATHER_PLAN_STATISTICS 비교

| 항목               | `EXPLAIN PLAN FOR`    | `GATHER_PLAN_STATISTICS` + 실행 |
| ------------------ | --------------------- | ------------------------------- |
| 실행 여부          | ❌ 실행 안 함 (예측용) | ✅ 실제 쿼리 실행                |
| PLAN 소스          | 옵티마이저 예측       | 실제 실행 결과                  |
| `A-Rows`           | ❌ 없음                | ✅ 실제 행 수 포함               |
| `Buffers`, `Reads` | ❌ 없음                | ✅ 포함 가능                     |
| 사용 뷰            | PLAN_TABLE            | V$SQL_PLAN_STATISTICS_ALL 등    |
| 추천 용도          | 사전 분석용           | 실전 튜닝/비교/정확한 진단용    |

> 실무 튜닝에는 `GATHER_PLAN_STATISTICS` + `DBMS_XPLAN.DISPLAY_CURSOR` 방식 권장

