<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# TIMING-18: Missing input or output delay
# TIMING-18: 입력 또는 출력 지연 누락

## Metadata
## 메타데이터

- **Check ID**: TIMING-18
- **ID 확인**: 타이밍-18
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

Missing input or output delay constraint on a port relative to clock edges.
클럭 에지에 관련된 포트의 입력 또는 출력 지연 제약 조건이 누락되었습니다.

## Message
## 메시지

```
An <MESSAGE_STRING> delay is missing on <NETLIST_ELEMENT> relative to the rising and/or falling clock edge(s) of <CLOCK_GROUP>.
```

**Example:**
**예:**
```
An input delay is missing on data_in[7] relative to the rising and/or falling clock edge(s) of sys_clk.
```

## Explanation
## 설명

This check identifies top-level I/O ports that lack proper `set_input_delay` or `set_output_delay` constraints.
이 검사는 적절한 `set_input_delay` 또는 `set_output_delay` 제약 조건이 부족한 최상위 I/O 포트를 식별합니다.
These constraints are essential for:
이러한 제약은 다음과 같은 경우에 필수적입니다.

- Defining the timing relationship between external devices and the FPGA
- 외부 장치와 FPGA 간의 타이밍 관계 정의
- Allowing timing analysis of paths that cross the chip boundary
- 칩 경계를 넘는 경로의 타이밍 분석 허용
- Ensuring proper setup and hold times are met at I/O interfaces
- I/O 인터페이스에서 적절한 설정 및 유지 시간이 충족되도록 보장
- Enabling optimization of I/O path timing during implementation
- 구현 중 I/O 경로 타이밍 최적화 활성화

Without these constraints:
이러한 제약이 없으면:
- I/O paths will not be properly analyzed or optimized
- I/O 경로가 제대로 분석되거나 최적화되지 않습니다.
- Timing violations may go undetected until hardware testing
- 하드웨어 테스트가 완료될 때까지 타이밍 위반이 감지되지 않을 수 있습니다.
- The design may fail to meet system-level timing requirements
- 설계가 시스템 수준 타이밍 요구 사항을 충족하지 못할 수 있습니다.

Common scenarios requiring input/output delays:
입력/출력 지연이 필요한 일반적인 시나리오:
- Synchronous interfaces with external devices
- 외부 장치와의 동기식 인터페이스
- Data buses synchronized to system clocks
- 시스템 클럭에 동기화된 데이터 버스
- Control signals with defined timing relationships
- 정의된 타이밍 관계가 있는 제어 신호
- Any I/O that interacts with clocked logic
- 클록 로직과 상호 작용하는 모든 I/O

## Generate Data
## 데이터 생성

Collect information about the unconstrained port:
제한되지 않은 포트에 대한 정보를 수집합니다.
1. Get port details: `report_property [get_ports <NETLIST_ELEMENT>]`
1. 포트 세부정보 가져오기: `report_property [get_ports <NETLIST_ELEMENT>]`
2. Check existing I/O constraints: `report_timing -from [get_ports <NETLIST_ELEMENT>]` or `report_timing -to [get_ports <NETLIST_ELEMENT>]`
2. 기존 I/O 제약 조건 확인: `report_timing -from [get_ports <NETLIST_ELEMENT>]` 또는 `report_timing -to [get_ports <NETLIST_ELEMENT>]`
3. List all unconstrained ports: `get_ports -filter {DIRECTION == IN && INPUT_DELAY == ""}`
3. 제한되지 않은 모든 포트 나열: `get_ports -filter {DIRECTION == IN && INPUT_DELAY == ""}`
4. List all related clocks: `report_clocks <CLOCK_GROUP>`
4. 모든 관련 시계 나열: `report_clocks <CLOCK_GROUP>`
5. Check clock interactions: `report_clock_interaction`
5. 시계 상호작용 확인: `report_clock_interaction`

## Flow
## 흐름

1. **Gather Data** → Collect port properties and identify related clocks
1. **데이터 수집** → 포트 속성 수집 및 관련 클럭 식별
2. **Analyze** → Determine external device timing requirements
2. **분석** → 외부 장치 타이밍 요구 사항 결정
3. **Resolve** → Add appropriate input/output delay constraints or mark as asynchronous
3. **해결** → 적절한 입력/출력 지연 제약 조건을 추가하거나 비동기식으로 표시
4. **Verify** → Confirm constraints are applied and timing analysis includes I/O paths
4. **확인** → 제약 조건이 적용되고 타이밍 분석에 I/O 경로가 포함되는지 확인

## Verification
## 검증

* [ ] Port has appropriate input_delay or output_delay constraint applied
* [ ] 포트에 적절한 input_delay 또는 output_delay 제약 조건이 적용됨
* [ ] I/O paths appear in timing reports
* [ ] I/O 경로가 타이밍 보고서에 나타납니다.
* [ ] Setup and hold requirements are analyzed
* [ ] 설정 및 보류 요구 사항이 분석됩니다.
* [ ] No new unresolved methodology warnings are created
* [ ] 해결되지 않은 새로운 방법론 경고가 생성되지 않습니다.
* [ ] Asynchronous paths are properly identified with false_path constraints if applicable
* [ ] 비동기 경로는 해당되는 경우 false_path 제약 조건으로 올바르게 식별됩니다.

## Final Report Guidance
## 최종 보고서 지침

Document the resolution:
해결 방법을 문서화합니다.
- If I/O delays were added: Include the specific XDC constraints with values and justification
- I/O 지연이 추가된 경우: 값 및 근거와 함께 특정 XDC 제약 조건을 포함합니다.
- If marked as asynchronous: Document why timing doesn't apply and include false_path constraint
- 비동기로 표시된 경우: 타이밍이 적용되지 않는 이유를 문서화하고 false_path 제약 조건을 포함합니다.
- Reference external device datasheets or interface specifications
- 외부 장치 데이터시트 또는 인터페이스 사양 참조
- List all affected ports and their constraints
- 영향을 받는 모든 포트와 해당 제약 조건을 나열합니다.
- Note any assumptions made about external timing
- 외부 타이밍에 대한 가정을 기록해 두십시오.

## References
## 참고자료

- [UG906](https://www.xilinx.com/content/dam/xilinx/support/documents/sw_manuals/xilinx2024_1/ug906-vivado-design-analysis.pdf) - Design Analysis and Closure Techniques
- [UG906](https://www.xilinx.com/content/dam/xilinx/support/documents/sw_manuals/xilinx2024_1/ug906-vivado-design-analytics.pdf) - 설계 분석 및 폐쇄 기법
- [UG903](https://www.xilinx.com/content/dam/xilinx/support/documents/sw_manuals/xilinx2024_1/ug903-vivado-using-constraints.pdf) - Using Constraints
- [UG903](https://www.xilinx.com/content/dam/xilinx/support/documents/sw_manuals/xilinx2024_1/ug903-vivado-using-constraints.pdf) - 제약 조건 사용
- [UG949](https://www.xilinx.com/content/dam/xilinx/support/documents/sw_manuals/xilinx2024_1/ug949-vivado-design-methodology.pdf) - UltraFast Design Methodology
- [UG949](https://www.xilinx.com/content/dam/xilinx/support/documents/sw_manuals/xilinx2024_1/ug949-vivado-design-methodology.pdf) - UltraFast 설계 방법론
