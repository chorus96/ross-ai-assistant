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
# TIMING-8: No common period between related clocks

## Metadata

- **Check ID**: TIMING-8
- **Severity**: Critical Warning
- **Group**: 1
- **Priority**: 1
- **Hierarchy Name**: Timing.Bad Practice
- **First Release**: 2013.3

## Description

Detects when two clocks that have timing paths between them (are related/timed together) do not have periods with a common expandable relationship, making it impossible to establish proper timing correlation.

## Message

```
The clocks <CLOCK_GROUP> and <CLOCK_GROUP> are found related (timed together) but have no common (expandable) period
<CLOCK_GROUP> 및 <CLOCK_GROUP> 클록이 관련되어 있지만(함께 시간이 측정됨) 공통(확장 가능한) 주기가 없습니다.
```

**Example:**
```
The clocks clk_100MHz and clk_133MHz are found related (timed together) but have no common (expandable) period
클록 clk_100MHz 및 clk_133MHz는 관련되어 있지만(함께 시간이 측정됨) 공통(확장 가능한) 주기가 없습니다.
```

## Explanation

This violation occurs when:
1. **Data paths exist** between two clock domains
2. **Clock periods are not rationally related** - no integer multiple or simple fractional relationship exists
3. **Timing analysis cannot find common period** - Vivado cannot expand the periods to find a common boundary

### What is a "Common Period"?

Clocks can have timing analysis performed between them if their periods can be expanded to a common boundary:

**Examples with Common Period (✅ No TIMING-8):**
- 10 ns (100 MHz) and 20 ns (50 MHz) → Common period: 20 ns (1x and 2x)
- 10 ns (100 MHz) and 5 ns (200 MHz) → Common period: 10 ns (1x and 0.5x)
- 8 ns (125 MHz) and 16 ns (62.5 MHz) → Common period: 16 ns (0.5x and 1x)

**Examples without Common Period (❌ TIMING-8):**
- 10 ns (100 MHz) and 7.5 ns (133.33 MHz) → No simple common period
- 6.4 ns (156.25 MHz) and 10 ns (100 MHz) → No integer relationship
- Arbitrary frequencies with irrational ratios

### Why This Matters

When clocks have no common period:
- ❌ **Timing analysis is unreliable** - Vivado may make incorrect assumptions
- ❌ **Phase relationship is undefined** - clocks drift relative to each other
- ❌ **Setup/hold analysis may be wrong** - timing may pass in simulation but fail in hardware
- ❌ **Metastability risk** - clock domain crossings are asynchronous

### Relationship to TIMING-6 and TIMING-7

| Check | Detects | Typical Cause | Primary Issue |
|-------|---------|---------------|---------------|
| **TIMING-6** | No common primary clock | Independent clock trees | Clock hierarchy |
| **TIMING-7** | No common node | Physically separate sources | Network topology |
| **TIMING-8** | No common period | Irrational frequency relationship | Timing correlation |

**TIMING-8** often appears alongside TIMING-6 and TIMING-7, indicating completely asynchronous clock domains.

### Common Root Causes

1. **Independent Oscillators**: Different frequency crystals/oscillators
2. **Unrelated PLLs**: Separate clock generators with different multiplication factors
3. **External Interfaces**: Different clock domains from system-level architecture
4. **Missing Constraints**: Asynchronous clocks not declared with `set_clock_groups`
5. **Transceivers**: GT clocks vs fabric clocks with no rational relationship

## Generate Data

Extract clock period information and analyze relationships:

**From Violation Object:**
```tcl
# Get violation details
# 위반 세부정보 가져오기
set violation [get_methodology_violations TIMING-8#1]
위반 설정 [get_methodology_violations TIMING-8#1]
set description [get_property DESCRIPTION $violation]
세트 설명 [get_property DESCRIPTION $violation]

# Parse clock names from description
# 설명에서 시계 이름을 구문 분석합니다.
# Expected format: "The clocks <clk1> and <clk2> are found related..."
# 예상 형식: "시계 <clk1> 및 <clk2>가 관련되어 있습니다..."
```

