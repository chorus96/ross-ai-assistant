<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# TIMING-16: Large setup violation
# TIMING-16: 대규모 설정 위반

## Metadata
## 메타데이터

- **Check ID**: TIMING-16
- **ID 확인**: 타이밍-16
- **Severity**: Warning
- **심각도**: 경고
- **Group**: TIMING
- **그룹**: 타이밍
- **Hierarchy**: Timing.Bad Practice
- **계층 구조**: Timing.Bad Practice
- **Owner**: gservel
- **소유자**: gservel

## Description
## 설명

Large setup violation between sequential elements that may be difficult to fix during post-placement implementation.
배치 후 구현 중에 수정하기 어려울 수 있는 순차 요소 간의 대규모 설정 위반입니다.

## Message
## 메시지

```
There is a large setup violation of <MESSAGE_STRING> ns between <NETLIST_ELEMENT> (clocked by <CLOCK_GROUP>) and <NETLIST_ELEMENT> (clocked by <CLOCK_GROUP>). Large setup violations at the end of those stages might be difficult to fix during the post-placement implementation flow and could be the result of non-optimal XDC constraints or non-optimal design architecture
```

**Example:**
**예:**
```
[Real violation message from actual design]
```

## Explanation
## 설명

This check identifies paths with large setup violations that appear early in the design flow (typically after synthesis or early placement).
이 검사는 설계 흐름 초기(일반적으로 합성 또는 초기 배치 후)에 나타나는 대규모 설정 위반이 있는 경로를 식별합니다.
Large setup violations at these stages often indicate fundamental issues with either the timing constraints or the design architecture itself, rather than just placement/routing issues.
이러한 단계에서 대규모 설정 위반은 단순한 배치/라우팅 문제가 아니라 타이밍 제약이나 설계 아키텍처 자체의 근본적인 문제를 나타내는 경우가 많습니다.

Common root causes include:
일반적인 근본 원인은 다음과 같습니다.
- Unrealistic clock period constraints
- 비현실적인 클럭 주기 제약
- Missing or incorrect clock relationships (set_clock_groups, set_false_path, set_max_delay)
- 누락되거나 잘못된 시계 관계(set_clock_groups, set_false_path, set_max_delay)
- Design architecture issues requiring pipelining or optimization
- 파이프라인 또는 최적화가 필요한 설계 아키텍처 문제
- Incorrect input/output delays affecting path budgets
- 경로 예산에 영향을 미치는 잘못된 입력/출력 지연

## Generate Data
## 데이터 생성

Collect information about the violating path:
위반 경로에 대한 정보를 수집합니다.
1. Get path details: `report_timing -from <NETLIST_ELEMENT> -to <NETLIST_ELEMENT> -delay_type max`
1. 경로 세부정보 가져오기: `report_timing -from <NETLIST_ELEMENT> -to <NETLIST_ELEMENT> -delay_type max`
2. Verify clock relationships: `report_clock_interaction`
2. 시계 관계 확인: `report_clock_interaction`

## Flow
## 흐름

1. **Gather Data** → Collect violation properties, path timing report, and clock information
1. **데이터 수집** → 위반 속성, 경로 타이밍 보고서 및 클럭 정보 수집

2. **Analyze** → Determine if issue is constraints or architectural
2. **분석** → 문제가 제약 조건인지 아키텍처인지 확인
Constraint Issues have
제약 문제가 있습니다
a. small timing requirements.
가. 작은 타이밍 요구 사항.
Typically < 1 ns.
일반적으로 1ns 미만입니다.
b. inter clock paths with high skew  |skew| > 0.750 ns -  See suboptimal clock network
비. 스큐가 높은 인터 클럭 경로 |skew| > 0.750ns - 최적이 아닌 클록 네트워크 참조
c. inter clock paths with high uncertainty - Phase Error (PE) -  See suboptimal clock network
c. 불확실성이 높은 클럭 간 경로 - PE(위상 오류) - 최적이 아닌 클럭 네트워크 참조

Architectural Issues
건축 문제
d.
Number of LOGIC_LEVELS is high for REQUIREMENT.
REQUIREMENT의 LOGIC_LEVELS 수가 높습니다.
Ask the user to examine Net/LUT logic levels in `report_qor_assessment`
사용자에게 `report_qor_assessment`에서 Net/LUT 논리 수준을 검사하도록 요청합니다.
```tcl
set lls [get_property LOGIC_LEVELS [get_timing_paths <PATH>]]
set req [get_property REQUIREMENT [get_timing_paths <PATH>]]
```
e.
Sub optimal clock network that is creating skew or uncertainties
왜곡이나 불확실성을 생성하는 최적이 아닌 클록 네트워크
f.
Logic delay is high v net delay
논리 지연이 높음 v 순 지연

