---
categories:
- Balance-Book
date: 2025-08-17
tags:
- Git 브랜치 전략
- Vercel Preview
- Railway
- CORS
- 환경변수
title: Balance-Book 개발기 (11) - feature-blazor-resume → dev 병합 & DEV
  환경 배포 정상화
---

`feature-blazor-resume` 브랜치를 `dev` 브랜치에 병합한 뒤 DEV 환경 배포까지 확인. 🚀

------------------------------------------------------------------------

## 1️⃣ feature → dev PR & Merge

-   `feature-blazor-resume` 브랜치에작업 내용 commit 및  GitHub에 push
-   GitHub에서 Pull Request 생성 → `dev` 브랜치로 병합
-   Merge 후 `dev` 브랜치 기준 Vercel Preview 배포가 트리거됨

``` bash
# feature-blazor-resume 브랜치 작업 완료 후 push
git add .
git commit -m "~~~"
git push origin feature-blazor-resume

# GitHub에서 PR 생성 → dev 브랜치에 Merge
```

------------------------------------------------------------------------

## 2️⃣ DEV 환경 변수 분리

Vercel Settings → **Environment Variables**에서 Production과 Preview를
분리해서 관리

``` plaintext
# 예시: Vercel 환경변수 설정
API_BASE_URL=https://balance-book-prod.up.railway.app   # Production Target
API_BASE_URL=https://balance-book-dev.up.railway.app    # Preview Target
```

-   **Production**: 실제 서비스용 (main 브랜치)
-   **Preview**: 개발/테스트용 (dev 브랜치 및 feature 브랜치)

------------------------------------------------------------------------

## 3️⃣ CORS 정책 수정 (Program.cs)

DEV 환경에서 API 호출 시 CORS 에러가 발생 → 허용 Origin 수정으로
해결했다.\
특히 `dev-balance-book.vercel.app`와 Preview 해시
도메인(`-iyabongs-projects.vercel.app`)을 추가.

``` csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy(name: MyAllowSpecificOrigins,
        policy =>
        {
            policy
                .SetIsOriginAllowed(origin =>
                    origin == "http://localhost:3000"
                    || origin == "https://balance-book.vercel.app"          // ✅ Production
                    || origin == "https://dev-balance-book.vercel.app"      // ✅ Dev 고정 도메인
                    || origin.EndsWith("-iyabongs-projects.vercel.app")     // ✅ Vercel Preview 해시 도메인
                )
                .AllowAnyHeader()
                .AllowAnyMethod();
        });
});
```

적용 후 Railway에 재배포 → Vercel Preview에서 정상 동작 확인.

------------------------------------------------------------------------

## 4️⃣ DEV 작동 확인 완료

-   dev 환경(Vercel Preview)에서 Railway 백엔드 API 정상 응답 확인
-   환경변수 Preview/Production 분리 확인
-   Merge 후 자동 배포 트리거 정상 작동
-   CORS 문제 해결 후 프런트에서 API 호출 성공

------------------------------------------------------------------------

## ✨ 배포단게별 URL 예시

-   Production : [https://balance-book.vercel.app/](https://balance-book.vercel.app/)
-   Dev : [https://dev-balance-book.vercel.app/](https://dev-balance-book.vercel.app/)
-   Feature : [https://balance-book-git-feature-blazor-resume-iyabongs-projects.vercel.app/](https://balance-book-git-feature-blazor-resume-iyabongs-projects.vercel.app/)
  ※ feature preview URL은 branch명에 따라 자동 생성.