<!-- ko-translation: enabled -->
---
name: vai-custom-op
license: Apache-2.0 WITH LLVM-exception
compatibility: Requires VitisAI 6.3+
metadata:
  version: 6.3.0
  stage: beta
description: Orchestrate multi-agent parallel development of multiple custom AIE operators
  for VAIML. Handles workspace partitioning, spawning worker agents, relaying insights
  between workers, gating integration on verified results, and serializing the final
  model stitching. Use when developing 2+ custom ops in parallel, when coordinating
  multi-agent custom op work, or when integrating multiple validated custom ops into
  a single model.
---
<!--- Copyright (C) 2026 Advanced Micro Devices, Inc. All rights reserved.--->

# Custom Op Workflow Orchestrator
# 맞춤형 작업 워크플로 조정자

## Description
## 설명

Coordinates parallel development of multiple custom AIE operators by spawning
생성을 통해 여러 사용자 정의 AIE 연산자의 병렬 개발을 조정합니다.
worker agents, managing hermetic workspaces, and serializing the final
작업자 에이전트, 밀폐 작업 공간 관리 및 최종 직렬화
integration into one model.
하나의 모델로 통합.
Each worker uses the `vai-custom-op` skill for
각 작업자는 'vai-custom-op' 스킬을 사용하여
Phases 1-5; this orchestrator owns Phase 6 (stitching) and Phase 7
1~5단계; 이 오케스트레이터는 6단계(스티칭) 및 7단계를 소유합니다.
(end-to-end verification).
(종단간 검증).

## Parse Arguments
## 인수 구문 분석

This skill expects arguments in this form:
이 기술은 다음 형식의 인수를 기대합니다.

```
[--t <activate_path>] --model <onnx_model> --vitisai-config <config_json> --ops <op1_nodes:op1_name> [<op2_nodes:op2_name> ...]
```

Extract from `$ARGUMENTS`:
`$ARGUMENTS`에서 추출:
- `--t <path>`: **(OPTIONAL)** Path to a Ryzen AI / Vitis AI / product virtual environment activate script.
- `--t <경로>`: **(선택 사항)** Ryzen AI / Vitis AI / 제품 가상 환경 활성화 스크립트의 경로입니다.
- Provide this only when the Python environment is NOT already on the shell.
- Python 환경이 셸에 아직 없는 경우에만 이를 제공하세요.
- Omit it when the environment is pre-provisioned (e.g. inside the official Vitis AI Docker container, where `flexml`, `onnx`, `onnxruntime`, etc. are already importable from system Python).
- 환경이 사전 프로비저닝된 경우(예: `flexml`, `onnx`, `onnxruntime` 등이 이미 시스템 Python에서 가져올 수 있는 공식 Vitis AI Docker 컨테이너 내부)인 경우 이를 생략합니다.
- If `--t` is provided, forward it verbatim to every spawned worker; if it is absent, do NOT invent a path -- spawn workers without `--t`.
- `--t`가 제공되면 생성된 모든 작업자에게 그대로 전달합니다. 없는 경우 경로를 만들지 마십시오. `--t` 없이 작업자를 생성하십시오.
- `--model <path>`: **(MANDATORY)** Path to the full ONNX model
- `--model <경로>`: **(필수)** 전체 ONNX 모델의 경로
- `--vitisai-config <path>`: **(MANDATORY)** Path to vitisai_config.json
- `--vitisai-config <경로>`: **(필수)** vitisai_config.json의 경로
- `--ops <nodes:name> [...]`: **(MANDATORY)** One or more op specs, each in the
- `--ops <nodes:name> [...]`: **(필수)** 하나 이상의 작업 사양, 각각
form `node1,node2:op_name` (comma-separated nodes, colon, custom op name)
`node1,node2:op_name` 형식(쉼표로 구분된 노드, 콜론, 사용자 정의 작업 이름)

