<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# TIMING-3: Invalid primary clock on Clock Modifying Block
# TIMING-3: 클럭 수정 블록의 잘못된 기본 클럭

## Metadata
## 메타데이터

- **Check ID**: TIMING-3
- **ID 확인**: 타이밍-3
- **Severity**: Critical Warning
- **심각도**: 심각한 경고
- **Group**: 1
- **그룹**: 1
- **Priority**: 1
- **우선순위**: 1
- **Hierarchy Name**: Timing
- **계층 구조 이름**: 타이밍
- **First Release**: 2025.1
- **첫 번째 릴리스**: 2025.1

## Description
## 설명

Clock modifying block settings do not match the expected settings where expected settings are based on input clock settings and clock modifying block parameters.
시계 수정 블록 설정은 예상 설정이 입력 시계 설정 및 시계 수정 블록 매개변수를 기반으로 하는 예상 설정과 일치하지 않습니다.

### Full Message Description property.
### 전체 메시지 설명 속성입니다.

```
A primary clock <CLOCK_GROUP> is created on the output pin or net <NETLIST_ELEMENT> of a Clock Modifying Block
```

**Example:**
**예:**
```
A primary clock GTX_CLK is created on the output pin or net ios_0/mmcm_0/CLKOUT2 of a Clock Modifying Block
```

* clock name: GTX_CLK
* 시계 이름: GTX_CLK
* Pin element: net ios_0/mmcm_0/CLKOUT2
* 핀 요소: net ios_0/mmcm_0/CLKOUT2
* Cell element: net ios_0/mmcm_0
* 셀 요소: net ios_0/mmcm_0

### Explanation
### 설명

The pin element has a clock defined on it that is different to the expected value.
핀 요소에는 예상 값과 다른 시계가 정의되어 있습니다.
There will be either a `create_clock` or `create_generated_clock` constraint defined in the constraints on the `net ios_0/mmcm_0/CLKOUT2` pin.
`net ios_0/mmcm_0/CLKOUT2` 핀의 제약 조건에 정의된 `create_clock` 또는 `create_generated_clock` 제약 조건이 있습니다.

The expected value is based off input clock and clock modifying properties.
예상 값은 입력 클록 및 클록 수정 속성을 기반으로 합니다.
The input clock can be taken to be pins `CLKIN1` or `CLKIN2` on an MMCM or use `get_timing_arcs` to find the correct source pin.
입력 클럭은 MMCM의 'CLKIN1' 또는 'CLKIN2' 핀으로 사용하거나 'get_timing_arcs'를 사용하여 올바른 소스 핀을 찾을 수 있습니다.

**Example:**
**예:**
An MMCM with an input clock CLKIN1 has:
입력 클록 CLKIN1이 있는 MMCM은 다음을 갖습니다.
* period = 10 ns
* 주기 = 10ns
* waveform = 50:50
* 파형 = 50:50

The MMCM has output pin used CLKOUT0:
MMCM에는 CLKOUT0을 사용하는 출력 핀이 있습니다.
* CLKFBOUT_MULT =30
* CLKFBOUT_MULT =30
* DIVCLK_DIVIDE =1
* DIVCLK_DIVIDE =1
* CLKOUT0_DIVIDE =30
* CLKOUT0_DIVIDE =30
* CLKOUT0_PHASE =0.000
* CLKOUT0_PHASE =0.000
* CLKOUT0_DUTY_CYCLE =0.500
* CLKOUT0_DUTY_CYCLE =0.500

Expected output clock
예상 출력 클럭
CLKOUT0 period = input period * (CLKFBOUT_MULT/DIVCLK_DIVIDE) / CLKOUT0_DIVIDE = 10 ns
CLKOUT0 기간 = 입력 기간 * (CLKFBOUT_MULT/DIVCLK_DIVIDE) / CLKOUT0_DIVIDE = 10ns

**DO**
✅  Understand the clock modifying equation from the documentation
✅ 문서에서 시계 수정 방정식을 이해하세요
✅  Perform the calculation to prove understanding of the message
✅ 메시지에 대한 이해를 증명하기 위해 계산을 수행합니다.
✅  Identify the constraint causing the issue
✅ 문제를 일으키는 제약 조건 식별
✅  Leverage Vivado documentation to determine the properties that influence how the clock is modified.
✅ Vivado 문서를 활용하여 시계 수정 방식에 영향을 미치는 속성을 확인하세요.

