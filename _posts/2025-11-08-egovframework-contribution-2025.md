---
---
title: "2025년 전자정부 표준프레임워크 컨트리뷰션 참가기"
date: 2025-11-08
categories: [eGovFramework,contribution]
tags: [전자정부프레임워크, 컨트리뷰션, 2025]
---

## 🙋‍♂️ 참여 계기

- **'내손으로 만들어가는 전자정부 표준프레임워크'** 라는 문구에 끌림
- **오픈소스 프로젝트에 기여**할 수 있는 좋은 기회라 생각하여 참여
🔗 행사 안내 페이지: [2025년 전자정부 표준프레임워크 컨트리뷰션 개최 안내](https://www.egovframe.go.kr/home/ntt/nttRead.do?pagerOffset=10&searchKey=&searchValue=&menuNo=74&bbsId=6&nttId=1914)

---

## 📝 참여 단계 소개

### 1️⃣ 💡 기여 방법 확인

- git 설치 등 환경 세팅 방법부터, 소스 수정 후 Pull Request 방법까지 동영상 가이드를 통해 확인
🔗 동영상 가이드: [전자정부 표준프레임워크 오픈소스 기여하는 방법](https://youtu.be/5QmO5LrYBM0?si=dzLGv7-syEjALG7O)


### 2️⃣ 🧿 기여 대상 기능 선정

- 선정 기능: "홈페이지 템플릿 소개" 팝업 닫기 기능
  - 선정 과정: frontend 템플릿을 실행해보던 중, **키보드 사용성 개선** 여지를 발견
🔗 저장소: [표준프레임워크 심플홈페이지 FrontEnd](https://github.com/eGovFramework/egovframe-template-simple-react)


### 3️⃣ 🔧 개선 사항
- 개선 전: 키보드로 "홈페이지 템플릿 소개" 팝업 닫기 불가
  - 키보드로 해당 팝업 열기는 가능(Tab키로 물음표 선택 후, Enter) 
- 개선 후: 
  - Tab키로 'X'버튼 지정하여 닫기 가능하게 개선
  - Esc키로 닫기 가능하게 개선

```diff
##########################################
# 수정 파일: src/js/ui.js
##########################################
    handleOpenPopup: function (e) {
      e.preventDefault();
      this.$tg.style.display = "block";
+      this.$tg.setAttribute("tabindex", "-1"); // Tab키가 기능하도록 tabindex 지정
+      this.$tg.focus({ preventScroll: true }); // Tab키가 기능하도록 focus 지정
+      this.$tg.addEventListener("keydown", this.handleEsc.bind(this)); // Esc키가 기능하도록 닫기버튼에 핸들러 추가
    },
    handleClosePopup: function (e) {
      e.preventDefault();
      this.$tg.style.display = "none";
+      this.$tg.removeEventListener("keydown", this.handleEsc.bind(this)); // 팝업 닫힐때 이벤트 핸들러 제거
+    },
+    handleEsc: function (e) {  // Esc키 이벤트 핸들러 작성
+      if (e.key === "Escape")
+        this.handleClosePopup(e);
    },
```

### 4️⃣ ✉ Pull Request & Merge

- 양식에 맞추어 수정 사유, 수정된 소스 내용, 스크린샷 등 작성하여 Pull Request
  - eGovFrameSupport 계정에 의해 검토 및 merge 수행
🔗 Pull Request: ["홈페이지 템플릿 소개" 팝업 닫기 기능(키보드 사용성) 개선 #95](https://github.com/eGovFramework/egovframe-template-simple-react/pull/95)

---

## 🏆 참여 후기

- 자기개발을 이어갈 수 있는 좋은 **동기부여**
- 행사 안내를 접하고, 기여 방법 확인부터 최종적으로 PR & Merge까지 모든 과정을 해내서 뿌뜻
- Github Id가 새겨진 피규어 트로피를 포함한 선물을 받아서 기쁨

**[피규어 트로피]**
![figure_trophy](/assets/img/egovframework-contribution/egovframework_contribution_2025_figure_trophy_iyabong.jpg)