<!-- ko-translation: enabled -->
---
name: vivado-revision-control
description: >
  Comprehensive Vivado project revision control strategies for standard, DFX
  (Partial Reconfiguration), IPI Block Design Container, and Segmented
  Configuration projects. Includes automated tools for project type detection,
  source analysis, settings capture, and build script generation. Use this skill
  whenever the user mentions version control, Git, build scripts, project
  portability, team collaboration, CI/CD for Vivado, exporting sources, preparing
  a project for handoff, or recreating projects — even if they don't explicitly
  say "revision control". Also trigger for "make my project portable",
  "automate project recreation", or "set up Git for my FPGA design".
license: MIT
metadata:
  version: "1.0.0"
---

<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->
# Vivado Revision Control Skill
# 비바도 개정 제어 기술

This skill automates revision control setup for Vivado projects through a
이 기술은 다음을 통해 Vivado 프로젝트에 대한 개정 제어 설정을 자동화합니다.
5-step pipeline, plus an optional interactive Step 2.5 for preserving
5단계 파이프라인과 보존을 위한 선택적인 대화형 2.5단계
existing generated outputs.
기존에 생성된 출력.
It works with Standard RTL-only projects, IPI
표준 RTL 전용 프로젝트인 IPI와 함께 작동합니다.
Block Designs (including Block Design Containers), DFX (Partial
블록 디자인(블록 디자인 컨테이너 포함), DFX(부분
Reconfiguration), and Versal Segmented Configuration projects.
재구성) 및 Versal 세그먼트 구성 프로젝트.

The pipeline detects project type, analyzes source locations, exports
파이프라인은 프로젝트 유형을 감지하고, 소스 위치를 분석하고, 내보냅니다.
components, captures settings, and generates a build script — producing a
구성 요소를 캡처하고, 설정을 캡처하고, 빌드 스크립트를 생성합니다.
self-contained directory that can recreate the project from scratch.
프로젝트를 처음부터 다시 만들 수 있는 자체 포함된 디렉터리입니다.

## When to use this skill
## 이 스킬을 언제 사용해야 할까요?

- Setting up version control for any Vivado project
- 모든 Vivado 프로젝트에 대한 버전 제어 설정
- Making a project portable for team members or CI/CD
- 팀 구성원 또는 CI/CD를 위해 프로젝트를 이식 가능하게 만들기
- Exporting and organizing all project sources
- 모든 프로젝트 소스 내보내기 및 구성
- Creating automated build scripts for reproducible builds
- 재현 가능한 빌드를 위한 자동화된 빌드 스크립트 생성
- Handling DFX, BDC, or Segmented Config revision control
- DFX, BDC 또는 세그먼트 구성 개정 제어 처리

## When NOT to use this skill
## 이 스킬을 사용하지 말아야 할 경우

- Project is not yet created (no .xpr file exists)
- 프로젝트가 아직 생성되지 않았습니다(.xpr 파일이 없음).
- Only need to modify an existing build script (edit it directly)
- 기존 빌드 스크립트만 수정하면 됩니다(직접 편집).
- Need timing analysis or design debugging (use baselining or other skills)
- 타이밍 분석 또는 설계 디버깅이 필요함(기준 설정 또는 기타 기술 사용)

## Prerequisites
## 전제조건

Before running, the Vivado project must be open with:
실행하기 전에, Vivado 프로젝트는 다음과 같이 열려 있어야 합니다:
- A valid PART property set
- 유효한 PART 속성 집합
- Source files added to the project
- 프로젝트에 추가된 소스 파일
- TOP module configured
- TOP 모듈 구성됨
- For DFX: PR_FLOW property enabled
- DFX의 경우: PR_FLOW 속성이 활성화되었습니다.
- For Segmented Config: SEGMENTED_CONFIGURATION property set (or Versal Gen2 device)
- 분할된 구성의 경우: SEGMENTED_CONFIGURATION 속성 집합(또는 Versal Gen2 장치)

## 5-Step Pipeline (+ optional Step 2.5)
## 5단계 파이프라인(+ 2.5단계 옵션)

All Tcl procedures live in `helper-procedures/helper_scripts.tcl`.
모든 Tcl 프로시저는 `helper-procedures/helper_scripts.tcl`에 있습니다.
Source this
출처는 이것
file first in Vivado, then execute each step sequentially.
먼저 Vivado에서 파일을 생성한 다음 각 단계를 순차적으로 실행하세요.

