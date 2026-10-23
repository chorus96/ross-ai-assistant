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
name: export-all-sources
이름: 모든 소스 내보내기
description: Step 3 — exports all project components (RTL, constraints, IP, BD, simulation, data) to an organized directory.
설명: 3단계 — 모든 프로젝트 구성 요소(RTL, 제약 조건, IP, BD, 시뮬레이션, 데이터)를 정리된 디렉터리로 내보냅니다.
---

# Export All Sources (Step 3)
# 모든 소스 내보내기(3단계)

**Source file:** `helper-procedures/helper_scripts.tcl`
**소스 파일:** `helper-procedures/helper_scripts.tcl`
**Proc name:** `export_all_sources`
**프록 이름:** `export_all_sources`

## Procedure
## 절차

```tcl
export_all_sources "RevisionControl" $flow_info $preserve_selection
```

**Parameters:**
**매개변수:**
- `export_dir` (string) — output directory, e.g. "RevisionControl"
- `export_dir`(문자열) — 출력 디렉토리, 예: "리비전 제어"
- `flow_info` (dict, optional) — output from Step 1.
- `flow_info`(dict, 선택 사항) — 1단계의 출력입니다.
- Enables DFX and Segmented Config-specific exports.
- DFX 및 세그먼트 구성 관련 내보내기를 활성화합니다.
- If omitted, assumes Standard project.
- 생략할 경우 표준 프로젝트로 가정합니다.
- `preserve_selection` (dict, optional) — output from Step 2.5's `prompt_preserve_outputs`/`build_preserve_selection`: `{enabled, ips, bds, cache_dir}`.
- `preserve_selection`(dict, 선택 사항) — 2.5단계의 `prompt_preserve_outputs`/`build_preserve_selection`의 출력: `{enabled, ips, bds, 캐시_dir}`.
- If omitted/`{}`, behaves exactly as before this feature existed (100% backward compatible with older/existing projects and callers).
- 생략된 경우/`{}`는 이 기능이 존재하기 전과 동일하게 작동합니다(이전/기존 프로젝트 및 호출자와 100% 역호환 가능).
- If it selects one or more IPs/BDs, those are routed to the preserve-outputs branch (see below) instead of the standard `write_ip_tcl`/`write_bd_tcl` export.
- 하나 이상의 IP/BD를 선택하면 표준 `write_ip_tcl`/`write_bd_tcl` 내보내기 대신 출력 보존 분기(아래 참조)로 라우팅됩니다.
- `cache_dir` must be an absolute path outside the project (remote-cache-only, per 2026-09-29 methodology feedback) — an invalid/missing `cache_dir` while IPs/BDs are named disables preservation for the whole export.
- 'cache_dir'은 프로젝트 외부의 절대 경로여야 합니다(원격 캐시 전용, 2026-09-29 방법론 피드백에 따라). IP/BD 이름이 지정되는 동안 'cache_dir'이 잘못되거나 누락되면 전체 내보내기에 대한 보존이 비활성화됩니다.

**Returns:** dictionary of export counts (rtl_files, constraint_files, ip_files, bd_files, sim_files, data_files, hook_scripts, dfx_dcp_files, pblock_files, noc_files, preserved_ip_outputs, preserved_bd_outputs).
**반환:** 내보내기 횟수 사전(rtl_files, Constraint_files, ip_files, bd_files, sim_files, data_files, Hook_scripts, dfx_dcp_files, pblock_files, noc_files, Preserved_ip_outputs, Preserved_bd_outputs).

## Preserve-Outputs Branch (per-IP/BD, optional)
## 보존 출력 분기(IP/BD별, 선택 사항)

