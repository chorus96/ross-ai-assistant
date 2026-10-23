<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Report Formatting Reference
# 보고서 형식 참조

This file defines mandatory formatting rules for the RTL lint markdown report.
이 파일은 RTL 린트 마크다운 보고서에 대한 필수 형식 지정 규칙을 정의합니다.
All generated reports **must** follow these conventions.
생성된 모든 보고서는 **반드시** 다음 규칙을 따라야 합니다.

---

## File Link Format
## 파일 링크 형식

**All file links MUST use workspace-relative paths (NOT absolute paths).**
**모든 파일 링크는 작업공간 상대 경로(절대 경로가 아님)를 사용해야 합니다.**

### Correct (clickable in VS Code):
### 정답(VS Code에서 클릭 가능):
```markdown
[design.sv:42](../../../design.sv#L42)
[module.sv:20](../../src/module.sv#L20)
```

### Wrong (will NOT open in VS Code preview):
### 잘못됨(VS Code 미리보기에서는 열리지 않음):
```markdown
[design.sv:42](/proj/user/project/design.sv#L42)
```

### Path Calculation
### 경로 계산

1. Determine where the markdown report will be saved
1. 마크다운 보고서를 저장할 위치 결정
2. Calculate relative path from report location to RTL source file
2. 보고서 위치에서 RTL 소스 파일까지의 상대 경로 계산
3. Use `../../` to navigate up directory levels, then path to file
3. `../../`를 사용하여 디렉터리 수준을 탐색한 다음 파일 경로를 탐색합니다.
4. Append `#L<line_number>` for line references
4. 줄 참조를 위해 `#L<line_number>`를 추가하세요.

**Example:**
**예:**
```
Workspace:
  project_root/
    src/design.sv                        ← TARGET
    vivado_agentic_ai_reports/
      rtl-lint/
        rtl_lint_report.md               ← REPORT

From report → target:
  ../../src/design.sv  (2 levels up, then down to src/)
```

### TCL Helper — Calculate Relative Paths
### TCL 도우미 — 상대 경로 계산

```tcl
# Execute through vivadoExecute tool
proc relative_path {from to} {
    set from_parts [file split [file dirname $from]]
    set to_parts [file split $to]
    set common 0
    foreach f $from_parts t $to_parts {
        if {$f eq $t} {incr common} else {break}
    }
    set up [expr {[llength $from_parts] - $common}]
    set rel [string repeat "../" $up]
    append rel [join [lrange $to_parts $common end] "/"]
    return $rel
}
```

---

## Diff Syntax Highlighting
## Diff 구문 강조

**Always** use ` ```diff ` for code blocks showing errors and fixes — never `verilog`, `systemverilog`, or `vhdl`.
**항상** 오류와 수정 사항을 표시하는 코드 블록에는 ```diff`를 사용하세요. 절대로 `verilog`, `systemverilog` 또는 `vhdl`을 사용하지 마세요.

| Prefix<br>접두사 | Meaning<br>의미 | Rendered Color<br>렌더링된 색상 |
|--------|---------|---------------|
| `-`    | Problematic / removed line<br>문제가 있거나 제거된 라인 | Red<br>빨간색 |
| `+`    | Corrected / added line<br>수정/추가된 라인 | Green<br>녹색 |
| (none)<br>(없음) | Context line (unchanged)<br>컨텍스트 줄(변경되지 않음) | Default<br>기본값 |

### Inline Comment Rules
### 인라인 댓글 규칙

1. **Every `-` line MUST have an inline comment** explaining what is being removed
1. **모든 `-` 줄에는 제거되는 항목을 설명하는 인라인 주석이 있어야 합니다**
2. **Every `+` line SHOULD have an inline comment** explaining the fix (when not obvious)
2. **모든 `+` 줄에는 수정 사항을 설명하는 인라인 주석이 있어야 합니다**(명확하지 않은 경우)
3. Use language-appropriate comment syntax:
3. 언어에 맞는 주석 구문을 사용하십시오.
   - Verilog/SystemVerilog: `// Comment`
   - Verilog/SystemVerilog: `// 코멘트`
   - VHDL: `-- Comment`
   - VHDL: `-- 주석`

### Verilog Example
### Verilog 예제
```diff
 module example(
-    input [7:0] unused_port,    // Remove unused port from module interface
     output wire data
 );
```

### VHDL Example
### VHDL 예
```diff
 entity top is port (
-a : in std_logic_vector(6 downto 0);  -- Remove unused port 'a' from entity
 b : in std_logic_vector(6 downto 0);
 y1 : out std_logic
 );
```

### Multi-line Removal Example
### 여러 줄 제거 예
```diff
-signal unread_sig : std_logic_vector(0 downto 0);  -- Remove unused signal declaration
 begin
-process(a,b)  -- Remove entire process since signal is unused
-variable unread_var : std_logic_vector(0 downto 0);  -- Remove this line
-begin  -- Remove this line
-unread_var(0 downto 0) := b(0 downto 0);  -- Remove this line
-unread_sig(0 downto 0) <= unread_var(0 downto 0);  -- Remove this line
-end process;  -- Remove this line
```

---

## Per-Violation Section Template
## 위반별 섹션 템플릿

Use this template for **each** violation in the report.
보고서의 **각** 위반에 대해 이 템플릿을 사용하세요.

**Source code requirement:** The "Problematic Code" and "Recommended Fix" blocks
**소스 코드 요구 사항:** "문제 코드" 및 "권장 수정" 블록
**must** contain real source code read from the original RTL file at the violation's
**반드시** 위반 시 원본 RTL 파일에서 읽은 실제 소스 코드를 포함해야 합니다.
file:line (±5 lines context).
파일:라인(±5라인 컨텍스트).
Never fabricate, paraphrase, or use placeholder code.
자리 표시자 코드를 조작하거나, 다른 말로 표현하거나, 사용하지 마십시오.
If the source file is unreadable, note this explicitly and fall back to the Vivado
소스 파일을 읽을 수 없는 경우 이를 명시적으로 기록하고 Vivado로 대체합니다.
message text.
메시지 텍스트.

```markdown
### [Issue Type] — [Message ID]

**Location**: [file.sv:line](relative/path/to/file.sv#Lline)

**Issue Description**: [Clear explanation from linter message]

**Problematic Code** *(from original source)*:
` ` `diff
 // Context lines from source file (unchanged)
- [actual line with error from source]    // ERROR: description
 // More context from source file
` ` `

**Recommended Fix**:
` ` `diff
 // Context lines (unchanged)
+ [corrected line]     // FIXED: description
 // More context
` ` `

**Rationale**: [Why this fix works, cite UG901 sections]

**Vivado Documentation**: [UG901/UG906 reference via vivado_doc_search]
```

---

## Report Output Location
## 보고서 출력 위치

```
vivado_agentic_ai_reports/
└── rtl-lint/
    ├── lint_report.rpt          ← Raw Vivado linter output
    └── rtl_lint_report.md       ← AI-generated analysis with fixes
```

Report directory path (TCL):
보고서 디렉터리 경로(TCL):
```tcl
set report_dir "${project_dir}/vivado_agentic_ai_reports/rtl-lint"
file mkdir $report_dir
set lint_report_file "${report_dir}/lint_report.rpt"
set markdown_report_file "${report_dir}/rtl_lint_report.md"
```