```tcl
source <skill-path>/helper-procedures/helper_scripts.tcl
```

### Step 1: Detect Project Flow
### 1단계: 프로젝트 흐름 감지

Run `detect_project_flow` to identify the project type.
프로젝트 유형을 식별하려면 `Detect_project_flow`를 실행하세요.
This returns a
이것은
`flow_info` dictionary needed by Step 3.
3단계에서는 `flow_info` 사전이 필요합니다.

```tcl
set flow_info [detect_project_flow]
set flow_type [dict get $flow_info flow_type]
```

Returns one of: "Standard", "DFX (Partial Reconfiguration)", "Segmented
"표준", "DFX(부분 재구성)", "세그먼트화" 중 하나를 반환합니다.
Configuration", "DFX + Segmented Configuration" — with optional "+ IPI BDC"
구성", "DFX + 세그먼트 구성" - 옵션 "+ IPI BDC" 포함
suffix if Block Design Containers are detected.
블록 디자인 컨테이너가 감지되면 접미사를 붙입니다.

For details on detection logic and edge cases, read
감지 논리 및 엣지 케이스에 대한 자세한 내용은 다음을 참조하세요.
[detect-project-flow/REFERENCE.md](detect-project-flow/REFERENCE.md).
[감지 프로젝트 흐름/REFERENCE.md](감지 프로젝트 흐름/REFERENCE.md).

### Step 2: Analyze Source Locations
### 2단계: 소스 위치 분석

Run `analyze_source_locations` to classify sources as Push Button (all local),
`analyze_source_locations`를 실행하여 소스를 푸시 버튼(모두 로컬)으로 분류합니다.
Remote (all external), or Mixed.
원격(모두 외부) 또는 혼합.

```tcl
set scenario_info [analyze_source_locations]
set scenario [dict get $scenario_info scenario]
```

If `scenario` is "Remote", skip Step 3 entirely — sources live in external
`시나리오`가 '원격'인 경우 3단계를 완전히 건너뜁니다. 소스는 외부에 있습니다.
repos and should not be duplicated.
저장소이므로 중복되어서는 안 됩니다.

For details, read
자세한 내용은 다음을 읽어보세요.
[analyze-source-locations/REFERENCE.md](analyze-source-locations/REFERENCE.md).
[분석 소스 위치/REFERENCE.md](분석 소스 위치/REFERENCE.md).

### Step 2.5: Preserve Generated Outputs? (optional, interactive)
### 2.5단계: 생성된 출력을 보존하시겠습니까? (선택사항, 대화형)

Ask whether to keep existing `.gen` outputs for selected IPs/BDs instead of
대신 선택한 IP/BD에 대한 기존 `.gen` 출력을 유지할지 묻습니다.
letting them be fully regenerated from Tcl on rebuild.
재구축 시 Tcl에서 완전히 재생성되도록 합니다.
This step does not
이 단계는 그렇지 않습니다.
exist in older versions of this skill and is entirely opt-in.
이 기술의 이전 버전에는 존재하며 완전히 선택되어 있습니다.
How to run it
실행 방법
depends on who's driving the pipeline:
파이프라인을 누가 주도하는지에 따라 다릅니다.

**Human typing directly into a live Vivado session** (GUI Tcl console, or a
**라이브 Vivado 세션에 직접 사람이 입력**(GUI Tcl 콘솔 또는
manually-driven batch session with a real tty) — as of 2026-09-21, you don't
실제 tty를 사용한 수동 구동 배치 세션) — 2026년 9월 21일 기준으로
need to do anything special: if you call Step 3 (`export_all_sources`)
특별한 작업이 필요합니다. 3단계(`export_all_sources`)를 호출하는 경우
without already having answered this yourself, it detects a real tty on
아직 직접 답변하지 않은 상태에서 실제 tty를 감지합니다.
stdin and, if the project has anything eligible, prompts you automatically
stdin 및 프로젝트에 적합한 것이 있으면 자동으로 메시지를 표시합니다.
before proceeding.
계속하기 전에.
You can still call `prompt_preserve_outputs` explicitly
여전히 `prompt_preserve_outputs`를 명시적으로 호출할 수 있습니다.
beforehand if you prefer to control exactly when the prompt happens:
프롬프트가 나타나는 시기를 정확하게 제어하려면 미리 다음을 수행하십시오.