When `preserve_selection` names a given standalone IP or user-created BD,
`preserve_selection`이 특정 독립형 IP 또는 사용자 생성 BD의 이름을 지정하면,
that item is **excluded** from the standard `write_ip_tcl`/`write_bd_tcl`
해당 항목은 표준 `write_ip_tcl`/`write_bd_tcl`에서 **제외**됩니다.
export path.
수출 경로.
As of 2026-09-29 (GP methodology feedback), the item's own
2026-09-29 기준(GP 방법론 피드백), 해당 항목 자체
`.srcs` directory (the `.xci`/`.bd` plus any sibling files — for a BD, this
`.srcs` 디렉토리(`.xci`/`.bd` 및 형제 파일 — BD의 경우 이
also includes any nested per-instance `ip/<name>/<name>.xci` customized only
또한 사용자 정의된 중첩된 인스턴스별 `ip/<name>/<name>.xci`도 포함됩니다.
inside that BD) and its matching `.gen` output directory are copied directly
해당 BD 내부) 및 일치하는 `.gen` 출력 디렉토리가 직접 복사됩니다.
into an external **remote cache directory** (`preserve_selection`'s
외부 **원격 캐시 디렉터리**(`preserve_selection`의
`cache_dir`, outside the project, never git-tracked) at:
`cache_dir`, 프로젝트 외부, 절대로 git 추적되지 않음):
`$cache_dir/<proj_name>/{ip,bd}/<name>/{srcs,gen}/`.
`$cache_dir/<proj_name>/{ip,bd}/<name>/{srcs,gen}/`.
Only that item's own
해당 항목의 고유 항목만
subtree is copied — never the whole project's `.gen`/`.srcs`, and never the
하위 트리는 복사됩니다. 전체 프로젝트의 `.gen`/`.srcs`는 절대 복사되지 않습니다.
`.xpr`.
`.xpr`.
`export_all_sources` also records, per selected item, the
`export_all_sources`는 또한 선택한 항목별로 다음을 기록합니다.
fileset-relative path of its `.xci`/`.bd` (e.g.
`.xci`/`.bd`의 파일 세트 상대 경로(예:
`sources_1/ip/<name>/<name>.xci`
`sources_1/ip/<이름>/<이름>.xci`
— do not assume `sources_1`, a BD/IP can live in a different fileset) so
— `sources_1`을 가정하지 마십시오. BD/IP는 다른 파일 세트에 있을 수 있습니다.)
`generate_build_script` knows exactly where inside the recreated project to
`generate_build_script`는 재생성된 프로젝트 내에서 정확히 어디에 있는지 알고 있습니다.
restore it, rather than guessing.
추측하기보다는 복원하세요.

**Preservation is whole-item only, not partial within a BD:** a BD is either
**보존은 전체 항목에 대해서만 이루어지며 BD 내에서는 부분 보존이 아닙니다.** BD는 다음 중 하나입니다.
preserved as a single unit (its `.srcs` directory and everything customized
단일 단위로 보존됩니다(`.srcs` 디렉토리와 모든 사용자 정의됨)
inside it) or not preserved at all — there is no way to preserve only some
내부) 또는 전혀 보존되지 않음 — 일부만 보존할 수 있는 방법이 없습니다.
of the IPs customized inside a given BD while regenerating others.
특정 BD 내에서 사용자 정의된 IP 중 다른 IP를 재생성합니다.
Partial
부분
preservation ACROSS separate top-level items (some standalone IPs/BDs
보존 ACROSS 별도의 최상위 항목(일부 독립형 IP/BD)
preserved, others left to the standard export path) is fully supported and
보존되고 나머지는 표준 내보내기 경로에 남음)은 완벽하게 지원되며
is the primary intended use case; partial preservation WITHIN a single BD
기본적으로 의도된 사용 사례입니다. 단일 BD 내 부분 보존
is an intentional scope cut (possible future enhancement), not a bug.
버그가 아닌 의도적인 범위 축소(향후 개선 가능)입니다.

