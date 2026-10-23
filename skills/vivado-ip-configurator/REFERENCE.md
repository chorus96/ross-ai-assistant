<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# vivado-ip-configurator — REFERENCE (read on demand)
# vivado-ip-configurator — 참조(요청 시 읽기)

Background detail for the vivado-ip-configurator skill: measurements, per-IP notes and the
vivado-ip-configurator 기술에 대한 배경 세부 정보: 측정, IP별 메모 및
long-form workflow prose.
긴 형식의 작업 흐름 산문.
`SKILL.md` is the entry point and `ipcfg::api` in
`SKILL.md`는 진입점이고 `ipcfg::api`는
`lib/ipcfg.tcl` is the authoritative protocol — consult this file only for detail
`lib/ipcfg.tcl`은 신뢰할 수 있는 프로토콜입니다. 자세한 내용은 이 파일을 참조하세요.
those two do not carry.
그 두 사람은 가지고 다니지 않습니다.

Where this file and `ipcfg::api` disagree, `ipcfg::api` wins: it ships with the code.
이 파일과 `ipcfg::api`가 일치하지 않는 경우 `ipcfg::api`가 승리합니다. 코드와 함께 제공됩니다.

# IP Configurator — Doc-Driven Configuration with Error Recovery
# IP 구성자 — 오류 복구 기능을 갖춘 문서 기반 구성

## Overview
## 개요

This skill configures Vivado IP blocks from natural-language prompts using a two-tier approach
이 기술은 2계층 접근 방식을 사용하여 자연어 프롬프트에서 Vivado IP 블록을 구성합니다.
that treats Vivado as the source of truth.
Vivado를 진실의 근원으로 취급합니다.
It does NOT rely on pre-built parameter databases
사전 구축된 매개변수 데이터베이스에 의존하지 않습니다.
or exhaustive JSON dumps of IP configuration spaces.
또는 IP 구성 공간의 철저한 JSON 덤프.

**Tier 1 — Doc-Driven First Attempt:**
**계층 1 - 문서 기반 첫 번째 시도:**
1. Search AMD documentation for the IP's configuration parameters
1. IP의 구성 매개변수에 대해서는 AMD 문서를 검색하세요.
2. Build a `set_property -dict` from the docs
2. 문서에서 `set_property -dict`를 빌드하세요.
3. Execute it via `vivado_execute`
3. `vivado_execute`를 통해 실행하세요.

**Tier 2 — Error-Driven Recovery (only if Tier 1 fails):**
**계층 2 - 오류 기반 복구(계층 1이 실패한 경우에만):**
1. Parse the error message from the failed `set_property -dict`
1. 실패한 `set_property -dict`의 오류 메시지를 구문 분석합니다.
2. Search docs for the specific parameters/constraints mentioned in the error
2. 오류에 언급된 특정 매개변수/제약조건에 대한 문서를 검색하세요.
3. Build a corrected `-dict` and retry
3. 수정된 `-dict`를 빌드하고 다시 시도하세요.

Key principle: `set_property -dict` is **atomic for hard errors** — if it throws, no
주요 원칙: `set_property -dict`는 **하드 오류에 대해 원자적**입니다. 오류가 발생하면 아니요
parameters are changed, and the error messages are rich and specific.
매개변수가 변경되고 오류 메시지가 풍부하고 구체적입니다.

**Caveat (critical):** In IP Integrator (BD) mode, an *unknown* CONFIG parameter does
**주의사항(중요):** BD(IP Integrator) 모드에서는 *알 수 없는* CONFIG 매개변수가
**not** throw.
**던지지 마세요**.
Vivado emits a non-fatal `CRITICAL WARNING [BD 41-1276] ... does not exist`
Vivado는 치명적이지 않은 'CRITICAL WARNING [BD 41-1276] ... 존재하지 않습니다'를 표시합니다.
(or `[IP_Flow 19-7090] Invalid parameter ...
(또는 `[IP_Flow 19-7090] 잘못된 매개변수...
Ignoring`), `catch` returns 0, and the script
Ignoring`), `catch`는 0을 반환하고 스크립트는
would falsely report success.
성공을 거짓으로 보고할 것입니다.
Therefore Tier 1 must **verify** that the parameters actually
따라서 Tier 1에서는 매개변수가 실제로
took effect (see Step 3, Phase 3 verification) and the agent must **scan the execute output**
적용되었으며(3단계, 3단계 확인 참조) 에이전트는 **실행 출력을 스캔**해야 합니다.
for these warning patterns before declaring SUCCESS — never trust the `SUCCESS:` line alone.
SUCCESS를 선언하기 전에 이러한 경고 패턴에 대해 — 절대 'SUCCESS:' 줄만 신뢰하지 마세요.

## Operating Modes
## 작동 모드

This skill runs in one of two modes.
이 스킬은 두 가지 모드 중 하나로 실행됩니다.
Choose the mode at the start of the session and keep it.
세션 시작 시 모드를 선택하고 유지하세요.

| Mode<br>모드 | When<br>언제 | Behavior<br>행동 |
|---|---|---|
| `unattended` (default)<br>'무인'(기본값) | Nobody is available to answer a question — a scripted, CI, or otherwise automated run<br>스크립트, CI 또는 기타 자동화된 실행 등 질문에 답할 수 있는 사람이 없습니다. | Deterministic and **silent**: never ask; make the documented best choice and record the assumption in the Step 1.5 ledger. Minimize MCP calls.<br>결정적이고 **자동**: 절대 묻지 마세요. 최선의 선택을 문서화하고 1.5단계 원장에 가정을 기록합니다. MCP 호출을 최소화합니다. |
| `interactive`<br>'상호작용' | A human is in the loop, turn by turn, building a real design<br>인간이 루프에 들어가서 차례대로 실제 디자인을 구축합니다. | May **ask clickable clarifying questions** when doc search is inconclusive, and proactively uses Block Automation / Configurable Example Designs to flesh out the design.<br>문서 검색이 결론을 내리지 못한 경우 **클릭 가능한 명확한 질문**을 하고 블록 자동화/구성 가능한 예제 디자인을 적극적으로 사용하여 디자인을 구체화할 수 있습니다. |

If the mode is not stated, assume `unattended`: a question nobody answers stalls the run
모드가 명시되지 않은 경우 '무인'으로 가정합니다. 질문에 아무도 답변하지 않으면 실행이 중단됩니다.
until it times out, whereas a documented best choice recorded in the ledger is always
시간이 초과될 때까지 원장에 기록된 문서화된 최선의 선택은 항상
recoverable.
복구 가능.
Switch to `interactive` only when the user has actually been conversing with
사용자가 실제로 대화를 나눈 경우에만 '대화형'으로 전환하세요.
you turn by turn.
당신은 차례대로 돌아갑니다.
Steps below are tagged **(both)**, **(interactive)**, or **(unattended)**
아래 단계에는 **(둘 다)**, **(대화형)** 또는 **(무인)** 태그가 지정되어 있습니다.
where the behavior differs.
행동이 다른 곳.
Everything not tagged applies to both.
태그가 지정되지 않은 모든 항목은 둘 다에 적용됩니다.

## Always report coverage (REQUIRED, both modes)
## 항상 적용 범위 보고(필수, 두 모드 모두)

You MUST tell the user, in plain language, whether the IP was **fully parameterized** from
IP가 **완전히 매개변수화**되었는지 여부를 일반 언어로 사용자에게 알려야 합니다.
the prompt — and if not, **exactly which part(s) of the prompt could not be applied, and why**.
프롬프트 — 그렇지 않은 경우 **정확히 프롬프트의 어떤 부분을 적용할 수 없었으며 그 이유**.
Never silently grade `partial` / `creation_only` / `negative`.
`partial` / `creation_only` / `negative`를 자동으로 평가하지 마세요.

Drive this from the Step 1.5 requirement ledger.
1.5단계 요구 사항 원장에서 이를 구동합니다.
Every atomic requirement ends in one outcome:
모든 원자적 요구 사항은 하나의 결과로 끝납니다.
- `applied` — set and verified (stuck + intent realized)
- `applied` — 설정 및 확인됨(멈춤 + 의도 실현)
- `default` — already correct by default (`value_src=default`, acceptable)
- `default` — 기본적으로 이미 정확함(`value_src=default`, 허용됨)
- `unapplied:<reason>` — could not be parameterized; `<reason>` is one of
- `unapplied:<reason>` — 매개변수화할 수 없습니다. `<이유>`는 다음 중 하나입니다.
`runtime-register-only`, `integration-derived`, `gated-no-enabler`, `not-a-config-param`,
`런타임 등록 전용`, `통합 파생`, `gated-no-enabler`, `not-a-config-param`,
`value-out-of-range`, `wrong-ip-for-capability`.
'값이 범위를 벗어났습니다', '기능에 대한 IP가 잘못되었습니다'.

Emit the coverage report whenever the result is not fully applied:
결과가 완전히 적용되지 않을 때마다 적용 범위 보고서를 내보냅니다.

```tcl
puts [ipcfg::coverage_report $ledger]   ;# COVERAGE:FULL | COVERAGE:PARTIAL + the unapplied list
```

Then say to the user, verbatim about the gaps: "I could **not** fully parameterize `<IP>` from
그런 다음 사용자에게 공백에 대해 다음과 같이 말합니다. "'<IP>'를 완전히 매개변수화할 수 **없습니다**.
the prompt.
프롬프트.
Applied: <list>.
적용됨: <목록>.
Could NOT apply: '<the exact prompt phrase>' because <reason>."
적용할 수 없습니다: '<정확한 프롬프트 문구>' 왜냐하면 <이유> 때문입니다."
Quote the actual prompt fragment for each gap — do not paraphrase it away.
각 공백에 대해 실제 프롬프트 부분을 인용하세요. 다른 말로 바꾸어 표현하지 마세요.

## MCP Tools Used
## 사용된 MCP 도구

Only THREE tools are needed:
세 가지 도구만 필요합니다:

| Tool<br>도구 | Purpose<br>목적 | When<br>언제 |
|------|---------|------|
| `vivado_doc_search`<br>`vivado_doc_search` | Find IP parameters, valid values, dependencies<br>IP 매개변수, 유효한 값, 종속성 찾기 | Tier 1 (always), Tier 2 (on error)<br>Tier 1(항상), Tier 2(오류 시) |
| `vivado_execute`<br>'vivado_execute' | Run `set_property -dict` and read-back commands<br>`set_property -dict` 및 다시 읽기 명령 실행 | Both tiers<br>두 계층 모두 |
| `vivado_log_messages`<br>`vivado_log_messages` | Additional diagnostics if error string is ambiguous<br>오류 문자열이 모호한 경우 추가 진단 | Tier 2 (rare)<br>2등급(희귀) |

## Workflow
## 작업 흐름

### Step 0: Part resolution (guarded — before anything else)
### 0단계: 부분 해결(보호 - 무엇보다 먼저)

Many IPs only exist on specific device families. **A part swap is destructive to a
많은 IP는 특정 장치 제품군에만 존재합니다. **부품 교환은 제품에 파괴적입니다.
design-in-progress**: existing IP may be incompatible with the new part, will need
설계 진행 중**: 기존 IP가 새 부품과 호환되지 않을 수 있습니다.
upgrading, and may then mismatch the original prompts.
업그레이드하면 원래 프롬프트와 일치하지 않을 수 있습니다.
So treat swapping as **opt-in and
따라서 교환을 **선택으로 처리하고
last-resort** *while other cells exist*.
최후의 수단** *다른 셀이 존재하는 동안*.
On an empty or isolated BD — a single-IP task — a
비어 있거나 격리된 BD에서 - 단일 IP 작업 -
swap is cheap, guard-approved and **expected**; "last resort" is never a licence to build the
스왑은 저렴하고 경비 승인을 받았으며 **예상**됩니다. "최후의 수단"은 결코
wrong IP instead:
대신 잘못된 IP:

1. **Check availability on the current part first** — `ipcfg::ip_availability {<candidates>}`
1. **현재 부분의 가용성을 먼저 확인하세요** — `ipcfg::ip_availability {<후보>}`
(**mandatory before any create**, see Step 0a′).
(**생성 전 필수**, 0a단계 참조).
If the IP is `AVAILABLE`, do **not** swap.
IP가 `AVAILABLE`인 경우 교환하지 **하지 마세요**.
2. **Only if `WRONG_PART`**, consider a swap.
2. **`WRONG_PART`**인 경우에만 교체를 고려하세요.
2. `ipcfg::ensure_part <part>` is **guarded**: if
2. `ipcfg::ensure_part <part>`는 **보호됩니다**: if
other `bd_cells` already exist it returns `WARN:SWAP_BLOCKED:...` instead of swapping.
다른 `bd_cells`이 이미 존재하면 교체 대신 `WARN:SWAP_BLOCKED:...`를 반환합니다.
   - **(interactive)** On `WARN:SWAP_BLOCKED`, **ask the user** (clickable) whether to swap,
   - **(대화형)** `WARN:SWAP_BLOCKED`에서 **사용자에게** (클릭 가능) 교체 여부를 묻고,
warning that existing IP may need upgrading and could mismatch their intent.
기존 IP를 업그레이드해야 할 수도 있고 의도와 일치하지 않을 수도 있음을 경고합니다.
Only call
통화만
`ipcfg::ensure_part <part> 1` (force) after they confirm.
`ipcfg::ensure_part <part> 1`(강제) 확인 후.
   - **(unattended)** When the BD holds no other cells the guard passes on its own and the
   - **(무인)** BD가 다른 셀을 보유하지 않은 경우 가드는 자체적으로 통과하고
swap proceeds; restore the part in cleanup.
스왑 수익; 부품을 정리하여 복원합니다.
Never force past `WARN:SWAP_BLOCKED` with
절대 `WARN:SWAP_BLOCKED`를 강제로 지나치지 마세요.
no human to confirm — report it as `unapplied:wrong-ip-for-capability` instead.
확인할 사람이 없습니다. 대신 `unapplied:wrong-ip-for-capability`로 보고하세요.
3. Never swap silently mid-design.
3. 설계 도중에 조용히 교체하지 마십시오.

**Step 0a — Doc search for device family (combined with parameter search):**
**0a단계 — 장치 제품군에 대한 문서 검색(매개변수 검색과 결합):**

Include device/family keywords in the Tier 1 doc search so a single call covers both
단일 통화로 두 가지 모두를 다룰 수 있도록 Tier 1 문서 검색에 장치/제품군 키워드를 포함합니다.
part compatibility and CONFIG parameters:
부품 호환성 및 CONFIG 매개변수:

```
vivado_doc_search("<IP name> supported devices configuration parameters")
```

From the results, extract:
결과에서 다음을 추출합니다.
- Which device families support this IP (Zynq-7000, Zynq US+, UltraScale+, Versal, etc.)
- 이 IP를 지원하는 장치 제품군(Zynq-7000, Zynq US+, UltraScale+, Versal 등)
- The CONFIG.* property names (used in Step 3)
- CONFIG.* 속성 이름(3단계에서 사용됨)

**Step 0a′ — MANDATORY availability gate (before any `create_bd_cell`):**
**0a단계' — 필수 가용성 게이트(`create_bd_cell` 전):**

Run the **candidate shortlist** — every IP you are choosing between, not the one you already
**후보 후보 목록** 실행 - 이미 IP가 아닌 선택 중인 모든 IP
favour — through the gate.
호의 - 문을 통해.
Build the shortlist from the prompt's **function words** as a catalog
프롬프트의 **기능 단어**에서 후보 목록을 카탈로그로 작성합니다.
**glob**, which the proc expands against this release, so the shortlist is discovered rather than
**glob**, 이 릴리스에 대해 프로세스가 확장되므로 최종 후보 목록이 검색됩니다.
pre-committed to the first name that came to mind:
마음에 떠오른 이름에 미리 커밋되었습니다.

```tcl
puts [ipcfg::ip_availability {*clk*wiz*}]     ;# prompt says "clocking wizard"
# IPAVAIL: part=<Versal Gen2 xc2ve part> \
#   clk_wizard=WRONG_PART:xilinx.com:ip:clk_wizard:1.0:parts={xcvc1502-... xcvc1902-viva1596-2MP-e-S ...}:devices=63 \
#   clkx5_wiz=AVAILABLE:xilinx.com:ip:clkx5_wiz:1.0
```

**Passing ONE pre-chosen name is the misuse this gate exists to prevent** — it can only confirm
**미리 선택된 이름 하나를 전달하는 것은 이 게이트가 방지하기 위해 존재하는 오용입니다** — 확인만 가능합니다
a name you already picked, so an IP that is `WRONG_PART` never gets compared against the one that
이미 선택한 이름이므로 `WRONG_PART`인 IP는 결코 선택한 IP와 비교되지 않습니다.
happens to work.
일이 발생합니다.
The proc says so (`SHORTLIST_OF_1:validator-only-call:...`); when you see that,
프로세스에서는 그렇게 말합니다(`SHORTLIST_OF_1:validator-only-call:...`). 당신이 그것을 볼 때,
re-run with a glob before creating anything.
아무것도 생성하기 전에 glob으로 다시 실행하세요.
A shortlist is not "the name plus a rationale"; it
후보 목록은 "이름과 근거"가 아닙니다. 그것
is every catalog IP whose name matches the function you were asked for.
요청한 기능과 이름이 일치하는 모든 카탈로그 IP입니다.

