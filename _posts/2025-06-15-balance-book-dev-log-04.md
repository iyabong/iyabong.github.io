---
title: "Balance-Book 개발기 (4) - .NET 백엔드와 Oracle ADB 연결 설정"
date: 2025-06-15
categories: [balance-book]
tags: [Oracle Cloud, Oracle Autonomous Database, 백엔드 연결, ODP.NET, EF Core, VS Code, Wallet]
---

Balance-Book 프로젝트에서 Oracle Autonomous Database(ADB)에 .NET 백엔드를 연결하기 위한 설정 과정을 정리했습니다.

---

## ✅ 주요 목표

- Oracle Cloud Infrastructure (OCI) CLI 설정
- Oracle Autonomous DB Wallet 구성
- VS Code, .NET 백엔드 애플리케이션에서 Oracle ADB 연결
- EF Core로 Oracle 테이블 연동

---

## 🧭 설정 요약

### 1️⃣ OCI CLI 및 API Key 구성


- `OCI CLI` 설치
- `oci setup config` 실행하여 `~/.oci/config` 생성
https://docs.oracle.com/en-us/iaas/Content/API/SDKDocs/cliinstall.htm 
- Tenancy OCID, User OCID, Region, API Key 입력
https://docs.oracle.com/en-us/iaas/Content/API/SDKDocs/cliconfigure.htm 
- 공개키(`oci_api_key_public.pem`)를 콘솔 > 사용자 > API Key에 등록

https://docs.oracle.com/en-us/iaas/Content/Identity/access/to_upload_an_API_signing_key.htm
```bash
# 예시 위치
C:\Users\[사용자명]\.oci\
├── config
├── oci_api_key.pem
└── oci_api_key_public.pem
```

---

### 2️⃣ Oracle Autonomous DB Wallet 설정

- DB > Database connection > Wallet 다운로드
- `C:\Users\[사용자명]\Oracle\network\admin\[데이터베이스명]` 폴더에 압축 해제
- 포함 파일: `tnsnames.ora`, `sqlnet.ora`, `cwallet.sso` 등
https://docs.oracle.com/en/cloud/paas/autonomous-database/serverless/adbsb/connect-download-wallet.html#GUID-B06202D2-0597-41AA-9481-3B174F75D4B1
---

### 3️⃣ VS Code 환경 구성

- Oracle Cloud VS Code 확장 설치
- `.oci/config` 자동 인식
- Autonomous Database 확인 및 SQL Worksheet 접속 가능

https://www.oracle.com/database/technologies/appdev/dotnet/adbdotnetquickstarts.html#second-option-tab
---

### 4️⃣ .NET 백엔드 연결 설정

#### 📂 `appsettings.json`

```json
{
  "Oracle": {
    "WalletDirectory": "C:\\Users\\[사용자명]\\Oracle\\network\\admin\\[데이터베이스명]"
  },
  "ConnectionStrings": {
    "DefaultConnection": "User Id=[User Id];Password=[패스워드];Data Source=[Data Source];Connection Timeout=30;"
  }
}
```

#### ⚙️ `Program.cs`

```csharp
var builder = WebApplication.CreateBuilder(args);

var walletPath = builder.Configuration["Oracle:WalletDirectory"];
OracleConfiguration.TnsAdmin = walletPath;
OracleConfiguration.WalletLocation = walletPath;

var connectionString = builder.Configuration.GetConnectionString("DefaultConnection");

builder.Services.AddDbContext<BalanceBookContext>(options =>
    options.UseOracle(connectionString));
```