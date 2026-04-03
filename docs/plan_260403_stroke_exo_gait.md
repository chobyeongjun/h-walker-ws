# Experiment Protocol: 뇌졸중 환자 대상 하지 외골격 보행 실험
Date: 2026-04-03
Author: ChoByeongJun
Status: DRAFT

> **[HARDWARE MISMATCH — 확인 필요]**
> 사용자 입력: "AK60-6 모터 4개 사용"
> hardware.yaml 등록 모터: AK60-6 x2 (left CAN 65, right CAN 33)
> 현재 펌웨어/하드웨어 설정은 bilateral 2-motor 구성만 지원.
> 4개 사용하려면 hardware.yaml 업데이트 및 펌웨어 수정 필요.
> 이 문서는 현재 등록된 2-motor 구성을 기준으로 작성. 4-motor 구성 확정 시 개정 필요.

> **[SAFETY GATE: DO_NOT_FLASH]**
> 최신 안전 감사 결과 (2026-04-03): CRITICAL 6건 미해결.
> 환자 대상 실험 전 모든 CRITICAL 항목 해결 및 재검증 필수.
> 특히: E-stop 미구현(CRITICAL-1), CAN 타임아웃 없음(CRITICAL-2).

---

## 1. Research Question & Hypothesis
- **연구 질문**: 케이블 드리븐 보행 보조 외골격이 편마비 뇌졸중 환자의 보행 대칭성(Symmetry Index)과 보행 속도를 개선하는가? (Assist ON vs Assist OFF, within-subject, treadmill)
- **H₀**: 케이블 보조 조건(Assist ON)과 비보조 조건(Assist OFF) 간 보행 대칭성 지수(SI)에 유의한 차이가 없다. (SI_ON = SI_OFF)
- **H₁**: 케이블 보조 조건에서 보행 대칭성 지수가 유의하게 감소한다. (SI_ON < SI_OFF, 단측 검정)

---

## 2. Experiment Design
- **설계 유형**: Within-subject crossover (2조건 × 1세션, 순서 ABBA counterbalancing)
- **독립변수**: 보조 여부 (Assist ON / Assist OFF)
- **종속변수 (Primary)**: 보행 대칭성 지수 SI = |ST_ipsi - ST_contra| / (0.5 × (ST_ipsi + ST_contra)) × 100 [%]
- **종속변수 (Secondary)**:
  - 보행 속도 [m/s] (트레드밀 고정 속도에서 GCP 주기로 추정)
  - 케이블 피크 보조력 [N] (loadcell)
  - 걸음 주기 변동계수 CV [%]
  - 주관적 피로도 (Borg RPE, 6-20점)
- **통제변수**: 트레드밀 속도 (1.25 m/s 고정), 케이블 장력 프리텐션 5N, 보조 타이밍 파라미터(GCP onset 0.60, peak 0.75, release 0.85)

---

## 3. Participants / Subjects
- **대상**: 편마비 뇌졸중 환자
- **포함 기준**:
  - 뇌졸중 발병 후 6개월 이상 경과
  - 독립 보행 가능 (FAC ≥ 3, 보조도구 허용)
  - 트레드밀 1.25 m/s 보행 가능
  - 연령 40-75세
  - 하지 근력 MRC ≥ 3/5 (환측 무릎 신전)
- **제외 기준**:
  - 심한 경직 (MAS ≥ 3)
  - 심폐 질환 (운동 금기)
  - 하지 골절/수술 이력 6개월 이내
  - 피부 감각 완전 소실 (케이블 접촉 부위)
  - 체중 > 90 kg (장비 하중 한계)
  - 인지 기능 저하 (MMSE < 24)