This is **not** optional and **not** replaceable by `ipcfg::vlnv_ok`.
이는 선택 사항이 아니며 `ipcfg::vlnv_ok`로 대체할 수 **없습니다**.
`get_ipdefs` (and therefore
`get_ipdefs`(따라서
`vlnv_ok`) lists IPs the current part **cannot instantiate**: on 2026.1 with a Versal Gen2 (`xc2ve*`) part, `vlnv_ok
`vlnv_ok`)는 현재 부분이 **인스턴스화할 수 없음**: Versal Gen2(`xc2ve*`) 부분인 `vlnv_ok가 있는 2026.1의 IP를 나열합니다.
xilinx.com:ip:clk_wizard` returns **1** and `create_bd_cell` then fails with
xilinx.com:ip:clk_wizard`는 **1**을 반환하고 `create_bd_cell`은 실패합니다.
`[BD 5-683] ... not supported for the current part`.
`[BD 5-683] ... 현재 부품에는 지원되지 않습니다`.
Worse, that failure is **not recoverable in
더 나쁜 것은 그 실패가 **복구 불가능하다는 것입니다.
Tier 2**: `catch` sees only `[Common 17-39] 'create_bd_cell' failed due to earlier errors`, and
계층 2**: `catch`에는 `[공통 17-39] 'create_bd_cell'이 이전 오류로 인해 실패했습니다'만 표시됩니다.
the `[BD 5-683]` naming the cause reaches only the log.
원인을 명명하는 `[BD 5-683]`은 로그에만 도달합니다.
Predict it here; do not discover it there.
여기서 예측해 보세요. 거기에서 그것을 발견하지 마십시오.

> **FIRST, check for `VARIANT_OF:` — one IP split across device generations.**
> **먼저 `VARIANT_OF:`를 확인하세요. — 하나의 IP가 기기 세대에 걸쳐 분할되어 있습니다.**
> A release can ship the *same* IP twice under different catalog names, each covering a different
> 릴리스는 *동일한* IP를 서로 다른 카탈로그 이름으로 두 번 제공할 수 있으며, 각각은 서로 다른 내용을 포함합니다.
> device generation, with **identical `DISPLAY_NAME`** and disjoint `SUPPORTED_PARTS`.
> **동일한 `DISPLAY_NAME`** 및 분리된 `SUPPORTED_PARTS`를 사용하여 장치 생성.
> 2026.1 ships
> 2026.1 선박
> "Clocking Wizard" three times: `clk_wiz` (7-series/UltraScale), `clk_wizard` (Versal Gen1 —
> "클러킹 마법사" 3회: `clk_wiz`(7 시리즈/UltraScale), `clk_wizard`(Versal Gen1 —
> **0** `xc2ve*` parts) and `clkx5_wiz` (Versal Gen2 — **375** `xc2ve*` parts and none of Gen1's).
> **0** `xc2ve*` 부분) 및 `clkx5_wiz`(Versal Gen2 — **375** `xc2ve*` 부분 및 Gen1 부분 없음).
> When the gate reports `VARIANT_OF:{a~b}`, the `AVAILABLE` one **is** the documented IP for this
> 게이트가 'VARIANT_OF:{a~b}'를 보고할 때 'AVAILABLE' 항목은 **이** 이에 대해 문서화된 IP입니다.
> device: **use it and do not swap.** Swapping would move to an older device generation just to
> 장치: **교체하지 말고 사용하세요.** 교환하면 이전 장치 세대로 이동하게 됩니다.
> reach an older edition of the same core.
> 동일한 코어의 이전 버전에 도달합니다.
> From the outside this pair looks identical to the
> 외부에서 보면 이 쌍은 다음과 동일해 보입니다.
> wrong-IP substitution the next rule forbids, which is why the gate distinguishes them for you
> 잘못된 IP 대체는 다음 규칙에서 금지되므로 게이트가 이를 구별합니다.
> instead of leaving it to judgement.
> 판단에 맡기는 대신.
>
> **THE SELECTION RULE — for a `SELECT_RULE:` verdict (different IPs, not variants).**
> **선택 규칙 — `SELECT_RULE:` 결과(변형이 아닌 다른 IP)에 대한 것입니다.**
> **Choose the IP by documented function FIRST** (PG321 "Versal Clocking Wizard" → `clk_wizard`).
> **문서화된 기능 FIRST로 IP를 선택하세요**(PG321 "Versal Clocking Wizard" → `clk_wizard`).
> A `WRONG_PART` verdict is a reason to consider a **guarded part swap**.
> 'WRONG_PART' 결과는 **보호된 부품 교환**을 고려하는 이유입니다.
> It is **NEVER** a reason
> **결코** 이유가 되지 않습니다
> to substitute a *different* IP that happens to be `AVAILABLE` on the current part.
> 현재 부분에서 'AVAILABLE'인 *다른* IP를 대체합니다.
> Concretely: on a Versal Gen2 (`xc2ve*`) part, `clk_wizard` is `WRONG_PART` while the similarly-named `clkx5_wiz`
> 구체적으로: Versal Gen2(`xc2ve*`) 부분에서 `clk_wizard`는 `WRONG_PART`인 반면 비슷한 이름의 `clkx5_wiz`는
> is `AVAILABLE`.
> '사용 가능'입니다.
> Picking `clkx5_wiz` because it creates without complaint is the **wrong IP**,
> 불만 없이 생성되기 때문에 `clkx5_wiz`를 선택하는 것은 **잘못된 IP**입니다.
> silently — the design builds, configures and verifies clean, so nothing ever errors.
> 조용히 - 디자인이 깔끔하게 구축, 구성 및 검증되므로 오류가 발생하지 않습니다.
> Swap the
> 교환하다
> part instead.
> 대신 부분.
>
> When the gate reports a `WRONG_PART` **and** an `AVAILABLE` candidate together it says
> 게이트가 `WRONG_PART` **및** `AVAILABLE` 후보를 함께 보고하면 다음과 같이 표시됩니다.
> `SELECT_RULE:...`, because that pair is the one case where a correct shortlist still leads to the
> `SELECT_RULE:...`, 해당 쌍은 올바른 후보 목록이 여전히 다음으로 이어지는 유일한 사례이기 때문입니다.
> wrong IP.
> IP가 잘못되었습니다.
> On that signal, the choice is **not yours to make on availability**: resolve it from
> 해당 신호에서 선택은 **가용성에 따라 결정할 수 있는 것이 아닙니다**: 다음에서 해결하세요.
> documentation (`vivado_doc_search "<function> supported devices"`, cite the PG) and pick the IP
> 문서(`vivado_doc_search "<function> 지원 장치"`, PG 인용) 및 IP 선택
> the doc names for the requested function. "The other one was already available" is never a
> 요청된 기능에 대한 문서 이름입니다. "다른 하나는 이미 사용 가능했습니다"는 결코
> reason, and neither is name similarity to the prompt's wording.
> 이유도 있고 이름도 프롬프트의 표현과 유사하지 않습니다.
>
> **A `WRONG_PART` IP is not "unavailable to you" — it is one part swap away, and that swap is the
> **`WRONG_PART` IP는 "사용할 수 없음"이 아닙니다. 이는 한 부분만 교체하는 것이며 해당 교체는
> sanctioned path here, not a last resort.** Do not conclude "the documented IP is `clk_wizard`,
> 최후의 수단이 아닌 여기에 승인된 경로가 있습니다.** "문서화된 IP는 `clk_wizard`입니다."라고 결론을 내리지 마십시오.
> but it is not supported on this part, therefore I must use the available one." That reasoning
> 하지만 이 부분은 지원하지 않기 때문에 가능한 것을 사용해야 합니다." 그 추론
> ends in the wrong IP every time.
> 매번 잘못된 IP로 끝납니다.
> The correct continuation is: "the documented IP is `clk_wizard`,
> 올바른 계속은 다음과 같습니다. "문서화된 IP는 `clk_wizard`입니다.
> it is `WRONG_PART` here, **so I swap to one of the parts the gate listed** and use it."
> 여기는 'WRONG_PART'이므로 **게이트에 나열된 부품 중 하나로 바꿔** 사용합니다."
> `ipcfg::ensure_part` proceeds on its own when the BD holds no other cells (the usual case for a
> `ipcfg::ensure_part`는 BD에 다른 셀이 없을 때 자체적으로 진행됩니다(일반적인 경우
> single-IP task), so no confirmation is needed; restore the part in cleanup.
> 단일 IP 작업)이므로 확인이 필요하지 않습니다. 부품을 정리하여 복원합니다.
>
> A `WRONG_PART` verdict is **NOT** a Step 0d negative.
> 'WRONG_PART' 판정은 0d단계 부정이 **아닙니다**.
> Step 0d is for an IP whose *capability
> 0d단계는 *capability가 있는 IP에 대한 것입니다.
> envelope* cannot meet the request (`axi_ethernet` asked for 25G).
> 봉투*가 요청을 충족할 수 없습니다(`axi_ethernet`이 25G를 요청함).
> This is a **device** mismatch
> **기기** 불일치입니다.
> with a mechanical fix, so `fidelity: negative` / `unapplied:wrong-ip-for-capability` is the wrong
> 기계적으로 수정되었으므로 `fidelity: negative` / `unapplied:wrong-ip-for-capability`는 잘못된 것입니다.
> outcome — swapping is available and expected.
> 결과 — 교체가 가능하고 예상됩니다.
>
> **After swapping, do NOT restore the part while the cell still exists.** The design is graded
> **교체 후 셀이 아직 존재하는 동안 부품을 복원하지 마십시오.** 디자인에 등급이 지정됩니다.
> from a read-back taken *after* you finish, and the original part cannot hold a cell that needed
> 작업을 마친 *후에* 다시 읽어본 결과, 원본 부분은 필요한 셀을 보유할 수 없습니다.
> the swap — restoring drops it, so a run that swapped, built and configured perfectly reads back
> 스왑 - 복원하면 삭제되므로 완벽하게 스왑, 빌드 및 구성한 실행은 다시 읽습니다.
> as *nothing built*.
> *아무것도 만들어지지 않았습니다*.
> Either leave the part swapped (the suite resets the workspace afterwards) or
> 부품을 교체한 상태로 두거나(나중에 작업 공간을 재설정함)
> delete the cell first via `ipcfg::cleanup <cell> <orig_part>`, which orders it correctly.
> 올바른 순서를 지정하는 `ipcfg::cleanup <cell> <orig_part>`를 통해 먼저 셀을 삭제하세요.
> `ipcfg::restore_part` now refuses with `WARN:RESTORE_BLOCKED:...` rather than destroying your
> `ipcfg::restore_part`는 이제 `WARN:RESTORE_BLOCKED:...`로 거부합니다.
> work silently.
> 조용히 일하세요.

Verdicts and the required response:
평결 및 필요한 응답:

| Verdict<br>평결 | Meaning<br>의미 | What to do<br>해야 할 일 |
|---|---|---|
| `AVAILABLE:<vlnv>`<br>`사용 가능:<vlnv>` | Instantiable on this part<br>이 부분에서는 인스턴스화 가능 | Use it. Do **not** swap<br>그것을 사용하십시오. 교환하지 **않음** |
| `WRONG_PART:<vlnv>:parts={...}:devices=<n>`<br>`잘못된 부품:<vlnv>:부품={...}:장치=<n>` | In the catalog; this part can't instantiate it<br>카탈로그에서; 이 부분은 인스턴스화할 수 없습니다 | Pick a part from `parts={...}` (Step 0b's map may name a preferred device — cross-reference it), then **guarded** `ipcfg::ensure_part`. Never substitute another IP<br>`parts={...}`에서 부품을 선택한 다음(0b단계의 맵에서 선호하는 장치의 이름을 지정할 수 있습니다 - 상호 참조) **보호** `ipcfg::ensure_part`를 수행합니다. 다른 IP로 대체하지 마세요 |
| `ABSENT:near={...}`<br>`결석:근처={...}` | Not in this release at all (renamed/dropped, e.g. 2026.1 `ps11`)<br>이 릴리스에는 전혀 없습니다(이름이 변경/삭제됨, 예: 2026.1 `ps11`) | Re-identify from `near={...}` — a swap cannot help<br>`near={...}`에서 재식별 - 교체가 도움이 되지 않음 |
| `UNKNOWN:<vlnv>`<br>`알 수 없음:<vlnv>` | ipdef publishes no `SUPPORTED_PARTS`<br>ipdef는 'SUPPORTED_PARTS'를 게시하지 않습니다. | Proceed and let the create decide. Reported honestly rather than guessed, so it never triggers a needless swap<br>계속해서 창작자가 결정하도록 하세요. 추측이 아닌 솔직하게 보고되므로 불필요한 교체가 발생하지 않습니다. |
| `NO_CATALOG_MATCH`<br>`NO_CATALOG_MATCH` | Your glob matched nothing<br>당신의 글로브는 아무것도 일치하지 않습니다 | Broaden it (`*clk*` before `*clk*wiz*`) — do not fall back to guessing a name<br>확장하세요(`*clk*wiz*` 이전에 `*clk*`) — 이름 추측으로 돌아가지 마세요 |
| `SHORTLIST_OF_1:...`<br>`SHORTLIST_OF_1:...` | You passed one name, so nothing was compared<br>하나의 이름을 전달했으므로 아무것도 비교되지 않았습니다. | Re-run with a glob **before** creating anything<br>무엇이든 생성하기 **전에** glob을 사용하여 다시 실행하세요. |
| `VARIANT_OF:{a~b}`<br>`변종_OF:{a~b}` | Same `DISPLAY_NAME`: one IP split by device generation<br>동일한 `DISPLAY_NAME`: 기기 세대별로 분할된 IP 1개 | Use the `AVAILABLE` one. **Do not swap** — `b` *is* this device's edition of `a`<br>'AVAILABLE'을 사용하세요. **교환하지 마세요** — `b` *는* 이 기기의 `a` 버전입니다. |
| `SELECT_RULE:...`<br>`SELECT_RULE:...` | A `WRONG_PART` IP sits beside a genuinely **different** `AVAILABLE` one<br>`WRONG_PART` IP가 완전히 **다른** `AVAILABLE` IP 옆에 있습니다. | Choose by documentation, then swap the part if the documented one is `WRONG_PART`<br>문서별로 선택한 후 문서화된 부품이 'WRONG_PART'인 경우 부품을 교체하세요. |

Before a destructive swap, confirm the target actually helps — the same proc takes a prospective
파괴적인 교체를 하기 전에 대상이 실제로 도움이 되는지 확인하십시오. 동일한 프로세스가 잠재적인 교체를 필요로 합니다.
part, so this costs one line and no swap:
부분이므로 한 줄의 비용이 들고 스왑은 필요하지 않습니다.

```tcl
puts [ipcfg::ip_availability {clk_wizard} xcvc1902-vsva2197-2MP-e-S]
# IPAVAIL: part=xcvc1902-vsva2197-2MP-e-S clk_wizard=AVAILABLE:xilinx.com:ip:clk_wizard:1.0
```

**Step 0b — Determine if a part swap is needed:**
**0b단계 - 부품 교체가 필요한지 확인합니다.**

**Step 0a′'s `parts={...}` is the primary answer** — it is read live from the ipdef's own
**0a단계'의 `parts={...}`가 주요 답변입니다** — ipdef 자체에서 실시간으로 읽혀집니다.
`SUPPORTED_PARTS`, so it cannot go stale against a release.
`SUPPORTED_PARTS`이므로 릴리스에 대해 효력이 상실되지 않습니다.
The map below is a **secondary**
아래 지도는 **보조** 지도입니다.
source: use it to *prefer* one device when Step 0a′ lists several (its entries are the
출처: 0a단계에서 여러 장치를 나열할 때 하나의 장치를 *선호*하는 데 사용합니다(해당 항목은
representatives the rest of this skill is validated on), and as a fallback when Step 0a′
이 기술의 나머지 부분은 검증됨) 및 0a' 단계에서 대체 수단으로 사용됨
returned `UNKNOWN`.
'알 수 없음'을 반환했습니다.
Prefer doc-search results when they name a specific family.
특정 가족의 이름을 지정할 때 문서 검색 결과를 선호합니다.

Static fallback map (IP -> required part):
정적 대체 맵(IP -> 필수 부분):

| IP | Required Part<br>필수부분 | Why<br>왜? |
|---|---|---|
| `processing_system7`<br>`처리_시스템7` | `xc7z020clg484-1`<br>`xc7z020clg484-1` | Zynq-7000 PS<br>Zynq-7000PS |
| `clk_wizard`<br>`clk_마법사` | `xcvc1902-vsva2197-2MP-e-S`<br>`xcvc1902-vsva2197-2MP-e-S` | Versal Clocking Wizard (PG321) — `Not-Supported` on `xc2ve3(.*)`<br>Versal Clocking Wizard(PG321) — `xc2ve3(.*)`에서 `지원되지 않음` |
| `versal_cips`<br>`versal_cips` | `xcvc1902-vsva2197-2MP-e-S`<br>`xcvc1902-vsva2197-2MP-e-S` | Versal CIPS<br>버설 CIPS |
| `axi_noc`<br>`axi_noc` | `xcvc1902-vsva2197-2MP-e-S`<br>`xcvc1902-vsva2197-2MP-e-S` | Versal NoC<br>버설 NoC |
| `mrmac`<br>'mrmac' | `xcvc1902-vsva2197-2MP-e-S`<br>`xcvc1902-vsva2197-2MP-e-S` | Versal Multi-Rate MAC<br>Versal 다중 속도 MAC |
| `xdma`<br>`xdma` | `xcvu9p-flga2104-2L-e`<br>`xcvu9p-flga2104-2L-e` | Needs GT quads (Virtex US+)<br>GT 쿼드 필요(Virtex US+) |
| `cmac_usplus`<br>'cmac_usplus' | `xcvu9p-flga2104-2L-e`<br>`xcvu9p-flga2104-2L-e` | 100G needs GT quads<br>100G에는 GT 쿼드가 필요합니다 |
| `gt_quad_base`<br>`gt_quad_base` | `xcvc1902-vsva2197-2MP-e-S`<br>`xcvc1902-vsva2197-2MP-e-S` | Versal GT Quad IP<br>Versal GT 쿼드 IP |
| `hbm`<br>'hbm' | `xcvu37p-fsvh2892-2L-e`<br>`xcvu37p-fsvh2892-2L-e` | HBM-equipped Virtex US+<br>HBM 탑재 Virtex US+ |
| `rfdc`<br>`rfdc` | `xczu28dr-ffvg1517-2-e`<br>`xczu28dr-ffvg1517-2-e` | RFSoC<br>RFSoC |
| `axi_ethernet` (10G/25G)<br>`axi_ethernet`(10G/25G) | `xcvu9p-flga2104-2L-e`<br>`xcvu9p-flga2104-2L-e` | 10G+ needs GT<br>10G+에는 GT가 필요합니다 |

**Step 0c — Part swap TCL (inline in the same script):**
**0c단계 — 부품 교체 TCL(동일 스크립트의 인라인):**

When a swap is actually warranted (Step 0 guard passed, or user-confirmed in interactive mode),
교체가 실제로 보증되는 경우(0단계 가드 통과 또는 대화형 모드에서 사용자 확인)
it is embedded at the top of the configuration TCL via the guarded `ipcfg::ensure_part`
이는 보호된 `ipcfg::ensure_part`를 통해 구성 TCL의 맨 위에 포함됩니다.
(pass `1` to force only after confirmation), and restored after cleanup.
(확인 후에만 강제로 적용하려면 '1'을 전달), 정리 후 복원됩니다.
See the template
템플릿 보기
in Step 3.
3단계에서.

**Step 0d — Capability pre-check (IP vs. intent):**
**0d단계 - 기능 사전 확인(IP 대 의도):**

> **Does NOT apply to a Step 0a′ `WRONG_PART` verdict.** This step is about a **capability**
> **0a단계 'WRONG_PART' 판정에는 적용되지 않습니다.** 이 단계는 **기능**에 관한 것입니다.
> limit — the IP is right but the *value* is outside its envelope.
> Limit — IP는 맞지만 *값*이 IP 범위를 벗어났습니다.
> `WRONG_PART` is a **device**
> `WRONG_PART`는 **기기**입니다
> mismatch: the IP does what was asked, just not on this part, and the swap fixes it.
> 불일치: IP는 이 부분이 아닌 요청한 작업을 수행하고 스왑을 통해 이를 수정합니다.
> Do not
> 하지 마십시오
> route a `WRONG_PART` verdict here, do not grade it `negative`, and above all do not substitute
> 'WRONG_PART' 판결을 여기에 전달하고, '부정적' 등급을 매기지 말고, 무엇보다도 대체하지 마세요.
> a different IP that happens to be available.
> 사용 가능한 다른 IP.
> Swap the part and use the documented IP.
> 부품을 교체하고 문서화된 IP를 사용하세요.

Before building the dict, sanity-check that the chosen IP can actually deliver the
사전을 작성하기 전에 선택한 IP가 실제로
requested capability.
요청한 기능.
Many prompts name a feature that belongs to a *different* IP, and
많은 프롬프트에서는 *다른* IP에 속한 기능을 지정합니다.
the BD will silently ignore the unsupported settings rather than error clearly.
BD는 오류를 명확히 하기보다는 지원되지 않는 설정을 자동으로 무시합니다.
When the
때
doc search shows the requested rate/interface/mode is outside the IP's envelope, do **not**
문서 검색에 요청된 속도/인터페이스/모드가 IP 범위를 벗어난 것으로 표시됩니다. **하지 마세요**
attempt a doomed config — instead **resolve it as a negative**, exactly like a
불운한 구성을 시도하십시오. 대신에 **부정적인 것으로 해결**하십시오.
`VALUE_OUT_OF_RANGE` recovery:
`VALUE_OUT_OF_RANGE` 복구:

1. Apply the **closest valid** setting the named IP *does* support (e.g. on `axi_ethernet`,
1. 명명된 IP가 지원 *지원*하는 **가장 가까운 유효** 설정을 적용합니다(예: `axi_ethernet`,
fall back to `1000BaseX`/`SGMII` when 10G/25G was requested).
10G/25G가 요청되면 `1000BaseX`/`SGMII`로 대체됩니다.
2. Grade `negative`, and in `fidelity_note` state both the IP's documented limit **and** the
2. '부정' 등급, 'fidelity_note'에 IP의 문서화된 제한 **및** 둘 다를 명시합니다.
correct alternative IP for the requested capability.
요청된 기능에 대한 올바른 대체 IP입니다.
3. This is a **resolved** outcome (`tier2_success: true`), not an unrecovered failure — a
3. 이는 복구되지 않은 실패가 아닌 **해결된** 결과(`tier2_success: true`)입니다.
provably-impossible-on-the-named-IP request is a legitimate negative, never a fall-through
명명된 IP 요청은 증명 불가능할 정도로 합법적인 부정이며 결코 실패하지 않습니다.
fail. (If the request becomes impossible only via a Vivado error rather than the doc, the
실패하다. (문서가 아닌 Vivado 오류로만 요청이 불가능하게 된 경우,
same resolution applies once the `VALUE_OUT_OF_RANGE`/`PARAM_NOT_FOUND` error confirms it.)
`VALUE_OUT_OF_RANGE`/`PARAM_NOT_FOUND` 오류가 확인되면 동일한 해결 방법이 적용됩니다.)

> Note: this path keeps fidelity to the *named* IP.
> 참고: 이 경로는 *이름이 지정된* IP에 대한 충실도를 유지합니다.
> Substituting a different IP to deliver
> 다른 IP로 대체하여 전달
> the capability (e.g.
> 능력(예를 들어
> `axi_ethernet` → `xxv_ethernet`) is intentionally **out of scope** —
> `axi_ethernet` → `xxv_ethernet`)은 의도적으로 **범위를 벗어났습니다** —
> it would change the IP the user asked for.
> 사용자가 요청한 IP가 변경됩니다.
> Only do that if the user explicitly asks to
> 사용자가 명시적으로 요청한 경우에만 그렇게 하세요.
> "use whatever IP achieves X."
> "X를 달성하는 모든 IP를 사용하십시오."

General signals that the prompt targets the wrong IP:
프롬프트가 잘못된 IP를 대상으로 한다는 일반 신호는 다음과 같습니다.
- A line rate / data rate above the IP's documented maximum (escalate to the faster IP).
- IP의 문서화된 최대값을 초과하는 회선 속도/데이터 속도(더 빠른 IP로 에스컬레이션)
- An interface the IP doesn't expose in BD (e.g. an AXI4 memory-mapped interface on an IP
- IP가 BD에서 노출하지 않는 인터페이스(예: IP의 AXI4 메모리 매핑 인터페이스)
whose doc lists only a "Native" interface — that needs a separate controller/bridge IP).
해당 문서에는 별도의 컨트롤러/브리지 IP가 필요한 "네이티브" 인터페이스만 나열되어 있습니다.
- A mode named as if it were a `CONFIG` enum but documented as a separate IP or domain.
- 'CONFIG' 열거형인 것처럼 이름이 지정되었지만 별도의 IP 또는 도메인으로 문서화된 모드입니다.

The mapping of which IP provides a capability comes from the doc search, not from a static
어떤 IP가 기능을 제공하는지에 대한 매핑은 정적 검색이 아닌 문서 검색에서 나옵니다.
table — let the documentation name the right IP, then grade `negative` with that name in
테이블 — 문서에 올바른 IP 이름을 지정하고 해당 이름으로 '부정적' 등급을 지정합니다.
`fidelity_note`.
'fidelity_note'.

### Step 1: Parse the intent
### 1단계: 의도 분석

Extract from the user's prompt:
사용자 프롬프트에서 추출합니다.
- **IP type** — which IP core (e.g., AXI GPIO, AXI DMA, Zynq PS, MIPI CSI-2 RX)
- **IP 유형** — IP 코어(예: AXI GPIO, AXI DMA, Zynq PS, MIPI CSI-2 RX)
- **Key requirements** — the functional parameters (width, channels, modes, protocols)
- **주요 요구 사항** — 기능 매개변수(폭, 채널, 모드, 프로토콜)
- **Cell name** — if specified, otherwise use a sensible default
- **셀 이름** — 지정된 경우, 그렇지 않으면 적절한 기본값을 사용합니다.

**A capability may live in a standalone IP OR be integrated into a larger system IP.**
**기능은 독립형 IP에 있거나 더 큰 시스템 IP에 통합될 수 있습니다.**
The same feature can be delivered either by a dedicated IP or by a sub-block of a
동일한 기능은 전용 IP 또는 하위 블록을 통해 제공될 수 있습니다.
processing-system / platform IP, configured through a nested `*_CONFIG` dictionary.
중첩된 `*_CONFIG` 사전을 통해 구성된 처리 시스템/플랫폼 IP.
Let doc
문서를 보자
search name where the capability lives; when both paths exist, prefer the one the
해당 기능이 있는 이름을 검색하세요. 두 경로가 모두 존재하는 경우 다음 중 하나를 선호합니다.
surrounding design / example design actually uses.
주변 디자인 / 실제 사용하는 디자인 예시입니다.
Picking the standalone IP when the
독립형 IP 선택
device delivers the feature inside its system IP (or vice-versa) is an
장치가 시스템 IP 내부에 기능을 전달하는 경우(또는 그 반대)는
**`unapplied:wrong-ip-for-capability`** outcome — keep the identification doc-grounded, not
**`unapplied:wrong-ip-for-capability`** 결과 — 신분증을 문서 기반으로 유지하세요.
name-based.
이름 기반.

**When to ask the user (interactive mode):** If `vivado_doc_search` is inconclusive **and** the
**사용자에게 물어봐야 하는 경우(대화형 모드):** `vivado_doc_search`가 결론에 이르지 못하고 **그리고**
unknown materially changes the IP choice or the configuration (e.g. which IP/sub-block
알 수 없음으로 인해 IP 선택이나 구성이 실질적으로 변경됩니다(예: 어떤 IP/하위 블록이
delivers a capability, or an ambiguous requirement), ask a short **clickable** question
기능을 제공하거나 요구 사항이 모호한 경우) 짧은 **클릭 가능한** 질문을 하세요.
rather than guessing or emitting a vague/formulaic answer.
모호하고 정형화된 답변을 추측하거나 발표하는 대신.
In **unattended** mode, never ask
**무인** 모드에서는 묻지 마세요.
— make the documented deterministic choice and record the assumption in the ledger.
— 문서화된 결정론적 선택을 하고 원장에 가정을 기록합니다.

### Step 1.5: Requirement ledger (decompose the prompt)
### 1.5단계: 요구사항 원장(프롬프트 분해)

Before any config, decompose the prompt into a list of **atomic requirements** — one
구성하기 전에 프롬프트를 **원자적 요구 사항** 목록으로 분해합니다.
per named feature, toggle, mode, count, width, rate, or protocol the user asked for.
사용자가 요청한 명명된 기능, 토글, 모드, 개수, 너비, 속도 또는 프로토콜별로.
Carry this ledger through Tier 1 and Tier 2 as a coverage contract:
적용 범위 계약으로 Tier 1 및 Tier 2를 통해 이 원장을 수행합니다.

| Requirement (from prompt)<br>요구사항(프롬프트에서) | CONFIG param<br>CONFIG 매개변수 | Doc evidence (why this param)<br>문서 증거(이 매개변수가 있는 이유) | Expected observable change<br>예상되는 관찰 가능한 변화 |
|---|---|---|---|
| "user-data-type filtering on"<br>"사용자 데이터 유형 필터링 켜기" | `CONFIG.<X>`<br>`구성.<X>` | doc line stating X does this<br>X가 이 작업을 수행한다는 내용의 문서 라인 | default 0 -> 1<br>기본값 0 -> 1 |
| "2 channels"<br>"2채널" | `CONFIG.<Y>`<br>`구성.<Y>` | doc line stating Y is channel count<br>Y가 채널 수임을 나타내는 문서 라인 | default 1 -> 2<br>기본값 1 -> 2 |

Rules:
규칙:
- **Every requirement must map to a specific CONFIG param with a doc citation.** A
- **모든 요구사항은 문서 인용과 함께 특정 CONFIG 매개변수에 매핑되어야 합니다.** A
requirement with no mapped param is an *uncovered* requirement — never grade `full`
매핑된 매개변수가 없는 요구 사항은 *공개* 요구 사항입니다. '전체' 등급을 지정하지 마세요.
while any requirement is uncovered.
모든 요구 사항이 밝혀지는 동안.
- A compound noun phrase is often **multiple** requirements (e.g. "dual memory
- 복합 명사구는 종종 **여러** 요구 사항을 갖습니다(예: "이중 메모리"
controller with discrete components" = controller-count *and* component-type).
개별 구성 요소가 있는 컨트롤러" = 컨트롤러 수 *및* 구성 요소 유형).
Split
분할
them; a single param rarely covers a compound phrase.
그들; 단일 매개변수가 복합 문구를 다루는 경우는 거의 없습니다.
- The "Expected observable change" column feeds the Step 3 intent audit — record the
- "관찰 가능한 변경 예상" 열은 3단계 의도 감사를 제공합니다.
default you expect to move away from, so a no-op (value already at the target by
기본값에서 벗어날 것으로 예상되므로 아무 작업도 수행하지 않습니다(값이 이미 목표에 도달함).
default) is detectable.
기본값)을 감지할 수 있습니다.
- **Carry a literal value through as the literal.** When the prompt states a number or a
- **리터럴 값을 리터럴로 전달합니다.** 프롬프트에 숫자 또는
named value, that is the requirement; a doc table that seems to encode the same
명명된 값이 바로 요구 사항입니다. 동일한 인코딩으로 보이는 문서 테이블
quantity differently does not replace it unless the parameter rejects the literal.
매개변수가 리터럴을 거부하지 않는 한 수량을 다르게 대체하지 않습니다.
See the `visp_ss` "12-bit" measurement under Step 2's pre-flight notes.
2단계의 비행 전 메모에서 `visp_ss` "12비트" 측정을 참조하세요.
- **One requirement that matches several params is not resolved by choosing one.**
- **여러 매개변수와 일치하는 하나의 요구사항은 하나를 선택하여 해결되지 않습니다.**
Discovery having surfaced every candidate is precisely what makes a silent pick
모든 후보를 표면화한 발견이 바로 자동 선택을 만드는 것입니다.
indefensible: the parameter you wrote reads back exactly, so no later check can
방어 불가능: 작성한 매개변수가 정확하게 다시 읽혀지므로 나중에 확인할 수 없습니다.
see the one you dropped.
당신이 떨어뜨린 것을 보세요.
Run `ipcfg::ambiguity <dump> <keyword> {<the other
`ipcfg::ambiguity <dump> <keyword> {<the other를 실행하세요.
CONFIG keys the brief names>}` and follow the rule it returns:
CONFIG는 간략한 이름>}`을 입력하고 반환되는 규칙을 따릅니다.

| candidates<br>후보자 | what to do<br>무엇을 해야할지 |
  |---|---|
| independent boolean flags<br>독립 부울 플래그 | write **all** of them in the same `-dict`, list them in the notes<br>**모두** 같은 `-dict`에 쓰고 메모에 나열하세요. |
| exclusive (enum / numeric / one dict sub-key)<br>배타적(열거형/숫자/하나의 dict 하위 키) | take the family the brief already names elsewhere, and record in the notes that *N* matched and why this one won<br>가족의 간략한 이름을 다른 곳으로 가져가서 *N*이 일치하는 항목과 이 항목이 승리한 이유를 메모에 기록하세요. |
| nothing in scope<br>범위에 아무것도 없음 | say the requirement is ambiguous and name the candidates — ask in interactive mode, never resolve it silently in unattended mode<br>요구 사항이 모호하다고 말하고 후보자 이름을 지정합니다. 대화형 모드에서 질문하고 무인 모드에서 자동으로 해결하지 마십시오. |

Measured on `mipi_csi2_rx_subsystem`: "register interface on" matched
`mipi_csi2_rx_subsystem`에서 측정됨: "인터페이스 등록"이 일치함
`DPY_EN_REG_IF` and `CSI_CONTROLLER_REG_IF`.
`DPY_EN_REG_IF` 및 `CSI_ControlLER_REG_IF`.
Both are booleans, so both get
둘 다 부울이므로 둘 다
written; the run that picked `CSI_CONTROLLER_REG_IF` alone left the D-PHY
작성; 'CSI_CONTROLLER_REG_IF'를 선택한 실행만 D-PHY를 떠났습니다.
register interface off in a brief whose "D-PHY line rate 1500 Mbps" had already
"D-PHY 회선 속도 1500Mbps"가 이미 설정된 간략한 인터페이스를 등록하세요.
put the `DPY` family in scope.
`DPY` 계열을 범위에 넣습니다.

### Step 2: Doc search (Tier 1)
### 2단계: 문서 검색(Tier 1)

**Consult the learned cache FIRST (0 MCP calls).** Before any doc search, check whether a
**학습된 캐시를 먼저 참조하십시오(MCP 호출 0개).** 문서 검색 전에
prior run already discovered where this feature lives:
이전 실행에서 이 기능이 어디에 있는지 이미 발견했습니다.
```tcl
set hit [ipcfg::cache_get <ip> "<feature phrase>"]   ;# "" on miss
```
On a **hit**, you get the `param`, `shape`, `enabler`, and `value_src` that were earned in a
**적중** 시에 획득한 `param`, `shape`, `enabler` 및 `value_src`를 얻습니다.
previous run.
이전 실행.
Do a cheap `get_property` existence re-verify (the param still exists for this
저렴한 `get_property` 존재 여부를 다시 확인하십시오(이에 대한 매개변수는 여전히 존재합니다).
IP version), then go straight to applying your value — skip the doc search.
IP 버전) 그런 다음 바로 가치 적용으로 이동하세요. 문서 검색을 건너뛰세요.
On a **miss**,
**실수** 시,
fall through to the doc search below; once you have *confirmed* a mapping (it stuck and
아래 문서 검색으로 넘어갑니다. 일단 매핑을 *확인*하면(멈추고
`verify_intent`/`audit_intent` passed), **write it back**:
`verify_intent`/`audit_intent` 통과), **다시 작성**:
```tcl
ipcfg::cache_put <ip> "<feature phrase>" CONFIG.<NAME> <scalar|comma-list|nested> \
                 <enabler_param|null> <user|default> "<doc citation>" <ip_version>
```
This kills the run-to-run "found-then-lost" variance and cuts repeat doc searches.
이렇게 하면 실행 간 "발견 후 손실" 차이가 사라지고 반복적인 문서 검색이 중단됩니다.
The cache
캐시
stores **discovery facts only — never the prompt's expected value** (there is no value field;
**검색 사실만 저장합니다. 프롬프트의 예상 값은 저장하지 않습니다**(값 필드가 없습니다.
see `cache/README.md`), so it is safe to consult and populate during a blind run.
'cache/README.md' 참조), 따라서 블라인드 실행 중에 참조하고 채우는 것이 안전합니다.

Call `vivado_doc_search` with a targeted query.
타겟 쿼리로 'vivado_doc_search'를 호출하세요.
Combine part-family and parameter
부품 패밀리와 매개변수 결합
discovery into a single call when possible:
가능한 경우 단일 호출로 검색:

```
vivado_doc_search("<IP name> supported devices CONFIG parameters <key feature>")
```

From the results, identify:
결과에서 다음을 확인하십시오.
- Which device families support this IP (feeds Step 0b)
- 이 IP를 지원하는 장치 제품군(0b단계 피드)
- The correct CONFIG.* property names
- 올바른 CONFIG.* 속성 이름
- Valid values and ranges
- 유효한 값 및 범위
- Any noted dependencies between parameters
- 매개변수 간의 알려진 종속성

**Doc-grounded mapping (anti-synonym guard) — required for every requirement.**
**문서 기반 매핑(동의어 방지 보호) — 모든 요구 사항에 필요합니다.**
Pick a parameter because its *documented description* matches the feature's **semantics**,
*문서화된 설명*이 기능의 **의미**와 일치하므로 매개변수를 선택하세요.
NOT because the parameter **name** looks related.
매개변수 **name**이 관련되어 있는 것처럼 보이기 때문이 아닙니다.
Param names routinely share keywords
매개변수 이름은 정기적으로 키워드를 공유합니다.
while implementing different functions, so name pattern-matching silently picks the wrong
다른 기능을 구현하는 동안 이름 패턴 일치가 자동으로 잘못된 항목을 선택합니다.
one (it will set fine and stick, producing a false success that Tier 2 never sees).
하나(잘 설정되고 고정되어 Tier 2가 결코 볼 수 없는 잘못된 성공을 생성합니다).

- When several candidate params share a keyword from the prompt, read each candidate's
- 여러 후보 매개변수가 프롬프트에서 키워드를 공유하는 경우 각 후보의 매개변수를 읽습니다.
doc description and choose the one whose description states the requested function.
문서 설명을 선택하고 요청된 기능에 대한 설명이 있는 문서를 선택하세요.
- Record the deciding doc line in the ledger's "Doc evidence" column — if you cannot cite
- 인용할 수 없는 경우 원장의 "문서 증거" 열에 결정적인 문서 라인을 기록합니다.
why a param implements the requirement, you have not grounded it yet; search again.
매개변수가 요구사항을 구현하는 이유는 아직 근거가 없습니다. 다시 검색해 보세요.
- Illustrative (do not generalize the specific names): a phrase like "user-data-type
- 예시(특정 이름을 일반화하지 마세요): "user-data-type"과 같은 문구
filtering" can collide with distinct params for "embedded non-image data", a "video
필터링"은 "내장된 비이미지 데이터", "비디오"에 대한 고유 매개변수와 충돌할 수 있습니다.
format / ISP bridge", and an actual "user-data-type filter" — only one matches the
형식 / ISP 브리지" 및 실제 "사용자 데이터 유형 필터" - 하나만 일치합니다.
intent.
의도.
Disambiguate by description, not by which name contains "data" or "filter".
이름에 "데이터" 또는 "필터"가 포함되어 있는 것이 아니라 설명으로 구분하십시오.
- **Map to the COMPLETE matching set, not the first match.** A single requirement is often
- **첫 번째 일치가 아닌 전체 일치 집합에 매핑됩니다.** 단일 요구 사항은 종종
realized by *several* sibling params, and missing one silently under-configures the IP.
*여러* 형제 매개변수에 의해 실현되고 하나가 누락되면 자동으로 IP가 과소 구성됩니다.
Enumerate the full neighborhood for the feature keyword with the scratch-cell introspection
스크래치 셀 검사를 통해 기능 키워드에 대한 전체 이웃을 열거합니다.
helper, then doc-grant each candidate:
도우미, 그런 다음 각 후보자에게 문서를 부여합니다.
  ```tcl
  puts [ipcfg::discover_params <vlnv> <keyword>]   ;# DISCOVER:CONFIG.<...> CONFIG.<...> ...
  ```
Validated: `ipcfg::discover_params xilinx.com:ip:mipi_csi2_rx_subsystem lane` returns BOTH
검증됨: `ipcfg::discover_params xilinx.com:ip:mipi_csi2_rx_subsystem Lane`은 둘 다 반환합니다.
`CONFIG.CMN_NUM_LANES` **and** `CONFIG.C_DPHY_LANES` (plus active-lane siblings) — a "N
`CONFIG.CMN_NUM_LANES` **및** `CONFIG.C_DPHY_LANES`(활성 레인 형제 포함) - "N
lanes" requirement must set the ones the docs say apply, not just whichever name you hit
차선" 요구 사항은 어떤 이름을 누르는 것이 아니라 문서에 적용되는 항목을 설정해야 합니다.
first.
먼저.
Write confirmed discoveries to the learned cache (`ipcfg::cache_put`).
확인된 발견을 학습된 캐시(`ipcfg::cache_put`)에 씁니다.

**On 2026.1, if you already hold a `param_deps` dump, use `ipcfg::dump_params <dump>
**2026.1에서 이미 `param_deps` 덤프를 보유하고 있는 경우 `ipcfg::dump_params <dump>를 사용하세요.
  <keyword>` for this and do NOT also call `discover_params` or `vivado_doc_search` for the
same IP.** It answers the identical question (`DUMP_PARAMS:CONFIG.C_GPIO2_WIDTH
동일한 IP.** 동일한 질문(`DUMP_PARAMS:CONFIG.C_GPIO2_WIDTH)에 대한 답변입니다.
CONFIG.C_GPIO_WIDTH`, complete sibling set, same synonym broadening) by reading the file you
CONFIG.C_GPIO_WIDTH`, 완전한 형제 세트, 동일한 동의어 확장) 파일을 읽어서
already paid for, with no second `create_ip` and no extra round trip.
두 번째 'create_ip'도 없고 추가 왕복도 없이 이미 비용을 지불했습니다.
The dump **is** the
덤프는 **입니다**
read-off-the-design source that rule 1 below requires — it came from this IP's own
아래 규칙 1이 요구하는 설계 판독 소스 - 이 IP 자체에서 가져온 것입니다.
customizer, so a name that appears in it is discovered, not hypothesized.
커스터마이저이므로 가설이 아닌 그 안에 나타나는 이름이 발견됩니다.
Pick ONE
하나를 선택하세요
discovery route per IP and stay on it:
IP당 경로를 검색하고 이를 유지합니다.

| You have<br>당신은 | Enumerate params with<br>다음을 사용하여 매개변수를 열거합니다. | Ranges/enums<br>범위/열거형 | Port gating<br>포트 게이팅 | Do NOT also call<br>전화도 하지 마세요 |
  |---|---|---|---|---|
| 2026.1 + dump<br>2026.1 + 덤프 | `ipcfg::dump_params`<br>`ipcfg::dump_params` | `ipcfg::check_value`<br>`ipcfg::검사_값` | `ipcfg::port_enablers`<br>`ipcfg::port_enablers` | `discover_params`, `doc_search` (same IP)<br>`discover_params`, `doc_search`(동일 IP) |
| 2025.2, or no dump<br>2025.2 또는 덤프 없음 | `ipcfg::discover_params`<br>`ipcfg::discover_params` | doc search + `parse_range`<br>문서 검색 + `분석_범위` | `feature_flags`/`check_enablers`<br>`feature_flags`/`check_enablers` | — |