**Validation**: Verify all mandatory arguments are present before proceeding.
**검증**: 계속하기 전에 모든 필수 인수가 있는지 확인하세요.
`--t` is intentionally optional -- treat its absence as a deliberate signal to use the system-provisioned environment, not as missing input.
'--t'는 의도적으로 선택 사항입니다. 해당 부재를 입력 누락이 아닌 시스템 제공 환경을 사용하기 위한 의도적인 신호로 처리합니다.
Do NOT search the filesystem for an activate script to plug in.
플러그인할 활성화 스크립트를 파일 시스템에서 검색하지 마십시오.

## Role Split
## 역할 분할

| Role<br>역할 | Owns<br>소유 | Tools<br>도구 |
|------|------|-------|
| **Orchestrator** (you, the team lead)<br>**오케스트레이터** (당신, 팀 리더) | Partition work, prepare hermetic workspaces, spawn workers, relay peer insights, gate integration, run Phases 6 and 7 serially.<br>작업 분할, 밀폐 작업 공간 준비, 작업자 생성, 동료 통찰력 전달, 게이트 통합, 6단계와 7단계를 순차적으로 실행합니다. | `Task`/`Agent` tool to spawn workers; direct edits for integration.<br>작업자를 생성하는 `Task`/`Agent` 도구; 통합을 위한 직접 편집. |
| **Worker** (one per custom op)<br>**작업자**(사용자 정의 작업당 하나) | Phases 1-5 inside its own hermetic workspace. Must prove BOTH successful compilation AND numeric accuracy before reporting back.<br>자체 밀폐 작업 공간 내부의 1~5단계. 다시 보고하기 전에 성공적인 컴파일과 수치 정확성을 모두 입증해야 합니다. | The `vai-custom-op-implementation` skill, scoped to a single `--nodes` set.<br>단일 `--nodes` 세트로 범위가 지정된 `vai-custom-op-implementation` 기술. |

## Workspace Layout
## 작업공간 레이아웃

Create one hermetic sub-directory per custom op plus a reserved directory for
사용자 지정 작업당 하나의 밀폐 하위 디렉터리와 다음을 위해 예약된 디렉터리를 만듭니다.
integration:
통합:

```
<parent_dir>/
  <op_name_1>/          # worker 1 workspace (hermetic)
    <full_model>.onnx   # copy (or symlink) of the original model
    vitisai_config.json  # copy of the original config
    ...                  # worker creates model_a.onnx, model_b.onnx,
                         # custom_op_<name>/, reference_data/, etc.
  <op_name_2>/          # worker 2 workspace (hermetic)
    ...
  integration/           # reserved for orchestrator Phase 6-7
```

**Hermeticity rule:** each worker reads and writes **only** inside its own
**밀폐성 규칙:** 각 작업자는 자신의 내부 **만** 읽고 씁니다.
`<parent_dir>/<op_name_N>/` directory.
`<상위_디렉터리>/<op_name_N>/` 디렉토리.
No worker touches another worker's
어떤 작업자도 다른 작업자의 손을 만지지 않습니다.
directory or the `integration/` directory.
디렉토리 또는 `integration/` 디렉토리.

## Orchestrator Workflow
## 오케스트레이터 워크플로

The orchestrator is data-driven.
오케스트레이터는 데이터 기반입니다.
Decisions must be taken based on real world data
실제 데이터를 기반으로 결정을 내려야 합니다.
and measurements.
그리고 측정.
Certain decisions are not black and white and require
특정 결정은 흑백이 아니며 요구 사항이 있습니다.
a full understand and proper bottleneck analysis to figure out the scope for the
문제의 범위를 파악하기 위한 완전한 이해와 적절한 병목 현상 분석
implementation of a custom op.
맞춤형 작전을 구현합니다.
Auxiliary tools and skills (such as `vai-perf-analysis`)
보조 도구 및 기술(예: 'vai-perf-analytic')
have been developed to help you in this task.
이 작업에 도움이 되도록 개발되었습니다.


