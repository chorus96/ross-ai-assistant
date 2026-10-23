<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# GP methodology feedback (2026-09-29) — resolution status
# GP 방법론 피드백(2026-09-29) — 해결 상태

GP (design review) sent feedback on the "preserve generated outputs" (Step
GP(설계 검토)에서 "생성된 출력 보존"에 대한 피드백을 보냈습니다(단계
2.5) feature of this skill, covering 5 points.
2.5) 이 스킬의 특징은 5개 포인트를 포괄합니다.
This doc maps each point to
이 문서는 각 지점을 다음으로 매핑합니다.
what was implemented, with code evidence, so it can be confirmed closed out
코드 증거와 함께 구현된 내용이 종료되었음을 확인할 수 있습니다.
without re-reading the whole diff.
전체 차이점을 다시 읽지 않고.
See
참조
[helper-procedures/README.md](helper-procedures/README.md#2026-09-29) for
[helper-procedures/README.md](helper-procedures/README.md#2026-09-29)
the full technical changelog and the 2 bugs found+fixed while verifying
전체 기술 변경 로그와 확인 중 발견된 버그 2개 및 수정됨
this.
이.

## 1.
## XPR in revision control vs. recreate via Tcl
## 개정 제어의 XPR과 Tcl을 통한 재생성

**GP's guidance:** don't revision-control the XPR (too verbose, not
**GP 지침:** XPR의 개정을 제어하지 마세요(너무 장황하지만
revision-control friendly) — recommend rebuilding via Tcl instead.
개정 제어 친화적) — 대신 Tcl을 통해 다시 빌드하는 것이 좋습니다.

**Status: ✅ Done.** No `.xpr` copy logic exists anywhere in the
**상태: ✅ 완료.** `.xpr` 복사 논리가 어디에도 존재하지 않습니다.
export/preserve path.
내보내기/보존 경로.
The only remaining `.xpr` reference is an explicit
남은 유일한 `.xpr` 참조는 명시적 참조입니다.
comment noting it's intentionally excluded:
의도적으로 제외되었다는 의견:
```
# helper_scripts.tcl:3105
Note: <proj_name>.xpr is intentionally NOT part of this export or check-in --
```

## 2.
## Behavior when "preserve output products" is selected
## "출력 제품 보존"을 선택한 경우의 동작

**GP's guidance:** still rebuild the project deterministically via script.
**GP의 지침:** 여전히 스크립트를 통해 프로젝트를 결정적으로 다시 빌드합니다.
For preserved XCIs/BDs, the script should do `read_ip`/`read_bd`/`add_files`
보존된 XCI/BD의 경우 스크립트는 `read_ip`/`read_bd`/`add_files`를 수행해야 합니다.
instead of sourcing `ip.tcl`/`bd.tcl` — as long as `.srcs`/`.gen` are
`ip.tcl`/`bd.tcl`을 소스로 사용하는 대신 — `.srcs`/`.gen`이 있는 한
available, Vivado will pick up the generated outputs from `.gen`
사용 가능하면 Vivado는 `.gen`에서 생성된 출력을 선택합니다.
automatically.
자동으로.

**Status: ✅ Done.** Both preserved-item code paths use `add_files` (one of
**상태: ✅ 완료.** 두 보존 항목 코드 경로 모두 `add_files`를 사용합니다(다음 중 하나).
the three methods GP named) instead of sourcing an exported `ip.tcl`/
내보낸 `ip.tcl`을 소싱하는 대신 GP라는 세 가지 방법/
`bd.tcl` script:
`bd.tcl` 스크립트:
- **Preserved IP:** `add_files -norecurse [...]` then
- **보존된 IP:** `add_files -norecurse [...]` 그런 다음
`catch {generate_target all [get_ips $name]}`.
`{generate_target all [get_ips $name]}`을 잡으세요.
- **Preserved BD:** `add_files -norecurse [...] {${bd_name}.bd}` then
- **보존된 BD:** `add_files -norecurse [...] {${bd_name}.bd}` 그런 다음
`generate_target all [get_files ${bd_name}.bd]`, plus `open_bd_design`
`generate_target all [get_files ${bd_name}.bd]` 및 `open_bd_design`
(a bug fix from this same pass — `add_files` alone doesn't open a BD as
(동일한 패스의 버그 수정 — `add_files`만으로는 BD를 열 수 없습니다.
a live design, which broke `validate_bd_design`/`save_bd_design`
`validate_bd_design`/`save_bd_design`을 깨뜨린 라이브 디자인
afterward) before `validate_bd_design`/`save_bd_design`.
이후) `validate_bd_design`/`save_bd_design` 이전에.

The cache restore (point 4 below) runs *before* this step, so `.srcs`/
캐시 복원(아래 4번 항목)은 이 단계 *전에* 실행되므로 `.srcs`/
`.gen` are already in place when `add_files`/`generate_target` run —
`add_files`/`generate_target` 실행 시 `.gen`이 이미 존재합니다 —
confirmed live: `generate_target` logs "Generated targets are already
확인된 라이브: `generate_target` 로그 "생성된 대상은 이미
up-to-date ... hence not re-generating" instead of resynthesizing.
최신 상태이므로 재합성하는 대신 "재생성되지 않습니다."

## 3.
## Handling mixed scenarios (partial preservation)
## 혼합 시나리오 처리(부분 보존)

**GP's guidance:** most users want to preserve a *selected* subset, not
**GP 지침:** 대부분의 사용자는 *선택한* 하위 집합을 보존하기를 원합니다.
all-or-nothing — list all XCIs (outside any BD) and BDs, let the user pick.
전부 아니면 전무 — 모든 XCI(BD 외부) 및 BD를 나열하고 사용자가 선택하도록 합니다.
Finer-grained picking (e.g. individual IPs inside one BD) is a Q4
보다 세분화된 선택(예: 하나의 BD 내의 개별 IP)은 Q4입니다.
enhancement, not required now.
강화, 지금은 필요하지 않습니다.

**Status: ✅ Already implemented** (pre-dates this pass, untouched by it):
**상태: ✅ 이미 구현되었습니다**(이 패스보다 먼저, 영향을 받지 않음):
`list_preservable_outputs` / `prompt_preserve_outputs` (interactive) /
`list_preservable_outputs` / `prompt_preserve_outputs`(대화형) /
`build_preserve_selection` (agentic) list every standalone XCI and every
`build_preserve_selection`(에이전트)은 모든 독립형 XCI 및 모든 목록을 나열합니다.
BD, and let the caller pick a subset of each.
BD를 선택하고 발신자가 각각의 하위 집합을 선택하도록 합니다.
Matches GP's description
GP의 설명과 일치
exactly.
정확히.
The one known limitation — preservation is whole-BD-only, no
한 가지 알려진 제한 사항 - 보존은 전체 BD에만 가능합니다.
cherry-picking individual IPs *inside* a single BD — is now explicit in
단일 BD *내부*에서 개별 IP를 선택하는 것이 이제 명시적으로 나타납니다.
the docs/prompts rather than an undocumented side effect, and lines up
문서화되지 않은 부작용이 아닌 문서/프롬프트 및 라인업
with GP's own note that finer granularity is a Q4 item, not a now-gap.
더 미세한 세분성은 현재 격차가 아닌 Q4 항목이라는 GP의 자체 메모와 함께.

## 4.
## Cache: remote only, hard error if missing
## 캐시: 원격 전용, 누락된 경우 하드 오류

**GP's guidance:** OK to support only a remote cache (directory outside the
**GP 지침:** 원격 캐시(외부 디렉터리)만 지원해도 괜찮습니다.
project) for now.
프로젝트) 현재로서는.
If that absolute path is missing at rebuild time, the
재구축 시 해당 절대 경로가 누락된 경우
rebuild should error out.
다시 빌드하면 오류가 발생합니다.

