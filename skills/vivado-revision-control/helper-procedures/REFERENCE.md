<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

---
name: helper-procedures
이름: 도우미 프로시저
description: Tcl procedure library API reference — all five pipeline procedures plus utilities.
설명: Tcl 프로시저 라이브러리 API 참조 — 다섯 가지 파이프라인 프로시저와 유틸리티 모두.
---

# Helper Procedures Library
# 도우미 프로시저 라이브러리

All revision control procedures live in `helper_scripts.tcl`.
모든 개정 제어 절차는 `helper_scripts.tcl`에 있습니다.
Source it once
소스 한번 받아보세요
before calling any procedure:
프로시저를 호출하기 전에:

```tcl
source <skill-path>/helper-procedures/helper_scripts.tcl
```

## Procedure Reference
## 절차 참조

### detect_project_flow
### 감지_프로젝트_흐름

```tcl
set flow_info [detect_project_flow]
```

No parameters.
매개변수가 없습니다.
Returns dict with: project_name, device, flow_type, is_dfx,
project_name, device, flow_type, is_dfx를 포함하는 dict를 반환합니다.
is_segmented_config, is_versal, is_ipi_bdc, bdc_count, bd_file_count, pblock_count.
is_segmented_config, is_versal, is_ipi_bdc, bdc_count, bd_file_count, pblock_count.

### analyze_source_locations
### analyze_source_locations

```tcl
set scenario_info [analyze_source_locations]
```

No parameters.
매개변수가 없습니다.
Returns dict with: project_dir, scenario, local_count,
project_dir, 시나리오, local_count,
remote_count, remote_paths, strategy.
원격_수, 원격_경로, 전략.

### list_preservable_outputs (utility)
### list_preservable_outputs(유틸리티)

```tcl
set items [list_preservable_outputs]
```

No parameters.
매개변수가 없습니다.
Returns dict with `ips` and `bds`, each a list of `{name type
`ips` 및 `bds`가 포함된 dict를 반환합니다. 각각 `{name type' 목록입니다.
has_gen gen_dir}` dicts — standalone IPs (XCIs outside any Block Design, via
has_gen gen_dir}` dicts — 독립형 IP(블록 디자인 외부의 XCI, 다음을 통해)
`IS_BD_CONTEXT`) and user-created top-level BDs, mirroring the exact
`IS_BD_CONTEXT`) 및 사용자가 생성한 최상위 BD를 사용하여 정확한 내용을 미러링합니다.
filtering `export_all_sources` itself uses so enumeration and export never
'export_all_sources' 필터링 자체가 사용되므로 열거 및 내보내기는 절대 사용되지 않습니다.
disagree on what's eligible.
무엇이 적격인지에 대해 동의하지 않습니다.
`has_gen` reports whether a `.gen` output
`has_gen`은 `.gen` 출력 여부를 보고합니다.
already exists for that item.
해당 항목이 이미 존재합니다.

### prompt_preserve_outputs (Step 2.5, optional/interactive)
### Prompt_preserve_outputs(2.5단계, 선택사항/대화형)

```tcl
set preserve_selection [prompt_preserve_outputs]
```