1. **Partition the node list.** Perform model analysis using compilation logs
1. **노드 목록을 분할합니다.** 컴파일 로그를 사용하여 모델 분석을 수행합니다.
or `vai-perf-analysis` - if runtime data is present - to figure out the optimal
또는 'vai-perf-analytic' - 런타임 데이터가 있는 경우 - 최적의 성능을 파악하기 위해
number of custom ops and which target nodes to include in one cluster.
사용자 정의 작업 수 및 하나의 클러스터에 포함할 대상 노드.
Each cluster becomes one worker and a custom op.
각 클러스터는 하나의 작업자와 사용자 지정 작업이 됩니다.
   1. **Sanity-check the high-level decomposition.** The orchestrator's
   1. **상위 수준 분해를 온전하게 확인합니다.** 오케스트레이터의
job is to decide *how many* custom ops are worth building and
임무는 *얼마나* 많은* 사용자 정의 작업을 구축할 가치가 있는지 결정하고
*which nodes* land in each cluster.
*각 클러스터에 어떤 노드*가 배치되는지입니다.
Custom ops have hard buffer
사용자 정의 작업에는 하드 버퍼가 있습니다.
limits (at most 2 inputs, 1 output -- see the `vai-custom-op-implementation` skill,
제한(최대 2개의 입력, 1개의 출력 - `vai-custom-op-implementation` 기술 참조,
"Design Constraints"), but the worker can pack extras into IFM/WTS
"설계 제약 조건"). 그러나 작업자는 IFM/WTS에 추가 항목을 넣을 수 있습니다.
via tiling and DMA, so a raw input/output count is **not** a strict
타일링과 DMA를 통해 이루어지므로 원시 입력/출력 개수는 엄격하지 **않습니다**
partitioning constraint.
분할 제약.
Use this step to catch decomposition-level
분해 수준을 포착하려면 이 단계를 사용하세요.
issues only:
문제만:
      - The cluster is so large that it should clearly be **split into
      - 클러스터가 너무 커서 분명히 **분할되어야 합니다.
multiple custom ops** (e.g., distinct compute regions with little
여러 사용자 지정 작업**(예: 거의 없는 별도의 컴퓨팅 영역)
data sharing, or a long chain that exceeds reasonable kernel
데이터 공유 또는 합리적인 커널을 초과하는 긴 체인
complexity).
복잡성).
      - The cluster is so trivial that it does **not warrant a custom op**
      - 클러스터는 너무 사소하여 **맞춤형 작업을 보장하지 않습니다**
at all (e.g., a single elementwise node already handled well by
(예를 들어 단일 요소별 노드는 이미 잘 처리되었습니다.
the default backend) and should be dropped.
기본 백엔드) 삭제해야 합니다.
      - Topology issues that no amount of packing can fix: e.g., a node
      - 아무리 패킹해도 해결할 수 없는 토폴로지 문제(예: 노드)
in the middle of the cluster has an output consumed both inside
클러스터 중간에는 내부에서 모두 소비되는 출력이 있습니다.
and outside the cluster in a way that would require duplicating
복제가 필요한 방식으로 클러스터 외부에
a large compute, or the cluster spans a control-flow boundary.
대규모 컴퓨팅이거나 클러스터가 제어 흐름 경계에 걸쳐 있습니다.