**Status: ✅ Done.** `_rc_validate_cache_dir` rejects anything that isn't an
**상태: ✅ 완료.** `_rc_validate_cache_dir`은 캐시가 아닌 모든 것을 거부합니다.
absolute path outside the project.
프로젝트 외부의 절대 경로.
At rebuild time (Step 1c):
재구축 시(1c단계):
```tcl
if {![file isdirectory $preserve_cache_dir]} {
    error "Cannot rebuild: remote cache directory for preserved IP/BD outputs is missing ..."
}
```
The same hard-error check runs again per preserved item (missing `srcs/`
보존된 항목마다 동일한 하드 오류 검사가 다시 실행됩니다(`srcs/` 누락).
or `gen/` for that specific IP/BD also halts the whole build).
또는 특정 IP/BD에 대한 `gen/`도 전체 빌드를 중지합니다.
Live-verified
실시간 검증
by renaming the cache dir and confirming rebuild stopped before touching
캐시 디렉토리의 이름을 바꾸고 터치하기 전에 재구축이 중지되었는지 확인하여
any project state.
모든 프로젝트 상태.

## 5.
## Generated artifacts inclusion rules
## 생성된 아티팩트 포함 규칙

**GP's guidance:** if a BD or XCI is chosen for preservation, preserve both
**GP 지침:** 보존을 위해 BD 또는 XCI를 선택한 경우 둘 다 보존하십시오.
its `.srcs` and its `.gen`.
`.srcs`와 `.gen`입니다.