**Analyze Clock Periods:**
```tcl
# Get clock objects
# 시계 객체 가져오기
set clk1 [get_clocks clk_100MHz]
clk1 설정 [get_clocks clk_100MHz]
set clk2 [get_clocks clk_133MHz]
clk2 설정 [get_clocks clk_133MHz]

# Extract period information
# 기간정보 추출
set period1 [get_property PERIOD $clk1]
기간1 설정 [get_property PERIOD $clk1]
set period2 [get_property PERIOD $clk2]
기간 2 설정 [get_property PERIOD $clk2]

puts "Clock 1: [get_property NAME $clk1]"
"시계 1: [get_property NAME $clk1]"을 넣습니다.
puts "  Period: $period1 ns ([expr 1000.0/$period1] MHz)"
puts " 기간: $기간1 ns ([expr 1000.0/$기간1] MHz)"
puts "  Sources: [get_property SOURCES $clk1]"
" 소스: [get_property SOURCES $clk1]"을 넣습니다.

puts "Clock 2: [get_property NAME $clk2]"
"시계 2: [get_property NAME $clk2]"를 넣습니다.
puts "  Period: $period2 ns ([expr 1000.0/$period2] MHz)"
" 기간: $기간2 ns ([expr 1000.0/$기간2] MHz)"를 넣습니다.
puts "  Sources: [get_property SOURCES $clk2]"
" 소스: [get_property SOURCES $clk2]"를 넣습니다.

# Check for rational relationship
# 합리적인 관계를 확인하세요.
set ratio [expr $period1 / $period2]
비율 설정 [expr $기간1 / $기간2]
puts "Period ratio: $ratio"
"기간 비율: $ratio"를 입력합니다.

# Test for expandable periods (common period)
# 확장 가능 기간 테스트(공통 기간)
set lcm_attempts {2 3 4 5 6 8 10 12 15 16 20}
lcm_attempts {2 3 4 5 6 8 10 12 15 16 20} 설정
set found_common 0
Found_common을 0으로 설정
foreach mult $lcm_attempts {
foreach 다중 $lcm_attempts {
set expanded1 [expr $period1 * $mult]
확장1 설정 [expr $기간1 * $mult]
set expanded2 [expr $period2 * $mult]
확장됨2 설정 [expr $기간2 * $mult]
if {abs($expanded1 - $expanded2) < 0.001} {
if {abs($expanded1 - $expanded2) < 0.001} {
puts "Common period found at ${mult}x: $expanded1 ns"
puts "${mult}x: $expanded1 ns에서 발견된 공통 기간"
set found_common 1
Found_common 1 설정
break
휴식
    }
}

if {!$found_common} {
만약 {!$found_common} {
puts "No common period found - clocks are likely asynchronous"
"공통 기간을 찾을 수 없습니다. 시계가 비동기일 가능성이 높습니다."
}
```

**Find Timing Paths:**
```tcl
# Report crossing paths
# 교차로 신고
report_timing -from [get_clocks $clk1] -to [get_clocks $clk2] \
보고_타이밍 -from [get_clocks $clk1] -to [get_clocks $clk2] \
-max_paths 10 -nworst 1 -file ./tmp/timing_paths_incompatible_clocks.rpt
-max_paths 10 -nworst 1 -file ./tmp/timing_paths_in Compatible_clocks.rpt

# Check for CDC implementation
# CDC 구현 확인
set cdc_paths [get_timing_paths -from [get_clocks $clk1] -to [get_clocks $clk2]]
cdc_paths [get_timing_paths -from [get_clocks $clk1] -to [get_clocks $clk2]] 설정
puts "Number of paths crossing clocks: [llength $cdc_paths]"
"클럭을 통과하는 경로 수: [llength $cdc_paths]"
```

