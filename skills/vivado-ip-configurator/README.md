<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# IP Configurator -- Usage Guide
# IP 구성기 - 사용 가이드

## What This Skill Does
## 이 기술의 역할

Turns a plain-language description of what you want an IP to do into the
IP에서 수행하려는 작업에 대한 일반 언어 설명을
`set_property -dict` that actually does it, then proves the IP came out that way.
실제로 그렇게 하는 `set_property -dict`는 IP가 그런 식으로 나왔다는 것을 증명합니다.

There is no parameter database behind it.
그 뒤에는 매개변수 데이터베이스가 없습니다.
Parameter names come from AMD documentation and
매개변수 이름은 AMD 문서에서 따왔습니다.
from Vivado's own rejection messages, so the skill works on IP it has never seen and on IP
Vivado 자체 거부 메시지를 통해 기술이 본 적이 없는 IP와 IP에서 작동합니다.
whose parameters changed between releases.
릴리스 간에 매개변수가 변경되었습니다.

The reason it verifies rather than just applies: **Vivado accepts writes that do nothing.**
단순히 적용하기보다는 검증하는 이유: **Vivado는 아무것도 하지 않는 쓰기를 허용합니다.**

- An unknown `CONFIG.*` key in a block design does not throw.
- 블록 디자인의 알 수 없는 'CONFIG.*' 키가 발생하지 않습니다.
- It raises a non-fatal
- 치명적이지 않은 결과를 초래합니다.
`CRITICAL WARNING [BD 41-1276]`, `catch` returns 0, and a script that trusts the exit
`CRITICAL WARNING [BD 41-1276]`, `catch`는 0을 반환하고 종료를 신뢰하는 스크립트
status reports success on a parameter that was ignored.
상태는 무시된 매개변수에 대한 성공을 보고합니다.
- A gated attribute reads back cleanly while the feature stays off.
- 게이트 속성은 기능이 꺼져 있는 동안 깨끗하게 다시 읽습니다.
- Set `RESET_TYPE` without
- 없이 `RESET_TYPE`을 설정하세요.
`USE_RESET` and the value sticks, the read-back agrees, and the reset pin never appears.
`USE_RESET` 및 값이 고정되고 다시 읽기가 동의하며 재설정 핀이 나타나지 않습니다.

So every run re-reads the cell, checks that each key exists and stuck, checks that enabling
따라서 실행될 때마다 셀을 다시 읽고, 각 키가 존재하고 멈춰 있는지 확인하고, 활성화되어 있는지 확인합니다.
flags are on, and reports coverage per phrase of your prompt.
플래그가 켜져 있고 프롬프트의 문구별로 적용 범위를 보고합니다.

## Prerequisites
## 전제조건

- **Vivado** with a project open and a **block design open** (the skill operates on
- 프로젝트가 열려 있고 **블록 디자인이 열려 있는 **Vivado**(스킬은 다음에서 작동함)
`get_bd_cells`; there is a standalone `create_ip` fallback for parameters that only exist
`get_bd_cells`; 존재하는 매개변수에 대한 독립형 `create_ip` 대체가 있습니다.
outside IPI).
IPI 외부).
- **Vivado MCP Server** — see [docs/reference/vivado-mcp-tools.md](../../docs/reference/vivado-mcp-tools.md).
- **Vivado MCP 서버** — [docs/reference/vivado-mcp-tools.md](../../docs/reference/vivado-mcp-tools.md)를 참조하세요.
The skill uses exactly three tools: `vivado_doc_search`, `vivado_execute`, and
이 스킬은 정확히 세 가지 도구인 'vivado_doc_search', 'vivado_execute' 및
`vivado_log_messages`.
`vivado_log_messages`.
- An agent that supports skills (Claude Code, Cursor, or the Copilot/VS Code integration).
- 기술(Claude Code, Cursor 또는 Copilot/VS Code 통합)을 지원하는 에이전트입니다.

Validated on **Vivado 2026.1**.
**Vivado 2026.1**에서 검증되었습니다.
Goldens are version-keyed because the IP catalog changes
Goldens는 IP 카탈로그가 변경되기 때문에 버전에 맞춰져 있습니다.
between releases.
릴리스 사이.

## Installation
## 설치

