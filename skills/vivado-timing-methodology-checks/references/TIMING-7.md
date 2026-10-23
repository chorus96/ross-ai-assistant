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
# TIMING-7: No common node between related clocks

## Metadata

- **Check ID**: TIMING-7
- **Severity**: Critical Warning
- **Group**: 1
- **Priority**: 1
- **Hierarchy Name**: Timing.Bad Practice
- **First Release**: 2013.3

## Description

Detects when two clocks that have timing paths between them (are related/timed together) do not share any common node in the clock network, indicating potentially problematic clock relationships or missing constraints.

## Message

```
The clocks <CLOCK_GROUP> and <CLOCK_GROUP> are related (timed together) but they have no common node.
<CLOCK_GROUP> 및 <CLOCK_GROUP> 클록은 관련되어 있지만(함께 시간이 측정됨) 공통 노드가 없습니다.
The design could fail in hardware.
하드웨어적으로 디자인이 실패할 수 있습니다.
To find a timing path between these clocks, run the following command: report_timing -from [get_clocks <CLOCK_GROUP>] -to [get_clocks <CLOCK_GROUP>]
이러한 클럭 사이의 타이밍 경로를 찾으려면 다음 명령을 실행하십시오. Report_timing -from [get_clocks <CLOCK_GROUP>] -to [get_clocks <CLOCK_GROUP>]
```

**Example:**
```
The clocks bftClk and wbClk are related (timed together) but they have no common node.
시계 bftClk와 wbClk는 관련되어 있지만(함께 시간이 측정됨) 공통 노드가 없습니다.
The design could fail in hardware.
하드웨어적으로 디자인이 실패할 수 있습니다.
To find a timing path between these clocks, run the following command: report_timing -from [get_clocks bftClk] -to [get_clocks wbClk]
이러한 클럭 사이의 타이밍 경로를 찾으려면 다음 명령을 실행하십시오. Report_timing -from [get_clocks bftClk] -to [get_clocks wbClk]
```

## Explanation

This violation occurs when:
1. **Data paths exist** between two clock domains (registers or ports)
2. **No common node** exists in the clock network topology
3. **Clock relationship is undefined** - Vivado cannot establish timing correlation

### Difference from TIMING-6

| Check | Focus | Issue |
|-------|-------|-------|
| **TIMING-6** | Clock hierarchy | No common primary/master clock in the clock tree |
| **TIMING-7** | Physical topology | No common physical node or connection point |

**TIMING-7** is typically more severe - it indicates clocks that are completely independent in both logical hierarchy and physical implementation.

### Why This Matters

When clocks have no common node:
- ❌ **No phase relationship** can be inferred
- ❌ **Random phase offset** at power-up
- ❌ **Metastability risk** on all clock domain crossings
- ❌ **Timing analysis may be incorrect** - assumes relationships that don't exist in hardware

### Common Root Causes

1. **Independent Input Clocks**: Multiple clock inputs driving separate domains
2. **Separate PLLs/MMCMs**: Independent clock generation blocks
3. **Missing Clock Constraints**: Asynchronous relationship not declared
4. **External Clock Sources**: Off-chip clocks with no phase correlation
5. **Design Bug**: Unintended clock domain crossing without CDC circuitry

## Generate Data

Extract and analyze the clock relationship:

**From Violation Object:**
```tcl
# Get violation details
# 위반 세부정보 가져오기
set violation [get_methodology_violations TIMING-7#1]
위반 설정 [get_methodology_violations TIMING-7#1]
set description [get_property DESCRIPTION $violation]
세트 설명 [get_property DESCRIPTION $violation]

# Parse clock names from description
# 설명에서 시계 이름을 구문 분석합니다.
# Expected format: "The clocks <clk1> and <clk2> are related..."
# 예상 형식: "시계 <clk1> 및 <clk2>가 관련되어 있습니다..."
```