Per-op concerns -- exact IFM/WTS packing layout, multi-output
작업별 문제 - 정확한 IFM/WTS 패킹 레이아웃, 다중 출력
kernel concat + tiling split, attribute vs.
커널 연결 + 타일링 분할, 속성 대
`lp_params` choice --
`lp_params` 선택 --
are deferred to the worker, which has full context on the kernel
커널에 대한 전체 컨텍스트를 갖고 있는 작업자에게 연기됩니다.
and tiling.
그리고 타일링.
Forward each surviving cluster to a worker as-is and
각 살아남은 클러스터를 있는 그대로 작업자에게 전달하고
let it negotiate the buffer constraints internally.
내부적으로 버퍼 제약 조건을 협상하도록 하세요.
2. **Prepare hermetic workspaces.** For each cluster, create
2. **밀폐형 작업 공간을 준비합니다.** 각 클러스터에 대해 다음을 생성합니다.
`<parent_dir>/<op_name>/` and copy (or symlink) the original model and
`<parent_dir>/<op_name>/` 및 원본 모델을 복사(또는 심볼릭 링크)하고
`vitisai_config.json` into it.
`vitsai_config.json`을 그 안에 넣습니다.
3. **Spawn workers in parallel.** Use the `Agent` tool to spawn one subagent
3. **작업자를 병렬로 생성합니다.** '에이전트' 도구를 사용하여 하나의 하위 에이전트를 생성합니다.
per custom op **simultaneously** (all Agent calls in a single message).
사용자 지정 작업별로 **동시에**(단일 메시지의 모든 에이전트 호출)
Each worker's prompt MUST instruct it to invoke the `/vai-custom-op-implementation` skill
각 작업자의 프롬프트는 반드시 `/vai-custom-op-implementation` 기술을 호출하도록 지시해야 합니다.
with the appropriate arguments.
적절한 인수로.
Use `subagent_type="vai-custom-op-worker"`
`subagent_type="vai-custom-op-worker"`를 사용하세요.
and `mode="bypassPermissions"`.
및 `mode="bypassPermissions"`.

**Worker prompt template** (one per custom op):
**작업자 프롬프트 템플릿**(맞춤 작업당 하나):
   ```
   You are a vai-custom-op-worker agent. Your ONLY task is to invoke the
   /vai-custom-op-implementation skill with the following arguments and report the result.

   Run this skill now:
   /vai-custom-op-implementation <operation_description> \
       [--t <activate_path>]    # include ONLY if the orchestrator was invoked with --t; otherwise omit this line entirely
       --model <parent_dir>/<op_name>/<full_model>.onnx \
       --vitisai-config <parent_dir>/<op_name>/vitisai_config.json \
       --nodes <nodes_for_this_op_only>

   Work ONLY inside <parent_dir>/<op_name>/. Execute Phases 1-5 of the
   /vai-custom-op-implementation skill in full -- Phase 5 (kernel optimization, including
   board-based measurement with /vai-perf-analysis when a board is
   reachable) is part of the skill's own contract and is mandatory.
   Apply techniques from BOTH /vai-aie-kernel-development AND
   /vai-aie-compiler-oriented-optimizations. Do NOT attempt Phase 6 or 7;
   those are owned by the orchestrator.

   When done, report back with: status, op_name, working_dir, yaml_path,
   compile_verdict, numeric_verdict, optimization_verdict, and any
   peer_insights.
   ```

**IMPORTANT:** Launch ALL workers in a single message with multiple
**중요:** 여러 메시지가 포함된 단일 메시지로 모든 작업자를 시작합니다.
Agent tool calls -- do not wait for one worker to finish before spawning
에이전트 도구 호출 - 작업자 한 명이 작업을 완료할 때까지 기다리지 말고 생성하세요.
the next.
다음.
The whole point of this orchestrator is parallel execution.
이 오케스트레이터의 요점은 병렬 실행입니다.
4. **Relay peer insights (optional).** Workers do **not** talk to each other.
4. **동료 통찰 전달(선택 사항).** 작업자는 서로 대화하지 **않습니다**.
If a worker surfaces an insight likely to help peers (e.g., a tiling pattern
작업자가 동료에게 도움이 될 만한 통찰력을 표면화하는 경우(예: 타일링 패턴
for similar shapes, a debugging discovery), the orchestrator forwards it to
비슷한 모양의 경우 디버깅 검색) 오케스트레이터는 이를 다음으로 전달합니다.
the relevant peers.
관련 동료.
5. **Gate integration on worker success.** For each worker, verify its report
5. **작업자 성공에 대한 게이트 통합.** 각 작업자에 대해 보고서를 확인합니다.
**before** integrating its op:
**작업 통합 전**:
   - Phase 3: `"Compilation successful"` AND non-empty
   - 3단계: `"컴파일 성공"` AND 비어 있지 않음