```tcl
set preserve_selection [prompt_preserve_outputs]
```

- Answering "n" (or just pressing enter) returns
- "n"이라고 대답하거나 Enter 키를 누르면 반환됩니다.
`{enabled 0 ips {} bds {} cache_dir {}}` — Steps 3 and 5 then behave
`{enabled 0 ips {} bds {} 캐시_dir {}}` — 3단계와 5단계는 이후 작동합니다.
exactly as before this feature existed.
이 기능이 존재하기 전과 똑같습니다.
- Answering "y" lists every eligible standalone IP and user-created BD (via
- "y"로 대답하면 모든 적격 독립형 IP 및 사용자 생성 BD가 나열됩니다.
`list_preservable_outputs`) — including XCIs that live outside any Block
`list_preservable_outputs`) — 블록 외부에 있는 XCI 포함
Design — and lets you pick which ones to preserve (`all` or a comma list).
디자인 - 보존할 항목('모두' 또는 쉼표 목록)을 선택할 수 있습니다.
This selection is **mixed/partial across separate top-level items** (some
이 선택은 **별도의 최상위 항목에 걸쳐 혼합/부분**(일부
standalone IPs/BDs preserved, others not) — that's the primary supported
독립형 IP/BD는 보존되고 다른 것들은 보존되지 않음) — 이것이 기본적으로 지원됩니다
case.
사건.
It is **not** partial *within* a single BD: a BD is preserved as one
단일 BD 내에서 부분적이지 **않습니다**: BD는 하나로 보존됩니다.
whole unit, and you cannot cherry-pick which of the IPs customized inside
내부에 사용자 정의된 IP를 선별적으로 선택할 수는 없습니다.
it get preserved vs. regenerated (confirmed as an intentional scope cut,
보존되거나 재생성됩니다(의도적인 범위 축소로 확인됨,
not a bug — flagged as a possible Q4 enhancement).
버그가 아닙니다. 4분기 개선 가능성이 있는 것으로 표시됨).
- If a selected item has no `.gen` output yet (never generated/synthesized),
- 선택한 항목에 아직 '.gen' 출력이 없는 경우(생성/합성된 적이 없음)
`export_all_sources` warns and exports it normally instead of preserving
`export_all_sources`는 보존하는 대신 경고하고 정상적으로 내보냅니다.
it — nothing to preserve yet is not a hard failure, but it also does not
그것 — 보존할 것이 아무것도 없지만 심각한 실패는 아니지만, 그렇지도 않습니다.
silently create an incomplete cache entry.
불완전한 캐시 항목을 자동으로 생성합니다.
- Preservation only supports a **remote cache**: a directory *outside* this
- 보존은 **원격 캐시**: 이 *외부의* 디렉터리만 지원합니다.
project.
프로젝트.
After picking IPs/BDs, you're prompted for that cache directory's
IP/BD를 선택하면 해당 캐시 디렉터리를 묻는 메시지가 표시됩니다.
absolute path.
절대 경로.
An invalid path (relative, empty, or nested inside the
유효하지 않은 경로(상대 경로, 비어 있거나 내부에 중첩된 경로)
project) disables preservation for this export.
프로젝트)는 이 내보내기에 대한 보존을 비활성화합니다.

This proc blocks on `gets stdin`, so it only works with a real tty attached
이 proc은 `gets stdin`을 차단하므로 실제 tty가 연결된 경우에만 작동합니다.
— which is exactly the condition `export_all_sources` now checks
— 이것이 바로 `export_all_sources`가 이제 확인하는 조건입니다.
(`_rc_stdin_is_tty`) before auto-invoking it.
(`_rc_stdin_is_tty`)를 자동 호출하기 전에.

**Automated CI / regression pipelines** (`run_pipeline.tcl`-style harnesses)
**자동화된 CI/회귀 파이프라인**(`run_pipeline.tcl` 스타일 하네스)
— skip this step entirely and pass `{}` (the default) to Steps 3 and 5.
— 이 단계를 완전히 건너뛰고 `{}`(기본값)를 3단계와 5단계에 전달합니다.
That
그
preserves the old fully-automated 5-step behavior unchanged.
기존의 완전 자동화된 5단계 동작을 변경하지 않고 유지합니다.