**Analyze Clock Network:**
```tcl
# Get clock objects
# 시계 객체 가져오기
set clk1 [get_clocks bftClk]
clk1 설정 [get_clocks bftClk]
set clk2 [get_clocks wbClk]
clk2 설정 [get_clocks wbClk]

# Check clock properties
# 시계 속성 확인
foreach clk [list $clk1 $clk2] {
foreach clk [목록 $clk1 $clk2] {
puts "Clock: [get_property NAME $clk]"
"시계: [get_property NAME $clk]"를 넣습니다.
puts "  Period: [get_property PERIOD $clk]"
" 기간: [get_property PERIOD $clk]"를 넣습니다.
puts "  Sources: [get_property SOURCES $clk]"
" 소스: [get_property SOURCES $clk]"를 넣습니다.
puts "  Is Generated: [get_property IS_GENERATED $clk]"
puts " 생성됨: [get_property IS_GENERATED $clk]"
if {[get_property IS_GENERATED $clk]} {
if {[get_property IS_GENERATED $clk]} {
puts "  Master Clock: [get_property MASTER_CLOCK $clk]"
" 마스터 시계: [get_property MASTER_CLOCK $clk]"를 넣습니다.
    }
puts "  Waveform: [get_property WAVEFORM $clk]"
" 파형: [get_property WAVEFORM $clk]"를 넣습니다.
}

# Find root sources
# 루트 소스 찾기
set src1 [get_property SOURCES $clk1]
src1 설정 [get_property SOURCES $clk1]
set src2 [get_property SOURCES $clk2]
src2 설정 [get_property SOURCES $clk2]

# Trace clock networks
# 추적 클럭 네트워크
report_clock_networks -name $clk1
보고_시계_네트워크 - 이름 $clk1
report_clock_networks -name $clk2
Report_clock_networks - 이름 $clk2
```

**Find Timing Paths:**
```tcl
# Report paths between the clock domains
# 클록 도메인 간의 보고 경로
report_timing -from [get_clocks $clk1] -to [get_clocks $clk2] \
보고_타이밍 -from [get_clocks $clk1] -to [get_clocks $clk2] \
-max_paths 20 -nworst 1 -path_type full_clock_expanded \
-max_paths 20 -n최악 1 -path_type full_clock_expanded \
-file ./tmp/timing_cross_clk1_to_clk2.rpt
-파일 ./tmp/timing_cross_clk1_to_clk2.rpt

report_timing -from [get_clocks $clk2] -to [get_clocks $clk1] \
보고_타이밍 -from [get_clocks $clk2] -to [get_clocks $clk1] \
-max_paths 20 -nworst 1 -path_type full_clock_expanded \
-max_paths 20 -n최악 1 -path_type full_clock_expanded \
-file ./tmp/timing_cross_clk2_to_clk1.rpt
-파일 ./tmp/timing_cross_clk2_to_clk1.rpt
```

**Analyze CDC Implementation:**
```tcl
# Look for synchronizer chains
# 싱크로나이저 체인을 찾아보세요
set paths [get_timing_paths -from [get_clocks $clk1] -to [get_clocks $clk2]]
경로 설정 [get_timing_paths -from [get_clocks $clk1] -to [get_clocks $clk2]]

foreach path $paths {
foreach 경로 $paths {
set endpoint [get_property ENDPOINT_PIN $path]
엔드포인트 설정 [get_property ENDPOINT_PIN $path]
# Check for multi-stage synchronizer (2+ FFs)
# 다단계 동기화 장치 확인(2+ FF)
set fanin [all_fanin -flat -levels 2 [get_pins $endpoint]]
팬인 설정 [all_fanin -plat -levels 2 [get_pins $endpoint]]
puts "Path endpoint: $endpoint"
"경로 끝점: $endpoint"를 넣습니다.
puts "Fanin stages: [llength $fanin]"
"Fanin 단계: [llength $fanin]"을 입력합니다.
}
```

