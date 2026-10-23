<!-- ko-translation: enabled -->
# Compiler options workflow
# 컴파일러 옵션 작업 흐름

Entry point: `/vai-flag-configuration`
진입점: `/vai-flag-configuration`

Iteratively improves the compilation configuration for an ONNX model.
ONNX 모델의 컴파일 구성을 반복적으로 개선합니다.
The
는
workflow analyzes your model, proposes a set of compiler options,
워크플로는 모델을 분석하고 컴파일러 옵션 세트를 제안합니다.
compiles with them, measures how many of the model's operators were placed on
그것들을 컴파일하고 모델의 연산자 중 몇 개가 배치되었는지 측정합니다.
the NPU rather than the CPU, measures inference time when a board is available,
CPU가 아닌 NPU가 보드를 사용할 수 있을 때 추론 시간을 측정하고,
and repeats until your stop condition is met.
중지 조건이 충족될 때까지 반복됩니다.
It never modifies the model.
모델을 수정하지 않습니다.

The points in [Board access](index.md#board-access) apply to this workflow.
[보드 액세스](index.md#board-access)의 사항이 이 워크플로에 적용됩니다.
For the manual procedure this workflow automates, see
이 워크플로가 자동화하는 수동 절차는 다음을 참조하세요.
[the Vitis AI documentation](https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/model_compilation/ep-options.html).
[Vitis AI 문서](https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/model_compilation/ep-options.html).

## When to use
## 언제 사용하나요?

* You want more of the model's operators offloaded to the NPU, rather than
* 당신은 NPU에 더 많은 모델 연산자를 오프로드하기를 원합니다.
falling back to the CPU.
CPU로 다시 떨어집니다.
* You want better inference time from compilation changes alone.
* 컴파일 변경만으로 더 나은 추론 시간을 원합니다.
* A model fails to compile and you want it to compile successfully.
* 모델이 컴파일에 실패했는데 성공적으로 컴파일되기를 원합니다.
* This counts as
* 이는 다음과 같이 계산됩니다.
success even if part of the model stays on the CPU; compiling and reaching
모델의 일부가 CPU에 남아 있어도 성공합니다. 컴파일 및 도달
full offload are separate goals.
전체 오프로드는 별도의 목표입니다.

You provide an ONNX model and your goals.
ONNX 모델과 목표를 제공합니다.
You get a `vitisai_config.json`,
`vitsai_config.json`이 표시됩니다.
along with the measured offload and, where a board was used, the board inference
측정된 오프로드 및 보드가 사용된 경우 보드 추론과 함께
time for every configuration tried.
시도된 모든 구성에 소요되는 시간입니다.

## Prerequisites
## 전제조건

Follow the installation procedure in
설치 절차를 따르십시오.
[the skill installation guide](../../README.md#quick-start), so that
[스킬 설치 가이드](../../README.md#quick-start)
your assistant has model access and the workflow files installed.
귀하의 어시스턴트에는 모델 액세스 권한이 있고 워크플로 파일이 설치되어 있습니다.

Board access is needed only if inference time is one of your goals.
추론 시간이 목표 중 하나인 경우에만 보드 액세스가 필요합니다.
Configure it
구성하세요
as described in [Board access](index.md#board-access).
[보드 액세스](index.md#board-access)에 설명된 대로입니다.
Without a board, the workflow scores
보드가 없으면 워크플로 점수가 매겨집니다.
compile success and offload from a host compile, and will not offer to make the
컴파일이 성공하고 호스트 컴파일에서 오프로드되며
model faster, because it has no measured inference time to work from.
작업할 측정된 추론 시간이 없기 때문에 모델을 더 빠르게 만듭니다.

## Quick start
## 빠른 시작

State the model, your goal, and how much autonomy you want.
모델, 목표, 원하는 자율성을 명시하세요.
The workflow asks
워크플로우가 묻습니다.
about anything you leave out:
당신이 빠뜨린 모든 것에 대해:

```text
/vai-flag-configuration Run autonomously on ./model.onnx targeting
ve2-xc2ve3858. Maximize NPU offload. Stop after 2 iterations without
improvement.
```

See [Examples](#examples) for fuller requests, and [Command reference](#command-reference) for the arguments.
더 자세한 요청은 [예](#examples)를, 인수는 [명령 참조](#command-reference)를 참조하세요.

## How it works
## 작동 원리

### Startup interview
### 스타트업 인터뷰

Before any analysis, the workflow settles five questions.
분석에 앞서 워크플로는 다섯 가지 질문을 해결합니다.
It resolves what it
그것은 무엇을 해결
can from your request, echoes back what it understood, and asks only about what
귀하의 요청에서 이해한 내용을 다시 반향하고 무엇에 대해서만 묻습니다.
is genuinely missing.
진짜 없어졌네요.

**Quantize the model first?** If you say yes, the workflow hands your model to
**모델을 먼저 양자화하시겠습니까?** 그렇다고 대답하면 워크플로에서 모델을 양자화합니다.
[Model quantization workflow](vai-quantization-guide.md) and continues once that returns.
[모델 양자화 워크플로](vai-퀀트화-guide.md) 반환되면 계속됩니다.
See
참조
[Handoffs](#handoffs).
[핸드오프](#핸드오프).

**Objectives.** One or more of: compile successfully, improve NPU offload,
**목표.** 다음 중 하나 이상: 성공적으로 컴파일, NPU 오프로드 개선,
improve inference time.
추론 시간을 향상시킵니다.
You give them in priority order.
우선순위대로 주십니다.
Inference time is
추론 시간은
offered only when a board is available.
보드가 있는 경우에만 제공됩니다.

**Mode.** Autonomous or human-in-the-loop.
**모드.** 자율 또는 인간 참여형.
See [Operating modes](#operating-modes).
[작동 모드](#operating-modes)를 참조하세요.

**Stop conditions.** See [Stop conditions](#stop-conditions).
**중지 조건.** [중지 조건](#stop-condition)을 참조하세요.

**Compile depth.** See [Compile depth](#compile-depth).
**컴파일 깊이.** [컴파일 깊이](#compile-length)를 참조하세요.

If inference time is one of your objectives, the workflow verifies that a board
추론 시간이 목표 중 하나인 경우 워크플로는 보드가
run actually works before it begins iterating.
run은 반복을 시작하기 전에 실제로 작동합니다.
If board access is not
보드 접근이 불가능한 경우
configured, it stops and asks how you run inference on your board.
구성된 경우 중지되고 보드에서 추론을 실행하는 방법을 묻습니다.
It will not
그렇지 않을 것이다
begin a latency-goal session it cannot measure.
측정할 수 없는 대기 시간 목표 세션을 시작합니다.
If you cannot provide board
보드를 제공할 수 없는 경우
access, it offers to switch to a compile or offload goal instead.
대신 컴파일 또는 오프로드 목표로 전환하도록 제안합니다.

### Pre-compilation analysis
### 사전 컴파일 분석

The workflow then analyzes your model and presents a report before compiling
그런 다음 워크플로는 모델을 분석하고 컴파일하기 전에 보고서를 제시합니다.
anything.
무엇이든.
The report covers:
보고서는 다음을 다루고 있습니다:

* The operator-type census, that is how many Conv, Gemm, MatMul, Softmax, and
* 연산자 유형 인구 조사, 즉 Conv, Gemm, MatMul, Softmax 및
other operators the model contains.
모델에 포함된 다른 연산자.
* Input and output shapes, and the largest layers by tensor size.
* 입력 및 출력 모양과 텐서 크기별 최대 레이어.
* Any custom operators the model uses.
* 모델이 사용하는 모든 사용자 정의 연산자.
* The compiler options it proposes for the first iteration, each with a source
* 첫 번째 반복에 대해 제안하는 컴파일러 옵션(각각 소스 포함)
reference and the reason it was chosen.
참고자료와 선택한 이유.

This report is a gate.
이 보고서는 게이트입니다.
Nothing compiles until it has been shown to you.
사용자에게 표시될 때까지 아무것도 컴파일되지 않습니다.
In
human-in-the-loop mode, the workflow waits for you to choose between a baseline
Human-In-The-Loop 모드에서는 워크플로가 기준선 중에서 선택할 때까지 기다립니다.
compile and a first optimized compile.
컴파일하고 첫 번째 최적화된 컴파일을 수행합니다.
In autonomous mode, it prints the report,
자율 모드에서는 보고서를 인쇄합니다.
states which compile it is starting and why, and proceeds.
컴파일이 시작되는 이유와 이유를 설명하고 진행합니다.

### The iteration loop
### 반복 루프

Each iteration runs three phases:
각 반복은 세 단계로 실행됩니다.

1. **Propose.** Selects non-default options not already tried, grounded in
1. **제안.** 아직 시도되지 않은 기본 옵션이 아닌 옵션을 선택합니다.
evidence from your model or the previous compile.
모델이나 이전 컴파일의 증거.
Proposals are limited to
제안은 다음으로 제한됩니다.
three added or changed options relative to the current best configuration,
현재 최상의 구성과 관련하여 추가되거나 변경된 세 가지 옵션,
which keeps the search interpretable.
검색을 해석 가능하게 유지합니다.
Widening that budget requires your
예산을 확대하려면 다음이 필요합니다.
consent.
동의.
2. **Compile.** Compiles with the proposed configuration, at the chosen depth.
2. **컴파일.** 선택한 깊이에서 제안된 구성으로 컴파일합니다.
3. **Validate.** Measures offload and, where a board is available, inference
3. **검증.** 오프로드를 측정하고 보드가 있는 경우 추론을 수행합니다.
time.
시간.
Records the result and checks the stop conditions.
결과를 기록하고 정지 조건을 확인합니다.

Every proposal cites a source you can open and states why that option was chosen
모든 제안에는 공개할 수 있는 소스를 인용하고 해당 옵션을 선택한 이유를 설명합니다.
for your model.
당신의 모델을 위해.
If you ask why a particular option was never tried, the workflow
특정 옵션을 시도하지 않은 이유를 묻는다면 워크플로는
answers from its own records rather than saying it was not considered.
고려되지 않았다는 것이 아니라 자체 기록에서 대답합니다.

### Operating modes
### 작동 모드

**Autonomous** proceeds without asking which options to keep, and runs until a
**자율**은 어떤 옵션을 유지할지 묻지 않고 진행되며,
stop condition is met.
중지 조건이 충족되었습니다.
Ask for this with a phrase such as "run autonomously" or
"자율적으로 실행하세요"와 같은 문구로 요청하거나
"compile the model fully automatically."
"완전 자동으로 모델을 컴파일합니다."

**Human-in-the-loop** presents each proposed set as a table, showing each option,
**Human-in-the-loop**는 제안된 각 세트를 표로 제시하고 각 옵션을 보여줍니다.
its source, the default it deviates from, the chosen value, and the rationale,
소스, 기본값에서 벗어난 값, 선택한 값 및 근거,
then waits for your approval before compiling.
그런 다음 컴파일하기 전에 승인을 기다립니다.
Ask for this with a phrase such
이런 문구로 물어보세요
as "run with human-in-the-loop."
"Human-In-The-Loop로 실행"으로.

Options that trade accuracy, or that change the deployment or runtime contract,
정확성을 교환하거나 배포 또는 런타임 계약을 변경하는 옵션,
are never applied in autonomous mode.
자율 모드에서는 적용되지 않습니다.
In human-in-the-loop mode they are offered
Human-In-The-Loop 모드에서는 다음이 제공됩니다.
with the trade-off stated.
트레이드 오프가 명시되어 있습니다.

### Compile depth
### 컴파일 깊이

Each iteration compiles at one of two depths.
각 반복은 두 가지 깊이 중 하나에서 컴파일됩니다.

**Fast** probes the front end and partitioning only.
**빠름**은 프런트 엔드와 파티셔닝만 조사합니다.
It answers whether offload
오프로드 여부에 대해 답변합니다.
improves with a given set of options, and reports unsupported operators and the
주어진 옵션 세트로 개선되고 지원되지 않는 연산자와
partition split.
파티션 분할.
It produces no NPU binary, no board-runnable artifacts, and no
NPU 바이너리도 생성하지 않고, 보드에서 실행 가능한 아티팩트도 생성하지 않습니다.
memory analysis.
기억 분석.

**Complete** runs the full compile.
**Complete**는 전체 컴파일을 실행합니다.
It is significantly slower, and it is
상당히 느리고,
required before any board timing, before the memory analysis, and for a final
보드 타이밍 전, 메모리 분석 전, 최종
offload figure you can rely on.
신뢰할 수 있는 오프로드 수치.

The workflow uses fast depth while exploring and switches to complete before
워크플로는 탐색하는 동안 빠른 깊이를 사용하고 이전에 완료하도록 전환합니다.
claiming a board latency result or a final offload number.
보드 지연 결과 또는 최종 오프로드 수를 주장합니다.
You can override
재정의할 수 있습니다
this, in the interview or in your prompt.
인터뷰나 프롬프트에서요.

### Stop conditions
### 정지 조건

The workflow stops when any of the following is true:
다음 중 하나라도 해당되면 워크플로가 중지됩니다.

* Offload reaches the target percentage you set.
* 오프로드가 설정한 목표 비율에 도달했습니다.
* A number of consecutive iterations bring no improvement.
* 여러 번 연속해서 반복해도 개선되지 않습니다.
* The default is two.
* 기본값은 2개입니다.
* No untried compiler options remain.
* 시도하지 않은 컴파일러 옵션은 남아 있지 않습니다.
* You tell it to stop.
* 그만하라고 말해요.
* Nothing compiles.
* 아무것도 컴파일되지 않습니다.
* See [No viable configuration](#no-viable-configuration)
* [실행 가능한 구성 없음](#no-viable-configuration)을 참조하세요.

When it stops, it states which condition fired and the numbers behind it.
중지되면 어떤 조건이 실행되었는지와 그 뒤에 있는 숫자가 표시됩니다.

## Command reference
## 명령 참조

```text
/vai-flag-configuration [prompt] \
  --model <path_to_onnx_model> \
  [--device <device-id>] \
  [--vitisai-config <path>] \
  [--board] \
  [--work-dir <path>] \
  [--spill-workdir <path>]
```

| Argument<br>인수 | Required<br>필수 | Default<br>기본값 | Description<br>설명 |
| --- | --- | --- | --- |
| `--model <path>`<br>`--model <경로>` | Yes<br>예 | none<br>없음 | ONNX model to optimize.<br>최적화할 ONNX 모델입니다. |
| `--device <device-id>`<br>`--device <장치 ID>` | No | none<br>없음 | Target device, such as `ve2-xc2ve3858`. If omitted, the workflow takes it from `--vitisai-config`, or asks. It never assumes a device. See [Model Compilation](https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/model_compilation/compiling.html).<br>'ve2-xc2ve3858'과 같은 대상 장치입니다. 생략하면 워크플로는 `--vitsai-config`에서 이를 가져오거나 묻습니다. 결코 장치를 가정하지 않습니다. [모델 편집](https://vitsai.docs.amd.com/projects/gen2/en/latest/docs/model_compilation/compiling.html)을 참조하세요. |
| `--vitisai-config <path>`<br>`--vitsai-config <경로>` | No | generated<br>생성됨 | An existing configuration to start from. See [Existing configuration](#existing-configuration).<br>시작할 기존 구성입니다. [기존 구성](#existing-configuration)을 참조하세요. |
| `--board`<br>`--보드` | No | off<br>떨어져서 | Declares that a board run is available, which enables the inference time objective. Takes no value; configure the board connection as described in [Board access](index.md#board-access).<br>추론 시간 목표를 활성화하는 보드 실행이 가능함을 선언합니다. 가치가 없습니다. [보드 액세스](index.md#board-access)에 설명된 대로 보드 연결을 구성합니다. |
| `--work-dir <path>`<br>`--work-dir <경로>` | No | `./flag_opt_work`<br>`./flag_opt_work` | Where iteration artifacts and the run record are written.<br>반복 아티팩트와 실행 기록이 기록되는 위치입니다. |
| `--spill-workdir <path>`<br>`--spill-workdir <경로>` | No | none<br>없음 | Output directory from an earlier complete compile. The workflow reads the memory analysis in it, rather than producing its own, to judge whether an option change is likely to help latency.<br>이전 전체 컴파일의 출력 디렉터리입니다. 워크플로는 자체적으로 생성하는 대신 메모리 분석을 읽어 옵션 변경이 대기 시간에 도움이 될지 여부를 판단합니다. |

> **Note**
> **참고**
>
> Objectives, operating mode, stop conditions, and compile depth are not
> 목표, 작동 모드, 정지 조건 및 컴파일 깊이는 포함되지 않습니다.
> command-line arguments.
> 명령줄 인수.
> The workflow settles them in the startup interview,
> 워크플로는 스타트업 인터뷰에서 이를 해결하고,
> and you can state any of them in your prompt.
> 프롬프트에 그 중 하나를 명시할 수 있습니다.

> **Tip**
> **팁**
>
> You do not have to work out these paths yourself.
> 이러한 경로를 직접 해결할 필요는 없습니다.
> Point the workflow at the
> 워크플로를 가리킵니다.
> working directory of an earlier compilation and it locates what it needs.
> 이전 컴파일의 작업 디렉터리를 검색하고 필요한 항목을 찾습니다.

### Existing configuration
### 기존 구성

When you supply `--vitisai-config`, every option already set in that file is
`--vitsai-config`를 제공하면 해당 파일에 이미 설정된 모든 옵션은
treated as fixed.
고정된 것으로 취급됩니다.
The workflow carries them forward into every iteration and
워크플로우는 이를 모든 반복에 전달하고
only adds or adjusts options on top.
상단에 옵션만 추가하거나 조정합니다.
It never drops, resets, or silently omits a
절대로 삭제되거나 재설정되거나 자동으로 생략되지 않습니다.
value you provided unless you explicitly ask it to.
명시적으로 요청하지 않는 한 제공한 값입니다.

Your original file is left untouched.
원본 파일은 그대로 유지됩니다.
Each iteration writes a new configuration
각 반복은 새로운 구성을 작성합니다.
into its own directory under the working directory.
작업 디렉토리 아래의 자체 디렉토리에 넣습니다.

## Scope
## 범위

Only a defined set of supported compiler options are explored.
지원되는 컴파일러 옵션의 정의된 세트만 탐색됩니다.

These configuration fields are tuned:
다음 구성 필드가 조정되었습니다.

| Field<br>필드 | Default<br>기본값 | Explored<br>탐험됨 |
| --- | --- | --- |
| `device`<br>`장치` | none<br>없음 | Never. Set once at startup and held fixed.<br>결코. 시작 시 한 번 설정하고 고정된 상태로 유지됩니다. |
| `keep_outputs`<br>`출력 유지` | false<br>거짓 | Never. Forced to `true` unless you opt out, because the offload and memory analysis depend on the artifacts it retains.<br>결코. 오프로드 및 메모리 분석은 유지하는 아티팩트에 따라 달라지므로 선택하지 않는 한 강제로 'true'로 설정됩니다. |
| `optimize_level`<br>`최적화_수준` | 2 | Yes. Level 3 applies more aggressive latency optimizations, after parallelism tuning.<br>그렇습니다. 수준 3은 병렬 처리 조정 후 보다 공격적인 대기 시간 최적화를 적용합니다. |
| `threshold_gops_percent`<br>`임계값_gops_퍼센트` | 20 | Yes, for an offload goal. Lowering it places more operators on the NPU.<br>예, 오프로드 목표입니다. 이를 낮추면 NPU에 더 많은 운영자가 배치됩니다. |
| `dp_size`<br>`dp_크기` | 1 | Yes, for a latency goal.<br>예, 지연 시간 목표입니다. |
| `tp_size`<br>`tp_size` | 0 | Yes, for a latency goal.<br>예, 지연 시간 목표입니다. |
| `preferred_data_storage`<br>`preferred_data_storage` | auto<br>자동 | Yes, when convolution or pooling operators dominate the model.<br>예, 컨볼루션 또는 풀링 연산자가 모델을 지배하는 경우입니다. |

For the meaning and permitted values of each, see
각각의 의미와 허용되는 값은 다음을 참조하세요.
[Vitis AI EP Configuration File](https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/model_compilation/ep-config-file.html).
[Vitis AI EP 구성 파일](https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/model_compilation/ep-config-file.html).

Alongside these, the workflow explores front-end compiler options(fe_args).
이와 함께 워크플로는 프런트엔드 컴파일러 옵션(fe_args)을 탐색합니다.
These are a
이들은
larger set than the configuration fields above, and the skill selects from them based
위의 구성 필드보다 더 큰 세트이며 스킬은 이를 기반으로 해당 필드에서 선택합니다.
on your model and the compile evidence.
귀하의 모델과 컴파일 증거에 대해.
For each option it
각 옵션마다
proposes, it states the option, the default it deviates from, the value chosen,
제안하고 옵션, 기본값에서 벗어나는 값, 선택한 값을 설명합니다.
and why that option applies to your model.
해당 옵션이 모델에 적용되는 이유를 알아보세요.

## Examples
## 예

The first three examples run autonomously.
처음 세 가지 예제는 자동으로 실행됩니다.
To review each iteration instead,
대신 각 반복을 검토하려면
replace "run autonomously" with "run with human-in-the-loop."
"자율적으로 실행"을 "인간이 참여하는 루프로 실행"으로 바꿉니다.

### Maximize offload and minimize inference time
### 오프로드 최대화 및 추론 시간 최소화

```text
/vai-flag-configuration Run autonomously. Maximize NPU offload, then minimize
inference time on board. If the board is unreachable, report the compile
results and say so. Post the path to the report containing the compile and 
board results. Write the winning flags to vitisai_config.json.
```

### Compile for maximum offload
### 최대 오프로드를 위한 컴파일

Compile only, with no board run:
보드 실행 없이 컴파일만 수행:

```text
/vai-flag-configuration Run autonomously. Maximize NPU offload. Compile only, 
using capability mode for faster iteration, validate with full compilation. 
Write the winning flags to vitisai_config.json. Post the path to the report, 
containing the compilation results.
```

### Compile, and measure on board
### 선상에서 컴파일 및 측정

```text
/vai-flag-configuration Run autonomously. Maximize NPU offload and minimize 
inference time on board. If the board is unreachable, report the compile results 
and say so. Write the winning options to vitisai_config.json.
```

### Reviewing each iteration
### 각 반복 검토

The same request, pausing for your approval before each compile:
각 컴파일 전에 승인을 위해 일시 중지되는 동일한 요청:

```text
/vai-flag-configuration Run with human-in-the-loop. Maximize NPU offload, then
minimize inference time on board. If the board is unreachable, report the compile 
results and say so. Post the path to the report containing the compile and board results. 
```

## Output
## 출력

Everything is written under the working directory:
모든 것은 작업 디렉토리 아래에 기록됩니다:

| Path<br>경로 | Contents<br>목차 |
| --- | --- |
| `best_vitisai_config.json`<br>`best_vitsai_config.json` | The best configuration found. Absent if nothing compiled.<br>최적의 구성을 찾았습니다. 아무것도 컴파일되지 않으면 존재하지 않습니다. |
| `optimization_summary.json`<br>`최적화_요약.json` | Machine-readable summary of the run, written even when no configuration compiled, so a failed run still yields structured output of every attempt.<br>구성이 컴파일되지 않은 경우에도 기록된 기계 판독 가능한 실행 요약이므로 실패한 실행은 여전히 모든 시도의 구조화된 출력을 생성합니다. |
| `optimization_ledger.json`<br>`최적화_ledger.json` | The full record of every configuration tried, which also prevents repeats.<br>시도한 모든 구성에 대한 전체 기록이 있어 반복도 방지됩니다. |
| `iteration_<n>/`<br>`반복_<n>/` | One directory per iteration, holding the configuration that was compiled, the compile result, and a self-describing `result.json` covering the proposal, the measurements, and the stop metrics the decision rested on.<br>반복당 하나의 디렉터리에는 컴파일된 구성, 컴파일 결과, 제안서, 측정값, 결정의 기반이 되는 중지 측정항목을 포함하는 자체 설명적 'result.json'이 포함됩니다. |

The final report gives a per-objective verdict of achieved, partially achieved,
최종 보고서는 목표별로 달성, 부분 달성,
or not achieved, each with the measured figure behind it, along with the reason
달성하지 못했거나 달성하지 못한 경우, 각각의 측정 수치와 그 이유가 함께 표시됩니다.
the loop stopped.
루프가 중지되었습니다.

## Handoffs
## 핸드오프

> **Warning**
> **경고**
>
> The handoffs below are experimental.
> 아래의 핸드오프는 실험적입니다.
> Each requires your explicit consent,
> 각각은 귀하의 명시적인 동의가 필요합니다.
> opens a separate and longer session, and is not a normal iteration.
> 별도의 더 긴 세션을 열며 일반적인 반복이 아닙니다.
> The
> 는
> workflow tells you when it is suggesting one, and you decide.
> 워크플로에서 제안 사항을 알려주면 사용자가 결정합니다.

**Quantization.** Offered at the start of a session, and again if compiler
**양자화.** 세션 시작 시 제공되며, 컴파일러인 경우 다시 제공됩니다.
options alone cannot reach your goal.
옵션만으로는 목표를 달성할 수 없습니다.
Control passes to
제어권은 다음으로 전달됩니다.
[Model quantization workflow](vai-quantization-guide.md), and the loop resumes on the model it
[모델 양자화 작업 흐름](vai-퀀트화-guide.md), 모델에서 루프가 다시 시작됩니다.
returns.
반환합니다.

**Custom operators.** Offered when operators remain on the CPU that no available
**사용자 정의 연산자.** 연산자가 CPU에 남아 있을 때 제공됩니다.
option can move, citing each remaining operator, its node count, and the
옵션은 남은 각 연산자, 해당 노드 수 및
compiler's own explanation.
컴파일러 자신의 설명.
Control passes to [Custom operator workflow](vai-custom-op.md), which is
제어는 [Custom Operator Workflow](vai-custom-op.md)로 전달됩니다.
a multi-phase workflow that may take substantial time.
상당한 시간이 걸릴 수 있는 다단계 작업 흐름.
This loop pauses until it
이 루프는 다음이 완료될 때까지 일시중지됩니다.
returns an integrated model, or you decline.
통합 모델을 반환하지 않으면 거부합니다.

## Session length
## 세션 길이

Session length is driven by how many iterations run and how deep each one
세션 길이는 반복 실행 횟수와 각 반복의 깊이에 따라 결정됩니다.
compiles.
컴파일합니다.
Ordered from cheapest to most expensive:
가장 저렴한 것부터 가장 비싼 것 순으로 정렬했습니다.

* **Proposing a set of options** is near-instant.
* **옵션 세트 제안**은 거의 즉각적입니다.
* **A fast compile** is the quick path, but still the dominant cost of an
* **빠른 컴파일**은 빠른 경로이지만 여전히 컴파일의 주요 비용입니다.
iteration.
반복.
* **A complete compile** is considerably slower, and a board run adds to it.
* **완전한 컴파일**은 상당히 느리고 보드 실행이 추가됩니다.

A large model explored at complete depth can therefore run for a long time.
따라서 완전한 깊이로 탐색된 대규모 모델은 오랫동안 실행될 수 있습니다.
Keep it bounded with an explicit stop condition in your prompt:
프롬프트에서 명시적인 중지 조건으로 제한을 유지하세요.

```text
Stop after 2 iterations without improvement.
```

## Limitations
## 제한사항

* No functional or numeric correctness checking.
* 기능적 또는 수치적 정확성 검사가 없습니다.
* Options that trade accuracy are
* 정확성을 거래하는 옵션은 다음과 같습니다.
used only with your consent, and never in autonomous mode.
귀하의 동의가 있는 경우에만 사용되며 자율 모드에서는 절대 사용되지 않습니다.
* Only a defined set of supported options are explored.
* 정의된 지원 옵션 세트만 탐색됩니다.
* 100% offload cannot be guaranteed.
* 100% 오프로드를 보장할 수는 없습니다.
* Whether an operator can run on the NPU
* 운영자가 NPU에서 실행할 수 있는지 여부
depends on compiler support for it.
컴파일러 지원에 따라 다릅니다.
* Inference time figures come from a real board run, never from a host-only
* 추론 시간 수치는 실제 보드 실행에서 나오며 호스트 전용에서는 나오지 않습니다.
compile.
컴파일.
* The workflow proposes against evidence from your model or your compile.
* 워크플로는 모델이나 컴파일의 증거에 대해 제안합니다.
* An
option with no supporting evidence is not proposed, however promising it
뒷받침하는 증거가 없는 옵션은 제안되지 않지만 유망합니다.
sounds.
소리.

### No viable configuration
### 실행 가능한 구성이 없습니다.

If the baseline configuration and every relevant combination fail to compile,
기본 구성 및 모든 관련 조합이 컴파일되지 않는 경우
the workflow stops and reports that compilation fails under all relevant
워크플로가 중지되고 모든 관련 항목에서 컴파일이 실패했음을 보고합니다.
combinations.
조합.
It does not churn through irrelevant combinations, and it never
관련없는 조합으로 휘젓지 않으며, 절대로
reduces or rewrites your ONNX model to force a compile.
ONNX 모델을 줄이거나 다시 작성하여 강제로 컴파일합니다.
Where an unsupported
지원되지 않는 곳
operator is the cause, it reports the operator and the compiler's reason.
연산자가 원인인 경우 연산자와 컴파일러의 이유를 보고합니다.
