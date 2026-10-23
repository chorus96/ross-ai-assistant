<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

````markdown
# TIMING-6: No common primary clock between related clocks

## Metadata

- **Check ID**: TIMING-6
- **Severity**: Critical Warning
- **Group**: 1
- **Priority**: 1
- **Hierarchy Name**: Timing.Bad Practice
- **First Release**: 2013.3

## Description

Detects when two clocks that have timing paths between them (are related/timed together) do not share a common primary clock source, indicating missing or incorrect clock relationship constraints.

## Message

```
The clocks <CLOCK_GROUP> and <CLOCK_GROUP> are related (timed together) but they have no common primary clock.
<CLOCK_GROUP> 및 <CLOCK_GROUP> 클록은 관련되어 있지만(함께 시간이 측정됨) 공통 기본 클록이 없습니다.
The design could fail in hardware.
하드웨어적으로 디자인이 실패할 수 있습니다.
To find a timing path between these clocks, run the following command: report_timing -from [get_clocks <CLOCK_GROUP>] -to [get_clocks <CLOCK_GROUP>]
이러한 클럭 사이의 타이밍 경로를 찾으려면 다음 명령을 실행하십시오. Report_timing -from [get_clocks <CLOCK_GROUP>] -to [get_clocks <CLOCK_GROUP>]
```

**Example:**
```
The clocks bftClk and wbClk are related (timed together) but they have no common primary clock.
시계 bftClk와 wbClk는 관련되어 있지만(함께 시간이 측정됨) 공통 기본 시계가 없습니다.
The design could fail in hardware.
하드웨어적으로 디자인이 실패할 수 있습니다.
To find a timing path between these clocks, run the following command: report_timing -from [get_clocks bftClk] -to [get_clocks wbClk]
이러한 클럭 사이의 타이밍 경로를 찾으려면 다음 명령을 실행하십시오. Report_timing -from [get_clocks bftClk] -to [get_clocks wbClk]
```

## Explanation

This violation occurs when:
1. **Data paths exist** between registers clocked by different clock domains
2. **No common primary clock** source exists in the clock tree hierarchy
3. **Clock relationship is undefined** - Vivado doesn't know if these clocks are:
   - Synchronous (related timing)
   - Asynchronous (no timing relationship)
   - Rationally related (integer multiple relationship)

Without explicit clock relationships, the design may:
- ❌ Assume incorrect timing relationships
- ❌ Report false passing timing that fails in hardware
- ❌ Miss critical CDC (Clock Domain Crossing) violations
- ❌ Have metastability issues in silicon

### Common Root Causes

1. **Independent Clock Sources**: Clocks driven by separate external oscillators or PLLs
2. **Missing Clock Groups**: Asynchronous clocks not declared with `set_clock_groups`
3. **Incorrect Clock Definitions**: Clock objects created at wrong hierarchy points
4. **Missing CDC Constraints**: Clock domain crossings without proper false_path or max_delay constraints

## Generate Data

Extract the following information from the violation and design:

**From Violation Object:**
```tcl
# Get the two clock names from the violation
# 위반에서 두 개의 시계 이름을 가져옵니다.
set violation [get_methodology_violations TIMING-6#1]
위반 설정 [get_methodology_violations TIMING-6#1]
set description [get_property DESCRIPTION $violation]
세트 설명 [get_property DESCRIPTION $violation]

# Parse clock names from description (extract from message)
# 설명에서 시계 이름 구문 분석(메시지에서 추출)
# Example: "The clocks bftClk and wbClk are related..."
# 예: "bftClk 및 wbClk 시계는 서로 관련되어 있습니다..."
```

**Gather Clock Information:**
```tcl
# For each clock mentioned in the violation
# 위반 사항에 언급된 각 시계에 대해
set clk1 [get_clocks bftClk]
clk1 설정 [get_clocks bftClk]
set clk2 [get_clocks wbClk]
clk2 설정 [get_clocks wbClk]

# Get clock properties
# 시계 속성 가져오기
get_property PERIOD $clk1
get_property 기간 $clk1
get_property SOURCES $clk1
get_property 소스 $clk1
get_property IS_GENERATED $clk1
get_property IS_GENERATED $clk1
get_property MASTER_CLOCK $clk1
get_property MASTER_CLOCK $clk1

get_property PERIOD $clk2
get_property 기간 $clk2
get_property SOURCES $clk2
get_property 소스 $clk2
get_property IS_GENERATED $clk2
get_property IS_GENERATED $clk2
get_property MASTER_CLOCK $clk2
get_property MASTER_CLOCK $clk2

# Find timing paths between the clocks
# 클록 사이의 타이밍 경로 찾기
report_timing -from [get_clocks $clk1] -to [get_clocks $clk2] -max_paths 10
보고_타이밍 -from [get_clocks $clk1] -to [get_clocks $clk2] -max_paths 10
```

