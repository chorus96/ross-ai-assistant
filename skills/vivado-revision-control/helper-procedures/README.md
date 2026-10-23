<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# helper_scripts.tcl — recent fixes
# helper_scripts.tcl — 최근 수정 사항

## 2026-09-29

### Change: preserve-outputs now uses an external remote cache, not an in-repo full-tree copy
### 변경: 보존 출력은 이제 저장소 내 전체 트리 복사본이 아닌 외부 원격 캐시를 사용합니다.

GP (design review) gave methodology feedback on the preserve-outputs feature
GP(설계 검토)는 출력 보존 기능에 대한 방법론 피드백을 제공했습니다.
that the 2026-08-23–2026-09-21 design didn't match on three points:
2026-08-23~2026-09-21 디자인은 세 가지 점에서 일치하지 않았습니다.

1. XPR should never be revision-controlled, in any flow.
1. XPR은 어떤 흐름에서도 개정 관리를 받아서는 안 됩니다.
2. Rebuild should stay deterministic via Tcl, but for a preserved item do
2. 재구축은 Tcl을 통해 결정론적으로 유지되어야 하지만 보존된 항목의 경우
`read_ip`/`read_bd`/`add_files` (already true here) with only that item's
해당 항목만 포함된 `read_ip`/`read_bd`/`add_files`(여기서는 이미 true)
own `.srcs`/`.gen` preserved — not the whole project's generated tree.
자신의 `.srcs`/`.gen`이 보존됩니다 — 전체 프로젝트의 생성된 트리가 아닙니다.
3. Only a **remote cache** (a directory outside the project) is supported
3. **원격 캐시**(프로젝트 외부 디렉터리)만 지원됩니다.
for now; if that absolute path is missing at rebuild time, rebuild must
지금은; 재구축 시 해당 절대 경로가 누락된 경우 재구축해야 합니다.
error out.
오류가 발생했습니다.

The 2026-09-21 consolidation had actually moved in the opposite direction on
2026-09-21 통합은 실제로 반대 방향으로 이동했습니다.
points 1 and 2: it copied the *entire* `<proj_name>.gen/.srcs/.xpr` — every
포인트 1과 2: *전체* `<proj_name>.gen/.srcs/.xpr`을 복사했습니다.
run, cache, and IP/BD output, plus the XPR itself — into `RevisionControl/`
실행, 캐시, IP/BD 출력과 XPR 자체를 `RevisionControl/`에 추가
(a git-controlled directory *inside* the exported project) whenever
(내보낸 프로젝트 *내부*의 git 제어 디렉토리)
preservation was used, then staged per-item pieces back out of that at
보존을 사용한 다음 항목별로 조각을 다시 단계적으로 배치했습니다.
rebuild time into `.rc_preserve_staging/`.
`.rc_preserve_staging/`에 시간을 다시 빌드합니다.

**What changed:**
**변경된 사항:**

- `export_all_sources` no longer copies anything into `$export_dir`/
- `export_all_sources`는 더 이상 `$export_dir`/에 아무것도 복사하지 않습니다.
`RevisionControl/` for preservation.
보존을 위한 `RevisionControl/`.
Instead, each preserved IP's/BD's own
대신에 보존된 각 IP/BD는
`.srcs` directory (the whole directory, not just the `.xci`/`.bd` file —
`.srcs` 디렉토리(`.xci`/`.bd` 파일뿐만 아니라 전체 디렉토리 —
see the side-effect fix below) and matching `.gen` directory are copied
아래 부작용 수정 사항 참조) 및 일치하는 `.gen` 디렉토리가 복사됩니다.
directly into an external cache directory the user supplies, at
사용자가 제공하는 외부 캐시 디렉토리에 직접
`$cache_dir/<proj_name>/{ip,bd}/<name>/{srcs,gen}/`.
`$cache_dir/<proj_name>/{ip,bd}/<name>/{srcs,gen}/`.
`$cache_dir` must be
`$cache_dir`은 다음과 같아야 합니다.
an absolute path outside the project (`_rc_validate_cache_dir` enforces
프로젝트 외부의 절대 경로(`_rc_validate_cache_dir`은 강제 실행)
this); export creates it if missing (export is the producer).
이); 누락된 경우 내보내기가 생성됩니다(내보내기가 생산자임).
- `prompt_preserve_outputs` (interactive) and `build_preserve_selection`
- `prompt_preserve_outputs`(대화형) 및 `build_preserve_selection`
(agentic) both gained a `cache_dir` parameter/prompt.
(에이전트) 둘 다 'cache_dir' 매개변수/프롬프트를 얻었습니다.
An invalid/missing
유효하지 않거나 누락됨
path degrades the whole selection to `enabled 0`, same as an
경로는 전체 선택을 '활성화 0'으로 저하시킵니다.
empty/no-match IP/BD selection already did.
비어 있음/일치하지 않는 IP/BD 선택이 이미 수행되었습니다.
- `generate_build_script`'s old **Step 0** (staging pieces of the full-tree
- `generate_build_script`의 이전 **0단계**(전체 트리 조각 준비)
copy into `.rc_preserve_staging/` before `create_project -force` could
`create_project -force`가 실행되기 전에 `.rc_preserve_staging/`에 복사하세요.
overwrite them) is gone entirely — no longer needed, since the external
덮어쓰기)가 완전히 사라졌습니다. 외부
cache isn't at risk from `create_project -force` in the first place.
캐시는 우선 `create_project -force`로 인해 위험에 처하지 않습니다.
- **Step 1c** now reads directly from the external cache
- **1c단계**는 이제 외부 캐시에서 직접 읽습니다.
(`preserved_cache_dir`, a literal absolute path baked into `build.tcl`) and
(`preserved_cache_dir`, `build.tcl`에 구운 리터럴 절대 경로) 및
is a **hard error**, not a graceful fallback: if the cache root is
**하드 오류**이며 정상적인 폴백이 아닙니다. 캐시 루트가 다음과 같은 경우
missing, or a specific preserved item's `srcs`/`gen` subfolders inside it
누락되었거나 그 안에 있는 특정 보존 항목의 `srcs`/`gen` 하위 폴더
are missing/incomplete, it raises a Tcl `error` and the whole rebuild
누락/불완전한 경우 Tcl '오류'가 발생하고 전체 재구축이 발생합니다.
stops.
중지합니다.
Confirmed with the user this should apply to *any* missing
사용자에게 이것이 누락된 *모든* 항목에 적용되어야 함을 확인했습니다.
preserved item's cache data, not just a missing cache root.
누락된 캐시 루트뿐만 아니라 항목의 캐시 데이터도 보존됩니다.
- Removed the BD "fallback safety-net `write_bd_tcl`" (written at export
- BD "fallback safety-net `write_bd_tcl`"을 제거했습니다(내보내기 시 작성됨).
time for every preserved BD, "just in case" its golden path went missing)
보존된 모든 BD에 대한 시간, "만일의 경우" 골든 경로가 사라졌을 경우)
— it existed specifically to paper over a missing preserved artifact,
— 사라진 보존 유물을 덮기 위해 특별히 존재했습니다.
which now contradicts the hard-error intent.
이는 이제 하드 오류 의도와 모순됩니다.
Steps 6/7's own "fall back to
6/7단계 자체의 "폴백"
sourcing the Tcl script" branches are gone too, since Step 1c already
Tcl 스크립트 소싱" 분기도 이미 1c단계에서 사라졌기 때문에 사라졌습니다.
guarantees the source exists before they run.
실행되기 전에 소스가 존재함을 보장합니다.
- Gitignore/README generation no longer varies based on preservation —
- Gitignore/README 생성은 더 이상 보존 여부에 따라 달라지지 않습니다.
`${proj_name}.gen/`/`.srcs/` are always ignored, `.rc_preserve_staging/`
`${proj_name}.gen/`/`.srcs/`는 항상 무시됩니다. `.rc_preserve_staging/`
is gone, and the README instead documents the external cache path and its
README는 대신 외부 캐시 경로와 해당 경로를 문서화합니다.
hard-error behavior when preservation was used.
보존이 사용될 때 하드 오류 동작.

**Side-effect fix:** copying the whole per-BD `.srcs` directory (not just
**부작용 수정:** BD별 `.srcs` 디렉토리 전체 복사(단순
the lone `<bd>.bd` file, as the 2026-09-11/09-21 designs did) also captures
2026-09-11/09-21 디자인과 마찬가지로 단독 `<bd>.bd` 파일도 캡처합니다.
any nested per-instance `ip/<name>/<name>.xci` files customized only inside
내부에서만 사용자 정의된 중첩된 인스턴스별 `ip/<name>/<name>.xci` 파일
that BD — closing the "Known limitation" documented in the 2026-09-21 and
해당 BD — 2026-09-21에 문서화된 "알려진 제한 사항"을 종료하고
2026-09-11 entries below (a rebuild could previously print
아래 2026-09-11 항목(재구축하면 이전에 인쇄할 수 있음)
`[BD 41-2576] File ... could not be found` for those nested IPs).
`[BD 41-2576] 중첩된 IP에 대한 파일 ...을 찾을 수 없습니다`).

**Initial verification (superseded by the follow-up pass below):** a plain
**초기 확인(아래 후속 단계로 대체):** 일반
`tclsh` source-only smoke test (no live Vivado) confirmed the edited file
`tclsh` 소스 전용 스모크 테스트(라이브 Vivado 없음)가 편집된 파일을 확인했습니다.
still sources cleanly, plus a standalone test of the new
여전히 깔끔하게 소스를 제공하고 새로운 기능에 대한 독립형 테스트도 제공합니다.
`_rc_validate_cache_dir` helper's edge cases (empty, relative, nested-under-
`_rc_validate_cache_dir` 도우미의 예외적인 경우(비어 있음, 상대, 중첩됨-
project, and a valid external path).
프로젝트 및 유효한 외부 경로).
At this point the reasoning was
이 시점에서 추론은 다음과 같습니다.
"`run_pipeline.tcl` always passes `preserve_selection={}`, so no full Vivado
"`run_pipeline.tcl`은 항상 `preserve_selection={}`을 전달하므로 전체 Vivado는 없습니다.
regression re-run is needed" — **this reasoning was correct in spirit but
회귀 재실행이 필요합니다." — **이 추론은 정신적으로 정확했지만
insufficient in practice: it doesn't cover a bug in code that runs
실제로는 불충분함: 실행되는 코드의 버그를 다루지 않습니다.
regardless of `preserve_selection`,** which is exactly what the follow-up
`preserve_selection`에 관계없이** 이는 정확히 후속 조치입니다.
pass below found.
아래를 통과하여 찾았습니다.

### Follow-up pass (same day): live validation surfaced 2 real bugs
### 후속 조치(당일): 실시간 검증으로 실제 버그 2개 발견

GP sent 7 action items to close this out before sharing for testing:
GP는 테스트를 위해 공유하기 전에 이를 종료하기 위해 7가지 작업 항목을 보냈습니다.
reflect the clarified 2026.2 behavior; confirm XPR removal (done above);
명확한 2026.2 동작을 반영합니다. XPR 제거를 확인합니다(위에서 수행됨).
verify/confirm partial-BD preservation is NOT supported; confirm multi-IP
부분 BD 보존 확인/확인은 지원되지 않습니다. 다중 IP 확인
selective preservation; add warn/error handling for preservation requested
선택적 보존; 요청된 보존을 위한 경고/오류 처리 추가
on a never-generated IP/BD; validate against the existing checklist; send
생성되지 않은 IP/BD에서; 기존 체크리스트와 비교하여 검증합니다. 보내다
an email/Confluence update.
이메일/Confluence 업데이트.
Working through these actually exercising
실제로 운동을 통해 이러한 작업을 수행합니다.
Vivado (not just `tclsh`) found what the initial pass missed:
Vivado(`tclsh`뿐만 아니라)는 초기 패스에서 놓친 것을 발견했습니다.