**If a selected item has no `.gen` output yet** (never generated/synthesized
**선택한 항목에 아직 `.gen` 출력이 없는 경우**(생성/합성되지 않음)
— `list_preservable_outputs`'s `has_gen` reports this in advance, but is
— `list_preservable_outputs`의 `has_gen`은 이를 미리 보고하지만
only surfaced as a warning in the interactive prompt, not enforced there),
대화형 프롬프트에서는 경고로만 표시되며 강제로 적용되지는 않습니다)
`export_all_sources` itself checks for this and, if the `.gen` directory
`export_all_sources` 자체가 이를 확인하고, `.gen` 디렉토리가
doesn't exist, prints a `WARNING` and exports that one item through the
존재하지 않는 경우 'WARNING'을 인쇄하고 해당 항목을 다음을 통해 내보냅니다.
standard `write_ip_tcl`/`write_bd_tcl` path instead of adding it to the
표준 `write_ip_tcl`/`write_bd_tcl` 경로를 추가하는 대신
cache — it does not silently write a cache entry with an empty/missing
캐시 — 비어 있거나 누락된 캐시 항목을 자동으로 쓰지 않습니다.
`gen/` subfolder (which would otherwise only surface as a confusing hard
`gen/` 하위 폴더(그렇지 않으면 혼란스러운 하드 폴더로만 나타남)
error much later, at rebuild time).
훨씬 나중에 다시 빌드할 때 오류가 발생합니다).
This check runs regardless of caller
이 확인은 호출자에 관계없이 실행됩니다.
(interactive or agentic), so an agentic caller that skips the interactive
(대화형 또는 에이전트), 따라서 대화형을 건너뛰는 에이전트 발신자는
`has_gen` warning still gets this enforcement.
`has_gen` 경고는 여전히 이 시행을 받습니다.
Items skipped this way are
이 방법으로 건너뛴 항목은 다음과 같습니다.
listed in the final export summary under "Preservation Requested But
최종 수출요약서의 '보존을 요청했으나
Skipped" — generate/synthesize them and re-export to actually preserve them.
건너뛰기" — 생성/합성하고 실제로 보존하기 위해 다시 내보냅니다.

If a selected item's `.xci`/`.bd` is located OUTSIDE `$proj_dir` (Remote/Mixed
선택한 항목의 `.xci`/`.bd`가 `$proj_dir` 외부에 있는 경우(원격/혼합)
scenario), it falls back to the standard export path instead — cache
시나리오) 대신 표준 내보내기 경로(캐시)로 대체됩니다.
preservation only ever captures what's inside `$proj_dir`, so there's
보존은 `$proj_dir` 내부의 내용만 캡처하므로 다음이 있습니다.
nothing local to copy for an external source.
외부 소스에 대해 복사할 로컬 항목은 없습니다.
There is no Tcl fallback
Tcl 대체가 없습니다.
written for a preserved item that succeeds: rebuild hard-errors instead if
성공한 보존 항목을 위해 작성됨: 대신 하드 오류를 다시 빌드합니다.
the cache is ever missing/incomplete (see
캐시가 누락되거나 불완전한 경우(참조
[generate-build-script/REFERENCE.md](../generate-build-script/REFERENCE.md)).
[생성-빌드-스크립트/REFERENCE.md](../생성-빌드-스크립트/REFERENCE.md)).

Every other, non-selected IP/BD in the same project is completely
동일한 프로젝트에서 선택되지 않은 다른 모든 IP/BD는 완전히
unaffected and continues through the standard export path below — this
영향을 받지 않고 아래의 표준 내보내기 경로를 통해 계속됩니다.
branching is per-item, not a project-wide switch.
분기는 프로젝트 전체 스위치가 아닌 항목별로 수행됩니다.

## What Gets Exported
## 내보낸 항목

