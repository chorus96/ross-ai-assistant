<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->
# VIO AXI Register Read/Write — Quick Start Prompts
# VIO AXI 레지스터 읽기/쓰기 - 빠른 시작 프롬프트

The example ships as source only (RTL + Tcl).
예제는 소스로만 제공됩니다(RTL + Tcl).
Build the PDI first, then debug it.
먼저 PDI를 빌드한 다음 디버그하세요.

## Step 1 — Build the PDI from source
## 1단계 - 소스에서 PDI 구축

Either run it yourself:
직접 실행해 보세요.

```bash
cd input
vivado -mode batch -source create_project.tcl
# then, in the same session or a new -mode tcl session:
#   launch_runs impl_1 -to_step write_device_image -jobs 8
#   wait_on_run impl_1
```

or ask the agent:
또는 상담원에게 문의하세요.

```
Build this example: source input/create_project.tcl in Vivado, then run
implementation to write_device_image to produce the PDI.
```

Resulting PDI and LTX:
결과 PDI 및 LTX:

```
input/axi_debug/axi_debug.runs/impl_1/axi_debug_bd_wrapper.pdi
input/axi_debug/axi_debug.runs/impl_1/axi_debug_bd_wrapper.ltx
```

## Step 2 — Program the board and discover VIO probes
## 2단계 - 보드 프로그래밍 및 VIO 프로브 검색

```
Use /hw-vio-debug to program input/axi_debug/axi_debug.runs/impl_1/axi_debug_bd_wrapper.pdi
onto the board via Vivado Hardware Manager. Associate the LTX probes file.
Discover the VIO core and list all input and output probes with their
current values.
```

## Step 3 — Write and read back a register
## 3단계 - 레지스터 쓰기 및 다시 읽기

```
Using the VIO, write 0xDEADBEEF to BRAM address 0x20, then read it back.
Report the rd_data value to confirm the round-trip.
```

## Step 4 — Send an AXI-Stream packet and check count
## 4단계 - AXI-Stream 패킷을 보내고 개수를 확인합니다.

```
Using the VIO, send a 16-beat AXI-Stream packet. Report the pkt_count
and stream_busy status after the transfer completes.
```