**Agentic (an AI agent driving Vivado via Bash/batch-mode Tcl calls, chatting
**Agentic(Bash/배치 모드 Tcl 호출, 채팅을 통해 Vivado를 구동하는 AI 에이전트)
with a human)** — do **not** call `prompt_preserve_outputs`: its `gets stdin`
인간과 함께)** — `prompt_preserve_outputs`를 호출하지 **하지 마세요**: `gets stdin`
has no real tty to block on and will just read EOF, silently disabling
차단할 실제 tty가 없으며 EOF만 읽고 자동으로 비활성화합니다.
preservation with no visible prompt at all.
눈에 보이는 프롬프트가 전혀 없는 상태로 보존됩니다.
Instead:
대신:

1. Call `list_preservable_outputs` (pure query, no stdin) to see what's eligible.
1. 무엇이 적합한지 확인하려면 `list_preservable_outputs`(표준 입력이 아닌 순수 쿼리)를 호출하세요.
2. If it returns any IPs or BDs, ask the human directly in the conversation
2. IP나 BD가 반환되면 대화에서 직접 사람에게 물어보세요.
which ones (if any) they want to preserve, AND for an absolute path to a
어떤 것(있는 경우)을 보존하고 싶은지, 그리고
cache directory outside the project to store the preserved outputs.
보존된 출력을 저장하기 위해 프로젝트 외부의 캐시 디렉토리.
3. Call `build_preserve_selection $chosen_ips $chosen_bds $cache_dir` with
3. 다음과 같이 `build_preserve_selection $chosen_ips $chosen_bds $cache_dir`을 호출하세요.
the human's answers to get the `preserve_selection` dict.
`preserve_selection` 딕셔너리를 얻기 위한 인간의 답변입니다.
4. If `list_preservable_outputs` returns nothing, skip asking — there's
4. `list_preservable_outputs`가 아무것도 반환하지 않으면 묻지 마세요.
nothing to preserve.
보존할 것이 아무것도 없습니다.

```tcl
set items [list_preservable_outputs]
# ... ask the human in conversation, then:
set preserve_selection [build_preserve_selection $chosen_ips $chosen_bds $cache_dir]
```

Pass the returned `preserve_selection` into Step 3 (`export_all_sources`).
반환된 `preserve_selection`을 3단계(`export_all_sources`)에 전달합니다.
For details, read
자세한 내용은 다음을 읽어보세요.
[helper-procedures/REFERENCE.md](helper-procedures/REFERENCE.md).
[helper-procedures/REFERENCE.md](helper-procedures/REFERENCE.md).

### Step 3: Export All Sources (conditional)
### 3단계: 모든 소스 내보내기(조건부)

Skip this step if Step 2 returned "Remote".
2단계에서 "원격"이 반환된 경우 이 단계를 건너뛰세요.
Otherwise, export RTL, constraints,
그렇지 않으면 RTL, 제약 조건,
IP (as Tcl via `write_ip_tcl`), Block Designs (as Tcl via `write_bd_tcl`),
IP(`write_ip_tcl`을 통한 Tcl), 블록 설계(`write_bd_tcl`을 통한 Tcl),
simulation files, and data files to an organized directory.
시뮬레이션 파일 및 데이터 파일을 정리된 디렉터리에 저장합니다.

Pass `flow_info` from Step 1 to enable DFX/Segmented Config-specific exports
DFX/분할 구성 관련 내보내기를 활성화하려면 1단계의 'flow_info'를 전달하세요.
(pblocks, DCPs, NoC solutions).
(pblock, DCP, NoC 솔루션).
Pass `preserve_selection` from Step 2.5 (or
2.5단계에서 `preserve_selection`을 전달합니다(또는
omit it) to control per-IP/BD output preservation.
생략) IP/BD별 출력 보존을 제어합니다.
If you omit it and stdin
생략하고 stdin을 하면
is a real tty, `export_all_sources` prompts for it itself (see Step 2.5
실제 tty입니다. `export_all_sources`는 자체적으로 이에 대한 프롬프트를 표시합니다(2.5단계 참조).
above) rather than silently skipping preservation.
위) 보존을 조용히 건너뛰는 것보다.
For any IP/BD named in
다음에 이름이 지정된 모든 IP/BD의 경우
`preserve_selection`, its own `.srcs`/`.gen` subtree (nothing else) is
`preserve_selection`, 자체 `.srcs`/`.gen` 하위 트리(다른 것은 없음)는 다음과 같습니다.
copied into the external `cache_dir` the selection specifies — never into
선택 항목이 지정하는 외부 `cache_dir`에 복사됩니다. 절대 복사되지 않습니다.
`export_dir`/`RevisionControl/` itself.
`export_dir`/`RevisionControl/` 자체.

