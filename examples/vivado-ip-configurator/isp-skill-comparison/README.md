<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Configuring an ISP: With and Without a Skill
# ISP 구성: 기술 유무

Can an AI agent turn a short functional request into the right Vivado IP configuration?
AI 에이전트가 짧은 기능 요청을 올바른 Vivado IP 구성으로 전환할 수 있나요?
This example compares the same request with and without the [`vivado-ip-configurator` skill](../../../skills/vivado-ip-configurator/SKILL.md).
이 예에서는 [`vivado-ip-configurator` 스킬](../../../skills/vivado-ip-configurator/SKILL.md)이 있는 것과 없는 동일한 요청을 비교합니다.

**In one recorded comparison, the skill helped GPT-5.6 Luna expose all 15 required streaming inputs, versus only six without it.
**한 기록된 비교에서 이 기술은 GPT-5.6 Luna가 필요한 스트리밍 입력 15개를 모두 표시하는 데 도움이 되었으며, 그렇지 않은 경우에는 6개만 표시되었습니다.
Neither Luna run fully passed.
Luna 실행이 완전히 통과되지 않았습니다.
GPT-5.6 Sol passed every check both with and without the skill.** This is an example of a useful but limited improvement, not a guarantee that a skill is necessary or sufficient.
GPT-5.6 Sol은 스킬 유무에 관계없이 모든 검사를 통과했습니다.** 이는 유용하지만 제한된 개선의 예이며, 스킬이 필요하거나 충분하다는 보장은 아닙니다.

## Goal
## 목표

Configure a hardened image signal processor (ISP) from a human-readable request.
사람이 읽을 수 있는 요청으로 강화된 이미지 신호 프로세서(ISP)를 구성합니다.
The user specifies video formats, input counts, and parallelism; the agent must discover the IP, parameter names, legal settings, and port-enablement dependencies.
사용자는 비디오 형식, 입력 수 및 병렬성을 지정합니다. 에이전트는 IP, 매개변수 이름, 법적 설정 및 포트 활성화 종속성을 검색해야 합니다.

An MCP server gives the agent access to Vivado and documentation.
MCP 서버는 에이전트에게 Vivado 및 문서에 대한 액세스를 제공합니다.
The skill adds a configuration and verification workflow, including helper commands and recovery guidance.
이 기술은 도우미 명령 및 복구 지침을 포함한 구성 및 확인 워크플로를 추가합니다.
Both conditions have the same MCP capabilities; the baseline is a capable agent with tools, not an agent without documentation.
두 조건 모두 동일한 MCP 기능을 갖습니다. 기준은 문서가 없는 에이전트가 아니라 도구를 갖춘 유능한 에이전트입니다.

## The Prompt
## 프롬프트

The following task was identical in all four completed runs.
다음 작업은 완료된 4개의 실행 모두에서 동일했습니다.
It contains no IP parameter names or Tcl implementation instructions.
여기에는 IP 매개변수 이름이나 Tcl 구현 지침이 포함되어 있지 않습니다.

> Configure a hardened ISP on the xc2ve3558-sfva1440-2MP-e-S with all three tiles enabled and multi-pass disabled.
> 세 개의 타일이 모두 활성화되고 다중 패스가 비활성화된 xc2ve3558-sfva1440-2MP-e-S에서 강화된 ISP를 구성합니다.
> Each tile needs four independent RAW12 inputs to its first ISP and one input to its second ISP—RAW12 on tile 1, 16-bit on tiles 0 and 2.
> 각 타일에는 첫 번째 ISP에 대한 4개의 독립적인 RAW12 입력과 두 번째 ISP에 대한 1개의 입력(타일 1의 경우 RAW12, 타일 0과 2의 경우 16비트)이 필요합니다.
> All inputs use four pixels per clock.
> 모든 입력은 클록당 4개의 픽셀을 사용합니다.
> Expose the corresponding streaming ports, leave other settings at defaults, verify the configuration and ports, and save the design.
> 해당 스트리밍 포트를 노출하고 다른 설정은 기본값으로 두고 구성과 포트를 확인한 후 디자인을 저장합니다.

RAW12 means 12 bits per pixel.
RAW12는 픽셀당 12비트를 의미합니다.
Four pixels per clock require a 48-bit data input; 16-bit pixels require 64 bits.
클록당 4개의 픽셀에는 48비트 데이터 입력이 필요합니다. 16비트 픽셀에는 64비트가 필요합니다.
The required boundary is:
필수 경계는 다음과 같습니다.

