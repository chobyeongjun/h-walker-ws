# 제어 알고리즘
## 임피던스: τ = K*(θ_ref-θ) + B*(dθ_ref-dθ), F_cable = τ/r_pulley
- K: 0.5-2.0 Nm/rad, B: 0.05-0.2 Nm·s/rad, 이산화: Tustin 111Hz
- 안전: |F_cable|>70N → 즉시 0
## ILC: u_{k+1}(t) = Q*[u_k(t) + L*e_k(t)]
- L: 0.3-0.7, Q: Butterworth LPF cutoff ~5Hz
- 보행 주기: shank IMU zero-crossing, phase variable 정규화
- 프로파일: 환자별/세션별 JSON 저장
## MPC-ILC (추후)
- horizon 10-20 steps, ILC warm-start, 30Hz 이내 solve
- OSQP 또는 acados
## 주의: 제어 루프 내 동적 할당 금지, 단위 명시 필수