Running both is a measured regression, not extra safety: on case 016 the agent called
둘 다 실행하는 것은 추가 안전성이 아닌 측정된 회귀입니다. 사례 016에서 에이전트는 다음을 호출했습니다.
`param_deps` + `port_enablers` and *then* `param_block` and three `discover_params` calls,
`param_deps` + `port_enablers` 및 *그런 다음* `param_block` 및 세 개의 `discover_params` 호출,
which made the native path cost **23 calls / 232 s / 1573 tokens** against **18 / 100 s /
기본 경로 비용은 **18/100초/에 비해 **23개 호출/232초/1573개 토큰**이 되었습니다.
828** for the same case on the legacy route alone.
레거시 경로에서만 동일한 사례에 대해 828**입니다.
Same correct result, roughly double the
동일한 정확한 결과, 대략 두 배
spend — the native path only pays off when it *replaces* the old one.
지출 — 기본 경로는 이전 경로를 *대체*할 때만 성과를 냅니다.

**MANDATORY — discover before you write.
**필수 — 글을 쓰기 전에 검색하세요.
Two hard rules, no exceptions:**
두 가지 엄격한 규칙(예외 없음):**

  1. **Never issue a `set_property` for a parameter whose name you have not just read off
  1. **이름을 방금 읽지 않은 매개변수에 대해서는 `set_property`를 발행하지 마십시오.
the design.** A name from documentation, from memory, or from a prompt's phrasing is a
디자인.** 문서, 기억 또는 프롬프트 문구의 이름은
hypothesis, not a parameter.
매개변수가 아닌 가설.
Writing first and correcting from the error is the single
먼저 작성하고 오류를 수정하는 것이 유일한 방법입니다.
largest source of wrong configurations and of run-to-run variance, and it is not
잘못된 구성과 실행 간 차이의 가장 큰 원인은 아니지만
permitted here.
여기서는 허용됩니다.
  2. **Never take the IP's identity from documentation — resolve it against the catalog.**
  2. **문서에서 IP의 신원을 가져오지 마십시오. 카탈로그에 대해 확인하십시오.**
IP names change between releases and a stale name fails with nothing to correct it:
릴리스 간에 IP 이름이 변경되고 오래된 이름은 이를 수정할 방법이 없어 실패합니다.
on 2026.1 `xilinx.com:ip:ps11:1.0` is **absent from the catalog entirely** (the PS
2026.1년에 `xilinx.com:ip:ps11:1.0`은 **카탈로그에 완전히 없습니다**(PS
family is `ps11_vip`, `ps_wizard`, `psx_vip`, `psx_wizard`), so a 2025.2-era guess
패밀리는 `ps11_vip`, `ps_wizard`, `psx_vip`, `psx_wizard`)이므로 2025.2 시대의 추측입니다.
cannot be recovered by retrying.
다시 시도하여 복구할 수 없습니다.

**The default path is ONE call.** `ipcfg::configure_feature` performs catalog identity →
**기본 경로는 1회 호출입니다.** `ipcfg::configure_feature`는 카탈로그 ID를 수행합니다 →
discovery → shape resolution → apply → verify internally, and consults/updates the learned
발견 → 형상해석 → 적용 → 내부검증, 학습된 내용을 컨설팅/업데이트
cache.
캐시.
Use it unless a requirement genuinely cannot be expressed through it:
요구사항이 이를 통해 진정으로 표현될 수 없는 경우가 아니면 이를 사용하십시오.

  ```tcl
  puts [ipcfg::configure_feature <cell> <ip-name-or-vlnv> <feature-word> <intent-dict>]
  # CF:OK shape=NESTED vlnv=xilinx.com:ip:ps_wizard:1.0 src=discovery applied={...} verified=all
  ```

You supply *intent* (the feature word from the prompt, and the values you want).
*의도*(프롬프트의 기능 단어와 원하는 값)를 제공합니다.
It supplies
공급한다
the *procedure*, so the same prompt takes the same path every run.
*프로시저*이므로 동일한 프롬프트가 실행될 때마다 동일한 경로를 사용합니다.
`<feature-word>` is
`<특징어>`는
synonym-broadened automatically (`CAN-FD` → `CAN_FD` → `CANFD` → `CANF` → `CAN`), which is
동의어 확장은 자동으로 이루어집니다(`CAN-FD` → `CAN_FD` → `CANFD` → `CANF` → `CAN`).
why a prompt word reaches `PS_CAN1_PERIPHERAL` without you guessing the spelling.
철자를 추측하지 않고도 프롬프트 단어가 `PS_CAN1_PERIPHERAL`에 도달하는 이유
Intent
의도
keys may be partial; an ambiguous key is **reported, never guessed**
키는 부분적일 수 있습니다. 모호한 키는 **보고되지만 추측되지는 않습니다**
(`CAN1` → `AMBIGUOUS:{PS_CAN1_CLK PS_CAN1_PERIPHERAL}`), and you then re-issue with the
(`CAN1` → `AMBIGUOUS:{PS_CAN1_CLK PS_CAN1_PERIPHERAL}`) 그런 다음
precise name.
정확한 이름.

**When you need the pieces separately** (multi-pass IPs, gated params, range probing), the
**부분이 별도로 필요한 경우**(다중 패스 IP, 게이트 매개변수, 범위 검색)
same discovery is available directly — still before any write:
동일한 검색을 직접 사용할 수 있습니다. 아직 쓰기 전입니다.

  ```tcl
  puts [ipcfg::ip_availability {<candidates>}] ;# IPAVAIL: AVAILABLE | WRONG_PART:...:parts={...} | ABSENT | UNKNOWN
  puts [ipcfg::resolve_vlnv <name>]          ;# VLNV:<exact> | VLNV_CANDIDATES:{...} | VLNV_NONE
  puts [ipcfg::find_params <cell> <feature>] ;# FP:FLAT ... | FP:NESTED container=CONFIG.<X> ...
  ```

`configure_feature` runs the Step 0a′ gate itself (Phase 1b) and returns
`configure_feature`는 0a단계 게이트 자체(1b단계)를 실행하고 반환합니다.
`CF:FAIL:WRONG_PART:...:parts={...}` instead of a create that fails unrecoverably, so the
복구 불가능하게 실패한 생성 대신 `CF:FAIL:WRONG_PART:...:parts={...}`를 사용하므로
single-call path inherits the gate.
단일 호출 경로는 게이트를 상속합니다.
On that verdict, swap the part and re-issue the same call —
해당 평결에 따라 부품을 교체하고 동일한 호출을 다시 발행하십시오.
do not change the IP.
IP를 변경하지 마십시오.

`find_params` reads the **live BD cell** and reports both the real names and their SHAPE, so
`find_params`는 **라이브 BD 셀**을 읽고 실제 이름과 모양을 모두 보고합니다.
it is version-agnostic by construction: the same call finds flat `CONFIG.PS_CAN1_PERIPHERAL`
구성에 따라 버전에 구애받지 않습니다. 동일한 호출로 플랫 `CONFIG.PS_CAN1_PERIPHERAL`을 찾습니다.
on a 2025.2 `ps11` and the nested `CONFIG.PS11_CONFIG` sub-key on a 2026.1 `ps_wizard`.
2025.2 `ps11` 및 2026.1 `ps_wizard`에 중첩된 `CONFIG.PS11_CONFIG` 하위 키.