**Check Existing Constraints:**
```tcl
# Review current clock group definitions
# 현재 클록 그룹 정의 검토
report_clock_interaction -delay_type min_max -file ./tmp/clock_interaction.rpt
Report_clock_interaction -delay_type min_max -file ./tmp/clock_interaction.rpt

# Look for existing false paths or max delays
# 기존 잘못된 경로 또는 최대 지연을 찾습니다.
report_exceptions -ignored -from [get_clocks $clk1] -to [get_clocks $clk2]
보고_예외 -무시 -from [get_clocks $clk1] -to [get_clocks $clk2]
```

## Flow

1. **Gather Data** → Identify clock sources and analyze network topology
2. **Analyze** → Determine if clocks are properly isolated or missing constraints
3. **Resolve** → Apply appropriate constraints based on the true relationship
4. **Verify** → Confirm clock relationship is properly defined

### Detailed Resolution Steps

#### Step 1: Verify Clock Independence

**Check if clocks are truly independent:**

```tcl
# Compare clock sources
# 클럭 소스 비교
set src1 [get_property SOURCES [get_clocks clk1]]
src1 설정 [get_property 소스 [get_clocks clk1]]
set src2 [get_property SOURCES [get_clocks clk2]]
src2 설정 [get_property 소스 [get_clocks clk2]]

if {$src1 eq $src2} {
만약 {$src1 eq $src2} {
puts "ERROR: Clocks share same source but no common node detected!"
"오류: 클럭이 동일한 소스를 공유하지만 공통 노드가 감지되지 않습니다!"
puts "  This indicates a clock definition problem."
puts " 이는 시계 정의 문제를 나타냅니다."
} else {
} 그렇지 않으면 {
puts "Clocks have independent sources"
"시계에는 독립적인 소스가 있습니다."
puts "  Source 1: $src1"
" 소스 1: $src1"을 넣습니다.
puts "  Source 2: $src2"
" 소스 2: $src2"를 넣습니다.
}
```

#### Step 2: Identify Clock Relationship Type

| Scenario | Clock Sources | Relationship | Action |
|----------|---------------|--------------|--------|
| **Different input ports** | Separate pins | Asynchronous | `set_clock_groups -asynchronous` |
| **Different PLLs** | Independent MMCMs | Asynchronous | `set_clock_groups -asynchronous` |
| **Same PLL, different outputs** | Shared MMCM | Check if synchronous | May need `create_generated_clock` fix |
| **Same source, wrong definition** | Should be related | Fix clock constraints | Re-define clocks properly |

#### Step 3: Apply Resolution

**CASE 1: Truly Asynchronous Clocks (Most Common)**
- **Type**: Automated Resolution
- **When**: Clocks from completely independent sources

```tcl
# Mark as asynchronous
# 비동기로 표시
set_clock_groups -asynchronous \
set_clock_groups -비동기 \
-group [get_clocks clk1] \
-그룹 [get_clocks clk1] \
-group [get_clocks clk2]
-그룹 [get_clocks clk2]
```

**Requirements:**
✅ Must have CDC synchronizers on all crossing paths
✅ Should verify with `report_cdc` 
✅ Document the independence in comments

```tcl
# Example with verification
# 검증 예
set_clock_groups -asynchronous \
set_clock_groups -비동기 \
-group [get_clocks bftClk] \
-그룹 [get_clocks bftClk] \
-group [get_clocks wbClk] \
-그룹 [get_clocks wbClk] \
-comment "Independent external clock sources with no phase relationship"
-comment "위상 관계가 없는 독립적인 외부 클럭 소스"

# Verify CDC implementation
# CDC 구현 확인
report_cdc -file ./tmp/cdc_verification.rpt
Report_cdc -파일 ./tmp/cdc_verification.rpt
```

**CASE 2: Physically Related (Same PLL) - Fix Clock Definitions**
- **Type**: User Resolution
- **When**: TIMING-7 appears but clocks should be related

