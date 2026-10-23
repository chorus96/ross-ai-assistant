<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# ASSIGN-2: Mixed Signed/Unsigned Arithmetic
# ASSIGN-2: 부호 있는/부호 없는 혼합 산술

**Rule:** ASSIGN-2 | **Severity:** CRITICAL WARNING | **Standard:** IEEE 1364-2005 §4.1
**규칙:** ASSIGN-2 | **심각도:** 심각한 경고 | **표준:** IEEE 1364-2005 §4.1

## Vivado Message
## 비바도 메시지
```
CRITICAL WARNING: [Synth 37-81] [ASSIGN-2]
Operator '+' in expression '((a + b) + c)' has mixed signed and unsigned types 
on its operands, the result will be unsigned.
```

## Root Cause
## 근본 원인

Arithmetic combines signed and unsigned operands.
연산은 부호 있는 피연산자와 부호 없는 피연산자를 결합합니다.
Per Verilog: if ANY operand is unsigned, ALL promote to unsigned — signed values lose sign information.
Verilog 기준: 피연산자 중 하나라도 부호가 없으면 모두 부호 없는 값으로 승격됩니다. 부호 있는 값은 부호 정보를 잃습니다.

**Common sources:** Port declaration inconsistency, implicit unsigned defaults, unsigned literals (`8'd5` vs `8'sd5`), bit-slicing (always strips signedness)
**공통 소스:** 포트 선언 불일치, 암시적 부호 없는 기본값, 부호 없는 리터럴(`8'd5` 대 `8'sd5`), 비트 슬라이싱(항상 부호 있는 부분 제거)

**Bit-slicing caveat:** `c[6:0]` is always unsigned even if `c` is signed.
**비트 분할 주의 사항:** `c[6:0]`은 `c`가 서명되어 있어도 항상 부호가 없습니다.
Cast explicitly: `$signed(c[6:0])`.
명시적으로 캐스트: `$signed(c[6:0])`.

## Fix: Explicit `$signed()` Casting
## 수정: 명시적 `$signed()` 캐스팅

Cast unsigned operands to signed.
부호 없는 피연산자를 부호 있는 피연산자로 캐스팅합니다.
Zero hardware overhead (semantic-only).
하드웨어 오버헤드가 없습니다(의미론적 전용).

```verilog
// Before: mixed signedness
input [6:0] a;            // Unsigned
input signed [6:0] b;     // Signed
result = a + b + c;       // All promote to unsigned — WRONG for negatives

// After: explicit casting
result = $signed(a) + b + c;  // All operands now signed — CORRECT
```

**Multiple unsigned operands:** Cast each one:
**부호 없는 여러 피연산자:** 각각을 캐스팅합니다.
```verilog
result = $signed(a) + $signed(b) + c;
```

**Operator coverage:** Works for `+`, `-`, `*`, `>>>`, comparisons.
**연산자 적용 범위:** `+`, `-`, `*`, `>>>`, 비교에 작동합니다.

**Optional:** Declare outputs as signed for downstream correctness:
**선택 사항:** 다운스트림 정확성을 위해 출력을 서명된 것으로 선언합니다.
```verilog
output signed [7:0] y1;
```

## Validation
## 검증

```tcl
synth_design -top <module> -part <part> -lint -file lint_recheck.rpt
# Verify: ASSIGN-2 count = 0
```

Test with negative values to verify correct arithmetic.
올바른 연산을 확인하려면 음수 값으로 테스트하세요.
`$signed()` adds zero LUT/FF overhead.
`$signed()`는 LUT/FF 오버헤드를 0으로 추가합니다.

## References
## 참고자료
**Xilinx:** UG901 "Signed and Unsigned Arithmetic", UG949
**자일링스:** UG901 "부호 있는 산술 및 부호 없는 산술", UG949
**IEEE:** 1364-2005 §4.1-4.2, 1800-2017 §11.8
**IEEE:** 1364-2005 §4.1-4.2, 1800-2017 §11.8