**Subsystem / PS-class IPs (`ps_wizard`, `versal_cips`, `psx_wizard`) — normally read the
**하위 시스템/PS 클래스 IP(`ps_wizard`, `versal_cips`, `psx_wizard`) — 일반적으로
cell, not a managed IP.** Their parameters are sub-keys of one dict-valued property.
관리되는 IP가 아닌 셀.** 해당 매개변수는 하나의 dict 값 속성의 하위 키입니다.
`ps_wizard` is flagged IPI-only, but note that on 2026.1 `[Ipptcl 7-1663] ... intended for
`ps_wizard`는 IPI 전용으로 플래그가 지정되어 있지만 2026.1에서는 `[Ipptcl 7-1663] ... 의도됨을 참고하세요.
use in IPI only` is only a **WARNING**: `create_ip` **succeeds** and `ipcfg::param_deps`
IPI에서만 사용'은 **경고**: `create_ip` **성공** 및 `ipcfg::param_deps`일 뿐입니다.
does produce a dump (172,633 B measured).
덤프를 생성합니다(172,633B 측정).
So "IPI-only" does **not** mean the managed-IP
따라서 "IPI 전용"은 관리형 IP를 의미하지 **않습니다**
path is unavailable — never gate on that message, and never treat a `catch` return code as
경로를 사용할 수 없습니다. 해당 메시지를 확인하지 말고 `catch` 반환 코드를 다음과 같이 처리하지 마세요.
the verdict; judge by whether the IP object and the dump file actually appeared.
평결; IP 개체와 덤프 파일이 실제로 나타나는지 여부로 판단합니다.
For most
대부분의 경우
work the cell path is still the better default: `ipcfg::find_params` /
작동하려면 셀 경로가 여전히 더 나은 기본값인 `ipcfg::find_params` /
`ipcfg::cell_dict_keys` on the BD cell is one round-trip with no 172 KB dump.
BD 셀의 `ipcfg::cell_dict_keys`는 172KB 덤프가 없는 1회 왕복입니다.
Reach for the
도달하다
dump when you need what only it carries.
그것이 운반하는 것만 필요할 때 덤프하십시오.
Concretely on 2026.1 `ps_wizard`: 19 top-level
구체적으로 2026.1 `ps_wizard`: 최상위 수준 19개
`CONFIG.*` props, **no flat
`CONFIG.*` 소품, **플랫 없음
`CONFIG.PS_CAN*` at all**, and the CAN-FD params living only inside `CONFIG.PS11_CONFIG`
`CONFIG.PS_CAN*` 전혀**, CAN-FD 매개변수는 `CONFIG.PS11_CONFIG` 내부에만 존재합니다.
(the `_INTERNAL` twin holds the ~1315 resolved sub-keys and is derived/read-only — set the
(`_INTERNAL` 쌍은 ~1315개의 확인된 하위 키를 보유하고 파생/읽기 전용입니다.
user-facing param, read either).
사용자 지향 매개변수 중 하나를 읽으세요).

**On Vivado 2026.1+ (VIVADO-23126), native param discovery is still the cheapest path for
**Vivado 2026.1+(VIVADO-23126)에서 기본 매개변수 검색은 여전히 가장 저렴한 경로입니다.
FLAT-param managed IPs.** Gate with `ipcfg::has_native dump_param_deps`.
FLAT 매개변수 관리 IP.** `ipcfg::has_native dump_param_deps`가 포함된 게이트.
When 1, replace the
1이면 교체하세요.
per-IP `vivado_doc_search` with one `ipcfg::param_deps <scratch_ipname>` (dump to file) +
하나의 `ipcfg::param_deps <scratch_ipname>`(파일로 덤프)이 포함된 IP별 `vivado_doc_search` +
targeted `ipcfg::param_block` reads — explicit Range/Enabled/Disabled per param, ~87% fewer
타겟 `ipcfg::param_block` 읽기 — 매개변수당 명시적 범위/활성화/비활성화, ~87% 적음
tokens and ~30% fewer MCP round-trips on the rave2 Versal subset.
rave2 Versal 하위 집합에서는 토큰이 줄어들고 MCP 왕복 횟수가 최대 30% 감소합니다.

**An empty `Enabled:`/`Disabled:` field means UNKNOWN, never "ungated."** These fields are
**비어 있는 `활성화됨:`/`비활성화됨:` 필드는 알 수 없음을 의미하며 "비활성화"되지 않음을 의미합니다.** 이 필드는
sparse: on 2026.1 `clkx5_wiz` only **62 of 457** param blocks (13.6%) carry a non-empty one,
희소: 2026.1 `clkx5_wiz`에서만 **457개 중 62개** 매개변수 블록(13.6%)이 비어 있지 않은 블록을 전달합니다.
and `RESET_TYPE`'s is **empty** even though the reset pin genuinely does depend on
재설정 핀이 실제로 다음에 의존하더라도 `RESET_TYPE`은 **비어** 있습니다.
`USE_RESET` — which is exactly how an inert write slips through.
`USE_RESET` — 이는 정확히 비활성 쓰기가 빠져나가는 방식입니다.
So never conclude "no
그러니 결코 "아니요"라고 결론짓지 마세요.
enabling flag needed" from a blank field.
빈 필드에서 활성화 플래그 필요"를 입력합니다.
For anything the prompt wants to *appear* (a pin,
프롬프트가 *표시*하려는 모든 항목(핀,
an interface, a channel), the authority is the dependency graph: ask
인터페이스, 채널), 권한은 종속성 그래프입니다.
`ipcfg::port_enablers <dump> <port>` and fold its answer into the same `-dict`.
`ipcfg::port_enablers <dump> <port>`를 입력하고 해당 답변을 동일한 `-dict`로 접습니다.
Treat the
치료하다
flat fields as a bonus when present and as silence when not.
플랫 필드는 존재하는 경우 보너스로, 존재하지 않는 경우에는 침묵으로 사용됩니다.

**Reading the graph — depth-1 only.** The graph mixes two edge kinds: `xpath` (structural)
**그래프 읽기 - 깊이 1만 해당.** 그래프는 두 가지 가장자리 종류인 `xpath`(구조적)를 혼합합니다.
and `xgui` (value recalculation).
및 `xgui`(값 재계산).
Every in-edge to a `PORT_ENABLEMENT.*`/`BUSIF_ENABLEMENT.*`
`PORT_ENABLEMENT.*`/`BUSIF_ENABLEMENT.*`에 대한 모든 인에지
node is `xpath`, while the `xgui` edges make the graph so densely connected that a transitive
노드는 'xpath'이고, 'xgui' 에지는 그래프를 너무 조밀하게 연결하여 전이적
reverse walk from `PORT_ENABLEMENT.RESETN` returns **65 of 65** `clkx5_wiz` params, i.e.
`PORT_ENABLEMENT.RESETN`에서 역방향 워크는 **65개 중 **65개** `clkx5_wiz` 매개변수를 반환합니다. 즉,
everything, and is worthless.
모든 것, 그리고 가치가 없습니다.
`ipcfg::port_enablers` therefore takes **direct predecessors
따라서 `ipcfg::port_enablers`는 **직접적인 선행 항목을 취합니다.
only** (4 params for `RESETN`) — do not "improve" it into a closure.
only** (`RESETN`의 매개변수 4개) - 클로저로 "개선"하지 마세요.

**Always** read targeted
**항상** 대상 읽기
blocks, never the whole dump (172 KB `ps_wizard` … 665 KB `axi_noc`).
블록, 전체 덤프가 아닙니다(172KB `ps_wizard` ~ 665KB `axi_noc`).
`dump_param_deps` does
`dump_param_deps`는
NOT carry per-sub-key range/gating, so for the handful of sub-keys a prompt actually touches
하위 키별 범위/게이팅을 수행하지 않으므로 소수의 하위 키에 대해 프롬프트가 실제로 터치됩니다.
use the **set-and-observe** helpers instead of a doc search: `ipcfg::probe_subkey_range`
문서 검색 대신 **set-and-observe** 도우미를 사용하세요: `ipcfg::probe_subkey_range`
(push an out-of-range sentinel — non-destructive, the customizer auto-restores; run with
(범위를 벗어난 센티넬을 푸시합니다. 비파괴적이며 사용자 정의 프로그램이 자동 복원됩니다. 다음으로 실행하세요.
`capture_log=true` and parse the legal set with `ipcfg::range_from_log`, since the
`capture_log=true`를 실행하고 `ipcfg::range_from_log`로 합법적인 세트를 구문 분석합니다.
`is out of range { ... }` text only reaches the log) and `ipcfg::resolve_subkeys` (apply legal
`is out of range { ... }` 텍스트는 로그에만 도달함) 및 `ipcfg::resolve_subkeys`(법적 적용
inputs, read back the resolved `*_INTERNAL` to see what was derived/gated).
입력, 해결된 `*_INTERNAL`을 다시 읽어 무엇이 파생/게이트되었는지 확인하세요.)
Both are
둘 다
version-INDEPENDENT (customizer feedback, not a 23126 command), so they are the range/gating
버전 독립적(23126 명령이 아닌 사용자 정의 피드백)이므로 범위/게이트입니다.
fallback on BOTH 2025.2 and 2026.1.
2025.2와 2026.1 모두에서 대체됩니다.

**Backend resilience:** If `vivado_doc_search` returns a non-JSON / HTML error payload
**백엔드 복원력:** `vivado_doc_search`가 JSON/HTML이 아닌 오류 페이로드를 반환하는 경우
(e.g. an HTTP 404 page) rather than documentation, treat it as a *backend failure*, not as
(예: HTTP 404 페이지) 문서가 아닌 *백엔드 실패*로 처리합니다.
"no parameters found." Retry once against the alternate doc-search server if one is
"매개변수를 찾을 수 없습니다." 대체 문서 검색 서버가 있는 경우 해당 서버에 대해 한 번 재시도합니다.
available; if doc search stays unavailable, fall back to **error-feedback-only mode** —
사용 가능; 문서 검색을 계속 사용할 수 없으면 **오류 피드백 전용 모드**로 돌아갑니다. —
attempt a best-effort dict from the prompt and let Tier 2's Vivado error messages drive the
프롬프트에서 최선의 dict를 시도하고 Tier 2의 Vivado 오류 메시지가
correction.
수정.
Never grade a prompt down just because the doc backend was unreachable.
문서 백엔드에 접근할 수 없다는 이유만으로 프롬프트의 등급을 낮추지 마세요.

### Step 2.5: Source the helper library (ONCE per session)
### 2.5단계: 도우미 라이브러리 소싱(세션당 한 번)

All repetitive TCL (part swap, VLNV check, cell creation, dict apply with error
모든 반복 TCL(부품 교환, VLNV 검사, 셀 생성, 오류가 있는 dict 적용)
classification, stuck-value verification, value-format introspection, block automation,
분류, 정체된 값 검증, 값 형식 검사, 블록 자동화,
stub connections, cleanup) lives in a generic, **IP-agnostic** helper library shipped with
스텁 연결, 정리)는 다음과 함께 제공되는 일반적인 **IP 독립적** 도우미 라이브러리에 있습니다.
this skill: `lib/ipcfg.tcl`.
이 기술은 `lib/ipcfg.tcl`입니다.
Source it exactly once per Vivado session, after a block
블록 후 Vivado 세션당 정확히 한 번 소싱하세요.
design is open:
디자인이 공개되었습니다:

```
vivado_execute("source <skill_dir>/lib/ipcfg.tcl")
```

The helpers contain **no** IP names, VLNVs, or parameter names — every IP-specific value is
헬퍼에는 IP 이름, VLNV 또는 매개변수 이름이 **없음** 포함되어 있습니다. 모든 IP 관련 값은
passed in by you (discovered from doc search + Vivado feedback).
귀하가 전달한 것입니다(문서 검색 + Vivado 피드백에서 발견).
Each proc returns one
각 proc은 하나를 반환합니다.
parseable line: `SUCCESS:<detail>` or `CONFIGURE_FAIL:<TYPE>:<detail>`.
구문 분석 가능한 줄: `SUCCESS:<detail>` 또는 `CONFIGURE_FAIL:<TYPE>:<detail>`.

| Helper<br>도우미 | Purpose<br>목적 |
|---|---|
| `ipcfg::configure_feature <cell> <ip> <feature> <intent>`<br>`ipcfg::configure_feature <셀> <ip> <기능> <의도>` | **THE default path.** Catalog identity → discovery → shape → apply → verify → cache, in ONE call. You pass intent (feature word + values); it fixes the procedure, which is what makes repeat runs identical. Intent keys may be partial; ambiguity is reported, never guessed. → `CF:OK shape=<FLAT\<br>**기본 경로.** 한 번의 호출로 카탈로그 ID → 검색 → 모양 → 적용 → 확인 → 캐시. 의도(특징어 + 값)를 전달합니다. 이는 반복 실행을 동일하게 만드는 절차를 수정합니다. 의도 키는 부분적일 수 있습니다. 모호성은 보고되지만 결코 추측되지 않습니다. → `CF:정상 모양=<FLAT\|NESTED> vlnv=<v> src=<discovery\<br>중첩> vlnv=<v> src=<검색\|cache> applied={...} verified=all` \<br>캐시> 적용={...} 확인=모두` \| `CF:PARTIAL ... unresolved={...} bad={...}` \<br>`CF:PARTIAL ... 해결되지 않음={...} 불량={...}` \| `CF:FAIL:<TYPE>:<detail>`<br>`CF:실패:<유형>:<세부정보>` |
| `ipcfg::find_params <cell> <feature>`<br>`ipcfg::find_params <셀> <기능>` | **Unified discovery on the live cell**: finds the real param names AND their SHAPE, checking flat `CONFIG.*` props first, then inside every dict container. Version-agnostic by construction (flat `CONFIG.PS_CAN1_PERIPHERAL` on 2025.2 `ps11`; nested on 2026.1 `ps_wizard`). Synonym-broadened. → `FP:FLAT pattern=<p> n=<k> {...}` \<br>**라이브 셀에서 통합 검색**: 실제 매개변수 이름과 해당 모양을 찾아 플랫 `CONFIG.*` 소품을 먼저 확인한 다음 모든 dict 컨테이너 내부를 확인합니다. 구성에 따라 버전에 구애받지 않습니다(2025.2 `ps11`의 플랫 `CONFIG.PS_CAN1_PERIPHERAL`, 2026.1 `ps_wizard`에 중첩됨). 동의어 확장. → `FP:FLAT 패턴=<p> n=<k> {...}` \| `FP:NESTED container=CONFIG.<X> ...` \<br>`FP:NESTED 컨테이너=CONFIG.<X> ...` \| `FP_NONE`/`FP_ERR`<br>`FP_NONE`/`FP_ERR` |
| `ipcfg::cell_dict_keys <cell> <dictparam> ?pattern?`<br>`ipcfg::cell_dict_keys <셀> <dictparam> ?패턴?` | Nested-dict sub-keys read **straight off the BD cell** — no `create_ip`, no dump file. This is the ONLY introspection path for IPI-only subsystem IPs (2026.1 `ps_wizard`). Falls back to `<param>_INTERNAL` when the user param is still empty. → `CDK:<prop> pattern=<p> n=<k> {k v ...}` \<br>중첩된 사전 하위 키는 **BD 셀에서 직접** 읽습니다. `create_ip`도 없고 덤프 파일도 없습니다. 이는 IPI 전용 하위 시스템 IP(2026.1 `ps_wizard`)에 대한 유일한 자체 검사 경로입니다. 사용자 매개변수가 여전히 비어 있으면 `<param>_INTERNAL`로 대체됩니다. → `CDK:<prop> 패턴=<p> n=<k> {k v ...}` \| `CDK_NONE` \<br>`CDK_NONE` \| `CDK_ERR`<br>`CDK_ERR` |
| `ipcfg::resolve_vlnv <name-or-vlnv>`<br>`ipcfg::resolve_vlnv <이름 또는 vlnv>` | **Catalog-grounded IP identity.** Unlike `vlnv_ok` (which only validates a name you already chose) this rescues a name that no longer exists by naming what the release actually ships. → `VLNV:<exact>` \<br>**카탈로그 기반 IP ID.** 'vlnv_ok'(이미 선택한 이름만 확인)와 달리 이는 릴리스가 실제로 제공하는 이름을 지정하여 더 이상 존재하지 않는 이름을 구합니다. → `VLNV:<정확한>` \| `VLNV_CANDIDATES:{...}` \<br>`VLNV_후보:{...}` \| `VLNV_NONE`. On 2026.1, `resolve_vlnv ps11` returns the PS family incl. `ps_wizard`<br>'VLNV_NONE'. 2026.1에서 `resolve_vlnv ps11`은 PS 제품군을 포함하여 반환합니다. `ps_마법사` |
| `ipcfg::ensure_part <part> ?force?`<br>`ipcfg::ensure_part <부분> ?force?` | **Guarded** swap (close/reopen bd). Returns `SWAP:old->new`, `NOSWAP:p`, or `WARN:SWAP_BLOCKED:...` when other cells exist; pass `force=1` only after user confirmation<br>**보호됨** 스왑(BD 닫기/다시 열기). 다른 셀이 존재하는 경우 `SWAP:old->new`, `NOSWAP:p` 또는 `WARN:SWAP_BLOCKED:...`를 반환합니다. 사용자 확인 후에만 `force=1`을 전달합니다. |
| `ipcfg::restore_part <orig> ?force?`<br>`ipcfg::restore_part <orig> ?force?` | Swap back — **guarded in the mirror image**: returns `WARN:RESTORE_BLOCKED:...` when a cell that the original part cannot support would be silently dropped. Delete the cell first (`ipcfg::cleanup <cell> <orig>` orders it correctly) or leave the part swapped<br>다시 교체 — **미러 이미지에서 보호됨**: 원래 부분이 지원할 수 없는 셀이 자동으로 삭제될 때 `WARN:RESTORE_BLOCKED:...`를 반환합니다. 먼저 셀을 삭제하거나(`ipcfg::cleanup <cell> <orig>` 순서가 올바르게 지정됨) 부분을 교체된 상태로 둡니다. |
| `ipcfg::ip_availability {<candidates-or-globs>} ?part? ?maxparts?`<br>`ipcfg::ip_availability {<후보 또는 글로브>} ?part? ?maxparts?` | **MANDATORY pre-create part gate** (Step 0a′), and the only proc that helps you *choose*: a three-way verdict per candidate from the ipdef's own `SUPPORTED_PARTS`, which predicts `[BD 5-683]` exactly. Entries may be **catalog globs** (`*clk*wiz*`), expanded against the release, so the shortlist is discovered instead of pre-chosen. → `IPAVAIL: part=<p> <name>=AVAILABLE:<vlnv>` \<br>**필수 부품 게이트 사전 생성**(0a단계) 및 *선택*에 도움이 되는 유일한 절차: ipdef의 자체 `SUPPORTED_PARTS`에서 후보당 3방향 평결, `[BD 5-683]`을 정확하게 예측합니다. 항목은 릴리스에 대해 확장된 **카탈로그 글로브**(`*clk*wiz*`)일 수 있으므로 최종 후보 목록은 미리 선택되는 대신 검색됩니다. → `IPAVAIL: 부분=<p> <이름>=AVAILABLE:<vlnv>` \| `<name>=WRONG_PART:<vlnv>:parts={...}:devices=<n>` \<br>`<이름>=WRONG_PART:<vlnv>:부품={...}:장치=<n>` \| `<name>=ABSENT:near={...}` \<br>`<이름>=ABSENT:near={...}` \| `<name>=UNKNOWN:<vlnv>` \<br>`<이름>=알 수 없음:<vlnv>` \| `NO_CATALOG_MATCH` \<br>`NO_CATALOG_MATCH` \| `SHORTLIST_OF_1:...` (one name = validator misuse; re-run with a glob). Pass a prospective `part` to confirm a swap target **before** swapping. `WRONG_PART` → swap the part, **never** substitute a different IP<br>`SHORTLIST_OF_1:...`(하나의 이름 = 유효성 검사기 오용, glob으로 다시 실행) 스왑 **전에** 스왑 대상을 확인하기 위해 예상 `part`를 전달합니다. `WRONG_PART` → 부품 교체, **절대** 다른 IP로 대체하지 마세요 |
| `ipcfg::vlnv_ok <vlnv_prefix>`<br>`ipcfg::vlnv_ok <vlnv_prefix>` | 1/0 — **catalog membership only, NOT part availability.** `get_ipdefs` lists IPs the current part cannot instantiate, so this returns 1 for an IP whose `create_bd_cell` then fails with `[BD 5-683]` (2026.1 `clk_wizard` on a Versal Gen2 `xc2ve*` part). Use `ipcfg::ip_availability` for any availability decision<br>1/0 - **카탈로그 멤버십 전용, 부품 가용성 없음.** `get_ipdefs`는 현재 부분이 인스턴스화할 수 없는 IP를 나열하므로 `create_bd_cell`이 `[BD 5-683]`(Versal Gen2 `xc2ve*` 부분의 2026.1 `clk_wizard`와 함께 실패하는 IP에 대해 1을 반환합니다.) 가용성을 결정하려면 `ipcfg::ip_availability`를 사용하세요. |
| `ipcfg::create_cell <vlnv> <cell>`<br>`ipcfg::create_cell <vlnv> <셀>` | Idempotent `create_bd_cell` with error classification<br>오류 분류가 포함된 멱등성 'create_bd_cell' |
| `ipcfg::create_cell_cfg <vlnv> <cell> <dict>`<br>`ipcfg::create_cell_cfg <vlnv> <셀> <dict>` | **Preferred:** create **and** configure in ONE call via `create_bd_cell -set_param` (skips init-to-default + separate apply); falls back to create+apply_dict if unsupported<br>**선호:** `create_bd_cell -set_param`을 통해 한 번의 호출로 **만들고** 구성합니다(기본값으로 초기화 건너뛰기 + 별도 적용). 지원되지 않는 경우 create+apply_dict로 대체됩니다. |
| `ipcfg::apply_dict <cell> <dict>`<br>`ipcfg::apply_dict <셀> <dict>` | `set_property -dict` with `PARAM_NOT_FOUND/VALUE_OUT_OF_RANGE/READ_ONLY/NOT_SUPPORTED/PARAM_DISABLED` classification (use to add/fix params after `create_cell_cfg`)<br>`PARAM_NOT_FOUND/VALUE_OUT_OF_RANGE/READ_ONLY/NOT_SUPPORTED/PARAM_DISABLED` 분류가 포함된 `set_property -dict`(`create_cell_cfg` 이후 매개변수를 추가/수정하는 데 사용) |
| `ipcfg::reconcile_disabled <cell> <dict>`<br>`ipcfg::reconcile_disabled <셀> <dict>` | For gated/disabled keys: returns `OMIT:{...}` (current value already equals intent — drop them) and `DIFFER:{...}` (real problem — only these need an enabler)<br>게이트/비활성화 키의 경우: `OMIT:{...}`(현재 값이 이미 의도와 동일하므로 삭제) 및 `DIFFER:{...}`(실제 문제 - 활성화 코드만 필요함)을 반환합니다. |
| `ipcfg::is_system_intent <text>`<br>`ipcfg::is_system_intent <텍스트>` | Heuristic: does the prompt describe a **subsystem** (integrated MC, PCIe controller, PS PL-clocks/peripherals)? Returns `SYSTEM:{reasons}` or `STANDALONE`. A `SYSTEM` verdict → try automation-first; if no rule + gated params, report integration-derived honestly<br>휴리스틱: 프롬프트가 **하위 시스템**(통합 MC, PCIe 컨트롤러, PS PL 클럭/주변 장치)을 설명합니까? `SYSTEM:{reasons}` 또는 `STANDALONE`을 반환합니다. 'SYSTEM' 판정 → 자동화 우선 시도; 규칙이 없고 게이트 매개변수가 있는 경우 통합 파생을 정직하게 보고합니다. |
| `ipcfg::rule_for_vlnv <dump> <vlnv>`<br>`ipcfg::rule_for_vlnv <덤프> <vlnv>` | Parse a captured `::bd::util_cmd rules dump` to get the Designer-Assistance rule for a cell's VLNV → `RULE:xilinx.com:bd_rule:<short> desc={...}` or `RULE:none` (version-free or full VLNV ok)<br>캡처된 `::bd::util_cmd 규칙 덤프`를 구문 분석하여 셀의 VLNV에 대한 Designer-Assistance 규칙을 가져옵니다 → `RULE:xilinx.com:bd_rule:<short> desc={...}` 또는 `RULE:none`(버전이 없거나 전체 VLNV 가능) |
| `ipcfg::run_automation <cell> <rule> ?config? ?baseline?`<br>`ipcfg::run_automation <셀> <규칙> ?config? ?기준?` | Run a block automation rule and report what it changed: `AUTOMATION:OK changed={..} added_cells={..} default_options={..}`. Checks `<config>` against the rule's live schema first (no mutation): fills omitted keys with GUI defaults; fails `UNKNOWN_AUTOMATION_KEY` / `ILLEGAL_AUTOMATION_VALUE` with the full schema. Also `MISSING_AUTOMATION_KEY`, `NEEDS_BOARD_PRESET`, `PRESET_OVERRIDDEN`<br>블록 자동화 규칙을 실행하고 변경 사항을 보고합니다: `AUTOMATION:OK 변경됨={..} added_cells={..} default_options={..}`. 먼저 규칙의 라이브 스키마에 대해 `<config>`를 확인합니다(변형 없음). 생략된 키를 GUI 기본값으로 채웁니다. 전체 스키마에서 `UNKNOWN_AUTOMATION_KEY`/`ILLEGAL_AUTOMATION_VALUE`에 실패합니다. 또한 `MISSING_AUTOMATION_KEY`, `NEEDS_BOARD_PRESET`, `PRESET_OVERRIDDEN` |
| `ipcfg::automation_rule_options <cell> <rule>`<br>`ipcfg::automation_rule_options <셀> <규칙>` | Preferred: the loaded rule's `get_rule_options` → `RULE_OPTIONS:keys={..} defaults={..} legal={..} schema=..` (values legal in the current context; case-sensitive)<br>선호: 로드된 규칙의 `get_rule_options` → `RULE_OPTIONS:keys={..} defaults={..} legal={..}schema=..`(현재 컨텍스트에서 유효한 값, 대/소문자 구분) |
| `ipcfg::automation_config_schema <rule_file>`<br>`ipcfg::automation_config_schema <규칙_파일>` | Offline fallback: parse rule `.tcl` for keys + defaults → `SCHEMA:keys={..} defaults={k v ..} file=<path>`; may be stale for newer releases<br>오프라인 폴백: 키 + 기본값에 대한 구문 분석 규칙 `.tcl` → `SCHEMA:keys={..} defaults={k v ..} file=<path>`; 최신 릴리스에서는 오래되었을 수 있습니다. |
| `ipcfg::snapshot_all <cell>`<br>`ipcfg::snapshot_all <셀>` | Capture the **entire** `CONFIG.*` dict in one pass (baseline for `config_diff`/`run_automation`)<br>한 번의 패스로 **전체** `CONFIG.*` 딕셔너리 캡처(`config_diff`/`run_automation`의 기준선) |
| `ipcfg::config_diff <cell> <baseline> ?reqmap?`<br>`ipcfg::config_diff <셀> <기준> ?reqmap?` | Full-config diff after an apply. Requested keys → `EXACT/RESOLVED/REVERTED`; other moved keys → `CHANGED` (IP side effect). Returns `{key old new class ...}`<br>적용 후 전체 구성 차이점. 요청된 키 → `EXACT/RESOLVED/REVERTED`; 다른 이동된 키 → `CHANGED`(IP 부작용). `{key old new class ...}`를 반환합니다. |
| `ipcfg::classify_change <want> <base> <cur> ?validset? ?tolerance?`<br>`ipcfg::classify_change <원함> <base> <cur> ?validset? ?관용?` | One-key verdict: `EXACT` / `RESOLVED` (numeric and within the `tolerance` the requirement supplied: `<abs>` or `<n>%`; never without one) / `NEAREST` (a **miss**: the IP snapped to a nearby legal value, reported as a diagnostic) / `REVERTED` (snapped to default/elsewhere); `UNCHANGED`/`CHANGED` when `want=""`<br>단일 키 판정: `EXACT` / `RESOLVED`(숫자 및 `허용치` 내에서 요구 사항 제공: `<abs>` 또는 `<n>%`; 하나 없이는 절대 없음) / `NEAREST`(**miss**: IP가 가까운 합법적인 값으로 스냅되고 진단으로 보고됨) / `REVERTED`(기본값/다른 위치로 스냅됨); `want=""`인 경우 `변경되지 않음`/`변경됨` |
| `ipcfg::discover_params <vlnv> <keyword>`<br>`ipcfg::discover_params <vlnv> <키워드>` | Scoped introspection on a throwaway `create_ip` cell → `DISCOVER:CONFIG.<...> ...` (the **complete** param neighborhood matching a feature keyword). Keyword is synonym-broadened, so a prompt word no longer misses the params. The one sanctioned `list_property` use; maps a requirement to the full sibling set (e.g. `CMN_NUM_LANES` **and** `C_DPHY_LANES`). Returns `DISCOVER_IPI_ONLY:<vlnv>` for IPI-only IPs (2026.1 `ps_wizard`) — switch to `ipcfg::find_params` on the BD cell<br>일회용 `create_ip` 셀에 대한 범위 검사 → `DISCOVER:CONFIG.<...> ...`(기능 키워드와 일치하는 **완전한** 매개변수 환경) 키워드는 동의어 확장이므로 프롬프트 단어에서 더 이상 매개변수를 놓치지 않습니다. 승인된 `list_property` 사용; 요구사항을 전체 형제 세트(예: `CMN_NUM_LANES` **및** `C_DPHY_LANES`)에 매핑합니다. IPI 전용 IP(2026.1 `ps_wizard`)에 대해 `DISCOVER_IPI_ONLY:<vlnv>`를 반환합니다. BD 셀에서 `ipcfg::find_params`로 전환합니다. |
| `ipcfg::native_caps` / `ipcfg::has_native <feature>`<br>`ipcfg::native_caps` / `ipcfg::has_native <기능>` | **Version gate (2026.1+).** Detects, once per session, whether the native VIVADO-23126 commands exist (by **command presence**, not version string — some `2026.1_*` builds report `2025.2.0`). `feature` = `dump_param_deps` \<br>**버전 게이트(2026.1+).** 기본 VIVADO-23126 명령이 존재하는지 여부를 세션당 한 번씩 감지합니다(버전 문자열이 아닌 **명령 존재**로 - 일부 `2026.1_*` 빌드 보고서 `2025.2.0`). `기능` = `dump_param_deps` \| `can_connect` → `0/1`. All native helpers below return `*_NA` when 0 so you fall back to the reactive path<br>`can_connect` → `0/1`. 아래의 모든 기본 도우미는 0일 때 `*_NA`를 반환하므로 반응 경로로 돌아갑니다. |
| `ipcfg::param_deps <target> ?file? ?vlnv? ?tmpdir?`<br>`ipcfg::param_deps <대상> ?file? ?vlnv? ?tmpdir?` | **2026.1+ fast param discovery.** `::debug::dump_param_deps` (resolved Range/Enabled/Disabled + dependency graph) written to a FILE → `PARAM_DEPS:<file>:<bytes> route=<cell\<br>**2026.1+ 빠른 매개변수 검색.** FILE에 기록된 `::debug::dump_param_deps`(해결된 범위/활성화/비활성화 + 종속성 그래프) → `PARAM_DEPS:<file>:<bytes> 경로=<cell\|ip\|scratch> ip=<name>` or `PARAM_DEPS_NA`. For **flat-param IPs** this REPLACES the per-IP `vivado_doc_search` (measured ~50–72 KB) — confirmed ~87% fewer tokens on the rave2 subset. **`target` should be the BD CELL you are configuring**: `dump_param_deps` itself rejects a cell name, but the cell carries its managed IP in `CONFIG.Component_Name` and that IP exists the moment the cell does (no save, no `generate_target`), so `param_deps` resolves it and dumps that (`route=cell`). Falls back to a managed-IP name (`route=ip`), then to building and removing a scratch IP from `vlnv` (`route=scratch`, name sanitised so passing a VLNV here can no longer produce `should contain only A-Z,a-z,0-9 and _`). Works even for IPs flagged IPI-only: `ps_wizard` dumps ~172,6xx B either way because 7-1663 is only a warning. **The cell route is not just cheaper, it is richer where it differs:** measured on 2026.1 `axi_noc2`, both routes give the same 86 parameter blocks, but from the cell the dependency metadata also carries the block-design interface propagation (`S00_AXI.DATA_WIDTH`, `M00_AXI.DATA_WIDTH`, `CLK.*.FREQ_HZ` — present from the cell, absent from a scratch IP), 186 vs 86 referenced `PARAM_VALUE` ids and 90,518 B vs 41,143 B. `PARAM_DEPS_NO_MANAGED_IP:<name>` means "not a BD cell, not a managed IP, and no `vlnv` given" (or the create genuinely produced no object, with the real error quoted) — success is judged by the dump materializing non-empty, never by a return code<br>스크래치> ip=<이름>` 또는 `PARAM_DEPS_NA`. **평면 매개변수 IP**의 경우 이는 IP당 'vivado_doc_search'(~50~72KB 측정)를 대체합니다. — rave2 하위 집합에서 ~87% 더 적은 토큰을 확인했습니다. **`target`은 구성 중인 BD CELL이어야 합니다**: `dump_param_deps` 자체는 셀 이름을 거부하지만 셀은 관리되는 IP를 `CONFIG.Component_Name`에 전달하고 해당 IP는 셀이 수행하는 순간(저장 없음, `generate_target` 없음) 존재하므로 `param_deps`가 이를 확인하고 이를 덤프합니다(`route=cell`). 관리형 IP 이름(`route=ip`)으로 대체한 다음 `vlnv`(`route=scratch`, 이름이 삭제되어 여기에 VLNV를 전달하면 더 이상 A-Z,a-z,0-9 및 _`만 포함해야 함)에서 스크래치 IP를 구축하고 제거합니다. IPI 전용 플래그가 지정된 IP에 대해서도 작동합니다. 7-1663은 경고일 뿐이므로 `ps_wizard`는 어느 쪽이든 ~172,6xx B를 덤프합니다. **셀 경로는 더 저렴할 뿐만 아니라 다른 부분이 더 풍부합니다.** 2026.1 `axi_noc2`에서 측정한 결과 두 경로 모두 동일한 86개의 매개변수 블록을 제공하지만 셀에서 종속성 메타데이터는 블록 설계 인터페이스 전파(`S00_AXI.DATA_WIDTH`, `M00_AXI.DATA_WIDTH`, `CLK.*.FREQ_HZ` — 셀에 있음, 없음)도 전달합니다. 스크래치 IP), 186 대 86 참조 `PARAM_VALUE` ID 및 90,518 B 대 41,143 B. `PARAM_DEPS_NO_MANAGED_IP:<name>`은 "BD 셀도 아니고, 관리되는 IP도 아니고, `vlnv`가 제공되지 않음"을 의미합니다(또는 실제 오류가 인용되어 생성이 실제로 객체를 생성하지 않았습니다) — 성공은 비어 있지 않고 구체화되지 않은 덤프로 판단됩니다. 반환 코드로 |
| `ipcfg::param_block <file> <param>`<br>`ipcfg::param_block <파일> <매개변수>` | **Targeted read** of ONE param's block from a `param_deps` dump (Range/Enabled/Disabled/Default). ALWAYS use this — never inline the whole dump (172 KB ps_wizard … 665 KB axi_noc); a blanket read is ~2× worse than 2025.2<br>`param_deps` 덤프에서 하나의 매개변수 블록에 대한 **대상 읽기**(범위/활성화/비활성화/기본값). 항상 이것을 사용하십시오. 전체 덤프를 인라인하지 마십시오(172KB ps_wizard … 665KB axi_noc). 포괄적인 읽기는 2025.2보다 ~2배 더 나쁩니다. |
| `ipcfg::dump_params <file> ?keyword?`<br>`ipcfg::dump_params <파일> ?키워드?` | **The dump-side `discover_params` — this is what makes the 2026.1 path a REPLACEMENT rather than an extra step.** Lists the user-facing params in a dump (`PARAM_VALUE.*` only, so no MODELPARAM twins), synonym-broadened and stopping at the first spelling that hits, exactly like `discover_params` → `DUMP_PARAMS:CONFIG.C_GPIO2_WIDTH CONFIG.C_GPIO_WIDTH` \<br>**덤프 측 `discover_params` — 이것이 2026.1 경로를 추가 단계가 아닌 교체로 만드는 것입니다.** 덤프에 사용자 대상 매개변수를 나열하고(`PARAM_VALUE.*`만 해당, MODELPARAM 쌍은 없음), 동의어가 확장되고 정확히 `discover_params`와 같이 적중하는 첫 번째 철자에서 중지됩니다 → `DUMP_PARAMS:CONFIG.C_GPIO2_WIDTH CONFIG.C_GPIO_WIDTH` \| `DUMP_PARAMS:none`. No keyword (or `*`) lists everything. Costs no round trip because the dump is already on disk. **When you use this, do not also call `discover_params`/`doc_search` for that IP**<br>`DUMP_PARAMS:없음`. 모든 것을 나열하는 키워드(또는 `*`)는 없습니다. 덤프가 이미 디스크에 있으므로 왕복 비용이 들지 않습니다. **이것을 사용할 때 해당 IP에 대해 `discover_params`/`doc_search`도 호출하지 마세요** |
| `ipcfg::param_range <file> <param>`<br>`ipcfg::param_range <파일> <매개변수>` | The legal values for one param, from the dump: `RANGE_NUM:1:32` (numeric) \<br>덤프의 한 매개변수에 대한 유효한 값: `RANGE_NUM:1:32`(숫자) \| `RANGE_LIST:ACTIVE_HIGH ACTIVE_LOW` (enum — values only, display labels dropped) \<br>`RANGE_LIST:ACTIVE_HIGH ACTIVE_LOW`(열거형 — 값만 표시, 라벨 삭제됨) \| `RANGE_NONE` (no range published, e.g. a plain boolean or free text) \<br>`RANGE_NONE`(범위가 게시되지 않음, 예: 일반 부울 또는 자유 텍스트) \| `PB_MISS:<param>`<br>`PB_MISS:<매개변수>` |
| `ipcfg::check_value <file> <param> <value>`<br>`ipcfg::check_value <파일> <매개변수> <값>` | **Pre-flight a value BEFORE writing it** — removes the failed-write round trip, and catches values Vivado accepts *without* complaint. Returns `VAL_OK` \<br>**값을 쓰기 전에 사전 비행** — 실패한 쓰기 왕복을 제거하고 Vivado가 *불만 없이* 허용하는 값을 포착합니다. `VAL_OK`를 반환합니다. \| `VAL_OUT_OF_RANGE:<p>=<v> legal=<lo>..<hi>` \<br>`VAL_OUT_OF_RANGE:<p>=<v> 법적=<lo>..<hi>` \| `VAL_NOT_IN_LIST:<p>=<v> legal={...}` \<br>`VAL_NOT_IN_LIST:<p>=<v> 법적={...}` \| `VAL_UNCHECKED:<p> (<why>)`. On a list miss it also reports the near-miss: `did_you_mean=ACTIVE_LOW` for `ACTIVELOW`. This is the guard for the silent-typo class — a CIPS request once wrote `PCIe0x2_10GbE` for the legal `PCIe0_x2_10GbE` and Vivado never objected. `VAL_UNCHECKED` is **not** a failure, it means the dump publishes no range to check against<br>`VAL_UNCHECKED:<p> (<이유>)`. 목록 누락에서는 'ACTIVELOW'에 대해 'did_you_mean=ACTIVE_LOW'인 'did_you_mean=ACTIVE_LOW'도 보고합니다. 이는 자동 오타 클래스에 대한 보호 장치입니다. CIPS 요청은 적법한 `PCIe0_x2_10GbE`에 대해 `PCIe0x2_10GbE`를 작성한 적이 있으며 Vivado는 결코 이의를 제기하지 않았습니다. `VAL_UNCHECKED`는 실패가 **아닙니다**. 이는 덤프가 확인할 범위를 게시하지 않음을 의미합니다. |
| `ipcfg::port_enablers <file> <port>`<br>`ipcfg::port_enablers <파일> <포트>` | **2026.1+ AUTHORITATIVE port gating — the ground truth `check_enablers` can only guess by name.** Reads the dependency graph in a `param_deps` dump and returns the settable params that decide whether `<port>` physically exists: `PE:RESETN:INTERFACE_SELECTION RESET_TYPE USE_DYN_RECONFIG USE_RESET`. Ask this **before** the first write and fold the answer into the same `-dict`. Works at interface **and** pin level (`GPIO2_IO_T` → `C_ALL_INPUTS_2 C_ALL_OUTPUTS_2 C_IS_DUAL`). `PE_MISS:<port>` = this IP publishes **no** gating for that port (all CIPS-class IPs) → fall back to `check_enablers`/`feature_flags`; it does **NOT** mean "ungated". Depth-1 by design: a transitive walk returns ~every param on the IP (65/65 on `clkx5_wiz`) and is worthless<br>**2026.1+ 인증된 포트 게이팅 — 실제 `check_enablers`는 이름으로만 추측할 수 있습니다.** `param_deps` 덤프에서 종속성 그래프를 읽고 `<port>`가 물리적으로 존재하는지 결정하는 설정 가능한 매개변수인 `PE:RESETN:INTERFACE_SELECTION RESET_TYPE USE_DYN_RECONFIG USE_RESET`을 반환합니다. 첫 번째 쓰기 전에 **전에** 이것을 물어보고 답을 동일한 `-dict`로 접으세요. 인터페이스 **및** 핀 수준(`GPIO2_IO_T` → `C_ALL_INPUTS_2 C_ALL_OUTPUTS_2 C_IS_DUAL`)에서 작동합니다. `PE_MISS:<포트>` = 이 IP는 해당 포트(모든 CIPS 클래스 IP)에 대한 게이팅 **없음**을 게시합니다. → `check_enablers`/`feature_flags`로 대체됩니다. **NOT**은 "게이트되지 않음"을 의미하지 않습니다. 깊이 1 설계: 전이적 걷기는 IP의 ~모든 매개변수(`clkx5_wiz`의 65/65)를 반환하며 쓸모가 없습니다. |
| `ipcfg::enablement_ports <file> ?glob?`<br>`ipcfg::enablement_ports <파일> ?glob?` | Lists every port/interface whose gating the graph publishes → `PE_PORTS:GPIO2 GPIO2_IO_I ... IP2INTC_IRQ`. Use when you know the behaviour but not how the IP spells the port (`IP2INTC_IRQ` the interface vs `IP2INTC_IRPT` the pin), and to distinguish "IP publishes no gating" (`PE_PORTS:none`) from "this port is ungated"<br>그래프 게이팅이 게시되는 모든 포트/인터페이스를 나열합니다 → `PE_PORTS:GPIO2 GPIO2_IO_I ... IP2INTC_IRQ`. 동작은 알지만 IP가 포트를 어떻게 표기하는지 모르는 경우(`IP2INTC_IRQ` 인터페이스 대 `IP2INTC_IRPT` 핀), "IP 게시 없음 게이팅"(`PE_PORTS:none`)과 "이 포트는 게이트 해제됨"을 구별하기 위해 사용합니다. |
| `ipcfg::param_dict <file> <dictparam> ?subkey?`<br>`ipcfg::param_dict <파일> <dictparam> ?하위 키?` | **Nested-dict introspection** for PS/CIPS-class IPs. `dump_param_deps` exposes the real sub-keys only inside one resolved dict value (`ps_wizard` `PS11_CONFIG_INTERNAL` = a ~1315-key flat Tcl dict). `subkey=""`→`PD_KEYS:<all top-level keys>`; else `PD:<k>=<value>`. Recovers sub-key **names + current/default values** with NO doc search. **Limit:** per-sub-key valid-range/gating is NOT in the dump → use the two set-and-observe helpers below<br>PS/CIPS 클래스 IP에 대한 **중첩 사전 검사**. `dump_param_deps`는 하나의 해결된 dict 값(`ps_wizard` `PS11_CONFIG_INTERNAL` = ~1315 키 플랫 Tcl dict) 내에서만 실제 하위 키를 노출합니다. `subkey=""`→`PD_KEYS:<모든 최상위 키>`; 그렇지 않으면 `PD:<k>=<값>`. 문서 검색 없이 하위 키 **이름 + 현재/기본값**을 복구합니다. **제한:** 하위 키당 유효 범위/게이팅이 덤프에 없습니다 → 아래 두 개의 설정 및 관찰 도우미를 사용하세요. |
| `ipcfg::probe_subkey_range <cell> <dictparam> <subkey> ?sentinel?`<br>`ipcfg::probe_subkey_range <셀> <dictparam> <하위 키> ?sentinel?` | **Per-sub-key RANGE via set-and-observe** (fills the gap `dump_param_deps` leaves). Pushes an out-of-range sentinel for ONE sub-key on a BD cell's dict user-param; the customizer rejects it and **auto-restores** (non-destructive). Returns `PSR_OUT_OF_RANGE:<k>` (validated input — the legal set is in the captured log) \<br>**설정 및 관찰을 통한 하위 키별 범위**(`dump_param_deps`가 남는 간격을 채웁니다.) BD 셀의 dict user-param에 있는 하나의 하위 키에 대해 범위를 벗어난 감시 장치를 푸시합니다. 맞춤설정 도구는 이를 거부하고 **자동 복원**(비파괴)합니다. `PSR_OUT_OF_RANGE:<k>` 반환(검증된 입력 - 합법적인 세트가 캡처된 로그에 있음) \| `PSR_NOTVALIDATED` (free-form input, no range) \<br>`PSR_NOTVALIDATED`(자유 형식 입력, 범위 없음) \| `PSR_NOKEY` (derived/ignored, not user-settable) \<br>`PSR_NOKEY`(파생/무시, 사용자 설정 불가) \| `PSR_ERR`. **Run with `capture_log=true`** — the legal-set text ("`is out of range { ... }`") only reaches the message log, NOT `$e`/`errorInfo`. **Version-independent** (2025.2 + 2026.1). Proven on 2026.1 `ps_wizard` (`PS_USE_PMCPL_CLK0 → { 0,1 }`)<br>'PSR_ERR'. **`capture_log=true`로 실행** — 합법적으로 설정된 텍스트("`is out of range { ... }`")는 `$e`/`errorInfo`가 아닌 메시지 로그에만 도달합니다. **버전 독립적**(2025.2 + 2026.1). 2026.1 `ps_wizard`에서 입증됨(`PS_USE_PMCPL_CLK0 → { 0,1 }`) |
| `ipcfg::range_from_log <console> <subkey>`<br>`ipcfg::range_from_log <콘솔> <하위 키>` | **Parser** for the `probe_subkey_range` companion: scan the `capture_log` payload for the sub-key's range line → `RANGE:<k>={ 0,1 }` \<br>`probe_subkey_range` 컴패니언에 대한 **파서**: 하위 키의 범위 라인에 대한 `capture_log` 페이로드를 스캔 → `RANGE:<k>={ 0,1 }` \| `RANGE_NONE:<k>`. Stateless (same pattern as `find_disabled`/`autofix_apply` taking the console)<br>`범위_없음:<k>`. 상태 비저장(콘솔을 사용하는 `find_disabled`/`autofix_apply`와 동일한 패턴) |
| `ipcfg::resolve_subkeys <cell> <userparam> <override> ?internalparam? ?wantkeys?`<br>`ipcfg::resolve_subkeys <셀> <사용자 매개변수> <재정의> ?internalparam? ?열쇠를 원해?` | **Gating/derivation via set-and-observe.** Applies a legal `{subkey value ...}` override, reads back the resolved `*_INTERNAL` dict → `RSK:<k v ...>` (or `RSK_KEYS:<n>`). Shows what an input derives/gates (proven: req 250 MHz → `ACT_FREQMHZ 249.997 DIVISOR0 4 SRCSEL NPLL BUF 1`). **Applies a real config change** (snapshot `CONFIG.<userparam>` first if rollback needed). Version-independent<br>**설정 및 관찰을 통한 게이팅/파생.** 유효한 `{하위 키 값 ...}` 재정의를 적용하고 해결된 `*_INTERNAL` dict → `RSK:<k v ...>`(또는 `RSK_KEYS:<n>`)를 다시 읽습니다. 입력이 무엇을 파생/게이트하는지 보여줍니다(검증됨: req 250 MHz → `ACT_FREQMHZ 249.997 DIVISOR0 4 SRCSEL NPLL BUF 1`). **실제 구성 변경 사항을 적용합니다**(롤백이 필요한 경우 'CONFIG.<userparam>' 스냅샷을 먼저 적용). 버전 독립적 |
| `ipcfg::coverage_report <ledger>`<br>`ipcfg::coverage_report <원장>` | Build the REQUIRED partial-coverage disclosure from the requirement ledger (`COVERAGE:FULL<br>요구 사항 원장(`COVERAGE:FULL)에서 필수 부분 적용 범위 공개를 작성합니다.|PARTIAL` + unapplied prompt parts)<br>PARTIAL` + 미적용 프롬프트 부분) |
| `ipcfg::snapshot <cell> <keys>`<br>`ipcfg::스냅샷 <셀> <키>` | Capture **baseline default** values BEFORE config (call right after create); returns `{key val ...}` for `verify_intent`<br>구성 전 **기준 기본값** 값을 캡처합니다(생성 직후 호출). `verify_intent`에 대해 `{key val ...}`을 반환합니다. |
| `ipcfg::verify_stuck <cell> <dict>`<br>`ipcfg::verify_stuck <셀> <dict>` | Returns the list of keys that did **not** take effect (missing **or** value-reverted; handles nested `*_CONFIG` dicts). Empty list = all stuck<br>적용되지 **않은** 키 목록을 반환합니다(누락 **또는** 값 복원, 중첩된 `*_CONFIG` 사전 처리). 빈 목록 = 모두 붙어 있음 |
| `ipcfg::check_enablers <cell> <dict>`<br>`ipcfg::check_enablers <셀> <dict>` | **The check `verify_stuck` cannot do.** Finds values that persisted while their enabling flag is still off, so the write is **inert** (`CONFIG.RESET_TYPE ACTIVE_LOW` with `USE_RESET false` reads back fine and produces no reset pin). Returns `INERT:<key>:enabler=<CONFIG.flag>=<value> ...` or `""`. Matched against **this cell's own** parameter names, case-insensitively (`USE_/ENABLE_/EN_/HAS_/INCLUDE_/IS_` and `C_`-prefixed forms, plus `<X>_ENABLE/_EN/_USED`); a flag you set truthily in the same dict is not reported. **This is a NAME heuristic — it can only find enablers that happen to be spelled like one.** On 2026.1 prefer `ipcfg::port_enablers`, which reads the actual dependency graph; keep this as the cross-check and as the only option on 2025.2<br>**`verify_stuck` 검사는 수행할 수 없습니다.** 활성화 플래그가 아직 꺼져 있는 동안 지속되는 값을 찾기 때문에 쓰기는 **비활성**입니다(`USE_RESET false`를 사용하는 `CONFIG.RESET_TYPE ACTIVE_LOW`는 다시 잘 읽고 재설정 핀을 생성하지 않습니다). `INERT:<key>:enabler=<CONFIG.flag>=<value> ...` 또는 `""`를 반환합니다. **이 셀 자체** 매개변수 이름과 일치하며 대소문자를 구분하지 않습니다(`USE_/ENABLE_/EN_/HAS_/INCLUDE_/IS_` 및 `C_` 접두어 형식 및 `<X>_ENABLE/_EN/_USED`). 동일한 사전에 진실하게 설정한 플래그는 보고되지 않습니다. **이것은 NAME 휴리스틱입니다. 철자가 1과 같은 인에이블러만 찾을 수 있습니다.** 2026.1에서는 실제 종속성 그래프를 읽는 `ipcfg::port_enablers`를 선호합니다. 이것을 교차 확인으로 유지하고 2025.2의 유일한 옵션으로 유지하십시오. |
| `ipcfg::feature_flags <cell> {<word> ...}`<br>`ipcfg::feature_flags <셀> {<단어> ...}` | Intent-side companion: given behavioural nouns from the prompt (`reset locked interrupt sg`), returns the cell's boolean params that enable them with current values — `FLAGS: reset:CONFIG.USE_RESET=false locked:CONFIG.USE_LOCKED=false` or `FLAGS:none`. Use for requests whose whole content **is** the flag ("expose the locked signal"), which name no attribute for `check_enablers` to catch<br>인텐트 측 컴패니언: 프롬프트('reset 잠긴 인터럽트 sg')의 동작 명사가 주어지면 현재 값(`FLAGS: 재설정:CONFIG.USE_RESET=false 잠김:CONFIG.USE_LOCKED=false` 또는 `FLAGS:none`)으로 활성화하는 셀의 부울 매개변수를 반환합니다. 전체 내용이 플래그("잠긴 신호 노출")인 요청에 사용합니다. 이 플래그에는 `check_enablers`가 포착할 속성이 없습니다. |
| `ipcfg::verify_intent <cell> <reqmap> ?flag_no_change? ?tolerances?`<br>`ipcfg::verify_intent <셀> <reqmap> ?flag_no_change? ?공차?` | Intent audit: given `{key want baseline ...}`, flags keys that **reverted**, landed on a **nearest-legal** value, or are **suspect-no-change** (value == default). Acceptance is exact; a difference passes only within a tolerance given for that key in `tolerances` (`{key <abs>\<br>인텐트 감사: `{키가 기준선을 원합니다 ...}`가 주어지면 **되돌린** 키, **가장 합법적인** 값에 도달한 키 또는 **의심스러운 변경 없음**(값 == 기본값)인 키에 플래그를 지정합니다. 수락은 정확합니다. 차이는 `tolerances`(`{key <abs>\)의 해당 키에 대해 지정된 허용 오차 내에서만 통과됩니다.|<n>% ...}`). Pass `flag_no_change=0` to return only hard misses once doc grounding confirms a default legitimately satisfies intent (`value_src=default`). Empty list = intent realized<br><n>% ...}`). 문서 기반이 기본값이 합법적으로 의도를 충족하는지 확인하면(`value_src=default`) 하드 미스만 반환하려면 `flag_no_change=0`을 전달합니다. 빈 목록 = 의도 실현 |
| `ipcfg::audit_intent <cell> <reqmap> ?tolerances?`<br>`ipcfg::audit_intent <셀> <reqmap> ?공차?` | Reports buckets: `EXACT:{...} RESOLVED:{k(=cur,want=w,tolerance=t) ...} BAD:{...}`; `RESOLVED` exists only for keys given a tolerance<br>보고서 버킷: `정확함:{...} 해결됨:{k(=cur,want=w,tolerance=t) ...} 불량:{...}`; '해결됨'은 허용 오차가 지정된 키에만 존재합니다. |
| `ipcfg::find_disabled <output>`<br>`ipcfg::find_disabled <출력>` | Scans raw execute output for non-fatal `disabled parameter`/`[BD 41-721]` warnings (gated params that `catch` returns 0 for). Returns `DISABLED:...` or `""`<br>치명적이지 않은 `disabled 매개변수`/`[BD 41-721]` 경고(`catch`가 0을 반환하는 게이트 매개변수)에 대한 원시 실행 출력을 검색합니다. `DISABLED:...` 또는 `""`를 반환합니다. |
| `ipcfg::param_format <cell> <key>`<br>`ipcfg::param_format <셀> <키>` | Reads a key back so you can learn its **shape** (comma list, nested `{{...}}`, etc.)<br>키를 다시 읽어 **모양**(쉼표 목록, 중첩된 `{{...}}` 등)을 학습할 수 있습니다. |
| `ipcfg::try_automation <cell> <rule> ?config?`<br>`ipcfg::try_automation <셀> <규칙> ?config?` | Generic `apply_bd_automation -rule <rule>` (Designer Assistance)<br>일반 `apply_bd_automation -rule <rule>`(디자이너 지원) |
| `ipcfg::add_stub <cell> <pin> <intf_vlnv> ?prop? ?val?`<br>`ipcfg::add_stub <셀> <핀> <intf_vlnv> ?prop? ?발?` | Boundary interface port for connection-derived params; mode auto-mirrors the pin<br>연결 파생 매개변수를 위한 경계 인터페이스 포트. 모드는 핀을 자동으로 미러링합니다. |
| `ipcfg::cleanup <cell> ?orig_part? ?force?`<br>`ipcfg::cleanup <셀> ?orig_part? ?힘?` | Deletes `STUB_*` ports, a **scratch** cell, and restores the part if given. Returns `SKIP:KEPT` instead of deleting a cell that holds pinned requirements — that cell is the deliverable; `force=1` overrides<br>`STUB_*` 포트, **스크래치** 셀을 삭제하고, 제공된 경우 해당 부분을 복원합니다. 고정된 요구 사항이 있는 셀을 삭제하는 대신 'SKIP:KEPT'를 반환합니다. 해당 셀이 결과물입니다. `force=1` 재정의 |
| `ipcfg::cache_get <ip> <feature>`<br>`ipcfg::cache_get <ip> <기능>` | **Consult first** (0 MCP calls): returns the earned discovery entry (param/shape/enabler/value_src/doc/version) as JSON, or `""` on miss<br>**먼저 문의**(0 MCP 호출): 획득한 검색 항목(param/shape/enabler/value_src/doc/version)을 JSON으로 반환하거나 누락 시 `""`를 반환합니다. |
| `ipcfg::cache_put <ip> <feature> <param> <shape> <enabler> <value_src> <doc> <ipver>`<br>`ipcfg::cache_put <ip> <feature> <param> <shape> <enabler> <value_src> <doc> <ipver>` | **Write back** a confirmed mapping. Stores discovery facts only — **never** the prompt's expected value (blind-safe)<br>확인된 매핑을 **다시 작성**하세요. 검색 사실만 저장 — 프롬프트의 예상 값 **절대로** 저장하지 않음(블라인드-세이프) |
| `ipcfg::cache_dump`<br>`ipcfg::캐시_덤프` | Print the whole learned cache (debug)<br>학습된 캐시 전체를 인쇄합니다(디버그). |

