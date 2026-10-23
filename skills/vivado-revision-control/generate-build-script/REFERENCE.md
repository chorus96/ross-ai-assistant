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
name: generate-build-script
이름: 생성-빌드-스크립트
description: Step 5 — generates automated build.tcl for one-command project recreation.
설명: 5단계 — 단일 명령 프로젝트 재생성을 위해 자동화된 build.tcl을 생성합니다.
---

# Generate Build Script (Step 5)
# 빌드 스크립트 생성(5단계)

**Source file:** `helper-procedures/helper_scripts.tcl`
**소스 파일:** `helper-procedures/helper_scripts.tcl`
**Proc name:** `generate_build_script`
**프록 이름:** `generate_build_script`

## Procedure
## 절차

```tcl
generate_build_script "RevisionControl/Scripts/build.tcl" "RevisionControl" $scenario_info
```

**Parameters:**
**매개변수:**
- `output_file` (string) — path for generated build.tcl
- `output_file`(문자열) — 생성된 build.tcl의 경로
- `output_dir` (string) — base directory containing Sources/ and Scripts/
- `output_dir`(문자열) — Sources/ 및 Scripts/를 포함하는 기본 디렉터리
- `scenario_info` (dict, optional) — output from Step 2 (`analyze_source_locations`).
- `scenario_info`(dict, 선택사항) — 2단계의 출력(`analyze_source_locations`).
- Used to determine `import_files` (local) vs `add_files` (remote) and to emit remote source paths in build.tcl.
- `import_files`(로컬)과 `add_files`(원격)을 결정하고 build.tcl에서 원격 소스 경로를 내보내는 데 사용됩니다.

**Returns:** the output file path.
**반환:** 출력 파일 경로.

## Generated Build Script Structure
## 생성된 빌드 스크립트 구조

The build.tcl follows this sequence:
build.tcl은 다음 순서를 따릅니다.

1. **Create project** — `create_project` with correct PART and BOARD_PART
1. **프로젝트 생성** — 올바른 PART 및 BOARD_PART를 사용하여 `create_project`
(1b: set IP_REPO_PATHS; 1c: restore any preserved outputs from the
(1b: IP_REPO_PATHS 설정; 1c: 보존된 출력을 다음에서 복원합니다.
external remote cache — see "Remote Cache Reuse" below — BEFORE anything
외부 원격 캐시 — 아래 "원격 캐시 재사용" 참조 — 무엇보다 먼저
else touches IPs/BDs; hard-errors if the cache is missing/incomplete)
그렇지 않으면 IP/BD에 닿습니다. 캐시가 없거나 불완전한 경우 하드 오류)
2. **Add RTL** — `import_files` for local sources from Sources/RTL/; `add_files` for remote sources (if Mixed scenario)
2. **RTL 추가** — Sources/RTL/의 로컬 소스에 대한 `import_files`; 원격 소스에 대한 `add_files`(혼합 시나리오인 경우)
3. **Add constraints** — add XDC files to constrs_1 fileset
3. **제약조건 추가** — XDC 파일을 constrs_1 파일 세트에 추가합니다.
4. **Add simulation** — add files to sim_1 fileset
4. **시뮬레이션 추가** — sim_1 파일 세트에 파일 추가
5. **Add data files** — add .mem, .coe, .csv, etc.
5. **데이터 파일 추가** - .mem, .coe, .csv 등을 추가합니다.
6. **Recreate IPs** — source each IP Tcl script from Sources/IP/, then `create_ip_run` for standalone OOC IPs.
6. **IP 재생성** — Sources/IP/에서 각 IP Tcl 스크립트를 소싱한 다음 독립형 OOC IP에 대해 'create_ip_run'을 사용합니다.
6. Preserved IPs (see below) are `add_files`'d from the restored `<proj_name>.srcs/<rel_dir>/` instead, per-IP, alongside this step.
6. 보존된 IP(아래 참조)는 이 단계와 함께 IP별로 복원된 `<proj_name>.srcs/<rel_dir>/`에서 `add_files`됩니다.
7. **Recreate BDs** — source each BD Tcl script from Sources/BD/.
7. **BD 재생성** — Sources/BD/에서 각 BD Tcl 스크립트를 소싱합니다.
7. Preserved BDs (see below) are `add_files`'d from the restored `<proj_name>.srcs/<rel_dir>/` instead, per-BD, in the same dependency-ordered loop.
7. 보존된 BD(아래 참조)는 동일한 종속성 순서 루프에서 BD별로 복원된 `<proj_name>.srcs/<rel_dir>/`에서 `add_files`입니다.
8. **Create BD wrappers** — `make_wrapper` + `import_files` for BDs that had wrappers in `.gen/` (preserved BDs are tracked through the same `bd_name_array` mechanism, so an auto-managed-wrapper preserved BD still gets its wrapper created)
8. **BD 래퍼 생성** — `.gen/`에 래퍼가 있는 BD의 경우 `make_wrapper` + `import_files`(보존된 BD는 동일한 `bd_name_array` 메커니즘을 통해 추적되므로 자동 관리 래퍼로 보존된 BD는 여전히 래퍼가 생성됩니다)
9. **Generate BD targets + IP runs** — `generate_target all` on each BD, then `create_ip_run` on BD files
9. **BD 대상 생성 + IP 실행** — 각 BD에서 `generate_target all`을 수행한 다음 BD 파일에서 `create_ip_run`
10. **Apply settings** — `source project_settings.tcl` (BEFORE finalize so TOP, PR_FLOW etc. are set)
10. **설정 적용** — `source project_settings.tcl`(TOP, PR_FLOW 등이 설정되도록 마무리하기 전)
11. **Finalize** — `update_compile_order` for sources and sim filesets
11. **완료** — 소스 및 SIM 파일 세트에 대한 `update_compile_order`