**Check Clock Sources:**
```tcl
# Determine if clocks are from same or different sources
# 클럭이 동일한 소스에서 나온 것인지 아니면 다른 소스에서 나온 것인지 확인
set src1 [get_property SOURCES $clk1]
src1 설정 [get_property SOURCES $clk1]
set src2 [get_property SOURCES $clk2]
src2 설정 [get_property SOURCES $clk2]

if {$src1 eq $src2} {
만약 {$src1 eq $src2} {
puts "WARNING: Clocks share same source but have incompatible periods!"
puts "경고: 시계는 동일한 소스를 공유하지만 기간이 호환되지 않습니다!"
puts "  This may indicate incorrect clock definitions"
puts " 이는 잘못된 시계 정의를 나타낼 수 있습니다."
} else {
} 그렇지 않으면 {
puts "Clocks have independent sources:"
puts "시계에는 독립적인 소스가 있습니다."
puts "  Source 1: $src1"
" 소스 1: $src1"을 넣습니다.
puts "  Source 2: $src2"
" 소스 2: $src2"를 넣습니다.
}

# Check if clocks are generated
# 클럭이 생성되었는지 확인
foreach clk [list $clk1 $clk2] {
foreach clk [목록 $clk1 $clk2] {
set is_gen [get_property IS_GENERATED $clk]
is_gen 설정 [get_property IS_GENERATED $clk]
if {$is_gen} {
{$is_gen}인 경우 {
set master [get_property MASTER_CLOCK $clk]
마스터 설정 [get_property MASTER_CLOCK $clk]
puts "Clock [get_property NAME $clk] is generated from: $master"
puts "시계 [get_property NAME $clk]는 다음에서 생성됩니다: $master"
    }
}
```

**Review Current Constraints:**
```tcl
# Check for existing clock group definitions
# 기존 시계 그룹 정의를 확인하세요.
report_clock_interaction -delay_type min_max -file ./tmp/clock_interaction.rpt
Report_clock_interaction -delay_type min_max -file ./tmp/clock_interaction.rpt

# Look for exceptions
# 예외를 찾아보세요
report_exceptions -from [get_clocks $clk1] -to [get_clocks $clk2] \
Report_Exceptions -from [get_clocks $clk1] -to [get_clocks $clk2] \
-file ./tmp/exceptions_between_clocks.rpt
-파일 ./tmp/Exceptions_between_clocks.rpt
```

## Flow

1. **Gather Data** → Extract clock periods and analyze frequency relationship
2. **Analyze** → Determine if clocks are truly asynchronous or incorrectly defined
3. **Resolve** → Apply appropriate constraints based on clock relationship
4. **Verify** → Confirm timing analysis is correct and CDC is implemented

### Detailed Resolution Steps

#### Step 1: Verify Period Incompatibility

Calculate the period relationship:

```tcl
set clk1 [get_clocks clk_100MHz]
clk1 설정 [get_clocks clk_100MHz]
set clk2 [get_clocks clk_133MHz]
clk2 설정 [get_clocks clk_133MHz]

set period1 [get_property PERIOD $clk1]
기간1 설정 [get_property PERIOD $clk1]
set period2 [get_property PERIOD $clk2]
기간 2 설정 [get_property PERIOD $clk2]

# Check if periods are simple multiples
# 마침표가 단순 배수인지 확인
set ratio [expr $period1 / $period2]
비율 설정 [expr $기간1 / $기간2]
set inverse_ratio [expr $period2 / $period1]
역비 설정 [expr $기간2 / $기간1]

puts "Period ratio: $ratio (could also be $inverse_ratio)"
"주기 비율: $ratio($inverse_ratio일 수도 있음)"를 입력합니다.
puts "Is integer multiple?
puts "정수는 배수인가요?
[expr {round($ratio) == $ratio}]"
[expr {라운드($비율) == $비율}]"

# Calculate frequencies
# 빈도 계산
set freq1 [expr 1000.0 / $period1]
주파수1 설정 [expr 1000.0 / $기간1]
set freq2 [expr 1000.0 / $period2]
주파수2 설정 [expr 1000.0 / $ period2]
puts "Frequency 1: $freq1 MHz"
"주파수 1: $freq1 MHz"를 입력합니다.
puts "Frequency 2: $freq2 MHz"
"주파수 2: $freq2 MHz"를 넣습니다.
```

#### Step 2: Determine True Clock Relationship

Ask the following questions:

**Q1: Are these clocks intentionally asynchronous?**
- Independent input clocks → Yes, **asynchronous**
- Different PLLs with unrelated multipliers → Yes, **asynchronous**
- Same PLL with incompatible dividers → Possibly a **design error**

**Q2: Do the clock definitions match hardware?**
```tcl
# Example: Check if PLL configuration matches constraints
# 예: PLL 구성이 제약 조건과 일치하는지 확인
# If you have a 100 MHz input and MMCM with:
# 100MHz 입력과 MMCM이 다음과 같은 경우:
#   - CLKOUT0: multiply by 10, divide by 10 = 100 MHz
#   - CLKOUT0: 10 곱하기, 10 나누기 = 100MHz
#   - CLKOUT1: multiply by 10, divide by 7.5 = 133.33 MHz
#   - CLKOUT1: 10을 곱하고 7.5로 나누기 = 133.33MHz
# These would have no common period
# 공통 기간이 없습니다.
```

