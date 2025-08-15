---
title: "Balance-Book 개발기 (10) - Blazor 기반 Resume 관리 프로그램 추가 & Vercel Preview 배포"
date: 2025-08-15
categories: [Balance-Book]
tags: [Blazor, Resume, Vercel, Preview 배포, Git 브랜치 전략, VS2022]
---

Balance-Book 웹앱에 이력 관리 기능(메뉴) 추가 개발을 위한 밑작업을 했다.
C#, .NET 및 Visual Studio 역략 향상을 위해 **Blazor WASM**으로 기술스택을 선택.  
Git 브랜치 전략을 보완(main외에 dev, feature- 브랜치 추가) 및 Vercel Preview 확인. 🌿

---

## 1️⃣ 브랜치 전략 정리

**브랜치 구조**
- `main` → Production 배포 (Vercel 기본 URL)
- `dev` → 개발 통합 브랜치 (Vercel Preview URL)
- `feature-blazor-resume` → Resume 기능 개발 전용 브랜치 (Vercel Preview URL)

모든 브랜치는 GitHub 연동으로 push 시 자동 배포되며, Preview URL은 브랜치별로 고유하게 발급됨.

브랜치 생성 예시:
```bash
# dev 브랜치 생성 및 원격 반영
git checkout -b dev
git push origin dev

# Resume 기능 개발 브랜치 생성
git checkout -b feature-blazor-resume
git push origin feature-blazor-resume
```

---

## 2️⃣ VS 2022에서 Blazor WASM 프로젝트 추가

**위치**  
`blazor/resume/BalanceBook.Resume.Client`

**프로젝트 생성 시 설정**
- 템플릿: Blazor WebAssembly App
- Target Framework: `.NET 9.0`
- `StaticWebAssetBasePath`를 `resume`으로 변경하여 `/resume` 경로로 접근 가능하게 설정

`BalanceBook.Resume.Client.csproj` 일부:
```xml
<Project Sdk="Microsoft.NET.Sdk.BlazorWebAssembly">

  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <StaticWebAssetBasePath>resume</StaticWebAssetBasePath>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.Components.WebAssembly" Version="9.0.5" />
    <PackageReference Include="Microsoft.AspNetCore.Components.WebAssembly.DevServer" Version="9.0.5" PrivateAssets="all" />
  </ItemGroup>

</Project>
```

---

## 3️⃣ Blazor 빌드 결과를 React public 폴더로 자동 복사

Blazor의 빌드 산출물(`wwwroot`)을 React의 `frontend/public/resume` 폴더로 복사하도록 설정.  
VS 2022 Publish 과정에 아래 타겟 추가

```xml
<Target Name="CopyToFrontend" AfterTargets="Publish">
  <!-- public/resume 폴더 없으면 생성 -->
  <MakeDir Directories="$(SolutionDir)frontend\public\resume"
           Condition="!Exists('$(SolutionDir)frontend\public\resume')" />
  <!-- wwwroot → public/resume 로 동기화 복사 -->
  <Exec Command='robocopy "$(PublishDir)wwwroot\." "$(SolutionDir)frontend\public\resume\." /MIR /NFL /NDL /NJH /NJS'
        IgnoreExitCode="true" />
</Target>
```

> `/MIR` 옵션은 폴더 구조를 동기화 복사하고, `/NFL /NDL /NJH /NJS`는 불필요한 로그 출력을 최소화

---

## 4️⃣ Vercel Preview 배포 확인

[https://balance-book-git-feature-blazor-resume-iyabongs-projects.vercel.app](https://balance-book-git-feature-blazor-resume-iyabongs-projects.vercel.app/)