All paths are relative to the script location using `[info script]`.
모든 경로는 `[info script]`를 사용하는 스크립트 위치를 기준으로 합니다.

## Remote Cache Reuse: Preserved IPs/BDs (optional, per-item)
## 원격 캐시 재사용: 보존된 IP/BD(선택 사항, 항목별)

If Step 2.5 (`prompt_preserve_outputs`/`build_preserve_selection`) selected
2.5단계(`prompt_preserve_outputs`/`build_preserve_selection`)를 선택한 경우
one or more IPs/BDs, those specific items skip the standard
하나 이상의 IP/BD, 해당 특정 항목은 표준을 건너뜁니다.
Tcl-regeneration path entirely.
Tcl 재생 경로 전체.
As of 2026-09-29 (GP methodology feedback),
2026년 9월 29일 기준(GP 방법론 피드백),
the source of truth for a preserved item's `.srcs`/`.gen` is an **external
보존된 항목의 `.srcs`/`.gen`에 대한 정보의 출처는 **외부입니다.
remote cache directory** — a path outside the project, supplied by the user
원격 캐시 디렉터리** — 사용자가 제공한 프로젝트 외부 경로
and baked into `build.tcl` as a literal absolute path (`preserve_cache_dir`).
리터럴 절대 경로(`preserve_cache_dir`)로 `build.tcl`에 구워졌습니다.
Unlike the project itself, this cache is never touched by `create_project
프로젝트 자체와는 달리 이 캐시는 `create_project'에 의해 절대 건드리지 않습니다.
-force`, so `generate_build_script` reads from it directly — no intermediate
-force`, 따라서 `generate_build_script`가 직접 읽습니다 — 중간 단계 없음
staging directory is needed:
준비 디렉터리가 필요합니다:

- **Step 1c** (runs right after `create_project`/IP repo setup):
- **1c단계**(`create_project`/IP 저장소 설정 직후 실행):
  1. First checks `[file isdirectory $preserve_cache_dir]` — if the cache
  1. 먼저 `[file isdirectory $preserve_cache_dir]`을 확인합니다. — 캐시가
root itself is missing, it **hard-errors** (`error`) and the whole
루트 자체가 누락된 경우 **하드 오류**(`error`)가 발생하고 전체
rebuild stops.
재구축이 중지됩니다.
  2. For each preserved IP/BD, checks that its own `<cache_dir>/<proj_name>/
  2. 보존된 각 IP/BD에 대해 자체 `<cache_dir>/<proj_name>/을 확인합니다.
{ip,bd}/<name>/{srcs,gen}/` subfolders both exist — if either is
{ip,bd}/<name>/{srcs,gen}/` 하위 폴더가 모두 존재합니다. 둘 중 하나가 있는 경우
missing, it **hard-errors** for that item (no fallback to
누락되었습니다. 해당 항목에 **하드 오류**가 발생했습니다(대체는 없음).
regeneration).
재생).
  3. Otherwise, copies that item's cached `srcs/*` into the new project's own
  3. 그렇지 않으면 해당 항목의 캐시된 `srcs/*`를 새 프로젝트의 자체 항목에 복사합니다.
