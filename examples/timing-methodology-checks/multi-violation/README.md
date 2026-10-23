<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->
# Multi-Violation Timing Methodology
# 다중 위반 타이밍 방법론

A dual-clock design with deliberately incorrect timing constraints, then
의도적으로 잘못된 타이밍 제약을 적용한 듀얼 클럭 설계
analyzed and resolved with the `timing-methodology-checks` skill.
'타이밍-방법론-점검' 스킬로 분석하고 해결했습니다.

The design source in `input/` is complete and self-contained (RTL + XDC + Tcl).
'input/'의 디자인 소스는 완전하고 독립적입니다(RTL + XDC + Tcl).
The skill operates on the **synthesized design** — synthesis must complete
스킬은 **합성된 디자인**에서 작동합니다. 합성이 완료되어야 합니다.
before running methodology checks.
방법론 검사를 실행하기 전에.
It depends on the
그것은에 따라 달라집니다
**`timing-methodology-checks` skill**, which lives at
**`타이밍-방법론-검사` 스킬**
[`skills/timing-methodology-checks/`](../../../skills/timing-methodology-checks/)
[`skills/timing-methodology-checks/`](../../../skills/timing-methodology-checks/)
in this repo.
이 레포에서.

## Goal
## 목표

Trigger multiple timing methodology violations from different categories and
다양한 카테고리에서 여러 타이밍 방법론 위반을 트리거하고
let the skill resolve them through its prioritized GROUP pipeline:
스킬이 우선순위가 지정된 GROUP 파이프라인을 통해 문제를 해결하도록 합니다.

1. **TIMING-18** — Duplicate `create_clock` on the same port (redundant definition).
1. **TIMING-18** — 동일한 포트에 'create_clock'을 중복합니다(중복 정의).
2. **TIMING-17** — `set_false_path` referencing a non-existent clock (typo).
2. **TIMING-17** — 존재하지 않는 시계를 참조하는 `set_false_path`(오타).
3. **Unconstrained clk_b** — Missing primary clock on the second clock port.
3. **Unconstrained clk_b** — 두 번째 클럭 포트에 기본 클럭이 없습니다.
4. **Missing clock groups** — Asynchronous CDC paths without `set_clock_groups`.
4. **누락된 시계 그룹** — `set_clock_groups`가 없는 비동기 CDC 경로.
5. **Missing I/O delays** — Unconstrained input/output ports.
5. **I/O 지연 누락** — 제한되지 않은 입력/출력 포트.

## Skills Used
## 사용된 스킬

- **timing-methodology-checks** — runs `report_methodology`, parses violations,
- **timing-methodology-checks** — `report_methodology` 실행, 위반 분석,
prioritizes by GROUP, traces clocks, proposes XDC fixes (remove redundant
GROUP별로 우선순위를 지정하고, 클럭을 추적하고, XDC 수정 사항을 제안합니다(중복 제거
constraints, add missing clocks, establish clock relationships), and generates
제약 조건, 누락된 시계 추가, 시계 관계 설정) 및 생성
a resolution report.
해결 보고서.
All operations go through Vivado Tcl via `vivado_execute`.
모든 작업은 `vivado_execute`를 통해 Vivado Tcl을 거칩니다.

## Prerequisites
## 전제조건

- **Vivado MCP server** — builds the design and provides the Tcl interface.
- **Vivado MCP 서버** — 설계를 구축하고 Tcl 인터페이스를 제공합니다.
- Vivado 2026.1+ installed.
- 비바도 2026.1+가 설치되었습니다.
- No hardware required (post-synthesis timing analysis only).
- 하드웨어가 필요하지 않습니다(합성 후 타이밍 분석만 해당).

## Starting Point
## 출발점