`<model_stem>/cache/vaiml_par_0/0/backend/gen_tg_adf/`
`<모델_줄기>/cache/vaiml_par_0/0/backend/gen_tg_adf/`
   - Phase 4b: `compare_npy.py` returns `SUCCESS`
   - 4b단계: `compare_npy.py`가 `SUCCESS`를 반환합니다.
If either check fails, ask the worker to iterate, reassign the op, or drop
두 가지 확인 중 하나라도 실패하면 작업자에게 반복하거나 작업을 재할당하거나 삭제하도록 요청하세요.
it from the integration batch. **Never integrate an unverified custom op.**
통합 배치에서 가져옵니다. **확인되지 않은 맞춤 작업을 통합하지 마세요.**
6. **Serialize Phases 6 and 7.** For each validated custom op in turn:
6. **6단계와 7단계를 직렬화합니다.** 검증된 각 사용자 지정 작업에 대해 차례로 다음을 수행합니다.
   - Stitch it into the current integrated model (Phase 6b of `vai-custom-op-implementation` skill).
   - 이를 현재 통합 모델('vai-custom-op-implementation' 기술의 6b단계)에 연결합니다.
   - Add its `custom_ops` entry to `integration/vitisai_config.json` (Phase 6c)
   - `integration/vitsai_config.json`에 `custom_ops` 항목을 추가합니다(6c단계).
with `op_config` pointing at the worker's YAML (absolute path).
작업자의 YAML(절대 경로)을 가리키는 `op_config`를 사용합니다.
   - Recompile to confirm the incremental integration still succeeds.
   - 증분 통합이 여전히 성공하는지 확인하기 위해 다시 컴파일합니다.
After all ops are integrated, run Phase 7 **once** on the final model.
모든 작업이 통합된 후 최종 모델에서 7단계를 **한 번** 실행하세요.

## Worker Contract
## 근로자 계약

Each worker agent MUST:
각 작업자 에이전트는 다음을 수행해야 합니다.

- Operate **only** inside its assigned `<parent_dir>/<op_name>/` directory.
- 할당된 `<parent_dir>/<op_name>/` 디렉토리 내에서만 **작동**하십시오.
- Execute Phases 1-5 of the `vai-custom-op-implementation` skill end-to-end, including a local
- 로컬을 포함하여 'vai-custom-op-implementation' 기술의 1~5단계를 처음부터 끝까지 실행합니다.
git repo with milestone commits.
마일스톤 커밋이 있는 git repo.
- **Not** attempt Phase 6 or Phase 7.
- 6단계나 7단계를 시도하지 **않습니다**.
- Those belong to the orchestrator.
- 그것들은 오케스트레이터에 속합니다.
- Return a structured report to the orchestrator with:
- 다음을 사용하여 구조화된 보고서를 조정자에게 반환합니다.
  - `status`: `READY_FOR_INTEGRATION` or `FAILED`
  - `상태`: `READY_FOR_INTEGRATION` 또는 `FAILED`
  - `op_name` and `working_dir` (absolute path)
  - `op_name` 및 `working_dir`(절대 경로)
  - `yaml_path`: absolute path to `custom_op_<name>/<name>.yaml`
  - `yaml_path`: `custom_op_<name>/<name>.yaml`의 절대 경로
(needed by the orchestrator for Phase 6c)
(6c단계의 오케스트레이터에 필요)
  - `compile_verdict`: success/failure plus `gen_tg_adf/` non-empty check
  - `compile_verdict`: 성공/실패 및 `gen_tg_adf/` 비어 있지 않은 확인
  - `numeric_verdict`: `compare_npy.py` max abs diff, mean abs diff, and
  - `numeric_verdict`: `compare_npy.py` 최대 절대값 차이, 평균 절대값 차이 및