**Bug 1 — `generate_build_script` crashed for ANY project with a
**버그 1 — `generate_build_script`가 다음과 같은 프로젝트에서 충돌했습니다.
user-created BD, even with `preserve_selection={}`.** Running the full
`preserve_selection={}`을 사용해도 사용자 생성 BD.** 전체 실행
9-project regression (`revision-control-validate`) failed all 9 projects
9개 프로젝트 회귀('revision-control-validate')가 9개 프로젝트 모두 실패했습니다.
identically with:
동일하게:
```
can't read "preserved_bd_rel_path(<bd_name>)": no such element in array
```
Root cause: Step 7's per-BD dependency-order loop builds one `puts $fd
근본 원인: 7단계의 BD별 종속성 순서 루프는 'puts $fd'를 빌드합니다.
"..."` template per BD, covering both the preserved and non-preserved
보존된 항목과 보존되지 않은 항목을 모두 포함하는 BD별 "..."` 템플릿
cases.
사례.
The line for the preserved case did an UNESCAPED
보존된 케이스의 라인은 UNESCAPED를 수행했습니다.
`$preserved_bd_rel_path($bd_name)` lookup inside that string — Tcl
해당 문자열 내부의 `$preserved_bd_rel_path($bd_name)` 조회 — Tcl
evaluates unescaped `$var`/`[cmd]` inside a double-quoted string
큰따옴표로 묶인 문자열 내에서 이스케이프되지 않은 `$var`/`[cmd]`를 평가합니다.
immediately, at the OUTER proc's (i.e.
OUTER proc에서 즉시(예:
`generate_build_script`'s own)
`generate_build_script` 자신의 것)
runtime, not later when the generated file is sourced.
생성된 파일이 소스로 제공될 때는 나중에 런타임이 아닙니다.
So this array
그래서 이 배열은
lookup ran for every BD in the project, preserved or not, and threw
보존 여부에 관계없이 프로젝트의 모든 BD에 대해 조회가 실행되어 던졌습니다.
whenever the BD wasn't actually in `preserved_bd_names` (the array element
BD가 실제로 `preserved_bd_names`(배열 요소)에 없을 때마다
only exists for actually-preserved BDs).
실제로 보존된 BD에만 존재합니다.)
This is a real regression from
이것은 실제 회귀입니다.
this same 2026-09-29 change, not pre-existing — confirmed by the fact ALL
기존이 아닌 동일한 2026-09-29 변경 — ALL 사실로 확인됨
9 projects failed identically the first time, none of which use
9개의 프로젝트가 처음에 동일하게 실패했으며 그 중 어느 것도 사용하지 않았습니다.
preservation. **Fixed** by computing `_bd_is_preserved` and
보존. **수정됨** `_bd_is_preserved`를 계산하여
`_preserved_bd_rel_dir_here` once per `bd_name` in an ordinary Tcl `if` at
일반 Tcl `if`에서 `bd_name`당 한 번 `_preserved_bd_rel_dir_here`
the OUTER level, before any `puts $fd` line references them, so the array
'puts $fd' 라인이 이를 참조하기 전의 OUTER 레벨이므로 배열은
is only ever touched when the lookup is guaranteed to succeed.
조회 성공이 보장되는 경우에만 터치됩니다.
Lesson for
레슨
future edits to this style of code: an unescaped array/var reference inside
이 스타일의 코드에 대한 향후 편집: 내부에 이스케이프되지 않은 배열/var 참조
a `puts $fd "..."` template needs the same existence guard a normal script
`puts $fd "..."` 템플릿에는 일반 스크립트와 동일한 존재 보호가 필요합니다.
would need at its own runtime — it's easy to forget because most such
자체 런타임에 필요할 것입니다. 대부분의 경우 잊어버리기 쉽습니다.
references are for values (a loop variable, a literal) that are always
참조는 항상 값(루프 변수, 리터럴)에 대한 것입니다.
safe to read.
안전하게 읽을 수 있습니다.

**Bug 2 — a preserved BD's `validate_bd_design`/`save_bd_design` silently
**버그 2 — 보존된 BD의 `validate_bd_design`/`save_bd_design`이 자동으로 유지됩니다.
no-op'd (pre-existing, first live-exercised by this pass's smoke test, not
no-op'd(기존에 이 패스의 연기 테스트를 통해 처음으로 실시간으로 실행되었지만
introduced by 2026-09-29).** Live end-to-end test: preserved
2026-09-29까지 도입).** 라이브 엔드투엔드 테스트: 보존
`MBV_ipi_design`'s `microblaze_v_preset` BD via `build_preserve_selection` +
`build_preserve_selection`을 통한 `MBV_ipi_design`의 `microblaze_v_preset` BD +
a real external cache dir, rebuilt in a clean directory.
깨끗한 디렉토리에 재구축된 실제 외부 캐시 디렉토리.
Rebuild printed
인쇄된 재구축
`Project recreated successfully!` but a full-text scan of the log showed:
'프로젝트가 성공적으로 다시 생성되었습니다!' 그러나 로그의 전체 텍스트 스캔 결과는 다음과 같습니다.
```
ERROR: [BD 5-159] Cannot find design named microblaze_v_preset
ERROR: [BD 5-104] A block design must be open to run this command.
```
Root cause: `add_files`-ing a preserved `.bd` registers it in the fileset
근본 원인: `add_files` - 보존된 `.bd`를 파일 세트에 등록합니다.
but does NOT open it as a live BD design in the current session —
하지만 현재 세션에서는 라이브 BD 디자인으로 열지 않습니다.
`validate_bd_design -design <name>` and `save_bd_design` both need
`validate_bd_design -design <name>` 및 `save_bd_design` 둘 다 필요합니다.
`open_bd_design` first.
'open_bd_design' 먼저.
The non-preserved path gets this "for free" because
보존되지 않은 경로는 이를 "무료"로 얻습니다. 왜냐하면
`source`-ing a `write_bd_tcl` script leaves the BD open as a side effect of
`write_bd_tcl` 스크립트를 `소스`하면 다음의 부작용으로 BD가 열린 상태로 유지됩니다.
recreating it.
그것을 다시 만드는 것.
Both calls were already wrapped in the existing `catch{}`
두 호출 모두 이미 기존 `catch{}`에 래핑되었습니다.
(added when this preserved-BD branch was first written), so the ERRORs
(이 보존된 BD 분기가 처음 작성되었을 때 추가됨) 따라서 ERROR는
never failed the build or `verify_rebuild`'s structural checks — they just
빌드 또는 `verify_rebuild`의 구조적 검사에 결코 실패하지 않았습니다.
silently no-op'd. **Fixed** by adding `catch {open_bd_design [get_files
조용히 아무 조작도 하지 않았습니다. `catch {open_bd_design [get_files)를 추가하여 **수정**
${bd_name}.bd]}` right after `add_files`, before `validate_bd_design`, in
${bd_name}.bd]}` `add_files` 바로 뒤, `validate_bd_design` 앞,
both the dependency-order loop and the glob-fallback loop.
종속성 순서 루프와 글로벌 폴백 루프 모두.
Re-ran the
다시 실행
rebuild after the fix: zero ERROR/CRITICAL WARNING lines in the log.
수정 후 재구축: 로그에 ERROR/CRITICAL WARNING 줄이 없습니다.

**Also added (item 5 of the 7):** `export_all_sources` now checks `.gen`
**또한 추가됨(7개 중 항목 5):** `export_all_sources`는 이제 `.gen`을 확인합니다.
existence for every selected IP/BD itself (not just the interactive
선택된 모든 IP/BD 자체에 대한 존재(대화형 IP/BD뿐만 아니라)
prompt's advisory note, which an agentic `build_preserve_selection` caller
에이전트 `build_preserve_selection` 호출자가 보내는 프롬프트의 권고 사항
never sees) — if missing, it WARNs and falls back to standard export for
전혀 볼 수 없음) - 누락된 경우 경고하고 표준 내보내기로 돌아갑니다.
that one item, rather than writing an incomplete cache entry that would
불완전한 캐시 항목을 작성하는 대신 해당 항목
only surface as a confusing hard error much later, at rebuild time.
훨씬 나중에 다시 빌드할 때 혼란스러운 하드 오류로만 표면화됩니다.
This
이
was a judgment call (warn-and-proceed vs. hard-block) surfaced to GP as an
GP에 대한 판단 요청(경고 및 진행 대 하드 차단)이었습니다.
open question, not yet confirmed either way.
공개 질문이지만 아직 어느 쪽이든 확인되지 않았습니다.

**Full end-to-end verification actually performed this pass** (superseding
**전체 엔드투엔드 검증이 실제로 이 패스를 수행했습니다**(대체
the "not live-verified" state of the initial pass):
초기 패스의 "실시간 확인되지 않음" 상태):
1. Full 9-project regression, re-run after Bug 1's fix: **6 PASS / 3
1. 전체 9개 프로젝트 회귀, 버그 1 수정 후 다시 실행: **6 PASS / 3
BLOCKED** (`ipi_bdc_dfx`, `modnoc-rtl-dfx`, `rtl_dfx`), identical split to
차단됨** (`ipi_bdc_dfx`, `modnoc-rtl-dfx`, `rtl_dfx`), 동일 분할
the documented pre-existing baseline.
문서화된 기존 기준선.
All 3 blocked for their
3개 모두 차단됨
already-documented reasons EXCEPT `ipi_bdc_dfx`, which failed with a NEW
NEW로 실패한 `ipi_bdc_dfx`를 제외하고 이미 문서화된 이유
signature (`ERROR: [BD 5-216] VLNV <xilinx.com:ip:clk_wizard:1.0> is not
서명(`오류: [BD 5-216] VLNV <xilinx.com:ip:clk_wizard:1.0>이 아닙니다.
supported for the current part`) — Vivado IP-catalog version drift on
현재 부분에 지원됨`) — Vivado IP 카탈로그 버전 드리프트
this specific 2026.2 build (SW Build 6636727, 2026-09-28), unrelated to
이 특정 2026.2 빌드(SW 빌드 6636727, 2026-09-28)는 다음과 관련이 없습니다.
this skill or this change.
이 스킬이나 이 변화.
2. Dedicated remote-cache smoke test against `MBV_ipi_design`'s
2. `MBV_ipi_design`에 대한 전용 원격 캐시 스모크 테스트
`microblaze_v_preset` BD (un-synthesized in the reference design, so
`microblaze_v_preset` BD(참조 디자인에서는 합성되지 않았으므로
`upgrade_ip [get_ips] -quiet` + `generate_target all` were needed first):
`upgrade_ip [get_ips] -quiet` + `generate_target all`이 먼저 필요했습니다):
requested preservation before generating → correctly warned + fell back
생성 전 보존 요청 → 올바르게 경고 + 폴백
(Bug-adjacent, confirms the new item-5 check); requested again after
(버그-인접, 새 항목 확인-5 확인); 이후에 다시 요청함
generating → cache got only that BD's own `srcs`/`gen` (confirmed the
생성 → 캐시는 BD 자체 `srcs`/`gen`만 가져왔습니다(확인됨)
nested per-instance IP files ARE captured, live, not just reasoned);
중첩된 인스턴스별 IP 파일은 추론이 아닌 라이브로 캡처됩니다.
`RevisionControl_testB/` had no `.xpr`/`.gen`/`.srcs`; rebuild restored
`RevisionControl_testB/`에는 `.xpr`/`.gen`/`.srcs`가 없습니다. 재건 복원
from cache and Vivado logged `Generated targets are already up-to-date
캐시 및 Vivado에서 '생성된 타겟은 이미 최신 상태입니다.
... hence not re-generating` (skip-resynthesis confirmed live); renaming
...따라서 재생성되지 않음`(생략-재합성 확인 라이브); 이름 바꾸기
the cache dir and re-running rebuild produced the expected hard error
캐시 디렉토리와 재구축을 다시 실행하면 예상되는 하드 오류가 발생했습니다.
(`error "Cannot rebuild: ..."`, exit code 1, halted before touching
(`오류 "재구축할 수 없음: ..."`, 종료 코드 1, 터치하기 전에 중지됨
project state).
프로젝트 상태).
3. **Not independently live-tested this pass:** the equivalent multi-IP
3. **이 패스는 독립적으로 실시간 테스트되지 않았습니다.** 동등한 다중 IP
mixed-selection path (some standalone IPs preserved, others not) — only
혼합 선택 경로(일부 독립 실행형 IP는 보존되고 다른 일부는 보존되지 않음) - 전용
confirmed via code review; structurally identical to the BD path that
코드 검토를 통해 확인되었습니다. BD 경로와 구조적으로 동일합니다.
WAS live-tested.
실시간 테스트를 거쳤습니다.
`MBV_ipi_design` only has one eligible item (the BD);
'MBV_ipi_design'에는 적격 항목이 하나만 있습니다(BD).
a design with ≥2 standalone IPs (check `RTL_only_design`/`ipi_multi_bd`
독립형 IP가 2개 이상인 설계(`RTL_only_design`/`ipi_multi_bd` 확인)
first) would be needed to close this out.
먼저) 이 문제를 해결하려면 필요합니다.

