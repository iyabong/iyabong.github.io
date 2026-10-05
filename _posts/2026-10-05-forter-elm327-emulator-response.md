---
title: "Forter — ELM327 에뮬레이터 응답 구현과 OBD-II 응답 해석"
date: 2026-10-05
categories: [Forter]
tags: [android, python, elm327, obd2, bluetooth, serial]
---

> This post was compiled by Claude.

## 보완할 점

1. 에뮬레이터가 명령을 받기만 하고 응답 없음
   - 앱은 `>` 프롬프트를 기다리다 5초 후 타임아웃
2. 응답 문자열을 손으로 쓰면 16진수 계산 실수 위험
   - PID 추가 시 `0100` 지원 목록 비트맵도 손으로 다시 계산해야 함
3. 앱 화면만으로 에뮬레이터 결과인지 실차 결과인지 구분 불가
   - 나중에 트립 데이터에 테스트 데이터가 섞일 위험

## 목표

1. 명령별 응답 + `>` 프롬프트 반환 → 앱에 RPM 696, 속도 0 km/h 표시
2. 응답을 Wikipedia "OBD-II PIDs" 표의 공식으로 자동 생성
3. 폰 연결 없이 응답을 확인하는 모드 (`--pids`)
4. 앱에 연결 기기 이름·MAC 표시

## 개념 정리

### 명령과 응답의 구조

```
CMD:      01 0C
           ↓  ↓
RESPONSE: 41 0C 0A E0
          │  │  └ 데이터 (A, B)
          │  └ PID 그대로 반환
          └ 01 + 0x40
```

| 바이트 | 값 | 의미 |
|---|---|---|
| `41` | 서비스 `01` + `0x40` | 정상 응답 |
| `0C` | PID | 요청한 PID 그대로 (RPM) |
| `0A` | A = 10 | 데이터 첫 바이트 |
| `E0` | B = 224 | 데이터 둘째 바이트 |

1. `0x40` = `0100 0000`, 비트 하나만 켜서 "이건 응답" 표시
   - 같은 버스에 요청과 응답이 섞이므로 첫 바이트로 구분
   - 외우는 법: 질문은 `0`, 대답은 `4`로 시작 (`01`→`41`, `09`→`49`)
   - 거절은 `7F`로 시작
2. PID를 되돌려주는 이유: 응답이 어느 요청의 것인지 확인
   - `010C`를 보냈는데 `410D...`가 오면 밀린 응답
   - 앱에서 `"41" + 보낸 PID`로 시작하는지 확인하면 감지 가능

### 16진수 표기

1. 16진수 1글자 = 4비트 (니블), 2글자 = 8비트 = 1바이트
   - `1` → `0001`, `8` → `1000` → `18` = `0001 1000`
   - 글자끼리 독립이라 글자별로 변환해 이어 붙이면 됨 (10진수는 불가)
2. `0x` 생략 이유
   - 이 프로토콜에서 숫자 자리는 전부 16진수 → 접두사가 정보를 더하지 않음
   - ELM327은 바이트가 아닌 ASCII 문자로 송수신 → `0x`는 매번 2바이트 낭비
3. 공백도 원래 있음 (`41 0C 0A E0`), 앱 초기화의 `ATS0`이 끔

### A, B 표기와 공식

1. 출처: Wikipedia "OBD-II PIDs" 공식 열의 표기
   - 머리말(`41 0C`)을 뺀 데이터 바이트에 앞에서부터 A, B, C, D
   - 원본은 SAE J1979의 Data A, Data B 표기로 알려짐 (유료 문서라 미확인)
2. `256A + B`: 바이트 두 개를 이어 붙여 큰 수 하나로
   - 10진수 `47` = `10×4 + 7`과 같은 원리, 한 칸이 256가지라 256을 곱함
   - 비트로 보면 A를 8칸 밀고 B를 붙임: `(a << 8) | b`
   - 16진수로는 그냥 나란히: `0A` + `E0` = `0x0AE0` = 2784
3. `/ 4`: 차가 0.25 rpm 단위를 정수로 보내려고 4배 해서 보냄
   - 표현 범위 0 ~ 16383.75 rpm
   - MAF의 `/ 100`도 같은 원리 (0.01 g/s 단위)
