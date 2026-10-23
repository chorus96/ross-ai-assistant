<!-- ko-translation: enabled -->
---
name: vai-custom-op-implementation
license: Apache-2.0 WITH LLVM-exception
compatibility: Requires VitisAI 6.3+
metadata:
  version: 6.3.0
  stage: beta
description: 'End-to-end custom AIE operator development for VAIML: extract a subgraph
  from an ONNX model, develop a custom kernel and tiling, validate with x86sim against
  a CPU reference, and re-integrate the custom op back into the full model. Use when
  the user wants to create a custom op, write a kernel, define tiling, compile for
  AIE, simulate with x86sim, or integrate a custom op into a model.'
---
<!--- Copyright (C) 2026 Advanced Micro Devices, Inc. All rights reserved.--->

# Custom Op Development Skill
# 맞춤형 작전 개발 기술

## Access command
## 액세스 명령

This skill must always be accessed through the worker, never in a different way.
이 기술은 항상 작업자를 통해 접근해야 하며 다른 방식으로는 접근할 수 없습니다.
Allowed worker: `vai-custom-op-worker`.
허용된 작업자: `vai-custom-op-worker`.

## Description
## 설명

End-to-end workflow for developing custom AIE operators: extract a target subgraph, develop the custom op in isolation, validate with x86sim, and stitch the custom op back into the full model.
맞춤형 AIE 연산자 개발을 위한 엔드투엔드 워크플로우: 대상 하위 그래프 추출, 개별적으로 맞춤형 작업 개발, x86sim으로 검증, 맞춤형 작업을 전체 모델에 다시 연결합니다.

For developing **multiple** custom ops in parallel with coordinated agent teams,
조율된 에이전트 팀과 함께 **여러** 맞춤형 작업을 동시에 개발하기 위해
use the `vai-custom-op` skill instead.
대신 `vai-custom-op` 스킬을 사용하세요.

**IMPORTANT:** When any step fails (compilation, simulation, numeric comparison, integration), do not resort to trial and error.
**중요:** 단계(컴파일, 시뮬레이션, 수치 비교, 통합)가 실패하는 경우 시행착오를 거치지 마십시오.
Instead, debug systematically: read the full error output, trace the data/control flow backward to find the root cause, and only then apply a targeted fix.
대신, 체계적으로 디버그하십시오. 전체 오류 출력을 읽고, 데이터/제어 흐름을 거꾸로 추적하여 근본 원인을 찾은 다음, 목표 수정 사항을 적용하십시오.
Check if there are other skills available that can help with debugging.
디버깅에 도움이 될 수 있는 다른 기술이 있는지 확인하세요.
See the **Troubleshooting** section at the end of this document for common errors and their solutions.
일반적인 오류와 해결 방법은 이 문서 끝에 있는 **문제 해결** 섹션을 참조하세요.

## Table of Contents
## 목차

