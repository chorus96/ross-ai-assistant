<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# TIMING-1: Invalid clock waveform on Clock Modifying Block
# TIMING-1: 클럭 수정 블록의 잘못된 클럭 파형

## Metadata
## 메타데이터

- **Check ID**: TIMING-1
- **ID 확인**: 타이밍-1
- **Severity**: Critical Warning
- **심각도**: 심각한 경고
- **Hierarchy Name**: Timing.Bad Practice
- **계층 구조 이름**: Timing.Bad Practice
- **First Release**: 2013.3
- **첫 번째 릴리스**: 2013.3

## Description
## 설명

Invalid clock waveform on Clock Modifying Block.
클럭 수정 블록의 클럭 파형이 잘못되었습니다.

### Full Message Description property.
### 전체 메시지 설명 속성입니다.

```
Invalid clock waveform for clock <CLOCK_GROUP> specified at a <MESSAGE_STRING> output <NETLIST_ELEMENT> that does not match the CMB settings. The waveform of the clock is <MESSAGE_STRING>. The expected waveform is <MESSAGE_STRING>
```
**Example:**
**예:**
```
Invalid clock waveform for clock GTX_CLK specified at a MMCME5 output ios_0/mmcm_0/CLKOUT2 that does not match the CMB settings. The waveform of the clock is 5 {0 2.5}. The expected waveform is 12.8 {0 6.4}
```
For the first clock
첫 번째 시계의 경우
5 {0 2.5}
* Period = 5
* 기간 = 5
* First edge = 0
* 첫 번째 모서리 = 0
* Second edge = 2.5
* 두 번째 가장자리 = 2.5

For the second clock
두 번째 시계의 경우
12.8 {0 6.4}
* Period = 12.8
* 기간 = 12.8
* First edge = 0
* 첫 번째 모서리 = 0
* Second edge = 6.4
* 두 번째 가장자리 = 6.4

### Explanation
### 설명
The first clock waveform is a result of a new `create_clock` definition applied to the pin object `ios_0/mmcm_0/CLKOUT2`
첫 번째 클록 파형은 핀 개체 `ios_0/mmcm_0/CLKOUT2`에 적용된 새로운 `create_clock` 정의의 결과입니다.

The expected waveform is based off one of the clocks on the input of the clock modifying block (CMB).
예상되는 파형은 CMB(클럭 수정 블록) 입력의 클록 중 하나를 기반으로 합니다.
There could be:
다음이 있을 수 있습니다.
* Multiple clock input pins with different clock objects on them
* 서로 다른 시계 개체가 있는 여러 시계 입력 핀
* Multiple clock objects on any given pin
* 특정 핀에 있는 여러 시계 객체
This is then updated by the settings on the clock modifying block.
그런 다음 이는 시계 수정 블록의 설정에 의해 업데이트됩니다.

**DO**
✅  Correctly identify the `create_clock` constraint triggering the issue.
✅ 문제를 유발하는 'create_clock' 제약 조건을 올바르게 식별하세요.
✅  Correctly idenfify the `create_clock` constraint generating the input constraint that is the expected expected values
✅ 예상 예상 값인 입력 제약 조건을 생성하는 `create_clock` 제약 조건을 올바르게 식별합니다.
✅  Perform the clock modificaiton calculation and match it with the constraint
✅ 시계 수정 계산을 수행하고 이를 제약 조건과 일치시킵니다.

**DO NOT**
**하지 마세요**
❌ Look at the first `create_clock` and assume it is problem `create_clock` related to the methodology warning.
❌ 첫 번째 `create_clock`을 보고 방법론 경고와 관련된 문제 `create_clock`이라고 가정합니다.


## Flow
## 흐름
1. Search the `./tmp/constraints.xdc` for `create_clock` constraint that creates <CLOCK_GROUP>.
1. <CLOCK_GROUP>을 생성하는 `create_clock` 제약 조건에 대해 `./tmp/constraints.xdc`를 검색하세요.
2. Capture the property values from get_clocks <CLOCK_GROUP> into a json file and make sure the LINE_NUMBER and FILE_NAME matches with the item 1 data.
2. get_clocks <CLOCK_GROUP>의 속성 값을 json 파일로 캡처하고 LINE_NUMBER 및 FILE_NAME이 항목 1 데이터와 일치하는지 확인하세요.
```tcl
set clk_info [get_clocks -of [get_pins ios_0/mmcm_0/CLKOUT2]]
```

