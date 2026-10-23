<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# TIMING-2: Invalid primary clock source pin
# TIMING-2: 잘못된 기본 클록 소스 핀

## Metadata
## 메타데이터

- **Check ID**: TIMING-2
- **ID 확인**: 타이밍-2
- **Severity**: Critcal warning
- **심각도**: 심각한 경고
- **Group**: 1
- **그룹**: 1
- **Priority**: 1
- **우선순위**: 1
- **First Release**: 2025.1
- **첫 번째 릴리스**: 2025.1

## Description
## 설명

Invalid primary clock source pin
잘못된 기본 클럭 소스 핀

## Message
## 메시지

```
A primary clock <CLOCK_GROUP> is created on an inappropriate pin <NETLIST_ELEMENT>. It is recommended to create a primary clock only on a proper clock source (input port or primitive output pin with no timing arc)
```

```real example
A primary clock new_clk is created on an inappropriate pin reg1_q_reg[0]/Q. It is recommended to create a primary clock only on a proper clock source (input port or primitive output pin with no timing arc)
```

### Explanation
### 설명

Incorrect object specfied for `create_clock` definition.
`create_clock` 정의에 잘못된 개체가 지정되었습니다.
Clocks should be defined either on a port or an output pin with no timing arcs
클럭은 타이밍 아크 없이 포트 또는 출력 핀에서 정의되어야 합니다.

### Flow
### 흐름
1. Using CLOCK_TRACING, trace from the <NETLIST ELEMENT>> to find the most appropriate object to set the `create_clock` constraint on.
1. CLOCK_TRACING을 사용하여 <NETLIST ELEMENT>>에서 추적하여 `create_clock` 제약 조건을 설정하는 데 가장 적합한 개체를 찾습니다.
2. Gather any information on CMBs that change the clock period or
2. 시계 주기를 변경하는 CMB에 대한 정보를 수집하거나
3. Write out the full constraints file
3. 전체 제약조건 파일 작성
3. Determine the best solution to offer the user.
3. 사용자에게 제공할 최상의 솔루션을 결정합니다.

**CASE 1:**
**사례 1:**
* **Type**: User Resolution
* **유형**: 사용자 해상도
* **Condfidence**: High
* **신뢰도**: 높음
* **Usefulness**: Low
* **유용성**: 낮음
* **When**: If there are no returned objects from CLOCK_TRACING,
* **시기**: CLOCK_TRACING에서 반환된 개체가 없는 경우
* **Action**: Recommend the user either removes the constraint or waives the violation
* **조치**: 사용자에게 제약 조건을 제거하거나 위반을 포기하도록 권장합니다.

**CASE 2:**
**사례 2:**
* **Type**: Auto Resolution
* **유형**: 자동 해상도
* **Condfidence**: High
* **신뢰도**: 높음
* **Usefulness**: Med
* **유용성**: 중간
* **When**:
* **언제**:
* [ ] CLOCK_TRACING returns a ground of power type driver
* [ ] CLOCK_TRACING은 전원 유형 드라이버의 접지를 반환합니다.
* **Action**: Remove the constraint
* **조치**: 제약조건을 제거하세요.

**CASE 3:**
**사례 3:**
* **Type**: Auto Resolution
* **유형**: 자동 해상도
* **Condfidence**: High
* **신뢰도**: 높음
* **Usefulness**: High
* **유용성**: 높음
* **When**:
* **언제**:
* [ ] There are returned objects from CLOCK_TRACING
* [ ] CLOCK_TRACING에서 반환된 개체가 있습니다.
* [ ] No constraint is defined on the CLOCK_TRACING objects
* [ ] CLOCK_TRACING 개체에 제약 조건이 정의되지 않았습니다.
* **Action**:
* **작업**:
* [ ] Propose a create_clock constraint on the CLOCK_TRACING object based off the period values included in current constraint accounting for any CMBs in the path
* [ ] 경로의 모든 CMB를 고려하는 현재 제약 조건에 포함된 기간 값을 기반으로 CLOCK_TRACING 개체에 create_clock 제약 조건을 제안합니다.
* [ ] Remove the existing constraint
* [ ] 기존 제약조건 제거