`<proj_name>.srcs/<rel_dir>/` and cached `gen/*` into
`<proj_name>.srcs/<rel_dir>/` 및 `gen/*`에 캐시됨
`<proj_name>.gen/<rel_dir>/`, where `<rel_dir>` is the exact
`<proj_name>.gen/<rel_dir>/`, 여기서 `<rel_dir>`은 정확한 이름입니다.
fileset-relative path `export_all_sources` recorded for that item (not
해당 항목에 대해 기록된 파일 세트 상대 경로 'export_all_sources'(아님)
a hardcoded `sources_1` guess — a BD/IP can live in a different
하드코딩된 `sources_1` 추측 — BD/IP는 다른 위치에 있을 수 있습니다.
fileset, e.g. a simulation-only BD under `sim_1/bd/<name>/`, confirmed
파일 세트(예: `sim_1/bd/<name>/` 아래의 시뮬레이션 전용 BD, 확인됨
on this skill's own `modnoc-rtl-flat` example design).
이 스킬 자체의 `modnoc-rtl-Flat` 예제 디자인에 대해).
This plants the
이는 식물을
preserved generated output where Vivado will look for it, before that
그 전에 Vivado가 검색할 생성된 출력을 보존합니다.
IP/BD is even added to the project.
IP/BD도 프로젝트에 추가됩니다.
- **Steps 6/7** then `add_files` the restored `.xci`/`.bd` from
- **6/7단계** 그런 다음 복원된 `.xci`/`.bd`를 `add_files`합니다.
`<proj_name>.srcs/<rel_dir>/` (instead of `source`-ing a
`<proj_name>.srcs/<rel_dir>/`(`source` 대신
`write_ip_tcl`/`write_bd_tcl` script) and call `generate_target`.
`write_ip_tcl`/`write_bd_tcl` 스크립트) `generate_target`을 호출합니다.
Because
왜냐하면
the matching `.gen` output was already restored in Step 1c, Vivado's own
일치하는 `.gen` 출력은 Vivado 자체의 1c단계에서 이미 복원되었습니다.
out-of-date check sees the generated products as current and does not
오래된 확인은 생성된 제품을 최신 제품으로 간주하고 그렇지 않습니다.
actually resynthesize/rebuild them — this relies on Vivado's normal
실제로 재합성/재구축 - 이는 Vivado의 정상에 의존합니다.
incremental-generation behavior, not a special flag.
특수 플래그가 아닌 증분 생성 동작입니다.
Since Step 1c already
이미 1c단계부터
guarantees these sources exist (erroring out the build otherwise), Steps
이러한 소스가 존재함을 보장합니다(그렇지 않으면 빌드 오류 발생).
6/7 do not re-check for them.
6/7은 다시 확인하지 마세요.
- This is a **per-IP/per-BD** decision, not a project-wide mode switch: any
- 이는 프로젝트 전체 모드 전환이 아닌 **IP별/BD별** 결정입니다.
IP/BD not named in `preserve_selection` goes through the exact same
`preserve_selection`에 이름이 지정되지 않은 IP/BD는 완전히 동일한 과정을 거칩니다.
`write_ip_tcl`/`write_bd_tcl` → `source` → `create_ip_run`/`generate_target`
`write_ip_tcl`/`write_bd_tcl` → `source` → `create_ip_run`/`generate_target`
path as before this feature existed, in the same build.tcl, in the same
이 기능이 존재하기 전과 동일한 build.tcl의 경로입니다.
project.
프로젝트.

Copying the whole per-BD `.srcs` directory at export time (not just the
내보내기 시 전체 BD별 `.srcs` 디렉토리 복사(단순히
lone `<bd>.bd` file) means a per-instance IP customized only inside a
`<bd>.bd` 파일만 있음)은 인스턴스 내부에서만 사용자 정의된 인스턴스별 IP를 의미합니다.
preserved BD is preserved too — this closes a gap present in an earlier
보존된 BD도 보존됩니다. 이는 이전 버전에 존재했던 격차를 해소합니다.
(pre-2026-09-29) version of this mechanism, where only the top-level `.bd`
(2026년 9월 29일 이전) 버전의 이 메커니즘에서는 최상위 `.bd`만
file was captured and nested `ip/<name>/<name>.xci` files could go missing
파일이 캡처되어 중첩된 `ip/<name>/<name>.xci` 파일이 누락될 수 있습니다.
on rebuild.
재건축 중.

### Missing-cache handling (hard failure, by design)
### 누락된 캐시 처리(설계상 하드 오류)

If the remote cache directory, or a specific preserved item's `srcs`/`gen`
원격 캐시 디렉터리 또는 특정 보존 항목의 `srcs`/`gen`인 경우
data inside it, is missing at rebuild time (wrong path, cache not
그 안에 있는 데이터가 재구축 시 누락되었습니다(잘못된 경로, 캐시가 아님).
mounted/restored, export never finished for that item), Step 1c raises a
마운트/복원됨, 해당 항목에 대한 내보내기가 완료되지 않음), 1c단계에서는
Tcl `error` and the whole `build.tcl` run stops — it does **not** fall back
Tcl `error` 및 전체 `build.tcl` 실행이 중지됩니다. — 폴백되지 **않습니다**
to Tcl regeneration for that item.
해당 항목에 대한 Tcl 재생성.
This is intentional: the user explicitly
이는 의도적인 것입니다. 사용자가 명시적으로
opted into a deterministic, cache-backed rebuild for that IP/BD, so a
해당 IP/BD에 대해 결정론적인 캐시 지원 재구축을 선택했기 때문에
missing cache is a setup problem worth surfacing loudly, not a "just
캐시 누락은 "단순한 문제"가 아니라 크게 언급할 가치가 있는 설정 문제입니다.
resynthesize instead" situation to paper over silently.
대신 재합성하라'는 상황은 조용히 끝났다.

