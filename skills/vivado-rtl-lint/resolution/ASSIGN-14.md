<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# ASSIGN-14: Duplicate Case Branches
# ASSIGN-14: 중복 케이스 분기

**Rule:** ASSIGN-14 | **Severity:** CRITICAL WARNING | **Standard:** IEEE 1800-2017 §12.5
**규칙:** ASSIGN-14 | **심각도:** 심각한 경고 | **표준:** IEEE 1800-2017 §12.5

## Vivado Message
## 비바도 메시지
```
CRITICAL WARNING: [Synth 37-140] [ASSIGN-14]
Duplicate case branches found. Hierarchy 'module_name', File 'design.sv', Line XX.
```

## Root Cause
## 근본 원인

Case statement contains duplicate case items (same selector value multiple times).
Case 문에 중복된 Case 항목이 포함되어 있습니다(동일한 선택기 값이 여러 번 표시됨).
Only first match executes; subsequent duplicates are unreachable dead code.
첫 번째 일치 항목만 실행됩니다. 후속 중복은 도달할 수 없는 데드 코드입니다.

**Common sources:** Copy-paste errors during FSM development, typo in case item values, attempting conditional logic via duplicate items
**공통 소스:** FSM 개발 중 복사-붙여넣기 오류, 케이스 항목 값의 오타, 중복 항목을 통한 조건부 논리 시도

## Fix Options
## 수정 옵션

### Option 1: Remove Duplicates (First match is correct)
### 옵션 1: 중복 항목 제거(첫 번째 일치 항목이 정확함)

```verilog
// Before
case(state)
   4'b1000 : next = 4'b1001;  // Executes
   4'b1000 : next = 4'b1010;  // Dead code
endcase

// After
case(state)
   4'b1000 : next = 4'b1001;
   default : next = 4'b0000;
endcase
```

### Option 2: Correct State Values (Sequential FSM copy-paste error)
### 옵션 2: 올바른 상태 값(순차 FSM 복사-붙여넣기 오류)

```verilog
// Before
4'b1000 : next = 4'b1001;  // State 8 → 9
4'b1000 : next = 4'b1010;  // Should be 4'b1001
4'b1000 : next = 4'b1011;  // Should be 4'b1010

// After
4'b1000 : next = 4'b1001;  // State 8  → 9
4'b1001 : next = 4'b1010;  // State 9  → 10
4'b1010 : next = 4'b1011;  // State 10 → 11
```

### Option 3: Add Conditional Logic (Transition depends on input)
### 옵션 3: 조건부 논리 추가(전환은 입력에 따라 다름)

```verilog
// Before: attempted conditional via duplicate
STATE_WAIT : next = STATE_PROC;  // Executes
STATE_WAIT : next = STATE_IDLE;  // Dead code

// After: nested if-else
STATE_WAIT : begin
   if (enable) next = STATE_PROC;
   else        next = STATE_IDLE;
end
```

Always add `default` clause when removing duplicates (prevents INFER-2).
중복을 제거할 때는 항상 `default` 절을 추가하세요(INFER-2 방지).

## VHDL Pattern
## VHDL 패턴

```vhdl
-- Before: duplicate
when "1000" => nextState <= "1001";
when "1000" => nextState <= "1010";

-- After: corrected
when "1000" => nextState <= "1001";
when "1001" => nextState <= "1010";
when others => nextState <= "0000";
```

## Validation
## 검증

```tcl
synth_design -top <module> -part <part> -lint -file lint_recheck.rpt
# Verify: ASSIGN-14 count = 0
```

## References
## 참고자료
**Xilinx:** UG901 Ch.3, UG906 Appendix B
**자일링스:** UG901 Ch.3, UG906 부록 B
**IEEE:** 1364-2005 §9.5, 1800-2017 §12.5
**IEEE:** 1364-2005 §9.5, 1800-2017 §12.5
**Related:** INFER-2 (incomplete case — add default when removing duplicates)
**관련:** INFER-2(불완전한 사례 - 중복 제거 시 기본값 추가)