| Category<br>카테고리 | Source<br>소스 | Destination<br>목적지 | Method<br>방법 |
|----------|--------|-------------|--------|
| RTL<br>RTL | Verilog/SV/VHDL/VHDL2019/VH/SVH in .srcs/<br>.srcs/의 Verilog/SV/VHDL/VHDL2019/VH/SVH | Sources/RTL/<br>소스/RTL/ | file copy<br>파일 복사 |
| Constraints<br>제약 | XDC in .srcs/ (not _ooc.xdc)<br>.srcs/의 XDC(_ooc.xdc 아님) | Sources/Constraints/<br>소스/제약조건/ | file copy<br>파일 복사 |
| Standalone IP<br>독립형 IP | IPs NOT inside Block Designs<br>블록 디자인 내부에 없는 IP | Sources/IP/<br>소스/IP/ | `write_ip_tcl`<br>`write_ip_tcl` |
| Block Designs<br>블록 디자인 | User-created BDs in .srcs/<br>.srcs/에 있는 사용자 생성 BD | Sources/BD/<br>소스/BD/ | `write_bd_tcl`<br>`write_bd_tcl` |
| Simulation<br>시뮬레이션 | Files from sim filesets<br>SIM 파일 세트의 파일 | Sources/Simulation/<br>소스/시뮬레이션/ | file copy<br>파일 복사 |
| Data<br>데이터 | .mem, .mif, .coe, .hex, .elf, .csv in project dir<br>프로젝트 디렉토리의 .mem, .mif, .coe, .hex, .elf, .csv | Sources/Data/<br>소스/데이터/ | file copy<br>파일 복사 |
| Hook Scripts<br>후크 스크립트 | Tcl pre/post hooks on synth/impl runs<br>synth/impl 실행 시 Tcl 사전/사후 후크 | Sources/Scripts/<br>소스/스크립트/ | file copy<br>파일 복사 |
| Pblocks (DFX)<br>P블록(DFX) | XDC containing pblock defs<br>pblock 정의를 포함하는 XDC | Sources/Constraints/<br>소스/제약조건/ | file copy<br>파일 복사 |
| DCPs (DFX)<br>DCP(DFX) | Locked static checkpoints<br>잠긴 정적 체크포인트 | Sources/Checkpoints/<br>소스/체크포인트/ | file copy<br>파일 복사 |
| NoC (Seg.Config)<br>NoC(세그먼트 구성) | .ncr solution files<br>.ncr 솔루션 파일 | Sources/NoC/<br>출처/NoC/ | file copy<br>파일 복사 |

## Critical Filtering Rules
## 중요한 필터링 규칙

**Skip generated files:** Any file with `IS_GENERATED == true` or in a `.gen/` directory is excluded.
**생성된 파일 건너뛰기:** `IS_GENERATED == true` 또는 `.gen/` 디렉토리에 있는 모든 파일은 제외됩니다.
These are auto-produced by Vivado and should not be version-controlled.
이는 Vivado에 의해 자동 생성되며 버전 관리가 되어서는 안 됩니다.

**Skip BD-embedded IPs:** IPs that are part of a Block Design (`IS_BD_CONTEXT == true`) are NOT exported via `write_ip_tcl` — they're already captured by `write_bd_tcl`.
**BD 내장 IP 건너뛰기:** 블록 설계(`IS_BD_CONTEXT == true`)의 일부인 IP는 `write_ip_tcl`을 통해 내보내지지 않습니다. 이미 `write_bd_tcl`에 의해 캡처되어 있습니다.
Attempting to export them separately produces warnings and redundant scripts.
별도로 내보내려고 하면 경고와 중복 스크립트가 생성됩니다.

**Skip auto-generated BDs:** BDs in `.gen/` or outside `.srcs/` are excluded.
**자동 생성된 BD 건너뛰기:** `.gen/` 또는 `.srcs/` 외부의 BD는 제외됩니다.
Only user-created BDs from the project's `.srcs/` directory are exported.
프로젝트의 `.srcs/` 디렉토리에서 사용자가 만든 BD만 내보냅니다.

