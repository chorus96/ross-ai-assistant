<!-- ko-translation: enabled -->
---
name: vivado-rtl-elaboration-analysis
description: >
  Analyze Vivado Oasys synthesis elaboration errors, critical warnings, and warnings
  from synthesis logs and provide actionable RTL code fixes. Log analysis is the
  primary mode; an active Vivado session is secondary. Inspect the user's actual RTL
  at every reported file and line before proposing a fix.
license: MIT
allowed-tools: vivadoExecute vivado_doc_search read_file file_search grep_search run_in_terminal create_file replace_string_in_file runSubagent
argument-hint: "[RTL/project path and optional runme.log or vivado.log]"
metadata:
  version: "1.0.0"
---

# RTL Elaboration Analysis — Oasys Synthesis
# RTL 정교화 분석 — Oasys 합성

## Purpose
## 목적

This skill analyzes **Oasys synthesis elaboration diagnostics** from Vivado and ties
이 스킬은 Vivado의 **Oasys 합성 정교 진단**을 분석하고 이를 연결합니다.
every finding to the user's actual RTL.
사용자의 실제 RTL에 대한 모든 결과.
Messages appear as `[Synth 8-XXX]` in logs.
메시지는 로그에 '[Synth 8-XXX]'로 표시됩니다.

The runtime input may be:
런타임 입력은 다음과 같습니다.
- RTL source files (`.v`, `.sv`, `.vhd`, `.vhdl`) and their project/file-list context
- RTL 소스 파일(`.v`, `.sv`, `.vhd`, `.vhdl`) 및 해당 프로젝트/파일 목록 컨텍스트
- A Vivado `runme.log`, `vivado.log`, messages CSV, or pasted diagnostic
- Vivado `runme.log`, `vivado.log`, 메시지 CSV 또는 붙여넣은 진단
- An active Vivado session for secondary MCP-mode analysis
- 보조 MCP 모드 분석을 위한 활성 Vivado 세션

The skill covers parser, type-checker, and elaboration phases of `synth_design` and
이 기술은 `synth_design`의 파서, 유형 검사기 및 정교화 단계를 다루고 있으며
reports:
보고서:
- Summary by severity and message ID
- 심각도 및 메시지 ID별 요약
- Root-cause explanation tied to actual RTL locations
- 실제 RTL 위치와 관련된 근본 원인 설명
- Hotspot analysis across affected RTL files
- 영향을 받은 RTL 파일 전체에 대한 핫스팟 분석
- RTL-level fix guidance with diff-formatted patches
- Diff 형식의 패치를 사용한 RTL 수준 수정 지침
- Tier classification and prioritized resolution order
- 계층 분류 및 우선순위 해결 순서

## ID Coverage
## 신분증 범위

This skill covers **Oasys message IDs** from testcases (`synth_code_8_xxx_testcase/`):
이 기술은 테스트 케이스(`synth_code_8_xxx_testcase/`)의 **Oasys 메시지 ID**를 다룹니다.
- **Critical Warnings (8 IDs)**: 4442, 5796, 5967, 6030, 6781, 6790, 6858, 13329
- **중요 경고(ID 8개)**: 4442, 5796, 5967, 6030, 6781, 6790, 6858, 13329
- **Errors (~400 IDs)**: 27-13331 (see `oasys_severity_lists.csv`)
- **오류(ID 최대 400개)**: 27-13331(`oasys_severity_lists.csv` 참조)
- **Warnings (~86 IDs)**: 85, 90, 151, 264, 302, 324, 327, 506, 567, 614, 689, etc.
- **경고(~86개 ID)**: 85, 90, 151, 264, 302, 324, 327, 506, 567, 614, 689 등

Reference: `oasys_severity_lists.csv` for full message descriptions.
참조: 전체 메시지 설명은 `oasys_severity_lists.csv`를 참조하세요.

## Scope
## 범위

Use this skill when:
다음과 같은 경우에 이 기술을 사용하십시오.
- A `vivado.log`, `runme.log`, or `synth_1/runme.log` contains Oasys elaboration messages.
- `vivado.log`, `runme.log` 또는 `synth_1/runme.log`에는 Oasys 정교화 메시지가 포함되어 있습니다.
- The user asks why synthesis failed or asks to fix synthesis warnings/errors.
- 사용자가 합성이 실패한 이유를 묻거나 합성 경고/오류를 수정하도록 요청합니다.
- The user provides RTL/project sources for source-context inspection.
- 사용자는 소스 컨텍스트 검사를 위해 RTL/프로젝트 소스를 제공합니다.

Do not use this skill when:
다음과 같은 경우에는 이 기술을 사용하지 마십시오.
- No log and no active Vivado session are available.
- 로그 및 활성 Vivado 세션을 사용할 수 없습니다.
- User needs implementation/timing/physical optimization analysis.
- 사용자는 구현/타이밍/물리적 최적화 분석이 필요합니다.
- The issue is unrelated to RTL elaboration or synthesis.
- 이 문제는 RTL 정교화 또는 합성과 관련이 없습니다.

## Inputs
## 입력

