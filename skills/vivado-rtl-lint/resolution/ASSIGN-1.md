<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# ASSIGN-1: Arithmetic Overflow / Result Truncation
# ASSIGN-1: 산술 오버플로/결과 잘림

**Rule:** ASSIGN-1 | **Severity:** WARNING | **Standard:** UG901 Ch.4
**규칙:** ASSIGN-1 | **심각도:** 경고 | **표준:** UG901 Ch.4

## Vivado Message
## 비바도 메시지
```
WARNING: [Synth 37-78] [ASSIGN-1]
Input operands(of sizes 7 8 7) of add/sub cluster ending with expression 
'((a - b) - c)' could yield maximum size of 10, but only width of 8 is being used.
```

## Root Cause
## 근본 원인

Arithmetic operations produce results requiring more bits than the assigned target width.
산술 연산을 수행하면 할당된 대상 너비보다 더 많은 비트가 필요한 결과가 생성됩니다.
Result is silently truncated.
결과는 자동으로 잘립니다.

**Width rules:** Addition: `max(widths) + 1`.
**너비 규칙:** 추가: `max(widths) + 1`.
Subtraction: `max(widths) + sign_bits`.
빼기: `max(widths) + sign_bits`.
Multi-operand: cascading accumulation.
다중 피연산자: 계단식 누적.

**Common sources:** Insufficient output width, mixed operand widths, cascaded operations, bit-slicing (strips carry info)
**공통 소스:** 불충분한 출력 폭, 혼합된 피연산자 폭, 계단식 연산, 비트 슬라이싱(스트립에 정보 전달)

**Bit-slicing caveat:** Reducing operand width via slicing doesn't reduce result precision needs.
**비트 슬라이싱 주의 사항:** 슬라이싱을 통해 피연산자 너비를 줄여도 결과 정밀도 요구 사항은 줄어들지 않습니다.
`a[5:0] - b - c` with 8-bit b,c still needs 10 bits.
8비트 b,c의 `a[5:0] - b - c`에는 여전히 10비트가 필요합니다.
Often triggers ASSIGN-2 as well (unsigned slice).
종종 ASSIGN-2(부호 없는 슬라이스)도 트리거합니다.

## Fix Options
## 수정 옵션

### Option 1: Widen Output (Recommended)
### 옵션 1: 출력 확대(권장)

Read required width directly from Vivado message ("maximum size of N").
Vivado 메시지에서 직접 필요한 너비를 읽습니다("최대 크기 N").

```verilog
// Before
output [7:0] y1;
y1 = a - b - c;       // 10-bit result truncated to 8

// After
output [9:0] y1;      // Widened to 10 bits
y1 = a - b - c;       // Full precision preserved
```

Update parent module connections to match new width.
새로운 너비와 일치하도록 상위 모듈 연결을 업데이트합니다.

**Width calculation helper:**
**너비 계산 도우미:**
```verilog
localparam RESULT_W = $clog2(2**A_W + 2**B_W);  // Auto-calculate
```

### Option 2: Saturation/Clamping (When interface is fixed)
### 옵션 2: 포화/클램핑(인터페이스가 고정된 경우)

Compute at full width internally, then clamp to output range.
내부적으로 전체 폭으로 계산한 다음 출력 범위로 고정합니다.

```verilog
reg [9:0] result_full;

always @(posedge clk) begin
   result_full = a - b - c;           // Full 10-bit calculation
   if (result_full[9])                // Negative (MSB set)
      y1 <= 8'h00;                    // Clamp to 0
   else if (result_full > 255)        // Exceeds 8-bit max
      y1 <= 8'hFF;                    // Clamp to 255
   else
      y1 <= result_full[7:0];         // Use exact value
end
```

**Signed saturation variant:**
**서명된 포화 변형:**
```verilog
if (result_full > 127)       y1 <= 8'h7F;
else if (result_full < -128) y1 <= 8'h80;
else                         y1 <= result_full[7:0];
```

### Decision Guide
### 결정 가이드

| Scenario<br>시나리오 | Fix<br>수정 |
|----------|-----|
| Full precision needed / interface changeable<br>완전한 정밀도 필요 / 인터페이스 변경 가능 | Option 1: Widen<br>옵션 1: 넓히기 |
| Fixed interface (IP, standard)<br>고정 인터페이스(IP, 표준) | Option 2: Saturate<br>옵션 2: 포화 |
| Intentional modulo arithmetic<br>의도적인 모듈로 연산 | Suppress with comment<br>댓글로 억제 |

## Validation
## 검증

```tcl
synth_design -top <module> -part <part> -lint -file lint_recheck.rpt
# Verify: ASSIGN-1 count = 0
report_utilization  # Widened: ~same LUTs. Saturate: +5-10 LUTs for comparators
```

## References
## 참고자료
**Xilinx:** UG901 Ch.4 "Arithmetic Operators", UG949 "RTL Coding Best Practices"
**자일링스:** UG901 Ch.4 "산술 연산자", UG949 "RTL 코딩 모범 사례"
**IEEE:** 1364-2005 §5.5, 1800-2017 §11.6 (Expression bit-width determination)
**IEEE:** 1364-2005 §5.5, 1800-2017 §11.6(표현식 비트 폭 결정)
