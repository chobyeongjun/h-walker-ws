---
name: h-walker-control
description: "H-Walker 케이블 드리븐 워커 보행 재활 로봇 개발. impedance, ILC, MPC, pose estimation, CAN, Teensy 키워드에 자동 활성화."
---

# H-Walker 제어 시스템

## 참조 파일 (해당 작업 시에만 읽을 것)
| 작업 | 파일 |
|------|------|
| 임피던스/ILC/MPC 제어 | references/control.md |
| Teensy CAN/센서 | references/firmware.md |
| ZED 포즈 추정 | references/perception.md |
| 실험/데이터 수집 | references/experiment.md |

## 핵심 상수
- Inner loop: 111Hz (Teensy), Outer: 10-30Hz (Jetson)
- AK60 max force: ~70N
- ILC learning gain: 0.3-0.7
- 조건: No Assist / Impedance / Impedance+ILC