Minimum required input:
최소 필수 입력:
- Vivado synthesis log (`vivado.log`, `runme.log`, or `synth_1/runme.log`), or
- Vivado 합성 로그(`vivado.log`, `runme.log` 또는 `synth_1/runme.log`) 또는
- Active Vivado session through MCP
- MCP를 통한 활성 Vivado 세션

Useful optional input:
유용한 선택적 입력:
- User RTL/project path accessible in the workspace
- 작업공간에서 액세스 가능한 사용자 RTL/프로젝트 경로
- Top module/entity and target part
- 최상위 모듈/엔티티 및 대상 부분
- Vivado project file, file list, or synthesis script
- Vivado 프로젝트 파일, 파일 목록 또는 합성 스크립트
- Constraints and included packages/headers
- 제약 조건 및 포함된 패키지/헤더
- Existing user RTL fixes or intended behavior
- 기존 사용자 RTL 수정 사항 또는 의도된 동작

## Mandatory Workflow
## 필수 워크플로우

Execute these steps sequentially in this exact order.
이 단계를 정확한 순서대로 순차적으로 실행하세요.

### Step 1: Locate and Identify Log File
### 1단계: 로그 파일 찾기 및 식별

Determine operating mode and find the synthesis log.
작동 모드를 결정하고 합성 로그를 찾습니다.

**Log Analysis Mode** (default when user provides a log file path or log exists in workspace):
**로그 분석 모드**(사용자가 로그 파일 경로를 제공하거나 작업공간에 로그가 있는 경우 기본값):
1. If `$ARGUMENTS` is a `.log` file path, use it directly
1. `$ARGUMENTS`가 `.log` 파일 경로인 경우 직접 사용하세요.
2. Otherwise search workspace:
2. 그렇지 않으면 작업공간을 검색하십시오.
   ```
   file_search("**/runme.log")
   file_search("**/vivado.log")
   file_search("**/*.log")
   ```
3. If multiple logs found, prefer `synth_1/runme.log` > `vivado.log` > others
3. 여러 로그가 발견되면 `synth_1/runme.log` > `vivado.log` > 기타를 선호하세요.
4. If no log found, switch to MCP Mode
4. 로그가 없으면 MCP 모드로 전환하세요.

**MCP Mode** (when no log exists or user requests live analysis):
**MCP 모드**(로그가 없거나 사용자가 실시간 분석을 요청하는 경우):
1. Load Vivado tools: `tool_search` with query `"vivado execute"`
1. Vivado 도구 로드: `"vivadoexecute"` 쿼리를 사용하여 `tool_search`
2. Follow TCL in [tcl-reference.md § Step 1](tcl-reference.md) to detect project/non-project
2. 프로젝트/비프로젝트를 감지하려면 [tcl-reference.md § 1단계](tcl-reference.md)의 TCL을 따르세요.
3. Run `synth_design` — TCL in [tcl-reference.md § Step 2](tcl-reference.md)
3. `synth_design` 실행 — [tcl-reference.md § 2단계](tcl-reference.md)의 TCL
4. Locate the resulting log file
4. 결과 로그 파일을 찾습니다.

**Verify:** Log file path is known and file exists.
**확인:** 로그 파일 경로가 알려져 있고 파일이 존재합니다.

### Step 2: Create Report Directory
### 2단계: 보고서 디렉터리 생성

Create the report directory **inside the test case directory** (same directory as the
**테스트 케이스 디렉토리 안에** 보고서 디렉토리를 생성합니다(테스트 케이스 디렉토리와 동일한 디렉토리).
log file).
로그 파일).
This ensures each test case has its own self-contained report.
이렇게 하면 각 테스트 사례에 자체 포함된 보고서가 포함됩니다.

```
<test_case_dir>/vivado_agentic_ai_reports/rtl-elaboration-analysis/
```

Where `<test_case_dir>` is the directory containing the vivado.log (e.g.,
여기서 `<test_case_dir>`은 vivado.log가 포함된 디렉터리입니다(예:
`error/8_27/bcnmxd_vhdl_neg_DataTypeMap_DataTypeMap4/`).
`오류/8_27/bcnmxd_vhdl_neg_DataTypeMap_DataTypeMap4/`).

Use `run_in_terminal` to create: `mkdir -p <test_case_dir>/vivado_agentic_ai_reports/rtl-elaboration-analysis`
`run_in_terminal`을 사용하여 `mkdir -p <test_case_dir>/vivado_agentic_ai_reports/rtl-elaboration-analytic`을 생성합니다.

Or if in MCP mode: `file mkdir <test_case_dir>/vivado_agentic_ai_reports/rtl-elaboration-analysis`
또는 MCP 모드인 경우: `file mkdir <test_case_dir>/vivado_agentic_ai_reports/rtl-elaboration-analytic`

**Rule:** Never create a single shared report directory at workspace root.
**규칙:** 작업공간 루트에 단일 공유 보고서 디렉터리를 만들지 마세요.
Each test
각 테스트
case gets its own `vivado_agentic_ai_reports/` alongside its log and source files.
케이스는 로그 및 소스 파일과 함께 자체 `vivado_agentic_ai_reports/`를 가져옵니다.