| Tile<br>타일 | First ISP<br>첫 번째 ISP | Second ISP<br>두 번째 ISP | Separate input interfaces<br>별도의 입력 인터페이스 |
|---|---|---|---:|
| 0 | Four RAW12 inputs, 48 bits each<br>4개의 RAW12 입력, 각각 48비트 | One 16-bit input, 64 bits<br>16비트 입력 1개, 64비트 | 5 |
| 1 | Four RAW12 inputs, 48 bits each<br>4개의 RAW12 입력, 각각 48비트 | One RAW12 input, 48 bits<br>RAW12 입력 1개, 48비트 | 5 |
| 2 | Four RAW12 inputs, 48 bits each<br>4개의 RAW12 입력, 각각 48비트 | One 16-bit input, 64 bits<br>16비트 입력 1개, 64비트 | 5 |

This explanatory table was **not** included in the agent prompt.
이 설명 테이블은 상담원 프롬프트에 포함되지 **않았습니다**.
Keep it and the results out of the agent's workspace when repeating the comparison.
비교를 반복할 때 상담원의 작업 공간 외부에 결과를 보관하세요.

## Recorded Results
## 기록된 결과

Recorded on September 29, 2026, using **Codex CLI 0.157.1**, **medium reasoning**, and **Vivado 2026.1**.
**Codex CLI 0.157.1**, **중간 추론** 및 **Vivado 2026.1**을 사용하여 2026년 9월 29일에 기록되었습니다.
These are the exact model identifiers configured through the AMD gateway; availability in another account or client may differ.
이는 AMD 게이트웨이를 통해 구성된 정확한 모델 식별자입니다. 다른 계정이나 클라이언트의 가용성은 다를 수 있습니다.
Copilot CLI and Claude Code were not used.
Copilot CLI와 Claude Code는 사용되지 않았습니다.

| Model<br>모델 | Skill<br>스킬 | Checks passed<br>검사 통과 | Required input interfaces present<br>필수 입력 인터페이스 존재 | Model time<br>모델 시간 |
|---|---|---:|---:|---:|
| `gpt-5.6-luna`<br>`gpt-5.6-루나` | With<br>와 | **95/113 (84%) — fail**<br>**95/113 (84%) — 실패** | **15/15** | 4m01s |
| `gpt-5.6-luna`<br>`gpt-5.6-루나` | Without<br>없이 | **67/113 (59%) — fail**<br>**67/113 (59%) — 실패** | **6/15** | 3m34s |
| `gpt-5.6-sol`<br>`gpt-5.6-sol` | With<br>와 | **113/113 (100%) — pass**<br>**113/113 (100%) — 통과** | **15/15** | 6m10s |
| `gpt-5.6-sol`<br>`gpt-5.6-sol` | Without<br>없이 | **113/113 (100%) — pass**<br>**113/113 (100%) — 통과** | **15/15** | 3m21s |

Each row represents one completed model attempt.
각 행은 하나의 완료된 모델 시도를 나타냅니다.
Times cover the model's tool use and recovery within that attempt, excluding environment setup and independent grading.
환경 설정 및 독립적 등급을 제외하고 해당 시도 내에서 모델의 도구 사용 및 복구에 소요되는 시간입니다.
These are observations, not average performance or success-rate estimates.
이는 평균 성능이나 성공률 추정치가 아닌 관찰 결과입니다.

### What the skill helped with
### 스킬이 도움이 된 것

Luna with the skill selected Advanced mode and exposed five independent input interfaces per tile.
스킬을 보유한 Luna는 고급 모드를 선택하고 타일당 5개의 독립적인 입력 인터페이스를 노출했습니다.
Without the skill, Luna left Basic mode enabled: it set internal input counts but exposed only two interfaces per tile. **A setting that says “four inputs” is not proof that four independently connectable ports exist.** The skill run handled this dependency better.
기술이 없으면 Luna는 기본 모드를 활성화된 상태로 두었습니다. 내부 입력 수를 설정했지만 타일당 두 개의 인터페이스만 노출했습니다. **"4개 입력"이라는 설정은 독립적으로 연결 가능한 4개의 포트가 존재한다는 증거가 아닙니다.** 기술 실행은 이러한 종속성을 더 잘 처리했습니다.

That improvement cost about 27 seconds of additional model time.
이러한 개선으로 인해 약 27초의 추가 모델 시간이 소요되었습니다.
It is the concrete value demonstrated here: more complete configuration of the requested interface boundary on Luna.
여기에 설명된 구체적인 값은 Luna에서 요청된 인터페이스 경계의 보다 완전한 구성입니다.

### What it did not solve
### 해결되지 않은 것