## Backward Compatibility
## 이전 버전과의 호환성

`preserved_ip_names`/`preserved_bd_names` (and the matching
`preserved_ip_names`/`preserved_bd_names`(및 일치하는 항목
`preserved_ip_rel_path`/`preserved_bd_rel_path` fileset-relative paths, plus
`preserved_ip_rel_path`/`preserved_bd_rel_path` 파일 세트 상대 경로 및
`preserved_cache_dir`) are read from globals populated by
`preserved_cache_dir`)은 다음에 의해 채워진 전역에서 읽혀집니다.
`export_all_sources`; if that proc was called without a `preserve_selection`
`export_all_sources`; 해당 프로시저가 `preserve_selection` 없이 호출된 경우
(the default, and the only option before this feature existed), both name
(기본값이자 이 기능이 존재하기 전의 유일한 옵션), 두 이름 모두
lists are empty and `generate_build_script` emits byte-for-byte the same
목록은 비어 있고 `generate_build_script`는 바이트 단위로 동일한 내용을 내보냅니다.
Steps 6/7/9 as before — no Step 1c is even emitted.
이전과 마찬가지로 6/7/9단계 — 1c단계도 생성되지 않습니다.
Older exported
이전 수출
`RevisionControl/` trees (from before this feature existed, or from the
`RevisionControl/` 트리(이 기능이 존재하기 전 또는
2026-08-23–2026-09-29 full-tree-copy design) rebuild exactly as before —
2026-08-23–2026-09-29 전체 트리 복사 디자인) 이전과 동일하게 재구축 —
nothing in `generate_build_script` checks for any of these unless it
`generate_build_script`에서는 아무것도 확인하지 않습니다.
already has a preserved name to look up.
이미 검색할 보존된 이름이 있습니다.

## Critical: Settings BEFORE Finalize
## 중요: 마무리 전 설정

`project_settings.tcl` is sourced in Step 10, after all sources, IPs, and BDs
`project_settings.tcl`은 10단계에서 모든 소스, IP 및 BD 뒤에 소스가 제공됩니다.
are added but BEFORE `update_compile_order`.
'update_compile_order' 전에 추가됩니다.
Properties like TOP, PR_FLOW, and
TOP, PR_FLOW 및 같은 속성
VERILOG_DEFINE affect how Vivado resolves compile order.
VERILOG_DEFINE은 Vivado가 컴파일 순서를 해결하는 방법에 영향을 미칩니다.
IP OOC runs must exist
IP OOC 실행이 존재해야 합니다.
(created in Steps 6 and 9) before settings for those runs can be applied.
(6단계와 9단계에서 생성) 해당 실행에 대한 설정을 적용하기 전에.

## Critical: IP OOC Runs
## 위험: IP OOC 실행

For standalone IPs (outside Block Designs), `create_ip_run` is called in Step 6
독립형 IP(블록 디자인 외부)의 경우 6단계에서 `create_ip_run`이 호출됩니다.
after sourcing the IP Tcl scripts.
IP Tcl 스크립트를 소싱한 후.
For IPs inside Block Designs, Step 9 runs
블록 디자인 내부 IP의 경우 9단계가 실행됩니다.
`generate_target all` on each BD followed by `create_ip_run` using:
다음을 사용하여 각 BD에서 'generate_target all'을 실행하고 'create_ip_run'을 실행합니다.
```tcl
create_ip_run [get_files -of_objects [get_fileset sources_1] $bd]
```
This ensures OOC synthesis runs exist before project settings are applied.
이렇게 하면 프로젝트 설정이 적용되기 전에 OOC 합성 실행이 존재하도록 보장됩니다.