### Step 3: Parse Elaboration Messages
### 3단계: 상세 메시지 구문 분석

Extract all elaboration messages from the log.
로그에서 모든 상세 메시지를 추출합니다.
Execute the parser:
파서를 실행합니다.

```bash
python3 <skill_dir>/parse_elab_messages.py <log_file> vivado_agentic_ai_reports/rtl-elaboration-analysis/messages.csv
```

Where `<skill_dir>` is the path to this skill's directory.
여기서 `<skill_dir>`은 이 스킬 디렉터리의 경로입니다.

**If Python is unavailable**, parse manually using `grep_search` on the log file:
**Python을 사용할 수 없는 경우** 로그 파일에서 `grep_search`를 사용하여 수동으로 구문 분석하세요.
- Pattern: `^(ERROR|CRITICAL WARNING|WARNING):\s*\[Synth\s+8-\d+\]`
- 패턴: `^(오류|중요한 경고|경고):\s*\[Synth\s+8-\d+\]`
- Extract: severity, message ID, text, file path, line number
- 추출: 심각도, 메시지 ID, 텍스트, 파일 경로, 줄 번호

**CSV column schema** (with header row, 7 columns):
**CSV 열 스키마**(헤더 행, 7개 열 포함):

| Col<br>콜 | Field<br>필드 | Description<br>설명 |
|-----|-------|-------------|
| 0 | S.No<br>S.No | Sequential serial number (1-based) for cross-referencing with report<br>보고서와의 상호참조를 위한 일련번호(1단위) |
| 1 | Severity<br>심각도 | `ERROR`, `CRITICAL WARNING`, `WARNING`, or `INFO`<br>`ERROR`, `CRITICAL WARNING`, `WARNING` 또는 `INFO` |
| 2 | Msg_ID<br>Msg_ID | Integer message ID from `[Synth 8-XXX]`<br>`[Synth 8-XXX]`의 정수 메시지 ID |
| 3 | Message<br>메시지 | Full message text<br>전체 메시지 텍스트 |
| 4 | File<br>파일 | Source file path (may be empty)<br>소스 파일 경로(비어 있을 수 있음) |
| 5 | Line<br>라인 | Source line number (0 if not available)<br>소스 줄 번호(사용할 수 없는 경우 0) |
| 6 | Language<br>언어 | `VLOG` or `VHDL` (detected from file extension)<br>`VLOG` 또는 `VHDL`(파일 확장자에서 감지됨) |

**Verify:** CSV exists.
**확인:** CSV가 존재합니다.
Row count (excluding header) matches message count from log.
행 수(헤더 제외)는 로그의 메시지 수와 일치합니다.

### Step 3.5: Create Resolution Checklist
### 3.5단계: 해결 체크리스트 만들기

Extract unique `msg_id` values (excluding INFO) from the CSV and classify each by
CSV에서 고유한 `msg_id` 값(INFO 제외)을 추출하고 각각을 다음과 같이 분류합니다.
resolution availability.
해상도 가용성.
Write to:
다음 주소로 편지를 보내세요:
`vivado_agentic_ai_reports/rtl-elaboration-analysis/resolution_checklist.md`
`vivado_agentic_ai_reports/rtl-elaboration-analytic/solution_checklist.md`

**Procedure:**
**절차:**
1. Read `messages.csv` (skip header row) and collect unique `msg_id` values (skip INFO)
1. `messages.csv` 읽기(헤더 행 건너뛰기) 및 고유 `msg_id` 값 수집(INFO 건너뛰기)
2. For each unique ID, look it up in [message-handlers.md](message-handlers.md)
2. 각 고유 ID에 대해 [message-handlers.md](message-handlers.md)에서 찾아보세요.
3. Write `resolution_checklist.md` with two sections:
3. 두 개의 섹션으로 `solution_checklist.md`를 작성하세요.

```markdown
# Resolution Checklist

## HAS RESOLUTION FILE
| ID | Severity | Short Name | Resolution File | Loaded? |
|----|----------|------------|-----------------|--------|
| <id> | <sev> | <name> | resolution/<file>.md | [ ] |

## NO RESOLUTION FILE (use vivado_doc_search or own analysis)
| ID | Severity | Description | Source |
|----|----------|-------------|--------|
| <id> | <sev> | <text> | vivado_doc_search / own analysis |
```

**Verify:** Every unique WARNING/ERROR/CRITICAL WARNING `msg_id` from the CSV
**확인:** CSV의 모든 고유한 경고/오류/중요한 경고 `msg_id`
appears in exactly one section.
정확히 한 섹션에 나타납니다.

### Step 4: Classify and Prioritize
### 4단계: 분류 및 우선순위 지정

Read the CSV and organize messages:
CSV를 읽고 메시지를 정리합니다.

**4a.
Severity grouping:**
심각도 그룹화:**
1. **Errors** — Must fix.
1. **오류** — 수정해야 합니다.
1. Block synthesis completion.
1. 블록 합성 완료.
2. **Critical warnings** — Should fix.
2. **중요한 경고** — 수정해야 합니다.
2. May cause incorrect synthesis results.
2. 잘못된 합성 결과가 발생할 수 있습니다.
3. **Warnings** — Review and fix or waive.
3. **경고** — 검토하고 수정하거나 포기하세요.

