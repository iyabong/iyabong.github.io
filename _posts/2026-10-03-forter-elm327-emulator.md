---
title: "Forter — Python ELM327 에뮬레이터로 차 없이 테스트"
date: 2026-10-03
categories: [Forter]
tags: [android, python, elm327, obd2, bluetooth, serial]
---

> This post was compiled by Claude.

## 보완할 점

1. `withTimeout`의 `TimeoutCancellationException`은 `CancellationException` 하위 클래스
   - 잡지 않으면 코루틴이 에러 없이 종료
   - 시동 OFF 시 실패 카운트 없이 20분 기록 누락된 증상과 동일
2. 타임아웃 후 늦게 온 응답이 입력 버퍼에 잔류
   - 다음 명령의 응답으로 읽힘
   - ELM327 통신에 요청 번호가 없어 한 번 밀리면 계속 밀림
3. 테스트 환경
   - 어댑터는 차 OBD2 소켓에 장착, 시동 OFF 시 전원 차단
   - 타임아웃, 늦은 응답 상황을 실차에서 재현하기 어려움

## 목표

1. 차 없이 노트북에서 ELM327 역할을 하는 에뮬레이터 구성
2. 앱 코드 변경 없이 연결 대상만 교체
3. 오늘 범위: 폰 → 노트북 SPP 연결, 앱의 첫 명령 수신 확인

## 옵션 선택

| # | 방법 | 장점 | 단점 |
|---|---|---|---|
| 1 | Python 패키지 `ELM327-emulator` | 바로 사용 | 오류 상황 연출 어려움 |
| 2 | 앱 내부 `FakeConnection` | 블루투스 설정 불필요 | 앱 코드에 테스트 분기 |
| 3 | 하드웨어 ECU 시뮬레이터 | 실차와 동일 | 10만 원 이상 |
| 4 | **직접 작성한 Python 에뮬레이터** | 상황 연출 자유, 앱 코드 무변경 | Windows 블루투스 설정 필요 |

- 4번 선택
- 업무에서 Python exe를 다룰 예정이라 Python 학습 겸용

## 구조

```
테스트: Forter 앱 ── 블루투스 RFCOMM ── COM4 (노트북 가상 포트) ── main.py
실차:   Forter 앱 ── 블루투스 RFCOMM ── UART (어댑터 내부)     ── ELM327 칩
```

1. 앞의 두 칸은 동일
2. 앱에서는 `Android-Vlink` 대신 `DESKTOP-NIT7PBI` 선택
3. 실제 어댑터도 같은 구조 (ELM327 칩의 직렬 신호를 블루투스 모듈이 무선으로 변환)

## 개념 정리

### COM 포트

1. 원래는 PC의 9핀 RS-232 물리 단자
2. 현재는 대부분 가상 COM 포트 (OS는 직렬 포트로 노출, 실제 전송은 USB/블루투스)
3. RFCOMM: RS-232 케이블을 무선으로 흉내 내는 블루투스 프로토콜
4. SPP: RFCOMM 위의 직렬 포트 프로필
5. Java 비유: `InputStream`/`OutputStream` 인터페이스 동일, 구현체만 다름

### 010C의 계층

| 층 | 정의 주체 | 역할 | 웹 비유 |
|---|---|---|---|
| OBD-II | SAE J1979 | `01` 현재 데이터, `0C` RPM | API 명세 |
| ELM327 텍스트 프로토콜 | ELM Electronics | 16진수를 문자로 송수신, `\r` 종료, `>` 프롬프트, AT 명령 | HTTP |
| RS-232 / RFCOMM | 통신 규격 | 바이트 운반 | TCP |

### 응답 `410C1AF8` 해석

1. `41`: 요청 모드 `01` + `0x40` (정상 응답)
2. `0C`: 대상 PID
3. `1AF8`: 데이터, `0x1AF8` = 6904
4. RPM = `(A×256+B)/4` = 1726

### 전송 형태

1. `010C`는 2바이트 이진값이 아닌 ASCII 문자 5개 (`'0' '1' '0' 'C' '\r'`)
2. 에뮬레이터에서는 `decode()`만으로 판독 가능
3. CAN 변환은 ELM327 칩 담당 → 에뮬레이터는 문자 대화만 흉내 내면 됨

## Windows 설정

### 1. Python, pyserial

```
python --version          # Python 3.14.3
pip install pyserial      # 설치 이름 pyserial, import 이름 serial
```

- 실무는 3.x, 최신보다 1~3단계 아래 버전이 일반적
- 2.x는 2020년 지원 종료

### 2. 페어링

