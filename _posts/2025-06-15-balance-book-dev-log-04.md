---
title: "Balance-Book 개발기 (4) - Oracle Cloud와 .NET 백엔드 연결"
date: 2025-06-15
categories: [balance-book]
tags: [Oracle Cloud, 백엔드 연결, ODP.NET, EF Core, VS Code, Wallet]
---

Balance-Book 프로젝트에서 Oracle Cloud에 호스팅된 DB와 .NET 백엔드를 직접 연결하기 위한 설정 과정을 정리했습니다.

---

## ✅ 주요 목표

- Oracle Cloud Infrastructure (OCI) CLI 설정
- Oracle Autonomous DB Wallet 구성
- VS Code, .NET 백엔드에서 Oracle 연결 성공
- EF Core로 Oracle 테이블 연동

---

## 🧭 설정 요약

### 1️⃣ OCI CLI 및 API Key 구성

- `oci setup config` 실행하여 `~/.oci/config` 생성
- Tenancy OCID, User OCID, Region, API Key 입력
- 공개키(`oci_api_key_public.pem`)를 콘솔 > 사용자 > API Key에 등록

```bash
# 예시 위치
C:\Users\1\.oci\
├── config
├── oci_api_key.pem
└── oci_api_key_public.pem
```

---

### 2️⃣ Oracle Autonomous DB Wallet 설정

- DB > Database connection > Wallet 다운로드
- `C:\Users\1\Oracle\network\admin\A` 폴더에 압축 해제
- 포함 파일: `tnsnames.ora`, `sqlnet.ora`, `cwallet.sso` 등

---

### 3️⃣ VS Code 환경 구성

- Oracle Cloud VS Code 확장 설치
- `.oci/config` 자동 인식
- Autonomous Database 확인 및 SQL Worksheet 접속 가능

---

### 4️⃣ .NET 백엔드 연결 설정

#### 📂 `appsettings.json`

```json
{
  "Oracle": {
    "WalletDirectory": "C:\\Users\\1\\Oracle\\network\\admin\\A"
  },
  "ConnectionStrings": {
    "DefaultConnection": "User Id=BALANCE_BOOK;Password=패스워드;Data Source=a_tp;Connection Timeout=30;"
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

---

## 🎯 접속 성공 및 오류 해결 요약

| 오류 메시지 | 원인 | 해결 방법 |
|-------------|------|------------|
| ORA-50000: Connection request timed out | Wallet 설정 누락 또는 잘못된 연결 문자열 | TnsAdmin 경로 지정, `a_tp` TNS 이름 사용 |
| ORA-00904: 부적합한 식별자 | EF Core 컬럼 매핑 오류 | `[Column("COL_NAME")]` 지정 또는 `HasColumnName` 사용 |

---

## 🧠 오늘 배운 점 요약

- Oracle은 TNS 방식이 가장 안전하며, Full Descriptor는 오류 유발 가능
- `appsettings.Local.json` 등 개인 설정 파일은 `.gitignore` 처리
- EF Core에서 Oracle은 대소문자 컬럼명을 구분하므로 매핑 주의

---

이제 Oracle에 저장된 루틴 데이터를 기반으로 백엔드 API를 확장해나갈 수 있습니다.  
다음 작업은 `/routine/calendar-status`, `/routine/check` 등의 API 설계가 될 예정입니다.