**Check Current Clock Groups:**
```tcl
# List all existing clock groups
# 모든 기존 시계 그룹 나열
report_clock_interaction -delay_type min_max -file ./tmp/clock_interaction.rpt
Report_clock_interaction -delay_type min_max -file ./tmp/clock_interaction.rpt
```

## Flow

1. **Gather Data** → Identify the two clocks and analyze their relationship
2. **Analyze** → Determine if clocks are truly asynchronous or should be synchronous
3. **Resolve** → Apply appropriate constraints based on the clock relationship
4. **Verify** → Re-run methodology checks to confirm fix

### Detailed Resolution Steps

#### Step 1: Identify the Clock Relationship

Run the suggested timing report to see the actual data paths:
```tcl
report_timing -from [get_clocks <clk1>] -to [get_clocks <clk2>] -max_paths 10 -file ./tmp/cross_clock_paths.rpt
보고_타이밍 -from [get_clocks <clk1>] -to [get_clocks <clk2>] -max_paths 10 -file ./tmp/cross_clock_paths.rpt
```

Analyze the report:
- **Are there valid data paths?** If yes, these clocks ARE related
- **What is the path type?** (register-to-register, input-to-register, etc.)
- **Are there CDC synchronizers?** (Look for 2+ flip-flop chains)

#### Step 2: Determine Clock Relationship Type

Ask these questions:

**Q1: Are these clocks from independent sources?**
- Different input ports → **Asynchronous**
- Different PLLs/MMCMs → **Asynchronous** (usually)
- Same PLL, different outputs → **Synchronous** (usually)

**Q2: Are the periods related?**
- Exact multiple (e.g., 100MHz and 200MHz) → **Synchronous**
- Irrational relationship (e.g., 100MHz and 133MHz) → **Asynchronous**

**Q3: Does the design have CDC circuitry?**
- 2-FF synchronizers present → **Asynchronous**
- Direct register-to-register → **Synchronous** (or design bug)

#### Step 3: Apply Resolution

**CASE 1: Asynchronous Clocks (Most Common)**
- **Type**: Automated Resolution
- **When**: Clocks are truly independent with no timing relationship

```tcl
# Mark clocks as asynchronous
# 시계를 비동기식으로 표시
set_clock_groups -asynchronous \
set_clock_groups -비동기 \
-group [get_clocks <clk1>] \
-그룹 [get_clocks <clk1>] \
-group [get_clocks <clk2>]
-그룹 [get_clocks <clk2>]
```

**DO:**
✅ Verify CDC synchronizers exist on all crossing paths
✅ Use `set_max_delay -datapath_only` for asynchronous paths if needed
✅ Document why these clocks are asynchronous

**DO NOT:**
❌ Mark clocks as asynchronous without verifying CDC circuits
❌ Use this to hide timing failures between related clocks


**CASE 2: Synchronous Clocks - Fix Clock Definitions**
- **Type**: User Resolution
- **When**: Clocks should share a common primary but don't due to incorrect constraints

```tcl
# Example: Both clocks should derive from same PLL
# 예: 두 클럭 모두 동일한 PLL에서 파생되어야 합니다.
# Instead of separate create_clock commands, use create_generated_clock
# 별도의 create_clock 명령 대신 create_generated_clock을 사용하세요.

# BEFORE (causes TIMING-6):
# 이전(TIMING-6 발생):
create_clock -period 10 -name clk1 [get_pins PLL/CLKOUT0]
create_clock -기간 10 -name clk1 [get_pins PLL/CLKOUT0]
create_clock -period 20 -name clk2 [get_pins PLL/CLKOUT1]
create_clock -기간 20 -name clk2 [get_pins PLL/CLKOUT1]

# AFTER (no TIMING-6):
# 이후(TIMING-6 없음):
create_clock -period 10 -name clk_primary [get_ports CLK_IN]
create_clock -기간 10 -name clk_primary [get_ports CLK_IN]
create_generated_clock -name clk1 -source [get_pins PLL/CLKIN] -master_clock clk_primary [get_pins PLL/CLKOUT0]
create_generated_clock -name clk1 -source [get_pins PLL/CLKIN] -master_clock clk_primary [get_pins PLL/CLKOUT0]
create_generated_clock -name clk2 -source [get_pins PLL/CLKIN] -master_clock clk_primary [get_pins PLL/CLKOUT1]
create_generated_clock -name clk2 -source [get_pins PLL/CLKIN] -master_clock clk_primary [get_pins PLL/CLKOUT1]
```

**CASE 3: False Path Between Specific Crossings**
- **Type**: User Resolution
- **When**: Most paths are valid but specific crossings are asynchronous

```tcl
# Set false path for specific control/status signals
# 특정 제어/상태 신호에 대해 잘못된 경로 설정
set_false_path -from [get_clocks clk1] -to [get_clocks clk2] -through [get_pins control_reg*/D]
set_false_path -from [get_clocks clk1] -to [get_clocks clk2] -through [get_pins control_reg*/D]
```