**DO NOT**
**하지 마세요**
❌ Guess the calculation
❌ 계산을 맞춰보세요


## Generate Data
## 데이터 생성
1. Get the properties of the clock modifying block
1. 시계 수정 블록의 속성을 가져옵니다.
```tcl
report_property [get_cells net ios_0/mmcm_0]
```
2. Trace the input clock to ports using Clock Tracing Methodology
2. 클럭 추적 방법론을 사용하여 포트에 대한 입력 클럭 추적
3. Get the details of the input clock such as period, waveform etc.
3. 주기, 파형 등과 같은 입력 클록의 세부 정보를 가져옵니다.

## Flow
## 흐름

1. Search the constraints `./tmp/constraints.xdc` for `create_clock` or `create_generated_clock` constraint that creates <CLOCK_GROUP>.
1. <CLOCK_GROUP>을 생성하는 `create_clock` 또는 `create_generated_clock` 제약 조건에 대한 제약 조건 `./tmp/constraints.xdc`를 검색하세요.
2. Understand the message.
2. 메시지를 이해하세요.
2. Get the input clock requirement and waveform, calculate the expected requirement and waveform.
2. 입력 클록 요구 사항과 파형을 얻고 예상 요구 사항과 파형을 계산합니다.
3. Compare the two.
3. 둘을 비교해보세요.
4. Capture the property values from get_clocks <CLOCK_GROUP> into a json file and make sure the LINE_NUMBER and FILE_NAME matches with the item 1 data.
4. get_clocks <CLOCK_GROUP>의 속성 값을 json 파일로 캡처하고 LINE_NUMBER 및 FILE_NAME이 항목 1 데이터와 일치하는지 확인하세요.
```tcl
set clk_info [get_clocks -of [get_pins ios_0/mmcm_0/CLKOUT2]]
```
5. Determine the best solution to offer the user.
5. 사용자에게 제공할 최상의 솔루션을 결정합니다.
**IMPORTANT**: All `when` conditions must be true.
**중요**: 모든 `when` 조건이 true여야 합니다.
AND together all conditions to trigger the case.
그리고 케이스를 트리거하는 모든 조건을 함께 사용합니다.

**CASE 1:**
**사례 1:**
* **Type**: Automated Resolution
* **유형**: 자동 해결
* **Condfidence**: High
* **신뢰도**: 높음
* **Usefulness**: High
* **유용성**: 높음
* **When**:
* **언제**:
* [ ] The actual period and waveform match with the expected period and waveform.
* [ ] 실제 주기 및 파형이 예상 주기 및 파형과 일치합니다.
* [ ] There are exceptions that reference the <CLOCK_GROUP> or generated clocks derived from it.
* [ ] <CLOCK_GROUP> 또는 여기에서 파생된 생성된 클록을 참조하는 예외가 있습니다.
* **Action**:
* **작업**:
1. Create a constraint to use the method described in UG903 `renaming auto derived clocks` using the `create_generated_clock` constraint after the definition of the clock on the input
1. 입력에 시계를 정의한 후 `create_generated_clock` 제약 조건을 사용하여 UG903 `자동 파생 시계 이름 바꾸기`에 설명된 방법을 사용하기 위한 제약 조건을 만듭니다.
2. Remove the clock definition of on <NETLIST_ELEMENT>.
2. <NETLIST_ELEMENT>의 시계 정의를 제거합니다.
3. Report that exceptions reference the clock name
3. 예외가 시계 이름을 참조한다고 보고합니다.
4. State that no assumptions have been made
4. 어떤 가정도 하지 않았음을 명시

