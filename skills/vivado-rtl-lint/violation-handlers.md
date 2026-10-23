<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Violation Handler Reference
# 위반 처리기 참조

This file contains per-violation-type instructions for generating fix recommendations
이 파일에는 수정 권장 사항을 생성하기 위한 위반 유형별 지침이 포함되어 있습니다.
in the RTL lint report.
RTL 린트 보고서에서.
For **every** linter violation detected, follow the handler below
**모든** 린터 위반이 감지되면 아래 핸들러를 따르세요.
that matches the rule ID.
규칙 ID와 일치합니다.
If no handler exists, fall back to vivado_doc_search + general knowledge.
핸들러가 없으면 vivado_doc_search + 일반 지식으로 돌아갑니다.

> **Mandatory**: Always load the corresponding `resolution/<RULE-ID>.md` guide with `read_file`
> **필수**: 항상 `read_file`을 사용하여 해당 `solution/<RULE-ID>.md` 가이드를 로드하세요.
> before generating the fix section.
> 수정 섹션을 생성하기 전에.
> The guide contains validated fix templates, rationale,
> 가이드에는 검증된 수정 템플릿, 근거,
> and verification steps.
> 및 확인 단계.

---

## Dispatch Table
## 파견 테이블

| Rule ID<br>규칙 ID   | Category<br>카테고리   | Handler Summary<br>핸들러 요약 |
|-----------|------------|-----------------|
| ASSIGN-1<br>할당-1  | Width<br>폭      | Widen output to required bit width, or apply saturation logic<br>필요한 비트 폭으로 출력을 넓히거나 포화 로직을 적용합니다. |
| ASSIGN-2<br>할당-2  | Signedness<br>서명성 | Add `$signed()` cast to unsigned operand(s)<br>부호 없는 피연산자에 `$signed()` 캐스트 추가 |
| ASSIGN-3<br>할당-3  | Shift<br>교대      | Correct shift parameter or add clipping validation<br>이동 매개변수를 수정하거나 클리핑 검증을 추가하세요. |
| ASSIGN-5<br>할당-5  | Unset<br>설정되지 않음      | Add missing assignment (wire/reg/logic)<br>누락된 할당 추가(와이어/등록/로직) |
| ASSIGN-6<br>할당-6  | Unused<br>미사용     | Remove signal declaration and all assignments<br>신호 선언 및 모든 할당 제거 |
| ASSIGN-7<br>할당-7  | Overlap<br>중복    | Adjust bit ranges to non-overlapping<br>비트 범위를 겹치지 않도록 조정 |
| ASSIGN-8<br>할당-8  | Dimension<br>차원  | Match array operand dimensions in comparison<br>비교에서 배열 피연산자 차원 일치 |
| ASSIGN-9<br>ASSIGN-9  | Port<br>항구       | Complete unassigned output port bits or remove port<br>할당되지 않은 출력 포트 비트를 완료하거나 포트를 제거합니다. |
| ASSIGN-10<br>할당-10 | Port<br>항구       | Remove or reduce width of unused I/O port bits<br>사용하지 않는 I/O 포트 비트의 너비를 제거하거나 줄입니다. |
| ASSIGN-12<br>할당-12 | Tri-state<br>삼국지  | Tie port to constant or use generate loop<br>포트를 상수에 연결하거나 생성 루프를 사용하십시오. |
| ASSIGN-14<br>할당-14 | Case<br>케이스       | Remove duplicate case branches or correct values<br>중복된 케이스 분기를 제거하거나 값을 수정하세요. |
| INFER-1<br>추론-1   | Latch<br>래치      | Complete all branches in combinational logic<br>조합 논리의 모든 분기를 완료하세요. |
| INFER-2<br>추론-2   | Case<br>케이스       | Add default clause or complete all case branches<br>기본 조항을 추가하거나 모든 사례 분기를 완료하세요. |
| INFER-3<br>추론-3   | Equality<br>평등   | Replace `===` with `==` or document intent<br>`===`를 `==` 또는 문서 의도로 바꾸세요. |
| CLOCK-1<br>시계-1   | Clock<br>시계      | Unify to single clock edge (posedge)<br>단일 클록 에지로 통합(posege) |
| QOR-1<br>QOR-1     | Precision<br>정밀도  | Widen intermediate signal or direct merge<br>중간 신호 확대 또는 직접 병합 |
| RESET-1<br>리셋-1   | Reset<br>재설정      | Use single async signal (reset OR set)<br>단일 비동기 신호 사용(재설정 또는 설정) |
| RESET-2<br>리셋-2   | Reset<br>재설정      | Add async reset assignments for all registers<br>모든 레지스터에 대한 비동기 재설정 할당 추가 |
| RESET-3<br>리셋-3   | Reset<br>재설정      | Remove conflicting reset/set — choose one<br>충돌하는 재설정/설정 제거 - 하나 선택 |

