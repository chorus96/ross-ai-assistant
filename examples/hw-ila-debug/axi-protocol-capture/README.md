<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->
# ILA AXI Protocol Capture
# ILA AXI 프로토콜 캡처

A design with an AXI-Lite master, an AXI-Stream packet generator, and a
AXI-Lite 마스터, AXI-Stream 패킷 생성기 및
**System ILA** monitoring both buses, then debugged with the `hw-ila-debug`
**시스템 ILA** 두 버스를 모두 모니터링한 후 'hw-ila-debug'로 디버깅함
skill.
기술.

Traffic is driven interactively through a VIO core so the user controls
트래픽은 VIO 코어를 통해 대화형으로 구동되므로 사용자가 제어할 수 있습니다.
exactly when and what the ILA captures.
ILA가 정확히 언제, 무엇을 캡처하는지.
The design source in `input/` is
`input/`의 디자인 소스는 다음과 같습니다.
complete and self-contained (RTL + Tcl).
완전하고 독립적입니다(RTL + Tcl).
The debug step depends on the
디버그 단계는 다음에 따라 다릅니다.
**`hw-ila-debug` skill**, which lives at
**`hw-ila-debug` 스킬**,
[`skills/hw-ila-debug/`](../../../skills/hw-ila-debug/) in this repo.
[`skills/hw-ila-debug/`](../../../skills/hw-ila-debug/) 이 저장소에 있습니다.

## Goal
## 목표

Use the ILA to capture live AXI protocol transactions on hardware:
ILA를 사용하여 하드웨어에서 실시간 AXI 프로토콜 트랜잭션을 캡처합니다.

1. **AXI-Lite write** — trigger on `AWVALID` rising edge, capture the full
1. **AXI-Lite 쓰기** — 'AWVALID' 상승 에지에서 트리거하고 전체 캡처
write handshake (AWADDR → AWREADY → WDATA → WREADY → BRESP).
핸드셰이크를 작성합니다(AWADDR → AWREADY → WDATA → WREADY → BRESP).
2. **AXI-Stream packet** — trigger on `TVALID` rising edge, capture a
2. **AXI-스트림 패킷** — 'TVALID' 상승 에지에서 트리거하고
multi-beat packet with incrementing TDATA and TLAST on the final beat.
마지막 비트에서 TDATA 및 TLAST가 증가하는 다중 비트 패킷입니다.
3. **Export** — write captured data to CSV for offline analysis.
3. **내보내기** — 오프라인 분석을 위해 캡처된 데이터를 CSV에 씁니다.

## Skills Used
## 사용된 스킬

- **hw-ila-debug** — discovers ILA cores and probes, configures triggers,
- **hw-ila-debug** — ILA 코어 및 프로브를 검색하고, 트리거를 구성합니다.
arms the ILA, waits for trigger, uploads captured data, and exports to CSV.
ILA를 준비하고, 트리거를 기다리고, 캡처된 데이터를 업로드하고, CSV로 내보냅니다.
All operations go through Vivado Hardware Manager Tcl via `vivado_execute`.
모든 작업은 `vivado_execute`를 통해 Vivado Hardware Manager Tcl을 통해 진행됩니다.

> **Companion:** This design also contains a VIO core — see
> **동반자:** 이 설계에는 VIO 코어도 포함되어 있습니다.
> [`examples/hw-vio-debug/axi-register-rw/`](../../hw-vio-debug/axi-register-rw/)
> [`예제/hw-vio-debug/axi-register-rw/`](../../hw-vio-debug/axi-register-rw/)
> for the VIO-focused tutorial using the same design.
> 동일한 디자인을 사용하는 VIO 중심 튜토리얼입니다.

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
- The **`hw-ila-debug` skill** installed (invoked as `/hw-ila-debug`).
- **`hw-ila-debug` 스킬**이 설치되었습니다(`/hw-ila-debug`로 호출됨).
- In this
- 이에
repo it lives at [`skills/hw-ila-debug/`](../../../skills/hw-ila-debug/).
repo는 [`skills/hw-ila-debug/`](../../../skills/hw-ila-debug/)에 있습니다.
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
(System ILA, 2 monitor slots) + axis_vio (VIO).
(시스템 ILA, 모니터 슬롯 2개) + axis_vio(VIO).

## How to Run
## 달리는 방법

**Follow the prompts in [`prompt.md`](prompt.md) to run this tutorial end to
**이 튜토리얼을 끝까지 실행하려면 [`prompt.md`](prompt.md)의 프롬프트를 따르세요.
end.** There are three ordered steps:
end.** 순서대로 세 가지 단계가 있습니다.

1. **Build** — `Step 1` sources `input/create_project.tcl` and implements to
1. **빌드** — `1단계`는 `input/create_project.tcl`을 소스로 하고 다음을 구현합니다.
`write_device_image`, producing the PDI and LTX.
`write_device_image`, PDI 및 LTX를 생성합니다.
2. **ILA AXI-Lite capture** — `Step 2` uses the `/hw-ila-debug` skill to
2. **ILA AXI-Lite 캡처** — `2단계`는 `/hw-ila-debug` 기술을 사용하여
program the PDI, trigger on AWVALID, drive a write via VIO, and capture
PDI 프로그래밍, AWVALID 트리거, VIO를 통한 쓰기 구동 및 캡처
the AXI-Lite handshake.
AXI-Lite 핸드셰이크.
3. **ILA AXI-Stream capture** — `Step 3` re-triggers on TVALID and captures
3. **ILA AXI-Stream 캡처** — '3단계'는 TVALID에서 다시 트리거되고 캡처됩니다.
a streaming packet.
스트리밍 패킷.

## Expected Behavior
## 예상되는 동작

The `hw-ila-debug` skill will:
`hw-ila-debug` 스킬은 다음과 같습니다:
1. Connect to `hw_server` and program the PDI via Vivado Hardware Manager.
1. 'hw_server'에 연결하고 Vivado Hardware Manager를 통해 PDI를 프로그래밍하세요.
2. Associate the LTX probes file and discover the ILA core (hw_ila_1).
2. LTX 프로브 파일을 연결하고 ILA 코어(hw_ila_1)를 검색합니다.
3. Set trigger on `AWVALID` rising edge, arm the ILA.
3. 'AWVALID' 상승 에지에 트리거를 설정하고 ILA를 준비합니다.
4. Drive a write (addr=0x10, data=0xCAFEBABE) via VIO to fire the trigger.
4. VIO를 통해 쓰기(addr=0x10, data=0xCAFEBABE)를 실행하여 트리거를 실행합니다.
5. Upload and report: AWADDR=`0x00000010`, WDATA=`0xCAFEBABE`, BVALID pulse.
5. 업로드 및 보고: AWADDR=`0x00000010`, WDATA=`0xCAFEBABE`, BVALID 펄스.
6. Re-trigger on `TVALID`, send an 8-beat stream packet via VIO.
6. 'TVALID'를 다시 트리거하고 VIO를 통해 8비트 스트림 패킷을 보냅니다.
7. Upload and report: TDATA increments 0x0→0x7, TLAST on beat 8.
7. 업로드 및 보고: TDATA는 0x0→0x7, 비트 8의 TLAST를 증가시킵니다.
