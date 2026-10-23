<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# ASSIGN-9: Unassigned Output Port Bits
# ASSIGN-9: 할당되지 않은 출력 포트 비트

**Rule:** ASSIGN-9 | **Severity:** WARNING | **Standard:** UG901
**규칙:** ASSIGN-9 | **심각도:** 경고 | **표준:** UG901

## Vivado Message
## 비바도 메시지
```
WARNING: [Synth 37-126] [ASSIGN-9] Some bits in IO 'y1' are not set. 
First unset bit index is -8. RTL Name 'y1', Hierarchy 'top', File 'top.v', Line 8.
```

## Root Cause
## 근본 원인

Output port has no driver (or only partial bits driven).
출력 포트에는 드라이버가 없습니다(또는 부분 비트만 구동됨).
Creates undefined/floating values.
정의되지 않은/부동 값을 생성합니다.

**Determine scenario first:** Check if ANY bits are assigned to this port.
**시나리오를 먼저 결정하십시오.** 이 포트에 할당된 비트가 있는지 확인하십시오.

## Fix Options
## 수정 옵션

### Scenario A: Completely Unassigned Output → Remove Port
### 시나리오 A: 완전히 할당되지 않은 출력 → 포트 제거

```verilog
// Before
module top(a, b, clk, unused_out);
   output [7:0] unused_out;  // No driver anywhere

// After — remove unused port
module top(a, b, clk);
   // Removed unused_out — update parent instantiations
```

### Scenario B: Partially Assigned → Complete the Assignment
### 시나리오 B: 부분적으로 할당됨 → 할당 완료

Identify which bits ARE assigned, then fill missing bits:
할당된 비트를 식별한 다음 누락된 비트를 채웁니다.

```verilog
// B1: Lower assigned, upper missing
output [3:0] out;
always@(*) out[2:0] = a[2:0];     // Bit [3] missing
// Fix: always@(*) out = {1'b0, a[2:0]};

// B2: Upper assigned, lower missing
always@(*) out[3:1] = a[2:0];     // Bit [0] missing
// Fix: always@(*) out = {a[2:0], 1'b0};

// B3: Non-contiguous gaps
always@(*) begin
   out[7:5] = a[2:0];
   out[2:0] = b[2:0];             // Bits [4:3] missing
end
// Fix: Add out[4:3] = 2'b00;
```

**Padding value:** Zero (default), sign extension (signed arithmetic), or constant (protocol requirements).
**패딩 값:** 0(기본값), 부호 확장(부호 있는 산술) 또는 상수(프로토콜 요구 사항).

## Validation
## 검증

```tcl
synth_design -top <module> -part <part> -lint -file lint_recheck.rpt
# Verify: ASSIGN-9 count = 0
```

For Scenario A: update parent module instantiations to remove port connection.
시나리오 A의 경우: 상위 모듈 인스턴스화를 업데이트하여 포트 연결을 제거합니다.

## References
## 참고자료
**Xilinx:** UG901 "I/O Assignment Best Practices", UG949
**Xilinx:** UG901 "I/O 할당 모범 사례", UG949
**Related:** ASSIGN-10 (unused input port bits)
**관련:** ASSIGN-10(사용되지 않은 입력 포트 비트)
