<!-- ko-translation: enabled -->
---
name: vivado-timing-methodology-checks
description: >
  Runs 55+ timing methodology checks (UG906) on synthesized FPGA designs to identify timing violations and generate actionable RTL/XDC fixes. Make sure to use this skill whenever user mentions: "timing methodology", "methodology checks", "methodology report", "report_methodology", "UG906", "fix timing issues", "resolve methodology", "timing violations", "methodology violations", "clock violations", or any specific check IDs (TIMING-1 through TIMING-57). Use when user asks "how do I fix", "resolve", "waive", or "understand" any methodology check, warning, or violation. Also use when debugging timing problems after synthesis/implementation, interpreting methodology violation reports, or when user provides violation IDs like "TIMING-1#1". Use even if user just says "I have methodology violations" or "my design has timing warnings".
license: MIT
compatibility: Requires Vivado 2026.1+
metadata:
  version: "2.0.0"
---

<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->
# Table of Contents
# 목차
- [Prerequisites](#prerequisites)
- [전제조건](#prerequisites)
- [Step-by-Step Instructions](#step-by-step-instructions)
- [단계별 지침](#단계별 지침)
  - [Step 1: Open the Run](#step-1-open-the-run)
  - [1단계: 실행 열기](#step-1-open-the-run)
  - [Step 2: Running report_methodology](#step-2-running-report_methodology)
  - [2단계: report_methodology 실행](#step-2-running-report_methodology)
  - [Step 3: Extract information from the constraints file](#step-3-extract-information-from-the-constraints-file)
  - [3단계: 제약 조건 파일에서 정보 추출](#step-3-extract-information-from-the-constraints-file)
  - [Step 4: Consider Check Rules](#step-4-consider-check-rules)
  - [4단계: 확인 규칙 고려](#step-4-consider-check-rules)
  - [Step 5: Group IDs and prioritize](#step-5-group-ids-and-prioritize)
  - [5단계: 그룹 ID 및 우선순위 지정](#step-5-group-ids-and-prioritize)
  - [Step 6: Generate data](#step-6-generate-data)
  - [6단계: 데이터 생성](#step-6-generate-data)
  - [Step 7: Follow Resolution Flow](#step-7-follow-resolution-flow)
  - [7단계: 해결 흐름 따르기](#step-7-follow-solution-flow)
  - [Step 8: Progress update](#step-8-progress-update)
  - [8단계: 진행상황 업데이트](#step-8-progress-update)
  - [Step 9: Repeat until all groups are resolved](#step-9-repeat-until-all-groups-are-resolved)
  - [9단계: 모든 그룹이 해결될 때까지 반복](#step-9-repeat-until-all-groups-are-resolved)
  - [Step 10: Final report](#step-10-final-report)
  - [10단계: 최종 보고서](#step-10-final-report)
- [Design Edits and Actions](#design-edits-and-actions)
- [디자인 편집 및 작업](#design-edits-and-actions)
  - [Automatic Fixes](#automatic-fixes)
  - [자동 수정](#automatic-fixes)
  - [User Directed Fixes](#user-directed-fixes)
  - [사용자 지정 수정](#user-directed-fixes)
  - [Creating Waivers](#creating-waivers)
  - [면책사항 생성](#creating-waivers)
- [Critical Rules](#critical-rules)
- [중요 규칙](#핵심-규칙)
- [File Locations](#file-locations)
- [파일 위치](#file-locations)
- [Integration with Other Skills](#integration-with-other-skills)
- [다른 스킬과의 통합](#integration-with-other-skills)


# Prerequisites
# 전제조건

1. Ensure your working directory is set up correctly:
1. 작업 디렉터리가 올바르게 설정되었는지 확인하세요.

```bash
# From your design directory (where the .dcp file is located)
# Step 1: Create working directory and symlink to skills
mkdir -p <cwd>/results
```
2. Understand whether the user input.
2. 사용자 입력 여부를 이해합니다.
  * Is it project or non-project flow
  * 프로젝트 흐름인가, 비프로젝트 흐름인가
  * Is there a checkpoint or design run already open or do you need to open it
  * 이미 열려 있는 체크포인트 또는 디자인 실행이 있습니까? 아니면 열어야 합니까?

**Non project directory structure:**
**비프로젝트 디렉토리 구조:**
```
<cwd>/
├── <design>.dcp                              # Your design checkpoint (optional, might be referenced from elsewhere)
|   ├── run.tcl                               # Vivado TCL script lists all the commands the agent executes
|   └── results/                              # Output directory (MUST BE CREATED FIRST)
|       ├── methodology.json                  # Generated violation data
|       ├── methodology_after_fix.json        # Generated violation data after fixes
|       ├── constraints.xdc                   # Backup before fixes
|       ├── constraints_after_fix.xdc         # Constraints after fixes
|       ├── constraints_diff.html             # HTML diff report
|       ├── run.log                           # full transcript of the skill run
|       ├── prompt.log                        # Capture of the user prompts entered
|       └── agent.log                         # output of agent verification used to confirm agent understanding of the issue and resolution
└── METHODOLOGY_RESOLUTION_REPORT.md  # Design report capturing what changes were made, resolutions, and remaining violations
```

**Project directory structure:**
**프로젝트 디렉토리 구조:**
```
<proj-dir>
└── <run-dir>
    ├── results/
    ├── run.tcl
    └── METHODOLOGY_RESOLUTION_REPORT.md
```

**Important:** When running scripts from the skill, understand the skill location and reference them `./scripts/`.
**중요:** 스킬에서 스크립트를 실행할 때 스킬 위치를 이해하고 `./scripts/`를 참조하세요.

# Step-by-Step Instructions
# 단계별 지침

## Step 1: Open the Design/Run
## 1단계: 디자인 열기/실행

### Non Project
### 비프로젝트
```tcl
open_checkpoint <design>.dcp
```

### Project
### 프로젝트
```tcl
open_project <proj_name>.xpr
open_run <run_name>
```

## Step 2: Running report_methodology
## 2단계: report_methodology 실행
* Run report methodology and process json file
* 보고서 방법론 실행 및 json 파일 처리

**Example script:**
**예제 스크립트:**
```tcl
report_methodology -json methodology.json
```

**Required properties to extract from methodology report:**
**방법론 보고서에서 추출하는 데 필요한 속성:**
* `RULE` - The violation rule (e.g., TIMING-1).
* `RULE` - 위반 규칙(예: TIMING-1)
* This is the check ID.
* 수표 ID입니다.
* `NAME` - The violation rule (e.g., TIMING-1#1) This is presented in the order they are reported for each ID
* `NAME` - 위반 규칙(예: TIMING-1#1) 각 ID에 대해 신고된 순서대로 표시됩니다.
* `SEVERITY` - Impact level
* '심각도' - 영향 수준
* `DETAILS` - Detailed description of the violation
* `DETAILS` - 위반에 대한 자세한 설명
* `NETLIST_ELEMENT` - Object causing violation extracted from DETAILS <cell: hier_name/leaf_cell_name> or <pin: hier_name/leaf_cell_name/Q>
* `NETLIST_ELEMENT` - DETAILS <cell: hier_name/leaf_cell_name> 또는 <pin: hier_name/leaf_cell_name/Q>에서 추출된 위반을 유발하는 개체


## Step 3: Extract information from the constraints file
## 3단계: 제약 조건 파일에서 정보 추출
Write out constraints and process
제약 조건 및 프로세스 작성

**Example script:**
**예제 스크립트:**
```tcl
write_xdc -force -exclude_physical ./results/constraints.xdc
```

**Required information to extract from xdc file:**
**xdc 파일에서 추출하는 데 필요한 정보:**
- Constraints are merged and written in a single file.
- 제약 조건이 병합되어 단일 파일에 기록됩니다.
- The report must tell the user the original file.
- 보고서는 사용자에게 원본 파일을 알려주어야 합니다.

**Example:** Capturing file name `pbf_pblocks.xdc` from unified XDC file.
**예:** 통합 XDC 파일에서 파일 이름 `pbf_pblocks.xdc` 캡처.
```xdc
####################################################################################
# Constraints from file : 'pbf_pblocks.xdc'
####################################################################################

create_clock -period 8.000 -name REF_CLK [get_ports REF_CLK_p]
```

Capture:
포착:
* ✅ Original Filename = pbf_pblocks.xdc
* ✅ 원본 파일명 = pbf_pblocks.xdc
* ✅ Line number = 2
* ✅ 줄 번호 = 2
* ✅ constraint type = create_clock
* ✅ 제약 유형 = create_clock
* ✅ clock object name = REF_CLK
* ✅ 시계 객체 이름 = REF_CLK
* ✅ target_object = REF_CLK_p
* ✅ target_object = REF_CLK_p
* ✅ target_object type = port
* ✅ target_object 유형 = 포트
* ✅ period = 8.000
* ✅ 기간 = 8.000


##  Step 4: Consider Check Rules
##  4단계: 점검 규칙 고려
For each of the rule ids found in the methodology report:
방법론 보고서에 있는 각 규칙 ID에 대해 다음을 수행합니다.
1. Understand if a check is supported
1. 수표가 지원되는지 이해하기
2. If it is, capture group, priority and if there is a reference doc
2. 그렇다면 캡처 그룹, 우선 순위 및 참조 문서가 있는 경우
3. General guildelines are always to be used.
3. 일반 지침은 항상 사용되어야 합니다.
4. Additional guidelines are contained in a reference doc for a given ID.
4. 해당 ID에 대한 추가 지침은 참조 문서에 포함되어 있습니다.
4. These can overwrite, or supplement general guidelines.
4. 이는 일반 지침을 덮어쓰거나 보완할 수 있습니다.

List of supported IDs are [./references/SUPPORTED_IDS.md](./references/SUPPORTED_IDS.md)
지원되는 ID 목록은 [./references/SUPPORTED_IDS.md](./references/SUPPORTED_IDS.md)입니다.
General guildelines [./references/GENERAL_GUIDELINES.md](./references/GENERAL_GUIDELINES.md)
일반 지침 [./references/GENERAL_GUIDELINES.md](./references/GENERAL_GUIDELINES.md)
Reference docs are in [./references/<ID>.md](./references/<ID>.md)
참조 문서는 [./references/<ID>.md](./references/<ID>.md)에 있습니다.

# Example: Find support level for TIMING-16 (supported) and TIMING-99 (not supported)
# 예: TIMING-16(지원됨) 및 TIMING-99(지원되지 않음)에 대한 지원 수준 찾기
```sh
./scripts/get_id_info.sh TIMING-16
# Returns
# ID=TIMING-16
# GROUP=1
# PRIORITY=-
# HAS_REFERENCE_FILE=1

../scripts/get_id_info.sh TIMING-99
# Error: ID TIMING-99 not found in SUPPORTED_IDS.md
```

##  Step 5:  Group IDs and prioritize
##  5단계: 그룹 ID 및 우선순위 지정
In SUPPPORTED_IDS.md file there is a "GROUP" and "PRIORITY" value.
SUPPPORTED_IDS.md 파일에는 "GROUP" 및 "PRIORITY" 값이 있습니다.
GROUP defines which order the groups should be resolved.
GROUP은 그룹을 해결해야 하는 순서를 정의합니다.
Group 1 is highest priority and should be solved first.
그룹 1은 우선순위가 가장 높으며 먼저 해결해야 합니다.
PRIORITY defines the order within the group.
PRIORITY는 그룹 내 순서를 정의합니다.
Priority 1 should be solved first before moving to priority 2 within the same group.
같은 그룹 내에서 우선순위 1을 먼저 해결한 후 우선순위 2로 이동해야 합니다.

✅ **Do:**
* Order them at this stage.
* 이 단계에서 주문하세요.

❌ **Do not:**
❌ **하지 마세요:**
* Solve them at this stage.
* 이 단계에서 문제를 해결하세요.
* We will solve them in the order of their groups and priorities in Step 5.
* 5단계에서는 그룹 및 우선순위에 따라 문제를 해결해 보겠습니다.

## Step 6: Generate data
## 6단계: 데이터 생성
Inside each reference is a `Generate Data` section.
각 참조 내부에는 '데이터 생성' 섹션이 있습니다.
Generate the data specified for each check ID.
각 수표 ID에 지정된 데이터를 생성합니다.

✅ **Do:**
* Collect all the data within the group being worked on
* 작업 중인 그룹 내 모든 데이터를 수집합니다.

❌ **Do not:**
❌ **하지 마세요:**
* Collect data for other groups at this stage
* 이 단계에서 다른 그룹을 위한 데이터 수집
* Collect the same data more than once.
* 동일한 데이터를 두 번 이상 수집합니다.

## Step 7: Follow Resolution Flow
## 7단계: 해결 흐름 따르기
* Each check has a defined flow to follow based on the reference documentation.
* 각 검사에는 참조 문서를 기반으로 따라야 할 정의된 흐름이 있습니다.
* In the order defined by GROUP and PRIORITY, follow the flow to resolution
* GROUP과 PRIORITY에 정의된 순서대로 해결의 흐름을 따릅니다.
* Resolutions are either:
* 해결 방법은 다음 중 하나입니다.
- **AUTO**: You can fix it automatically
- **AUTO**: 자동으로 수정할 수 있습니다.
- **USER**: You must get user input; never guess their design intent
- **사용자**: 사용자 입력을 받아야 합니다. 그들의 디자인 의도를 결코 추측하지 마십시오

**Critical**: Prioritize reference files over general guidelines when they exist.
**중요**: 참조 파일이 있는 경우 일반 지침보다 우선순위를 지정하세요.

* Confirm the message understanding by outputting in the agent.log file:
* Agent.log 파일에 출력하여 메시지 이해를 확인합니다.
[x] Find the problematic constraint or issue
[x] 문제가 있는 제약 조건이나 문제를 찾습니다.
[x] Understand why it is problematic
[x] 문제가 되는 이유 이해
[x] Understand the correct resolution and why it is correct
[x] 올바른 해상도와 그것이 올바른 이유를 이해합니다.
[x] Understand why the selected resolution is better than the other options if they exist
[x] 선택한 해상도가 다른 옵션보다 나은 이유(존재하는 경우) 이해
* Increases confidence that any constraint fixes do not have unintended consequences
* 모든 제약 조건 수정으로 인해 의도하지 않은 결과가 발생하지 않는다는 확신을 높입니다.

## Step 8: Progress update
## 8단계: 진행 상황 업데이트
At the end of each group stage, checks are regenerated and fix criteria is tested.
각 그룹 단계가 끝나면 검사가 다시 생성되고 수정 기준이 테스트됩니다.
```tcl
reset_timing
read_xdc <new_constraints.xdc>   
report_methodology
write_json_report -report methodology -file ./tmp/methodology_after_fix_group_n.json
```

## Step 9: Repeat until all groups are resolved
## 9단계: 모든 그룹이 해결될 때까지 반복합니다.
Repeat steps 2-8 until all groups are resolved.
모든 그룹이 해결될 때까지 2~8단계를 반복합니다.

## Step 10: Final report
## 10단계: 최종 보고서
At the end of all groups, generate `METHODOLOGY_RESOLUTIONS_REPORT.md`
모든 그룹의 마지막에 `METHODOLOGY_RESOLUTIONS_REPORT.md`를 생성합니다.

**FILE LOCATION: Write to `./METHODOLOGY_RESOLUTION_REPORT.md` in the current working directory (the design directory where the .dcp file is located).
**파일 위치: 현재 작업 디렉터리(.dcp 파일이 있는 디자인 디렉터리)의 `./METHODOLOGY_RESOLUTION_REPORT.md`에 씁니다.
Do NOT use parent directory paths like `../` or absolute paths.**
`../` 또는 절대 경로와 같은 상위 디렉토리 경로를 사용하지 마십시오.**

1. Summary
1. 요약
   - Methodology violations before and after
   - 전후의 방법론 위반
   - Include IDs that are not resolved because they are not supported
   - 지원되지 않아 확인되지 않은 ID를 포함합니다.
   - Include IDs that are not resolved because user decision is required and user has not made a decision yet.
   - 사용자 결정이 필요하고 사용자가 아직 결정을 내리지 않았기 때문에 해결되지 않은 ID를 포함합니다.
2. Resolved Violations
2. 해결된 위반
  - Foreach violation resolved, capture the action in a concise way
  - Foreach 위반이 해결되었으며 간결한 방식으로 작업을 캡처합니다.
  - Include user decisions for each violation
  - 각 위반에 대한 사용자 결정을 포함합니다.
  - Group IDs whose actions are the same when the same action solves multiple violations together to make the report more concise
  - 동일한 조치로 여러 위반 사항을 함께 해결하여 보고서를 더욱 간결하게 만들 때 해당 조치가 동일한 그룹 ID
  - Include a snippet of changes made to constraint file with diffs,
  - 차이점이 있는 제약 조건 파일에 대한 변경 내용을 포함합니다.
3. Waivers section
3. 면제 섹션
4. Remaining Actions
4. 남은 작업

**CRITICAL MARKDOWN FORMATTING:**
**중요한 마크다운 형식:**
- Use proper markdown code fences with THREE backticks (```) for TCL code blocks
- TCL 코드 블록에 백틱 3개(````)가 있는 적절한 마크다운 코드 펜스를 사용하세요.
- Use single backticks (`) for inline code in tables and text
- 테이블과 텍스트의 인라인 코드에 단일 백틱(`) 사용
- Do NOT escape backticks with backslashes or forward slashes
- 백틱이나 슬래시를 사용하여 백틱을 이스케이프 처리하지 마세요.
- Write markdown directly to the file without escaping special characters
- 특수 문자를 이스케이프 처리하지 않고 파일에 직접 마크다운 쓰기
- **IMPORTANT**: Write actual newlines (`\n`), not the literal string `/n`
- **중요**: 리터럴 문자열 `/n`이 아닌 실제 줄 바꿈(`\n`)을 작성하세요.

**HOW TO WRITE THE MARKDOWN FILE PROPERLY:**
**마크다운 파일을 올바르게 작성하는 방법:**

When generating `METHODOLOGY_RESOLUTION_REPORT.md`, you MUST write it as a properly formatted markdown file with actual newlines.
`METHODOLOGY_RESOLUTION_REPORT.md`를 생성할 때 실제 줄 바꿈이 포함된 올바른 형식의 마크다운 파일로 작성해야 합니다.
Use one of these approaches:
다음 접근 방식 중 하나를 사용하십시오.

**Build content string with proper newlines:**
**적절한 줄 바꿈을 사용하여 콘텐츠 문자열 작성:**
```tcl
set content "# Methodology Resolution Report\n\n"
append content "**Design:** $design_name\n"
append content "**Date:** $timestamp\n\n"
append content "## 1. Summary\n\n"
# ... continue building content
set report_file [open "./METHODOLOGY_RESOLUTION_REPORT.md" w]
puts -nonewline $report_file $content
close $report_file
```
**CRITICAL MARKDOWN FORMATTING:**
**중요한 마크다운 형식:**
- Use proper markdown code fences with THREE backticks (```) for TCL code blocks
- TCL 코드 블록에 백틱 3개(````)가 있는 적절한 마크다운 코드 펜스를 사용하세요.
- Use single backticks (`) for inline code in tables and text
- 테이블과 텍스트의 인라인 코드에 단일 백틱(`) 사용
- Do NOT escape backticks with backslashes or forward slashes
- 백틱이나 슬래시를 사용하여 백틱을 이스케이프 처리하지 마세요.
- Write markdown directly to the file without escaping special characters
- 특수 문자를 이스케이프 처리하지 않고 파일에 직접 마크다운 쓰기
- **IMPORTANT**: Write actual newlines (`\n`), not the literal string `/n`
- **중요**: 리터럴 문자열 `/n`이 아닌 실제 줄 바꿈(`\n`)을 작성하세요.
- **TABLES**: Use exactly ONE pipe `|` between columns and at start/end of rows
- **테이블**: 열 사이와 행의 시작/끝에 정확히 하나의 파이프 `|`를 사용합니다.
- **TABLES**: Separator rows must have SINGLE pipes only: `|----------|----------|`
- **표**: 구분 행에는 단일 파이프만 있어야 합니다. `|----------|----------|`
- **NEVER use double pipes `||`** - this breaks markdown table rendering
- **이중 파이프 `||`**를 사용하지 마세요. 이로 인해 마크다운 테이블 렌더링이 중단됩니다.

**CORRECT table format:**
**올바른 표 형식:**
```markdown
| Column 1 | Column 2 | Column 3 |
|----------|----------|----------|
| Value A  | Value B  | Value C  |
```
**INCORRECT table format (DO NOT USE):**
**잘못된 테이블 형식(사용하지 마세요):**
```markdown
| Check ID | Severity | Count |
|----------|----------|-------||  ← WRONG! Double pipe breaks rendering
```

**PER-VIOLATION TABLE FORMAT:**
**위반별 표 형식:**
For each resolved violation, include a comparison table with this exact format (following TIMING-1.md Final Report example):
해결된 각 위반에 대해 정확한 형식의 비교표를 포함하십시오(TIMING-1.md 최종 보고서 예 참조).

| Aspect<br>측면 | Period<br>기간 | Waveform<br>파형 | Clock Definition Object<br>시계 정의 객체 | Constraint<br>제약 |
|--------|--------|----------|------------------------|-----------|
| **Before**<br>**이전** | <value> ns<br><값> ns | {<rise> <fall>}<br>{<상승> <하락>} | <clock_name><br><시계_이름> | `<full constraint command>`<br>`<전체 제약 명령>` |
| **After**<br>**이후** | <value> ns<br><값> ns | {<rise> <fall>}<br>{<상승> <하락>} | <clock_name><br><시계_이름> | `<full constraint command>`<br>`<전체 제약 명령>` |

Requirements:
요구사항:
- Column headers must be: Aspect, Constraint, Period, Waveform, Clock Definition Object
- 열 헤더는 Aspect, Constraint, Period, Waveform, Clock Definition Object여야 합니다.
- Constraint column must show FULL TCL command with all options, source, and target objects
- 제약 조건 열에는 모든 옵션, 소스 및 대상 개체와 함께 FULL TCL 명령이 표시되어야 합니다.
- Do NOT truncate constraint commands - include complete `-source`, `-name`, `-multiply_by`, `-divide_by`, and target pin/port specifications
- 제약 조건 명령을 자르지 마십시오. 전체 `-source`, `-name`, `-multiply_by`, `-divide_by` 및 대상 핀/포트 사양을 포함합니다.
- Period column format: numeric value followed by "ns" (e.g., "8.000 ns")
- 기간 열 형식: 숫자 값 뒤에 "ns"가 붙음(예: "8.000 ns")
- Waveform column format: {rise fall} (e.g., "{0 4.000}")
- 파형 열 형식: {rise fall}(예: "{0 4.000}")

Create a diff report in HTML file.
HTML 파일로 차이점 보고서를 만듭니다.
- review with before on the left and after on the right.
- 왼쪽이 이전, 오른쪽이 이후로 검토하세요.

  ### Example: Before and After Comparison
  ### 예: 전후 비교

| Check ID<br>신분증 확인 | Severity<br>심각도 | Count Before<br>이전 계산 | Count After<br>이후 계산 | Status & Resolution<br>상태 및 해결 방법 |
  |----------|----------|--------------|-------------|---------------------|
| TIMING-1<br>타이밍-1 | Critical Warning<br>심각한 경고 | 5 | 0 | **Resolved**: Modified `create_clock` period in pbf_timing.xdc<br>**해결됨**: pbf_timing.xdc에서 'create_clock' 기간을 수정했습니다. |
| TIMING-2<br>타이밍-2 | Critical Warning<br>심각한 경고 | 3 | 0 | **Resolved**: Added `set_clock_groups -asynchronous` for clock domain crossing<br>**해결됨**: 클럭 도메인 교차를 위해 'set_clock_groups -asynchronous'를 추가했습니다. |
| TIMING-3<br>타이밍-3 | Critical Warning<br>심각한 경고 | 2 | 0 | **Resolved**: Converted to `create_generated_clock`<br>**해결됨**: `create_generated_clock`으로 변환됨 |
| TIMING-17<br>타이밍-17 | Info<br>정보 | 2 | 2 | **Waived**: Design intent verified with user<br>**면제**: 사용자가 디자인 의도를 확인했습니다. |
| TIMING-5<br>타이밍-5 | Warning<br>경고 | 1 | 0 | **Resolved**: Commented out redundant constraint<br>**해결됨**: 중복 제약 조건을 주석 처리했습니다. |
| SYNTH-5<br>합성-5 | Warning<br>경고 | 864 | 864 | **Not Supported**: Connection to uninitialized flip-flop (requires RTL changes)<br>**지원되지 않음**: 초기화되지 않은 플립플롭에 연결(RTL 변경 필요) |
| SYNTH-6<br>합성-6 | Warning<br>경고 | 1012 | 1012 | **Not Supported**: Timing of unconnected port (requires RTL changes)<br>**지원되지 않음**: 연결되지 않은 포트의 타이밍(RTL 변경 필요) |
| TIMING-18<br>타이밍-18 | Warning<br>경고 | 160 | 160 | **Not Supported**: Missing input or output delay constraints (requires user decision)<br>**지원되지 않음**: 입력 또는 출력 지연 제약 조건 누락(사용자 결정 필요) |
| LUTAR-1<br>루타-1 | Warning<br>경고 | 27 | 27 | **Not Supported**: LUT equation term check (requires RTL review)<br>**지원되지 않음**: LUT 방정식 항 확인(RTL 검토 필요) |
| **Total**<br>**합계** | | **2076** | **2065** | **11 violations resolved, 2065 require RTL changes or user decisions**<br>**11개 위반이 해결되었으며 2065년에는 RTL 변경 또는 사용자 결정이 필요함** |

  ### Example: TIMING3#2 Resolution
  ### 예: TIMING3#2 분해능
  - **Violation**: TIMING-1#1 - Invalid clock waveform on Clock Modifying Block
  - **위반**: TIMING-1#1 - 클록 수정 블록의 잘못된 클록 파형
  - **Action**: Modified `create_clock` period from 5.000 to 12.800 in `pbf_timing.xdc` on line 45
  - **조치**: 45행의 `pbf_timing.xdc`에서 `create_clock` 기간을 5.000에서 12.800으로 수정했습니다.
  - **Reasoning**: User confirmed resolution
  - **추론**: 사용자가 확인한 해결 방법
  #### Before
  #### 이전
  ```tcl
  create_clock -period 8.000 -name REF_CLK [get_ports REF_CLK]
  ```

  #### After
  #### 이후
  ```tcl
  # Fixed TIMING-3#2: Changed from create_clock to create_generated_clock
  # Preserves clock name SYS_CLK while respecting MMCM output
  create_generated_clock -name SYS_CLK -source [get_pins ios_0/mmcm_0/CLKIN1] -multiply_by 30 -divide_by 30 [get_pins ios_0/mmcm_0/CLKOUT1]
   ```

# Design Edits and Actions
# 디자인 편집 및 작업

## Automatic Fixes
## 자동 수정
When the reference documentation marks a resolution as **AUTO** with high confidence, apply the fix directly without prompting:
참조 문서에 높은 신뢰도로 해결 방법이 **AUTO**로 표시되어 있으면 메시지를 표시하지 않고 직접 수정 사항을 적용하세요.

1. Create backup file: `cp constraints.xdc constraints_after_fix.xdc`
1. 백업 파일 생성: `cp Constraints.xdc Constraints_after_fix.xdc`
2. Apply the change using the **modify** approach (replace/add/remove constraint as specified)
2. **수정** 접근 방식을 사용하여 변경 사항 적용(지정된 대로 제약 조건 교체/추가/제거)
3. Report exactly what was changed with before/after code blocks
3. 코드 블록 이전/이후로 변경된 사항을 정확하게 보고합니다.
4. Re-run `report_methodology` to verify the fix
4. 'report_methodology'를 다시 실행하여 수정 사항을 확인하세요.
5. Generate HTML diff file showing differences between `constraints.xdc` and `constraints_after_fix.xdc`
5. `constraints.xdc`와 `constraints_after_fix.xdc`의 차이점을 보여주는 HTML diff 파일 생성

**Note**: For AUTO fixes, always use the "modify" approach (direct replacement).
**참고**: 자동 수정의 경우 항상 "수정" 접근 방식(직접 교체)을 사용하십시오.
Only use "comment out" for USER-directed fixes where the user explicitly requests it.
사용자가 명시적으로 요청하는 경우 사용자가 직접 수정하는 경우에만 "주석 처리"를 사용하세요.



## User Directed Fixes
## 사용자 지시 수정
1. Generate Vivado command based on user's choice
1. 사용자 선택에 따라 Vivado 명령어 생성
2. Show command for review
2. 검토를 위해 명령 표시
3. Ask: "Execute this command?
3. 질문: "이 명령을 실행하시겠습니까?
3. [Y/n]"
4. If yes,
4. 그렇다면,
a.
Create a new file: `cp constraints.xdc constraints_after_fix.xdc`
새 파일을 만듭니다: `cp Constraints.xdc Constraints_after_fix.xdc`
b.
Apply the change (remove/modify/add constraint)
변경사항 적용(제약조건 제거/수정/추가)
c.
Report exactly what was changed
무엇이 변경되었는지 정확하게 보고하세요.
d.
Say: "Re-run report_methodology to verify"
다음과 같이 말하세요. "report_methodology를 다시 실행하여 확인하세요."
e.
Generate a HTML file detailing the differences between `constraints.xdc` and  `constraints_after_fix.xdc`
`constraints.xdc`와 `constraints_after_fix.xdc`의 차이점을 자세히 설명하는 HTML 파일을 생성합니다.
5. If no, save command to script for manual execution
5. 그렇지 않은 경우 수동 실행을 위해 명령을 스크립트에 저장하십시오.

## Creating Waivers
## 권리 포기 생성
1. Ask: "Provide justification (will be recorded):"
1. 질문: "정당한 이유를 제시하세요(기록됩니다):"
2. Collect: reason for waiving
2. 수집: 포기 사유
3. Format: `<username> <date> "<reason>"`
3. 형식: `<사용자 이름> <날짜> "<이유>"`
4. Generate: `create_waiver -of_objects [get_methodology_violations TIMING-3#1] -user poldark -description <description> -tags <related_waiver_tag>"`
4. 생성: `create_waiver -of_objects [get_methodology_violations TIMING-3#1] -user poldark -description <설명> -tags <관련_waiver_tag>"`
5. Warn: "This violation will be ignored in future runs"
5. 경고: "이 위반은 향후 실행에서 무시됩니다."

---

# Critical Rules
# 중요한 규칙

## DO:
✅ Always read `references/<CHECK_ID>.md` when it exists before responding
✅ `references/<CHECK_ID>.md`가 있으면 응답하기 전에 항상 읽어보세요.
✅ Verify AUTO conditions before claiming you can fix it
✅ 문제를 해결할 수 있다고 주장하기 전에 AUTO 조건을 확인하세요
✅ Present ALL options for USER resolutions
✅ 사용자 해결을 위한 모든 옵션 제공
✅ For AUTO fixes: Apply immediately without prompting (high/medium confidence cases)
✅ 자동 수정의 경우: 프롬프트 없이 즉시 적용(높은/중간 신뢰도 사례)
✅ For USER fixes: Ask confirmation before modifying any files
✅ 사용자 수정: 파일을 수정하기 전에 확인 요청
✅ Document every change made
✅ 모든 변경사항을 문서화하세요
✅ Solve violations by group (Group-1 before Group-2)
✅ 그룹별 위반사항 해결(그룹-2 이전에 그룹-1)
✅ Solve violations by priority (Group-1-P1 before Group-1-P2)
✅ 우선순위에 따라 위반사항 해결(Group-1-P2 이전에 Group-1-P1)
✅ Re-run report_methodology after completing each group
✅ 각 그룹을 완료한 후 report_methodology를 다시 실행하세요.
✅ Generate HTML diff after constraint modifications
✅ 제약 조건 수정 후 HTML diff 생성
✅ Recommend verification after fixes
✅ 수정 후 확인 권장
✅ Use single pipes `|` in markdown tables
✅ 마크다운 테이블에 단일 파이프 `|` 사용
✅ Verify table formatting before writing files
✅ 파일을 쓰기 전에 테이블 형식을 확인하세요

## DON'T:
## 하지 말아야 할 것:
❌ Never guess design intent for USER resolutions
❌ USER 해상도에 대한 설계 의도를 절대 추측하지 마세요.
❌ Never apply AUTO fixes without verifying the conditions in the reference docs
❌ 참조 문서의 조건을 확인하지 않고 AUTO 수정 사항을 적용하지 마십시오.
❌ Never apply USER fixes without explicit user confirmation
❌ 명시적인 사용자 확인 없이는 USER 수정 사항을 적용하지 마세요.
❌ Never skip reading the reference documentation
❌ 참조 문서 읽기를 건너뛰지 마세요.
❌ Never approve waivers without proper justification
❌ 적절한 정당성 없이 면제를 승인하지 마십시오.
❌ Never proceed if violation properties are missing
❌ 위반 속성이 누락된 경우 진행하지 마세요.
❌ Never skip to next group without re-running report_methodology
❌ Report_methodology를 다시 실행하지 않고 다음 그룹으로 건너뛰지 마세요.
❌ **Never use double pipes `||` in markdown tables**
❌ **마크다운 테이블에서는 이중 파이프 `||`를 절대 사용하지 마세요**
❌ Never end table separator rows with `||`
❌`||`로 테이블 구분 행을 종료하지 마세요.
❌ Never add extra pipes at end of table rows
❌ 테이블 행 끝에 추가 파이프를 추가하지 마세요.


# File Locations
# 파일 위치

**Scripts:**
**스크립트:**
- `./scripts/trace_clock_tree.tcl` - Clock tracing utility
- `./scripts/trace_clock_tree.tcl` - 시계 추적 유틸리티
- `./scripts/get_id_info.sh`       - Utility to understand supported checks
- `./scripts/get_id_info.sh` - 지원되는 검사를 이해하는 유틸리티

**Reference Docs:**
**참조 문서:**
- `./references/<CHECK_ID>.md` - Resolution guidance per violation
- `./references/<CHECK_ID>.md` - 위반별 해결 지침
- Example: `./references/TIMING-1.md`
- 예: `./references/TIMING-1.md`

---

# Integration with Other Skills
# 다른 기술과의 통합

Use alongside:
함께 사용:
- **creating-vivado-tcl**: Generate constraint scripts after determining fix
- **creating-vivado-tcl**: 수정 사항을 결정한 후 제약 조건 스크립트 생성
- **design-analysis**: Understand design context before choosing resolution
- **디자인 분석**: 해상도를 선택하기 전에 디자인 컨텍스트를 이해하세요.
- **creating-custom-qor-suggestion-checks**: Cross-reference with QoR recommendations
- **creating-custom-qor-suggestion-checks**: QoR 권장 사항과의 상호 참조
