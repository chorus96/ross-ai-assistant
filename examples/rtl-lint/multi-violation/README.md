<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->
# RTL Lint Multi-Violation
# RTL 린트 다중 위반

A single Verilog module with intentional RTL lint violations across multiple
여러 시스템에 걸쳐 의도적인 RTL Lint 위반이 있는 단일 Verilog 모듈
categories, then analyzed with the `rtl-lint` skill.
그런 다음 `rtl-lint` 스킬로 분석합니다.

The design in `input/` is a single RTL file — no Vivado project required.
`input/`의 디자인은 단일 RTL 파일이므로 Vivado 프로젝트가 필요하지 않습니다.
The
는
skill runs `synth_design -lint` directly on the source.
스킬은 소스에서 직접 `synth_design -lint`를 실행합니다.
It depends on the
그것은에 따라 달라집니다
**`rtl-lint` skill**, which lives at [`skills/rtl-lint/`](../../../skills/rtl-lint/)
**`rtl-lint` 스킬**, [`skills/rtl-lint/`](../../../skills/rtl-lint/)에 위치
in this repo.
이 레포에서.

## Goal
## 목표

Trigger at least 8 distinct lint rule violations from different categories and
다양한 카테고리에서 최소 8개의 개별 린트 규칙 위반을 트리거하고
let the skill detect, report, and propose fixes for each one:
기술이 각 항목에 대한 수정 사항을 감지, 보고 및 제안하도록 합니다.

1. **ASSIGN-1** — Arithmetic overflow (result wider than output)
1. **ASSIGN-1** — 산술 오버플로(결과가 출력보다 넓음)
2. **ASSIGN-2** — Mixed signed/unsigned arithmetic
2. **ASSIGN-2** — 부호 있는/부호 없는 혼합 산술
3. **ASSIGN-3** — Shift amount exceeds signal width
3. **ASSIGN-3** — 이동량이 신호 폭을 초과합니다.
4. **ASSIGN-5** — Signal used but never assigned
4. **ASSIGN-5** — 신호가 사용되었지만 할당되지 않았습니다.
5. **ASSIGN-6** — Signal assigned but never read (dead code)
5. **ASSIGN-6** — 신호가 할당되었으나 읽히지 않음(데드 코드)
6. **ASSIGN-14** — Duplicate case branches
6. **ASSIGN-14** — 중복된 사례 분기
7. **INFER-1** — Latch inferred (missing else in combinational block)
7. **INFER-1** — 추론된 래치(조합 블록에서 다른 항목이 누락됨)
8. **INFER-2** — Incomplete case statement (no default)
8. **INFER-2** — 불완전한 사례 설명(기본값 없음)
9. **CLOCK-1** — Mixed clock edges (posedge + negedge same clock)
9. **CLOCK-1** — 혼합 클록 에지(posedge + negedge 동일한 클록)

## Skills Used
## 사용된 스킬

- **rtl-lint** — loads the RTL file, runs `synth_design -lint`, parses the
- **rtl-lint** — RTL 파일을 로드하고, `synth_design -lint`를 실행하고,
violation report, and generates a markdown report with per-violation analysis
위반 보고서 및 위반별 분석을 통해 마크다운 보고서 생성
including source code context, root cause, and proposed fixes with diffs.
소스 코드 컨텍스트, 근본 원인 및 제안된 차이점 수정 사항을 포함합니다.
All operations go through Vivado Tcl via `vivado_execute`.
모든 작업은 `vivado_execute`를 통해 Vivado Tcl을 거칩니다.

## Prerequisites
## 전제조건

- **Vivado MCP server** — provides the Tcl interface for `synth_design -lint`.
- **Vivado MCP 서버** — `synth_design -lint`에 대한 Tcl 인터페이스를 제공합니다.
- Vivado 2026.1+ installed.
- 비바도 2026.1+가 설치되었습니다.
- No hardware or synthesis required — lint is a pre-synthesis check.
- 하드웨어나 합성이 필요하지 않습니다. 린트는 합성 전 검사입니다.

## Starting Point
## 출발점