- [Parse Arguments](#parse-arguments)
- [인수 구문 분석](#parse-arguments)
- [Environment Setup Convention](#environment-setup-convention)
- [환경 설정 규칙](#environment-setup-convention)
- [Critical Workflow Rules](#critical-workflow-rules)
- [중요한 작업 흐름 규칙](#tical-workflow-rules)
  - [1.
  - Use Sub-Agents for Compilation and Simulation](#1-use-sub-agents-for-compilation-and-simulation)
  - 컴파일 및 시뮬레이션을 위해 하위 에이전트 사용](#1-컴파일 및 시뮬레이션을 위해 하위 에이전트 사용)
  - [2.
  - Use Local Git for State Management](#2-use-local-git-for-state-management)
  - 상태 관리를 위해 로컬 Git 사용](#2-use-local-git-for-state-management)
  - [3.
  - Using a board for validation and performance data retrival](#3-using-a-board-for-validation-and-performance-data-retrival)
  - 검증 및 성능 데이터 검색을 위해 보드 사용](#3-검증 및 성능 데이터 검색을 위해 보드 사용)
  - [4.
  - X86 Simulation using runner_type, and re-compiling](#4-x86-simulation-using-runner_type-and-re-compiling)
  - runner_type을 사용한 X86 시뮬레이션 및 재컴파일](#4-x86-simulation-using-runner_type-and-re-compiling)
- [Development Methodology](#development-methodology)
- [개발 방법론](#development-methodology)
- [Design Constraints: Inputs, Outputs, and Standard Library](#design-constraints-inputs-outputs-and-standard-library)
- [설계 제약 조건: 입력, 출력 및 표준 라이브러리](#design-constraints-inputs-outputs-and-standard-library)
  - [Multi-input: pack extras into WTS via tiling and DMA](#multi-input-pack-extras-into-wts-via-tiling-and-dma)
  - [다중 입력: 타일링 및 DMA를 통해 추가 항목을 WTS에 압축](#multi-input-pack-extras-into-wts-via-tiling-and-dma)
  - [Multi-output: concatenate in the kernel, split via tiling and DMA](#multi-output-concatenate-in-the-kernel-split-via-tiling-and-dma)
  - [다중 출력: 커널에서 연결, 타일링 및 DMA를 통해 분할](#multi-output-concatenate-in-the-kernel-split-via-tiling-and-dma)
  - [Innermost-dim DMA alignment: reshape over pad](#innermost-dim-dma-alignment-reshape-over-pad)
  - [가장 안쪽의 희미한 DMA 정렬: 패드 위에서 모양 변경](#innermost-dim-dma-alignment-reshape-over-pad)
  - [Quantized subgraphs: absorb the boundary DQ/Q, make the op an int op](#quantized-subgraphs-absorb-the-boundary-dqq-make-the-op-an-int-op)
  - [양자화된 하위 그래프: 경계 DQ/Q를 흡수하고 연산을 int 연산으로 만듭니다.](#Quantized-subgraphs-absorb-the-boundary-dqq-make-the-op-an-int-op)
- [Compiler Flags: Do Not Touch the Model-Wide Knobs](#compiler-flags-do-not-touch-the-model-wide-knobs)
- [컴파일러 플래그: 모델 전체 손잡이를 건드리지 마십시오](#compiler-flags-do-not-touch-the-model-wide-knobs)
- [Stack and Heap Sizing](#stack-and-heap-sizing)
- [스택 및 힙 크기 조정](#stack-and-heap-sizing)
  - [The four checks you MUST apply](#the-four-checks-you-must-apply)
  - [반드시 적용해야 하는 4가지 수표](#the-four-checks-you-must-apply)
- [Phase 1: Subgraph Extraction](#phase-1-subgraph-extraction)
- [1단계: 하위 그래프 추출](#phase-1-subgraph-extraction)
  - [1a.
  - Extract the subgraph (Model A)](#1a-extract-the-subgraph-model-a)
  - 하위 그래프 추출(모델 A)](#1a-extract-the-subgraph-model-a)
  - [1b.
  - Generate CPU reference data](#1b-generate-cpu-reference-data)
  - CPU 참조 데이터 생성](#1b-generate-cpu-reference-data)
  - [1c.
  - Create Model B (Custom Op model)](#1c-create-model-b-custom-op-model)
  - 모델 B 생성(사용자 정의 작업 모델)](#1c-create-model-b-custom-op-model)
- [Phase 2: Create the Custom Op](#phase-2-create-the-custom-op)
- [2단계: 사용자 정의 작업 생성](#phase-2-create-the-custom-op)
  - [2a.
  - Context analysis](#2a-context-analysis)
  - 컨텍스트 분석](#2a-컨텍스트-분석)
  - [2b.
  - ONNX Custom Op YAML](#2b-onnx-custom-op-yaml)
  - ONNX 사용자 지정 작업 YAML](#2b-onnx-custom-op-yaml)
  - [2c.
  - Config YAML](#2c-config-yaml-custom_op_namenameyaml)
  - YAML 구성](#2c-config-yaml-custom_op_namenameyaml)
  - [2d.
  - Kernel C++](#2d-kernel-c-custom_op_namenamecpp)
  - 커널 C++](#2d-kernel-c-custom_op_namenamecpp)
  - [2e.
  - Tiling Python](#2e-tiling-python-custom_op_namename_tilingpy)
  - 타일링 Python](#2e-tiling-python-custom_op_namename_tilingpy)
  - [2f.
  - Vitis AI Config](#2f-vitis-ai-config-vitisai_configjson)
  - Vitis AI 구성](#2f-vitis-ai-config-vitisai_configjson)
  - [2g.
  - Tutorial Examples for Inspiration](#2g-tutorial-examples-for-inspiration)
  - 영감을 위한 튜토리얼 예제](#2g-tutorial-examples-for-inspiration)
- [Phase 3: Compile](#phase-3-compile)
- [3단계: 컴파일](#phase-3-compile)
- [Phase 4: Simulate and Validate](#phase-4-simulate-and-validate)
- [4단계: 시뮬레이션 및 검증](#phase-4-simulate-and-validate)
  - [4a.
  - Run x86sim with the same inputs from Phase 1b](#4a-run-x86sim-with-the-same-inputs-from-phase-1b)
  - 1b단계와 동일한 입력으로 x86sim 실행](#4a-run-x86sim-with-the-same-inputs-from-phase-1b)
  - [4b.
  - Compare x86sim output against CPU reference](#4b-compare-x86sim-output-against-cpu-reference)
  - CPU 참조와 x86sim 출력 비교](#4b-compare-x86sim-output-against-cpu-reference)
  - [Debugging (only if 4b fails)](#debugging-only-if-4b-fails)
  - [디버깅(4b가 실패한 경우에만)](#debugging-only-if-4b-fails)
- [Phase 5: Kernel Optimization](#phase-5-kernel-optimization)
- [5단계: 커널 최적화](#phase-5-kernel-optimization)
  - [Optimization is data-driven -- ALWAYS, board or no board](#optimization-is-data-driven----always-board-or-no-board)
  - [최적화는 데이터 중심입니다 -- 항상, 보드가 있거나 없음](#optimization-is-data-driven----always-board-or-no-board)
- [Phase 6: Model Re-integration](#phase-6-model-re-integration)
- [6단계: 모델 재통합](#phase-6-model-re-integration)
  - [6a.
  - Create the integration directory](#6a-create-the-integration-directory)
  - 통합 디렉터리 생성](#6a-create-the-integration-directory)
  - [6b.
  - Stitch the custom op into the full model](#6b-stitch-the-custom-op-into-the-full-model)
  - 사용자 정의 작업을 전체 모델에 연결](#6b-stitch-the-custom-op-into-the-full-model)
  - [6c.
  - Set up vitisai_config.json](#6c-set-up-vitisai_configjson)
  - vitisai_config.json 설정](#6c-set-up-vitsai_configjson)
- [Phase 7: End-to-End Verification](#phase-7-end-to-end-verification)
- [7단계: 엔드투엔드 검증](#phase-7-end-to-end-verification)
  - [7a.
  - Compile the integrated model (REQUIRED)](#7a-compile-the-integrated-model-required)
  - 통합 모델 컴파일(필수)](#7a-compile-the-integrated-model-필수)
  - [7b.
  - Generate full-model CPU reference](#7b-generate-full-model-cpu-reference)
  - 전체 모델 CPU 참조 생성](#7b-generate-full-model-cpu-reference)
  - [7c.
  - Compare outputs](#7c-compare-outputs)
  - 출력 비교](#7c-비교-출력)
- [Key Architecture Facts](#key-architecture-facts)
- [주요 아키텍처 사실](#key-architecture-facts)
- [Script Reference](#script-reference)
- [스크립트 참조](#script-reference)
- [Troubleshooting](#troubleshooting)
- [문제 해결](#troubleshooting)
  - [Common Errors](#common-errors)
  - [일반적인 오류](#common-errors)
  - [Environment Setup Checklist](#environment-setup-checklist)
  - [환경 설정 체크리스트](#environment-setup-checklist)

## Parse Arguments
## 인수 구문 분석

This skill expects arguments in this form:
이 기술은 다음 형식의 인수를 기대합니다.

```
[operation_description] [--t <activate_path>] --model <onnx_model> --vitisai-config <config_json> --nodes <node1> [<node2> ...]
```

Extract from `$ARGUMENTS`:
`$ARGUMENTS`에서 추출:
- **Operation description**: what the custom op should do and which ops to target
- **작업 설명**: 맞춤 작업이 수행해야 하는 작업과 대상으로 삼을 작업
- `--t <path>`: **(OPTIONAL)** Path to a Ryzen AI / Vitis AI / product virtual environment activate script.
- `--t <경로>`: **(선택 사항)** Ryzen AI / Vitis AI / 제품 가상 환경 활성화 스크립트의 경로입니다.
- Provide this only when the Python environment is NOT already on the shell.
- Python 환경이 셸에 아직 없는 경우에만 이를 제공하세요.
- Omit it when the environment is pre-provisioned (e.g. inside the official Vitis AI Docker container, where `flexml`, `onnx`, `onnxruntime`, etc. are already importable from system Python).
- 환경이 사전 프로비저닝된 경우(예: `flexml`, `onnx`, `onnxruntime` 등이 이미 시스템 Python에서 가져올 수 있는 공식 Vitis AI Docker 컨테이너 내부)인 경우 이를 생략합니다.
- See "Environment Setup Convention" below.
- 아래의 "환경 설정 규칙"을 참조하세요.
- `--model <path>`: **(MANDATORY)** Path to the ONNX model file
- `--model <경로>`: **(필수)** ONNX 모델 파일의 경로
- `--vitisai-config <path>`: **(MANDATORY)** Path to the vitisai_config.json file
- `--vitisai-config <경로>`: **(필수)** vitisai_config.json 파일 경로
- `--nodes <node1> [<node2> ...]`: **(MANDATORY)** Node names or op types to extract as a custom op subgraph
- `--nodes <node1> [<node2> ...]`: **(필수)** 사용자 정의 작업 하위 그래프로 추출할 노드 이름 또는 작업 유형

Skill directory: the directory containing this SKILL.md file (referred to as `<skill_dir>` below).
스킬 디렉터리: 이 SKILL.md 파일이 포함된 디렉터리입니다(아래 `<skill_dir>` 참조).

**Validation**: Before proceeding, verify all mandatory arguments are present.
**검증**: 계속하기 전에 모든 필수 인수가 있는지 확인하세요.
If any are missing, ask the user to provide them.
누락된 항목이 있는 경우 사용자에게 제공하도록 요청하세요.
Do NOT treat `--t` as missing-and-required: its absence is a deliberate signal to skip venv sourcing (see next section).
`--t`를 누락 및 필수 항목으로 처리하지 마십시오. 이 항목이 없으면 venv 소싱을 건너뛰라는 의도적인 신호입니다(다음 섹션 참조).

## Environment Setup Convention
## 환경 설정 규칙

This skill supports two deployment models:
이 기술은 두 가지 배포 모델을 지원합니다.

1. **Sourced venv**: the user passes `--t <path>` and every shell that runs `python3` / `aiecompiler` must `source <path>` first.
1. **소스 venv**: 사용자는 `--t <path>`를 전달하고 `python3` / `aiecompiler`를 실행하는 모든 셸은 먼저 `source <path>`를 사용해야 합니다.
2. **Pre-provisioned environment** (e.g. official Vitis AI Docker container, sandbox image): the Python packages and binaries are already on `PATH` / `PYTHONPATH`.
2. **사전 프로비저닝된 환경**(예: 공식 Vitis AI Docker 컨테이너, 샌드박스 이미지): Python 패키지와 바이너리는 이미 `PATH` / `PYTHONPATH`에 있습니다.
2. `--t` is NOT provided; no sourcing is needed.
2. `--t`는 제공되지 않습니다. 소싱이 필요하지 않습니다.

Wherever this document shows `<ENV_SETUP>` inside a bash snippet, expand it as follows:
이 문서가 bash 조각 내에서 `<ENV_SETUP>`을 표시할 때마다 다음과 같이 확장합니다.
- If `--t <path>` was provided: replace the `<ENV_SETUP>` line with `source <path>`.
- `--t <path>`가 제공된 경우: `<ENV_SETUP>` 줄을 `source <path>`로 바꿉니다.
- Otherwise: delete the `<ENV_SETUP>` line entirely.
- 그렇지 않은 경우: `<ENV_SETUP>` 줄을 완전히 삭제하세요.

**Do NOT auto-detect or guess an activate script.** If `--t` is absent, trust that the calling shell already has `python3 -c "import flexml"` working.
**활성화 스크립트를 자동 감지하거나 추측하지 마세요.** `--t`가 없으면 호출 셸에 이미 `python3 -c "import flexml"`이 작동하고 있다고 믿으세요.
Searching the filesystem for a venv to source is a bug, not a fallback.
파일 시스템에서 소스에 대한 Venv를 검색하는 것은 폴백이 아니라 버그입니다.

A quick sanity check you MAY run once at startup (especially when `--t` is absent) is `python3 -c "import flexml, onnx, onnxruntime"`.
시작 시(특히 `--t`가 없는 경우) 한 번 실행할 수 있는 빠른 온전성 검사는 `python3 -c "import flexml, onnx, onnxruntime"`입니다.
If that fails, stop and ask the user how the environment should be set up rather than hunting for one.
실패하면 환경을 찾기보다는 중지하고 사용자에게 환경을 어떻게 설정해야 하는지 물어보세요.

## Critical Workflow Rules
## 중요한 작업 흐름 규칙

### 1.
### Use Sub-Agents for Compilation and Simulation
### 컴파일 및 시뮬레이션을 위해 하위 에이전트 사용

**NEVER run compilation or simulation commands directly in the main agent.** These produce verbose output that pollutes the main context window.
**주 에이전트에서 직접 컴파일 또는 시뮬레이션 명령을 실행하지 마십시오.** 이는 기본 컨텍스트 창을 오염시키는 자세한 출력을 생성합니다.
Always use the vai-custom-op-worker sub-agent.
항상 vai-custom-op-worker 하위 에이전트를 사용하십시오.

**For compilation:**
**컴파일용:**
```
Use Agent tool with subagent_type="vai-custom-op-worker" and a prompt like:
"Run the following command and report whether it succeeded or failed,
including any error messages if it failed:
bash -c '
<ENV_SETUP>
export DEBUG_VAIML_PARTITION=1
export FLEXML_PRINT_VITISTOOLS_OUTPUT=1
cd <working_dir>
python <skill_dir>/scripts/compile.py <model>.onnx
'"
```

**For x86sim** (same cache as the compile above; the x86sim config selects the runner -- see Critical Workflow Rule 4):
**x86sim의 경우**(위의 컴파일과 동일한 캐시, x86sim 구성이 실행기를 선택합니다. 중요한 작업 흐름 규칙 4 참조):
```
Use Agent tool with subagent_type="vai-custom-op-worker" and a prompt like:
"Run the following command and report the full output, especially any
printf statements from the kernel and the final pass/fail status:
bash -c '
<ENV_SETUP>
cd <working_dir>
python <skill_dir>/scripts/run.py <model>.onnx \
    --input-dir <working_dir>/reference_data \
    --output-dir <working_dir>/x86sim_outputs \
    --vitisai-config vitisai_config_x86sim.json --num-runs 1
'"
```

The sub-agent will return a concise summary.
하위 에이전트는 간결한 요약을 반환합니다.
Only escalate to the main context if there are errors that need debugging.
디버깅이 필요한 오류가 있는 경우에만 기본 컨텍스트로 에스컬레이션하세요.

### 2.
### Use Local Git for State Management
### 상태 관리를 위해 로컬 Git 사용

**ALWAYS create a local git repo in the working directory** at the start of development.
**개발 시작 시 항상 작업 디렉터리에 로컬 git 저장소를 생성하세요**.
Commit after every successful milestone so you can easily revert to a known-good state when experimenting. **EXCEPTION** If the user doesn't have git available in their workspace, you can skip this section and the saving of milestones.
실험할 때 알려진 양호한 상태로 쉽게 되돌릴 수 있도록 모든 성공적인 이정표 후에 커밋합니다. **예외** 사용자의 작업공간에 Git이 없으면 이 섹션과 마일스톤 저장을 건너뛸 수 있습니다.
You should still do defensive programming and create backups if you're not sure about the changes you're about to perform.
수행하려는 변경 사항이 확실하지 않은 경우에도 여전히 방어적인 프로그래밍을 수행하고 백업을 생성해야 합니다.

```bash
cd <working_dir>
git init
git add -A
git commit -m "Initial custom op files for <op_name>"
```

**Commit at these milestones:**
**다음 단계에 커밋하세요.**
- After creating all initial files (Phase 2)
- 모든 초기 파일 생성 후(2단계)
- After successful compilation (Phase 3)
- 성공적인 컴파일 후(3단계)
- After successful x86sim (Phase 4)
- x86sim 성공 후(4단계)
- Before attempting any optimization (Phase 5)
- 최적화를 시도하기 전(5단계)
- After each successful optimization step (Phase 5)
- 각 성공적인 최적화 단계(5단계) 이후

### 3.
### Using a board for validation and performance data retrival
### 검증 및 성능 데이터 검색을 위해 보드 사용

Always establish that there is a board or a script to execute inference on a board
항상 보드나 보드에서 추론을 실행하는 스크립트가 있는지 확인하세요.
before starting:
시작하기 전에:

- Use the `scripts/run_on_board.py` next to this skill, unless the user tells you otherwise, in which case
- 사용자가 달리 지시하지 않는 한 이 스킬 옆에 있는 `scripts/run_on_board.py`를 사용하세요.
you need to follow the user's instructions and commands.
사용자의 지시와 명령을 따라야 합니다.

- If the board run script fails to reach a board (unreachable board or no board), STOP and ask the user
- 보드 실행 스크립트가 보드에 도달하지 못하는 경우(보드에 연결할 수 없거나 보드가 없음) 중지하고 사용자에게 물어보세요.
if and how they run inference on their board (their own script, harness, or command).
보드에서 추론을 실행하는지 여부와 방법(자체 스크립트, 하네스 또는 명령)
Board execution
이사회 실행
might be mandatory or optional depending on user's goals and it is not possible to disambiguate,
사용자의 목표에 따라 필수 또는 선택 사항일 수 있으며 명확하게 구분할 수는 없습니다.
unless explicitely mentioned in the prompt.
프롬프트에서 명시적으로 언급되지 않는 한.
You MUST prompt the user with the following warnings about not having
사용자에게 다음과 같은 경고 메시지를 표시해야 합니다.
a board in the loop:
루프에 있는 보드:
    - Not having a board might impact functional correctness.
    - 보드가 없으면 기능적 정확성에 영향을 미칠 수 있습니다.
    - Without board testing, this skill can only rely on x86sim for
    - 보드 테스트가 없으면 이 기술은 x86sim에만 의존할 수 있습니다.
numeric accuracy which doesn't map one to one to the hardware.
하드웨어에 일대일로 매핑되지 않는 수치 정확도.
    - Not having a board impact the data-driven optimization loop because optimization will only be guided by static metrics
    - 보드가 데이터 기반 최적화 루프에 영향을 주지 않는 이유는 최적화가 정적 메트릭에 의해서만 안내되기 때문입니다.
such as static code analysis and initiation intervals.
정적 코드 분석 및 시작 간격 등.
Without dynamic data optimization is a best guess and it can result
동적 데이터가 없는 최적화는 최선의 추측이며 결과는 다음과 같습니다.
in sub-optimal solutions
최적이 아닌 솔루션에서
The user MUST give explicit consent to the above points.
사용자는 위 사항에 대해 명시적인 동의를 해야 합니다.
- If the user cannot provide board access, offer to switch to static data collection for performance guidance
- 사용자가 보드 액세스를 제공할 수 없는 경우 성능 지침을 위해 정적 데이터 수집으로 전환하도록 제안합니다.
and function validation throuh x86 simulation.
x86 시뮬레이션을 통한 기능 검증.
- **Headless (no interactive user):** don't block on the question -- proceed with
- **헤드리스(대화형 사용자 없음):** 질문을 차단하지 말고 계속 진행하세요.
static analysis (Phase 5a) and report that latency was not measured.
정적 분석(5a단계) 및 지연 시간이 측정되지 않았음을 보고합니다.

### 4.
### X86 Simulation using runner_type, and re-compiling
### runner_type을 사용한 X86 시뮬레이션 및 재컴파일

**One compile serves both hardware and x86 runners, but you MUST ask for the x86
**하나의 컴파일은 하드웨어와 x86 실행기를 모두 제공하지만 x86을 요청해야 합니다.
half.** The default build emits only the hardware artifacts (`ctrl_pkts.xclbin`,
half.** 기본 빌드는 하드웨어 아티팩트(`ctrl_pkts.xclbin`,
`aie_control.elf`) and records `runner_type: "hw"` in the cache.
`aie_control.elf`) 및 캐시에 `runner_type: "hw"`를 기록합니다.
The x86
x86
simulator library (`libx86_sim.so`) is built only when `aiecompiler_args`
시뮬레이터 라이브러리(`libx86_sim.so`)는 `aiecompiler_args`인 경우에만 빌드됩니다.
carries both of these flags:
다음 플래그를 모두 전달합니다.

```json
      "aiecompiler_args": "--compile-for-x86sim=true --x86sim-as-shared-library=true",
```

**Always include `--msg-severity="77-23879:error:100"`** to promote the hardware-only L2 shared-buffer overlap warning to a compile error (x86sim ignores it, but it corrupts results on the board).
**항상 `--msg-severity="77-23879:error:100"`**을 포함하여 하드웨어 전용 L2 공유 버퍼 겹침 경고를 컴파일 오류로 승격합니다(x86sim은 이를 무시하지만 보드의 결과가 손상됩니다).

**The runner is chosen at run time, by the config you hand `run.py`.** Keep a
**러너는 런타임 시 `run.py` 구성에 의해 선택됩니다.**
copy of the vitisai config with one extra key and use different processes for compilation and simulation:
하나의 추가 키를 사용하여 vitisai 구성을 복사하고 컴파일 및 시뮬레이션을 위해 다른 프로세스를 사용합니다.

```json
    "runner_type": "x86sim",
```

Pass that copy to `run.py` and the VitisAI EP loads the simulator as a shared
해당 복사본을 `run.py`에 전달하면 VitisAI EP가 시뮬레이터를 공유로 로드합니다.
library, so an ordinary ONNX Runtime session *is* the simulation -- against the
라이브러리이므로 일반 ONNX 런타임 세션은 *시뮬레이션입니다*.
same Phase 3 cache, with no recompile.
동일한 3단계 캐시, 재컴파일 없음.
Pass the normal config and the same cache
일반 구성과 동일한 캐시를 전달합니다.
runs on the board.
보드에서 실행됩니다.

**Never put `runner_type: "x86sim"` in the config you compile with.** The value
**컴파일하는 구성에 절대로 `runner_type: "x86sim"`을 넣지 마세요.** 값
is written into the cache, and omitting the key later means *unspecified*, not
캐시에 기록되며 나중에 키를 생략한다는 것은 *지정되지 않음*을 의미합니다.
`"hw"` -- so a subsequent board run falls back to the cached `x86sim`, tries to
`"hw"` -- 후속 보드 실행은 캐시된 `x86sim`으로 돌아가서 다음을 시도합니다.
load the x86-64 `libx86_sim.so` on the aarch64 board, and fails (see
aarch64 보드에 x86-64 `libx86_sim.so`를 로드했는데 실패했습니다(참조
Troubleshooting).
문제 해결).
The effective runner is the config's value when present,
효과적인 러너는 존재하는 경우 구성의 값입니다.
otherwise the cached one.
그렇지 않으면 캐시된 것입니다.

**x86sim is functionally correct, not bit-accurate.** It compiles the kernel for
**x86sim은 기능적으로 정확하지만 비트 정확도는 아닙니다.** x86sim은 커널을 컴파일합니다.
native x86 while the AIE path may use emulated fp32, so the two round
AIE 경로는 에뮬레이트된 fp32를 사용할 수 있지만 기본 x86이므로 두 라운드
differently on values near a rounding boundary.
반올림 경계 근처의 값은 다르게 나타납니다.
Use it to catch functional bugs
기능적 버그를 잡는 데 사용하십시오.
(wrong tile, wrong core offset, wrong axis, garbage output); keep the board as
(잘못된 타일, 잘못된 코어 오프셋, 잘못된 축, 가비지 출력); 보드를 그대로 유지
the authority for any exactness claim.
정확성 주장에 대한 권한.

**Both runners are the same `run.py`, so an x86sim run destroys a board run's
**두 러너 모두 동일한 `run.py`이므로 x86sim 실행은 보드 실행을 파괴합니다.
profiling data in the same directory.** Profiling is always on and the artifacts
동일한 디렉터리에 있는 데이터를 프로파일링합니다.** 프로파일링은 항상 켜져 있으며 아티팩트
land in the working directory, not in `--output-dir`.
`--output-dir`이 아닌 작업 디렉토리에 위치합니다.
Session creation deletes the
세션 생성 시 삭제됨
whole `record_timer_*.json` glob while `partition.json`/`summary.csv` survive
`partition.json`/`summary.csv`가 유지되는 동안 전체 `record_timer_*.json` glob
stale, so spot-checking those two misleads. x86sim also writes only host
부실하므로 그 두 가지 오해를 찾아보세요. x86sim은 또한 호스트만 씁니다.
wall-clock timers -- `record_timer_ts.json` (NPU cycles) is **board-only**.
벽시계 타이머 -- `record_timer_ts.json`(NPU 주기)은 **보드 전용**입니다.
Copy
복사
board artifacts aside, or simulate elsewhere, before running `/vai-perf-analysis`.
`/vai-perf-analytic`을 실행하기 전에 아티팩트를 따로 보관하거나 다른 곳에서 시뮬레이션하세요.

**You MUST re-run `compile.py` after changing any custom op file** -- kernel
**사용자 정의 op 파일을 변경한 후에는 `compile.py`를 다시 실행해야 합니다** -- 커널
.cpp, tiling Python, YAML config, ONNX model, or `vitisai_config.json` -- before
.cpp, 타일링 Python, YAML 구성, ONNX 모델 또는 `vitsai_config.json` -- 이전
simulating or running on the board.
보드에서 시뮬레이션하거나 실행합니다.
Nothing in the run path recompiles.
실행 경로의 어떤 것도 다시 컴파일되지 않습니다.

## Development Methodology
## 개발 방법론
Start with a single-stamp, 16-core implementation -- x86sim validates correctness on all 16 cores.
단일 스탬프, 16코어 구현으로 시작하십시오. x86sim은 16개 코어 모두에서 정확성을 검증합니다.
Use the `mul/2_distributed` or `negate/2_distributed` tutorials as starting templates.
'mul/2_distributed' 또는 'negate/2_distributed' 튜토리얼을 시작 템플릿으로 사용하세요.

**Always use the AieConfig signature:** `getTiling(opInterface, tiling: tensor_expr.AieConfig)` -- an `AieConfig` has one entry per overlay stamp, so it works on a single-stamp overlay and requires no rewrite when the op moves to a multi-stamp one (see `tiling.md`).
**항상 AieConfig 서명을 사용하세요.** `getTiling(opInterface, tiling: tensor_expr.AieConfig)` -- `AieConfig`에는 오버레이 스탬프당 하나의 항목이 있으므로 단일 스탬프 오버레이에서 작동하며 작업이 다중 스탬프 오버레이로 이동할 때 다시 쓸 필요가 없습니다(`tiling.md` 참조).

**Honor `tp_size` (from the vitisai_config).** `tp_size: N` provisions the overlay with N stamps (N*16 cores) but does not by itself use them -- your tiling decides.
**`tp_size`(vitisai_config에서)를 존중합니다.** `tp_size: N`은 N 스탬프(N*16 코어)로 오버레이를 프로비저닝하지만 자체적으로 사용하지는 않습니다. 타일링이 결정합니다.
If `tp_size: N` (N>1), distribute across all N stamps (`assert len(tiling) == N`) by giving each its own slice of the DDR tensor; configuring only one leaves N-1 stamps idle.
`tp_size: N`(N>1)인 경우 각 스탬프에 고유한 DDR 텐서 조각을 제공하여 모든 N 스탬프에 배포합니다(`assert len(tiling) == N`). 하나만 구성하면 N-1 스탬프가 유휴 상태가 됩니다.
Treat a user-provided `tp_size` as a hard requirement.
사용자가 제공한 'tp_size'를 엄격한 요구사항으로 처리하세요.

1. **4x4 distributed** (single-stamp) -- scalar kernel, validate correctness with x86sim
1. **4x4 분산**(단일 스탬프) -- 스칼라 커널, x86sim으로 정확성 검증
2. **Optimized kernel** -- vectorize with `aie::vector`, add `AIE_PIPELINE` hints
2. **최적화된 커널** -- `aie::벡터`를 사용하여 벡터화하고 `AIE_PIPELINE` 힌트를 추가합니다.

## Design Constraints: Inputs, Outputs, and Standard Library
## 설계 제약: 입력, 출력 및 표준 라이브러리

Custom ops supports many inputs while being only limited by the amount of channels connecting
사용자 정의 작업은 연결하는 채널 수에 의해서만 제한되는 동시에 많은 입력을 지원합니다.
the DDR to the memory tiles.
DDR을 메모리 타일에 추가합니다.
Each channel can be configured independently during tiling and
타일링 중에 각 채널을 독립적으로 구성할 수 있으며
associated to different (or the same) DDR inputs.
다른(또는 동일한) DDR 입력과 연결됩니다.
The connections from the memory tiles to the cores
메모리 타일에서 코어까지의 연결
are limited to 2, i.e, there are only 2 physical channels that we can use to push data from the memory tiles
2개로 제한됩니다. 즉, 메모리 타일에서 데이터를 푸시하는 데 사용할 수 있는 물리적 채널은 2개뿐입니다.
into the core tiles.
코어 타일에 넣습니다.
The best possible solution when dealing with more than 2 inputs is to use the DMA controllers
2개 이상의 입력을 처리할 때 가능한 가장 좋은 솔루션은 DMA 컨트롤러를 사용하는 것입니다.
to pack or concatenate the variables when reading from DDR into the memory tiles.
DDR에서 메모리 타일로 읽을 때 변수를 묶거나 연결합니다.
This comes at zero cost
비용은 전혀 들지 않습니다.
but it is not a generic solution to all the custom ops since whatever variables are created in the memory tiles
그러나 모든 변수가 메모리 타일에 생성되므로 모든 사용자 정의 작업에 대한 일반적인 솔루션은 아닙니다.
still need to be themselves tiled and transferred to the cores.
여전히 타일링되어 코어로 전송되어야 합니다.

Custom ops only support **1 output tensor**.
맞춤 작업은 **1개의 출력 텐서**만 지원합니다.
The best known solution for dealing with multiple outputs - within one
여러 출력을 처리하는 가장 잘 알려진 솔루션 - 하나 내에서
custom op - is to concatenate the multiple results in the memory tile varible and use `onnx.Slice` or another custom op
맞춤 작업 - 메모리 타일 변수의 여러 결과를 연결하고 'onnx.Slice' 또는 다른 맞춤 작업을 사용하는 것입니다.
to split the results into two or more, while updating the respective consumers.
각각의 소비자를 업데이트하면서 결과를 둘 이상으로 분할합니다.

When possible, always prefer a solution that works around the arity limits using the Tiling API and the DMA controllers - not by adding nodes to the ONNX graph - as to avoid round trips to the DDR memory which reduce data locality and are expensive.
가능하다면 항상 ONNX 그래프에 노드를 추가하는 것이 아니라 타일링 API 및 DMA 컨트롤러를 사용하여 개수 제한을 해결하는 솔루션을 선호합니다. 이는 데이터 지역성을 줄이고 비용이 많이 드는 DDR 메모리로의 왕복을 방지하기 위한 것입니다.

### Multi-input: pack extras into WTS via tiling and DMA
### 다중 입력: 타일링 및 DMA를 통해 추가 항목을 WTS에 포함

Favor L2 packing: whenever possible, carefully choose a L2 location for each input
L2 패킹 선호: 가능하면 각 입력에 대해 L2 위치를 신중하게 선택하세요.
and read it into L2 from L3 with a dedicated channel; send the resulting packet to L1.
전용 채널을 사용하여 L3에서 L2로 읽습니다. 결과 패킷을 L1으로 보냅니다.
You usually need two L2 buffers; one to be sent across columns to L1, and one to be
일반적으로 두 개의 L2 버퍼가 필요합니다. 하나는 열을 통해 L1으로 전송되고 다른 하나는
sent across rows.
여러 행에 걸쳐 전송됩니다.

If absolutely necessary, you may resort to packing data in L3 using ONNX
꼭 필요한 경우 ONNX를 사용하여 L3에 데이터를 패킹할 수 있습니다.
`Concat` nodes.
'Concat' 노드.
Keep the ONNX graph lean -- avoid auxiliary `Concat`/`Reshape` nodes
ONNX 그래프를 간결하게 유지 - 보조 'Concat'/'Reshape' 노드 방지
whose only purpose is to assemble WTS.
그의 유일한 목적은 WTS를 조립하는 것입니다.
The kernel can read each component
커널은 각 구성요소를 읽을 수 있습니다.
from a fixed offset inside the IFM/WTS buffers.
IFM/WTS 버퍼 내부의 고정 오프셋에서.

Example: an op with 3 dynamic inputs (`input_a`, `input_b`, `input_c`)
예: 3개의 동적 입력(`input_a`, `input_b`, `input_c`)이 있는 작업
plus constant `weights`.
플러스 상수 '가중치'.
Concat the dynamic inputs into IFM and pack
동적 입력을 IFM으로 연결하고 압축합니다.
`input_c` alongside `weights` in WTS:
WTS의 `가중치`와 함께 `input_c`:
```
IFM = [input_a(64), input_b(64)]      = [128]    -- dynamic inputs concatenated
WTS = [weights(8448), input_c(960)]    = [9408]   -- weights + dynamic input packed
```
The kernel reads `input_b` from `ifm[64]` and `input_c` from `wts[8448]`.
커널은 `ifm[64]`에서 `input_b`를 읽고 `wts[8448]`에서 `input_c`를 읽습니다.
Choose IFM vs.
IFM과 IFM을 선택하세요.
WTS based on the broadcast pattern: on the 4x4 overlay
방송 패턴 기반 WTS: 4x4 오버레이
IFM is broadcast column-wise and WTS row-wise.
IFM은 열 단위로 방송되고 WTS는 행 단위로 방송됩니다.

### Multi-output: concatenate in the kernel, split via tiling and DMA
### 다중 출력: 커널에서 연결, 타일링 및 DMA를 통해 분할

Write all logical outputs concatenated into one contiguous OFM buffer in
하나의 연속 OFM 버퍼에 연결된 모든 논리 출력을 씁니다.
the kernel.
커널.
Use tiling transforms on `ofm_mem` (`Reshape`, `Transpose`,
`ofm_mem`에서 타일링 변환을 사용합니다(`Reshape`, `Transpose`,
`TileTo`, `SpatialDistribute2D`) so the L2 -> DDR DMA reorders/splits
`TileTo`, `SpatialDistribute2D`) L2 -> DDR DMA 재정렬/분할
the concatenated OFM and each logical output lands contiguously in DDR.
연결된 OFM과 각 논리 출력은 DDR에 연속적으로 배치됩니다.
This avoids per-invocation ONNX `Slice`/`Reshape` nodes downstream.
이렇게 하면 호출별 ONNX 'Slice'/'Reshape' 노드 다운스트림이 방지됩니다.
For
에 대한
chained ops, the concatenated output can feed directly as the next op's
연결된 작업, 연결된 출력은 다음 작업으로 직접 공급될 수 있습니다.
IFM without any explicit split.
명시적인 분할이 없는 IFM.

### Innermost-dim DMA alignment: reshape over pad
### 가장 안쪽의 희미한 DMA 정렬: 패드 위의 모양 변경

The infrastructure automatically pads the innermost dimension up to the
인프라는 가장 안쪽 차원을 자동으로 채웁니다.
DMA-alignment granularity.
DMA 정렬 세분성.
That pad is expensive, and on the output side
저 패드는 비싸고 출력측에는
it typically also forces a trailing slice to recover the live extent --
또한 일반적으로 후행 슬라이스가 라이브 범위를 복구하도록 강제합니다.
both of which should be avoided whenever possible.
가능하면 둘 다 피해야 합니다.

The preferred fix is to **reshape the tensor in L3 memory** so the
선호되는 수정 방법은 **L3 메모리의 텐서를 재구성**하여
innermost dimension is already a multiple of the DMA granularity (e.g.
가장 안쪽 차원은 이미 DMA 세분성의 배수입니다(예:
fold a small inner dim into an adjacent outer one).
작은 내부 딤을 인접한 외부 딤으로 접습니다).
This costs nothing
비용이 전혀 들지 않습니다.
at runtime when it collapses to a layout change.
런타임 시 레이아웃 변경으로 인해 축소될 때.

When you reshape at the boundary, **pass the original shape into the
경계에서 모양을 변경할 때 **원래 모양을
custom op via ONNX attributes** and forward those attributes into
ONNX 속성**을 통해 사용자 정의 작업을 수행하고 해당 속성을 다음으로 전달합니다.
`lp_params` from the tiling Python.
타일링 Python의 `lp_params`.
The boundary shape exists only to
경계 모양은 다음에만 존재합니다.
satisfy DMA alignment; the original shape usually carries semantic
DMA 정렬을 만족하고; 원래 모양은 일반적으로 의미를 전달합니다.
meaning (e.g.
의미(예:
`[B, H, W, C]`), and both the tiling and `kernel.cpp` need
`[B, H, W, C]`), 타일링과 `kernel.cpp` 모두 필요합니다.
that meaning to compute strides, loop bounds, and any per-axis logic
이는 보폭, 루프 경계 및 축별 논리를 계산한다는 의미입니다.
correctly.
올바르게.

Caveat: the reshape must be consistent with your tiling.
주의 사항: 모양 변경은 타일링과 일치해야 합니다.
A dimension
차원
that is squeezed (or fused away) by the reshape cannot be tiled along
모양 변경으로 인해 압착된(또는 융합된) 타일은 타일링할 수 없습니다.
afterwards, so any axis you still need as a tiling axis must survive
이후에는 타일링 축으로 필요한 모든 축이 살아남아야 합니다.
the reshape intact.
모양이 그대로 유지됩니다.
Plan the boundary shape and the tiling together.
경계 모양과 타일링을 함께 계획합니다.

Recipe:
레시피:

- **ONNX graph**: declare the custom op's IFM/OFM with a single flat
- **ONNX 그래프**: 단일 플랫으로 맞춤 작업의 IFM/OFM을 선언합니다.
inner dim that is already DMA-aligned (e.g.
이미 DMA 정렬된 내부 희미함(예:
`shape = [H * W * C]`).
`모양 = [H * W * C]`).
Pass the original geometric shape (`H`, `W`, `C`, ...) as **op
원래 기하학적 모양(`H`, `W`, `C`, ...)을 **op로 전달합니다.
attributes** on the custom op so the semantic meaning is not lost
의미론적 의미가 손실되지 않도록 사용자 정의 작업의 속성**
at the boundary.
경계에서.
Insert `onnx.Reshape` nodes if necessary.
필요한 경우 'onnx.Reshape' 노드를 삽입하세요.
- **Tiling Python**: reshape the L3 (DDR) tensor to the flat view
- **Python 타일링**: L3(DDR) 텐서를 플랫 뷰로 재구성합니다.
before handing it to the DMA, e.g.
DMA에 전달하기 전에, 예를 들어
`ifm.getTensorVar().Reshape([flat_size])` before
`ifm.getTensorVar().Reshape([plat_size])` 이전
`set_l3_to_l2_transfer`, and the symmetric form for OFM with
`set_l3_to_l2_transfer` 및 OFM의 대칭 형식
`set_l2_to_l3_transfer`.
`set_l2_to_l3_transfer`.
Tile in 1D space at the DMA boundary, and
DMA 경계에서 1D 공간의 타일링 및
forward the geometric attributes into `lp_params` so the kernel
기하학적 속성을 'lp_params'로 전달하여 커널
receives them.
그들을 받습니다.
- **Kernel (`kernel.cpp`)**: read `H`, `W`, `C` (and any per-tile
- **커널(`kernel.cpp`)**: `H`, `W`, `C`(및 모든 타일별 읽기)
fields such as `tile_x_start`, `tile_y_start`, `tile_rows`,
`tile_x_start`, `tile_y_start`, `tile_rows`와 같은 필드,
`tile_cols` for a distributed tiling) from `lp_params`.
분산 타일링의 경우 `tile_cols`) `lp_params`에서.
The kernel
커널
keeps full geometric awareness -- strides, loop bounds, per-axis
완전한 기하학적 인식(보행, 루프 경계, 축별) 유지
logic -- while the DMA path only ever sees a 1D, DMA-aligned buffer.
논리 - DMA 경로에는 1D, DMA 정렬 버퍼만 표시됩니다.

Apply this to **every** custom-op surface -- IFM, OFM, intermediates,
IFM, OFM, 중간체 등 **모든** 맞춤형 작업 표면에 이를 적용합니다.
flow/grid -- not just the first one that complained.
흐름/그리드 - 불평한 첫 번째 것이 아닙니다.
If you padded an
패딩을 덧대었다면
inner dim to make compilation pass, you should have used this pattern
내부 희미하게 컴파일을 통과하려면 이 패턴을 사용해야 합니다.
instead.
대신.

### Quantized subgraphs: absorb the boundary DQ/Q, make the op an int op
### 양자화된 하위 그래프: 경계 DQ/Q를 흡수하고 연산을 int 연산으로 만듭니다.

A quantized ONNX model might not contain low-precision operations but it can
양자화된 ONNX 모델에는 정밀도가 낮은 연산이 포함되지 않을 수도 있지만
contain **float operations wrapped in a `DQ -> op -> Q` bracket** -- *fake
**`DQ -> op -> Q` 괄호로 묶인 부동 소수점 연산 포함** -- *가짜
quantization*.
양자화*.
The tensor is stored in the quantized type,
텐서는 양자화된 유형으로 저장됩니다.
`DequantizeLinear` widens it to float, the op computes in float, and
`DeQuantizeLinear`는 이를 부동 소수점으로 확장하고 연산은 부동 소수점으로 계산합니다.
`QuantizeLinear` narrows it back:
`QuantizeLinear`는 범위를 다시 좁힙니다:

```
int8 --DQ--> float --[Conv]--> float --Q--> int8
```

Fake quantization is **int8 only** (VINT8) and is expressed with the
가짜 양자화는 **int8만**(VINT8)이며 다음과 같이 표현됩니다.
standard ONNX `QuantizeLinear` / `DequantizeLinear` ops.
표준 ONNX `QuantizeLinear` / `DeQuantizeLinear` 작업.

The two halves of the bracket are **inverse functions**:
괄호의 두 부분은 **역함수**입니다.
`DQ(x) = (x_q - zp) * scale` and `Q(y) = round(y / scale) + zp` compose to
`DQ(x) = (x_q - zp) * scale` 및 `Q(y) = round(y / scale) + zp`로 구성됩니다.
the identity (up to rounding).
ID(반올림까지).

The bracketed `op` is usually a single operation, but a number of known
대괄호로 묶인 'op'은 일반적으로 단일 작업이지만 알려진 여러 가지 작업이 있습니다.
chains count as one quantized pattern and must be absorbed **together**.
사슬은 하나의 양자화된 패턴으로 간주되며 **함께** 흡수되어야 합니다.
The most common by far are a conv followed by its activation:
지금까지 가장 일반적인 것은 전환 후 활성화되는 것입니다.

- `Conv -> Relu`
- `전환 -> Relu`
- `Conv -> LeakyRelu`
- `전환 -> LeakyRelu`
- `AveragePool -> Mul`
- `AveragePool -> Mul`

(For a comprehensive list check the `mixed-precision` skill suite)
(전체 목록을 보려면 '혼합 정밀도' 기술 모음을 확인하세요)

**The DQ and the Q are a precision annotation, not compute.** They exist
**DQ와 Q는 계산이 아닌 정밀 주석입니다.**
so the graph stays type-correct in ONNX semantics.
따라서 그래프는 ONNX 의미론에서 올바른 유형을 유지합니다.
On the NPU nothing widens to float and
NPU에서는 아무것도 뜨지 않고 넓어지지 않습니다.
nothing narrows back -- the FE detects the pattern and emits genuine lower-precision kernels.
아무것도 좁히지 않습니다. FE는 패턴을 감지하고 실제 저정밀도 커널을 방출합니다.
The
는
round trip through float is never materialized, and the data is processed in the quantized type.
float를 통한 왕복은 절대 구현되지 않으며, 데이터는 양자화된 형태로 처리됩니다.

A custom op can be represented using the real lower-precision types.
실제 정밀도가 낮은 유형을 사용하여 사용자 정의 작업을 표현할 수 있습니다.
So we can consider the QDQ nodes to be part of the custom op and then
따라서 QDQ 노드를 맞춤형 작업의 일부로 간주한 다음
absorb them into your kernel.
커널에 흡수하십시오.
Your custom op is then an op in the
그러면 사용자 정의 작업은
**quantized type itself**.
**양자화된 유형 자체**.

`tutorial/conv2d/conv3x3_int8_2stamp` is a complete worked example (see its
`tutorial/conv2d/conv3x3_int8_2stamp`는 완전한 작업 예제입니다(자세한 내용은
README section "INT8 requantization"); the saturation and rounding modes it
README 섹션 "INT8 재양자화"); 채도 및 반올림 모드
sets at kernel entry are *strictly required* for bit-exactness, not a
커널 항목의 세트는 비트 정확성을 위해 *엄격히 필요*합니다.
nicety.
친절함.

> **Not every quantization format is a bracket.** MX6/BFP quantization is
> **모든 양자화 형식이 대괄호인 것은 아닙니다.** MX6/BFP 양자화는
> a *single* op -- the AMD Quark `com.amd.quark.BFPQuantizeDequantize`
> *단일* 작업 -- AMD Quark `com.amd.quark.BFPQuantizeDeQuarkize`
> node, which quantizes and dequantizes in one shot -- placed on each
> 한 번에 양자화 및 역양자화하는 노드 - 각 노드에 배치됨
> input of the operation it quantizes:
> 양자화하는 연산의 입력:
> ```
> com.amd.quark.BFPQuantizeDequantize -> op
> com.amd.quark.BFPQuantizeDeQuarkize -> op
> ```
> so detect it by that shape -- there is no trailing counterpart to pair
> 따라서 그 모양으로 감지하십시오. 쌍을 이룰 후속 상대가 없습니다.
> with, and looking for a `DQ -> op -> Q` sandwich will miss it entirely.
> with를 사용하고 `DQ -> op -> Q` 샌드위치를 찾으면 완전히 놓칠 것입니다.

> **MX6 CUSTOM OPS ARE EXPERIMENTAL AND HAVE NO KNOWN EXAMPLES.** You MUST NOT
> **MX6 사용자 정의 OPS는 실험적이며 알려진 예가 없습니다.**
> start one on your own initiative -- stop, say so, and get explicit user consent.
> 스스로 시작하십시오. 중지하고 그렇게 말하고 명시적인 사용자 동의를 얻으십시오.

For more detail on any of this -- the quantization formats and how they
이에 대한 자세한 내용은 양자화 형식 및 양자화 방법을 참조하세요.
encode scales, which precisions each device supports, and how quantized
스케일 인코딩, 각 장치가 지원하는 정밀도 및 양자화 방법
boundaries are handled at the whole-model level -- see the
경계는 전체 모델 수준에서 처리됩니다.
**`mixed-precision`** skill (the quantization orchestrator).
**`혼합 정밀도`** 기술(양자화 조정자).
It is the
그것은
reference for model-level quantization decisions; this section covers
모델 수준의 양자화 결정에 대한 참조; 이 섹션에서는 다룹니다.
only what a custom op author needs at the subgraph boundary.
하위 그래프 경계에서 사용자 정의 작업 작성자에게 필요한 것만.

## Compiler Flags: Do Not Touch the Model-Wide Knobs
## 컴파일러 플래그: 모델 전체 손잡이를 건드리지 마십시오.

The `vitisai_config.json` flags are **model-wide**: one overlay and one set of
`vitisai_config.json` 플래그는 **모델 전체**입니다. 하나의 오버레이와 하나의 세트
compiler settings serve every partition.
컴파일러 설정은 모든 파티션에 적용됩니다.
A flag you change to make your op fit
작전에 적합하도록 변경하는 플래그
therefore changes how every other layer is compiled, and the damage does not show
따라서 다른 모든 레이어가 컴파일되는 방식이 변경되고 손상은 표시되지 않습니다.
up where you made it -- your op keeps compiling and passing x86sim, while the
당신이 만든 곳까지 - 당신의 작업은 x86sim을 계속 컴파일하고 전달하는 반면
failure surfaces at integration (Phases 6-7), in layers you never touched.
통합 시(6~7단계) 실패 표면은 건드리지 않은 레이어에 있습니다.
Treat
치료하다
the following as fixed unless the user explicitly tells you otherwise:
사용자가 명시적으로 다르게 지시하지 않는 한 다음은 수정된 것입니다.

- **Overlay selection.** The overlay fixes the core-tile grid and the DMA
- **오버레이 선택.** 오버레이는 코어 타일 그리드와 DMA를 수정합니다.
source/destination topology for the entire model.
전체 모델에 대한 소스/대상 토폴로지.
Design your tiling for the
타일링을 디자인하세요.
overlay you were given instead of asking for a different one.
다른 오버레이를 요청하는 대신 오버레이가 제공되었습니다.
- **`optimize_level`.** It has wide repercussions -- it can change which overlay
- **`optimize_level`.** 광범위한 영향을 미칩니다. 어떤 오버레이를 변경할 수 있습니다.
is chosen and how many AIE-to-DDR connections are provisioned -- and it applies
선택되었으며 AIE-DDR 연결이 몇 개 프로비저닝되어 있으며 적용됩니다.
to the whole model.
전체 모델에.
Leave it alone.
그대로 두십시오.
- **`stack_size` / `heap_size`.** These steal L1 from the tiler's buffers.
- **`stack_size` / `heap_size`.** 타일러의 버퍼에서 L1을 훔칩니다.
- Leave
- 휴가
them at their defaults and shrink the kernel instead -- see
기본값으로 설정하고 대신 커널을 축소합니다.
[Stack and Heap Sizing](#stack-and-heap-sizing).
[스택 및 힙 크기 조정](#stack-and-heap-sizing).

## Stack and Heap Sizing
## 스택 및 힙 크기 조정

**Do NOT add or modify `stack_size` or `heap_size` in a custom-op YAML.** A stack
**맞춤 작업 YAML에서 `stack_size` 또는 `heap_size`를 추가하거나 수정하지 마세요.** 스택
or heap overflow is a signal that the **kernel** has too much live state.
또는 힙 오버플로는 **커널**에 라이브 상태가 너무 많다는 신호입니다.
Enlarging `stack_size`/`heap_size` silently steals L1 from the tiler's
`stack_size`/`heap_size`를 확대하면 타일러의 L1을 자동으로 훔칩니다.
double-buffered inputs and outputs (those you can transfer into with
이중 버퍼링된 입력 및 출력(다음으로 전송할 수 있는 것)
`set_l2_to_l1_transfer`), so a "fix" here quietly shrinks the tiles you
`set_l2_to_l1_transfer`), 따라서 여기서 "수정"을 하면 타일이 조용히 축소됩니다.
can afford elsewhere and masks a kernel that is too heavy.
다른 곳에서는 여유가 있고 너무 무거운 커널을 마스킹할 수 있습니다.
The measured peak in
측정된 피크는
the `.calltree`/`.map` artifacts is a **diagnostic** to tell you what to shrink
`.calltree`/`.map` 아티팩트는 무엇을 축소해야 하는지 알려주는 **진단**입니다.
in the kernel -- never a value to copy into the YAML.
커널에서 -- YAML에 복사할 값이 없습니다.
See "Determining the stack
"스택 결정"을 참조하십시오.
and heap size" in `README.md` for how to read those two files, but ignore its
두 파일을 읽는 방법은 `README.md`에 나와 있지만 무시하세요.
closing advice to put the measured peak in the YAML.
측정된 피크를 YAML에 넣으라는 종료 조언.

### The four checks you MUST apply
### 반드시 적용해야 하는 4가지 수표

Measure as above, then genuinely apply **all four** of the following:
위와 같이 측정한 후 다음의 **네 가지**를 모두 적용하십시오.

1. **Move scratch out of the stack/heap into a framework-managed buffer.** Any
1. **스택/힙의 스크래치를 프레임워크 관리 버퍼로 이동합니다.** 모두
multi-KB working buffer belongs in an ADF/DM buffer; the usual idiom is to
다중 KB 작업 버퍼는 ADF/DM 버퍼에 속합니다. 일반적인 관용어는 다음과 같습니다
over-allocate the OFM L1 buffer so it holds `output + scratch` and to drain
OFM L1 버퍼를 과도하게 할당하여 '출력 + 스크래치'를 유지하고 배수합니다.
only the real output sub-view on the L1->L2 DMA.
L1->L2 DMA의 실제 출력 하위 보기만 가능합니다.
The kernel binds the full
커널은 전체를 바인딩합니다.
allocation, so it has room to scribble with no `static`, no heap, no stack.
할당이 없으므로 `정적`, 힙, 스택 없이 낙서할 수 있는 공간이 있습니다.
See
참조
the oversized-OFM-buffer section of `tutorial/conv2d/README.md` for the sizing
크기 조정을 위한 `tutorial/conv2d/README.md`의 대형 OFM 버퍼 섹션
rule and the sliced drain.
규칙과 얇게 썬 배수구.
2. **Shrink live sets via loop splitting** (see `vai-aie-kernel-development`):
2. **루프 분할을 통해 라이브 세트 축소**(`vai-aie-kernel-development` 참조):
break one fat loop into passes that hand intermediates through DM, so each loop
하나의 굵은 루프를 DM을 통해 중간에 전달하는 패스로 나누므로 각 루프
has fewer simultaneously-live vectors.
동시 활성화 벡터 수가 더 적습니다.
3. **Keep temporaries small (per-row / 8-lane)** (see `vai-aie-kernel-development`):
3. **임시 파일을 작게 유지하세요(행당/8레인)**(`vai-aie-kernel-development` 참조):
process one 8-lane row at a time instead of keeping full 64-lane tiles live;
전체 64레인 타일을 실시간으로 유지하는 대신 한 번에 하나의 8레인 행을 처리합니다.
use named locals instead of addressable vector arrays; keep reductions in
주소 지정이 가능한 벡터 배열 대신 명명된 지역 변수를 사용하세요. 감소를 유지하다
registers.
레지스터.
Remove dynamic/temporary allocations and static scratch/LUTs (they
동적/임시 할당 및 정적 스크래치/LUT를 제거합니다.
consume heap).
힙을 소비합니다).
4. **Constrain pipelining on the pressure-heavy loops.** Over-aggressive
4. **압력이 심한 루프의 파이프라인을 제한합니다.** 지나치게 공격적입니다.
pipelining/unrolling widens live ranges and forces spills; disable or limit it
파이프라이닝/언롤링은 활성 범위를 넓히고 유출을 발생시킵니다. 비활성화하거나 제한
on the specific loop the `.lst` shows spilling, and re-measure.
특정 루프에서 `.lst`는 유출을 표시하고 다시 측정합니다.

A correct kernel that fits the default budget beats a marginally faster one that needs a bigger budget.
기본 예산에 맞는 올바른 커널은 더 큰 예산이 필요한 약간 더 빠른 커널을 능가합니다.

## Phase 1: Subgraph Extraction
## 1단계: 하위 그래프 추출

**Note:** If the model contains only a single node, there is no subgraph to extract -- skip Phase 1a and use the model directly as both Model A and the basis for Model B.
**참고:** 모델에 노드가 하나만 포함된 경우 추출할 하위 그래프가 없습니다. 1a단계를 건너뛰고 모델을 모델 A와 모델 B의 기초로 직접 사용합니다.

**Understand the subgraph's neighbors before extracting.** A custom op's producers and consumers constrain the DDR memory layout of its inputs and outputs.
**추출하기 전에 하위 그래프의 이웃을 이해하십시오.** 맞춤형 연산의 생산자와 소비자는 입력 및 출력의 DDR 메모리 레이아웃을 제한합니다.
Matching those layouts inside the custom op is normally cheap -- a few cycles in the cores or a different DMA configuration, negligible next to the main compute.
사용자 지정 작업 내에서 이러한 레이아웃을 일치시키는 것은 일반적으로 저렴합니다. 즉, 코어의 몇 사이클 또는 다른 DMA 구성은 기본 컴퓨팅 옆에서 무시할 수 있습니다.
Ignoring them forces extra data movement at integration, which can be far more expensive and can make the whole op impossible to integrate.
이를 무시하면 통합 시 추가 데이터 이동이 강제되어 훨씬 더 많은 비용이 들고 전체 작업을 통합하는 것이 불가능해질 수 있습니다.
So investigate the neighbors' DDR layout requirements up front rather than discovering the mismatch in Phase 6/7.
따라서 6/7단계에서 불일치를 발견하기보다는 이웃의 DDR 레이아웃 요구 사항을 미리 조사하십시오.

Since `--nodes` is provided as a skill argument, extract the subgraph directly.
`--nodes`가 스킬 인수로 제공되므로 하위 그래프를 직접 추출합니다.
The script auto-infers input/output tensor cut points from the node names.
스크립트는 노드 이름에서 입력/출력 텐서 컷 포인트를 자동 추론합니다.

### 1a.
### Extract the subgraph (Model A)
### 하위 그래프 추출(모델 A)

```bash
<ENV_SETUP>
mkdir -p <working_dir>
python <skill_dir>/scripts/onnx_cut.py \
    --model <full_model>.onnx \
    --nodes <node_1> [<node_2> ...] \
    --output <working_dir>/model_a.onnx
```

The extracted `model_a.onnx` is the **reference model** -- it uses standard ONNX ops and runs on CPU for golden reference verification.
추출된 'model_a.onnx'는 **참조 모델**입니다. 이는 표준 ONNX 작업을 사용하고 골든 참조 확인을 위해 CPU에서 실행됩니다.

If the node names are ambiguous or you need to inspect the model first, use `--report`:
노드 이름이 모호하거나 모델을 먼저 검사해야 하는 경우 `--report`를 사용하세요.
```bash
python <skill_dir>/scripts/onnx_cut.py --model <full_model>.onnx --report
```

### 1b.
### Generate CPU reference data
### CPU 참조 데이터 생성

Run the extracted model on CPU to produce golden reference outputs:
CPU에서 추출된 모델을 실행하여 골든 참조 출력을 생성합니다.

```bash
python <skill_dir>/scripts/compile.py \
    <working_dir>/model_a.onnx --cpu \
    --input-dir <working_dir>/reference_data \
    --output-dir <working_dir>/reference_data
```

**Pass both `--input-dir` and `--output-dir`, pointing at `reference_data`.**
**`reference_data`를 가리키는 `--input-dir`과 `--output-dir`을 모두 전달합니다.**
`--output-dir` covers only the outputs; without `--input-dir` the inputs go to
`--output-dir`은 출력만 다룹니다. `--input-dir` 없이 입력은 다음으로 이동합니다.
`<model_stem>_inputs/` and every later step expecting `reference_data/inputs.npz`
`<model_stem>_inputs/` 및 `reference_data/inputs.npz`가 필요한 모든 이후 단계
(Phase 4 x86sim, Phase 5 board run) fails to find it.
(Phase 4 x86sim, Phase 5 보드 실행)을 찾지 못했습니다.
The directory is generated
디렉토리가 생성됩니다
on first use and its inputs reused on re-runs.
처음 사용할 때 입력이 다시 실행될 때 재사용됩니다.

This saves:
이렇게 하면 다음이 절약됩니다.
- `reference_data/inputs.npz` (plus one `.npy` per input) -- inputs used
- `reference_data/inputs.npz`(입력당 하나의 `.npy` 추가) -- 사용된 입력
(**must be reused for x86sim in Phase 4**)
(**4단계에서 x86sim에 재사용되어야 함**)
- `reference_data/outputs.npz` -- golden reference outputs
- `reference_data/outputs.npz` -- 골든 참조 출력
- `reference_data/<output_name>.npy` -- individual output files for comparison with `compare_npy.py`
- `reference_data/<output_name>.npy` -- `compare_npy.py`와 비교하기 위한 개별 출력 파일

### 1c.
### Create Model B (Custom Op model)
### 모델 B 생성(Custom Op 모델)

Generate the custom op ONNX model for the nodes from Model A that correspond to the custom op using `onnx_stitch.py`:
'onnx_stitch.py'를 사용하여 사용자 정의 작업에 해당하는 모델 A의 노드에 대한 사용자 정의 작업 ONNX 모델을 생성합니다.

```bash
python <skill_dir>/scripts/onnx_stitch.py \
    --model <working_dir>/model_a.onnx \
    --nodes <node_name_or_op_type> \
    --custom-op-name <op_name>
```

## Phase 2: Create the Custom Op
## 2단계: 사용자 정의 작업 생성

### 2a.
### Context analysis
### 상황 분석

Phase 1 highlights the importance of looking at the surrounding context.
1단계에서는 주변 상황을 살펴보는 것의 중요성을 강조합니다.
A custom op does not exist on its own -- it is part of a bigger model.
사용자 정의 작업은 자체적으로 존재하지 않으며 더 큰 모델의 일부입니다.
That context must be established before you write a line of kernel code, and it must stay in view for the rest of the flow.
해당 컨텍스트는 커널 코드 줄을 작성하기 전에 설정되어야 하며 나머지 흐름에서 계속 확인되어야 합니다.
Subgraph extraction exists only to shorten the round trip from modification to verification;
하위 그래프 추출은 수정에서 검증까지의 왕복 시간을 단축하기 위해서만 존재합니다.

A custom op may inherit DDR layout requirements from its producer and consumer.
맞춤형 작업은 생산자와 소비자로부터 DDR 레이아웃 요구 사항을 상속받을 수 있습니다.
A convolution or matmul feeding the op, for instance, constrains the layout of the DDR tensor it ingests.
예를 들어 op에 공급되는 컨볼루션 또는 matmul은 그것이 수집하는 DDR 텐서의 레이아웃을 제한합니다.
Encode those constraints in the YAML through the `ddr_tensor_layout` / `onnx_tensor_layout` fields.
`ddr_tensor_layout` / `onnx_tensor_layout` 필드를 통해 YAML에서 해당 제약 조건을 인코딩합니다.
If you don't, the compiler inserts transposes around the custom op, and those transposes routinely cost more than the custom op saves.
그렇지 않으면 컴파일러가 사용자 정의 작업 주위에 전치를 삽입하고 이러한 전치 작업은 사용자 정의 작업에서 절약하는 것보다 일상적으로 비용이 더 많이 듭니다.

**The goal of the custom op is to absorb any data movement that is not part of the original ONNX model.** Even when the baseline (or `model_a.onnx`) contains a transpose merely to connect two operations -- one of which becomes the custom op -- that transpose must be absorbed into the custom op rather than left standing beside it.
**커스텀 작업의 목표는 원래 ONNX 모델의 일부가 아닌 모든 데이터 이동을 흡수하는 것입니다.** 기준선(또는 `model_a.onnx`)에 단순히 두 작업을 연결하기 위한 전치가 포함되어 있는 경우에도(그 중 하나는 사용자 정의 작업이 됩니다) 해당 전치는 옆에 두지 않고 사용자 정의 작업에 흡수되어야 합니다.

DDR Layout is an essential component of the custom op; it determines the tiling.
DDR 레이아웃은 맞춤형 작업의 필수 구성 요소입니다. 타일링을 결정합니다.
Discovering a layout requirement after the kernel works means the tiling you built and tuned was never the final one.
커널이 작동한 후 레이아웃 요구 사항을 발견한다는 것은 귀하가 만들고 조정한 타일링이 결코 최종 타일링이 아니라는 것을 의미합니다.

Consequence of not considering the custom op context:
사용자 정의 작업 컨텍스트를 고려하지 않은 결과:

- **Implementation is thrown away** Absorbing a layout usually forces a different retiling, e.g. transposed L3/L2 shapes or a different DMA configuration.
- **구현은 폐기됩니다** 레이아웃을 흡수하면 일반적으로 다른 타일링이 적용됩니다. 전치된 L3/L2 모양 또는 다른 DMA 구성.
- That invalidates the kernel structure, the L1/L2 buffer shapes, and the tiling script together.
- 이는 커널 구조, L1/L2 버퍼 모양 및 타일링 스크립트를 함께 무효화합니다.
- **The optimization rounds are spent against the wrong target.** Every round of board-guided tuning is measured on a tiling that is about to be replaced.
- **최적화 라운드는 잘못된 목표에 대해 소비됩니다.** 보드 기반 튜닝의 모든 라운드는 곧 교체될 타일링에서 측정됩니다.
- **The dead end arrives late and is expensive to leave.** By the time an op is complete and validated, a retiling is a large, risky change on working code.
- **막다른 골목은 늦게 도착하고 떠나는 데 비용이 많이 듭니다.** 작업이 완료되고 검증될 때까지 타일링 작업은 작업 코드에 대한 크고 위험한 변경입니다.
- In practice the layout fix gets deferred, and the model ships with a compiler-inserted transpose.
- 실제로 레이아웃 수정이 연기되고 모델은 컴파일러에 삽입된 전치와 함께 제공됩니다.

Doing the analysis up front costs one compilation of the neighbourhood.
분석을 미리 수행하려면 이웃을 한 번 편집하는 데 비용이 듭니다.
Doing it late costs the kernel.
늦게 수행하면 커널 비용이 발생합니다.

#### Output of this phase
#### 이 단계의 출력

Produce a report of the form:
다음 형식의 보고서를 생성합니다.

```
input_<n>:
  async: <false|true>
  onnx_tensor_layout: <layout>
  ddr_tensor_layout: <layout>
output_<n>:
  async: <false|true>
  onnx_tensor_layout: <layout>
  ddr_tensor_layout: <layout>
```

If no layout change is required, leave the report empty and the next stage won't set any DDR requirements.
레이아웃 변경이 필요하지 않은 경우 보고서를 비워두면 다음 단계에서 DDR 요구 사항이 설정되지 않습니다.
If one argument doesn't need to change but another one does, just set `async: <false|true>` and don't set `onnx_tensor_layout`/`ddr_tensor_layout`.
한 인수를 변경할 필요가 없지만 다른 인수는 변경할 필요가 있는 경우 `async: <false|true>`를 설정하고 `onnx_tensor_layout`/`ddr_tensor_layout`을 설정하지 마세요.
The `async` field is required when specifying the DDR tensor layout, but `onnx_tensor_layout`/`ddr_tensor_layout` are optional.
DDR 텐서 레이아웃을 지정할 때 `async` 필드는 필수이지만 `onnx_tensor_layout`/`ddr_tensor_layout`은 선택 사항입니다.

`input_<n>` and `output_<n>` can be set seperately.
`input_<n>`과 `output_<n>`은 별도로 설정할 수 있습니다.
This means that if we only have requirements on the output we don't need to specify a layout for the input and vice-versa.
즉, 출력에 대한 요구사항만 있는 경우 입력에 대한 레이아웃을 지정할 필요가 없으며 그 반대의 경우도 마찬가지입니다.

As the onnx/ddr layout information can only be applied to the original onnx inputs and outputs.
onnx/ddr 레이아웃 정보는 원래 onnx 입력 및 출력에만 적용될 수 있습니다.
Scratch buffers don't have any layout because they are only writen and read by the same custom op.
스크래치 버퍼에는 동일한 사용자 정의 작업으로만 쓰고 읽기 때문에 레이아웃이 없습니다.

### 2b.
### ONNX Custom Op YAML
### ONNX 사용자 지정 작업 YAML

Model B (the custom op ONNX model) was already created in Phase 1c.
모델 B(맞춤형 Op ONNX 모델)는 이미 1c단계에서 생성되었습니다.
Now create the remaining files in `<working_dir>/`:
이제 `<working_dir>/`에 나머지 파일을 만듭니다.

Model B was generated in Phase 1.
모델 B는 1단계에서 생성되었습니다.
If you need to customize it further (e.g., add extra attributes), you can edit the .onnxtxt version.
추가로 사용자 정의해야 하는 경우(예: 추가 속성 추가) .onnxtxt 버전을 편집할 수 있습니다.
See [README.md](../../README.md) for the `.onnxtxt` format and how to convert between `.onnx` and `.onnxtxt`.
`.onnxtxt` 형식과 `.onnx`와 `.onnxtxt` 간 변환 방법은 [README.md](../../README.md)를 참조하세요.

The model must have `shape` and `data_type` attributes on the custom op node (required by VAIML shape inference):
모델에는 사용자 정의 작업 노드에 'shape' 및 'data_type' 속성이 있어야 합니다(VAIML 모양 추론에 필요함).

**Manual creation (if needed) using .onnxtxt format:**
**.onnxtxt 형식을 사용한 수동 생성(필요한 경우):**
```
<
    ir_version: 7,
    opset_import: ["" : 13, "mydomain" : 1]
>
test(float[<shape>] <inputs>) => (float[<shape>] <output>) {
    <output> = mydomain.<op_name><data_type = "float32", shape = [<shape>]>(<inputs>)
}
```

**Using Python (onnx helper):**
**Python 사용(onnx 도우미):**
```python
node = onnx.helper.make_node(
    "<op_name>",
    inputs=[...],
    outputs=[...],
    domain="mydomain",
    shape=[...],        # REQUIRED: output shape
    data_type="float32" # REQUIRED: output data type
)
```

### 2c.
### Config YAML (`custom_op_<name>/<name>.yaml`)
### YAML 구성(`custom_op_<이름>/<이름>.yaml`)

```yaml
---
tiling: "<name>_tiling.py"
```

Need a different DDR layout than the ONNX one (e.g.
ONNX와 다른 DDR 레이아웃이 필요합니다(예:
NCHW -> HCWN_C8 for a conv)?
전환의 경우 NCHW -> HCWN_C8)?
Add `onnx_tensor_layout` / `ddr_tensor_layout` per input/output and VAIML folds
입력/출력 및 VAIML 접기당 `onnx_tensor_layout` / `ddr_tensor_layout` 추가
the conversion into the DMA (prefer this over pre-arranging data offline):
DMA로의 변환(오프라인으로 데이터를 미리 정렬하는 것보다 선호함):
```yaml
  inputs: # add one entry per input
    - onnx_tensor_layout: "N C H W"
      ddr_tensor_layout: "H C/8 W N 8"   # HCWN_C8 (C vectorized by 8)
```
See `tutorial/conv2d/README.md` -> "Input layout: HCWN_C8, and who converts to
`tutorial/conv2d/README.md`를 참조하세요 -> "입력 레이아웃: HCWN_C8, 누가 변환하는지
it" and the main [README.md](../../README.md) "Data layout specification".
it" 및 기본 [README.md](../../README.md) "데이터 레이아웃 사양"입니다.

You can figure out which data layout is used by the producer and consumer of your custom op by looking at the compilation log.
컴파일 로그를 보면 사용자 정의 작업의 생산자와 소비자가 어떤 데이터 레이아웃을 사용하는지 파악할 수 있습니다.
It is emitted as a INFO.
INFO로 출력됩니다.
For instance:
예를 들어:

```
Custom op Unary1 (mydomain.unary) operand 0 uses a different memory layout than its producer, possibly causing suboptimal data movement. This op: ONNX tensor layout = "A B C D", DDR Layout = "A B C/8 D 8"; producer op: ONNX layout = "A B C D", DDR layout = "A B C D".
```

### 2d.
### Kernel C++ (`custom_op_<name>/<name>.cpp`)
### 커널 C++(`custom_op_<이름>/<이름>.cpp`)

**MANDATORY: Target a 4x4 distributed implementation out of the box.** This is the default and expected starting point -- write the kernel so it can run on all 16 cores with broadcast IFM and per-core OFM collection.
**필수: 즉시 4x4 분산 구현을 목표로 합니다.** 이것이 기본이자 예상되는 시작점입니다. 브로드캐스트 IFM 및 코어당 OFM 수집을 통해 16개 코어 모두에서 실행될 수 있도록 커널을 작성합니다.
You can use the following templates (see below) to get started.
다음 템플릿(아래 참조)을 사용하여 시작할 수 있습니다.

- **Unary op** (one IFM): read `tutorial/negate/2_distributed/`
- **단항 연산**(IFM 1개): `tutorial/negate/2_distributed/`를 읽어보세요.
(`negate_tiling.py`, `kernel/negate_kernel.cpp`, `custom_op_negate.yaml`)
(`negate_tiling.py`, `kernel/negate_kernel.cpp`, `custom_op_negate.yaml`)
in full.
전체.
- **Binary op** (two IFMs): read `tutorial/mul/2_distributed/`
- **바이너리 작업**(두 개의 IFM): `tutorial/mul/2_distributed/`를 읽어보세요.
(`custom_mul_tiling.py`, `kernel/custom_mul_kernel.cpp`,
(`custom_mul_tiling.py`, `kernel/custom_mul_kernel.cpp`,
`custom_op_mul.yaml`) in full. **The second IFM must be
`custom_op_mul.yaml`) 전체. **두 번째 IFM은 다음과 같아야 합니다.
Transpose-distributed across columns due to the overlay
오버레이로 인해 열 전체에 걸쳐 전치-분산됨
architecture** -- the `mul/2_distributed` template is the canonical
아키텍처** -- `mul/2_distributed` 템플릿이 표준입니다.
example of how to do this.
이를 수행하는 방법의 예입니다.
- **Six-column stamping**: read `tutorial/mul/4_distributed_6stamp/`
- **6열 스탬핑**: `tutorial/mul/4_distributed_6stamp/`를 읽어보세요.
(and the negate equivalent if you need a unary 6-stamp reference)
(그리고 단항 6-스탬프 참조가 필요한 경우에는 그에 상응하는 부정)
so you understand how the 4x4 grid is replicated across the 6
그래서 4x4 그리드가 6개 그리드에 걸쳐 복제되는 방식을 이해했습니다.
columns of the overlay.
오버레이의 열입니다.

The focus on this section is to get a kernel that is functionally correct.
이 섹션의 초점은 기능적으로 올바른 커널을 얻는 것입니다.
Performance is a secondary concern but some optimizations that don't increase complexity can already be tackled here.
성능은 부차적인 문제이지만 복잡성을 증가시키지 않는 일부 최적화는 이미 여기에서 해결될 수 있습니다.
For example, you should prefer native hardware type like `bfloat16` over software emulated types like `float32/64`.
예를 들어 `float32/64`와 같은 소프트웨어 에뮬레이트 유형보다 `bfloat16`과 같은 기본 하드웨어 유형을 선호해야 합니다.

#### Analyze Parallelism Before Choosing a Tiling Strategy
#### 타일링 전략을 선택하기 전에 병렬성을 분석하십시오.

**CRITICAL: Do a proper analysis of the algorithm's parallelism.** The presence of a sequential dependency in one dimension does NOT mean the entire operation is inherently sequential.
**중요: 알고리즘의 병렬성을 적절하게 분석하십시오.** 한 차원에 순차적 종속성이 존재한다고 해서 전체 작업이 본질적으로 순차적이라는 의미는 아닙니다.
Different dimensions of the computation may have independent parallelism opportunities.
계산의 다양한 차원은 독립적인 병렬화 기회를 가질 수 있습니다.

**Methodology -- work backward from the output to the inputs:**
**방법론 - 출력에서 입력으로 역방향 작업:**

1. **Identify the output dimensions.** What is the shape of the output tensor?
1. **출력 차원을 식별합니다.** 출력 텐서의 모양은 무엇입니까?
1. Each dimension is a candidate for tiling/distribution across cores.
1. 각 차원은 코어 전체의 타일링/분산을 위한 후보입니다.

2. **For each output dimension, trace the data dependencies.** Ask: "Can two different output elements along this dimension be computed independently?" If yes, this dimension can be tiled across cores.
2. **각 출력 차원에 대해 데이터 종속성을 추적합니다.** 질문: "이 차원에 따른 두 개의 서로 다른 출력 요소를 독립적으로 계산할 수 있습니까?" 그렇다면 이 차원은 코어 전체에 타일링될 수 있습니다.

3. **Separate sequential dependencies from parallel dimensions.** An operation may have a loop-carried dependency in one dimension (e.g., a recurrence across a sequence) while being fully parallel in another dimension (e.g., independent output features).
3. **병렬 차원과 별도의 순차 종속성.** 작업은 한 차원(예: 시퀀스 전반에 걸쳐 반복)에서 루프 전달 종속성을 가질 수 있지만 다른 차원(예: 독립적인 출력 기능)에서는 완전히 병렬일 수 있습니다.
3. These are **separate concerns** -- a dependency in one dimension does not eliminate parallelism in another.
3. 이는 **별도의 문제**입니다. 한 차원의 종속성이 다른 차원의 병렬성을 제거하지 않습니다.

4. **Choose the tiling strategy based on the parallel dimensions.** Distribute the parallel dimensions across the 4x4 grid.
4. **평행 치수를 기준으로 타일링 전략을 선택합니다.** 4x4 그리드에 평행 치수를 배포합니다.
4. Keep the sequential dimensions within each core (processed in a loop).
4. 각 코어 내에서 순차적 차원을 유지합니다(루프에서 처리됨).

5. **Only after exhausting all dimensions** should you consider implementing the op sequentially on a single core.
5. **모든 차원을 모두 사용한 후에만** 단일 코어에서 작업을 순차적으로 구현하는 것을 고려해야 합니다.
5. A proper analysis may reveal that an operation assumed to be "inherently sequential" actually has significant parallelism in its output dimensions.
5. 적절한 분석을 통해 "본질적으로 순차적"이라고 가정된 작업이 실제로 출력 차원에서 상당한 병렬성을 갖는다는 사실을 밝힐 수 있습니다.

**Example pattern:** An operation computes `output[seq, features]` where `seq` has a recurrence (each step depends on the previous) but `features` is fully independent.
**예시 패턴:** 작업은 `output[seq, feature]`를 계산합니다. 여기서 `seq`는 반복되지만(각 단계는 이전 단계에 따라 다름) `features`는 완전히 독립적입니다.
The wrong conclusion is "recurrence means single core." The correct approach is to tile the `features` dimension across 16 cores while processing `seq` sequentially within each core.
잘못된 결론은 "재발은 단일 코어를 의미한다"입니다. 올바른 접근 방식은 16개 코어에 걸쳐 '기능' 차원을 타일링하는 동시에 각 코어 내에서 'seq'를 순차적으로 처리하는 것입니다.
This gives 16x parallel throughput on the feature computation even though the sequence is sequential.
이는 시퀀스가 ​​순차적이더라도 특징 계산에서 16배의 병렬 처리량을 제공합니다.

**Common mistake:** Seeing a sequential dependency and immediately deciding the op has no parallelism at all.
**일반적인 실수:** 순차 종속성을 보고 즉시 작업을 결정하는 것은 병렬성이 전혀 없습니다.
This wastes 15 of 16 available cores.
이로 인해 사용 가능한 코어 16개 중 15개가 낭비됩니다.
Always ask: "Which dimensions CAN be parallelized?" before asking "Which dimensions CANNOT?"
항상 질문하세요: "병렬화할 수 있는 차원은 무엇입니까?" "어떤 측정기준을 사용할 수 없나요?"라고 묻기 전에

**NOTE:** If you need to include `printf` and `<cstdio>` for debug purposes, you must safeguard it with `#ifdef __X86SIM__`.
**참고:** 디버그 목적으로 `printf` 및 `<cstdio>`를 포함해야 하는 경우 `#ifdef __X86SIM__`을 사용하여 보호해야 합니다.
This ensure that the debug messages are only active during x86 simulation and won't break AIE compilation.
이렇게 하면 디버그 메시지가 x86 시뮬레이션 중에만 활성화되고 AIE 컴파일이 중단되지 않습니다.

**Unary kernel template:**
**단항 커널 템플릿:**
```cpp
#include <adf.h>
#include <aie_api/aie.hpp>

namespace custom_ops {
static constexpr int <name>_lp_size = 1;

template <typename dtype_ifm, typename dtype_ofm>
__attribute__((noinline)) void <kernel_name>(
    adf::input_buffer_conf<dtype_ifm, adf::bpc_sync_0d> &__restrict ifm,
    adf::output_buffer_conf<dtype_ofm, adf::bpc_sync_0d> &__restrict ofm,
    const uint32_t (&lp_params)[<name>_lp_size]) {
  auto *in = (dtype_ifm *__restrict)(ifm.data());
  auto *out = (dtype_ofm *__restrict)(ofm.data());
  uint32_t size = lp_params[0];
  for (unsigned i = 0; i < size; ++i) {
    out[i] = /* compute */;
  }
}
} // namespace custom_ops
```

**Binary kernel template** -- add second input buffer (and a template parameter `dtype_wts`) between `ifm` and `ofm`.
**바이너리 커널 템플릿** -- `ifm`과 `ofm` 사이에 두 번째 입력 버퍼(및 템플릿 매개변수 `dtype_wts`)를 추가합니다.

#### Kernel Guidelines Checklist
#### 커널 지침 체크리스트

Go through every item below before you call the kernel done.
커널 완료를 호출하기 전에 아래의 모든 항목을 살펴보세요.
These are silent
이들은 침묵한다
failure modes -- the kernel compiles either way.
실패 모드 - 커널은 어느 쪽이든 컴파일됩니다.

- [ ] **CRITICAL: Always prefer AIE API library calls over ad-hoc implementations of algorithms -- never use `std::`.** The AIE API (`aie::` namespace, from `<aie_api/aie.hpp>`) provides highly optimized, vectorized operations that are portable across AIE architectures.
- [ ] **중요: 항상 알고리즘의 임시 구현보다 AIE API 라이브러리 호출을 선호합니다. 절대로 `std::`를 사용하지 마십시오.** AIE API(`<aie_api/aie.hpp>`의 `aie::` 네임스페이스)는 AIE 아키텍처 전체에 이식할 수 있는 고도로 최적화되고 벡터화된 작업을 제공합니다.
- Ad-hoc implementations (LUTs, polynomial approximations) are slower, less accurate, and not portable.
- 임시 구현(LUT, 다항식 근사)은 속도가 느리고 정확도가 떨어지며 이식성이 없습니다.
**Before writing or optimizing any kernel math you MUST read the bundled AIE API
**커널 수학을 작성하거나 최적화하기 전에 번들로 제공되는 AIE API를 읽어야 합니다.
reference** --
참고** --
`aie_api_references.md`, which
`aie_api_references.md`,
ships with the **vai-aie-kernel-development** skill.
**vai-aie-kernel-development** 스킬과 함께 제공됩니다.
It is the detailed catalog of
상세한 카탈로그 입니다
AIE API primitives -- core (load/store, arithmetic, reductions), fused /
AIE API 프리미티브 -- 코어(로드/저장, 산술, 축소), 융합/
advanced (`mac_square`, `msc_square`, `reduce_add_v`,
고급(`mac_square`, `msc_square`, `reduce_add_v`,
`invsqrt`, `to_float`/`to_fixed`), and transcendentals/composition examples
`invsqrt`, `to_float`/`to_fixed`) 및 초월/구성 예
(`tanh`, `exp2`, sigmoid-via-tanh) -- giving each primitive's dtype support,
(`tanh`, `exp2`, sigmoid-via-tanh) -- 각 기본 요소의 dtype 지원 제공,
vectorization, accumulator type, and a math-pattern -> API lookup.
벡터화, 누산기 유형 및 수학 패턴 -> API 조회.
Going through
통과하다
it is not optional: it is how you pick the idiomatic vectorized/fused op instead
선택 사항이 아닙니다. 대신 관용적 벡터화/융합 연산을 선택하는 방법입니다.
of guessing a signature, hand-rolling a LUT/polynomial, or falling back to
서명을 추측하거나, LUT/다항식을 직접 롤링하거나,
`std::`.
`표준::`.
For anything the reference does not cover, look the headers up in the active
참조에서 다루지 않는 내용은 활성 페이지에서 헤더를 찾아보세요.
product environment; its
제품환경; 그
"If it's not here: discover from the headers"
"여기에 없으면 헤더에서 검색하세요."
section covers how to resolve that directory and which header answers which
섹션에서는 해당 디렉토리를 확인하는 방법과 어떤 헤더가 어떤 응답을 하는지 설명합니다.
question -- including the rule to scope every search to it, because a
질문 - 모든 검색 범위를 지정하는 규칙을 포함합니다.
`find`/`grep -r` from the env root stalls.
env 루트에서 `find`/`grep -r`이 정지됩니다.

- [ ] **Always set the core's saturation + rounding modes once at kernel entry** -- in every kernel, unconditionally.
- [ ] **항상 커널 시작 시 코어의 포화 + 반올림 모드를 한 번 설정합니다** -- 모든 커널에서 무조건.
- This is core-wide state (not per-op) that governs any **narrowing cast out of the accumulator**: `int32 >> shift -> int8/uint8` (requant) *and* `fp32 accum -> bf16`:
- 이는 **누산기의 축소 형변환**을 제어하는 코어 전체 상태(작업별이 아님)입니다: `int32 >> Shift -> int8/uint8`(reQuant) *및* `fp32 accum -> bf16`:

  ```cpp
  aie::tile::current().set_saturation(aie::saturation_mode::saturate);  // clamp, don't wrap
  aie::tile::current().set_rounding(aie::rounding_mode::conv_even);     // round-half-to-even; match your reference
  ```

These are persistent core state, so guard with a `static` flag and set before any `to_vector<...>(shift)` cast. **Strictly required for int8/quantized bit-exactness** (wrong modes = the classic "correct except +/-1 on some elements", or wrapped-overflow garbage); **best practice for bf16** too (they affect the `fp32->bf16` rounding), though tolerance-based bf16 checks often pass without it.
이는 지속적인 코어 상태이므로 `static` 플래그로 보호하고 `to_Vector<...>(shift)` 캐스트 전에 설정합니다. **int8/양자화된 비트 정확성에 엄격히 필요함**(잘못된 모드 = 고전적인 "일부 요소에서 +/-1을 제외하고 올바름" 또는 래핑된 오버플로 가비지) **bf16에 대한 모범 사례** 역시(`fp32->bf16` 반올림에 영향을 미침), 허용 오차 기반 bf16 검사는 종종 그것 없이 통과합니다.
Don't rely on defaults -- set them to match your reference.
기본값에 의존하지 말고 참조 내용과 일치하도록 설정하세요.
See `tutorial/conv2d/conv3x3_int8_2stamp` and its README "Set saturation + rounding mode for any narrowing cast".
`tutorial/conv2d/conv3x3_int8_2stamp` 및 해당 README "협소형 변환에 대한 채도 + 반올림 모드 설정"을 참조하세요.

- [ ] **For 4x4 distributed kernels** -- always use `aie::tile::current().id().row % 4` and `aie::tile::current().id().col % 4` to compute offsets into broadcast buffers.
- [ ] **4x4 분산 커널의 경우** -- 항상 `aie::tile::current().id().row % 4` 및 `aie::tile::current().id().col % 4`를 사용하여 오프셋을 브로드캐스트 버퍼로 계산합니다.
- Keep the `% 4` even in single-stamp kernels where it looks redundant: `id()` returns absolute physical coordinates, so dropping it silently breaks multi-stamp overlays (every stamp but stamp 0 indexes out of bounds and emits zeros).
- 중복되어 보이는 단일 스탬프 커널에서도 `% 4`를 유지합니다. `id()`는 절대 물리적 좌표를 반환하므로 이를 삭제하면 멀티 스탬프 오버레이가 자동으로 중단됩니다(스탬프 0을 제외한 모든 스탬프는 범위를 벗어난 인덱스를 생성하고 0을 방출합니다).

### 2e.
### Tiling Python (`custom_op_<name>/<name>_tiling.py`)
### 타일링 Python(`custom_op_<name>/<name>_tiling.py`)

**General instructions:**
**일반 지침:**

Follow the example of kernel tiling and tiling script creation from the `tiling.md` file.
`tiling.md` 파일에서 커널 타일링 및 타일링 스크립트 생성의 예를 따르십시오.

**Very important:** Report to the user the data flow relation that came out of your analysis.
**매우 중요:** 분석을 통해 나온 데이터 흐름 관계를 사용자에게 보고합니다.
It needs to be symbolic and contain information about how tiles of outputs relate to tiles of inputs.
이는 상징적이어야 하며 출력 타일이 입력 타일과 어떻게 관련되는지에 대한 정보를 포함해야 합니다.
For instance, it should look like this:
예를 들어 다음과 같아야 합니다.
```
 * Input tiles (buffer `ifm`) of size T + K
 * Kernel weights (buffer `wts`) of size K, need to stay unchanged across all kernel calls.
 * Output tiles (buffer `ofm`) of size T
 * Tiles are indexed from i = 0 to i = L_out / T - 1
 * For the tile indexed by i:
   * The tile of output produced is out[T * i] to out[T * (i+1) - 1],
   * One tile of input is needed, from in[T * i] to in[T * (i+1) + K - 1].
```

Use this symbolic relation to create the tiling script, and report to the user the data movement that you determined:
이 기호 관계를 사용하여 타일링 스크립트를 만들고 결정한 데이터 이동을 사용자에게 보고합니다.
* which core executes which tile (give a table with row/column layout),
* 어떤 코어가 어떤 타일을 실행하는지(행/열 레이아웃이 있는 테이블 제공)
* what does each L2 buffer contain,
* 각 L2 버퍼에는 무엇이 포함되어 있습니까?
* which data from the L2 buffer goes to each column, which goes to each row (if the WTS channel is used),
* L2 버퍼의 어떤 데이터가 각 열로 이동하는지, 각 행으로 이동하는지(WTS 채널이 사용되는 경우)
* what is the position of each result tile collected back in L2.
* L2에서 다시 수집된 각 결과 타일의 위치는 무엇입니까?
Then implement a tiling script using that information.
그런 다음 해당 정보를 사용하여 타일링 스크립트를 구현합니다.

**4x4 distributed template (default starting point):**
**4x4 분산 템플릿(기본 시작점):**

This is the canonical 16-core unary tiling, mirroring `negate/2_distributed`.
이는 'negate/2_distributed'를 미러링하는 표준 16코어 단항 타일링입니다.
IFM is broadcast across columns, OFM is collected per-core via `SpatialDistribute2D`.
IFM은 여러 열에 걸쳐 브로드캐스트되고 OFM은 `SpatialDistribute2D`를 통해 코어별로 수집됩니다.
Double-buffered (ping/pong) at both L2 and L1.
L2와 L1 모두에서 이중 버퍼링(핑/퐁)됩니다.

Sample tiling script:
샘플 타일링 스크립트:
```python
from pathlib import Path

import tensor_expr


def getTiling(
    opInterface: tensor_expr.OperatorInfo, tiling: tensor_expr.AieConfig
) -> None:
    OVERLAY_ROWS = 4
    OVERLAY_COLS = 4

    # One entry per overlay stamp/phase. This template configures stamp 0; see
    # the note below for spreading the work over every stamp/phase.
    spu = tiling[0][0]

    ifm, ofm = opInterface  # unary; for binary add `wts`
    assert ifm.getPaddedShape() == ofm.getPaddedShape(), (
        "ifm/ofm padded shape inconsistent"
    )

    # Per-core tile size (in elements). 0x800 = 2K bf16 elements = 4KB.
    # Sized so IFM ping+pong (broadcast across all 4 rows) and OFM
    # ping+pong fit in the 64KB L1, avoiding the 0xB000-0xD000 stack/heap.
    tileSize = 0x0800

    assert ifm.getPaddedShape()[0] % (tileSize * OVERLAY_COLS * OVERLAY_ROWS) == 0, (
        f"IFM shape {ifm.getPaddedShape()[0]} is not divisible by the tile size {tileSize} and overlay dimensions {OVERLAY_COLS}x{OVERLAY_ROWS}"
    )
    nb_temporal_iterations = ifm.getPaddedShape()[0] // (tileSize * OVERLAY_COLS * OVERLAY_ROWS)

    # L1 addresses: IFM needs OVERLAY_ROWS x tileSize per buffer
    # because each core in a column holds a full row of the broadcast.
    ifm_ping_addr = 0x0000  # to 0x3FFF (4 * tileSize = 16KB)
    ifm_pong_addr = 0x4000  # to 0x7FFF
    ofm_ping_addr = 0xD000  # to 0xDFFF (tileSize = 4KB)
    ofm_pong_addr = 0xE000  # to 0xEFFF

    # Total elements in the tensor
    size = 1
    for dim in ifm.getPaddedShape():
        size *= dim

    # L1 (core) buffers -- IFM holds the full column-broadcast slice
    ifm_mk = tensor_expr.TensorVar.make(
        [tensor_expr.Location(ifm_ping_addr), tensor_expr.Location(ifm_pong_addr)],
        [OVERLAY_ROWS, tileSize],
        type=ifm.getDType(),
    )
    ofm_mk = tensor_expr.TensorVar.make(
        [tensor_expr.Location(ofm_ping_addr), tensor_expr.Location(ofm_pong_addr)],
        [tileSize],
        type=ofm.getDType(),
    )

    # L2 (mem) buffers -- one full 4x4 tile across all cores, ping/pong
    L2_full_tile_size = tileSize * OVERLAY_ROWS * OVERLAY_COLS
    ifm_mem = tensor_expr.TensorVar.make(
        locations=[tensor_expr.Location(0, 0, 0), tensor_expr.Location(0, 0, 0x10000)],
        shape=[OVERLAY_COLS, OVERLAY_ROWS, tileSize],
        type=ifm.getDType(),
    )
    # Set the number of ping-pong buffer rotations of ifm_mem that will take place
    ifm_mem.setTemporalIterations(nb_temporal_iterations)
    ofm_mem = tensor_expr.TensorVar.make(
        locations=[tensor_expr.Location(1, 0, 0), tensor_expr.Location(1, 0, 0x10000)],
        shape=[OVERLAY_COLS, OVERLAY_ROWS, tileSize],
        type=ofm.getDType(),
    )
    # Set the number of ping-pong buffer rotations of ofm_mem that will take place
    ofm_mem.setTemporalIterations(nb_temporal_iterations)

    # DDR -> L2: stream the flattened tensor in L2-sized tiles
    ifm_l3_l2 = [ch for ch in spu.get_l3_to_l2_channels() if ch.mem_tile_port.tile_col == 0][0]
    spu.set_l3_to_l2_transfer(
        ifm_l3_l2,
        ifm.getTensorVar().Reshape([size]).TileBy(index=0, tileLen=L2_full_tile_size),
        ifm_mem.Reshape([L2_full_tile_size]),
    )

    # L2 -> L1 IFM: column-broadcast. Each column gets 1/4 of L2;
    # every core in that column receives the full quarter.
    ifm_l2_l1 = spu.get_l2_to_l1_channels(broadcast=tensor_expr.broadcast_on.COLUMNS)
    tiled_ifm_l2 = ifm_mem.TileTo(index=0, numTiles=OVERLAY_COLS)
    for col, ch in enumerate(ifm_l2_l1):
        spu.set_l2_to_l1_transfer(ch, tiled_ifm_l2[col], ifm_mk)

    # L1 -> L2 OFM: each core writes 1/16 of the tile, gathered via
    # SpatialDistribute2D into the [COLS, ROWS, tileSize] L2 buffer.
    ofm_l1_l2 = spu.get_l1_to_l2_channels()
    distributed_ofm_l2 = ofm_mem.Reshape(
        [OVERLAY_COLS, OVERLAY_ROWS, tileSize]
    ).SpatialDistribute2D(
        dimA=0, dimB=1, tileACount=OVERLAY_COLS, tileBCount=OVERLAY_ROWS
    )
    for col, ch in enumerate(ofm_l1_l2):
        spu.set_l1_to_l2_transfer(
            ch,
            ofm_mk,
            distributed_ofm_l2[col],
        )

    # L2 -> DDR
    ofm_l2_l3 = spu.get_l2_to_l3_channels()[0]
    spu.set_l2_to_l3_transfer(
        ofm_l2_l3,
        ofm_mem.Reshape([L2_full_tile_size]),
        ofm.getTensorVar().Reshape([size]).TileBy(index=0, tileLen=L2_full_tile_size),
    )

    # Each kernel invocation processes tileSize elements
    spu.set_kernel_params([tileSize])
    # Set the arguments to be the L1 buffers we passed (plus the lp_params)
    spu.set_kernel_arguments([ifm_mk, ofm_mk])
    # Mandatory: give the kernel's name and containing file
    spu.set_kernel_function_name("sample_op")
    spu.set_kernel_impl(Path("sample_op.cpp"))
    # Mandatory: give the number of kernel calls (even for 1 call)
    spu.set_kernel_nb_calls(nb_temporal_iterations)
```

**Using every stamp:** the template above fills stamp 0 only.
**모든 스탬프 사용:** 위의 템플릿은 스탬프 0만 채웁니다.
To use a multi-stamp overlay, loop `for stamp_idx, stamp in enumerate(tiling)` and give each stamp its **own slice of the DDR tensor** (the L1/L2 addresses stay identical per stamp) -- see `negate/4_distributed_vectorized_2stamp` and `mul/3_distributed_2stamp` for the canonical split.
멀티 스탬프 오버레이를 사용하려면 `for stamp_idx, stamp in enumerate(tiling)`을 반복하고 각 스탬프에 **고유한 DDR 텐서 조각**을 제공합니다(L1/L2 주소는 스탬프마다 동일하게 유지됩니다). 표준 분할에 대해서는 `negate/4_distributed_Vectorized_2stamp` 및 `mul/3_distributed_2stamp`를 참조하세요.

For binary ops: add a second operand, typically named `wts`, to the `opInterface` unpack; create a corresponding `wts_mem` L2 variable at `Location(3, 0, 0)` (typically row-broadcast via `wts_mem.TileTo(index=0, numTiles=OVERLAY_ROWS)`), and add `set_l3_to_l2_transfer` / `set_l2_to_l1_transfer` calls.
바이너리 연산의 경우: 일반적으로 `wts`라는 이름의 두 번째 피연산자를 `opInterface` 압축 해제에 추가합니다. `Location(3, 0, 0)`에 해당 `wts_mem` L2 변수를 생성하고(일반적으로 `wts_mem.TileTo(index=0, numTiles=OVERLAY_ROWS)`를 통해 행 브로드캐스트) `set_l3_to_l2_transfer` / `set_l2_to_l1_transfer` 호출을 추가합니다.
Use `spu.get_l2_to_l1_channels(broadcast=tensor_expr.broadcast_on.ROWS)` as channels to pass that input from L2 to L1.
해당 입력을 L2에서 L1로 전달하려면 `spu.get_l2_to_l1_channels(broadcast=tensor_expr.broadcast_on.ROWS)`를 채널로 사용하세요.
See `mul/2_distributed` for the canonical binary 4x4 pattern.
표준 바이너리 4x4 패턴은 `mul/2_distributed`를 참조하세요.

For details on 4x4-core distribution and additional patterns, see [README.md](../../README.md) for L2 shape `[COLS, ROWS, tileSize]` patterns.
4x4 코어 배포 및 추가 패턴에 대한 자세한 내용은 L2 형태 `[COLS, ROWS, TileSize]` 패턴에 대한 [README.md](../../README.md)를 참조하세요.

### 2f.
### Vitis AI Config (`vitisai_config.json`)
### Vitis AI 구성(`vitsai_config.json`)

```json
{
  "passes": [{
    "name": "vaiml_partition",
    "plugin": "vaip-pass_vaiml_partition",
    "vaiml_config": {
      "keep_outputs": true,
      "optimize_level": 2,
      "enable_f32_to_bf16_conversion": true,
      "logging_level": "info",
      "fe_args": "small-tensor-threshold-unwrapping=0",
      "experiment_features": ["KeepOrphanNodes"],
      "aiecompiler_args": "--compile-for-x86sim=true --x86sim-as-shared-library=true --msg-severity=\"77-23879:error:100\"",
      "custom_ops": {
        "mydomain.<op_name>": {
          "op_config": "custom_op_<name>/<name>.yaml"
        }
      }
    }
  }],
  "target": "VAIML",
  "targets": [{"name": "VAIML", "pass": ["vaiml_partition"]}]
}
```

`--compile-for-x86sim=true --x86sim-as-shared-library=true` build the x86
`--compile-for-x86sim=true --x86sim-as-shared-library=true` x86 빌드
simulator library (`libx86_sim.so`). **CRITICAL: keep them in every config you
시뮬레이터 라이브러리(`libx86_sim.so`). **중요: 모든 구성에 이를 유지하세요.
compile with, in every phase** -- Phase 3, each Phase 5 re-compile, and the
모든 단계**에서 컴파일 - 3단계, 각 5단계 재컴파일 및
Phase 7 integrated compile.
7단계 통합 컴파일.
`--msg-severity="77-23879:error:100"` promotes the L2 shared-buffer overlap
`--msg-severity="77-23879:error:100"`은 L2 공유 버퍼 겹침을 촉진합니다.
warning to an error, so hardware-only buffer-placement bugs fail the compile
오류에 대한 경고로 인해 하드웨어 전용 버퍼 배치 버그로 인해 컴파일이 실패합니다.
instead of silently corrupting results on the board.
보드의 결과를 조용히 손상시키는 대신.

Compile with this config as-is.
이 구성을 있는 그대로 컴파일합니다.
For Phase 4, copy it to
4단계의 경우 다음 위치에 복사하세요.
`vitisai_config_x86sim.json` and add one key:
`vitisai_config_x86sim.json`을 입력하고 키 하나를 추가하세요.

```json
      "runner_type": "x86sim",
```

That copy is only ever passed to `run.py`, never to `compile.py` -- one cache
해당 복사본은 `run.py`에만 전달되고 `compile.py`에는 전달되지 않습니다. -- 하나의 캐시
serves both runners.
두 주자를 모두 제공합니다.
See Critical Workflow Rule 4.
중요한 작업 흐름 규칙 4를 참조하세요.

### 2g.
### Tutorial Examples for Inspiration
### 영감을 위한 튜토리얼 예제

Complete working examples are bundled at `<skill_dir>/tutorial/`:
전체 작업 예제는 `<skill_dir>/tutorial/`에 번들로 제공됩니다.

| Tutorial<br>튜토리얼 | Description<br>설명 | Key pattern<br>주요 패턴 |
|----------|------------|-------------|
| `negate/0_untiled/`<br>`부정/0_untiled/` | Simplest unary op<br>가장 간단한 단항 연산 | 1x1, no tiling, whole tensor in L1<br>1x1, 타일링 없음, L1의 전체 텐서 |
| `negate/1_tiled/`<br>`부정/1_타일/` | Tiled unary op<br>타일식 단항 연산 | TileBy(), double buffering<br>TileBy(), 이중 버퍼링 |
| `negate/2_distributed/`<br>`부정/2_분산/` | 4x4 distributed unary<br>4x4 분산 단항 | 16-core tiling, broadcast, core ID offsets<br>16코어 타일링, 브로드캐스트, 코어 ID 오프셋 |
| `negate/3_distributed_and_vectorized/`<br>`부정/3_분산_및_벡터화/` | Optimized unary<br>최적화된 단항 | aie::vector, pipelining<br>aie::벡터, 파이프라이닝 |
| `mul/0_untiled/`<br>`mul/0_untiled/` | Simplest binary op<br>가장 간단한 바이너리 연산 | Two inputs (IFM + WTS)<br>입력 2개(IFM + WTS) |
| `mul/1_tiled/`<br>`멀/1_타일/` | Tiled binary op<br>타일식 바이너리 연산 | Separate L2 columns for IFM/WTS<br>IFM/WTS에 대한 별도의 L2 열 |
| `mul/2_distributed/`<br>`멀/2_분산/` | 4x4 distributed binary<br>4x4 분산 바이너리 | IFM column broadcast, WTS row broadcast<br>IFM 칼럼 방송, WTS 행 방송 |
| `mul/5_performant_overlay`<br>`mul/5_performant_overlay` | Performant overlay<br>고성능 오버레이 | Channel distribution for the performant overlay<br>고성능 오버레이를 위한 채널 배포 |
| `topk/0_untiled/`<br>`topk/0_untiled/` | Custom attributes<br>맞춤 속성 | Reads ONNX attributes (k)<br>ONNX 속성 읽기(k) |
| `topk/1_tiled/`<br>`topk/1_tiled/` | Async outputs<br>비동기 출력 | Asynchronous buffer management<br>비동기 버퍼 관리 |
| `reducemax/`<br>`최대 감소/` | Reduction over different axes<br>다른 축에 대한 축소 | Outer-axis vs inner-most-axis (de-interleave) strategies<br>외부 축과 가장 안쪽 축(디인터리브) 전략 |
| `gridsample2d/`<br>`그리드샘플2d/` | Complex 2D op<br>복잡한 2D 작업 | Multi-input, 4x4 distribution<br>다중 입력, 4x4 배포 |
| `affinegrid2d/`<br>`affinegrid2d/` | Affine grid generation<br>아핀 그리드 생성 | Vectorized and distributed variants<br>벡터화 및 분산 변형 |
| `gemm/`<br>`보석/` | GEMM + bias (== pointwise 1x1 conv)<br>GEMM + 편향(== 점별 1x1 전환) | `aie::mmul`, operand pre-blocking, per-row weight distribution, bias folded into weights, multi-memtile L2 staging, weight streaming<br>`aie::mmul`, 피연산자 사전 차단, 행별 가중치 분포, 가중치로 접힌 바이어스, 다중 멤타일 L2 스테이징, 가중치 스트리밍 |
| `conv2d/conv3x3_int8_2stamp/`<br>`conv2d/conv3x3_int8_2stamp/` | Spatial 3x3 stride-1 INT8 conv<br>공간 3x3 보폭-1 INT8 전환 | `aie::mmul` int8, halo rows, in-register sliding window (`shuffle_down_fill`), multi-call depth reduction (OFM async ratio), **persistent psum/scratch via an oversized OFM buffer (not stack/heap)**, int8 requant, native-faithful reproduction<br>`aie::mmul` int8, halo 행, 레지스터 내 슬라이딩 윈도우(`shuffle_down_fill`), 다중 호출 깊이 감소(OFM 비동기 비율), **대형 OFM 버퍼(스택/힙 아님)를 통한 지속적인 psum/스크래치**, int8 re퀀트, 기본에 충실한 재생 |

Read these files to understand patterns for your specific use case.
특정 사용 사례의 패턴을 이해하려면 다음 파일을 읽어보세요.
Each tutorial contains:
각 튜토리얼에는 다음이 포함됩니다.
- `.onnxtxt` / `.onnx` -- model definition
- `.onnxtxt` / `.onnx` -- 모델 정의
- `custom_op_*/` -- kernel .cpp, tiling .py, config .yaml
- `custom_op_*/` -- 커널 .cpp, 타일링 .py, 구성 .yaml
- `vitisai_config.json` -- compilation config
- `vitsai_config.json` -- 컴파일 구성
- `run.py` -- test script
- `run.py` -- 테스트 스크립트

## Phase 3: Compile
## 3단계: 컴파일

**IMPORTANT: Use `compile.py` to compile the model.**
**중요: 'compile.py'를 사용하여 모델을 컴파일하세요.**
**IMPORTANT: Run this in a sub-agent -- see "Critical Workflow Rules" above.**
**중요: 하위 에이전트에서 실행하세요. 위의 "중요한 작업 흐름 규칙"을 참조하세요.**

Commit your files to the local git repo before compiling:
컴파일하기 전에 파일을 로컬 git repo에 커밋합니다.
```bash
cd <working_dir> && git add -A && git commit -m "Ready to compile <op_name>"
```

```bash
bash -c '
<ENV_SETUP>
export DEBUG_VAIML_PARTITION=1
export FLEXML_PRINT_VITISTOOLS_OUTPUT=1
cd <working_dir>
python <skill_dir>/scripts/compile.py <model_b>.onnx
'
```

Successful compilation prints `Compilation successful`.
성공적인 컴파일은 `Compilation 성공적인`을 인쇄합니다.

**CRITICAL**: The compiler may report success even when tiling fails (VAIML falls back to CPU silently).
**중요**: 타일링이 실패하더라도 컴파일러는 성공을 보고할 수 있습니다(VAIML은 자동으로 CPU로 폴백됩니다).
Always verify:
항상 확인하세요:
- Check for `"Compilation successful"` in the output
- 출력에서 `"컴파일 성공"`을 확인하세요.
- Verify `<model_stem>/cache/vaiml_par_0/0/backend/gen_tg_adf/` is non-empty -- if empty, tiling failed
- `<model_stem>/cache/vaiml_par_0/0/backend/gen_tg_adf/`가 비어 있지 않은지 확인하세요. 비어 있으면 타일링이 실패합니다.

**IMPORTANT:** When compilation fails, do not guess at fixes.
**중요:** 컴파일이 실패하면 수정 사항을 추측하지 마세요.
Read the full error output, identify the root cause (tiling plugin, AIE compiler, or linker), and apply a targeted fix.
전체 오류 출력을 읽고 근본 원인(타일링 플러그인, AIE 컴파일러 또는 링커)을 식별하고 대상 수정 사항을 적용합니다.
See the **Troubleshooting** section for common compilation errors.
일반적인 컴파일 오류는 **문제 해결** 섹션을 참조하세요.

## Phase 4: Simulate and Validate
## 4단계: 시뮬레이션 및 검증

**IMPORTANT: Run x86sim in a sub-agent -- see "Critical Workflow Rules" above.**
**중요: 하위 에이전트에서 x86sim을 실행하세요. 위의 "중요한 작업 흐름 규칙"을 참조하세요.**

The validation flow uses the **same inputs** for both CPU reference and x86sim, then compares outputs with `compare_npy.py`.
유효성 검사 흐름은 CPU 참조와 x86sim 모두에 **동일한 입력**을 사용한 다음 출력을 'compare_npy.py'와 비교합니다.

### 4a.
### Run x86sim with the same inputs from Phase 1b
### 1b단계와 동일한 입력으로 x86sim 실행

The simulation is just an ONNX Runtime session driven by the x86sim config
시뮬레이션은 x86sim 구성에 의해 구동되는 ONNX 런타임 세션입니다.
(Critical Workflow Rule 4), so you use the ordinary `run.py` against the Phase 3
(중요한 작업 흐름 규칙 4) 따라서 3단계에서는 일반 `run.py`를 사용합니다.
cache -- no recompile -- and point `--input-dir` at the `reference_data/`
캐시 -- 재컴파일 안 함 -- `reference_data/`에서 `--input-dir`을 가리킵니다.
directory that `compile.py --cpu` wrote in Phase 1b, so both runs see identical
`compile.py --cpu`가 1b단계에서 작성한 디렉토리이므로 두 실행 모두 동일하게 표시됩니다.
data.
데이터.

```bash
bash -c '
<ENV_SETUP>
cd <working_dir>
python <skill_dir>/scripts/run.py <model_b>.onnx \
    --vitisai-config vitisai_config_x86sim.json \
    --input-dir <working_dir>/reference_data \
    --output-dir <working_dir>/x86sim_outputs \
    --num-runs 1
'
```

Outputs land in `x86sim_outputs/` as `.npy` files named after the output tensor,
출력은 출력 텐서의 이름을 딴 `.npy` 파일로 `x86sim_outputs/`에 저장됩니다.
in the same format as the CPU reference -- feed them straight to
CPU 참조와 동일한 형식으로 - CPU 참조에 바로 입력합니다.
`compare_npy.py`.
`비교_npy.py`.

**`--num-runs 1` is a correctness requirement, not a performance tip.** The
**`--num-runs 1`은 성능 팁이 아니라 정확성 요구 사항입니다.**
default of 10 gives you an all-zero `outputs.npz` while still reporting success
기본값 10은 성공을 보고하는 동시에 모두 0인 'outputs.npz'를 제공합니다.
under x86sim (Troubleshooting 22).
x86sim에서 (문제 해결 22).
Nothing to time here anyway; large tensors take
어쨌든 여기서는 시간을 낼 일이 없습니다. 큰 텐서는
minutes.
분.

### 4b.
### Compare x86sim output against CPU reference
### x86sim 출력을 CPU 참조와 비교

Use `compare_npy.py` to check for numeric mismatches between the CPU golden reference (Phase 1b) and the x86sim output:
`compare_npy.py`를 사용하여 CPU 골든 참조(1b단계)와 x86sim 출력 간의 숫자 불일치를 확인하세요.

```bash
python <skill_dir>/scripts/compare_npy.py \
    <working_dir>/reference_data/<output_name>.npy \
    <working_dir>/x86sim_outputs/<output_name>.npy \
    --rtol 0.01 --atol 0.015 --verbose
```

This prints:
이것은 다음을 인쇄합니다:
- Max absolute difference
- 최대 절대 차이
- Mean absolute difference
- 평균 절대차
- Number of elements exceeding tolerance
- 공차를 초과하는 요소 수
- `SUCCESS` or `ERROR` verdict
- `SUCCESS` 또는 `ERROR` 결과

If the comparison passes, the custom op is functionally correct -- proceed to Phase 5 (kernel optimization).
비교가 통과되면 사용자 정의 작업이 기능적으로 올바른 것입니다. 5단계(커널 최적화)로 진행합니다.

If the comparison **fails**, do not randomly change tiling or kernel code.
비교가 **실패**하는 경우 타일링이나 커널 코드를 무작위로 변경하지 마세요.
Instead, work through the **Debugging** subsection below: inspect the generated tiling header (step 1) and add printf to the kernel (step 2) to trace the root cause of the mismatch before applying any fix.
대신 아래 **디버깅** 하위 섹션을 진행하세요. 생성된 타일링 헤더를 검사하고(1단계) printf를 커널에 추가(2단계)하여 수정 사항을 적용하기 전에 불일치의 근본 원인을 추적하세요.

### Debugging (only if 4b fails)
### 디버깅(4b가 실패한 경우에만)

The following debug strategies are listed in terms of complexity.
다음 디버그 전략은 복잡성 측면에서 나열됩니다.
Inpecting the ADF header file is the cheapest thing to do while printf debugging requires a recompilation and might take a bit longer.
ADF 헤더 파일을 검사하는 것은 가장 저렴한 작업이지만 printf 디버깅에는 재컴파일이 필요하고 시간이 조금 더 걸릴 수 있습니다.

#### 1.
#### Inspect the generated tiling header
#### 생성된 타일링 헤더 검사

The compiler generates a tiling header that shows the actual ADF parameters derived from your Python tiler.
컴파일러는 Python 타일러에서 파생된 실제 ADF 매개변수를 표시하는 타일링 헤더를 생성합니다.
This is the first thing to check when outputs are wrong:
출력이 잘못되었을 때 가장 먼저 확인해야 할 사항은 다음과 같습니다.

```
<working_dir>/<model_stem>/cache/vaiml_par_0/custom_ops/<opname>_<instancename>_[0-9]*_multi_layer_overlay.h
```
There is one such header per instance of the custom op.
사용자 정의 작업의 인스턴스당 하나의 헤더가 있습니다.
You need to find the problematic header either as given by the AIE Compiler.
AIE 컴파일러에서 제공한 문제 헤더를 찾아야 합니다.

Read this file and verify:
이 파일을 읽고 다음을 확인하십시오.
- Tile sizes match what you expect
- 타일 크기가 예상한 것과 일치합니다.
- Buffer addresses and shapes are correct
- 버퍼 주소와 모양이 정확합니다.
- Number of kernel invocations matches your tiling strategy
- 커널 호출 수가 타일링 전략과 일치합니다.
- L2 and L1 buffer dimensions are consistent
- L2 및 L1 버퍼 크기는 일관됩니다.

See [README.md](../../README.md) "Debugging Tiling" section for details.
자세한 내용은 [README.md](../../README.md) "타일링 디버깅" 섹션을 참조하세요.

#### 2.
#### Printf debugging
#### Printf 디버깅

**Only add printf AFTER Phase 3 compilation succeeds.** Guard all printf calls with `#ifdef __X86SIM__` so they compile for x86sim but not for AIE.
**3단계 컴파일이 성공한 후에만 printf를 추가하세요.** `#ifdef __X86SIM__`으로 모든 printf 호출을 보호하여 x86sim에 대해서는 컴파일하지만 AIE에 대해서는 컴파일하지 않도록 하세요.
After adding printf, re-run `compile.py` and then `run.py` to see the debug output.
printf를 추가한 후 'compile.py'를 다시 실행한 다음 'run.py'를 다시 실행하여 디버그 출력을 확인하세요.

**Wrap all printf calls in `#ifdef __X86SIM__` guards** so they are only active during x86 simulation and won't break AIE compilation if left in accidentally:
**모든 printf 호출을 `#ifdef __X86SIM__` 가드로 래핑하여 x86 시뮬레이션 중에만 활성화되고 실수로 남아 있는 경우 AIE 컴파일을 중단하지 않도록 합니다.

**Debug iteration loop:**
**디버그 반복 루프:**
1. Edit the kernel .cpp to add printf (see examples below)
1. 커널 .cpp를 편집하여 printf를 추가합니다(아래 예 참조).
2. Re-run `compile.py` (normal config) to rebuild with the printf statements
2. printf 문으로 다시 빌드하려면 `compile.py`(일반 구성)를 다시 실행하세요.
3. Re-run `run.py` with the x86sim config and `--input-dir reference_data`
3. x86sim 구성 및 `--input-dir reference_data`를 사용하여 `run.py`를 다시 실행하세요.
4. Inspect the printf output to identify the problem
4. 문제를 식별하려면 printf 출력을 검사하세요.
5. Fix the kernel or tiling, re-compile, re-run x86sim, compare again
5. 커널 또는 타일링 수정, 재컴파일, x86sim 재실행, 다시 비교
6. When done, remove debug printf before finalizing
6. 완료되면 마무리하기 전에 디버그 printf를 제거하십시오.

**What to print to diagnose mismatches** (see [README.md](../../README.md) for full details):
**불일치 진단을 위해 인쇄할 내용**(자세한 내용은 [README.md](../../README.md) 참조):

```cpp
#ifdef __X86SIM__
#include <cstdio>
#endif

// 1. Verify kernel is called the expected number of times
#ifdef __X86SIM__
static uint32_t call_count = 0;
printf("Kernel call #%u, tile_size=%u\n", call_count, lp_params[0]);
call_count++;
#endif

// 2. Check input data arrived correctly (first/last elements)
#ifdef __X86SIM__
printf("  in[0]=%.4f in[%u]=%.4f\n", (float)in[0], size-1, (float)in[size-1]);
#endif

// 3. Check computed output values
#ifdef __X86SIM__
if (i < 4) {
    printf("  [%u] in=%.4f out=%.4f\n", i, (float)in[i], (float)out[i]);
}
#endif

// 4. For 4x4: verify core position and data offsets
#ifdef __X86SIM__
int row = aie::tile::current().id().row % 4;
int col = aie::tile::current().id().col % 4;
printf("Core (%d,%d): offset=%u\n", row, col, offset);
#endif
```

**Common causes of numeric mismatches:**
**숫자 불일치의 일반적인 원인:**
- **Wrong tile size in lp_params**: kernel processes more/fewer elements than the tile contains
- **lp_params의 잘못된 타일 크기**: 커널은 타일에 포함된 것보다 더 많거나 적은 요소를 처리합니다.
- **Wrong buffer offset in 4x4**: core reads from wrong position in broadcast buffer
- **4x4의 잘못된 버퍼 오프셋**: 코어가 브로드캐스트 버퍼의 잘못된 위치에서 읽습니다.
- **Wrong data movement**: L2->L1 transfer shape doesn't match kernel expectation
- **잘못된 데이터 이동**: L2->L1 전송 형태가 커널 예상과 일치하지 않습니다.
- **Padding mismatch**: compiler pads tensors (use `getPaddedShape()` in tiling)
- **패딩 불일치**: 컴파일러가 텐서를 패드합니다(타일링에서 `getPendedShape()` 사용).
- **bf16 precision**: expected when comparing bf16 kernel output against float32 CPU reference (rtol=0.01, atol=0.015 should accommodate this)
- **bf16 정밀도**: bf16 커널 출력을 float32 CPU 참조와 비교할 때 예상됩니다(rtol=0.01, atol=0.015는 이를 수용해야 함).

**Tip:** Using `#ifdef __X86SIM__` guards means printf will only be active during x86 simulation and won't break AIE compilation.
**팁:** `#ifdef __X86SIM__` 가드를 사용하면 printf가 x86 시뮬레이션 중에만 활성화되고 AIE 컴파일이 중단되지 않는다는 의미입니다.
However, clean up debug printf before finalizing the kernel.
그러나 커널을 마무리하기 전에 디버그 printf를 정리하십시오.

#### 3.
#### Manually inject ADF code
#### ADF 코드 수동 삽입

**WARNING:** Only use this if all possibilities to adapt the python tiling code are exhausted for the `AieConfig` tiling API.
**경고:** `AieConfig` 타일링 API에 대해 Python 타일링 코드를 적용할 수 있는 모든 가능성이 소진된 경우에만 이 방법을 사용하십시오.
Manually injecting ADF is decouraged because it creates (verbose) code that is hard to maintain or adapt to other AIE topologies.
ADF를 수동으로 삽입하는 것은 다른 AIE 토폴로지에 적용하거나 유지 관리하기 어려운 (자세한) 코드를 생성하므로 권장되지 않습니다.
Howerver, if you hit a limit on what the tiling `AieConfig` tiling API can represent, and before resorting to other legacy APIs, this is the path you must take.
그러나 타일링 `AieConfig` 타일링 API가 나타낼 수 있는 내용에 대한 제한에 도달하고 다른 레거시 API를 사용하기 전에 이 경로를 따라야 합니다.

The ADF header is a header file where the data movement and scheduling of AIE kernels is done.
ADF 헤더는 AIE 커널의 데이터 이동 및 스케줄링이 수행되는 헤더 파일입니다.
This is the file used to integrate your operator with the overall AIE graph.
이는 연산자를 전체 AIE 그래프와 통합하는 데 사용되는 파일입니다.
When you provide a python tiling script, this file is generated for you - togehter with a kernel.yaml file - and contains the data movement instructions the tiling script requested.
Python 타일링 스크립트를 제공하면 이 파일이 kernel.yaml 파일과 함께 생성되며 타일링 스크립트가 요청한 데이터 이동 지침이 포함되어 있습니다.

You can provide your own manually-written ADF header.
직접 작성한 ADF 헤더를 제공할 수 있습니다.
To do so, its name needs to be supplied in the custom op config's optional `ml_adf_header` field, alongside the kernel YAML directory:
그렇게 하려면 커널 YAML 디렉터리와 함께 사용자 정의 op 구성의 선택적 `ml_adf_header` 필드에 해당 이름을 제공해야 합니다.

```yaml
ml_adf_header: "<mydomain>_<mycustomop>_<output_name>_<instance_nb>_multi_layer_overlay.h"
kernel_yaml_dir: "<dir_to>/<mydomain>.<mycustomop>_<instance_nb>.yaml>"
```

> Note: We provide the directory containing the kernel YAML and not the filename itself.
> 참고: 파일 이름 자체가 아닌 커널 YAML이 포함된 디렉터리를 제공합니다.
> The reason is that the filename is automatically constructed after the name of the operation and instance number.
> 그 이유는 작업 이름과 인스턴스 번호 뒤에 파일 이름이 자동으로 구성되기 때문입니다.

Providing ADF header yourself is possible if you need to modify the automatically-generated ADF header.
자동으로 생성된 ADF 헤더를 수정해야 하는 경우 ADF 헤더를 직접 제공할 수 있습니다.
Doing so stops the auto-generation of ADF headers and kernel YAML for the given custom op.
이렇게 하면 지정된 사용자 지정 작업에 대한 ADF 헤더 및 커널 YAML의 자동 생성이 중지됩니다.
In this case, you can start from the automatically generated header.
이 경우 자동으로 생성된 헤더에서 시작할 수 있습니다.
Assuming that `mydomain.mycustomop` is the name of the custom op in ONNX, you can find the generated ADF header under `custom_ops` inside your results/artifact folder, with the name `mydomain_mycustomop_0_multi_layer_overlay.h`.
`mydomain.mycustomop`이 ONNX의 사용자 정의 작업 이름이라고 가정하면 결과/아티팩트 폴더 내 `custom_ops`에서 `mydomain_mycustomop_0_multi_layer_overlay.h`라는 이름으로 생성된 ADF 헤더를 찾을 수 있습니다.
The generated kernel YAML is located next to it in the same directory.
생성된 커널 YAML은 동일한 디렉터리의 옆에 있습니다.

Before modifying the custom op registration YAML with the new two files, you must perform the following adaptations:
새로운 두 파일로 사용자 정의 작업 등록 YAML을 수정하기 전에 다음 조정을 수행해야 합니다.
  - Rename `mydomain.mycustomop_0.yaml` -> `mydomain.mycustomop.yaml` as to drop the instance number.
  - 인스턴스 번호를 삭제하려면 `mydomain.mycustomop_0.yaml` -> `mydomain.mycustomop.yaml`의 이름을 바꾸세요.
  - Edit the "stamp" entry to have a `defined_value` of 1.
  - '정의된_값'이 1이 되도록 '스탬프' 항목을 편집합니다.
  - We always want a stamp value of 1 even if we implement more than one stamp.
  - 두 개 이상의 스탬프를 구현하더라도 우리는 항상 스탬프 값 1을 원합니다.
  - This value doesn't really set the number of stamps being used, it is just an indicator to the backend that stamp is implemented by the custom op itself.
  - 이 값은 실제로 사용되는 스탬프 수를 설정하지 않으며, 스탬프가 사용자 정의 작업 자체에 의해 구현된다는 것을 백엔드에 표시하는 것일 뿐입니다.
    ```yaml
      - name: stamp
        data_type: uint32
    +   defined_value: 1
    ```
  - Edit the number of channels used by the DDR tensors.
  - DDR 텐서가 사용하는 채널 수를 편집합니다.
  - Those are named `num_ifm_shim_ch`, `num_wts_shim_ch`, `num_ifm<N>_shim_ch`, `num_scratch_0_in_shim_ch`, `num_scratch_0_out_shim_ch` and `num_ofm_shim_ch`.
  - 그것들의 이름은 `num_ifm_shim_ch`, `num_wts_shim_ch`, `num_ifm<N>_shim_ch`, `num_scratch_0_in_shim_ch`, `num_scratch_0_out_shim_ch` 및 `num_ofm_shim_ch`입니다.
  - Ideally you have MLIR files that result from the previous compilation from where you can deduce the numbers.
  - 이상적으로는 숫자를 추론할 수 있는 이전 컴파일의 결과인 MLIR 파일이 있습니다.
    ```yaml
      - name: num_ifm_shim_ch
        data_type: uint32
    +   defined_value: 2
    ```

## Phase 5: Kernel Optimization
## 5단계: 커널 최적화

**Phase 5 is MANDATORY.** Passing Phase 4 with a scalar fallback kernel
**5단계는 필수입니다.** 스칼라 대체 커널을 사용하여 4단계 통과
is NOT a valid stopping point -- the kernel must be optimized before
유효한 중지 지점이 아닙니다. 먼저 커널을 최적화해야 합니다.
proceeding to Phase 6.
6단계로 진행됩니다.
A correct-but-unoptimized kernel is an
정확하지만 최적화되지 않은 커널은
incomplete custom op.
불완전한 맞춤 작전.

**Mandatory exit criteria** (all must hold before reporting Phase 5
**필수 종료 기준**(5단계 보고 전에 모두 보유해야 함)
complete or proceeding to Phase 6):
완료 또는 6단계로 진행 중):

- BOTH `/vai-aie-kernel-development` AND `/vai-aie-compiler-oriented-optimizations`
- `/vai-aie-kernel-development` 및 `/vai-aie-compiler-지향-최적화` 둘 다
were invoked.
호출되었습니다.
The two are complementary:
두 가지는 상호보완적입니다.
  - `vai-aie-kernel-development` covers architecture-level patterns
  - 'vai-aie-kernel-development'는 아키텍처 수준 패턴을 다룹니다.
(vectorization, loop splitting for register pressure, in-place
(벡터화, 레지스터 압력을 위한 루프 분할, 내부
updates, VLIW slot awareness, NOINLINE core pattern).
업데이트, VLIW 슬롯 인식, NOINLINE 코어 패턴).
  - `vai-aie-compiler-oriented-optimizations` covers Peano-compiler-specific
  - 'vai-aie-compiler-지향-최적화'는 Peano 컴파일러 관련 내용을 다룹니다.
pragmas and hints (restrict pointers, DM bank annotations, loop
pragma 및 힌트(포인터 제한, DM 뱅크 주석, 루프
pragmas / hints, software pipelining pre-RA vs post-RA, loop
pragma/힌트, RA 이전과 RA 이후의 소프트웨어 파이프라이닝, 루프
versioning, vector alignment, sub-32-bit handling, optimization
버전 관리, 벡터 정렬, 32비트 이하 처리, 최적화
remarks).
비고).
Skipping either skill is not allowed -- many wins (e.g. unblocking
두 스킬 중 하나를 건너뛰는 것은 허용되지 않습니다. 많은 승리(예: 차단 해제)
software pipelining, fixing alignment-driven scalarization) live only
소프트웨어 파이프라이닝, 정렬 기반 스케일링 수정) 라이브 전용
in the compiler-oriented skill.
컴파일러 지향 기술에서.
- **Vectorization was attempted.** A scalar `for` loop is acceptable as
- **벡터화가 시도되었습니다.** 스칼라 'for' 루프는 다음과 같이 허용됩니다.
the *final* kernel ONLY if vectorization was tried, broke
*최종* 커널은 벡터화가 시도된 경우에만 중단되었습니다.
correctness or was structurally infeasible for this op, and the
이 작업에서는 정확성이 있거나 구조적으로 실행 불가능했습니다.
reason is documented in the report (or `optimization_verdict` /
이유는 보고서(또는 `optimization_verdict`/
`peer_insights` when running under `vai-custom-op`).
`vai-custom-op`에서 실행 중인 경우 `peer_insights`).
- All other applicable techniques from both skills were considered and
- 두 기술의 다른 모든 적용 가능한 기술이 고려되었으며
applied where they fit.
적합한 곳에 적용됩니다.
- Phase 4 (compile + x86sim + `compare_npy.py`) was re-run after each
- 4단계(컴파일 + x86sim + `compare_npy.py`)가 각 단계 이후에 다시 실행되었습니다.
optimization change and still passes.
최적화가 변경되었으나 여전히 통과합니다.
- **Static metrics were inspected after every compile** -- the `.lst` schedule,
- **매 컴파일 후에 정적 측정항목을 검사했습니다** -- `.lst` 일정,
see step 5a -- and each optimization was chosen from what they showed.
5a단계를 참조하세요. 각 최적화는 표시된 내용 중에서 선택되었습니다.
This
이
applies with or without a board; there is no exemption.
보드 유무에 관계없이 적용됩니다. 면제는 없습니다.
- **Board-based measurement was performed before integration whenever
- **보드 기반 측정은 통합 전에 언제든지 수행되었습니다.
a board is available.** See step 5b below.
보드를 사용할 수 있습니다.** 아래 5b단계를 참조하십시오.
Board measurement is part
보드 측정은 일부입니다
of Phase 5, not deferred to Phase 7 -- the kernel's per-op cost,
7단계로 연기되지 않은 5단계 - 커널의 연산당 비용,
bottleneck class (compute vs DMA vs sync), and the impact of each
병목 현상 클래스(컴퓨팅 vs DMA vs 동기화) 및 각각의 영향
optimization step must be known on real hardware before declaring
선언하기 전에 실제 하드웨어에서 최적화 단계를 알아야 합니다.
Phase 5 complete.
5단계가 완료되었습니다.
Skipping board measurement is allowed ONLY when
보드 측정 건너뛰기는 다음 경우에만 허용됩니다.
no board is reachable, and the absence must be documented.
이사회에 연락할 수 없으며 부재 사실을 문서로 기록해야 합니다.
NPU compute
NPU 컴퓨팅
time is measured via `/vai-perf-analysis`.
시간은 `/vai-perf-analytic`을 통해 측정됩니다.
- Each successful optimization step is committed to the local git repo.
- 성공적인 각 최적화 단계는 로컬 git 저장소에 커밋됩니다.

### Optimization is data-driven -- ALWAYS, board or no board
### 최적화는 데이터 기반입니다. 항상, 보드가 있거나 없거나

The performance is very important. **MANDATORY:** After **each compilation**, inspect the
성능은 매우 중요합니다. **필수:** **각 컴파일** 후에
Peano-generated `.lst` (disassembly listing) files to see how the kernel can be
커널이 어떻게 작동하는지 확인하기 위한 Peano 생성 `.lst`(디스어셈블리 목록) 파일
improved further.
더욱 개선되었습니다.

**Never** change a kernel speculatively: measure, optimize the bottleneck the data
**절대로** 추측에 따라 커널을 변경하지 마세요. 데이터 병목 현상을 측정하고 최적화하세요.
points to, re-measure to confirm the change actually helped, then repeat.
가리키고 재측정하여 실제로 도움이 되었는지 확인한 다음 반복합니다.
Step 5
5단계
requires **both** kinds of metric - static and dynamic - whenever both are obtainable:
두 가지 측정항목을 모두 얻을 수 있을 때마다 **두 가지** 종류의 측정항목(정적 및 동적)이 필요합니다.

If no board is reachable, the static half of step 5 is still mandatory on every
연결할 수 있는 보드가 없는 경우 모든 보드에서 5단계의 정적 절반은 여전히 필수입니다.
compile and the exit criteria above still apply; you simply cannot rank
컴파일하고 위의 종료 기준이 여전히 적용됩니다. 당신은 단순히 순위를 매길 수 없습니다
optimizations by measured impact and must lean on the best-practice lists from
측정된 영향을 통해 최적화하고 다음의 모범 사례 목록을 활용해야 합니다.
the two skills above.
위의 두 가지 스킬입니다.

1. Load the **vai-aie-kernel-development** skill (invoke `/vai-aie-kernel-development`)
1. **vai-aie-kernel-development** 스킬 로드(`/vai-aie-kernel-development` 호출)
AND the **vai-aie-compiler-oriented-optimizations** skill
그리고 **vai-aie-컴파일러 중심 최적화** 기술
(invoke `/vai-aie-compiler-oriented-optimizations`).
(`/vai-aie-compiler- Oriented-optimizations`를 호출하세요).
2. Apply the optimization techniques from both skills to the kernel:
2. 두 기술의 최적화 기술을 커널에 적용합니다.
restrict pointers, DM bank annotations, NOINLINE core pattern,
포인터 제한, DM 뱅크 주석, NOINLINE 코어 패턴,
vectorization, loop pipelining pragmas, software pipelining hints,
벡터화, 루프 파이프라이닝 프라그마, 소프트웨어 파이프라이닝 힌트,
loop versioning, vector alignment, etc.
루프 버전 관리, 벡터 정렬 등
3. After each optimization change, loop back to **Phase 4** (simulate
3. 각 최적화 변경 후에는 **단계 4**로 되돌아갑니다(시뮬레이션
and validate) to confirm the kernel is still functionally correct.
및 유효성 검사)을 통해 커널이 여전히 기능적으로 올바른지 확인합니다.
4. Commit each successful optimization step to the local git repo.
4. 성공적인 각 최적화 단계를 로컬 git 저장소에 커밋합니다.
5. **Measure -- static metrics always, dynamic metrics whenever a board is
5. **측정 - 항상 정적 메트릭, 보드가 있을 때마다 동적 메트릭
available.
가능합니다.
MANDATORY.**
필수.**

**5a.
Static metrics (every compile, no board required).**
정적 지표(모든 컴파일, 보드 필요 없음).**
**Finding the `.lst`:** don't guess the core coordinate -- glob the AIE build
**`.lst` 찾기:** 핵심 좌표를 추측하지 말고 AIE 빌드를 glob하세요.
tree and use whatever kernel listing actually exists.
트리를 만들고 실제로 존재하는 커널 목록을 사용하세요.
Make a strong effort to
힘써 노력하라
locate it; if you can't find one, fail loudly and re-check rather than silently
그것을 찾으십시오; 찾을 수 없다면 조용히 실패하기보다는 큰 소리로 실패하고 다시 확인하세요.
skipping the analysis.
분석을 생략합니다.
**What to extract from the `.lst`:** the emitted VLIW schedule
**`.lst`에서 추출할 내용:** 내보낸 VLIW 일정
(software-pipelining status and II, vector vs. scalar slot usage, spills, and
(소프트웨어 파이프라인 상태 및 II, 벡터 대 스칼라 슬롯 사용량, 유출 및
stalls).
노점).
Feed those findings back into the optimization loop to target the
이러한 결과를 최적화 루프에 다시 공급하여
specific bottleneck the disassembly reveals.
분해를 통해 특정 병목 현상이 드러납니다.
See
참조
`/vai-aie-compiler-oriented-optimizations` for how to read and act on these
이를 읽고 조치하는 방법에 대한 `/vai-aie-compiler- Oriented-optimizations`
listings and the related optimization remarks.
목록 및 관련 최적화 설명.

**5b.
Dynamic metrics (board).** Use the `run_on_board.py` script from
동적 지표(보드).** 다음의 `run_on_board.py` 스크립트를 사용하세요.
`<skill_dir>/scripts/`, or another user-provided mechanism to validate on
`<skill_dir>/scripts/` 또는 유효성을 검사할 다른 사용자 제공 메커니즘
the board.
보드.
To do so:
그렇게 하려면:
   - Reuse the Phase 3 cache directly, with no recompile.
   - 재컴파일 없이 3단계 캐시를 직접 재사용합니다.
   - Just pass the normal
   - 그냥 평범하게 넘기세요
`vitisai_config.json`, not the x86sim copy (Critical Workflow Rule 4).
x86sim 복사본이 아닌 `vitisai_config.json`(중요 작업 흐름 규칙 4).
   - Run on the board, **reusing the same `inputs.npz` from Phase 1b** so the
   - 보드에서 실행하고 **1b단계의 동일한 `inputs.npz`를 재사용**하여
board outputs can be compared against the CPU golden reference.
보드 출력은 CPU 골든 레퍼런스와 비교할 수 있습니다.
You usually only need to pass the working directory as `-p` argument and
일반적으로 작업 디렉터리를 `-p` 인수로 전달하고
the target board host name / IP address; you may also be provided with
타겟 보드 호스트 이름/IP 주소; 당신은 또한 제공받을 수 있습니다
an username + password / SSH key to use, which you need to use in that case.
사용할 사용자 이름 + 비밀번호 / SSH 키(이 경우 사용해야 함)
The arguments after `--` are forwarded to `run.py` (check the `run.py` syntax):
`--` 뒤의 인수는 `run.py`로 전달됩니다(`run.py` 구문 확인).
     ```bash
     bash -c '
     <ENV_SETUP>
     cd <working_dir>
     python <skill_dir>/scripts/run_on_board.py \
         -p <working_dir> \
         --boardhost <BOARDHOST> \
         [--board-user <BOARD_USER>] \
         [--board-password <PASSWORD>] \
         [--board-key <SSH_KEY>] \
         -- <model_b>.onnx \
            --input-dir <working_dir>/reference_data \
            --output-dir <working_dir>/board_outputs
     '
     ```
Add `--dry-run` first to preview the command before executing.
실행하기 전에 명령을 미리 보려면 `--dry-run`을 먼저 추가하세요.
   - **Verify numeric accuracy on the board.** Hardware execution can diverge
   - **보드의 수치 정확성을 확인하세요.** 하드웨어 실행이 다를 수 있습니다.
from x86sim (different rounding, real DMA behavior, race conditions), so
x86sim에서 (다른 반올림, 실제 DMA 동작, 경쟁 조건)
re-run the comparison from Phase 4b against the board outputs:
보드 출력에 대해 4b단계의 비교를 다시 실행합니다.
     ```bash
     python <skill_dir>/scripts/compare_npy.py \
         <working_dir>/reference_data/<output_name>.npy \
         <working_dir>/board_outputs/<output_name>.npy \
         --rtol 0.01 --atol 0.015 --verbose
     ```
If x86sim passed but the board fails, the bug is hardware-specific (DMA
x86sim은 통과했지만 보드가 실패하면 버그는 하드웨어에 특정한 것입니다(DMA
alignment, bank conflicts, sync issues) -- do not proceed with performance
정렬, 은행 충돌, 동기화 문제) -- 성능을 진행하지 마세요.
optimization until correctness on the board is restored.
보드의 정확성이 복원될 때까지 최적화합니다.
   - **MANDATORY** Then invoke the **vai-perf-analysis** skill (`/vai-perf-analysis`) on the run
   - **필수** 그런 다음 실행 시 **vai-perf-analytic** 기술(`/vai-perf-analytic`)을 호출합니다.
artifacts to get a per-op timing breakdown.
작업별 타이밍 분석을 얻기 위한 아티팩트입니다.
The canonical entry point
정식 진입점
is `vai-perf-analysis/scripts/ai_extract.py` -- it reports the actual NPU
'vai-perf-analytic/scripts/ai_extract.py'입니다. 실제 NPU를 보고합니다.
compute time per op.
작업당 시간을 계산합니다.
Do NOT measure custom-op performance with Python
Python으로 맞춤 작업 성능을 측정하지 마세요.
`time.time()` wrappers around the inference call: wall-clock
추론 호출을 둘러싼 `time.time()` 래퍼: 벽시계
measurements include host-side overhead (data movement, ORT dispatch,
측정에는 호스트 측 오버헤드(데이터 이동, ORT 디스패치,
sync waits) that can be **10-20x larger** than the NPU compute time
동기화 대기) NPU 컴퓨팅 시간보다 **10~20배** 더 클 수 있음
itself.
그 자체.
When the user asks "how fast is this op", reach for
사용자가 "이 작업의 속도는 얼마나 되나요?"라고 묻는 경우
`ai_extract.py` first; only fall back to wall-clock if nothing else is
먼저 `ai_extract.py`; 다른 것이 없는 경우에만 벽시계로 돌아갑니다.
available, and label the result as wall-clock so it is not confused
사용 가능하며 결과를 벽시계로 표시하여 혼동하지 않도록 합니다.
with NPU compute time.
NPU 컴퓨팅 시간으로.
When the question is specifically "how long did *this* custom op
질문이 구체적으로 "*이* 맞춤 작업을 수행한 기간은 얼마나 됩니까?"인 경우
take", pass `--query <op_name>` (the op's registered name) to
take", `--query <op_name>`(op의 등록된 이름)을 전달합니다.
`ai_extract.py`.
`ai_extract.py`.
The compact overview and `--top N` views can both
간략한 개요와 `--top N` 보기는 둘 다 가능합니다.
miss or under-report a single op, so the per-op number must come from
단일 작업을 놓치거나 과소보고하므로 작업당 숫자는 다음에서 와야 합니다.
`--query`.
`--쿼리`.
Use the report to answer: how much NPU compute time is the custom op
보고서를 사용하여 답변: 사용자 정의 작업의 NPU 컴퓨팅 시간은 얼마나 됩니까?
actually taking?
실제로 복용?
Where in the kernel (compute vs.
커널의 위치(컴퓨팅 대
DMA vs. sync) is
DMA 대 동기화)는
that time being spent?
그 시간을 보내고 있니?
Which optimization from
어느 최적화에서
`vai-aie-kernel-development` targets that bottleneck?
'vai-aie-kernel-development'가 병목 현상을 해결한다고요?
**The loop is MANDATORY, not a single pass.** Use what the metrics tell you to
**루프는 단일 패스가 아닌 필수입니다.** 측정항목에서 알려주는 내용을 사용하세요.
choose the next optimization from step 2, then repeat steps 3-5 -- re-measuring
2단계에서 다음 최적화를 선택한 후 3~5단계를 반복하여 다시 측정
the static metrics on every compile and the dynamic ones whenever a board is
모든 컴파일의 정적 메트릭과 보드가 있을 때마다 동적 메트릭
available -- until the data shows the kernel is no longer a bottleneck or
사용 가능 - 데이터에 커널이 더 이상 병목 현상이 없거나
further optimization shows diminishing returns.
추가 최적화는 수익 감소를 보여줍니다.
One optimization pass, or a
하나의 최적화 단계 또는
pass whose effect you never re-measured, does not complete Phase 5.
효과를 다시 측정하지 않은 패스는 5단계를 완료하지 않습니다.

If no board is available, keep running the loop on static metrics alone and
사용 가능한 보드가 없으면 정적 측정항목에서만 루프를 계속 실행하고
apply the best-practice optimizations from `vai-aie-kernel-development`, but flag
`vai-aie-kernel-development`의 모범 사례 최적화를 적용하지만 플래그를 지정합니다.
clearly to the user that the gains are unverified and may regress
이익이 확인되지 않았으며 회귀할 수 있음을 사용자에게 분명히 알립니다.
real-hardware performance.
실제 하드웨어 성능.

Only proceed to Phase 6 once the kernel is both correct and optimized.
커널이 정확하고 최적화된 후에만 6단계로 진행하세요.

Self-audit: Before declaring a kernel's tiling final and optimize, verify
자체 감사: 커널 타일링 최종 및 최적화를 선언하기 전에
that all the following distribution and optimzation axes are exercised.
다음의 모든 분포 및 최적화 축이 실행됩니다.

```
Kernel code:
- [ ] Did you make aggressive use of vectorization?
- [ ] Did you rely on the AIE API instead of doing ad-hoc implementations?
- [ ] Did you use the best practices from sibling skills on kernel programming?
- [ ] Are `stack_size`/`heap_size` left at defaults? (They must be -- if the kernel overflows, apply the four checks in "Stack and Heap Sizing" instead of raising the budget.)

Tiling:
- [ ] Did you design tiling from first principles starting from the output and considering the algorithm?
- [ ] After each L3<->L2 Transpose, is the innermost (DMA burst) axis still large? Don't push a small/sub-word axis innermost -- reorder in-kernel instead.
- [ ] Did you make use of Transpose/Repeat to minimize L3<->L2 Data transfers?
- [ ] Does the tiling instantiate kernels on all columns of the overlay?
- [ ] Does the tiling instantiate kernels on all stamps of the overlay (honoring `tp_size`; see Development Methodology)?
- [ ] If you decided for a 1x1 tiling implementation, is it due to overlay or algorithm constraints?
```



## Phase 6: Model Re-integration
## 6단계: 모델 재통합

After validating the custom op, create an integration directory and stitch the custom op back into the full model.
사용자 정의 작업을 검증한 후 통합 디렉터리를 생성하고 사용자 정의 작업을 전체 모델에 다시 연결합니다.

### 6a.
### Create the integration directory
### 통합 디렉터리 생성

Create a new directory next to the custom op working directory to hold the integrated model and its config.
통합 모델과 해당 구성을 보관할 사용자 정의 작업 디렉터리 옆에 새 디렉터리를 만듭니다.
`<integration_dir>` defaults to `integration/`:
`<integration_dir>`의 기본값은 `integration/`입니다.

```bash
mkdir -p <integration_dir>
```

The `<integration_dir>` should only contain one `record_timer_ts.json` - `record_timer_ts.json` in sub-directories are also forbidden which means that one folder contains only one board run - from executing the final model on the board.
`<integration_dir>`에는 하나의 `record_timer_ts.json`만 포함되어야 합니다. 하위 디렉터리의 `record_timer_ts.json`도 금지됩니다. 즉, 하나의 폴더에 하나의 보드 실행만 포함되어 보드에서 최종 모델을 실행하는 것이 금지됩니다.
`<integration_dir>` is called `integration` by default and stays right under `<working_dir>`.
`<integration_dir>`은 기본적으로 `integration`이라고 불리며 `<working_dir>` 바로 아래에 있습니다.
The cache directory with the compilation artifacts must also stay in `integration`: this is inteded to be an hermetic directory from which we can analyze performance data with AI Analyzer.
컴파일 아티팩트가 있는 캐시 디렉터리도 '통합'에 있어야 합니다. 이는 AI 분석기로 성능 데이터를 분석할 수 있는 밀폐형 디렉터리여야 합니다.

### 6b.
### Stitch the custom op into the full model
### 맞춤형 작업을 전체 모델에 결합

```bash
<ENV_SETUP>
python <skill_dir>/scripts/onnx_stitch.py \
    --model <full_model>.onnx \
    --nodes <node_name_or_op_type_1> [<node_name_or_op_type_2> ...] \
    --custom-op-name <op_name> \
    --domain mydomain \
    --output <integration_dir>/integrated_model.onnx
```

This replaces the original subgraph with a `mydomain.<op_name>` custom op node, preserving:
이렇게 하면 원래 하위 그래프가 `mydomain.<op_name>` 사용자 정의 작업 노드로 바뀌고 다음이 유지됩니다.
- All other nodes in the graph
- 그래프의 다른 모든 노드
- Initializers used by the custom op (as additional inputs)
- 사용자 정의 작업에서 사용되는 초기화 프로그램(추가 입력으로)
- Graph inputs and outputs
- 그래프 입력 및 출력
- Opset declarations (adds the custom domain)
- Opset 선언(사용자 정의 도메인 추가)

### 6c.
### Set up vitisai_config.json
### vitisai_config.json 설정

Copy the original vitisai_config.json to the integration directory and add a `custom_ops` entry pointing to the custom op implementation from `<working_dir>`:
원본 vitisai_config.json을 통합 디렉터리에 복사하고 `<working_dir>`에서 사용자 정의 op 구현을 가리키는 `custom_ops` 항목을 추가합니다.

```bash
cp <original_vitisai_config> <integration_dir>/vitisai_config.json
```

Then edit `<integration_dir>/vitisai_config.json` to add the `custom_ops` section inside `vaiml_config`, with the `op_config` path pointing to the YAML in the custom op working directory:
그런 다음 `<integration_dir>/vitsai_config.json`을 편집하여 `vaiml_config` 내에 `custom_ops` 섹션을 추가하고 `op_config` 경로는 사용자 정의 op 작업 디렉터리의 YAML을 가리킵니다.

```json
"custom_ops": {
    "mydomain.<op_name>": {
        "op_config": "<working_dir>/custom_op_<name>/<name>.yaml"
    }
}
```

The path to `op_config` must be either absolute or relative to `<integration_dir>`.
`op_config`에 대한 경로는 절대 경로이거나 `<integration_dir>`에 대한 상대 경로여야 합니다.
Since the kernel .cpp, tiling .py, and YAML config live in `<working_dir>/custom_op_<name>/`, use the absolute path to avoid ambiguity.
커널 .cpp, 타일링 .py 및 YAML 구성은 `<working_dir>/custom_op_<name>/`에 있으므로 모호성을 피하기 위해 절대 경로를 사용하십시오.

**Add the x86sim flags to this copied config** :
**복사된 구성에 x86sim 플래그를 추가하세요**:

```json
      "aiecompiler_args": "--compile-for-x86sim=true --x86sim-as-shared-library=true",
```

## Phase 7: End-to-End Verification
## 7단계: 엔드투엔드 검증

**This phase is MANDATORY.** At minimum, the integrated model must compile successfully.
**이 단계는 필수입니다.** 최소한 통합 모델이 성공적으로 컴파일되어야 합니다.

### 7a.
### Compile the integrated model (REQUIRED)
### 통합 모델 컴파일(필수)

**Run in a sub-agent.** This verifies the custom op is correctly wired into the full model and that the compiler accepts it.
**하위 에이전트에서 실행.** 이는 사용자 정의 작업이 전체 모델에 올바르게 연결되고 컴파일러가 이를 수락하는지 확인합니다.

```bash
bash -c '
<ENV_SETUP>
export DEBUG_VAIML_PARTITION=1
export FLEXML_PRINT_VITISTOOLS_OUTPUT=1
cd <integration_dir>
python <skill_dir>/scripts/compile.py \
    integrated_model.onnx \
    --vitisai-config vitisai_config.json
'
```

Verify: check for `"Compilation successful"` and verify that the custom op appears in a list of ops starting by:
확인: `"컴파일 성공'`을 확인하고 다음으로 시작하는 작업 목록에 사용자 지정 작업이 나타나는지 확인합니다.
```
-----------------------
INFO: [VAIML-COMPILE 1000] Operation statistics for AIE partitions:
 Operations encountered:
-----------------------
[...]
  templatedGraph:mydomain.myop ...
```
The op MUST NOT appear in a list of ops starting by:
작업은 다음으로 시작하는 작업 목록에 표시되어서는 안 됩니다.
```
-----------------------
INFO: [VAIML-COMPILE 1000] Operation statistics for CPU partitions:
 Operations encountered:
-----------------------
```


**IMPORTANT:** When integration compilation fails, trace the root cause systematically -- this is a different failure mode from Phase 3.
**중요:** 통합 컴파일이 실패하면 근본 원인을 체계적으로 추적하십시오. 이는 3단계와는 다른 실패 모드입니다.
The full model context introduces new constraints (e.g., different tensor shapes, shim channel conflicts), so the bug may be either in the config or in the custom op itself.
전체 모델 컨텍스트에는 새로운 제약 조건(예: 다른 텐서 모양, 심 채널 충돌)이 도입되므로 버그는 구성이나 사용자 정의 작업 자체에 있을 수 있습니다.
The bug is not from compile.py, do not attempt to fix this script.
버그는 compile.py에서 발생한 것이 아닙니다. 이 스크립트를 수정하려고 시도하지 마세요.

### 7b.
### Generate full-model CPU reference
### 전체 모델 CPU 참조 생성

```bash
python <skill_dir>/scripts/compile.py \
    <full_model>.onnx --cpu \
    --input-dir <integration_dir>/full_reference_data \
    --output-dir <integration_dir>/full_reference_data
```

Same rule as Phase 1b: pass `--input-dir` too, or the full-model inputs land in
1b단계와 동일한 규칙: `--input-dir`도 전달하지 않으면 전체 모델 입력이
`<model_stem>_inputs/` and the 6c comparison has no inputs to feed the integrated
`<model_stem>_inputs/` 및 6c 비교에는 통합된 데이터를 공급할 입력이 없습니다.
model with.
모델.

### 7c.
### Compare outputs
### 출력 비교

Run x86sim on the integrated model and compare against the original model's CPU output to verify end-to-end correctness.
통합 모델에서 x86sim을 실행하고 원래 모델의 CPU 출력과 비교하여 엔드투엔드 정확성을 확인합니다.

## Key Architecture Facts
## 주요 아키텍처 사실

See [README.md](../../README.md) for the complete architecture reference.
전체 아키텍처 참조는 [README.md](../../README.md)를 참조하세요.

**Quick reminders:**
**빠른 알림:**
- VAIML converts float32 -> bfloat16 automatically; kernels receive bfloat16
- VAIML은 float32 -> bfloat16을 자동으로 변환합니다. 커널은 bfloat16을 수신합니다.
- L1: 64KB total, 0xA000-0xDFFF reserved for stack/heap
- L1: 총 64KB, 스택/힙용으로 예약된 0xA000-0xDFFF
- L2: 512KB per mem tile column; input tensor placement: for column-wise broadcast to L1 on tiles 0-2, for row-wise broadcast to L1 on tiles 2-3
- L2: 메모리 타일 열당 512KB; 입력 텐서 배치: 타일 0-2의 L1에 대한 열 단위 브로드캐스트의 경우, 타일 2-3의 L1에 대한 행 단위 브로드캐스트의 경우
- Output in L2 on tiles 0-2
- 타일 0-2의 L2 출력
- Tile sizes must be aligned to kernel vectorization (e.g., multiples of 16 for bfloat16 vectors)
- 타일 크기는 커널 벡터화에 맞춰 정렬되어야 합니다(예: bfloat16 벡터의 경우 16의 배수).
- DMA requires 32-bit aligned transfers
- DMA에는 32비트 정렬 전송이 필요합니다.
- **No stdc++ runtime**: `std::exp`, `std::tanh`, `std::sqrt`, `<cmath>`, `<algorithm>` are NOT available.
- **stdc++ 런타임 없음**: `std::exp`, `std::tanh`, `std::sqrt`, `<cmath>`, `<algorithm>`을 사용할 수 없습니다.
- Use the AIE API (`aie::` namespace) and NLF hardware intrinsics (`::tanh()`, `::exp2()`) instead
- 대신 AIE API(`aie::` 네임스페이스) 및 NLF 하드웨어 내장 함수(`::tanh()`, `::exp2()`)를 사용하세요.
- Custom ops support at most 2 inputs in the core tiles; pack extras into IFM/WTS via tiling or the DMA controllers, see "Multi-input" in Design Constraints.
- 사용자 지정 작업은 코어 타일에서 최대 2개의 입력을 지원합니다. 타일링이나 DMA 컨트롤러를 통해 IFM/WTS에 추가 기능을 추가합니다. 설계 제약 조건의 "다중 입력"을 참조하세요.
- Custom ops produce 1 output; concatenate outputs in the kernel and split using slicing, see "Multi-output" in Design Constraints.
- 사용자 정의 작업은 1개의 출력을 생성합니다. 커널에서 출력을 연결하고 슬라이싱을 사용하여 분할합니다. 설계 제약 조건의 "다중 출력"을 참조하세요.

## Script Reference
## 스크립트 참조

| Script<br>스크립트 | Usage<br>사용법 |
|--------|-------|
| `compile.py`<br>`컴파일.py` | `python <skill_dir>/scripts/compile.py <model>.onnx` (VitisAI compile) or `compile.py <m> --cpu --output-dir <dir>` (CPU inference)<br>`python <skill_dir>/scripts/compile.py <model>.onnx`(VitisAI 컴파일) 또는 `compile.py <m> --cpu --output-dir <dir>`(CPU 추론) |
| `run.py`<br>`run.py` | `python <skill_dir>/scripts/run.py <model>.onnx` -- runs on VitisAI hardware, **or in x86sim** when the `--vitisai-config` you pass sets `runner_type: "x86sim"`<br>`python <skill_dir>/scripts/run.py <model>.onnx` -- 전달한 `--vitisai-config`가 `runner_type: "x86sim"`을 설정하는 경우 VitisAI 하드웨어 또는 **x86sim**에서 실행됩니다. |
| `onnx_cut.py`<br>'onnx_cut.py' | `python <skill_dir>/scripts/onnx_cut.py --model <m> --report` or `--nodes <n1> [<n2>] --output <out>`<br>`python <skill_dir>/scripts/onnx_cut.py --model <m> --report` 또는 `--nodes <n1> [<n2>] --output <out>` |
| `create_custom_op_model.py`<br>`create_custom_op_model.py` | `python <skill_dir>/scripts/create_custom_op_model.py --model model_a.onnx --custom-op-name <n> --output model_b.onnx` or `--inspect`<br>`python <skill_dir>/scripts/create_custom_op_model.py --model model_a.onnx --custom-op-name <n> --output model_b.onnx` 또는 `--inspect` |
| `onnx_stitch.py`<br>'onnx_stitch.py' | `python <skill_dir>/scripts/onnx_stitch.py --model <m> --nodes <n1> [<n2>] --custom-op-name <n> --output <out>`<br>`python <skill_dir>/scripts/onnx_stitch.py --model <m> --nodes <n1> [<n2>] --custom-op-name <n> --output <out>` |
| `compare_npy.py`<br>`비교_npy.py` | `python <skill_dir>/scripts/compare_npy.py <ref>.npy <test>.npy --rtol 0.01 --atol 0.015`<br>`python <skill_dir>/scripts/compare_npy.py <ref>.npy <테스트>.npy --rtol 0.01 --atol 0.015` |
| `run_on_board.py`<br>`run_on_board.py` | `python <skill_dir>/scripts/run_on_board.py` (run on remote board via SSH)<br>`python <skill_dir>/scripts/run_on_board.py`(SSH를 통해 원격 보드에서 실행) |
| `custom_ops_utils.py`<br>`custom_ops_utils.py` | Imported by other scripts (not called directly)<br>다른 스크립트에서 가져옴(직접 호출되지 않음) |

## Troubleshooting
## 문제 해결

### Common Errors
### 일반적인 오류

1. **`custom op shape infer get shape and data_type failed`** -- The custom op node in the ONNX model is missing the required `shape` and `data_type` attributes.
1. **`사용자 정의 작업 형태 infer get shape 및 data_type 실패`** -- ONNX 모델의 사용자 정의 작업 노드에 필수 `shape` 및 `data_type` 속성이 누락되었습니다.
2. **`unsupported data type 1`** -- XRT not installed; compilation succeeded but inference cannot run.
2. **`지원되지 않는 데이터 유형 1`** -- XRT가 설치되지 않았습니다. 컴파일이 성공했지만 추론을 실행할 수 없습니다.
3. **Buffer overlap in L1** -- Check that ping/pong addresses don't overlap and avoid 0xA000-0xDFFF.
3. **L1의 버퍼 겹침** -- 핑/퐁 주소가 겹치지 않는지 확인하고 0xA000-0xDFFF를 피하세요.
4. **DMA alignment errors** -- Innermost dimension must allow 32-bit aligned transfers.
4. **DMA 정렬 오류** -- 가장 안쪽 차원은 32비트 정렬 전송을 허용해야 합니다.
5. **Shape mismatch in transfers** -- L2 buffer inner dims must match L1 buffer shape exactly.
5. **전송의 모양 불일치** -- L2 버퍼 내부 치수는 L1 버퍼 모양과 정확히 일치해야 합니다.
6. **`kernel_ratios` mismatch** -- Async buffer acquire/release count in C++ must match Python `kernel_ratios`.
6. **`kernel_ratios` 불일치** -- C++의 비동기 버퍼 획득/해제 횟수는 Python `kernel_ratios`와 일치해야 합니다.
7. **Compilation reports success but tiling failed silently** -- Check for `"Compilation successful"`, `"ERROR:"` messages during compilation, and that the op is under `"Operation Statistics for CPU"` instead of `"Operation statistics for AIE Partitions"`.
7. **컴파일이 성공을 보고했지만 타일링이 자동으로 실패했습니다** -- 컴파일 중에 `"컴파일 성공"`, `"ERROR:"` 메시지를 확인하고 작업이 `"AIE 파티션에 대한 작업 통계"` 대신 `"CPU에 대한 작업 통계"` 아래에 있는지 확인합니다.
8. **x86sim NaN / garbage** -- When `enable_f32_to_bf16_conversion: true`, data files must contain bfloat16 data, not float32.
8. **x86sim NaN / 쓰레기** -- `enable_f32_to_bf16_conversion: true`인 경우 데이터 파일에는 float32가 아닌 bfloat16 데이터가 포함되어야 합니다.
10. **`RepeatToMatch` shape error** -- Use `.Reshape([flat_size])` on DDR tensor before `set_l3_to_l2_transfer`.
10. **`RepeatToMatch` 모양 오류** -- `set_l3_to_l2_transfer` 이전의 DDR 텐서에서 `.Reshape([Flat_size])`를 사용하세요.
11. **`Reshape sizes invalid`** -- Use `getPaddedShape()` instead of `getShape()` when computing buffer sizes.
11. **`크기 조정이 잘못되었습니다`** -- 버퍼 크기를 계산할 때 `getShape()` 대신 `getPendedShape()`를 사용하세요.
12. **`set_l2_to_l1_transfer()` TypeError with 2D list** -- `SpatialDistribute2D` returns a nested 2D Python list of `TensorExpr`s, which `set_l2_to_l1_transfer()` does not accept.
12. **`set_l2_to_l1_transfer()` 2D 목록의 TypeError** -- `SpatialDistribute2D`는 `set_l2_to_l1_transfer()`가 허용하지 않는 `TensorExpr`의 중첩된 2D Python 목록을 반환합니다.
12. Use `tensorvar.TileTo(index=0, numTiles=n_rows)` instead, which returns a single `TensorExpr` encoding the row distribution (also works with columns).
12. 대신 `tensorvar.TileTo(index=0, numTiles=n_rows)`를 사용하세요. 이는 행 분포를 인코딩하는 단일 `TensorExpr`을 반환합니다(열에서도 작동함).
13. **`We have not finished Reshape for TensorExprs`** -- This VAIML error is **non-fatal**.
13. **`TensorExprs에 대한 Reshape가 완료되지 않았습니다`** -- 이 VAIML 오류는 **치명적이지 않습니다**.
13. The framework logs it as a plugin warning (tags `VAIML-MKT 13999` / `VAIML-COMPILE 1000`) but does not abort compilation.
13. 프레임워크는 이를 플러그인 경고(`VAIML-MKT 13999` / `VAIML-COMPILE 1000` 태그)로 기록하지만 컴파일을 중단하지 않습니다.
13. If this error shows up, despite the process exiting with 0 and printing `"Success!"`, the compilation of the custom op failed.
13. 이 오류가 나타나면 프로세스가 0으로 종료되고 `"Success!"`가 인쇄되었음에도 불구하고 사용자 정의 작업 컴파일이 실패한 것입니다.
13. You need to fix the tiling in that case.
13. 이 경우 타일링을 수정해야 합니다.
14. **Compilation exit code 0 masks tiling errors** -- VAIML may log `Reshape sizes invalid` or `RepeatToMatch shape mismatch` errors but still exit 0 with `"Custom ops model compiled successfully"`.
14. **컴파일 종료 코드 0은 타일링 오류를 마스킹합니다** -- VAIML은 '잘못된 크기 조정' 또는 'RepeatToMatch 모양 불일치' 오류를 기록할 수 있지만 여전히 '"사용자 정의 작업 모델이 성공적으로 컴파일되었습니다"'와 함께 0을 종료합니다.
14. These plugin errors are not propagated to the process exit code.
14. 이러한 플러그인 오류는 프로세스 종료 코드로 전파되지 않습니다.
14. Always check the actual log output for errors, not just the exit code.
14. 종료 코드뿐 아니라 항상 실제 로그 출력에서 ​​오류를 확인하세요.
14. A model that compiles with logged errors may still fail at runtime.
14. 오류가 기록된 상태로 컴파일되는 모델은 런타임 시 여전히 실패할 수 있습니다.
15. **Kernel signature mismatch from tiling script** -- The kernel must have the number of arguments set in `set_kernel_params` plus the last argument being `lp_params`.
15. **타일링 스크립트의 커널 서명 불일치** -- 커널에는 `set_kernel_params`에 설정된 인수 수와 `lp_params`인 마지막 인수가 있어야 합니다.
16. **AIE runtime failure despite successful compilation** -- The custom op compiles and VAIML reports 100% AIE offload, but `sess.run()` throws `RUNTIME_EXCEPTION: Custom op '<name>' couldn't be executed on the AIE`.
16. **성공적인 컴파일에도 불구하고 AIE 런타임 실패** -- 사용자 정의 작업이 컴파일되고 VAIML이 100% AIE 오프로드를 보고하지만 `sess.run()`에서 `RUNTIME_EXCEPTION: 사용자 정의 작업 '<name>'을(를) AIE에서 실행할 수 없습니다'가 발생합니다.
16. CPU fallback is not supported for custom ops.
16. 사용자 지정 작업에는 CPU 대체가 지원되지 않습니다.
16. Common causes: (a) kernel C++ signature does not match the generated ADF call from tiling, (b) tiling produces data movement specs that pass compile validation but fail hardware execution, (c) `gen_tg_adf/` files contain errors masked by exit code 0.
16. 일반적인 원인: (a) 커널 C++ 서명이 타일링에서 생성된 ADF 호출과 일치하지 않음, (b) 타일링이 컴파일 유효성 검사를 통과했지만 하드웨어 실행에 실패하는 데이터 이동 사양을 생성함, (c) `gen_tg_adf/` 파일에 종료 코드 0으로 마스크된 오류가 포함되어 있음.
16. Check `l2_to_l3_tiling_analysis.csv` and the `pm_reload_analysis0.cc` generated source for clues.
16. 단서를 찾으려면 `l2_to_l3_tiling_analytic.csv`와 `pm_reload_analytic0.cc` 생성 소스를 확인하세요.
17. **`mydomain:<op_name>(-1) is not a registered function/op`** -- OnnxRuntime cannot find the custom op during session creation.
17. **`mydomain:<op_name>(-1)은 등록된 함수/op이 아닙니다`** -- OnnxRuntime은 세션 생성 중에 사용자 정의 작업을 찾을 수 없습니다.
17. This happens when the custom op shared library is not built or not correctly loaded.
17. 이는 사용자 정의 op 공유 라이브러리가 빌드되지 않았거나 올바르게 로드되지 않았을 때 발생합니다.
17. Note: `custom_ops_utils.register_custom_ops_from_vitisai_config()` registers ops at the Python level, but `VitisAIExecutionProvider` performs its own C++ op lookup independently.
17. 참고: `custom_ops_utils.register_custom_ops_from_vitsai_config()`는 Python 수준에서 작업을 등록하지만 `VitisAIExecutionProvider`는 자체 C++ 작업 조회를 독립적으로 수행합니다.
17. For VAIML compilation, use `compile.py` which handles registration correctly.
17. VAIML 컴파일의 경우 등록을 올바르게 처리하는 `compile.py`를 사용하세요.
17. For standalone inference, ensure the shared library path is correct in `vitisai_config.json`.
17. 독립형 추론의 경우 `vitsai_config.json`에서 공유 라이브러리 경로가 올바른지 확인하세요.
18. **`Reshape()` after `Tile()` is not supported** -- Calling `.Reshape()` on a `TensorExpr` that has already been tiled (via `Tile()` or `TileBy()`) causes `RuntimeError: "not finished Reshape"`.
18. **`Tile()` 이후의 `Reshape()`는 지원되지 않습니다** -- (`Tile()` 또는 `TileBy()`를 통해) 이미 타일링된 `TensorExpr`에서 `.Reshape()`를 호출하면 `RuntimeError: "not done Reshape"`가 발생합니다.
18. Reshape must be applied before tiling operations.
18. 타일링 작업 전에 모양 변경을 적용해야 합니다.
18. Reorder the chain to `.Reshape([...]).TileBy(...)` instead of `.TileBy(...).Reshape([...])`.
18. `.TileBy(...).Reshape([...])` 대신 `.Reshape([...]).TileBy(...)`로 체인 순서를 변경합니다.
19. **Wrong dimension tiling on multi-dimensional data** -- When tiling multi-dimensional data (e.g., `[Cin, Din, Hin, Win]`), tile on the correct dimension to preserve data layout.
19. **다차원 데이터의 잘못된 차원 타일링** -- 다차원 데이터를 타일링할 때(예: `[Cin, Din, Hin, Win]`) 올바른 차원에 타일을 배치하여 데이터 레이아웃을 유지하세요.
19. Flattening to 1D then tiling loses the multi-dimensional structure.
19. 1D로 평면화한 후 타일링하면 다차원 구조가 손실됩니다.
19. Wrong: `Reshape([Cin*Din*HW]).Tile(0, ...)` (mixes Cin and D dimensions).
19. 잘못됨: `Reshape([Cin*Din*HW]).Tile(0, ...)`(Cin 및 D 치수 혼합).
19. Right: `Reshape([Cin, Din*HW]).Tile(1, ...)` (tiles D within each Cin channel).
19. 오른쪽: `Reshape([Cin, Din*HW]).Tile(1, ...)`(각 Cin 채널 내의 타일 D).
20. **Daemon endpoint bind failure on board** -- If a board-side runtime daemon log reports `"Failed to create acceptor endpoint ..., exception: bind: Cannot assign requested address"`, the daemon cannot bind to its loopback endpoint -- typically because the relevant IPv4/IPv6 stack is disabled on the board host.
20. **보드의 데몬 끝점 바인딩 실패** - 보드 측 런타임 데몬 로그에서 `"수락자 끝점 생성 실패..., 예외: 바인드: 요청된 주소를 할당할 수 없음'`을 보고하는 경우 데몬은 루프백 끝점에 바인딩할 수 없습니다. 일반적으로 보드 호스트에서 관련 IPv4/IPv6 스택이 비활성화되어 있기 때문입니다.
20. This is an infrastructure issue, not a kernel/tiling bug.
20. 이는 커널/타일링 버그가 아닌 인프라 문제입니다.
20. Check the loopback configuration on the board host.
20. 보드 호스트의 루프백 구성을 확인하십시오.
21. **`Failed to load x86sim library ...` on the board**, then `RUNTIME_EXCEPTION ...
21. **보드에 `x86sim 라이브러리를 로드하지 못했습니다...`**, 그런 다음 `RUNTIME_EXCEPTION ...
21. HW context creation unsuccessful` -- `runner_type` resolved to `x86sim` (see the preceding `[VAIML-CUSTOMOP] runner_type:` line), so the board tried to load the x86-64 `libx86_sim.so` on aarch64.
21. HW 컨텍스트 생성 실패` -- `runner_type`이 `x86sim`으로 확인되었으므로(이전 `[VAIML-CUSTOMOP]runner_type:` 줄 참조) 보드는 aarch64에서 x86-64 `libx86_sim.so`를 로드하려고 시도했습니다.
21. Either you passed the x86sim config, or you *compiled* with the key: the cache stores it, and omitting it later means unspecified, not `"hw"`.
21. x86sim 구성을 전달했거나 키를 사용하여 *컴파일*했습니다. 캐시에 이를 저장하고 나중에 생략한다는 것은 `"hw"`가 아니라 지정되지 않음을 의미합니다.
21. Check with `grep -o 'runner_type[^,}]*' <cache>/vaiml_par_0/compile_flags.json` and recompile without it.
21. `grep -o 'runner_type[^,}]*' <cache>/vaiml_par_0/compile_flags.json`을 확인하고 그것 없이 다시 컴파일하십시오.
22. **x86sim outputs all zeros, yet `Execution successful` and exit code 0** -- `--num-runs` left at its default of 10.
22. **x86sim은 모두 0을 출력하지만 `실행 성공` 및 종료 코드는 0** -- `--num-runs`는 기본값인 10으로 남아 있습니다.
22. Only the first iteration computes (`ERROR: Layered execution is supported only for designs that have a single run iteration`); the rest yield zeros and `run.py` saves the last.
22. 첫 번째 반복에서만 계산됩니다('오류: 계층화된 실행은 단일 실행 반복이 있는 설계에만 지원됩니다'). 나머지는 0을 산출하고 `run.py`는 마지막을 저장합니다.
22. Use `--num-runs 1`.
22. `--num-runs 1`을 사용하세요.
22. Not a cache/config problem -- recompiling with `runner_type: "x86sim"` changes nothing.
22. 캐시/구성 문제가 아닙니다. `runner_type: "x86sim"`으로 다시 컴파일하면 아무 것도 변경되지 않습니다.

### Environment Setup Checklist
### 환경 설정 체크리스트

```bash
<ENV_SETUP>                                    # source <activate>, or omit if pre-provisioned
export DEBUG_VAIML_PARTITION=1                 # Show partition info
export FLEXML_PRINT_VITISTOOLS_OUTPUT=1        # Show aiecompiler output
rm -rf /tmp/$USER/vaip/.cache                  # Clear cache if needed
```
