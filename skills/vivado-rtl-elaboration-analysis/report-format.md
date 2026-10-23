<!-- ko-translation: enabled -->
# Report Format — RTL Elaboration Analysis
# 보고서 형식 - RTL 정교화 분석

## Output Files
## 출력 파일

The skill produces these files in the report directory:
이 기술은 보고서 디렉터리에 다음 파일을 생성합니다.

| File<br>파일 | Purpose<br>목적 |
|------|---------|
| `messages.csv`<br>`messages.csv` | Parsed messages: S.No, severity, msg_id, text, file, line, language (with header row)<br>구문 분석된 메시지: S.No, 심각도, msg_id, 텍스트, 파일, 줄, 언어(헤더 행 포함) |
| `report_data.json`<br>`보고서_데이터.json` | Structured analysis results for machine consumption<br>기계 소비에 대한 구조적 분석 결과 |
| `REPORT.md`<br>`보고서.md` | Human-readable report with fixes<br>수정 사항이 포함된 사람이 읽을 수 있는 보고서 |

---

## REPORT.md Structure
## REPORT.md 구조

```markdown
# RTL Elaboration Analysis Report

**Design**: <top_module>
**Log file**: <path_to_log>
**Messages found**: <total> (E:<errors> CW:<crit_warnings> W:<warnings>)
**Actionable**: <count> | **Advisory**: <count>

## Summary

| Severity | Count | Actionable |
|----------|-------|------------|
| ERROR | N | N |
| CRITICAL WARNING | N | N |
| WARNING | N | N |

## Errors (Tier 1 — Immediate Fixes)

### [Synth 8-128] 'my_signal' is not declared
**CSV S.No**: 1
**File**: [path/to/file.v](path/to/file.v#L42)
**Cause**: Signal used but never declared.
**Fix**:
\```diff
--- a/path/to/file.v
+++ b/path/to/file.v
@@ -41,2 +41,3 @@
 // existing code context
+wire [7:0] my_signal;
 assign out = my_signal;
\```
**Corrected code (paste-ready)** — copy into `path/to/file.v` around line 41:
\```verilog
// existing code context
wire [7:0] my_signal;
assign out = my_signal;
\```

### [Synth 8-XXX] ...
...

## Critical Warnings (Tier 1)

...

## Warnings (Tier 2 — Design Improvements)

### [Synth 8-566] inferring latch for variable 'state_reg'
**CSV S.No**: 5
**File**: [path/to/fsm.sv](path/to/fsm.sv#L100)
**Cause**: Incomplete case/if coverage in combinational always block.
**Fix**:
\```diff
--- a/path/to/fsm.sv
+++ b/path/to/fsm.sv
@@ -98,4 +98,5 @@
 always_comb begin
     case (state)
         IDLE: next_state = RUN;
+        default: next_state = IDLE;
     endcase
\```
**Corrected code (paste-ready)** — copy into `path/to/fsm.sv` around line 98:
\```systemverilog
always_comb begin
    case (state)
        IDLE: next_state = RUN;
        default: next_state = IDLE;
    endcase
\```

## Advisory (Tier 3 — No Code Fix Needed)

| ID | Message | File | Line |
|----|---------|------|------|
| 564 | referenced signal 'clk' should be ... | ctrl.v | 200 |

## Cascading Errors

The following errors may be caused by earlier failures:

- [Synth 8-402] `failed synthesizing module 'sub_mod'` — likely caused by errors above in sub_mod
```

---

## Diff Format Rules
## 차이점 형식 규칙

1. Use unified diff format with `--- a/` and `+++ b/` headers
1. `--- a/` 및 `+++ b/` 헤더와 함께 통합 diff 형식을 사용합니다.
2. Use **workspace-relative paths** in diff headers (not absolute)
2. diff 헤더에 **작업 공간 상대 경로** 사용(절대 아님)
3. Include 2-3 lines of surrounding context
3. 주변 맥락을 2~3줄 포함하세요.
4. Use `@@` hunk headers with approximate line numbers
4. 대략적인 줄 번호와 함께 `@@` 덩어리 헤더를 사용하세요.
5. Mark additions with `+`, deletions with `-`, context with space
5. 추가사항은 '+'로, 삭제사항은 '-'로, 컨텍스트는 공백으로 표시하세요.
6. Wrap diffs in ` ```diff ` fenced code blocks
6. ``diff ` 울타리 코드 블록으로 diff를 래핑합니다.

---

## Paste-Ready Fix Rules
## 붙여넣기 준비 수정 규칙