SUCCESS/ERROR
성공/오류
  - `optimization_verdict`: which Phase 5 optimizations were applied,
  - `optimization_verdict`: 5단계 최적화가 적용되었습니다.
drawn from BOTH /vai-aie-kernel-development (vectorization at minimum,
/vai-aie-kernel-development(최소 벡터화,
NOINLINE core pattern, loop splitting, etc.) AND
NOINLINE 코어 패턴, 루프 분할 등) AND
/vai-aie-compiler-oriented-optimizations (restrict pointers, DM bank
/vai-aie-컴파일러 지향-최적화(포인터 제한, DM 뱅크
annotations, loop pragmas / hints, software pipelining, vector
주석, 루프 pragma/힌트, 소프트웨어 파이프라이닝, 벡터
alignment, etc.); the Phase-4 re-validation result after each;
정렬 등); 각 단계 이후의 4단계 재검증 결과
board-measurement results from /vai-perf-analysis (per-op time,
/vai-perf-analytic의 보드 측정 결과(작업당 시간,
bottleneck class, before/after deltas) OR a documented reason no
병목 현상 클래스, 델타 이전/이후) 또는 문서화된 이유 아니요
board was reachable; and the final kernel classification
보드에 접근할 수 있었습니다. 최종 커널 분류
(vectorized / scalar-fallback-with-justification).
(벡터화/정렬을 통한 스칼라 대체).
A scalar kernel
스칼라 커널
without a documented vectorization attempt, or a "board available
문서화된 벡터화 시도 없이 또는 "보드 사용 가능"
but not measured" state, is a FAILED verdict.
측정되지 않음' 상태는 FAILED 판정입니다.
  - `peer_insights` (optional): anything the orchestrator should forward
  - `peer_insights`(선택 사항): 오케스트레이터가 전달해야 하는 모든 것

## Blocking Requirements
## 차단 요구 사항

- **BLOCKING:** A custom op must pass BOTH the compilation and the numeric
- **차단:** 사용자 정의 작업은 컴파일과 숫자를 모두 통과해야 합니다.
accuracy checks before being integrated.
통합되기 전에 정확성을 확인합니다.
No exceptions.
예외는 없습니다.
- **BLOCKING:** Phase 6 (stitch + `vitisai_config.json` edits) and Phase 7
- **차단:** 6단계(스티치 + `vitsai_config.json` 편집) 및 7단계
(end-to-end verification) must be executed serially by the orchestrator.
(종단 간 확인)은 오케스트레이터에 의해 순차적으로 실행되어야 합니다.
Parallel edits to the full model or its config will corrupt the integration.
전체 모델이나 해당 구성을 동시에 편집하면 통합이 손상됩니다.

## Systematic Debugging at Every Step
## 모든 단계에서 체계적인 디버깅

**IMPORTANT:** When any step fails -- worker compilation, x86sim, numeric comparison, or integration compile -- do not resort to trial and error.
**중요:** 작업자 컴파일, x86sim, 수치 비교 또는 통합 컴파일 등 단계가 실패하는 경우 시행착오를 거치지 마세요.
Debug systematically: read the full error output, trace backward to find the root cause, and apply a targeted fix.
체계적인 디버그: 전체 오류 출력을 읽고, 역추적하여 근본 원인을 찾고, 목표 수정 사항을 적용합니다.
Check if there are other skills available that can help with debugging.
디버깅에 도움이 될 수 있는 다른 기술이 있는지 확인하세요.
When a worker reports FAILED, ask for the specific error message and root cause analysis before telling it to retry.
작업자가 FAILED를 보고하면 재시도를 지시하기 전에 구체적인 오류 메시지와 근본 원인 분석을 요청하세요.