---

## Detailed Handlers
## 상세한 핸들러

### ASSIGN-1 — Arithmetic Overflow / Result Truncation
### ASSIGN-1 — 산술 오버플로/결과 잘림

1. Load `resolution/ASSIGN-1.md`
1. '해상도/ASSIGN-1.md' 로드
2. Extract required vs current bit width from message:
2. 메시지에서 필요한 비트 너비와 현재 비트 너비를 추출합니다.
`"could yield maximum size of N, but only width of M is being used"`
``N의 최대 크기를 생성할 수 있지만 M의 너비만 사용됩니다.'`
3. **Primary fix** (recommended): Widen output to required bit width
3. **기본 수정**(권장): 출력을 필요한 비트 너비로 확대
   - Example: `output [7:0] result;` → `output [9:0] result;`
   - 예: `[7:0] 결과 출력;` → `[9:0] 결과 출력;`
   - Preserves full arithmetic precision
   - 전체 산술 정밀도를 유지합니다.
4. **Alternative fix** (constrained interface): Apply saturation logic (Option 3 from guide)
4. **대체 수정**(제한된 인터페이스): 포화 논리 적용(가이드의 옵션 3)
5. Include operand analysis table showing bit widths and worst-case calculation
5. 비트 폭과 최악의 경우 계산을 보여주는 피연산자 분석 테이블 포함
6. Use exact markdown formatting from resolution guide's "Fix Template for Analysis Reports"
6. 해결 가이드의 "분석 보고서 템플릿 수정"에서 정확한 마크다운 형식을 사용하세요.

### ASSIGN-2 — Mixed Signed/Unsigned Arithmetic
### ASSIGN-2 - 부호 있는/부호 없는 혼합 산술

1. Load `resolution/ASSIGN-2.md`
1. '해상도/ASSIGN-2.md' 로드
2. Extract operator type, expression, and operand signedness
2. 연산자 유형, 표현식 및 피연산자 부호 추출
3. **Primary fix**: Add `$signed()` cast to unsigned operand(s)
3. **기본 수정**: 부호 없는 피연산자에 `$signed()` 캐스트 추가
   - Example: `result = a + b;` → `result = $signed(a) + b;`
   - 예: `결과 = a + b;` → `결과 = $signed(a) + b;`
   - Zero hardware overhead (semantic-only)
   - 하드웨어 오버헤드 없음(의미론적 전용)
4. **Optional enhancement**: Declare output/reg as `signed`
4. **선택적 개선 사항**: 출력/등록을 '서명됨'으로 선언
5. Use exact markdown formatting from resolution guide
5. 해결 가이드의 정확한 마크다운 형식을 사용하세요.

### ASSIGN-3 — Shift Overflow
### ASSIGN-3 — 시프트 오버플로

1. Load `resolution/ASSIGN-3.md`
1. '해상도/ASSIGN-3.md' 로드
2. Extract shift operator (`>>` / `<<`), signal width, shift amount from message:
2. 메시지에서 이동 연산자(`>>` / `<<`), 신호 폭, 이동량을 추출합니다.
`"operator >> in expression ... exceeds range.
표현식의 ``연산자 >> ...가 범위를 초과합니다.
LHS width: N, RHS value: M"`
LHS 너비: N, RHS 값: M"`
3. Valid shift range: 0 to (signal_width − 1)
3. 유효한 이동 범위: 0 ~ (signal_width − 1)
4. **Primary fix** (most common): Correct shift amount parameter
4. **1차 수정**(가장 일반적): 올바른 이동량 매개변수
   - Example: `parameter SHIFTSIZE = 30;` → `parameter SHIFTSIZE = 2;`
   - 예: `매개변수 SHIFTSIZE = 30;` → `매개변수 SHIFTSIZE = 2;`