This typically indicates incorrect clock definitions. The clocks may share a PLL but were defined incorrectly.

```tcl
# BEFORE (causes TIMING-7):
# 이전(TIMING-7 발생):
create_clock -period 10 -name clkA [get_pins PLL/CLKOUT0]
create_clock -기간 10 -name clkA [get_pins PLL/CLKOUT0]
create_clock -period 5 -name clkB [get_pins PLL/CLKOUT1]
create_clock -기간 5 -name clkB [get_pins PLL/CLKOUT1]

# AFTER (proper definition):
# 이후(적절한 정의):
# Define primary clock at the PLL input
# PLL 입력에서 기본 클록 정의
create_clock -period 10 -name clk_primary [get_ports CLK_IN]
create_clock -기간 10 -name clk_primary [get_ports CLK_IN]

# Define generated clocks from outputs
# 출력에서 생성된 클록 정의
create_generated_clock -name clkA \
create_generated_clock -name clkA \
-source [get_pins PLL/CLKIN1] \
-소스 [get_pins PLL/CLKIN1] \
-master_clock clk_primary \
-master_clock clk_primary \
-divide_by 1 \
-divide_by 1 \
[get_pins PLL/CLKOUT0]
[get_pins PLL/CLKOUT0]

create_generated_clock -name clkB \
create_generated_clock -name clkB \
-source [get_pins PLL/CLKIN1] \
-소스 [get_pins PLL/CLKIN1] \
-master_clock clk_primary \
-master_clock clk_primary \
-divide_by 2 \
-divide_by 2 \
[get_pins PLL/CLKOUT1]
[get_pins PLL/CLKOUT1]
```

**DO:**
✅ Define primary clock at the actual source (input port or oscillator)
✅ Use `create_generated_clock` for PLL/MMCM outputs
✅ Specify proper -source and -master_clock relationships

**DO NOT:**
❌ Create primary clocks on PLL outputs (use generated clocks instead)
❌ Ignore relationships between PLL outputs


**CASE 3: Logically Related but Physically Separate**
- **Type**: User Resolution  
- **When**: Clocks are meant to be synchronous but physically isolated

This is rare but can occur with:
- Multiple identical frequency clocks from independent sources
- Clocks that should be rationally related

```tcl
# If periods are related (e.g., 100 MHz and 200 MHz)
# 기간이 관련된 경우(예: 100MHz 및 200MHz)
# but from separate sources, you may need:
# 그러나 별도의 소스에서 다음이 필요할 수 있습니다.

# Option A: If truly synchronous in design intent
# 옵션 A: 설계 의도가 실제로 동기적인 경우
set_clock_groups -physically_exclusive \
set_clock_groups -물리적으로_독점적 \
-group [get_clocks clk_100] \
-그룹 [get_clocks clk_100] \
-group [get_clocks clk_200]
-그룹 [get_clocks clk_200]

# Option B: If asynchronous despite same frequency
# 옵션 B: 동일한 주파수에도 불구하고 비동기인 경우
set_clock_groups -asynchronous \
set_clock_groups -비동기 \
-group [get_clocks clk_100] \
-그룹 [get_clocks clk_100] \
-group [get_clocks clk_200]
-그룹 [get_clocks clk_200]
```

**CASE 4: Specific Paths are Asynchronous**
- **Type**: User Resolution
- **When**: Most paths need timing but specific crossings don't

```tcl
# For quasi-static control signals
# 준정적 제어 신호용
set_false_path -from [get_clocks clk1] -to [get_clocks clk2] \
set_false_path -from [get_clocks clk1] -to [get_clocks clk2] \
-through [get_pins config_reg*/D]
-[get_pins config_reg*/D]를 통해

# For async signals with max delay requirement
# 최대 지연 요구 사항이 있는 비동기 신호의 경우
set_max_delay 5.0 -datapath_only \
set_max_delay 5.0 -datapath_only \
-from [get_clocks clk1] -to [get_clocks clk2] \
-[get_clocks clk1]에서 -to [get_clocks clk2] \
-through [get_pins async_data*/D]
-[get_pins async_data*/D]를 통해
```

