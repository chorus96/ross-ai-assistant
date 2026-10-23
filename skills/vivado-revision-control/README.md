<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Recent changes (2026-09-29)
# 최근 변경사항 (2026-09-29)

Methodology feedback from GP (design review) on the preserve-outputs feature
출력 보존 기능에 대한 GP(설계 검토)의 방법론 피드백
— see [helper-procedures/README.md](helper-procedures/README.md#2026-09-29)
— [helper-procedures/README.md](helper-procedures/README.md#2026-09-29)를 참조하세요.
for the full technical detail, or
전체 기술 세부정보를 보려면
[GP_FEEDBACK_RESOLUTION.md](GP_FEEDBACK_RESOLUTION.md) for a point-by-point
포인트별 [GP_FEEDBACK_RESOLUTION.md](GP_FEEDBACK_RESOLUTION.md)
mapping of GP's 5 feedback items to what was implemented, with code
코드를 사용하여 GP의 5가지 피드백 항목을 구현된 항목에 매핑
evidence.
증거.
Replaces the 2026-08-23–2026-09-21 full-tree
2026-08-23~2026-09-21 전체 트리를 대체합니다.
`project_1.gen/.srcs/.xpr`-copy-inside-`RevisionControl/` design with a
`project_1.gen/.srcs/.xpr`-copy-inside-`RevisionControl/` 디자인
per-item copy into an external **remote cache directory** the user supplies:
사용자가 제공하는 외부 **원격 캐시 디렉토리**에 항목별 복사:

1. **XPR is never revision-controlled, in any flow** — the full `.xpr` copy
1. **XPR은 어떤 흐름에서도 개정 제어되지 않습니다** — 전체 `.xpr` 복사본
that Interactive Flow used to make inside `RevisionControl/` is gone.
`RevisionControl/` 내부를 만드는 데 사용된 대화형 흐름이 사라졌습니다.
`RevisionControl/`'s own shape no longer varies based on whether
`RevisionControl/`의 자체 모양은 더 이상 여부에 따라 달라지지 않습니다.
preservation was used at all.
보존이 전혀 사용되지 않았습니다.
2. **Only the selected item's own `.srcs`/`.gen` are preserved, not the whole
2. **선택한 항목 자체 `.srcs`/`.gen`만 보존되며 전체가 보존되지 않습니다.
project's generated tree** — `export_all_sources` now copies each
프로젝트의 생성된 트리** - 이제 'export_all_sources'가 각각을 복사합니다.
preserved IP's/BD's own per-item directory straight into
IP/BD의 항목별 디렉토리를 바로 보존합니다.
`<cache_dir>/<proj_name>/{ip,bd}/<name>/{srcs,gen}/`, where `cache_dir` is
`<cache_dir>/<proj_name>/{ip,bd}/<name>/{srcs,gen}/`, 여기서 `cache_dir`은
an absolute path outside the project (`prompt_preserve_outputs`/
프로젝트 외부의 절대 경로(`prompt_preserve_outputs`/
`build_preserve_selection` now also collect this path).
이제 `build_preserve_selection`도 이 경로를 수집합니다.
Nothing is ever
아무것도 없습니다
copied into `RevisionControl/` for preservation anymore.
더 이상 보존을 위해 `RevisionControl/`에 복사되었습니다.
3. **Rebuild hard-errors on a missing/incomplete cache** instead of silently
3. **자동 대신 누락/불완전한 캐시에 대한 하드 오류를 다시 작성**
falling back to Tcl regeneration — the old `.rc_preserve_staging/`
Tcl 재생성으로 되돌아가기 — 이전 `.rc_preserve_staging/`
staging step is gone (no longer needed: the external cache isn't at risk
준비 단계가 사라짐(더 이상 필요하지 않음: 외부 캐시가 위험하지 않음)
from `create_project -force`), and so is the BD "fallback safety-net
`create_project -force`에서), BD "fallback safety-net"도 마찬가지입니다.
`write_bd_tcl`" that used to paper over a missing preserved artifact.
누락된 보존 아티팩트를 덮는 데 사용된 `write_bd_tcl`"입니다.
4. **Side-effect fix:** copying the whole per-BD `.srcs` directory (not just
4. **부작용 수정:** BD별 `.srcs` 디렉토리 전체 복사(단순
the lone `<bd>.bd` file) also preserves nested per-instance
단독 `<bd>.bd` 파일)도 중첩된 인스턴스별로 유지됩니다.
`ip/<name>/<name>.xci` files customized only inside that BD — closing the
해당 BD 내에서만 사용자 정의된 `ip/<name>/<name>.xci` 파일 — 닫기
"Known limitation" noted in the 2026-09-21 and 2026-09-11 entries below.
아래 2026-09-21 및 2026-09-11 항목에 '알려진 제한사항'이 언급되어 있습니다.

## Follow-up validation pass (same day) — 2 bugs found and fixed
## 후속 검증 통과(당일) - 버그 2개 발견 및 수정

GP sent 7 action items to close out before sharing this redesign for
GP는 이 재설계를 공유하기 전에 종료할 7가지 작업 항목을 보냈습니다.
testing (clarify 2026.2 behavior, confirm XPR removal, confirm whole-BD-only
테스트(2026.2 동작 명확화, XPR 제거 확인, 전체 BD 전용 확인
preservation, confirm multi-IP selective preservation, add warn/error
보존, 다중 IP 선택적 보존 확인, 경고/오류 추가
handling for un-generated preserve targets, validate against the existing
생성되지 않은 보존 대상을 처리하고 기존 대상과 비교하여 유효성을 검사합니다.
checklist, communicate results).
체크리스트, 결과 전달).
See
참조
[helper-procedures/README.md](helper-procedures/README.md#2026-09-29) for
[helper-procedures/README.md](helper-procedures/README.md#2026-09-29)
full detail.
전체 세부 사항.
Closing this out surfaced two real bugs, both now fixed:
이 문제를 해결하면 두 가지 실제 버그가 드러났으며 이제 둘 다 수정되었습니다.

1. **`generate_build_script` crashed for ANY project with a user-created
1. **`generate_build_script`는 사용자가 생성한 모든 프로젝트에서 충돌을 일으켰습니다.
BD, even with preservation never used** — an unescaped
BD, 보존된 경우에도 사용된 적이 없음** — 이스케이프되지 않음
`$preserved_bd_rel_path($bd_name)` array lookup inside a `puts $fd "..."`
`puts $fd "..."` 내부의 `$preserved_bd_rel_path($bd_name)` 배열 조회
template string evaluated at template-generation time, for every BD, not
템플릿 생성 시 평가되는 템플릿 문자열(BD가 아닌 모든 BD에 대해)
just preserved ones.
그냥 보존된 것들.
Caught by re-running the 9-project regression (all
9개 프로젝트 회귀(모두
9 failed identically) before this could reach testing.
9개는 동일하게 실패함) 이것이 테스트에 도달하기 전에.
Fixed by
수정자:
computing the preserved/not-preserved check and the array lookup in an
보존/보존되지 않은 검사와 배열 조회를 계산합니다.
outer `if` before the template lines.
템플릿 줄 앞의 외부 `if`.
2. **A preserved BD's `validate_bd_design`/`save_bd_design` silently
2. **보존된 BD의 `validate_bd_design`/`save_bd_design`을 자동으로
no-op'd** (`ERROR: [BD 5-159] Cannot find design`, caught by an existing
no-op'd** (`오류: [BD 5-159] 디자인을 찾을 수 없습니다`, 기존
`catch{}` so it never failed the build) because `add_files`-ing a
`catch{}`를 사용하면 빌드가 실패하지 않습니다.) `add_files`를 사용하므로
restored `.bd` doesn't open it as a live BD design — fixed by adding
복원된 `.bd`는 라이브 BD 디자인으로 열리지 않습니다. 추가하여 수정했습니다.
`open_bd_design` right after `add_files`, before `validate_bd_design`.
`add_files` 바로 뒤, `validate_bd_design` 앞의 `open_bd_design`.

Also added: `export_all_sources` now warns and falls back to standard
또한 추가됨: 'export_all_sources'는 이제 경고하고 표준으로 돌아갑니다.
export (not a hard block) if preservation is requested for an IP/BD with no
없는 IP/BD에 대해 보존이 요청된 경우 내보내기(하드 블록 아님)
`.gen` output yet, and both the interactive prompt and the docs now say
`.gen` 출력은 아직이며 대화형 프롬프트와 문서 모두 이제 다음과 같이 말합니다.
explicitly that a BD is preserved as one whole unit (no partial
BD는 하나의 전체 유닛으로 보존된다는 점을 명시적으로 명시합니다.
preservation of individual IPs inside a single BD).
단일 BD 내에서 개별 IP 보존).

**Verification:** full 9-project regression re-run after both fixes — 6
**검증:** 두 가지 수정 후 전체 9개 프로젝트 회귀 재실행 — 6
PASS / 3 BLOCKED, identical to the documented pre-existing baseline (no
PASS / 3 BLOCKED, 문서화된 기존 기준과 동일(없음
regressions).
회귀).
Plus a dedicated live smoke test of the remote-cache flow
또한 원격 캐시 흐름에 대한 전용 라이브 스모크 테스트도 제공됩니다.
itself (export → cache contents → rebuild restores + skips resynthesis →
자체(내보내기 → 캐시 콘텐츠 → 복원 재구축 + 재합성 건너뛰기 →
missing cache hard-errors) against `MBV_ipi_design`'s `microblaze_v_preset`
캐시 하드 오류 누락) `MBV_ipi_design`의 `microblaze_v_preset`에 대한
BD — all steps confirmed working after the fixes above.
BD — 위 수정 후 모든 단계가 작동하는 것으로 확인되었습니다.

# Recent changes (2026-09-21)
# 최근 변경사항(2026-09-21)

Two changes, both verified live against a real interactive Vivado session
두 가지 변경 사항, 둘 다 실제 대화형 Vivado 세션에 대해 실시간으로 확인되었습니다.
(via a pty, not simulated) — see
(시뮬레이트되지 않은 pty를 통해) — 참조
[helper-procedures/README.md](helper-procedures/README.md#2026-09-21) for
[helper-procedures/README.md](helper-procedures/README.md#2026-09-21)
the full detail on each:
각각에 대한 자세한 내용은 다음과 같습니다.

1. **Consolidated preserve-outputs onto a single `project_1.gen/.srcs/.xpr`
1. **통합된 보존 출력을 단일 `project_1.gen/.srcs/.xpr`로 통합
artifact.** Removed the separate `.sources/IP|BD`, `.gen/<name>`,
아티팩트.** 별도의 `.sources/IP|BD`, `.gen/<name>`을 제거했습니다.
`.XPR/<proj>.xpr` per-item mechanism — the full-tree copy is now the only
`.XPR/<proj>.xpr` 항목별 메커니즘 — 이제 전체 트리 복사본이 유일한 항목입니다.
thing `export_all_sources` creates, and `generate_build_script` stages
`export_all_sources`가 생성하는 것 및 `generate_build_script` 단계
what it needs out of that copy at rebuild time instead (new
대신 재구축 시 해당 복사본에서 필요한 것(새 항목)
`.rc_preserve_staging/`, rebuild-scratch only, not checked in).
`.rc_preserve_staging/`, 다시 빌드-스크래치만, 체크인되지 않음).
Along the
따라
way, fixed a real bug (an IP/BD's source was assumed to always live under
방식으로 실제 버그를 수정했습니다(IP/BD의 소스는 항상
the `sources_1` fileset — false for at least one BD in this skill's own
`sources_1` 파일 세트 — 이 스킬 자체에 있는 하나 이상의 BD에 대해 false입니다.
`modnoc-rtl-flat` example design) and a real gitignore bug (the preserved
`modnoc-rtl-Flat` 예제 디자인) 및 실제 gitignore 버그(보존됨)
full-tree copy was being unconditionally gitignored, silently defeating
전체 트리 복사본이 무조건 무시되어 조용히 패배했습니다.
the whole point of preserving it).
보존의 요점).
One known, pre-existing limitation
알려진 기존 제한 사항 중 하나
remains undocumented no longer: BD preservation only stages the
더 이상 문서화되지 않은 채로 남아 있습니다. BD 보존은 단지
top-level `.bd`, not nested per-instance IP files.
중첩된 인스턴스별 IP 파일이 아닌 최상위 `.bd`입니다.
2. **`export_all_sources` auto-prompts for Step 2.5** (preserve generated
2. **`export_all_sources` 2.5단계에 대한 자동 프롬프트**(생성된 보존
outputs) when a human is running the pipeline directly at a real
출력) 인간이 실제 파이프라인을 직접 실행하는 경우
interactive terminal and hasn't already answered the question themselves
대화형 터미널이며 아직 질문 자체에 답변하지 않았습니다.
— previously you had to already know Step 2.5 existed and call
— 이전에는 이미 2.5단계가 존재한다는 것을 알고 전화를 걸어야 했습니다.
`prompt_preserve_outputs` yourself before Step 3, or preservation was
3단계 이전에 스스로 `prompt_preserve_outputs`를 수행했거나 보존이
silently skipped.
조용히 건너뛰었습니다.
Detected via a new `_rc_stdin_is_tty` helper;
새로운 `_rc_stdin_is_tty` 도우미를 통해 감지되었습니다.
automated/CI and agentic callers (no real tty on stdin) are unaffected.
자동/CI 및 에이전트 호출자(stdin에는 실제 tty가 없음)는 영향을 받지 않습니다.

# Recent changes (2026-09-16)
# 최근 변경사항(2026-09-16)

Fixes from a regression re-run of `revision-control-validate` against the
'revision-control-validate'에 대한 회귀 재실행으로 인한 수정 사항
2026-08-10 baseline — see
2026-08-10 기준 — 참조
[helper-procedures/README.md](helper-procedures/README.md#2026-09-16) for the
[helper-procedures/README.md](helper-procedures/README.md#2026-09-16)
full detail on each, including how they were verified.
확인 방법을 포함하여 각각에 대한 자세한 내용입니다.

1. **RM-owned IP/RTL parented outside the RM's own fileset can be invisible to
1. **RM 자체 파일 세트 외부에 있는 RM 소유 IP/RTL은 다른 사용자에게 표시되지 않을 수 있습니다.
export** — a real, generalizable DFX authoring gap in principle (the union
내보내기** — 원칙적으로 일반화 가능한 실제 DFX 작성 격차(연합
logic itself is straightforward and still worth keeping as a hardening).
논리 자체는 간단하며 여전히 강화할 가치가 있습니다.
**Correction (2026-09-21 regression re-run):** this was NOT actually
**수정(2026-09-21 회귀 재실행):** 실제로는 그렇지 않았습니다.
verified against a real repro.
실제 재현과 비교하여 검증되었습니다.
`modnoc-rtl-dfx`'s `axi_dbg_hub_rm1/2`,
`modnoc-rtl-dfx`의 `axi_dbg_hub_rm1/2`,
`axis_vio_pl_master_to_ddr_rm1/2`, `perf_axi_tg_pl_master_to_ddr_rm1/2`
`axis_vio_pl_master_to_ddr_rm1/2`, `perf_axi_tg_pl_master_to_ddr_rm1/2`
module-not-found errors — originally cited here as evidence this fix
모듈을 찾을 수 없는 오류 — 원래는 이 수정 사항에 대한 증거로 여기에 인용되었습니다.
worked — were confirmed to reproduce identically both before and after
작업 — 전후 모두 동일하게 재현되는 것으로 확인되었습니다.
this fix (the `.xci` files were already visible to the plain
이 수정 사항('.xci' 파일은 이미 일반 사용자에게 표시되었습니다)
`get_files -of_objects $rm` query pre-fix; the union added nothing for
`get_files -of_objects $rm` 쿼리 접두사; 노조는 아무것도 추가하지 않았다
this project).
이 프로젝트).
Their real cause is an IP-catalog version lock (every IP
실제 원인은 IP 카탈로그 버전 잠금(모든 IP
in that project, RM-owned or not, shows `current catalog revision is
해당 프로젝트에서 RM 소유 여부에 관계없이 '현재 카탈로그 개정판은 다음과 같습니다.
'unknown'` from the moment the project opens) — an environment/IP-version
프로젝트가 열린 순간부터 '알 수 없음') — 환경/IP 버전
issue, not a fileset-visibility gap.
문제는 파일 세트 가시성 격차가 아닙니다.
See
참조
[helper-procedures/README.md](helper-procedures/README.md#2026-09-16) for
[helper-procedures/README.md](helper-procedures/README.md#2026-09-16)
the full corrected writeup.
전체 수정된 글입니다.
Not a fix for `rtl_dfx`'s `axi_traffic_gen_N`
`rtl_dfx`의 `axi_traffic_gen_N`에 대한 수정 사항이 아닙니다.
or `modnoc-rtl-dfx`'s `RP1_rm1`/`RP1_rm2` — those remain the
또는 `modnoc-rtl-dfx`의 `RP1_rm1`/`RP1_rm2` — 이것들은 그대로 유지됩니다.
already-documented pre-existing example-design defects (see below).
이미 문서화되어 있는 기존 예제 설계 결함(아래 참조)
2. **`verify_rebuild` false-PASS gap (Issue #37)** — it never parsed
2. **`verify_rebuild` false-PASS 간격(문제 #37)** — 구문 분석되지 않았습니다.
`rebuild.log`, only live structural project state, so a caught-and-logged
`rebuild.log`, 라이브 구조 프로젝트 상태만 있으므로 포착 및 기록됩니다.
failure (BDC Monitor-mode's `BD 41-258`, or a NoC PHY IP's `generate_target`
실패(BDC 모니터 모드의 `BD 41-258` 또는 NoC PHY IP의 `generate_target`
failure) still reported PASS.
실패) 여전히 PASS를 보고했습니다.
`verify_rebuild` now takes an optional
`verify_rebuild`는 이제 선택 사항을 취합니다.
rebuild-log path and scans it for `ERROR:` lines.
다시 빌드 로그 경로를 검색하고 'ERROR:' 줄을 검색합니다.
3. **NoC memory-controller PHY IP `.elf`/simulation-output failures
3. **NoC 메모리 컨트롤러 PHY IP `.elf`/시뮬레이션 출력 오류
(`modnoc-rtl-dfx`, `ipi_bdc_dfx`, `segmented_config`) — investigated, not
(`modnoc-rtl-dfx`, `ipi_bdc_dfx`, `segmented_config`) — 조사됨, 아님
fixable from this skill.** Confirmed root cause: a Vivado *installation*
이 스킬로 고칠 수 있습니다.** 확인된 근본 원인: Vivado *설치*
gap (missing files under `data/ip/xilinx/noc_mc_ddr5_phy_v1_0/hdl/bfm/
간격(`data/ip/xilinx/noc_mc_ddr5_phy_v1_0/hdl/bfm/ 아래에 파일이 누락됨)
PINOUT/`), not an export/rebuild bug — see
PINOUT/`), 내보내기/재구축 버그가 아님 — 참조
[generate-build-script/REFERENCE.md](generate-build-script/REFERENCE.md)'s
[generate-build-script/REFERENCE.md](generate-build-script/REFERENCE.md)의
new "Known Limitation" section for the full diagnosis.
전체 진단을 위한 새로운 "알려진 제한 사항" 섹션.

# Recent changes (2026-09-11)
# 최근 변경사항(2026-09-11)

Team decision on the Interactive Flow (Step 2.5 preserve-outputs) directory
대화형 흐름(단계 2.5 출력 보존) 디렉터리에 대한 팀 결정
layout — see
레이아웃 - 참조
[helper-procedures/README.md](helper-procedures/README.md#2026-09-11) for the
[helper-procedures/README.md](helper-procedures/README.md#2026-09-11)
full detail.
전체 세부 사항.
`export_all_sources` now also copies the complete original
`export_all_sources`는 이제 전체 원본도 복사합니다.
`<proj_name>.gen/`, `<proj_name>.srcs/`, and `<proj_name>.xpr` directly inside
`<proj_name>.gen/`, `<proj_name>.srcs/` 및 `<proj_name>.xpr` 직접 내부
`RevisionControl/` whenever Step 2.5 preserved at least one IP/BD — additive
2.5단계에서 하나 이상의 IP/BD를 보존할 때마다 `RevisionControl/` - 추가
to, not a replacement for, the existing per-item `.sources/.gen/.XPR`
기존 항목별 `.sources/.gen/.XPR`을 대체하는 것이 아니라
mechanism that `generate_build_script`'s rebuild still relies on.
`generate_build_script`의 재구축이 여전히 의존하는 메커니즘입니다.
Classic
클래식
Flow (no preservation) is unchanged: those directories stay external,
흐름(보존 없음)은 변경되지 않습니다. 해당 디렉터리는 외부에 유지됩니다.
untouched siblings of `RevisionControl/`.
`RevisionControl/`의 손대지 않은 형제.

# Recent changes (2026-09-02)
# 최근 변경사항(2026-09-02)

Fixed Step 2.5 ("preserve generated outputs") never producing any visible
2.5단계("생성된 출력 보존")가 눈에 보이는 결과를 생성하지 않는 문제를 수정했습니다.
interaction when the skill is run agentically (an AI agent driving Vivado,
스킬이 에이전트적으로 실행될 때 상호 작용(Vivado를 구동하는 AI 에이전트,
not a human at a real tty) — see
실제 tty에는 인간이 아님) — 참조
[helper-procedures/README.md](helper-procedures/README.md#2026-09-02) for
[helper-procedures/README.md](helper-procedures/README.md#2026-09-02)
the root cause and fix.
근본 원인을 해결하고 수정합니다.
New `build_preserve_selection` proc lets an agent
새로운 `build_preserve_selection` 프로세스를 통해 에이전트가
ask the human directly in conversation instead of relying on
의존하는 대신 대화에서 직접 인간에게 물어보십시오.
`prompt_preserve_outputs`'s `gets stdin`, which an agent's batch/Bash-driven
`prompt_preserve_outputs`의 `stdin` 가져오기(에이전트의 배치/Bash 기반)
Tcl calls can never actually reach.
Tcl 호출은 실제로 도달할 수 없습니다.
Purely additive; the human-interactive
순수 첨가제; 인간 상호 작용
and automated-CI paths are unchanged.
자동화된 CI 경로는 변경되지 않습니다.

# Recent changes (2026-08-23)
# 최근 변경사항 (2026-08-23)

Meeting-driven feature request, plus one small fix, both in
회의 기반 기능 요청과 하나의 작은 수정 사항 모두
`helper-procedures/helper_scripts.tcl` — see
`helper-procedures/helper_scripts.tcl` — 참조
[helper-procedures/README.md](helper-procedures/README.md#2026-08-23) for the
[helper-procedures/README.md](helper-procedures/README.md#2026-08-23)
full detail, including how each was verified.
각각의 확인 방법을 포함한 자세한 내용입니다.

1. **New "preserve generated outputs" step (Step 2.5, interactive-only)** —
1. **새로운 "생성된 출력 보존" 단계(2.5단계, 대화형 전용)** —
lets the user pick standalone IPs/BDs whose existing `.gen` output should
사용자가 기존 `.gen` 출력이 있어야 하는 독립형 IP/BD를 선택할 수 있습니다.
be reused as-is on rebuild ("golden project" reuse) instead of being
재구축하는 대신 있는 그대로 재사용("골든 프로젝트" 재사용)
regenerated from Tcl.
Tcl에서 재생성되었습니다.
Off by default: `run_pipeline.tcl` (used by
기본적으로 꺼짐: `run_pipeline.tcl`(에서 사용됨)
`revision-control-validate`) never calls it, so every existing automated
`revision-control-validate`)는 절대 호출하지 않으므로 기존의 모든 자동화
run is unaffected.
실행에는 영향이 없습니다.
2. **`.csv` wasn't recognized as a data file** — added to the Data export
2. **`.csv`는 데이터 파일로 인식되지 않았습니다** — 데이터 내보내기에 추가됨
category alongside `.mem .mif .coe .hex .elf`.
`.mem .mif .coe .hex .elf`와 함께 카테고리를 지정하세요.

A full 9-project regression pass on 2026-08-23/24 confirmed the automated
2026년 8월 23일 24일 전체 9개 프로젝트 회귀 통과를 통해 자동화된 것으로 확인되었습니다.
5-step path is unchanged by this patch: same 6 PASS / 3 BLOCKED split as
이번 패치에서는 5단계 경로가 변경되지 않습니다. 6 PASS / 3 BLOCKED 분할과 동일합니다.
before, and the 3 blocked projects (`ipi_bdc_dfx`, `modnoc-rtl-dfx`,
이전에 차단된 프로젝트 3개(`ipi_bdc_dfx`, `modnoc-rtl-dfx`,
`rtl_dfx`) are the same pre-existing example-design defects noted below, not
`rtl_dfx`)는 아래에 언급된 것과 동일한 기존 예제 설계 결함입니다.
regressions from this change.
이 변경으로 인한 회귀.

# Recent fixes (2026-08-10)
# 최근 수정 사항(2026-08-10)

Found via a full validation pass over the 9 `vivado-revision-control` example
9개의 `vivado-revision-control` 예시에 대한 전체 검증 패스를 통해 발견되었습니다.
designs.
디자인.
All in `helper-procedures/helper_scripts.tcl` — see
모두 `helper-procedures/helper_scripts.tcl`에 있습니다 — 참조
[helper-procedures/README.md](helper-procedures/README.md) for the full
전체에 대한 [helper-procedures/README.md](helper-procedures/README.md)
detail on each, including how they were verified.
확인 방법을 포함하여 각각에 대해 자세히 설명합니다.

1. **Versal detection never fired** — `detect_project_flow` matched board
1. **Versal 감지가 실행되지 않음** — `Detect_project_flow`가 보드와 일치함
names (`vck*`, `vek*`, ...) against what's actually a part string; added
실제로 부분 문자열에 대한 이름(`vck*`, `vek*`, ...) 추가됨
part-number prefix matches (`xcv*`/`xc2v*`).
부품 번호 접두어가 일치합니다(`xcv*`/`xc2v*`).
2. **Missing DFX RM source files failed silently** — export now warns
2. **누락된 DFX RM 소스 파일이 자동으로 실패했습니다** — 이제 내보내기에 경고가 표시됩니다.
instead of skipping a stale/broken file reference with no trace.
추적 없이 오래되거나 손상된 파일 참조를 건너뛰는 대신.
3. **`create_reconfig_module` never passed `-top`** — Vivado couldn't
3. **`create_reconfig_module`은 `-top`을 통과하지 못했습니다** — Vivado는 실패했습니다
associate an RM with its partition def's module, surfacing as a confusing
RM을 해당 파티션 정의의 모듈과 연결하여 혼란스러운 것으로 나타납니다.
"module name should match" error at rebuild time.
재구축 시 "모듈 이름이 일치해야 합니다" 오류가 발생했습니다.
4. **RM file attachment used a command combo that doesn't work** — replaced
4. **RM 파일 첨부에서 작동하지 않는 명령 콤보를 사용했습니다** — 교체됨
with `add_files -of_objects`, confirmed live against Vivado 2026.2 to be
`add_files -of_objects`를 사용하면 Vivado 2026.2에 대한 라이브가 확인되었습니다.
the only form that actually attaches a file to an RM's fileset.
실제로 RM의 파일 세트에 파일을 첨부하는 유일한 양식입니다.
5. **Step 6 didn't exclude RM-owned IPs** — they were recreated at project
5. **6단계에서는 RM 소유 IP를 제외하지 않았습니다** — 프로젝트에서 다시 생성되었습니다.
scope before Step 12 attached them to their RM, causing "IP name already
12단계에서 범위를 RM에 연결했기 때문에 "IP 이름이 이미
in use."
사용 중입니다."

**Known limitation, not fixable here:** `rtl_dfx` and `modnoc-rtl-dfx` still
**알려진 제한 사항, 여기서는 수정할 수 없습니다:** `rtl_dfx` 및 `modnoc-rtl-dfx`는 여전히
show `REBUILD_RESULT=BLOCKED` after all of the above — confirmed via a
위의 모든 작업 후에 `REBUILD_RESULT=BLOCKED`를 표시합니다. —를 통해 확인됨
scratch-copy rebuild of the untouched original projects that this is a
손길이 닿지 않은 원본 프로젝트의 스크래치 복사 재구축
pre-existing defect in those example designs, not in this skill.
이 기술이 아닌 해당 예제 디자인에 기존 결함이 있습니다.
