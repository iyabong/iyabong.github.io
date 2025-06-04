---
title: "오라클 PL/SQL - 2025.06.04"
date: 2025-06-04
categories: [oracle, plsql]
tags: [PL/SQL, 급여 갱신, Procedure]
---

## 프로시저 개요
지정 부서 사원들의 급여를 조정하는 프로시저

---

### 프로시저 코드
 - (입력한) 부서의 사원들 급여를 => (입력한) 배율로 조정
```sql
create or replace PROCEDURE UPDATE_SALARY_BY_DEPT (
    P_DNAME         IN VARCHAR2,    -- 부서명
    P_MULTIPLIER    IN NUMBER,      -- 조정배율
    P_UPD_EMP_CNT   OUT NUMBER      -- 급여 갱신된 사원 수
)
IS
    V_DEPTNO    DEPT.DEPTNO%TYPE;
BEGIN
    -- 1. 부서명으로 부서번호 조회
    SELECT DEPTNO INTO V_DEPTNO
    FROM DEPT
    WHERE DNAME = UPPER(P_DNAME)
    AND ROWNUM = 1;

    -- 2. 급여 갱신
    UPDATE EMP
    SET SAL = SAL * P_MULTIPLIER
    WHERE DEPTNO = V_DEPTNO;

    -- 3. 갱신된 행 수 반환
    P_UPD_EMP_CNT := SQL%ROWCOUNT;
END;
```

### 사용(실행) 예

- 영업팀의 급여를 1.1배로 갱신

```sql
DECLARE
    V_COUNT NUMBER;
BEGIN
    UPDATE_SALARY_BY_DEPT('SALES', 1.1, V_COUNT);
    DBMS_OUTPUT.PUT_LINE(V_COUNT || '명 급여 갱신');
END;
```