4. 256은 고정, 바이트 수에 따라 거듭제곱만 달라짐

| 바이트 수 | 공식 | 예 |
|---|---|---|
| 1 | A | 속도 `0D` |
| 2 | 256A + B | RPM `0C`, MAF `10` |
| 4 | 256³A + 256²B + 256C + D | 주행거리 `A6` (2012년식 지원 여부 미확인) |

- 예외: 산소 센서(`14`~`1B`)는 A, B가 서로 다른 값, 일부 PID는 부호 있는 값

### 정의 위치

| 문서 | 다루는 것 | 비고 |
|---|---|---|
| SAE J1979 / ISO 15031-5 | 서비스, PID, 공식 원본 | 유료 |
| Wikipedia "OBD-II PIDs" | Services 표, Service 01 PID 표, A·B 공식 | 실무 참고서 |
| ELM327 데이터시트 | AT 명령, 문자 송수신, `>` 프롬프트, 오류 메시지 | 값의 의미는 거의 없음 |

- `010C` 해석: 앞 두 글자로 Services 표, 뒤 두 글자로 PID 표 조회

### PID 00: 지원 PID 비트맵

1. 다른 PID와 해석 방식이 다름
   - 다른 PID: 바이트 → 공식 → 숫자 하나
   - PID 00: 4바이트 = 32비트 = 예/아니오 32개 (Wiki 공식 칸 "Bit encoded")
2. 비트맵 = 칸 위치가 곧 항목 번호인 지도 (극장 좌석표, Linux 권한 `755`와 같은 방식)
3. 32칸이 왼쪽부터 PID 01 ~ 20에 대응, 1이면 지원
4. `A0`, `B0`, `C0`, `D0`는 모두 각 바이트의 끝 비트지만 32칸 중 위치가 다름 (PID 08, 10, 18, 20)
5. D0(PID 20) = 다음 목록(`0120`) 존재 표시 → `0100` → `0120` → `0140` 순으로 이어서 조회

## 에뮬레이터 설계

### 구조

```
state, PIDS                                ← 데이터 (값, 변환 규칙)
one_byte, two_bytes, supported_bitmap      ← 값 → 바이트
obd_response, handle_obd, handle_at, reply_for  ← 질문 → 대답
print_pids, serve                          ← 실행 (확인 모드 / 폰 연결)
```

### 값은 10진수, 16진수는 코드가

```python
state = {
    "load":     20.0,   # %
    "coolant":  85,     # °C
    "rpm":      696,    # 실차 공회전 값
    "speed":    0,      # km/h
    "maf":      2.5,    # g/s
    "throttle": 15.0,   # %
    "voltage":  12.6,   # V (ATRV)
}
```

### PID 표 = Wiki 표와 1:1

```python
# PID: (이름, state 키, 인코더)  — 인코더 = Wiki 공식의 역함수
PIDS = {
    0x04: ("Engine load",       "load",     lambda v: one_byte(v * 255 / 100)),  # 100/255 * A
    0x05: ("Coolant temp",      "coolant",  lambda v: one_byte(v + 40)),         # A - 40
    0x0C: ("Engine RPM",        "rpm",      lambda v: two_bytes(v * 4)),         # (256A + B) / 4
    0x0D: ("Vehicle speed",     "speed",    lambda v: one_byte(v)),              # A
    0x10: ("MAF air flow rate", "maf",      lambda v: two_bytes(v * 100)),       # (256A + B) / 100
    0x11: ("Throttle position", "throttle", lambda v: one_byte(v * 255 / 100)),  # 100/255 * A
}
```

1. 공식에서 `/ 4`면 인코더는 `* 4`, `- 40`이면 `+ 40`
2. `lambda v: ...` = Java의 `v -> ...`
3. 다음 계획 PID(04, 05, 10, 11)도 함께 등록

### 0100 비트맵 자동 계산

```python
def supported_bitmap():
    bits = 0
    for pid in PIDS:
        if 0x01 <= pid <= 0x20:
            bits |= 1 << (0x20 - pid)        # PID 01 = 맨 왼쪽(31번 비트)
    return [(bits >> shift) & 0xFF for shift in (24, 16, 8, 0)]   # 8칸씩 A, B, C, D
```

