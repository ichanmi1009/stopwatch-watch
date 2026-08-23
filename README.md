# FPGA Stopwatch & Digital Watch

Basys3 FPGA 보드에서 스톱워치와 디지털 시계를 구현한 Verilog 프로젝트입니다. 두 기능은 Control Unit과 Datapath로 나누어 설계했으며, 스위치로 모드를 선택해 하나의 FND와 LED를 함께 사용하도록 구성했습니다.

- 개발 기간: 2026.04.16 ~ 2026.04.20
- 개발 환경: Xilinx Vivado 2020.2, Vivado Simulator
- 설계 언어: Verilog HDL
- FPGA Board: Digilent Basys3
- 입력 클럭: 100 MHz
- 프로젝트 형태: 2인 팀 프로젝트

## 주요 구현 내용

Watch 기능과 최상위 모듈 통합을 담당했습니다.

- watch_control_unit.v: 수정 모드에서 사용하는 버튼 입력 제어
- watch_datapath.v: 시간 카운터, 수정할 자릿수 선택, 시간 증가·감소 기능 구현
- top_stopwatch_watch.v: Stopwatch와 Watch 연결 및 출력 선택 MUX 구성
- Watch 버튼 조작과 AM/PM LED 동작 시뮬레이션
- 버튼 디바운싱 적용 및 FPGA 실보드 동작 확인

## 동작 구조

버튼 입력은 Debounce 모듈을 거쳐 Stopwatch와 Watch의 Control Unit으로 전달됩니다. 각 Control Unit은 버튼 입력에 따라 제어 신호를 만들고, Datapath는 이 신호에 맞춰 시간 값을 변경합니다. 마지막으로 sw[1]에서 선택한 모드의 시간 값을 FND Controller로 전달합니다.

<p align="center">
  <img src="assets/system-architecture.png" alt="Stopwatch and Watch system architecture" width="100%">
</p>

## 주요 기능

### Stopwatch

- btnR: Run/Stop 전환
- btnL: 현재 시간과 Lap Time 초기화
- btnD: Up/Down Counting 전환
- btnU: 현재 시간을 Lap Time으로 저장
- sw[3]: 실시간 값과 저장된 Lap Time 선택
- 100 Hz Tick을 기준으로 msec, sec, min, hour 순서로 시간 증가

### Digital Watch

- Reset 시 12시로 초기화
- 100 Hz Tick을 이용한 시간 증가
- sw[2]가 켜진 경우에만 시간 수정 가능
- btnL, btnR로 수정할 자릿수 이동
- btnU, btnD로 선택한 시간 값 증가·감소
- Hour 값이 12 이상이면 LED[2]로 PM 표시

### 공통 기능

- 100 kHz 주기로 버튼을 샘플링하고 8-bit Shift Register를 이용해 채터링 제거
- Debounce 출력의 상승 엣지를 검출해 버튼 입력을 1클럭 펄스로 변환
- sw[0]으로 msec/sec 화면과 min/hour 화면 선택
- 1 kHz Scan Clock으로 4자리 FND 순차 점등

## 설계 과정에서 수정한 내용

### Watch 시간 수정 방식

처음에는 Watch의 시간 수정 기능도 FSM으로 구성했습니다. 하지만 state 변경 시점과 버튼 입력 시점이 겹치면서 자릿수가 제대로 선택되지 않거나 입력이 무시되는 문제가 있었습니다.

Watch는 여러 state를 유지하기보다 수정 모드인지 아닌지만 구분하면 됐기 때문에 FSM을 제거했습니다. 대신 sw[2]를 수정 모드 Enable로 사용하고, sw[2]가 꺼져 있을 때는 방향 버튼 입력이 전달되지 않도록 변경했습니다.

### Lap Time 출력 분리

Lap Time을 저장한 뒤에도 실시간 Counter는 계속 동작해야 합니다. 실시간 값과 저장값을 같은 경로에서 처리했을 때 Lap Time이 안정적으로 표시되지 않아 두 값을 별도 신호로 분리했습니다.

