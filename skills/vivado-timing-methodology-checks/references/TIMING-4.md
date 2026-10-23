<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# TIMING-4: Invalid primary clock redefinition on a clock tree
# TIMING-4: 클록 트리에서 잘못된 기본 클록 재정의

## Metadata
## 메타데이터

- **Check ID**: TIMING-4
- **ID 확인**: 타이밍-4
- **Severity**: Critical Warning
- **심각도**: 심각한 경고
- **Group**: TIMING
- **그룹**: 타이밍
- **Hierarchy Name**: Timing.Bad Practice
- **계층 구조 이름**: Timing.Bad Practice
- **First Release**: 2013.3
- **첫 번째 릴리스**: 2013.3

## Description
## 설명

Invalid clock redefinition on a clock tree.
시계 트리에서 잘못된 시계 재정의.

### Full Message Description Property
### 전체 메시지 설명 속성

```
Invalid clock redefinition on a clock tree. The primary clock <CLOCK_GROUP> is defined downstream of clock <CLOCK_GROUP> and overrides its insertion delay and/or waveform definition
```

**Example:**
**예:**
```
Invalid clock redefinition on a clock tree. The primary clock clk_b is defined downstream of clock clk_a and overrides its insertion delay and/or waveform definition
```

### Explanation
### 설명

A primary clock is defined on a net or pin that is downstream from another primary clock.
기본 클럭은 다른 기본 클럭의 다운스트림인 네트 또는 핀에 정의됩니다.
When a clock is redefined downstream on the same clock tree, it overrides the upstream clock's insertion delay and/or waveform definition.
동일한 클록 트리에서 클록이 다운스트림으로 재정의되면 업스트림 클록의 삽입 지연 및/또는 파형 정의가 재정의됩니다.
This creates conflicts in clock analysis and can lead to incorrect timing results.
이는 클럭 분석에 충돌을 일으키고 잘못된 타이밍 결과를 초래할 수 있습니다.

Clock definitions should typically follow this hierarchy:
클록 정의는 일반적으로 다음 계층 구조를 따라야 합니다.
1. Primary clocks should be defined at input ports or appropriate clock source pins
1. 기본 클록은 입력 포트 또는 적절한 클록 소스 핀에서 정의되어야 합니다.
2. Generated clocks should be used to define downstream clocks derived from primary clocks
2. 생성된 클록은 기본 클록에서 파생된 다운스트림 클록을 정의하는 데 사용되어야 합니다.
3. Multiple primary clocks should not be defined on the same clock tree unless intentional and properly understood
3. 의도적이고 적절하게 이해되지 않는 한 여러 기본 클록을 동일한 클록 트리에 정의해서는 안 됩니다.

**DO**
✅ Identify both clock definitions - the upstream clock and the downstream redefinition
✅ 업스트림 클록과 다운스트림 재정의 등 두 가지 클록 정의를 모두 식별하세요.
✅ Verify which clock is the intended primary clock
✅ 어떤 시계가 의도된 기본 시계인지 확인하세요
✅ Check if there are timing exceptions or other constraints that reference the downstream clock name
✅ 다운스트림 클럭 이름을 참조하는 타이밍 예외나 기타 제약 조건이 있는지 확인하세요.
✅ Use `trace_clock_tree.json` to trace the clock tree and confirm the primary clock is defined at the correct object
✅ `trace_clock_tree.json`을 사용하여 시계 트리를 추적하고 기본 시계가 올바른 개체에 정의되어 있는지 확인하세요.
✅ Use `report_clocks` to understand the clock tree structure
✅ 시계 트리 구조를 이해하려면 `report_clocks`를 사용하세요.

**DO NOT**
**하지 마세요**
❌ Assume the first clock definition in the XDC file is the problematic one
❌ XDC 파일의 첫 번째 시계 정의에 문제가 있다고 가정합니다.
❌ Remove clock definitions without checking for references in timing exceptions
❌ 타이밍 예외의 참조를 확인하지 않고 시계 정의를 제거합니다.
❌ Ignore insertion delay or waveform differences between the clocks
❌ 삽입 지연이나 클록 간의 파형 차이를 무시합니다.

## Generate Data
## 데이터 생성

1. Identify both clocks mentioned in the violation message
1. 위반 메시지에 언급된 두 시계를 모두 식별하세요.
```tcl
# Get properties of the downstream redefined clock
report_property [get_clocks <downstream_clock_name>]

# Get properties of the upstream clock
report_property [get_clocks <upstream_clock_name>]
```