**DO NOT**
**하지 마세요**
❌ Automate fixes when the the constraint file value is null
❌ 제약 조건 파일 값이 null인 경우 자동 수정
❌ Automate fixes when the the line number value is null
❌ 줄 번호 값이 null인 경우 수정을 자동화합니다.
❌ Automate fixes when the the information does not match the item 1 info.
❌ 정보가 항목 1 정보와 일치하지 않는 경우 수정을 자동화합니다.

3. Find the `create_clock` source constraint line that relates to the clock mentioned "The expected waveform..".
3. "예상 파형.."이라고 언급된 클럭과 관련된 `create_clock` 소스 제약 라인을 찾습니다.

**DO**
✅  Use the `./CLOCK_TRACING.md` and `trace_clock_treee.tcl` to trace the clocks
✅ 시계를 추적하려면 `./CLOCK_TRACING.md` 및 `trace_clock_treee.tcl`을 사용하세요.
✅  First try tracing back to the CMB input pin and look for clocks
✅ 먼저 CMB 입력 핀을 추적해 시계를 찾아보세요.


**DO NOT**
**하지 마세요**
❌ Offer automated suggestiones when the the constraint file value is null
❌ 제약 조건 파일 값이 null인 경우 자동 제안 제공
❌ Offer automated suggestiones when the the line number value is null
❌ 줄 번호 값이 null인 경우 자동 제안 제공
❌ Offer automated suggestiones when the multiply and divide calculations do not match the result
❌ 곱셈과 나눗셈 계산이 결과와 일치하지 않는 경우 자동 제안 제공
❌ Trace through GT* blocks
❌ GT* 블록을 통한 추적

3. Determine the best solution to offer the user.
3. 사용자에게 제공할 최상의 솔루션을 결정합니다.
**CASE 1:**
**사례 1:**
* **Type**: Automated Resolution
* **유형**: 자동 해결
* **Condfidence**: High
* **신뢰도**: 높음
* **Usefulness**: High
* **유용성**: 높음
* **When**: The period and waveform match with the expected period and waveform and there are exceptions that reference the <CLOCK_GROUP> or generated clocks derived from it.
* **시기**: 주기 및 파형이 예상 주기 및 파형과 일치하며 <CLOCK_GROUP> 또는 여기에서 파생된 생성된 클럭을 참조하는 예외가 있습니다.
1. Create a constraint to use the method described in UG903 `renaming auto derived clocks` using the `create_generated_clock` constraint after the definition of the clock on the input
1. 입력에 시계를 정의한 후 `create_generated_clock` 제약 조건을 사용하여 UG903 `자동 파생 시계 이름 바꾸기`에 설명된 방법을 사용하기 위한 제약 조건을 만듭니다.
2. Remove the clock definition of  on <NETLIST_ELEMENT>.
2. <NETLIST_ELEMENT>의 시계 정의를 제거합니다.
3. Report that exceptions reference the clock name
3. 예외가 시계 이름을 참조한다고 보고합니다.
4. State that no assumptions have been made
4. 어떤 가정도 하지 않았음을 명시

**DO**
✅ When there is a matching Period
✅ 일치하는 기간이 있는 경우

**DO NOT**
**하지 마세요**
❌ Offer this resolution when the Period (or periods) is different for both clocks
❌ 두 시계의 기간(또는 기간)이 다른 경우 이 해결 방법을 제공하세요.

**CASE 2:**
**사례 2:**
* **Type**: Automated Resolution
* **유형**: 자동 해결
* **Condfidence**: High
* **신뢰도**: 높음
* **Usefulness**: High
* **유용성**: 높음
* **When**: If the Period and waveform match with the expected Period and waveform and there are no exceptions that reference the <CLOCK_GROUP> or generated clocks derived from it.
* **시기**: 기간 및 파형이 예상 기간 및 파형과 일치하고 <CLOCK_GROUP> 또는 여기에서 파생된 생성된 클럭을 참조하는 예외가 없는 경우.
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
* **When**: If the Period does not match with the expected Period and there are no signal nets connected to DCLK pin of the object.
* **시기**: 주기가 예상 주기와 일치하지 않고 개체의 DCLK 핀에 연결된 신호망이 없는 경우.
* **Assumption**: Input clock to CMB is the desired frequency and the modifying parameters are worst case.
* **가정**: CMB에 대한 입력 클럭은 원하는 주파수이고 수정 매개변수는 최악의 경우입니다.
* The clock definition on the pin should be removed.
* 핀의 클럭 정의를 제거해야 합니다.

