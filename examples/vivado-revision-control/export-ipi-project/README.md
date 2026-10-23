<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->
# Export Mixed-Source IPI Project
# 혼합 소스 IPI 프로젝트 내보내기

A mixed-source Vivado project (RTL + standalone XCI IP + IPI Block Design +
혼합 소스 Vivado 프로젝트(RTL + 독립형 XCI IP + IPI 블록 디자인 +
XDC constraints) exported for version control using the
XDC 제약 조건)을 사용하여 버전 제어를 위해 내보냈습니다.
`vivado-revision-control` skill.
'vivado-revision-control' 스킬.

The design source in `input/` is complete and self-contained (RTL + Tcl).
`input/`의 디자인 소스는 완전하고 독립적입니다(RTL + Tcl).
The
는
skill operates on the *open project* — no synthesis or implementation required.
기술은 *공개 프로젝트*에서 작동합니다. 합성이나 구현이 필요하지 않습니다.
It depends on the **`vivado-revision-control` skill**, which lives at
에 살고 있는 **`vivado-revision-control` 스킬**에 달려 있습니다.
[`skills/vivado-revision-control/`](../../../skills/vivado-revision-control/)
[`skills/vivado-revision-control/`](../../../skills/vivado-revision-control/)
in this repo.
이 레포에서.

## Goal
## 목표

Demonstrate the full 5-step revision-control pipeline on a project that
프로젝트에서 전체 5단계 개정 제어 파이프라인을 시연합니다.
contains all major source types:
모든 주요 소스 유형을 포함합니다.

