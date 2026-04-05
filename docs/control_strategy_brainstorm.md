# H-Walker 보행 재활 로봇 제어 전략 - 큰 그림

## Context
케이블 드리븐 워커 보행 재활 로봇. ZED 카메라로 발목 위치를 실시간 추적하고,
**정상인의 현재 보행 기준에서 점진적으로 수정된 desired trajectory** (예: stride length +1cm)를
추종하도록 케이블 장력으로 보조. **Treadmill + Overground 모두**에서 robust하게 동작해야 함.

### 실험 대상: 정상인 (건강한 성인)
- 목적: 제어기가 보행 궤적을 원하는 방향으로 수정할 수 있음을 증명
- 정상인에서 검증 → 이후 환자 적용의 근거
- 정상인 장점: 보행 일정, 데이터 품질 높음, 다양한 조건 시도 가능

### 핵심 요구사항
- 안전: loadcell 기반으로 급격한/과도한 힘 방지 (Teensy admittance가 담당)
- 정확: desired trajectory를 잘 따라가도록 (Jetson impedance + adaptive가 담당)
- Robust: treadmill(정속) + overground(가변속) 모두 대응
- 제약: 케이블은 당기기만 가능 (F ≥ 0), 최대 70N

---

## 1. 전체 제어 아키텍처

```
┌───────────────────────────────────────────────────────────┐
│  JETSON (30Hz) - "지능 레이어"                              │
│                                                           │
│  [IMU+ZED Fusion]                                         │
│    IMU (111Hz) → GCP (보행 phase)                          │
│    ZED (30Hz)  → x_ankle (발목 3D 위치)                     │
│    Kalman Filter로 융합 → 연속적 발목 상태 추정                │
│                                                           │
│  [Trajectory] x_ref = lookup(GCP) ← 데이터 기반 보간 궤적    │
│  [Error]      e = x_ref - x_ankle                         │
│                                                           │
│  [Impedance]  F = K(t)*e + B(t)*ė                        │
│       ↑                                                   │
│  [Adaptive]   K(t), B(t) Lyapunov 기반 자동 조정             │
│       ↓                                                   │
│  [MPC]        0 ≤ F ≤ 70N 제약 내 최적 보정 (optional)      │
│       ↓                                                   │
│  F_desired → Teensy로 전송                                 │
└───────────────────────────┬───────────────────────────────┘
                            ↓
┌───────────────────────────────────────────────────────────┐
│  TEENSY (333Hz) - "안전 레이어" (기존 firmware 유지)          │
│                                                           │
│  Admittance: M*dv/dt + C*v = F_desired - F_loadcell       │
│  → 과도한 힘 자동 감쇠, 급격한 변화 방지                       │
│  → 70N 초과 시 즉시 차단                                    │
│  → Velocity PID → Current → AK60 Motor                    │
└───────────────────────────────────────────────────────────┘
```

| 블록 | 역할 | 왜 필요한가 |
|------|------|------------|
| **IMU+ZED Fusion** | 발목 상태 연속 추정 | IMU=빠른GCP + ZED=정확한위치 |
| **Desired Trajectory** | 데이터 기반 보간 궤적 | stride +1cm 등 개인화된 목표 |
| **Impedance** | 위치 오차 → 적절한 힘 변환 | trajectory tracking 핵심 |
| **Adaptive** | K, B Lyapunov 기반 자동 조정 | 속도/개인차 자동 대응 |
| **MPC** | 케이블 F≥0 제약 내 최적화 | 제약의 명시적 처리 (optional) |
| **Admittance (Teensy)** | loadcell 기반 안전 힘 제어 | 급격한 힘 방지, 안전 보장 |

---

## 2. Desired Trajectory 생성 - 핵심 연구 과제