- **목표 N**: 16명 (α=0.05, power=0.80, Cohen's d=0.8 예상 — 선행 연구: Warnica 2017 exoskeleton symmetry d≈0.85 참조)
  - Power analysis: paired t-test, 단측 α=0.05, power=0.80, d=0.8 → n≈13; dropout 20% 감안 → 16명
- **IRB/윤리**: 필요 (환자 대상 기기 연구). IRB 승인 전 실험 불가. 승인 번호: TBD.

---

## 4. Equipment
- **로봇/장비**:
  - CubeMars AK60-6 V1.1 KV80 × 2대 (left CAN 65, right CAN 33)
    - 연속 토크: 3.0 Nm, 피크 토크: 9.0 Nm
    - 피크 케이블 보조력: ~50 N (peak_force_n, GCP 0.75 기준)
    - 최대 허용 전류: 20 A (SW 제한)
  - Teensy 4.1 (ARM Cortex-M7, 600 MHz)
  - 케이블-스풀 구동 메커니즘 (스풀 반경 40 mm, 효율 86.9%)
- **센서**:
  - EBIMU-9DOFV5 × 2 (좌/우 하지 IMU, UART 921600 baud, 1000 Hz)
    - 측정: 가속도 ±16g, 자이로 ±2000 dps, 지자기 ±8.1 Gauss
  - Loadcell × 2 (아날로그, A16/A6 핀, 50 Hz LPF 후 사용)
    - 교정: left bias -679.2N / sensitivity 550.3 N/V
- **소프트웨어**: 현재 펌웨어 버전 (feat/ms2-fork-validation 브랜치)
- **캘리브레이션**:
  - 실험 전 loadcell zero-offset 측정 (케이블 slack 상태)
  - IMU attitude calibration (정지 상태 10초)
  - 트레드밀 속도 tachometer 확인

---

## 5. Safety Criteria (중단 기준)

**[경고: 아래 기준은 현재 소프트웨어에 E-stop 미구현(CRITICAL-1) 상태에서 수동 절차로만 적용 가능. IRB 제출 전 E-stop 구현 필수.]**

- **즉시 중단 (operator 수동)**:
  - 환자 통증/불편 호소
  - 낙상 징후 (balance support 필요)
  - 장비 이상음 (케이블 마찰, 모터 진동)
  - BLE/CAN 통신 끊김 (현재 자동 감지 미구현 — CRITICAL-2)
- **소프트웨어 자동 정지**:
  - 케이블 꼬임 감지: 보조력 > 300 N AND ERPM > 500 동시 발생 3 ticks (9ms)
  - 포지션 제한: ±4500° from initial
- **토크 제한**: 최대 전류 20 A → 최대 토크 약 9 Nm (sw_max_current_a 기준, 피크 토크의 100% — 임상 시 50% 이하로 조정 권장)
- **시간 제한**: 조건당 최대 연속 보행 15분, 전체 실험 60분 이내
- **생체 지표**: 심박수 > 150 bpm 시 휴식, > 170 bpm 시 중단 (Polar 심박 모니터 착용)
- **케이블 피크 보조력**: 50 N (PEAK_FORCE_N). 환자 체형에 따라 30-50 N 범위 조정.

---

## 6. Procedure (실험 절차)
| Step | 내용 | 시간 | 비고 |
|------|------|------|------|
| 1 | 동의서 서명 + 인구통계/임상 정보 수집 | 10 min | IRB approved form |
| 2 | 신체 계측 (키, 체중, 하지 길이) | 5 min | |
| 3 | 케이블 장치 착용 + IMU 부착 | 10 min | Velcro 고정 확인 |
| 4 | 장비 캘리브레이션 (loadcell, IMU) | 5 min | 정지 상태 |
| 5 | 트레드밀 적응 보행 (Assist OFF) | 5 min | 1.25 m/s |
| 6 | Baseline 측정 (Assist OFF, 무보조) | 5 min | 3분 안정 후 2분 기록 |
| 7 | 조건 A (ABBA 첫 번째) — 3회 반복 | 15 min | 순서: ABBA counterbalancing |
| 8 | 휴식 + Borg RPE 기록 | 5 min | |
| 9 | 조건 B — 3회 반복 | 15 min | |
| 10 | 휴식 + Borg RPE 기록 | 5 min | |
| 11 | 조건 A (ABBA 마지막) — 3회 반복 | 15 min | |
| 12 | 장비 제거 + 불편 사항 청취 | 5 min | |
| 13 | 설문지 작성 (SUS, 만족도) | 5 min | |
| **합계** | | **~105 min** | 실제 보행 시간: ~45 min |

---

## 7. Data Collection
- **파일 네이밍**: `YYMMDD_SXX_[CondName]_T[N].csv`
  - 예: `260403_S01_AssistON_T1.csv`
- **저장 위치**: Teensy SD카드 (BUILTIN_SDCARD) + BLE 실시간 전송 (20 ms 주기)
- **채널** (hardware.yaml 기준):
  - `timestamp_ms` — Teensy millis()
  - `gcp_left`, `gcp_right` — Gait Cycle Phase [0.0-1.0]
  - `force_left_n`, `force_right_n` — Loadcell [N] (50 Hz LPF 후)
  - `current_cmd_left_a`, `current_cmd_right_a` — 명령 전류 [A]
  - `imu_angle_left_deg`, `imu_angle_right_deg` — Euler angle [°]
  - `imu_angvel_left_dps`, `imu_angvel_right_dps` — Angular velocity [°/s]
  - `step_time_left_s`, `step_time_right_s` — 걸음 주기 [s]
  - `assist_mode` — 0: Force, 1: Position
- **샘플링 주파수**: 111 Hz (LOG_PERIOD_MS=9)
- **데이터 검증**: 수집 직후 `python3 validate.py` 실행 (결측 채널, 타임스탬프 갭 확인)

---

## 8. Statistical Analysis Plan
- **Primary** (보행 대칭성 SI): paired t-test (Assist ON vs Assist OFF)
  - 정규성 검정: Shapiro-Wilk (n=16, n<50)
  - 비정규 시: Wilcoxon signed-rank test
  - 유의수준: α=0.05 (단측)
  - 효과크기: Cohen's d (paired)
- **Secondary** (4개 변수): Wilcoxon signed-rank + Bonferroni 보정 (α_adj = 0.05/4 = 0.0125)
- **효과크기 보고**: Cohen's d (paired) for parametric; rank-biserial r for nonparametric
- **소프트웨어**: Python 3 (scipy.stats, pingouin)
- **Power analysis 근거**: Cohen's d=0.8 (medium-large), α=0.05, power=0.80 → n≈13, target N=16 (dropout 20%)

---

## 9. Expected Results
- SI: Assist ON에서 Assist OFF 대비 15-25% 감소 예상 (선행 연구 참조)
  - 예상 Fig 1: SI bar chart (Assist ON vs OFF, paired, error bar = 95% CI)
- 케이블 피크 보조력: 30-50 N 범위 (설정값 50 N, 환자별 편차 예상)
  - 예상 Fig 2: Force profile normalized by GCP (mean ± SD)
- 걸음 주기 변동계수 CV: Assist ON에서 감소 예상 (p < 0.05)

---

## 10. Paper Structure
- **Working Title**: Cable-Driven Bilateral Gait Assist Exoskeleton for Stroke Rehabilitation
- **Target**: IEEE Transactions on Neural Systems and Rehabilitation Engineering (TNSRE) / JNER

```
1. Introduction — 편마비 보행 대칭성 문제 + 케이블 드리븐 보조 gap
2. Methods — 2.1 장치, 2.2 참가자, 2.3 프로토콜, 2.4 데이터 분석
3. Results — Fig 1: SI, Fig 2: Force Profile, Table 1: temporal-spatial
4. Discussion — 임상적 의의, 한계 (단일 세션, 소표본)
5. Conclusion
```

---

## Revision History
| Date | Change | Author |
|------|--------|--------|
| 2026-04-03 | Initial draft | ChoByeongJun |