**CASE 5: Create Waiver (Rare)**
- **Type**: User Resolution
- **When**: Violation is understood and acceptable

```tcl
create_waiver -type METHODOLOGY -id {TIMING-7} \
create_waiver -type 방법론 -id {TIMING-7} \
-objects [get_methodology_violations TIMING-7#1] \
-개체 [get_methodology_violations TIMING-7#1] \
-user [get_property USER [current_project]] \
-user [get_property 사용자 [현재_프로젝트]] \
-description "Clocks clk1 and clk2 are from independent PCB oscillators.
-description "시계 clk1 및 clk2는 독립적인 PCB 발진기에서 나온 것입니다.
All crossings use proper XPM_CDC primitives with MTBF > 1000 years.
모든 교차점은 MTBF가 1000년보다 큰 적절한 XPM_CDC 프리미티브를 사용합니다.
Clock relationship explicitly defined as asynchronous."
비동기식으로 명시적으로 정의된 시계 관계입니다."
```

### DON'T:
❌ Apply `set_clock_groups -asynchronous` without verifying CDC circuits
❌ Ignore this violation - indicates potential metastability issues
❌ Assume clocks are related when they have no common node
❌ Fix TIMING-7 without also checking for TIMING-6 and TIMING-8


## Verification

* [ ] Methodology violation TIMING-7 is no longer present
* [ ] Related TIMING-6 and TIMING-8 violations also resolved
* [ ] `report_clock_interaction` shows intended clock relationship
* [ ] All clock domain crossings have proper CDC implementation
* [ ] CDC report (`report_cdc`) shows no unknown crossings
* [ ] Timing reports correctly include or exclude cross-domain paths
* [ ] No new unresolved methodology warnings created

**Verification Script:**
```tcl
# Re-run methodology checks
# 방법론 점검 재실행
report_methodology -checks {TIMING-6 TIMING-7 TIMING-8} \
report_methodology -{TIMING-6 TIMING-7 TIMING-8} 확인 \
-file ./tmp/methodology_timing_6_7_8_after_fix.rpt
-파일 ./tmp/methodology_timing_6_7_8_after_fix.rpt

# Verify clock relationships
# 시계 관계 확인
report_clock_interaction -delay_type min_max \
report_clock_interaction -delay_type min_max \
-file ./tmp/clock_interaction_after_fix.rpt
-파일 ./tmp/clock_interaction_after_fix.rpt

# Check CDC implementation
# CDC 구현 확인
report_cdc -file ./tmp/cdc_after_fix.rpt
보고서_cdc -파일 ./tmp/cdc_after_fix.rpt

# Verify no unknown clock crossings
# 알 수 없는 시계 교차점 확인
set unknown_cdc [get_cdc_violations -of_objects [get_cdc_paths -from * -to *]]
알 수 없는_cdc 설정 [get_cdc_violations -of_objects [get_cdc_paths -from * -to *]]
if {[llength $unknown_cdc] > 0} {
if {[llength $unknown_cdc] > 0} {
puts "WARNING: Unknown CDC violations still present!"
"경고: 알 수 없는 CDC 위반이 여전히 존재합니다!"라고 표시됩니다.
} else {
} 그렇지 않으면 {
puts "SUCCESS: All CDC paths properly defined"
"성공: 모든 CDC 경로가 올바르게 정의되었습니다"라고 표시됩니다.
}
```

## Final Report Guidance

Document your resolution comprehensively:

### Clock Topology Analysis

| Clock | Source | Type | Period | Root Node |
|-------|--------|------|--------|-----------|
| `bftClk` | `CLK_BFT` port | Primary | 10.000 ns | External source A |
| `wbClk` | `CLK_WB` port | Primary | 8.000 ns | External source B |

