# 실험 프로토콜
## 조건: No Assist / Impedance Only / Impedance+ILC
## Treadmill: 고정 속도, ILC 학습 유리, 카메라 삼각대 고정
## Overground: 자유 속도, phase variable 정규화 필수, 카메라 워커 장착
## 데이터: Teensy SD(111Hz) + ZED 포즈(60-90Hz), timestamp 동기화
## 전송: teensy_to_gdrive.py (USB Serial), sbrio_to_gdrive.py (RNDIS)
## 폴더: data/raw/S{번호}_{환경}_{조건}_{날짜}/
## 타겟: IEEE RAL, primary outcome: GDI 변화량
