<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# ASSIGN-3: Shift Overflow
# ASSIGN-3: 시프트 오버플로

**Rule:** ASSIGN-3 | **Severity:** CRITICAL WARNING | **Standard:** IEEE 1364-2005 §5.1.13
**규칙:** ASSIGN-3 | **심각도:** 심각한 경고 | **표준:** IEEE 1364-2005 §5.1.13

## Vivado Message
## 비바도 메시지
```
CRITICAL WARNING: [Synth 37-92] [ASSIGN-3]
Right-hand side of operator >> in expression (signal >> SHIFT_AMOUNT) 
exceeds the range of left-hand side. LHS width: N, RHS value: M.
```

## Root Cause
## 근본 원인

Shift amount ≥ signal bit width.
이동량 ≥ 신호 비트 폭.
Per IEEE 1364-2005 §5.1.13, logical shift by amount ≥ width always produces zero — creating dead logic or incorrect conditionals.
IEEE 1364-2005 §5.1.13에 따라 양 ≥ 너비에 따른 논리적 이동은 항상 0을 생성하여 데드 로직 또는 잘못된 조건을 생성합니다.

**Common sources:** Parameter misconfiguration, copy-paste from different-width design, missing width validation
**공통 소스:** 잘못된 매개변수 구성, 다양한 너비 디자인에서 복사하여 붙여넣기, 너비 유효성 검사 누락

## Fix Options
## 수정 옵션

### Option 1: Correct Shift Parameter
### 옵션 1: 올바른 Shift 매개변수

```verilog
// Before
parameter SHIFT_AMOUNT = 30;          // Exceeds 13-bit width
if (data_in >> SHIFT_AMOUNT) ...      // Always 0

// After
parameter SHIFT_AMOUNT = 2;           // Valid range: 0 to width-1
if (data_in >> SHIFT_AMOUNT) ...      // Functionally correct
```

### Option 2: Add Validation (Parameterized designs)
### 옵션 2: 검증 추가(매개변수화된 설계)

```verilog
localparam WIDTH = $bits(din);
localparam MAX_SHIFT = WIDTH - 1;
localparam SAFE_SHIFT = (SHIFT > MAX_SHIFT) ? MAX_SHIFT : SHIFT;

assign dout = din >> SAFE_SHIFT;

initial if (SHIFT > MAX_SHIFT)
   $warning("SHIFT (%0d) exceeds MAX (%0d), clipped", SHIFT, MAX_SHIFT);
```

**Runtime variable shifts:**
**런타임 변수 이동:**
```verilog
assign dout = (shift_amt >= WIDTH) ? 0 : (din >> shift_amt);
```

## Validation
## 검증

```tcl
synth_design -top <module> -part <part> -lint -file lint_recheck.rpt
# Verify: ASSIGN-3 count = 0
```

## References
## 참고자료
**Xilinx:** UG901 Ch.3 "Shift Operators", UG906 Appendix B
**자일링스:** UG901 Ch.3 "교대 근무자", UG906 부록 B
**IEEE:** 1364-2005 §5.1.13, 1800-2017 §11.4.10
**IEEE:** 1364-2005 §5.1.13, 1800-2017 §11.4.10
