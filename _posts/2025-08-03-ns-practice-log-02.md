---
title: "Nexacro + Spring 연동 실습기 (2) - 파일 다운로드 구현 및 Multipart 관련 에러 방지"
date: 2025-08-03
categories: [Nexacro]
tags: [Nexacro17, Spring MVC, FileDownload, FileDownTransfer, Multipart, 에러]
---

## 1️⃣ FileDownload, FileDownTransfer 컴포넌트 실습 과정

### 📦 실습 목표
- Nexacro 17에서 FileDownload, FileDownTransfer 컴포넌트로 파일(엑셀) 다운로드 실습
- Spring(MVC) 백엔드와 연동하여 실제 파일 응답 처리
- 파라미터 전송 방식별 동작 차이 체크

### 🛠️ 주요 실습 내용

- **넥사크로 XFDL 버튼에 스크립트 연결**
    - FileDownload: 정적 파일 직접 다운로드(`.set_url()` + `.download()`)
    - FileDownTransfer: 동적 다운로드/파라미터 포함 전송(`.download(url)` 혹은 `.setPostData()`)


- **Spring Controller 구현**
    - `void` 타입 직접 OutputStream으로 응답
    - `ResponseEntity<byte[]>` RESTful 방식 실습


- **배포(deploy) 후 톰캣에서 실제 테스트**
    - nexacro studio에서 'Deploy' 실행하여 웹 배포경로에 반영되어야 웹브라우저에서 변경사항 확인 가능
    - 웹브라우저에서 FileDownTransfer 컴포넌트의 onerror, onsucces 이벤트 캐치 불가(런타임 전용 이벤트)
    - trace()사용시, 웹브라우저에서는 console.log() 형태로 작동

---

## 2️⃣ Multipart 에러 원인 및 방지법

### ❗ 발생 가능한 에러 현상

- 넥사크로에서 FileDownTransfer 컴포넌트 사용시, `.setPostDat()` 사용시,  
  Spring에서 Multipart Casting 에러발생

---

### 🔎 원인 분석

- 넥사크로가 `.setPostData()`로 multipart/form-data 요청을 보냄
- Spring에서 **MultipartResolver**가 제대로 적용되어 있지 않거나,  
  multipart로 변환된 request를  
  다른 타입(HttpServletRequest 등)으로 잘못 캐스팅할 때 발생

---

### 🛠️ 방지/해결 방법

1. **MultipartResolver를 올바르게 등록**
    - web.xml, Java Config, 또는 Spring Boot 설정에 `multipartResolver` bean 명확히 추가

2. **컨트롤러/필터/인터셉터에서 올바른 타입으로 캐스팅**
    - multipart 요청일 때만 MultipartHttpServletRequest로 캐스팅
    - 필요시 instanceof 체크

```java
if (request instanceof MultipartHttpServletRequest) {
    MultipartHttpServletRequest multiReq = (MultipartHttpServletRequest) request;
    // 파라미터 추출 등 안전하게 처리
}
```

3. **FileDownTransfer 사용시, multipart 관련 에러 방지**
    - 다운로드 실행 전, `.clearPostDataList();` 호출
      ※ FileDownload 컴포넌트 사용 등 우회(단, 다운로드 전 파일생성 필요)