---
title: Balance-Book 개발기 (9) - Azure 환경에 백엔드 배포 🚀
date: 2025-07-20
categories: [Balance-Book]
tags: [Azure, Docker, App Service, Backend, 배포]
---

Balance-Book 백엔드 Docker 이미지를 Azure App Service**로 배포하는 과정을 정리했다.

> 전체 흐름은 Docker 기반 빌드 → Docker Hub 업로드 → Azure에서 이미지 pull → 환경변수 설정 순으로 진행되었다.

---

## 1. App Service 생성 🏗️

Azure Portal에서 App Service를 생성할 때 **Container 기반**으로 설정한다.

### 주요 구성

- **출시 방식**: Docker Container  
- **운영 체제**: Linux  
- **리전**: Korea Central  
- **플랜**: B1 (1.75GB RAM / 1vCPU)  
- **도커 이미지 소스**: Docker Hub (공개)

🎯 기본 도메인
`https://balance-book-backend-xxxxx.azurewebsites.net`

---

## 2. Azure 전용 Dockerfile 추가 🐳

Azure는 **80번 포트**로만 연결되므로 별도의 Dockerfile을 구성했다.  
Wallet 복원 및 ASP.NET Core 포트 설정도 포함.

### 📄 Dockerfile.azure

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . ./
WORKDIR /src/BalanceBook
RUN dotnet publish -c Release -o /out

FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
RUN apt-get update && \
    apt-get install -y unzip curl && \
    rm -rf /var/lib/apt/lists/*
EXPOSE 80

COPY --from=build /out .
COPY entrypoint_azure.sh .
RUN chmod +x entrypoint_azure.sh
ENTRYPOINT ["./entrypoint_azure.sh"]
```

### 📄 entrypoint_azure.sh

```bash
#!/bin/bash
export ASPNETCORE_URLS=http://+:80
dotnet BalanceBook.dll
```

---

## 3. Docker Build & Push 과정 🛠️

로컬에서 이미지를 빌드하고 Docker Hub로 push한다.  
Azure는 이 이미지를 pull해서 컨테이너로 실행한다.

```bash
docker build -f Dockerfile.azure -t iyabong/balance-book-backend .
docker push iyabong/balance-book-backend
```

> Docker Hub 계정: `iyabong`

---

## 4. Azure에서 배포 및 테스트 ✅

환경변수는 `.env` 대신 Azure Portal에서 직접 설정한다.

### 🔐 등록한 환경변수

- Supabase 연결 관련 변수
- OCI ADB 연결 관련 변수(+BASE64 Encoded Wallet파일)
 

### 🔍 테스트 결과

- `{기본 도메인}/api/card` → 정상 데이터 응답 확인

---

## ✅ 향후 계획

- 프론트엔드(Vercel)와 연결
- Docker build & push 자동화(Github Actions)