**4b.
Actionability classification:**
실행 가능성 분류:**

For each message, look up `msg_id` in [message-handlers.md](message-handlers.md):
각 메시지에 대해 [message-handlers.md](message-handlers.md)에서 `msg_id`를 찾아보세요.

- **Tier 1 (Directly Fixable):** Deterministic fix — apply without user input
- **계층 1(직접 수정 가능):** 결정적 수정 — 사용자 입력 없이 적용
- **Tier 2 (Fixable with Context):** Fix requires reading surrounding RTL
- **계층 2(컨텍스트로 수정 가능):** 수정하려면 주변 RTL을 읽어야 합니다.
- **Tier 3 (Advisory):** Report but flag as "requires design decision"
- **계층 3(권고):** 보고하지만 '설계 결정 필요'로 플래그 지정
- **Not in table:** Treat as Tier 2, use `resolution/uncovered-synth-messages.md`
- **표에 없음:** Tier 2로 취급, `solution/uncovered-synth-messages.md` 사용

**4c.
File hotspot analysis:**
파일 핫스팟 분석:**
- Group messages by source file
- 소스 파일별로 메시지 그룹화
- Rank files by total message count (descending)
- 총 메시지 수를 기준으로 파일 순위 지정(내림차순)
- Files with >10 messages are "hotspots" — recommend structural review
- 메시지가 10개를 초과하는 파일은 "핫스팟"입니다. 구조 검토를 권장합니다.

**4d.
Dependency detection:**
종속성 감지:**
- Identify root cause messages vs cascading effects
- 근본 원인 메시지와 계단식 영향을 식별합니다.
- Prioritize root cause fixes
- 근본 원인 수정 우선순위 지정
- Dependency classification only sets fix **priority** and section wording.
- 종속성 분류는 수정 **우선순위** 및 섹션 문구만 설정합니다.
- It NEVER
- 절대로
removes a message from the report: cascade, duplicate, and read-abort messages
보고서에서 메시지 제거: 계단식 메시지, 중복 메시지, 읽기-중단 메시지
(e.g.
`Failed to read`, `second declaration ignored`, `syntax error near`) each
`읽지 못했습니다`, `두 번째 선언이 무시됨`, `근접 구문 오류`) 각각
still get their own `###` section in Step 6 (may be brief and marked
여전히 6단계에서 자체 `###` 섹션을 얻습니다(간단하고 표시될 수 있음).
`Cascade of 8-XXX`, but must be present).
'8-XXX의 캐스케이드'(반드시 존재해야 함).

### Step 5: Generate Fixes
### 5단계: 수정 사항 생성

Process **every non-INFO message** in messages.csv.
message.csv에서 **INFO가 아닌 모든 메시지**를 처리합니다.
The folder/log may be named after
폴더/로그 이름은 다음과 같이 지정될 수 있습니다.
one "target" ID, but that NEVER limits the report to that ID alone — all messages in
하나의 "대상" ID이지만 보고서를 해당 ID만으로 제한하지 않습니다.
the CSV are in scope.
CSV가 범위 내에 있습니다.
For each actionable message (Tier 1 and Tier 2), in priority
실행 가능한 각 메시지(계층 1 및 계층 2)에 대해 우선순위
order (errors first):
순서(오류 우선):

1. **Read RTL source context:**
1. **RTL 소스 컨텍스트 읽기:**
   - Resolve file path from CSV
   - CSV에서 파일 경로 확인
   - If relative or filename-only: `file_search("**/<filename>")`
   - 상대 또는 파일 이름만 있는 경우: `file_search("**/<filename>")`
   - Read source with `read_file(resolved_path, line-10, line+10)`
   - `read_file(resolved_path, line-10, line+10)`으로 소스 읽기

2. **Look up message handler (MANDATORY):**
2. **메시지 처리기 조회(필수):**
   - For every ID in "HAS RESOLUTION FILE": **MUST** load `resolution/<file>.md`
   - "해상도 파일 있음"의 모든 ID에 대해: **필수** `solution/<file>.md`를 로드해야 합니다.
with `read_file` BEFORE writing any fix.
수정 사항을 작성하기 전에 `read_file`을 사용하세요.
Follow the guide's patterns exactly.
가이드의 패턴을 정확하게 따르세요.
   - For every ID in "NO RESOLUTION FILE": Use `vivado_doc_search` for UG901 guidance.
   - "NO RESOLUTION FILE"의 모든 ID에 대해: UG901 안내를 위해 `vivado_doc_search`를 사용하세요.
   - After loading each resolution file, mark it `[x]` in the checklist.
   - 각 해상도 파일을 로드한 후 체크리스트에 '[x]'로 표시하세요.
   - **Do NOT proceed to Step 6 if any resolution file remains unloaded.**
   - **해상도 파일이 언로드된 상태로 남아 있으면 6단계로 진행하지 마십시오.**