```tcl
if {$scenario ne "Remote"} {
    export_all_sources "RevisionControl" $flow_info $preserve_selection
}
```

For details on filtering rules, BD-IP handling, and DFX exports, read
필터링 규칙, BD-IP 처리 및 DFX 내보내기에 대한 자세한 내용은 다음을 참조하세요.
[export-all-sources/REFERENCE.md](export-all-sources/REFERENCE.md).
[내보내기-모든 소스/REFERENCE.md](내보내기-모든 소스/REFERENCE.md).

### Step 4: Capture Project Settings
### 4단계: 프로젝트 설정 캡처

Save all non-default project properties to a Tcl script that can restore them
기본이 아닌 모든 프로젝트 속성을 복원할 수 있는 Tcl 스크립트에 저장합니다.
during rebuild.
재건축 중.

```tcl
file mkdir RevisionControl/Scripts
capture_project_settings "RevisionControl/Scripts/project_settings.tcl"
```

Captures: PART, BOARD_PART, TARGET_LANGUAGE, PR_FLOW,
캡처: PART, BOARD_PART, TARGET_LANGUAGE, PR_FLOW,
SEGMENTED_CONFIGURATION, TOP, VERILOG_DEFINE, synthesis/implementation
SEGMENTED_CONFIGURATION, TOP, VERILOG_DEFINE, 합성/구현
directives, and file-specific properties (VHDL LIBRARY, IS_GLOBAL_INCLUDE).
지시문 및 파일별 속성(VHDL LIBRARY, IS_GLOBAL_INCLUDE).

For details, read
자세한 내용은 다음을 읽어보세요.
[capture-project-settings/REFERENCE.md](capture-project-settings/REFERENCE.md).
[캡처-프로젝트-설정/REFERENCE.md](캡처-프로젝트-설정/REFERENCE.md).

### Step 5: Generate Build Script
### 5단계: 빌드 스크립트 생성

Create build.tcl for one-command project recreation.
단일 명령 프로젝트 재생성을 위해 build.tcl을 만듭니다.

```tcl
generate_build_script "RevisionControl/Scripts/build.tcl" "RevisionControl" $scenario_info
```

The generated script uses relative paths, imports local sources (`import_files`),
생성된 스크립트는 상대 경로를 사용하고 로컬 소스(`import_files`)를 가져옵니다.
references remote sources in-place (`add_files`), recreates IP and BD from Tcl
원격 소스를 내부에서 참조(`add_files`)하고 Tcl에서 IP 및 BD를 다시 생성합니다.
scripts, creates OOC IP runs, generates BD targets, sources project_settings.tcl,
스크립트, OOC IP 실행 생성, BD 대상 생성, project_settings.tcl 소스,
and finalizes compile order.
컴파일 순서를 마무리합니다.
For any IP/BD preserved in Step 2.5, it instead
2.5단계에서 보존된 IP/BD의 경우 대신
restores that item's `.srcs`/`.gen` from the external cache directory and
외부 캐시 디렉터리에서 해당 항목의 `.srcs`/`.gen`을 복원하고
`add_files`/`read_ip`/`read_bd`s it directly — no `write_ip_tcl`/`write_bd_tcl`
`add_files`/`read_ip`/`read_bd`는 직접적으로 — `write_ip_tcl`/`write_bd_tcl`이 아닙니다.
replay, no resynthesis — while every non-preserved IP/BD still goes through
재생, 재합성 없음 — 보존되지 않은 모든 IP/BD는 계속 진행됩니다.
the standard Tcl-regeneration path.
표준 Tcl 재생 경로.
This branching is per-IP/BD, never global
이 분기는 IP/BD별로 이루어지며 전역적이지는 않습니다.
for the whole project.
전체 프로젝트에 대해.
If the cache directory (or a specific preserved
캐시 디렉토리(또는 특정 보존된 디렉토리)가
item's data inside it) is missing at rebuild time, the generated script
항목의 데이터)가 재구축 시 누락되고 생성된 스크립트가 누락됩니다.
**hard-errors** rather than silently falling back to regeneration.
조용히 재생성으로 돌아가는 것이 아니라 **하드 오류**입니다.

