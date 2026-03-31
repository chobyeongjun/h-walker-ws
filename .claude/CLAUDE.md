# H-Walker Workspace

## 프로젝트 개요
케이블 드리븐 워커 장착형 보행 재활 로봇. Treadmill/Overground 두 가지 실험 환경.

## 구조
- src/hw_common/ → 공통 (CAN, 시리얼, 센서)
- src/hw_perception/ → ZED + MediaPipe 포즈 추정
- src/hw_control/ → 임피던스, ILC, MPC-ILC
- src/hw_treadmill/ → 트레드밀 전용
- src/hw_overground/ → 지면보행 전용
- firmware/ → Teensy 4.1 (PlatformIO)

## 빌드
- C++: cmake -B build && cmake --build build
- Python: pip install -r requirements.txt
- Teensy: cd firmware && pio run -t upload

## 핵심 제약
- 제어 주기: Teensy inner 111Hz / Jetson outer 10-30Hz
- AK60 max cable force: ~70N, 초과 시 즉시 0
- ZED: HD1080/HD1200만, 초기화 실패 시 reset_zed
- data/ 폴더는 Git에 올리지 않음
