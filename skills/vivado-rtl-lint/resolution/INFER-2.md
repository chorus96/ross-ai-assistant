<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# INFER-2: Incomplete Case Statement
# INFER-2: 불완전한 사례 진술서

**Rule:** INFER-2 | **Severity:** CRITICAL WARNING | **Standard:** UG901 Ch.4.3.5
**규칙:** 추론-2 | **심각도:** 심각한 경고 | **표준:** UG901 Ch.4.3.5

## Vivado Message
## 비바도 메시지
```
CRITICAL WARNING: [Synth 37-75] [INFER-2] Case statement items not fully specified
and does not have a default item. Hierarchy 'top_rtl', File 'top.sv', Line 40.
```

## Root Cause
## 근본 원인

Case statements without complete value coverage and no `default` clause create undefined behavior — leading to latch inference (triggers INFER-1) and simulation-synthesis mismatch.
완전한 값 적용 범위가 없고 `default` 절이 없는 Case 문은 정의되지 않은 동작을 생성하여 래치 추론(INFER-1 트리거) 및 시뮬레이션-합성 불일치로 이어집니다.

**Common sources:** Missing default, FSM with unused states, commented-out branches, sparse encoding, parameter-based case exceeding branches
**공통 소스:** 누락된 기본값, 사용되지 않은 상태가 있는 FSM, 주석 처리된 분기, 희소 인코딩, 분기를 초과하는 매개변수 기반 사례

## Pattern Recognition Table
## 패턴 인식 테이블

| ID | Pattern<br>패턴 | Root Cause<br>근본 원인 |
|----|---------|------------|
| C1 | `case(sel) 2'b0:...; endcase` (no default)<br>`case(sel) 2'b0:...; endcase` (기본값 없음) | Missing default clause<br>기본 조항 누락 |
| C2 | FSM with N-1 states only<br>N-1 상태만 있는 FSM | Incomplete state coverage<br>불완전한 주 적용 범위 |
| C3 | `case(state) p1:...; // p16: ...; endcase`<br>`케이스(상태) p1:...; // p16: ...; 최종 케이스` | Commented branch (debug)<br>주석 처리된 분기(디버그) |
| C4 | 3-bit selector, 5 branches, no default<br>3비트 선택기, 5개 분기, 기본값 없음 | Sparse encoding incomplete<br>스파스 인코딩이 불완전함 |
| C5 | `typedef enum {A,B,C}; case(e) A:...; B:...; endcase`<br>`typedef 열거형 {A,B,C}; 사례(e) A:...; 비:...; 최종 케이스` | Enum subset handling<br>열거형 하위 집합 처리 |
| C6 | `case(WIDTH_PARAM[3:0]) 4'd8:...; 4'd16:...; endcase`<br>`case(WIDTH_PARAM[3:0]) 4'd8:...; 4'd16:...; 최종 케이스` | Parameter can exceed branches<br>매개변수가 분기를 초과할 수 있습니다. |
| C7 | One-hot `case(1'b1) state[0]:...; state[3]:...; endcase`<br>원-핫 `case(1'b1) state[0]:...; 상태[3]:...; 최종 케이스' | Assumes exclusivity, no error case<br>독점성을 가정하며 오류 사례는 없습니다. |
| C8 | `case(bus[7:0]) 8'h0A:...; 8'hFF:...; endcase`<br>`case(버스[7:0]) 8'h0A:...; 8'hFF:...; 최종 케이스` | Bus too wide for branches<br>지점에 비해 버스 폭이 너무 넓음 |
| C9 | `case(sel) /* synthesis full_case */`<br>`case(sel) /* 합성 full_case */` | Pragma without actual coverage<br>실제 적용 범위가 없는 Pragma |
| C10 | Casez with don't-cares, gaps in coverage<br>무관심한 Casez, 보장 범위의 공백 | Don't-care logic incomplete<br>무정의 논리가 불완전함 |
| C11 | Case in generate block with parameter<br>매개변수를 사용하여 블록을 생성하는 경우 | Generate-time incomplete<br>생성 시간이 불완전함 |

**Critical notes:**
**중요 사항:**
- **C9:** `full_case`/`parallel_case` pragmas are synthesis directives only — still need explicit `default`
- **C9:** `full_case`/`parallel_case` pragma는 합성 지시문일 뿐입니다. 여전히 명시적인 `default`가 필요합니다.
- **C7:** One-hot case should add `default` to catch multi-hot errors
- **C7:** 원-핫 케이스에서는 멀티 핫 오류를 포착하기 위해 '기본값'을 추가해야 합니다.
- **C3:** Commented branches during debug leave incomplete coverage
- **C3:** 디버그 중 주석 처리된 분기가 불완전한 적용 범위를 남깁니다.

## Fix Options
## 수정 옵션

### Option 1: Add Default Clause (Most common)
### 옵션 1: 기본 조항 추가(가장 일반적)

```verilog
// BEFORE
always@(*) begin
   case(sel)
      2'b00: out = a;
      2'b01: out = b;
   endcase  // Missing 2'b10, 2'b11
end

// AFTER
always@(*) begin
   case(sel)
      2'b00: out = a;
      2'b01: out = b;
      default: out = 8'h00;  // Safe value for unspecified encodings
   endcase
end
```

### Option 2: Complete All Branches (Full coverage needed)
### 옵션 2: 모든 지점 완료(전체 보장 필요)

```verilog
// FSM: enumerate all states + safety default
always@(*) begin
   case(state)
      p1:  nextState = p2;
      p2:  nextState = p3;
      p15: nextState = p1;
      p16: nextState = p2;      // Explicitly handle all states
      default: nextState = p1;  // Safety net (should never reach)
   endcase
end
```

### Option 3: SystemVerilog `unique case`
### 옵션 3: SystemVerilog '고유 사례'

```verilog
always@(*) begin
   unique case(state)        // Compiler verifies completeness
      p1:  nextState = p2;
      p15: nextState = p1;
      default: nextState = p1;  // Still required
   endcase
end
```

## INFER-1 Relationship
## INFER-1 관계

INFER-2 often causes INFER-1 on signals assigned within case branches.
INFER-2는 케이스 분기 내에 할당된 신호에서 INFER-1을 발생시키는 경우가 많습니다.
Fixing INFER-2 by adding `default` typically resolves both simultaneously.
'기본값'을 추가하여 INFER-2를 수정하면 일반적으로 두 가지가 동시에 해결됩니다.
Always cross-check for paired INFER-1 violations.
항상 쌍을 이루는 INFER-1 위반을 교차 확인하세요.

## Validation
## 검증

```tcl
synth_design -top <module> -part <part> -lint
# Verify: INFER-2 count = 0, check INFER-1 also resolved
get_cells -hierarchical -filter {REF_NAME =~ LD*}
# Expected: {} (no latch primitives)
```

## References
## 참고자료
**Xilinx:** UG901 Ch.4.3.5, UG949 RTL practices, UG906 SYNTH-8-155, Answer 41851
**자일링스:** UG901 Ch.4.3.5, UG949 RTL 사례, UG906 SYNTH-8-155, 답변 41851
**IEEE:** 1364-2005 §9.5, 1800-2017 §12.5 (unique/priority case)
**IEEE:** 1364-2005 §9.5, 1800-2017 §12.5(고유/우선 사례)
**Related:** INFER-1 (latch inference), ASSIGN-14 (duplicate case branches)
**관련:** INFER-1(래치 추론), ASSIGN-14(중복 사례 분기)
