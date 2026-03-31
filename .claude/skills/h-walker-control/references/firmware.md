# Teensy 펌웨어
## AK60 CAN: FlexCAN_T4, ID 0x00{motor_id}
- Command 8bytes: [pos,vel,Kp,Kd,torque] 각 16bit
- 범위: pos[-12.5,12.5]rad, vel[-45,45]rad/s, torque[-18,18]Nm
## Cascaded PID: Current(1000Hz)→Velocity(333Hz)→Position(111Hz)
- 대역폭 3:1, Anti-windup: clamping
## Serial (Teensy↔Jetson): USB 115200, $CMD,val1,val2,...,checksum\n
- 명령: SET_K, SET_B, SET_REF, GET_STATE, E_STOP
## 로드셀 LSB205: ADC 12bit 16x oversample, Butterworth 20Hz
## IMU: 보행 주기 검출 (각속도 zero-crossing)
## PlatformIO: teensy41, arduino, lib: FlexCAN_T4, ADC, SD