No parameters.
매개변수가 없습니다.
Calls `list_preservable_outputs`, asks (via `gets stdin`)
`list_preservable_outputs`를 호출하고 묻습니다(`gets stdin`을 통해).
whether to preserve generated outputs, and if yes, which named IPs/BDs
생성된 출력을 보존할지 여부, 그렇다면 IP/BD라는 이름을 지정합니다.
(`all` or a comma list) and, per 2026-09-29 methodology feedback, an
('모두' 또는 쉼표 목록) 및 2026년 9월 29일 방법론 피드백에 따라
absolute path to a **remote cache directory** (outside the project) to
**원격 캐시 디렉터리**(프로젝트 외부)에 대한 절대 경로
store the preserved `.srcs`/`.gen`.
보존된 `.srcs`/`.gen`을 저장합니다.
Returns `{enabled 0|1, ips {name ...},
`{활성화된 0|1, ips {이름 ...}을 반환합니다.
bds {name ...}, cache_dir <absolute path>}`.
bds {이름 ...}, 캐시_디렉터리 <절대 경로>}`.
An invalid/empty cache path
유효하지 않거나 비어 있는 캐시 경로
degrades the whole selection back to `enabled 0`, same as an empty/no-match
비어 있거나 일치하지 않는 것과 마찬가지로 전체 선택을 '활성화된 0'으로 다시 저하시킵니다.
IP/BD selection. **Blocks on stdin — only call from an interactive
IP/BD 선택. **stdin의 블록 — 대화형에서만 호출
session**, never from an automated/batch pipeline.
세션**, 자동화된/일괄 파이프라인에서는 절대 발생하지 않습니다.
Automated callers should
자동 호출자는 다음을 수행해야 합니다.
pass `{}` (or omit the argument) to `export_all_sources`/`generate_build_script`
`{}`를 `export_all_sources`/`generate_build_script`에 전달(또는 인수 생략)
instead, which is fully backward compatible with the pre-existing 5-step
대신 기존 5단계와 완전히 역호환됩니다.
pipeline.
파이프라인.
An AI agent driving Vivado non-interactively should not call this
비대화식으로 Vivado를 구동하는 AI 에이전트는 이를 호출해서는 안 됩니다.
proc either (its `gets stdin` can't reach the human) — use
proc 중 하나(`gets stdin`이 인간에게 도달할 수 없음) — 사용
`build_preserve_selection` below instead.
대신 아래의 `build_preserve_selection`을 사용하세요.

### build_preserve_selection (Step 2.5, agentic/non-interactive)
### build_preserve_selection(2.5단계, 에이전트/비대화형)

```tcl
set preserve_selection [build_preserve_selection $sel_ips $sel_bds $cache_dir]
```

- `sel_ips` — list of IP names to preserve (already chosen, e.g. by an AI
- `sel_ips` — 보존할 IP 이름 목록(예: AI에 의해 이미 선택됨)
agent that asked the human directly in conversation)
대화 중에 인간에게 직접 물어본 에이전트)
- `sel_bds` — list of BD names to preserve
- `sel_bds` — 보존할 BD 이름 목록
- `cache_dir` — absolute path to a remote cache directory OUTSIDE the
- `cache_dir` — 외부의 원격 캐시 디렉토리에 대한 절대 경로
project, to store the preserved `.srcs`/`.gen` (per 2026-09-29 methodology
보존된 `.srcs`/`.gen`을 저장하기 위한 프로젝트(2026-09-29 방법론에 따라)
feedback, only a remote/external cache is supported)
피드백, 원격/외부 캐시만 지원됨)

Stdin-free equivalent of `prompt_preserve_outputs`, for callers that cannot
'prompt_preserve_outputs'와 동등한 Stdin-free입니다.
block on a real tty.
진짜 tty를 차단하세요.
Validates the given names against
주어진 이름의 유효성을 검사합니다.
`list_preservable_outputs` the same way the interactive path does, and
대화형 경로와 동일한 방식으로 `list_preservable_outputs`
validates `cache_dir` the same way `prompt_preserve_outputs` does (absolute,
`prompt_preserve_outputs`와 동일한 방식으로 `cache_dir`을 검증합니다(절대,
non-empty, not nested under the project); an empty/fully-invalid IP/BD
비어 있지 않고 프로젝트 아래에 중첩되지 않음) 비어 있거나 완전히 유효하지 않은 IP/BD
selection OR an invalid `cache_dir` returns `{enabled 0 ips {} bds {}
선택 또는 잘못된 `cache_dir`이 `{enabled 0 ips {} bds {}를 반환합니다.
cache_dir {}}`.
캐시_디렉터리 {}}`.
The intended flow: call `list_preservable_outputs` yourself,
의도된 흐름: `list_preservable_outputs`를 직접 호출하세요.
ask the human which (if any) to preserve AND for the cache directory path,
캐시 디렉토리 경로에 대해 AND를 보존할지(있는 경우) 사람에게 물어보십시오.
then call this proc with their answers.
그런 다음 답변을 사용하여 이 proc을 호출하세요.
Returns `{enabled 0|1, ips {name
`{활성화된 0|1, ips {이름을 반환합니다.
...}, bds {name ...}, cache_dir <absolute path>}`, same shape as
...}, bds {이름 ...}, 캐시_디렉토리 <절대 경로>}`, 같은 모양
`prompt_preserve_outputs`.
'prompt_preserve_outputs'.

### export_all_sources
### 수출_모든_소스

```tcl
set export_stats [export_all_sources $export_dir $flow_info $preserve_selection]
```

