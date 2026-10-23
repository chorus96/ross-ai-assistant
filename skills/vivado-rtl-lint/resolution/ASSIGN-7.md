<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# ASSIGN-7: Multiple Drivers / Overlapping Bit Ranges
# ASSIGN-7: 다중 드라이버/겹치는 비트 범위

**Rule:** ASSIGN-7 | **Severity:** CRITICAL WARNING | **Standard:** UG901 Ch.4
**규칙:** ASSIGN-7 | **심각도:** 심각한 경고 | **표준:** UG901 Ch.4

## Vivado Message
## 비바도 메시지
```
CRITICAL WARNING: [Synth 37-90] [ASSIGN-7] Multiple assignments detected on array y1, 
first multi-driven bit index is 3. RTL Name 'y1', Hierarchy 'top', File 'top.v', Line 8.
```

## Root Cause
## 근본 원인

Multiple always blocks assign to overlapping bits of the same signal.
여러 개의 항상 블록은 동일한 신호의 겹치는 비트에 할당됩니다.
Each register bit must have exactly ONE driver — multiple drivers create race conditions and non-deterministic behavior.
각 레지스터 비트에는 정확히 하나의 드라이버가 있어야 합니다. 즉, 여러 드라이버가 경쟁 조건과 비결정적 동작을 생성합니다.

## Fix: Adjust to Non-Overlapping Bit Ranges
## 수정: 겹치지 않는 비트 범위로 조정

Identify the overlapping bit from the message, then adjust one range to eliminate overlap.
메시지에서 중복되는 비트를 식별한 다음 한 범위를 조정하여 중복을 제거합니다.

```verilog
// Before: bit 3 driven by both blocks
always@(posedge clk) signal[3:0] <= source_a[3:0];
always@(posedge clk) signal[7:3] <= source_b[7:3];  // Overlap at bit 3

// After (Option A: adjust lower range)
always@(posedge clk) signal[2:0] <= source_a[2:0];  // Bits 2:0
always@(posedge clk) signal[7:3] <= source_b[7:3];  // Bits 7:3

// After (Option B: adjust upper range)
always@(posedge clk) signal[3:0] <= source_a[3:0];  // Bits 3:0
always@(posedge clk) signal[7:4] <= source_b[7:4];  // Bits 7:4
```

Choose boundary based on natural field alignment (nibble/byte) and minimal changes.
자연스러운 필드 정렬(니블/바이트) 및 최소 변경 사항을 기반으로 경계를 선택합니다.

## Validation
## 검증

```tcl
synth_design -top <module> -part <part> -lint -file lint_recheck.rpt
# Verify: ASSIGN-7 count = 0
```

## References
## 참고자료
**Xilinx:** UG901 Ch.4 "Sequential Logic Coding — One driver per bit"
**Xilinx:** UG901 Ch.4 "순차 논리 코딩 - 비트당 하나의 드라이버"