**Q3: Are CDC circuits present?**
- 2+ FF synchronizer chains → Design expects **asynchronous** clocks
- Direct connections → Either **synchronous** (design bug) or missing CDC

#### Step 3: Apply Resolution

**CASE 1: Truly Asynchronous Clocks (Most Common)**
- **Type**: Automated Resolution
- **When**: Clocks have incompatible periods by design

```tcl
# Mark clocks as asynchronous
# 시계를 비동기식으로 표시
set_clock_groups -asynchronous \
set_clock_groups -비동기 \
-group [get_clocks clk_100MHz] \
-그룹 [get_clocks clk_100MHz] \
-group [get_clocks clk_133MHz]
-그룹 [get_clocks clk_133MHz]
```

**This is the correct solution when:**
✅ Clocks are from independent sources
✅ Period relationship is intentionally irrational
✅ CDC synchronizers are present on all crossings
✅ No timing correlation is needed or expected

**Example with verification:**
```tcl
# Define asynchronous relationship
# 비동기 관계 정의
set_clock_groups -asynchronous \
set_clock_groups -비동기 \
-group [get_clocks clk_100MHz] \
-그룹 [get_clocks clk_100MHz] \
-group [get_clocks clk_133MHz] \
-그룹 [get_clocks clk_133MHz] \
-comment "DDR4 controller (100 MHz) and PCIe interface (133 MHz) - independent clock domains"
-주석 "DDR4 컨트롤러(100MHz) 및 PCIe 인터페이스(133MHz) - 독립 클록 도메인"

# Verify CDC implementation
# CDC 구현 확인
report_cdc -from [get_clocks clk_100MHz] -to [get_clocks clk_133MHz] \
report_cdc -from [get_clocks clk_100MHz] -to [get_clocks clk_133MHz] \
-file ./tmp/cdc_100_to_133.rpt
-파일 ./tmp/cdc_100_to_133.rpt
```

**CASE 2: Clocks Should Be Related - Fix Definitions**
- **Type**: User Resolution
- **When**: TIMING-8 indicates incorrect clock constraints

```tcl
# BEFORE (causes TIMING-8):
# 이전(TIMING-8 발생):
create_clock -period 10.000 -name clk1 [get_pins MMCM/CLKOUT0]
create_clock -기간 10.000 -name clk1 [get_pins MMCM/CLKOUT0]
create_clock -period 7.500 -name clk2 [get_pins MMCM/CLKOUT1]  # Wrong period!
create_clock -기간 7.500 -name clk2 [get_pins MMCM/CLKOUT1] # 기간이 잘못되었습니다!

# AFTER (correct relationship):
# 이후(올바른 관계):
# First define primary input clock
# 먼저 기본 입력 클럭을 정의합니다.
create_clock -period 10.000 -name clk_in [get_ports CLK_IN]
create_clock -기간 10.000 -name clk_in [get_ports CLK_IN]

# Define generated clocks with correct multiply/divide
# 올바른 곱셈/나눗셈으로 생성된 클록을 정의하세요.
create_generated_clock -name clk1 \
create_generated_clock -name clk1 \
-source [get_pins MMCM/CLKIN1] \
-소스 [get_pins MMCM/CLKIN1] \
-master_clock clk_in \
-master_clock clk_in \
-multiply_by 1 \
-곱하기_1 \
[get_pins MMCM/CLKOUT0]
[get_pins MMCM/CLKOUT0]

create_generated_clock -name clk2 \
create_generated_clock -name clk2 \
-source [get_pins MMCM/CLKIN1] \
-소스 [get_pins MMCM/CLKIN1] \
-master_clock clk_in \
-master_clock clk_in \
-multiply_by 4 -divide_by 3 \
-곱하기_4로 -나누기_로 3 \
[get_pins MMCM/CLKOUT1]
[get_pins MMCM/CLKOUT1]
```