**Finding**: No common node - clocks originate from independent external sources

### Resolution Applied

```tcl
# Added to constraints.xdc
# Constraints.xdc에 추가됨
# Line: 145
# 라인: 145

set_clock_groups -asynchronous \
set_clock_groups -비동기 \
-group [get_clocks bftClk] \
-그룹 [get_clocks bftClk] \
-group [get_clocks wbClk] \
-그룹 [get_clocks wbClk] \
-comment "Independent external oscillators with no phase relationship"
-코멘트 "위상 관계가 없는 독립적인 외부 발진기"

# Justification:
# 근거:
# - bftClk drives the BFT processing logic (125 MHz)
# - bftClk는 BFT 처리 로직(125MHz)을 구동합니다.
# - wbClk drives the Wishbone interface (100 MHz)
# - wbClk는 Wishbone 인터페이스(100MHz)를 구동합니다.
# - No hardware synchronization between clock sources
# - 클럭 소스 간 하드웨어 동기화가 없습니다.
# - All crossings protected by XPM_CDC_ARRAY_SINGLE synchronizers
# - XPM_CDC_ARRAY_SINGLE 동기화 장치로 보호되는 모든 교차점
```

### CDC Verification

**Crossing Paths Found:** 24 signals from bftClk → wbClk, 8 signals from wbClk → bftClk

| Signal Path | CDC Type | Sync Stages | MTBF |
|-------------|----------|-------------|------|
| `bft_valid` → `wb_valid_sync` | XPM_CDC_SINGLE | 2-FF | >1000 years |
| `bft_data[31:0]` → `wb_data_sync[31:0]` | XPM_CDC_ARRAY_SINGLE | 2-FF | >1000 years |
| `wb_ack` → `bft_ack_sync` | XPM_CDC_SINGLE | 2-FF | >1000 years |

**CDC Report Summary:**
- ✅ All 32 crossing signals have proper CDC synchronizers
- ✅ No unsafe clock domain crossings detected
- ✅ MTBF requirements met for all paths

### Verification Results

Before Fix:
- ❌ TIMING-7#1: bftClk ↔ wbClk  
- ❌ TIMING-7#2: wbClk ↔ bftClk
- ⚠️  32 unconstrained clock crossings

After Fix:
- ✅ TIMING-7 violations resolved
- ✅ Clock interaction report shows "Asynchronous" relationship
- ✅ All CDC paths properly synchronized
- ✅ No new methodology warnings

### Design Impact

**Timing Analysis:**
- Clock domains now properly isolated
- No timing paths analyzed between bftClk and wbClk domains
- CDC paths excluded from timing (as intended)

**Hardware Reliability:**
- Metastability risk mitigated by 2-stage synchronizers
- MTBF > 1000 years for all crossings
- Design safe for independent clock sources

## References

- [UG906](https://www.xilinx.com/content/dam/xilinx/support/documents/sw_manuals/xilinx2024_2/ug906-vivado-design-analysis.pdf) - Design Analysis and Closure Techniques (Chapter 2: Timing Constraints, Clock Domain Crossings)
- [UG903](https://www.xilinx.com/content/dam/xilinx/support/documents/sw_manuals/xilinx2024_2/ug903-vivado-using-constraints.pdf) - Using Constraints (Chapter 5: Timing Constraints, `set_clock_groups`)
- [UG949](https://www.xilinx.com/content/dam/xilinx/support/documents/sw_manuals/xilinx2024_2/ug949-vivado-design-methodology.pdf) - UltraFast Design Methodology (CDC Guidelines, Metastability)
- [UG953](https://www.xilinx.com/content/dam/xilinx/support/documents/sw_manuals/xilinx2024_2/ug953-vivado-7series-libraries.pdf) - XPM_CDC Library Guide
- [CLOCK_TRACING.md](./CLOCK_TRACING.md) - Clock Tracing Methodology

````
