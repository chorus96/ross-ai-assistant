<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# ASSIGN-5: Undriven Signal
# ASSIGN-5: 구동되지 않은 신호

**Rule:** ASSIGN-5 | **Severity:** WARNING | **Impact:** Functional bug (X-propagation)
**규칙:** ASSIGN-5 | **심각도:** 경고 | **영향:** 기능적 버그(X-전파)

## Vivado Message
## 비바도 메시지
```
WARNING: [Synth 37-95] [ASSIGN-5] Some bits in '(signal_name)' are not set. 
First unset bit index is '(bit_index)'.
RTL Name '(signal_name)', Hierarchy '(module)', File '(file.v)', Line (N).
```

## Root Cause
## 근본 원인

Signal declared but never assigned a value.
신호가 선언되었지만 값이 할당되지 않았습니다.
Produces X in simulation and undefined hardware behavior. **Higher priority than ASSIGN-6** — this is a functional bug, not just dead code.
시뮬레이션 및 정의되지 않은 하드웨어 동작에서 X를 생성합니다. **ASSIGN-6보다 높은 우선순위** — 이는 단순한 데드 코드가 아닌 기능적 버그입니다.

**Common sources:** Missing driver, incomplete struct initialization, conditional assignment gaps (missing else/default)
**공통 소스:** 드라이버 누락, 불완전한 구조체 초기화, 조건부 할당 공백(else/기본값 누락)

## Fix Options
## 수정 옵션

**Detection step (mandatory):** Check if signal is actually used:
**감지 단계(필수):** 신호가 실제로 사용되는지 확인합니다.
```bash
grep -n "signal_name" file.v | wc -l
```
- **1 match** (declaration only) → Option 1: Remove
- **일치 1개**(선언만) → 옵션 1: 제거
- **2+ matches** (declaration + usage) → Option 2: Add assignment
- **2개 이상의 일치**(선언 + 사용법) → 옵션 2: 할당 추가

Show ONLY the applicable fix in the report.
보고서에는 해당 수정 사항만 표시됩니다.

### Option 1: Remove Unused Signal
### 옵션 1: 사용하지 않는 신호 제거

```diff
-wire [7:0] w1;    // Never assigned or used — remove
```

### Option 2: Add Missing Assignment
### 옵션 2: 누락된 과제 추가

```verilog
// Before
wire [8:0] sum;
assign y1 = sum[6:0];     // sum is undefined

// After
wire [8:0] sum;
assign sum = {2'b0, a} + {2'b0, b};    // Added driver
assign y1 = sum[6:0];
```

For sequential logic, ensure all paths assign the signal and add reset values.
순차 로직의 경우 모든 경로가 신호를 할당하고 재설정 값을 추가하는지 확인하십시오.

## Validation
## 검증

```tcl
synth_design -top <module> -part <part> -lint -file lint_recheck.rpt
# Verify: ASSIGN-5 count = 0
report_drc -ruledeck default_checks
```

## References
## 참고자료
**Xilinx:** UG901 "Signal Assignment", UG949
**자일링스:** UG901 "신호 할당", UG949
**Related:** ASSIGN-6 (opposite: assigned but not read), ASSIGN-10 (unused ports)
**관련:** ASSIGN-6(반대: 할당되었지만 읽히지 않음), ASSIGN-10(사용되지 않는 포트)