Input files in `input/`:
`input/`에 입력 파일:
- `src/timing_violation_top.v` — Dual-clock design (clk_a 100 MHz, clk_b 150 MHz)
- `src/timing_violation_top.v` — 듀얼 클록 설계(clk_a 100MHz, clk_b 150MHz)
with pipeline logic in both domains and a 2-stage CDC synchronizer crossing
두 도메인 모두에 파이프라인 논리와 2단계 CDC 동기화 장치 교차
from domain A to domain B.
도메인 A에서 도메인 B로.
- `constraints/timing_broken.xdc` — Intentionally incorrect constraints:
- `constraints/timing_broken.xdc` — 의도적으로 잘못된 제약 조건:
  - Correct `create_clock` for clk_a at 100 MHz
  - 100MHz에서 clk_a에 대한 `create_clock` 수정
  - Duplicate `create_clock` on clk_a with different name (TIMING-18)
  - 다른 이름으로 clk_a에 'create_clock'을 복제합니다(TIMING-18).
  - `set_false_path` to non-existent clock `clk_b_typo` (TIMING-17)
  - 존재하지 않는 시계 `clk_b_typo`에 대한 `set_false_path`(TIMING-17)
  - Missing `create_clock` for clk_b (unconstrained paths)
  - clk_b에 대한 `create_clock` 누락(제약되지 않은 경로)
  - Missing `set_clock_groups` for async CDC (TIMING-6 area)
  - 비동기 CDC에 대한 `set_clock_groups` 누락(TIMING-6 영역)
  - Missing `set_input_delay` / `set_output_delay` (TIMING-9/10)
  - `set_input_delay` / `set_output_delay` 누락(TIMING-9/10)
- `create_project.tcl` — creates the project and adds sources.
- `create_project.tcl` — 프로젝트를 생성하고 소스를 추가합니다.

## How to Run
## 달리는 방법

**Follow the prompts in [`prompt.md`](prompt.md) to run this tutorial end to
**이 튜토리얼을 끝까지 실행하려면 [`prompt.md`](prompt.md)의 프롬프트를 따르세요.
end.** There are two ordered steps:
end.** 순서대로 두 가지 단계가 있습니다.

1. **Build + Synthesize** — `Step 1` sources `input/create_project.tcl` and
1. **빌드 + 합성** — `1단계`는 `input/create_project.tcl`을 소스로 하고
runs synthesis to produce the elaborated netlist.
정교한 넷리스트를 생성하기 위해 합성을 실행합니다.
2. **Resolve violations** — `Step 2` uses the `/timing-methodology-checks`
2. **위반 해결** — `2단계`에서는 `/timing-methodology-checks`를 사용합니다.
skill to run `report_methodology`, identify all violations, prioritize
`report_methodology`를 실행하고 모든 위반 사항을 식별하고 우선 순위를 지정하는 기술
by GROUP, and generate fixes + resolution report.
GROUP별로 수정사항 + 해결 보고서를 생성합니다.

## Expected Behavior
## 예상되는 동작

The `timing-methodology-checks` skill will:
'타이밍 방법론 확인' 기술은 다음과 같습니다.
1. Open the synthesized run (`open_run synth_1`).
1. 합성된 실행(`open_run synth_1`)을 엽니다.
2. Run `report_methodology -json methodology.json`.
2. `report_methodology -json Methodology.json`을 실행합니다.
3. Extract constraints via `write_xdc`.
3. `write_xdc`를 통해 제약 조건을 추출합니다.
4. Identify violations and prioritize by GROUP (GROUP 1 first):
4. 위반 사항을 식별하고 GROUP별로 우선순위를 지정합니다(GROUP 1 먼저).
   - **TIMING-18**: Remove duplicate `create_clock clk_a_dup`.
   - **타이밍-18**: 중복된 `create_clock clk_a_dup`을 제거합니다.
   - **TIMING-17**: Remove or fix `set_false_path` with non-existent clock.
   - **TIMING-17**: 존재하지 않는 시계가 있는 `set_false_path`를 제거하거나 수정합니다.
   - Add missing `create_clock -period 6.667 -name clk_b [get_ports clk_b]`.
   - 누락된 `create_clock -기간 6.667 -name clk_b [get_ports clk_b]`를 추가하세요.
   - Add `set_clock_groups -asynchronous -group clk_a -group clk_b`.
   - 'set_clock_groups -asynchronous -group clk_a -group clk_b'를 추가하세요.
5. Re-run `report_methodology` after fixes to confirm resolution.
5. 해결 방법을 확인하려면 수정 후 'report_methodology'를 다시 실행하세요.
6. Generate `METHODOLOGY_RESOLUTION_REPORT.md` with before-after comparison.
6. 전후 비교를 통해 `METHODOLOGY_RESOLUTION_REPORT.md`를 생성합니다.
