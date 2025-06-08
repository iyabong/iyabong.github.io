---
title: "Balance-Book 개발기 (1) - 프로젝트 개요"
date: 2025-05-24
categories: [balance-book]
tags: [프로젝트 소개, 주요 기능, 아키텍처 개요, 기술 스택, 개발 환경, 소스 관리, 배포, 디렉터리 구조]
---

# 💡 프로젝트 소개
**Balance-Book**은 여러 개의 카드/대출을 관리할 수 있는 **가계부 웹앱**입니다.  
- URL: [https://balance-book.vercel.app/](https://balance-book.vercel.app/)

---

## 🛠️ 주요 기능
- **카드**
  - 카드 목록
  - 카드별 충전 및 거래내역 관리  

- **대출**
  - 대출 Summary
  - 이용자별 대출/상환 히스토리

---

## 🧱 아키텍처 개요
```text
                           [React (Vercel)]
                                |
         +----------------------+----------------------+
         |                                             |
         v                                             v
 [Supabase Edge Function]                [ASP.NET Core API (Render + Railway)]
         |                                             |
 [Supabase (대출, 상환)]                 [Supabase (카드, 거래내역)]
```
---

## 🧱 기술 스택

- **프론트엔드**
  - React (JSX)

- **백엔드**
  - 대출관리: Supabase Edge Function (Serverless API)
  - 카드관리: ASP.NET Core (C#)

- **데이터베이스**:  
  - Supabase (PostgreSQL)  
  
---

## 🛠️ 개발 환경
- **개발 툴**:  
  - Visual Studio Code

- **로컬 실행**:  
  - 프론트엔드: `npm start`  
  - 백엔드: `dotnet run`

- **환경변수 관리**:  
  - `.env` 파일 사용

---

## 🧾 소스 관리

- **Git + GitHub**

  - URL:  [https://github.com/iyabong/balance-book](https://github.com/iyabong/balance-book)

---

## 🚀 배포
- **프론트엔드**

  - Vercel 연동  (GitHub push 시 자동 배포)

- **백엔드**
  - Render + Railway (Dockerfile 기반)  


---

## 🗂️ 디렉터리 구조

```bash
/frontend
  └── card/
  └── loan/
  └── history/
  └── components/
  └── App.jsx

/backend
  └── Controllers/
  └── Services/
  └── Models/
  └── Program.cs
  └── appsettings.json
```