**Local vs. remote constraint files:** each constraint fileset's file-tail list (used later by `generate_build_script` to rebuild `import_files`/`add_files` calls) only includes files that `_rc_is_exportable_local` confirms were actually copied into `Sources/Constraints`.
**로컬 대 원격 제약 파일:** 각 제약 파일 세트의 파일 꼬리 목록(나중에 `import_files`/`add_files` 호출을 다시 빌드하기 위해 `generate_build_script`에 의해 사용됨)에는 `_rc_is_exportable_local`이 실제로 `Sources/Constraints`에 복사되었음을 확인하는 파일만 포함됩니다.
Files outside the project directory (Mixed/Remote scenario) are never copied — they're referenced in-place via the "Remote constraints" `add_files` block instead — so listing them here would target a path `build.tcl` never creates, aborting the rebuild.
프로젝트 디렉토리 외부의 파일(혼합/원격 시나리오)은 복사되지 않습니다. 대신 "원격 제약 조건" `add_files` 블록을 통해 내부에서 참조됩니다. 따라서 여기에 나열하면 `build.tcl`이 생성되지 않는 경로를 대상으로 하여 재구축이 중단됩니다.
This also governs DFX pblock XDCs: a *local* pblock XDC is copied into `Sources/Constraints` and must stay in the list, or `build.tcl` silently drops the pblock constraint; a *remote* one must not appear here, or it gets double-imported.
이는 DFX pblock XDC에도 적용됩니다. *local* pblock XDC는 `Sources/Constraints`에 복사되고 목록에 유지되어야 합니다. 그렇지 않으면 `build.tcl`이 자동으로 pblock 제약 조건을 삭제합니다. *원격* 항목은 여기에 표시되어서는 안 됩니다. 그렇지 않으면 이중으로 가져오게 됩니다.

**generate wrapper for BD:** If BD wrapper is created in `.gen/`.
**BD용 래퍼 생성:** BD 래퍼가 `.gen/`에 생성된 경우.
Check for .v or .vhd with name `bd_name_wrapper`.
이름이 `bd_name_wrapper`인 .v 또는 .vhd를 확인하세요.
If `bd_name_wrapper` is added to the top hierarchy.
최상위 계층에 `bd_name_wrapper`가 추가된 경우.
If above both are true, Then generate flag bd_name_wrapper_flag = 1. repeat the same for all the User-created BDs in .srcs
위의 두 가지가 모두 true이면 bd_name_wrapper_flag = 1 플래그를 생성합니다. .srcs의 모든 사용자 생성 BD에 대해 동일한 작업을 반복합니다.
## Edge Cases
## 엣지 케이스

- **Encrypted IP** — `write_ip_tcl` may fail; use Remote strategy instead
- **암호화된 IP** — `write_ip_tcl`이 실패할 수 있습니다. 대신 원격 전략을 사용하세요
- **Flat RTL hierarchy** — all files land in Sources/RTL/ regardless of original subfolder structure
- **평평한 RTL 계층 구조** — 원래 하위 폴더 구조에 관계없이 모든 파일이 Sources/RTL/에 위치합니다.
- **Multiple constraint filesets** — all XDC files from all constraint filesets are exported
- **다중 제약 파일 세트** — 모든 제약 파일 세트의 모든 XDC 파일을 내보냅니다.
- **DFX without checkpoints** — if no locked DCPs exist yet, a note is printed; user must export after implementation
- **체크포인트가 없는 DFX** — 잠긴 DCP가 아직 없으면 메모가 인쇄됩니다. 사용자는 구현 후 내보내야 합니다.

## When to Use vivado_doc_search
## vivado_doc_search를 사용해야 하는 경우

- If `write_ip_tcl` or `write_bd_tcl` produces unexpected output, search for their documentation
- `write_ip_tcl` 또는 `write_bd_tcl`이 예상치 못한 출력을 생성하는 경우 해당 문서를 검색하세요.
- If unsure about IP file types or BD properties, search for `get_property IS_BD_CONTEXT` or `get_ips`
- IP 파일 형식이나 BD 속성이 확실하지 않은 경우 `get_property IS_BD_CONTEXT` 또는 `get_ips`를 검색하세요.
