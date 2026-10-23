<!-- ko-translation: enabled -->
---
name: vai-quantization-guide
license: MIT
compatibility: Requires VitisAI 6.3+
metadata:
  version: 6.3.0
  stage: beta
description: Master orchestrator for mixed-precision ONNX model quantization targeting
  AMD NPU (Strix/Telluride). Guides users through quantization strategy selection,
  accuracy validation, compiler flag configuration, and board execution. Supports
  full bf16, full vint8, hybrid BF16/VINT8 configurations, and edge quantization in
  runtime. Specialized quantization methods are provided as optional plugin skills
  discovered at runtime.
---
<!--- Copyright (C) 2026 Advanced Micro Devices, Inc. All rights reserved.--->

# Mixed-Precision Quantization Orchestrator
# 혼합 정밀도 양자화 조정자

## Scope
## 범위

This skill helps you **quantize ONNX models** for deployment on AMD NPU
이 기술은 AMD NPU에 배포하기 위해 **ONNX 모델을 양자화**하는 데 도움이 됩니다.
hardware (Strix or Telluride).
하드웨어(Strix 또는 Telluride).
It covers the full journey from an unquantized
그것은 양자화되지 않은 것에서부터 전체 여정을 다룹니다.
FP32 model to a hardware-optimized mixed-precision model.
FP32 모델을 하드웨어에 최적화된 혼합 정밀도 모델로 변환합니다.
Specifically, it:
구체적으로는 다음과 같습니다.

- Analyzes an input model (quantized or not)
- 입력 모델 분석(양자화 여부)
- Selects a quantization strategy (VINT8, BF16, or a mix)
- 양자화 전략(VINT8, BF16 또는 혼합)을 선택합니다.
- Applies quantization via the AMD Quark tool
- AMD Quark 도구를 통해 양자화 적용
- Validates accuracy against a reference
- 참조에 대한 정확성을 검증합니다.
- Determines the correct compiler flags
- 올바른 컴파일러 플래그를 결정합니다.
- Compiles and optionally executes on NPU hardware
- NPU 하드웨어에서 컴파일하고 선택적으로 실행합니다.
- Iterates until accuracy and performance targets are met
- 정확성과 성능 목표가 충족될 때까지 반복합니다.

**What this skill does NOT do**: It does not train models, alter model
**이 기술이 수행하지 않는 작업**: 모델을 훈련하지 않고 모델을 변경하지 않습니다.
architecture (layer count, shapes), or generate training data.
아키텍처(레이어 수, 모양) 또는 교육 데이터 생성.
It operates
작동한다
solely on the quantization and precision-mapping of an existing ONNX model.
기존 ONNX 모델의 양자화 및 정밀 매핑에만 의존합니다.

---

## Model Modification Policy (Mandatory)
## 모델 수정 정책(필수)

> **A model is quantized ONLY through Quark configurations.** This is the
> **모델은 Quark 구성을 통해서만 양자화됩니다.** 이는
> governing rule of this skill.
> 이 스킬의 지배 규칙.
> When something needs to change about how a model
> 모델의 방식을 변경해야 하는 경우
> is quantized — accuracy, precision mapping, unsupported ops, dtype mismatches,
> 양자화됨 — 정확성, 정밀 매핑, 지원되지 않는 작업, dtype 불일치,
> partitioning problems — the response is to **write or adjust a Quark JSON
> 파티셔닝 문제 — 응답은 **Quark JSON을 작성하거나 조정하는 것입니다.
> configuration and (re)quantize**, never to hand-edit the ONNX graph.
> 구성 및 (재)양자화**, ONNX 그래프를 직접 편집하지 마세요.

**NEVER perform ad hoc ONNX graph surgery** to work around a problem.
**문제를 해결하기 위해 임시 ONNX 그래프 수술을 수행하지 마세요**.
This
이
includes (but is not limited to) inserting/removing/rewiring nodes, editing
노드 삽입/제거/재배선, 편집이 포함되지만 이에 국한되지는 않습니다.
initializers, rewriting `value_info`, or patching tensor dtypes/shapes directly
초기화, 'value_info' 재작성 또는 텐서 dtypes/shapes 직접 패치
on the model.
모델에.
If a problem *seems* like it would benefit from ad hoc rewriting
문제가 임시 재작성으로 도움이 될 것 같은 *생각* 경우
of the model, that is a signal to express the fix as a Quark configuration
모델의 수정 사항을 Quark 구성으로 표현하는 신호입니다.
instead (e.g., `--mode requantize` with a new config), not to edit the graph.
대신(예: 새로운 구성으로 `--mode reQuantize`) 그래프를 편집하지 마세요.

There are exactly **two** permitted exceptions where direct modification of the
직접 수정하는 경우 정확히 **두 가지** 허용되는 예외가 있습니다.
ONNX model is allowed:
ONNX 모델이 허용됩니다.

1. **The `vai-vaip-patching` skill — and only for the specific edits it supports.**
1. **`vai-vaip-patching` 스킬 — 지원하는 특정 편집에만 해당됩니다.**
Namely: retagging `com.microsoft` Q/DQ to the default ONNX domain, adding
즉, 'com.microsoft' Q/DQ를 기본 ONNX 도메인에 다시 태그 지정하고 추가합니다.
`shape`/`data_type` attributes to custom ops (`ExtendedQuantizeLinear`,
사용자 정의 작업에 대한 `shape`/`data_type` 속성(`ExtendedQuantizeLinear`,
`ExtendedDequantizeLinear`, BFP QDQ), and fixing zero-point `bf16`/`fp16`
`ExtendedDeQuantizeLinear`, BFP QDQ) 및 영점 `bf16`/`fp16` 고정
dtypes.
dtypes.
No other edits may be made under this exception.
이 예외 사항에는 다른 편집이 불가능합니다.
Invoke it via
그것을 통해 호출
`/vai-vaip-patch`; do not reimplement its edits by hand.
`/vai-vaip-패치`; 편집 내용을 직접 다시 구현하지 마십시오.

2. **ViT models — the MHA-MX9 quantization pipeline.** ViT-style encoders
2. **ViT 모델 — MHA-MX9 양자화 파이프라인.** ViT 스타일 인코더
require graph surgery *by construction* (fused-QKV split, BFP wrapper
*구성상* 그래프 수술이 필요함(융합-QKV 분할, BFP 래퍼
insertion on MHA blocks, attention-path QDQ stripping, and bf16 residual
MHA 블록에 삽입, 주의 경로 QDQ 스트리핑 및 bf16 잔여
`Cast` conversion).
'캐스트' 변환).
This is the **only** quantization pathway permitted to
이는 허용되는 **유일한** 양자화 경로입니다.
modify the graph directly, and only through the dedicated ViT/MHA-MX9
전용 ViT/MHA-MX9를 통해서만 그래프를 직접 수정할 수 있습니다.
scripts — never as improvised one-off edits.
스크립트 — 즉석에서 일회성 편집을 수행하지 않습니다.
If graph surgery appears
그래프수술이 나타나는 경우
necessary and the model is **not** a ViT going through this pipeline, do
필요하고 모델이 이 파이프라인을 통과하는 ViT가 **아닙니다**.
**not** do it: write a Quark configuration instead.
**하지 마세요**: 대신 Quark 구성을 작성하세요.

In every other situation, changing the model means changing the Quark config and
다른 모든 상황에서 모델을 변경한다는 것은 Quark 구성과
requantizing — full stop.
재양자화 — 마침표.

---

## Terminology: Quantization Formats
## 용어: 양자화 형식

Before you begin, here is what the key terms mean:
시작하기 전에 주요 용어의 의미는 다음과 같습니다.

| Term<br>기간 | What it is<br>그것은 무엇입니까 | Accuracy<br>정확도 | Speed<br>속도 | Hardware<br>하드웨어 |
|------|-----------|----------|-------|----------|
| **BF16** (BFloat16)<br>**BF16** (BFloat16) | 16-bit floating point with 8-bit exponent. All ops run in reduced-precision float.<br>8비트 지수를 포함하는 16비트 부동 소수점입니다. 모든 연산은 정밀도가 낮은 부동 소수점에서 실행됩니다. | Highest (near FP32)<br>최고(FP32 근처) | Baseline (1x)<br>기준선(1x) | STX + VE2<br>STX+VE2 |
| **FP16** (Float16)<br>**FP16** (플로트16) | 16-bit floating point with 5-bit exponent. Similar to BF16 but native on Telluride.<br>5비트 지수를 포함하는 16비트 부동 소수점입니다. BF16과 유사하지만 Telluride에 기본입니다. | High<br>높음 | ~1x | **VE2 only**<br>**VE2만 해당** |
| **VINT8**<br>**빈트8** | 8-bit integer with fake-quantization (DQ→op→Q pattern). Scale/zero-point per tensor.<br>가짜 양자화(DQ→op→Q 패턴)가 포함된 8비트 정수. 텐서당 스케일/영점. | Lower (depends on model)<br>낮은(모델에 따라 다름) | 2–4x faster<br>2~4배 더 빠름 | STX + VE2<br>STX+VE2 |
| **Mixed BF16/VINT8**<br>**혼합 BF16/VINT8** | Some layers in BF16 (accuracy-sensitive), others in VINT8 (performance).<br>일부 레이어는 BF16(정확도에 민감)이고 다른 레이어는 VINT8(성능)입니다. | Tunable<br>조정 가능 | 1.2–2x faster<br>1.2~2배 더 빠름 | STX + VE2<br>STX+VE2 |