3. **Resolve** → Update constraints or flag for design changes
3. **해결** → 설계 변경에 대한 구속조건 또는 플래그 업데이트
Determine if it is a constraint or structural issue.
제약사항인지 구조적 문제인지 확인합니다.
**Constraints**
**제약조건**
1. For tight timing requirements, see if adequate clock exceptions are in place between clock groups.
1. 엄격한 타이밍 요구 사항의 경우 클록 그룹 간에 적절한 클록 예외가 있는지 확인하세요.
1. Explore:
1. 탐색:
a.
`set_clock_groups`
`세트_시계_그룹`
b.
`set_max_delay -datapath_only`
`set_max_delay -datapath_only`

**Clock Network**
**클럭 네트워크**
If it is a suboptimal clock network issue
최적이 아닌 클럭 네트워크 문제인 경우
1. Determine if an MBUF can be used in Versal or later technology
1. Versal 이상의 기술에서 MBUF를 사용할 수 있는지 확인
2. For Ultrascale, if there is PE, determine if the same MMCM output can be used with BUFGCE_DIV to remove phase error.
2. Ultrascale의 경우 PE가 있는 경우 동일한 MMCM 출력을 BUFGCE_DIV와 함께 사용하여 위상 오류를 제거할 수 있는지 확인합니다.

**Logic level**
**논리 수준**
If it is a logic level issue
로직 레벨 문제인 경우
1. Determine if retiming is set globally at synthesis
1. 리타이밍이 합성 시 전역적으로 설정되었는지 확인
2. Suggest that RTL recoding could improve the logic levels and point at the file name and line number
2. RTL 기록이 논리 수준을 향상시키고 파일 이름과 줄 번호를 가리킬 수 있다고 제안합니다.
```tcl
get_property LINE_NUMBER [get_cells -of [get_pins [get_property ENDPOINT_PIN [get_timing_path <PATH>]]]]
get_property FILE_NAME [get_cells -of [get_pins [get_property ENDPOINT_PIN [get_timing_path <PATH>]]]]
```

4. **Verify** → Re-run methodology checks and timing analysis
4. **검증** → 방법론 점검 및 타이밍 분석 재실행

### DON'T:
### 하지 말아야 할 것:
❌ Rerun methodology checks or timing analysis if there is no updated constraint or RTL
❌ 업데이트된 제약 조건이나 RTL이 없는 경우 방법론 검사 또는 타이밍 분석을 다시 실행합니다.


## Verification
## 검증

* [ ] Setup violation is reduced to acceptable level or properly constrained
* [ ] 설정 위반이 허용 가능한 수준으로 줄어들거나 적절하게 제한됩니다.
* [ ] Clock relationships are correctly defined
* [ ] 시계 관계가 올바르게 정의되었습니다.
* [ ] No new unresolved methodology warnings are created
* [ ] 해결되지 않은 새로운 방법론 경고가 생성되지 않습니다.
* [ ] Timing closure strategy is appropriate for the violation severity
* [ ] 위반 심각도에 따라 타이밍 종결 전략이 적절함

## Final Report Guidance
## 최종 보고서 지침

Document the resolution:
해결 방법을 문서화합니다.
- If constraints were updated: List the specific XDC changes
- 제약 조건이 업데이트된 경우: 특정 XDC 변경 사항을 나열하세요.
- If design changes needed: Document the architectural recommendations
- 설계 변경이 필요한 경우: 아키텍처 권장 사항을 문서화합니다.
- Include before/after timing numbers
- 타이밍 번호 이전/이후 포함
- Note any waived violations with justification
- 정당한 사유와 함께 면제된 위반 사항을 기록하세요.

## References
## 참고자료

- [UG906](https://docs.amd.com/r/en-US/ug906-vivado-design-analysis) - Design Analysis and Closure Techniques
- [UG906](https://docs.amd.com/r/en-US/ug906-vivado-design-analytic) - 설계 분석 및 클로저 기술
- [UG903](https://docs.amd.com/r/en-US/ug903-vivado-using-constraints) - Using Constraints
- [UG903](https://docs.amd.com/r/en-US/ug903-vivado-using-constraints) - 제약 조건 사용
- [UG949](https://docs.amd.com/r/en-US/ug949-vivado-design-methodology) - UltraFast Design Methodology (UltraScale/UltraScale+)
- [UG949](https://docs.amd.com/r/en-US/ug949-vivado-design-methodology) - UltraFast 설계 방법론(UltraScale/UltraScale+)
- [UG1388](https://docs.amd.com/r/en-US/ug1388-acap-system-integration-validation-methodology) - Vivado Design Methodology (Versal)
- [UG1388](https://docs.amd.com/r/en-US/ug1388-acap-system-integration-validation-methodology) - Vivado 설계 방법론(Versal)
- [CommandHistory](/VIVADO_IMPLEMENTATION_COMMAND_HISTORY.md) - Finding command history in Vivado
- [CommandHistory](/VIVADO_IMPLEMENTATION_COMMAND_HISTORY.md) - Vivado에서 명령 기록 찾기