**Status: ✅ Done.** Cache restore copies both
**상태: ✅ 완료.** 캐시 복원은 두 가지 모두 복사합니다.
`<cache>/<proj>/{ip,bd}/<name>/srcs/` and `.../gen/` into the rebuilt
`<cache>/<proj>/{ip,bd}/<name>/srcs/` 및 `.../gen/`을 재구축된 파일에 추가
project's `${proj_name}.srcs/`/`${proj_name}.gen/` — both directories, not
프로젝트의 `${proj_name}.srcs/`/`${proj_name}.gen/` — 두 디렉토리 모두, 아님
just the bare `.xci`/`.bd` file.
그냥 `.xci`/`.bd` 파일만 있으면 됩니다.
This also fixed a pre-existing gap:
이는 또한 기존의 격차를 해결했습니다.
copying the whole `.srcs` directory (not just the `.bd`) now captures
전체 `.srcs` 디렉토리(`.bd`뿐만 아니라)를 복사하면 이제 캡처됩니다.
nested per-instance IPs customized only inside a preserved BD.
보존된 BD 내부에서만 사용자 정의된 중첩된 인스턴스별 IP입니다.

## Verification performed
## 검증 수행

Full 9-project regression re-run after the fixes above: 6 PASS / 3 BLOCKED,
위 수정 후 전체 9개 프로젝트 회귀 재실행: 6 PASS / 3 BLOCKED,
identical to the documented pre-existing baseline (no regressions from this
문서화된 기존 기준과 동일합니다(이로부터 회귀 없음).
redesign).
재설계).
Plus a dedicated hand-run smoke test of the remote-cache flow
또한 원격 캐시 흐름에 대한 전용 수동 실행 연기 테스트
itself against `MBV_ipi_design`'s `microblaze_v_preset` BD — export, cache
`MBV_ipi_design`의 `microblaze_v_preset` BD에 대한 자체 — 내보내기, 캐시
contents, rebuild-restores-and-skips-resynthesis, and missing-cache
콘텐츠, 재구축 복원 및 건너뛰기 재합성 및 캐시 누락
hard-error all confirmed working.
하드 오류가 모두 확인되었습니다.

## Out of scope for this feedback thread
## 이 피드백 스레드의 범위를 벗어났습니다.

These are pre-existing, unrelated gaps — not part of GP's 5 points above,
이는 GP의 위 5가지 사항에 포함되지 않은 기존의 관련 없는 격차입니다.
and not affected by this pass:
이 패스의 영향을 받지 않습니다.
- Confluence checklist items #1 (property gaps), #10 (data files), #37
- Confluence 체크리스트 항목 #1(속성 격차), #10(데이터 파일), #37
(BDC Monitor-mode), #52 (doc review), #65 (unknown) remain open/
(BDC 모니터 모드), #52(문서 검토), #65(알 수 없음)는 계속 열려 있음/
unverified.
확인되지 않았습니다.
- Additional checks A–D (property-level diffing, double-run safety
- 추가 점검 A–D(속성 수준 차이, 이중 실행 안전성
regression, Git LFS tracking, encrypted-IP handling) remain
회귀, Git LFS 추적, 암호화된 IP 처리)는 그대로 유지됩니다.
not-yet-automated.
아직 자동화되지 않았습니다.

## Status as of 2026-09-29
## 2026-09-29 현재 현황

All changes described here are **uncommitted** working-tree edits in
여기에 설명된 모든 변경 사항은 **커밋되지 않은** 작업 트리 편집입니다.
`agentic-ai-suite` (baseline: commit `04312979`, 2026-09-21).
`agentic-ai-suite`(기준: 커밋 `04312979`, 2026-09-21).
Not yet
아직은 아님
pushed/shared.
푸시/공유.