1. 장치 관리자에서 `인텔(R) 무선 Bluetooth(R)`, `RFCOMM Protocol TDI` 확인
2. 마우스 동글(2.4G Receiver)은 블루투스 아님
3. 갤럭시는 블루투스 설정 화면을 연 동안에만 검색 가능
4. 페어링 후 `S25 Ultra` 항목 3개, 모두 같은 폰

| 항목 | 의미 |
|---|---|
| 전화기 아이콘, 페어링됨 | 클래식 블루투스 (SPP 사용) |
| USB 3.0 문구 | USB 케이블 연결 |
| 페어링됨 (처음엔 `Bluetooth LE Device ...`) | BLE |

- PowerShell `InstanceId` 접두사로 구분: `BTHENUM` 클래식, `BTHLE` BLE, `USB`

### 3. 수신 COM 포트

1. 추가 Bluetooth 옵션 → COM 포트 → 추가 → 수신(장치에서 연결 시작)
2. 결과: **COM4**
3. 방향 = 연결을 거는 쪽 (데이터 방향 아님)
   - 수신: 폰 → 노트북 연결 (이번 경우)
   - 송신: 노트북 → 다른 기기 연결
4. 연결 후 데이터는 양방향

## 저장소 구조

```
forter/
├── app/
└── tools/
    └── elm327-emulator/
        ├── main.py
        └── requirements.txt
```

| 대상 | 선택 | 근거 |
|---|---|---|
| 상위 폴더 | `tools/` | 독립 실행 도구. 앱이 import하는 헬퍼는 소스 내 `util` 패키지 |
| 단수/복수 | `tools` | 저장소 최상위 폴더는 복수형, Java/Kotlin 패키지는 단수형 |
| 폴더명 | `elm327-emulator` | kebab-case, 전부 소문자 (Windows/Linux 대소문자 차이 회피) |
| 약어 | `emulator` | `emu`는 뜻이 바로 안 읽힘 |
| 실행 파일 | `main.py` | 폴더명과 중복 회피, Go의 `main.go`와 같은 관례 |
| 의존성 파일 | `requirements.txt` | 고정 이름, Dependabot과 편집기가 인식 |

## 코드

```python
import serial

PORT = "COM4"

port = serial.Serial(PORT, timeout=1)  # 1초마다 read 반환 → Ctrl+C 가능
print(f"{PORT} 열림, 폰 연결 대기 중...")

buf = b""
while True:
    data = port.read(1)          # 1바이트 읽기, 없으면 1초 후 b""
    if not data:
        continue
    if data == b"\r":            # ELM327 명령 종료 문자
        print("받음:", buf.decode(errors="replace"))
        buf = b""
    else:
        buf += data
```

1. `b""`: 바이트 문자열 (Java `byte[]`), 시리얼 포트는 바이트 단위 송수신
2. `timeout=1`: 무한 대기 방지
3. 실행: `python -u main.py`
   - `-u`: 출력 버퍼링 해제
   - 콘솔이 아니라고 판단되면 약 8KB 단위로 몰아서 출력되는 문제 방지
   - Java 비유: `flush()` 미호출 `BufferedOutputStream`

## 문제 해결

| # | 증상 | 원인 | 해결 |
|---|---|---|---|
| 1 | `can't find '__main__' module` | 폴더를 실행 | `main.py` 직접 지정 (폴더 실행은 `__main__.py` 필요) |
| 2 | `SyntaxError` at `buf = b ""` | `b`와 따옴표 사이 공백 | `b""` |
| 3 | 수정 후 같은 에러 | 미저장 (`dir`의 시간·크기 변화 없음) | 저장 후 재실행 |
| 4 | 출력 멈춤 | 명령 프롬프트 선택 모드 | Esc |

## 결과

```
D:\git\forter\tools\elm327-emulator>python -u main.py
COM4 열림, 폰 연결 대기 중...
받음: ATZ
```

1. 폰 Forter에서 `DESKTOP-NIT7PBI` 선택 → 노트북에 `ATZ` 수신
2. 차 없이 SPP 통신 경로 확보
3. 응답 미구현이라 앱은 5초 후 타임아웃 (현 단계 정상)

## 다음 단계

1. 명령별 응답 + `>` 프롬프트 반환 → 앱에 RPM 696, 속도 0 표시 확인
2. 에뮬레이터에 "무응답", "지연 응답" 시나리오 추가
3. 해당 시나리오로 `SppConnection.send()` 수정 검증
   - `withTimeoutOrNull` + `IOException`
   - 전송 전 입력 버퍼 비우기
