<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# INFER-3: Case Equality Converted for Synthesis
# INFER-3: 합성을 위해 변환된 대소문자 동일성

**Rule:** INFER-3 | **Severity:** INFO | **Standard:** IEEE 1800-2017 §11.4.5
**규칙:** 추론-3 | **심각도:** 정보 | **표준:** IEEE 1800-2017 §11.4.5

## Vivado Message
## 비바도 메시지
```
INFO: [Synth 37-XX] [INFER-3]
'===' is converted to '==' for synthesis.
RTL Name '(signal)', Hierarchy '(module)', File '(file.sv)', Line (line).
```

## Root Cause
## 근본 원인

The `===` (case equality) and `!==` (case inequality) operators compare all four logic
`===`(대소문자 동일) 및 `!==`(대소문자 불일치) 연산자는 네 가지 논리를 모두 비교합니다.
values (0, 1, X, Z).
값(0, 1, X, Z).
Since synthesis targets real hardware (only 0 and 1), Vivado converts
합성은 실제 하드웨어(0과 1만)를 대상으로 하기 때문에 Vivado는
`===` → `==` and `!==` → `!=`.
`===` → `==` 및 `!==` → `!=`.
This is usually harmless, but can cause simulation-synthesis
이는 일반적으로 무해하지만 시뮬레이션-합성을 유발할 수 있습니다.
mismatch if the design intentionally tests for X or Z states.
디자인이 의도적으로 X 또는 Z 상태를 테스트하는 경우 불일치가 발생합니다.

**Common sources:** Copy-paste from testbench code, overly cautious coding style,
**공통 소스:** 테스트벤치 코드에서 복사하여 붙여넣기, 지나치게 신중한 코딩 스타일,
leftover assertions or debug comparisons in RTL
RTL의 남은 주장 또는 디버그 비교

## Fix Options
## 수정 옵션

### Option 1: Replace with `==` (Recommended — No X/Z intent)
### 옵션 1: `==`로 교체(권장 - X/Z 의도 없음)

Most RTL should use `==` since synthesis cannot represent X/Z.
대부분의 RTL은 합성이 X/Z를 나타낼 수 없기 때문에 `==`를 사용해야 합니다.
Explicit replacement
명시적 교체
documents the intent and eliminates the info message.
의도를 문서화하고 정보 메시지를 제거합니다.

```verilog
// Before
always_comb begin
   if (data === 8'hFF)        // 4-state comparison — unnecessary in synthesis
      valid = 1'b1;
end

// After
always_comb begin
   if (data == 8'hFF)         // 2-state comparison — matches synthesis behavior
      valid = 1'b1;
end
```

### Option 2: Keep `===` with Intent Comment (Simulation debug code)
### 옵션 2: 인텐트 코멘트와 함께 `===` 유지(시뮬레이션 디버그 코드)

If `===` is deliberately used for simulation-only X-detection (e.g., checking for
시뮬레이션 전용 X-탐지에 '==='를 의도적으로 사용한 경우(예:
uninitialized values), keep it but document the purpose:
초기화되지 않은 값), 유지하되 목적을 문서화하세요.

```verilog
// Intentional: detect uninitialized bus in simulation
// synthesis translate_off
always @(posedge clk) begin
   if (data_bus === 8'hxx)
      $warning("Uninitialized data_bus at time %0t", $time);
end
// synthesis translate_on
```

**Best practice:** Wrap simulation-only `===` checks in `translate_off`/`translate_on`
**모범 사례:** `translate_off`/`translate_on`에서 시뮬레이션 전용 `===` 검사를 래핑합니다.
or `` `ifdef SIMULATION`` guards so they don't appear in synthesis at all.
또는 ````ifdef SIMULATION``은 합성에 전혀 나타나지 않도록 보호합니다.

### Option 3: Use `inside` for Value Matching (SystemVerilog)
### 옵션 3: 값 일치를 위해 'inside' 사용(SystemVerilog)

For matching against multiple valid values, `inside` is cleaner and has well-defined
여러 유효한 값과 일치하는 경우 `inside`가 더 깨끗하고 잘 정의되어 있습니다.
synthesis behavior:
합성 거동:

```verilog
// Before
if (state === IDLE || state === DONE)

// After
if (state inside {IDLE, DONE})    // No X/Z ambiguity
```

## Decision Guide
## 결정 가이드

| Scenario<br>시나리오 | Fix<br>수정 |
|----------|-----|
| Normal RTL comparison<br>일반 RTL 비교 | Option 1: Replace with `==`<br>옵션 1: `==`로 바꾸기 |
| Simulation debug / X-detection<br>시뮬레이션 디버그 / X-검출 | Option 2: Keep with `translate_off` guard<br>옵션 2: 'translate_off' 가드를 사용하여 유지 |
| Multi-value matching<br>다중 값 일치 | Option 3: Use `inside`<br>옵션 3: '내부' 사용 |
| Intentional and must stay<br>의도적이며 유지되어야 함 | Suppress with comment, accept INFO<br>댓글로 억제하고 INFO를 수락하세요. |

## Simulation-Synthesis Mismatch Risk
## 시뮬레이션-합성 불일치 위험

| Operator<br>운영자 | Simulation (X input)<br>시뮬레이션(X 입력) | Synthesis<br>합성 | Mismatch?<br>불일치? |
|----------|---------------------|-----------|-----------|
| `===`    | Returns 0 (X ≠ val)<br>0을 반환합니다(X ≠ 값). | `==` behavior<br>`==` 행동 | Possible if X expected<br>X가 예상되는 경우 가능 |
| `==`     | Returns X<br>X를 반환합니다.           | Returns 0 or 1<br>0 또는 1을 반환합니다. | Possible if X expected<br>X가 예상되는 경우 가능 |

In practice, for signals that are always 0 or 1 in hardware, `===` and `==` produce
실제로 하드웨어에서 항상 0 또는 1인 신호의 경우 `===` 및 `==`는 다음을 생성합니다.
identical results.
동일한 결과.
The risk only exists when X/Z states are meaningful (testbench,
위험은 X/Z 상태가 의미 있는 경우에만 존재합니다(테스트벤치,
uninitialized memory, tri-state buses).
초기화되지 않은 메모리, 3상태 버스).

## Validation
## 검증

```tcl
synth_design -top <module> -part <part> -lint -file lint_recheck.rpt
# Verify: INFER-3 count = 0 (or accepted with documentation)
# Verify simulation still passes after replacement:
#   launch_simulation -mode behavioral
```

## References
## 참고자료
**Xilinx:** UG901 Ch.4 "Operators", UG906 "Synthesis Report Messages"
**Xilinx:** UG901 Ch.4 "연산자", UG906 "합성 보고서 메시지"
**IEEE:** 1800-2017 §11.4.5 (Case equality), §11.4.6 (Wildcard equality)
**IEEE:** 1800-2017 §11.4.5(대소문자 동일), §11.4.6(와일드카드 동일)
**Related:** INFER-1 (latch inference), INFER-2 (incomplete case)
**관련:** INFER-1(래치 추론), INFER-2(불완전한 사례)
