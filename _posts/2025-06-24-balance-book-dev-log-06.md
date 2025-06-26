---
title: "Balance-Book 개발기 (6) - 백엔드 배포환경에 ADB 연결"
date: 2025-06-24
categories: [balance-book]
tags: [Oracle, OCI, Railway, Render, Docker, 백엔드 배포, 환경변수]
---

## 📝 작업 개요

.NET 백엔드를 Railway에 배포하고, OCI ADB <-> Railway <-> API 호출 테스트 완료.  

Railway와 비슷한 방식으로 Render에 적용 - 2025.06.26

---

## 🔧 주요 작업 내용

### 📦 1. Oracle Wallet을 Base64로 변환하여 Docker 이미지에 통합

- `Wallet 파일(.zip)` → base64로 인코딩 후 환경변수 또는 Secret File로 등록

#### ✅ Railway
- Railway 환경변수(Variables)에 base64 문자열 등록
- Docker 컨테이너 내 `entrypoint.sh`에서 복호화 & unzip

```bash
echo "$WALLET_B64" | base64 -d > /app/wallet.zip
unzip -o /app/wallet.zip -d /app/wallet
```

#### ✅ Render
- Secret File 업로드 가능. 단, 바이너리 파일은 불가. 
- `.zip`을 base64로 변환한 `.txt` 파일을 Secret File로 등록
- `entrypoint_render.sh`파일로 복호화 및 unzip 수행
  ※ Render 배포용 .sh 파일 분리 

```bash
cat /etc/secrets/Wallet_A_b64.txt | base64 -d > /app/wallet.zip
unzip /app/wallet.zip -d /app/wallet
```

---

### 🔐 2. .NET 백엔드 연결 문자열 환경변수로 분리

```csharp
var user = Environment.GetEnvironmentVariable("");
var pw = Environment.GetEnvironmentVariable("");
var dataSource = builder.Configuration.GetConnectionString("dataSource");

var connectionString = $"User Id={user};Password={pw};Data Source={dataSource};Connection Timeout=30;";
```

- 민감 정보는 Railway/Render의 환경변수 기능으로 관리
- 연결 문자열 일부만 appsettings.json에 정의하고, 나머지는 런타임 조합

---

### 🌍 3. 외부 IP 확인 및 출력

- `curl ifconfig.me` 를 `entrypoint.sh` 또는 `.NET 앱 시작 시점`에 추가하여 로그에 외부 IP 출력

```bash
echo "🌍 현재 외부 IP:"
curl ifconfig.me || wget -qO- ifconfig.me
```

---

### 🛡️ 4. Oracle Cloud ACL(Access Control List) 설정

- Oracle Autonomous DB → Network 설정에서
- `Access type: Allow secure access from specified IPs and VCNs`
- ACL 항목에 외부 배포 서버의 IP를 수동 등록

#### ⚠️ 주의
- Railway / Render는 재배포 시마다 외부 IP가 바뀔 수 있음
- 그에 따라 Oracle ACL에서 접속 차단 발생 가능
  
#### ✅ 대안
- Oracle REST API로 현재 IP를 자동 등록하는 스크립트 구성
- GitHub Actions 또는 앱 실행 시점에 IP → ACL 업데이트 자동화

---

### 🚀 5. 배포 후, API 호출하여 정상 응답 확인

- Railway
  [https://balance-book-production.up.railway.app/api/routine/calendar?startDate=20250606&endDate=20250611](https://balance-book-production.up.railway.app/api/routine/calendar?startDate=20250606&endDate=20250611)  
  
- Render
  [https://balance-book.onrender.com/api/routine/calendar?startDate=20250606&endDate=20250611](https://balance-book.onrender.com/api/routine/calendar?startDate=20250606&endDate=20250611)

```json
[
  {
    "date": "2025-06-08T00:00:00",
    "dayOfWeek": "SUN",
    "routineChecks": [
      {
        "templateId": "T100",
        "status": "PROG",
        "checkTime": "2025-06-08T00:00:00"
      }
    ]
  },
  ...
]
```

---

## 🔮 향후 계획

- Oracle ACL 자동화 스크립트(Curl + REST API) 구성
- 루틴체크기능 프론트 <-> 백엔드 연결