```bash
npx skills add . --skill vivado-ip-configurator
```

## How to Use It
## 사용 방법

Just describe the IP you want.
원하는 IP를 설명하면 됩니다.
The skill triggers on ordinary engineering language — you do
이 기술은 일반적인 엔지니어링 언어에서 실행됩니다.
not name it, and you do not describe its protocol.
이름을 지정하지 않고 프로토콜을 설명하지 않습니다.

```
Add an AXI GPIO to the block design with a 12-bit output-only channel,
all outputs, plus a second 4-bit input channel with interrupts enabled.
```

```
I need a MIPI CSI-2 RX subsystem over D-PHY for a 4-lane camera: 4 active
lanes, 4 pixels per clock, RAW12, all virtual channels, 4096 line-buffer
depth, active-lanes detection and user-data-type filtering on, ISP bridge
and register interface on, 1500 Mbps line rate.
```

```
Configure the clocking wizard for a 100 MHz input and three outputs at
200, 150 and 74.25 MHz, with an active-low synchronous reset and a
locked output.
```

### Operating modes
### 작동 모드

Say which one you want at the start of the session; otherwise it assumes `unattended`.
세션 시작 시 원하는 것을 말하세요. 그렇지 않으면 '무인'으로 간주됩니다.

| Mode<br>모드 | Use when<br>다음과 같은 경우에 사용하세요. | Behaviour<br>행동 |
|---|---|---|
| `unattended` (default)<br>'무인'(기본값) | Scripted, CI, or batch runs<br>스크립트, CI 또는 배치 실행 | Never asks. Makes the documented best choice and records the assumption in its ledger. Minimizes MCP calls.<br>절대 묻지 마세요. 문서화된 최선의 선택을 하고 원장에 가정을 기록합니다. MCP 호출을 최소화합니다. |
| `interactive`<br>'상호작용' | You are working turn by turn<br>차례대로 일을 하고 계시네요 | May ask clarifying questions when doc search is inconclusive, and will use Block Automation and Configurable Example Designs to flesh out the design.<br>문서 검색이 결론에 도달하지 못한 경우 명확한 질문을 할 수 있으며 블록 자동화 및 구성 가능한 예제 디자인을 사용하여 디자인을 구체화합니다. |

The default is deliberate: a question nobody answers stalls a batch run until it times out,
기본값은 의도적입니다. 질문에 아무도 대답하지 않으면 시간이 초과될 때까지 배치 실행이 중단됩니다.
whereas a recorded assumption is always recoverable.
기록된 가정은 항상 복구 가능합니다.

### What you get back
### 당신이 돌려받는 것

Every run reports coverage.
모든 실행은 적용 범위를 보고합니다.
When something could not be applied, the skill quotes the exact
적용할 수 없는 것이 있을 때, 스킬은 정확한 내용을 인용합니다.
phrase from your prompt and names the reason rather than quietly grading itself down:
조용히 등급을 매기기보다는 프롬프트에서 문구를 선택하고 이유를 설명하세요.

```
I could not fully parameterize axi_noc2 from the prompt.
Applied: NUM_SI=0, NUM_MI=0, NUM_NSI=2, NUM_CLKS=0.
Could NOT apply: "4 memory controllers (NUM_MC)" because integration-derived.
```

Reasons are drawn from a fixed set: `runtime-register-only`, `integration-derived`,
이유는 고정된 세트인 '런타임 등록 전용', '통합 파생',
`gated-no-enabler`, `not-a-config-param`, `value-out-of-range`, `wrong-ip-for-capability`.
'gated-no-enabler', 'not-a-config-param', 'value-out-of-range', 'wrong-ip-for-capability'.
The overall result is graded `full`, `full_standalone`, `partial`, `negative`, or
전체 결과는 'full', 'full_standalone', 'partial', 'negative' 또는 'negative'로 등급이 매겨집니다.
`creation_only`.
'창조_만'.

### Part swaps are guarded
### 부품 교환이 보호됩니다.