## Integration Phase (Orchestrator Only)
## 통합 단계(Orchestrator만 해당)

### Phase 6: Model Re-integration (Serial)
### 6단계: 모델 재통합(직렬)

For each validated custom op, in order:
검증된 각 사용자 지정 작업에 대해 순서대로:

1. **Stitch** the custom op into the current integrated model using
1. **맞춤**을 사용하여 현재 통합 모델에 맞춤 작업을 연결합니다.
`onnx_stitch.py` from the `vai-custom-op-implementation` skill's scripts directory.
`vai-custom-op-implementation` 스킬의 스크립트 디렉터리에 있는 `onnx_stitch.py`.
2. **Update** `integration/vitisai_config.json` to add the `custom_ops` entry
2. **업데이트** `integration/vitsai_config.json`을 추가하여 `custom_ops` 항목 추가
with `op_config` pointing to the worker's YAML (absolute path).
작업자의 YAML(절대 경로)을 가리키는 `op_config`를 사용합니다.
3. **Recompile** the integrated model to confirm the incremental integration
3. 증분 통합을 확인하기 위해 통합 모델을 **재컴파일**합니다.
succeeds before adding the next custom op.
다음 사용자 정의 작업을 추가하기 전에 성공합니다.

### Phase 7: End-to-End Verification (Once)
### 7단계: 엔드투엔드 검증(1회)

After all custom ops are integrated:
모든 사용자 정의 작업이 통합된 후:

1. Compile the final integrated model.
1. 최종 통합 모델을 컴파일합니다.
2. Generate full-model CPU reference data.
2. 전체 모델 CPU 참조 데이터를 생성합니다.
3. Run x86sim on the integrated model.
3. 통합 모델에서 x86sim을 실행합니다.
4. Compare x86sim outputs against CPU reference.
4. x86sim 출력을 CPU 참조와 비교합니다.

This phase is **mandatory**.
이 단계는 **필수**입니다.
At minimum, the integrated model must compile
최소한 통합 모델은 다음을 컴파일해야 합니다.
successfully.
성공적으로.

## Final Report
## 최종 보고서

After Phase 7 completes, generate a markdown report at
7단계가 완료되면 다음 위치에서 가격인하 보고서를 생성하세요.
`<parent_dir>/report.md` summarizing the entire workflow.
전체 작업 흐름을 요약하는 `<parent_dir>/report.md`
Use the following
다음을 사용하십시오
structure:
구조:

```markdown
# Custom Op Development Report

## Summary
- **Model**: <original model path>
- **Custom ops developed**: <count>
- **Final status**: <all integrated / partial / failed>

## Per-Op Results

### <op_name_1>
- **Target nodes**: <node list>
- **Status**: READY_FOR_INTEGRATION | FAILED
- **Compilation**: success/failure
- **Numeric accuracy**: max abs diff, mean abs diff, SUCCESS/ERROR
- **Tiling strategy**: <Tiling_4x4 / Tiling_1x1, tile size, distribution>
- **Working directory**: <absolute path>
- **Issues encountered**: <brief description of failures and how they were resolved>

### <op_name_2>
...

## Integration
- **Integrated model**: <path to final model>
- **Integration compilation**: success/failure
- **End-to-end numeric accuracy**: max abs diff, mean abs diff (if run)

## Performance
- **Compilation time**: <per-op and total>
- **x86sim time**: <per-op>
- **Kernel configuration**: <vectorized / scalar, vector width, pipelining>

## Issues & Resolutions
| Issue | Root Cause | Resolution |
|-------|-----------|------------|
| ...   | ...       | ...        |
```

Include all attempts, not just the final successful run -- this makes the
최종 성공적인 실행뿐만 아니라 모든 시도를 포함합니다.
report useful for future reference when developing similar ops.
유사한 작전을 개발할 때 나중에 참고할 때 유용한 보고서입니다.
