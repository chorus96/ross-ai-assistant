<!-- ko-translation: enabled -->
# Custom operator workflow
# 맞춤형 운영자 작업 흐름

Entry point: `/vai-custom-op`
진입점: `/vai-custom-op`

Implements a custom operator end to end and integrates it into your build.
사용자 정의 연산자를 엔드 투 엔드로 구현하고 이를 빌드에 통합합니다.
The
는
workflow writes the three artifacts that make up a custom operator: the AIE C++
워크플로는 사용자 정의 연산자를 구성하는 세 가지 아티팩트인 AIE C++를 작성합니다.
kernel, the Python tiler, and the YAML configuration.
커널, Python 타일러 및 YAML 구성.
It then wires them into
그런 다음 이를 연결합니다.
your compilation and runtime configurations, and validates the result in
컴파일 및 런타임 구성을 확인하고 결과를 검증합니다.
simulation and, when a board is available, on hardware.
시뮬레이션 및 보드가 있는 경우 하드웨어에서 수행됩니다.

The points in [Board access](index.md#board-access) apply to this workflow.
[보드 액세스](index.md#board-access)의 사항이 이 워크플로에 적용됩니다.
For the manual procedure
수동 절차의 경우
this workflow automates, see
이 워크플로는 자동화됩니다.
[the Vitis AI documentation](https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/additional_information/custom_ops/custom_ops_introduction.html).
[Vitis AI 문서](https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/additional_information/custom_ops/custom_ops_introduction.html).

## When to use
## 언제 사용하나요?

* Your model contains an operator the NPU does not support natively.
* 모델에 NPU가 기본적으로 지원하지 않는 연산자가 포함되어 있습니다.
* You want several ONNX-level operations fused into a single custom operator.
* 여러 ONNX 수준 작업을 단일 사용자 지정 연산자로 통합하려고 합니다.
* You are porting an operator you have already implemented elsewhere and want
* 이미 다른 곳에서 구현했고 원하는 연산자를 이식하고 있습니다.
the surrounding integration written for you.
당신을 위해 작성된 주변 통합.

You provide a working Vitis AI environment in which compilation and simulation
컴파일과 시뮬레이션이 가능한 Vitis AI 환경을 제공합니다.
succeed, together with either the model or a specification of the operator you
성공하면 운영자의 모델이나 사양과 함께
want implemented.
구현을 원합니다.
You get an AIE C++ kernel, a Python tiler, a YAML
AIE C++ 커널, Python 타일러, YAML이 제공됩니다.
configuration, and the edits to your compilation and runtime configurations
구성, 컴파일 및 런타임 구성에 대한 편집
needed to use them.
그것들을 사용하는 데 필요합니다.

## Prerequisites
## 전제조건

Follow the installation procedure in
설치 절차를 따르십시오.
[the skill installation guide](../../README.md#quick-start), so that
[스킬 설치 가이드](../../README.md#quick-start)
your assistant has model access and the workflow files installed.
귀하의 어시스턴트에는 모델 액세스 권한이 있고 워크플로 파일이 설치되어 있습니다.
Configure
구성
board access as described in [Board access](index.md#board-access).
[보드 액세스](index.md#board-access)에 설명된 대로 보드 액세스를 수행합니다.

> **Note**
> **참고**
>
> Ideally keep a board available for the whole session.
> 이상적으로는 전체 세션 동안 보드를 사용할 수 있도록 유지하십시오.
> The workflow runs on
> 워크플로는 다음에서 실행됩니다.
> hardware to validate the operator once it is integrated, and to drive its
> 통합되면 운영자를 검증하고 이를 구동하는 하드웨어
> optimization loop.
> 최적화 루프.
> Without a board it falls back on simulation, which is not
> 보드가 없으면 시뮬레이션으로 돌아갑니다.
> a one-to-one match for the hardware, so numeric mismatches are more likely
> 하드웨어에 대한 일대일 일치이므로 숫자 불일치가 발생할 가능성이 더 높습니다.
> and the operator is not tuned for performance.
> 그리고 운영자는 성능에 맞춰 조정되지 않았습니다.
> If the board becomes
> 보드가 되면
> unavailable partway through, the session continues, but everything after that
> 도중에는 사용할 수 없으며 세션은 계속되지만 그 이후에는 모두
> point is unvalidated.
> 포인트가 검증되지 않았습니다.

### Verify
### 확인

Launch your assistant:
어시스턴트를 실행하세요:

```shell
~/.local/bin/claude
```

Run `claude` directly if `~/.local/bin` is on your `PATH`.
`~/.local/bin`이 `PATH`에 있으면 `claude`를 직접 실행하세요.
Type
유형
`/vai-custom-op` and confirm the workflow appears as available.
`/vai-custom-op`을 실행하고 워크플로가 사용 가능한 것으로 나타나는지 확인합니다.

## Quick start
## 빠른 시작

Invoke the workflow with the model, the compilation configuration, and the node
모델, 컴파일 구성 및 노드를 사용하여 워크플로 호출
or nodes you want converted:
또는 변환하려는 노드:

```text
/vai-custom-op "Create an element-wise multiply custom op" \
  --model ./model.onnx \
  --vitisai-config ./vitisai_config.json \
  --ops Mul_1
```

See [Command reference](#command-reference) for the full argument list, and [Examples](#examples) for other
전체 인수 목록은 [명령 참조](#command-reference)를 참조하고 다른 인수 목록은 [예](#examples)를 참조하세요.
ways to specify what to replace.
대체할 항목을 지정하는 방법.

## How it works
## 작동 원리

The workflow proceeds through eight phases:
워크플로우는 8단계로 진행됩니다.

1. **Subgraph extraction.** Extracts the target node into an isolated subgraph,
1. **하위 그래프 추출.** 대상 노드를 격리된 하위 그래프로 추출합니다.
Model A, and generates CPU reference outputs for it.
모델 A는 이에 대한 CPU 참조 출력을 생성합니다.
2. **File creation.** Generates the kernel C++ code, the tiling Python script,
2. **파일 생성.** 커널 C++ 코드, 타일링 Python 스크립트,
the YAML configuration, and a new ONNX model, Model B, that wraps the
YAML 구성과 새로운 ONNX 모델인 Model B
subgraph as a single custom operator node.
단일 사용자 정의 연산자 노드로 하위 그래프를 작성합니다.
3. **Compilation.** Compiles the custom operator model with Vitis AI.
3. **컴파일.** Vitis AI로 맞춤형 연산자 모델을 컴파일합니다.
4. **Simulation and board validation.** Runs the compiled model on the x86
4. **시뮬레이션 및 보드 검증.** x86에서 컴파일된 모델을 실행합니다.
simulator and compares the output against the CPU reference.
시뮬레이터를 실행하고 출력을 CPU 참조와 비교합니다.
A board run
보드런
follows, to obtain correctness and performance figures.
정확성과 성능 수치를 얻으려면 다음과 같이 하십시오.
5. **Optimization.** Tunes the operator for performance, using further board
5. **최적화.** 추가 보드를 사용하여 성능에 맞게 운영자를 조정합니다.
runs as feedback.
피드백으로 실행됩니다.
6. **Re-integration.** Stitches the validated operator back into the original
6. **재통합.** 검증된 연산자를 원본에 다시 연결합니다.
model, replacing the nodes it was built from.
모델을 구축한 노드를 교체합니다.
7. **End-to-end verification.** Compiles and runs the integrated model on the
7. **엔드 투 엔드 검증.** 통합 모델을 컴파일하고 실행합니다.
board.
보드.
8. **Reporting.** Writes a structured report to `report.md`.
8. **보고.** 'report.md'에 구조화된 보고서를 작성합니다.

When you name several operators, each is developed in parallel by its own worker
여러 운영자의 이름을 지정하면 각 운영자는 자체 작업자에 의해 병렬로 개발됩니다.
agent.
대리인.
Once all of them pass compilation and numeric validation, the
모두 컴파일 및 수치 검증을 통과하면
orchestrator stitches them into the model one at a time and verifies the result.
오케스트레이터는 이를 한 번에 하나씩 모델에 연결하고 결과를 확인합니다.

## Command reference
## 명령 참조

```text
/vai-custom-op [prompt] \
  --model <path_to_onnx_model> \
  --vitisai-config <path_to_vitisai_config.json> \
  --ops <nodes:op_name> [<nodes:op_name> ...]
```

| Argument<br>인수 | Required<br>필수 | Description<br>설명 |
| --- | --- | --- |
| `--model <path>`<br>`--model <경로>` | Yes<br>예 | Path to the full ONNX model.<br>전체 ONNX 모델에 대한 경로입니다. |
| `--vitisai-config <path>`<br>`--vitsai-config <경로>` | Yes<br>예 | Path to the `vitisai_config.json` holding your compilation and runtime options.<br>컴파일 및 런타임 옵션이 포함된 `vitsai_config.json`의 경로입니다. |
| `--ops <nodes:name> [...]`<br>`--ops <노드:이름> [...]` | Yes<br>예 | One or more operator specifications, in `node1,node2:op_name` format: a comma-separated list of ONNX node names, a colon, then the name to give the resulting custom operator. Separate multiple specifications with spaces.<br>'node1,node2:op_name' 형식의 하나 이상의 연산자 사양: 쉼표로 구분된 ONNX 노드 이름 목록, 콜론, 결과 사용자 지정 연산자를 제공하는 이름입니다. 여러 사양을 공백으로 구분하세요. |

You can give a free-form prompt instead of `--ops`, or alongside it, when the
`--ops` 대신 또는 그와 함께 자유 형식 프롬프트를 제공할 수 있습니다.
nodes you want replaced are easier to describe than to list.
교체하려는 노드는 나열하는 것보다 설명하는 것이 더 쉽습니다.

## Examples
## 예

### One node
### 노드 1개

Convert the ONNX node `/layer_0/Mul_1` into a custom operator named `mymul`:
ONNX 노드 `/layer_0/Mul_1`을 `mymul`이라는 사용자 정의 연산자로 변환합니다.

```text
/vai-custom-op "Create an element-wise multiply custom op" \
  --model ./model.onnx \
  --vitisai-config ./vitisai_config.json \
  --ops /layer_0/Mul_1:mymul
```

### Several nodes
### 여러 노드

Convert `/layer_0/Mul_1` and `/layer_2/AffineGrid` into custom operators
`/layer_0/Mul_1` 및 `/layer_2/AffineGrid`를 사용자 정의 연산자로 변환
named `mymul` and `myaffinegrid`.
'mymul'과 'myaffinegrid'로 명명되었습니다.
Each is developed in parallel by its own
각각은 자체적으로 병렬로 개발됩니다.
worker agent:
작업자 에이전트:

```text
/vai-custom-op \
  --model ./model.onnx \
  --vitisai-config ./vitisai_config.json \
  --ops /layer_0/Mul_1:mymul /layer_2/AffineGrid:myaffinegrid
```

### Every instance of an operator
### 연산자의 모든 인스턴스

If the nodes are easier to describe than to list, use a free-form prompt:
노드를 나열하는 것보다 설명하는 것이 더 쉬운 경우 자유 형식 프롬프트를 사용하십시오.

```text
/vai-custom-op \
  --model ./model.onnx \
  --vitisai-config ./vitisai_config.json \
  Replace all instances of `onnx.AffineGrid` by a custom op named
  `mydomain.myaffinegrid`.
```

### A subgraph
### 하위 그래프

Name the boundaries and state whether each is included:
경계의 이름을 지정하고 각 경계가 포함되는지 여부를 명시하십시오.

```text
/vai-custom-op \
  --model ./model.onnx \
  --vitisai-config ./vitisai_config.json \
  Replace all ops between `Conv_7` (excluded) and `Softmax_2` (included) by a
  custom op.
```

### A specific optimization
### 특정 최적화

The workflow optimizes for performance by default, using a tuning loop driven by
워크플로는 기본적으로 다음에 의해 구동되는 튜닝 루프를 사용하여 성능을 최적화합니다.
board measurements.
보드 측정.
To direct that effort, say what you want.
그러한 노력을 지시하려면 원하는 것을 말하십시오.
Absorbing the data
데이터 흡수
layout transformations caused by transposes around an operator is a common case:
연산자 주변의 전치로 인한 레이아웃 변환은 일반적인 경우입니다.

```text
/vai-custom-op \
  --model ./model.onnx \
  --vitisai-config ./vitisai_config.json \
  Replace `Softmax_2` by a custom op that absorbs the transpositions that
  surround it.
```

## Iterate
## 반복

Once the basic flow works, refine the operator with follow-up prompts in the
기본 흐름이 작동하면 다음의 후속 프롬프트로 연산자를 구체화합니다.
same session:
동일한 세션:

```text
Add tiling so this custom op supports tensors larger than L1 memory.
```

```text
Distribute the computation across all 16 AIE cores using the 4x4 overlay.
```

```text
Optimize the kernel with vectorization using aie::vector intrinsics.
```

Each of these has a worked example bundled with the product, under the
이들 각각에는 제품과 함께 번들로 제공되는 작업 예제가 있습니다.
`vai-custom-op-implementation` tutorial directory: `mul/1_tiled` for tiling,
`vai-custom-op-implementation` 튜토리얼 디렉토리: 타일링을 위한 `mul/1_tiled`,
`mul/2_distributed` for distribution, and
배포용 `mul/2_distributed`
`negate/3_distributed_and_vectorized` for vectorization.
벡터화를 위한 `negate/3_distributed_and_Vectorized`.
See the
참조
[Bundled examples and utilities](https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/additional_information/custom_ops/custom_ops_introduction.html#bundled-examples-and-utilities)
[번들 예제 및 유틸리티](https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/additional_information/custom_ops/custom_ops_introduction.html#bundled-examples-and-utilities)
section of the Vitis AI documentation for more details.
자세한 내용은 Vitis AI 문서 섹션을 참조하세요.

## Board runs
## 보드런

The workflow runs models on the board as part of developing an operator, but you
워크플로는 연산자 개발의 일부로 보드에서 모델을 실행하지만
can also ask it to run one directly, without creating a custom operator.
사용자 정의 연산자를 만들지 않고도 직접 실행하도록 요청할 수도 있습니다.
With
와
board access configured as described in [Board access](index.md#board-access):
[보드 액세스](index.md#board-access)에 설명된 대로 구성된 보드 액세스:

```text
Run the model ./model.onnx on my board using configuration
./vitisai_config.json using the /vai-custom-op skill.
```

Add input files, if you have them:
입력 파일이 있는 경우 추가합니다.

```text
The inputs are located in ./model_inputs.
```

And point at an existing build, if the model is already compiled:
모델이 이미 컴파일된 경우 기존 빌드를 가리킵니다.

```text
The model is already compiled, the compiled artifacts are in ./mymodel_cache.
```

## Session length
## 세션 길이

Sessions can be long, and the time is dominated by compilation, simulation, and
세션은 길어질 수 있으며 시간은 컴파일, 시뮬레이션 및
board iterations rather than by the assistant's reasoning.
보조자의 추론보다는 보드 반복.
A single optimized
최적화된 싱글
element-wise operator, such as a multiply or a divide, sits at the short end.
곱셈이나 나눗셈과 같은 요소별 연산자는 짧은 끝에 위치합니다.
Fusing several ONNX-level operations into one custom operator takes considerably
여러 ONNX 수준 작업을 하나의 사용자 지정 연산자로 융합하는 데는 상당한 시간이 걸립니다.
longer.
더 이상.
Start with a simple operator to calibrate what your setup costs before
간단한 운영자로 시작하여 이전에 설정 비용을 교정하세요.
attempting a fused one.
융합을 시도하고 있습니다.

## Limitations
## 제한사항

### Data layout
### 데이터 레이아웃

The workflow knows about the input and output data layout requirements that
워크플로는 입력 및 출력 데이터 레이아웃 요구 사항을 알고 있습니다.
compilation imposes on the operator, but does not always account for them.
컴파일은 연산자에 부과되지만 항상 설명되는 것은 아닙니다.
The
는
result is extra transpose or pad operations that can outweigh any gain from the
그 결과로 얻는 이득보다 더 클 수 있는 추가 조옮김 또는 패드 작업이 발생합니다.
operator itself.
운영자 그 자체.
Check the generated graph.
생성된 그래프를 확인하세요.
If you see them:
당신이 그들을 본다면 :

```text
Absorb the Transposes around the generated custom op.
```

### Repeated operators
### 반복 연산자

Modifying several instances of the same custom operator can fail.
동일한 사용자 정의 연산자의 여러 인스턴스를 수정하면 실패할 수 있습니다.
The workflow
워크플로우
usually recovers by renaming them, but not always.
일반적으로 이름을 바꾸면 복구되지만 항상 그런 것은 아닙니다.
If it does not:
그렇지 않은 경우:

```text
Rename each instance of the custom op with its own name.
```

### Accuracy bias
### 정확도 편향

The workflow prefers a functionally correct operator to a faster one that
작업흐름은 더 빠른 연산자보다 기능적으로 올바른 연산자를 선호합니다.
changes the behavior of the original model.
원본 모델의 동작을 변경합니다.
If you can accept a looser
더 느슨한 것을 받아들일 수 있다면
tolerance, say so:
관용, 이렇게 말해보세요:

```text
Accept inaccurate outputs with a relative tolerance of 5%.
```

### Complex operators
### 복합 연산자

Aggressively fused operators, such as a fully fused multi-head attention, may
완전히 융합된 다중 헤드 어텐션과 같이 공격적으로 융합된 연산자는
not converge.
수렴하지 않음.
These need several sub-kernels, or round trips through scratch
여기에는 여러 하위 커널이 필요하거나 처음부터 왕복이 필요합니다.
buffers at different memory levels, and the decomposition is beyond what the
다양한 메모리 수준에서 버퍼링되며 분해는
workflow will find on its own.
워크플로가 자체적으로 검색됩니다.
Agree on a decomposition first, then ask it to
먼저 분해에 동의한 다음 요청하세요.
implement that:
다음을 구현하십시오.

```text
Implement the GroupQueryAttention_1 op as a custom op. Please look at
the mathematical expression of the GQA op and suggest a decomposition of this
op; conceive the custom op combining each of the individual ops you
decomposed.
```

### Other
### 기타

* Tiling implementations can fail unexpectedly in combination with aggressive
* 공격적인 타일링 구현과 함께 타일링 구현이 예기치 않게 실패할 수 있습니다.
optimizations, such as optimization level O3.
최적화 수준 O3과 같은 최적화.
* The workflow does not have precise instructions for building MX6 or MX9 custom
* 워크플로에는 MX6 또는 MX9 커스텀 빌드에 대한 정확한 지침이 없습니다.
operators.
연산자.
* Without a board, numeric mismatches and reduced performance are considerably
* 보드가 없으면 수치 불일치와 성능 저하가 상당히 심합니다.
more likely, because the result is validated only in simulation.
결과는 시뮬레이션에서만 검증되기 때문에 가능성이 더 높습니다.

## Implementation
## 구현

You do not need this to use the workflow.
워크플로를 사용하는 데는 이것이 필요하지 않습니다.
It is here for reference when
그럴 때 참고용으로 여기에 있습니다.
troubleshooting an installation.
설치 문제 해결.

The custom operator workflow consists of two skills and one agent:
사용자 정의 운영자 워크플로는 두 가지 기술과 하나의 에이전트로 구성됩니다.

| Component<br>구성 요소 | Type<br>유형 | Purpose<br>목적 |
| --- | --- | --- |
| `vai-custom-op`<br>`vai-custom-op` | Skill<br>스킬 | Entry point. Orchestrates several custom operators in parallel, using worker agents.<br>진입점. 작업자 에이전트를 사용하여 여러 사용자 지정 연산자를 병렬로 조정합니다. |
| `vai-custom-op-implementation`<br>`vai-custom-op-구현` | Skill<br>스킬 | End-to-end development of a single custom AIE operator. Invoked by `vai-custom-op`.<br>단일 맞춤형 AIE 연산자의 엔드투엔드 개발. 'vai-custom-op'에 의해 호출됩니다. |
| `vai-custom-op-worker`<br>`vai-custom-op-worker` | Agent<br>대리인 | Constrained worker spawned by the orchestrator, one per operator.<br>오케스트레이터에 의해 운영자당 하나씩 생성되는 제한된 작업자입니다. |

```text
ai_utils/
  skills/
    vai-custom-op/                # Entry point, invoked as /vai-custom-op
      SKILL.md                    # Orchestrator workflow definition
    vai-custom-op-implementation/ # Single-operator implementation skill
      README.md                   # AIE architecture reference
      tiling.md                   # Tiling technical reference
      scripts/                    # Compile, simulate, cut and stitch ONNX
      tutorial/                   # mul, negate, topk, softmax, and others
  agents/
    vai-custom-op-worker.md       # Worker agent definition
```

Only `/vai-custom-op` is a user entry point.
`/vai-custom-op`만이 사용자 진입점입니다.
The implementation skill and the
구현 기술과
worker agent are invoked on your behalf and should not be called directly.
작업자 에이전트는 사용자를 대신하여 호출되므로 직접 호출하면 안 됩니다.
