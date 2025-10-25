---
title: "Nexacro + Spring 연동 실습기 (3) - Eclipse 프로젝트 세팅 참고 및 .gitignore 보완"
date: 2025-10-25
categories: [Nexacro]
tags: [Nexacro17, Spring MVC, Eclipse, Gitignore, 프로젝트 세팅, Tomcat]
---

## 🧭 작업 개요

이전까지의 Nexacro + Spring 연동 실습 환경을 다른 PC에서도 동일하게 구축하기 위해  
`.gitignore` 설정을 완화하고, Eclipse 서버 세팅 및 컨트롤러 호출 화면을 **스크린샷 형태로 기록**했습니다.

- GitHub Repository: [https://github.com/iyabong/ns](https://github.com/iyabong/ns)

---

## 🛠️ 1️⃣ .gitignore 보완 (완화)

기존에는 `.project`, `.classpath`, `.settings/` 등 Eclipse 설정 파일을 전부 무시했지만,  
**다른 PC에서 clone 시 프로젝트 세팅이 초기화되는 문제**가 있었음.

### ✅ 변경 요점

- Eclipse 설정 및 빌드 산출물 파일 일부를 **Git 추적 대상으로 포함**    

```diff
##########################################
# Eclipse 설정 파일 무시
##########################################
-.classpath
-.project
-.settings/
+.classpath
+.project
+.settings/

##########################################
# 빌드/배포 산출물 무시
##########################################
-/build/
-/target/
+#/build/
+#/target/
```

📁 또한, 환경 복원 참고용 자료를 보관하기 위해  
`SETTING_REF/` 폴더를 새로 만들고 **Eclipse/Tomcat 세팅 관련 캡처 이미지**를 추가함.

---

## 🖼️ 2️⃣ 참고 스크린샷

### 📌 Eclipse 서버 세팅 화면  
`SETTING_REF/eclipse_server_setting_01.png`

![SETTING_REF/eclipse_server_setting_01.png](https://raw.githubusercontent.com/iyabong/ns/refs/heads/master/SETTING_REF/eclipse_server_setting_01.png)

> Web Modules, Tomcat 로그 설정 확인  
> 서버 포트 및 모듈 매핑 상태를 기록해둠

---

### 📌 Controller & JSP 테스트 화면  
`SETTING_REF/controller_jsp.png`

![SETTING_REF/controller_jsp.png](https://raw.githubusercontent.com/iyabong/ns/refs/heads/master/SETTING_REF/controller_jsp.png)

> `HomeController.java` → `/home/hello.do`  
> “Hello from Spring MVC with JDK 1.7!!!” 출력 확인  

---

## 💬 정리

- `.gitignore` 완화로 세팅 파일 추적 → 새 환경에서도 즉시 실행 가능  
- `SETTING_REF/` 폴더에 캡처 저장 → 환경 복원 시 시각적 참고

---

## 🔗 이전 글

- [Nexacro + Spring 연동 실습기 (1) - 프로젝트 구성 및 Git 업로드](https://iyabong.github.io/posts/ns-practice-log-01/)
- [Nexacro + Spring 연동 실습기 (2) - 파일 다운로드 구현 및 Multipart 관련 에러 방지](https://iyabong.github.io/posts/ns-practice-log-02/)