5. **Alternative fix**: Add parameter validation and clipping
5. **대체 수정**: 매개변수 확인 및 클리핑 추가
   - `localparam SAFE_SHIFT = (SHIFT > MAX) ?
   - `localparam SAFE_SHIFT = (SHIFT > MAX) ?
   - MAX : SHIFT;`
   - 최대 : SHIFT;`
6. Reference IEEE 1364-2005 Section 5.1.13
6. IEEE 1364-2005 섹션 5.1.13 참조
7. Use "Example Agent Output Template" from resolution guide
7. 해결 가이드의 "에이전트 출력 템플릿 예"를 사용하세요.

### ASSIGN-5 — Variable Used But Not Set
### ASSIGN-5 - 변수가 사용되었지만 설정되지 않음

1. Load `resolution/ASSIGN-5.md`
1. '해상도/ASSIGN-5.md' 로드
2. Extract signal name, type, and usage location
2. 신호 이름, 유형, 사용 위치 추출
3. **Primary fix**: Add missing assignment
3. **기본 수정**: 누락된 할당 추가
   - Wire → continuous assignment; Reg → procedural in always block; Logic → always_comb/ff
   - 와이어 → 연속 할당; Reg → 항상 블록의 절차; 논리 → Always_comb/ff
   - Ensure ALL conditional paths assign the signal
   - 모든 조건부 경로가 신호를 할당하는지 확인하세요.
4. **Priority**: Fix ASSIGN-5 **before** ASSIGN-6 (functional > code quality)
4. **우선순위**: ASSIGN-6 **전** ASSIGN-5 수정(기능 > 코드 품질)

### ASSIGN-6 — Signal Assigned But Not Read
### ASSIGN-6 - 신호가 할당되었으나 읽히지 않음

1. Load `resolution/ASSIGN-6.md`
1. '해상도/ASSIGN-6.md' 로드
2. **Primary fix**: Remove signal declaration and all assignments
2. **기본 수정**: 신호 선언 및 모든 할당 제거
3. Clean up empty logic blocks if they only assigned this signal
3. 이 신호만 할당한 경우 빈 논리 블록을 정리합니다.

### ASSIGN-7 — Multiple Assignments to Same Bit
### ASSIGN-7 - 동일한 비트에 대한 다중 할당

1. Load `resolution/ASSIGN-7.md`
1. '해상도/ASSIGN-7.md' 로드
2. Extract signal name, bit ranges, overlap info
2. 신호 이름, 비트 범위, 중복 정보 추출
3. **Primary fix**: Adjust bit ranges to non-overlapping
3. **기본 수정**: 비트 범위를 겹치지 않도록 조정
   - Example: `[3:0]` and `[7:3]` overlap at bit 3 → fix to `[2:0]` and `[7:3]`
   - 예: `[3:0]` 및 `[7:3]`이 비트 3에서 겹치는 경우 → `[2:0]` 및 `[7:3]`으로 고정

### ASSIGN-8 — Array Dimension Mismatch in Comparison
### ASSIGN-8 — 비교 시 배열 크기 불일치

1. Load `resolution/ASSIGN-8.md`
1. '해상도/ASSIGN-8.md' 로드
2. Extract operator, operand dimensions, and port declarations from message
2. 메시지에서 연산자, 피연산자 차원 및 포트 선언 추출
3. **Primary fix**: Match operand dimensions (both 1D or both 2D)
3. **1차 수정**: 피연산자 크기 일치(모두 1D 또는 둘 다 2D)
4. **Alternative**: Flatten multidimensional array or use element-wise comparison
4. **대안**: 다차원 배열을 평면화하거나 요소별 비교 사용
5. Reference IEEE 1800-2017 §11.4.5
5. 참조 IEEE 1800-2017 §11.4.5

### ASSIGN-9 — Unassigned Output Port Bits
### ASSIGN-9 - 할당되지 않은 출력 포트 비트