**CASE 2:**
**사례 2:**
* **Type**: Automated Resolution
* **유형**: 자동 해결
* **Condfidence**: High
* **신뢰도**: 높음
* **Usefulness**: High
* **유용성**: 높음
* **When**:
* **언제**:
* [ ] The actual period and waveform match with the expected period and waveform.
* [ ] 실제 주기 및 파형이 예상 주기 및 파형과 일치합니다.
* [ ] There are no exceptions that reference the <CLOCK_GROUP> or generated clocks derived from it.
* [ ] <CLOCK_GROUP> 또는 여기에서 파생된 생성된 클록을 참조하는 예외는 없습니다.
* **Action**:
* **작업**:
1. Remove the definition of any clocks on <NETLIST_ELEMENT>.
1. <NETLIST_ELEMENT>에서 모든 시계 정의를 제거합니다.
2. Report that no exceptions reference the clock name.
2. 시계 이름을 참조하는 예외가 없음을 보고합니다.
3. State that no assumptions have been made
3. 어떤 가정도 하지 않았음을 명시

**CASE 3:**
**사례 3:**
* **Type**: Automated Resolution
* **유형**: 자동 해결
* **Condfidence**: High
* **신뢰도**: 높음
* **Usefulness**: High
* **유용성**: 높음
* **When**:
* **언제**:
* [ ] The actual period and waveform do not match with the expected period and waveform.
* [ ] 실제 주기 및 파형이 예상 주기 및 파형과 일치하지 않습니다.
* [ ] There are no CLOCK type nets connected to DCLK pin of the object
* [ ] 객체의 DCLK 핀에 연결된 CLOCK 유형 네트가 없습니다.
* **Assumption**: Input clock to CMB is the desired frequency and the modifying parameters are worst case.
* **가정**: CMB에 대한 입력 클럭은 원하는 주파수이고 수정 매개변수는 최악의 경우입니다.
* The clock definition on the pin should be removed.
* 핀의 클럭 정의를 제거해야 합니다.
* **Action**:
* **작업**:
1. Remove the clock definition on <NETLIST_ELEMENT>
1. <NETLIST_ELEMENT>에서 시계 정의를 제거합니다.
2. Report the assumption.
2. 가정을 보고합니다.
2. State "If this assumption is incorrect, then update the M/D values on the MMCM and remove the constraint"
2. "이 가정이 잘못된 경우 MMCM의 M/D 값을 업데이트하고 제약 조건을 제거하십시오"라고 상태를 지정합니다.

**Example to find if DCLK has connected pins**
**DCLK에 핀이 연결되었는지 확인하는 예**
```tcl
set nets [get_nets -filter {TYPE==CLOCK} -of [get_pins -filter {REF_PIN_NAME==DCLK} -of [get_cells -of [get_pins output ios_0/mmcm_0/CLKOUT2]]]]
if {[llength $net] == 0} {puts "Assume input clock is the intent"} else {puts "Reconfiguration of MMCM is possible"}
```
**CASE 4:**
**사례 4:**
* **Type**: Automated Resolution
* **유형**: 자동 해결
* **Condfidence**: Medium
* **신뢰도**: 중간
* **Usefulness**: High
* **유용성**: 높음
* **When**:
* **언제**:
* [ ] The actual period and waveform do not match with the expected period and waveform.
* [ ] 실제 주기 및 파형이 예상 주기 및 파형과 일치하지 않습니다.
* [ ] There are CLOCK type nets connected to DCLK pin of the object
* [ ] 객체의 DCLK 핀에 연결된 CLOCK 유형의 네트가 있습니다.
* **Assumption**: Output clock to CMB is the desired frequency and the modifying parameters are not set to the worst case.
* **가정**: CMB에 대한 출력 클럭은 원하는 주파수이고 수정 매개변수는 최악의 경우로 설정되지 않습니다.
* The pin definition is correct.
* 핀 정의가 정확합니다.
* **Action**:
* **작업**:
1. Recommend updated clock modifying parameters so that the output clock is automatically generated.
1. 출력 클록이 자동으로 생성되도록 업데이트된 클록 수정 매개변수를 권장합니다.
2. Remove the clock constraint on <NETLIST_ELEMENT>
2. <NETLIST_ELEMENT>에서 시계 제약 조건을 제거합니다.
3. Rename the generated clock to match the name of the output clock removed
3. 제거된 출력 클럭의 이름과 일치하도록 생성된 클럭의 이름을 바꿉니다.
4. Report the assumption.
4. 가정을 보고합니다.
4. State "If this assumption is incorrect, then please update the configuration or constraints or waive the issue"
4. "이 가정이 잘못된 경우 구성이나 제약 조건을 업데이트하거나 문제를 포기하십시오."라고 명시합니다.

