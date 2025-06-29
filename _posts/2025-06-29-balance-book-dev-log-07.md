---
title: "Balance-Book 개발기 (7) - Oracle ADB ACL 구성 전략"
date: 2025-06-29
categories: [balance-book]
tags: [Oracle, ACL, IP, CIDR, Railway, Render, 클라우드]
---

## 📝 개요

Oracle Autonomous Database(ADB)의 ACL(Access Control List)을 구성하면서  
Render, Railway 배포 환경의 IP 특성과 변동성에 맞춰 **IP 단일 등록 + CIDR 대역 등록을 병행**한 방식 정리.

---

## 🧪 테스트 요약

### 🔁 Restart / (Re)deploy 후 외부 IP 변동 여부

| 플랫폼  | Restart       | (Re)deploy      |
| ------- | ------------- | --------------- |
| Render  | ❌ (변경 없음) | ❌ (변경 없음)   |
| Railway | ❌ (변경 없음) | ✅ (매번 변경됨) |

→ **Railway는 CIDR 대역 등록 필수**, Render는 단일 IP 등록으로 가능 

---

## 🌐 외부 IP 확인 방법
1. Docker ENTRYPOINT에 IP 출력 스크립트 작성
```bash
echo "🌍 현재 외부 IP:"
curl ifconfig.me || wget -qO- ifconfig.me
```

2. Restart 또는 Deploy 후, 로그 확인
   - 서비스 선택 → Logs

---

## 🔒 ACL 구성 방식

### 📍 정적 IP (집 근처 카페 등 노트북 사용처)

- 직접 확인한 공용 IP를 ACL에 단일 IP로 등록
- Oracle ADB → Access Control List → `IP address`로 등록  
  

### 📍 동적 IP 대역 (Railway 등)

- Railway Deploy시마다 IP가 변동되긴 하나, `AAA.BBB.CCC.DDD` IP의 'DDD' 영역만 변동되는 점 확인 
- `/24` CIDR 블록으로 IP 변동영역 대응할 수 있도록 등록

```text
AAA.BBB.CCC.0/24
```
- Oracle ADB → Access Control List → `CIDR block`으로 등록


### ✅ OCI CLI를 이용한 ACL LIST Update 방법
```bash
oci db autonomous-database update \
  --autonomous-database-id <YOUR_DB_OCID> \
  --region ap-seoul-1 \
  --whitelisted-ips '[
    "A.B.C.0/24",
    ...
  ]'
```
> 주의: 기존 ip 목록 덮어쓰기 됨.
---

## 🔒 보안 측면 고려

- CIDR 범위는 가능한 좁게 설정 (예: `/24`)
- 향후 IP가 완전히 다른 대역으로 바뀔 경우 수동 갱신 필요
- 자동화 필요 시 curl + oci CLI 스크립트로 구성 가능

---

## 🧩 참고: 기존 연결 흐름

> 자세한 연결 과정은 [개발기 (6)](https://iyabong.github.io/posts/balance-book-dev-log-06/) 참고

---

## ✅ 결론

- IP가 바뀌는 플랫폼이라도 CIDR을 잘 활용하면 충분히 안정적인 연결 가능
- 필요 시 ACL LIST Update 자동화 구현 예정 (CLI 기반)