1. Load `resolution/ASSIGN-9.md`
1. '해상도/ASSIGN-9.md' 로드
2. Determine scenario: completely unassigned (Scenario A) vs partially assigned (Scenario B)
2. 시나리오 결정: 완전히 할당되지 않음(시나리오 A) 및 부분적으로 할당됨(시나리오 B)
3. **Scenario A** (no bits assigned): Remove port from module and update parent instantiations
3. **시나리오 A**(비트 할당 없음): 모듈에서 포트를 제거하고 상위 인스턴스화를 업데이트합니다.
4. **Scenario B** (some bits assigned): Complete the assignment — fill missing bit ranges with zero, sign-extension, or protocol-required constants
4. **시나리오 B**(일부 비트 할당): 할당 완료 - 누락된 비트 범위를 0, 부호 확장 또는 프로토콜 필수 상수로 채웁니다.

### ASSIGN-10 — Unused I/O Port Bits
### ASSIGN-10 - 사용되지 않은 I/O 포트 비트

1. Load `resolution/ASSIGN-10.md`
1. '해상도/ASSIGN-10.md' 로드
2. Identify which port bits are actually read vs unused
2. 실제로 읽혀진 포트 비트와 사용되지 않은 포트 비트를 식별합니다.
3. **Primary fix** (port serves no purpose): Remove unused port from module interface
3. **기본 수정**(포트는 용도가 없음): 모듈 인터페이스에서 사용되지 않는 포트를 제거합니다.
4. **Alternative** (some bits used): Reduce port width to only the bits actually consumed
4. **대안**(일부 비트 사용됨): 포트 너비를 실제로 사용된 비트만큼만 줄입니다.
5. Update all parent instantiations after port changes
5. 포트 변경 후 모든 상위 인스턴스화 업데이트

### ASSIGN-12 — Unconnected/Tri-State Port
### ASSIGN-12 — 연결되지 않음/삼중 상태 포트

1. Load `resolution/ASSIGN-12.md`
1. '해상도/ASSIGN-12.md' 로드
2. **Primary fix** (recommended): Tie to constant
2. **기본 수정**(권장): 상수에 연결
   - `.port_name(1'bz)` → `.port_name(1'b0)` or `.port_name(1'b1)`
   - `.port_name(1'bz)` → `.port_name(1'b0)` 또는 `.port_name(1'b1)`
3. **Alternative fix**: Generate loop for multi-instance designs
3. **대체 수정**: 다중 인스턴스 설계를 위한 루프 생성
4. Reference IEEE 1364-2005 Section 12.3.3
4. IEEE 1364-2005 섹션 12.3.3 참조

### ASSIGN-14 — Duplicate Case Branches
### ASSIGN-14 — 중복 케이스 분기

1. Load `resolution/ASSIGN-14.md`
1. '해상도/ASSIGN-14.md' 로드
2. Identify which case items share the same selector value
2. 동일한 선택기 값을 공유하는 사례 항목 식별
3. **Primary fix** (first match correct): Remove duplicate branch
3. **기본 수정**(첫 번째 일치 항목 수정): 중복 분기 제거
4. **Alternative** (copy-paste error): Correct state values to unique, sequential encoding
4. **대체**(복사-붙여넣기 오류): 상태 값을 고유한 순차 인코딩으로 수정합니다.
5. **Alternative** (conditional logic intended): Convert duplicate to `if-else` inside single case item
5. **대안**(조건부 논리 의도): 단일 사례 항목 내에서 중복을 'if-else'로 변환
6. Always add `default` clause when modifying (prevents INFER-2)
6. 수정 시 항상 `default` 절을 추가합니다(INFER-2 방지).

### INFER-1 — Latch Inference
### INFER-1 - 래치 추론

1. Load `resolution/INFER-1.md`
1. '해상도/INFER-1.md' 로드
2. Identify pattern from the Pattern Recognition Table (P1–P12)
2. 패턴 인식표(P1~P12)에서 패턴 식별
3. **Primary fix** (no value retention needed): Complete combinational coverage — add `else`/`default` branch
3. **기본 수정**(값 유지 필요 없음): 전체 조합 적용 범위 - `else`/`default` 분기 추가
   - Use blocking `=` (not `<=`) in combinational blocks
   - 조합 블록에서 차단 `=`(`<=` 아님) 사용
   - Never self-reference (`sig = sig` still creates a latch)
   - 자체 참조하지 않음(`sig = sig`는 여전히 래치를 생성함)