For details, DFX RM ordering, and missing-`.gen` handling, read
자세한 내용은 DFX RM 주문 및 누락된 `.gen` 처리를 참조하세요.
[generate-build-script/REFERENCE.md](generate-build-script/REFERENCE.md).
[생성-빌드-스크립트/REFERENCE.md](생성-빌드-스크립트/REFERENCE.md).

## Pipeline Data Flow
## 파이프라인 데이터 흐름

```
Step 1: detect_project_flow
  → flow_info dict (passed to Step 3)

Step 2: analyze_source_locations
  → scenario ("Push Button" / "Remote" / "Mixed")
  → If Remote: skip Step 3

Step 2.5: prompt_preserve_outputs (optional, interactive only)
  → preserve_selection dict {enabled, ips, bds, cache_dir} (passed to Step 3)
  → omitted/declined ⇒ {} ⇒ Steps 3 and 5 behave exactly as before this existed
  → if omitted AND stdin is a real tty AND something is eligible, Step 3
    auto-calls this on your behalf (added 2026-09-21) instead of silently
    defaulting to no preservation — no tty (CI/agentic) ⇒ unchanged
  → cache_dir must be an absolute path OUTSIDE the project (remote cache
    only) — an invalid/missing path disables preservation for this export

Step 3: export_all_sources(dir, flow_info, preserve_selection)
  → Sources/ directory with RTL, IP, BD, Constraints (dir itself never
    changes shape based on preserve_selection)
  → (only if preserve_selection selected something) each preserved item's
    OWN .srcs/.gen subtree is copied into cache_dir — never into dir itself

Step 4: capture_project_settings(file)
  → project_settings.tcl

Step 5: generate_build_script(file, dir, scenario_info)
  → build.tcl (automated rebuild; restores any preserved item's .srcs/.gen
    from cache_dir at rebuild time — hard-errors if cache_dir or a
    preserved item's data inside it is missing)
```

## Output Structure
## 출력 구조

`RevisionControl/` exports *next to* the live Vivado project and its own
`RevisionControl/`은 라이브 Vivado 프로젝트와 자체 프로젝트 *옆에* 내보냅니다.
shape never changes based on whether Step 2.5 preservation was used —
2.5단계 보존 사용 여부에 따라 모양이 변경되지 않습니다.
it never contains `<proj>.gen/`, `<proj>.srcs/`, or `<proj>.xpr` from the
여기에는 `<proj>.gen/`, `<proj>.srcs/` 또는 `<proj>.xpr`이 포함되지 않습니다.
original project (those stay untouched where they are), and XPR is never
원본 프로젝트(현재 위치에 그대로 유지됨)이며 XPR은 절대로
revision-controlled in any flow.
모든 흐름에서 개정 제어.

```
project_1.gen/            # live original Vivado project — sibling, untouched
project_1.srcs/
project_1.xpr

RevisionControl/
├── Sources/
│   ├── RTL/              # Verilog/SV/VHDL sources
│   ├── Constraints/      # XDC files (+ pblocks.xdc for DFX)
│   ├── IP/               # write_ip_tcl scripts — non-preserved standalone
│   │                     # IPs only (no .xci copied); preserved IPs are
│   │                     # skipped here entirely (see below)
│   ├── BD/               # write_bd_tcl scripts — non-preserved BDs only
│   │                     # (no .bd copied); preserved BDs are skipped here
│   │                     # entirely, with no Tcl fallback (see below)
│   ├── Simulation/       # Testbenches
│   ├── Data/             # .mem, .coe, .mif, .hex, .elf, .csv files
│   ├── Scripts/          # Tcl pre/post hook scripts
│   ├── Checkpoints/      # (DFX only) locked static DCPs
│   └── NoC/              # (Segmented Config only) .ncr files
└── Scripts/
    ├── project_settings.tcl
    └── build.tcl
```

### Preserved outputs (Step 2.5 answered "Yes", ≥1 IP/BD selected)
### 보존된 출력(2.5단계에서 "예"로 응답, ≥1 IP/BD 선택)

