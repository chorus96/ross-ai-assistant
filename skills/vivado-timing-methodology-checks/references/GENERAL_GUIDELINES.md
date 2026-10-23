<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# General Methodology Check Resolution Guidelines
# 일반 방법론 확인 해결 지침

This document provides general guidance for resolving timing methodology violations when a specific reference document (e.g., `TIMING-X.md`) does not exist for the check ID.
이 문서는 수표 ID에 대한 특정 참조 문서(예: 'TIMING-X.md')가 존재하지 않는 경우 타이밍 방법론 위반을 해결하기 위한 일반적인 지침을 제공합니다.

**Priority**: Specific check reference files (`TIMING-X.md`) take precedence over these general guidelines when they exist.
**우선순위**: 특정 검사 참조 파일(`TIMING-X.md`)이 존재하는 경우 이러한 일반 지침보다 우선합니다.

---

## Table of Contents
## 목차
- [Understanding Methodology Violations](#understanding-methodology-violations)
- [방법론 위반 이해](#understanding-methodology-violations)
- [General Resolution Process](#general-resolution-process)
- [일반 해결 프로세스](#general-solution-process)
- [Common Violation Categories](#common-violation-categories)
- [일반적인 위반 카테고리](#common-violation-categories)
- [Agent Understanding Verification Checklist](#agent-understanding-verification-checklist)
- [에이전트 이해 확인 체크리스트](#agent-understanding-verification-checklist)
- [Resolution Patterns](#resolution-patterns)
- [해상도 패턴](#solution-patterns)
- [When to Ask for User Input](#when-to-ask-for-user-input)
- [사용자 입력을 요청하는 경우](#when-to-ask-for-user-input)

---

## Understanding Methodology Violations
## 방법론 위반 이해

Methodology violations indicate potential timing issues, constraint errors, or design practices that may lead to:
방법론 위반은 다음과 같은 결과를 초래할 수 있는 잠재적인 타이밍 문제, 제약 오류 또는 설계 관행을 나타냅니다.
- Incorrect timing analysis results
- 잘못된 타이밍 분석 결과
- Functional failures in hardware
- 하드웨어의 기능적 오류
- Unreliable behavior across PVT (Process, Voltage, Temperature) corners
- PVT(프로세스, 전압, 온도) 코너 전반에서 신뢰할 수 없는 동작
- Incorrect clock domain crossing handling
- 잘못된 클럭 도메인 교차 처리

**Key Principle**: Every violation exists for a reason.
**주요 원칙**: 모든 위반에는 이유가 있습니다.
Understanding the root cause and resolving correctly is more important than simply clearing the violation count.
단순히 위반 횟수를 지우는 것보다 근본 원인을 이해하고 올바르게 해결하는 것이 더 중요합니다.

---

## General Resolution Process
## 일반 해결 프로세스

### 1.
### Analyze the Violation
### 위반 분석

**Extract and review:**
**추출 및 검토:**
- `RULE` - The check ID (e.g., TIMING-16)
- `RULE` - 수표 ID(예: TIMING-16)
- `NAME` - The specific violation instance (e.g., TIMING-16#1)
- `NAME` - 특정 위반 인스턴스(예: TIMING-16#1)
- `SEVERITY` - Critical Warning, Warning, Info, Advisory
- '심각도' - 심각한 경고, 경고, 정보, 주의
- `DETAILS` - Full description from Vivado
- `DETAILS` - Vivado의 전체 설명
- `NETLIST_ELEMENT` - The specific object(s) causing the violation
- `NETLIST_ELEMENT` - 위반을 일으키는 특정 개체

**Understand context:**
**맥락 이해:**
- What timing constraint is involved?
- 어떤 타이밍 제약이 관련되어 있나요?
- What design element is flagged?
- 어떤 디자인 요소가 표시되나요?
- What is the design intent for this element?
- 이 요소의 설계 의도는 무엇입니까?
- Are there related constraints that might conflict?
- 충돌할 수 있는 관련 제약 조건이 있습니까?

### 2.
### Determine Root Cause
### 근본 원인 파악

Ask yourself:
스스로에게 물어보세요:
- Is this a constraint error (wrong value, applied in the wrong place, missing constraint, conflicting constraint)?
- 이것은 제약 조건 오류(잘못된 값, 잘못된 위치에 적용, 제약 조건 누락, 제약 조건 충돌)입니까?
- Is this a design issue (incorrect clock topology, improper CDC handling)?
- 이것이 디자인 문제입니까(잘못된 시계 토폴로지, 부적절한 CDC 처리)?
- Is this expected behavior that should be waived?
- 포기해야 할 예상되는 동작입니까?
- Does the violation indicate missing design information?
- 위반은 설계 정보 누락을 의미합니까?

### 3.
### Identify Resolution Type
### 해결 유형 식별

Classify as:
다음과 같이 분류하세요.
- **AUTO**: Can be fixed automatically based on clear rules
- **AUTO**: 명확한 규칙에 따라 자동으로 수정 가능
  - Redundant constraints
  - 중복 제약
  - Clearly incorrect values (e.g., negative period, clock pessimism is turned off)
  - 분명히 잘못된 값(예: 음수 기간, 시계 비관론이 꺼짐)
  - Missing insertion delay
  - 삽입 지연 누락
  
- **USER**: Requires user decision, can not be certain what the user intends
- **사용자**: 사용자 결정이 필요하며 사용자가 의도한 것이 무엇인지 확신할 수 없습니다.
  - Ambiguous clock relationships
  - 모호한 시계 관계
  - Choice between multiple valid solutions
  - 여러 유효한 솔루션 중에서 선택
  - Not clear which value the user intends
  - 사용자가 어떤 값을 의도하는지 명확하지 않음
  - Fundamental design changes
  - 기본적인 디자인 변경
  - Waiver justification
  - 포기의 정당성

### 4.
### Verify Understanding Before Acting
### 행동하기 전에 이해를 확인하십시오
- Print understanding information to `agent.log` in a clear readable format using tables
- 테이블을 사용하여 명확하게 읽을 수 있는 형식으로 'agent.log'에 이해 정보를 인쇄합니다.
- Make an assessment of confidence level `high||med||low` for each section
- 각 섹션에 대한 신뢰 수준 `high||med||low`를 평가합니다.
- Make an overall assessment of confidence level.
- 신뢰 수준을 전반적으로 평가합니다.

**CRITICAL**: Before proposing or applying any fix, complete the [Agent Understanding Verification Checklist](#agent-understanding-verification-checklist) below.
**중요**: 수정사항을 제안하거나 적용하기 전에 아래 [상담원 이해 확인 체크리스트](#agent-understanding-verification-checklist)를 완료하세요.

---

## Common Violation Categories
## 일반적인 위반 카테고리

### Clock Definition Issues
### 시계 정의 문제
**Examples**: Missing clocks, incorrect periods, wrong sources
**예**: 누락된 시계, 잘못된 기간, 잘못된 소스

**General Resolution**:
**일반 해결**:
1. Trace the clock network to identify the true source
1. 시계 네트워크를 추적하여 실제 소스 식별
2. Verify the intended frequency
2. 의도한 주파수를 확인하세요.
3. Check if user generated clocks are needed for clock modifying blocks or can the auto definition be used
3. 시계 수정 블록에 사용자 생성 시계가 필요한지 또는 자동 정의를 사용할 수 있는지 확인하십시오.
4. For USER decisions: Present options with tradeoffs
4. 사용자 결정의 경우: 장단점이 있는 옵션 제시

### Clock Relationship Issues
### 시계 관계 문제
**Examples**: Missing asynchronous relationships, incorrect synchronous grouping
**예**: 비동기 관계 누락, 잘못된 동기 그룹화

**General Resolution**:
**일반 해결**:
1. Analyze clock domain crossing paths
1. 클록 도메인 교차 경로 분석
2. Determine if clocks are truly asynchronous or related
2. 시계가 실제로 비동기인지 또는 관련되어 있는지 확인
3. For asynchronous clocks: `set_clock_groups -asynchronous`
3. 비동기 시계의 경우: `set_clock_groups -asynchronous`
4. For related clocks: Verify master/derived relationship
4. 관련 시계의 경우: 마스터/파생 관계 확인
5. For USER decisions: Ask about design intent for the clock domains
5. 사용자 결정: 클록 도메인의 설계 의도에 대해 문의

### Constraint Conflicts
### 제약 충돌
**Examples**: Over-constrained paths, contradictory requirements
**예**: 지나치게 제한된 경로, 모순되는 요구 사항

**General Resolution**:
**일반 해결**:
1. Identify all constraints affecting the flagged path/object
1. 플래그가 지정된 경로/객체에 영향을 미치는 모든 제약 조건을 식별합니다.
2. Determine which constraint represents true design intent.
2. 실제 설계 의도를 나타내는 구속조건을 결정합니다.
2. Always ask the user if it is not clear.
2. 명확하지 않은 경우 항상 사용자에게 문의하세요.
3. Remove or modify conflicting constraints
3. 충돌하는 제약조건 제거 또는 수정
4. For AUTO: Remove clearly redundant constraints
4. AUTO의 경우: 명확하게 중복되는 제약 조건 제거
5. For USER: Present the conflict and ask which constraint is correct
5. 사용자의 경우: 충돌을 제시하고 어떤 제약 조건이 올바른지 묻습니다.

### Missing Constraints
### 누락된 제약조건
**Examples**: Unconstrained paths, missing I/O delays
**예**: 제약되지 않은 경로, I/O 지연 누락

**General Resolution**:
**일반 해결**:
1. Identify what constraint type is missing
1. 누락된 제약 조건 유형 식별
2. Determine if the path/object should be constrained
2. 경로/개체를 제한해야 하는지 결정
3. For USER: Always ask for timing requirements (never assume)
3. 사용자의 경우: 항상 타이밍 요구 사항을 묻습니다(가정하지 않음).
4. Collect: clock domain, delay values, interface type
4. 수집: 클럭 도메인, 지연 값, 인터페이스 유형

---

## Agent Understanding Verification Checklist
## 상담원 이해 확인 체크리스트

Before proposing or implementing any resolution, verify your understanding by documenting the following in `agent.log`:
해결책을 제안하거나 구현하기 전에 `agent.log`에 다음 사항을 문서화하여 이해를 확인하십시오.

```
=== VIOLATION UNDERSTANDING: <CHECK_ID>#<N> ===

[ ] PROBLEM IDENTIFICATION
    What specific constraint or design element is problematic?
    → 

[ ] ROOT CAUSE ANALYSIS
    Why is this flagged as a violation?
    → 
    
    What rule or timing requirement is being violated?
    → 

[ ] DESIGN INTENT COMPREHENSION
    What is the designer trying to achieve with this element?
    → 
    
    Is this violation expected/acceptable given the design intent?
    → 

[ ] RESOLUTION UNDERSTANDING
    What is the correct resolution?
    → 
    
    Why is this resolution correct?
    → 
    
    What assumptions have been made to make this decision?
    → 

    What timing/functional impact does this resolution have?
    → 

[ ] ALTERNATIVES EVALUATION (if multiple options exist)
    What other resolutions were considered?
    → 
    
    Why is the selected resolution better than alternatives?
    → 

[ ] UNINTENDED CONSEQUENCES CHECK
    Could this fix negatively impact other constraints or design elements?
    → 
    
    Are there related violations that might be affected?
    → 

[ ] CLASSIFICATION
    Resolution type: [ ] AUTO  [ ] USER  [ ] WAIVER
    Justification for classification:
    → 

=== END VERIFICATION ===
```

**Rule**: If you cannot confidently fill out this checklist, classify the resolution as **USER** and ask for guidance.
**규칙**: 이 체크리스트를 자신있게 작성할 수 없는 경우 해결 방법을 **사용자**로 분류하고 지침을 요청하세요.

---

## Resolution Patterns
## 해상도 패턴

### Pattern 1: Removing Redundant Constraints
### 패턴 1: 중복 제약 조건 제거
**When**: Same constraint applied multiple times to the same object
**시기**: 동일한 객체에 동일한 제약 조건이 여러 번 적용됨

**AUTO Criteria**:
**자동 기준**:
- Constraints are byte-for-byte identical
- 제약 조건은 바이트 단위로 동일합니다.
- Applied to identical targets
- 동일한 대상에 적용
- No variation in options/parameters
- 옵션/매개변수에 변화가 없음

**Actions**:
**작업**:
1. Identify the duplicate lines in the XDC
1. XDC에서 중복 라인 식별
2. Keep the first occurrence
2. 첫 번째 항목 유지
3. Comment out subsequent duplicates with explanation
3. 설명과 함께 후속 중복 항목에 주석 처리
4. Document which file each duplicate came from
4. 각 사본이 어느 파일에서 왔는지 문서화

**Example**:
**예**:
```tcl
# From timing_constraints.xdc (line 45) - KEPT
create_clock -period 10.000 [get_ports REF_CLK]

# From imported_constraints.xdc (line 12) - REMOVED: Duplicate of timing_constraints.xdc line 45
# create_clock -period 10.000 [get_ports REF_CLK]
```

### Pattern 2: Correcting Clock Periods
### 패턴 2: 클럭 주기 수정
**When**: Clock period doesn't match actual frequency
**시기**: 클록 주기가 실제 주파수와 일치하지 않습니다.

**USER Criteria** (always ask):
**사용자 기준**(항상 물어보세요):
- "What is the correct frequency for clock `X`?"
- "클럭 `X`의 정확한 주파수는 무엇입니까?"
- "Should this be `Y` MHz or `Z` MHz?"
- "이것은 `Y` MHz여야 합니까, 아니면 `Z` MHz여야 합니까?"
- Present current value and ask for confirmation
- 현재 가치를 제시하고 확인을 요청하세요.

**AUTO Criteria** (rare):
**자동 기준**(드물게):
- Only if documentation or IP core parameters explicitly state the frequency
- 문서나 IP 핵심 매개변수가 주파수를 명시적으로 명시한 경우에만
- And violation shows a clear mismatch
- 그리고 위반은 명백한 불일치를 보여줍니다.

**Actions**:
**작업**:
1. Get user confirmation of correct frequency
1. 올바른 빈도에 대한 사용자 확인 받기
2. Calculate correct period (ns = 1000 / MHz)
2. 정확한 주기 계산(ns = 1000/MHz)
3. Modify the `create_clock` command
3. `create_clock` 명령 수정
4. Document the change with frequency conversion
4. 빈도 변환을 통해 변경 사항을 문서화합니다.

### Pattern 3: Establishing Clock Relationships
### 패턴 3: 시계 관계 설정
**When**: Paths between clocks are not properly characterized
**시기**: 클록 사이의 경로가 제대로 특성화되지 않았습니다.

**USER Criteria** (default):
**사용자 기준**(기본값):
- "Are clocks `A` and `B` asynchronous to each other?"
- "시계 'A'와 'B'는 서로 비동기인가요?"
- "Is clock `B` derived from clock `A`?"
- "시계 'B'는 시계 'A'에서 파생되었나요?"

**AUTO Criteria**:
**자동 기준**:
- Only if netlist clearly shows MMCM/PLL generating one from another
- 넷리스트에 서로 생성되는 MMCM/PLL이 명확하게 표시되는 경우에만
- And no existing clock relationship constraints contradict this
- 그리고 기존 시계 관계 제약 조건이 이에 모순되지 않습니다.

**Actions**:
**작업**:
1. Trace clock sources using `all_fanin -flat -only_cells`
1. `all_fanin -plat -only_cells`를 사용하여 클럭 소스 추적
2. Identify relationship (asynchronous, generated, related)
2. 관계 식별(비동기, 생성, 관련)
3. For asynchronous: Add `set_clock_groups -asynchronous`
3. 비동기식의 경우: `set_clock_groups -asynchronous` 추가
4. For generated: Convert to `create_generated_clock`
4. 생성된 경우: `create_generated_clock`으로 변환
5. For related: Verify existing constraints are correct
5. 관련: 기존 제약 조건이 올바른지 확인하세요.

### Pattern 4: Creating Waivers
### 패턴 4: 면제 생성
**When**: Violation is expected and acceptable
**시기**: 위반이 예상되며 허용 가능

**USER Criteria** (always):
**사용자 기준**(항상):
- Must get explicit waiver justification from user
- 사용자로부터 명시적인 포기 사유를 얻어야 합니다.
- Must document design intent
- 설계 의도를 문서화해야 함
- Must confirm violation does not impact functionality
- 위반이 기능에 영향을 미치지 않는지 확인해야 함

**Actions**:
**작업**:
1. Ask: "This violation appears to be `<description>`.
1. 질문: "이 위반사항은 `<설명>`인 것 같습니다.
1. Is this expected behavior?"
1. 이게 예상된 동작인가요?"
2. If yes, ask: "Please provide justification for waiving (will be recorded in design):"
2. 그렇다면 다음과 같이 질문하십시오. "포기 사유를 제공해 주십시오(설계에 기록됩니다)."
3. Collect: Reason, any compensating design measures, verification performed
3. 수집: 이유, 보상 설계 조치, 수행된 검증
4. Generate waiver command with full documentation
4. 전체 문서를 포함하여 면제 명령 생성
5. Add tags for related waivers if part of a pattern
5. 패턴의 일부인 경우 관련 면제에 대한 태그를 추가하세요.

**Example Exchange**:
**교환 예시**:
```
Agent: "TIMING-17 flags that clock CLK_100 has no input delay constraints. 
        Is this clock intended to be unconstrained?"
User:  "Yes, it's a free-running test clock, not used in functional paths"
Agent: "I'll create a waiver. Please provide justification:"
User:  "Test clock for debug only, isolated from functional logic"
```

---

## When to Ask for User Input
## 사용자 입력을 요청하는 경우

### Always Ask (USER) For:
### 항상 물어보세요(사용자):

1. **Design Intent Questions**
1. **디자인 의도 질문**
   - Clock frequencies and relationships
   - 클록 주파수 및 관계
   - Whether paths are critical or can be ignored
   - 경로가 중요한지 또는 무시할 수 있는지 여부
   - Interface timing requirements
   - 인터페이스 타이밍 요구 사항
   - Whether violations are expected behavior
   - 위반이 예상되는 동작인지 여부

2. **Choice Between Valid Alternatives**
2. **유효한 대안 중 선택**
   - Multiple clocks could be the source
   - 여러 시계가 소스가 될 수 있습니다.
   - Either asynchronous or synchronous grouping is valid
   - 비동기식 또는 동기식 그룹화가 유효합니다.
   - Different constraint formulations achieve same goal
   - 서로 다른 제약 공식이 동일한 목표를 달성합니다.

3. **Waivers**
3. **면제**
   - Justification for accepting a violation
   - 위반을 수락하는 근거
   - Confirmation that violation doesn't affect functionality
   - 위반이 기능에 영향을 미치지 않는다는 확인

4. **Modifications with Broad Impact**
4. **광범위한 영향을 미치는 수정**
   - Changes that affect multiple constraints
   - 여러 제약 조건에 영향을 미치는 변경 사항
   - Modifications to primary clock definitions
   - 기본 클록 정의 수정
   - Anything that could impact timing closure
   - 타이밍 종료에 영향을 미칠 수 있는 모든 것

### Can Proceed Automatically (AUTO) When:
### 다음과 같은 경우 자동으로 진행할 수 있습니다(AUTO).

1. **Clear Constraint Errors**
1. **제약조건 오류 지우기**
   - Duplicate constraints (identical)
   - 중복 제약조건(동일)
   - Constraints on non-existent objects
   - 존재하지 않는 객체에 대한 제약
   - Syntax errors in constraint commands
   - 제약 조건 명령의 구문 오류

2. **Obvious Redundancies**
2. **명백한 중복**
   - Multiple identical timing exceptions
   - 여러 개의 동일한 타이밍 예외
   - Superfluous constraints that are subsumed by others
   - 다른 사람이 포함하는 불필요한 제약

3. **Netlist-Derived Information**
3. **Netlist에서 파생된 정보**
   - Generated clocks where source is unambiguous
   - 소스가 명확하게 생성된 시계
   - Clock relationships evident from BUFG/MMCM topology
   - BUFG/MMCM 토폴로지에서 명백한 클록 관계

**Golden Rule**: When in doubt, classify as USER and ask.
**황금률**: 의심스러운 경우 USER로 분류하고 질문하세요.

---

## Output to agent.log
## Agent.log로 출력

For every violation being resolved, output your verification checklist to `./results/agent.log`:
모든 위반 사항이 해결되면 확인 체크리스트를 `./results/agent.log`에 출력하세요.

```bash
cat >> ./results/agent.log << 'EOF'

=== VIOLATION UNDERSTANDING: TIMING-16#1 ===

[ ✓ ] PROBLEM IDENTIFICATION
    What specific constraint or design element is problematic?
    → create_clock constraint on port CLK_100 with period 5.000 ns
    
[ ✓ ] ROOT CAUSE ANALYSIS
    Why is this flagged as a violation?
    → The actual clock frequency is 100 MHz (10 ns period), but constraint specifies 200 MHz (5 ns)
    
    What rule or timing requirement is being violated?
    → Clock period does not match the actual frequency delivered by the external source
    
... (continue with full checklist) ...

=== END VERIFICATION ===

EOF
```

This creates a record of your reasoning and ensures thorough understanding before action.
이를 통해 추론에 대한 기록이 생성되고 조치를 취하기 전에 철저한 이해가 보장됩니다.

---

## Summary
## 요약

**Remember**:
**기억하세요**:
1. Read specific check reference files first (`TIMING-X.md`)
1. 특정 검사 참조 파일을 먼저 읽으십시오(`TIMING-X.md`)
2. These general guidelines apply when no specific reference exists
2. 특정 참조가 없는 경우 이러한 일반 지침이 적용됩니다.
3. Complete the understanding verification checklist before acting
3. 행동하기 전에 이해 확인 체크리스트를 완료하세요
4. When in doubt, ask the user
4. 의심스러운 경우 사용자에게 물어보세요.
5. Document all reasoning in `agent.log`
5. 모든 추론을 `agent.log`에 문서화하세요.
6. Prioritize understanding over quick fixes
6. 빠른 수정보다 이해를 우선시합니다.

**The goal is correct resolution, not just clearing violation counts.**
**목표는 위반 횟수를 지우는 것이 아니라 올바른 해결입니다.**