```tcl
set nets [get_nets -filter {TYPE==CLOCK} -of [get_pins -filter {REF_PIN_NAME==DCLK} -of [get_cells -of [get_pins output ios_0/mmcm_0/CLKOUT2]]]]
if {[llength $net] == 0} {puts "Assume input clock is the intent"}
```

1. Remove the clock definition on <NETLIST_ELEMENT>
1. <NETLIST_ELEMENT>에서 시계 정의를 제거합니다.
2. Report the assumption.
2. 가정을 보고합니다.
2. State "If this assumption is incorrect, then update the M/D values on the MMCM and remove the constraint"
2. "이 가정이 잘못된 경우 MMCM의 M/D 값을 업데이트하고 제약 조건을 제거하십시오"라고 상태를 지정합니다.

**CASE 4:**
**사례 4:**
* **Type**: Automated Resolution
* **유형**: 자동 해결
* **Condfidence**: Medium
* **신뢰도**: 중간
* **Usefulness**: High
* **유용성**: 높음
* **When**: If the Period does not match with the expected Period and there are signal nets connected to DCLK pin of the object.
* **시기**: 주기가 예상 주기와 일치하지 않고 개체의 DCLK 핀에 연결된 신호망이 있는 경우.
* **Assumption**: Output clock to CMB is the desired frequency and the modifying parameters are not set to the worst case.
* **가정**: CMB에 대한 출력 클럭은 원하는 주파수이고 수정 매개변수는 최악의 경우로 설정되지 않습니다.
* The pin definition is correct.
* 핀 정의가 정확합니다.

```tcl
set nets [get_nets -filter {TYPE==CLOCK} -of [get_pins -filter {REF_PIN_NAME==DCLK} -of [get_cells -of [get_pins output ios_0/mmcm_0/CLKOUT2]]]]
if {[llength $net] != 0} {puts "potential reconfigurable waveform"}
```

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

**CASE 5:**
**사례 5:**
* **Type**: Automated Resolution
* **유형**: 자동 해결
* **Condfidence**: Medium
* **신뢰도**: 중간
* **Usefulness**: High
* **유용성**: 높음
* **When**: If the periods match but waveforms do not and the first edge o the first clock is 0
* **시기**: 기간은 일치하지만 파형은 일치하지 않고 첫 번째 클럭의 첫 번째 에지가 0인 경우
* **Assumption**: User has failed to adjust the CLKOUTn_DUTY_CYLE setting on the MMCM cell.
* **가정**: 사용자가 MMCM 셀의 CLKOUTn_DUTY_CYLE 설정을 조정하는 데 실패했습니다.
* Assumes the constraint on the output of the MMCM is correct
* MMCM의 출력에 대한 제약 조건이 정확하다고 가정합니다.

```tcl
set hi [string range [expr (${second_edge}-${first_edge})/${period}] 0 4]
set_property CLKOUT0_DUTY_CYCLE $hi [get_cells ios_0/mmcm_0]
```

1. Remove the clock constraint on <NETLIST_ELEMENT>
1. <NETLIST_ELEMENT>에서 시계 제약 조건을 제거합니다.
2. Update the CLKOUTn_DUTY_CYCLE
2. CLKOUTn_DUTY_CYCLE 업데이트
3. Report the assumption.
3. 가정을 보고합니다.
3. State "If this assumption is incorrect, then please resolve the issue manually"
3. "이 가정이 잘못된 경우 문제를 수동으로 해결하십시오"라고 명시합니다.

