<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->
# ILA AXI Protocol Capture — Quick Start Prompts
# ILA AXI 프로토콜 캡처 - 빠른 시작 프롬프트

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

## Step 2 — Program the board and capture an AXI-Lite write
## 2단계 - 보드 프로그래밍 및 AXI-Lite 쓰기 캡처

```
Use /hw-ila-debug to program input/axi_debug/axi_debug.runs/impl_1/axi_debug_bd_wrapper.pdi
onto the board via Vivado Hardware Manager. Associate the LTX probes file.
Then trigger the ILA on a rising edge of AWVALID. Use the VIO to drive a
write to address 0x10 with data 0xCAFEBABE, capture the ILA waveform, and
export to CSV.
```

## Step 3 — Capture an AXI-Stream packet
## 3단계 - AXI-Stream 패킷 캡처

```
Now trigger the ILA on a rising edge of TVALID. Use the VIO to send an
8-beat AXI-Stream packet (pkt_length=8). Capture the waveform showing
TDATA incrementing and TLAST on the final beat. Export to CSV.
```
