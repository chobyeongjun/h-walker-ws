# Safety Audit Report — h-walker-ws
**Date:** 2026-04-03
**Branch:** feat/ms2-fork-validation
**Auditor:** /skiro-safety (Phase 1-3 full pipeline)
**Target:** firmware/src/Treadmill_main.ino (3787 lines)

---

## Gate Decision

```
┌─────────────────────────────────────────┐
│  10 CRITICAL remaining → ❌ DO NOT FLASH │
└─────────────────────────────────────────┘
```

## Summary

| Category | Count |
|----------|-------|
| CRITICAL | 10 |
| WARNING  | 12 |
| PASS     | 8 |
| N/A      | 1 |

Previous audit (Phase 2 only): CRITICAL 6, WARNING 6, PASS 8
This audit (Phase 2 + Phase 3 fork agent): CRITICAL 10, WARNING 12, PASS 8

Phase 3 fork agent added: **4 new CRITICAL**, **6 new WARNING**
All new CRITICALs are race condition / timing issues that Phase 2 grep patterns could not detect.

---

## CRITICAL Findings

### [CRITICAL-1] E-Stop 핸들러 부재
- **증거:** grep "e_stop|ESTOP|emergency" → 0건
- **물리적 결과:** 비상 시 모터를 즉시 정지시킬 소프트웨어/하드웨어 경로 없음
- **Confidence:** 10/10
- **Source:** Phase 2

### [CRITICAL-2] 하드웨어 워치독 미사용
- **증거:** grep "wdt_enable|wdt_reset|WDOG" firmware/ → 0건. Teensy 4.1 WDOG1/WDOG2 비활성.
- **물리적 결과:** 펌웨어 hang 시 모터가 마지막 전류 명령으로 계속 구동
- **Confidence:** 10/10
- **Source:** Phase 2

### [CRITICAL-3] [MULTI-SOURCE] CAN 수신 타임아웃 부재
- **증거:** canReceiveCallback (line 1279) — 타임스탬프/타임아웃 체크 없음
- **물리적 결과:** CAN 버스 단선 시 stale motor_position_deg/velocity 값으로 제어 지속
- **Confidence:** 10/10 (Phase 2 + COMM-2)
- **Source:** Phase 2 + Communication Specialist

### [CRITICAL-4] [MULTI-SOURCE] Serial handleCommand 파라미터 무제한 쓰기
- **증거:** line 3056 — `PEAK_FORCE_N = cmd.substring(2).toFloat()` (clampf 없음)
  BLE line 3339은 `clampf(0.0f, 150.0f)` 적용됨. Serial 측 14+ 파라미터 무제한.
- **물리적 결과:** `pf99999` 입력 시 99999N 보조력 명령 → 케이블 파단/환자 낙상
- **Confidence:** 10/10 (Phase 2 + COMM-3)
- **Source:** Phase 2 + Communication Specialist

### [CRITICAL-5] [MULTI-SOURCE] ISR 내 Serial.print 호출
- **증거:** line 2140-2155 (controllerStep, 333Hz), line 2052-2054 (positionLoopStep)
  Serial.print는 TX 버퍼 풀 시 blocking.
- **물리적 결과:** ISR 실행 시간 초과 → 제어 루프 jitter/누락 → 모터 명령 타이밍 불안정
- **Confidence:** 10/10 (Phase 2 + TIMING-1 + TIMING-4)
- **Source:** Phase 2 + Timing Specialist (2개 코드 경로 확인)

### [CRITICAL-6] ISR 우선순위 미설정
- **증거:** timerCtrl (line 2492), timer1k (line 2493) — priority() 미호출.
  CAN FIFO ISR이 제어 ISR 선점 가능.
- **물리적 결과:** 제어 ISR 실행 지연/누락
- **Confidence:** 8/10
- **Source:** Phase 2

