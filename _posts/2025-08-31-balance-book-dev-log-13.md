---
categories:
- Balance-Book
date: 2025-08-31
tags:
- Git
- Branch
- Merge
- Squash
- Release
title: Balance-Book 개발기 (13) - dev → main squash 머지 & 배포 🚀
---

`dev` 브랜치에서 검증된 내용을 `main` 브랜치에 **squash merge** 방식으로 배포까지 진행했다.

---

## 1️⃣ 브랜치 상태 확인

- `feature-blazor-resume` → `dev` 머지 완료 상태
- 이후 `dev`에서 추가된 작업까지 포함해 최종 `main` 배포 준비

```bash
git checkout dev
git pull origin dev
git log feature-blazor-resume..dev   # dev만의 커밋 확인
```

---

## 2️⃣ GitHub Pull Request 생성

- base: `main`
- compare: `dev`
- **Merge pull request** 버튼의 ▼ 선택 → **Squash and merge** 클릭  
- 커밋 메시지를 릴리즈용으로 수정

예시:
```
Release: Resume 메뉴(Blazor WASM) 추가 (2025-08-31)
- Blazor Resume 서브앱 구조 최초 커밋
- CORS 설정 보완 (Preview 도메인 대응)
- README 파일 추가
```

---

## 3️⃣ 배포 확인 🔗

- `main` 브랜치 = Production 배포  
- Resume 메뉴 `/resume/projects` 정상 동작 확인  
- 총 8개의 커밋이 1개의 커밋으로 합쳐짐 (squash 결과)

---


## 4️⃣ 배포 스크린샷 🖼️

### Squash and merged 확인
![squash-merged](/assets/img/balance-book/2025-08-31-squash-merged.png)

### Production 배포 확인
 - React Main
![react-main](/assets/img/balance-book/2025-08-31-react-main.png)
 - Blazor WASM Resume
![blazor-wasm-resume](/assets/img/balance-book/2025-08-31-blazor-wasm-resume.png)