2. Find the pins/nets where each clock is defined
2. 각 클록이 정의된 핀/네트 찾기
report_property Start Time : 1772102992888919 (2026 02 26 03 49 52)
report_property 시작 시간 : 1772102992888919 (2026 02 26 03 49 52)
Property           Type     Read-only  Visible  Value
속성 유형 읽기 전용 표시 값
CLASS              string   true       true     clock
CLASS 문자열 참 참 시계
FILE_NAME          string   true       true     <project_dir>/sources_1/bd/<bd_name>/ip/<bd_name>_ps_block_0/bd_0/ip/ip_0/<clock_source>.xdc
FILE_NAME 문자열 true true <project_dir>/sources_1/bd/<bd_name>/ip/<bd_name>_ps_block_0/bd_0/ip/ip_0/<clock_source>.xdc
INPUT_JITTER       double   true       true     0.000
INPUT_JITTER 이중 참 참 0.000
IS_GENERATED       bool     true       true     0
IS_GENERATED bool 참 참 0
IS_PROPAGATED      bool     true       true     1
IS_PROPAGATED bool 참 참 1
IS_USER_GENERATED  bool     true       true     0
IS_USER_GENERATED bool 참 참 0
IS_VIRTUAL         bool     true       true     0
IS_VIRTUAL bool 참 참 0
LINE_NUMBER        int      true       true     52
LINE_NUMBER int true true 52
MODULE             string   true       true     Static
MODULE 문자열 true true 정적
NAME               string   true       true     clk_pl_0
NAME 문자열 참 참 clk_pl_0
PERIOD             double   true       true     3.000
PERIOD 이중 참 참 3.000
SOURCE_PINS        string*  true       true     hwtest_harness_top_i/Base_layer/ps_block/inst/pspmc_0/inst/PS9_inst/PMCRCLKCLK[0]
SOURCE_PINS 문자열* true true hwtest_harness_top_i/Base_layer/ps_block/inst/pspmc_0/inst/PS9_inst/PMCRCLKCLK[0]
SYSTEM_JITTER      double   true       true     needs timing update***
SYSTEM_JITTER double true true 타이밍 업데이트 필요***
WAVEFORM           double*  true       true     0.000 1.500
WAVEFORM double* 참 참 0.000 1.500
WEIGHT             double   true       true     1.000
WEIGHT 더블 참 참 1.000
report_property End Time : 1772102992890408 (2026 02 26 03 49 52)
report_property 종료 시간 : 1772102992890408 (2026 02 26 03 49 52)
```tcl
# For the downstream clock
get_pins [get_property SOURCE_PINS [get_clocks <upstream_clock_name>]]
get_ports [get_property SOURCE_PINS [get_clocks <upstream_clock_name>]]

# For the upstream clock
get_pins [get_property SOURCE_PINS [get_clocks <downstream_clock_name>]]

```

3. Trace the connectivity between the two clock definition points
3. 두 클럭 정의 지점 간의 연결을 추적합니다.
```tcl
# Use Clock Tracing Methodology to verify the downstream relationship
# See ../references/CLOCK_TRACING.md
```

4. Check for constraints that reference the downstream clock
4. 다운스트림 클록을 참조하는 제약 조건을 확인하세요.
Searc `constraints.xdc` file the downstream clock name
'constraints.xdc' 파일을 검색하여 다운스트림 시계 이름을 저장하세요.

## Flow
## 흐름

1. Search the `./tmp/constraints.xdc` for both `create_clock` constraints
1. 두 `create_clock` 제약 조건에 대해 `./tmp/constraints.xdc`를 검색하세요.
   - Find the constraint creating the upstream clock
   - 업스트림 클록을 생성하는 제약 조건 찾기
   - Find the constraint creating the downstream clock
   - 다운스트림 클록을 생성하는 제약 조건 찾기

2. Verify the clock definitions in the constraints match the violation message
2. 제약 조건의 시계 정의가 위반 메시지와 일치하는지 확인하세요.
```tcl
# Capture properties
set downstream_clk [get_clocks <downstream_clock_name>]
set upstream_clk [get_clocks <upstream_clock_name>]
```

**DO NOT**
**하지 마세요**
❌ Automate fixes when the constraint file value is null
❌ 제약 조건 파일 값이 null인 경우 자동 수정
❌ Automate fixes when the line number value is null
❌ 줄 번호 값이 null인 경우 자동 수정
❌ Automate fixes when the information does not match the violation data
❌ 정보가 위반 데이터와 일치하지 않는 경우 자동 수정

3. Determine the clock tree relationship
3. 클록 트리 관계 결정
   - Use Clock Tracing Methodology to confirm the downstream relationship
   - 클럭 추적 방법론을 사용하여 다운스트림 관계 확인

4. Determine the best solution to offer the user
4. 사용자에게 제공할 최상의 솔루션 결정