### [CRITICAL-7] ★NEW ISR 실행 시간 측정 기구 전무
- **증거:** ISR_Control (line 2499~) — 시작/종료 시간 측정 없음, deadline 초과 감지 없음
- **물리적 결과:** ISR이 3ms 예산을 초과해도 시스템이 감지 불가 → 제어 루프 누락 무인식
- **Confidence:** 9/10
- **Source:** Timing Specialist (TIMING-5)

### [CRITICAL-8] ★NEW CAN ISR ↔ ISR_Control 데이터 레이스
- **Writer:** canReceiveCallback:1295 (CAN FIFO ISR)
- **Reader:** controllerStep:2449 안전 체크 (ISR_Control)
- **변수:** motor_position_deg[], motor_velocity_erpm[], motor_current_a[]
- **물리적 결과:** CAN 콜백이 ISR_Control 실행 중간에 선점 → 안전 정지 조건 평가 시
  업데이트 전/후 값 혼재 → 위치 제한(4500°) 초과를 놓칠 수 있음
- **Confidence:** 9/10
- **Source:** Race Condition Specialist (RACE-1)

### [CRITICAL-9] ★NEW desiredCurrent_A ISR-to-ISR 동기화 부재
- **Writer:** ISR_Control:2368 (333Hz)
- **Reader:** ISR_Current1kHz:2603 (1kHz)
- **변수:** desiredCurrent_A[2]
- **물리적 결과:** ISR_Control이 LEFT 쓰기 완료, RIGHT 쓰기 전에 ISR_Current1kHz 선점
  → LEFT=새값, RIGHT=구값으로 CAN 전송 → 양쪽 모터 비대칭 명령
  → 안전 정지 시 한쪽만 즉시 정지, 반대쪽 1ms 지연 → 환자 쏠림
- **Confidence:** 9/10
- **Source:** Race Condition Specialist (RACE-2)

### [CRITICAL-10] ★NEW [MULTI-SOURCE] loadcellRaw_N loop()↔ISR_Control 레이스
- **Writer:** loop():3685 (noInterrupts 없이 loadcell 갱신)
- **Reader:** ISR_Control:2469 안전 체크 (300N 꼬임 감지)
- **변수:** loadcellRaw_N[2]
- **물리적 결과:** loop()가 LEFT 갱신 후 RIGHT 갱신 전에 ISR 선점
  → ISR이 새/구 loadcell 값 혼재로 꼬임 감지 오판
  → SD 쓰기 지연(~100ms) 시 loadcell 갱신 정지 → ISR이 100ms된 stale 값으로 어드미턴스 제어
- **Confidence:** 10/10 (RACE-3 + TIMING-2)
- **Source:** Race Condition Specialist + Timing Specialist

---

## WARNING Findings

### [WARNING-1] CAN TX에 CRC/checksum 없음
- **증거:** sendCurrentCommand (line 1320-1338) — 4바이트 raw 전송. VESC 프로토콜 의존.
- **Confidence:** 7/10
- **Source:** Phase 2

### [WARNING-2] BLE 50ms 타임아웃 시 불완전 명령 실행
- **증거:** BleComm.cpp:172 — "pf50" → "pf5"로 잘림 가능
- **Confidence:** 8/10
- **Source:** Phase 2

### [WARNING-3] Serial 타임아웃 동일 취약점
- **증거:** line 3206 — millis() - last_rx_ms > 50 시 불완전 명령
- **Confidence:** 8/10
- **Source:** Phase 2

### [WARNING-4] [MULTI-SOURCE] Serial/BLE 동시 명령 + motorEnabled 비원자적 토글
- **증거:** loop():3610-3614 순차 처리, line 2908 `motorEnabled = !motorEnabled` (RMW 비원자적)
  ISR_Control이 안전 정지(motorEnabled=false) 직후 loop()가 !false=true 쓰기 가능
- **물리적 결과:** 안전 정지 후 즉각 재활성화 가능
- **Confidence:** 8/10 (Phase 2 + RACE-4/8)
- **Source:** Phase 2 + Race Condition Specialist