**Concise per-prompt template** (replaces the verbose phase-by-phase script — the phases
**간결한 프롬프트별 템플릿**(상세한 단계별 스크립트를 대체합니다.
below are still the mental model, but the helpers collapse each to one line):
아래는 여전히 멘탈 모델이지만 헬퍼는 각각 한 줄로 축소됩니다.

```tcl
set cell <cell_name>
set orig [get_property PART [current_project]]
# Phase 0+1: MANDATORY availability gate over the candidate shortlist (Step 0a'), then swap
# ONLY on WRONG_PART. Never substitute a different IP because it happens to be AVAILABLE.
set av [ipcfg::ip_availability {<function-glob e.g. *clk*wiz*>}]
puts $av
if {[string match "*<name>=ABSENT:*" $av]} { puts "CONFIGURE_FAIL:VLNV_NOT_FOUND"; return }
if {[string match "*<name>=WRONG_PART:*" $av]} {
    # confirm the target BEFORE the destructive swap, then swap (guarded -- see Step 0)
    puts [ipcfg::ip_availability {<name>} <required_part>]
    puts [ipcfg::ensure_part <required_part>]   ;# may return WARN:SWAP_BLOCKED -> confirm in interactive mode
}
# Phase 1.5 (2026.1+, FREE once you already have a param_deps dump): ask the
# dependency graph which params decide whether the port the prompt wants EXISTS,
# and fold them into $d BEFORE the first write. This is what turns the inert-write
# class from a Tier-2 recovery into a Tier-1 pass. PE_MISS = this IP publishes no
# gating for that port -- fall through to the name heuristics, do NOT read it as
# "ungated". (See Step 1 for the dump; skip this block when there is no dump.)
if {[info exists pd_file]} {
    puts [ipcfg::enablement_ports $pd_file]            ;# how does this IP spell the port?
    puts [ipcfg::port_enablers $pd_file <PORT_NAME>]   ;# -> PE:<port>:<params to set>
}
# Phase 2+3 in ONE call: create + parameterize (snapshot defaults first for the intent audit)
set d [list CONFIG.PARAM_A {a} CONFIG.PARAM_B {b}]
puts "APPLY:[ipcfg::create_cell_cfg <vendor:library:name> $cell $d]"
set base [ipcfg::snapshot $cell [dict keys $d]]   ;# (defaults still readable for unset keys)
set bad [ipcfg::verify_stuck $cell $d]
# Build reqmap {key want baseline ...} for the "enable/select" requirements:
set req {}
foreach {k v} $d { lappend req $k $v [dict get $base $k] }
set suspect [ipcfg::verify_intent $cell $req]
# A value can stick, match intent, and still do nothing because its feature is off:
set inert [ipcfg::check_enablers $cell $d]
# For every signal/feature the prompt asks to have or expose, check its flag directly:
puts [ipcfg::feature_flags $cell {<nouns from the prompt: reset locked interrupt ...>}]
if {[llength $bad] == 0 && [llength $suspect] == 0 && $inert eq ""} {
    puts "SUCCESS: $cell configured"
} elseif {[llength $bad] != 0} {
    puts "STUCK_FAIL:$bad"
} elseif {$inert ne ""} {
    puts "INERT_FAIL:$inert"   ;# set the named enabler and re-apply
} else {
    puts "INTENT_FAIL:$suspect"
}
# Finally, emit coverage from the requirement ledger (REQUIRED when not fully applied):
# ledger = {"<prompt phrase>" CONFIG.PARAM_A applied  "<prompt phrase>" CONFIG.PARAM_B unapplied:integration-derived ...}
puts [ipcfg::coverage_report $ledger]
```

Note: `create_cell_cfg` is the single-call fast path.
참고: `create_cell_cfg`는 단일 호출 빠른 경로입니다.
When you need ordered/multi-pass
주문/다중패스가 필요한 경우
configuration (enablers first — see Phase 3), create with the enabler dict, then use
구성(인에이블러 먼저 - 3단계 참조), 인에이블러 dict로 생성한 다음 사용
`ipcfg::apply_dict` for the dependent pass.
종속 패스의 경우 `ipcfg::apply_dict`입니다.

`verify_stuck` is strictly stronger than the old existence-only loop: it catches the IPI
`verify_stuck`은 기존 존재 전용 루프보다 엄격하게 강력합니다. IPI를 포착합니다.
**silent revert** case where a key exists but the value snapped back (e.g. a peripheral
**자동 되돌리기** 키가 존재하지만 값이 다시 스냅되는 경우(예: 주변 장치)
`ENABLE` that reverts to `0`), which a `get_property` existence check misses.
'0'으로 되돌아가는 'ENABLE'), 'get_property' 존재 확인이 누락되었습니다.
You still scan
아직도 스캔 중이신가요?
the raw output for warning patterns as a backstop, but `STUCK_FAIL:` is the primary signal.
경고 패턴에 대한 원시 출력은 백스톱으로 사용되지만 'STUCK_FAIL:'이 기본 신호입니다.

**Intent audit (the non-error failure mode) — required.** `verify_stuck` only proves the
**의도 감사(오류가 아닌 실패 모드) - 필수입니다.** `verify_stuck`은
params *you chose* persisted; it cannot tell whether you chose the *right* param for a named
*당신이 선택한* 매개변수는 지속됩니다; 이름이 지정된 매개변수에 *올바른* 매개변수를 선택했는지 여부는 알 수 없습니다.
feature.
특징.
The most dangerous benchmark failure is a **wrong-but-valid** param: it exists, it
가장 위험한 벤치마크 실패는 **잘못되었지만 유효한** 매개변수입니다.
accepts your value, it sticks — no error, no `STUCK_FAIL`, no warning — so error-driven
귀하의 가치를 받아들이면 그대로 유지됩니다. 오류도 없고 `STUCK_FAIL`도 없고 경고도 없습니다. 따라서 오류가 발생합니다.
Tier 2 never fires and the prompt silently scores as a false success.
Tier 2는 실행되지 않으며 프롬프트는 자동으로 거짓 성공으로 평가됩니다.
To catch this without
이것을 잡지 않고 잡으려면
any IP-specific knowledge, `snapshot` the baseline *before* configuring, then run
IP 관련 지식, 구성 *전* 기준선 '스냅샷'을 실행한 다음 실행합니다.
`verify_intent` *after*:
`verify_intent` *후*:

- A requirement that means **enable/select/turn-on** must produce an **observable change
- **활성화/선택/켜기**를 의미하는 요구 사항은 **관찰 가능한 변경을 생성해야 합니다.
from the default**.
기본값에서 **.
If the param you set was *already* at the requested value by default
설정한 매개변수가 기본적으로 요청된 값에 *이미* 있는 경우
(`suspect-no-change`), you almost certainly set the wrong param — the param that truly
(`suspect-no-change`), 잘못된 매개변수를 설정한 것이 거의 확실합니다.
implements the feature is still sitting at its default.
기능을 구현하면 여전히 기본값으로 유지됩니다.
- A non-empty `verify_intent` list is a **new Tier-2 trigger** (`INTENT_FAIL`) even though
- 비어 있지 않은 `verify_intent` 목록은 **새로운 Tier-2 트리거**(`INTENT_FAIL`)입니다.
nothing errored: go back to Step 2, re-read the doc descriptions for the suspect
오류 없음: 2단계로 돌아가서 피의자에 대한 문서 설명을 다시 읽으세요.
requirement, remap to the correctly-grounded param, and re-apply.
요구 사항을 충족하려면 올바르게 기반된 매개변수에 다시 매핑하고 다시 적용하세요.
- Cross-check against the Step 1.5 ledger: every requirement must be observably realized,
- 1.5단계 원장을 대조하여 확인합니다. 모든 요구사항은 관찰 가능하게 실현되어야 합니다.
not merely "set without error."
단순히 "오류 없이 설정"하는 것이 아닙니다.

**Closing re-assert pass (after the LAST write on the cell) — required.** Every check
**재확인 통과(셀에 마지막으로 쓴 후) — 필수입니다.** 모든 확인
above compares the design against the dict in front of it, at the moment that dict is
위의 디자인은 앞에 있는 dict와 디자인을 비교합니다. 현재 dict는 다음과 같습니다.
applied.
적용되었습니다.
None of them runs again.
그들 중 누구도 다시 실행되지 않습니다.
On an IP where one parameter is derived from others,
하나의 매개변수가 다른 매개변수에서 파생되는 IP에서는
a later write recomputes an earlier one, and the transcript stays green all the way to
나중 쓰기는 이전 쓰기를 다시 계산하고 성적표는 계속 녹색으로 유지됩니다.
the report:
보고서:

```tcl
# after the final apply on this cell, before the audits
puts [ipcfg::reassert $cell {CONFIG.NUM_SI 0 CONFIG.NUM_MI 0 CONFIG.NUM_NSI 2 CONFIG.NUM_CLKS 0}]
puts [ipcfg::audit_requirements $cell]
```

`reassert` re-reads everything pinned with `ipcfg::require`, plus the structural keys you
`reassert`는 `ipcfg::require`로 고정된 모든 항목과 구조적 키를 다시 읽습니다.
pass explicitly, re-applies whatever moved, and confirms the repair with `verify_stuck`.
명시적으로 전달하고 이동한 내용을 다시 적용한 다음 'verify_stuck'으로 복구를 확인합니다.
Pass **port and clock counts** explicitly whether or not you pinned them — they are what
고정 여부에 관계없이 **포트 및 클럭 수**를 명시적으로 전달합니다.
the rest of the IP derives from.
나머지 IP는 다음에서 파생됩니다.
A value that differs from the pinned one is drift unless
고정된 값과 다른 값은 다음을 제외하고 드리프트입니다.
the requirement carries a tolerance that covers it.
요구 사항에는 이를 포괄하는 허용 오차가 포함됩니다.

`REASSERT_FAIL` means the value will not hold while a later write keeps re-deriving it:
`REASSERT_FAIL`은 나중에 쓰기가 계속해서 값을 다시 파생하는 동안 값이 유지되지 않음을 의미합니다.
satisfy that write's precondition first and re-run the sequence in order.
해당 쓰기의 전제 조건을 먼저 충족하고 순서대로 시퀀스를 다시 실행합니다.
Measured on
측정 날짜
`axi_noc2` under an LPDDR5 MC brief: `NUM_CLKS` was set to 0 with the AXI ports already
LPDDR5 MC 개요의 `axi_noc2`: `NUM_CLKS`는 이미 AXI 포트에서 0으로 설정되었습니다.
at 0 and verified there, then the `MC_GUI` and `DDRMC5_CONFIG` writes put it back to 1,
0에서 확인하고 'MC_GUI' 및 'DDRMC5_CONFIG'를 쓰면 다시 1로 돌아갑니다.
and the run reported a clean configuration for a design that had an AXI clock the brief
실행 결과 AXI 시계가 포함된 디자인에 대한 깔끔한 구성이 보고되었습니다.
had explicitly excluded.
명시적으로 제외했습니다.

The same pass answers the three questions a read-back that *matches* still leaves open.
동일한 패스는 *일치*하는 다시 읽기가 아직 열려 있는 세 가지 질문에 답합니다.

`REASSERT_UNWRITTEN` — the value is right and `CONFIG.<K>.VALUE_SRC` is still `DEFAULT`,
`REASSERT_UNWRITTEN` — 값이 정확하고 `CONFIG.<K>.VALUE_SRC`가 여전히 `DEFAULT`입니다.
so the IP is holding it up, not you.
그래서 IP가 그것을 붙잡고 있는 것이지 당신이 아닙니다.
Almost always the write never ran: a Tcl error
거의 항상 쓰기가 실행되지 않았습니다. Tcl 오류
anywhere in a semicolon-chained command kills every statement after it, and a proc called
세미콜론으로 묶인 명령의 어느 곳에서든 그 뒤의 모든 명령문이 종료되고 proc이 호출됩니다.
with the wrong argument shape raises exactly that error.
잘못된 인수 모양을 사용하면 정확히 해당 오류가 발생합니다.
Measured on `emb_mem_gen`: the
`emb_mem_gen`에서 측정됨:
`set_property -dict` after a bad `ipcfg::require` call never executed, the cell kept its
잘못된 `ipcfg::require` 호출이 실행되지 않은 후 `set_property -dict`를 실행하면 셀이 해당 값을 유지합니다.
defaults, and `MEMORY_PRIMITIVE` still read `BRAM` — which is the default — so the
기본값이고 `MEMORY_PRIMITIVE`는 여전히 `BRAM`을 읽습니다(기본값).
read-back agreed with the brief on a cell nothing had configured.
다시 읽어보니 아무것도 구성하지 않은 셀에 대한 개요에 동의했습니다.
`reassert` re-writes
'reassert'를 사용하면 다시 작성됩니다.
these so the value is yours (and so it survives an IP version whose default moves), but
따라서 값은 귀하의 것입니다(따라서 기본 이동 IP 버전에서도 유지됩니다).
the real repair is upstream: find the command that failed, because every other key it
실제 수리는 업스트림입니다. 실패한 명령을 찾으세요. 왜냐하면 다른 모든 키는 실패했기 때문입니다.
carried is missing too.
운반된 것도 없어졌습니다.

`REASSERT_INERT` — the attribute stuck and the feature gating it is off, so the IP never
`REASSERT_INERT` — 속성이 고착되고 이를 제어하는 기능이 꺼져 있으므로 IP는 절대로
grows the port.
항구를 성장시킵니다.
Measured on `clkx5_wiz`: `RESET_TYPE=ACTIVE_LOW` was written and reads
`clkx5_wiz`에서 측정됨: `RESET_TYPE=ACTIVE_LOW`가 작성되고 읽혔습니다.
back exactly, `USE_RESET` defaults to **false** and was never written, and the delivered
정확히 말하면 `USE_RESET`의 기본값은 **false**이며 작성된 적이 없으며 전달되었습니다.
cell had pins `clk_in1 clk_out1..3` and no reset input at all under a brief that asked for
셀에는 'clk_in1 clk_out1..3' 핀이 있고 요청한 브리핑에서 재설정 입력이 전혀 없습니다.
an active-low reset.
액티브 로우 리셋.
Set the enabler first, re-write the attribute, re-run the pass.
먼저 Enabler를 설정하고 속성을 다시 작성한 후 패스를 다시 실행하십시오.

`REASSERT_FORCED` — the value is right, `VALUE_SRC` says `DEFAULT`, and unlike
`REASSERT_FORCED` — 값이 정확하고 `VALUE_SRC`에 `DEFAULT`가 표시됩니다.
`REASSERT_UNWRITTEN` the re-write does **not** take: the IP is holding the parameter and
`REASSERT_UNWRITTEN` 다시 쓰기는 수행되지 **않습니다**: IP가 매개변수를 보유하고 있으며
ignoring you.
당신을 무시합니다.
Measured on the same `clkx5_wiz`, and it is the *opposite* end of the
동일한 `clkx5_wiz`에서 측정되었으며, 이는 *반대* 끝입니다.
`REASSERT_INERT` case above: once `CLKOUT_DRIVES` names an `MBUFGCE`, the wizard forces
위의 `REASSERT_INERT` 사례: `CLKOUT_DRIVES`가 `MBUFGCE`의 이름을 지정하면 마법사는 강제로
`USE_RESET` true and marks the parameter DISABLED, so a write comes back
`USE_RESET`이 true이고 매개변수가 DISABLED로 표시되므로 쓰기가 다시 수행됩니다.
`BD 41-721 ... is ignored`, `VALUE_SRC` never leaves `DEFAULT` — and the `resetn` pin the
`BD 41-721 ...은 무시됩니다`, `VALUE_SRC`는 `DEFAULT`를 벗어나지 않습니다. — 그리고 `resetn`은
brief asked for is on the cell regardless.
요청한 간략한 내용은 관계없이 휴대전화에 있습니다.
This is a correct design, so do not chase it:
이것은 올바른 디자인이므로 쫓지 마십시오.
report the value, say it is IP-forced, and move on.
값을 보고하고 IP 강제라고 말하고 계속 진행합니다.
Trying to "fix" the provenance here
여기에서 출처를 "수정"하려고 합니다.
costs a round-trip per attempt and cannot succeed.
시도당 왕복 비용이 발생하며 성공할 수 없습니다.
Note which enabler forced it, because
어떤 활성화 프로그램이 이를 강제했는지 기록해 두세요.
that is the evidence the value is deliberate rather than accidentally default.
이는 그 가치가 실수로 불이행된 것이 아니라 고의로 이루어졌다는 증거입니다.

The remainder of Step 3 documents the phases the helpers implement, plus the
3단계의 나머지 부분에서는 도우미가 구현하는 단계와
error/format/automation/stub recovery patterns.
오류/형식/자동화/스텁 복구 패턴.
Use the helpers; drop to raw TCL only for an
도우미를 사용하십시오. 에 대해서만 원시 TCL로 드롭
IP-specific quirk a helper does not cover.
도우미가 다루지 않는 IP 관련 문제입니다.

### Step 3: Build and execute the -dict
### 3단계: -dict 빌드 및 실행

Compose a SINGLE TCL script with four phases: part swap (if needed), VLNV validation,
부품 교체(필요한 경우), VLNV 검증,
cell creation, and configuration.
셀 생성 및 구성.
Use `catch` around every mutating command so errors
모든 변경 명령 주위에 `catch`를 사용하여 오류가 발생하도록 하세요.
produce parseable output instead of aborting the script.
스크립트를 중단하는 대신 구문 분석 가능한 출력을 생성합니다.

```tcl
# /tmp/configure_<ip_name>.tcl

# --- Phase 0: Part swap (only if IP needs a different device family) ---
# AI: set _need_swap to 1 and _target_part to the required part when
# the doc search or fallback map indicates a part change is needed.
# Omit this entire block when no swap is needed.
set _orig_part [get_property PART [current_project]]
set _target_part "<required_part>"
set _need_swap [expr {$_orig_part ne $_target_part}]
if {$_need_swap} {
    close_bd_design [current_bd_design]
    set_property PART $_target_part [current_project]
    open_bd_design [get_files benchmark_bd.bd]
    puts "PART_SWAP:$_orig_part->$_target_part"
}

# --- Phase 1: VLNV pre-validation ---
set _vlnv_matches [get_ipdefs -filter {VLNV =~ "<vendor:library:name>:*"}]
if {$_vlnv_matches eq ""} {
    if {$_need_swap} {
        close_bd_design [current_bd_design]
        set_property PART $_orig_part [current_project]
        open_bd_design [get_files benchmark_bd.bd]
    }
    puts "CONFIGURE_FAIL:VLNV_NOT_FOUND:<vendor:library:name> not available for $_target_part"
    return
}

# --- Phase 2: Create cell ---
if {[llength [get_bd_cells <cell_name>]] == 0} {
    if {[catch {create_bd_cell -type ip -vlnv <vendor:library:name> <cell_name>} _err]} {
        if {$_need_swap} {
            close_bd_design [current_bd_design]
            set_property PART $_orig_part [current_project]
            open_bd_design [get_files benchmark_bd.bd]
        }
        puts "CONFIGURE_FAIL:CREATE_ERROR:$_err"
        return
    }
}

# --- Phase 3: Apply configuration with structured error handling ---
set _prop_dict [list \
    CONFIG.PARAM_A {value_a} \
    CONFIG.PARAM_B {value_b} \
    CONFIG.PARAM_C {value_c} \
]
# Empty-dict guard: set_property -dict {} throws "Missing name/value pair".
# For "default settings" prompts where intent == defaults, skip configuration.
if {[llength $_prop_dict] == 0} {
    puts "SUCCESS: <cell_name> created with default configuration"
    return
}
if {[catch {set_property -dict $_prop_dict [get_bd_cells <cell_name>]} _cfg_err]} {
    set _err_type "UNKNOWN"
    if {[string match {*does not exist*} $_cfg_err]}      {set _err_type "PARAM_NOT_FOUND"}
    if {[string match {*is out of the range*} $_cfg_err]}  {set _err_type "VALUE_OUT_OF_RANGE"}
    if {[string match {*read-only*} $_cfg_err]}            {set _err_type "READ_ONLY"}
    if {[string match {*It is read-only*} $_cfg_err]}      {set _err_type "READ_ONLY"}
    if {[string match {*not supported*} $_cfg_err]}        {set _err_type "NOT_SUPPORTED"}
    puts "CONFIGURE_FAIL:${_err_type}:$_cfg_err"
    return
}

# --- Phase 3 verification: catch SILENTLY-IGNORED params (IPI CRITICAL WARNINGs) ---
# In BD mode an unknown CONFIG key emits a non-fatal CRITICAL WARNING and catch returns 0,
# so the dict above can "succeed" without applying. Verify every key actually exists on the
# cell. get_property on a specific, already-set key is verification (not parameter discovery),
# so it does not violate the "no list_property/report_property" rule.
set _missing {}
foreach {_k _v} $_prop_dict {
    if {[catch {get_property $_k [get_bd_cells <cell_name>]}]} {
        lappend _missing $_k
    }
}
if {[llength $_missing] > 0} {
    puts "CONFIGURE_FAIL:PARAM_NOT_FOUND:silently ignored (do not exist): $_missing"
    return
}

puts "SUCCESS: <cell_name> configured"
```

**Output-scan backstop:** Even with the verification loop above, after `vivado_execute`
**출력 스캔 백스톱:** 위의 확인 루프를 사용하더라도 'vivado_execute' 이후
returns, scan its full stdout for `Cannot set the parameter`, `does not exist`,
반환되면 '매개변수를 설정할 수 없습니다', '존재하지 않습니다'에 대한 전체 표준 출력을 검색합니다.
`Invalid parameter`, or `Ignoring`.
`잘못된 매개변수` 또는 `무시`.
If any appear, treat the result as
나타나는 경우 결과를 다음과 같이 처리하십시오.
`CONFIGURE_FAIL:PARAM_NOT_FOUND` regardless of whether a `SUCCESS:` line was printed — the
`SUCCESS:` 줄이 인쇄되었는지 여부에 관계없이 `CONFIGURE_FAIL:PARAM_NOT_FOUND` —
warning may reference a parameter the loop could not catch (e.g. a key inside a nested dict).
경고는 루프가 포착할 수 없는 매개변수(예: 중첩된 사전 내부의 키)를 참조할 수 있습니다.

**Nested-dict params:** When a CONFIG value is itself a dictionary (e.g.
**Nested-dict 매개변수:** CONFIG 값 자체가 사전인 경우(예:
`CONFIG.CPM_CONFIG {...}`, `CONFIG.PS_PMC_CONFIG {...}`, or a transceiver-wizard config),
`CONFIG.CPM_CONFIG {...}`, `CONFIG.PS_PMC_CONFIG {...}` 또는 트랜시버 마법사 구성),
invalid *sub-keys* are silently dropped with `[IP_Flow 19-7090] ...
잘못된 *하위 키*는 `[IP_Flow 19-7090] ...을 사용하여 자동으로 삭제됩니다.
Ignoring`.
무시`.
The
는
top-level `get_property` check above will pass even though sub-keys were ignored.
하위 키가 무시되더라도 위의 최상위 `get_property` 검사는 통과됩니다.
`ipcfg::verify_stuck` already inspects `*_CONFIG` keys sub-key by sub-key, so a non-empty
`ipcfg::verify_stuck`은 이미 `*_CONFIG` 키를 하위 키별로 검사하므로 비어 있지 않은
`STUCK_FAIL` list names the exact sub-keys that did not apply.
`STUCK_FAIL` 목록은 적용되지 않은 정확한 하위 키의 이름을 지정합니다.

**STUCK recovery — value-format introspection (generic, no per-IP table):** The most common
**STUCK 복구 — 값 형식 검사(일반, IP별 테이블 없음):** 가장 일반적인
reason a key reverts (rather than erroring) is that your **value shape** does not match what
오류가 아닌 키가 되돌아가는 이유는 **값 형태**가 무엇과 일치하지 않기 때문입니다.
the IP expects.
IP가 기대합니다.
This is fully recoverable without any IP-specific knowledge:
이는 IP 관련 지식 없이도 완전히 복구 가능합니다.

1. Read the current value back with `ipcfg::param_format <cell> <key>` to learn the shape.
1. `ipcfg::param_format <cell> <key>`로 현재 값을 다시 읽어 모양을 알아보세요.
2. **Mirror that shape**, substituting your target value:
2. **그 모양을 미러링**하여 목표 값을 대체합니다.
   - **Comma-list params** (e.g.
   - **쉼표 목록 매개변수**(예:
   - Versal `clk_wizard`): `CLKOUT_USED` reads back as
   - Versal `clk_wizard`): `CLKOUT_USED`는 다음과 같이 다시 읽습니다.
`true,false,false,...` and `CLKOUT_REQUESTED_OUT_FREQUENCY` as a comma list — supply one
`true,false,false,...` 및 `CLKOUT_REQUESTED_OUT_FREQUENCY`를 쉼표 목록으로 사용 — 하나 제공
comma-separated string per the read-back length, **not** per-clock keys.
시계별 키가 **아님** 다시 읽기 길이별로 쉼표로 구분된 문자열입니다.
   - **Nested list-of-pairs sub-keys** (e.g.
   - **중첩된 쌍 목록 하위 키**(예:
   - Versal `versal_cips` peripherals): a peripheral
   - Versal `versal_cips` 주변 장치): 주변 장치
