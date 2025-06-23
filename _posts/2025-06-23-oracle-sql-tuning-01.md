---
title: "[Oracle 튜닝] SQL 실행계획 분석"
date: 2025-06-23
categories: [Oracle, SQL 튜닝]
tags: [oracle, sql, tuning, execution plan, scott]
---

Oracle Autonomous Database(ADB) 환경에서 실습 스키마 구성 후,
기본 SQL 튜닝 실습 수행

---

## ✅ 실습 환경

- Oracle ADB (Autonomous Transaction Processing)
- SQL Developer
- 실습 계정: `SCOTT`
- 기본 테이블: `EMP`, `DEPT`

---

## ✅ 실습 스키마(SCOTT)

```sql
-- 1. 사용자 생성 (기본 테이블스페이스: DATA)
CREATE USER SCOTT IDENTIFIED BY [PASSWORD]
DEFAULT TABLESPACE DATA
TEMPORARY TABLESPACE TEMP
QUOTA UNLIMITED ON DATA;

-- 2. 권한 부여
GRANT CONNECT, RESOURCE TO SCOTT;
GRANT CREATE SESSION, CREATE TABLE, CREATE VIEW, CREATE SEQUENCE, CREATE PROCEDURE TO SCOTT;
```
- 실습용 테이블 스크립트 [https://gist.github.com/julianhyde/686145afd0af9fdf61f7b57c5a7299b7](https://gist.github.com/julianhyde/686145afd0af9fdf61f7b57c5a7299b7)

---

## ✅ 실행계획 확인

```sql
/* 1. 실행계획 저장 */
EXPLAIN PLAN FOR
SELECT E.ENAME, E.JOB, D.DNAME, E.SAL
FROM EMP E
JOIN DEPT D ON E.DEPTNO = D.DEPTNO;

/* 2. 저장된 실행계획 조회 */
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

```sql
Plan hash value: 3321496967
 
---------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name     | Rows  | Bytes | Cost (%CPU)| Time     |    TQ  |IN-OUT| PQ Distrib |
---------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |          |    13 |   442 |     4   (0)| 00:00:01 |        |      |            |
|   1 |  PX COORDINATOR       |          |       |       |            |          |        |      |            |
|   2 |   PX SEND QC (RANDOM) | :TQ10000 |    13 |   442 |     4   (0)| 00:00:01 |  Q1,00 | P->S | QC (RAND)  |
|*  3 |    HASH JOIN          |          |    13 |   442 |     4   (0)| 00:00:01 |  Q1,00 | PCWP |            |
|   4 |     TABLE ACCESS FULL | DEPT     |     4 |    52 |     2   (0)| 00:00:01 |  Q1,00 | PCWP |            |
|   5 |     PX BLOCK ITERATOR |          |    13 |   273 |     2   (0)| 00:00:01 |  Q1,00 | PCWC |            |
|   6 |      TABLE ACCESS FULL| EMP      |    13 |   273 |     2   (0)| 00:00:01 |  Q1,00 | PCWP |            |
---------------------------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   3 - access("E"."DEPTNO"="D"."DEPTNO")
 
Note
-----
   - automatic DOP: Computed Degree of Parallelism is 4
```

---

## ✅ 1. 기본 JOIN 실행계획

```sql
SELECT E.ENAME, E.JOB, D.DNAME, E.SAL
FROM EMP E
JOIN DEPT D ON E.DEPTNO = D.DEPTNO;
```

- `HASH JOIN` 사용됨
- `EMP`, `DEPT` 모두 `TABLE ACCESS FULL`
- 병렬 처리(PX COORDINATOR) 자동 활성화됨
- DEPT.DEPTNO는 PK라 인덱스 있음에도 풀스캔된 이유:
  → 데이터 소량 + PX 병렬 처리의 영향

---

## ✅ 2. COUNT(*) vs COUNT(컬럼)

```sql
SELECT COUNT(*) FROM DEPT;
```

- `INDEX FULL SCAN (PK_DEPT)` 사용됨

```sql
SELECT COUNT(DNAME) FROM DEPT;
```

- `TABLE ACCESS FULL` 발생
- 이유: `DNAME`은 NULL 여부 체크 필요 → 인덱스 없음

---

## ✅ 3. 힌트 사용

```sql
SELECT /*+ FULL(DEPT) */ COUNT(*) FROM DEPT;
```

- 인덱스 우선 선택을 강제로 **테이블 풀스캔으로 유도**
- 실행계획 결과: `TABLE ACCESS FULL`

---

## ✅ 4. ADB의 PX 처리 영향

- PX 관련 연산자 (`PX COORDINATOR`, `PX SEND`, `PX BLOCK`) 항상 포함됨
- 옵티마이저가 병렬 처리 기준으로 실행계획을 잡기 때문에
  **로컬 Oracle과 다른 PLAN 구조**가 나올 수 있음

> 힌트 (`FULL`, `INDEX`, `NO_PARALLEL` 등)는 잘 적용됨

---

## ✅ 정리

| 쿼리                    | 액세스 방식         | 힌트    | 특징                      |
| ----------------------- | ------------------- | ------- | ------------------------- |
| `COUNT(*)`              | `INDEX FULL SCAN`   | ❌       | 인덱스만으로 처리 가능    |
| `COUNT(DNAME)`          | `TABLE ACCESS FULL` | ❌       | 컬럼 null 확인 필요       |
| `COUNT(*) + FULL(DEPT)` | `TABLE ACCESS FULL` | ✅       | 힌트로 인덱스 무시        |
| `JOIN EMP-DEPT`         | `HASH JOIN`         | PX 자동 | 소량 데이터라도 PX 실행됨 |