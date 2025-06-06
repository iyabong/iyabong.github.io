---
title: "Balance-Book 리팩토링"
date: 2025-06-06
categories: [balance-book]
tags: [프로젝트 리팩토링, fallback 처리, API 상태 점검, Dockerfile, .NET]
---

Balance-Book 프로젝트명 변경 및 가용성 증대 작업

---

## 1️⃣ 프로젝트명 리팩토링 (`BalanceBook.CardApi` → `BalanceBook`)

### 🎯 목적
- 백엔드 프로젝트에 카드API 외 기능 확장을 반영하기 위해 `BalanceBook`으로 변경

### 🧭 변경 사항

1. **폴더 및 파일 이름 변경**
   - 프로젝트 폴더: `BalanceBook.CardApi/` → `BalanceBook/`
   - `.csproj`: `BalanceBook.CardApi.csproj` → `BalanceBook.csproj`

2. **솔루션 파일 수정**
   - `.sln`파일 프로젝트 경로 및 참조명 변경
   - VS Code 전역 검색(`Ctrl + Shift + F`)으로 `"CardApi"` 키워드 일괄 수정  

3. **Dockerfile 수정**
   - `WORKDIR`: `/src/BalanceBook.CardApi` → `/src/BalanceBook`
   - `ENTRYPOINT`: `["dotnet", "BalanceBook.CardApi.dll"]` → `["dotnet", "BalanceBook.dll"]`


---

## 2️⃣ Fallback 기능 추가

### 🎯 목적
- 주 서버 이상발생시, 예비 서버 호출하여 고가용성 확보

### 🧭 구현 개요

1. Health Check API 추가
```CS
        [HttpGet("health")]
        public Task<IActionResult> Health() => Task.FromResult<IActionResult>(Ok("Healthy"));       
```

2. 백엔드 서비스 2군데 배포
   - [Railway](https://railway.com/)
   - [Render](https://render.com/)     


3. Fallback 수행
   - 주 서버 Health Check 수행
      (응답이 없으면) → 예비 서버로 Fallback 

---