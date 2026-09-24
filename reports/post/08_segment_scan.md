# 실험 후 레포트: 7세그먼트 자리 스캔

작성일 2026-09-24.

[실험 전 레포트](../pre/08_segment_scan.md) · [해시·입력 기록](../../build/sim/result.json)

## Vivado GUI 과정과 사전 결과 비교

공개 배포 템플릿(v2.0.1) 기반 환경에서 Vivado 2026.1 GUI의 New Project를 실행하여 `lab2_segment_scan` 프로젝트를 생성했습니다. 타깃 부품은 `xc7s75fgga484-1` (Spartan-7, 패키지 fgga484, 속도 등급 -1)을 선택했습니다.

RTL 소스인 `src/segment_scan8.v`, `src/input_frontend.v`, `src/lab2_segment_scan.v`를 Design Sources로 등록하고, 자체 검증용 테스트벤치 `sim/tb_segment_scan8.sv`를 Simulation Sources에, 핀 및 클록 제약 파일 `constraints/lab2_segment_scan.xdc`를 Constraints에 추가했습니다. 이때 Copy sources into project 옵션을 해제하여 VS Code 작업 폴더의 원본 파일을 직접 참조하도록 설정했습니다.

설계 최상위 모듈(Design Top)은 `lab2_segment_scan`, 시뮬레이션 최상위 모듈(Simulation Top)은 `tb_segment_scan8`로 분리 지정했습니다.

Run Simulation → Run Behavioral Simulation을 실행했습니다. Vivado 기본 시뮬레이션 설정은 1000 ns에서 일시 정지(checks=150)하므로, 상단 메뉴의 Run All(또는 F3)을 실행하여 테스트벤치의 $finish 호출 시점까지 시뮬레이션을 완료했습니다.

Tcl Console에서 `LAB2_PASS segment_scan checks=194` 출력과 1306 ns($finish called at 1306000 ps) 정상 종료를 확인했습니다. VS Code(Icarus Verilog/VaporView)의 사전 시뮬레이션 결과와 비교했을 때, digits 2개 묶음(`7654_3210`, `fedc_ba98`)에 대한 0~F 진리표 패턴 매핑, index=0~7 순차 순회, 활성 슬롯과 blank(소등) 슬롯의 교대 발생, enable=0에서의 점등/소등 상태 유지, 비영 상태에서의 동기 리셋 초기화 등 총 194개 검사 항목과 타이밍 전이 시각이 100% 일치함을 대조했습니다.

코어 모듈의 논리적 출력 `select`는 1일 때 해당 자리가 켜지는 active-high 방식이며, 자리 전환 시마다 `select=00`인 blank 구간을 거침으로써 인접 자리 간 잔상 및 고스팅(ghosting) 현상을 방지하는 스캔 구조를 확인했습니다.

## 합성·구현·bit

Flow Navigator에서 Run Synthesis → Run Implementation → Generate Bitstream을 단계별로 실행하였으며, Design Runs 패널에서 `synth_design Complete!` 및 `write_bitstream Complete!` 상태를 확인했습니다. GUI 빌드 로그를 보관했습니다.

* **생성 파일**: `vivado/lab2_segment_scan.runs/impl_1/lab2_segment_scan.bit`
* **배포 파일**: lab2_segment_scan.bit (SHA-256 해시값 기록 완료)
* **핀 배치 확인**: Elaborated Design 및 Implemented Design의 I/O Ports 창에서 주 클록(`clk`=B6), 리셋(`rst`=K4), 버튼(`button`=N8), 스위치(`sw[7:0]`), LED(`led[7:0]`), 7세그먼트 데이터 8핀(`seg_data[7:0]`), 공통 단자 8핀(`seg_com[7:0]`) 등 총 35개 포트가 XDC 명세대로 `LVCMOS33` 규격과 지정 핀에 올바르게 할당되었음을 대조했습니다.

### 타이밍 및 경고(Warning) 분석

1. **내부 클록 타이밍 결과**:
   * 온보드 1 kHz 클록(`trainer_1khz`, 주기 1,000,000.000 ns) 제약 조건에서 Open Implemented Design → Timing Summary를 확인한 결과, Setup WNS = 999998.062 ns, Hold WHS = 0.119 ns, Failing Endpoints = 0개(전체 17개)로 내부 동기식 경로의 타이밍을 안정적으로 만족했습니다. 보드 주기가 1 ms로 매우 길기 때문에 Setup Slack(WNS)이 수십만 ns 수준으로 크게 보고되었습니다.
2. **TIMING-18 경고**:
   * 외부 입출력 지연(I/O delay) 제약 누락 관련 경고입니다. 리셋, 버튼, 스위치는 비동기 입력이므로 `set_false_path`로 예외 처리하였으며, LED 및 7세그먼트 출력 포트(총 24개)는 외부 동기 클록으로 래치되는 버스가 아닌 시각 표시용 인터페이스이므로 타이밍 제약을 추가하지 않아 발생한 정상적인 경고임을 확인했습니다.
3. **상수 출력 경고**:
   * `lab2_segment_scan.v` 코드에서 미사용 상위 LED 비트(`led[7:3]`)를 `5'b00000`으로 고정 배선(`assign led = {5'b00000, index};`)하였기 때문에 발생한 경고임을 확인했습니다.