### 2.1 데이터 기반 보간 (주력 방법)
```
Phase 1: Multi-stride 데이터 수집 (보조 없이, ~15분)
  조건 1: "자연스럽게" → stride ~1.30m
  조건 2: "조금 크게"  → stride ~1.35-1.40m
  조건 3: "조금 작게"  → stride ~1.20-1.25m
  조건 4: "아주 크게"  → stride ~1.45-1.50m
  조건 5: "아주 작게"  → stride ~1.10-1.15m
  각 조건: ZED 발목 궤적 + IMU GCP → stride별 GCP 정규화 궤적

Phase 2: 보간으로 desired trajectory 생성
  5개 (stride_length, trajectory) 쌍에서
  원하는 stride length에 대해 각 GCP 시점별 cubic interpolation
  → 그 사람이 실제로 해당 stride로 걸을 때의 자연스러운 궤적에 근접
```

### 2.2 Baseline 변동성 처리
```
Trimmed mean (상하위 10% 제거): baseline 궤적의 "중심"
Std 보존: 자연 변동성 → 수정 크기의 기준
  Δ = 0.5σ (쉬움) / 1.0σ (적당) / 2.0σ (도전적)

GCP 정규화 + 101-point 리샘플링 (현재 IMU HS-to-HS 기반과 동일)
이상치 stride 제거: stride time이 평균 ± 2σ 벗어나면 제거
```

### 2.3 2D 동시 수정
```
데이터 보간에서 자연스럽게 해결:
큰 stride 데이터에 이미 높은 clearance 포함
→ stride +1cm 보간하면 clearance도 자동으로 적절히 증가
→ 별도 수학 모델 없이 물리적으로 타당한 2D 궤적
```

### 2.4 확장성
```
stride, clearance, cadence 등 다차원 보행 파라미터 공간에서 보간 가능
trajectory = f(stride_length, clearance, cadence, ...)
```

---

## 3. IMU + ZED 센서 퓨전

### 왜 퓨전이 필요한가
```
IMU만: GCP 정확 but 발목 위치 모름 → impedance error 계산 불가
ZED만: 발목 위치 알지만 GCP 부정확 (30Hz, 지연) → reference 조회 부정확
퓨전: IMU→GCP(빠른phase) + ZED→x_ankle(정확한위치) = 정확한 error
```

### Kalman Filter 퓨전
```
State: [x_ankle, dx_ankle, GCP, dGCP]
Prediction (IMU 111Hz): x += dx*dt, dx += a_imu*dt
Update (ZED 30Hz): x_corrected = x_predicted + K_kalman*(x_zed - x_predicted)

→ 111Hz 연속 발목 상태 추정 + ZED 보정
→ ZED 누락 시에도 IMU 예측 지속
```

### 시간 동기화
```
타임스탬프 기반 매칭 + IMU velocity로 ZED 프레임 간 보간
→ IMU 고주파(111Hz) + ZED 정확도 = best of both
```

---

## 4. 제어 개념 상세

### 4.1 Impedance Control
```
F = K*(x_ref - x_ankle) + B*(dx_ref - dx_ankle)

가상 스프링(K) + 가상 댐퍼(B) = 부드러운 가이드
환자가 잘 따라가면 e→0 → F→0 (순응적)
Jetson: "무엇을 할지" / Teensy admittance: "어떻게 안전하게"
```

### 4.2 Adaptive Control + Lyapunov 안정성
```
문제: K, B 고정 → 빠른/느린 보행, 개인차에 안 맞음
해결: K, B를 Lyapunov 기반으로 자동 조정

Lyapunov 함수:
  V = (1/2)*m*ė² + (1/2)*K**e² + (1/(2γ))*(K-K*)²
  V ≥ 0, V=0 ↔ e=0, K=K*

dV/dt ≤ 0이 되도록 유도한 adaptation law:
  dK/dt = -γ * e * ė
  조건: B > c (가상 댐퍼 > 실제 마찰)

→ 안정성 수학적 보장
→ 논문에 Theorem + Proof 형태로 제시

Lyapunov 적용 범위:
  Impedance → 안정성 "검증"에 사용
  Adaptive  → 안정성 "설계"에 사용 ← 핵심
  MPC       → terminal cost 설계에 사용
```

