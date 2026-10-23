<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# ASSIGN-10: Unused I/O Port Bits
# ASSIGN-10: 사용되지 않은 I/O 포트 비트

**Rule:** ASSIGN-10 | **Severity:** WARNING | **Standard:** UG901 Ch.4, IEEE 1364-2005 §12.3
**규칙:** ASSIGN-10 | **심각도:** 경고 | **표준:** UG901 Ch.4, IEEE 1364-2005 §12.3

## Vivado Message
## 비바도 메시지
```
WARNING: [Synth 37-125] [ASSIGN-10]
Some bits in IO '(port_name)' are not read. First unread bit index is '(bit_index)'.
RTL Name '(port_name)', Hierarchy '(module_name)', File '(file.v)', Line (line_num).
```

## Root Cause
## 근본 원인

Port declared but never used (or only some bits used) in design logic.
포트가 선언되었지만 디자인 로직에서 사용된 적이 없습니다(또는 일부 비트만 사용됨).
Wastes routing resources and adds unnecessary interface complexity.
라우팅 리소스를 낭비하고 불필요한 인터페이스 복잡성을 추가합니다.

**Common sources:** Leftover ports from earlier design iteration, copy-paste from reference design, port width too large for actual usage
**공통 소스:** 이전 설계 반복에서 남은 포트, 참조 설계에서 복사하여 붙여넣기, 실제 사용하기에는 포트 너비가 너무 큼

## Fix Options
## 수정 옵션

### Option 1: Remove Unused Port (When port serves no purpose)
### 옵션 1: 사용하지 않는 포트 제거(포트가 아무런 용도로 사용되지 않는 경우)

```diff
 module data_processor(
    input clk,
    input [7:0] data_in,
-   input [7:0] control,      // Never read — remove
    output reg [7:0] result
 );
```

**Update all parent instantiations:**
**모든 상위 인스턴스화 업데이트:**
```diff
-data_processor u1(.clk(clk), .data_in(din), .control(ctrl), .result(res));
+data_processor u1(.clk(clk), .data_in(din), .result(res));
```

### Option 2: Reduce Port Width (When only some bits used)
### 옵션 2: 포트 너비 줄이기(일부 비트만 사용되는 경우)

```diff
 module byte_processor(
-   input [7:0] config,       // Only bits [1:0] actually used
+   input [1:0] config,       // Reduced to actual usage
    input [7:0] data,
    output reg [7:0] result
 );
```

Update parent instantiations to pass only used bits: `.config(cfg[1:0])`.
사용된 비트만 전달하도록 상위 인스턴스화를 업데이트합니다: `.config(cfg[1:0])`.

## Validation
## 검증

```tcl
synth_design -top <module> -part <part> -lint -file lint_recheck.rpt
# Verify: ASSIGN-10 count = 0
# Search for all instantiations of modified module:
grep -r "module_name" src/ --include="*.v" --include="*.sv"
```

## References
## 참고자료
**Xilinx:** UG901 Ch.4 "Port Declarations", UG949 "Interface Optimization"
**자일링스:** UG901 Ch.4 "포트 선언", UG949 "인터페이스 최적화"
**IEEE:** 1364-2005 §12.3, 1800-2017 §23.2.2
**IEEE:** 1364-2005 §12.3, 1800-2017 §23.2.2
**Related:** ASSIGN-6 (signal not read), ASSIGN-9 (unassigned output)
**관련:** ASSIGN-6(신호를 읽지 않음), ASSIGN-9(할당되지 않은 출력)
