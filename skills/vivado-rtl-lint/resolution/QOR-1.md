<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# QOR-1: Arithmetic Precision Loss
# QOR-1: 산술 정밀도 손실

**Rule:** QOR-1 | **Severity:** WARNING | **Standard:** UG901 Ch.4
**규칙:** QOR-1 | **심각도:** 경고 | **표준:** UG901 Ch.4

## Vivado Message
## 비바도 메시지
```
WARNING: [Synth 37-74] [QOR-1]
Arithmetic Operator '(in1 + in2)' resulted in loss of precision. 
This prevented merging optimization to its load operator '(temp + in3)' 
from happening and may degrade QoR. 
Full precision width of preceding operator is 11, however only 10 bits are selected.
```

## Root Cause
## 근본 원인

Intermediate arithmetic result truncated before the next operation, preventing the synthesizer from merging operator trees.
다음 연산 전에 중간 산술 결과가 잘려 합성기가 연산자 트리를 병합하는 것을 방지합니다.
Results in separate adder stages instead of a single merged tree — increased logic depth, larger area, worse timing.
단일 병합 트리 대신 별도의 가산기 단계가 발생합니다. 즉, 논리 깊이가 증가하고 영역이 넓어지며 타이밍이 나빠집니다.

**Common sources:** Intermediate signal sized for final output (not for computation), copy-paste from different-width design
**공통 소스:** 최종 출력용 중간 신호 크기(계산용 아님), 다양한 너비 설계에서 복사하여 붙여넣기

## Fix Options
## 수정 옵션

### Option 1: Preserve Full Precision (When intermediate signal needed)
### 옵션 1: 전체 정밀도 유지(중간 신호가 필요한 경우)

```verilog
// Before: truncates carry bit
wire [P1-1:0] w1;              // 10 bits — too narrow
assign w1 = in1 + in2;        // 11-bit result truncated
assign out = w1 + in3;        // Cannot merge with previous

// After: full precision intermediate
wire [P1:0] w1_full;           // 11 bits — preserves carry
assign w1_full = in1 + in2;   // Full result
assign out = w1_full + in3;   // Can now merge operators
```

**Auto-calculate width:**
**너비 자동 계산:**
```verilog
localparam W1_WIDTH = P1 + 1;
wire [W1_WIDTH-1:0] w1_full;
```

### Option 2: Remove Intermediate (When signal not needed elsewhere)
### 옵션 2: 중간 제거(다른 곳에서는 신호가 필요하지 않은 경우)

```verilog
// Before
wire [9:0] w1 = in1 + in2;      // Unnecessary intermediate
wire [10:0] w2 = w1 + in3;

// After: single merged expression
wire [10:0] w2 = (in1 + in2) + in3;  // Single operator tree
```

### Decision Guide
### 결정 가이드

| Scenario<br>시나리오 | Fix<br>수정 |
|----------|-----|
| Intermediate used elsewhere<br>다른 곳에서 사용되는 중간체 | Option 1: Widen intermediate<br>옵션 1: 중급 확대 |
| Intermediate only feeds next op<br>중급은 다음 작업에만 피드를 제공합니다. | Option 2: Remove intermediate<br>옵션 2: 중간체 제거 |
| Truncation architecturally required<br>구조적으로 잘림이 필요함 | Keep but use full precision for next op<br>유지하되 다음 작업에는 전체 정밀도를 사용하세요. |

## Validation
## 검증

```tcl
synth_design -top <module> -part <part> -lint -file lint_recheck.rpt
# Verify: QOR-1 count = 0
report_utilization -hierarchical       # Expect LUT reduction
report_design_analysis -logic_level_distribution  # Expect reduced logic levels
```

## References
## 참고자료
**Xilinx:** UG901 Ch.4 "Arithmetic Operators", UG906 Appendix B, UG949 "Operator Merging"
**Xilinx:** UG901 Ch.4 "산술 연산자", UG906 부록 B, UG949 "연산자 병합"
**IEEE:** 1364-2005 §5.5, 1800-2017 §11.4.4
**IEEE:** 1364-2005 §5.5, 1800-2017 §11.4.4
**Related:** ASSIGN-1 (arithmetic overflow), ASSIGN-2 (width mismatch)
**관련:** ASSIGN-1(산술 오버플로), ASSIGN-2(너비 불일치)