sw[3]을 MUX의 선택 신호로 사용해 실시간 값과 Lap Time을 전환하고, Clear 입력 시 두 값이 함께 초기화되도록 수정했습니다.

### 버튼 입력 펄스 처리

버튼의 Level 값을 그대로 사용하면 한 번 눌렀을 때 입력이 여러 클럭 동안 유지될 수 있습니다. Debounce 출력에 상승 엣지 검출을 추가해 버튼을 누를 때마다 1클럭 펄스가 한 번만 발생하도록 했습니다.

## 시뮬레이션 및 FPGA 확인

Stopwatch의 Run/Stop, Up/Down Counting, Clear, Lap Time 저장과 조회 동작을 확인했습니다. Watch에서는 수정 모드가 아닐 때 버튼 입력이 차단되는지, 선택한 자릿수의 시간이 정상적으로 변경되는지 확인했습니다.

### Stopwatch 버튼 동작

btnR 입력 후 시간이 증가하고, Mode 변경 후 감소하는 과정과 Clear 입력 시 0으로 초기화되는 과정을 확인했습니다.

<p align="center">
  <img src="assets/stopwatch-simulation.png" alt="Stopwatch button simulation waveform" width="100%">
</p>

### Watch 시간 수정

sw[2]을 켠 뒤 btnR로 Hour 자릿수를 선택하고, btnD 입력으로 시간이 12에서 11로 감소하는 것을 확인했습니다.

<p align="center">
  <img src="assets/watch-simulation.png" alt="Watch button control simulation waveform" width="100%">
</p>

### AM/PM LED

Hour 값이 12 이상일 때 LED[2]가 켜지고, 11로 변경하면 LED[2]가 꺼지는 것을 확인했습니다.

<p align="center">
  <img src="assets/am-pm-simulation.png" alt="AM and PM LED simulation waveform" width="100%">
</p>

### FPGA 실보드 동작

Basys3 보드에서 FND 출력, 버튼 조작, Stopwatch/Watch 모드 전환을 확인했습니다.

<p align="center">
  <img src="assets/hardware-result.jpg" alt="Stopwatch running on the Basys3 FPGA board" width="85%">
</p>

## 파일 구성

### RTL

- top_stopwatch_watch.v: 전체 모듈 연결과 Stopwatch/Watch 출력 선택
- button_debounce.v: 버튼 채터링 제거와 상승 엣지 검출
- control_unit.v: Stopwatch state 제어와 제어 신호 생성
- stopwatch_datapath.v: Stopwatch 시간 계수와 Lap Time 저장
- watch_control_unit.v: Watch 수정 버튼 입력 제어
- watch_datapath.v: Watch 시간 계수, 자릿수 선택 및 시간 변경
- fnd_controller.v: FND 자릿수 선택, 숫자 분리 및 7-Segment 출력

### 문서 및 이미지

- assets: 시스템 구조도, 시뮬레이션 파형, FPGA 실보드 사진
- docs/final-report.docx: 프로젝트 완료보고서
- docs/presentation.pptx: 최종 발표자료
- docs/schedule.xlsx: 프로젝트 일정표
- docs/development-logs: 날짜별 개발일지

## Vivado 실행 방법

1. Vivado 2020.2에서 RTL Project를 생성합니다.
2. rtl 폴더의 Verilog 파일 7개를 Design Sources에 추가합니다.
3. top_stopwatch_watch를 Top Module로 설정합니다.
4. Basys3의 Clock, Button, Switch, FND, LED 핀에 맞는 XDC Constraint를 추가합니다.
5. Synthesis, Implementation, Bitstream Generation 순서로 진행합니다.
6. 생성된 Bitstream을 Basys3에 Programming해 동작을 확인합니다.

현재 저장소에는 XDC Constraint와 Testbench 원본이 포함되어 있지 않습니다.

## 추가 예정

- Basys3 XDC Constraint와 Testbench 추가
- 버튼 샘플링용 분주 클럭을 Clock Enable 방식으로 변경
- 반복해서 사용할 수 있는 Self-checking Testbench 작성
- 알람과 사용자 시간 저장 기능 추가