Smoke test scratch logs kept at
스모크 테스트 스크래치 로그는 다음 위치에 보관됩니다.
`agentic-ai-suite/tmp/preserve_smoke_test/*.log` (bulky Vivado working
`agentic-ai-suite/tmp/preserve_smoke_test/*.log`(대량 Vivado 작동
directories deleted after use).
사용 후 삭제된 디렉토리).

## 2026-09-21

### Change: consolidate preserve-outputs onto a single project_1.gen/.srcs/.xpr artifact
### 변경: 보존 출력을 단일 project_1.gen/.srcs/.xpr 아티팩트로 통합

Removed the separate `.sources/IP|BD`, `.gen/<name>`, `.XPR/<proj>.xpr`
별도의 `.sources/IP|BD`, `.gen/<name>`, `.XPR/<proj>.xpr`을 제거했습니다.
per-item mechanism (added 2026-08-23) that existed in parallel with the
와 병행하여 존재했던 항목별 메커니즘(2026-08-23에 추가됨)
full-tree `<proj>.gen/.srcs/.xpr` copy (added 2026-09-11) — the full-tree
전체 트리 `<proj>.gen/.srcs/.xpr` 복사본(2026-09-11에 추가됨) — 전체 트리
copy is now the only preserve-outputs artifact `export_all_sources` creates.
copy는 이제 'export_all_sources'가 생성하는 유일한 보존 출력 아티팩트입니다.

**Why two mechanisms existed:** the per-item one was added first and is what
**두 가지 메커니즘이 존재하는 이유:** 항목별 메커니즘이 먼저 추가되었으며
`generate_build_script` actually read from; the full-tree copy was added
`generate_build_script`는 실제로 읽습니다. 전체 트리 복사본이 추가되었습니다
later as a human-browsable reference and was explicitly documented as
나중에 사람이 탐색할 수 있는 참조로 명시적으로 문서화되었습니다.
"additive, not a replacement... making rebuild actually consume it directly
"교체 아닌 추가…재구축하면 실제로 직접 소비"
would be a separate, larger change, out of scope." This change is that
범위를 벗어나 별도의 더 큰 변경이 있을 것입니다." 이번 변화는
larger change.
더 큰 변화.

**The collision this required solving:** `generate_build_script`'s
**충돌 해결이 필요함:** `generate_build_script`'s
`create_project $proj_name . -force` runs before anything else touches
`create_project $proj_name . -force`는 다른 것이 닿기 전에 실행됩니다.
IPs/BDs.
IP/BD.
Investigated whether this collides with the full-tree copy at
이것이 전체 트리 복사본과 충돌하는지 조사했습니다.
rebuild time: under the *documented* invocation convention (cwd = a
재구축 시간: *문서화된* 호출 규칙에 따라(cwd = a
directory containing `RevisionControl/` as a subdirectory, per SKILL.md's
SKILL.md에 따라 `RevisionControl/`을 하위 디렉토리로 포함하는 디렉토리
"After Completion" section and `generate-build-script/REFERENCE.md`'s own
"완료 후" 섹션 및 `generate-build-script/REFERENCE.md` 자체
comment), it does not — confirmed via a live end-to-end rebuild, cwd puts
설명), 그렇지 않습니다 — 실시간 엔드투엔드 재구축을 통해 확인되었으며, cwd puts
the freshly created project as a *sibling* of `RevisionControl/`, not at the
새로 생성된 프로젝트를 `RevisionControl/`의 *형제*로,
same path as the preserved copy inside it.
그 안에 있는 보존된 사본과 동일한 경로입니다.
But a different, very plausible
그러나 다른, 매우 그럴듯한
invocation (`cd RevisionControl; vivado -mode batch -source Scripts/build.tcl`)
호출(`cd RevisionControl; vivado -mode 배치 -source Scripts/build.tcl`)
*would* put `create_project -force` at the same path as the preserved copy,
*보존된 복사본과 동일한 경로에 `create_project -force`를 넣을 것입니다.
destroying it before anything read it.
무엇이든 읽기 전에 그것을 파괴하세요.
Rather than depend on which
어느쪽에 의존하기보다는
convention whoever runs `build.tcl` happens to use, `generate_build_script`
`build.tcl`을 실행하는 사람은 누구나 `generate_build_script`를 사용하게 됩니다.
now emits a new **Step 0**: before `create_project` runs, it stages exactly
이제 새로운 **0단계**를 내보냅니다. `create_project`가 실행되기 전에 정확하게 단계화됩니다.
what's needed for each preserved IP/BD (that item's `.gen` output + `.xci`/
보존된 각 IP/BD에 필요한 것(해당 항목의 `.gen` 출력 + `.xci`/
`.bd` source) out of the full-tree copy into `.rc_preserve_staging/`, and
`.bd` 소스)를 전체 트리 복사본에서 `.rc_preserve_staging/`으로 복사하고
Steps 1c/6/7 read from there instead of `.gen`/`.srcs` directly.
1c/6/7 단계는 `.gen`/`.srcs` 대신 거기에서 직접 읽습니다.
`.rc_preserve_staging/` is rebuild-time-only scratch — regenerated every
`.rc_preserve_staging/`은 재구축 시간 전용 스크래치입니다.
run, gitignored, never created by `export_all_sources` itself.
실행, gitignored, `export_all_sources` 자체에 의해 생성되지 않았습니다.