Many IPs exist only on certain device families.
많은 IP는 특정 장치 제품군에만 존재합니다.
Swapping the part on a design that already
이미 디자인된 부품을 교체합니다.
has cells is destructive, so `ipcfg::ensure_part` refuses and returns `WARN:SWAP_BLOCKED`.
has 셀은 파괴적이므로 `ipcfg::ensure_part`는 `WARN:SWAP_BLOCKED`를 거부하고 반환합니다.
In `interactive` mode it asks you first; in `unattended` mode it will only swap on an
`대화형` 모드에서는 먼저 묻습니다. '무인' 모드에서는 다음 기기에서만 교체됩니다.
otherwise-empty block design, and restores the original part afterwards.
그렇지 않으면 비어 있는 블록 디자인을 만들고 나중에 원래 부품을 복원합니다.

### The helper library
### 도우미 라이브러리

`lib/ipcfg.tcl` holds the repetitive Tcl — availability gating, create, dict-plus-verify,
`lib/ipcfg.tcl`은 반복적인 Tcl — 가용성 게이팅, 생성, dict-plus-verify,
enabler checks, stub connections, cleanup.
인에이블러 검사, 스텁 연결, 정리.
Source it once per session, after a block design
블록 디자인 후 세션당 한 번씩 소싱합니다.
is open.
열려 있습니다.
The skill does this itself; you only need to know it exists if you are reading the
스킬은 이 작업을 자체적으로 수행합니다. 당신이 읽고 있는 경우에만 그것이 존재한다는 것을 알아야 합니다.
transcripts.
성적표.

`cache/learned_params.json` ships empty.
`cache/learned_params.json`은 비어 있습니다.
It accumulates *where a feature lives*
*기능이 있는 곳*에 누적됩니다.
(`feature → CONFIG.PARAM`) as the skill earns that mapping at runtime, which stops it
(`feature → CONFIG.PARAM`) 스킬이 런타임에 해당 매핑을 획득하여 이를 중지하므로
re-deriving the same lookup every run.
매 실행마다 동일한 조회를 다시 파생합니다.
It never stores a parameter *value*, so it cannot
매개변수 *값*을 저장하지 않으므로
leak an expected answer.
예상 답변을 유출합니다.
Delete it to reset; see [cache/README.md](cache/README.md).
재설정하려면 삭제하세요. [cache/README.md](cache/README.md)를 참조하세요.

## Limitations
## 제한사항

- **Some parameters are unreachable by design.** The Versal NoC's memory-controller identity
- **일부 매개변수는 설계상 도달할 수 없습니다.** Versal NoC의 메모리 컨트롤러 ID
(`NUM_MC`, `NUM_MCP`, the `DDRMC5_CONFIG` sub-dictionary) only becomes valid once the NoC
(`NUM_MC`, `NUM_MCP`, `DDRMC5_CONFIG` 하위 사전)은 NoC가 설정된 후에만 유효해집니다.
is instantiated as an integrated memory controller through device or board automation.
장치 또는 보드 자동화를 통해 통합 메모리 컨트롤러로 인스턴스화됩니다.
On
a bare PL NoC cell those keys stay gated, and the skill reports them
베어 PL NoC 셀에서는 해당 키가 게이트된 상태로 유지되고 스킬이 이를 보고합니다.
`unapplied:integration-derived` instead of pretending otherwise.
다른 척하는 대신 `unapplied:integration-derived`.
- **Widths that are read-only on the boundary port are connection-derived.** The skill can
- **경계 포트에서 읽기 전용인 너비는 연결에서 파생됩니다.** 스킬은 다음을 수행할 수 있습니다.
drive a stub port so the cell adopts a connected width, but where that fails it grades
셀이 연결된 너비를 채택하도록 스텁 포트를 구동하지만 실패하는 경우 등급이 지정됩니다.
`partial` and records the value under `runtime_params`.
`partial`이고 `runtime_params` 아래에 값을 기록합니다.
- **Two Tier-2 retries, then it escalates.** It will not grind indefinitely on an error.
- **Tier-2가 두 번 재시도된 후 에스컬레이션됩니다.** 오류가 발생해도 무기한으로 진행되지는 않습니다.
- **A block design must be open.** The standalone `create_ip` path exists only as a fallback
- **블록 디자인이 열려 있어야 합니다.** 독립 실행형 'create_ip' 경로는 대체 경로로만 존재합니다.
for parameters unavailable in IPI.
IPI에서 사용할 수 없는 매개변수의 경우.