### [WARNING-5] MAX_VEL_ERPM 실질적 제한 불가
- **증거:** sw_max_vel_erpm(47000) > 물리적 max(~35280 ERPM)
- **Confidence:** 8/10
- **Source:** Phase 2

### [WARNING-6] delay() in setup
- **증거:** line 1353, 1355, 3578 — setup()에서만. 제어 루프에는 없음.
- **Confidence:** 6/10
- **Source:** Phase 2

### [WARNING-7] ★NEW ISR 내 sinf/sqrtf 호출
- **증거:** line 2331 sinf(), line 2307 sinf(), line 1917-1980 sqrtf()
  ISR_Control(333Hz) 컨텍스트. 단독 위험 낮으나 누적 ISR 시간 불명.
- **물리적 결과:** ISR 실행 시간 증가 → jitter
- **Confidence:** 7/10
- **Source:** Timing Specialist (TIMING-3)

### [WARNING-8] ★NEW BLE TX snprintf 오버플로 가능성
- **증거:** BleComm.cpp:69-91 — `mark * 100` 시 uint32_t 오버플로 → UB
  bleTxBuffer[256] — 최악 시 초과 가능
- **물리적 결과:** 모니터링 데이터 전송 중단 (조용한 실패)
- **Confidence:** 7/10
- **Source:** Communication Specialist (COMM-4)

### [WARNING-9] ★NEW IMU 버퍼 오류 → GCP 고정
- **증거:** line 1711-1713 — sop_pos==-1 시 전체 버퍼 리셋(136바이트 손실)
  IMU 노이즈 버스트 → GCP 고정 → Force Profile 구간에서 고정 시 150N 무기한 지속
- **물리적 결과:** 케이블 과장력
- **Confidence:** 8/10
- **Source:** Communication Specialist (COMM-5)

### [WARNING-10] ★NEW GaitDetector update/getGCP 동기화 없음
- **Writer:** loop():1793 — GaitDetector::update()
- **Reader:** ISR_Control:2093 — getGCP()
- GaitDetector 내부 hs_timestamp, avg_step_time은 비volatile. update() 도중 ISR 선점 시
  GCP가 순간적으로 >1.0 계산 → Force Profile 판정 오류
- **Confidence:** 8/10
- **Source:** Race Condition Specialist (RACE-5)

### [WARNING-11] ★NEW logBuffer 링 버퍼 부분적 보호
- **증거:** ISR_Control:2589 (LogEntry 구조체 쓰기) ↔ loop():1519 (읽기)
  큰 구조체 복사 중 불완전 데이터 SD 기록 가능
- **물리적 결과:** 보행 데이터 손상 (안전 직접 영향 낮음)
- **Confidence:** 7/10
- **Source:** Race Condition Specialist (RACE-6)

### [WARNING-12] ★NEW BLE/Serial 전송 시 motor data 보호 없음
- **증거:** loop():3637-3642 — noInterrupts() 밖에서 motor_position_deg[] 읽기
- **물리적 결과:** 모니터링 데이터 순간 왜곡 (제어 직접 영향 없음)
- **Confidence:** 6/10
- **Source:** Race Condition Specialist (RACE-7)

---

## PASS Findings

| # | Item | Evidence |
|---|------|----------|
| PASS-1 | 전류 제한 clampf | line 371, 2368 — clampf(-MAX_CURRENT_A, MAX_CURRENT_A), MAX=20A |
| PASS-2 | State Machine | enum ControlMode (line 63), volatile currentMode (line 185) |
| PASS-3 | 케이블 꼬임 감지 | SAFETY_PAYOUT_FORCE_N=300N, 3 ticks (line 169-172) |
| PASS-4 | IMU checksum 검증 | line 927-933 — 패킷 체크섬 계산 및 비교 |
| PASS-5 | 위치 제한 | POSITION_LIMIT_DEG=4500 (line 167) |
| PASS-6 | 제어 루프 타이밍 정의 | CONTROL_PERIOD_MS=3 (333Hz), IntervalTimer (line 2492) |
| PASS-7 | dt 안전 범위 검사 | line 2507 — dt <= 0 or > 0.05 → 기본값 |
| PASS-8 | 메인루프 지연 감지 | line 3603 — >50ms 시 경고 출력 |