**CASE 4:**
**사례 4:**
* **Type**: Auto Resolution
* **유형**: 자동 해상도
* **Condfidence**: High
* **신뢰도**: 높음
* **Usefulness**: High
* **유용성**: 높음
* **When**:
* **언제**:
* [ ] There are returned objects from CLOCK_TRACING
* [ ] CLOCK_TRACING에서 반환된 개체가 있습니다.
* [ ] A constraint is defined on the CLOCK_TRACING objects
* [ ] CLOCK_TRACING 개체에 제약 조건이 정의되었습니다.
* [ ] The constraint period and waveform matches the <CLOCK_GROUP> definition accounting for clock modifying blocks.
* [ ] 제약 기간 및 파형은 클럭 수정 블록을 고려하는 <CLOCK_GROUP> 정의와 일치합니다.
* [ ] The clock name is referenced by exceptions elsewhere in the constraints file
* [ ] 시계 이름은 제약 조건 파일의 다른 곳에서 예외에 의해 참조됩니다.
* **Action**:
* **작업**:
* [ ] Create a `create_generated_clock` constraint on the <NETLIST_ELEMENT> using the existing name.
* [ ] 기존 이름을 사용하여 <NETLIST_ELEMENT>에 `create_generated_clock` 제약 조건을 생성합니다.
* [ ] Remove the existing constraint
* [ ] 기존 제약조건 제거

**CASE 5:**
**사례 5:**
* **Type**: Auto Resolution
* **유형**: 자동 해상도
* **Condfidence**: High
* **신뢰도**: 높음
* **Usefulness**: High
* **유용성**: 높음
* **When**:
* **언제**:
* [ ] There are returned objects from CLOCK_TRACING
* [ ] CLOCK_TRACING에서 반환된 개체가 있습니다.
* [ ] A constraint is defined on the CLOCK_TRACING objects
* [ ] CLOCK_TRACING 개체에 제약 조건이 정의되었습니다.
* [ ] The constraint period and waveform matches the <CLOCK_GROUP> definition accounting for clock modifying blocks.
* [ ] 제약 기간 및 파형은 클럭 수정 블록을 고려하는 <CLOCK_GROUP> 정의와 일치합니다.
* [ ] The clock name is not referenced by exceptions elsewhere in the constraints file
* [ ] 제약 조건 파일의 다른 곳에서 예외가 시계 이름을 참조하지 않습니다.
* **Action**:
* **작업**:
* [ ] Remove the existing constraint
* [ ] 기존 제약조건 제거

**CASE 6:**
**사례 6:**
* **Type**: User Resolution
* **유형**: 사용자 해상도
* **Condfidence**: High
* **신뢰도**: 높음
* **Usefulness**: Low
* **유용성**: 낮음
* **When**:
* **언제**:
* [ ] There are returned objects from CLOCK_TRACING
* [ ] CLOCK_TRACING에서 반환된 개체가 있습니다.
* [ ] A constraint is defined on the CLOCK_TRACING objects
* [ ] CLOCK_TRACING 개체에 제약 조건이 정의되었습니다.
* [ ] The constraint period and waveform oes not match the <CLOCK_GROUP> definition accounting for clock modifying blocks.
* [ ] 제약 기간 및 파형이 클럭 수정 블록을 설명하는 <CLOCK_GROUP> 정의와 일치하지 않습니다.
* [ ] The clock name is referenced by exceptions elsewhere in the constraints file
* [ ] 시계 이름은 제약 조건 파일의 다른 곳에서 예외에 의해 참조됩니다.
* **Action**:
* **작업**:
* [ ] Ask the user if they want to remove or waive the constraint.
* [ ] 사용자에게 제약 조건을 제거하거나 포기할지 묻습니다.
* Print the clock differences so they can see what is expected v defined.
* 시계 차이를 인쇄하여 예상되는 내용과 정의된 내용을 확인할 수 있습니다.



**Use**
**사용**
See [./REPORTING_CLOCK_DIFFERENCES](./REPORTING_CLOCK_DIFFERENCES.md)
[./REPORTING_CLOCK_DIFFERENCES](./REPORTING_CLOCK_DIFFERENCES.md)를 참조하세요.


### Examples
### 예

*To be documented with real-world examples.
*실제 사례를 문서화합니다.
As required*
필요에 따라*

## References
## 참고자료

- UltraFast Design Methodology Guide (UG949)
- UltraFast 설계 방법론 가이드(UG949)
- Vivado Design Suite User Guide: Design Analysis and Closure Techniques (UG906)
- Vivado Design Suite 사용자 가이드: 설계 분석 및 폐쇄 기술(UG906)
- Vivado Design Suite User Guide: Using Constraints (UG903)
- Vivado 디자인 스위트 사용자 가이드: 제약 조건 사용(UG903)
- [Clock Tracing Methodology](./CLOCK_TRACING.md)
- [클럭 추적 방법론](./CLOCK_TRACING.md)