4. **Alternative** (value retention intended): Convert to sequential logic with `posedge clk`
4. **대안**(값 유지 의도): `posedge clk`를 사용하여 순차 논리로 변환
5. **P7/P8 note**: Pre-initialization does NOT prevent latches — explicit `else`/`default` required
5. **P7/P8 참고**: 사전 초기화는 래치를 방지하지 않습니다. 명시적인 `else`/`default`가 필요합니다.
6. Cross-check: INFER-1 often pairs with INFER-2 — fixing the case statement may resolve both
6. 교차 확인: INFER-1은 종종 INFER-2와 쌍을 이룹니다. 사례 문을 수정하면 두 가지 모두 해결될 수 있습니다.

### INFER-2 — Incomplete Case Statement
### INFER-2 — 불완전한 사례 진술서

1. Load `resolution/INFER-2.md`
1. '해상도/INFER-2.md' 로드
2. Identify pattern from Pattern Recognition Table (C1–C11)
2. 패턴 인식표(C1~C11)에서 패턴 식별
3. **Primary fix**: Add `default` clause with safe output value
3. **기본 수정**: 안전한 출력 값이 포함된 `default` 절 추가
4. **Alternative** (full coverage needed): Enumerate all case branches explicitly
4. **대안**(전체 적용 범위 필요): 모든 사례 분기를 명시적으로 열거합니다.
5. Cross-check: Fixing INFER-2 typically resolves paired INFER-1 violations on the same signals
5. 교차 확인: INFER-2를 수정하면 일반적으로 동일한 신호에서 쌍을 이루는 INFER-1 위반이 해결됩니다.
6. **C9 note**: `full_case`/`parallel_case` pragmas do NOT substitute for `default`
6. **C9 참고**: `full_case`/`parallel_case` pragma는 `default`를 대체하지 않습니다.

### INFER-3 — Case Equality Conversion
### INFER-3 — 대소문자 동일 변환

1. Load `resolution/INFER-3.md`
1. '해상도/INFER-3.md' 로드
2. `===` (4-state case equality) is converted to `==` for synthesis
2. `===`(4상태 대소문자 동일)은 합성을 위해 `==`로 변환됩니다.
3. **Primary fix** (X/Z not needed): Replace `===` with `==` explicitly to match synthesis behavior
3. **기본 수정**(X/Z 필요 없음): 합성 동작과 일치하도록 명시적으로 `===`를 `==`로 바꾸세요.
4. **Alternative** (simulation debug): Keep `===` and wrap in `translate_off`/`translate_on` guard
4. **대안**(시뮬레이션 디버그): `===`를 유지하고 `translate_off`/`translate_on` 가드로 래핑
5. **Concern**: May cause simulation-synthesis mismatch if design relies on X/Z comparison
5. **우려사항**: 설계가 X/Z 비교에 의존하는 경우 시뮬레이션-합성 불일치가 발생할 수 있습니다.
6. Reference IEEE 1800-2017 §11.4.5 (Equality operators)
6. IEEE 1800-2017 §11.4.5(동등 연산자) 참조

### CLOCK-1 — Mixed Clock Edges
### CLOCK-1 — 혼합 클록 에지

1. Load `resolution/CLOCK-1.md`
1. '해상도/CLOCK-1.md' 로드
2. Identify which always blocks use `negedge` vs `posedge` of the same clock
2. 항상 동일한 시계의 `negedge`와 `posedge`를 사용하는 블록을 식별합니다.
3. **Primary fix**: Change `negedge` → `posedge` on the minority block (minimal change)
3. **1차 수정**: 소수 블록의 `negedge` → `posedge` 변경(최소 변경)
   - Do NOT merge always blocks or change assignment styles
   - 항상 블록을 병합하거나 할당 스타일을 변경하지 마십시오.
4. **DDR exception** (rare): If both edges are intentional for DDR I/O, use ODDR/IDDR primitives instead
4. **DDR 예외**(드물게): 두 에지가 DDR I/O용으로 의도된 경우 대신 ODDR/IDDR 프리미티브를 사용합니다.
5. Verify: After fix, `report_control_sets` should show reduced control set count
5. 확인: 수정 후 `report_control_sets`에 감소된 제어 세트 수가 표시되어야 합니다.

### QOR-1 — Arithmetic Precision Loss
### QOR-1 — 산술 정밀도 손실

