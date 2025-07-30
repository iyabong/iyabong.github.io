---
title: "[Excel 실습] 부서별 노트북 리스트 관리"
date: 2025-07-30
categories: [Excel]
tags: [엑셀 실습, 시트 참조, 기준정보, IF, VLOOKUP]
---

## 🧾 실습 개요

부서별로 배정된 노트북 목록을 관리하는 시나리오를 기반으로,  
**노트북의 유형(LAPTOP_TYPE)**을 자동 분류하는 수식을 실습했습니다.

---

## 🗂️ 데이터 시트 구성

| 시트명         | 내용                            |
| -------------- | ------------------------------- |
| `LaptopMaster` | LAPTOP_ID별 TYPE 사전 정의      |
| `LaptopList`   | 노트북 ID, 부서명 목록 (입력값) |

---

## 🎯 목표

`LaptopList` 시트의 `C열 (LAPTOP_TYPE)`에 다음 기준으로 값을 자동으로 채우도록 합니다:

1. `LAPTOP_ID`가 `LaptopMaster` 시트에 존재 → 해당 TYPE 반환  
2. 부서명(DEPT_NAME)이 존재하면 `"TO BE ASSIGNED"`  
3. 부서명도 없으면 `"NO TYPE"`  

---

## 🧠 사용 함수

### ✅ `IF` 중첩 버전

```excel
=IF(COUNTIF(LaptopMaster!A:A,A3),
    VLOOKUP(A3,LaptopMaster!A:C,3,FALSE),
    IF(B3<>"","TO BE ASSIGNED","NO TYPE")
)
```

### ✅ `IFS` 함수 버전 (Excel 2016+)

```excel
=IFS(COUNTIF(LaptopMaster!A:A,A3),VLOOKUP(A3,LaptopMaster!A:C,3,FALSE),B3<>"","TO BE ASSIGNED",TRUE,"NO TYPE")
```

> `IFS`는 조건 순서를 엄격하게 따라가며, 마지막 `TRUE`는 else 역할을 합니다.

---

## 💡 포인트

- `IFS` 함수는 구버전 Excel에서는 `#NAME?` 오류가 발생할 수 있음  
- 호환성을 위해 `IF` 중첩을 쓰는 것이 안전  
- `A:A` 같은 열 전체 참조는 편리하지만 성능 이슈 있을 수 있음  
- 시트 참조는 `시트명!셀` 형식으로, `!`는 시트와 셀을 구분

---

## 📝 정리

이번 실습을 통해 사내 장비 자산 관리와 같은 반복 업무를  
엑셀 수식으로 효율화할 수 있습니다.  
VLOOKUP, IF/IFS 함수는 실무에서 정말 자주 쓰이며,  
조건 분기 로직을 명확하게 설계하는 것이 핵심입니다.

---

## 📂 관련 실습 파일

- 📎 [실습파일 다운로드](../assets/files/2025-07-30-excel-practice-if-vlookup_LaptopList.xlsx)