## DFX: RM Ordering
## DFX: RM 주문

For DFX projects, RM (Reconfigurable Module) Block Designs must be sourced
DFX 프로젝트의 경우 RM(Reconfigurable Module) 블록 설계를 소싱해야 합니다.
BEFORE the static design in Step 7.
7단계의 정적 설계 전.
The static design's Block Design Containers
정적 디자인의 블록 디자인 컨테이너
reference RMs by name — if the RMs don't exist yet, BDC instantiation fails:
이름으로 RM 참조 — RM이 아직 존재하지 않으면 BDC 인스턴스화가 실패합니다.

```
ERROR: [BD 41-1279] Block Container 'rp1_container' is referencing
an instance 'rp1rm1' that does not exist in the design.
```

The current `generate_build_script` procedure captures BD dependency ordering
현재 `generate_build_script` 프로시저는 BD 종속성 순서를 캡처합니다.
and emits child-before-parent sourcing in the generated build.tcl.
생성된 build.tcl에서 상위 이전 하위 소스를 내보냅니다.
This is
이것은
intended to preserve correct ordering for multi-BD and BDC designs, including
다음을 포함하여 다중 BD 및 BDC 설계의 올바른 순서를 유지하기 위한 것입니다.
DFX projects where RM-owned BDs must not be recreated as independent top-level
RM 소유의 BD를 독립적인 최상위 레벨로 재생성해서는 안 되는 DFX 프로젝트
BDs.
BD.

## Known Limitation: BDC Debug Hierarchies with Monitor-Mode Interfaces
## 알려진 제한 사항: 모니터 모드 인터페이스가 있는 BDC 디버그 계층

Some `make_wrapper`/`validate_bd_design` failures on a BDC (Block Design
BDC에서 일부 `make_wrapper`/`validate_bd_design` 실패(블록 설계)
Container) child are **not** fixable by reordering the generated `build.tcl`
컨테이너) 하위 항목은 생성된 `build.tcl`을 재정렬하여 수정할 수 **없습니다**
and are **not** a locked/out-of-date IP problem (see below) — they are a
잠겨 있거나 오래된 IP 문제가 **아닙니다**(아래 참조).
genuine Vivado limitation in how `write_bd_tcl` exports (and Tcl subsequently
`write_bd_tcl`을 내보내는 방식에 대한 진정한 Vivado 제한 사항(및 이후 Tcl)
replays) a specific interface pattern.
재생) 특정 인터페이스 패턴.

**Pattern:** a BDC child containing hierarchy-wide debug IP (e.g.
**패턴:** 계층 전체 디버그 IP를 포함하는 BDC 하위(예:
`axi_dbg_hub` + `axis_ila` + `axis_vio`) that exposes a **Monitor-mode**
**모니터 모드**를 노출하는 `axi_dbg_hub` + `axis_ila` + `axis_vio`)
interface pin (e.g.
인터페이스 핀(예:
`SLOT_0_AXI`) wired to the parent design via
`SLOT_0_AXI`)를 통해 상위 디자인에 연결됨
`connect_bd_intf_net`.
`connect_bd_intf_net`.
Observed on `ipi_bdc_flat_flow` (Versal `xcve2802`,
`ipi_bdc_plat_flow`에서 관찰됨(Versal `xcve2802`,
BD `design_1` → BDC `debug_hier`):
BD `design_1` → BDC `debug_hier`):

```
ERROR: [BD 41-258] Could not find the cell: 'debug_hier_inst_0' at the level
of hierarchy: 'design_1', when trying to connect the interface net:
'axi_noc_0_M00_AXI_RID' to interface: '/SLOT_0_AXI_rid'
```

A Monitor interface's AXI ID-width sub-pins (`_rid`, `_bid`, etc.) are
모니터 인터페이스의 AXI ID 너비 하위 핀(`_rid`, `_bid` 등)은 다음과 같습니다.
decomposed based on the connected master's ID width, which Vivado resolves by
Vivado가 해결하는 연결된 마스터의 ID 너비를 기반으로 분해됩니다.
walking the full design hierarchy — something it does when the design is
전체 디자인 계층 구조 살펴보기 — 디자인이 다음과 같을 때 수행되는 작업
built interactively, but which does not happen correctly when Tcl-replaying
대화형으로 구축되었지만 Tcl 재생 시 올바르게 발생하지 않습니다.
an exported design.
수출된 디자인.
An ordinary `S_AXI` interface on the same cell
동일한 셀의 일반 `S_AXI` 인터페이스
materializes fine; the Monitor interface's sub-pins never appear.
괜찮아요; 모니터 인터페이스의 하위 핀은 절대 나타나지 않습니다.