- `export_dir` — target directory (e.g., "RevisionControl")
- `export_dir` — 대상 디렉터리(예: "RevisionControl")
- `flow_info` — (optional) dict from detect_project_flow
- `flow_info` — (선택 사항) discover_project_flow의 dict
- `preserve_selection` — (optional) dict from `prompt_preserve_outputs`/
- `preserve_selection` — (선택 사항) `prompt_preserve_outputs`의 dict/
`build_preserve_selection`.
'빌드_보존_선택'.
When omitted/`{}` or `enabled 0`, behavior is
생략/`{}` 또는 `활성화 0`인 경우 동작은 다음과 같습니다.
unchanged from before this existed.
이것이 존재하기 전과 변함이 없었습니다.
If left at its raw default AND stdin
원시 기본값과 stdin으로 남겨둔 경우
is a real interactive tty (`_rc_stdin_is_tty`) AND something is eligible,
실제 대화형 tty(`_rc_stdin_is_tty`)이고 자격이 있는 항목입니다.
`export_all_sources` calls `prompt_preserve_outputs` itself instead of
`export_all_sources`는 대신 `prompt_preserve_outputs` 자체를 호출합니다.
silently skipping preservation.
자동으로 보존을 건너뜁니다.
When it selects IPs/BDs, those are
IP/BD를 선택하면 다음과 같습니다.
excluded from the standard `write_ip_tcl`/`write_bd_tcl` export; instead,
표준 `write_ip_tcl`/`write_bd_tcl` 내보내기에서 제외됩니다. 대신에
each selected item's own `.srcs` directory and matching `.gen` output
선택한 각 항목의 `.srcs` 디렉토리 및 일치하는 `.gen` 출력
directory are copied directly into `preserve_selection`'s `cache_dir`
디렉토리는 `preserve_selection`의 `cache_dir`에 직접 복사됩니다.
(external, outside the project, never git-tracked) at
(외부, 프로젝트 외부, 절대 git 추적되지 않음) at
`$cache_dir/<proj_name>/{ip,bd}/<name>/{srcs,gen}/`.
`$cache_dir/<proj_name>/{ip,bd}/<name>/{srcs,gen}/`.
`export_all_sources`
`export_all_sources`
also records each preserved item's fileset-relative `.xci`/`.bd` path
또한 각 보존 항목의 파일 세트 상대 `.xci`/`.bd` 경로를 기록합니다.
(`preserved_ip_rel_path`/`preserved_bd_rel_path`) and the resolved cache
(`preserved_ip_rel_path`/`preserved_bd_rel_path`) 및 확인된 캐시
directory (`preserved_cache_dir`) for `generate_build_script` to restore
복원할 `generate_build_script`용 디렉터리(`preserved_cache_dir`)
from at rebuild time.
재구축 시점부터.
Preservation granularity is whole-item only: with
보존 세분성은 전체 항목에만 적용됩니다.
multiple standalone IPs/BDs eligible, any subset may be selected (mixed
여러 독립 실행형 IP/BD 적격, 모든 하위 집합 선택 가능(혼합)
preservation across items is the primary use case) — but a single BD is
항목 간 보존이 주요 사용 사례입니다.) 그러나 단일 BD는
always preserved or not preserved as one unit; you cannot preserve some of
항상 하나의 단위로 보존되거나 보존되지 않습니다. 일부는 보존할 수 없습니다.
the IPs customized inside one BD while regenerating others.
하나의 BD 내부에서 IP를 사용자 정의하고 다른 BD를 재생성합니다.
If a selected
선택한 경우
item has no `.gen` output yet, `export_all_sources` warns and exports it
항목에는 아직 `.gen` 출력이 없습니다. `export_all_sources`에서 경고하고 내보냅니다.
through the standard path instead of preserving it (never writes an
보존하는 대신 표준 경로를 통해(절대
incomplete cache entry) — see
불완전한 캐시 항목) - 참조
[export-all-sources/REFERENCE.md](../export-all-sources/REFERENCE.md).
[내보내기-모든 소스/REFERENCE.md](../내보내기-모든-소스/REFERENCE.md).

Returns dict of file counts by category, including `preserved_ip_outputs`
`preserved_ip_outputs`를 포함하여 카테고리별 파일 수를 반환합니다.
and `preserved_bd_outputs` when preservation was used.
보존이 사용된 경우 'preserved_bd_outputs'입니다.

### capture_project_settings
### 캡처_프로젝트_설정

```tcl
set output_path [capture_project_settings $output_file]
```

- `output_file` — path for the .tcl output
- `output_file` — .tcl 출력 경로

Returns the output file path.
출력 파일 경로를 반환합니다.

### generate_build_script
### generate_build_script

```tcl
set output_path [generate_build_script $output_file $output_dir $scenario_info]
```

- `output_file` — path for build.tcl
- `output_file` — build.tcl의 경로
- `output_dir` — base directory with Sources/ and Scripts/
- `output_dir` — Sources/ 및 Scripts/가 있는 기본 디렉터리
- `scenario_info` — (optional) dict from `analyze_source_locations`.
- `scenario_info` — (선택 사항) `analyze_source_locations`의 dict입니다.
- Controls `import_files` vs `add_files` and emits remote paths.
- `import_files`와 `add_files`를 제어하고 원격 경로를 내보냅니다.

Returns the output file path.
출력 파일 경로를 반환합니다.

### pr_verify (utility)
### pr_verify(유틸리티)

```tcl
pr_verify
```

Validates DFX configuration: checks PR_FLOW, reconfigurable cells, Pblocks.
DFX 구성 검증: PR_FLOW, 재구성 가능한 셀, Pblock을 확인합니다.
Returns 1 on success, errors on failure.
성공하면 1을 반환하고, 실패하면 오류를 반환합니다.
Only meaningful for DFX projects.
DFX 프로젝트에만 의미가 있습니다.