1. `0x20 - pid`: 오른쪽에서 몇 번째 칸인지 (비트 연산은 오른쪽 끝이 0번)
2. `1 << n`: 그 칸에만 1, `|=`: 기존 비트맵에 겹쳐 적기
3. `>> shift` + `& 0xFF`: 원하는 8칸을 끌어와 그것만 남김
4. `PIDS`에 한 줄 추가하면 `0100` 응답도 자동 반영
5. 역함수 `bitmap_to_pids()`: 비트맵 → PID 목록, 실차 `0100` 응답 해석에도 사용 예정

### 응답 생성

```python
def obd_response(service, pid, data):
    return bytes([service + 0x40, pid, *data]).hex().upper()
```

- `010C` 처리 흐름: `\r`까지 수신 → `AT` 아님 → 서비스 01, PID 0C → `two_bytes(696 * 4)` = `[10, 224]` → `410C0AE0` → `\r\r>` 붙여 전송
- 차가 모르는 PID, 다른 서비스: `NO DATA` (실제 ELM327과 동일)
- 공백·소문자 명령도 처리 (`01 0c` → `010C`)

### 확인 모드 `--pids`

```
$ python -u main.py --pids
[0100] supported PIDs → 410018198000
BYTE  HEX  BIN       RANGE  PIDS
----------------------------------------
A     18   00011000  01-08  04 05
B     19   00011001  09-10  0C 0D 10
C     80   10000000  11-18  11
D     00   00000000  19-20  -

CMD(HEX)  RESPONSE(HEX)  DATA(DEC)         MEANING
------------------------------------------------------------
0104      410433         [51]              Engine load = 20.0
0105      41057D         [125]             Coolant temp = 85
010C      410C0AE0       [10, 224]         Engine RPM = 696
010D      410D00         [0]               Vehicle speed = 0
0110      411000FA       [0, 250]          MAF air flow rate = 2.5
0111      411126         [38]              Throttle position = 15.0
```

1. 표 하나로 비트맵과 값을 같이 보이니 열이 꼬임 → 표 2개로 분리
2. 표 1의 PIDS 열은 `PIDS`를 베낀 게 아니라 응답 비트를 다시 해석한 결과 → 비트맵 계산 검산
3. `DATA(DEC)` = 데이터 부분만 10진수 (머리말 제외), Wiki 공식에 바로 대입 가능
4. 열 너비는 제목줄과 내용 줄에 같은 숫자 (`:<9` = 왼쪽 정렬 9칸)

### 결정 사항

| 대상 | 선택 | 근거 |
|---|---|---|
| 확인 옵션 | `--test` → `--pids` | 하는 일(PID 목록 출력)이 드러나는 이름 |
| 폴더명 | `elm327-emulator` 유지 | `elm327`만으로는 앱 `obd/Elm327.kt`와 혼동, `emu`는 표준 약어지만 폴더는 풀어 씀 |
| 에뮬레이터 버전 | 붙이지 않음 | 같은 저장소의 개발 도구, git 이력이 버전 역할 |
| `import serial` | 맨 위 | 노트북에 pyserial 설치됨, 함수 안 import 불필요 |

- emulator vs simulator: 상대가 진짜와 구분 못 하게 프로토콜을 흉내 내면 emulator, 내부 동작을 모델링하면 simulator → 현재는 emulator

## 앱 변경

```kotlin
status = "연결: ${device.name} (${device.address})\n"
status += "0100: $supported\nRPM: $rpm\n속도: $speed km/h"
```

1. 응답 내용은 건드리지 않고 연결 기기로 구분 → 에뮬레이터를 실제와 동일하게 유지
2. MAC 포함: 이름은 바뀔 수 있지만 MAC은 고정, 트립 데이터 출처 표시에 재사용 예정
3. `versionName` `0.1.4` → `0.1.5` (`app/build.gradle.kts`)

### 앱이 보내는 것

- HTTP 요청이 아닌, SPP 선으로 `명령 + \r` 바이트 전송 (`SppConnection.send()`)
- `main.py`는 요청마다 호출되는 게 아니라 COM4를 계속 읽는 루프

| 순서 | 보냄 | 위치 |
|---|---|---|
| 1 | `ATZ` | `Elm327.init()` |
| 2~6 | `ATE0` `ATL0` `ATS0` `ATH0` `ATSP0` | `Elm327.init()` |
| 7 | `0100` | `MainActivity` |
| 8 | `010C` | `Elm327.rpm()` |
| 9 | `010D` | `Elm327.speed()` |