Every actionable fix includes, **in addition to** the diff, a **Corrected code
실행 가능한 모든 수정 사항에는 **차이점 외에도** **수정된 코드가 포함되어 있습니다.
(paste-ready)** block so the user can copy the code directly into their RTL.
(붙여넣기 가능)** 사용자가 코드를 RTL에 직접 복사할 수 있도록 차단합니다.

1. The block contains ONLY the final corrected code — no `+`/`-`/space prefixes,
1. 블록에는 최종 수정된 코드만 포함됩니다. `+`/`-`/공백 접두어는 없습니다.
no `--- a/` `+++ b/` headers, and no `@@` hunk lines.
`--- a/` `++++ b/` 헤더도 없고 `@@` 덩어리 라인도 없습니다.
2. Choose the fence language from the CSV `Language` column:
2. CSV '언어' 열에서 펜스 언어를 선택하세요.
`VLOG` -> ` ```verilog ` (or ` ```systemverilog ` for SystemVerilog),
`VLOG` -> ` ```verilog `(또는 SystemVerilog의 경우 ` ```systemverilog `),
`VHDL` -> ` ```vhdl `.
`VHDL` -> ` ```vhdl `.
3. Include just enough surrounding lines to locate where the code goes, and add a
3. 코드가 어디로 가는지 찾을 수 있을 만큼만 주변 줄을 포함하고
short lead-in noting the file and approximate line (e.g. "copy into `file.v`
파일과 대략적인 줄을 나타내는 짧은 도입부(예: 'file.v'에 복사)
around line 41").
라인 41" 주변).
4. The code must be the actual corrected RTL (verify by re-reading the source).
4. 코드는 실제로 수정된 RTL이어야 합니다(소스를 다시 읽어 확인).
5. This block only PRESENTS the code; the skill does NOT modify the RTL source
5. 이 블록은 코드만 표시합니다. 기술은 RTL 소스를 수정하지 않습니다.
files.
파일.
Applying the change is the user's action.
변경 사항을 적용하는 것은 사용자의 작업입니다.

---

## File Link Format
## 파일 링크 형식

The CSV `File` column contains paths **relative to the report directory** (the
CSV '파일' 열에는 **보고서 디렉터리에 상대적인** 경로가 포함되어 있습니다(
parser computes this automatically).
파서는 이것을 자동으로 계산합니다).
Use them directly as the link target:
링크 대상으로 직접 사용하십시오.

`[filename.v](CSV_File_value#L42)`
`[파일 이름.v](CSV_파일_값#L42)`

Example: CSV has `../../../memstream.sv` → link: `[memstream.sv](../../../memstream.sv#L73)`
예: CSV에는 `../../../memstream.sv`가 있습니다. → 링크: `[memstream.sv](../../../memstream.sv#L73)`

- Use the CSV path as-is — no extra computation needed
- CSV 경로를 있는 그대로 사용하세요. 추가 계산이 필요하지 않습니다.
- Never use absolute paths in links
- 링크에 절대 경로를 사용하지 마세요
- Display text = just the filename for readability
- 표시 텍스트 = 가독성을 위한 파일 이름

---

## report_data.json Schema
## report_data.json 스키마

```json
{
  "design": "<top_module>",
  "log_file": "<path>",
  "timestamp": "<ISO-8601>",
  "summary": {
    "total": 15,
    "errors": 3,
    "critical_warnings": 2,
    "warnings": 10,
    "actionable": 12,
    "advisory": 3
  },
  "messages": [
    {
      "severity": "ERROR",
      "msg_id": 128,
      "text": "'my_signal' is not declared",
      "file": "src/top.v",
      "line": 42,
      "language": "VLOG",
      "tier": 1,
      "category": "undeclared_identifier",
      "fix_applied": true,
      "fix_description": "Added wire declaration",
      "cascading": false
    }
  ]
}
```

---

## Grouping and Ordering
## 그룹화 및 정렬

1. Group by severity: ERROR → CRITICAL WARNING → WARNING
1. 심각도별로 그룹화: ERROR → CRITICAL WARNING → WARNING
2. Within each group, sort by tier (1 first, then 2, then 3)
2. 각 그룹 내에서 계층별로 정렬합니다(1 먼저, 2, 3 순).
3. Within each tier, sort by file path then line number
3. 각 계층 내에서 파일 경로를 기준으로 정렬한 다음 줄 번호를 기준으로 정렬합니다.
4. Tier 3 messages go in a summary table, not individual sections
4. Tier 3 메시지는 개별 섹션이 아닌 요약 표에 들어갑니다.
5. Cascading errors go in a separate section at the end
5. 계단식 오류는 마지막에 별도의 섹션에 표시됩니다.