> Additional specialized quantization formats (e.g. block floating-point or
> 추가적인 특수 양자화 형식(예: 블록 부동 소수점 또는
> extended per-layer precision) may be available as **optional plugin skills**.
> 확장된 레이어별 정밀도)는 **선택적 플러그인 기술**로 제공될 수 있습니다.
> See [Plugin Skills](#plugin-skills).
> [플러그인 스킬](#plugin-skills)을 참조하세요.

### Hardware Compatibility
### 하드웨어 호환성

| Device<br>장치 | Supported formats<br>지원되는 형식 |
|--------|-------------------|
| **STX** (Strix)<br>**STX** (스트릭스) | BF16, VINT8, Mixed BF16/VINT8<br>BF16, VINT8, 혼합 BF16/VINT8 |
| **VE2** (Telluride)<br>**VE2**(텔루라이드) | All of the above **plus** FP16<br>위의 모든 것 **+** FP16 |

> **Important**: FP16 is only supported on the VE2 (Telluride) device.
> **중요**: FP16은 VE2(Telluride) 장치에서만 지원됩니다.
> If you target STX (Strix), you can only use BF16 and VINT8 combinations.
> STX(Strix)를 대상으로 하는 경우 BF16 및 VINT8 조합만 사용할 수 있습니다.
> Optional plugin skills may add formats with their own hardware constraints.
> 선택적 플러그인 기술은 자체 하드웨어 제약으로 형식을 추가할 수 있습니다.

---

## User Stories: Where to Start
## 사용자 스토리: 어디서부터 시작해야 할까요?

Pick the scenario that best matches your situation:
상황에 가장 적합한 시나리오를 선택하세요.

### Story 1: "I have an FP32 model and want maximum performance"
### 이야기 1: "FP32 모델을 가지고 있고 최대 성능을 원합니다"

**Goal**: Deploy on NPU as fast as possible, accepting some accuracy loss.
**목표**: 약간의 정확도 손실을 감수하면서 가능한 한 빨리 NPU에 배포합니다.

**Recommended approach**:
**권장 접근 방식**:
1. Try full VINT8 first (`--config full_vint8.json`)
1. 먼저 전체 VINT8을 사용해 보세요(`--config full_vint8.json`)
2. Check accuracy against FP32 reference
2. FP32 참조에 대한 정확성 확인
3. If accuracy is OK → done, this is the fastest option
3. 정확도가 양호 → 완료되면 이것이 가장 빠른 옵션입니다.
4. If accuracy fails → use `--explore` to find which layers need BF16
4. 정확도가 떨어지면 → `--explore`를 사용하여 BF16이 필요한 레이어를 찾으세요.

**Command**:
**명령**:
```
/vai-quantization-guide --model my_model.onnx --device ve2 --explore
```

### Story 2: "I have a quantized model and want to deploy it"
### 이야기 2: "양자화된 모델이 있고 이를 배포하고 싶습니다."

**Goal**: Compile and run an already-quantized model on NPU.
**목표**: NPU에서 이미 양자화된 모델을 컴파일하고 실행합니다.

**Recommended approach**:
**권장 접근 방식**:
1. Run the skill to analyze quantization patterns and determine flags
1. 기술을 실행하여 양자화 패턴을 분석하고 플래그를 결정합니다.
2. Compile and validate on hardware
2. 하드웨어에서 컴파일 및 검증

**Command**:
**명령**:
```
/vai-quantization-guide --model my_quantized_model.onnx --device ve2 --stage exec
```

### Story 3: "I need high accuracy AND good performance"
### 이야기 3: "높은 정확성과 좋은 성능이 필요해요"

**Goal**: Find the optimal accuracy/performance trade-off using mixed precision.
**목표**: 혼합 정밀도를 사용하여 최적의 정확도/성능 절충점을 찾습니다.

**Recommended approach**:
**권장 접근 방식**:
1. Run in exploration mode — the skill will iteratively move layers between
1. 탐색 모드에서 실행 - 기술이 레이어 간에 레이어를 반복적으로 이동합니다.
VINT8 and BF16 until accuracy targets are met
정확도 목표가 충족될 때까지 VINT8 및 BF16
2. This produces a Quark JSON config you can reuse
2. 그러면 재사용할 수 있는 Quark JSON 구성이 생성됩니다.

**Command**:
**명령**:
```
/vai-quantization-guide --model my_model.onnx --device ve2 --explore \
  --atol 0.01 --calibration-data calib_data.npy
```

---

## Execution Modes Explained
## 실행 모드 설명

| Mode<br>모드 | What happens step by step<br>단계별로 무슨 일이 일어나는지 |
|------|---------------------------|
| `--mode full`<br>`--모드 전체` | (1) Analyze model → (2) Run FP32 baseline on CPU → (3) Quantize with Quark → (4) Determine compiler flags → (5) Compile for NPU → (6) Compare accuracy → (7) Report results<br>(1) 모델 분석 → (2) CPU에서 FP32 기준선 실행 → (3) Quark로 양자화 → (4) 컴파일러 플래그 결정 → (5) NPU용 컴파일 → (6) 정확도 비교 → (7) 결과 보고 |
| `--mode dry-run`<br>`--모드 테스트 실행` | (1) Analyze model → (2) Run FP32 baseline → (3) Run BF16 baseline → (4) Run VINT8 baseline → (5) Report accuracy of each WITHOUT modifying the model → (6) Recommend a strategy<br>(1) 모델 분석 → (2) FP32 기준 실행 → (3) BF16 기준 실행 → (4) VINT8 기준 실행 → (5) 모델 수정 없이 각 정확도 보고 → (6) 전략 추천 |
| `--mode requantize`<br>`--모드 재양자화` | (1) Strip existing quantization → (2) Apply new quantization from `--config` → (3) Validate accuracy → (4) Report improvement<br>(1) 기존 양자화 제거 → (2) `--config`에서 새로운 양자화 적용 → (3) 정확성 검증 → (4) 보고서 개선 |

### `--mode dry-run` in detail
### `--mode dry-run` 자세히 알아보기

Use this when you want to **understand your model's quantization behavior
**모델의 양자화 동작을 이해하고 싶을 때 사용하세요.
before committing** to a specific strategy.
특정 전략을 실행하기 전에**
No files are modified.
파일이 수정되지 않습니다.

Steps executed:
실행 단계:
1. **Analyze** — inspect the model for existing quantization nodes, op types,
1. **분석** — 기존 양자화 노드, 작업 유형에 대한 모델을 검사합니다.
input/output shapes.
입력/출력 모양.
2. **FP32 baseline** — run the model on CPU with `CPUExecutionProvider` to
2. **FP32 기준** — `CPUExecutionProvider`를 사용하여 CPU에서 모델을 실행하여
produce golden reference outputs.
골든 레퍼런스 출력을 생성합니다.
3. **BF16 baseline** — quantize a temporary copy to full BF16, run on CPU,
3. **BF16 기준** — 임시 복사본을 전체 BF16으로 양자화하고 CPU에서 실행합니다.
compare against FP32 (reports PSNR and element-wise pass rate).
FP32와 비교합니다(PSNR 및 요소별 통과율 보고).
This shows
이것은 보여줍니다
the accuracy ceiling for NPU deployment.
NPU 배포의 정확도 한도.
4. **VINT8 baseline** — quantize a temporary copy to full VINT8, run on CPU,
4. **VINT8 기준** — 임시 복사본을 전체 VINT8로 양자화하고 CPU에서 실행합니다.
compare against FP32.
FP32와 비교해보세요.
This shows the performance-optimum accuracy.
이는 성능 최적화 정확도를 보여줍니다.
5. **Report** — print a summary table comparing BF16 and VINT8 accuracy, and
5. **보고서** — BF16 및 VINT8 정확도를 비교하는 요약 표를 인쇄합니다.
recommend a strategy:
전략을 추천합니다:
   - If VINT8 passes tolerances → "Use full VINT8 (fastest)"
   - VINT8이 허용 오차를 통과하는 경우 → "전체 VINT8 사용(가장 빠름)"
   - If VINT8 fails but BF16 passes → "Use mixed BF16/VINT8 with exploration"
   - VINT8은 실패했지만 BF16은 통과하는 경우 → "탐색과 함께 BF16/VINT8 혼합 사용"
   - If both fail → "Model may need architectural changes or looser tolerances"
   - 둘 다 실패하는 경우 → "모델에 아키텍처 변경 또는 느슨한 공차가 필요할 수 있습니다."

**Example output (regression model)**:
**예제 출력(회귀 모델)**:
```
┌────────────────┬──────────┬────────────┬────────────────┐
│ Configuration  │ PSNR(dB) │ Pass Rate  │ Recommendation │
├────────────────┼──────────┼────────────┼────────────────┤
│ Full BF16      │   46.2   │   100.0%   │ Accuracy ref   │
│ Full VINT8     │   22.1   │    68.4%   │ ✗ fails atol   │
└────────────────┴──────────┴────────────┴────────────────┘
→ Recommendation: Use --explore to find optimal mixed BF16/VINT8 config
```

**Example output (classification model)**:
**출력 예(분류 모델)**:
```
┌────────────────┬──────────┬────────────┬──────────┬────────────────┐
│ Configuration  │ PSNR(dB) │ Pass Rate  │ Top-1    │ Recommendation │
├────────────────┼──────────┼────────────┼──────────┼────────────────┤
│ Full BF16      │   46.2   │   100.0%   │ ✓ match  │ Accuracy ref   │
│ Full VINT8     │   16.1   │     7.1%   │ ✓ match  │ ✓ USE THIS     │
└────────────────┴──────────┴────────────┴──────────┴────────────────┘
→ Recommendation: Use full VINT8 (top-1 preserved, fastest configuration).
  Low PSNR is expected for classification logits and does not indicate a problem.
```

**When to use**: Before spending time on quantization.
**사용 시기**: 양자화에 시간을 보내기 전.
Helps decide whether
여부를 결정하는 데 도움이 됩니다.
full VINT8 is sufficient or mixed-precision is needed.
전체 VINT8이면 충분하거나 혼합 정밀도가 필요합니다.

### `--mode requantize` in detail
### `--모드 재양자화` 자세히 알아보기

Use this when you have an **already-quantized model** whose quantization you
양자화를 수행하려는 **이미 양자화된 모델**이 있는 경우 이 기능을 사용하십시오.
want to replace (e.g., switch from full VINT8 to a mixed config, or update
교체를 원함(예: 전체 VINT8에서 혼합 구성으로 전환 또는 업데이트)
scale factors with better calibration data).
더 나은 교정 데이터를 사용하는 스케일 팩터).

Steps executed:
실행 단계:
1. **Strip quantization** — invoke the `/vai-dequantize` sub-skill to remove all
1. **스트립 양자화** — `/vai-deQuantize` 하위 기술을 호출하여 모든 것을 제거합니다.
existing QDQ nodes (QuantizeLinear, DequantizeLinear) and any `com.amd.quark`
기존 QDQ 노드(QuantizeLinear, DequalizeLinear) 및 `com.amd.quark`
quantization nodes.
양자화 노드.
This produces a clean FP32 model.
이렇게 하면 깨끗한 FP32 모델이 생성됩니다.
2. **Apply new quantization** — quantize the stripped model using the config
2. **새로운 양자화 적용** — 구성을 사용하여 제거된 모델을 양자화합니다.
specified by `--config`.
`--config`로 지정됩니다.
If `--calibration-data` is provided, use it for
`--calibration-data`가 제공되면 이를 사용하십시오.
calibration; otherwise Quark uses random data.
교정; 그렇지 않으면 Quark는 임의의 데이터를 사용합니다.
3. **Validate accuracy** — compare the newly quantized model against the FP32
3. **정확도 검증** — 새로 양자화된 모델을 FP32와 비교
baseline (the stripped model).
기준선(제거된 모델).
Report PSNR, pass rate, and per-output metrics.
PSNR, 통과율 및 출력당 지표를 보고합니다.
4. **Report improvement** — if previous accuracy metrics are available (from a
4. **개선 사항 보고** — 이전 정확도 지표를 사용할 수 있는 경우(
prior dry-run or full run), show the delta.
사전 테스트 실행 또는 전체 실행), 델타를 표시합니다.

**Example**:
**예**:
```bash
# Original model was quantized with full VINT8 and accuracy was poor.
# Requantize with backbone in VINT8, head in BF16:
/vai-quantization-guide --model model_vint8.onnx --mode requantize \
  --config backbone_int8_head_bf16.json --calibration-data calib.npy
```

**When to use**:
**사용 시기**:
- You got a quantized model from a colleague and want to try a different config
- 동료로부터 양자화된 모델을 얻었고 다른 구성을 시도하고 싶습니다.
- Your first quantization attempt failed accuracy checks
- 첫 번째 양자화 시도가 정확도 확인에 실패했습니다.
- You have better calibration data and want to re-calibrate
- 더 나은 교정 데이터가 있고 재교정을 원합니다.
- You want to switch between quantization strategies (e.g., full VINT8 → mixed BF16/VINT8)
- 양자화 전략(예: 전체 VINT8 → 혼합 BF16/VINT8) 간에 전환하고 싶습니다.

---

## Quark Availability
## 쿼크 가용성

**AMD Quark ships with the Vitis AI installation**, alongside `flexml` and the
**AMD Quark는 'flexml' 및 'flexml'과 함께 Vitis AI 설치와 함께 제공됩니다**
`VitisAIExecutionProvider`, so quantization, compilation and board execution all
`VitisAIExecutionProvider`, 즉 양자화, 컴파일 및 보드 실행이 모두 가능합니다.
run in the **same** environment.
**같은** 환경에서 실행하세요.
Before quantization can proceed, the skill
양자화가 진행되기 전에 스킬은
simply confirms Quark is importable there:
간단히 Quark를 가져올 수 있음을 확인합니다.

```bash
python -c "import quark" 2>/dev/null
```

If the import fails, the environment is not a complete Vitis AI environment.
가져오기에 실패하면 해당 환경은 완전한 Vitis AI 환경이 아닙니다.
**STOP and ask the user** how their Vitis AI environment should be set up (or
**중지하고 사용자에게** Vitis AI 환경을 어떻게 설정해야 하는지 물어보세요(또는
for the correct `--t` activate script) instead of searching the filesystem for
올바른 `--t` 활성화 스크립트를 위해) 파일 시스템을 검색하는 대신
another interpreter.
다른 통역사.
Only if the user explicitly asks to install it, install it
사용자가 명시적으로 설치를 요청하는 경우에만 설치합니다.
into that same environment:
동일한 환경으로:

```bash
pip install amd-quark
# or, for a specific version:
pip install amd-quark==0.12
```

> **Note**: Never go hunting elsewhere on the filesystem for Quark, and never
> **참고**: 절대로 파일 시스템의 다른 곳에서 Quark를 찾으러 가지 마세요.
> create a side `.venv` for it — mixing a separate Quark environment with the
> 이를 위해 `.venv` 측면을 생성합니다. 별도의 Quark 환경과
> Vitis AI one is what this skill deliberately avoids.
> 바이티스 AI 1번은 이 스킬이 의도적으로 피하는 것입니다.

---

## Parse Arguments
## 인수 구문 분석

This skill expects arguments in this form:
이 기술은 다음 형식의 인수를 기대합니다.

```
--model <onnx_model> [--t <activate_path>] [--config <quark_config.json>] [--device stx|ve2] [--target-accuracy <metric>] [--atol 0.015] [--rtol 0.01] [--mode full|dry-run|requantize] [--stage compile|exec] [--explore] [--calibration-data <path>] [--edge-quant none|input|output|both] [--edge-quant-auto] [--plugin <name> [plugin-args...]]
```

Extract from `$ARGUMENTS`:
`$ARGUMENTS`에서 추출:
- `--model <path>`: **(MANDATORY)** Path to input ONNX model (.onnx or .onnxtxt)
- `--model <경로>`: **(필수)** ONNX 모델 입력 경로(.onnx 또는 .onnxtxt)
- `--t <path>`: **(OPTIONAL)** Path to a Vitis AI environment activate script, used for the compile / board steps.
- `--t <경로>`: **(선택 사항)** 컴파일/보드 단계에 사용되는 Vitis AI 환경 활성화 스크립트의 경로입니다.
- Provide this only when the Vitis AI Python environment is NOT already on the shell.
- Vitis AI Python 환경이 셸에 아직 없는 경우에만 이를 제공하세요.
- Omit it when the environment is pre-provisioned (e.g. inside the Vitis AI Docker container, where `flexml` and the `VitisAIExecutionProvider` are already importable).
- 환경이 사전 프로비저닝된 경우 이를 생략합니다(예: `flexml` 및 `VitisAIExecutionProvider`를 이미 가져올 수 있는 Vitis AI Docker 컨테이너 내부).
- See [Environment Setup Convention](#environment-setup-convention).
- [환경 설정 규칙](#environment-setup-convention)을 참조하세요.
- `--config <path>`: **(OPTIONAL)** Path to Quark JSON configuration preset
- `--config <경로>`: **(선택 사항)** Quark JSON 구성 사전 설정 경로
- `--device <stx|ve2>`: **(OPTIONAL, default: ve2)** Target device (stx=Strix/BF16, ve2=Telluride/FP16)
- `--device <stx|ve2>`: **(선택 사항, 기본값: ve2)** 대상 장치(stx=Strix/BF16, ve2=Telluride/FP16)
- `--target-accuracy <metric>`: **(OPTIONAL)** Target accuracy metric (e.g., "psnr>30")
- `--target-accuracy <metric>`: **(선택사항)** 목표 정확도 측정항목(예: "psnr>30")
- `--atol <float>`: **(OPTIONAL, default: 0.015)** Absolute tolerance for output comparison
- `--atol <float>`: **(선택 사항, 기본값: 0.015)** 출력 비교에 대한 절대 허용 오차
- `--rtol <float>`: **(OPTIONAL, default: 0.01)** Relative tolerance (percent) for output comparison
- `--rtol <float>`: **(선택 사항, 기본값: 0.01)** 출력 비교에 대한 상대 허용 오차(퍼센트)
- `--mode <full|dry-run|requantize>`: **(OPTIONAL, default: full)** Execution mode
- `--mode <full|dry-run|reQuantize>`: **(선택 사항, 기본값: 전체)** 실행 모드
- `--stage <compile|exec>`: **(OPTIONAL, default: compile)** Stage to execute
- `--stage <compile|exec>`: **(선택 사항, 기본값: 컴파일)** 실행할 단계
- `--explore`: **(OPTIONAL)** Enable exploration mode to find optimal configuration
- `--explore`: **(선택 사항)** 최적의 구성을 찾으려면 탐색 모드를 활성화하세요.
- `--calibration-data <path>`: **(OPTIONAL)** Calibration data (.npy or .npz files)
- `--calibration-data <path>`: **(선택 사항)** 교정 데이터(.npy 또는 .npz 파일)
- `--edge-quant <none|input|output|both>`: **(OPTIONAL, default: none)** Control edge quantization in runtime:
- `--edge-Quant <none|input|output|both>`: **(선택 사항, 기본값: 없음)** 런타임 시 가장자리 양자화 제어:
  - `none`: No edge quantization (Q/DQ at boundaries become NPU kernels)
  - `none`: 에지 양자화 없음(경계의 Q/DQ가 NPU 커널이 됨)
  - `input`: Enable input-side quantization in RT (`input-quantization-in-rt=1`)
  - `input`: RT에서 입력 측 양자화 활성화(`input-퀀트화-in-rt=1`)
  - `output`: Enable output-side dequantization in RT (`output-dequantization-in-rt=1`)
  - `output`: RT에서 출력 측 역양자화 활성화(`output-deQuantization-in-rt=1`)
  - `both`: Enable both input and output (`edge-quantization-in-rt=1`)
  - `both`: 입력과 출력을 모두 활성화합니다(`edge-Quantization-in-rt=1`).
- `--edge-quant-auto`: **(OPTIONAL)** Automatically detect whether edge quantization is beneficial by compiling twice (with and without) and comparing kernel counts
- `--edge-Quant-auto`: **(선택 사항)** 두 번 컴파일하고(포함 및 제외) 커널 수를 비교하여 에지 양자화가 유익한지 자동으로 감지합니다.
- `--plugin <name> [plugin-args...]`: **(OPTIONAL)** Invoke an optional plugin skill by name (see [Plugin Skills](#plugin-skills)).
- `--plugin <name> [plugin-args...]`: **(선택 사항)** 이름으로 선택적 플러그인 기술을 호출합니다([플러그인 기술](#plugin-skills) 참조).
- The plugin is discovered at runtime; any arguments after the name are passed through to the plugin's own sub-commands/options.
- 플러그인은 런타임 시 검색됩니다. 이름 뒤의 모든 인수는 플러그인 자체 하위 명령/옵션으로 전달됩니다.

## Environment Setup Convention
## 환경 설정 규칙

The **whole workflow runs in a single environment**: a Vitis AI installation
**전체 워크플로우는 단일 환경**에서 실행됩니다: Vitis AI 설치
provides `quark` for quantization together with `flexml` and the
`flexml`과 함께 양자화를 위한 `quark`를 제공하며
`VitisAIExecutionProvider` for compilation and board execution.
컴파일 및 보드 실행을 위한 `VitisAIExecutionProvider`.
There is no
없다
separate quantization venv and no environment hand-off between phases.
별도의 양자화 장치가 있고 단계 간 환경 핸드오프가 없습니다.

This skill supports two deployment models for that environment:
이 기술은 해당 환경에 대해 두 가지 배포 모델을 지원합니다.

1. **Sourced environment**: the user passes `--t <path>` and every shell the
1. **소스 환경**: 사용자는 `--t <path>`를 전달하고 모든 쉘은
skill runs must `source <path>` first.
스킬 실행은 `소스 <경로>`를 먼저 실행해야 합니다.
2. **Pre-provisioned environment** (e.g. the Vitis AI Docker container): the
2. **사전 프로비저닝된 환경**(예: Vitis AI Docker 컨테이너):
Python packages and binaries are already on `PATH` / `PYTHONPATH`.
Python 패키지와 바이너리는 이미 `PATH` / `PYTHONPATH`에 있습니다.
`--t` is
`--t`는
NOT provided; no sourcing is needed.
제공되지 않습니다. 소싱이 필요하지 않습니다.

Wherever this document shows `<ENV_SETUP>` inside a bash snippet, expand it as
이 문서가 bash 조각 내에서 `<ENV_SETUP>`을 표시할 때마다 다음과 같이 확장합니다.
follows:
다음과 같습니다:
- If `--t <path>` was provided: replace the `<ENV_SETUP>` line with `source <path>`.
- `--t <path>`가 제공된 경우: `<ENV_SETUP>` 줄을 `source <path>`로 바꿉니다.
- Otherwise: delete the `<ENV_SETUP>` line entirely.
- 그렇지 않은 경우: `<ENV_SETUP>` 줄을 완전히 삭제하세요.

**Do NOT auto-detect or guess an activate script.** If `--t` is absent, trust
**활성화 스크립트를 자동 감지하거나 추측하지 마세요.** `--t`가 없으면 신뢰하세요.
that the calling shell already has `python3 -c "import flexml, quark"` working.
호출 셸에는 이미 `python3 -c "import flexml, quark"`가 작동하고 있습니다.
Searching the filesystem for an environment to source is a bug, not a fallback.
파일 시스템에서 소스 환경을 검색하는 것은 폴백이 아니라 버그입니다.

Before the first compile, verify the toolchain is usable:
첫 번째 컴파일 전에 툴체인을 사용할 수 있는지 확인하세요.

```bash
<ENV_SETUP>
python3 -c "import onnxruntime as ort; \
  assert 'VitisAIExecutionProvider' in ort.get_available_providers(); print('VitisAI EP OK')"
```

If that fails, **STOP and ask the user** how the Vitis AI environment should be
실패하면 **중지하고 사용자에게** Vitis AI 환경이 어떻게 되어야 하는지 물어보세요.
set up (or for the correct `--t` activate script) rather than hunting for one.
찾기보다는 설정(또는 올바른 `--t` 활성화 스크립트를 위해)하세요.

The VAIML frontend/partitioner also expects these knobs, so export them in the
VAIML 프런트엔드/파티셔너도 이러한 손잡이를 예상하므로
compile shell:
쉘 컴파일:

```bash
export DEBUG_VAIML_PARTITION=2
export DEBUG_LOG_LEVEL=info
export BF16_SELECT=True
export ADD_TARGET_TO_CFG=True
```

## Board Execution: Ask First (Mandatory)
## 이사회 실행: 먼저 물어보세요(필수)

**Before running any part of the workflow, establish whether the user wants to
**워크플로우의 일부를 실행하기 전에 사용자가 원하는지 여부를 설정하십시오.
execute on a board and how.** Do not defer this to the end of Phase 6 — the
보드에서 실행하는 방법.** 이를 6단계 끝까지 미루지 마십시오.
answer determines whether accuracy can be confirmed on real hardware or only
대답은 실제 하드웨어에서 정확성을 확인할 수 있는지 아니면 하드웨어에서만 확인할 수 있는지를 결정합니다.
estimated on CPU.
CPU로 추정됩니다.
Ask:
질문하세요:

> "Do you want to run the quantized model on a board?
> "양자화된 모델을 보드에서 실행하시겠습니까?
> 1. **No** — compile only; accuracy is validated on CPU against the FP32 reference.
> 1. **아니요** — 컴파일 전용입니다. 정확성은 FP32 참조를 기준으로 CPU에서 검증됩니다.
> 2. **Yes, using the bundled board-run script** — please provide board address, user name,
> 2. **예, 번들로 제공되는 보드 실행 스크립트를 사용합니다** — 보드 주소, 사용자 이름,
> password, key file or SSH_AUTH_SOCK that I need to connect.
> 연결해야 하는 비밀번호, 키 파일 또는 SSH_AUTH_SOCK.
> 3. **Yes** — tell me how to reach and run on your board (host, credentials/SSH
> 3. **예** - 보드에 접근하고 실행하는 방법을 알려주세요(호스트, 자격 증명/SSH
> config, board type, and the script/harness/command you use), and I will use that."
> 구성, 보드 유형 및 사용하는 스크립트/하네스/명령)를 사용하겠습니다."


Resolution rules:
해결 규칙:
- If the user picks option 3, capture the exact command/harness now and use it
- 사용자가 옵션 3을 선택한 경우 지금 정확한 명령/하네스를 캡처하여 사용하세요.
verbatim in Phase 6 — do not guess a board command.
6단계에서는 그대로 — 보드 명령을 추측하지 마세요.
- If the user picks option 1, say plainly that accuracy claims will rest on CPU
- 사용자가 옵션 1을 선택하는 경우 정확성 주장은 CPU에 달려 있다고 분명히 말하십시오.
reference comparisons only, and that on-hardware behavior is unverified.
참조 비교만 해당되며 하드웨어에서의 동작은 확인되지 않았습니다.
- **Headless (no interactive user):** don't block — proceed compile-only and
- **헤드리스(대화형 사용자 없음):** 차단하지 않음 — 컴파일만 진행하고
report that board execution was not performed.
보드 실행이 수행되지 않았음을 보고합니다.

## Static Shape Preservation (Mandatory)
## 정적 형상 보존(필수)

The VAIML compiler requires all tensor shapes to be fully static (no dynamic
VAIML 컴파일러에서는 모든 텐서 형태가 완전히 정적이어야 합니다(동적 형태가 아님).
dimensions).
치수).
When the input model has only static shapes, the quantized output
입력 모델에 정적 모양만 있는 경우 양자화된 출력은
model **must** also have only static shapes.
모델에는 **반드시** 정적 모양만 있어야 합니다.

> The graph-surgery steps described below apply **only** within the two permitted
> 아래 설명된 그래프 수술 단계는 허용된 두 가지 범위 내에서만 **만** 적용됩니다.
> exceptions of the [Model Modification Policy](#model-modification-policy-mandatory)
> [모델 수정 정책] 예외(#model-modification-policy-mandatory)
> (the `vai-vaip-patching` skill and the ViT MHA-MX9 pipeline).
> ('vai-vaip-patching' 기술 및 ViT MHA-MX9 파이프라인).
> They are **not** a
> 그들은 **아닙니다**
> license to hand-edit models in the general case — outside those exceptions,
> 일반적인 경우에 모델을 직접 편집할 수 있는 라이선스(해당 예외 제외)
> shape/precision problems are fixed by adjusting the Quark config and
> 모양/정밀도 문제는 Quark 구성을 조정하여 해결되며
> requantizing, not by editing the graph.
> 그래프를 편집하는 것이 아니라 재양자화하는 것입니다.

> **RULE**: Never introduce dynamic or unranked tensor dimensions during
> **규칙**: 동적 또는 순위가 지정되지 않은 텐서 차원을 도입하지 마세요.
> quantization.
> 양자화.
> If the input model has shape `[1, 1024, 1536]`, every
> 입력 모델의 모양이 '[1, 1024, 1536]'인 경우,
> intermediate and output tensor in the quantized model must also have fully
> 양자화된 모델의 중간 및 출력 텐서는 또한 완전히 있어야 합니다.
> resolved static shapes.
> 해결된 정적 모양.
> Violating this causes the VAIML frontend to emit
> 이를 위반하면 VAIML 프런트엔드에서 방출이 발생합니다.
> `WARNING: [VAIML-FRONTEND 2018] Operation/s with NON-STATIC SHAPE have been
> `경고: [VAIML-FRONTEND 2018] 비정적 형태의 작업이 수행되었습니다.
> detected in MODEL` and crash during ONNX-to-TOSA lowering.
> MODEL`에서 감지되고 ONNX에서 TOSA로 낮추는 동안 충돌이 발생합니다.

**Checks to enforce**:
**시행할 확인사항**:
1. After every graph surgery step (node insertion, removal, rewiring), run
1. 모든 그래프 수술 단계(노드 삽입, 제거, 재배선) 후에 다음을 실행합니다.
`onnx.shape_inference.infer_shapes(model, data_prop=True)` to propagate
전파할 `onnx.shape_inference.infer_shapes(model, data_prop=True)`
shapes through the modified graph.
수정된 그래프를 통해 모양을 만듭니다.
2. After saving the final model, verify no tensor has unknown dimensions:
2. 최종 모델을 저장한 후 알 수 없는 차원이 있는 텐서가 없는지 확인하십시오.
   ```python
   for vi in model.graph.value_info:
       shape = vi.type.tensor_type.shape
       for dim in shape.dim:
           assert dim.dim_value > 0, f"Dynamic dim in {vi.name}"
   ```
3. Custom ops (`com.amd.quark` domain) are not handled by
3. 사용자 정의 작업(`com.amd.quark` 도메인)은 다음에서 처리되지 않습니다.
`onnx.shape_inference`.
'onnx.shape_inference'.
After inserting any `com.amd.quark` quantization
`com.amd.quark` 양자화를 삽입한 후
nodes, manually propagate shapes: output shape = first input shape.
노드, 수동으로 모양 전파: 출력 모양 = 첫 번째 입력 모양.
4. When splitting, slicing, or reshaping weight tensors (e.g., QKV split),
4. 가중치 텐서를 분할, 슬라이싱 또는 재구성할 때(예: QKV 분할),
compute the output shapes explicitly from the input tensor dimensions —
입력 텐서 차원에서 명시적으로 출력 형태를 계산합니다.
never use `-1` or symbolic dimensions.
절대로 '-1'이나 기호 차원을 사용하지 마세요.

## Default Tolerances (XOAH Defaults)
## 기본 공차(XOAH 기본값)

These tolerances are the standard acceptance criteria for **regression-type**
이러한 허용 오차는 **회귀 유형**에 대한 표준 허용 기준입니다.
models (where output values are directly consumed):
모델(출력 값이 직접 소비되는 경우):
- **atol**: 0.015 (absolute tolerance)
- **atol**: 0.015(절대 허용오차)
- **rtol**: 0.01 (1% relative tolerance)
- **rtol**: 0.01(상대 공차 1%)
- **PSNR threshold**: 30 dB (for models whose output values are consumed directly, e.g. segmentation masks, super-resolution pixels, embeddings)
- **PSNR 임계값**: 30dB(분할 마스크, 초해상도 픽셀, 임베딩과 같이 출력 값이 직접 소비되는 모델의 경우)

Users may override these, but the defaults represent industry-standard acceptance
사용자는 이를 무시할 수 있지만 기본값은 업계 표준 수용을 나타냅니다.
criteria for quantized model deployment on NPU.
NPU의 양자화된 모델 배포 기준.

**These tolerances do NOT apply to classification models.** For classification,
**이러한 공차는 분류 모델에는 적용되지 않습니다.** 분류의 경우
the pass/fail criterion is argmax preservation (top-1 or top-k match).
통과/실패 기준은 argmax 보존(top-1 또는 top-k 일치)입니다.
A
classification model with PSNR=10 dB but correct top-1 is a PASS; a regression
PSNR=10dB이지만 올바른 top-1을 갖는 분류 모델은 PASS입니다. 회귀
model with PSNR=25 dB is a FAIL regardless of argmax.
PSNR=25dB인 모델은 argmax에 관계없이 FAIL입니다.
Always check the task
항상 과제를 확인하세요
type first (Phase 0, step 3) before applying any threshold.
임계값을 적용하기 전에 먼저 입력하십시오(단계 0, 3단계).

## Quantization Pipeline: Native Quark vs.
## 양자화 파이프라인: 기본 Quark와 Quark
## Manual FE Pipeline
## 수동 FE 파이프라인

This skill supports two approaches.
이 기술은 두 가지 접근 방식을 지원합니다.
Choose the one appropriate for your model:
귀하의 모델에 적합한 것을 선택하십시오:

### Approach A: Native Quark Quantization (Preferred)
### 접근 방식 A: 기본 Quark 양자화(선호)

Use Quark directly on the ONNX model to apply quantization and compare
ONNX 모델에서 직접 Quark를 사용하여 양자화 적용 및 비교
numerical accuracy.
수치적 정확성.
Quark natively supports mixed-precision quantization with
Quark는 기본적으로 혼합 정밀도 양자화를 지원합니다.
per-layer configuration, sensitivity analysis, and accuracy comparison against
레이어별 구성, 민감도 분석 및 정확도 비교
the FP32 baseline — all in one step, without manual patching.
FP32 기준 — 수동 패치 없이 모두 한 단계로 완료됩니다.

**When to use**: This is the default and preferred approach.
**사용 시기**: 이것이 기본이자 선호되는 접근 방식입니다.
Quark operates
쿼크는 작동한다
directly on the ONNX graph and produces a quantized model ready for
ONNX 그래프에 직접 입력하여 즉시 사용할 수 있는 양자화된 모델을 생성합니다.
compilation.
편집.
Use this whenever FE (Front-End compiler) transformations are
FE(프런트엔드 컴파일러) 변환이 필요할 때마다 이것을 사용하십시오.
**not** required before quantization.
**아닙니다** 양자화 전에는 필요하지 않습니다.

### Approach B: FE-Transformed Pipeline
### 접근법 B: FE 변환 파이프라인

Apply FE transformations first, then quantize the transformed model.
먼저 FE 변환을 적용한 다음 변환된 모델을 양자화합니다.
This
이
pipeline is only necessary when FE performs quantization-sensitive
파이프라인은 FE가 양자화에 민감한 작업을 수행하는 경우에만 필요합니다.
transformations that alter the graph structure (e.g., op fusions, layout
그래프 구조를 변경하는 변환(예: 연산 융합, 레이아웃
changes, constant folding) in ways that affect how quantization should be
변경, 지속적인 폴딩) 양자화가 어떻게 이루어져야 하는지에 영향을 미칩니다.
applied.
적용되었습니다.

**When to use**: Only when the model requires FE transformations that:
**사용 시기**: 모델에 다음과 같은 FE 변환이 필요한 경우에만
- Fuse operations (e.g., Conv+BN, Conv+Relu) before quantization boundaries
- 양자화 경계 이전의 융합 연산(예: Conv+BN, Conv+Relu)
- Change tensor layouts or shapes in ways that affect quantization ranges
- 양자화 범위에 영향을 미치는 방식으로 텐서 레이아웃이나 모양을 변경합니다.
- Apply optimizations that Quark cannot replicate on the original ONNX
- Quark가 원본 ONNX에 복제할 수 없는 최적화 적용

**Risk**: If FE transformations introduce graph changes not representable back
**위험**: FE 변환으로 인해 다시 표현할 수 없는 그래프 변경이 발생하는 경우
in ONNX, then the quantization applied in this pipeline comes from the
ONNX에서 이 파이프라인에 적용된 양자화는
**original** (pre-transformation) model, not the transformed one.
변형된 모델이 아닌 **원래**(변환 전) 모델입니다.
This
이
mismatch can introduce additional error.
불일치로 인해 추가 오류가 발생할 수 있습니다.
Monitor for this by comparing the
비교하여 이를 모니터링합니다.
native Quark quantization output with the FE-pipeline output — if they
FE 파이프라인 출력을 사용한 기본 Quark 양자화 출력 — 만약 그렇다면
diverge significantly, the FE transformations are affecting quantization
크게 발산하면 FE 변환이 양자화에 영향을 미치고 있습니다.
ranges and the FE-transformed pipeline is warranted.
범위 및 FE 변환 파이프라인이 보증됩니다.

**Recommendation**: Start with Approach A (native Quark).
**권장 사항**: 접근 방식 A(기본 Quark)로 시작하세요.
Only fall back to
다음으로만 복귀
Approach B if accuracy results on the compiled model diverge from what Quark
컴파일된 모델의 정확도 결과가 Quark의 정확도와 다른 경우 접근 방식 B
predicts, indicating that FE transformations are changing quantization
FE 변환이 양자화를 변화시키고 있음을 나타내는 예측
behavior.
행동.

## Workflow Phases
## 워크플로 단계

### Phase 0: Model Analysis
### 0단계: 모델 분석

1. **Load and inspect the model**:
1. **모델 로드 및 검사**:
   ```python
   import onnx
   model = onnx.load(model_path)
   ```

2. **Determine model state**:
2. **모델 상태 확인**:
   - Check for QuantizeLinear/DequantizeLinear nodes → quantized (legacy QDQ)
   - QuantizeLinear/DeQuantizeLinear 노드 확인 → 양자화(레거시 QDQ)
   - Check for `com.amd.quark` domain quantization ops → quantized by a
   - `com.amd.quark` 도메인 양자화 작업을 확인하세요 →
specialized method.
특화된 방법.
If a matching plugin skill is available under
일치하는 플러그인 기술을 사용할 수 있는 경우
`skills/experimental-methods/`, delegate its analysis to that plugin
`skills/experimental-methods/`, 해당 플러그인에 분석 위임
(see [Plugin Skills](#plugin-skills)); otherwise report that the model
([플러그인 스킬](#plugin-skills) 참조); 그렇지 않으면 모델이
uses a specialized quantization not handled by the core pipeline.
핵심 파이프라인에서 처리되지 않는 특수한 양자화를 사용합니다.
   - No quantization nodes → non-quantized (float model)
   - 양자화 노드 없음 → 양자화되지 않음(부동 모델)

3. **Classify the model's task type** (determines which accuracy metric to use):
3. **모델의 작업 유형 분류**(사용할 정확도 측정항목 결정):

| Task type<br>작업 유형 | How to identify<br>식별 방법 | Primary accuracy metric<br>기본 정확도 측정항목 |
   |-----------|----------------|------------------------|
| **Classification**<br>**분류** | Output is a single tensor of shape `[N, num_classes]` (logits or probabilities). Typical indicators: final compute ops include Softmax, Gemm, MatMul, or GlobalAveragePool feeding a dense layer; output has no spatial dimensions.<br>출력은 `[N, num_classes]`(로지트 또는 확률) 형태의 단일 텐서입니다. 일반적인 지표: 최종 컴퓨팅 작업에는 밀도가 높은 레이어를 공급하는 Softmax, Gemm, MatMul 또는 GlobalAveragePool이 포함됩니다. 출력에는 공간 차원이 없습니다. | **Top-k accuracy**: does the argmax (top-1) or top-5 set match FP32? Element-wise logit differences are irrelevant.<br>**Top-k 정확도**: argmax(top-1) 또는 top-5 세트가 FP32와 일치합니까? 요소별 로짓 차이는 관련이 없습니다. |
| **Detection**<br>**탐지** | Multiple outputs (boxes, scores, classes); ops like NonMaxSuppression, ScatterND, TopK near output<br>다중 출력(박스, 점수, 클래스) NonMaxSuppression, ScatterND, TopK와 같은 연산은 출력 근처에 있습니다. | **mAP / IoU**: do bounding boxes and class predictions match FP32 within standard detection tolerances?<br>**mAP / IoU**: 경계 상자 및 클래스 예측이 표준 감지 허용 오차 내에서 FP32와 일치합니까? |
| **Regression / Generation**<br>**회귀/생성** | Output values are directly consumed (pixel values, embeddings, coordinates, waveforms)<br>출력 값이 직접 소비됩니다(픽셀 값, 임베딩, 좌표, 파형). | **Element-wise PSNR + atol/rtol**: the numeric value itself matters.<br>**요소별 PSNR + atol/rtol**: 숫자 값 자체가 중요합니다. |

**Determine the task type before running any accuracy check.** All
**정확성 검사를 실행하기 전에 작업 유형을 결정하십시오.** 모두
subsequent pass/fail decisions use the metric that matches the task type.
후속 합격/실패 결정은 작업 유형과 일치하는 측정항목을 사용합니다.
Do NOT reject a classification model because its element-wise PSNR or
요소별 PSNR 또는
atol/rtol is poor; it passes if argmax is preserved, regardless of logit
atol/rtol은 좋지 않습니다. 로짓에 관계없이 argmax가 보존되면 통과합니다.
magnitude differences.
크기 차이.
Do NOT accept a regression model without verifying
검증 없이 회귀 모델을 받아들이지 마십시오.
element-wise tolerance; argmax preservation is meaningless when the output
요소별 허용오차; argmax 보존은 출력시 의미가 없습니다.
values themselves are consumed.
가치 자체가 소비됩니다.

When the task type is ambiguous, infer from output shape and context:
작업 유형이 모호한 경우 출력 형태와 컨텍스트에서 추론합니다.
   - Output `[N, C]` where C is large (100-10000) and no spatial dims → classification
   - C가 크고(100-10000) 공간 디밍이 없는 `[N, C]` 출력 → 분류
   - Output `[N, C, H, W]` where H, W > 1 → regression (segmentation, generation)
   - 출력 `[N, C, H, W]` 여기서 H, W > 1 → 회귀(분할, 생성)
   - Multiple outputs with mixed shapes → detection or multi-task (check each output independently)
   - 형태가 혼합된 다중 출력 → 감지 또는 다중 작업(각 출력을 독립적으로 확인)

4. **Report model characteristics**:
4. **보고서 모델 특성**:
   - Total nodes, op types present
   - 총 노드, 작업 유형 존재
   - Input/output shapes and dtypes
   - 입력/출력 모양 및 dtype
   - **Task type** (classification / detection / regression)
   - **작업 유형**(분류/탐지/회귀)
   - Quantization coverage (% of nodes quantized)
   - 양자화 범위(양자화된 노드의 %)
   - Island structure (contiguous quantized/float regions)
   - 섬 구조(인접한 양자화/부동 영역)

### Phase 1: Baseline Establishment
### 1단계: 기준선 확립

#### Sample count requirements
#### 샘플 수 요구 사항

**Calibration**: Use at least 4 representative samples (more is better with
**보정**: 최소 4개의 대표 샘플을 사용합니다.
real data; with random data, more samples can widen the MinMax range and hurt
실제 데이터; 무작위 데이터를 사용하면 더 많은 샘플이 MinMax 범위를 넓히고 손상될 수 있습니다.
VINT8 precision).
VINT8 정밀).
If `--calibration-data` is not provided, use
`--calibration-data`가 제공되지 않은 경우 다음을 사용하세요.
`UseRandomData: True` with a fixed seed for reproducibility.
재현성을 위해 고정 시드가 있는 'UseRandomData: True'입니다.

**Evaluation**: Use at least 100 samples for initial validation, 1000 for
**평가**: 초기 검증에는 최소 100개, 검증에는 1000개 이상의 샘플을 사용합니다.
final reporting.
최종보고.
A single sample is never sufficient; classification models
단일 샘플로는 결코 충분하지 않습니다. 분류 모델
on random inputs have small logit margins, so single-sample results are
무작위 입력에 대한 로짓 마진은 작으므로 단일 표본 결과는 다음과 같습니다.
dominated by noise.
소음이 지배적입니다.
Always evaluate on a fixed seed for reproducibility.
재현성을 위해 항상 고정된 시드를 기준으로 평가하십시오.

For **non-quantized models**, establish accuracy baselines:
**비양자화 모델**의 경우 정확도 기준을 설정합니다.

1. **FP32 CPU Reference**: Run model on CPU with ONNXRuntime CPUExecutionProvider
1. **FP32 CPU 참조**: ONNXRuntime CPUExecutionProvider를 사용하여 CPU에서 모델 실행
2. **Full BF16 Baseline** (or FP16 for ve2): Quantize entire model to BF16/FP16
2. **전체 BF16 기준선**(또는 ve2의 경우 FP16): 전체 모델을 BF16/FP16으로 양자화
   - This is the accuracy ceiling for NPU execution
   - 이는 NPU 실행의 정확도 한도입니다.
   - Use config: `full_bf16.json`
   - 구성 사용: `full_bf16.json`
   - Compare to FP32: should have PSNR > 40dB typically
   - FP32와 비교: 일반적으로 PSNR > 40dB를 가져야 합니다.

3. **Full VINT8 Baseline**: Quantize entire model to INT8
3. **전체 VINT8 기준선**: 전체 모델을 INT8로 양자화
   - This is the performance ceiling
   - 성능 상한선입니다
   - Use config: `full_vint8.json`
   - 구성 사용: `full_vint8.json`
   - Compare to FP32: check against atol/rtol thresholds
   - FP32와 비교: atol/rtol 임계값에 대해 확인

**Decision Point** (apply the metric determined by task type in Phase 0):
**결정 지점**(0단계의 작업 유형에 따라 결정된 측정항목 적용):

- **Classification models**: If top-1 prediction matches FP32 → **DONE**.
- **분류 모델**: 상위 1개 예측이 FP32와 일치하는 경우 → **완료**.
Do NOT proceed to exploration based on element-wise PSNR or atol/rtol
요소별 PSNR 또는 atol/rtol을 기반으로 탐색을 진행하지 마세요.
failures alone.
혼자 실패.
Logit magnitudes can differ substantially without affecting
로짓 크기는 영향을 주지 않고 크게 달라질 수 있습니다.
the decision boundary.
결정 경계.
Only explore if the argmax changes.
argmax가 변경되는 경우에만 탐색하세요.
- **Detection models**: If bounding box IoU and class predictions match FP32
- **탐지 모델**: 경계 상자 IoU 및 클래스 예측이 FP32와 일치하는 경우
(compute via model-specific post-processing, e.g.
(모델별 후처리를 통해 계산합니다.
NMS, then compare box
NMS, 비교 상자
coordinates with IoU ≥ 0.5 and class IDs matching).
IoU ≥ 0.5 및 클래스 ID 일치와 함께 조정됩니다.
If detection accuracy
탐지 정확도가 있는 경우
cannot be evaluated in this environment, report it as requiring external
이 환경에서 평가할 수 없는 경우 외부 요구 사항으로 보고하세요.
validation and proceed to compilation without exploration. → **DONE**.
검증하고 탐색 없이 컴파일을 진행합니다. → **완료**.
- **Regression models**: If element-wise metrics meet thresholds (PSNR ≥ 30 dB
- **회귀 모델**: 요소별 측정항목이 임계값을 충족하는 경우(PSNR ≥ 30dB)
AND pass rate ≥ 99.9% at configured atol/rtol) → **DONE**.
AND 구성된 atol/rtol에서 통과율 ≥ 99.9%) → **완료**.
- Otherwise → proceed to Phase 2 (mixed-precision exploration).
- 그렇지 않은 경우 → 2단계(혼합 정밀도 탐색)로 진행합니다.

### Phase 2: Mixed-Precision Strategy Selection
### 2단계: 혼합 정밀도 전략 선택

Based on accuracy analysis, select a quantization strategy:
정확도 분석을 기반으로 양자화 전략을 선택합니다.

#### Strategy A: Legacy Mixed-Precision (Multi-Island)
#### 전략 A: 레거시 혼합 정밀도(다중 섬)

Use when: Floating-point operations form contiguous regions (heads/tails/middles).
사용 시기: 부동 소수점 연산이 연속 영역(머리/꼬리/가운데)을 형성합니다.

Patterns supported:
지원되는 패턴:
- `vint8 → bf16 tail` (quantized core, floating output)
- `vint8 → bf16 tail`(양자화 코어, 플로팅 출력)
- `bf16 head → vint8` (floating input, quantized core)
- `bf16 head → vint8`(플로팅 입력, 양자화된 코어)
- `bf16 → vint8 → bf16` (floating head+tail, quantized middle)
- `bf16 → vint8 → bf16` (플로팅 헤드+테일, 양자화된 중간)
- `vint8 → bf16 → vint8` (quantized head+tail, floating middle)
- `vint8 → bf16 → vint8` (양자화된 머리+꼬리, 부동 중간)
- Four-island+ models (alternating regions)
- 4개 섬+ 모델(교대 지역)

Configuration: Uses `group_args: "mixed_precision"` flag which enables:
구성: 다음을 활성화하는 `group_args: "mixed_precision"` 플래그를 사용합니다.
- `match-power-of-two-quant-kernel=1` — QDQ-aware FP conversion
- `match-power-of-two-Quant-kernel=1` — QDQ 인식 FP 변환

#### Strategy B: Specialized Quantization via Plugins
#### 전략 B: 플러그인을 통한 전문화된 양자화

Some models benefit from specialized quantization formats or per-layer
일부 모델은 특수한 양자화 형식 또는 레이어별 이점을 얻습니다.
precision schemes that are not part of the core BF16/VINT8 pipeline (for
핵심 BF16/VINT8 파이프라인의 일부가 아닌 정밀 구성표(용
example, block floating-point formats or extended per-layer precision for
예를 들어, 부동 소수점 형식을 차단하거나 레이어별 확장 정밀도를
frequently-interleaved floating-point layers).
자주 인터리빙되는 부동 소수점 레이어).

These methods are provided as **optional plugin skills** discovered at runtime.
이러한 메서드는 런타임 시 발견되는 **선택적 플러그인 기술**로 제공됩니다.
When a model's accuracy or structure calls for such a method, check whether a
모델의 정확성이나 구조로 인해 그러한 방법이 필요한 경우
matching plugin is available and, if so, delegate to it via the plugin facility
일치하는 플러그인을 사용할 수 있으며, 그렇다면 플러그인 기능을 통해 해당 플러그인에 위임하세요.
(see [Plugin Skills](#plugin-skills)).
([플러그인 스킬](#plugin-skills) 참조).
Each plugin defines its own scope,
각 플러그인은 자체 범위를 정의합니다.
hardware constraints, experimental-feature consent banners, version checks,
하드웨어 제약, 실험적 기능 동의 배너, 버전 확인,
and sub-commands/options — the core orchestrator does not hard-code any of
및 하위 명령/옵션 - 핵심 오케스트레이터는 다음 중 어떤 것도 하드 코딩하지 않습니다.
these.
이것들.

### Phase 3: Quantization Execution
### 3단계: 양자화 실행

1. **If model is already quantized and mode=requantize**:
1. **모델이 이미 양자화되어 있고 모드=재양자화**인 경우:
   - Invoke `/vai-dequantize` to strip existing quantization
   - 기존 양자화를 제거하려면 `/vai-deQuantize`를 호출하세요.
   - Proceed with fresh quantization
   - 새로운 양자화를 진행하세요

2. **Apply selected strategy**:
2. **선택한 전략 적용**:
   ```bash
   python mixed_precision_quantize.py \
       --input <model.onnx> \
       --output <model_quantized.onnx> \
       --config <selected_config.json> \
       [--calibration-data <data.npz>] \
       [--fp-nodes "node1,node2"] \
       [--fp-precision bf16|fp16]
   ```

Note: `--fp-nodes` accepts exact ONNX node names (as shown in the model
참고: `--fp-nodes`는 정확한 ONNX 노드 이름을 허용합니다(모델에 표시된 대로).
graph, e.g., `/layer4/layer4.1/conv2/Conv`).
그래프(예: `/layer4/layer4.1/conv2/Conv`).
These are passed as exact
이는 정확한 것으로 전달됩니다.
string matches to Quark's `specific_layer_config`.
문자열은 Quark의 `특정_레이어_구성`과 일치합니다.
For regex patterns
정규식 패턴의 경우
(e.g., `^/layer4/.*`), use the JSON config file's `target_node_names`
(예: `^/layer4/.*`), JSON 구성 파일의 `target_node_names`를 사용하세요.
field instead.
대신 필드.

3. **If exploration mode is enabled**:
3. **탐색 모드가 활성화된 경우**:
   - Iteratively try different layer assignments
   - 다양한 레이어 할당을 반복적으로 시도해 보세요.
   - Use sensitivity analysis to identify accuracy-critical layers
   - 민감도 분석을 사용하여 정확도가 중요한 레이어 식별
   - Move the most sensitive layers to higher precision first
   - 가장 민감한 레이어를 더 높은 정밀도로 먼저 이동하세요.

### Phase 4: Model Patching (if needed)
### 4단계: 모델 패치(필요한 경우)

For models with `com.amd.quark` custom ops targeting VAIP (not VAIML directly):
VAIP를 대상으로 하는 `com.amd.quark` 사용자 정의 작업이 있는 모델의 경우(VAIML이 직접 아님):
- Invoke `/vai-vaip-patch` to add shape/data_type attributes
- Shape/data_type 속성을 추가하려면 `/vai-vaip-patch`를 호출하세요.
- Fix zero-point dtypes (bf16/fp16 not representable in onnxtxt)
- 0점 dtype 수정(bf16/fp16은 onnxtxt에서 표현할 수 없음)

### Phase 5: Compiler Flag Determination
### 5단계: 컴파일러 플래그 결정

Invoke `/vai-fe-dtype --model <quantized_model.onnx>` to determine:
`/vai-fe-dtype --model <Quantized_model.onnx>`를 호출하여 다음을 확인합니다.
- Whether `enable_f32_to_bf16_conversion=true` is needed for BF16.
- BF16에 `enable_f32_to_bf16_conversion=true`가 필요한지 여부입니다.
- Whether `enable_f32_to_bf16_conversion=false` in combination with `enable_f32_to_f16_conversion=true` is needed (necessary for F16).
- `enable_f32_to_f16_conversion=true`와 함께 `enable_f32_to_bf16_conversion=false`가 필요한지 여부(F16에 필요)
- Whether `group_args: "mixed_precision"` or `match-power-of-two-quant-kernel=1` is needed (the choices are equivalent).
- `group_args: "mixed_precision"` 또는 `match-power-of-two-Quant-kernel=1`이 필요한지 여부(선택 사항은 동일함)
- Device-specific flags (ve2 for fp16)
- 장치별 플래그(fp16의 경우 ve2)
- Whether edge quantization in RT should be enabled based on `--edge-quant`
- `--edge-Quant`를 기반으로 RT의 에지 양자화를 활성화해야 하는지 여부
setting or auto-detection (`--edge-quant-auto`)
설정 또는 자동 감지(`--edge-Quant-auto`)

`fe_args` and `fe_experiment` are aliases, as are `group_args` and
`fe_args` 및 `fe_experiment`는 별칭이며 `group_args` 및
`experiments`; compiler logs may print either spelling.
'실험'; 컴파일러 로그는 두 철자를 모두 인쇄할 수 있습니다.
Write `fe_args` and
`fe_args`를 쓰고
`group_args` in generated configs.
생성된 구성의 `group_args`

If `--edge-quant` was specified:
`--edge-Quant`가 지정된 경우:
- `input` → append `input-quantization-in-rt=1` to `fe_args`
- `input` → `fe_args`에 `input-퀀트화-in-rt=1` 추가
- `output` → append `output-dequantization-in-rt=1` to `fe_args`
- `output` → `fe_args`에 `output-deQuantization-in-rt=1` 추가
- `both` → append `edge-quantization-in-rt=1` to `fe_args`
- `both` → `fe_args`에 `edge-Quantization-in-rt=1` 추가

If `--edge-quant-auto` was specified:
`--edge-Quant-auto`가 지정된 경우:
- Analyze the model for standalone Q at inputs and DQ at outputs
- 입력의 독립형 Q와 출력의 DQ에 대한 모델 분석
- If found, compile once without and once with `edge-quantization-in-rt=1`
- 발견되면 `edge-Quantization-in-rt=1`을 사용하지 않고 한 번 컴파일합니다.
- Compare kernel counts and recommend the better configuration
- 커널 수를 비교하고 더 나은 구성을 권장합니다.

Generate the appropriate `vitisai_config.json`.
적절한 `vitsai_config.json`을 생성합니다.

### Phase 6: Compilation and Execution
### 6단계: 컴파일 및 실행

This phase turns a quantized model into an NPU-runnable artifact.
이 단계에서는 양자화된 모델을 NPU 실행 가능 아티팩트로 전환합니다.
It is
그것은
**interactive**: after quantization succeeds, offer compilation as an explicit
**대화형**: 양자화가 성공한 후 컴파일을 명시적으로 제공합니다.
choice, and only after a successful compilation proceed with the board decision
선택하고 성공적인 편집이 이루어진 후에만 이사회 결정을 진행합니다.
already taken in [Board Execution: Ask First](#board-execution-ask-first-mandatory).
[보드 실행: 먼저 물어보기](#board-execution-ask-first-mandatory)에서 이미 수행되었습니다.
Never silently compile or run — and never skip the offer either.
자동으로 컴파일하거나 실행하지 말고 제안을 건너뛰지도 마십시오.
The compile /
컴파일 /
board scripts referenced here are deployed in the `ai_utils` bundle as siblings
여기에 참조된 보드 스크립트는 'ai_utils' 번들에 형제로 배포됩니다.
of this skill, in the scripts folder of the `vai-custom-op-implementation`
이 스킬의 `vai-custom-op-implementation` 스크립트 폴더에 있습니다.
skill.
기술.
Before running the snippets below, set
아래 스니펫을 실행하기 전에 다음을 설정하세요.
`VAI_CUSTOM_OP_IMPLEMENTATION_SCRIPTS` to that scripts folder, and
해당 스크립트 폴더에 `VAI_CUSTOM_OP_IMPLEMENTATION_SCRIPTS`를 추가하고
`VAI_FLAG_CONFIGURATION_SCRIPTS` to the script directory of the
`VAI_FLAG_CONFIGURATION_SCRIPTS`를 스크립트 디렉토리에
`vai-flag-configuration` skill.
'vai-flag-configuration' 스킬.

#### Step 6.0 — Activate the Vitis AI environment
#### 6.0단계 — Vitis AI 환경 활성화

Compilation and board execution use the **same** environment as quantization: a
컴파일 및 보드 실행은 양자화와 **동일한** 환경을 사용합니다.
Vitis AI installation, which provides the **VAIML compiler toolchain** — the
**VAIML 컴파일러 툴체인**을 제공하는 Vitis AI 설치 —
`flexml` package and, crucially, the `VitisAIExecutionProvider` in ONNXRuntime —
`flexml` 패키지와 ONNXRuntime의 `VitisAIExecutionProvider` —
next to Quark.
쿼크 옆에 있습니다.

Follow the [Environment Setup Convention](#environment-setup-convention): in the
[환경 설정 규칙](#environment-setup-convention)을 따르십시오:
Vitis AI Docker container the environment is already provisioned and nothing
Vitis AI Docker 컨테이너 환경은 이미 프로비저닝되어 있으며 아무것도 없습니다.
needs sourcing; otherwise source the activate script passed via `--t`.
소싱이 필요합니다. 그렇지 않으면 `--t`를 통해 전달된 활성화 스크립트를 소싱합니다.
Export
수출
the VAIML knobs in the same shell:
동일한 쉘의 VAIML 손잡이:

```bash
<ENV_SETUP>
export DEBUG_VAIML_PARTITION=2
export DEBUG_LOG_LEVEL=info
export BF16_SELECT=True
export ADD_TARGET_TO_CFG=True
```

**Verify the toolchain is usable** before compiling:
**컴파일하기 전에 툴체인을 사용할 수 있는지 확인**하세요.

```bash
python -c "import onnxruntime as ort; \
  assert 'VitisAIExecutionProvider' in ort.get_available_providers(); print('VitisAI EP OK')"
```

- If the VitisAI EP is unavailable after the setup above, **STOP and ask the
- 위 설정 후 VitisAI EP를 사용할 수 없는 경우 **중지하고 담당자에게 문의하세요.
user** for the correct Vitis AI environment (or the `--t` activate script) —
user** 올바른 Vitis AI 환경(또는 `--t` 활성화 스크립트) —
do not assume, hardcode, or search for one.
가정하거나, 하드코딩하거나, 검색하지 마십시오.

#### Step 6.1 — Prompt: compile the mixed-precision model with VAIML (a choice)
#### 6.1단계 - 프롬프트: VAIML을 사용하여 혼합 정밀도 모델 컴파일(선택 사항)

Once the quantized model exists (and its accuracy is validated), **ask the user
양자화된 모델이 존재하고 그 정확성이 검증되면 **사용자에게 문의하세요.
whether to compile it with the VAIML compiler**:
VAIML 컴파일러로 컴파일할지 여부**:

> "Quantization is complete.
> "양자화가 완료되었습니다.
> Do you want to compile the mixed-precision model
> 혼합 정밀도 모델을 컴파일하시겠습니까?
> with the VAIML compiler now?
> 지금 VAIML 컴파일러를 사용하고 있나요?
> [compile / skip]"
> [컴파일 / 건너뛰기]"

- If the user **skips**, report the quantized model path and stop here — do not
- 사용자가 **건너뛰기**하는 경우 양자화된 모델 경로를 보고하고 여기서 중지합니다.
compile or run.
컴파일하거나 실행합니다.
- If the user **compiles**:
- 사용자가 **컴파일**하는 경우:
  1. Determine the compiler flags first by invoking
  1. 다음을 호출하여 먼저 컴파일러 플래그를 결정합니다.
`/vai-fe-dtype --model <quantized_model.onnx> --device <stx|ve2> --output-config vitisai_config.json`
`/vai-fe-dtype --model <Quantized_model.onnx> --device <stx|ve2> --output-config vitisai_config.json`
(Phase 5).
(5단계).
Ensure the generated `vitisai_config.json` sets
생성된 `vitsai_config.json` 세트를 확인하세요.
`keep_outputs: true` so the partition/CpuBecause artifacts are written for
`keep_outputs: true`이므로 파티션/CpuBecause 아티팩트가 작성됩니다.
Step 6.3.
6.3단계.
  2. Compile in two steps.
  2. 두 단계로 컴파일합니다.
  2. First **patch the model for VAIP** so `com.amd.quark`
  2. 먼저 **VAIP용 모델을 패치**하여 `com.amd.quark`를 실행합니다.
EQDQ/BFP nodes carry the `shape`/`data_type` XIR attributes and bf16
EQDQ/BFP 노드는 `shape`/`data_type` XIR 속성 및 bf16을 전달합니다.
zero-points (a no-op for plain QDQ VINT8/BF16 models):
영점(일반 QDQ VINT8/BF16 모델의 경우 작동하지 않음):
     ```bash
     python3 scripts/patch_model_for_vaip.py \
         --input <quantized_model.onnx> --output <quantized_model_patched.onnx>
     ```
Then **compile the patched model** with the shared VAIML compile driver,
그런 다음 공유 VAIML 컴파일 드라이버를 사용하여 **패치된 모델**을 컴파일하고,
which registers any `com.amd.quark` custom ops found in the graph before
이전 그래프에서 발견된 `com.amd.quark` 사용자 정의 작업을 등록합니다.
creating the session:
세션 생성:
     ```bash
     python3 ../custom-op-implementation/scripts/compile.py \
         <quantized_model_patched.onnx> \
         --vitisai-config vitisai_config.json --cache-dir <model_stem>
     ```
For plain QDQ (VINT8/BF16) models the patch step is optional and you can
일반 QDQ(VINT8/BF16) 모델의 경우 패치 단계는 선택 사항이며 다음을 수행할 수 있습니다.
compile the quantized model directly with `compile.py`.
`compile.py`를 사용하여 양자화된 모델을 직접 컴파일합니다.
For a fast frontend/partition-only probe (no AIE binary — useful to check
빠른 프런트엔드/파티션 전용 프로브의 경우(AIE 바이너리 없음 - 확인하는 데 유용함)
partitioning before a full compile) use the `vai-flag-configuration`
전체 컴파일 전 파티셔닝) `vai-flag-configuration`을 사용하세요.
probe instead:
대신 조사해 보세요:
     ```bash
     python3 "$VAI_FLAG_CONFIGURATION_SCRIPTS/compile_probe.py" <quantized_model.onnx> \
         --vitisai-config vitisai_config.json --mode capability \
         --cache-dir <model_stem> -o compile_probe.json
     ```
  3. **Verify the compile actually succeeded.** ORT can print "Compilation
  3. **실제로 컴파일이 성공했는지 확인하세요.** ORT는 "컴파일"을 인쇄할 수 있습니다.
successful" while AIE errors force CPU fallback — check the log for errors
AIE 오류가 CPU 폴백을 강제하는 동안 "성공" — 로그에서 오류 확인
and confirm the cache dir is populated, as the compile scripts' own error
컴파일 스크립트 자체 오류로 캐시 디렉토리가 채워졌는지 확인하세요.
scan does.
스캔합니다.

#### Step 6.2 — Inspect the partitioning (recommended after every compile)
#### 6.2단계 - 파티셔닝 검사(매 컴파일 후 권장)

A successful compile does not mean the model reached a single NPU partition.
성공적인 컴파일이 모델이 단일 NPU 파티션에 도달했다는 의미는 아닙니다.
Invoke the `vai-partition-analysis` sub-skill on the compile output to report the
컴파일 출력에서 'vai-partition-analytic' 하위 기술을 호출하여 다음을 보고합니다.
operation statistics, the partition count, and the AIE/CPU split, and — when
작업 통계, 파티션 수, AIE/CPU 분할, 그리고 — 언제
more than one partition was produced — to propose remediation (frontend flags
둘 이상의 파티션이 생성되었습니다 - 수정을 제안하기 위해(프런트엔드 플래그
first, Quark-config changes when the break is quantization-caused):
먼저, 중단이 양자화로 인해 발생하면 Quark-config가 변경됩니다.

```
/vai-partition-analysis --compile-dir <model_stem> --model <quantized_model.onnx> \
    --config vitisai_config.json --device <stx|ve2>
```

The goal is **1 partition / 100% on AIE**.
목표는 **1개의 파티션/AIE에서 100%**입니다.
If a single partition with 100% offloading
100% 오프로드가 가능한 단일 파티션인 경우
is not possible, then the next best goals would combine the smallest possible number
불가능하다면 차선책은 가능한 가장 작은 숫자를 결합하는 것입니다.
of partitions with the highest possible offloading percentage.
오프로드 비율이 가장 높은 파티션입니다.
For example:
예를 들면:
- "1 partition and greater than 98% offloading"
- "파티션 1개 및 98% 이상의 오프로드"
- "less than 3 partitions and close to 100% offloading".
- "파티션이 3개 미만이고 거의 100% 오프로드됩니다."

If `vai-partition-analysis` proposes a
`vai-partition-analytic`이 제안하는 경우
change, apply it and re-compile (Step 6.1) to evaluate — `vai-partition-analysis`
변경하고 적용한 후 다시 컴파일(6.1단계)하여 평가 — `vai-partition-analytic`
always closes with a re-compilation.
항상 재컴파일로 닫힙니다.

#### Step 6.3 — Run the compiled model on the board
#### 6.3단계 - 보드에서 컴파일된 모델 실행

**Only after a successful compilation**, act on the board decision taken up
**성공적인 편집 후에만** 이사회 결정에 따라 조치를 취하세요.
front in [Board Execution: Ask First](#board-execution-ask-first-mandatory).
[보드 실행: 먼저 물어보세요](#board-execution-ask-first-mandatory)의 앞부분입니다.
If
that question was never asked, ask it now before going any further.
그 질문은 한 번도 물어본 적이 없습니다. 더 이상 진행하기 전에 지금 물어보세요.

- **Board execution declined** — report the compiled artifacts and stop.
- **보드 실행이 거부되었습니다** — 컴파일된 아티팩트를 보고하고 중지합니다.
- **Bundled board-run script** — Use the `run_on_board.py` the `scripts` folder of the
- **번들 보드 실행 스크립트** — `scripts` 폴더의 `run_on_board.py`를 사용합니다.
`vai-custom-op-implementation` skill.
`vai-custom-op-implementation` 스킬.
It reuses the Phase 6.1 compile cache (no recompile)
Phase 6.1 컴파일 캐시를 재사용합니다(재컴파일 없음).
and handles SSH, file sync, the `com.amd.quark` custom-op registration, and Telluride
SSH, 파일 동기화, `com.amd.quark` 사용자 정의 작업 등록 및 Telluride를 처리합니다.
board locking:
보드 잠금:
  ```bash
  # Remote Strix board via SSH
  python3 "$VAI_CUSTOM_OP_IMPLEMENTATION_SCRIPTS/run_on_board.py" \
      --boardhost <BOARDHOST> [--board-user <BOARD_USER>] \
      [--board-password <PASSWORD>] [--board-key <BOARD_SSH_KEY>] \
      -p . -- <quantized_model.onnx> \
      --input-dir <inputs_dir> --output-dir board_outputs
  ```
Note that user, password, SSH key are optional.
사용자, 비밀번호, SSH 키는 선택사항입니다.
- **User's own board setup** — run the exact command / harness the user gave
- **사용자 자신의 보드 설정** — 사용자가 제공한 정확한 명령/하네스를 실행합니다.
you, against the Phase 6.1 compile cache.
당신은 Phase 6.1 컴파일 캐시에 반대합니다.
Do not invent board commands, hosts,
보드 명령, 호스트,
or flags.
또는 깃발.

After a successful board run, compare the board outputs against the
성공적인 보드 실행 후 보드 출력을 비교하십시오.
full-precision CPU reference and report accuracy metrics and performance.
완전 정밀도 CPU 참조 및 보고 정확도 지표 및 성능.

#### Step 6.4 — Report
#### 6.4단계 - 보고

Report the quantized model path, the compiler flags used, the partition count
양자화된 모델 경로, 사용된 컴파일러 플래그, 파티션 수를 보고합니다.
and AIE/CPU split (from Step 6.2), and — if the board was run — the on-board
AIE/CPU 분할(6.2단계에서) 및 - 보드가 실행된 경우 - 온보드
accuracy and performance versus the FP32 reference.
FP32 레퍼런스에 비해 정확성과 성능이 뛰어납니다.


### Phase 7: Iteration (if accuracy/performance not met)
### 7단계: 반복(정확성/성능이 충족되지 않은 경우)

If results don't meet criteria:
결과가 기준을 충족하지 않는 경우:
1. **Accuracy too low**: Move more layers to higher precision
1. **정확도가 너무 낮음**: 더 높은 정밀도로 더 많은 레이어를 이동합니다.
   - Identify failing output elements
   - 실패한 출력 요소 식별
   - Use sensitivity analysis to find problematic layers
   - 민감도 분석을 사용하여 문제가 있는 레이어 찾기
   - Promote the most sensitive layers to BF16
   - 가장 민감한 레이어를 BF16으로 승격
2. **Performance too slow**: Move more layers to lower precision
2. **성능이 너무 느림**: 더 많은 레이어를 이동하여 정밀도를 낮추세요.
   - VINT8 should be faster than BF16 (flag if not)
   - VINT8은 BF16보다 빨라야 합니다(그렇지 않은 경우 플래그).
3. **Loop back** to Phase 2 with updated configuration
3. 업데이트된 구성을 사용하여 2단계로 **반복**

## Performance Expectations
## 성과 기대

| Configuration<br>구성 | Relative Speed<br>상대 속도 | Relative Accuracy<br>상대 정확도 |
|---------------|----------------|-------------------|
| Full BF16<br>전체 BF16 | 1.0x (baseline)<br>1.0x(기준선) | Highest<br>최고 |
| Full VINT8<br>풀 VINT8 | 2-4x faster<br>2~4배 더 빠름 | Lower (may fail tolerance)<br>낮음(허용 오차가 발생할 수 있음) |
| BF16+VINT8 mixed<br>BF16+VINT8 혼합 | 1.2-2x faster<br>1.2-2배 더 빠름 | Between BF16 and VINT8<br>BF16과 VINT8 사이 |

## Plugin Skills
## 플러그인 스킬

The orchestrator supports **optional plugin skills** located under
오케스트레이터는 아래에 있는 **선택적 플러그인 기술**을 지원합니다.
`skills/experimental-methods/`.
`기술/실험 방법/`.
Plugins extend the quantization workflow with
플러그인은 다음을 통해 양자화 작업 흐름을 확장합니다.
specialized capabilities (e.g. block floating-point formats, extended per-layer
특수 기능(예: 부동 소수점 형식 차단, 레이어별 확장
precision) that are not part of the core BF16/VINT8 pipeline.
정밀도)는 핵심 BF16/VINT8 파이프라인의 일부가 아닙니다.
The orchestrator
오케스트레이터
provides a generic facility for calling a plugin: it never hard-codes which
플러그인을 호출하기 위한 일반적인 기능을 제공합니다.
plugins exist, validates availability from the filesystem, and only when the
플러그인이 존재하고, 파일 시스템에서 가용성을 검증하며,
plugin is present does it execute the plugin and its own sub-commands/options.
플러그인이 존재하면 플러그인과 자체 하위 명령/옵션을 실행합니다.

### Invoking a Plugin
### 플러그인 호출

```
/vai-quantization-guide --model <path.onnx> --plugin <plugin-name> [plugin-specific args...]
```

or, equivalently, the shorthand:
또는 동등하게 다음과 같은 약어를 사용합니다.

```
/vai-plugin <plugin-name> --model <path.onnx> [plugin-specific args...]
```

### Plugin-Calling Facility (Discovery + Validation + Delegation)
### 플러그인 호출 기능(검색 + 검증 + 위임)

Follow this protocol whenever a plugin is requested (or when model analysis
플러그인이 요청될 때마다(또는 모델 분석 시) 이 프로토콜을 따르십시오.
indicates a specialized quantization the core pipeline does not handle):
핵심 파이프라인이 처리하지 않는 특수한 양자화를 나타냅니다):

1. **Resolve** the plugin path: `skills/experimental-methods/<plugin-name>/SKILL.md`.
1. **해결** 플러그인 경로: `skills/experimental-methods/<plugin-name>/SKILL.md`.
2. **Validate availability** by checking whether that file exists.
2. 해당 파일이 존재하는지 확인하여 **사용 가능 여부를 확인하세요**.
2. Do this
2. 이렇게 하세요
dynamically — do **not** maintain a hard-coded list of plugin names.
동적으로 — 하드코딩된 플러그인 이름 목록을 유지 **하지 마세요**.
3. **If unavailable**: inform the user the plugin is not installed, and list
3. **사용할 수 없는 경우**: 사용자에게 플러그인이 설치되지 않았음을 알리고 목록을 나열합니다.
what **is** available by enumerating the directories under
아래의 디렉토리를 열거하여 사용할 수 있는 **이 무엇**인지
`skills/experimental-methods/`.
`기술/실험 방법/`.
Stop — do not attempt the specialized method
중지 — 특수한 방법을 시도하지 마십시오
in the core pipeline.
코어 파이프라인에서.
4. **If available**: read the plugin's `SKILL.md` to discover its scope,
4. **사용 가능한 경우**: 플러그인의 `SKILL.md`를 읽어 범위를 알아보세요.
hardware constraints, experimental-feature consent banner, version checks,
하드웨어 제약, 실험적 기능 동의 배너, 버전 확인,
and its own sub-commands/options.
및 자체 하위 명령/옵션.
Display any consent banner and obtain
동의 배너를 표시하고 동의를 얻습니다.
user consent before proceeding.
계속하기 전에 사용자 동의를 얻어야 합니다.
5. **Delegate**: execute the plugin's protocol, passing through any
5. **위임**: 플러그인의 프로토콜을 실행하고
plugin-specific arguments to its sub-commands/options.
하위 명령/옵션에 대한 플러그인별 인수입니다.
The core orchestrator
핵심 오케스트레이터
does not interpret plugin-specific flags itself.
플러그인별 플래그 자체를 해석하지 않습니다.
6. **Resume**: when the plugin returns its quantized model, resume the core
6. **재개**: 플러그인이 양자화된 모델을 반환하면 코어를 재개합니다.
workflow at Phase 5 (Compiler Flag Determination) for compilation and
컴파일 및 작업을 위한 5단계(컴파일러 플래그 결정)의 워크플로
execution.
실행.
The plugin's `SKILL.md` supplies any additional compiler flags
플러그인의 `SKILL.md`는 추가 컴파일러 플래그를 제공합니다.
its format requires.
형식이 필요합니다.

### Available Plugins
### 사용 가능한 플러그인

Plugins ship under `experimental-methods/` and are deployed by the installer
플러그인은 'experimental-methods/' 아래에 제공되며 설치 프로그램에 의해 배포됩니다.
when present.
존재할 때.
They are not referenced by any core skill or agent definition —
핵심 기술이나 상담원 정의에서 참조되지 않습니다.
enumerate the `experimental-methods/` directory at runtime to see which are
런타임에 `experimental-methods/` 디렉토리를 열거하여 어떤 디렉토리가 있는지 확인하세요.
installed.
설치되었습니다.

**Red flags**:
**위험 신호**:
- If VINT8 is slower than BF16 → report compilation/mapping issue
- VINT8이 BF16보다 느린 경우 → 컴파일/매핑 문제 보고
- If mixed-precision is slower than full BF16 → too many precision transitions
- 혼합 정밀도가 전체 BF16보다 느린 경우 → 정밀도 전환이 너무 많음

## Dry-Run Mode
## 드라이런 모드

When `--mode dry-run`:
`--mode dry-run`인 경우:
- Run FP32 CPU reference
- FP32 CPU 참조 실행
- Run BF16 baseline (compile-only or exec)
- BF16 기준 실행(컴파일 전용 또는 실행)
- Run VINT8 baseline (compile-only or exec)
- VINT8 기준 실행(컴파일 전용 또는 실행)
- Report accuracy metrics WITHOUT modifying the model
- 모델 수정 없이 정확도 지표 보고
- Recommend a strategy based on results
- 결과에 따른 전략 추천

## Requantize Mode
## 재양자화 모드

When `--mode requantize`:
`--mode re양자화`인 경우:
- Strip existing quantization (`/vai-dequantize`)
- 기존 양자화 제거(`/vai-deQuantize`)
- Apply new quantization based on `--config`
- `--config`를 기반으로 새로운 양자화 적용
- Validate accuracy against baselines
- 기준선에 대한 정확성 검증
- Report improvement over previous quantization
- 이전 양자화에 대한 개선 보고

## Exploration Mode
## 탐험 모드

When `--explore`:
`--탐색`인 경우:
1. Start with full VINT8
1. 전체 VINT8로 시작하세요
2. Run sensitivity analysis on each layer
2. 각 계층에서 민감도 분석 실행
3. Move most sensitive layers to BF16, one at a time
3. 가장 민감한 레이어를 한 번에 하나씩 BF16으로 이동
4. After each change, measure accuracy delta
4. 각 변경 후 정확도 델타를 측정합니다.
5. Stop when accuracy threshold is met
5. 정확도 임계값이 충족되면 중지
6. Report the minimal set of BF16 layers needed
6. 필요한 최소 BF16 레이어 세트를 보고합니다.

This produces an optimal Quark JSON config file that can be reused.
그러면 재사용할 수 있는 최적의 Quark JSON 구성 파일이 생성됩니다.

### Pre-exploration Gate
### 탐험 전 게이트

**Before entering exploration, verify that exploration is warranted:**
**탐사에 들어가기 전에 탐색이 정당한지 확인하십시오.**

1. If task type is **classification** and top-1 matches FP32 → do NOT explore.
1. 작업 유형이 **분류**이고 상위 1이 FP32와 일치하는 경우 → 탐색하지 마세요.
Report success immediately with full VINT8.
전체 VINT8로 즉시 성공을 보고하세요.
2. If task type is **detection** and IoU/mAP pass → do NOT explore.
2. 작업 유형이 **감지**이고 IoU/mAP 통과인 경우 → 탐색하지 마세요.
3. Only enter exploration for regression models that fail element-wise
3. 요소별로 실패한 회귀 모델에 대해서만 탐색을 입력하세요.
thresholds, or classification/detection models where the functional metric
기능적 측정 기준이 있는 임계값 또는 분류/탐지 모델
(argmax, IoU) itself fails.
(argmax, IoU) 자체가 실패합니다.

Skipping this gate check caused the observed failure where a classification
이 게이트 확인을 건너뛰면 분류가 실패하는 것으로 관찰되었습니다.
model (ResNet50) was sent into exploration despite correct top-1.
올바른 top-1에도 불구하고 모델(ResNet50)이 탐색에 전송되었습니다.
Check it
확인해보세요
before any exploration logic runs.
탐색 논리가 실행되기 전에.

### Convergence Guardrails
### 컨버전스 가드레일

The exploration mode uses the following guardrails to ensure the search
탐색 모드에서는 다음과 같은 가드레일을 사용하여 검색을 보장합니다.
converges efficiently and avoids wasting iterations:
효율적으로 수렴하고 반복 낭비를 방지합니다.

| Guardrail<br>난간 | Rule<br>규칙 | Rationale<br>이론적 근거 |
|-----------|------|-----------|
| **Sensitivity ranking**<br>**민감도 순위** | Layers are sorted by dynamic range (max<br>레이어는 동적 범위(최대 |activation<br>활성화|) before exploration<br>) 탐사 전 | High dynamic range → high quantization error → try BF16 first<br>높은 동적 범위 → 높은 양자화 오류 → BF16을 먼저 시도 |
| **One-at-a-time promotion**<br>**한 번에 하나씩 프로모션** | Move exactly one layer from INT8 → BF16 per iteration<br>반복당 INT8 → BF16에서 정확히 한 레이어 이동 | Isolates the accuracy impact of each layer<br>각 레이어의 정확도 영향을 분리합니다. |
| **Monotonic accuracy**<br>**단조적 정확도** | If promoting a layer does not improve accuracy, skip it and try the next<br>레이어를 승격해도 정확도가 향상되지 않으면 건너뛰고 다음 단계를 시도해 보세요. | Avoids wasting BF16 budget on insensitive layers<br>민감하지 않은 레이어에서 BF16 예산 낭비 방지 |
| **Performance budget**<br>**성능 예산** | Stop if the number of BF16 layers exceeds 30% of total compute layers<br>BF16 레이어 수가 전체 컴퓨팅 레이어의 30%를 초과하면 중지됩니다. | Beyond this, the model is mostly BF16 and mixed-precision offers little speed benefit<br>이 외에도 모델은 대부분 BF16이며 혼합 정밀도는 속도 이점을 거의 제공하지 않습니다. |
| **PSNR plateau**<br>**PSNR 고원** | Stop when PSNR improves by less than 0.5 dB across 3 consecutive promotions<br>3회 연속 프로모션에서 PSNR이 0.5dB 미만으로 향상되면 중지 | Diminishing returns — further promotions won't meaningfully help<br>수익 감소 - 추가 프로모션은 의미 있는 도움이 되지 않습니다. |
| **Pass rate target**<br>**합격률 목표** | Stop when element-wise pass rate ≥ 99.9% at configured atol/rtol<br>구성된 atol/rtol에서 요소별 합격률 ≥ 99.9%일 때 중지 | Accuracy threshold met<br>정확도 임계값 충족 |
| **Max iterations**<br>**최대 반복 횟수** | Cap at min(50, total_compute_layers) iterations<br>최소(50, total_compute_layers) 반복으로 제한 | Prevents runaway exploration<br>폭주탐사 방지 |
| **Rollback on regression**<br>**회귀 시 롤백** | If promoting a layer degrades accuracy (pass rate drops), revert it<br>레이어 승격으로 인해 정확도가 저하되는 경우(통과율 저하) 되돌리세요. | Guards against interactions where a BF16 layer hurts accuracy<br>BF16 레이어가 정확성을 저하시키는 상호 작용으로부터 보호합니다. |

### Exploration Output
### 탐색 결과

The exploration produces:
탐색을 통해 다음이 생성됩니다.
1. **Sensitivity report**: Ranked list of layers with their sensitivity scores
1. **민감도 보고서**: 민감도 점수가 포함된 레이어 순위 목록
2. **Promotion log**: Each iteration's accuracy delta and decision
2. **승격 로그**: 각 반복의 정확도 델타 및 결정
3. **Optimal config**: A Quark JSON config with the minimal BF16 set
3. **최적 구성**: 최소 BF16 세트가 포함된 Quark JSON 구성
4. **Summary**: Number of BF16 layers, expected speedup vs. full BF16
4. **요약**: BF16 레이어 수, 전체 BF16 대비 예상되는 속도 향상

## Identifying Layers to Exclude from Quantization: Step-by-Step Guide
## 양자화에서 제외할 레이어 식별: 단계별 가이드

The key question in mixed-precision quantization is: **which layers should
혼합 정밀도 양자화의 주요 질문은 다음과 같습니다. **어떤 레이어를 사용해야 합니까?
remain in higher precision (BF16) to preserve accuracy?** This section
정확도를 유지하기 위해 더 높은 정밀도(BF16)를 유지하시겠습니까?** 이 섹션
provides a systematic approach with a worked example.
실제 사례를 통해 체계적인 접근 방식을 제공합니다.

### Step 1: Establish Baselines
### 1단계: 기준선 설정

Run the model in three configurations and record accuracy metrics:
세 가지 구성으로 모델을 실행하고 정확도 측정항목을 기록합니다.

```bash
# FP32 reference (golden output)
python compare_accuracy.py --reference model_fp32.onnx --model model_fp32.onnx

# Full BF16 baseline (accuracy ceiling for NPU)
python mixed_precision_quantize.py -i model_fp32.onnx -o model_bf16.onnx -c full_bf16.json
python compare_accuracy.py --reference model_fp32.onnx --model model_bf16.onnx

# Full VINT8 baseline (performance ceiling)
python mixed_precision_quantize.py -i model_fp32.onnx -o model_vint8.onnx -c full_vint8.json
python compare_accuracy.py --reference model_fp32.onnx --model model_vint8.onnx
```

**Decision**: If VINT8 meets your tolerance (e.g., PSNR > 30 dB, pass rate > 99.9%), stop here — use full VINT8.
**결정**: VINT8이 허용 오차(예: PSNR > 30dB, 통과율 > 99.9%)를 충족하는 경우 여기에서 중지하고 전체 VINT8을 사용하세요.

### Step 2: Run Per-Layer Sensitivity Analysis
### 2단계: 레이어별 민감도 분석 실행

Identify which layers contribute most to quantization error:
양자화 오류에 가장 큰 영향을 미치는 레이어를 식별합니다.

```bash
python mixed_precision_quantize.py -i model_fp32.onnx -o /dev/null \
    -c full_vint8.json --explore --explore-num-samples 50
```

This outputs a ranked list:
그러면 순위 목록이 출력됩니다.
```
Layer sensitivity (highest = most accuracy-critical):
  /model/head/Softmax:        42.1503
  /model/head/MatMul_1:       38.7291
  /model/neck/Add_3:          21.4820
  /model/backbone/Conv_47:    18.3102
  /model/backbone/Conv_46:    15.2710
  /model/backbone/Relu_44:     9.0031
  ...
```

Layers with high dynamic range (large activation magnitudes) lose the most
동적 범위가 높은(큰 활성화 크기) 레이어가 가장 많이 잃습니다.
information when quantized to INT8 and are prime candidates for BF16.
INT8로 양자화될 때 정보가 되며 BF16의 주요 후보입니다.

### Step 3: Iterative Layer Promotion
### 3단계: 반복적인 레이어 승격

Starting from full VINT8, promote the most sensitive layers to BF16 one at a
전체 VINT8부터 시작하여 가장 민감한 레이어를 하나씩 BF16으로 승격합니다.
time:
시간:

```bash
# Iteration 1: Promote the most sensitive layer
python mixed_precision_quantize.py -i model_fp32.onnx -o model_iter1.onnx \
    -c full_vint8.json --fp-nodes "/model/head/Softmax" --fp-precision bf16
python compare_accuracy.py --reference model_fp32.onnx --model model_iter1.onnx

# Iteration 2: Add the next most sensitive layer
python mixed_precision_quantize.py -i model_fp32.onnx -o model_iter2.onnx \
    -c full_vint8.json \
    --fp-nodes "/model/head/Softmax,/model/head/MatMul_1" --fp-precision bf16
python compare_accuracy.py --reference model_fp32.onnx --model model_iter2.onnx
```

Track the accuracy improvement at each iteration:
각 반복에서 정확도 개선을 추적합니다.
```
Iteration 0 (full VINT8):     PSNR=18.2 dB, pass_rate=72.1%  ← fails
Iteration 1 (+Softmax):       PSNR=24.7 dB, pass_rate=88.3%  ← improving
Iteration 2 (+MatMul_1):      PSNR=31.4 dB, pass_rate=97.5%  ← improving
Iteration 3 (+Add_3):         PSNR=34.8 dB, pass_rate=99.8%  ← close
Iteration 4 (+Conv_47):       PSNR=36.2 dB, pass_rate=99.95% ← PASS
```

Stop when the pass rate threshold is met (≥ 99.9%).
합격률 임계값(≥ 99.9%)이 충족되면 중지합니다.

### Step 4: Validate Patterns Across the Model
### 4단계: 모델 전체의 패턴 검증

Often, sensitivity follows structural patterns.
민감도는 구조적 패턴을 따르는 경우가 많습니다.
In the example above, the
위의 예에서는
head layers (Softmax, MatMul) are the most sensitive.
헤드 레이어(Softmax, MatMul)가 가장 민감합니다.
Check if a structural
구조적인지 확인해보세요
rule captures the pattern:
규칙은 패턴을 캡처합니다.

```json
{
  "specific_layer_config": [
    {
      "target_node_names": ["^/model/head/.*"],
      "layer_config": {
        "input_tensors": { "spec": "BFloat16Spec", "params": {} },
        "weight": { "spec": "BFloat16Spec", "params": {} },
        "bias": { "spec": "BFloat16Spec", "params": {} }
      }
    }
  ]
}
```

This is more maintainable than listing individual nodes and generalizes
이는 개별 노드를 나열하고 일반화하는 것보다 유지 관리가 더 쉽습니다.
better to model variants.
변형을 모델링하는 것이 더 좋습니다.

**IMPORTANT — JSON Format vs.
**중요 — JSON 형식과 JSON 형식 비교
Quark `QConfig.from_dict` Format:**
Quark `QConfig.from_dict` 형식:**

The JSON format shown above (list of `{"target_node_names": [...], "layer_config": {...}}`
위에 표시된 JSON 형식(`{"target_node_names": [...], "layer_config": {...}}` 목록)
objects) is our custom format, consumed by `mixed_precision_quantize.py`.
객체)는 `mixed_precision_Quantize.py`에서 사용되는 사용자 정의 형식입니다.
It is NOT the
그것은 아니다
format that Quark's `QConfig.from_dict` expects.
Quark의 `QConfig.from_dict`가 기대하는 형식입니다.
The two formats for `specific_layer_config`
`특정_레이어_구성`의 두 가지 형식
are:
다음과 같습니다:

- **Our JSON** (used in config files and throughout this guide):
- **저희 JSON**(구성 파일과 이 가이드 전체에서 사용됨):
  ```json
  "specific_layer_config": [
    {"target_node_names": ["^/head/.*"], "layer_config": {"input_tensors": {"spec": "BFloat16Spec", "params": {}}, ...}}
  ]
  ```

- **Quark `QConfig.from_dict`** (paired lists of `[layer_config_dict, node_names_list]`):
- **Quark`QConfig.from_dict`**(`[layer_config_dict, node_names_list]`의 쌍 목록):
  ```json
  "specific_layer_config": [
    [{"input_tensors": {"spec": "BFloat16Spec", "params": {}}, ...}, ["^/head/.*"]]
  ]
  ```

`mixed_precision_quantize.py` bridges the two: it reads our JSON, builds
`mixed_precision_Quantize.py`는 두 가지를 연결합니다. JSON을 읽고 빌드합니다.
`dict[QLayerConfig, list[str]]`, and passes that to the `QConfig()` constructor
`dict[QLayerConfig, list[str]]`, 그리고 이를 `QConfig()` 생성자에 전달합니다.
directly (not through `from_dict`).
(`from_dict`를 통하지 않고) 직접적으로.
If you need to call `QConfig.from_dict`
`QConfig.from_dict`를 호출해야 하는 경우
yourself, use the paired-list format.
자신의 경우에는 쌍 목록 형식을 사용하십시오.

`QLayerConfig` accepts keyword arguments `input_tensors`, `activation`, `weight`,
`QLayerConfig`는 키워드 인수 `input_tensors`, `activation`, `weight`를 허용합니다.
`bias`, and `output_tensors`, each a `QTensorConfig` (or `None`).
`bias` 및 `output_tensors`, 각각 `QTensorConfig`(또는 `None`)입니다.
Its `from_dict`
'from_dict'입니다
parses the same keys, each containing `{"spec": "<SpecName>", "params": {...}}`.
각각 `{"spec": "<SpecName>", "params": {...}}`를 포함하는 동일한 키를 구문 분석합니다.

**IMPORTANT — Pattern Syntax Rules for `target_node_names`:**
**중요 — `target_node_names`에 대한 패턴 구문 규칙:**

Quark's `get_all_target_nodes` interprets patterns as follows:
Quark의 `get_all_target_nodes`는 패턴을 다음과 같이 해석합니다.
- **Regex patterns**: MUST start with `^` AND contain `.*` (e.g., `"^/layer4/.*"`)
- **정규식 패턴**: `^`로 시작하고 `.*`를 포함해야 합니다(예: `"^/layer4/.*"`).
- **Exact node names**: Plain strings without `^` prefix (e.g., `"/layer4/layer4.1/conv2/Conv"`)
- **정확한 노드 이름**: `^` 접두사가 없는 일반 문자열(예: `"/layer4/layer4.1/conv2/Conv"`)
- Patterns starting with `^` but missing `.*` will trigger a Quark warning and may not match
- `^`로 시작하지만 `.*`가 누락된 패턴은 Quark 경고를 트리거하고 일치하지 않을 수 있습니다.

Examples:
예:
```json
"target_node_names": [
    "^/layer4/.*",
    "^/model/(head|neck)/.*",
    "/conv1/Conv"
]
```

**CPU Evaluation Limitation:**
**CPU 평가 제한:**

Mixed-precision models (containing BF16 Cast ops or `com.amd.quark` custom ops)
혼합 정밀도 모델(BF16 Cast 연산 또는 `com.amd.quark` 사용자 정의 연산 포함)
**cannot be evaluated on CPU** with ONNXRuntime's CPUExecutionProvider.
**ONNXRuntime의 CPUExecutionProvider를 사용하여 CPU에서 평가할 수 없습니다**.
The
는
BF16 Cast ops and `com.amd.quark` domain ops are only supported
BF16 Cast 작업 및 `com.amd.quark` 도메인 작업만 지원됩니다.
on NPU hardware.
NPU 하드웨어에서.
When running accuracy evaluation during exploration, use the
탐색 중 정확도 평가를 실행할 때 다음을 사용하십시오.
FP32 model for reference and compare against the full VINT8 model (which IS
참조용 FP32 모델과 전체 VINT8 모델(IS
CPU-evaluable).
CPU 평가 가능).
The mixed-precision model's accuracy must be validated on-device
혼합 정밀도 모델의 정확성은 기기에서 검증되어야 합니다.
after compilation.
컴파일 후.

### Step 5: Compile and Verify on CPU or NPU
### 5단계: CPU 또는 NPU에서 컴파일 및 확인

```bash
/vai-fe-dtype --model model_mixed.onnx --analyze
```

Use the generated `vaiml_config.json` to compile and execute the model on the
생성된 `vaiml_config.json`을 사용하여 모델을 컴파일하고 실행합니다.
NPU, e.g., using the VitisAI execution provider.
NPU(예: VitisAI 실행 공급자 사용)

### Worked Example: BEVFormer Detection Head
### 작업 예: BEVFormer 감지 헤드

A BEVFormer model with ~800 nodes shows poor accuracy in full VINT8:
~800개의 노드가 있는 BEVFormer 모델은 전체 VINT8에서 정확도가 낮습니다.
- Full BF16 baseline: PSNR=48.2 dB, pass_rate=100%
- 전체 BF16 기준: PSNR=48.2dB, pass_rate=100%
- Full VINT8: PSNR=12.4 dB, pass_rate=41.3%
- 전체 VINT8: PSNR=12.4dB, pass_rate=41.3%

**Sensitivity analysis** reveals:
**민감도 분석** 결과:
1. `pts_bbox_head` (detection head): Contains Softmax, MatMul, and
1. `pts_bbox_head`(탐지 헤드): Softmax, MatMul 및
ScatterND with very high dynamic ranges (>100).
매우 높은 동적 범위(>100)를 갖춘 ScatterND.
These are numerical
이는 숫자입니다.
operations sensitive to precision loss.
정밀도 손실에 민감한 작업.
2. `img_neck` (feature pyramid): Add operations that aggregate multi-scale
2. `img_neck`(기능 피라미드): 다중 규모를 집계하는 작업 추가
features are moderately sensitive (dynamic range 15–25).
기능은 적당히 민감합니다(동적 범위 15~25).
3. `img_backbone` (ResNet/Swin): Conv+Relu chains are relatively robust
3. `img_backbone`(ResNet/Swin): Conv+Relu 체인은 비교적 견고합니다.
to INT8 quantization (dynamic range < 10).
INT8 양자화(동적 범위 < 10).

**Resulting configuration**:
**결과 구성**:
- Backbone (Conv, Relu): **VINT8** — robust to quantization, fastest
- 백본(Conv, Relu): **VINT8** — 양자화에 강력하고 가장 빠릅니다.
- Neck (Add, Concat): **VINT8** — moderate sensitivity but tolerable
- Neck (Add, Concat): **VINT8** — 보통 감도이지만 허용 가능
- Detection head: **BF16** — high sensitivity, needs full precision
- 감지 헤드: **BF16** — 고감도, 완전한 정밀도 필요
- Excluded nodes: Gather, Unsqueeze, ScatterND at model boundaries — these
- 제외된 노드: 모델 경계에서 Gather, Unsqueeze, ScatterND — 이러한
are shape/indexing ops that don't benefit from quantization
양자화의 이점을 얻지 못하는 모양/인덱싱 작업입니다.

This corresponds to the `backbone_int8_all_layers_other_bf16.json` preset,
이는 `backbone_int8_all_layers_other_bf16.json` 사전 설정에 해당합니다.
which achieves PSNR=35.1 dB at ~2x the speed of full BF16.
이는 전체 BF16의 ~2배 속도에서 PSNR=35.1dB를 달성합니다.

## Listings of Quark configurations
## Quark 구성 목록

### full_vint8.json
### full_vint8.json

```json
{
    "global_config": {
        "input_tensors": { "spec": "XInt8Spec", "params": {} },
        "weight": { "spec": "XInt8Spec", "params": {} },
        "bias": { "spec": "XInt8Spec", "params": {} }
    },
    "extra_options": {
        "OpTypesToQuantize": [
            "Conv", "Mul"
        ],
        "OptimizeModel": false,
        "UseRandomData": false,
        "ConvertBNToConv": true,
        "ConvertSigmoidToHardSigmoid": false,
        "ConvertClipToRelu": true,
        "SplitLargeKernelPool": false,
        "ReplaceClip6Relu": true,
        "ConvertReduceMeanToGlobalAvgPool": false,
        "RemoveQDQConvClip": false,
        "RemoveQDQConvPRelu": false,
        "RemoveQDQConvRelu": false,
        "RemoveQDQConvLeakyRelu": false
    }
}
```

### full_bf16.json
### full_bf16.json

```json
{
    "global_config": {
        "input_tensors": { "spec": "BFloat16Spec", "params": {} },
        "weight": { "spec": "BFloat16Spec", "params": {} },
        "bias": { "spec": "BFloat16Spec", "params": {} }
    },
    "extra_options": {
        "UseRandomData": false,
        "IgnoreWarnings": false,
        "SimplifyModel": false,
        "OptimizeModel": false,
        "BF16QDQToCast": true
    }
}
```

### Sample of vint8+bf16 mixed config: backbone_int8_all_layers_other_bf16.json
### Vint8+bf16 혼합 구성 샘플: backbone_int8_all_layers_other_bf16.json

```json
{
  "global_config": {
    "input_tensors": { "spec": "XInt8Spec", "params": {} },
    "weight": { "spec": "XInt8Spec", "params": {} },
    "bias": { "spec": "XInt8Spec", "params": {} }
  },
  "specific_layer_config": [
    {
      "target_node_names": ["^(?!/model/module/(img_backbone|img_neck)/).*"],
      "layer_config": {
        "input_tensors": { "spec": "BFloat16Spec", "params": {} },
        "weight": { "spec": "BFloat16Spec", "params": {} },
        "bias": { "spec": "BFloat16Spec", "params": {} }
      }
    }
  ],
  "exclude": {
    "node_names": ["^(/Gather|/model/module/pts_bbox_head/(Unsqueeze_64|Concat_36|Reshape_2[0-4]|Tile_1|MatMul|Gather_33|Slice_(70|72|76|80)|Sub|Sub_[1-3]|Div_[2-5]|ScatterND_1[3-5]|Transpose_1|Clip|Clip_[12]|Log))$"],
    "subgraphs": []
  },
  "extra_options": {
    "OpTypesToQuantize": [],
    "ActivationSymmetric": true,
    "UseRandomData": false,
    "ConvertBNToConv": true,
    "ConvertSigmoidToHardSigmoid": false,
    "ConvertClipToRelu": false,
    "ConvertSplitToSlice": true,
    "SplitLargeKernelPool": false,
    "ReplaceClip6Relu": true,
    "ConvertReduceMeanToGlobalAvgPool": false,
    "RemoveQDQConvClip": false,
    "RemoveQDQConvPRelu": false,
    "RemoveQDQConvRelu": false,
    "RemoveQDQConvLeakyRelu": false,
    "Int32Bias": false,
    "IgnoreWarnings": false,
    "SimplifyModel": false,
    "OptimizeModel": false,
    "BF16QDQToCast": true,
    "EnableDualQuantNodePairs": true
  }
}
```

---

## Edge Quantization in Runtime
## 런타임의 에지 양자화

### Overview
### 개요

Edge quantization controls **where** the data type conversions at model
에지 양자화는 모델에서 데이터 유형이 변환되는 **어디**를 제어합니다.
boundaries are performed — on the host (by the runtime) or on the NPU (as
경계는 호스트(런타임 기준) 또는 NPU(예:
a kernel).
커널).
This is an orthogonal optimization to the precision-assignment
이는 정밀도 할당에 대한 직교 최적화입니다.
choices (vint8/bf16) above.
위의 선택사항(vint8/bf16).

### The `--edge-quant` Option
### `--edge-Quant` 옵션

```
/vai-quantization-guide --model model.onnx --device ve2 --edge-quant both
```

| Value<br>가치 | FE Flag Generated<br>FE 플래그 생성됨 | Effect<br>효과 |
|-------|-------------------|--------|
| `none`<br>'없음' | (no flag)<br>(플래그 없음) | Default. Boundary Q/DQ handled by NPU kernels<br>기본값. NPU 커널에 의해 처리되는 경계 Q/DQ |
| `input`<br>'입력' | `input-quantization-in-rt=1`<br>`입력-양자화-in-rt=1` | Input Q handled by runtime<br>런타임에 의해 처리되는 입력 Q |
| `output`<br>'출력' | `output-dequantization-in-rt=1`<br>`출력-역양자화-in-rt=1` | Output DQ handled by runtime<br>런타임에 의해 처리되는 출력 DQ |
| `both`<br>'둘 다' | `edge-quantization-in-rt=1`<br>`가장자리 양자화-in-rt=1` | Both handled by runtime<br>둘 다 런타임에 의해 처리됨 |

### When to Use `--edge-quant`
### `--edge-Quant`를 사용해야 하는 경우

Use edge quantization when your model has:
모델에 다음이 있는 경우 에지 양자화를 사용하십시오.

1. **Float inputs → int8 core**: The model receives fp32 data but the
1. **Float 입력 → int8 core**: 모델은 fp32 데이터를 수신하지만
backbone runs in int8.
백본은 int8에서 실행됩니다.
Instead of dedicating an NPU QuantizeLinear
NPU QuantizeLinear 전용 대신
kernel to the input conversion, offload it to the runtime.
커널을 입력 변환으로 변환하고 이를 런타임으로 오프로드합니다.

2. **Int8 core → float outputs**: The model produces int8 results that
2. **Int8 코어 → 부동 소수점 출력**: 모델은 다음과 같은 int8 결과를 생성합니다.
must be dequantized to fp32 for the application.
애플리케이션을 위해 fp32로 역양자화되어야 합니다.
Offload the
오프로드
DequantizeLinear to the runtime.
DeQuantizeLinear를 런타임에 적용합니다.

3. **Mixed-precision models where the input/output precision differs
3. **입력/출력 정밀도가 다른 혼합 정밀도 모델
from the compute precision**: Common in YOLO models with int8 backbone
계산 정밀도에서**: int8 백본을 사용하는 YOLO 모델에서 일반적입니다.
but float detection heads.
하지만 플로트 감지 헤드.

**Do NOT use** when:
**사용하지 마세요**:
- Model inputs and outputs are already int8 (no conversion needed)
- 모델 입력 및 출력은 이미 int8입니다(변환 필요 없음).
- Full BF16 models (no quantization at all)
- 전체 BF16 모델(양자화 전혀 없음)
- The Q/DQ at the boundary is part of a fake QDQ pair (these are handled
- 경계의 Q/DQ는 가짜 QDQ 쌍의 일부입니다.
differently by the QDQ detection algorithm)
QDQ 감지 알고리즘에 따라 다름)

### The `--edge-quant-auto` Option (Stretch Goal: Auto Mode)
### `--edge-Quant-auto` 옵션(스트레치 목표: 자동 모드)

When `--edge-quant-auto` is specified, the skill automatically evaluates
`--edge-Quant-auto`가 지정되면 스킬이 자동으로 평가합니다.
whether edge quantization is beneficial:
에지 양자화가 유익한지 여부:

```
/vai-quantization-guide --model model.onnx --device ve2 --edge-quant-auto
```

**Algorithm**:
**알고리즘**:

1. **Detect boundary Q/DQ ops**: Scan the model for QuantizeLinear ops
1. **경계 Q/DQ 작업 감지**: QuantizeLinear 작업에 대한 모델 스캔
that consume model inputs and DequantizeLinear ops that produce model
모델 입력을 소비하는 모델과 모델을 생성하는 DeQuantizeLinear 작업
outputs.
출력.
If none exist, report "edge quantization not applicable."
존재하지 않는 경우 "에지 양자화 적용 불가"를 보고합니다.

2. **Compile without edge quant**: Run the compiler without edge-quant
2. **에지 퀀트 없이 컴파일**: 에지 퀀트 없이 컴파일러를 실행합니다.
flags and record:
플래그 및 기록:
   - NPU kernel count
   - NPU 커널 수
   - Number of `QuantizeLinear` / `DequantizeLinear` kernels at edges
   - 가장자리의 `QuantizeLinear` / `DeQuantizeLinear` 커널 수
   - Scale factors used at boundaries
   - 경계에 사용되는 축척 계수

3. **Compile with edge quant**: Run the compiler with
3. **에지 퀀트로 컴파일**: 다음을 사용하여 컴파일러를 실행합니다.
`edge-quantization-in-rt=1` and record:
`edge-Quantization-in-rt=1` 및 기록:
   - NPU kernel count (should be lower by the number of edge Q/DQ ops)
   - NPU 커널 수(에지 Q/DQ 작업 수만큼 낮아야 함)
   - Whether any new CpuBecause messages appeared
   - 새로운 CpuBecause 메시지가 나타났는지 여부
   - Scale factors in `flexmlrt-hsi.json` `rt_transformations`
   - `flexmlrt-hsi.json` `rt_transformations`의 배율 인수

4. **Compare and recommend**:
4. **비교 및 추천**:
   - If kernel count decreased AND no new CpuBecause → **recommend enabling**
   - 커널 수가 감소하고 새 CpuBecause가 없는 경우 → **활성화하는 것이 좋습니다**
   - If CpuBecause appeared → **recommend disabling** (edge quant broke
   - CpuBecause가 나타나는 경우 → **비활성화 권장**(에지 퀀트 깨짐)
something downstream)
뭔가 다운스트림)
   - If kernel count unchanged → edge Q/DQ were already handled differently
   - 커널 개수가 변경되지 않은 경우 → 에지 Q/DQ가 이미 다르게 처리된 경우

5. **Report tradeoffs**:
5. **장단점 보고**:
   ```
   ┌──────────────────────┬────────────┬────────────────┐
   │ Configuration        │ NPU Kernels│ Edge Q/DQ in RT│
   ├──────────────────────┼────────────┼────────────────┤
   │ Without edge quant   │    247     │      0         │
   │ With edge quant      │    243     │      4         │
   └──────────────────────┴────────────┴────────────────┘
   → Recommendation: Enable edge-quantization-in-rt=1
     Saves 4 NPU kernels by offloading boundary Q/DQ to runtime.
     Scale factors propagated to flexmlrt-hsi.json.
   ```

6. **Performance estimation** (when board is available):
6. **성능 추정**(보드를 사용할 수 있는 경우):
   - If `--stage exec` is also specified, run both configurations on
   - `--stage exec`도 지정된 경우 두 구성을 모두 실행합니다.
hardware and report the latency difference
하드웨어 및 대기 시간 차이 보고
   - Typical expectation: edge quant reduces latency by 1-5% depending
   - 일반적인 기대: 에지 퀀트는 상황에 따라 대기 시간을 1~5% 줄입니다.
on model size (bigger models see smaller relative benefit)
모델 크기에 따라(더 큰 모델은 상대적인 이점이 더 작음)

### How It Works Internally (Domain Knowledge)
### 내부 작동 방식(도메인 지식)

The FPConversionPass in the compiler uses edge quantization flags to decide
컴파일러의 FPConversionPass는 에지 양자화 플래그를 사용하여 결정합니다.
which Q/DQ operations at function boundaries should be preserved (for
함수 경계에서 어떤 Q/DQ 연산을 유지해야 하는지(예:
runtime handling) vs. converted (into NPU kernels):
런타임 처리) 및 변환(NPU 커널로):

```
With input-quantization-in-rt=1:
  ┌─────┐    ┌───┐    ┌─────────┐
  │ arg │ -> │ Q │ -> │ NPU ops │ -> ...
  └─────┘    └───┘    └─────────┘
                ↑
     This Q is PRESERVED in the compile graph.
     Runtime performs: float_input * (1/scale) → int8
     Scale factor recorded in flexmlrt-hsi.json.

Without input-quantization-in-rt=1:
  ┌─────┐    ┌───────────────────┐    ┌─────────┐
  │ arg │ -> │ NPU QuantizeLinear│ -> │ NPU ops │ -> ...
  └─────┘    └───────────────────┘    └─────────┘
                       ↑
     This Q becomes an NPU kernel (QuantizeLinear3D or similar).
```

The MoveCastDown pass then moves any residual cast operations (from
그런 다음 MoveCastDown 패스는 잔여 캐스트 작업을 이동합니다(
`fp-conversion-keep-function-signature=1`) closer to the function boundary,
`fp-conversion-keep-function-signature=1`) 함수 경계에 더 가깝습니다.
so they combine with the edge quantization annotation rather than creating
그래서 그들은 생성하는 대신 가장자리 양자화 주석과 결합합니다.
separate cast kernels mid-graph.
별도의 캐스트 커널 중간 그래프.

### Example: YOLO Model with Edge Quantization
### 예: Edge Quantization을 사용한 YOLO 모델

A YOLO model with float inputs, int8 backbone, and mixed detection head:
부동 소수점 입력, int8 백본 및 혼합 감지 헤드가 있는 YOLO 모델:

```bash
# Without edge quant: 3 partitions due to input/output Q/DQ kernels
/vai-quantization-guide --model yolov8m_int8.onnx --device ve2 \
  --edge-quant none

# With edge quant: single partition, boundary Q/DQ offloaded to RT
/vai-quantization-guide --model yolov8m_int8.onnx --device ve2 \
  --edge-quant both

# Auto-detect: compiles both ways and recommends
/vai-quantization-guide --model yolov8m_int8.onnx --device ve2 \
  --edge-quant-auto
```

### Integration with Other Flags
### 다른 플래그와의 통합

Edge quantization flags compose with other FE experiment flags:
에지 양자화 플래그는 다른 FE 실험 플래그와 구성됩니다.

```json
{
  "enable_f32_to_bf16_conversion": true,
  "fe_args": "match-power-of-two-quant-kernel=1 edge-quantization-in-rt=1"
}
```

The full flag precedence for a mixed-precision model with edge quant:
에지 퀀트를 사용하는 혼합 정밀도 모델의 전체 플래그 우선 순위:
1. `enable_f32_to_bf16_conversion=true` — force-converts naked fp ops to bf16
1. `enable_f32_to_bf16_conversion=true` — 기본 fp 작업을 bf16으로 강제 변환합니다.
2. `match-power-of-two-quant-kernel=1` — matches Q/DQ as kernels at island boundaries
2. `match-power-of-two-Quant-kernel=1` — 섬 경계에서 Q/DQ를 커널로 일치시킵니다.
3. `edge-quantization-in-rt=1` — offloads boundary Q/DQ to runtime
3. `edge-Quantization-in-rt=1` — 경계 Q/DQ를 런타임으로 오프로드합니다.

These three flags together represent the maximum mixed-precision configuration
이 세 가지 플래그는 함께 최대 혼합 정밀도 구성을 나타냅니다.
with runtime-handled edges.
런타임 처리 가장자리가 있습니다.
The edge quant flag does NOT conflict with any
Edge Quant 플래그는 어떤 플래그와도 충돌하지 않습니다.
other flag; it changes where certain Q/DQ operations are executed.
다른 플래그; 특정 Q/DQ 작업이 실행되는 위치가 변경됩니다.
