---
title: 차계부 프로젝트 (1) - 구상 및 시뮬레이션
date: 2026-02-16
categories: [차량 데이터 수집]
tags: [OBD2, upstash, Redis, Supabase, Telegram, Python, Serverless]
---

## 💡 프로젝트 소개
출퇴근 용도로 주로 쓰는 차(Kia Forte Hybrid Lpi) 운행데이터를 수집하여,
운행일지나 특이사항 알림을 받아볼 수 있는 프로그램 개발

---

## 🛠️ 아키텍처

1.  **Car (OBD-II Scanner)**: 차량 진단 포트에서 속도, RPM, 온도 데이터 추출
 [Vgate iCar Pro 2S](https://vgatemall.com/products-detail/i-80/)
2.  **Edge Device (Torque 앱)**: Python 스크립트로 데이터 수집 + GPS 정보 결합
 [Torque Pro](https://play.google.com/store/apps/details?id=org.prowl.torque&pli=1)
3.  **Buffer (Upstash Redis)**: Serverless Redis에 데이터를 임시 저장 (Cloudflare Worker가 가져가기 전까지)
 [upstash](https://upstash.com/)
4.  **Notification (Telegram)**: 주행 시작/종료 시 리포트 발송

---

## 1. Upstash (Serverless Redis) 도입

* **Region**: AWS Tokyo (`ap-northeast-1`) - 한국과 가장 가까워서 선택.
* **전송 전략 (Batching)**:
    * Upstash 무료 플랜은 하루 10,000 명령어로 제한됨.
    * 0.1초마다 쏘면 금방 터짐.
    * **해결:** 메모리 버퍼(`[]`)에 50개를 모았다가 한 번에 전송(`BATCH_SIZE = 50`)하는 방식으로 최적화.

---

## 2. 텔레그램 봇 (Telegram Bot) 연동

* **@BotFather**로 봇 생성 -> Token 획득
* `getUpdates`로 내 Chat ID 확인
* 기능: 채팅방에 알림 메시지 발송
    * 프로그램 시작 시: "주행 데이터 수집 시작" 알림
    * 프로그램 종료 시: "주행 리포트(거리, 시간, 연비)" 요약 발송

---

## 3. Python 수집기 구현 (`driver.py`)

`obd` 라이브러리로 차와 통신하고, `requests`로 UPSTASH에 쏘는 구조


### 💻 코드

```python
import obd
import time
import json
import requests
import random
from datetime import datetime

# ==========================================
# 1. 설정 (Configuration)
# ==========================================

# [upstash]
UPSTASH_URL = "UPSTASH_URL" # Redis > Details
UPSTASH_TOKEN = "UPSTASH_TOKEN" # Redis > Details
# [Telegram]
TELEGRAM_TOKEN = "TELEGRAM_TOKEN" # @BotFather > /newbot
CHAT_ID = "CHAT_ID" # https://core.telegram.org/api


# 데이터 전송 주기
BATCH_SIZE = 50  # 50개 모아서 전송
IS_SIMULATION = False # 초기값

# ==========================================
# 2. 통신 함수들
# ==========================================
def send_telegram(msg):
    """텔레그램 메시지 전송"""
    url = f"https://api.telegram.org/bot{TELEGRAM_TOKEN}/sendMessage"
    payload = {"chat_id": CHAT_ID, "text": msg}
    try:
        requests.post(url, json=payload)
    except Exception as e:
        print(f"❌ 텔레그램 실패: {e}")

def send_to_upstash(data_list):
    """Upstash로 데이터 전송 (핵심!)"""
    try:
        # 리스트를 JSON 문자열로 변환
        payload = json.dumps(data_list)
        
        headers = {
            "Authorization": f"Bearer {UPSTASH_TOKEN}",
            "Content-Type": "application/json"
        }
        
        # 실제 전송
        response = requests.post(UPSTASH_URL, headers=headers, data=payload)
        
        if response.status_code == 200:
            return True
        else:
            print(f"🔥 Upstash 에러({response.status_code}): {response.text}")
            return False
    except Exception as e:
        print(f"🔥 전송 실패: {e}")
        return False

# ==========================================
# 3. 메인 코드
# ==========================================
print("🚗 프로그램 시작...")

# OBD 연결 시도
try:
    connection = obd.OBD()
    if connection.is_connected():
        print("✅ 차량 연결 성공! (Real Mode)")
        IS_SIMULATION = False
    else:
        raise Exception("Not connected")
except:
    print("⚠️ 차량 연결 실패 -> 시뮬레이션 모드 (가짜 데이터)")
    IS_SIMULATION = True
    connection = None

send_telegram(f"🟢 주행 시작!\n모드: {'실전' if not IS_SIMULATION else '시뮬레이션'}")

SESSION_ID = datetime.now().strftime("%Y%m%d_%H%M%S")
buffer = []
start_time = time.time()
total_dist = 0

print(f"🚀 수집 시작 (Session: {SESSION_ID})")
print("   (점 '.' 하나당 데이터 50개 전송됨)")

try:
    while True:
        # --- [1] 데이터 수집 ---
        now = time.time()
        
        # 가짜 GPS (나중에 실제 GPS 코드로 교체)
        gps = {
            "lat": 37.5665 + (random.random() * 0.001),
            "lon": 126.9780 + (random.random() * 0.001)
        }

        # OBD 데이터 읽기
        if not IS_SIMULATION and connection:
            cmd_speed = connection.query(obd.commands.SPEED)
            cmd_rpm = connection.query(obd.commands.RPM)
            
            speed = cmd_speed.value.magnitude if not cmd_speed.is_null() else 0
            rpm = cmd_rpm.value.magnitude if not cmd_rpm.is_null() else 0
        else:
            # 시뮬레이션
            speed = random.randint(0, 100)
            rpm = random.randint(1000, 3000)

        # 데이터 패키징
        packet = {
            "car_id": "kia_forte",
            "ts": now,
            "sid": SESSION_ID,
            "data": {
                "spd": speed,
                "rpm": rpm,
                "gps": gps
            }
        }
        
        buffer.append(packet)
        
        # 거리 누적 (단순 계산)
        if speed > 0:
            total_dist += (speed * (0.1 / 3600))

        # --- [2] 버퍼 전송 (50개 찰 때마다) ---
        if len(buffer) >= BATCH_SIZE:
            success = send_to_upstash(buffer)
            if success:
                print(".", end="", flush=True)
            buffer = [] # 버퍼 비우기

        time.sleep(0.1) # 0.1초 대기

except KeyboardInterrupt:
    print("\n\n🛑 주행 종료 (Ctrl+C 감지)")
    
    # --- [3] 잔반 처리 (Flush) ---
    if len(buffer) > 0:
        print(f"📦 남은 데이터 {len(buffer)}개 전송 중...", end="")
        send_to_upstash(buffer)
        print(" 완료!")

    # 리포트 생성
    duration = int(time.time() - start_time)
    minute = duration // 60
    second = duration % 60
    
    report = (
        f"🏁 [주행 종료 리포트]\n"
        f"⏱️ 시간: {minute}분 {second}초\n"
        f"📏 거리: {total_dist:.2f} km\n"
        f"💾 저장된 세션: {SESSION_ID}"
    )
    send_telegram(report)
    print("👋 프로그램 완전 종료")

except Exception as e:
    print(f"\n❌ 에러 발생: {e}")
    send_telegram(f"🚨 에러로 종료: {e}")
```    