sub-key reads back as `{{ENABLE 0} {IO {PS_MIO 0 ..
하위 키는 `{{ENABLE 0} {IO {PS_MIO 0 ..
1}}}`.
To enable it, re-send the
활성화하려면
**same structure** with `ENABLE 1` (keep the read-back `IO {...}` routing).
**동일한 구조**와 `ENABLE 1`(다시 읽기 `IO {...}` 라우팅 유지).
Sending a
보내는 중
flat `{ENABLE 1}` is silently rejected.
플랫 `{ENABLE 1}`은 자동으로 거부됩니다.
   - **Scalar mismatch**: just supply the documented value/units the read-back implies.
   - **스칼라 불일치**: 다시 읽기에서 암시하는 문서화된 값/단위를 제공하면 됩니다.
3. Re-apply and re-run `verify_stuck`.
3. 'verify_stuck'을 다시 적용하고 다시 실행하세요.
3. Only after a format-mirrored retry still fails do you
3. 포맷 미러링 재시도가 여전히 실패한 후에야
grade `creation_only`/`partial`.
등급 `creation_only`/`partial`.

This single read-back-then-mirror loop replaces any per-IP value-format table — it works for
이 단일 읽기 후 미러 루프는 모든 IP별 값 형식 테이블을 대체합니다.
any param whose value snapped back, because the IP itself tells you the shape it wants.
IP 자체가 원하는 모양을 알려주기 때문에 값이 다시 스냅된 모든 매개변수입니다.

**Exact by default; tolerance only from the requirement:** Some params are *requests* the IP
**기본적으로 정확합니다. 요구 사항에서만 허용되는 허용 범위:** 일부 매개변수는 IP *요청*입니다.
quantizes to the nearest achievable value (PLL/MMCM frequencies, divider ratios, lane counts
달성 가능한 가장 가까운 값으로 양자화합니다(PLL/MMCM 주파수, 분배기 비율, 레인 수
rounded to a legal set).
유효한 세트로 반올림됨)
A read-back of 720 for a required 719.98, or 4095 for a required
필수 719.98의 경우 720, 필수의 경우 4095 다시 읽기
4096, is a **miss** (`NEAREST`), not a pass.
4096은 통과가 아닌 **실패**(`NEAREST`)입니다.
Accept a difference only when the DesignSpec
DesignSpec이 지정된 경우에만 차이를 허용합니다.
states a tolerance for that value, and pass it explicitly: `ipcfg::require $cell $d $reason
해당 값에 대한 허용 오차를 명시하고 이를 명시적으로 전달합니다: `ipcfg::require $cell $d $reason
{CONFIG.<KEY> 0.1%}` (or `<abs>`); `audit_requirements` then reports `RESOLVED` within it.
{CONFIG.<KEY> 0.1%}`(또는 `<abs>`); 'audit_requirements'는 그 안에 'RESOLVED'를 보고합니다.
On `NEAREST`, the achieved value tells you what the IP can do: change the inputs that
'NEAREST'에서 달성된 값은 IP가 무엇을 할 수 있는지 알려줍니다. 즉, 입력을 변경하는 것입니다.
quantize it (e.g. the clock search) or escalate.
이를 양자화(예: 시계 검색)하거나 에스컬레이션합니다.
Do not widen a tolerance to make it pass.
통과하기 위해 공차를 넓히지 마십시오.
Record an accepted in-tolerance value as `applied-resolved:<achieved>` so request vs achieved
허용된 허용 오차 값을 'applied-resolved:<achieved>'로 기록하여 요청과 달성 여부를 확인합니다.
is disclosed.
공개됩니다.
Use a full-config baseline + diff to see
전체 구성 기준 + 차이점을 사용하여 확인하세요.
both the resolution and any IP side effects in one shot:
해상도와 IP 부작용을 모두 한 번에 해결:
```tcl
set base [ipcfg::snapshot_all $cell]
ipcfg::apply_dict $cell $d
puts [ipcfg::config_diff $cell $base $d]   ;# {key old new EXACT|RESOLVED|REVERTED|CHANGED ...}
```

**Automation-first for SYSTEM intent (do this BEFORE fighting gated CONFIG):**
**시스템 의도에 대한 자동화 우선(Gated CONFIG와 싸우기 전에 이 작업을 수행하십시오):**
Run `ipcfg::is_system_intent <prompt>` first.
먼저 `ipcfg::is_system_intent <프롬프트>`를 실행하세요.
If it returns `SYSTEM:{...}` (integrated
`SYSTEM:{...}`(통합)을 반환하는 경우
memory controllers, a PCIe controller, PS PL-clocks/peripherals), the prompt describes a
메모리 컨트롤러, PCIe 컨트롤러, PS PL 클럭/주변 장치), 프롬프트에서는 다음을 설명합니다.
*subsystem* that Vivado realizes through **block automation** / a **Configurable Example
Vivado가 **블록 자동화**를 통해 구현하는 *하위 시스템* / **구성 가능한 예시
Design**, not through plain standalone `CONFIG.*`.
디자인**, 단순한 독립 실행형 'CONFIG.*'를 통하지 않습니다.
Fighting the gated CONFIG there wastes
게이트된 CONFIG와 싸우는 것은 낭비입니다.
calls (validated: `NUM_MC`/`NUM_MCP` on a bare `axi_noc2` cell are **disabled** and every
호출(검증됨: 베어 `axi_noc2` 셀의 `NUM_MC`/`NUM_MCP`는 **비활성화**되고 모든
`set_property` is silently ignored with `[IP_Flow 19-3374] ... disabled parameter ...
`[IP_Flow 19-3374] ... 비활성화된 매개변수 ...에서는 `set_property`가 자동으로 무시됩니다.
ignored`).
무시`).
Do **not** hardcode which IPs need it.
필요한 IP를 하드코딩하지 **하지 마세요**.
Instead:
대신:

1. **Enumerate the loaded rules directly from Vivado** (the authoritative source — `get_bd_automation_rules` does *not* exist, but the internal rule manager does dump the registry).
1. **Vivado에서 직접 로드된 규칙을 열거합니다**(권한 있는 소스인 `get_bd_automation_rules`는 존재하지 *않지만* 내부 규칙 관리자가 레지스트리를 덤프합니다).
1. Run, in one execute, the rule dump and read the printed `[DBG] RulesMap:` block:
1. 한 번에 규칙 덤프를 실행하고 인쇄된 `[DBG] RulesMap:` 블록을 읽습니다.
   ```tcl
   ::bd::util_cmd rules dump
   # prints: "<vlnv>":["<description>", "<rule-short-name>", "<rules.tcl path>"]
   ```
The dump is a C-level message (it does **not** return a value and cannot be captured
덤프는 C 수준 메시지입니다(값을 반환하지 **않음** 및 캡처할 수 없음).
in-proc), so read it from the execute **output**.
in-proc) 실행 **출력**에서 읽어보세요.
Find the row whose key matches your
키와 일치하는 행을 찾으세요.
cell's VLNV; the **2nd quoted field is the rule short-name**, and the rule id is
세포의 VLNV; **두 번째로 인용된 필드는 규칙 짧은 이름**이고 규칙 ID는 다음과 같습니다.
`xilinx.com:bd_rule:<short>` (validated: `axi_noc2`, `ps_wizard`, `versal_cips`,
`xilinx.com:bd_rule:<short>`(검증됨: `axi_noc2`, `ps_wizard`, `versal_cips`,
`pcie_versal`, `microblaze`, … all have rules).
`pcie_versal`, `microblaze`, … 모두 규칙이 있습니다).
If you have the dump text in a Tcl var,
Tcl var에 덤프 텍스트가 있는 경우,
`ipcfg::rule_for_vlnv $dump <vlnv>` extracts the id for you.
`ipcfg::rule_for_vlnv $dump <vlnv>`는 ID를 추출합니다.
The 3rd field is the rule's
세 번째 필드는 규칙의 필드입니다.
`rules.tcl`/`bd.tcl` path — open it (or `vivado_doc_search`) to learn the rule's `-config`
`rules.tcl`/`bd.tcl` 경로 — 이를 열어(또는 `vivado_doc_search`) 규칙의 `-config`를 알아보세요.
option names.
옵션 이름.
Also check whether the IP ships a **Configurable Example Design**.
또한 IP가 **구성 가능한 예제 디자인**을 제공하는지 확인하세요.
2. **Run the automation and read what it changed** so you can confirm the result against intent (this is the
2. **자동화를 실행하고 변경된 내용을 읽어보세요** 의도와 대조하여 결과를 확인할 수 있습니다(이것은
automation-first path; it gets its **own** Tier-2 budget, separate from the per-param
자동화 우선 경로; 매개변수별 예산과 별도로 **자체** Tier-2 예산을 얻습니다.
retry cap).
다시 시도 제한).
Pass the rule's options — a **bare apply is often a no-op** for subsystem
규칙의 옵션을 전달합니다. **기본 적용은 하위 시스템에 대해 작동하지 않는 경우가 많습니다**
features (validated: `apply_bd_automation -rule xilinx.com:bd_rule:axi_noc2` with no
기능(검증됨: `apply_bd_automation -rule xilinx.com:bd_rule:axi_noc2` 없음
`-config` leaves `NUM_MC` at 0; the MC options must be supplied):
`-config`는 `NUM_MC`를 0으로 유지합니다. MC 옵션을 제공해야 함):
   ```tcl
   set base [ipcfg::snapshot_all $cell]
   puts [ipcfg::run_automation $cell xilinx.com:bd_rule:<short> {<opt> <val> ...} $base]
   # -> AUTOMATION:OK changed={key old new CHANGED ...} added_cells={...}
   ```
Read the result: the `changed` keys (and any `added_cells`) are what automation set — match
결과를 읽어보세요. `changed` 키(및 `add_cells`)는 자동화 세트와 일치합니다.
them to the requirement instead of guessing gated CONFIG. (`ipcfg::try_automation` still
게이트된 CONFIG를 추측하는 대신 요구 사항에 따라 설정합니다. (`ipcfg::try_automation`은 아직
exists for a fire-only apply when you do not need the change report.)
변경 보고서가 필요하지 않은 경우 화재 전용 적용을 위해 존재합니다.)
3. **(interactive)** When multiple rules / example designs could apply, ask the user
3. **(대화형)** 여러 규칙/예시 디자인을 적용할 수 있는 경우 사용자에게 물어보세요.
(clickable) which to apply rather than guessing.
(클릭 가능) 추측하기보다는 적용할 항목입니다.

Per the give-up gate, you may not grade a requirement `integration-derived` until **either**
포기 게이트에 따라 **둘 중 하나**가 될 때까지 '통합 파생' 요구 사항을 채점할 수 없습니다.
the rule dump shows no rule for the VLNV (and no example design realizes it) **or** the rule
규칙 덤프에는 VLNV에 대한 규칙이 표시되지 않습니다(그리고 이를 실현하는 예제 디자인도 없음) **또는** 규칙
was applied with its options and its reported changes did not produce the value.
옵션과 함께 적용되었으며 보고된 변경 사항으로 인해 값이 생성되지 않았습니다.
A `SYSTEM` intent
'SYSTEM' 인텐트
whose params are disabled with **no** rule/enabler is legitimately integration-derived —
**no** 규칙/활성화로 매개변수가 비활성화된 매개변수는 합법적으로 통합에서 파생되었습니다.
report it via the coverage report, do not fake a value.
적용 범위 보고서를 통해 보고하고 값을 위조하지 마십시오.

For `versal_cips`, note (validated) that PL-clock frequencies, `PS_NUM_FABRIC_RESETS`, and
`versal_cips`의 경우 PL 클록 주파수 `PS_NUM_FABRIC_RESETS` 및
`PS_USE_PMCPL_CLK*` **do** stick directly inside `CONFIG.PS_PMC_CONFIG`; automation is for
`PS_USE_PMCPL_CLK*` **do**는 `CONFIG.PS_PMC_CONFIG` 안에 직접 붙이십시오. 자동화는
generating the external clk/reset **wiring**, not for setting those values.
해당 값을 설정하기 위한 것이 아니라 외부 clk/reset **배선**을 생성합니다.
Always prefer the
항상 선호하는
direct nested-dict path when `verify_stuck` confirms it stuck, and fall back to automation
'verify_stuck'이 중단되었음을 확인한 경우 직접 중첩된 사전 경로를 확인하고 자동화로 돌아갑니다.
(per docs) only for connectivity the dict cannot express.
(문서별) 연결에 대해서만 dict가 표현할 수 없습니다.

**Phase 3 procedure — enabler-first, multi-pass (the default, not an exception):**
**3단계 절차 — Enabler 우선, 다중 패스(기본값, 예외 없음):**
Treat ordered configuration as the *normal* path, not a special case.
순서가 지정된 구성을 특별한 경우가 아닌 *일반* 경로로 처리합니다.
Many params are
많은 매개변수는
**gated**: they only become settable once a parent param selects a mode/type/topology or
**게이트**: 상위 매개변수가 모드/유형/토폴로지를 선택한 후에만 설정 가능해집니다.
turns a feature on.
기능을 켭니다.
Setting a gated child before its enabler makes the child silently
활성화자 앞에 게이트된 하위를 설정하면 하위가 자동으로 설정됩니다.
**disabled/ignored** (see `PARAM_DISABLED` recovery in Step 5).
**비활성화/무시**(5단계의 `PARAM_DISABLED` 복구 참조)
So, by default:
따라서 기본적으로 다음과 같습니다.

1. **Pass 1 — enablers first:** set the params that *select or enable* something —
1. **패스 1 — 활성화 요소 먼저:** 무언가를 *선택하거나 활성화*하는 매개변수 설정 —
mode/type/device-selection, topology, channel/controller counts, and feature
모드/유형/장치 선택, 토폴로지, 채널/컨트롤러 수 및 기능
`ENABLE` toggles.
'활성화'가 전환됩니다.
These are the params other params depend on.
이는 다른 매개변수가 의존하는 매개변수입니다.
2. **Pass 2 — dependents:** set the params that only make sense once Pass 1 applied.
2. **패스 2 - 종속 항목:** 패스 1이 적용된 후에만 의미가 있는 매개변수를 설정합니다.
3. **Verify:** run `verify_stuck` + `verify_intent` + **`check_enablers`** after Pass 2.
3. **확인:** 패스 2 이후 `verify_stuck` + `verify_intent` + **`check_enablers`**를 실행합니다.

How to tell which params are enablers — **ask the cell, do not infer**.
어떤 매개변수가 활성자인지 확인하는 방법 — **셀에 물어보고 추론하지 마세요**.
`ipcfg::feature_flags
`ipcfg::feature_flags
<cell> {<nouns from the prompt>}` lists the boolean params on this cell that enable those
nouns, with their current values, so Pass 1 is built from the live parameter list rather
현재 값을 사용하는 명사이므로 Pass 1은 실시간 매개변수 목록에서 작성됩니다.
than from a guess about naming.
이름을 추측하는 것보다.
Docs and compound phrasing ("select device type X, then
문서 및 복합 표현("장치 유형 X를 선택한 다음
configure its N controllers") remain useful corroboration, and `find_disabled` /
N 컨트롤러 구성")은 유용한 확증으로 남아 있으며 `find_disabled` /
`PARAM_DISABLED` feedback still corrects the ordering when a gate is not name-shaped.
'PARAM_DISABLED' 피드백은 게이트가 이름 모양이 아닌 경우에도 여전히 순서를 수정합니다.

**On 2026.1, prefer the graph over the naming heuristic — and know its limits.** When you
**2026.1에서는 명명 휴리스틱보다 그래프를 선호하고 그 한계를 알아두세요.**
have a `param_deps` dump, `ipcfg::port_enablers <dump> <port>` is the *authority* on what
`param_deps` 덤프가 있습니다. `ipcfg::port_enablers <dump> <port>`는 무엇에 대한 *권한*입니다
Pass 1 must contain, because it reads the IP's own `PORT_ENABLEMENT`/`BUSIF_ENABLEMENT`
패스 1에는 IP 자체 `PORT_ENABLEMENT`/`BUSIF_ENABLEMENT`를 읽기 때문에 포함되어야 합니다.
dependencies instead of pattern-matching names.
패턴 일치 이름 대신 종속성.
Read its three answers correctly:
세 가지 답변을 올바르게 읽으십시오.

- `PE:<port>:<params>` — set **all** of these in Pass 1.
- `PE:<포트>:<매개변수>` — Pass 1에서 **모두** 설정합니다.
- Ask at pin level as well as
- 핀 레벨에서도 물어보세요.
interface level when the request is about direction or width: `GPIO2` needs only
요청이 방향이나 너비에 관한 것일 때 인터페이스 수준: 'GPIO2'만 필요
`C_IS_DUAL`, but the tri-state pin `GPIO2_IO_T` needs `C_ALL_INPUTS_2 C_ALL_OUTPUTS_2
`C_IS_DUAL`, 하지만 3상태 핀 `GPIO2_IO_T`에는 `C_ALL_INPUTS_2 C_ALL_OUTPUTS_2가 필요합니다.
C_IS_DUAL`, so "make GPIO2 bidirectional" is a three-param request.
C_IS_DUAL`이므로 "GPIO2를 양방향으로 설정"은 3개 매개변수 요청입니다.
- `PE_DERIVED:<port>:<params>` — the port is gated by a **computed** value with no
- `PE_DERIVED:<port>:<params>` — 포트는 **계산된** 값으로 게이트됩니다.
user-facing twin (`CLK1_CE` hangs off `C_CLK_TREE1`, itself computed from 9 params).
사용자 지향 트윈(`CLK1_CE`는 `C_CLK_TREE1`에 매달려 있으며 자체적으로 9개 매개변수로 계산됩니다.)
There is no single flag to set: change the listed contributors and re-read the pins.
설정할 단일 플래그는 없습니다. 나열된 기여자를 변경하고 핀을 다시 읽으십시오.
- `PE_MISS:<port>` / `PE_PORTS:none` — this IP publishes **no** gating for that port.
- `PE_MISS:<포트>` / `PE_PORTS:none` — 이 IP는 해당 포트에 대해 **no** 게이팅을 게시합니다.
This is **not** "ungated." All CIPS-class IPs land here (2026.1 `ps_wizard`: 34 graph
이것은 "개폐되지 않은" 것이 **아닙니다**. 모든 CIPS급 IP가 여기에 위치합니다(2026.1 `ps_wizard`: 34 그래프
nodes, zero enablement nodes), as does any port you have merely misspelled — check
노드, 활성화 노드 없음), 철자가 틀린 포트도 마찬가지입니다. — 확인하세요.
`ipcfg::enablement_ports <dump>` for the IP's own spelling first, then fall back to
먼저 IP 자체 철자에 대해 `ipcfg::enablement_ports <dump>`를 사용한 다음 다음으로 대체합니다.
`feature_flags`/`check_enablers` and the `PARAM_DISABLED` feedback loop.
`feature_flags`/`check_enablers` 및 `PARAM_DISABLED` 피드백 루프.

Also pre-flight each value with `ipcfg::check_value <dump> <param> <value>` before the write.
또한 쓰기 전에 `ipcfg::check_value <dump> <param> <value>`를 사용하여 각 값을 미리 비행합니다.
An out-of-range number saves a failed round trip, and an enum near-miss
범위를 벗어난 숫자는 실패한 왕복 및 열거형을 거의 놓칠 수 없게 저장합니다.
(`did_you_mean=ACTIVE_LOW` for `ACTIVELOW`) catches the silent-typo class that Vivado accepts
(`ACTIVELOW`의 경우 `did_you_mean=ACTIVE_LOW`)는 Vivado가 허용하는 자동 오타 클래스를 포착합니다.
without complaint.
불만없이.
`VAL_UNCHECKED` just means no range is published — proceed.
`VAL_UNCHECKED`는 게시된 범위가 없음을 의미합니다. 계속 진행하세요.

**A `VAL_OK` on the literal the brief stated ends the question.** Where the request names a
**설명된 요약에 'VAL_OK'가 있으면 질문이 종료됩니다.** 요청에
value outright, that value is the answer as long as the parameter accepts it; a
값은 매개변수가 허용하는 한 그 값이 답입니다. 에
documentation table that appears to encode the same quantity differently does not outrank
동일한 수량을 다르게 인코딩하는 것으로 보이는 문서 테이블의 순위가 높지 않습니다.
it.
Measured on `visp_ss`: a brief asking for a "12-bit basic data format" was written as
`visp_ss`에서 측정: "12비트 기본 데이터 형식"을 요청하는 간략한 내용이 다음과 같이 작성되었습니다.
`C_TILE0_ISP0_BASIC_DATA_FORMAT=16` on the reasoning that 16 is the code for RAW12.
16이 RAW12의 코드라는 추론에 대한 `C_TILE0_ISP0_BASIC_DATA_FORMAT=16`.
The
는
parameter is a plain bit width with no published enum, `12` pre-flights clean, sticks, and
매개변수는 게시된 열거형이 없는 일반 비트 너비이며 '12' 사전 비행이 깨끗하고 고정되며
reads back — and all three tiles shipped at the wrong format with a confident note
다시 읽습니다 — 타일 세 개 모두 자신 있게 메모와 함께 잘못된 형식으로 배송되었습니다.
explaining the encoding.
인코딩을 설명합니다.
Decode only what the brief left unstated, and if you do depart
간략히 언급되지 않은 내용만 해독하고, 떠난 경우
from a literal, put the departure and its evidence in `notes` where it can be reviewed.
문자 그대로 출발과 그 증거를 검토할 수 있는 '노트'에 넣으세요.

**The gated write that reports success — read this before declaring `full`.** An attribute
**성공을 보고하는 게이트 쓰기 - 'full'을 선언하기 전에 이 내용을 읽어보세요.** 속성
whose feature is switched off is *not* rejected: it applies, it reads back, `verify_stuck` is
꺼진 기능은 거부되지 *않습니다*: 적용되고 다시 읽혀지며 'verify_stuck'은
empty, and the IP silently has no such port.
비어 있고 IP에 자동으로 그러한 포트가 없습니다.
Measured on `clkx5_wiz`: with `CONFIG.RESET_TYPE
`clkx5_wiz`에서 측정됨: `CONFIG.RESET_TYPE 사용
ACTIVE_LOW` set and `USE_RESET` left `false`, `RESET_TYPE` reads `ACTIVE_LOW` while the cell's
ACTIVE_LOW` 설정 및 `USE_RESET` 왼쪽 `false`, `RESET_TYPE`은 `ACTIVE_LOW`를 읽는 반면 셀은
pins are exactly `{clk_in1 clk_out1}` — the requested active-low reset does not exist.
핀은 정확히 `{clk_in1 clk_out1}`입니다. 요청된 액티브 로우 재설정이 존재하지 않습니다.
Setting
설정
`USE_RESET true` is what grows `resetn`.
`USE_RESET true`는 `resetn`을 증가시킵니다.
So **a request for a signal or feature needs its flag,
따라서 **신호나 기능에 대한 요청에는 해당 플래그가 필요합니다.
not just its attribute**: run `ipcfg::check_enablers <cell> <dict>` after every apply and treat
해당 속성뿐만 아니라**: 모든 적용 및 처리 후에 `ipcfg::check_enablers <cell> <dict>`를 실행하세요.
a non-empty `INERT:` line as an unfinished configuration, not a warning.
비어 있지 않은 `INERT:` 줄은 경고가 아닌 완료되지 않은 구성입니다.

Concrete IP examples that need this ordering (illustrative, not exhaustive — RFDC mixer
이 순서가 필요한 구체적인 IP 예(완전하지 않은 예시 - RFDC 믹서)
type requires tile/slice enables; `v_tc` generator sizes require `max_lines_per_frame`).
유형에는 타일/슬라이스 활성화가 필요합니다. `v_tc` 생성기 크기에는 `max_lines_per_frame`이 필요합니다.)
Split Phase 3 into ordered passes within the same script:
동일한 스크립트 내에서 3단계를 순서가 지정된 패스로 분할합니다.

```tcl
# --- Phase 3a: Enable parent features first ---
set_property -dict [list \
    CONFIG.ENABLE_PARENT {1} \
    CONFIG.PARENT_MODE  {some_mode} \
] [get_bd_cells <cell_name>]

# --- Phase 3b: Set dependent params (now valid) ---
set _prop_dict [list \
    CONFIG.CHILD_PARAM_A {value_a} \
    CONFIG.CHILD_PARAM_B {value_b} \
]
if {[catch {set_property -dict $_prop_dict [get_bd_cells <cell_name>]} _cfg_err]} {
    # ... error handling as above ...
}
```

IPs known to require multi-pass ordering:
다중 패스 주문이 필요한 것으로 알려진 IP:

| IP | Pass 1 (enables)<br>패스 1(활성화) | Pass 2 (dependent)<br>패스 2(종속) |
|---|---|---|
| `rfdc`<br>`rfdc` | Tile enable, slice enable, sampling rate<br>타일 활성화, 슬라이스 활성화, 샘플링 속도 | Mixer type, NCO freq, decimation<br>믹서 유형, NCO 주파수, 데시메이션 |

Note: `v_tc` GEN_* timing parameters (VACTIVE, HACTIVE, frame sizes) are
참고: `v_tc` GEN_* 타이밍 매개변수(VACTIVE, HACTIVE, 프레임 크기)는
runtime AXI-Lite register values programmed by the VTC driver, not design-time
디자인 타임이 아닌 VTC 드라이버에 의해 프로그래밍된 런타임 AXI-Lite 레지스터 값
CONFIG properties.
구성 속성.
Only `enable_generation` and `max_lines_per_frame` are
'enable_ Generation' 및 'max_lines_per_frame'만 가능합니다.
design-time params.
디자인 타임 매개변수.

**Phase 4 — Stub connections for connection-derived params (generic):** Some parameters are
**4단계 — 연결 파생 매개변수(일반)에 대한 스텁 연결:** 일부 매개변수는 다음과 같습니다.
read-only in BD because their value propagates from an interface connection, not
BD에서는 해당 값이 인터페이스 연결에서 전파되기 때문에 읽기 전용입니다.
`set_property`.
`세트_속성`.
After Phases 1-3 succeed, attach a boundary interface port with
1~3단계가 성공하면 경계 인터페이스 포트를 연결합니다.
`ipcfg::add_stub` — it auto-detects the pin's interface VLNV requirement (you pass the VLNV)
`ipcfg::add_stub` — 핀의 인터페이스 VLNV 요구 사항을 자동 감지합니다(VLNV를 전달함).
and **mirrors the pin's mode** (an external port attached to a `Slave` pin must itself be a
**핀의 모드를 미러링합니다**('슬레이브' 핀에 연결된 외부 포트는 그 자체가
`Slave` port; passing the wrong mode yields `[BD 41-172] modes ... do not match`):
'슬레이브' 포트; 잘못된 모드를 전달하면 `[BD 41-172] 모드 ... 일치하지 않음`이 발생합니다.):

```tcl
puts [ipcfg::add_stub $cell <pin_name> <intf_vlnv> ?<CONFIG_prop> <val>?]
# e.g. drive an AXI-Stream slave input so the cell adopts the connected width:
puts [ipcfg::add_stub $cell S_AXIS xilinx.com:interface:axis_rtl:1.0 TDATA_NUM_BYTES 8]
```

**Important nuance (validated):** for many AXI-Stream *receive* inputs the width parameter is
**중요한 뉘앙스(검증됨):** 많은 AXI-Stream *수신* 입력의 경우 너비 매개변수는 다음과 같습니다.
**read-only on the boundary port too** (`[BD 41-737] ...
**경계 포트에서도 읽기 전용입니다** (`[BD 41-737] ...
TDATA_NUM_BYTES ... read-only`) —
TDATA_NUM_BYTES ... 읽기 전용`) —
the width is inherited from whatever master is connected at integration time, and a bare
너비는 통합 시 연결된 모든 마스터에서 상속되며 기본
boundary port has no master to define it.
경계 포트에는 이를 정의할 마스터가 없습니다.
`add_stub` tolerates this (the property set is
`add_stub`은 이를 허용합니다(속성 세트는
wrapped in `catch`): the connection still elaborates, but you **cannot force** the width
`catch`로 래핑됨): 연결은 여전히 정교하지만 너비를 **강제할 수는 없습니다**
standalone.
독립형.
When the target width param is still wrong after the stub, that feature is
스텁 후에도 대상 너비 매개변수가 여전히 잘못된 경우 해당 기능은 다음과 같습니다.
genuinely **integration-derived** — grade `partial` and record the param in `runtime_params`,
진정으로 **통합 파생** — '부분' 등급을 매기고 'runtime_params'에 매개변수를 기록합니다.
do not keep fighting it.
계속 싸우지 마십시오.
Use stubs to *enable elaboration / drive settable port widths*, not
스텁을 사용하여 *정교화를 활성화하고 설정 가능한 포트 너비를 구동*합니다.
to fake values the datapath must supply.
값을 가짜로 만들려면 데이터 경로가 제공해야 합니다.

`ipcfg::cleanup <cell> ?orig_part? ?force?` removes all `STUB_*` ports (and their nets),
`ipcfg::cleanup <셀> ?orig_part? ?force?`는 모든 `STUB_*` 포트(및 해당 네트)를 제거합니다.
deletes a scratch cell, and restores the part if you pass the original — so a single call
스크래치 셀을 삭제하고 원본을 전달하면 부품을 복원하므로 단일 호출
replaces the manual stub/port/part teardown.
수동 스텁/포트/부품 분해를 대체합니다.

**It will not delete the cell you were asked to build.** A cell holding pinned
**작성 요청을 받은 셀은 삭제되지 않습니다.** 고정된 셀은 삭제되지 않습니다.
requirements is the deliverable by definition, and cleanup returns `SKIP:KEPT` for it
요구 사항은 정의에 따른 결과물이며 정리는 이에 대해 `SKIP:KEPT`를 반환합니다.
while still tearing down the stubs and the part.
여전히 스텁과 부품을 찢어내는 동안.
Measured on `axi_gpio`: a run configured
`axi_gpio`에서 측정됨: 구성된 실행
its cell correctly, verified it, then closed with `ipcfg::cleanup` on that cell —
해당 셀을 올바르게 확인하고 해당 셀에서 `ipcfg::cleanup`으로 닫았습니다.
`delete_bd_objs` removed the only evidence, and the design read back afterwards had no
`delete_bd_objs`는 유일한 증거를 제거했으며 나중에 다시 읽어온 디자인에는 증거가 없습니다.
such cell.
그런 세포.
Pass `force=1` only for a cell that really was scratch.
실제로 스크래치였던 셀에 대해서만 'force=1'을 전달합니다.

**When no part swap is needed**, omit Phase 0 entirely and remove the `_need_swap`
**부품 교체가 필요하지 않은 경우** 0단계를 완전히 생략하고 `_need_swap`을 제거하세요.
restore blocks — the script simplifies to Phases 1-3 (and optionally Phase 4) only.
블록 복원 — 스크립트는 1~3단계(및 선택적으로 4단계)로만 단순화됩니다.

**Cleanup TCL (always runs after recording results):**
**정리 TCL(항상 결과 기록 후 실행):**

```tcl
# One call deletes STUB_* ports + the cell, and restores the part if it was swapped.
# Pass $orig only when a swap happened; omit (or pass "") otherwise.
puts [ipcfg::cleanup <cell_name> $orig]
```

This replaces the manual `delete_bd_objs` / part-restore blocks.
이는 수동 `delete_bd_objs` / 부분 복원 블록을 대체합니다.
It is safe to call even when
전화해도 안전해요
no stubs or swap were used (the port/part steps are no-ops).
스텁이나 스왑은 사용되지 않았습니다(포트/부분 단계는 작동하지 않습니다).

Execute:
실행:
```
vivado_execute("source /tmp/configure_<cell_name>.tcl")
```

### Step 4: Handle result
### 4단계: 결과 처리

Parse the output line.
출력 라인을 구문 분석합니다.
Possible prefixes:
가능한 접두사:

| Output Prefix<br>출력 접두사 | Meaning<br>의미 | Next Step<br>다음 단계 |
|---|---|---|
| `SUCCESS:`<br>`성공:` | All parameters applied<br>모든 매개변수가 적용됨 | Run the self-audit below before reporting<br>신고 전 아래 자체 감사를 실행하세요. |
| `STUCK_FAIL:...`<br>`걸림 실패:...` | A key reverted / was silently ignored<br>키가 되돌려졌거나 자동으로 무시되었습니다. | Tier 2: format-mirror retry or `PARAM_DISABLED` recovery<br>계층 2: 포맷 미러 재시도 또는 `PARAM_DISABLED` 복구 |
| `INTENT_FAIL:...`<br>`의도_실패:...` | A key is `suspect-no-change` (likely wrong param)<br>키는 'suspect-no-change'입니다(잘못된 매개변수일 가능성이 있음). | Tier 2: remap via doc descriptions, re-apply<br>Tier 2: 문서 설명을 통해 다시 매핑하고 다시 적용 |
| `CONFIGURE_FAIL:VLNV_NOT_FOUND:...`<br>`CONFIGURE_FAIL:VLNV_NOT_FOUND:...` | IP not available even after part swap<br>부품 교체 후에도 IP를 사용할 수 없음 | Record failure — no retry<br>기록 실패 - 재시도 없음 |
| `CONFIGURE_FAIL:<err_type>:...`<br>`CONFIGURE_FAIL:<err_type>:...` | Configuration error<br>구성 오류 | Enter Tier 2 using `<err_type>`<br>`<err_type>`을 사용하여 Tier 2에 진입 |