3. **For Tier 2 — inspect surrounding code:**
3. **Tier 2의 경우 - 주변 코드 검사:**
   - Read module/entity header for port declarations
   - 포트 선언에 대한 모듈/엔티티 헤더 읽기
   - Read signal declarations for type/width information
   - 유형/너비 정보에 대한 신호 선언 읽기
   - Search workspace for matching names (fuzzy match for typos)
   - 일치하는 이름에 대한 작업 공간 검색(오타에 대한 퍼지 일치)

4. **Generate diff-formatted fix:**
4. **차이 형식 수정 생성:**
   ```diff
   - old_code  // <- problem description
   + new_code  // <- fix description
   ```

5. **For Tier 3 — write advisory:**
5. **계층 3의 경우 - 권고 사항 작성:**
   - Explain the issue
   - 문제를 설명하세요.
   - List design options the user should consider
   - 사용자가 고려해야 할 목록 디자인 옵션

**Scaling — Branch Decision:**
**확장 — 지점 결정:**

- **≤50 messages → Standard Flow:** Proceed directly to Step 6.
- **메시지 50개 이하 → 표준 흐름:** 6단계로 바로 진행하세요.
- **>50 messages → Extended Flow:** Execute Steps 5A and 5B first.
- **>50개 메시지 → 확장 흐름:** 먼저 5A 및 5B 단계를 실행합니다.

---

### Extended Flow (>50 messages only)
### 확장 흐름(>50개 메시지만 해당)

#### Step 5A: Generate Per-File Message Parts
#### 5A단계: 파일별 메시지 부분 생성

Split the CSV into per-file message reports:
CSV를 파일별 메시지 보고서로 분할합니다.

```bash
cd vivado_agentic_ai_reports/rtl-elaboration-analysis
python3 <skill_dir>/csv_to_per_file_parts.py messages.csv ./
```

**Generated files:**
**생성된 파일:**
- `messages_by_file.txt` — Summary with error counts + file rankings
- `messages_by_file.txt` — 오류 수 + 파일 순위 요약
- `messages_by_file_part1.txt` — Easiest files (start here)
- `messages_by_file_part1.txt` — 가장 쉬운 파일(여기서 시작)
- `messages_by_file_partN.txt` — Progressive difficulty
- `messages_by_file_partN.txt` — 점진적 난이도

#### Step 5B: Per-File Subagent Analysis
#### 5B단계: 파일별 하위 에이전트 분석

Launch one subagent per part file.
부품 파일당 하나의 하위 에이전트를 실행합니다.
Each subagent processes up to 20 RTL files.
각 하위 에이전트는 최대 20개의 RTL 파일을 처리합니다.

For each file in the part:
부품의 각 파일에 대해 다음을 수행합니다.
1. Parse messages
1. 메시지 구문 분석
2. Locate source file with `file_search`
2. `file_search`로 소스 파일 찾기
3. For each message: read source, look up handler, load resolution file, write diffs
3. 각 메시지에 대해: 소스 읽기, 핸들러 조회, 해결 파일 로드, diff 작성
4. Return per-file analysis markdown
4. 파일별 분석 마크다운 반환

Save outputs to `file_analysis/<filename>_analysis.md`
출력을 `file_analytic/<filename>_analytic.md`에 저장합니다.

---

### Step 6: Write Report
### 6단계: 보고서 작성

**The workflow is incomplete until REPORT.md exists.**
**REPORT.md가 존재할 때까지 워크플로는 불완전합니다.**

Save to: `vivado_agentic_ai_reports/rtl-elaboration-analysis/REPORT.md`
저장 위치: `vivado_agentic_ai_reports/rtl-elaboration-analytic/REPORT.md`

**Report structure:**
**보고서 구조:**

```markdown
# RTL Elaboration Analysis Report

## Summary
| Severity | Count | Actionable | Advisory |
|----------|-------|------------|----------|
| Error | N | N | N |
| Critical Warning | N | N | N |
| Warning | N | N | N |

## File Hotspots
| File | Errors | Crit. Warnings | Warnings | Total |
|------|--------|----------------|----------|-------|

## Message Type Distribution
| ID | Severity | Count | Description | Actionable? |
|----|----------|-------|-------------|-------------|

## Fixes — Errors (Priority 1)
### [Synth 8-XXX] <description>
**CSV S.No:** <serial number(s) from messages.csv>
**File:** [file.v:42](../../path/to/file.v#L42)
**Message:** <full message text>
**Root Cause:** <explanation>

**Problematic Code:**
\```diff
  context line
- problematic line  // <- what's wrong
  context line
\```

**Recommended Fix:**
\```diff
  context line
+ fixed line  // <- what changed and why
  context line
\```

**Corrected Code (paste-ready):**
\```verilog
// Final corrected code only - no diff markers. Paste this over the flagged
// line(s) in <file>. Language tag = verilog / systemverilog / vhdl per the
// CSV Language column.
fixed line
\```

> You can copy the code block above and paste it directly into `<file>` at line(s)
> `<N>` to apply this fix in your RTL.

**Rationale:** <why this fix is correct>

## Fixes — Critical Warnings (Priority 2)
...

## Fixes — Warnings (Priority 3)
...

## Advisory — Design Decisions Required
...

## Recommendations
### Immediate (fix now)
### High-Impact Quick Wins
### Design Review Items
### Accept/Waive
```