**DO:**
✅ Use `create_generated_clock` for PLL/MMCM outputs
✅ Match multiply/divide factors to actual hardware configuration
✅ Verify periods match expected hardware behavior

**DO NOT:**
❌ Create primary clocks on PLL outputs with arbitrary periods
❌ Ignore MMCM configuration when defining clocks


**CASE 3: Max Delay Constraints for Specific Paths**
- **Type**: User Resolution
- **When**: Most paths are unconstrained but specific crossings need limits

```tcl
# For quasi-static control signals with max delay requirement
# 최대 지연 요구사항이 있는 준정적 제어 신호용
set_max_delay 10.0 -datapath_only \
set_max_delay 10.0 -datapath_only \
-from [get_clocks clk_100MHz] -to [get_clocks clk_133MHz] \
-[get_clocks clk_100MHz]에서 -to [get_clocks clk_133MHz] \
-through [get_pins control_*/D]
-[get_pins control_*/D]를 통해

# For completely asynchronous signals
# 완전 비동기 신호의 경우
set_false_path \
set_false_path \
-from [get_clocks clk_100MHz] -to [get_clocks clk_133MHz] \
-[get_clocks clk_100MHz]에서 -to [get_clocks clk_133MHz] \
-through [get_pins async_status_*/D]
-[get_pins async_status_*/D]를 통해
```

**CASE 4: Physically Exclusive Clocks**
- **Type**: User Resolution
- **When**: Clocks never exist simultaneously

```tcl
# For mode-multiplexed clocks (e.g., different boot modes)
# 모드 다중화 클록의 경우(예: 다른 부팅 모드)
set_clock_groups -physically_exclusive \
set_clock_groups -물리적으로_독점적 \
-group [get_clocks boot_clk_100] \
-그룹 [get_clocks boot_clk_100] \
-group [get_clocks run_clk_133]
-그룹 [get_clocks run_clk_133]
```

**CASE 5: Create Waiver (Rare)**
- **Type**: User Resolution
- **When**: Violation is understood and acceptable

```tcl
create_waiver -type METHODOLOGY -id {TIMING-8} \
create_waiver -type 방법론 -id {TIMING-8} \
-objects [get_methodology_violations TIMING-8#1] \
-개체 [get_methodology_violations TIMING-8#1] \
-user [get_property USER [current_project]] \
-user [get_property 사용자 [현재_프로젝트]] \
-description "Clocks clk_100MHz (PCIe) and clk_133MHz (DDR) are asynchronous by design.
-description "클럭 clk_100MHz(PCIe) 및 clk_133MHz(DDR)는 설계상 비동기식입니다.
All 15 crossing signals use XPM_CDC_SINGLE or XPM_CDC_ARRAY_SINGLE synchronizers.
15개의 교차 신호는 모두 XPM_CDC_SINGLE 또는 XPM_CDC_ARRAY_SINGLE 동기화 장치를 사용합니다.
Clock groups constraint applied in line 234 of constraints.xdc."
Constraints.xdc의 라인 234에 시계 그룹 제약 조건이 적용되었습니다."
```

### DON'T:
❌ Ignore TIMING-8 without verifying CDC implementation
❌ Apply asynchronous constraints without confirming clock independence
❌ Use `set_clock_groups -asynchronous` to hide synchronous timing failures
❌ Leave clocks without any relationship constraint when paths exist


## Verification

* [ ] Methodology violation TIMING-8 is no longer present
* [ ] Related TIMING-6 and TIMING-7 violations also resolved
* [ ] `report_clock_interaction` shows appropriate clock relationship
* [ ] All clock domain crossings have proper CDC synchronizers
* [ ] CDC report shows no unsafe crossings
* [ ] Timing analysis correctly handles or excludes cross-domain paths
* [ ] No new unresolved methodology warnings created

