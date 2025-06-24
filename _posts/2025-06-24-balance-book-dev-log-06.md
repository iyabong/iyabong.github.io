---
title: "Balance-Book 개발기 (6) - 백엔드 배포환경에 ADB 연결"
date: 2025-06-24
categories: [balance-book]
tags: [Oracle, OCI, Railway, Docker, 백엔드 배포, 환경변수]
---

## 작업 개요

.NET 백엔드를 Railway에 배포하고, OCI ADB <-> Railway <-> API 호출 테스트 완료. 

---

## 주요 작업 내용

### 1. Oracle Wallet을 Base64로 변환하여 Docker 이미지에 포함

- `Wallet 파일(.zip)` → base64로 인코딩 후 Railway 환경변수로 등록
- Docker 컨테이너 내 `entrypoint.sh`에서 복호화 & unzip

```bash
echo "$WALLET_B64" | base64 -d > /app/wallet.zip
unzip -o /app/wallet.zip -d /app/wallet
```

---

### 2. .NET 백엔드 연결 문자열 환경변수로 분리

```csharp
var user = Environment.GetEnvironmentVariable("");
var pw = Environment.GetEnvironmentVariable("");
var dataSource = builder.Configuration.GetConnectionString("dataSource");

var connectionString = $"User Id={user};Password={pw};Data Source={dataSource};Connection Timeout=30;";
```

- 민감 정보는 Railway의 Variables 탭에서 설정
- 연결 문자열 일부만 appsettings.json에 정의하고, 나머지는 런타임 조합

---

### 3. Railway Deploy Logs에서 외부 IP 확인

- `curl ifconfig.me` 를 entrypoint.sh에 추가
- 로그에서 실제 Railway 배포 서버의 외부 IP 확인 가능

```bash
echo "🌍 현재 외부 IP:"
curl ifconfig.me || wget -qO- ifconfig.me
```

---

### 4. Oracle Cloud ACL(Access Control List) 설정

- Oracle Autonomous DB → Network 설정에서
- `Access type: Allow secure access from specified IPs and VCNs`
- ACL 항목에 Railway 외부 IP를 등록

---

### 5. 배포 확인

- Railway 배포 URL에서 API 정상 응답 확인
  [https://balance-book-production.up.railway.app/api/routine/calendar?startDate=20250606&endDate=20250611](https://balance-book-production.up.railway.app/api/routine/calendar?startDate=20250606&endDate=20250611)

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

## 향후 계획

- Render에도 적용 & 테스트
  [https://render.com/](https://render.com/)
- 루틴체크기능 프론트 <-> 백엔드 연결
