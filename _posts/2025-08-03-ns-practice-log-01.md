---
title: "Nexacro + Spring 연동 실습기 (1) - 프로젝트 구성 및 Git 업로드"
date: 2025-08-03
categories: [Nexacro]
tags: [Nexacro17, Spring MVC, Git, 프로젝트 구성, 실습]
---

## 🔧 프로젝트 개요

Nexacro17과 Spring(Java 1.7 기반)을 연동하는 실습 프로젝트를 구성했습니다.  
IDE는 각각 Nexacro Studio와 Eclipse를 사용하며, Git으로 버전 관리를 시작했습니다.

---

## 📁 디렉터리 구조

```plaintext
ns/
├── nexacro/             ← Nexacro Studio 개발 소스 (.xfdl, .xadl 등)
├── WebContent/
│   ├── WEB-INF/         ← Spring 설정 및 JSP 포함
│   └── nexacro-deploy/  ← Deploy 결과물 (자동 생성, Git 무시 대상)
├── src/                 ← Java 소스 코드 (Controller, Service 등)
├── .gitignore           ← Eclipse + Nexacro 환경 최적화
├── .project, .classpath
```

---

## 🚫 .gitignore 설정

다음 항목들은 Git 추적 대상에서 제외했습니다:

- Eclipse 설정 파일  
  `.project`, `.classpath`, `.settings/`
- 넥사크로 Deploy 결과물  
  `WebContent/nexacro-deploy/`
- 임시 파일 및 자동 생성물  
  `*.xfdl$`, `*.geninfo`, `$temporary$/`

---

## ✅ GitHub 업로드 절차

1. Git 초기화  
   ```bash
   git init
   ```
2. `.gitignore` 생성 및 커밋  
3. GitHub Desktop에서 첫 커밋 → `Publish repository`
4. GitHub 저장소:  
   [https://github.com/iyabong/ns](https://github.com/iyabong/ns)

---

## 🖥️ 실행 환경

- Nexacro Studio 17
- Eclipse JEE (Dynamic Web Project)
- Apache Tomcat 8.x
- Java 1.7
- Spring MVC

---

## 🧪 화면 호출 테스트

- `DispatcherServlet` URL 패턴을 `/` → `*.do`로 수정하여 정적 리소스(index.html) 404 문제 해결
- `nexacro-deploy/_web_/index.html` 호출 성공 확인
- Nexacro 화면이 정상 로딩됨

---

## ✅ 향후 계획

- Nexacro → Spring Controller 연동 실습
- Excel파일 다운로드 실습 (FileDownload, FileDownloadTransfer 컴포넌트 사용)

---

## 🔗 관련 사이트

- Nexacro 공식 기술지원: [https://support.tobesoft.co.kr/Support](https://support.tobesoft.co.kr/Support)
- Eclipse IDE (2020-06 버전): [https://www.eclipse.org/downloads/packages/release/2020-06/r](https://www.eclipse.org/downloads/packages/release/2020-06/r)
- Apache Tomcat 아카이브: [https://archive.apache.org/dist/tomcat/](https://archive.apache.org/dist/tomcat/)