Note: `PART_SWAP:` lines are informational — record the target part in results as
참고: `PART_SWAP:` 줄은 정보 제공용입니다. 결과의 대상 부분을 다음과 같이 기록합니다.
`part_swapped`.
'부분 교환'.

**Self-audit before declaring SUCCESS (required):** A `SUCCESS:` line is *necessary but
**SUCCESS 선언 전 자체 감사(필수):** 'SUCCESS:' 줄은 *필요하지만
not sufficient*.
충분하지 않습니다*.
Before reporting, confirm all of:
보고하기 전에 다음 사항을 모두 확인하세요.
1. `ipcfg::verify_stuck` returned empty (no revert).
1. `ipcfg::verify_stuck`이 비어 있는 상태로 반환되었습니다(되돌리기 없음).
2. `ipcfg::verify_intent` flagged no **hard reverts**.
2. `ipcfg::verify_intent`에 **하드 되돌리기**가 없음으로 표시되었습니다.
2. A `suspect-no-change` (value already
2. '의심스러운 변화 없음'(값은 이미
equalled the default) is a **soft** flag, not an automatic failure: re-read the doc
기본값과 같음)은 자동 실패가 아닌 **소프트** 플래그입니다. 문서를 다시 읽으세요.
description for that requirement.
해당 요구사항에 대한 설명입니다.
If the param you set is the doc-correct one and its
설정한 매개변수가 문서에 맞는 매개변수이고 그 매개변수가
default legitimately satisfies the intent, **accept it** and record `value_src=default`
기본값은 의도를 합법적으로 충족하고 **수락**하고 `value_src=default`를 기록합니다.
in the ledger (a user/answer not changing a param does not make the default wrong).
원장에서 (매개 변수를 변경하지 않은 사용자/답변은 기본값을 잘못 만들지 않습니다).
Only
만
treat it as `INTENT_FAIL` when a *competing* param better matches the doc description —
*경쟁* 매개변수가 문서 설명과 더 잘 일치하는 경우 `INTENT_FAIL`로 처리합니다. —
then remap and re-apply. (Re-run `verify_intent` with `flag_no_change=0` once you have
그런 다음 다시 매핑하고 다시 적용하십시오. ('flag_no_change=0'을 사용하여 'verify_intent'를 다시 실행하세요.
confirmed the grounding, so only hard reverts remain.)
접지가 확인되었으므로 하드 복귀만 남습니다.)
3. `ipcfg::find_disabled <raw_output>` returned `""` (no non-fatal gated-param warning).
3. `ipcfg::find_disabled <raw_output>`이 `""`를 반환했습니다(치명적이지 않은 Gated-param 경고 없음).
4. `ipcfg::check_enablers` returned `""` — **nothing applied is inert**.
4. `ipcfg::check_enablers`가 `""`를 반환했습니다. — **적용된 항목은 비활성 상태입니다**.
4. A non-empty
4. 비어 있지 않은
`INERT:` line names the flag to set: set it, re-apply, re-check.
`INERT:` 줄은 설정할 플래그의 이름을 지정합니다. 설정하고, 다시 적용하고, 다시 확인하세요.
This is a hard gate, not
이건 어려운 문이지, 그렇지 않아
a soft one; the value reading back correctly is exactly why it needs its own check.
부드러운 것; 올바르게 다시 읽는 값이 바로 자체 확인이 필요한 이유입니다.
And
그리고
for every signal/feature the prompt asks to *have* or *expose* (a reset, a locked output,
모든 신호/기능에 대해 프롬프트는 *가지거나* *노출*(재설정, 잠긴 출력,
an interrupt, a second channel), confirm its flag is on via `ipcfg::feature_flags` — an
인터럽트, 두 번째 채널) `ipcfg::feature_flags`를 통해 해당 플래그가 켜져 있는지 확인합니다.
attribute like `RESET_TYPE` alone does not create the port.
`RESET_TYPE`과 같은 속성만으로는 포트를 생성하지 않습니다.
5. **Ledger coverage:** every requirement from Step 1.5 is mapped to a param that is
5. **원장 적용 범위:** 1.5단계의 모든 요구 사항은 다음과 같은 매개변수에 매핑됩니다.
confirmed by 1–4.
1~4로 확인됩니다.
Any uncovered or suspect requirement downgrades the result and
발견되지 않았거나 의심스러운 요구 사항이 있으면 결과가 다운그레이드되고
triggers the matching Tier-2 recovery instead of reporting `full`.
'전체'를 보고하는 대신 일치하는 Tier-2 복구를 트리거합니다.

### Step 5: Error recovery (Tier 2)
### 5단계: 오류 복구(계층 2)

The `CONFIGURE_FAIL` line contains a classified error type.
'CONFIGURE_FAIL' 줄에는 분류된 오류 유형이 포함되어 있습니다.
Each type has a
각 유형에는
different recovery strategy:
다양한 복구 전략:

| Error Type<br>오류 유형 | Vivado Message Pattern<br>Vivado 메시지 패턴 | Recovery<br>회복 |
|---|---|---|
| `VLNV_NOT_FOUND`<br>`VLNV_NOT_FOUND` | IP unavailable after part swap<br>부품 교체 후 IP를 사용할 수 없음 | Try doc search for correct part, swap again if plausible; otherwise record failure<br>올바른 부품을 찾기 위해 문서 검색을 시도하고 그럴듯하면 다시 교체하십시오. 그렇지 않으면 기록 실패 |
| `PARAM_NOT_FOUND`<br>`PARAM_NOT_FOUND` | `[BD 41-1276] Cannot set the parameter X ... does not exist` (hard error) **or** the non-fatal `[BD 41-1276] ... does not exist` / `[IP_Flow 19-7090] Invalid parameter ... Ignoring` caught by the Phase 3 verification loop / output scan<br>`[BD 41-1276] 매개변수 X를 설정할 수 없습니다 ... 존재하지 않습니다`(하드 오류) **또는** 치명적이지 않은 `[BD 41-1276] ... 존재하지 않습니다` / `[IP_Flow 19-7090] 잘못된 매개변수 ... 무시`가 3단계 확인 루프/출력 스캔에 의해 포착되었습니다. | Drop unknown params; doc-search for correct names; rebuild dict<br>알 수 없는 매개변수를 삭제하세요. 정확한 이름을 위한 문서 검색; 딕셔너리 다시 작성 |
| `VALUE_OUT_OF_RANGE`<br>`VALUE_OUT_OF_RANGE` | `[IP_Flow 19-3461] Value 'V' is out of the range ... Valid values are - A, B, C`<br>`[IP_Flow 19-3461] 값 'V'가 범위를 벗어났습니다... 유효한 값은 - A, B, C입니다` | Parse valid values from the error text; pick closest match; rebuild dict<br>오류 텍스트에서 유효한 값을 구문 분석합니다. 가장 가까운 일치 항목을 선택하십시오. 딕셔너리 다시 작성 |
| `READ_ONLY`<br>`읽기 전용` | `[BD 41-737] Cannot set the parameter X ... It is read-only`<br>`[BD 41-737] X 매개변수를 설정할 수 없습니다. 읽기 전용입니다` | Strip read-only params and re-run — **no doc search needed** (saves 1 MCP call)<br>읽기 전용 매개변수를 제거하고 다시 실행 — **문서 검색이 필요하지 않음**(1개의 MCP 호출 저장) |
| `NOT_SUPPORTED`<br>`지원되지 않음` | `[BD 5-683] VLNV ... is not supported for the current part`<br>`[BD 5-683] VLNV ... 현재 부분에서는 지원되지 않습니다` | **Should never reach Tier 2** — Step 0a′'s `ipcfg::ip_availability` predicts it before the create, which matters because `catch` here yields only `[Common 17-39] ... failed due to earlier errors` (the 5-683 goes to the log, so there is nothing to classify). If it does fire: re-run the gate, then swap the part — do **not** substitute a different IP<br>**계층 2에 도달해서는 안 됩니다** — 0a단계의 `ipcfg::ip_availability`는 생성 전에 이를 예측합니다. 여기서 `catch`는 `[Common 17-39] ... 이전 오류로 인해 실패함`(5-683이 로그로 이동하므로 분류할 항목이 없음)만 생성하기 때문에 중요합니다. 작동하는 경우: 게이트를 다시 실행한 다음 부품을 교체하세요. 다른 IP로 대체하지 **마세요** |
| `PARAM_DISABLED`<br>`PARAM_DISABLED` | `disabled parameter ... ignored` / `[BD 41-721]` (often **non-fatal** — caught by `ipcfg::find_disabled` or the output scan, not a thrown error)<br>`disabled 매개변수 ... 무시됨` / `[BD 41-721]` (종종 **치명적이지 않음** — `ipcfg::find_disabled` 또는 출력 스캔에 의해 포착되며 오류가 발생하지 않음) | **Not terminal, not `creation_only`.** Find + set the gating parent first, then re-apply (see below)<br>**터미널이 아니고 'creation_only'가 아닙니다.** 먼저 게이팅 상위 항목을 찾아 설정한 다음 다시 적용하세요(아래 참조). |
| `INTENT_FAIL`<br>`의도_실패` | *No Vivado message* — `verify_intent` flagged a **hard revert**, or a `suspect-no-change` that doc re-check shows a *competing* param better matches<br>*Vivado 메시지 없음* — `verify_intent`가 **하드 되돌리기**를 플래그 지정했거나 문서 재확인에서 *경쟁* 매개변수가 더 잘 일치함을 보여주는 `suspect-no-change`를 표시했습니다. | Re-read doc descriptions; remap the suspect requirement to the correctly-grounded param and re-apply. If the param is doc-correct and the default satisfies intent, accept as `value_src=default` (not a failure)<br>문서 설명을 다시 읽으세요. 의심되는 요구 사항을 올바르게 기반된 매개변수에 다시 매핑하고 다시 적용합니다. 매개변수가 문서에 정확하고 기본값이 의도를 충족하는 경우 'value_src=default'로 수락합니다(실패 아님). |
| `UNKNOWN`<br>'알 수 없음' | Anything else<br>다른 것 | Call `vivado_log_messages` for detail, then doc-search<br>자세한 내용은 `vivado_log_messages`를 호출한 다음 문서 검색을 통해 확인하세요. |

**For `VLNV_NOT_FOUND` after no part swap was attempted:** The IP may need a
**부품 교환을 시도하지 않은 후 `VLNV_NOT_FOUND`의 경우:** IP에
different device family.
다른 장치 제품군.
Use `vivado_doc_search("<ip_name> supported devices")` to
`vivado_doc_search("<ip_name> 지원 장치")`를 사용하여
find the correct family, select a part from the fallback map, and retry with a part
올바른 제품군을 찾고 대체 맵에서 부품을 선택한 다음 부품으로 다시 시도하세요.
swap.
교환.
This is the one case where Tier 2 can recover from a VLNV failure.
이는 Tier 2가 VLNV 오류로부터 복구할 수 있는 한 가지 사례입니다.

**For `READ_ONLY` errors:** The read-only parameter often already has the desired
**`READ_ONLY` 오류의 경우:** 읽기 전용 매개변수에는 이미 원하는 값이 이미 있는 경우가 많습니다.
default value (e.g., `C_EXT_RESET_HIGH` defaults to active-low).
기본값(예: `C_EXT_RESET_HIGH`의 기본값은 활성-낮음)입니다.
Drop it from the
에서 삭제하세요.
dict and retry immediately — no doc search, no wasted MCP call:
dict 및 즉시 재시도 — 문서 검색 없음, 낭비되는 MCP 통화 없음:

```tcl
# Tier 2 retry — read-only params stripped
set_property -dict [list \
    CONFIG.PARAM_A {value_a} \
    CONFIG.PARAM_C {value_c} \
] [get_bd_cells <cell_name>]
puts "SUCCESS: <cell_name> configured (read-only params skipped)"
```

**For `VALUE_OUT_OF_RANGE` errors:** Vivado's error text includes the valid set, so this
**`VALUE_OUT_OF_RANGE` 오류의 경우:** Vivado의 오류 텍스트에는 유효한 세트가 포함되어 있으므로
is the **default zero-doc-search recovery** — never spend an MCP doc call here.
**기본 제로 문서 검색 복구**입니다. 여기서 MCP 문서 호출을 사용하지 마십시오.
Parse the
구문 분석
list directly:
직접 나열:
```
Valid values are - 6, 8, 10, 12
```
Then decide deterministically:
그런 다음 결정론적으로 결정합니다.
- If the requested value maps to a member of the set (or a clear nearest member that still
- 요청된 값이 집합의 구성원(또는 여전히 가장 가까운 구성원)에 매핑되는 경우
satisfies intent), apply it and grade `full`.
의도를 만족) 적용하고 'full' 등급을 매깁니다.
- If the requested value is **provably outside** the valid set with no equivalent (e.g. a
- 요청된 값이 상응하는 값이 없는 유효한 세트 **외부**인 경우(예:
4-tap scaler when the minimum is 6), apply the closest valid value and grade `negative`,
최소값이 6인 경우 4탭 스케일러), 가장 가까운 유효한 값과 '음수' 등급을 적용하고,
recording the valid set in `fidelity_note`.
'fidelity_note'에 유효한 세트를 기록합니다.

**For `PARAM_DISABLED` errors (gated parameter) — reconcile first, then find the enabler:**
**`PARAM_DISABLED` 오류(게이트 매개변수)의 경우 — 먼저 조정한 다음 인에이블러를 찾으세요.**
A disabled/ignored parameter means the param is real but currently *gated*.
비활성화/무시된 매개변수는 매개변수가 실제이지만 현재 *게이트됨*을 의미합니다.
This is
이것은
**recoverable** and must never be graded `creation_only` on first sight.
**복구 가능**하며 첫눈에 'creation_only' 등급을 받아서는 안 됩니다.
Recovery
회복
(IP-agnostic):
(IP에 구애받지 않음):

1. **Detect it even when non-fatal.** A gated param frequently does not throw — Vivado
1. **치명적이지 않은 경우에도 감지합니다.** 게이트 매개변수는 자주 발생하지 않습니다 — Vivado
emits a non-fatal warning and `catch` returns 0.
치명적이지 않은 경고를 내보내고 'catch'는 0을 반환합니다.
Always run
항상 실행
`ipcfg::find_disabled <raw_output>` (and the output scan) after apply; a `DISABLED:...`
적용 후 `ipcfg::find_disabled <raw_output>`(및 출력 스캔); '비활성화됨:...'
result is the trigger, not just a thrown `CONFIGURE_FAIL:PARAM_DISABLED`.
결과는 단순히 `CONFIGURE_FAIL:PARAM_DISABLED`가 발생한 것이 아니라 트리거입니다.
2. **Reconcile before hunting an enabler.** Run `ipcfg::reconcile_disabled <cell> <dict>`:
2. **인에이블러를 찾기 전에 조정하십시오.** `ipcfg::reconcile_disabled <cell> <dict>`를 실행하십시오.
   - **`OMIT:{...}`** — these gated keys already hold the value you wanted (the read-only /
   - **`생략:{...}`** — 이러한 게이트 키는 이미 원하는 값을 보유하고 있습니다(읽기 전용 /
gated value equals your intent). **Drop them and move on** — no enabler search, no
게이트된 가치는 귀하의 의도와 동일합니다). **삭제하고 계속 진행** — 지원 요소 검색 없음, 없음
re-apply.
다시 신청하세요.
This saves MCP calls and is the common case.
이는 MCP 호출을 저장하며 일반적인 경우입니다.
   - **`DIFFER:{...}`** — only these are genuine problems (gated value ≠ intent).
   - **`다름:{...}`** — 이는 실제 문제입니다(제한된 값 ≠ 의도).
   - Continue
   - 계속
to step 3 for the `DIFFER` set only.
'DIFFER' 세트의 경우에만 3단계로 진행하세요.
3. **Find the enabler via doc search (DIFFER keys only):**
3. **문서 검색을 통해 Enabler 찾기(다른 키만 해당):**
   ```
   vivado_doc_search("<IP name> <gated_param_or_feature> prerequisite enable depends on")
   ```
Look for the parent param that selects the mode/type/device/topology which un-gates it.
게이트를 해제하는 모드/유형/장치/토폴로지를 선택하는 상위 매개변수를 찾으세요.
4. **Set the enabler first, then the still-differing gated params** — the enabler-first
4. **인에이블러를 먼저 설정한 다음 여전히 다른 게이트 매개변수를 설정합니다** — 인에이블러 우선
multi-pass from Phase 3:
3단계의 다중 패스:
   ```tcl
   # Pass 1: the enabler the doc named (mode/type/device-selection/feature toggle)
   puts "ENABLER:[ipcfg::apply_dict $cell [list CONFIG.<ENABLER> {<value>}]]"
   # Pass 2: re-apply the DIFFER params (now valid)
   puts "GATED:[ipcfg::apply_dict $cell [list CONFIG.<GATED_A> {a}]]"
   puts "STUCK:[ipcfg::verify_stuck $cell [list CONFIG.<GATED_A> {a}]]"
   ```
5. **Re-verify** with `verify_stuck` + `verify_intent`.
5. **다시 확인** `verify_stuck` + `verify_intent`를 사용하세요.
5. Only if a `DIFFER` param still will
5. `DIFFER` 매개변수가 여전히
not take *after* the enabler is confirmed set do you grade `partial` (see the tightened
*인에이블러가 확인된 후* 사용하지 않음 '부분' 등급을 지정합니까(강화된 내용 참조)
give-up gate in Step 4), and record that exact prompt fragment as `unapplied:gated-no-enabler`.
4단계의 포기 게이트) 정확한 프롬프트 부분을 'unapplied:gated-no-enabler'로 기록하세요.

This generalizes the validated "live inputs only appear once an IO-type selector is set"
이는 검증된 "라이브 입력은 IO 유형 선택기가 설정된 후에만 나타납니다"를 일반화합니다.
case and any "device-type selection un-gates the controller/channel params" pattern — the
경우 및 모든 "장치 유형 선택은 컨트롤러/채널 매개변수를 해제합니다" 패턴 —
agent discovers the specific enabler from docs, the procedure stays IP-independent.
에이전트가 문서에서 특정 Enabler를 발견하면 절차는 IP 독립적으로 유지됩니다.

**For `PARAM_NOT_FOUND` errors:** This is the main case requiring a doc search.
**`PARAM_NOT_FOUND` 오류의 경우:** 문서 검색이 필요한 주요 사례입니다.
The parameter name in the error is the one to search for:
오류의 매개변수 이름은 다음과 같이 검색할 이름입니다.
```
vivado_doc_search("<ip_name> <param_from_error> valid parameter names")
```

**For `PARAM_NOT_FOUND` in BD — standalone IP fallback:** Some CONFIG params
**BD의 `PARAM_NOT_FOUND`의 경우 — 독립형 IP 폴백:** 일부 CONFIG 매개변수
exist in standalone mode (`create_ip`) but are hidden in BD mode (`create_bd_cell`).
독립형 모드(`create_ip`)에는 존재하지만 BD 모드(`create_bd_cell`)에서는 숨겨져 있습니다.
When a `PARAM_NOT_FOUND` error occurs for a param that doc search confirms is
문서 검색에서 확인한 매개변수에 대해 'PARAM_NOT_FOUND' 오류가 발생하는 경우
valid for the IP, try the standalone path as a validation mechanism:
IP에 유효한 경우 유효성 검사 메커니즘으로 독립 실행형 경로를 사용해 보세요.

```tcl
# Tier 2 fallback: standalone IP validation
create_ip -vlnv <vlnv> -module_name <name>_standalone -dir /tmp
set_property -dict [list \
    CONFIG.PARAM_A {value_a} \
    CONFIG.PARAM_B {value_b} \
] [get_ips <name>_standalone]
puts "STANDALONE_OK:<id>"
# Cleanup standalone IP
remove_files [get_files /tmp/<name>_standalone/<name>_standalone.xci]
file delete -force /tmp/<name>_standalone
```

Testing showed that originally hypothesized BD-restricted params
테스트 결과 원래 가설은 BD 제한 매개변수인 것으로 나타났습니다.
(`axi_timer` C_ONE_TIMER_ONLY, `axis_interconnect` ARB_ALGORITHM,
(`axis_timer` C_ONE_TIMER_ONLY, `axis_interconnect` ARB_ALGORITHM,
`axis_switch` DECODER_BASE) are actually not configurable in ANY mode
`axis_switch` DECODER_BASE)는 실제로 ANY 모드에서 구성할 수 없습니다.
for current IP versions.
현재 IP 버전의 경우.
The standalone fallback remains available for
독립 실행형 폴백은 계속해서 사용할 수 있습니다.
future cases where a param genuinely exists in standalone but not BD.
매개변수가 독립 실행형에는 실제로 존재하지만 BD는 존재하지 않는 미래의 경우입니다.

The fidelity grade for standalone-validated configs is `full_standalone`.
독립형 검증 구성의 충실도 등급은 `full_standalone`입니다.

**For runtime-only features — annotation instead of failure:** Some requested
**런타임 전용 기능 — 실패 대신 주석:** 일부 요청됨
features are controlled by software register writes, not design-time CONFIG
기능은 디자인 타임 CONFIG가 아닌 소프트웨어 레지스터 쓰기에 의해 제어됩니다.
properties.
속성.
When doc search returns a register map instead of CONFIG params,
문서 검색이 CONFIG 매개변수 대신 레지스터 맵을 반환하는 경우,
emit a structured annotation:
구조화된 주석을 내보냅니다.

```
RUNTIME_CONFIG:<id>:<feature>=<value>:<register>[<bits>]@<offset>
```

Examples:
예:
- `RUNTIME_CONFIG:60:CPOL=1,CPHA=1:SPICR[4:3]@0x60`
- `RUNTIME_CONFIG:60:CPOL=1,CPHA=1:SPICR[4:3]@0x60`
- `RUNTIME_CONFIG:100:SPEED_P0=100G,FEC=RS-FEC:PORT0_CONFIGURATION_REV1[7:0]@0x0100`
- `RUNTIME_CONFIG:100:SPEED_P0=100G,FEC=RS-FEC:PORT0_CONFIGURATION_REV1[7:0]@0x0100`

The benchmark result gets a `runtime_params` field:
벤치마크 결과는 `runtime_params` 필드를 얻습니다.

```json
{
  "runtime_params": {
    "CPOL": {"value": 1, "register": "SPICR", "bit": 3, "offset": "0x60"},
    "CPHA": {"value": 1, "register": "SPICR", "bit": 4, "offset": "0x60"}
  }
}
```

IPs with known runtime-only features:
알려진 런타임 전용 기능이 있는 IP:

| IP | Runtime Feature<br>런타임 기능 | Register<br>등록 | Notes<br>메모 |
|---|---|---|---|
| `axi_quad_spi`<br>`axi_quad_spi` | CPOL, CPHA<br>CPOL, CPHA | SPICR (0x60)<br>스파이커(0x60) | SPI clock polarity/phase<br>SPI 클록 극성/위상 |
| `mrmac`<br>'mrmac' | Port speed, FEC, PTP<br>포트 속도, FEC, PTP | Per-port config registers<br>포트별 구성 레지스터 | All port config is runtime<br>모든 포트 구성은 런타임입니다. |
| `axi_ethernet`<br>`축_이더넷` | DMA enable<br>DMA 활성화 | External — DMA is a separate IP<br>외부 — DMA는 별도의 IP입니다. | Not a register; requires companion IP<br>레지스터가 아닙니다. 컴패니언 IP가 필요합니다 |
| `v_tc`<br>`v_tc` | GEN_* timing params<br>GEN_* 타이밍 매개변수 | Generation registers (0x60-0x84)<br>생성 레지스터(0x60-0x84) | 1080p60 timing is set by VTC driver<br>1080p60 타이밍은 VTC 드라이버에 의해 설정됩니다. |

When runtime params are identified, the design-time CONFIG params that ARE
런타임 매개변수가 식별되면 디자인 타임 CONFIG 매개변수는
available should still be set via `set_property -dict`.
available은 여전히 `set_property -dict`를 통해 설정되어야 합니다.
The fidelity grade
충실도 등급
becomes `full` (with `runtime_params` populated) since the design-time config
디자인 타임 구성 이후 `full`(`runtime_params`가 채워짐)이 됩니다.
is complete and the runtime portion is documented.
완료되었으며 런타임 부분이 문서화되었습니다.

**Give-up gate — do NOT grade `creation_only`/`partial` prematurely.** You may never
**포기 관문 - `creation_only`/`partial` 등급을 미리 채점하지 마세요.**
conclude a parameter is integration-derived, connection-derived, gated, or runtime-only
매개변수가 통합 파생, 연결 파생, 게이트형 또는 런타임 전용이라고 결론을 내립니다.
**from reasoning alone**.
**추론만으로**.
Before downgrading a requirement, you MUST have evidence of all:
요구 사항을 다운그레이드하기 전에 다음 사항에 대한 증거가 있어야 합니다.
1. **You actually attempted the literal value with `set_property` and it failed/reverted.**
1. **실제로 `set_property`를 사용하여 리터럴 값을 시도했지만 실패/복귀했습니다.**
A param you merely *assumed* is derived does not count.
단순히 *가정* 파생되었다고 가정한 매개변수는 포함되지 않습니다.
In particular:
특히:
   - **Try the global / top-level key form, not just per-instance keys.** Many counts and
   - **단순히 인스턴스별 키가 아닌 전역/최상위 키 형식을 사용해 보세요.** 개수가 많고
widths are set by a single global `CONFIG.<X>` even when per-port `CONFIG.<Sn>_<X>`
너비는 포트별 `CONFIG.<Sn>_<X>`인 경우에도 단일 전역 `CONFIG.<X>`에 의해 설정됩니다.
keys are read-only (e.g. a NoC data width applied once globally rather than per slave
키는 읽기 전용입니다(예: 슬레이브당이 아닌 전체적으로 한 번 적용되는 NoC 데이터 너비
port).
포트).
Attempt the global key before declaring the value connection-derived.
연결 파생 값을 선언하기 전에 전역 키를 시도하십시오.
   - Run `verify_stuck` to confirm it genuinely reverted, not that you skipped it.
   - 건너뛴 것이 아니라 정말로 되돌려졌는지 확인하려면 `verify_stuck`을 실행하세요.
2. **Enabler search attempted** (if disabled/ignored): you ran the `PARAM_DISABLED`
2. **인에이블러 검색 시도**(비활성화/무시된 경우): `PARAM_DISABLED`를 실행했습니다.
reconciliation (below) and, for any `DIFFER` key, doc-searched + set the gating parent
조정(아래) 및 'DIFFER' 키에 대해 문서 검색 + 게이팅 상위 설정
first — and the child *still* would not take.
먼저 — 그런데 그 아이는 *여전히* 받아들이려고 하지 않았습니다.
3. **Block Automation checked** (for integration/connectivity features): you ran
3. **블록 자동화 확인**(통합/연결 기능용): 실행했습니다.
`ipcfg::is_system_intent` and, if `SYSTEM`, enumerated the loaded rules with
`ipcfg::is_system_intent`, `SYSTEM`인 경우 로드된 규칙을 다음과 같이 열거합니다.
`::bd::util_cmd rules dump` (→ `ipcfg::rule_for_vlnv`) and either no rule exists for the
`::bd::util_cmd 규칙 덤프`(→ `ipcfg::rule_for_vlnv`) 및
VLNV (and no example design realizes it) or `ipcfg::run_automation` was applied
VLNV(및 이를 실현하는 예제 디자인 없음) 또는 `ipcfg::run_automation`이 적용되었습니다.
**with the rule's options** and its reported changes did not realize the value.
**규칙의 옵션** 및 보고된 변경 사항을 사용하여 값을 실현하지 못했습니다.
Integration-derived
통합 파생
is only valid after automation could not deliver it.
자동화가 이를 전달할 수 없는 경우에만 유효합니다.
4. **Doc confirms no design-time path.** A doc search shows the feature is genuinely
4. **문서에서는 디자인 타임 경로가 없음을 확인했습니다.** 문서 검색에 따르면 이 기능은 실제로 존재하는 것으로 나타났습니다.
runtime-register-only or integration/connection-derived (not just absent from your dict).
런타임 등록 전용 또는 통합/연결 파생(딕셔너리에 없는 것이 아니라).

If any of 1–4 is missing, keep recovering — the downgrade is not yet justified.
1~4개 중 하나라도 누락된 경우 계속 복구하십시오. 다운그레이드가 아직 정당화되지 않았습니다.
Whatever
뭐든지
you finally cannot apply MUST appear in the Step 1.5 ledger as `unapplied:<reason>` and in
최종적으로 신청할 수 없음은 1.5단계 원장에 `unapplied:<reason>`으로 나타나야 하며
the `ipcfg::coverage_report` output (see "Always report coverage").
`ipcfg::coverage_report` 출력("항상 적용 범위 보고" 참조)

**Maximum retries: 2.** If it still fails after 2 Tier-2 attempts, report the error
**최대 재시도 횟수: 2.** 2번의 Tier-2 시도 후에도 여전히 실패하면 오류를 보고하세요.
to the user with the diagnostic info and ask for guidance.
사용자에게 진단 정보를 전달하고 안내를 요청합니다.

## Board binding: the parameters you must not answer yourself
## 보드 바인딩: 스스로 대답해서는 안 되는 매개변수

When the project has a `board_part`, the board file has already answered a large
프로젝트에 `board_part`가 있으면 보드 파일이 이미 큰 응답을 받은 것입니다.
part of the configuration, and every value you write by hand over one of those
구성의 일부이며, 그 중 하나에 직접 작성하는 모든 값
answers is you disagreeing with the board vendor about their own hardware.
대답은 자신의 하드웨어에 대해 보드 공급업체와 동의하지 않는다는 것입니다.