If Step 2.5 preserved anything, the only artifact is an external **remote
2.5단계에서 보존된 항목이 있는 경우 유일한 아티팩트는 외부 **원격입니다.
cache directory** — a path outside the project that the user supplied
캐시 디렉터리** — 사용자가 제공한 프로젝트 외부 경로
(`cache_dir` in `preserve_selection`), never copied into `RevisionControl/`
(`preserve_selection`의 `cache_dir`), `RevisionControl/`에 복사되지 않음
and never git-tracked:
결코 git-tracked 되지 않습니다:

```
<cache_dir>/
└── project_1/
    ├── ip/
    │   └── <ip_name>/
    │       ├── srcs/     # copy of that IP's own .srcs directory (the .xci
    │       │              # and any sibling files it had)
    │       └── gen/      # copy of that IP's own .gen output directory
    └── bd/
        └── <bd_name>/
            ├── srcs/     # copy of that BD's own .srcs directory (the .bd,
            │              # PLUS any nested per-instance ip/<name>/<name>.xci
            │              # files customized only inside that BD)
            └── gen/      # copy of that BD's own .gen output directory
```

Only the *selected* item's own `.srcs`/`.gen` subtree is copied — never the
*선택한* 항목의 자체 `.srcs`/`.gen` 하위 트리만 복사됩니다.
whole project's generated tree, and never the `.xpr`.
전체 프로젝트의 생성된 트리이며 절대 `.xpr`이 아닙니다.
Copying the whole
전체 복사하기
per-BD `.srcs` directory (not just the lone `<bd>.bd` file) also means a
BD별 `.srcs` 디렉토리(단지 `<bd>.bd` 파일뿐만 아니라)는 다음을 의미합니다.
per-instance IP customized only inside a preserved BD is preserved too.
보존된 BD 내부에서만 사용자 정의된 인스턴스별 IP도 보존됩니다.

At rebuild time, `generate_build_script`'s generated `build.tcl` reads
재구축 시 `generate_build_script`가 생성한 `build.tcl`이 다음을 읽습니다.
directly from this cache directory (an absolute path baked into `build.tcl`
이 캐시 디렉토리에서 직접(`build.tcl`에 구운 절대 경로)
at export time) and restores each preserved item's `.srcs`/`.gen` into the
내보내기 시) 보존된 각 항목의 `.srcs`/`.gen`을
newly created project's own tree before `add_files`/`read_ip`/`read_bd`ing
`add_files`/`read_ip`/`read_bd`ing 전에 새로 생성된 프로젝트 자체 트리
it — no `write_ip_tcl`/`write_bd_tcl` replay, no resynthesis. **If the cache
그것 — `write_ip_tcl`/`write_bd_tcl` 재생 없음, 재합성 없음. **캐시인 경우
directory, or a specific preserved item's data inside it, is missing at
디렉터리 또는 그 안에 있는 특정 보존 항목의 데이터가 누락되었습니다.
rebuild time, `build.tcl` hard-errors and stops** rather than silently
재구축 시간, `build.tcl` 하드 오류 및 자동 중지**
falling back to Tcl regeneration — preservation is an explicit opt-in, so a
Tcl 재생성으로 후퇴 - 보존은 명시적인 선택이므로
missing cache is a setup problem to surface, not paper over.
캐시 누락은 종이 위에 올려놓는 것이 아니라 표면에 나타나는 설정 문제입니다.

## Special Project Types
## 특별 프로젝트 유형

For DFX projects, read [dfx-revision-control/REFERENCE.md](dfx-revision-control/REFERENCE.md).
DFX 프로젝트의 경우 [dfx-revision-control/REFERENCE.md](dfx-revision-control/REFERENCE.md)를 읽어보세요.
Key requirement: In build.tcl, RM Block Designs must be sourced BEFORE the
주요 요구 사항: build.tcl에서 RM 블록 설계는
static design, or BDC instantiation fails.
정적 디자인 또는 BDC 인스턴스화가 실패합니다.

For Segmented Config (Versal), read
분할된 구성(Versal)의 경우 다음을 읽어보세요.
[segmented-config-revision-control/REFERENCE.md](segmented-config-revision-control/REFERENCE.md).
[세그먼트-구성-개정-제어/REFERENCE.md](세그먼트-구성-개정-제어/REFERENCE.md).
Key requirement: SEGMENTED_CONFIGURATION property must be captured and restored.
주요 요구 사항: SEGMENTED_CONFIGURATION 속성을 캡처하고 복원해야 합니다.
Versal Gen2 devices (xcve2*, xcvp2*, xcvm2*) have this implicitly enabled.
Versal Gen2 장치(xcve2*, xcvp2*, xcvm2*)에는 이 기능이 암시적으로 활성화되어 있습니다.

