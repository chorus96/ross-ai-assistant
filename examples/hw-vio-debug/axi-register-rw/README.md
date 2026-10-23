<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->
# VIO AXI Register Read/Write
# VIO AXI 레지스터 읽기/쓰기

A design with a VIO-controlled AXI-Lite master and AXI-Stream packet
VIO 제어 AXI-Lite 마스터 및 AXI-Stream 패킷을 사용한 설계
generator, then debugged with the `hw-vio-debug` skill.
생성기를 실행한 후 `hw-vio-debug` 스킬로 디버깅합니다.

The user drives AXI-Lite register writes and reads, and controls streaming
사용자는 AXI-Lite 레지스터 쓰기 및 읽기를 구동하고 스트리밍을 제어합니다.
traffic, entirely through VIO output probes — observing results through VIO
VIO 출력 프로브를 통한 트래픽 — VIO를 통한 결과 관찰
input probes.
입력 프로브.
The design source in `input/` is complete and self-contained
`input/`의 디자인 소스는 완전하고 독립적입니다.
(RTL + Tcl).
(RTL + Tcl).
The debug step depends on the **`hw-vio-debug` skill**, which
디버그 단계는 **`hw-vio-debug` 스킬**에 따라 달라집니다.
lives at [`skills/hw-vio-debug/`](../../../skills/hw-vio-debug/) in this repo.
이 저장소의 [`skills/hw-vio-debug/`](../../../skills/hw-vio-debug/)에 있습니다.

## Goal
## 목표

Use the VIO to interactively control and observe an AXI-based design on live
VIO를 사용하여 실시간으로 AXI 기반 설계를 대화식으로 제어하고 관찰하세요.
hardware:
하드웨어:

1. **Discover** VIO cores and list all probes with names, directions, widths.
1. **탐색** VIO 코어를 검색하고 이름, 방향, 너비와 함께 모든 프로브를 나열합니다.
2. **Write a register** — drive wr_addr + wr_data via VIO outputs, pulse
2. **레지스터 쓰기** — VIO 출력을 통해 wr_addr + wr_data 구동, 펄스
start_wr, observe the done pulse and busy deassertion.
start_wr, 완료된 펄스와 사용 중인 Deassertion을 관찰합니다.
3. **Read back** — drive rd_addr, pulse start_rd, read rd_data from VIO input
3. **다시 읽기** — rd_addr 구동, start_rd 펄스, VIO 입력에서 rd_data 읽기
to confirm the round-trip.
왕복을 확인하기 위해
4. **Stream control** — set pkt_length, pulse start_stream, monitor pkt_count
4. **스트림 제어** — pkt_length, pulse start_stream, pkt_count 모니터링 설정
incrementing.
증가.

## Skills Used
## 사용된 스킬

- **hw-vio-debug** — discovers VIO cores and probes, reads input values
- **hw-vio-debug** — VIO 코어 및 프로브를 검색하고 입력 값을 읽습니다.
(`refresh_hw_vio`), drives output values (`set_property OUTPUT_VALUE` +
(`refresh_hw_vio`), 출력 값 구동(`set_property OUTPUT_VALUE` +
`commit_hw_vio`), checks activity, and resets outputs.
`commit_hw_vio`), 활동을 확인하고 출력을 재설정합니다.
All operations go
모든 작업이 진행됩니다.
through Vivado Hardware Manager Tcl via `vivado_execute`.
'vivado_execute'를 통해 Vivado 하드웨어 관리자 Tcl을 통해.

> **Companion:** This design also contains an ILA core — see
> **동반자:** 이 설계에는 ILA 코어도 포함되어 있습니다.
> [`examples/hw-ila-debug/axi-protocol-capture/`](../../hw-ila-debug/axi-protocol-capture/)
> [`예제/hw-ila-debug/axi-protocol-capture/`](../../hw-ila-debug/axi-protocol-capture/)
> for the ILA-focused tutorial using the same design.
> 동일한 디자인을 사용하는 ILA 중심 튜토리얼입니다.

## Prerequisites
## 전제조건

- **Vivado MCP server** — builds the design from source and provides the
- **Vivado MCP 서버** — 소스에서 설계를 구축하고
Hardware Manager Tcl interface for ILA/VIO interaction.
ILA/VIO 상호 작용을 위한 하드웨어 관리자 Tcl 인터페이스.
- **`hw_server`** running on the hardware side (e.g.
- **`hw_server`** 하드웨어 측에서 실행 중(예:
- `TCP:<host>:3121`).
- `TCP:<호스트>:3121`).
- The **`hw-vio-debug` skill** installed (invoked as `/hw-vio-debug`).
- **`hw-vio-debug` 스킬**이 설치되었습니다(`/hw-vio-debug`로 호출됨).
- In this
- 이에
repo it lives at [`skills/hw-vio-debug/`](../../../skills/hw-vio-debug/).
repo는 [`skills/hw-vio-debug/`](../../../skills/hw-vio-debug/)에 있습니다.
- Vivado 2026.1+ installed.
- 비바도 2026.1+가 설치되었습니다.
- A Versal board (developed on VCK190, `xcvc1902-vsva2197-2MP-e-S`).
- Versal 보드(VCK190에서 개발, 'xcvc1902-vsva2197-2MP-e-S').

## Starting Point
## 출발점