**Comprehensive Verification:**
```tcl
# Re-run methodology checks for all related violations
# 모든 관련 위반 사항에 대해 방법론 검사를 다시 실행합니다.
report_methodology -checks {TIMING-6 TIMING-7 TIMING-8} \
report_methodology -{TIMING-6 TIMING-7 TIMING-8} 확인 \
-file ./tmp/methodology_clock_relationship_after_fix.rpt
-파일 ./tmp/methodology_clock_relationship_after_fix.rpt

# Verify clock interaction matrix
# 클록 상호 작용 매트릭스 확인
report_clock_interaction -delay_type min_max -significant_digits 4 \
report_clock_interaction -delay_type min_max -significant_digits 4 \
-file ./tmp/clock_interaction_detailed.rpt
-파일 ./tmp/clock_interaction_detailed.rpt

# Check CDC implementation
# CDC 구현 확인
report_cdc -details -file ./tmp/cdc_detailed.rpt
report_cdc -details -file ./tmp/cdc_detailed.rpt

# Verify no timing paths between async clocks (if marked as async)
# 비동기 시계 사이에 타이밍 경로가 없는지 확인하세요(비동기로 표시된 경우).
set paths [get_timing_paths -from [get_clocks clk1] -to [get_clocks clk2]]
경로 설정 [get_timing_paths -from [get_clocks clk1] -to [get_clocks clk2]]
if {[llength $paths] > 0} {
if {[llength $paths] > 0} {
puts "WARNING: Timing paths still exist between clocks marked as asynchronous!"
puts "경고: 비동기로 표시된 클럭 사이에 타이밍 경로가 여전히 존재합니다!"
puts "  This may indicate missing set_clock_groups constraint"
puts " 이는 set_clock_groups 제약 조건이 누락되었음을 나타낼 수 있습니다."
} else {
} 그렇지 않으면 {
puts "SUCCESS: No timing paths between asynchronous clocks"
"성공: 비동기 클록 사이에 타이밍 경로가 없습니다."
}

# Verify CDC synchronizers exist
# CDC 동기화 장치가 있는지 확인하세요.
set cdc_violations [get_cdc_violations -of_objects [get_cdc_paths -from [get_clocks clk1] -to [get_clocks clk2]]]
cdc_violations [get_cdc_violations -of_objects [get_cdc_paths -from [get_clocks clk1] -to [get_clocks clk2]]] 설정
if {[llength $cdc_violations] > 0} {
if {[llength $cdc_violations] > 0} {
puts "ERROR: CDC violations detected!"
"오류: CDC 위반이 감지되었습니다!"라고 표시됩니다.
foreach viol $cdc_violations {
foreach viol $cdc_violations {
puts "  [get_property NAME $viol]: [get_property DESCRIPTION $viol]"
"[get_property NAME $viol]: [get_property 설명 $viol]"을 넣습니다.
    }
} else {
} 그렇지 않으면 {
puts "SUCCESS: All CDC paths properly implemented"
"성공: 모든 CDC 경로가 올바르게 구현되었습니다"라고 표시됩니다.
}
```

## Final Report Guidance

Provide comprehensive documentation of the resolution:

### Clock Period Analysis

| Clock | Period (ns) | Frequency (MHz) | Ratio | Common Period? |
|-------|-------------|-----------------|-------|----------------|
| `clk_100MHz` | 10.000 | 100.000 | 1.000 | No |
| `clk_133MHz` | 7.500 | 133.333 | 0.750 | No |

**Period Relationship:** 10.0 ns / 7.5 ns = 1.333... (irrational ratio)
**Finding:** No common expandable period - clocks are asynchronous

### Clock Source Analysis

| Clock | Source | Type | Origin |
|-------|--------|------|--------|
| `clk_100MHz` | `CLK_100_P` port | Primary | External oscillator (PCIe reference) |
| `clk_133MHz` | `CLK_133_P` port | Primary | External oscillator (DDR4 reference) |

**Conclusion:** Independent clock sources with no phase or frequency relationship

### Resolution Applied

```tcl
# Added to constraints.xdc
# Constraints.xdc에 추가됨
# Line: 187-192
# 회선: 187-192

# Clock group definition for asynchronous clock domains
# 비동기 클록 도메인에 대한 클록 그룹 정의
set_clock_groups -asynchronous \
set_clock_groups -비동기 \
-group [get_clocks clk_100MHz] \
-그룹 [get_clocks clk_100MHz] \
-group [get_clocks clk_133MHz] \
-그룹 [get_clocks clk_133MHz] \
-comment "PCIe (100 MHz) and DDR4 (133 MHz) are independent external clocks"
-comment "PCIe(100MHz)와 DDR4(133MHz)는 독립적인 외부 클럭입니다."

# Justification:
# 근거:
# - clk_100MHz: PCIe reference clock from external oscillator
# - clk_100MHz: 외부 발진기의 PCIe 기준 클록
# - clk_133MHz: DDR4 memory controller clock from separate oscillator
# - clk_133MHz: 별도 발진기의 DDR4 메모리 컨트롤러 클럭
# - No hardware synchronization between sources
# - 소스 간 하드웨어 동기화가 없습니다.
# - Periods have no rational relationship (1.333... ratio)
# - 기간에는 합리적인 관계가 없습니다(1.333... 비율).
# - All 23 crossing signals protected with proper CDC
# - 23개 교차신호 모두 적절한 CDC로 보호
```

