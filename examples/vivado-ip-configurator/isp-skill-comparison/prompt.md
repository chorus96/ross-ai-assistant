<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# ISP Configuration — Comparison Prompts
# ISP 구성 - 비교 프롬프트

## Step 1 — Prepare each trial
## 1단계 - 각 시험 준비

Use a new working directory, new agent conversation, and new Vivado session for each condition.
각 조건에 대해 새 작업 디렉터리, 새 에이전트 대화, 새 Vivado 세션을 사용하세요.
Configure the same model and reasoning effort for both.
두 가지 모두에 대해 동일한 모델과 추론 노력을 구성합니다.
The recorded runs used Codex CLI with `gpt-5.6-luna` or `gpt-5.6-sol`, each at medium reasoning, via the AMD gateway.
기록된 실행은 AMD 게이트웨이를 통해 'gpt-5.6-luna' 또는 'gpt-5.6-sol'과 함께 각각 중간 추론에서 Codex CLI를 사용했습니다.

Connect Vivado MCP and AMD documentation-search MCP in both conditions.
두 조건 모두에서 Vivado MCP와 AMD 문서 검색 MCP를 연결합니다.
Stage only the skill-enabled condition's `vivado-ip-configurator` at `.agents/skills/vivado-ip-configurator/`.
`.agents/skills/vivado-ip-configurator/`에서 스킬 활성화 조건의 `vivado-ip-configurator`만 준비합니다.
Ensure the baseline cannot read user-installed skills, the skill repository, or previous trial files through either its shell or MCP server.
기준선이 셸이나 MCP 서버를 통해 사용자가 설치한 기술, 기술 저장소 또는 이전 평가판 파일을 읽을 수 없는지 확인하세요.

Copy only `input/create_project.tcl` into each new trial directory.
각 새 평가판 디렉터리에는 `input/create_project.tcl`만 복사하세요.
Before the timed task, start Vivado in that directory and source the script through `vivado_execute`:
시간이 지정된 작업 전에 해당 디렉터리에서 Vivado를 시작하고 'vivado_execute'를 통해 스크립트를 소싱하세요.

```tcl
source create_project.tcl
```

Confirm Vivado 2026.1, the requested part, and an empty `benchmark_bd`.
Vivado 2026.1, 요청한 부분, 비어 있는 'benchmark_bd'를 확인하세요.
Keep the session open.
세션을 열어두세요.
For the skill condition, the recorded harness also sourced the staged skill's `lib/ipcfg.tcl` during preparation; the baseline had no helper loaded.
스킬 조건의 경우 기록된 하네스는 준비 중에 단계적 스킬의 `lib/ipcfg.tcl`을 소스로 제공했습니다. 기준선에 도우미가 로드되지 않았습니다.

Do not give the agent this README, the result tables, or `results/`.
에이전트에게 이 README, 결과 테이블 또는 `results/`를 제공하지 마십시오.
Paste only the task below and the condition-specific suffix.
아래 작업과 조건별 접미사만 붙여넣으세요.
The original harness also supplied the fresh session ID with an instruction to reuse that session and leave it open.
또한 원래 하네스는 해당 세션을 재사용하고 열어두라는 지시와 함께 새로운 세션 ID를 제공했습니다.
That ID is infrastructure context, not a configuration hint.
해당 ID는 구성 힌트가 아닌 인프라 컨텍스트입니다.

## Step 2 — Use the same functional task
## 2단계 - 동일한 기능적 작업 사용

```text
Configure a hardened ISP on the xc2ve3558-sfva1440-2MP-e-S with all three tiles enabled and multi-pass disabled. Each tile needs four independent RAW12 inputs to its first ISP and one input to its second ISP—RAW12 on tile 1, 16-bit on tiles 0 and 2. All inputs use four pixels per clock. Expose the corresponding streaming ports, leave other settings at defaults, verify the configuration and ports, and save the design.
```

Append this for the **with-skill** condition:
**기술 있음** 조건에 대해 다음을 추가하세요.

```text
Use the vivado-ip-configurator skill provided in .agents/skills/vivado-ip-configurator/SKILL.md for this task.
Do not read benchmark answer keys, previous runs or other workspaces.
```

Append this for the **without-skill** condition:
**기술 없음** 조건에 대해 다음을 추가하세요.

```text
This is the no-skill baseline. Use Vivado MCP and documentation directly, without skill files or ipcfg helper libraries.
Do not read benchmark answer keys, previous runs or other workspaces.
```

## Step 3 — Record, then evaluate
## 3단계 — 기록 후 평가

Let each attempt finish without supplying configuration hints.
구성 힌트를 제공하지 않고 각 시도가 완료되도록 하세요.
Record model time, tool calls, errors, and the final saved design.
모델 시간, 도구 호출, 오류 및 최종 저장된 설계를 기록합니다.
Recovery within the same attempt is allowed; a failed final result should remain a failure rather than silently being replaced by a better retry.
동일한 시도 내에서 복구가 허용됩니다. 실패한 최종 결과는 더 나은 재시도로 조용히 대체되기보다는 실패로 남아 있어야 합니다.

After the model finishes, independently read the effective settings and actual interface pins from Vivado.
모델이 완료된 후 Vivado에서 효과적인 설정과 실제 인터페이스 핀을 독립적으로 읽습니다.
Compare them with the functional requirements in the [example README](README.md#the-prompt).
[예제 README](README.md#the-prompt)의 기능 요구 사항과 비교해 보세요.
Keep evaluation feedback out of the measured conversation.
평가 피드백을 측정된 대화에서 제외하세요.
The [recorded results](results/README.md) illustrate why checking only an agent's report or internal live-input count is insufficient.
[기록된 결과](results/README.md)는 상담원의 보고서나 내부 실시간 입력 개수만 확인하는 것만으로는 부족한 이유를 보여줍니다.

The setup script and prompts are a manual walkthrough, not an automated reproduction of the recorded sandbox or 113-check harness.
설정 스크립트 및 프롬프트는 기록된 샌드박스 또는 113 확인 하네스의 자동 재생산이 아닌 수동 연습입니다.
The machine-readable check records provide the evaluation contract and observed outcomes for inspection.
기계 판독 가능 점검 기록은 평가 계약 및 점검 결과를 제공합니다.