Input files in `input/`:
`input/`에 입력 파일:
- `src/axi_lite_master.v` — VIO-controlled AXI4-Lite master.
- `src/axi_lite_master.v` — VIO로 제어되는 AXI4-Lite 마스터.
- On a `start_wr`
- `start_wr`에서
pulse, issues one write (address + data set by VIO).
펄스, 한 번의 쓰기(VIO에 의해 설정된 주소 + 데이터)를 발행합니다.
On `start_rd`, issues
`start_rd`에서 문제 발생
one read.
하나 읽었습니다.
Status fed back to VIO (rd_data, busy, done).
상태는 VIO에 피드백됩니다(rd_data, 사용 중, 완료).
- `src/axis_pkt_gen.v` — VIO-controlled AXI-Stream packet generator.
- `src/axis_pkt_gen.v` — VIO 제어 AXI-Stream 패킷 생성기.
- On a
- 에
`start_stream` pulse, sends one packet of `pkt_length` beats with
`start_stream` 펄스, `pkt_length` 비트의 패킷 하나를 다음과 함께 보냅니다.
incrementing TDATA.
TDATA를 증가시킵니다.
Status: stream_busy, pkt_count.
상태: stream_busy, pkt_count.
- `create_project.tcl` — builds the block design: Versal CIPS + AXI-Lite
- `create_project.tcl` — 블록 디자인 구축: Versal CIPS + AXI-Lite
master → BRAM controller + AXI-Stream generator → FIFO + axis_ila
마스터 → BRAM 컨트롤러 + AXI-Stream 생성기 → FIFO + axis_ila
(System ILA) + axis_vio (VIO, 8 outputs + 5 inputs).
(시스템 ILA) + axis_vio (VIO, 8개 출력 + 5개 입력).

### VIO Probe Map
### VIO 프로브 맵

| Probe<br>프로브 | Direction<br>방향 | Width<br>폭 | Signal<br>신호 |
|-------|-----------|-------|--------|
| probe_out0<br>프로브_아웃0 | OUT<br>아웃 | 1 | start_wr<br>start_wr |
| probe_out1<br>프로브_아웃1 | OUT<br>아웃 | 1 | start_rd<br>start_rd |
| probe_out2<br>프로브_아웃2 | OUT<br>아웃 | 32 | wr_addr<br>wr_addr |
| probe_out3<br>프로브_아웃3 | OUT<br>아웃 | 32 | wr_data<br>wr_data |
| probe_out4<br>프로브_아웃4 | OUT<br>아웃 | 32 | rd_addr<br>rd_addr |
| probe_out5<br>프로브_아웃5 | OUT<br>아웃 | 1 | start_stream<br>시작_스트림 |
| probe_out6<br>프로브_아웃6 | OUT<br>아웃 | 8 | pkt_length<br>pkt_length |
| probe_out7<br>프로브_아웃7 | OUT<br>아웃 | 1 | sink_read_en<br>sing_read_en |
| probe_in0<br>프로브_인0 | IN | 32 | rd_data<br>rd_data |
| probe_in1<br>프로브_인1 | IN | 1 | busy<br>바쁜 |
| probe_in2<br>프로브_인2 | IN | 1 | done<br>완료 |
| probe_in3<br>프로브_인3 | IN | 1 | stream_busy<br>stream_busy |
| probe_in4<br>프로브_인4 | IN | 16 | pkt_count<br>pkt_count |

## How to Run
## 달리는 방법

**Follow the prompts in [`prompt.md`](prompt.md) to run this tutorial end to
**이 튜토리얼을 끝까지 실행하려면 [`prompt.md`](prompt.md)의 프롬프트를 따르세요.
end.** There are four ordered steps:
end.** 순서대로 4가지 단계가 있습니다.

1. **Build** — `Step 1` sources `input/create_project.tcl` and implements to
1. **빌드** — `1단계`는 `input/create_project.tcl`을 소스로 하고 다음을 구현합니다.
`write_device_image`, producing the PDI and LTX.
`write_device_image`, PDI 및 LTX를 생성합니다.
2. **Discover** — `Step 2` uses the `/hw-vio-debug` skill to program the PDI
2. **발견** — `2단계`는 `/hw-vio-debug` 기술을 사용하여 PDI를 프로그래밍합니다.
and list all VIO probes.
모든 VIO 프로브를 나열합니다.
3. **Write + Read** — `Step 3` drives a register write then reads it back.
3. **쓰기 + 읽기** — `3단계`에서는 레지스터 쓰기를 구동한 다음 다시 읽습니다.
4. **Stream** — `Step 4` sends an AXI-Stream packet and checks pkt_count.
4. **스트림** — `4단계`는 AXI-Stream 패킷을 보내고 pkt_count를 확인합니다.

## Expected Behavior
## 예상되는 동작

The `hw-vio-debug` skill will:
`hw-vio-debug` 스킬은 다음과 같습니다:
1. Connect to `hw_server` and program the PDI via Vivado Hardware Manager.
1. 'hw_server'에 연결하고 Vivado Hardware Manager를 통해 PDI를 프로그래밍하세요.
2. Associate the LTX probes file and discover the VIO core (hw_vio_1).
2. LTX 프로브 파일을 연결하고 VIO 코어(hw_vio_1)를 검색합니다.
3. List all 13 probes with names, directions, widths, and current values.
3. 이름, 방향, 너비 및 현재 값과 함께 프로브 13개를 모두 나열합니다.
4. Drive wr_addr=0x20, wr_data=0xDEADBEEF, pulse start_wr → busy=1→0.
4. 드라이브 wr_addr=0x20, wr_data=0xDEADBEEF, 펄스 start_wr → busy=1→0.
5. Drive rd_addr=0x20, pulse start_rd → rd_data=`0xDEADBEEF` (round-trip).
5. rd_addr=0x20, 펄스 start_rd → rd_data=`0xDEADBEEF`(왕복)을 구동합니다.
6. Set pkt_length=16, pulse start_stream → pkt_count increments to 1.
6. pkt_length=16으로 설정하면 pulse start_stream → pkt_count가 1로 증가합니다.