**A real bug found and fixed along the way:** the old per-item mechanism
**진짜 버그가 발견되어 수정되었습니다:** 이전 항목별 메커니즘
(and my first pass at Step 0) assumed a preserved IP/BD's source always
(그리고 0단계의 첫 번째 패스)는 항상 보존된 IP/BD 소스를 가정했습니다.
lives under the `sources_1` fileset.
`sources_1` 파일 세트에 있습니다.
Confirmed false on this skill's own
이 스킬 자체가 허위로 확인되었습니다
`modnoc-rtl-flat` example design — its `xlnoc` BD lives under
`modnoc-rtl-Flat` 예시 디자인 — `xlnoc` BD는 다음과 같습니다.
`sim_1/bd/xlnoc/xlnoc.bd`.
`sim_1/bd/xlnoc/xlnoc.bd`.
Fixed by having `export_all_sources` record each
각각 `export_all_sources` 레코드를 가짐으로써 수정되었습니다.
preserved item's actual fileset-relative path (`preserved_ip_rel_path`/
보존된 항목의 실제 파일 세트 상대 경로(`preserved_ip_rel_path`/
`preserved_bd_rel_path`, new globals) at export time — computed via the
`preserved_bd_rel_path`, 새 전역) 내보내기 시 — 다음을 통해 계산됩니다.
already-live Vivado file object, never guessed — and having Step 0 stage
이미 활성화된 Vivado 파일 객체, 추측할 수 없음 — 0단계 단계
from that exact path.
그 정확한 경로에서.
Also added a guard: if a selected item's source lives
또한 가드를 추가했습니다: 선택한 항목의 소스가 살아있는 경우
outside `$proj_dir` entirely (Remote/Mixed scenario), it now falls back to
`$proj_dir` 외부에서 완전히(원격/혼합 시나리오), 이제 다음으로 대체됩니다.
standard export instead of recording a path that could never resolve (the
결코 확인할 수 없는 경로를 기록하는 대신 표준 내보내기(
full-tree copy only ever captures what's inside `$proj_dir`).
전체 트리 복사는 `$proj_dir` 내부 내용만 캡처합니다.

**Also fixed:** the generated `.gitignore`/`README.md` unconditionally
**또한 수정됨:** 무조건 생성된 `.gitignore`/`README.md`
ignored `${proj_name}.gen/`/`.srcs/` — harmless for Classic Flow (external
`${proj_name}.gen/`/`.srcs/` 무시 — Classic Flow에 무해합니다(외부
siblings, a gitignore inside `RevisionControl/` can't reach them anyway) but
형제 자매, `RevisionControl/` 내부의 gitignore는 어쨌든 그들에게 접근할 수 없습니다) 하지만
silently defeated Interactive Flow's entire purpose, since that's now the
Interactive Flow의 전체 목적을 조용히 무산시켰습니다.
one artifact that must be checked in.
체크인해야 하는 아티팩트 하나.
Both are now conditional on whether
둘 다 이제 조건부입니다.
anything was actually preserved.
실제로 무엇이든 보존되었습니다.

**Known limitation, pre-existing, not introduced here:** BD preservation
**알려진 제한 사항, 기존, 여기에 소개되지 않음:** BD 보존
stages only the top-level `<bd>.bd`, not a per-instance `ip/<name>/<name>.xci`
인스턴스별 `ip/<name>/<name>.xci`가 아닌 최상위 `<bd>.bd`만 스테이지합니다.
subdirectory that can sit alongside it for IPs customized only inside that
그 내부에서만 사용자 정의된 IP를 위해 옆에 있을 수 있는 하위 디렉터리
BD — confirmed the old `.sources/BD` mechanism had the identical gap (also
BD — 이전 `.sources/BD` 메커니즘에 동일한 차이가 있음을 확인했습니다(또한
copied only the single `.bd` file).
단일 `.bd` 파일만 복사했습니다).
A rebuild can print `[BD 41-2576] File
다시 빌드하면 `[BD 41-2576] 파일을 인쇄할 수 있습니다.
... could not be found` for those nested IPs even though the BD's own `.bd`
... BD 자체 `.bd`에도 불구하고 중첩된 IP에 대해서는 찾을 수 없습니다.
restores correctly; not a hard failure, just incomplete preservation for
올바르게 복원합니다. 심각한 고장이 아니라 단지 불완전한 보존일 뿐입니다.
that BD's nested IPs.
해당 BD의 중첩 IP입니다.
Worth a dedicated fix, not attempted here.
여기서는 시도하지 않았지만 전용 수정이 필요합니다.

**Verified:**
**확인됨:**
- `tclsh` source-only smoke test after every edit (syntax/brace check).
- 모든 편집 후 `tclsh` 소스 전용 스모크 테스트(구문/중괄호 확인).
- Live end-to-end test via a real pty (not simulated) on `modnoc-rtl-flat`:
- `modnoc-rtl-Flat`에서 실제 pty(시뮬레이션 아님)를 통한 실시간 엔드투엔드 테스트:
auto-prompt fires, answering `y`/`all` produces the consolidated structure
자동 프롬프트가 실행되고 `y`/`all`이라고 대답하면 통합 구조가 생성됩니다.
(no `.sources`/`.gen`/`.XPR`), and — new this session — actually **ran the
(`.sources`/`.gen`/`.XPR` 없음) 그리고 — 이 새로운 세션 — 실제로 **실행되었습니다.
generated `build.tcl`** end-to-end: `.rc_preserve_staging/` staged
생성된 `build.tcl`** 엔드투엔드: `.rc_preserve_staging/` staged
correctly (including the `sim_1`-fileset `xlnoc` case), the preserved BD's
올바르게(`sim_1`-fileset `xlnoc` 사례 포함) 보존된 BD의
`.bd` was found and added, `project_1.gen/.srcs/.xpr` survived the rebuild
`.bd`가 발견되어 추가되었습니다. `project_1.gen/.srcs/.xpr`은 재구축 후에도 살아 남았습니다.
completely intact (byte-identical file count before/after), and the
완전히 손상되지 않았습니다(이전/이후의 바이트와 동일한 파일 수).
rebuild finished with "Project recreated successfully!" (with the known
"프로젝트가 성공적으로 재작성되었습니다!"라는 메시지와 함께 재구축이 완료되었습니다. (알려진 것과 함께
nested-IP limitation above producing warnings, not a failure).
오류가 아닌 경고 생성 이상의 중첩 IP 제한).
- Classic Flow (non-preserve) sanity re-run on `MBV_ipi_design`:
- `MBV_ipi_design`에서 클래식 흐름(비보존) 온전성 재실행:
`REBUILD_RESULT=PASS`, confirmed zero `.sources`/`.gen`/`.XPR`/
`REBUILD_RESULT=PASS`, 0으로 확인됨 `.sources`/`.gen`/`.XPR`/
`.rc_preserve_staging` artifacts leak in when nothing was preserved.
아무것도 보존되지 않은 경우 `.rc_preserve_staging` 아티팩트가 누출됩니다.

### Change: `export_all_sources` now auto-prompts for Step 2.5 in a real interactive session
### 변경 사항: `export_all_sources`는 이제 실제 대화형 세션에서 2.5단계에 대한 자동 프롬프트를 표시합니다.

Previously, Step 2.5 (`prompt_preserve_outputs`) was opt-in in the strongest
이전에는 2.5단계(`prompt_preserve_outputs`)가 가장 강력한 옵션으로 선택되었습니다.
sense: a human running the pipeline step-by-step had to already know it
의미: 파이프라인을 단계별로 실행하는 사람은 이미 그것을 알고 있어야 했습니다.
existed and call it themselves before Step 3, or preservation was silently
3단계 이전에 존재하고 스스로 호출했거나 보존이 자동으로 수행되었습니다.
skipped with no indication anything was even eligible.
적격하다는 표시도 없이 건너뛰었습니다.

**Fix:** new helper `_rc_stdin_is_tty` (`exec test -t 0`, fails closed — i.e.
**수정:** 새로운 도우미 `_rc_stdin_is_tty`(`exec test -t 0`, 실패 시 닫힘 - 예:
returns "not a tty" — on any error) detects a real interactive terminal.
오류 발생 시 "tty가 아님"을 반환)은 실제 대화형 터미널을 감지합니다.
`export_all_sources` now checks, right after its preserve_selection
`export_all_sources`는 이제 보존 선택 직후에 확인합니다.
parameter defaults: if the caller left `preserve_selection` at its raw `{}`
매개변수 기본값: 호출자가 원시 `{}`에서 `preserve_selection`을 떠난 경우
default *and* stdin is a real tty *and* `list_preservable_outputs` finds at
기본 *그리고* stdin은 실제 tty입니다 *그리고* `list_preservable_outputs`는 다음에서 찾습니다.
least one eligible IP/BD, it calls `prompt_preserve_outputs` itself before
적격한 IP/BD가 하나 이상 있으면 'prompt_preserve_outputs' 자체를 호출합니다.
proceeding, instead of silently defaulting to no preservation.
보존하지 않음을 자동으로 기본값으로 설정하는 대신 계속 진행합니다.

This only changes behavior for a genuine interactive human session:
이는 실제 대화형 휴먼 세션의 동작만 변경합니다.
- Automated/CI callers (`run_pipeline.tcl`-style harnesses) never have a
- 자동화된/CI 호출자(`run_pipeline.tcl` 스타일 하네스)에는
real tty on stdin (confirmed via `tclsh` smoke test with both `/dev/null`-
stdin의 실제 tty(`/dev/null`을 모두 사용한 `tclsh` 스모크 테스트를 통해 확인됨-
and pipe-redirected stdin, matching how `revision-control-validate`'s
그리고 `revision-control-validate`의 방식과 일치하는 파이프 리디렉션 stdin
`run_project.sh` actually invokes Vivado) — `_rc_stdin_is_tty` returns 0,
`run_project.sh`는 실제로 Vivado를 호출합니다) — `_rc_stdin_is_tty`는 0을 반환합니다.
so this code path is never reached and behavior is byte-for-byte
따라서 이 코드 경로는 도달하지 않으며 동작은 바이트 단위입니다.
unchanged.
변함없이.
- Agentic callers (an AI agent driving Vivado via Bash/batch-mode Tcl calls)
- 에이전트 호출자(Bash/배치 모드 Tcl 호출을 통해 Vivado를 구동하는 AI 에이전트)
also have no real tty on stdin — unaffected.
또한 stdin에는 실제 tty가 없습니다. 영향을 받지 않습니다.
They should keep using
계속 사용해야 합니다.
`build_preserve_selection` as documented.
문서화된 대로 `build_preserve_selection`.
- A human who already called `prompt_preserve_outputs` or
- 이미 `prompt_preserve_outputs`를 호출한 사람 또는
`build_preserve_selection` themselves and passed a populated
`build_preserve_selection` 자체를 입력하고 채워진 값을 전달했습니다.
`preserve_selection` in is also unaffected — the auto-check only fires
'preserve_selection'도 영향을 받지 않습니다. 자동 확인만 실행됩니다.
when the argument is still the untouched default.
인수가 여전히 변경되지 않은 기본값인 경우.

Not verified against a live interactive Vivado GUI/tty session this
실시간 대화형 Vivado GUI/tty 세션에 대해 확인되지 않았습니다.
session (no such session was available to test from) — verified only that
세션(테스트에 사용할 수 있는 세션이 없음) — 다음만 확인되었습니다.
the non-tty path is unaffected (via `tclsh`) and that the file still parses
tty가 아닌 경로는 영향을 받지 않으며(`tclsh`를 통해) 파일은 여전히 구문 분석됩니다.
cleanly.
깨끗하게.
If this doesn't actually prompt in a real GUI Tcl console (Vivado's
실제 GUI Tcl 콘솔(Vivado의
console may not expose a conventional process stdin the same way a shell
콘솔은 쉘과 같은 방식으로 기존 프로세스 stdin을 노출하지 않을 수 있습니다.
does), that would need a follow-up fix specific to how Vivado's GUI console
그렇습니다), Vivado의 GUI 콘솔이 어떻게 작동하는지에 대한 후속 수정이 필요합니다.
handles `gets stdin`/`exec test -t 0`.
`gets stdin`/`exec test -t 0`을 처리합니다.

## 2026-09-16

Three fixes from a regression re-run of `revision-control-validate` (2026-09-16
'revision-control-validate' 회귀 재실행으로 인한 세 가지 수정 사항(2026-09-16)
vs. the 2026-08-10 baseline) against all 9 example designs.
9가지 예시 디자인 모두에 대해 2026-08-10 기준과 비교합니다.
All in
올인
`helper_scripts.tcl` unless noted.
따로 언급하지 않는 한 `helper_scripts.tcl`.

### Fix: RM-owned IP/RTL parented outside the RM's own fileset was invisible to export
### 수정: RM 자체 파일 세트 외부에 있는 RM 소유 IP/RTL은 내보낼 수 없습니다.

Both `export_all_sources`'s DFX RM export loop and `capture_project_settings`'s
`export_all_sources`의 DFX RM 내보내기 루프와 `capture_project_settings` 모두
`dfx_reconfig_modules` manifest capture discovered RM-owned files via
`dfx_reconfig_modules` 매니페스트 캡처는 다음을 통해 발견된 RM 소유 파일을 캡처합니다.
`get_files -quiet -of_objects [get_reconfig_modules $rm]` only.
`get_files -quiet -of_objects [get_reconfig_modules $rm]`만 가능합니다.
If an IP was
IP가 다음과 같은 경우
added to the project while a *different* fileset (typically `sources_1`) was
*다른* 파일 세트(일반적으로 `sources_1`)가 프로젝트에 추가된 경우
Vivado's "current" fileset — even though the IP is only instantiated from
Vivado의 "현재" 파일 세트 — IP가 다음에서만 인스턴스화되는 경우에도 마찬가지입니다.
inside that RM's own RTL hierarchy — that query never sees it.
해당 RM의 자체 RTL 계층 구조 내에서 해당 쿼리는 이를 볼 수 없습니다.
The file then
그러면 파일
falls through to being recreated at project scope on rebuild (Step 6 of
재구축 시 프로젝트 범위에서 다시 생성됩니다(6단계).
`generate_build_script`), invisible to that RM's own out-of-context
`generate_build_script`), 해당 RM 자신의 컨텍스트 외부에는 표시되지 않음
synthesis, producing `[Synth 8-439] module 'X' not found`.
합성, `[Synth 8-439] 모듈 'X'를 찾을 수 없음`을 생성합니다.

This is an ordinary, easy-to-hit real-world DFX authoring mistake (forgetting
이는 평범하고 쉽게 발생하는 실제 DFX 작성 실수입니다(잊어버리기).
to switch `current_fileset` before adding an IP via the IP Catalog/GUI), not
IP 카탈로그/GUI를 통해 IP를 추가하기 전에 `current_fileset`을 전환하려면)
specific to a broken example — confirmed each RM also has its own named
깨진 예에만 해당 - 각 RM에도 고유한 이름이 있음을 확인했습니다.
fileset on disk (e.g.
디스크의 파일 세트(예:
`rp1_rm1/`) that the export logic never queried.
`rp1_rm1/`) 내보내기 로직이 쿼리한 적이 없습니다.

**Fix:** new shared helper `_rc_get_rm_files {rm rm_name}` unions
**수정:** 새로운 공유 도우미 `_rc_get_rm_files {rm rm_name}` 공용체
`get_files -of_objects $rm` with `get_files -of_objects [get_filesets $rm_name]`
`get_files -of_objects $rm`과 `get_files -of_objects [get_filesets $rm_name]`
(dedup'd by normalized path), used identically at both call sites so a file
(정규화된 경로로 중복 제거됨), 두 호출 사이트 모두에서 동일하게 사용되므로 파일
that's exported is also attached at rebuild time (or neither, never one
내보낸 파일은 재구축 시에도 첨부됩니다.
without the other).
다른 것 없이).

**NOT actually verified — correction from the 2026-09-21 regression re-run:**
**실제로 확인되지 않음 - 2026-09-21 회귀 재실행 수정 사항:**
this fix was originally claimed to target `modnoc-rtl-dfx`'s six
이 수정 사항은 원래 `modnoc-rtl-dfx`의 6개 버전을 대상으로 한다고 주장되었습니다.
module-not-found errors (`axi_dbg_hub_rm1/2`, `axis_vio_pl_master_to_ddr_rm1/2`,
모듈을 찾을 수 없는 오류(`axi_dbg_hub_rm1/2`, `axis_vio_pl_master_to_ddr_rm1/2`,
`perf_axi_tg_pl_master_to_ddr_rm1/2`, first seen in the 2026-09-16 regression
`perf_axi_tg_pl_master_to_ddr_rm1/2`, 2026-09-16 회귀에서 처음 확인됨
report).
보고).
A 2026-09-21 re-run against the current (post-fix) code reproduced
재현된 현재(수정 후) 코드에 대해 2026-09-21 재실행
the identical errors, and direct comparison of the pre-fix (2026-09-16) and
동일한 오류, 접두사(2026-09-16)와 직접 비교
post-fix (2026-09-21) export trees showed the `.xci` files for all six RMs
post-fix(2026-09-21) 내보내기 트리에는 6개 RM 모두에 대한 '.xci' 파일이 표시되었습니다.
were **already present** in `Sources/RTL/` before this fix existed — i.e.
이 수정 사항이 존재하기 전에 `Sources/RTL/`에 **이미 존재**했습니다. 즉,
`get_files -of_objects $rm` alone already found them for this project, and
`get_files -of_objects $rm`만으로도 이미 이 프로젝트에서 해당 항목을 찾았으며
`_rc_get_rm_files`'s union added nothing.
`_rc_get_rm_files`의 연합은 아무것도 추가하지 않았습니다.
The real cause of these six errors
이 여섯 가지 오류의 실제 원인
is an IP-catalog version lock: from the moment `modnoc-rtl-dfx` is opened,
IP 카탈로그 버전 잠금입니다. 'modnoc-rtl-dfx'가 열리는 순간부터
every IP in the project (RM-owned and not) logs
프로젝트의 모든 IP(RM 소유 및 소유하지 않음)는 로그에 기록됩니다.
`WARNING: [IP_Flow 19-2162] IP 'X' is locked ... current catalog revision is
`경고: [IP_Flow 19-2162] IP 'X'가 잠겨 있습니다. 현재 카탈로그 개정은
'unknown'` — an environment/IP-version issue (the IP never regenerates
'알 수 없음'` — 환경/IP 버전 문제(IP가 재생성되지 않음)
output products, so synth can't find its module), unrelated to fileset
출력 제품이므로 신디사이저가 해당 모듈을 찾을 수 없음), 파일 세트와 관련 없음
visibility.
가시성.
This reproduces identically with or without this fix.
이 수정 프로그램을 사용하거나 사용하지 않고 동일하게 재현됩니다.

The union logic in `_rc_get_rm_files` is still plausible as a general
`_rc_get_rm_files`의 결합 논리는 일반적으로 여전히 그럴듯합니다.
hardening (an IP added under the wrong `current_fileset` is a real class of
강화(잘못된 `current_fileset` 아래에 추가된 IP는
mistake), but it has **no confirmed real-world repro** as of 2026-09-21 —
실수), 하지만 2026년 9월 21일 현재 **확인된 실제 재현은 없습니다** —
treat it as code-reviewed-but-unverified until one is found, not as a fix
수정 사항이 아니라 발견될 때까지 코드 검토를 거쳤지만 확인되지 않은 것으로 처리합니다.
proven against `modnoc-rtl-dfx`.
'modnoc-rtl-dfx'에 대해 입증되었습니다.

**Explicitly NOT fixed, and not attempted:** `rtl_dfx`'s `axi_traffic_gen_N`
**명시적으로 수정되지 않았으며 시도되지 않았습니다:** `rtl_dfx`의 `axi_traffic_gen_N`
and `modnoc-rtl-dfx`'s `RP1_rm1`/`RP1_rm2` stale-absolute-path failures — both
및 `modnoc-rtl-dfx`의 `RP1_rm1`/`RP1_rm2` 오래된 절대 경로 오류 — 둘 다
already confirmed (2026-08-10 entry below) to be pre-existing defects in the
이미 확인된 사항(2026-08-10 아래 항목)은 기존 결함인 것으로 확인되었습니다.
example designs themselves, reproducible even on a pristine, never-exported
예제 디자인 자체는 결코 수출되지 않은 깨끗한 상태에서도 재현 가능합니다.
scratch copy of the original `.xpr`.
원본 `.xpr`의 스크래치 복사본.
Re-litigating those was out of scope.
이를 재소송하는 것은 범위를 벗어났습니다.

### Fix: `verify_rebuild` false-PASS gap (Issue #37)
### 수정: `verify_rebuild` false-PASS 간격(문제 #37)

`verify_rebuild` only checked live structural project state (BD/top/run
'verify_rebuild'는 실제 구조 프로젝트 상태만 확인했습니다(BD/top/run
presence, IP count) — it never parsed `rebuild.log`, so a
존재 여부, IP 수) — `rebuild.log`를 구문 분석하지 않았으므로
generate_build_script step that caught a failure and printed `ERROR: ...`
실패를 포착하고 `ERROR: ...`를 인쇄한 generate_build_script 단계
without aborting (confirmed cases: BDC Monitor-mode's `BD 41-258` on
중단하지 않고 (확인된 사례: BDC 모니터 모드의 `BD 41-258` on
`ipi_bdc_flat_flow`, and a NoC PHY IP `generate_target` failure on
`ipi_bdc_plat_flow` 및 NoC PHY IP `generate_target` 실패
`segmented_config` — see below) still reported an overall PASS.
`segmented_config` — 아래 참조)는 여전히 전체 PASS를 보고했습니다.

**Fix:** `verify_rebuild` now takes an optional second argument,
**수정:** `verify_rebuild`는 이제 선택적 두 번째 인수를 취합니다.
`{rebuild_log ""}`.
`{rebuild_log ""}`.
If a path is given and the file exists, it's scanned for
경로가 지정되고 파일이 존재하는 경우 해당 파일을 검색합니다.
`^ERROR:` lines; any found set `all_ok 0` (capped display: first 5 + a
`^오류:` 줄; 발견된 세트 `all_ok 0`(제한된 표시: 처음 5 + a
count).
카운트).
No allowlist — every `ERROR:`-prefixed line seen so far is a genuine
허용 목록 없음 — 지금까지 확인된 모든 'ERROR:' 접두사 줄은 진짜입니다.
failure; add one only if a real benign case turns up.
실패; 진짜 양성 사례가 발견된 경우에만 하나를 추가하세요.
Backward compatible:
이전 버전과 호환:
existing single-arg callers are unaffected.
기존 단일 인수 호출자는 영향을 받지 않습니다.

Also added, in the separate `revision-control-validate` validation harness
또한 별도의 `revision-control-validate` 유효성 검사 하네스에 추가되었습니다.
(`run_project.sh`, not part of this skill/repo): an additive
(`run_project.sh`, 이 기술/저장소의 일부가 아님): 추가
`grep -c '^ERROR:'` safety-net check that emits `PROJECT_RESULT=WARN` if the
`grep -c '^ERROR:'` `PROJECT_RESULT=WARN`을 내보내는 안전망 검사
rebuild log both reports `REBUILD_RESULT=PASS` and contains `ERROR:` lines —
재구축 로그는 모두 `REBUILD_RESULT=PASS`를 보고하고 `ERROR:` 줄을 포함합니다 —
independent of the Tcl-side fix, doesn't overwrite `REBUILD_RESULT`.
Tcl 측 수정과 별도로 `REBUILD_RESULT`를 덮어쓰지 않습니다.

**Verified against:** dry-run grep of the 2026-09-16 report's existing
**검증 대상:** 2026-09-16 보고서의 기존 테스트 실행 grep
`rebuild.log` files.
`rebuild.log` 파일.
`ipi_bdc_flat_flow` — 0 pre-existing `ERROR:` lines
`ipi_bdc_Flat_flow` — 기존 `ERROR:` 라인 0개
outside the 8 known `BD 41-258` ones (this fix correctly turns that PASS into
알려진 8개의 'BD 41-258' 이외의 것(이 수정으로 해당 PASS가 올바르게 변경됨)
a FAIL).
실패).
Unexpectedly also found `segmented_config` — reported as a clean
예기치 않게 'segmented_config'도 발견되었습니다. — 깨끗한 것으로 보고되었습니다.
PASS in every prior run — actually contains 20 `ERROR:` lines (NoC PHY IP
모든 이전 실행에서 PASS — 실제로는 20개의 `ERROR:` 라인을 포함합니다(NoC PHY IP
`generate_target` failure, see the next entry) that no prior structural check
`generate_target` 실패, 다음 항목 참조) 사전 구조 확인이 없음
caught; this fix correctly surfaces that too.
잡혔다; 이 수정 사항은 그 문제도 올바르게 나타냅니다.

### Investigated, NOT fixable: NoC memory-controller PHY IP `.elf`/simulation-output failures
### 조사됨, 수정 불가능: NoC 메모리 컨트롤러 PHY IP `.elf`/시뮬레이션 출력 오류

Affects `modnoc-rtl-dfx` and `ipi_bdc_dfx` (`ERROR: [Common 17-275] File does
`modnoc-rtl-dfx` 및 `ipi_bdc_dfx`에 영향을 미칩니다(`ERROR: [Common 17-275] 파일은
not exist [...bd_<hash>_MC0_ddrc_0_phy_ddrmc.elf]` at OOC-synth time) and
OOC 합성 시간에 [...bd_<hash>_MC0_ddrc_0_phy_ddrmc.elf]`가 존재하지 않음) 및
`segmented_config` (`generate_target` failing outright for the top-level BD,
`segmented_config`(`generate_target`은 최상위 BD에 대해 완전히 실패합니다.
previously masked by the Issue #37-style false-PASS gap above).
이전에는 위의 Issue #37 스타일 false-PASS 간격으로 가려졌습니다).

**Confirmed root cause: a Vivado *installation* gap, not an export/rebuild
**확인된 근본 원인: 내보내기/재구축이 아닌 Vivado *설치* 공백
bug.** The NoC memory-controller PHY IP (`noc_mc_ddr5_phy_v1_0`) generates
버그.** NoC 메모리 컨트롤러 PHY IP(`noc_mc_ddr5_phy_v1_0`)는
its simulation BFM output by copying numbered `map_read_<N>.sv`/
번호가 매겨진 `map_read_<N>.sv`/를 복사하여 시뮬레이션 BFM 출력
`pinout_<N>.sv` files from the Vivado install's own
Vivado 설치 자체의 `pinout_<N>.sv` 파일
`data/ip/xilinx/noc_mc_ddr5_phy_v1_0/hdl/bfm/PINOUT/` directory.
`data/ip/xilinx/noc_mc_ddr5_phy_v1_0/hdl/bfm/PINOUT/` 디렉토리.
On the
에
install used for this diagnosis (2026.2,
이번 진단에 사용된 설치(2026.2,
`/proj/xbuilds/CustTA_plus/9999.0_2026_0724_2226plus/...`), several indices
`/proj/xbuilds/CustTA_plus/9999.0_2026_0724_2226plus/...`), 여러 인덱스
are missing entirely (confirmed by direct filesystem listing — `pinout_14`,
완전히 누락되었습니다(직접 파일 시스템 목록 — `pinout_14`로 확인됨,
`pinout_19`/`map_read_19` do not exist, neighboring indices do), so
`pinout_19`/`map_read_19`가 존재하지 않으며 인접한 인덱스는 존재합니다), 따라서
generation fails identically regardless of whether the skill is involved at
기술이 관련되어 있는지 여부에 관계없이 생성이 동일하게 실패합니다.
all.
모두.
Also confirmed none of the 9 example designs ship a pre-generated
또한 9가지 예시 디자인 중 사전 생성된 디자인이 제공되지 않는 것으로 확인되었습니다.
`.gen`/`.elf` for this IP, so there is no existing valid artifact anywhere to
이 IP에 대한 `.gen`/`.elf`이므로 어디에도 기존의 유효한 아티팩트가 없습니다.
preserve/copy through export (the fix pattern used for `preserve_selection`
내보내기를 통해 보존/복사('preserve_selection'에 사용되는 수정 패턴)
does not apply — there's nothing to preserve).
적용되지 않습니다. 보존할 항목이 없습니다.)

**Not fixed.** No Tcl change in this skill can synthesize a Vivado IP-library
**수정되지 않았습니다.** 이 스킬의 Tcl 변경으로 인해 Vivado IP 라이브러리를 합성할 수 없습니다.
file that doesn't exist on disk.
디스크에 존재하지 않는 파일입니다.
Documented in
문서화됨
[generate-build-script/REFERENCE.md](../generate-build-script/REFERENCE.md)'s
[generate-build-script/REFERENCE.md](../generate-build-script/REFERENCE.md)의
"Known Limitation" section, and the misleading debug-net-only hint in
"알려진 제한 사항" 섹션 및 오해의 소지가 있는 디버그 넷 전용 힌트
`generate_build_script`'s `generate_target` catch block (Step 9) was broadened
`generate_build_script`의 `generate_target` catch 블록(9단계)이 확장되었습니다.
to also name this failure class, since the old hint would have sent someone
이 실패 클래스의 이름도 지정해야 합니다. 이전 힌트가 누군가에게 메시지를 보냈을 것이기 때문입니다.
chasing ILA/mark_debug connections instead of a missing install file.
누락된 설치 파일 대신 ILA/mark_debug 연결을 추적합니다.

## 2026-09-11

### Change: full `<proj_name>.gen/.srcs/.xpr` copy inside `RevisionControl/` for Interactive Flow
### 변경: 대화형 흐름에 대한 `RevisionControl/` 내부 `<proj_name>.gen/.srcs/.xpr` 전체 복사본

Team decision, reviewed against the skill's documented output structure: the
기술의 문서화된 출력 구조와 비교하여 검토된 팀 결정:
per-item preserve-outputs mechanism added 2026-08-23 (`.sources/`, `.gen/`,
항목별 출력 보존 메커니즘이 2026년 8월 23일에 추가되었습니다(`.sources/`, `.gen/`,
`.XPR/` under the export dir, scoped to only the IPs/BDs a user chose to
내보내기 디렉토리 아래의 `.XPR/`, 사용자가 선택한 IP/BD로만 범위 지정됨
preserve) is being extended, not replaced, for Interactive Flow.
보존)은 Interactive Flow를 위해 교체되지 않고 확장되고 있습니다.
Whenever
언제라도
`preserve_selection` (Step 2.5) selects at least one IP or BD,
`preserve_selection`(2.5단계)은 하나 이상의 IP 또는 BD를 선택합니다.
`export_all_sources` now additionally copies the **entire** original project
`export_all_sources`는 이제 **전체** 원본 프로젝트를 추가로 복사합니다.
directories — `$proj_dir/<proj_name>.gen`, `$proj_dir/<proj_name>.srcs`,
디렉터리 — `$proj_dir/<proj_name>.gen`, `$proj_dir/<proj_name>.srcs`,
`$proj_dir/<proj_name>.xpr` — directly into `$export_dir` under the
`$proj_dir/<proj_name>.xpr` — 아래의 `$export_dir`에 직접
project's own name (not dot-prefixed).
프로젝트 자체 이름(접두사가 붙지 않음).

**Scope, deliberately:**
**의도적으로 범위:**

- Interactive Flow only (`preserve_any == 1`).
- 대화형 흐름만 해당(`preserve_any == 1`).
- Classic Flow
- 클래식 흐름
(`preserve_selection` omitted/declined) is completely unchanged —
(`preserve_selection` 생략/거절)은 완전히 변경되지 않았습니다 —
`<proj_name>.gen/.srcs/.xpr` remain external, untouched siblings of
`<proj_name>.gen/.srcs/.xpr`은 외부의 영향을 받지 않은 형제로 남아 있습니다.
`$export_dir`, exactly as before this change.
`$export_dir`, 이 변경 전과 똑같습니다.
- Additive to, not a replacement for, the existing `.sources/IP|BD`,
- 기존 `.sources/IP|BD`를 대체하는 것이 아니라 추가되는 것입니다.
`.gen/<name>`, `.XPR/<proj>.xpr` per-item copies — `generate_build_script`'s
`.gen/<name>`, `.XPR/<proj>.xpr` 항목별 복사본 — `generate_build_script`
rebuild logic still reads from those, unchanged by this patch.
재구축 논리는 이 패치에서 변경되지 않은 채로 계속해서 읽습니다.
- **Unfiltered:** unlike every other export category in this skill, this copy
- **필터링되지 않음:** 이 기술의 다른 모든 내보내기 범주와 달리 이 사본은
is not limited to RTL/constraints/IP-BD-scripts or even to the
RTL/제약 조건/IP-BD-스크립트 또는 심지어
specifically-preserved items — it's the whole `.gen`/`.srcs` tree,
특별히 보존된 항목 — 전체 `.gen`/`.srcs` 트리입니다.
including every synthesis/impl run and every IP/BD's generated output.
모든 합성/구현 실행과 모든 IP/BD에서 생성된 출력을 포함합니다.
This
이
is an intentional, called-out exception to the "skip generated files, they
"생성된 파일 건너뛰기"에 대한 의도적인 호출 예외입니다.
shouldn't be version-controlled" rule that governs the rest of
버전 관리를 해서는 안 됩니다." 규칙은 나머지 부분을 관리합니다.
`export_all_sources` (see
`export_all_sources`(참조
[export-all-sources/REFERENCE.md](../export-all-sources/REFERENCE.md)).
[내보내기-모든 소스/REFERENCE.md](../내보내기-모든-소스/REFERENCE.md)).
Expect export size and downstream git repo size to scale with the whole
내보내기 크기와 다운스트림 git repo 크기가 전체적으로 확장될 것으로 예상됩니다.
project, not with the curated subset this skill otherwise tracks.
이 기술이 추적하는 선별된 하위 집합이 아닌 프로젝트입니다.

**Not changed:** `generate_build_script` does not consume this new top-level
**변경되지 않음:** `generate_build_script`는 이 새로운 최상위 수준을 사용하지 않습니다.
copy — it still rebuilds via Tcl regeneration / the per-item golden-`.gen`
복사 — 여전히 Tcl 재생성/항목별 Golden-`.gen`을 통해 다시 빌드됩니다.
path.
경로.
Making rebuild actually skip regeneration by reopening the copied
다시 빌드하면 실제로 복사된 항목을 다시 열어 재생성을 건너뜁니다.
`.xpr` directly would be a separate, larger change, out of scope here.
`.xpr`은 여기서 범위를 벗어나는 별도의 더 큰 변경 사항입니다.

**Verified:** `tclsh` source-only smoke test (file parses, no brace/bracket
**검증됨:** `tclsh` 소스 전용 스모크 테스트(파일 구문 분석, 중괄호/괄호 없음)
errors introduced). **Not yet verified against a live Vivado export** — next
오류가 발생했습니다). **라이브 Vivado 내보내기에 대해 아직 검증되지 않았습니다** — 다음
step is running `revision-control-validate` with preservation enabled on an
단계에서 보존이 활성화된 `revision-control-validate`를 실행 중입니다.
IP/BD-bearing example design and confirming `<proj_name>.gen/.srcs/.xpr`
IP/BD 베어링 예시 설계 및 `<proj_name>.gen/.srcs/.xpr` 확인
actually land inside `RevisionControl/` with the expected contents.
실제로 예상된 내용이 있는 `RevisionControl/` 내부에 착륙합니다.

## 2026-09-07

### Fix: preserved BD silently dropped (and its wrapper along with it) when the golden `.gen`/`.sources` path doesn't resolve
### 수정: 골든 `.gen`/`.sources` 경로가 확인되지 않을 때 보존된 BD가 자동으로 삭제되었습니다(및 그 래퍼도 함께).

Team feedback validating "preserve generated outputs" (Step 2.5) on a real
실제 환경에서 "생성된 출력 보존"(2.5단계)을 검증하는 팀 피드백
design: a top-level BD wrapper that came from the original project's `.gen`
디자인: 원본 프로젝트의 `.gen`에서 나온 최상위 BD 래퍼
folder failed to import into the recreated project — both the BD *and* its
폴더를 다시 생성된 프로젝트로 가져오지 못했습니다. BD *및* 해당 폴더 모두
wrapper went missing — reproducing specifically when the exported
래퍼가 누락되었습니다. - 내보낸 파일을 구체적으로 재현하는 중입니다.
`RevisionControl/` tree's internal directory layout was changed after export.
내보낸 후 `RevisionControl/` 트리의 내부 디렉터리 레이아웃이 변경되었습니다.

**Root cause:** a BD selected for preservation skips `write_bd_tcl` entirely
**근본 원인:** 보존을 위해 선택한 BD가 `write_bd_tcl`을 완전히 건너뜁니다.
at export time (by design — only `.gen` output + the `.bd` source are
내보내기 시(설계상 — `.gen` 출력 + `.bd` 소스만
copied, to avoid unnecessary regeneration).
불필요한 재생성을 피하기 위해 복사됨).
At rebuild time,
재구축 시,
`generate_build_script` looks for that `.bd` at a single hardcoded path
`generate_build_script`는 하드코딩된 단일 경로에서 `.bd`를 찾습니다.
(`$base_dir/.sources/BD/<name>.bd`); if it's not there, the code only
(`$base_dir/.sources/BD/<이름>.bd`); 없으면 코드만
printed a `WARNING` and moved on — there was no fallback, because no Tcl
`WARNING`을 인쇄하고 계속 진행했습니다. Tcl이 없기 때문에 대체가 없었습니다.
script had ever been written for that BD.
그 BD를 위해 스크립트가 작성된 적이 있습니다.
With the BD never added to the
BD가 추가되지 않은 상태에서
project, Step 8's wrapper creation (`get_files ${bd_name}.bd`) then also came
프로젝트, 8단계의 래퍼 생성(`get_files ${bd_name}.bd`)도 나왔습니다.
back empty and silently skipped `make_wrapper` too — one root cause, two
다시 비어 있고 `make_wrapper`도 자동으로 건너뛰었습니다. 하나의 근본 원인, 두 개
missing artifacts.
유물이 누락되었습니다.

**Fix (all in `helper_scripts.tcl`):**
**수정(모두 `helper_scripts.tcl`에 있음):**

1. `export_all_sources` — preserved BDs now *also* get a `write_bd_tcl` Tcl
1. `export_all_sources` — 보존된 BD는 이제 *또한* `write_bd_tcl` Tcl을 얻습니다.
script written to `Sources/BD/<name>.tcl`, in addition to the golden
골든 스크립트 외에 `Sources/BD/<name>.tcl`에 작성된 스크립트
`.gen`/`.sources` copy.
`.gen`/`.sources` 사본.
`write_bd_tcl` doesn't resynthesize anything, so
`write_bd_tcl`은 아무것도 재합성하지 않으므로
this is cheap at export time and doesn't defeat the point of
이것은 수출시 저렴하며 요점을 무시하지 않습니다.
preservation — the golden path is still tried first at rebuild time.
보존 — 골든 경로는 여전히 재구축 시 먼저 시도됩니다.
2. `generate_build_script` (dependency-ordered BD sourcing) — if the
2. `generate_build_script`(종속성 순서에 따른 BD 소싱) —
preserved `.sources/BD/<name>.bd` isn't found at rebuild time, it now
보존된 `.sources/BD/<name>.bd`는 재구축 시 발견되지 않습니다. 이제
falls back to `source`-ing the new `Sources/BD/<name>.tcl` instead of
대신 새로운 `Sources/BD/<name>.tcl`을 `source`로 대체합니다.
leaving the BD (and its wrapper) out of the rebuild.
BD(및 해당 래퍼)를 재구축에서 제외합니다.
3. `generate_build_script` (legacy no-dependency-order fallback path) — same
3. `generate_build_script`(레거시 비종속순서 대체 경로) — 동일
fallback added; also had to make the generic Tcl glob-loop in that path
대체가 추가되었습니다. 또한 해당 경로에 일반 Tcl glob-loop를 만들어야 했습니다.
skip preserved BD names first, since they now have a `.tcl` file too and
이제 `.tcl` 파일도 있으므로 보존된 BD 이름을 먼저 건너뛰고
that loop would otherwise force a full regeneration on every rebuild,
그렇지 않으면 해당 루프는 재구축할 때마다 전체 재생성을 강제로 수행하게 됩니다.
defeating preservation even when the golden path would have resolved
황금길이 해결되었을 때에도 보존을 무너뜨리는 것
fine.
괜찮아.

Net effect: golden-`.gen` reuse is still the primary, fast path and still
순 효과: Golden-`.gen` 재사용은 여전히 기본적이고 빠른 경로이며 여전히
skips resynthesis when it resolves; only when it can't resolve does the BD
문제가 해결되면 재합성을 건너뜁니다. 해결할 수 없는 경우에만 BD가 해결됩니다.
fall back to full Tcl regeneration instead of disappearing.
사라지는 대신 전체 Tcl 재생으로 돌아갑니다.

**Verified:** `tclsh` source-only smoke test (file parses cleanly, both
**검증됨:** `tclsh` 소스 전용 연기 테스트(파일이 깔끔하게 구문 분석됨, 둘 다
edited procs still load) — **not yet verified against a live Vivado
편집된 프로세스는 여전히 로드됨) — **아직 라이브 Vivado에 대해 검증되지 않았습니다.
rebuild.** Next step: run `revision-control-validate` against a BD-bearing
재구축.** 다음 단계: BD 베어링에 대해 `revision-control-validate` 실행
example design with preservation enabled, deliberately remove/rename
보존이 활성화된 예제 디자인, 의도적으로 제거/이름 바꾸기
`.sources/BD` after export, and confirm the rebuild now falls back instead
내보낸 후 '.sources/BD'를 선택하고 이제 재구축을 확인하세요.
of dropping the BD.
BD를 버리면 된다.

**Known related gap, not fixed here:** preserved standalone IPs (XCIs) have
**알려진 관련 격차, 여기서 수정되지 않음:** 보존된 독립 실행형 IP(XCI)는
the identical structural issue in the same file (`export_all_sources` around
동일한 파일의 동일한 구조적 문제(`export_all_sources` 주변)
the IP preserve branch, `generate_build_script`'s Step 6 preserved-IP
IP 보존 분기 `generate_build_script`의 6단계 보존 IP
handling) — only the BD path was in scope for this fix.
처리) — BD 경로만 이 수정 사항의 범위에 포함되었습니다.

## 2026-09-02

### Fix: Step 2.5 never actually asked anything in agentic use
### 수정: 2.5단계에서는 에이전트 사용 시 실제로 아무 것도 묻지 않았습니다.

Team feedback running the skill agentically (Claude driving Vivado, not a
스킬을 에이전트 방식으로 실행하는 팀 피드백(Claude가 Vivado를 운전함)
human at a real tty): "no interaction related to O/Ps generation" ever
실제 tty의 인간): "O/P 생성과 관련된 상호 작용이 없습니다"
happened.
일어났다.
Root cause: `prompt_preserve_outputs` only works via `gets stdin`,
근본 원인: `prompt_preserve_outputs`는 `gets stdin`을 통해서만 작동합니다.
which requires a real tty; the docs only covered "human at an interactive
실제 tty가 필요합니다. 문서에서는 "인간의 대화형"만 다루었습니다.
tty" and "automated CI pipeline, skip entirely" and had no guidance for an
tty" 및 "자동화된 CI 파이프라인, 완전히 건너뛰기"에 대한 지침이 없었습니다.
agent that *should* ask the human but can't do it through Tcl's stdin.
인간에게 물어봐야 *하지만 Tcl의 표준 입력을 통해 그것을 할 수 없는 에이전트.
An
agent following the old docs would either skip Step 2.5 as if it were CI, or
이전 문서를 따르는 에이전트는 마치 CI인 것처럼 2.5단계를 건너뛰거나,
call `prompt_preserve_outputs` and get silent EOF -- either way, no prompt
`prompt_preserve_outputs`를 호출하고 자동 EOF를 얻습니다. 어느 쪽이든 프롬프트가 표시되지 않습니다.
ever reached the human.
인간에게 도달한 적이 있다.

**Fix:** added `build_preserve_selection {sel_ips sel_bds}` -- a stdin-free
**수정:** `build_preserve_selection {sel_ips sel_bds}`를 추가했습니다. -- stdin-free
version of `prompt_preserve_outputs` that validates an already-chosen
이미 선택된 것을 검증하는 `prompt_preserve_outputs` 버전
list of names against `list_preservable_outputs`.
`list_preservable_outputs`에 대한 이름 목록입니다.
`SKILL.md` and
`SKILL.md` 및
`helper-procedures/REFERENCE.md` now spell out three cases instead of two:
`helper-procedures/REFERENCE.md`는 이제 두 가지 사례 대신 세 가지 사례를 설명합니다.
human-interactive (`prompt_preserve_outputs`), automated CI (skip, pass
인간 대화형(`prompt_preserve_outputs`), 자동화된 CI(건너뛰기, 통과
`{}`), and agentic (call `list_preservable_outputs`, ask the human directly
`{}`) 및 에이전트(`list_preservable_outputs` 호출, 사람에게 직접 물어보기
in conversation, then call `build_preserve_selection` with the answer).
대화에서 답과 함께 `build_preserve_selection`을 호출하세요).

Purely additive -- `prompt_preserve_outputs`, `list_preservable_outputs`,
순수 추가 -- `prompt_preserve_outputs`, `list_preservable_outputs`,
`export_all_sources`, and `generate_build_script` are unchanged, so both
`export_all_sources` 및 `generate_build_script`는 변경되지 않았으므로 둘 다
pre-existing paths (interactive-tty and automated-CI) behave exactly as
기존 경로(interactive-tty 및 automatic-CI)는 다음과 동일하게 작동합니다.
before.
전에.
Verified with a `tclsh` source-only smoke test (no brace/bracket
`tclsh` 소스 전용 연기 테스트로 확인되었습니다(버팀대/브래킷 없음).
errors introduced).
오류가 발생했습니다).

## 2026-08-23

### New: "preserve generated outputs" (golden-project reuse), Step 2.5
### 신규: "생성된 출력 보존"(골든 프로젝트 재사용), 2.5단계

Meeting-driven feature request: let a user reuse an existing, known-good
회의 기반 기능 요청: 사용자가 기존의 알려진 좋은 기능을 재사용할 수 있도록 합니다.
`.gen` output for a standalone IP or user-created BD across an export/rebuild
내보내기/재구축 시 독립 실행형 IP 또는 사용자 생성 BD에 대한 '.gen' 출력
cycle instead of always regenerating it from Tcl.
항상 Tcl에서 재생성하는 대신 순환합니다.

**What was added:**
**추가된 내용:**

- `list_preservable_outputs` — enumerates eligible standalone IPs (XCIs
- `list_preservable_outputs` — 적격한 독립형 IP(XCI)를 열거합니다.
outside any BD, via `IS_BD_CONTEXT`) and user-created BDs.
BD 외부(`IS_BD_CONTEXT`를 통해) 및 사용자 생성 BD.
- `prompt_preserve_outputs` — new, optional Step 2.5.
- `prompt_preserve_outputs` — 신규, 선택 사항 2.5단계.
- Interactive-only
- 대화형 전용
(blocks on `gets stdin`); lists eligible items and lets the user pick which
(`gets stdin`에 대한 블록); 적합한 항목을 나열하고 사용자가 선택할 수 있도록 합니다.
to preserve, or `all`, or none.
보존하거나 '모두' 또는 없음.
Returns a `{enabled ips bds}` dict.
`{enabled ips bds}` 사전을 반환합니다.
- `export_all_sources` / `generate_build_script` both take an optional
- `export_all_sources` / `generate_build_script` 모두 선택 사항을 취합니다.
`preserve_selection` argument.
`preserve_selection` 인수.
For selected IPs/BDs: `write_ip_tcl` /
선택한 IP/BD의 경우: `write_ip_tcl` /
`write_bd_tcl` are skipped entirely; the existing `.gen` output plus the
`write_bd_tcl`은 완전히 건너뜁니다. 기존 `.gen` 출력과
`.xci`/`.bd` source are copied into new `.gen/` / `.sources/` / `.XPR/`
`.xci`/`.bd` 소스는 새로운 `.gen/` / `.sources/` / `.XPR/`에 복사됩니다.
directories under the export dir.
내보내기 디렉토리 아래의 디렉토리.
These directories are created **only**
이 디렉터리는 **만** 생성됩니다.
when something is actually selected — never on a "No" answer, never
무언가가 실제로 선택되었을 때 — 절대 "아니오"라고 대답하지 마세요.
expected in an older/existing export.
이전/기존 내보내기에서 예상됩니다.
- On rebuild, `generate_build_script` restores the preserved `.gen` into the
- 재구축 시 `generate_build_script`는 보존된 `.gen`을
new project's own `.gen` tree (Step 1c) **before** `add_files`-ing the
새 프로젝트의 자체 `.gen` 트리(1c단계) `add_files` 작업 **전**
preserved source — Vivado's own out-of-date check then sees the IP/BD as
보존된 소스 - Vivado 자체의 오래된 확인을 통해 IP/BD를 다음과 같이 확인합니다.
current and skips real resynthesis/regeneration (standard Vivado
현재이며 실제 재합성/재생을 건너뜁니다(표준 Vivado
incremental behavior, no special flag involved).
증분 동작, 특수 플래그가 포함되지 않음).
If a preserved item's
보존된 물품의 경우
`.gen` is missing at rebuild time, it's logged and that one item falls back
재구축 시 `.gen`이 누락되어 기록되고 한 항목이 대체됩니다.
to normal Tcl regeneration — does not hard-fail the rebuild.
일반 Tcl 재생성 — 재구축이 하드 실패하지 않습니다.

**Not exercised by automation:** `run_pipeline.tcl` (the driver used by
**자동화로 실행되지 않음:** `run_pipeline.tcl`(에서 사용되는 드라이버)
`revision-control-validate`) never calls `prompt_preserve_outputs`, so
`revision-control-validate`)는 `prompt_preserve_outputs`를 호출하지 않으므로
`preserve_selection` is always `{}` in the automated 9-project pass — the
`preserve_selection`은 자동화된 9개 프로젝트 패스에서 항상 `{}`입니다.
pre-existing fully-automated behavior is unchanged.
기존의 완전 자동화된 동작은 변경되지 않습니다.
Verified with a
로 확인됨
`tclsh` source-only smoke test (confirms no brace/bracket syntax errors) and
`tclsh` 소스 전용 연기 테스트(중괄호/괄호 구문 오류가 없는지 확인) 및
by re-running the full 9-project regression, which came back with the same
전체 9개 프로젝트 회귀를 다시 실행하여 동일한 결과가 나왔습니다.
6 PASS / 3 BLOCKED split as the 2026-08-10 baseline.
2026-08-10 기준으로 6 PASS / 3 BLOCKED로 분할됩니다.

### Fix: `.csv` not recognized as a data file
### 수정: `.csv`가 데이터 파일로 인식되지 않습니다.

`export_all_sources`'s Data-file export category (`data_exts`) only matched
`export_all_sources`의 데이터 파일 내보내기 범주(`data_exts`)만 일치합니다.
`.mem .mif .coe .hex .elf`, so `.csv` files (e.g. coefficient/init tables)
`.mem .mif .coe .hex .elf`, 즉 `.csv` 파일(예: 계수/초기화 테이블)
were left out of both the `get_files`-based loop and the fallback
`get_files` 기반 루프와 폴백 모두에서 제외되었습니다.
project-root glob.
프로젝트 루트 글로브.
Added `.csv` to `data_exts` in both places.
두 위치 모두에서 `data_exts`에 `.csv`를 추가했습니다.

**Not verified against a live project:** none of the 9 example designs
**라이브 프로젝트에 대해 확인되지 않음:** 9가지 예제 디자인 중 없음
contain a `.csv` file, so this is confirmed by code inspection only, not by
`.csv` 파일이 포함되어 있으므로 이는 코드 검사로만 확인되며
an export run that actually picks one up (same limitation noted for the
실제로 하나를 선택하는 내보내기 실행(
other data extensions in the 2026-08-10 pass, below).
2026-08-10 패스의 기타 데이터 확장은 아래 참조).

## Versal detection in `detect_project_flow` never fired
## `Detect_project_flow`의 Versa 감지가 실행되지 않았습니다.

`$device` holds the actual part string (e.g.
`$device`는 실제 부분 문자열을 보유합니다(예:
`xcvc1902-vsva2197-2MP-e-S`), not
`xcvc1902-vsva2197-2MP-e-S`), 아님
a family/board name.
가족/보드 이름.
The existing `string match` checks (`*versal*`, `vek*`,
기존 `문자열 일치`는 (`*versal*`, `vek*`,
`vck*`, `vmk*`, `vhk*`) only ever match board names, so `is_versal` was always
`vck*`, `vmk*`, `vhk*`)는 보드 이름만 일치하므로 `is_versal`은 항상
`0` — segmented-config handling for Versal designs was silently skipped.
'0' — Versal 설계에 대한 분할 구성 처리가 자동으로 건너뛰었습니다.

**Fix:** added part-number prefix matches — `xcv*` for Gen1 (`xcvc`/`xcve`/
**수정:** 부품 번호 접두사 일치 추가 — Gen1의 경우 `xcv*`(`xcvc`/`xcve`/
`xcvp`/`xcvm`) and `xc2v*` for Gen2 (`xc2ve`/`xc2vp`/`xc2vm`/`xc2vc`, e.g.
Gen2의 경우 `xcvp`/`xcvm`) 및 `xc2v*`(`xc2ve`/`xc2vp`/`xc2vm`/`xc2vc`, 예:
`xc2ve3858`) — alongside the original board-name patterns, which are kept as
`xc2ve3858`) — 원래 보드 이름 패턴과 함께 유지됩니다.
a fallback in case a board string is ever passed instead of a part.
부품 대신 보드 문자열이 전달되는 경우의 대체입니다.

## Missing RM source files failed silently in `export_all_sources`
## 누락된 RM 소스 파일이 `export_all_sources`에서 자동으로 실패했습니다.

When a reconfig module's registered file path didn't resolve to a real file
재구성 모듈의 등록된 파일 경로가 실제 파일로 확인되지 않는 경우
on disk, the DFX export loop just `continue`d past it with no message.
디스크에서는 DFX 내보내기 루프가 메시지 없이 '계속' 지나갔습니다.
The
는
gap only surfaced later as a confusing rebuild-time error (e.g. a
격차는 나중에 혼란스러운 재구축 시간 오류로 표면화되었습니다(예:
partition/RM module-name mismatch) with no link back to the real cause.
파티션/RM 모듈 이름 불일치) 실제 원인에 대한 링크가 없습니다.

**Fix:** the loop now prints `WARNING: RM '<name>' file not found at export
**수정:** 이제 루프가 `WARNING: RM '<name>' 파일을 내보낼 때 찾을 수 없음을 인쇄합니다.
time, skipping: <path>` before skipping, so the stale/broken reference is
time, skipping: <path>` 건너뛰기 전에, 따라서 오래되거나 손상된 참조는
diagnosable at export time instead of at rebuild time.
재구축 시가 아닌 내보내기 시 진단 가능합니다.

## DFX reconfig-module rebuild failures (`rtl_dfx`, `modnoc-rtl-dfx`)
## DFX 재구성 모듈 재구축 실패(`rtl_dfx`, `modnoc-rtl-dfx`)

`generate_build_script`'s emitted `dfx_define_partition` proc had two
`generate_build_script`가 내보낸 `dfx_define_partition` proc에는 두 개가 있습니다.
independent bugs, both verified live against Vivado 2026.2 (not inferred from
독립적인 버그, 둘 다 Vivado 2026.2에 대해 라이브로 검증되었습니다(에서 추론되지 않음).
docs):
문서):

1. **`create_reconfig_module` never passed `-top`.** Without it, Vivado can't
1. **`create_reconfig_module`은 `-top`을 통과하지 못했습니다.** 이것이 없으면 Vivado는 할 수 없습니다
associate the RM with its partition def's module name — this surfaced as
RM을 해당 파티션 정의의 모듈 이름과 연결합니다. 이는 다음과 같이 표시됩니다.
`modnoc-rtl-dfx`'s confusing `partition definition 'RP1' module name should
`modnoc-rtl-dfx`의 혼란스러운 `파티션 정의 'RP1' 모듈 이름은 다음과 같습니다.
match with 'top' name of any one of its reconfigurable module` error.
재구성 가능한 모듈 오류 중 하나의 '상위' 이름과 일치합니다.
Fixed by passing `-top $rm_module` (the RM's own top module name, already
`-top $rm_module`(RM 자체의 최상위 모듈 이름은 이미
captured during export via `MODULE_NAME`, just never threaded through).
`MODULE_NAME`을 통해 내보내는 동안 캡처되었으며 스레드된 적이 없음)

2. **RM file attachment used a command/argument combo that doesn't work.**
2. **RM 파일 첨부에서 작동하지 않는 명령/인수 콤보를 사용했습니다.**
`import_files -of_objects [get_reconfig_modules ...]` silently no-ops for
`import_files -of_objects [get_reconfig_modules ...]`에 대한 자동 no-ops
freshly-copied IP (`.xci`) sources, and the existing `.xci` fallback
새로 복사된 IP(`.xci`) 소스 및 기존 `.xci` 대체
(`get_filesets -of_objects` → `add_files -fileset`) never actually attaches
(`get_filesets -of_objects` → `add_files -fileset`)은 실제로 연결되지 않습니다.
anything either — `get_filesets -of_objects <reconfig_module>` returns a
무엇이든 — `get_filesets -of_objects <reconfig_module>`은
value that *looks* like a fileset name but isn't one `-fileset` can resolve
파일 세트 이름처럼 *보이지만* 해결할 수 있는 `-fileset`이 아닌 값
by name.
이름으로.
Trying `add_files -fileset [get_reconfig_modules ...]` directly
`add_files -fileset [get_reconfig_modules ...]`를 직접 시도
errors outright with `Adding files to a fileset created by RM is not
'RM에서 생성된 파일 세트에 파일을 추가하는 것은 불가능합니다'라는 오류가 발생합니다.
supported.
지원됩니다.
Please use 'add_files <> -of_objects [get_reconfig_modules
'add_files <> -of_objects [get_reconfig_modules'를 사용하세요.
   <>]'` — Vivado's own error message names the fix. Replaced both the
primary attempt and the broken fallback with a single
한 번의 시도로 기본 시도와 깨진 폴백
`add_files -norecurse -of_objects [get_reconfig_modules $rm_name] $file`,
`add_files -norecurse -of_objects [get_reconfig_modules $rm_name] $file`,
confirmed via a from-scratch repro to actually attach the file.
실제로 파일을 첨부하기 위해 처음부터 다시 작성하여 확인했습니다.

3. **Step 6 (`Recreate IP Cores`) didn't exclude RM-owned IPs.** Every IP's
3. **6단계('IP 코어 재생성')에서는 RM 소유 IP를 제외하지 않았습니다.** 모든 IP는
`write_ip_tcl` script was exported unconditionally, so Step 6 recreated
`write_ip_tcl` 스크립트를 무조건 내보냈으므로 6단계가 다시 생성되었습니다.
RM-owned IPs (e.g.
RM 소유 IP(예:
`axi_traffic_gen_1`) at the project/`sources_1` scope
`axi_traffic_gen_1`) 프로젝트/`sources_1` 범위에서
*before* Step 12 tried to attach the same IP's raw `.xci` copy to its RM —
*이전* 12단계에서는 동일한 IP의 원시 `.xci` 복사본을 해당 RM에 연결하려고 시도했습니다.
causing `IP name 'axi_traffic_gen_1' is already in use in this project`.
'IP 이름 'axi_traffic_gen_1'이 이 프로젝트에서 이미 사용 중입니다.'
Fixed by skipping IP export for any IP whose `.xci` tail is already in
`.xci` 꼬리가 이미 있는 IP에 대해 IP 내보내기를 건너뛰는 방식으로 수정되었습니다.
`dfx_rm_file_tails` (the same exclusion list Step 2's RTL import and Step
`dfx_rm_file_tails`(동일한 제외 목록 2단계의 RTL 가져오기 및 단계
7's BD sourcing already use), mirroring the existing BD-context skip.
7의 BD 소싱은 이미 사용 중), 기존 BD 컨텍스트 건너뛰기를 미러링합니다.

**Residual, NOT fixed by the above (pre-existing bugs in the example designs,
**위의 방법으로 수정되지 않은 잔여 문제(예제 디자인에 기존 버그가 있음)
not the skill):**
스킬 아님):**

- `rtl_dfx`'s RM OOC synthesis (`rp1_rm1_synth_1` etc.) fails with
- `rtl_dfx`의 RM OOC 합성(`rp1_rm1_synth_1` 등)이 실패합니다.
`[Synth 8-439] module 'axi_traffic_gen_N' not found` **even on the
`[Synth 8-439] 모듈 'axi_traffic_gen_N'을 찾을 수 없습니다` **심지어
pristine, never-exported original project** — confirmed by running
내보내지 않은 원본 프로젝트** — 실행하여 확인
`launch_runs synth_1` directly against a scratch copy of the untouched
손대지 않은 스크래치 복사본에 대해 직접 `launch_runs synth_1`
`.xpr`.
`.xpr`.
This is an inherent defect in the reference design itself; no
이는 참조 설계 자체에 내재된 결함입니다. 아니
export/rebuild script change can fix it.
내보내기/재구축 스크립트 변경으로 문제를 해결할 수 있습니다.
- `modnoc-rtl-dfx`'s `RP1_rm1`/`RP1_rm2` RMs reference their own top-level
- `modnoc-rtl-dfx`의 `RP1_rm1`/`RP1_rm2` RM은 자체 최상위 수준을 참조합니다.
RTL (`RP1_rm1.v` etc.) via a stale absolute path baked into the original
원본에 구운 오래된 절대 경로를 통한 RTL(`RP1_rm1.v` 등)
project (missing the `modnoc-rtl-dfx/project_mode.srcs/RP1_rm1/imports/...`
프로젝트(`modnoc-rtl-dfx/project_mode.srcs/RP1_rm1/imports/...'가 누락됨)
path segment).
경로 세그먼트).
Already correctly surfaced by the "Missing RM source files"
"RM 소스 파일 누락"으로 이미 올바르게 표시되었습니다.
fix above (`WARNING: RM '<name>' file not found at export time`), but not
위의 수정 사항(`경고: RM '<name>' 파일을 내보낼 때 찾을 수 없음`), 그러나 그렇지 않음
repairable from the skill side — the real file just isn't at the path the
기술 측면에서 복구 가능 - 실제 파일은 경로에 있지 않습니다.
project's `.xpr` claims.
프로젝트의 `.xpr` 주장.

Both projects will still show `REBUILD_RESULT=BLOCKED` after these fixes —
이러한 수정 후에도 두 프로젝트 모두 `REBUILD_RESULT=BLOCKED`로 표시됩니다.
for a correctly diagnosed reason instead of a confusing one, not because the
혼란스러운 이유가 아닌 정확하게 진단된 이유 때문에
fixes are incomplete.
수정이 불완전합니다.