**Confirmed not an ordering bug:** tested 5 independent orderings of
**주문 버그가 아닌 것으로 확인됨:** 5개의 독립적인 주문을 테스트했습니다.
`generate_target`/`validate_bd_design`/the failing `connect_bd_intf_net` call
`generate_target`/`validate_bd_design`/`connect_bd_intf_net` 호출 실패
against real Versal-capable Vivado (2026.2) on the actual repro — moving
실제 Versal 지원 Vivado(2026.2)에 대한 실제 재현 — 이동
`generate_target` earlier/later, validating immediately before the connect,
`generate_target` 이전/나중에, 연결 직전에 유효성을 검사하고,
and moving the connect itself to the end of the BD script.
연결 자체를 BD 스크립트의 끝으로 이동합니다.
All five produced
5개 모두 생산
the identical error.
동일한 오류.
Direct pin inspection confirmed the sub-pins simply
직접 핀 검사로 간단하게 서브 핀 확인
never materialize via Tcl replay, regardless of sequencing.
순서에 관계없이 Tcl 재생을 통해 실현되지 않습니다.

Because `validate_bd_design` fails on the BDC child, that BD is left invalid
BDC 하위 항목에서 `validate_bd_design`이 실패하므로 해당 BD는 유효하지 않은 상태로 남습니다.
for the rest of the session — so the parent's `make_wrapper` call then hits
나머지 세션 동안 — 따라서 상위의 `make_wrapper` 호출이 히트합니다.
Vivado's generic `ERROR: [Common 17-39] 'make_wrapper' failed due to earlier
Vivado의 일반 `ERROR: [Common 17-39] 'make_wrapper'가 이전 문제로 인해 실패했습니다.
errors` guard.
오류 가드.
This is a cascading symptom of the same root cause, not an
이는 동일한 근본 원인의 계단식 증상이며,
independent bug.
독립적인 버그.

**Do not** reintroduce a per-child `generate_target` call in Step 7, or
**하지 마세요** 7단계에서 하위 항목별 'generate_target' 호출을 다시 도입하지 마세요.
otherwise reorder `generate_build_script`'s BD sourcing, to try to fix this —
그렇지 않으면 `generate_build_script`의 BD 소싱을 재정렬하여 이 문제를 해결해 보세요 —
that path is exhausted.
그 길은 지쳤습니다.
A real fix requires either (a) Xilinx fixing
실제 수정에는 (a) Xilinx 수정이 필요합니다.
`write_bd_tcl`'s handling of this pattern, or (b) the skill detecting
`write_bd_tcl`의 이 패턴 처리 또는 (b) 기술 감지
Monitor-mode BDC boundary interfaces at export time and special-casing them
내보내기 시 모니터 모드 BDC 경계 인터페이스 및 특수 케이스
(documenting a manual "Set Up Debug" wizard reconnect instead of relying on
(수동 "디버그 설정" 마법사에 의존하는 대신 재연결을 문서화함)
Tcl replay).
Tcl 재생).

## Known Limitation: NoC Memory-Controller PHY IP Missing `.elf`/Simulation Outputs
## 알려진 제한: NoC 메모리 컨트롤러 PHY IP 누락 `.elf`/시뮬레이션 출력

Designs containing a NoC IP with a memory controller (e.g.
메모리 컨트롤러와 함께 NoC IP를 포함하는 설계(예:
`axi_noc_0` /
`axi_noc_0` /
`MC0_ddrc_0`) can fail an OOC synthesis run with:
`MC0_ddrc_0`)은 다음과 같은 경우 OOC 합성 실행에 실패할 수 있습니다.

```
ERROR: [Common 17-275] File does not exist [.../bd_<hash>_MC0_ddrc_0_phy_ddrmc.elf]
```

or fail `generate_target` for the containing top-level BD outright, e.g.:
또는 포함된 최상위 BD에 대해 `generate_target`을 완전히 실패합니다. 예:

```
CRITICAL WARNING: [IP_Flow 19-663] Failed to copy file
'<vivado_install>/data/ip/xilinx/noc_mc_ddr5_phy_v1_0/hdl/bfm/PINOUT/map_read_19.sv',
it does not exist.
ERROR: [IP_Flow 19-3505] IP Generation error: Failed to generate IP
'.../bd_<hash>_MC0_ddrc_0_phy'. Failed to generate 'Verilog Simulation' outputs.
ERROR: [BD 41-1030] Generation failed for the IP Integrator block MC0_ddrc
ERROR: generate_target failed for <top>.bd: ERROR: [Common 17-39]
'generate_target' failed due to earlier errors.
```