### capture_verification_manifest (optional, utility)
### Capture_verification_manifest(선택 사항, 유틸리티)

```tcl
capture_verification_manifest $output_file $export_dir
```

- `output_file` — path for the manifest .tcl output (e.g.
- `output_file` — 매니페스트 .tcl 출력 경로(예:
`RevisionControl/Scripts/verification_manifest.tcl`)
`RevisionControl/Scripts/verification_manifest.tcl`)
- `export_dir` — the export directory used in `export_all_sources` (used to
- `export_dir` — `export_all_sources`에 사용되는 내보내기 디렉터리(다음에 사용됨)
derive the expected BD list from the actual exported `Sources/BD/*.tcl` files)
실제 내보낸 'Sources/BD/*.tcl' 파일에서 예상 BD 목록을 파생시킵니다.)

Run in the **original** project's session, after `export_all_sources`.
`export_all_sources` 이후 **원래** 프로젝트 세션에서 실행하세요.
Snapshots expected BD names, top module (and whether it's an auto-managed
스냅샷 예상 BD 이름, 최상위 모듈(및 자동 관리 여부)
wrapper with no physical file), run names, and IP count, so a later
실제 파일이 없는 래퍼), 실행 이름, IP 개수 등을 포함하므로 나중에
`verify_rebuild` call can diff the recreated project against it.
'verify_rebuild' 호출을 통해 다시 생성된 프로젝트를 비교할 수 있습니다.
Returns the
다음을 반환합니다.
output file path.
출력 파일 경로.

### verify_rebuild (optional, utility)
### verify_rebuild(선택 사항, 유틸리티)

```tcl
verify_rebuild $manifest_file
```

- `manifest_file` — path to the manifest written by `capture_verification_manifest`
- `manifest_file` — `capture_verification_manifest`에 의해 작성된 매니페스트의 경로

Run in the **recreated** project's session, after sourcing `build.tcl`.
`build.tcl`을 소싱한 후 **다시 생성된** 프로젝트 세션에서 실행합니다.
Checks (in order): all expected Block Designs present, top module resolves
확인(순서대로): 예상되는 모든 블록 디자인이 존재하고 최상위 모듈이 해결됩니다.
(explicitly catches the "auto-managed wrapper never generated" bug class —
("자동 관리 래퍼가 생성되지 않음" 버그 클래스를 명시적으로 포착 —
top references `<bd>_wrapper` but no file exists), all expected runs present,
상위 참조 `<bd>_wrapper`가 있지만 파일이 없음), 예상되는 모든 실행이 존재함,
and IP count is not lower than expected.
IP 수가 예상보다 낮지 않습니다.
Prints a PASS/FAIL/WARN report per
PASS/FAIL/WARN 보고서를 인쇄합니다.
category and returns 1 if all checks passed, 0 otherwise.
모든 검사가 통과되면 1을 반환하고, 그렇지 않으면 0을 반환합니다.

## Complete Pipeline Example
## 완전한 파이프라인 예

```tcl
source helper-procedures/helper_scripts.tcl

set flow_info [detect_project_flow]
set scenario_info [analyze_source_locations]
set scenario [dict get $scenario_info scenario]

# Step 2.5 is optional. Use {} (the default) in automated/batch pipelines.
# A human at a real tty can call prompt_preserve_outputs (interactive).
# An AI agent should call list_preservable_outputs, ask the human directly,
# then call build_preserve_selection -- see SKILL.md Step 2.5 for the flow.
set preserve_selection [prompt_preserve_outputs]

if {$scenario ne "Remote"} {
    export_all_sources "RevisionControl" $flow_info $preserve_selection
}

file mkdir RevisionControl/Scripts
capture_project_settings "RevisionControl/Scripts/project_settings.tcl"
generate_build_script "RevisionControl/Scripts/build.tcl" "RevisionControl" $scenario_info

# Optional: capture a manifest for later rebuild verification
capture_verification_manifest "RevisionControl/Scripts/verification_manifest.tcl" "RevisionControl"
```

Then, in a fresh Vivado session pointed at a clean copy of `RevisionControl/`:
그런 다음 새로운 Vivado 세션에서 `RevisionControl/`의 깨끗한 복사본을 가리킵니다.

```tcl
source helper-procedures/helper_scripts.tcl
source Scripts/build.tcl
verify_rebuild "Scripts/verification_manifest.tcl"
```

## Extending
## 확장

Don't modify helper_scripts.tcl directly.
helper_scripts.tcl을 직접 수정하지 마세요.
Create a separate file and source
별도의 파일과 소스 생성
both:
둘 다:

```tcl
source helper-procedures/helper_scripts.tcl
source my_custom_helpers.tcl
```