1. Load `resolution/QOR-1.md`
1. '해상도/QOR-1.md' 로드
2. Trace arithmetic chain to identify precision loss point
2. 정밀 손실 지점을 식별하기 위해 산술 체인을 추적합니다.
3. **Primary fix** (recommended): Widen intermediate signal
3. **1차 수정**(권장): 중간 신호 확대
   - Allows Vivado to merge operators for optimal QoR
   - Vivado가 최적의 QoR을 위해 연산자를 병합할 수 있도록 허용
4. **Alternative**: Direct merge (bypass intermediate signal)
4. **대안**: 직접 병합(중간 신호 우회)
5. **Alternative**: Explicit truncation with intent comment
5. **대안**: 인텐트 주석을 사용한 명시적 잘림
6. Reference IEEE 1364-2005 Section 4.1.14
6. IEEE 1364-2005 섹션 4.1.14 참조

### RESET-1 — Multiple Async Reset/Set
### RESET-1 — 다중 비동기 재설정/설정

1. Load `resolution/RESET-1.md`
1. `해상도/RESET-1.md` 로드
2. **Primary fix**: Use single async signal (reset OR set, not both)
2. **기본 수정**: 단일 비동기 신호 사용(재설정 또는 설정, 둘 다 아님)

### RESET-2 — Incomplete Async Reset Coverage
### RESET-2 - 불완전한 비동기 재설정 범위

1. Load `resolution/RESET-2.md`
1. '해상도/RESET-2.md' 로드
2. **Primary fix**: Add async reset assignments for all registers in the process
2. **기본 수정**: 프로세스의 모든 레지스터에 대한 비동기 재설정 할당 추가

### RESET-3 — Async Reset/Set Conflict
### RESET-3 — 비동기 재설정/설정 충돌

1. Load `resolution/RESET-3.md`
1. `해상도/RESET-3.md` 로드
2. **Primary fix**: Remove conflict — choose reset OR set based on design intent
2. **기본 수정**: 충돌 제거 - 설계 의도에 따라 재설정 또는 설정 선택
   - Standard practice: async reset to 0 is most common
   - 표준 사례: 0으로의 비동기 재설정이 가장 일반적입니다.

---

## Resolution Guide Workflow (Generic)
## 해결 가이드 워크플로(일반)

```
For each violation with rule ID <RULE-ID>:
  1. Load: resolution/<RULE-ID>.md
  2. Parse linter message → extract signal, width, file:line, expression
  3. Read source code at file:line for context
  4. Select fix option from guide (Primary recommended)
  5. Generate markdown section using guide's fix template
  6. Include rationale and verification steps from guide
  7. Add UG901/IEEE documentation references
```

## Handling Unknown Violations
## 알 수 없는 위반 처리

If no resolution guide exists for a rule ID:
규칙 ID에 대한 해결 가이드가 없는 경우:
1. Use vivado_doc_search to query UG901 for the violation type
1. vivado_doc_search를 사용하여 UG901에 위반 유형을 쿼리하세요.
2. Note in report: "No resolution guide available for [RULE-ID]"
2. 보고서 참고 사항: "[RULE-ID]에 사용할 수 있는 해결 가이드가 없습니다."
3. Provide best-effort fix based on Vivado documentation
3. Vivado 문서를 기반으로 최선의 수정 사항 제공
4. Consider creating a new `resolution/<RULE-ID>.md` for future use
4. 나중에 사용할 수 있도록 새로운 '해상도/<RULE-ID>.md'를 만드는 것이 좋습니다.

## Future Extensibility
## 향후 확장성

- Add new guides to `resolution/` following naming convention: `<RULE-ID>.md`
- 명명 규칙에 따라 `solution/`에 새 가이드를 추가합니다: `<RULE-ID>.md`
- Each guide should include: Problem Statement, Detection Pattern, Standard Resolution, Fix Template
- 각 가이드에는 문제 설명, 감지 패턴, 표준 해결, 수정 템플릿이 포함되어야 합니다.
- Current categories: ASSIGN-*, INFER-*, CLOCK-*, QOR-*, RESET-*
- 현재 카테고리: ASSIGN-*, INFER-*, CLOCK-*, QOR-*, RESET-*
- Potential future categories: TIMING-*, CDC-*, SYNTH-*
- 잠재적인 미래 카테고리: TIMING-*, CDC-*, SYNTH-*