**Formatting rules:**
**형식 지정 규칙:**
- Each actionable fix has TWO code blocks: (a) a `diff` block showing the change,
- 실행 가능한 각 수정 사항에는 두 개의 코드 블록이 있습니다: (a) 변경 사항을 보여주는 `diff` 블록,
and (b) a **Corrected Code (paste-ready)** block that contains ONLY the final
(b) 최종 코드만 포함된 **수정된 코드(붙여넣기 가능)** 블록
fixed code with NO diff markers (`+`/`-`/space) and NO `@@` hunk headers, so the
차이점 표시(`+`/`-`/공백)와 `@@` 덩어리 헤더가 없는 고정 코드이므로
user can copy it directly into the RTL.
사용자는 이를 RTL에 직접 복사할 수 있습니다.
- The paste-ready block uses a language fence chosen from the CSV `Language` column:
- 붙여넣기 가능 블록은 CSV '언어' 열에서 선택한 언어 펜스를 사용합니다.
`VLOG` -> ` ```verilog ` (or ` ```systemverilog ` for SV), `VHDL` -> ` ```vhdl `.
`VLOG` -> ` ```verilog `(또는 SV의 경우 ` ```systemverilog `), `VHDL` -> ` `vhdl `.
- The paste-ready block must contain real code taken from the actual RTL with the
- 붙여넣기 가능 블록에는 실제 RTL에서 가져온 실제 코드가 포함되어야 합니다.
fix applied (verify by re-reading the source), plus just enough surrounding
수정 사항이 적용되었습니다(소스를 다시 읽어 확인). 그리고 주변 환경도 충분합니다.
lines to unambiguously locate where it goes.
그것이 어디로 가는지 명확하게 찾아내는 선.
- **MANDATORY**: Immediately after every **Corrected Code (paste-ready)** block, add
- **필수**: 모든 **수정된 코드(붙여넣기 가능)** 블록 바로 뒤에는 다음을 추가하세요.
one explicit line telling the user the code can be copied and pasted into the
사용자에게 코드를 복사하여 붙여넣을 수 있음을 알려주는 명시적인 한 줄
target RTL, naming the file and the line(s), e.g.:
대상 RTL, 파일 및 줄 이름 지정, 예:
`> You can copy the block above and paste it directly into \`<file>\` at line(s) <N> to apply this fix.`
`> 위 블록을 복사하여 \`<파일>\`의 <N> 행에 직접 붙여넣어 이 수정 사항을 적용할 수 있습니다.`
Every recommended fix in every generated report must carry this line.
생성된 모든 보고서의 모든 권장 수정 사항에는 이 줄이 포함되어야 합니다.
- This report only PRESENTS the corrected code; it does NOT modify the user's RTL
- 이 보고서는 수정된 코드만 제시합니다. 사용자의 RTL을 수정하지 않습니다.
source files.
소스 파일.
Applying the paste-ready code is left to the user.
붙여넣기 가능 코드를 적용하는 것은 사용자의 몫입니다.
- **Fences in the generated REPORT.md must be PLAIN** (` ```diff `, ` ```verilog `,
- **생성된 REPORT.md의 펜스는 일반이어야 합니다** (` ```diff `, ` ```verilog `,
` ```vhdl `) with NO backslash escaping.
``vhdl `) 백슬래시 이스케이프가 없습니다.
The `\`\`\`` shown in this skill's template
이 스킬의 템플릿에 표시된 `\`\`\``
is only escaped so the example nests inside this doc; never write `\`` before a
이스케이프만 수행되므로 예제가 이 문서 내에 중첩됩니다. 절대로 ``\``를 앞에 쓰지 마세요.
backtick in the actual report, or the code block will not render.
실제 보고서에 백틱을 추가하지 않으면 코드 블록이 렌더링되지 않습니다.
- All file references are clickable markdown links with line numbers.
- 모든 파일 참조는 줄 번호가 포함된 클릭 가능한 마크다운 링크입니다.
- **MANDATORY — no message dropped**: Every non-INFO message in messages.csv MUST be
- **필수 — 삭제된 메시지 없음**: message.csv에 있는 모든 비 INFO 메시지는 다음과 같아야 합니다.
represented by a `###` section with File link, Problematic Code (diff block),
파일 링크, 문제 코드(diff 블록)가 포함된 `###` 섹션으로 표시됩니다.
Recommended Fix (diff block), and a Corrected Code (paste-ready) block.
권장 수정(diff 블록) 및 수정된 코드(붙여넣기 가능) 블록.
Never
절대로
demote a message to only a Message-Type-Distribution table row or an Advisory
메시지를 Message-Type-Distribution 테이블 행 또는 Advisory로만 수준 내리기
bullet in place of its `###` section.
`###` 섹션 대신 글머리 기호를 사용하세요.
- **One section per distinct diagnostic.** A distinct diagnostic = a unique
- **고유한 진단당 하나의 섹션.** 구별된 진단 = 고유한
(Msg_ID + root cause + location).
(Msg_ID + 근본 원인 + 위치).
Two messages with different IDs, or the same ID
ID가 다르거나 ID가 동일한 두 개의 메시지
at different file:line with different root causes, are distinct and each get their
서로 다른 근본 원인이 있는 서로 다른 파일:라인에서는 서로 구별되며 각각 고유한 결과를 얻습니다.
own section.
자신의 섹션.
- **Grouping identical repeats is allowed and expected for large logs.** When the
- **큰 로그의 경우 동일한 반복을 그룹화하는 것이 허용되며 예상됩니다.**
SAME Msg_ID with the SAME root cause repeats across many instances/signals
동일한 근본 원인이 있는 동일한 Msg_ID가 여러 인스턴스/신호에서 반복됩니다.
(common in real designs and whenever total messages > 50), combine them into ONE
(실제 디자인에서 일반적이며 총 메시지가 50개를 초과할 때마다) 이를 하나로 결합합니다.
`###` section that includes an **Occurrences** table listing every occurrence
모든 발생 항목을 나열하는 **Occurrences** 테이블을 포함하는 `###` 섹션
(S.No + instance/signal/line).
(S.No + 인스턴스/신호/라인).
This keeps the section count sane WITHOUT dropping
이렇게 하면 삭제하지 않고 섹션 수를 정상적으로 유지합니다.
any message — every row must still appear in some section's Occurrences list.
모든 메시지 — 모든 행은 일부 섹션의 발생 목록에 계속 나타나야 합니다.
- This applies to **cascade / duplicate / read-abort** messages too.
- 이는 **연속/중복/읽기-중단** 메시지에도 적용됩니다.
- If a message is
- 메시지가 다음과 같은 경우
a downstream effect of another, still give it a `###` section; it may be brief and
다른 효과의 다운스트림 효과에는 여전히 `###` 섹션을 제공합니다. 짧을 수도 있고
state `Cascade of 8-XXX — resolved by that fix`, but it must exist.
'8-XXX의 연속 — 해당 수정으로 해결됨'이라고 명시하지만 존재해야 합니다.
- A folder or log named after a single "target" ID does NOT reduce the required
- 단일 "대상" ID의 이름을 딴 폴더나 로그는 필수 항목을 줄이지 않습니다.
sections: cover every non-INFO message regardless of the folder name.
섹션: 폴더 이름에 관계없이 INFO가 아닌 모든 메시지를 포함합니다.
- Message section count in REPORT.md must equal non-INFO rows in messages.csv.
- REPORT.md의 메시지 섹션 수는 message.csv의 INFO가 아닌 행과 같아야 합니다.
Verify this count explicitly before finishing (Step 7 check #1).
완료하기 전에 이 개수를 명시적으로 확인하세요(7단계 확인 #1).
If sections are
섹션이 다음과 같은 경우
grouped by unique ID, instead verify that the total occurrences listed across all
고유 ID로 그룹화하는 대신 전체 항목에 걸쳐 총 발생 횟수가 나열되는지 확인하세요.
sections equals the non-INFO row count — every non-INFO row must be accounted for.
섹션은 INFO가 아닌 행 수와 같습니다. INFO가 아닌 모든 행을 고려해야 합니다.

### Step 7: Verify Report Accuracy
### 7단계: 보고서 정확성 확인

1. **Every non-INFO row is accounted for.** Either (a) `###` section count == non-INFO
1. **INFO가 아닌 모든 행이 고려됩니다.** (a) `###` 섹션 수 == INFO가 아닌 행
rows in messages.csv, or (b) when identical repeats are grouped by unique ID, the
message.csv의 행 또는 (b) 동일한 반복이 고유 ID로 그룹화되는 경우
total occurrences listed across all sections == non-INFO rows.
모든 섹션에 걸쳐 나열된 총 발생 수 == INFO가 아닌 행.
If any row is
행이 있는 경우
unaccounted for, a message was dropped — add it before finishing.
설명되지 않은 메시지가 삭제되었습니다. 완료하기 전에 추가하세요.
This is a HARD
이것은 어렵다
STOP; do not report completion while rows are unaccounted for.
중지; 행이 설명되지 않은 동안에는 완료를 보고하지 않습니다.
2. All errors have fix recommendations (Tier 1/2) or advisory notes (Tier 3)
2. 모든 오류에는 수정 권장 사항(Tier 1/2) 또는 권고 사항(Tier 3)이 있습니다.
3. All code in diff blocks is from actual RTL files (verify by re-reading)
3. diff 블록의 모든 코드는 실제 RTL 파일에서 가져온 것입니다(다시 읽어 확인).
4. Every actionable fix has a Corrected Code (paste-ready) block with no diff markers
4. 실행 가능한 모든 수정 사항에는 차이점 표시가 없는 수정된 코드(붙여넣기 가능) 블록이 있습니다.
5. Every Corrected Code (paste-ready) block is followed by the explicit copy-paste
5. 모든 수정된 코드(붙여넣기 가능) 블록 뒤에는 명시적인 복사-붙여넣기가 옵니다.
line naming the target file and line(s)
대상 파일과 라인의 이름을 지정하는 라인
6. Cascade/duplicate/read-abort messages each have their own `###` section (not just
6. 연속/중복/읽기-중단 메시지에는 각각 자체 `###` 섹션이 있습니다.
a table row or Advisory bullet)
테이블 행 또는 권고 글머리 기호)
7. File links point to correct locations
7. 파일 링크가 올바른 위치를 가리킵니다.
8. No fabricated violations or fixes
8. 조작된 위반 사항이나 수정 사항이 없습니다.
9. `messages.csv` (Step 3) and `resolution_checklist.md` (Step 3.5) exist in the
9. `messages.csv`(3단계) 및 `solution_checklist.md`(3.5단계)가
report directory
보고서 디렉토리
10. Every resolution file in the checklist is marked `[x]` (loaded)
10. 체크리스트의 모든 해상도 파일은 `[x]`로 표시됩니다(로드됨).
11. All code fences in REPORT.md are plain (` ```diff `/` ```verilog `/` ```vhdl `)
11. REPORT.md의 모든 코드 펜스는 일반입니다(` ```diff `/` ```verilog `/` ```vhdl `).
with NO backslash before any backtick — grep the report for `\`` and fix any hit.
백틱 앞에 백슬래시를 사용하지 마세요. `\``에 대한 보고서를 grep하고 적중 항목을 수정하세요.

### Step 8: Apply Fixes to RTL
### 8단계: RTL에 수정 사항 적용

**This step comes AFTER the report is written.** The REPORT.md documents the
**이 단계는 보고서가 작성된 후에 수행됩니다.** REPORT.md는 보고서를 문서화합니다.
original problematic code and the recommended fix.
원래 문제가 있는 코드와 권장 수정 사항.
Only after the report is
신고가 완료된 후에야
complete should the fixes be applied to the actual RTL source files.
수정 사항이 실제 RTL 소스 파일에 적용되어야 완료됩니다.

1. For each fix in REPORT.md, apply the recommended change to the source file
1. REPORT.md의 각 수정 사항에 대해 권장 변경 사항을 소스 파일에 적용합니다.
2. Use `replace_string_in_file` to make each edit
2. 각 편집에는 `replace_string_in_file`을 사용하세요.
3. Do NOT modify RTL before Step 6 — the report's diff blocks must show the
3. 6단계 전에는 RTL을 수정하지 마십시오. 보고서의 diff 블록에는
original (unfixed) code with `- ` prefix and the fix with `+ ` prefix
`- ` 접두사가 있는 원본(고정되지 않은) 코드와 `+ ` 접두사가 있는 수정 코드

**Workflow order is critical:**
**워크플로 순서가 중요합니다.**
- Steps 1–5: Analyze log, read original RTL, generate diffs
- 1~5단계: 로그 분석, 원본 RTL 읽기, 차이점 생성
- Step 6: Write REPORT.md with diffs (source is still original)
- 6단계: 차이점이 포함된 REPORT.md 작성(소스는 여전히 원본임)
- Step 7: Verify report
- 7단계: 보고서 확인
- Step 8: Apply fixes to actual RTL files
- 8단계: 실제 RTL 파일에 수정 사항 적용
- Step 9: Re-run synthesis to verify fixes
- 9단계: 합성을 다시 실행하여 수정 사항 확인

## Resolution Policy
## 해결정책

- Fix errors before critical warnings, then warnings.
- 중요한 경고 전에 오류를 수정한 다음 경고를 수정하세요.
- Use `message-handlers.md` and the referenced `resolution/` guide for each ID.
- 각 ID에 대해 `message-handlers.md` 및 참조된 `해결/` 가이드를 사용하세요.
- Read real source code before generating any diff; never fabricate RTL.
- 차이점을 생성하기 전에 실제 소스 코드를 읽으십시오. 절대로 RTL을 조작하지 마십시오.
- Preserve design behavior and flag any change that requires a design decision.
- 설계 동작을 유지하고 설계 결정이 필요한 변경 사항을 표시합니다.

## Directory Assumptions
## 디렉토리 가정

This skill expects these helper files beside this file:
이 기술에는 이 파일 옆에 다음 도우미 파일이 필요합니다.
- `message-handlers.md`
- `메시지 처리기.md`
- `resolution-guide.md`
- `해상도-guide.md`
- `report-format.md`
- `보고서 형식.md`
- `tcl-reference.md`
- `tcl-reference.md`
- `parse_elab_messages.py`
- `parse_elab_messages.py`
- `csv_to_per_file_parts.py`
- `csv_to_per_file_parts.py`
- `resolution/`
- `해상도/`

If any file is missing, continue with best-effort analysis from `message.txt`
파일이 누락된 경우 'message.txt'에서 최선의 분석을 계속하세요.
If any file is missing, continue with best-effort analysis from the log and actual
파일이 누락된 경우 로그 및 실제에서 최선의 분석을 계속합니다.
RTL source, and clearly identify the missing asset in the report.
RTL 소스를 확인하고 보고서에서 누락된 자산을 명확하게 식별합니다.