4. **DRC 경고 (CFGBVS-1)**:
   * Bank 0의 전압 속성(CFGBVS/CONFIG_VOLTAGE)이 지정되지 않아 발생한 경고입니다. 실제 보드 회로도 기준을 확인해야 하므로 임의의 전압값을 억지로 넣지 않았으며, 비트스트림이 정상 생성되었음을 확인했습니다.

## 보드 기록·촬영 상태

Combo II-DLD S75 보드의 전원 및 JTAG 케이블을 연결하고 온보드 클록 선택 스위치를 1 kHz로 맞춘 뒤, Hardware Manager의 Auto Connect를 통해 `xc7s75` 디바이스에 `lab2_segment_scan.bit`를 다운로드하여 실물 동작을 검증했습니다.

K4 푸시버튼으로 리셋을 인가한 후, DIP SW1~4(`sw[7:4]`)로 첫 번째 자리(COM[7])에 표시할 16진수 숫자를 설정하고 8자리 7세그먼트 FND에 표시되는 패턴을 실측하여 사진과 영상을 촬영했습니다. 이 회로는 `enable=1`로 고정되어 자동 시분할 스캔하므로 N8 버튼은 조작할 필요가 없습니다.

| 순서 | 조작 조건 | 스위치 설정 (SW1..4) | 기대 표시 문자열 (COM[7] $\rightarrow$ COM[0]) | 실측 7세그먼트 표시 | 동작 상태 및 해석 | 사진 |
|---|---|---|---|---|---|---|
| 1 | K4 리셋 인가 | SW1..4=0000(0) | 0 1 2 3 4 5 6 7 | 0 1 2 3 4 5 6 7 | 코어 index=0 초기화, 첫 자리 0 표시 | [초기화](../../evidence/08/board/photos/step1_reset.jpg) |
| 2 | 첫 자리 9 설정 | SW1..4=1001(9) | 9 1 2 3 4 5 6 7 | 9 1 2 3 4 5 6 7 | 첫 자리만 9로 변경, 나머지 1~7 유지 | [9 설정](../../evidence/08/board/photos/step2_sw9.jpg) |
| 3 | 첫 자리 A 설정 | SW1..4=1010(A) | A 1 2 3 4 5 6 7 | A 1 2 3 4 5 6 7 | 16진 문자 'A' 정상 표출 | [A 설정](../../evidence/08/board/photos/step3_swA.jpg) |
| 4 | 첫 자리 F 설정 | SW1..4=1111(F) | F 1 2 3 4 5 6 7 | F 1 2 3 4 5 6 7 | 16진 문자 'F' 정상 표출 | [F 설정](../../evidence/08/board/photos/step4_swF.jpg) |

[7세그먼트 스캔 보드 시연 영상](../../evidence/08/board/videos/demo.mp4)

* 1 kHz 클록 기반에서 한 자리가 1 ms 켜지고 1 ms 꺼지므로 자리당 주기는 2 ms이며, 8자리 전체를 순회하는 데 16 ms가 소요되어 약 62.5 Hz의 프레임 레이트로 잔상 효과(POV)에 의해 8자리가 동시에 켜져 있는 것처럼 안정적으로 표시됨을 확인했습니다.
* 코어의 `select` 비트 순서와 극성이 래퍼(`lab2_segment_scan.v`)에서 비트 반전 및 순서 역전(`assign seg_com = ~{selected[0], ..., selected[7]};`)을 거쳐 보드의 active-low 공통 단자(`COM[7:0]`)로 변환되어, `index=0` 자리가 정확히 최좌측 COM[7]에 매핑됨을 입증했습니다.
* `led[2:0]`에 현재 스캔 중인 `index` 값이 출력되며, 스캔 속도가 빠르기 때문에 육안으로는 3개 LED가 모두 희미하게 점등된 상태로 관찰됨을 확인했습니다.

## 결론

시분할 다중화(Time-division Multiplexing) 기법과 blanking 슬롯을 포함한 8자리 7세그먼트 동적 구동 회로의 194개 전수 검사 항목에 대해 VS Code Icarus Verilog와 Vivado GUI XSim 간 시뮬레이션 결과가 100% 동일함을 확인했습니다.

8자리의 데이터를 동시에 출력하는 대신, 단일 버스(`seg_data`)를 공유하고 자리 선택 신호(`seg_com`)를 순차적으로 활성화하는 동적 스캔 구조를 통해 FPGA 핀 사용량을 대폭 절감할 수 있음을 검증했습니다. 또한 각 자리가 켜지는 사이에 반드시 1클록의 blanking(소등) 구간을 두어 이전 숫자가 다음 자리로 번지는 고스팅 현상을 원천 차단함을 확인했습니다.

Spartan-7(`xc7s75fgga484-1`) 타깃으로 합성, 구현, 내부 1 kHz 클록 타이밍 충족(WNS/WHS 마진 확보) 및 비트스트림 생성을 완료하였으며, Combo II-DLD S75 보드 상에서 스위치 조작에 따른 첫 자리 16진수 변경과 8자리 숫자의 깜빡임 없는 선명한 시연 동작을 실측 검증했습니다.