**Confirmed root cause: a Vivado *installation* gap, not an export/rebuild
**확인된 근본 원인: 내보내기/재구축이 아닌 Vivado *설치* 공백
bug.** The NoC memory-controller PHY IP (`noc_mc_ddr5_phy_v1_0` in this
버그.** NoC 메모리 컨트롤러 PHY IP(여기서는 `noc_mc_ddr5_phy_v1_0`)
install) generates its simulation BFM output by copying numbered
설치)은 번호를 복사하여 시뮬레이션 BFM 출력을 생성합니다.
`map_read_<N>.sv` / `pinout_<N>.sv` files from the Vivado install's own
Vivado 설치 자체의 `map_read_<N>.sv` / `pinout_<N>.sv` 파일
`data/ip/xilinx/noc_mc_ddr5_phy_v1_0/hdl/bfm/PINOUT/` directory.
`data/ip/xilinx/noc_mc_ddr5_phy_v1_0/hdl/bfm/PINOUT/` 디렉토리.
On the
에
install this was diagnosed against
이 설치에 대해 진단되었습니다.
(`2026.2`, `/proj/xbuilds/CustTA_plus/9999.0_2026_0724_2226plus/...`), several
(`2026.2`, `/proj/xbuilds/CustTA_plus/9999.0_2026_0724_2226plus/...`), 여러
indices are missing from that directory entirely (confirmed by direct
해당 디렉토리에서 색인이 완전히 누락되었습니다(직접 확인됨).
filesystem inspection: `pinout_14.sv` and `pinout_19.sv`/`map_read_19.sv` do
파일 시스템 검사: `pinout_14.sv` 및 `pinout_19.sv`/`map_read_19.sv` 수행
not exist, while neighboring indices do) — so **any** attempt to generate
존재하지 않지만 이웃 인덱스는 존재함) — **모든** 생성을 시도합니다.
this IP's targets on this install fails identically, whether from the
이 설치에서 이 IP의 대상은 동일하게 실패합니다.
original project, a from-scratch rebuild, or a completely unexported scratch
원본 프로젝트, 처음부터 다시 빌드 또는 완전히 내보내지 않은 스크래치
copy.
복사.
Confirmed further: none of the 9 `vivado-revision-control` example
추가 확인: 9개의 `vivado-revision-control` 예시 중 없음
designs ship a pre-generated `.gen`/`.elf` for this IP (they're lightweight,
설계에서는 이 IP에 대해 사전 생성된 `.gen`/`.elf`를 제공합니다(가벼우며,
sources-only checkouts), so there is nothing valid to copy through export
소스 전용 체크아웃), 따라서 내보내기를 통해 복사할 수 있는 유효한 항목이 없습니다.
either — a "preserve/copy the existing artifact" fix (the pattern already
— "기존 아티팩트 보존/복사" 수정(패턴은 이미
used for `preserve_selection`) is not viable here; there is no working
`preserve_selection`에 사용됨)은 여기에서 실행 가능하지 않습니다. 일이 없다
artifact anywhere to preserve.
보존할 수 있는 유물

**Not fixable from this skill.** No `export_all_sources`/`generate_build_script`
**이 스킬로는 수정할 수 없습니다.** `export_all_sources`/`generate_build_script` 없음
change can synthesize a Vivado IP-library file that doesn't exist on disk.
변경은 디스크에 존재하지 않는 Vivado IP 라이브러리 파일을 합성할 수 있습니다.
Affected projects: `modnoc-rtl-dfx`, `ipi_bdc_dfx` (fails at OOC-synth time),
영향을 받는 프로젝트: `modnoc-rtl-dfx`, `ipi_bdc_dfx`(OOC-synth 시간에 실패),
and `segmented_config` (fails earlier, at `generate_target`, but the
및 `segmented_config`(이전에는 `generate_target`에서 실패했지만
*current* `verify_rebuild`/`run_project.sh` structural checks do not catch
*현재* `verify_rebuild`/`run_project.sh` 구조 검사가 포착되지 않음
this — see the transcript-ERROR-scan check added to `verify_rebuild` for a
이것 — `verify_rebuild`에 추가된 Transcript-ERROR-scan 검사를 참조하세요.
partial mitigation: it turns this into a visible `[FAIL]`/`WARN` instead of a
부분 완화: 이를 표시하는 대신 눈에 보이는 `[FAIL]`/`WARN`으로 바꿉니다.
silent `PASS`).
침묵 `PASS`).
If you hit this on a different Vivado install, check whether
다른 Vivado 설치에서 이 문제가 발생하면 다음을 확인하십시오.
`data/ip/xilinx/noc_mc_ddr5_phy_v1_0/hdl/bfm/PINOUT/` is complete on that
`data/ip/xilinx/noc_mc_ddr5_phy_v1_0/hdl/bfm/PINOUT/`이 완료되었습니다.
install; if it's not, this is an install/patch issue for whoever owns that
설치; 그렇지 않은 경우 이는 해당 소유자의 설치/패치 문제입니다.
Vivado deployment, not something to re-investigate here.
Vivado 배포는 여기서 다시 조사할 내용이 아닙니다.

