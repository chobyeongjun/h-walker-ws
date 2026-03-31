# 포즈 추정
## 파이프라인: ZED RGB(60-90Hz)→MediaPipe(2D)→PointCloud(3D)→Joints
## ZED X Mini: S/N 52277959, GMSL2, SDK 5.2.1, HD1080/HD1200만
- 초기화 실패: reset_zed (리부팅 아님)
## MediaPipe: 하체 keypoint (hip,knee,ankle,heel,toe)
- numpy 1.26.4 호환 주의
## 3D 변환: 2D pixel→ZED point cloud→metric(m)
- NaN/0 depth 필터링 필수, 카메라 45도 사선 배치
## 보행 분석: heel strike 기반 주기, GDI primary outcome
## GPU: ZED+MediaPipe ~4GB, PyTorch는 venv_dl 사용