Both Luna runs misinterpreted the depth requirement as applying to entire tiles.
두 Luna 실행 모두 깊이 요구 사항을 전체 타일에 적용하는 것으로 잘못 해석했습니다.
In the skill run, the first ISP on tiles 0 and 2 incorrectly used 16-bit formats, making eight inputs 64 bits wide instead of 48.
기술 실행에서 타일 0과 2의 첫 번째 ISP는 16비트 형식을 잘못 사용하여 8개의 입력을 48비트 대신 64비트 너비로 만들었습니다.
The baseline also left the active paths at 16 bits, including paths it reported as RAW12.
또한 기준선은 RAW12로 보고된 경로를 포함하여 활성 경로를 16비트로 유지했습니다.
Both claimed completion despite unmet requirements.
둘 다 충족되지 않은 요구 사항에도 불구하고 완료를 주장했습니다.

After each run finished, we queried Vivado for the effective IP settings and the ports actually exposed by the design.
각 실행이 완료된 후 우리는 효과적인 IP 설정과 설계에 의해 실제로 노출된 포트에 대해 Vivado에 쿼리했습니다.
We compared these against the original request using 113 checks, including input counts, pixel formats, pixels per clock, and data widths.
우리는 입력 개수, 픽셀 형식, 클록당 픽셀 및 데이터 너비를 포함하여 113개의 검사를 사용하여 이를 원래 요청과 비교했습니다.
For example, a requested RAW12 input at four pixels per clock had to expose a 48-bit data port.
예를 들어, 클록당 4픽셀로 요청된 RAW12 입력은 48비트 데이터 포트를 노출해야 했습니다.
The scores reflect these observed settings and ports, rather than the agent's completion message.
점수는 에이전트의 완료 메시지가 아니라 관찰된 설정 및 포트를 반영합니다.
Evaluation results were not fed back to the model to correct its design; the [evaluation details](results/README.md#what-the-score-measures) explain the checks and their limits.
설계 수정을 위해 평가 결과가 모델에 피드백되지 않았습니다. [평가 세부정보](results/README.md#what-the-score-measures)에서는 점검 사항과 한계를 설명합니다.

**Sol needed no skill to score 100% on this task.** Its no-skill run was also faster than its skill run.
**Sol은 이 작업에서 100% 점수를 얻기 위해 기술이 필요하지 않았습니다.** 기술이 없는 실행도 기술 실행보다 빨랐습니다.
The evidence supports testing the skill with your chosen model and workload, rather than assuming every model benefits equally.
증거는 모든 모델의 이점이 동일하다고 가정하는 대신 선택한 모델과 워크로드로 기술을 테스트하는 것을 뒷받침합니다.
A full score here means the enumerated configuration and interface checks passed—not that the design was proven on hardware.
여기서 만점은 열거된 구성 및 인터페이스 검사를 통과했음을 의미하며 설계가 하드웨어에서 입증되었다는 의미는 아닙니다.

## Prerequisites
## 전제조건

- Vivado 2026.1 with a valid license and support for `xc2ve3558-sfva1440-2MP-e-S`.
- 'xc2ve3558-sfva1440-2MP-e-S'에 대한 유효한 라이선스 및 지원이 포함된 Vivado 2026.1.
- A configured Vivado MCP server and AMD documentation-search MCP server.
- 구성된 Vivado MCP 서버 및 AMD 문서 검색 MCP 서버.
- See [Getting Started](../../../docs/getting-started/README.md) and [Vivado MCP setup](../../../docs/getting-started/vivado-mcp.md).
- [시작하기](../../../docs/getting-started/README.md) 및 [Vivado MCP 설정](../../../docs/getting-started/vivado-mcp.md)을 참조하세요.
- Codex CLI and access to the model you want to compare.
- Codex CLI 및 비교하려는 모델에 액세스합니다.
- Use the same model and reasoning setting for both conditions.
- 두 조건 모두에 대해 동일한 모델과 추론 설정을 사용합니다.
- The `vivado-ip-configurator` skill installed only for the skill-enabled condition.
- 스킬 활성화 조건에만 'vivado-ip-configurator' 스킬이 설치됩니다.
- No board, camera, RTL sources, or bitstream are needed for this configuration exercise.
- 이 구성 연습에는 보드, 카메라, RTL 소스 또는 비트스트림이 필요하지 않습니다.

## Structure & Input Files
## 구조 및 입력 파일

- [`prompt.md`](prompt.md) — setup and copyable task instructions for both conditions.
- [`prompt.md`](prompt.md) — 두 조건 모두에 대한 설정 및 복사 가능한 작업 지침입니다.
- [`input/create_project.tcl`](input/create_project.tcl) — creates an empty project and block design; contains no ISP solution.
- [`input/create_project.tcl`](input/create_project.tcl) — 빈 프로젝트와 블록 디자인을 생성합니다. ISP 솔루션이 포함되어 있지 않습니다.
- [`results/README.md`](results/README.md) — measurement method, token/cost estimates, isolation details, and limitations.
- [`results/README.md`](results/README.md) — 측정 방법, 토큰/비용 추정, 격리 세부 정보 및 제한 사항.
- [`results/measurements.json`](results/measurements.json) — portable recorded metrics and all 113 check outcomes for each run.
- [`results/measurements.json`](results/measurements.json) — 휴대용으로 기록된 측정항목과 각 실행에 대한 113개의 확인 결과입니다.
- This is evaluation evidence, not an agent input.
- 이는 상담원 입력이 아닌 평가 증거입니다.

## How to Run
## 달리는 방법

Follow [`prompt.md`](prompt.md).
[`prompt.md`](prompt.md)를 따르세요.
Start two fresh conversations and two separate empty Vivado projects.
두 개의 새로운 대화와 두 개의 별도의 빈 Vivado 프로젝트를 시작하세요.
Give each the same task, appending only the appropriate skill-use instruction.
적절한 기술 사용 지침만 추가하여 각각에게 동일한 작업을 제공합니다.
For an honest baseline, remove skill access from both the agent and its MCP server; simply asking an already skill-enabled conversation to ignore the skill is insufficient.
정직한 기준을 위해 에이전트와 해당 MCP 서버 모두에서 스킬 액세스를 제거하십시오. 단순히 이미 스킬이 활성화된 대화에 스킬을 무시하도록 요청하는 것만으로는 충분하지 않습니다.

The recorded Luna trials used separate sandboxed agent processes, MCP server processes, and Vivado sessions.
기록된 Luna 평가판에서는 별도의 샌드박스 에이전트 프로세스, MCP 서버 프로세스 및 Vivado 세션을 사용했습니다.
They ran sequentially on the same host, with the grader and previous results hidden.
그레이더와 이전 결과가 숨겨진 상태로 동일한 호스트에서 순차적으로 실행되었습니다.
See [the execution method](results/README.md#how-the-recorded-trials-were-run) for the exact scope and the weaker isolation used in the earlier Sol comparison.
이전 Sol 비교에서 사용된 정확한 범위와 약한 격리는 [실행 방법](results/README.md#how-the-recorded-trials-were-run)을 참조하세요.
The walkthrough recreates the task; it does not automatically reproduce that sandbox or the full benchmark harness.
연습에서는 작업을 다시 만듭니다. 해당 샌드박스나 전체 벤치마크 하네스를 자동으로 재현하지 않습니다.

## Expected Output and Review
## 예상 출력 및 검토

Expect a saved block design and an agent report of its chosen settings and interfaces.
저장된 블록 설계와 선택한 설정 및 인터페이스에 대한 에이전트 보고서를 기대합니다.
Independently inspect the **actual** ports and effective configuration against the table above.
위의 표를 기준으로 **실제** 포트와 효과적인 구성을 독립적으로 검사합니다.
Do not accept an agent's completion message as the result.
결과로 에이전트의 완료 메시지를 수락하지 마세요.

The recorded grader checked IP identity, enabled tiles, independent-input mode, live-input counts, disabled multi-pass settings, active formats and pixels per clock, interface presence/protocol/direction, actual data-pin widths, shared input-3 settings, selected default output widths, and saved-design presence.
기록된 그레이더는 IP ID, 활성화된 타일, 독립 입력 모드, 실시간 입력 수, 비활성화된 다중 패스 설정, 활성 형식 및 클록당 픽셀, 인터페이스 존재/프로토콜/방향, 실제 데이터 핀 너비, 공유 입력 3 설정, 선택된 기본 출력 너비 및 저장된 디자인 존재를 확인했습니다.
Its 113 checks overlap; they are not 113 independent requirements.
113개의 검사가 중복됩니다. 이는 독립적인 요구사항이 아닙니다.

The goal is a configured IP boundary.
목표는 구성된 IP 경계입니다.
Complete system wiring, synthesis, implementation, and hardware operation are outside this example.
전체 시스템 배선, 합성, 구현 및 하드웨어 작동은 이 예에서 제외됩니다.
Unrelated-default coverage is limited to selected checks, so the score is not a guarantee that every other setting remained unchanged.
관련되지 않은 기본 적용 범위는 선택된 검사로 제한되므로 점수는 다른 모든 설정이 변경되지 않은 상태로 유지된다는 것을 보장하지 않습니다.