1. **Detect** project flow type (Standard RTL + IPI).
1. **감지** 프로젝트 흐름 유형(표준 RTL + IPI).
2. **Analyze** source locations — `top.v` and `timing.xdc` are added from
2. **분석** 소스 위치 - `top.v` 및 `timing.xdc`가 다음에서 추가됩니다.
`input/src/` and `input/constraints/`, which sit *outside* the project
프로젝트 *외부*에 있는 `input/src/` 및 `input/constraints/`
directory (`input/revctrl_demo/`), so the scenario is **Mixed**: they stay
디렉터리(`input/revctrl_demo/`)이므로 시나리오는 **혼합**입니다.
remote, while the IP and BD (generated inside the project's own tree) are
IP 및 BD(프로젝트 자체 트리 내부에 생성됨)는 원격입니다.
local.
지역.
3. **Export** IP via `write_ip_tcl` to `Sources/IP/` and BD via
3. **내보내기** IP를 `write_ip_tcl`을 통해 `Sources/IP/`로, BD를 통해
`write_bd_tcl` to `Sources/BD/`.
`write_bd_tcl`을 `소스/BD/`에 추가합니다.
`top.v` and `timing.xdc` are *not*
`top.v`와 `timing.xdc`는 *아닙니다*
copied — see [Imported vs.
복사됨 - [가져온 것과 가져온 것 비교]를 참조하세요.
Added Files](#imported-vs-added-files) below.
추가된 파일](#imported-vs-add-files) 아래.
4. **Capture** non-default project settings (PART, BOARD_PART, TOP, etc.).
4. **캡처** 기본이 아닌 프로젝트 설정(PART, BOARD_PART, TOP 등).
5. **Generate** `build.tcl` that recreates the project from scratch.
5. **생성** 프로젝트를 처음부터 다시 생성하는 `build.tcl`을 생성합니다.

### Imported vs.
### 수입품 vs.
### Added Files
### 추가된 파일

The skill distinguishes two ways a source can end up back in the recreated
이 기술은 소스가 다시 생성될 수 있는 두 가지 방법을 구별합니다.
project, based on where it lived in the *original* project:
*원본* 프로젝트의 위치를 기준으로 한 프로젝트:

- **Imported (local)** — the file lives inside the project directory.
- **가져옴(로컬)** — 파일은 프로젝트 디렉터리 내에 있습니다.
- It's
- 그것은
physically copied into `Sources/<Category>/` during export, and `build.tcl`
내보내는 동안 `Sources/<Category>/`에 물리적으로 복사되고 `build.tcl`
brings it back with `import_files`.
`import_files`로 다시 가져옵니다.
- **Added (remote)** — the file lives outside the project directory (as
- **추가됨(원격)** — 파일이 프로젝트 디렉터리 외부에 있습니다(예:
`top.v` and `timing.xdc` do here, since `input/revctrl_demo/` is a
`top.v`와 `timing.xdc`는 여기서 수행됩니다. `input/revctrl_demo/`는
subdirectory of `input/`, not the other way around).
`input/`의 하위 디렉토리에 있어야 하며 그 반대는 아닙니다).
It's never copied;
결코 복사되지 않습니다.
`build.tcl` references it in place at its original absolute path with
`build.tcl`은 원래 절대 경로에서 이를 참조합니다.
`add_files`.
`추가_파일`.

A project is classified **Push Button** if every source is local, **Remote**
모든 소스가 로컬인 경우 프로젝트는 **푸시 버튼**, **원격**으로 분류됩니다.
if every source is remote, and **Mixed** if it's a combination — as it is in
모든 소스가 원격인 경우 **혼합** 조합인 경우 — 그대로
this example.
이 예.

## Skills Used
## 사용된 스킬

- **vivado-revision-control** — detects project type, exports all sources,
- **vivado-revision-control** — 프로젝트 유형을 감지하고 모든 소스를 내보냅니다.
captures settings, and generates a portable build script.
설정을 캡처하고 이식 가능한 빌드 스크립트를 생성합니다.
All operations
모든 작업
go through Vivado Tcl procedures via `vivado_execute`.
`vivado_execute`를 통해 Vivado Tcl 절차를 진행합니다.

## Prerequisites
## 전제조건

- **Vivado MCP server** — provides the Tcl interface to the open project.
- **Vivado MCP 서버** — 열려 있는 프로젝트에 Tcl 인터페이스를 제공합니다.
- Vivado 2026.1+ installed.
- 비바도 2026.1+가 설치되었습니다.
- No hardware required (this is a project-management skill, not HW debug).
- 하드웨어가 필요하지 않습니다(이것은 HW 디버그가 아닌 프로젝트 관리 기술입니다).

## Starting Point
## 출발점

Input files in `input/`:
`input/`에 입력 파일:
- `src/top.v` — RTL top module that instantiates a Clocking Wizard (XCI)
- `src/top.v` — XCI(클러킹 마법사)를 인스턴스화하는 RTL 최상위 모듈
and a Block Design wrapper.
및 블록 디자인 래퍼.
Contains a 32-bit counter on the 200 MHz domain.
200MHz 도메인에 32비트 카운터가 포함되어 있습니다.
- `constraints/timing.xdc` — clock constraints + output delay + false path.
- `constraints/timing.xdc` — 클럭 제약 + 출력 지연 + 잘못된 경로.
- `create_project.tcl` — creates the project with:
- `create_project.tcl` — 다음을 사용하여 프로젝트를 생성합니다.
  - RTL source (`top.v`)
  - RTL 소스(`top.v`)
  - Standalone XCI IP (`clk_wiz_0`: 100 MHz → 200 MHz)
  - 독립형 XCI IP(`clk_wiz_0`: 100MHz → 200MHz)
  - IPI Block Design (`bd_subsystem`: CIPS providing pl0_ref_clk + pl0_resetn)
  - IPI 블록 설계(`bd_subsystem`: pl0_ref_clk + pl0_resetn을 제공하는 CIPS)
  - Constraint file (`timing.xdc`)
  - 제약조건 파일(`timing.xdc`)

## How to Run
## 달리는 방법

**Follow the prompts in [`prompt.md`](prompt.md) to run this tutorial end to
**이 튜토리얼을 끝까지 실행하려면 [`prompt.md`](prompt.md)의 프롬프트를 따르세요.
end.** There are three ordered steps:
end.** 순서대로 세 가지 단계가 있습니다.

1. **Create project** — `Step 1` sources `input/create_project.tcl`.
1. **프로젝트 생성** — `1단계`는 `input/create_project.tcl`을 소스로 합니다.
2. **Export** — `Step 2` uses the `/vivado-revision-control` skill to export
2. **내보내기** — `2단계`에서는 `/vivado-revision-control` 기술을 사용하여 내보냅니다.
all sources and generate `build.tcl`.
모든 소스를 확인하고 `build.tcl`을 생성합니다.
3. **Verify** — `Step 3` runs the generated `build.tcl` to confirm the
3. **확인** — `3단계`에서는 생성된 `build.tcl`을 실행하여
project recreates correctly.
프로젝트가 올바르게 다시 생성됩니다.

## Expected Behavior
## 예상되는 동작

The `vivado-revision-control` skill will:
'vivado-revision-control' 스킬은 다음과 같습니다:
1. Detect the project as "Standard" flow type, "Mixed" scenario (see
1. 프로젝트를 "표준" 흐름 유형, "혼합" 시나리오로 감지합니다(참조
[Imported vs.
[수입 vs.
Added Files](#imported-vs-added-files)).
추가된 파일](#가져온-vs-추가-파일)).
2. Export local sources to a `RevisionControl/` directory structure:
2. 로컬 소스를 'RevisionControl/' 디렉터리 구조로 내보냅니다.
   - `Sources/IP/clk_wiz_0.tcl` (via `write_ip_tcl`)
   - `소스/IP/clk_wiz_0.tcl`(`write_ip_tcl`을 통해)
   - `Sources/BD/bd_subsystem.tcl` (via `write_bd_tcl`)
   - `Sources/BD/bd_subsystem.tcl`(`write_bd_tcl`을 통해)
   - `top.v` and `timing.xdc` are **not** exported here — they remain at
   - `top.v` 및 `timing.xdc`는 여기로 내보내지지 **않습니다** — 그대로 유지됩니다
their original location under `input/`, and `build.tcl` adds them back
`input/` 아래의 원래 위치 및 `build.tcl`이 이를 다시 추가합니다.
in place with `add_files` pointing at that absolute path.
해당 절대 경로를 가리키는 `add_files`가 있는 위치에 있습니다.
3. Capture project settings (PART, BOARD_PART, TOP) to `Scripts/project_settings.tcl`.
3. 프로젝트 설정(PART, BOARD_PART, TOP)을 `Scripts/project_settings.tcl`에 캡처합니다.
4. Generate `Scripts/build.tcl` — a single-command project recreation script.
4. 단일 명령 프로젝트 재생 스크립트인 `Scripts/build.tcl`을 생성합니다.
5. Verification: running `build.tcl` produces a project with the same TOP,
5. 확인: `build.tcl`을 실행하면 동일한 TOP가 있는 프로젝트가 생성됩니다.
IP count, and BD as the original.
IP 개수, BD가 원본입니다.