**CASE 1:**
**사례 1:**
* **Type**: Automated Resolution
* **유형**: 자동 해결
* **When**: The downstream clock has the same period and waveform as would be expected from the upstream clock, and the downstream clock name is referenced in timing exceptions
* **언제**: 다운스트림 클록은 업스트림 클록에서 예상되는 것과 동일한 주기 및 파형을 가지며 다운스트림 클록 이름은 타이밍 예외에서 참조됩니다.

1. Remove the `create_clock` constraint on the downstream pin/net
1. 다운스트림 핀/넷에서 'create_clock' 제약 조건을 제거합니다.
2. Create a `create_generated_clock` constraint on the downstream pin/net that references the upstream clock
2. 업스트림 클록을 참조하는 다운스트림 핀/네트에 `create_generated_clock` 제약 조건을 생성합니다.
   - Use the same clock name as the removed constraint
   - 제거된 제약 조건과 동일한 시계 이름을 사용하세요.
   - Set `-source` to point to the upstream clock's definition point
   - 업스트림 클록의 정의 지점을 가리키도록 `-source`를 설정합니다.
   - Maintain the same period and waveform
   - 동일한 주기와 파형을 유지

**DO**
✅ Preserve the clock name when it's referenced elsewhere
✅ 다른 곳에서 참조될 때 시계 이름을 유지하세요
✅ Consider if the constraint is coming from an IP file
✅ 제약 조건이 IP 파일에서 오는지 고려하세요.

**DO NOT**
**하지 마세요**
❌ Apply this fix if the period or waveform is intentionally different
❌ 주기나 파형을 의도적으로 다르게 한 경우 이 수정사항을 적용하세요.

**CASE 2:**
**사례 2:**
* **Type**: Automated Resolution
* **유형**: 자동 해결
* **When**: The downstream clock has the same period and waveform as the upstream clock, and is NOT referenced in any timing exceptions
* **언제**: 다운스트림 클록은 업스트림 클록과 동일한 주기 및 파형을 가지며 타이밍 예외에서 참조되지 않습니다.

1. Remove the `create_clock` constraint on the downstream pin/net
1. 다운스트림 핀/넷에서 'create_clock' 제약 조건을 제거합니다.
2. Allow the clock to propagate naturally from the upstream definition
2. 클록이 업스트림 정의에서 자연스럽게 전파되도록 허용

**CASE 3:**
**사례 3:**
* **Type**: User Resolution
* **유형**: 사용자 해상도
* **When**: The downstream clock has different period or waveform than expected from the upstream clock
* **언제**: 다운스트림 클록의 주기 또는 파형이 업스트림 클록에서 예상한 것과 다릅니다.

**Prompt**:
**프롬프트**:
```
The downstream clock <downstream_clock_name> has a different period/waveform than would be expected from the upstream clock <upstream_clock_name>. 

Downstream clock: period=<period>, waveform={<waveform>}
Expected from upstream: period=<expected_period>, waveform={<expected_waveform>}

This requires manual review to determine:
1. Do you want to waive the methodology violation?
2. Should the downstream clock be removed?
3. Should the upstream clock be modified?
```


## Additional Checks
## 추가 점검

- Verify if there are any clock modifying blocks (MMCM, PLL, BUFG) between the two clock definitions
- 두 클록 정의 사이에 클록 수정 블록(MMCM, PLL, BUFG)이 있는지 확인하십시오.
- Check if the constraints come from an IP like clocking wizard
- 제약 조건이 클럭킹 마법사와 같은 IP에서 오는지 확인하세요.
- Check if both clocks are used in exceptions
- 예외에서 두 시계가 모두 사용되는지 확인

## References
## 참고자료

- UltraFast Design Methodology Guide (UG949)
- UltraFast 설계 방법론 가이드(UG949)
- Vivado Design Suite User Guide: Design Analysis and Closure Techniques (UG906)
- Vivado Design Suite 사용자 가이드: 설계 분석 및 폐쇄 기술(UG906)
- Vivado Design Suite User Guide: Using Constraints (UG903) - Section on Clock Constraints
- Vivado Design Suite 사용자 가이드: 제약 조건 사용(UG903) - 클록 제약 조건 섹션
- [Clock Tracing Methodology](./CLOCK_TRACING.md)
- [클럭 추적 방법론](./CLOCK_TRACING.md)
- [IP Constraints](./IP_CONSTRAINTS.md)
- [IP 제약](./IP_CONSTRAINTS.md)

## Examples
## 예

*To be documented with real-world examples from TIMING-4 testcase as required.*
*필요에 따라 TIMING-4 테스트 사례의 실제 사례를 문서화합니다.*
