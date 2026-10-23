<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# TIMING-17: Non-clocked sequential cell
# TIMING-17: 클록되지 않은 순차 셀

## Metadata
## 메타데이터

- **Check ID**: TIMING-17
- **ID 확인**: 타이밍-17
- **Severity**: Critical Warning
- **심각도**: 심각한 경고
- **Group**: TIMING
- **그룹**: 타이밍
- **Hierarchy**: Timing.Bad Practice
- **계층 구조**: Timing.Bad Practice
- **Owner**: gservel
- **소유자**: gservel

## Description
## 설명

Sequential cell clock pin is not reached by a timing clock.
순차 셀 클럭 핀은 타이밍 클럭에 의해 도달되지 않습니다.

## Message
## 메시지

```
The clock pin <NETLIST_ELEMENT> is not reached by a timing clock
```

**Example:**
**예:**
```
The clock pin reg1_q_reg[0]/C is not reached by a timing clock
```

## Explanation
## 설명

This check identifies sequential elements (flip-flops, latches, block RAM, DSP registers, etc.) whose clock pins are not driven by a properly defined timing clock.
이 검사는 클록 핀이 적절하게 정의된 타이밍 클록에 의해 구동되지 않는 순차 요소(플립플롭, 래치, 블록 RAM, DSP 레지스터 등)를 식별합니다.
This is a critical issue because:
이는 다음과 같은 이유로 중요한 문제입니다.

- The sequential element will not be included in timing analysis
- 순차 요소는 타이밍 분석에 포함되지 않습니다.
- Timing paths to/from this element will not be analyzed or optimized
- 이 요소와의 타이밍 경로는 분석되거나 최적화되지 않습니다.
- The implementation tools may make incorrect placement/routing decisions
- 구현 도구가 잘못된 배치/라우팅 결정을 내릴 수 있음
- The element may not function correctly in hardware
- 요소가 하드웨어에서 올바르게 작동하지 않을 수 있습니다.

Common root causes include:
일반적인 근본 원인은 다음과 같습니다.
- Missing `create_clock` or `create_generated_clock` constraints
- 'create_clock' 또는 'create_generated_clock' 제약조건 누락
- Clock gating logic that blocks clock propagation
- 클록 전파를 차단하는 클록 게이팅 로직
- Use of asynchronous resets or enables on clock pins
- 비동기식 재설정 사용 또는 클록 핀 활성화
- Combinatorial logic driving clock pins without proper clock definition
- 적절한 클록 정의 없이 클록 핀을 구동하는 조합 논리
- Clock selection muxes without proper clock definitions
- 적절한 클록 정의가 없는 클록 선택 다중화

## Generate Data
## 데이터 생성

Collect information about the non-clocked sequential cell:
클록되지 않은 순차 셀에 대한 정보를 수집합니다.
1. Get cell details: `report_property [get_cells <NETLIST_ELEMENT>]`
1. 셀 세부정보 가져오기: `report_property [get_cells <NETLIST_ELEMENT>]`
2. Trace clock path: `report_timing -to <NETLIST_ELEMENT>/C`
2. 추적 클럭 경로: `report_timing -to <NETLIST_ELEMENT>/C`
3. Check clock definitions: `report_clocks`
3. 시계 정의 확인: `report_clocks`
4. Find all non-clocked cells: `get_cells -filter {IS_SEQUENTIAL && CLOCK == ""}`
4. 클럭되지 않은 모든 셀 찾기: `get_cells -filter {IS_SEQUENTIAL && CLOCK == ""}`
5. Check driver net: `get_nets -of [get_pins <NETLIST_ELEMENT>/C]`
5. 드라이버 넷 확인: `get_nets -of [get_pins <NETLIST_ELEMENT>/C]`

## Flow
## 흐름

1. **Gather Data** → Collect cell properties and clock path information
1. **데이터 수집** → 셀 속성 및 클럭 경로 정보 수집
2. **Analyze** → Trace clock source and determine why no timing clock exists
2. **분석** → 클럭 소스를 추적하고 타이밍 클럭이 존재하지 않는 이유를 확인합니다.
3. **Resolve** → Add missing clock constraints or fix clock path issues
3. **해결** → 누락된 클럭 제약 조건을 추가하거나 클럭 경로 문제 수정
4. **Verify** → Confirm cell now has valid timing clock
4. **확인** → 이제 셀에 유효한 타이밍 시계가 있는지 확인하세요.

## Verification
## 검증

* [ ] Clock pin is reached by a properly defined timing clock
* [ ] 적절하게 정의된 타이밍 클럭에 의해 클럭 핀에 도달함
* [ ] Sequential cell appears in timing reports
* [ ] 타이밍 보고서에 순차 셀이 나타남
* [ ] No new unresolved methodology warnings are created
* [ ] 해결되지 않은 새로운 방법론 경고가 생성되지 않습니다.
* [ ] Clock network is properly defined throughout the design
* [ ] 클록 네트워크가 설계 전반에 걸쳐 적절하게 정의됩니다.

## Final Report Guidance
## 최종 보고서 지침

Document the resolution:
해결 방법을 문서화합니다.
- Identify the root cause (missing constraint vs. design issue)
- 근본 원인 식별(제약조건 누락 및 설계 문제)
- If constraints were added: Include the specific XDC commands
- 제약 조건이 추가된 경우: 특정 XDC 명령을 포함합니다.
- If design needed changes: Document the clock network modifications
- 설계에 변경이 필요한 경우: 클록 네트워크 수정 사항을 문서화합니다.
- List all affected sequential cells
- 영향을 받는 모든 순차 셀 나열
- Confirm all cells are now properly clocked
- 이제 모든 셀이 올바르게 클럭되었는지 확인하세요.

## References
## 참고자료

- [UG906](https://www.xilinx.com/content/dam/xilinx/support/documents/sw_manuals/xilinx2024_1/ug906-vivado-design-analysis.pdf) - Design Analysis and Closure Techniques
- [UG906](https://www.xilinx.com/content/dam/xilinx/support/documents/sw_manuals/xilinx2024_1/ug906-vivado-design-analytics.pdf) - 설계 분석 및 폐쇄 기법
- [UG903](https://www.xilinx.com/content/dam/xilinx/support/documents/sw_manuals/xilinx2024_1/ug903-vivado-using-constraints.pdf) - Using Constraints
- [UG903](https://www.xilinx.com/content/dam/xilinx/support/documents/sw_manuals/xilinx2024_1/ug903-vivado-using-constraints.pdf) - 제약 조건 사용
- [UG949](https://www.xilinx.com/content/dam/xilinx/support/documents/sw_manuals/xilinx2024_1/ug949-vivado-design-methodology.pdf) - UltraFast Design Methodology
- [UG949](https://www.xilinx.com/content/dam/xilinx/support/documents/sw_manuals/xilinx2024_1/ug949-vivado-design-methodology.pdf) - UltraFast 설계 방법론