### [N/A] 배터리 전압 모니터링
24V 전원공급장치 사용 (배터리 아님). IMU battery 필드는 IMU 자체 전원.

---

## Phase 3 검증 결과

### Fork Agent 트리거 조건
| 조건 | 값 | 충족 |
|------|-----|------|
| Treadmill_main.ino 라인 수 | 3787 (≥100) | ✅ |
| volatile 키워드 | 96건 | ✅ |
| ISR 키워드 | 100건 | ✅ |
| IntervalTimer 키워드 | 3건 | ✅ |

→ **Phase 3 정상 발동**

### Specialist 실행 결과
| Specialist | 실행 | 분석 파일 | 발견 건수 |
|---|---|---|---|
| Timing | ✅ | Treadmill_main.ino (ISR_Control, ISR_Current1kHz, loop) | 5건 (CRITICAL 1, WARNING 1, INFO 1, 기존확인 2) |
| Communication | ✅ | Treadmill_main.ino + BleComm.h/cpp | 6건 (WARNING 3, INFO 1, 기존확인 2) |
| Race Condition | ✅ | Treadmill_main.ino (volatile/ISR/loop 전체) | 8건 (CRITICAL 3, WARNING 4, INFO 1) |

### Phase 3이 추가로 발견한 이슈 (Phase 2에서 미탐지)
1. **CRITICAL-7** ISR 마감 초과 감지 없음 — grep으로 탐지 불가 (부재 패턴)
2. **CRITICAL-8** CAN ISR ↔ ISR_Control 데이터 레이스 — 코드 흐름 분석 필요
3. **CRITICAL-9** desiredCurrent_A ISR-to-ISR 동기화 — ISR 간 선점 관계 분석 필요
4. **CRITICAL-10** loadcellRaw_N loop↔ISR 레이스 — loop/ISR 경계 분석 필요
5. **WARNING-7** ISR 내 sinf/sqrtf 호출
6. **WARNING-8** BLE TX snprintf 오버플로
7. **WARNING-9** IMU 버퍼 오류 → GCP 고정
8. **WARNING-10** GaitDetector 동기화 부재
9. **WARNING-11** logBuffer 링 버퍼 보호 부족
10. **WARNING-12** BLE/Serial 전송 시 motor data 보호 없음

### Phase 2에서 이미 발견한 이슈와 중복 [MULTI-SOURCE]
| Phase 2 | Phase 3 | 결과 |
|---------|---------|------|
| CRITICAL-3 (CAN 수신 타임아웃) | COMM-2 | confidence 9→10 |
| CRITICAL-4 (Serial 파라미터 무제한) | COMM-3 | confidence 9→10 |
| CRITICAL-5 (ISR Serial.print) | TIMING-1/4 | confidence 9→10, 2번째 코드경로 확인 |
| WARNING-4 (동시 명령) | RACE-4/8 | confidence 7→8, RMW 비원자성 추가 발견 |
| RACE-3 (loadcellRaw_N) | TIMING-2 | 2개 specialist 독립 확인, confidence 10 |

### 결론
Phase 2 (grep 기반 정적 스캔)은 **패턴 매칭 가능한 이슈**를 효과적으로 탐지했으나,
Phase 3 fork agent는 **코드 흐름 분석이 필요한 레이스 컨디션과 타이밍 이슈** 4건의
새 CRITICAL을 발견. 특히 ISR 간 선점 관계(CRITICAL-8, 9)와 loop-ISR 경계(CRITICAL-10)는
grep으로 탐지 불가능한 구조적 안전 결함.

Phase 3 실행으로 CRITICAL 6 → 10, WARNING 6 → 12로 증가.
**모든 새 CRITICAL은 레이스 컨디션/타이밍 카테고리** — Phase 2 보완 영역 정확히 커버.
