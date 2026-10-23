<!-- ko-translation: enabled -->
# Model quantization workflow
# 모델 양자화 작업 흐름

Entry point: `/vai-quantization-guide`
진입점: `/vai-Quantization-guide`

Quantizes a model, assigning precision per layer where mixed precision is
모델을 양자화하여 혼합 정밀도가 있는 레이어당 정밀도를 할당합니다.
warranted.
보증.
Choosing that precision by hand is error-prone and
그 정밀도를 수동으로 선택하는 것은 오류가 발생하기 쉽고
topology-dependent, and tuning it takes many experiment cycles.
토폴로지에 따라 다르며 튜닝에는 많은 실험 주기가 필요합니다.
The workflow
워크플로우
automates the cycle: it analyzes the model, selects a strategy, quantizes,
주기를 자동화합니다. 모델을 분석하고, 전략을 선택하고, 수량화하고,
compiles, evaluates accuracy against an FP32 baseline, and iterates until your
FP32 기준에 대해 컴파일하고 정확도를 평가하며
targets are met.
목표가 달성되었습니다.

The points in [Board access](index.md#board-access) apply to this workflow.
[보드 액세스](index.md#board-access)의 사항이 이 워크플로에 적용됩니다.
For the manual procedure
수동 절차의 경우
this workflow automates, see the [Model Quantization](https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/model_quantization/model_quantization.html) and [Mixed Precision Compilation](https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/model_compilation/mixed_precision_compilation.html) sections of the Vitis AI documentation.
이 워크플로는 자동화됩니다. Vitis의 [모델 양자화](https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/model_Quantization/model_Quantization.html) 및 [혼합 정밀도 컴파일](https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/model_compilation/mixed_precision_compilation.html) 섹션을 참조하세요. AI 문서.

## When to use
## 언제 사용하나요?

* Uniform low-precision quantization has cost you more accuracy than you can
* 균일한 저정밀도 양자화로 인해 가능한 것보다 정확도가 더 높아졌습니다.
accept.
받아들입니다.
* You want to find the smallest set of layers that must stay at higher
* 더 높은 위치에 있어야 하는 가장 작은 레이어 세트를 찾고 싶습니다.
precision.
정밀도.
* You want to know whether mixed precision is worth pursuing at all, before
* 혼합 정밀도를 추구할 가치가 있는지 알고 싶습니다.
committing to it.
그것에 전념합니다.

You provide the model, calibration data, and explicit accuracy and performance
모델, 교정 데이터, 명시적인 정확도 및 성능을 제공합니다.
objectives.
목표.
The workflow has no body of knowledge about what counts as a good
워크플로우에는 무엇이 상품으로 간주되는지에 대한 지식이 없습니다.
quantized model on your device, so it cannot infer targets you have not stated.
장치의 양자화 모델이므로 명시하지 않은 목표를 추론할 수 없습니다.
You get a quantized or requantized model, the per-layer precision assignment
양자화 또는 재양자화 모델, 레이어별 정밀도 할당을 얻습니다.
that produced it, and the measured accuracy for that assignment.
그것을 생산한 사람과 그 과제에 대해 측정된 정확도입니다.

## Prerequisites
## 전제조건

Follow the installation procedure in
설치 절차를 따르십시오.
[the skill installation guide](../../README.md#quick-start), so that
[스킬 설치 가이드](../../README.md#quick-start)
your assistant has model access and the workflow files installed.
귀하의 어시스턴트에는 모델 액세스 권한이 있고 워크플로 파일이 설치되어 있습니다.
Quark must be
쿼크는 반드시
available for any run that quantizes a model; the workflow checks for it before
모델을 양자화하는 모든 실행에 사용할 수 있습니다. 워크플로는 전에 이를 확인합니다.
proceeding.
진행.

Configure board access as described in [Board access](index.md#board-access).
[보드 액세스](index.md#board-access)에 설명된 대로 보드 액세스를 구성합니다.

> **Note**
> **참고**
>
> Ideally keep a board available for the whole session.
> 이상적으로는 전체 세션 동안 보드를 사용할 수 있도록 유지하십시오.
> The workflow measures
> 워크플로 측정
> accuracy and performance on hardware to steer its iterations.
> 반복을 조종하기 위한 하드웨어의 정확성과 성능.
> Without a
> 없이
> board, it is limited to compile-stage results, and cannot verify what a
> 보드에서는 컴파일 단계의 결과로 제한되며, 어떤 내용인지 확인할 수 없습니다.
> precision assignment costs in practice.
> 실제로 정밀 할당 비용.

## Quick start
## 빠른 시작

You can be as brief or as specific as you like.
원하는 대로 간단하게 또는 구체적으로 작성할 수 있습니다.
At its simplest, point the
가장 간단하게는 다음을 가리킨다.
workflow at a model:
모델의 작업 흐름:

```text
Use the quantization workflow for ./my_model.onnx
```

The workflow then asks which device you target, checks whether the model is
그런 다음 워크플로에서는 어떤 장치를 대상으로 하는지 묻고 모델이 맞는지 확인합니다.
already quantized, proposes a strategy based on its structure, confirms that you
이미 양자화되어 그 구조에 기반한 전략을 제안하고,
want to proceed, and guides you through the rest.
진행하고 나머지 과정을 안내합니다.

For full control, give the options explicitly:
모든 권한을 부여하려면 옵션을 명시적으로 제공하세요.

```text
/vai-quantization-guide --model ./resnet50.onnx --config full_vint8.json \
  --device ve2 --stage exec --explore
```

See [Command reference](#command-reference) for the full argument list.
전체 인수 목록은 [명령 참조](#command-reference)를 참조하세요.

## How it works
## 작동 원리

The workflow proceeds through six phases:
워크플로우는 6단계로 진행됩니다.

1. **Model analysis.** Determines whether the model is already quantized, and
1. **모델 분석.** 모델이 이미 양자화되었는지 확인하고
inspects operator types and structure.
연산자 유형과 구조를 검사합니다.
2. **Baseline.** Runs the FP32 model on the CPU, as the accuracy reference.
2. **기준선.** 정확도 기준으로 CPU에서 FP32 모델을 실행합니다.
3. **Quantization.** Applies the specified configuration with Quark, whether
3. **양자화.** Quark를 사용하여 지정된 구성을 적용합니다.
full VINT8, full BF16, or mixed BF16 and VINT8.
전체 VINT8, 전체 BF16 또는 혼합 BF16 및 VINT8.
4. **Accuracy validation.** Compares the quantized model's outputs against the
4. **정확도 검증.** 양자화된 모델의 출력을
FP32 baseline.
FP32 기준.
5. **Exploration.** With `--explore`, iterates over configurations, such as
5. **탐색.** `--explore`를 사용하면 다음과 같은 구성을 반복합니다.
BF16 head and tail or mixed BF16 and VINT8, to find the best accuracy and
BF16 머리와 꼬리 또는 BF16과 VINT8을 혼합하여 최고의 정확도와
performance tradeoff.
성능 절충.
Specialized formats can be applied through optional
Optional을 통해 특화된 포맷 적용 가능
plugin skills.
플러그인 스킬.
6. **Board execution.** Compiles the final model and runs it on the NPU.
6. **보드 실행.** 최종 모델을 컴파일하고 NPU에서 실행합니다.

Throughout, the workflow selects compiler options deterministically, by
전체적으로 워크플로는 다음을 통해 결정론적으로 컴파일러 옵션을 선택합니다.
classification rather than by guess, and uses seed-based calibration data so
추측이 아닌 분류를 수행하고 시드 기반 교정 데이터를 사용하므로
that results are reproducible.
그 결과는 재현 가능합니다.

## Modes
## 모드

`--mode full`
`--모드 전체`
The default.
기본값입니다.
Runs the complete workflow described above, from analysis
분석부터 위에 설명된 전체 워크플로우를 실행합니다.
through board execution.
보드 실행을 통해.

`--mode dry-run`
`--모드 테스트 실행`
Runs the model on the CPU at FP32, full BF16, and full VINT8, and reports the
FP32, 전체 BF16 및 전체 VINT8의 CPU에서 모델을 실행하고
accuracy of each without modifying the model.
모델을 수정하지 않고도 각 정확도를 높일 수 있습니다.
FP32 is your reference, and the
FP32는 귀하의 참고 자료이며
BF16 and VINT8 figures bound what each single-precision strategy can achieve.
BF16 및 VINT8 수치는 각 단정밀도 전략이 달성할 수 있는 범위를 제한합니다.
The workflow prints a comparison table and recommends a strategy.
워크플로우는 비교표를 인쇄하고 전략을 추천합니다.

Run this first.
이것을 먼저 실행하십시오.
It tells you whether mixed precision is worth pursuing at
혼합 정밀도를 추구할 가치가 있는지 여부를 알려줍니다.
all, and it is the quickest way to gauge what a longer run will cost on your
그리고 이는 장거리 달리기에 드는 비용을 측정하는 가장 빠른 방법입니다.
setup.
설정.

`--mode requantize`
`--모드 재양자화`
Strips the existing Q/DQ nodes from a quantized model, applies the new
양자화된 모델에서 기존 Q/DQ 노드를 제거하고 새로운 노드를 적용합니다.
configuration given by `--config`, validates accuracy against the FP32
`--config`로 제공되는 구성은 FP32에 대한 정확성을 검증합니다.
baseline, and reports whether accuracy improved.
기준선을 작성하고 정확도가 향상되었는지 보고합니다.
Use this when an earlier
이전에 이것을 사용하십시오
attempt failed its accuracy checks and you want to try a different strategy,
시도가 정확성 확인에 실패했으며 다른 전략을 시도하고 싶습니다.
such as moving some layers to BF16.
일부 레이어를 BF16으로 이동하는 등.

`--explore`
`--탐색`
Searches for a mixed-precision assignment automatically.
혼합 정밀도 할당을 자동으로 검색합니다.
It scores each layer
각 레이어에 점수를 매깁니다.
by activation dynamic range, ranks them, then moves the highest-ranked layers
활성화 동적 범위를 기준으로 순위를 매긴 다음 가장 높은 순위의 레이어를 이동합니다.
from VINT8 to BF16 one at a time, measuring accuracy after every promotion.
VINT8에서 BF16까지 한 번에 하나씩, 모든 프로모션 후에 정확도를 측정합니다.

Each iteration is a full requantize and evaluate, so the cost grows linearly
각 반복은 전체 재양자화 및 평가이므로 비용은 선형적으로 증가합니다.
with the number of promotions.
프로모션 횟수와 함께.

The search keeps itself bounded.
검색 자체가 제한적으로 유지됩니다.
It skips a promotion that does not improve
개선되지 않는 프로모션을 건너뜁니다.
accuracy, and rolls back one that makes accuracy worse.
정확성을 높이고 정확성을 악화시키는 것을 롤백합니다.
It stops when any of
다음 중 하나라도 발생하면 중지됩니다.
the following is true:
다음은 사실입니다:

   * The pass rate reaches 99.9%.
   * 합격률은 99.9%에 이릅니다.
   * PSNR improves by less than 0.5 dB across three consecutive promotions.
   * PSNR은 세 번의 연속 프로모션에서 0.5dB 미만으로 향상됩니다.
   * BF16 reaches 30% of compute layers.
   * BF16은 컴퓨팅 레이어의 30%에 도달합니다.

   > **Note**
   > **참고**
   >
   > Activation dynamic range is a proxy.
   > 활성화 동적 범위는 프록시입니다.
   > It correlates with INT8 error, but
   > INT8 오류와 관련이 있지만
   > does not capture layer interactions, accumulation effects, or
   > 레이어 상호 작용, 누적 효과를 포착하지 못하거나
   > post-processing sensitivity.
   > 후처리 감도.
   > A layer that ranks low can still be the one
   > 순위가 낮은 레이어도 여전히 레이어일 수 있습니다.
   > costing you accuracy.
   > 정확성이 저하됩니다.

## Command reference
## 명령 참조

```text
/vai-quantization-guide \
  --model <path_to_onnx_model> \
  [--config <path_to_quark_config.json>] \
  [--device <device>] \
  [--mode <full|dry-run|requantize>] \
  [--stage <compile|exec>] \
  [--explore]
```

| Argument<br>인수 | Required<br>필수 | Description<br>설명 |
| --- | --- | --- |
| `--model <path>`<br>`--model <경로>` | Yes<br>예 | Path to the ONNX model, either `.onnx` or `.onnxtxt`.<br>ONNX 모델의 경로(`.onnx` 또는 `.onnxtxt`)입니다. |
| `--config <path>`<br>`--config <경로>` | No | Path to a Quark JSON configuration preset.<br>Quark JSON 구성 사전 설정의 경로입니다. |
| `--device <device>`<br>`--device <장치>` | No | Target device. Defaults to `ve2`.<br>대상 장치. 기본값은 've2'입니다. |
| `--mode <full<br>`--모드 <전체|dry-run<br>드라이런|requantize>`<br>재양자화>` | No | Execution mode. Defaults to `full`. See [Modes](#modes).<br>실행 모드. 기본값은 '전체'입니다. [모드](#modes)를 참조하세요. |
| `--stage <compile<br>`--stage <컴파일|exec>`<br>실행>` | No | `compile` compiles only; `exec` compiles and runs on the NPU. Defaults to `compile`.<br>`compile`은 컴파일만 합니다. `exec`는 NPU에서 컴파일하고 실행됩니다. 기본값은 '컴파일'입니다. |
| `--explore`<br>`--탐색` | No | Enables exploration mode, to search for an optimal mixed-precision assignment.<br>최적의 혼합 정밀도 할당을 검색하기 위해 탐색 모드를 활성화합니다. |
| `--target-accuracy <metric>`<br>`--목표 정확도 <측정항목>` | No | Accuracy objective, as a quoted expression, for example `"psnr>30"`.<br>따옴표로 묶인 표현인 정확도 목표(예: `"psnr>30"`) |
| `--atol <float>`<br>`--atol <플로트>` | No | Absolute tolerance. Defaults to `0.015`.<br>절대적인 관용. 기본값은 '0.015'입니다. |
| `--rtol <float>`<br>`--rtol <플로트>` | No | Relative tolerance. Defaults to `0.01`.<br>상대적인 허용오차. 기본값은 '0.01'입니다. |
| `--calibration-data <path>`<br>`--교정-데이터 <경로>` | No | Calibration data, as `.npy` or `.npz` files.<br>교정 데이터(`.npy` 또는 `.npz` 파일) |
| `--work-dir <path>`<br>`--work-dir <경로>` | No | Where the run stores its iterations and ledger.<br>실행이 반복과 원장을 저장하는 위치입니다. |
| `-t <path>`<br>`-t <경로>` | No | Path to the Vitis AI environment activation script, needed for the compile and board stages when you are not already in a provisioned environment. Omit inside the Vitis AI Docker container.<br>아직 프로비저닝된 환경이 아닌 경우 컴파일 및 보드 단계에 필요한 Vitis AI 환경 활성화 스크립트의 경로입니다. Vitis AI Docker 컨테이너 내부에서는 생략합니다. |

## Examples
## 예

The workflow accepts structured invocations, free-form prompts, and a mix of the
워크플로는 구조화된 호출, 자유 형식 프롬프트 및 다음의 혼합을 허용합니다.
two.
둘.

### Explore against reference data
### 참조 데이터 탐색

A structured invocation.
구조화된 호출.
Explores mixed precision against explicit absolute and
명시적 절대값과 혼합 정밀도를 탐색합니다.
relative tolerances, using the supplied calibration data:
제공된 교정 데이터를 사용하여 상대 공차:

```text
/vai-quantization-guide --model /path/to/model_fp32.onnx --explore
--atol .15 --rtol 0.1 --calibration-data ifm_0.npy
```

### Requantize for higher accuracy
### 더 높은 정확도를 위해 재양자화

A mixed prompt: the intent in prose, followed by the invocation that carries it
혼합 프롬프트: 산문의 의도와 이를 전달하는 호출이 뒤따름
out.
밖으로.
Use this shape when the reason for a request matters and you want the
요청 이유가 중요하고 원하는 경우 이 모양을 사용하세요.
workflow to take it into account:
이를 고려하는 워크플로:

```text
Requantize a vint8 model with a vint8+bf16 strategy. The full VINT8 model has
lower accuracy than required.

/vai-quantization-guide --model /path/to/model_vint8.onnx --mode requantize
--device ve2-xc2ve3858 --target-accuracy "psnr>40" --calibration-data ifm_0.npy
ifm_1.npy ifm_2.npy
```

### Quantize and verify offload
### 오프로드 양자화 및 확인

A free-form prompt covering quantization, compilation, partition analysis, and a
양자화, 컴파일, 파티션 분석 및
board run in a single request:
단일 요청으로 보드 실행:

```text
/vai-quantization-guide Apply quantization to the model:
/path/to/vit_model_fp32.onnx. After quantization you should compile the
model. Attempt to achieve full offloading. Perform partition analysis and
report in detail what was achieved. If the compilation validates full
offloading, run on the board to identify the inference time speedup.
```

## Iterate
## 반복

Once the basic flow works, refine the result with follow-up prompts in the same
기본 흐름이 작동하면 동일한 후속 프롬프트로 결과를 구체화합니다.
session:
세션:

```text
Dequantize the model and requantize with the backbone in VINT8 and head and
tail in BF16.
```

```text
Keep the accuracy-sensitive layers in BF16 and quantize the rest to VINT8.
```

## Session length
## 세션 길이

Session length is dominated by compile and board iterations, not by the
세션 길이는 컴파일 및 보드 반복에 의해 좌우됩니다.
assistant's reasoning, so it depends on your host machine, your board setup, and
어시스턴트의 추론이므로 호스트 시스템, 보드 설정 및
the size of the model.
모델의 크기.
The list below is relative rather than absolute, ordered
아래 목록은 절대적인 것이 아니라 상대적인 것입니다.
from fastest to slowest:
가장 빠른 것부터 가장 느린 것까지:

* **A dry run**, or a narrowly specified quantization request, is the quickest
* **모의 실행** 또는 좁게 지정된 양자화 요청이 가장 빠릅니다.
path.
경로.
It does not compile, so it is bounded by evaluation alone.
컴파일되지 않으므로 평가만으로 제한됩니다.
* **Compilation** with the Vitis AI compiler dominates any run that reaches it.
* Vitis AI 컴파일러를 사용한 **컴파일**은 이에 도달하는 모든 실행을 지배합니다.
* **Board time** splits into setup and inference, both highly dependent on
* **보드 시간**은 설정과 추론으로 나누어지며, 둘 다 다음에 크게 의존합니다.
context and data.
컨텍스트와 데이터.
* **Exploration** calls Quark in a loop, and is substantially slower than a
* **탐색**은 루프에서 Quark를 호출하며, Quark보다 상당히 느립니다.
guided quantization or a dry run.
유도된 양자화 또는 연습 실행.
Its cost grows with the number of layer
레이어 수에 따라 비용이 증가합니다.
promotions.
프로모션.
* **ViT models** go through a rigid pipeline, and are normally much faster than
* **ViT 모델**은 엄격한 파이프라인을 통과하며 일반적으로 ViT 모델보다 훨씬 빠릅니다.
exploration.
탐험.

Measure a dry run on your own setup first, and use it to gauge what a full
먼저 자체 설정에서 테스트 실행을 측정하고 이를 사용하여 전체 설정이 얼마나 되는지 측정합니다.
exploration will cost.
탐사 비용이 발생합니다.

## Configuration ownership
## 구성 소유권

Do not hand-write Quark configurations.
Quark 구성을 직접 작성하지 마십시오.
The workflow derives them from model
워크플로는 모델에서 파생됩니다.
analysis, with target node names and regular expressions for the layers it
대상 노드 이름과 레이어에 대한 정규 표현식을 사용한 분석
promotes, exclude lists for shape and indexing operations that do not need
필요하지 않은 모양 및 인덱싱 작업에 대한 목록을 승격하고 제외합니다.
quantizing, and the extra options that matter for VINT8+BF16 and VINT8+FP16.
양자화 및 VINT8+BF16 및 VINT8+FP16에 중요한 추가 옵션.

Do not hand-write `vitisai_config.json` either.
'vitsai_config.json'도 직접 작성하지 마세요.
The workflow determines the
워크플로우는 다음을 결정합니다.
compiler options your quantized model needs, and is aware of
양자화된 모델에 필요한 컴파일러 옵션을 알고 있습니다.
[Compiler options workflow](vai-flag-configuration.md).
[컴파일러 옵션 작업 흐름](vai-flag-configuration.md).

By policy, the workflow does not perform ONNX graph surgery.
정책에 따라 워크플로는 ONNX 그래프 수술을 수행하지 않습니다.
It produces a new
그것은 새로운 것을 생산한다
quantized or requantized model from a new Quark configuration.
새로운 Quark 구성의 양자화 또는 재양자화 모델.

## Supported data types
## 지원되는 데이터 유형

* BF16, FP16, and VINT8 single precision, plus VINT8+BF16 and VINT8+FP16 mixed
* BF16, FP16 및 VINT8 단정밀도와 VINT8+BF16 및 VINT8+FP16 혼합
precision.
정밀도.
* BF16 models cannot run under ONNX Runtime, because there is no CPU reference
* BF16 모델은 CPU 참조가 없기 때문에 ONNX 런타임에서 실행할 수 없습니다.
for BF16.
BF16의 경우.
* Extended quantization, as defined by Quark, is not supported in this release.
* Quark에서 정의한 확장 양자화는 이번 릴리스에서 지원되지 않습니다.
* Data types beyond those listed are available only through experimental
* 나열된 것 이외의 데이터 유형은 실험을 통해서만 사용할 수 있습니다.
methods.
방법.

## Partition analysis
## 파티션 분석

Partition analysis aims for full offload: a single partition carrying 100% of
파티션 분석은 전체 오프로드를 목표로 합니다. 즉, 단일 파티션이 100% 데이터를 전달합니다.
GOPS on the NPU.
NPU의 GOPS.
A model that fits one partition maps onto a single NPU block;
하나의 파티션에 맞는 모델은 단일 NPU 블록에 매핑됩니다.
models that do not are spread across several.
여러 모델에 걸쳐 분산되지 않은 모델.
Where a single partition is not
단일 파티션이 아닌 경우
achievable, the workflow falls back to secondary targets that favor high offload
달성 가능한 경우 워크플로는 높은 오프로드를 선호하는 보조 대상으로 대체됩니다.
and a low partition count.
그리고 파티션 수가 적습니다.
It adjusts compiler options first, and changes the
컴파일러 옵션을 먼저 조정하고
quantization only if that is not enough.
충분하지 않은 경우에만 양자화합니다.

Support for edge quantization in the runtime is limited.
런타임 시 에지 양자화 지원은 제한됩니다.

## Limitations
## 제한사항

* The workflow has only a partial view of the kernels and graph patterns the
* 워크플로우에는 커널과 그래프 패턴의 부분적인 보기만 있습니다.
compiler provides, and less of their constraints.
컴파일러는 제약 조건을 덜 제공합니다.
`CpuBecause` can therefore
따라서 `CpuBecause`는
appear even with a valid quantization.
유효한 양자화가 있어도 나타납니다.
The workflow recognizes the most common
워크플로는 가장 일반적인
`CpuBecause` cases, but not all of them.
'CpuBecause' 사례이지만 전부는 아닙니다.
* The workflow does not account for `dp_size`, `tp_size`, `overlay`, or
* 워크플로는 `dp_size`, `tp_size`, `overlay`를 고려하지 않습니다.
`device`, and assumes a VE2 device.
'device'이며 VE2 장치를 가정합니다.
Follow up with
후속 조치
[Compiler options workflow](vai-flag-configuration.md) if you need those set.
[컴파일러 옵션 워크플로우](vai-flag-configuration.md) 해당 세트가 필요한 경우.
* The exploration algorithm is deliberately simple, designed for fast loop
* 탐색 알고리즘은 의도적으로 단순하며 빠른 루프를 위해 설계되었습니다.
iterations rather than thorough search.
철저한 검색보다는 반복.
* Warnings are not suppressed, by design.
* 의도적으로 경고는 표시되지 않습니다.
* The orchestrator and worker split is fully supported only in Claude Code.
* 오케스트레이터와 작업자 분할은 Claude Code에서만 완전히 지원됩니다.

## Implementation
## 구현

You do not need this to use the workflow.
워크플로를 사용하는 데는 이것이 필요하지 않습니다.
It is here for reference when
그럴 때 참고용으로 여기에 있습니다.
troubleshooting an installation.
설치 문제 해결.

The quantization workflow consists of five skills and one agent, plus optional
양자화 워크플로는 5개의 스킬과 1개의 에이전트, 그리고 선택 사항으로 구성됩니다.
plugin skills discovered at runtime:
런타임 시 발견된 플러그인 기술:

| Component<br>구성 요소 | Type<br>유형 | Purpose<br>목적 |
| --- | --- | --- |
| `vai-quantization-guide`<br>`vai-양자화 가이드` | Skill<br>스킬 | Entry point. Orchestrates end-to-end mixed-precision quantization.<br>진입점. 엔드투엔드 혼합 정밀도 양자화를 조율합니다. |
| `vai-fe-args`<br>'바이페아르그스' | Skill<br>스킬 | Determines the compiler options a mixed-precision model needs.<br>혼합 정밀도 모델에 필요한 컴파일러 옵션을 결정합니다. |
| `vai-dequantize-model`<br>`vai-역양자화 모델` | Skill<br>스킬 | Strips quantization from a model, for requantization.<br>재양자화를 위해 모델에서 양자화를 제거합니다. |
| `vai-partition-analysis`<br>'VAI-파티션-분석' | Skill<br>스킬 | Analyzes how a model is partitioned across the NPU and CPU, and reports the offloading impact of each quantization and compilation iteration.<br>NPU와 CPU에서 모델이 분할되는 방식을 분석하고 각 양자화 및 컴파일 반복의 오프로드 영향을 보고합니다. |
| `vai-vaip-patching`<br>`vai-vaip-패칭` | Skill<br>스킬 | Patches models for compatibility, covering shapes and data types.<br>모양과 데이터 유형을 포함하여 호환성을 위해 모델을 패치합니다. |
| `vai-ffn-quantization`<br>`vai-ffn-양자화` | Skill<br>스킬 | Detects and quantizes FFN layers, that is MLP and MHA projections.<br>MLP 및 MHA 투영인 FFN 레이어를 감지하고 양자화합니다. |
| `vai-mixed-precision-worker`<br>`바이-혼합-정밀 작업자` | Agent<br>대리인 | Constrained worker spawned by the orchestrator.<br>오케스트레이터에 의해 생성된 제한된 작업자입니다. |

```text
ai_utils/
  skills/
    vai-quantization-guide/            # Entry point, /vai-quantization-guide
      SKILL.md                         # Orchestrator workflow definition
      README.md                        # Mixed-precision reference
      scripts/                         # Quantization, dequantization,
                                       # and accuracy comparison
    vai-fe-args/                       # Compiler option determination
      SKILL.md
    vai-dequantize-model/              # Model dequantization
      SKILL.md
    vai-vaip-patching/                 # Compatibility patching
      SKILL.md
    vai-ffn-quantization/              # FFN detection and quantization
      SKILL.md
    experimental-methods/              # Optional plugins, found at runtime
      <plugin-name>/
        SKILL.md
  agents/
    vai-mixed-precision-worker.md      # Worker agent definition
```

Only `/vai-quantization-guide` is a user entry point.
`/vai-Quantization-guide`만이 사용자 진입점입니다.
The supporting skills and
지원 스킬과
the worker agent are invoked on your behalf and should not be called directly.
작업자 에이전트는 사용자를 대신하여 호출되며 직접 호출하면 안 됩니다.

Plugin skills are optional and are discovered from the filesystem at runtime.
플러그인 기술은 선택 사항이며 런타임 시 파일 시스템에서 검색됩니다.
If
one is installed, the workflow offers it where relevant, and asks for your
하나가 설치되면 워크플로에서 해당 항목을 제공하고 다음을 요청합니다.
consent before applying it.
적용하기 전에 동의하십시오.
