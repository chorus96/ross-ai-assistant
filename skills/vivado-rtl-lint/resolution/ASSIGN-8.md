<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# ASSIGN-8: Array Dimension Mismatch in Comparison
# ASSIGN-8: 비교 시 배열 크기 불일치

**Rule:** ASSIGN-8 | **Severity:** CRITICAL WARNING | **Standard:** IEEE 1800-2017 §11.4.5
**규칙:** ASSIGN-8 | **심각도:** 심각한 경고 | **표준:** IEEE 1800-2017 §11.4.5

## Vivado Message
## 비바도 메시지
```
CRITICAL WARNING: [Synth 37-100] [ASSIGN-8]
Operator '<op>' in expression '(operand1 <op> operand2)' has arrays 
of different dimensions as operands.
```
Applies to: `==`, `!=`, `===`, `!==`, `>=`, `>`, `<`, `<=`
적용 대상: `==`, `!=`, `===`, `!==`, `>=`, `>`, `<`, `<=`

## Root Cause
## 근본 원인

Comparison operators applied to arrays with mismatched dimensional structure.
차원 구조가 일치하지 않는 배열에 적용되는 비교 연산자입니다.
Per IEEE 1800-2017: if operands have different dimensions, the result is undefined.
IEEE 1800-2017에 따라: 피연산자의 차원이 다른 경우 결과는 정의되지 않습니다.

**Common sources:** Port declaration mismatch (1D vs 2D), copy-paste errors, packed array syntax confusion (`[M:0][N:0]` vs `[M:0]`)
**공통 소스:** 포트 선언 불일치(1D vs 2D), 복사-붙여넣기 오류, 압축 배열 구문 혼동(`[M:0][N:0]` vs `[M:0]`)

## Fix Options
## 수정 옵션

### Option 1: Match Operand Dimensions (Most common)
### 옵션 1: 피연산자 차원 일치(가장 일반적)

```verilog
// Before: 1D (16-bit) vs 2D (128-bit)
module test(input [15:0] in1, input [15:0][7:0] in2, output wire out1);
   assign out1 = in1 == in2;   // Undefined behavior

// After: both 1D
module test(input [15:0] in1, input [15:0] in2, output wire out1);
   assign out1 = in1 == in2;   // Valid: 16-bit == 16-bit
```

Update parent module to pass matching-dimension signal.
일치 차원 신호를 전달하도록 상위 모듈을 업데이트합니다.

### Option 2: Flatten Array (When 2D structure needed elsewhere)
### 옵션 2: 배열 편평화(2D 구조가 다른 곳에 필요한 경우)

```verilog
wire [127:0] in2_flat;
genvar i;
generate
   for (i = 0; i < 16; i = i + 1) begin : flatten
      assign in2_flat[i*8 +: 8] = in2[i];
   end
endgenerate
assign out1 = in1 == in2_flat[15:0];  // Compare specific bits
```

### Option 3: Element-Wise Comparison
### 옵션 3: 요소별 비교

```verilog
wire [15:0] matches;
genvar i;
generate
   for (i = 0; i < 16; i = i + 1) begin : cmp
      assign matches[i] = (in1[i] == in2[i][0]);
   end
endgenerate
assign out1 = &matches;  // All elements match
```

## Validation
## 검증

```tcl
synth_design -top <module> -part <part> -lint -file lint_recheck.rpt
# Verify: ASSIGN-8 count = 0
```

## References
## 참고자료
**Xilinx:** UG901 Ch.2 "Arrays and Memory", UG906 Appendix B
**Xilinx:** UG901 Ch.2 "어레이 및 메모리", UG906 부록 B
**IEEE:** 1800-2017 §11.4.4 (Relational), §11.4.5 (Equality), §7.4 (Packed arrays)
**IEEE:** 1800-2017 §11.4.4(관계형), §11.4.5(동등성), §7.4(패키지 배열)