**CASE 6:**
**사례 6:**
* **Type**: Automated Resolution
* **유형**: 자동 해결
* **Condfidence**: Medium
* **신뢰도**: 중간
* **Usefulness**: High
* **유용성**: 높음
* **When**: If the periods match but waveforms do not and the first edge of the first clock is not at 0
* **시기**: 기간은 일치하지만 파형은 일치하지 않고 첫 번째 클럭의 첫 번째 에지가 0이 아닌 경우
* **Assumption**: User has failed to adjust the CLKOUTn_DUTY_CYLE and CLKOUTn_PHASE settings on the MMCM cell.
* **가정**: 사용자가 MMCM 셀의 CLKOUTn_DUTY_CYLE 및 CLKOUTn_PHASE 설정을 조정하는 데 실패했습니다.
* Assumes the constraint on the output of the MMCM is correct.
* MMCM의 출력에 대한 제약 조건이 정확하다고 가정합니다.

```tcl
set hi [string range [expr (${second_edge}-${first_edge})/ ${period}] 0 4]
set_property CLKOUT0_DUTY_CYCLE $hi [get_cells ios_0/mmcm_0]
set_property CLKOUT0_PHASE <value> [get_cells ios_0/mmcm_0]; # value between -360 and 360
```

1. Remove the clock constraint on <NETLIST_ELEMENT>
1. <NETLIST_ELEMENT>에서 시계 제약 조건을 제거합니다.
2. Update the CLKOUTn_DUTY_CYCLE.
2. CLKOUTn_DUTY_CYCLE을 업데이트합니다.
2. Make sure the constraint matches the pin.
2. 구속조건이 핀과 일치하는지 확인하세요.
2. CLKOUT0_DUTY_CYCLE for pin CLKOUT0
2. 핀 CLKOUT0에 대한 CLKOUT0_DUTY_CYCLE
3. Report the assumption.
3. 가정을 보고합니다.
3. State "If this assumption is incorrect, then please resolve the issue manually"
3. "이 가정이 잘못된 경우 문제를 수동으로 해결하십시오"라고 명시합니다.

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
```
* **Explaination**: Offer an explanation <USER> - <DATE> - <REASON>
* **설명**: 설명 제공 <사용자> - <날짜> - <이유>
* **Examples**:
* **예**:
```
poldark 01/26/26 "Clock redefinition required to test pulses on the <CLOCK_GROUP> that could be higher frequency than nominal clock"
```

## Verification
## 검증
* [ ] Methodology violation is no longer present
* [ ] 방법론 위반이 더 이상 존재하지 않습니다.
* [ ] No new unresolved methodology warnings are created
* [ ] 해결되지 않은 새로운 방법론 경고가 생성되지 않습니다.
* [ ] Pin has the expected timing characteristics or is waived
* [ ] 핀이 예상되는 타이밍 특성을 갖거나 면제되었습니다.


## Final Report
## 최종 보고서
Print a table that shows the difference in:
다음의 차이점을 보여주는 표를 인쇄하세요.
* constraint
* 제약
* Period
* 기간
* waveform
* 파형
* clock definition object
* 시계 정의 객체


| Aspect<br>측면 | Constraint<br>제약 | Period<br>기간 | Waveform<br>파형 | Clock Definition Object<br>시계 정의 객체 |
|--------|-----------|-------------|----------|------------------------|
| **Before**<br>**이전** | `create_clock -period 5 -waveform {0 2.5} [get_pins ios_0/mmcm_0/CLKOUT2]`<br>`create_clock -기간 5 -파형 {0 2.5} [get_pins ios_0/mmcm_0/CLKOUT2]` | 5 ns | {0 2.5} | GTX_CLK<br>GTX_CLK |
| **After**<br>**이후** | `create_generated_clock -source [get_pins ios_0/mmcm_0/CLKIN1] -multiply_by 1 -divide_by 1 [get_pins ios_0/mmcm_0/CLKOUT2]`<br>`create_generated_clock -source [get_pins ios_0/mmcm_0/CLKIN1] -multiply_by 1 -divide_by 1 [get_pins ios_0/mmcm_0/CLKOUT2]` | 12.8 ns | {0 6.4} | GTX_CLK_GEN<br>GTX_CLK_GEN |


## References
## 참고자료

- UltraFast Design Methodology Guide (UG949)
- UltraFast 설계 방법론 가이드(UG949)
- Vivado Design Suite User Guide: Design Analysis and Closure Techniques (UG906)
- Vivado Design Suite 사용자 가이드: 설계 분석 및 폐쇄 기술(UG906)
- Clock Tracing Methodology (../references/CLOCK_TRACING.md)
- 클럭 추적 방법론(../references/CLOCK_TRACING.md)