**Example to find if DCLK has connected pins**
**DCLK에 핀이 연결되었는지 확인하는 예**
```tcl
set nets [get_nets -filter {TYPE==CLOCK} -of [get_pins -filter {REF_PIN_NAME==DCLK} -of [get_cells -of [get_pins output ios_0/mmcm_0/CLKOUT2]]]]
if {[llength $net] == 0} {puts "Assume input clock is the intent"} else {puts "Reconfiguration of MMCM is possible"}
```

**CASE 5:**
**사례 5:**
* **Type**: User Resolution
* **유형**: 사용자 해상도
* **Condfidence**: Medium
* **신뢰도**: 중간
* **Usefulness**: High
* **유용성**: 높음
* **When**:
* **언제**:
* [ ] Expected and actual periods match but waveforms do not.
* [ ] 예상 주기와 실제 주기는 일치하지만 파형은 일치하지 않습니다.
* [ ] The waveform first edge on the actual clock is 0
* [ ] 실제 클럭의 파형 첫 번째 에지는 0입니다.
* **Assumption**: User has failed to adjust the CLKOUTn_DUTY_CYLE setting on the MMCM cell.
* **가정**: 사용자가 MMCM 셀의 CLKOUTn_DUTY_CYLE 설정을 조정하는 데 실패했습니다.
* Assumes the constraint on the output of the MMCM is correct
* MMCM의 출력에 대한 제약 조건이 정확하다고 가정합니다.
* **Action**:
* **작업**:
Propose the following:
다음을 제안합니다:
1. Remove the clock constraint on <NETLIST_ELEMENT>
1. <NETLIST_ELEMENT>에서 시계 제약 조건을 제거합니다.
2. Update the CLKOUTn_DUTY_CYCLE
2. CLKOUTn_DUTY_CYCLE 업데이트
3. Report the assumption.
3. 가정을 보고합니다.
3. State "If this assumption is incorrect, then please resolve the issue manually"
3. "이 가정이 잘못된 경우 문제를 수동으로 해결하십시오"라고 명시합니다.

* **Example**:
* **예**:
```tcl
set hi [string range [expr (${second_edge}-${first_edge})/${period}] 0 4]
set_property CLKOUT0_DUTY_CYCLE $hi [get_cells ios_0/mmcm_0]
```

**CASE 6:**
**사례 6:**
* **Type**: User Resolution
* **유형**: 사용자 해상도
* **Condfidence**: Medium
* **신뢰도**: 중간
* **Usefulness**: High
* **유용성**: 높음
* **When**:
* **언제**:
* [ ] Expected and actual periods match but waveforms do not.
* [ ] 예상 주기와 실제 주기는 일치하지만 파형은 일치하지 않습니다.
* [ ] The waveform first edge on the actual clock is not at 0
* [ ] 실제 클럭의 파형 첫 번째 에지가 0이 아닙니다.
* **Assumption**: User has failed to adjust the CLKOUTn_DUTY_CYLE and CLKOUTn_PHASE settings on the MMCM cell.
* **가정**: 사용자가 MMCM 셀의 CLKOUTn_DUTY_CYLE 및 CLKOUTn_PHASE 설정을 조정하는 데 실패했습니다.
* Assumes the constraint on the output of the MMCM is correct.
* MMCM의 출력에 대한 제약 조건이 정확하다고 가정합니다.
* **Action**:
* **작업**:
1. Remove the clock constraint on <NETLIST_ELEMENT>
1. <NETLIST_ELEMENT>에서 시계 제약 조건을 제거합니다.
2. Update the CLKOUTn_DUTY_CYCLE.
2. CLKOUTn_DUTY_CYCLE을 업데이트합니다.
2. Make sure the constraint matches the pin.
2. 구속조건이 핀과 일치하는지 확인하세요.
2. CLKOUT0_DUTY_CYCLE for pin CLKOUT0
2. 핀 CLKOUT0에 대한 CLKOUT0_DUTY_CYCLE
2. Update the CLKOUTn_PHASE.
2. CLKOUTn_PHASE를 업데이트합니다.
2. Make sure the constraint matches the pin.
2. 구속조건이 핀과 일치하는지 확인하세요.
2. CLKOUT0_PHASE for pin CLKOUT0
2. 핀 CLKOUT0에 대한 CLKOUT0_PHASE
3. Report the assumption.
3. 가정을 보고합니다.
3. State "If this assumption is incorrect, then please resolve the issue manually"
3. "이 가정이 잘못된 경우 문제를 수동으로 해결하십시오"라고 명시합니다.