**CASE 4: Create Waiver**
- **Type**: User Resolution
- **When**: Violation is understood and accepted (rare for TIMING-6)

```tcl
# Provide detailed justification
# 자세한 근거를 제시하세요
create_waiver -type METHODOLOGY -id {TIMING-6} \
create_waiver -type 방법론 -id {TIMING-6} \
-objects [get_methodology_violations TIMING-6#1] \
-개체 [get_methodology_violations TIMING-6#1] \
-user [get_property USER [current_project]] \
-user [get_property 사용자 [현재_프로젝트]] \
-description "Clocks clk1 and clk2 are properly constrained with set_max_delay on individual paths.
-description "클럭 clk1 및 clk2는 개별 경로의 set_max_delay로 적절하게 제한됩니다.
Common primary clock relationship not applicable for this design architecture."
이 디자인 아키텍처에는 공통 기본 클록 관계가 적용되지 않습니다."
```

### DON'T:
❌ Apply fixes without understanding the actual clock relationship
❌ Mark all clocks as asynchronous to make violations disappear
❌ Ignore this violation - it indicates potential hardware failures


## Verification

* [ ] Methodology violation TIMING-6 is no longer present
* [ ] Related TIMING-7 and TIMING-8 violations also resolved
* [ ] `report_clock_interaction` shows proper clock relationships
* [ ] CDC paths have appropriate synchronization circuits
* [ ] Timing analysis properly includes or excludes clock domain crossings as intended
* [ ] No new unresolved methodology warnings created

**Verification Commands:**
```tcl
# Re-run methodology checks
# 방법론 점검 재실행
report_methodology -file ./tmp/methodology_after_fix.rpt
보고서_방법론 -파일 ./tmp/methodology_after_fix.rpt

# Verify clock interaction
# 시계 상호 작용 확인
report_clock_interaction -delay_type min_max -file ./tmp/clock_interaction_after_fix.rpt
Report_clock_interaction -delay_type min_max -file ./tmp/clock_interaction_after_fix.rpt

# Check for CDC violations
# CDC 위반 사항 확인
report_cdc -file ./tmp/cdc_check.rpt
보고서_cdc -파일 ./tmp/cdc_check.rpt
```

## Final Report Guidance

Document the resolution with the following information:

### Resolution Summary Table

| Aspect | Clock 1 | Clock 2 | Relationship | Resolution |
|--------|---------|---------|--------------|------------|
| Clock Name | `bftClk` | `wbClk` | Asynchronous | `set_clock_groups` |
| Period | 10.000 ns | 8.000 ns | No common factor | Independent domains |
| Source | `CLK_BFT` port | `CLK_WB` port | Different inputs | Added async group |
| CDC Present | Yes | Yes | 2-FF synchronizers | Verified |

### Applied Constraints

```tcl
# Added to constraints.xdc
# Constraints.xdc에 추가됨
set_clock_groups -asynchronous \
set_clock_groups -비동기 \
-group [get_clocks bftClk] \
-그룹 [get_clocks bftClk] \
-group [get_clocks wbClk]
-그룹 [get_clocks wbClk]

# Reason: These clocks originate from independent external sources with
# 이유: 이 클럭은 다음과 같은 독립적인 외부 소스에서 발생합니다.
# no phase relationship.
# 위상 관계가 없습니다.
# All clock domain crossings are protected with
# 모든 클록 도메인 교차는 다음으로 보호됩니다.
# proper CDC synchronizers.
# 적절한 CDC 동기화 장치.
```

### Verification Results

- ✅ TIMING-6#1 resolved
- ✅ TIMING-6#2 resolved  
- ✅ Clock interaction report confirms asynchronous relationship
- ✅ All CDC paths have 2+ stage synchronizers
- ✅ No timing violations on cross-domain paths

### Design Impact

- Document any CDC paths found
- List all signals crossing between these clock domains
- Confirm CDC methodology is followed (XPM_CDC, handshaking, etc.)
- Note any max_delay or false_path constraints added

## References

- [UG906](https://www.xilinx.com/content/dam/xilinx/support/documents/sw_manuals/xilinx2024_2/ug906-vivado-design-analysis.pdf) - Design Analysis and Closure Techniques (Chapter 2: Timing Constraints)
- [UG903](https://www.xilinx.com/content/dam/xilinx/support/documents/sw_manuals/xilinx2024_2/ug903-vivado-using-constraints.pdf) - Using Constraints (Chapter 5: Timing Constraints)
- [UG949](https://www.xilinx.com/content/dam/xilinx/support/documents/sw_manuals/xilinx2024_2/ug949-vivado-design-methodology.pdf) - UltraFast Design Methodology (Clock Domain Crossing Guidelines)
- [CLOCK_TRACING.md](./CLOCK_TRACING.md) - Clock Tracing Methodology

````