### CDC Implementation Verification

**Clock Crossing Summary:**
- Paths from clk_100MHz → clk_133MHz: 15 signals
- Paths from clk_133MHz → clk_100MHz: 8 signals
- Total crossings: 23 signals

**CDC Protection:**

| Signal Group | Count | CDC Type | Stages | Status |
|-------------|-------|----------|--------|--------|
| PCIe status → DDR ctrl | 8 | XPM_CDC_SINGLE | 2-FF | ✅ Safe |
| PCIe data → DDR ctrl | 6 | XPM_CDC_ARRAY_SINGLE | 2-FF | ✅ Safe |
| PCIe valid → DDR ctrl | 1 | XPM_CDC_SINGLE | 3-FF | ✅ Safe |
| DDR status → PCIe | 5 | XPM_CDC_SINGLE | 2-FF | ✅ Safe |
| DDR valid → PCIe | 3 | XPM_CDC_GRAY | Gray + 2-FF | ✅ Safe |

**MTBF Analysis:**
- All synchronizers: MTBF > 1000 years
- Synchronizer depth: 2-3 stages per AMD recommendations
- Gray coding used for multi-bit counters

### Verification Results

**Before Fix:**
- ❌ TIMING-8#1: clk_100MHz and clk_133MHz no common period
- ❌ TIMING-8#2: clk_133MHz and clk_100MHz no common period
- ❌ TIMING-6#1, #2: No common primary clock
- ❌ TIMING-7#1, #2: No common node
- ⚠️  23 unconstrained clock domain crossings

**After Fix:**
- ✅ All TIMING-6, TIMING-7, TIMING-8 violations resolved
- ✅ Clock interaction report shows "Asynchronous" relationship
- ✅ All 23 CDC paths properly synchronized
- ✅ report_cdc shows 0 violations
- ✅ No timing paths analyzed between async domains (as intended)
- ✅ No new methodology warnings

### Design Impact

**Timing Analysis:**
- Removed invalid timing paths between incompatible clock domains
- Timing closure simplified by eliminating impossible constraints
- CDC paths properly excluded from setup/hold analysis

**Hardware Reliability:**
- Eliminated potential metastability on all 23 crossings
- MTBF requirements exceeded for all synchronizers
- Design safe for independent clock sources with phase drift

**Performance:**
- No impact on individual clock domain performance
- CDC synchronizers add 2-3 cycle latency (expected and acceptable)

## References

- [UG906](https://www.xilinx.com/content/dam/xilinx/support/documents/sw_manuals/xilinx2024_2/ug906-vivado-design-analysis.pdf) - Design Analysis and Closure Techniques (Chapter 2: Timing Constraints)
- [UG903](https://www.xilinx.com/content/dam/xilinx/support/documents/sw_manuals/xilinx2024_2/ug903-vivado-using-constraints.pdf) - Using Constraints (Chapter 5: `set_clock_groups`, Clock Domain Crossing Constraints)
- [UG949](https://www.xilinx.com/content/dam/xilinx/support/documents/sw_manuals/xilinx2024_2/ug949-vivado-design-methodology.pdf) - UltraFast Design Methodology (Clock Requirements, CDC Best Practices)
- [UG953](https://www.xilinx.com/content/dam/xilinx/support/documents/sw_manuals/xilinx2024_2/ug953-vivado-7series-libraries.pdf) - 7 Series Libraries Guide (XPM_CDC components)
- [WP272](https://www.xilinx.com/content/dam/xilinx/support/documents/white_papers/wp272.pdf) - Vivado Design Suite White Paper: Clock Domain Crossing
- [CLOCK_TRACING.md](./CLOCK_TRACING.md) - Clock Tracing Methodology

````