## After Completion
## 완료 후

Test the generated build script in a clean directory:
깨끗한 디렉터리에서 생성된 빌드 스크립트를 테스트합니다.
```bash
vivado -mode batch -source RevisionControl/Scripts/build.tcl
```

For DFX projects, configure Git LFS for DCP files:
DFX 프로젝트의 경우 DCP 파일용 Git LFS를 구성합니다.
```bash
git lfs track "*.dcp"
```

### Optional: Verify the Rebuild Matches the Original
### 선택 사항: 재구축이 원본과 일치하는지 확인

Vivado gives no built-in signal that a recreated project actually matches the
Vivado는 재현된 프로젝트가 실제로 프로젝트와 일치한다는 내장된 신호를 제공하지 않습니다.
original — a silently-skipped BD, a missing wrapper module, or a dropped run
원본 — 자동으로 건너뛴 BD, 누락된 래퍼 모듈 또는 실행 중단
won't error until (or unless) synthesis/implementation is attempted.
합성/구현이 시도될 때까지(또는 시도하지 않는 한) 오류가 발생하지 않습니다.
Two
2개
utility procedures close this gap:
유틸리티 절차는 이러한 격차를 해소합니다.

```tcl
# In the ORIGINAL project's session, right after export_all_sources:
capture_verification_manifest "RevisionControl/Scripts/verification_manifest.tcl" "RevisionControl"

# In the RECREATED project's session, after sourcing build.tcl:
verify_rebuild "RevisionControl/Scripts/verification_manifest.tcl"
```

`verify_rebuild` checks that all expected Block Designs, the top module (including
`verify_rebuild`는 예상되는 모든 블록 디자인, 최상위 모듈(포함)을 확인합니다.
auto-managed wrapper modules with no physical source file), and all expected
물리적 소스 파일이 없는 자동 관리 래퍼 모듈) 및 예상되는 모든 항목
runs are present, plus an IP-count sanity check — and prints a PASS/FAIL report.
실행이 존재하고 IP 개수 온전성 검사가 이루어지며 PASS/FAIL 보고서가 인쇄됩니다.
For details, read
자세한 내용은 다음을 읽어보세요.
[helper-procedures/REFERENCE.md](helper-procedures/REFERENCE.md).
[helper-procedures/REFERENCE.md](helper-procedures/REFERENCE.md).

## Vivado Documentation Lookup
## Vivado 문서 조회

If you need more information about specific Vivado Tcl commands used in this
여기서 사용되는 특정 Vivado Tcl 명령어에 대한 추가 정보가 필요한 경우
skill (write_ip_tcl, write_bd_tcl, write_checkpoint, create_project, etc.),
스킬(write_ip_tcl, write_bd_tcl, write_checkpoint, create_project 등),
use the `vivado_doc_search` MCP tool to search the Vivado documentation.
Vivado 문서를 검색하려면 `vivado_doc_search` MCP 도구를 사용하세요.

Relevant Vivado User Guides:
관련 Vivado 사용자 가이드:
- **UG892** — Vivado Design Flows Overview
- **UG892** — Vivado 디자인 흐름 개요
- **UG895** — Using IP in Vivado
- **UG895** — Vivado에서 IP 사용
- **UG896** — Partial Reconfiguration (DFX)
- **UG896** — 부분 재구성(DFX)
- **UG994** — Designing IP Subsystems Using IP Integrator
- **UG994** — IP Integrator를 사용하여 IP 하위 시스템 설계
- **UG1281** — Versal System Integration (Segmented Configuration)
- **UG1281** — Versal 시스템 통합(세그먼트 구성)

## References
## 참고자료

- [helper-procedures/REFERENCE.md](helper-procedures/REFERENCE.md) — Full API reference for all Tcl procedures
- [helper-procedures/REFERENCE.md](helper-procedures/REFERENCE.md) — 모든 Tcl 프로시저에 대한 전체 API 참조
- [helper-procedures/helper_scripts.tcl](helper-procedures/helper_scripts.tcl) — The Tcl implementation (single source of truth)
- [helper-procedures/helper_scripts.tcl](helper-procedures/helper_scripts.tcl) — Tcl 구현(단일 소스)