### 4.3 MPC (Model Predictive Control, optional)
```
매 30Hz: 현재 상태 → 미래 예측 → 제약 내 최적 F 계산
  minimize Σ[Q*(x_ref-x_pred)² + R*F²]
  s.t. 0 ≤ F ≤ 70N, |dF/dt| ≤ limit

MPC 모델: System ID로 획득 (아래 섹션 참조)
장점: 케이블 F≥0 제약을 사전 최적화 (impedance는 사후 clamp)
없어도 Impedance+Adaptive만으로 기본 동작 가능
```

---

## 5. System Identification (정상인 기준)

### 모델 구조
```
2차 시스템: m*ẍ + c*ẋ + k*x = F_cable
이산화: x(t+1) = a₁*x(t) + a₂*x(t-1) + b₁*F(t) + b₂*F(t-1)
GCP 구간별 분리 가능: stance / early swing / late swing 각각 다른 모델
```

### 실험 프로토콜
```
Phase A (2분): Free walking → baseline 수집
Phase B (3분): Random perturbation
  - 인가 시점: GCP 60~85% (swing)
  - 힘 크기: 5~40N (uniform random)
  - 지속: 100~300ms
  - 동시 기록: F_cable (loadcell 333Hz) + x_ankle (ZED 30-60Hz)
```

### Model Fitting & 검증
```
Least Squares: θ* = (ΦᵀΦ)⁻¹ΦᵀY
검증: train/validation split, R² > 0.8 목표
정상인 N명 → 파라미터 범위 (m, c, k) → 논문 Table + MPC robust 근거
```

---

## 6. Treadmill vs Overground

```
Treadmill: 반복성 높음, adaptive 빠르게 수렴, GCP 정확
Overground: 속도 변화, adaptive 자동 대응, GCP 정규화로 흡수
둘 다: Impedance(모델 불필요) + Adaptive(자동적응) + Teensy(안전) = robust
```

---

## 7. 연구 로드맵 (정상인 실험)

### Phase 1: 기반 구축
- ZED → 발목 3D 실시간 추적 파이프라인
- IMU + ZED Kalman Filter 퓨전
- Multi-stride 데이터 수집 → 데이터 기반 보간 궤적 생성
- Jetson ↔ Teensy F_desired 통신

### Phase 2: Impedance Control
- Jetson impedance: F = K*e + B*ė
- GCP 구간별 gain scheduling
- Teensy admittance 연동 검증

### Phase 3: Adaptive + Lyapunov
- dK/dt = -γ*e*ė adaptation law 구현
- 안정성 수학적 증명 (논문 핵심 contribution)
- Treadmill 수렴 확인 → Overground robustness 확인

### Phase 4: System ID + MPC (optional)
- Random perturbation → 개인별 모델 획득
- OSQP QP solver on Jetson
- Impedance + Adaptive + MPC 통합

### Phase 5: 정상인 실험 & 논문
- 피험자: 정상인 5-10명
- 조건: No Assist / Impedance / Impedance+Adaptive / (+MPC)
- 환경: Treadmill + Overground
- 수정: stride +0.5σ / +1.0σ / +2.0σ
- 메트릭: trajectory RMSE, stride length 변화, 수렴 시간
- 타겟: IEEE RAL

---

## 8. 학습이 필요한 핵심 개념

| 개념 | 왜 필요한가 | 깊이 |
|------|------------|------|
| **Impedance/Admittance** | 제어의 기본 뼈대 | 깊게 (Hogan 1985) |
| **Lyapunov Stability** | Adaptive 안정성 증명 | 깊게 (논문 필수) |
| **Kalman Filter** | IMU+ZED 퓨전 | 중간 (Extended KF) |
| **System Identification** | MPC 모델 획득 | 중간 (선형 SI) |
| **QP / OSQP** | MPC solver | 기본 (사용법) |

---

## 핵심 파일 (구현 시)
- `firmware/src/Treadmill_main.ino` (L2116-2443): Teensy 안전 레이어 (기존 유지)
- `src/hw_control/impedance/`: Jetson impedance + adaptive controller (신규)
- `src/hw_control/mpc_ilc/`: MPC controller (신규)
- `src/hw_perception/benchmarks/`: ZED ankle tracking + sensor fusion
- `src/hw_treadmill_force/core/`: Jetson ↔ Teensy 통신