## Locked/Out-of-Date IPs Can Surface as a make_wrapper Failure
## 잠겨 있거나 오래된 IP가 make_wrapper 실패로 나타날 수 있음

A `make_wrapper` failure during rebuild is frequently a symptom, not the root
재구축 중 `make_wrapper` 오류는 근본 원인이 아닌 증상인 경우가 많습니다.
cause:
원인:

```
ERROR: [Common 17-39] 'make_wrapper' failed due to earlier errors.
```

This generic Vivado message means the session already has a logged error from
이 일반 Vivado 메시지는 세션에 이미 로그된 오류가 있음을 의미합니다.
an earlier command — usually `validate_bd_design` failing on a BD (often a
이전 명령 - 일반적으로 BD에서 `validate_bd_design`이 실패합니다(종종
Block Design Container child) that contains locked or out-of-date IPs.
잠겨 있거나 오래된 IP가 포함된 블록 디자인 컨테이너 하위)입니다.
Once
한 번
Vivado logs that error, later unrelated commands like `make_wrapper` can also
Vivado는 해당 오류를 기록하며 나중에 `make_wrapper`와 같은 관련 없는 명령도 가능합니다.
fail with the same "failed due to earlier errors" message, which hides the
동일한 "이전 오류로 인해 실패했습니다"라는 메시지와 함께 실패합니다.
real cause.
진짜 원인.

The generated build.tcl wraps `validate_bd_design`, the BDC child's
생성된 build.tcl은 BDC 하위 항목인 `validate_bd_design`을 래핑합니다.
`generate_target all`, and `make_wrapper` in `catch` blocks (Steps 7 and 8) so
`catch` 블록의 `generate_target all` 및 `make_wrapper`(7단계 및 8단계)
each prints its own diagnostic instead of aborting silently.
각각은 자동으로 중단하는 대신 자체 진단을 인쇄합니다.
When you see a
당신이 볼 때
`make_wrapper` failure, scroll back up the rebuild log for a `WARNING:
`make_wrapper` 실패, `경고:'에 대한 재구축 로그를 위로 스크롤하세요.
validate_bd_design reported an error for <bd>` or `WARNING: generate_target
verify_bd_design이 <bd>` 또는 `WARNING: generate_target에 대한 오류를 보고했습니다.
failed for <bd>` — that earlier message names the actual BD at fault.
failed for <bd>` — 이전 메시지에서는 실제 BD에 결함이 있다고 명시했습니다.

**Fix:** on the ORIGINAL project (not the recreated one):
**수정:** 원래 프로젝트(재작성된 프로젝트 아님):
```tcl
open_bd_design <bd_name>.bd
report_ip_status
```
then `upgrade_ip` the flagged IPs and re-run the export/build pipeline.
그런 다음 플래그가 지정된 IP를 'upgrade_ip'하고 내보내기/빌드 파이프라인을 다시 실행하세요.
The
는
recreated project rebuilds an exact copy of the original's IP state, so a
다시 생성된 프로젝트는 원본 IP 상태의 정확한 복사본을 다시 작성하므로
locked IP there will always reproduce the same failure downstream.
잠긴 IP는 항상 다운스트림에서 동일한 오류를 재현합니다.

## Testing
## 테스트

Always test the generated build.tcl in a clean directory before committing:
커밋하기 전에 항상 깨끗한 디렉터리에서 생성된 build.tcl을 테스트하세요.
```bash
mkdir /tmp/test_rebuild && cd /tmp/test_rebuild
cp -r /path/to/RevisionControl .
vivado -mode batch -source RevisionControl/Scripts/build.tcl
```

## When to Use vivado_doc_search
## vivado_doc_search를 사용해야 하는 경우

- For `create_project` options (e.g., `-in_memory`, `-part` vs `-board_part`)
- `create_project` 옵션의 경우(예: `-in_memory`, `-part` 및 `-board_part`)
- For `update_compile_order` behavior
- `update_compile_order` 동작의 경우
- For `generate_target` usage when recreating IP or BD
- IP 또는 BD를 다시 생성할 때 `generate_target` 사용법의 경우