Vivado exposes this the same way on every IP: a `CONFIG.<X>_BOARD_INTERFACE`
Vivado는 이를 모든 IP에서 동일한 방식으로 노출합니다: `CONFIG.<X>_BOARD_INTERFACE`
property.
재산.
Naming a board interface on it applies that interface's preset.
보드 인터페이스의 이름을 지정하면 해당 인터페이스의 사전 설정이 적용됩니다.
Measured on a Versal Gen2 board / 2026.1 (the values each preset writes are the board's own):
Versal Gen2 보드/2026.1에서 측정됨(각 사전 설정이 쓰는 값은 보드 고유의 값임):

| IP | hooks<br>후크 | what the board then owns<br>그렇다면 이사회는 무엇을 소유하고 있는가? |
|----|-------|--------------------------|
| `mipi_csi2_rx_subsystem`<br>`mipi_csi2_rx_subsystem` | `DPHYRX_BOARD_INTERFACE`<br>`DPHYRX_BOARD_INTERFACE` | `CMN_NUM_LANES`, `CMN_NUM_PIXELS`, `CMN_PXL_FORMAT`, `CSI_BUF_DEPTH`, `DPY_LINE_RATE`, `C_DPHY_LANES`<br>`CMN_NUM_LANES`, `CMN_NUM_PIXELS`, `CMN_PXL_FORMAT`, `CSI_BUF_DEPTH`, `DPY_LINE_RATE`, `C_DPHY_LANES` |
| `ps_wizard`<br>`ps_마법사` | `PS_BOARD_INTERFACE`<br>`PS_BOARD_INTERFACE` | `PS11_CONFIG`, `MMI_CONFIG`, `PS11_CONFIG_APPLIED` 0→1<br>`PS11_CONFIG`, `MMI_CONFIG`, `PS11_CONFIG_APPLIED` 0→1 |
| `axi_noc2`<br>`axi_noc2` | 28, e.g. `C0_CH0_LPDDR5_BOARD_INTERFACE`<br>28(예: `C0_CH0_LPDDR5_BOARD_INTERFACE` | `NUM_MC`, `NUM_MCP`, `DDRMC5_CONFIG` (the whole memory configuration: device type and speed, width, JEDEC timing tables), `DDRMC5_INTERLEAVE_SIZE`<br>`NUM_MC`, `NUM_MCP`, `DDRMC5_CONFIG`(전체 메모리 구성: 장치 유형 및 속도, 너비, JEDEC 타이밍 테이블), `DDRMC5_INTERLEAVE_SIZE` |
| `axi_gpio`<br>`axi_gpio` | `GPIO_BOARD_INTERFACE`, `GPIO2_BOARD_INTERFACE`<br>`GPIO_BOARD_INTERFACE`, `GPIO2_BOARD_INTERFACE` | `C_GPIO_WIDTH`<br>`C_GPIO_WIDTH` |
| `axi_iic`<br>`axi_iic` | `IIC_BOARD_INTERFACE`<br>`IIC_BOARD_INTERFACE` | pin-out constraints<br>핀아웃 제약 |

So the rule is generic, and `ipcfg::board_bind` implements it without knowing
따라서 규칙은 일반적이며 `ipcfg::board_bind`는 이를 알지 못한 채 구현합니다.
any of these parameter names — it discovers the hooks with `board_hooks`, binds
이러한 매개변수 이름 중 하나 — `board_hooks`로 후크를 발견하고 바인드합니다.
them, and diffs `CONFIG.*` across the bind to learn what the preset wrote:
그리고 바인드 전체에서 'CONFIG.*'를 비교하여 사전 설정이 작성한 내용을 알아봅니다.

```tcl
ipcfg::board_bind cam {CONFIG.DPHYRX_BOARD_INTERFACE <dphy_board_if>} {} "one camera"
# BOARD:bound cam hooks=1 owned=7 {CMN_NUM_LANES CMN_NUM_PIXELS CMN_PXL_FORMAT
#   CSI_BUF_DEPTH C_DPHY_LANES DPHYRX_BOARD_INTERFACE DPY_LINE_RATE}
```

Two things then hold.
그러면 두 가지가 유지됩니다.
`ipcfg::apply_dict` refuses a later write to any of those
`ipcfg::apply_dict`는 나중에 해당 항목에 대한 쓰기를 거부합니다.
keys (`CONFIGURE_FAIL:BOARD_OWNED:...`).
키(`CONFIGURE_FAIL:BOARD_OWNED:...`).
For a spec-driven deviation use
사양에 따른 편차 사용의 경우
`ipcfg::require <cell> <dict> <reason> {} {<spec statement>}` (the 5th argument,
`ipcfg::require <cell> <dict> <reason> {} {<사양 설명>}` (5번째 인수,
`board_override`), which records the reason; `ipcfg::audit_board` at the end fails on
`board_override`), 이유를 기록합니다. 마지막에 `ipcfg::audit_board`가 실패합니다.
anything that moved without one and lists the recorded overrides.
하나도 없이 이동한 모든 항목과 기록된 재정의 목록을 나열합니다.
Both matter because overwriting a preset does not throw.
사전 설정을 덮어써도 문제가 발생하지 않기 때문에 둘 다 중요합니다.

### A dict property is REPLACE, not merge — and nothing says so
### dict 속성은 병합이 아니라 REPLACE입니다. 그리고 아무것도 그렇게 말하지 않습니다.

This is the sharpest edge in the whole area.
이것은 전체 지역에서 가장 날카로운 가장자리입니다.
After binding the board,
보드를 묶은 후,
`ps_wizard`'s `CONFIG.PS11_CONFIG` holds the whole preset, dozens of sub-keys.
`ps_wizard`의 `CONFIG.PS11_CONFIG`에는 전체 사전 설정, 수십 개의 하위 키가 포함되어 있습니다.
One innocuous-looking
무해해 보이는 한 마리

```tcl
set_property CONFIG.PS11_CONFIG {PS_NUM_FABRIC_RESETS 1} [get_bd_cells ps]
```

leaves **only a handful**.
**한줌**만 남습니다.
The ones that vanish include `PMC_CRP_PL0/PL1/PL2_REF_CTRL_FREQMHZ`,
사라지는 것에는 `PMC_CRP_PL0/PL1/PL2_REF_CTRL_FREQMHZ`가 포함됩니다.
the PL clock frequencies the entire fabric is clocked from, and the write
전체 패브릭이 클럭되는 PL 클럭 주파수 및 쓰기
returns success.
성공을 반환합니다.
Nothing in `verify_stuck` or `verify_intent` sees it either:
`verify_stuck` 또는 `verify_intent`에서는 다음 중 어느 것도 이를 볼 수 없습니다.
the sub-key the caller asked for did take.
발신자가 요청한 하위 키가 사용되었습니다.

`ipcfg::apply_dict` therefore folds a write to a board-owned dict into what is
따라서 `ipcfg::apply_dict`는 보드 소유 dict에 대한 쓰기를 다음과 같이 접습니다.
already on the cell, and says so (`merged into board-owned PS11_CONFIG`).
이미 셀에 있고 그렇게 말합니다(`보드 소유 PS11_CONFIG`에 병합됨).
The
는
conflict check runs first, so merging can never quietly re-assert a preset value
충돌 검사가 먼저 실행되므로 병합이 미리 설정된 값을 자동으로 다시 주장할 수 없습니다.
over an intended change.
의도된 변화에 대해
Measured after the fix: the same write keeps all but one
수정 후 측정: 동일한 쓰기가 하나만 제외하고 모두 유지됩니다.
of the preset sub-keys and `audit_board` reports a clean `BOARD:OK`.
사전 설정된 하위 키와 'audit_board'가 깨끗한 'BOARD:OK'를 보고합니다.

### Deviating from the board on purpose
### 고의로 보드에서 벗어나는 행위

A spec that excludes hardware the board preset enables has to overrule the board
보드 사전 설정을 통해 하드웨어를 제외하는 사양은 보드를 압도해야 합니다.
on exactly the sub-key that carries it, with the deviation flag so the audit
편차 플래그를 사용하여 이를 전달하는 하위 키에 대해 감사를 수행합니다.
records a decision rather than drift:
표류보다는 결정을 기록합니다.

```tcl
ipcfg::apply_dict ps {CONFIG.MMI_CONFIG {<key> <value>}} 1   ;# 1 = deliberate
```

Two things decide whether that write is safe, and neither is visible in its
두 가지 사항에 따라 해당 쓰기가 안전한지 여부가 결정되며 둘 다 해당 항목에 표시되지 않습니다.
result.
결과.
The first is how the cell was bound.
첫 번째는 셀이 바인딩된 방식입니다.
Measured on a Versal Gen2 board with one key
하나의 키가 있는 Versal Gen2 보드에서 측정됨
written to `MMI_CONFIG`: after `ipcfg::board_bind`, Vivado **refused** the write
`MMI_CONFIG`에 기록됨: `ipcfg::board_bind` 이후 Vivado가 쓰기를 **거부**했습니다.
and left every sub-key and a clean `BOARD:OK` standing; after a bare
모든 하위 키와 깨끗한 'BOARD:OK'를 그대로 두었습니다. 맨손으로
`set_property CONFIG.PS_BOARD_INTERFACE`, the same write **succeeded** and left only
`set_property CONFIG.PS_BOARD_INTERFACE`, 동일한 쓰기 **성공** 및 왼쪽만 남음
a few sub-keys, having dropped the DisplayPort with the rest.
몇 개의 하위 키, 나머지와 함께 DisplayPort를 삭제했습니다.
The refusal is the good
거절은 좋은 것이다
outcome — `ipcfg::autofix_apply` isolates the key Vivado would not take — and it
결과 - `ipcfg::autofix_apply`는 Vivado가 취하지 않는 키를 분리합니다.
only happens for a cell whose ownership was recorded.
소유권이 기록된 셀에 대해서만 발생합니다.

The second is whether the preset enabled the hardware at all, which is worth
두 번째는 사전 설정이 하드웨어를 전혀 활성화했는지 여부입니다.
reading before writing anything.
무엇이든 쓰기 전에 읽는 것.
`MMI_CONFIG_INTERNAL` holds the resolved state
`MMI_CONFIG_INTERNAL`은 해결된 상태를 유지합니다.
(thousands of sub-keys against `MMI_CONFIG`'s few dozen) and is a **readback**: `apply_dict` to
(`MMI_CONFIG`의 수십 개에 비해 수천 개의 하위 키) **읽기**: `apply_dict`
it returns `SUCCESS:applied` and changes nothing.
`SUCCESS:applied`를 반환하고 아무것도 변경하지 않습니다.

`UDH_GT` is the one to be careful with, because it is not a feature's switch: it
`UDH_GT`는 기능의 스위치가 아니기 때문에 주의해야 합니다.
names the protocol the **whole GT quad** carries, so writing one protocol removes
**전체 GT 쿼드**가 전달하는 프로토콜의 이름을 지정하므로 하나의 프로토콜을 작성하면 제거됩니다.
the others along with their sub-keys (`MMI_DP_ENABLE_BEFORE_PL` among them).
다른 키와 하위 키(그 중 `MMI_DP_ENABLE_BEFORE_PL`)
When
언제
the preset selects a combined DisplayPort + USB protocol, a run that wrote `UDH_GT USB` while excluding
사전 설정은 결합된 DisplayPort + USB 프로토콜을 선택하고 'UDH_GT USB'를 쓴 실행을 제외합니다.
PCIe shipped a display design with no DisplayPort.
PCIe는 DisplayPort가 없는 디스플레이 디자인을 출시했습니다.
An absent
결석
`MMI_DP_ENABLE_BEFORE_PL` is correct for hardware that has no DP and a defect for
`MMI_DP_ENABLE_BEFORE_PL`은 DP가 없고 결함이 있는 하드웨어에 적합합니다.
anything that sends a frame to one — `ipiasm::display_note` reports which case a
프레임을 보내는 모든 것 - `ipiasm::display_note`는 어떤 경우인지 보고합니다.
design is in.
디자인이 들어있습니다.

### The enable flag, and the decoy
### 활성화 플래그 및 미끼

Some IP gate the hook behind a boolean, and until it is set the preset's values
일부 IP는 부울 뒤의 후크를 게이트하고 사전 설정 값이 설정될 때까지
are *outside the parameter's legal range*, so the bind is rejected outright.
*매개변수의 법적 범위를 벗어났기 때문에* 바인딩이 완전히 거부됩니다.
`axi_noc2` is the case in hand: without `CONFIG.MC_BOARD_INTRF_EN` the preset's
`axi_noc2`가 해당 사례입니다. `CONFIG.MC_BOARD_INTRF_EN`이 없으면 사전 설정의
`DDRMC5_INTERLEAVE_SIZE=2048` fails with "Valid values are - 0".
`DDRMC5_INTERLEAVE_SIZE=2048`이 "유효한 값은 - 0"으로 실패합니다.
`board_bind` handles this by retrying behind the flags `board_enablers` finds,
`board_bind`는 `board_enablers`가 찾은 플래그 뒤에서 재시도하여 이를 처리합니다.
and says which one it needed.
그리고 어느 것이 필요한지 말합니다.

Note that `DDRMC5_BOARD_INTRF_EN`, *inside* the `CONFIG.DDRMC5_CONFIG` dict, is
`CONFIG.DDRMC5_CONFIG` 딕셔너리 *내부*인 `DDRMC5_BOARD_INTRF_EN`은 다음과 같습니다.
a decoy: the board preset itself sets it false, and it is not the control.
미끼: 보드 사전 설정 자체가 이를 false로 설정하며 이는 제어가 아닙니다.
A run
달리기
that went after it instead spent 35 minutes trying to make it stick, reported
그 후에는 그것을 붙이기 위해 35분을 소비했다고 보고되었습니다.
"the tool keeps reverting that dict field", hand-edited `DDRMC5_CONFIG` to force
"도구가 해당 dict 필드를 계속 되돌립니다", `DDRMC5_CONFIG`를 직접 편집하여 강제로
it, desynced the NoC-General controller count from the DDRMC preset count, and
DDRMC 사전 설정 카운트에서 NoC-General 컨트롤러 카운트를 비동기화하고
shipped a design with no addressable DRAM window.
주소 지정이 가능한 DRAM 창이 없는 디자인을 출시했습니다.
The real control is the
진짜 통제는
top-level `MC_BOARD_INTRF_EN`, and `board_bind` finds it by inspection.
최상위 `MC_BOARD_INTRF_EN` 및 `board_bind`가 검사를 통해 찾습니다.

### Why not block automation
### 자동화를 차단하지 않는 이유

`apply_bd_automation -rule xilinx.com:bd_rule:axi_noc`, which the Vivado design
Vivado 설계인 `apply_bd_automation -rule xilinx.com:bd_rule:axi_noc`
tutorials use to attach memory, **does not exist for `axi_noc2` in 2026.1** — it
튜토리얼은 메모리를 연결하는 데 사용됩니다. **2026.1의 `axi_noc2`에는 존재하지 않습니다** —
returns `[BD 41-569] The rule ... cannot be found`.
`[BD 41-569] 규칙을 찾을 수 없습니다'를 반환합니다.
`xilinx.com:bd_rule:axi_noc2`
`xilinx.com:bd_rule:axi_noc2`
and `ps_wizard` automation do exist, but read their live schemas with
및 `ps_wizard` 자동화가 존재하지만 다음을 사용하여 라이브 스키마를 읽습니다.
`ipcfg::automation_rule_options` first: on a board whose memory is declared as board
`ipcfg::automation_rule_options` 첫 번째: 메모리가 보드로 선언된 보드에서
interfaces they may offer only `None` for the controller count, and
인터페이스는 컨트롤러 수에 대해 '없음'만 제공할 수 있습니다.
`apply_board_connection` needs an existing controller pin.
`apply_board_connection`에는 기존 컨트롤러 핀이 필요합니다.
Board binding is then the
보드 바인딩은 다음과 같습니다.
working path for Gen2 NoC memory, so `ipiasm::attach_board_memory` wraps
Gen2 NoC 메모리의 작업 경로이므로 `ipiasm::attach_board_memory`가 래핑됩니다.
`board_bind` rather than the documented automation.
문서화된 자동화가 아닌 `board_bind`.

### Board memory recipe (axi_noc2, 2026.1)
### 보드 메모리 레시피(axi_noc2, 2026.1)

`ipiasm::attach_board_memory <noc> <n> <DesignSpec regions>` (`{name base size}` dicts, or
`ipiasm::attach_board_memory <noc> <n> <DesignSpec 지역>` (`{이름 기본 크기}` 사전, 또는
explicit `{MC_CHAN_REGION0 <r0> MC_CHAN_REGION1 <r1>}` pairs)
명시적 `{MC_CHAN_REGION0 <r0> MC_CHAN_REGION1 <r1>}` 쌍)
sets, in ONE `set_property -dict`: the `Cn_CH0_<family>_BOARD_INTERFACE` hooks,
하나의 `set_property -dict`에 설정: `Cn_CH0_<family>_BOARD_INTERFACE` 후크,
`NUM_MC`, and the region selectors the spec needs.
`NUM_MC` 및 사양에 필요한 지역 선택기입니다.
Selectors are chosen from the IP's
선택자는 IP에서 선택됩니다.
parameter-dependency dump, whose range list gives every legal selector with its address
범위 목록이 모든 적합한 선택기에 해당 주소를 제공하는 매개변수 종속성 덤프
range (read from a scratch managed IP, cached per VLNV; the block design is not touched).
범위(VLNV별로 캐시된 스크래치 관리 IP에서 읽음, 블록 디자인은 건드리지 않음)
Binding the hooks alone yields
후크만 바인딩하면 결과가 나옵니다.
`NUM_MC`, `NUM_MCP`, `MC_BOARD_INTRF_EN` and whatever the board's memory presets
`NUM_MC`, `NUM_MCP`, `MC_BOARD_INTRF_EN` 및 보드의 메모리 사전 설정
declare (typically `DDRMC5_CONFIG` and `DDRMC5_INTERLEAVE_SIZE`); do not write those.
선언(일반적으로 `DDRMC5_CONFIG` 및 `DDRMC5_INTERLEAVE_SIZE`); 그런 것 쓰지 마세요.
Check the board's `preset.xml`: if it carries no `MC_CHAN_REGION*`, any region beyond
보드의 `preset.xml`을 확인하세요. 보드에 `MC_CHAN_REGION*`이 없으면 그 너머의 모든 지역
the IP default must ride in the same bind.
IP 기본값은 동일한 바인드에 있어야 합니다.
Region selectors are otherwise
지역 선택기는 그렇지 않습니다.
design-owned (PG456: Region 0 defaults `DDR_CH0_LEGACY`, Region 1 defaults `NONE`).
디자인 소유(PG456: 지역 0의 기본값은 'DDR_CH0_LEGACY', 지역 1의 기본값은 'NONE').
Writing a region in a later `set_property` can reset preset-owned `DDRMC5_CONFIG`
나중에 `set_property`에 영역을 작성하면 사전 설정 소유 `DDRMC5_CONFIG`를 재설정할 수 있습니다.
keys, so the helper refuses to report success on preset drift.
키를 사용하므로 도우미는 사전 설정된 드리프트에 대한 성공 보고를 거부합니다.
No DDR window exists
DDR 창이 존재하지 않습니다.
until a NoC input is connected to an MC port (`CONFIG.CONNECTIONS {MC_0 ...}` on the
NoC 입력이 MC 포트('CONFIG.CONNECTIONS {MC_0 ...}')에 연결될 때까지
NoC slave/INI interface, e.g.
NoC 슬레이브/INI 인터페이스, 예:
`ipiasm::add_noc_si`); only then check windows and address.
`ipiasm::add_noc_si`); 그런 다음 창과 주소를 확인하십시오.

## Performance Targets
## 성과 목표

| Metric<br>미터법 | Target<br>대상 |
|--------|--------|
| Tier 1 success rate (easy prompts)<br>Tier 1 성공률(쉬운 프롬프트) | >90% |
| Tier 1 success rate (medium prompts)<br>Tier 1 성공률(중간 메시지) | >70% |
| Tier 1 success rate (hard prompts)<br>Tier 1 성공률(하드 프롬프트) | >40% |
| Tier 2 recovery rate (when Tier 1 fails)<br>Tier 2 복구율(Tier 1이 실패하는 경우) | >80% |
| True fidelity rate (full + full_standalone)<br>실제 충실도 비율(full + full_standalone) | >90% |
| Total MCP calls (Tier 1, no part swap)<br>총 MCP 통화(계층 1, 부품 교환 없음) | 2-3 |
| Total MCP calls (Tier 1, with part swap)<br>총 MCP 통화(Tier 1, 부품 교체 포함) | 3-4 |
| Total MCP calls (Tier 2 recovery)<br>총 MCP 통화(계층 2 복구) | 4-6 |
| Total MCP calls (Phase 4 stub connections)<br>총 MCP 호출(4단계 스텁 연결) | +1 (added to Tier 1 or 2)<br>+1(Tier 1 또는 2에 추가됨) |
| Total MCP calls (standalone fallback)<br>총 MCP 호출(독립형 대체) | +1 (Tier 2 only)<br>+1(티어 2에만 해당) |

## Fidelity Grades
## 충실도 등급

| Grade<br>등급 | Meaning<br>의미 |
|-------|---------|
| `full`<br>'가득' | CONFIG params match prompt intent; runtime params documented if applicable<br>CONFIG 매개변수는 프롬프트 의도와 일치합니다. 해당되는 경우 런타임 매개변수를 문서화합니다. |
| `full_standalone`<br>`완전 독립형` | Config validated via `create_ip` (standalone); param unavailable in BD mode<br>'create_ip'(독립형)을 통해 구성이 검증되었습니다. BD 모드에서는 매개변수를 사용할 수 없습니다. |
| `partial`<br>'부분적' | IP configured but some features not settable via CONFIG or connections<br>IP가 구성되었지만 일부 기능은 CONFIG 또는 연결을 통해 설정할 수 없습니다. |
| `negative`<br>'부정적' | Prompt asked for values outside Vivado's valid range<br>Vivado의 유효 범위를 벗어난 값을 요청하는 프롬프트 |
| `creation_only`<br>`창조_만` | IP created but key requested features aren't CONFIG properties<br>IP가 생성되었지만 주요 요청 기능이 CONFIG 속성이 아닙니다. |

## What NOT to do
## 하지 말아야 할 것

- **Do not query parameters one at a time** — always use `set_property -dict` for batch ops
- **한 번에 하나씩 매개변수를 쿼리하지 마세요** — 일괄 작업에는 항상 `set_property -dict`를 사용하세요.
- **Do not try to read the full parameter space** before configuring — go straight to the
- **구성하기 전에 전체 매개변수 공간을 읽으려고 하지 마십시오** — 바로
-dict. **Exception (sanctioned):** a *keyword-scoped* `ipcfg::discover_params <vlnv>
-dict. **예외(승인됨):** *키워드 범위* `ipcfg::discover_params <vlnv>
  <keyword>` on a throwaway scratch cell, used to map a requirement to its COMPLETE sibling
set.
설정합니다.
That is targeted neighborhood discovery, not a blind full dump, and is the only
이는 블라인드 전체 덤프가 아닌 타겟 동네 검색이며 유일한 방법입니다.
permitted `list_property` use.
`list_property` 사용이 허용됩니다.
- **Do not retry more than 2 times** — escalate to user
- **2번 이상 재시도하지 마세요** — 사용자에게 에스컬레이션하세요.
- **Do not hardcode IP versions** — use version-free VLNVs
- **IP 버전을 하드코딩하지 마세요** — 버전이 없는 VLNV를 사용하세요.
- **Do not issue individual `set_property` calls** for each parameter — one `-dict`, one call
- **각 매개변수에 대해 개별 `set_property` 호출을 실행하지 마세요** — 하나의 `-dict`, 하나의 호출
- **Do not use Phase 4 stubs when the param IS settable via CONFIG** — stubs are only for
- **CONFIG를 통해 매개변수를 설정할 수 있는 경우 4단계 스텁을 사용하지 마세요** — 스텁은
genuinely connection-derived read-only params
실제로 연결에서 파생된 읽기 전용 매개변수
- **Do not mix standalone and BD validation** in the same script — standalone is a separate path
- **독립 실행형과 BD 유효성 검사를 동일한 스크립트에 혼합하지 마십시오** — 독립 실행형은 별도의 경로입니다.
- **Do not trust a `catch` of `0` (or a `SUCCESS:` line) alone in BD mode** — unknown params
- **BD 모드에서 `0`(또는 `SUCCESS:` 줄)의 `catch`만 신뢰하지 마세요** — 알 수 없는 매개변수
emit non-fatal CRITICAL WARNINGs; always run the Phase 3 verification loop and scan output
치명적이지 않은 중요 경고를 발산합니다. 항상 3단계 검증 루프를 실행하고 출력을 스캔합니다.
- **Do not configure an IP that the docs show cannot deliver the requested capability** —
- **문서에서 요청한 기능을 제공할 수 없는 IP는 구성하지 마세요** —
grade `negative` and name the correct IP instead of forcing an ignored setting
무시된 설정을 강제하는 대신 '부정적' 등급을 지정하고 올바른 IP 이름을 지정하세요.
- **Do not spend a doc-search call on a `VALUE_OUT_OF_RANGE` error** — the valid set is in
- **`VALUE_OUT_OF_RANGE` 오류에 대해 문서 검색 호출을 낭비하지 마세요** — 유효한 세트는
the error text; parse it directly
오류 텍스트; 직접 분석해 보세요
- **Do not hand-roll the repetitive TCL** (part swap, create, dict+verify, stub, cleanup) —
- **반복적인 TCL을 직접 작성하지 마세요**(부분 교환, 생성, dict+검증, 스텁, 정리) —
call the `ipcfg::*` helpers from `lib/ipcfg.tcl` (sourced once per session); drop to raw
`lib/ipcfg.tcl`에서 `ipcfg::*` 도우미를 호출합니다(세션당 한 번씩 소스 제공). 원시로 드롭
TCL only for a quirk no helper covers
도우미가 포함되지 않은 특이한 TCL 전용
- **Do not declare success on an empty `catch` in BD mode** — a non-empty `STUCK_FAIL` list
- **BD 모드에서 빈 `catch`에 성공을 선언하지 마세요** — 비어 있지 않은 `STUCK_FAIL` 목록
from `ipcfg::verify_stuck` means a key reverted; recover before grading `full`
`ipcfg::verify_stuck`은 키가 되돌려졌다는 의미입니다. '전체' 등급을 매기기 전에 복구
- **Do not set an attribute and call the feature delivered** — `RESET_TYPE` without
- **속성을 설정하지 말고 제공된 기능을 호출하세요** — `RESET_TYPE` 없이
`USE_RESET`, a parity setting without `C_USE_PARITY`, an SG width without `c_include_sg`:
`USE_RESET`, `C_USE_PARITY`가 없는 패리티 설정, `c_include_sg`가 없는 SG 너비:
the write sticks and the port never appears.
쓰기가 중단되고 포트가 나타나지 않습니다.
`ipcfg::check_enablers` must come back empty
`ipcfg::check_enablers`는 비어 있어야 합니다.
- **Do not guess a reverted param's value shape** — read it back with `ipcfg::param_format`
- **되돌린 매개변수의 값 형태를 추측하지 마세요** — `ipcfg::param_format`으로 다시 읽어보세요
and mirror the structure (comma list / nested `{{...}}`) instead of a per-IP format table
IP별 형식 테이블 대신 구조(쉼표 목록/중첩 `{{...}}`)를 미러링합니다.
- **Do not keep fighting a width that is read-only on the boundary port too** — it is
- **경계 포트에서도 읽기 전용인 너비와 계속 싸우지 마세요** — 그렇습니다.
integration-derived; grade `partial` and record it in `runtime_params`
통합 파생; `partial` 등급을 매기고 `runtime_params`에 기록합니다.
- **Do not pick a param by name similarity** — ground every requirement in the param's
- **이름 유사성으로 매개변수를 선택하지 마세요** — 매개변수의 모든 요구사항을 근거로 합니다.
documented *description*; same-keyword params often implement different functions
문서화된 *설명*; 동일한 키워드 매개변수는 종종 다른 기능을 구현합니다.
- **Do not trust a stuck value that equals the default** for an enable/select requirement —
- **활성화/선택 요구 사항에 대한 기본값과 동일한 고정 값을 신뢰하지 마십시오** —
`verify_intent` flags it `suspect-no-change`; it usually means you set the wrong param
`verify_intent`는 이를 `suspect-no-change`로 표시합니다. 이는 일반적으로 잘못된 매개변수를 설정했음을 의미합니다.
- **Do not grade `creation_only` on a disabled/ignored param** before running the
- **비활성화/무시된 매개변수에 대해 `creation_only` 등급을 매기지 마세요**
`PARAM_DISABLED` reconciliation (`ipcfg::reconcile_disabled`; only `DIFFER` keys need an
`PARAM_DISABLED` 조정(`ipcfg::reconcile_disabled`; `DIFFER` 키에만
enabler — `OMIT` keys already hold the wanted value)
인에이블러 - 'OMIT' 키에 이미 원하는 값이 포함되어 있음)
- **Do not declare SUCCESS on the `SUCCESS:` line alone** — also require empty
- **`SUCCESS:` 줄에서만 SUCCESS를 선언하지 마세요** — 또한 비어 있어야 합니다.
`verify_intent` (hard reverts), empty `find_disabled`, and full Step 1.5 ledger coverage
`verify_intent`(하드 되돌리기), 빈 `find_disabled` 및 전체 1.5단계 원장 적용 범위
- **Do not declare a param "derived"/"runtime"/"gated" from reasoning** — you must have an
- **추론에서 매개변수 "파생"/"런타임"/"게이트"를 선언하지 마세요** —
actual failed `set_property` (including the global/top-level key form), enumerated the
실제로 실패한 `set_property`(전역/최상위 키 형식 포함), 열거된
loaded automation rules (`::bd::util_cmd rules dump` → `ipcfg::rule_for_vlnv`), and a doc
로드된 자동화 규칙(`::bd::util_cmd 규칙 덤프` → `ipcfg::rule_for_vlnv`) 및 문서
citation first (see the give-up gate)
먼저 인용(포기 게이트 참조)
- **Do not swap parts silently mid-design** — `ipcfg::ensure_part` is guarded; on
- **설계 도중에 조용히 부품을 교체하지 마세요** — `ipcfg::ensure_part`는 보호됩니다. 에
`WARN:SWAP_BLOCKED` confirm with the user (interactive mode) before forcing
`WARN:SWAP_BLOCKED`를 강제하기 전에 사용자에게 확인(대화형 모드)하세요.
- **Do not treat `suspect-no-change` as an automatic failure** — a doc-correct param whose
- **'suspect-no-change'를 자동 실패로 처리하지 마십시오** — 문서에 맞는 매개변수
default satisfies intent is `value_src=default`, which is acceptable
기본 만족 의도는 'value_src=default'이며 이는 허용됩니다.
- **Do not finish without disclosing coverage** — when the IP is not fully parameterized,
- **범위를 공개하지 않고 끝내지 마십시오** — IP가 완전히 매개변수화되지 않은 경우,
emit `ipcfg::coverage_report` and tell the user the exact prompt fragment(s) you could not
`ipcfg::coverage_report`를 내보내고 사용자에게 전달하지 못한 정확한 프롬프트 조각을 알려줍니다.
apply and why
신청하고 그 이유
- **Do not create-then-configure in two calls when one works** — prefer
- **하나가 작동할 때 두 호출로 생성 후 구성하지 마세요** — 선호
`ipcfg::create_cell_cfg` (single `create_bd_cell -set_param`); fall back to two-step only
`ipcfg::create_cell_cfg`(단일 `create_bd_cell -set_param`); 2단계로만 돌아가기
when needed for ordered/multi-pass config
주문/다중 패스 구성이 필요한 경우