Input files in `input/`:
`input/`에 입력 파일:
- `src/lint_violation_top.v` — Single Verilog module with 10 intentional bugs:
- `src/lint_violation_top.v` — 10개의 의도적인 버그가 있는 단일 Verilog 모듈:
arithmetic overflow, mixed signedness, excessive shift, undriven signals,
산술 오버플로, 혼합 부호, 과도한 이동, 구동되지 않는 신호,
dead code, duplicate case items, latches, incomplete case, mixed clock edges,
데드 코드, 중복된 케이스 항목, 래치, 불완전한 케이스, 혼합된 클럭 에지,
and incomplete reset coverage.
불완전한 재설정 적용 범위.

## How to Run
## 달리는 방법

**Follow the prompts in [`prompt.md`](prompt.md).** This is a single-step
**[`prompt.md`](prompt.md)의 프롬프트를 따르세요.** 이는 단일 단계입니다.
tutorial:
튜토리얼:

1. **Lint** — The skill loads the RTL, runs `synth_design -lint`, parses the
1. **Lint** — 스킬은 RTL을 로드하고, `synth_design -lint`를 실행하고,
report, and generates a per-violation analysis with fix proposals.
수정 제안과 함께 위반별 분석을 보고하고 생성합니다.

## Expected Behavior
## 예상되는 동작

The `rtl-lint` skill will:
`rtl-lint` 스킬은 다음과 같습니다.
1. Load the Verilog file via `read_verilog`.
1. `read_verilog`를 통해 Verilog 파일을 로드합니다.
2. Run `synth_design -top lint_violation_top -part xcvc1902-vsva2197-2MP-e-S -lint -file lint_report.rpt`.
2. `synth_design -top lint_violation_top -part xcvc1902-vsva2197-2MP-e-S -lint -file lint_report.rpt`를 실행합니다.
3. Parse the report (11 violations across 8 rule IDs).
3. 보고서를 구문 분석합니다(8개 규칙 ID에 걸쳐 11개 위반).
4. For each violation, read the source code context, identify root cause,
4. 각 위반에 대해 소스 코드 컨텍스트를 읽고 근본 원인을 식별하고
and propose a fix with diff.
diff를 사용하여 수정 사항을 제안합니다.
5. Generate `rtl_lint_report.md` with summary table and per-violation sections.
5. 요약 테이블과 위반별 섹션이 포함된 `rtl_lint_report.md`를 생성합니다.

### Verified Violations (Vivado 2026.1)
### 확인된 위반 사항(Vivado 2026.1)

| Rule ID<br>규칙 ID | Count<br>개수 | Severity<br>심각도 | Description<br>설명 |
|---------|-------|----------|-------------|
| ASSIGN-1<br>할당-1 | 1 | WARNING<br>경고 | Arithmetic overflow at line 137<br>137행의 산술 오버플로 |
| ASSIGN-2<br>할당-2 | 1 | CRITICAL<br>심각 | Mixed signedness at line 45<br>45행의 혼합 부호 |
| ASSIGN-3<br>할당-3 | 1 | CRITICAL<br>심각 | Shift overflow at line 53<br>53번째 줄의 Shift 오버플로 |
| ASSIGN-5<br>할당-5 | 1 | WARNING<br>경고 | Undriven phantom_sig at line 59<br>59행의 구동되지 않은 phantom_sig |
| ASSIGN-6<br>할당-6 | 2 | WARNING<br>경고 | Dead code (dead_signal, mixed_result bit 8)<br>데드 코드(dead_signal, Mixed_result 비트 8) |
| ASSIGN-14<br>할당-14 | 1 | CRITICAL<br>심각 | Duplicate case branch at line 75<br>75행의 중복 사례 분기 |
| INFER-1<br>추론-1 | 2 | CRITICAL<br>심각 | Latches (mux_out, sel_out) at lines 90, 101<br>라인 90, 101의 래치(mux_out, sel_out) |
| INFER-2<br>추론-2 | 1 | CRITICAL<br>심각 | Incomplete case at line 100<br>100행의 불완전한 사례 |
| CLOCK-1<br>시계-1 | 1 | CRITICAL<br>심각 | Mixed clock edges at line 127<br>라인 127에서 혼합된 클럭 에지 |

Total: 11 violations, 6 CRITICAL + 5 WARNING.
총계: 위반 11개, 심각 6개 + 경고 5개.