* **Example**:
* **예**:
```tcl
set hi [string range [expr (${second_edge}-${first_edge})/ ${period}] 0 4]
set_property CLKOUT0_DUTY_CYCLE $hi [get_cells ios_0/mmcm_0]
set_property CLKOUT0_PHASE <value> [get_cells ios_0/mmcm_0]; # value between -360 and 360
```

**CASE 7:**
**사례 7:**
* **Type**: User Resolution
* **유형**: 사용자 해상도
* **Condfidence**: High
* **신뢰도**: 높음
* **Usefulness**: Low
* **유용성**: 낮음
* **Prompt**:
* **프롬프트**:
```
Any other scenario, prompt the user do you want to: 
a) Remove the constraint that creates <CLOCK_GROUP>
b) Use a create_generated_clock constraint based off the expected constraints to account for insertion delay
c) Leave the constraint as is and create a waiver

**DO**
✅  Use the [CLOCK_TRACING.md](./CLOCK_TRACING.md) to trace the clocks
✅  First try tracing back to the CMB input pin and look for clocks

**DO NOT**
❌ Offer automated suggestiones when the the constraint file value is null
❌ Offer automated suggestiones when the the line number value is null
❌ Offer automated suggestiones when the multiply and divide calculations do not match the result
❌ Automate fixes when the the information does not match the item 1 info.
❌ Trace through GT* blocks

## Verification

* [ ] Methodology violation is no longer present
* [ ] No new unresolved methodology warnings are created


## Final Report
Print a table using [./REPORTING_CLOCK_DIFFERENCES](./REPORTING_CLOCK_DIFFERENCES.md)
Print and constraint changes before and after


## Examples

**Details**
* SYS_CLK is defined on pin get_pins ios_0/mmcm_0/CLKOUT1
* There are references to SYS_CLK in the constraints file
* CLKFBOUT_MULT =30
* DIVCLK_DIVIDE =1
* CLKOUT0_DIVIDE =30


**Before**
```XDC
create_clock -period 8.000 -name SYS_CLK [get_pins ios_0/mmcm_0/CLKOUT1]
create_clock -기간 8.000 -name SYS_CLK [get_pins ios_0/mmcm_0/CLKOUT1]
set_clock_groups -asynchronous -group [get_clocks SYS_CLK] -group [get_clocks {HOSTCLK}]
set_clock_groups -asynchronous -group [get_clocks SYS_CLK] -group [get_clocks {HOSTCLK}]

```
**After**
```XDC 
# Fixed TIMING-3#2: Changed from create_clock to create_generated_clock
# TIMING-3#2 수정: create_clock에서 create_generated_clock으로 변경됨
# Preserves clock name SYS_CLK while respecting MMCM output
# MMCM 출력을 존중하면서 시계 이름 SYS_CLK를 유지합니다.
create_generated_clock -name SYS_CLK -source [get_pins ios_0/mmcm_0/CLKIN1] -multiply_by 1 -divide_by 1 [get_pins ios_0/mmcm_0/CLKOUT1]
create_generated_clock -name SYS_CLK -source [get_pins ios_0/mmcm_0/CLKIN1] -multiply_by 1 -divide_by 1 [get_pins ios_0/mmcm_0/CLKOUT1]
```

## References

- UltraFast Design Methodology Guide (UG949)
- Vivado Design Suite User Guide: Design Analysis and Closure Techniques (UG906)
- Vivado Design Suite User Guide: Using Constraints (UG903)
- [Clock Tracing Methodology](./CLOCK_TRACING.md)