| 앱 (Kotlin) | 에뮬레이터 (Python) |
|---|---|
| `write("$command\r")` | `if data == b"\r":`에서 명령 완성 |
| `if (c == '>') break` | `port.write(f"{resp}\r\r>")` |

## 문제 해결

| # | 증상 | 원인 | 해결 |
|---|---|---|---|
| 1 | IDE가 Python 3.14.8 다운로드 제안 | 인터프리터 미등록 | 취소, `which python`으로 찾은 기존 3.14.3 등록 (새로 받으면 pyserial 없음) |
| 2 | Python 안내줄 계속 표시 | 코드 인텔리전스 미설정 | Enable → LSP4IJ 플러그인 설치 → 재시작 |
| 3 | `SyntaxError` | `min[...]`, 쉼표 누락, docstring 3칸 들여쓰기 | Python은 첫 오류에서 멈춤 → 하나씩 수정 |
| 4 | `NameError`, `AttributeError` | `replay_for`, `unsupported_bitmap`, `itmes` 오타 | `Did you mean:` 힌트대로 수정 |
| 5 | 터미널에 `{cmd}`가 글자 그대로 | `f`가 따옴표 안 (`"f받음..."`) | `f"받음..."` |
| 6 | 표 열 어긋남 | 제목줄과 내용 줄 너비 숫자 불일치 | 세 줄 모두 같은 너비 |
| 7 | 저장했는데 반영 안 됨 | 주석만 수정 | 코드 줄 수정 (Python은 실행마다 파일을 새로 읽음) |
| 8 | `.idea/markdown.xml` 계속 Untracked | `.gitignore`에 `./idea/...` | `.idea/markdown.xml` |
| 9 | `adb: command not found` | Git Bash PATH에 SDK 없음 | 전체 경로 또는 `~/.bashrc`에 `platform-tools` 추가 |
| 10 | `adb pair` → `protocol fault` | 공용 Wi-Fi(172.16 대역) 방화벽 추정 | 미해결, USB로 빌드 |

- Traceback 읽기: 맨 아래가 원인과 터진 위치, 위로 갈수록 호출한 쪽 (Java 스택 트레이스와 반대)
- 10번 확인 내역: 코드를 인자로 전달(`adb pair IP:포트 코드`), adb 36.0.2 최신, `kill-server` 후 재시도 → 모두 동일 → 네트워크 외 원인 없음

## 결과

```
$ python -u main.py
COM4 열림, 폰 연결 대기 중...
받음: ATZ    → 보냄: ELM v2.3
받음: ATE0   → 보냄: OK
받음: ATL0   → 보냄: OK
받음: ATS0   → 보냄: OK
받음: ATH0   → 보냄: OK
받음: ATSP0  → 보냄: OK
받음: 0100   → 보냄: 410018198000
받음: 010C   → 보냄: 410C0AE0
받음: 010D   → 보냄: 410D00
```

```
Forter v0.1.4-b03c1cb
연결: DESKTOP-XXXXXXX (F4:96:34:XX:XX:XX)
0100: 410018198000
RPM: 696
속도: 0 km/h
```

1. 차 없이 앱 → 블루투스 → 에뮬레이터 → 앱 화면 전체 흐름 동작
2. 앱 초기화 순서 전체 확인
3. 연결 기기로 에뮬레이터/실차 구분 가능

## 다음 단계

1. `handle_at()`의 `"ELM v2.3"` → `"ELM327 v2.3"` (실제 어댑터와 일치, 커밋 누락)
2. 응답 지연·무응답 시나리오 추가 (`time.sleep(DELAY)`)
3. 해당 시나리오로 `SppConnection.send()` 수정 검증
   - `withTimeoutOrNull` + `IOException`
   - 전송 전 입력 버퍼 비우기
   - 응답이 `"41" + PID`로 시작하는지 확인
4. 실차 `0100` 응답을 `bitmap_to_pids()`로 해석 → Forte 지원 PID 확인
5. 트립 데이터에 연결 기기 MAC 저장 (테스트 데이터 분리)
6. 집 Wi-Fi에서 무선 디버깅 재시도
