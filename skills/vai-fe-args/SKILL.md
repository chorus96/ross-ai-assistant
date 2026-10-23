<!-- ko-translation: enabled -->
---
name: vai-fe-args
license: Apache-2.0 WITH LLVM-exception
compatibility: Requires VitisAI 6.3+
metadata:
  version: 6.3.0
  stage: beta
description: Determine the correct fe_args and vaiml_config flags for a mixed-precision
  ONNX model based on its quantization pattern. Analyzes model structure to identify
  island topology, quantization types, and required compiler features. Targets single
  NPU partition (embedded Telluride scenario).
---
<!--- Copyright (C) 2026 Advanced Micro Devices, Inc. All rights reserved.--->

# FE Args Skill
# FE Args 스킬

## Access command
## 액세스 명령

This skill must always be accessed through the worker, never in a different way.
이 기술은 항상 작업자를 통해 접근해야 하며 다른 방식으로는 접근할 수 없습니다.
Allowed workers: `vai-mixed-precision-worker`, or the `vai-flag-configuration`
허용되는 작업자: `vai-mixed-precision-worker` 또는 `vai-flag-configuration`
skill, which is its own entry point.
스킬은 그 자체의 진입점입니다.

## Description
## 설명

Determines the correct `fe_args` frontend flags and `vaiml_config` settings for a
올바른 'fe_args' 프런트엔드 플래그와 'vaiml_config' 설정을 결정합니다.
mixed-precision ONNX model.
혼합 정밀도 ONNX 모델.
Analyzes the model's quantization topology to select
모델의 양자화 토폴로지를 분석하여 선택
appropriate compiler flags for single-NPU-partition compilation.
단일 NPU 파티션 컴파일에 적합한 컴파일러 플래그.

`fe_args` and `fe_experiment` are aliases, as are `group_args` and
`fe_args` 및 `fe_experiment`는 별칭이며 `group_args` 및
`experiments`; compiler logs may use either spelling.
'실험'; 컴파일러 로그에서는 두 철자를 모두 사용할 수 있습니다.
Prefer `fe_args` and `group_args` in new configs.
새 구성에서는 `fe_args` 및 `group_args`를 선호합니다.

## Parse Arguments
## 인수 구문 분석

This skill expects arguments in this form:
이 기술은 다음 형식의 인수를 기대합니다.

```
--model <onnx_model> [--analyze] [--device stx|ve2] [--output-config <vitisai_config.json>]
```

- `--model <path>`: **(MANDATORY)** Path to the quantized ONNX model
- `--model <path>`: **(필수)** 양자화된 ONNX 모델의 경로
- `--analyze`: **(OPTIONAL)** Print detailed analysis without generating config
- `--analyze`: **(선택 사항)** 구성을 생성하지 않고 자세한 분석을 인쇄합니다.
- `--device <stx|ve2>`: **(OPTIONAL, default: ve2)** Target device
- `--device <stx|ve2>`: **(선택 사항, 기본값: ve2)** 대상 장치
- `--output-config <path>`: **(OPTIONAL)** Path to write generated vitisai_config.json
- `--output-config <경로>`: **(선택 사항)** 생성된 vitisai_config.json을 쓸 경로

## Model Classification
## 모델 분류

Analyze the model and classify into one of these categories:
모델을 분석하고 다음 범주 중 하나로 분류합니다.

### Category 1: Pure VINT8
### 카테고리 1: 순수 VINT8

**Pattern**: Entire graph is wrapped in QuantizeLinear/DequantizeLinear (QDQ)
**패턴**: 전체 그래프가 QuantizeLinear/DeQuantizeLinear(QDQ)로 래핑됩니다.
```
input(int8) → DQ → op → Q → DQ → op → Q → output(int8)
```

**Required flags**:
**필수 플래그**:
```json
{
  "enable_f32_to_bf16_conversion": false,
  "fe_args": ""
}
```

No special flags needed.
특별한 플래그가 필요하지 않습니다.
The compiler handles pure QDQ models natively.
컴파일러는 순수 QDQ 모델을 기본적으로 처리합니다.

### Category 2: Two-Island Models (BF16/FP16 + VINT8)
### 카테고리 2: 2섬 모델(BF16/FP16 + VINT8)

**Pattern**: One floating-point region and one quantized region.
**패턴**: 부동 소수점 영역 1개와 양자화 영역 1개.

Sub-patterns:
하위 패턴:
- **Float head → VINT8 core**: `input(fp) → [fp ops] → Q → DQ → [int8 ops] → output`
- **플로트 헤드 → VINT8 코어**: `input(fp) → [fp ops] → Q → DQ → [int8 ops] → 출력`
- **VINT8 core → Float tail**: `input(int8) → [int8 ops] → Q → DQ → [fp ops] → output`
- **VINT8 코어 → 플로트 테일**: `input(int8) → [int8 ops] → Q → DQ → [fp ops] → 출력`
- **Float head + tail, VINT8 middle**: `input(fp) → [fp] → Q/DQ → [int8] → Q/DQ → [fp] → output`
- **플로트 헤드 + 테일, VINT8 중간**: `input(fp) → [fp] → Q/DQ → [int8] → Q/DQ → [fp] → 출력`
- **VINT8 head + tail, Float middle**: `input(int8) → [int8] → Q/DQ → [fp] → Q/DQ → [int8] → output`
- **VINT8 헤드 + 테일, 플로트 중간**: `input(int8) → [int8] → Q/DQ → [fp] → Q/DQ → [int8] → 출력`

**Required flags**:
**필수 플래그**:
```json
{
  "enable_f32_to_bf16_conversion": true,
  "fe_args": "match-power-of-two-quant-kernel=1"
}
```

For **ve2** (FP16 instead of BF16):
**ve2**(BF16 대신 FP16):
```json
{
  "enable_f32_to_f16_conversion": true,
  "fe_args": "match-power-of-two-quant-kernel=1",
  "device": "ve2"
}
```

**Explanation**: `match-power-of-two-quant-kernel=1` enables the MLLIB
**설명**: 'match-power-of-two-Quant-kernel=1'을 사용하면 MLLIB가 활성화됩니다.
QuantizeLinear kernel matcher for BF16/FP16 → Int8 transitions at island
BF16/FP16용 QuantizeLinear 커널 일치자 → 아일랜드에서 Int8 전환
boundaries.
경계.
Without this, the quantize operation between floating-point and
이것이 없으면 부동소수점과 부동소수점 사이의 양자화 연산은
int8 regions won't be recognized as an NPU kernel.
int8 영역은 NPU 커널로 인식되지 않습니다.

### Category 3: Multi-Island Models (4+ alternating regions)
### 카테고리 3: 다중섬 모델(4개 이상의 교대 지역)

**Pattern**: Multiple alternating quantized and floating-point regions.
**패턴**: 여러 개의 교대 양자화 및 부동 소수점 영역.
```
[int8] → [fp] → [int8] → [fp] → [int8] → ...
```

**Required flags**:
**필수 플래그**:
```json
{
  "enable_f32_to_bf16_conversion": true,
  "fe_args": "match-power-of-two-quant-kernel=1"
}
```

**Explanation**:
**설명**:
- `match-power-of-two-quant-kernel=1` — same as Category 2
- `match-power-of-two-Quant-kernel=1` — 카테고리 2와 동일

**The shorthand** `group_args: "mixed_precision"` is equivalent to both flags:
**약칭** `group_args: "mixed_precision"`은 두 플래그 모두와 동일합니다.
```json
{
  "enable_f32_to_bf16_conversion": true,
  "group_args": "mixed_precision"
}
```
This expands internally to: `match-power-of-two-quant-kernel=1`
이는 내부적으로 `match-power-of-two-Quant-kernel=1`로 확장됩니다.

### Category 4: Pure BF16/FP16 (Non-Quantized, Force-Converted)
### 카테고리 4: 순수 BF16/FP16(비양자화, 강제 변환)

**Pattern**: Entire model is floating-point, to be force-converted to bf16/fp16.
**패턴**: 전체 모델은 부동 소수점이며 bf16/fp16으로 강제 변환됩니다.

**Required flags**:
**필수 플래그**:
```json
{
  "enable_f32_to_bf16_conversion": true,
  "fe_args": ""
}
```

For FP16 (ve2 only):
FP16의 경우(ve2만 해당):
```json
{
  "enable_f32_to_f16_conversion": true,
  "device": "ve2"
}
```

## Analysis Algorithm
## 분석 알고리즘

```python
def classify_model(model):
    """Classify model quantization topology."""
    has_qdq = any(n.op_type in ("QuantizeLinear", "DequantizeLinear")
                  for n in model.graph.node)
    # com.amd.quark quantization ops indicate a specialized method (e.g.
    # block floating-point or extended per-layer precision) handled by an
    # optional plugin skill, not by the core pipeline.
    has_specialized = any(n.domain == "com.amd.quark" and "Quantize" in n.op_type
                          for n in model.graph.node)
    has_naked_fp = has_floating_point_ops_without_qdq(model)

    # Count islands (contiguous regions of same precision)
    islands = count_precision_islands(model)

    if has_specialized:
        # Delegate flag determination to the matching plugin skill under
        # skills/experimental-methods/. If none is installed, report that the
        # model uses a specialized quantization the core pipeline cannot flag.
        return "SPECIALIZED"
    if has_qdq:
        if not has_naked_fp:
            return "PURE_VINT8"
        if islands <= 3:
            return "TWO_ISLAND"
        return "MULTI_ISLAND"
    return "PURE_FLOAT"
```

## Generated vitisai_config.json Template
## 생성된 vitisai_config.json 템플릿

```json
{
  "passes": [
    {
      "name": "vaiml_partition",
      "plugin": "vaip-pass_vaiml_partition",
      "vaiml_config": {
        "keep_outputs": true,
        "optimize_level": 2,
        "enable_f32_to_bf16_conversion": <DETERMINED>,
        "logging_level": "info",
        "fe_args": "<DETERMINED>",
        "device": "<DEVICE>"
      }
    }
  ],
  "target": "VAIML",
  "targets": [
    { "name": "VAIML", "pass": ["vaiml_partition"] }
  ]
}
```

## Single NPU Partition (Embedded Scenario)
## 단일 NPU 파티션(내장형 시나리오)

For Telluride architecture targeting a single NPU partition:
단일 NPU 파티션을 대상으로 하는 Telluride 아키텍처의 경우:
- All operations must be mapped to NPU (no CPU fallback)
- 모든 작업은 NPU에 매핑되어야 합니다(CPU 대체 없음).
- The entire model must be quantized in some way (no fp32 operations)
- 전체 모델은 어떤 방식으로든 양자화되어야 합니다(fp32 작업 없음).
- This means every operation is either:
- 이는 모든 작업이 다음 중 하나임을 의미합니다.
  - Standard QDQ (vint8)
  - 표준 QDQ(vint8)
  - Force-converted from fp32 to bf16/fp16
  - fp32에서 bf16/fp16으로 강제 변환됨
  - Quantized by a specialized method handled by an optional plugin skill
  - 선택적 플러그인 기술로 처리되는 특수 방법으로 양자화됨
(e.g. block floating-point or extended per-layer precision)
(예: 블록 부동 소수점 또는 확장된 레이어별 정밀도)

If `CpuBecause` messages appear in compilation logs, it indicates operations
컴파일 로그에 'CpuBecause' 메시지가 나타나면 작업이 수행되었음을 나타냅니다.
that could not be mapped to NPU.
NPU에 매핑할 수 없습니다.
Common causes:
일반적인 원인:
- Missing force-conversion flag for naked fp32 ops
- 기본 fp32 작업에 대한 강제 변환 플래그가 누락되었습니다.
- YAML constraint failures (dtype mismatch in kernels)
- YAML 제약 조건 실패(커널의 dtype 불일치)
- Unsupported operations
- 지원되지 않는 작업

## Flag Interaction Matrix
## 플래그 상호작용 매트릭스

| Model Type<br>모델 유형 | f32_to_bf16<br>f32_to_bf16 | f32_to_f16<br>f32_to_f16 | match-pow2-quant<br>매치파우2퀀트 |
|------------|:-----------:|:----------:|:----------------:|
| Pure VINT8<br>퓨어 VINT8 | - | - | - |
| 2-island (bf16)<br>2섬(bf16) | ✓ | - | ✓ |
| 2-island (fp16)<br>2섬(fp16) | - | ✓ | ✓ |
| Multi-island (bf16)<br>다중섬(bf16) | ✓ | - | ✓ |
| Multi-island (fp16)<br>다중 섬(fp16) | - | ✓ | ✓ |
| Pure BF16<br>순수 BF16 | ✓ | - | - |
| Pure FP16<br>순수 FP16 | - | ✓ | - |

> Models quantized by a specialized method (block floating-point, extended
> 특수한 방법으로 양자화된 모델(블록 부동 소수점, 확장
> per-layer precision, etc.) are flagged by the corresponding **plugin skill**
> 레이어별 정밀도 등)은 해당 **플러그인 스킬**에 의해 표시됩니다.
> under `skills/experimental-methods/`, which supplies its own flag rows.
> 자체 플래그 행을 제공하는 `skills/experimental-methods/` 아래에 있습니다.
> The
> 는
> core skill does not determine flags for those formats.
> 핵심 기술은 해당 형식에 대한 플래그를 결정하지 않습니다.

---

## Edge Quantization in Runtime (RT)
## 런타임(RT)의 에지 양자화

Edge quantization allows the runtime (FLEXMLRT/VART) to handle data type
에지 양자화를 통해 런타임(FLEXMLRT/VART)이 데이터 유형을 처리할 수 있습니다.
conversions at model boundaries — inputs and outputs — rather than having the
모델 경계(입력 및 출력)에서 변환하는 대신
NPU kernel graph perform them.
NPU 커널 그래프가 이를 수행합니다.
This is relevant for mixed-precision models
이는 혼합 정밀도 모델과 관련이 있습니다.
where the model's input/output types differ from the NPU compute types.
여기서 모델의 입력/출력 유형은 NPU 컴퓨팅 유형과 다릅니다.

### What Edge Quantization Does
### 에지 양자화가 하는 일

When enabled, edge quantization **preserves** standalone Quantize (at input)
활성화되면 에지 양자화는 독립형 Quantize(입력 시)를 **보존**합니다.
or Dequantize (at output) operations in the compiled graph metadata
또는 컴파일된 그래프 메타데이터의 역양자화(출력 시) 작업
(`flexmlrt-hsi.json`) rather than converting them into NPU kernels.
(`flexmlrt-hsi.json`) NPU 커널로 변환하는 대신.
The
는
runtime then handles these conversions as host-side data transformations.
그런 다음 런타임은 이러한 변환을 호스트 측 데이터 변환으로 처리합니다.

Concretely:
구체적으로:
- **Input edge quantization**: A standalone `QuantizeLinear` at the model input
- **입력 가장자리 양자화**: 모델 입력의 독립형 'QuantizeLinear'
(arg → Q → NPU ops) is annotated for runtime processing.
(arg → Q → NPU ops)는 런타임 처리를 위해 주석이 추가됩니다.
The runtime
런타임
performs the float→int8 conversion on the host before feeding data to NPU.
NPU에 데이터를 공급하기 전에 호스트에서 float→int8 변환을 수행합니다.
- **Output edge dequantization**: A standalone `DequantizeLinear` at the model
- **출력 에지 역양자화**: 모델의 독립형 'DeQuantizeLinear'
output (NPU ops → DQ → return) is annotated for runtime processing.
출력(NPU ops → DQ → return)에는 런타임 처리를 위한 주석이 추가됩니다.
The
는
runtime performs the int8→float conversion on the host after NPU computation.
런타임은 NPU 계산 후 호스트에서 int8→float 변환을 수행합니다.

### FE Flags for Edge Quantization
### 에지 양자화를 위한 FE 플래그

| Flag<br>깃발 | Description<br>설명 | Default<br>기본값 |
|------|-------------|---------|
| `edge-quantization-in-rt=1`<br>`가장자리 양자화-in-rt=1` | Enable both input quantization and output dequantization in RT<br>RT에서 입력 양자화와 출력 역양자화를 모두 활성화합니다. | `false`<br>'거짓' |
| `input-quantization-in-rt=1`<br>`입력-양자화-in-rt=1` | Enable only input-side quantization in RT<br>RT에서 입력 측 양자화만 활성화 | `false`<br>'거짓' |
| `output-dequantization-in-rt=1`<br>`출력-역양자화-in-rt=1` | Enable only output-side dequantization in RT<br>RT에서는 출력측 역양자화만 활성화합니다. | `false`<br>'거짓' |

**Relationship**: `edge-quantization-in-rt=1` is the **union** of
**관계**: `edge-Quantization-in-rt=1`은 다음의 **합집합**입니다.
`input-quantization-in-rt=1` and `output-dequantization-in-rt=1`.
`입력-양자화-in-rt=1` 및 `출력-역양자화-in-rt=1`.
Setting the
설정
umbrella flag is equivalent to setting both individual flags.
우산 플래그는 두 개별 플래그를 모두 설정하는 것과 같습니다.

### How Edge Quantization Interacts with FPConversionPass
### Edge Quantization이 FPConversionPass와 상호 작용하는 방법

The FPConversionPass QDQ detection algorithm uses these flags to decide
FPConversionPass QDQ 감지 알고리즘은 이러한 플래그를 사용하여 결정합니다.
whether standalone Q/DQ operations at function boundaries should be preserved
기능 경계에서 독립형 Q/DQ 연산을 보존해야 하는지 여부
or converted:
또는 변환됨:

1. **With `input-quantization-in-rt=1` or `edge-quantization-in-rt=1`**:
1. **`input-Quantization-in-rt=1` 또는 `edge-Quantization-in-rt=1` 사용**:
Standalone Quantize ops that consume a block argument (function input)
블록 인수(함수 입력)를 사용하는 독립형 Quantize 작업
directly AND are not part of a fake QDQ pair are added to the
직접적으로 그리고 가짜 QDQ 쌍의 일부가 아닙니다.
`keepTypeSet`.
`keepTypeSet`.
Their types are preserved so the runtime can handle the
해당 유형은 런타임에서 처리할 수 있도록 보존됩니다.
conversion.
전환.
The scale factor from the QuantizeLinear is propagated to the
QuantizeLinear의 스케일 팩터는 다음으로 전파됩니다.
`rt_transformations` annotation in the HSI JSON.
HSI JSON의 `rt_transformations` 주석.

2. **With `output-dequantization-in-rt=1` or `edge-quantization-in-rt=1`**:
2. **`output-deQuantization-in-rt=1` 또는 `edge-Quantization-in-rt=1` 사용**:
Standalone Dequantize ops that feed a function return AND are not part of
함수 반환을 제공하고 다음의 일부가 아닌 독립형 역양자화 작업
a fake QDQ pair are added to the `keepTypeSet`.
가짜 QDQ 쌍이 `keepTypeSet`에 추가됩니다.
Their types are preserved
해당 유형이 보존됩니다.
so the runtime can handle the output conversion.
런타임에서 출력 변환을 처리할 수 있습니다.
The scale factor from the
스케일 팩터
DequantizeLinear is propagated to the output `rt_transformations`.
DeQuantizeLinear는 'rt_transformations' 출력으로 전파됩니다.

3. **Without these flags**: Edge Q/DQ operations are treated like any other
3. **이러한 플래그가 없으면**: Edge Q/DQ 작업은 다른 작업과 마찬가지로 처리됩니다.
operation — they may be force-converted to the target type (bf16/fp16) and
작업 - 대상 유형(bf16/fp16)으로 강제 변환될 수 있으며
implemented as NPU DequantizeLinear/QuantizeLinear kernels.
NPU DeQuantizeLinear/QuantizeLinear 커널로 구현되었습니다.

### Interaction with MoveCastDown Pass
### MoveCastDown Pass와의 상호 작용

The `MoveCastDown` pass attempts to move `tosa.cast` operations closer to
'MoveCastDown' 패스는 'tosa.cast' 작업을 더 가까이 이동하려고 시도합니다.
function boundaries (block arguments or return ops), past layout
함수 경계(블록 인수 또는 반환 작업), 과거 레이아웃
transformation ops (reshape, transpose).
변환 작업(모양 변경, 전치).
This matters for edge quantization
이는 에지 양자화에 중요합니다.
because:
왜냐하면:

- When `fp-conversion-keep-function-signature=1` is active, casts are inserted
- `fp-conversion-keep-function-signature=1`이 활성화되면 캐스트가 삽입됩니다.
at function edges to convert between the original signature type and the
함수 가장자리에서 원래 서명 유형과 서명 유형 사이를 변환합니다.
internal NPU type.
내부 NPU 유형.
- MoveCastDown moves these casts past reshape/transpose chains so they sit
- MoveCastDown은 이러한 캐스트를 변형/전치 체인을 지나 이동하여 자리를 잡게 합니다.
directly at the boundary, making them candidates for RT handling.
경계에서 직접적으로 RT 처리 후보가 됩니다.
- If edge quantization in RT is ALSO enabled, the combined effect is that the
- RT의 에지 양자화도 활성화된 경우 결합된 효과는 다음과 같습니다.
runtime handles both the type cast AND the quantization at the boundary,
런타임은 유형 캐스팅과 경계에서의 양자화를 모두 처리합니다.
reducing NPU kernel count.
NPU 커널 수를 줄입니다.

### When to Enable Edge Quantization in RT
### RT에서 Edge Quantization을 활성화해야 하는 경우

| Scenario<br>시나리오 | Recommendation<br>추천 | Rationale<br>이론적 근거 |
|----------|---------------|-----------|
| Model inputs are float, NPU works in int8<br>모델 입력은 부동 소수점이며 NPU는 int8에서 작동합니다. | `input-quantization-in-rt=1`<br>`입력-양자화-in-rt=1` | Runtime quantizes inputs on host — avoids an NPU Q kernel<br>런타임은 호스트의 입력을 양자화하여 NPU Q 커널을 방지합니다. |
| Model outputs are float, NPU produces int8<br>모델 출력은 float이고 NPU는 int8을 생성합니다. | `output-dequantization-in-rt=1`<br>`출력-역양자화-in-rt=1` | Runtime dequantizes outputs on host — avoids an NPU DQ kernel<br>런타임은 호스트의 출력을 역양자화하여 NPU DQ 커널을 방지합니다. |
| Both of the above<br>위의 두 가지 모두 | `edge-quantization-in-rt=1`<br>`가장자리 양자화-in-rt=1` | Enables both directions<br>양방향 활성화 |
| Pure int8-in/int8-out model<br>순수 int8-in/int8-out 모델 | Not needed<br>필요하지 않음 | No float↔int8 conversion at edges<br>가장자리에서 float⇔int8 변환 없음 |
| Full BF16 model<br>전체 BF16 모델 | Not needed<br>필요하지 않음 | No quantization at all<br>양자화 전혀 없음 |
| Models with floating-point boundaries<br>부동 소수점 경계가 있는 모델 | Evaluate case-by-case<br>사례별로 평가 | If the outermost boundary op feeds directly into return, edge quant may help<br>가장 바깥쪽 경계 작업이 직접 수익으로 공급되는 경우 가장자리 퀀트가 도움이 될 수 있습니다. |

### Tradeoffs
### 트레이드오프

| Aspect<br>측면 | Edge Quant Enabled<br>엣지 퀀트 활성화 | Edge Quant Disabled<br>엣지 퀀트 비활성화 |
|--------|-------------------|---------------------|
| **NPU kernel count**<br>**NPU 커널 수** | Fewer (Q/DQ handled by RT)<br>더 적음(RT에서 Q/DQ 처리) | More (Q/DQ are NPU kernels)<br>더보기 (Q/DQ는 NPU 커널입니다) |
| **Host CPU work**<br>**호스트 CPU 작업** | More (float↔int8 on host)<br>더 보기(호스트의 float⇔int8) | Less (NPU handles everything)<br>적게(NPU가 모든 것을 처리함) |
| **Latency**<br>**지연** | May increase if host is bottleneck<br>호스트에 병목 현상이 발생하면 증가할 수 있음 | May increase if NPU is kernel-count limited<br>NPU가 커널 개수로 제한된 경우 증가할 수 있습니다. |
| **Data movement**<br>**데이터 이동** | Int8 crosses PCIe/AXI (smaller)<br>Int8은 PCIe/AXI(더 작음)를 교차합니다. | Float crosses PCIe/AXI (larger)<br>Float는 PCIe/AXI(대형)와 교차합니다. |
| **Accuracy**<br>**정확도** | Identical (same math, different executor)<br>동일(동일한 수학, 다른 실행자) | Identical<br>동일 |
| **Compilation**<br>**편집** | Scale factors in `flexmlrt-hsi.json`<br>`flexmlrt-hsi.json`의 배율 인수 | Scale factors baked into NPU graph<br>NPU 그래프에 구운 배율 인자 |

**General guidance**:
**일반 지침**:
- For **embedded** (single NPU partition, Telluride): edge quantization in RT
- **임베디드**(단일 NPU 파티션, Telluride): RT의 에지 양자화
is beneficial when the model has float inputs but an int8 core.
모델에 float 입력이 있지만 int8 코어가 있는 경우 유용합니다.
The runtime
런타임
handles the quantization efficiently on the host NPU interface.
호스트 NPU 인터페이스에서 효율적으로 양자화를 처리합니다.
- For **performance-sensitive** deployments: measure both configurations.
- **성능에 민감한** 배포의 경우: 두 구성을 모두 측정합니다.
Edge quant reduces NPU kernel pressure but adds host-side work.
Edge Quant는 NPU 커널 압력을 줄이지만 호스트 측 작업을 추가합니다.
- For **VAIP** flow: edge quantization flags may be needed to produce
- **VAIP** 흐름의 경우: 생성하려면 에지 양자화 플래그가 필요할 수 있습니다.
correct `flexmlrt-hsi.json` annotations that the runtime stack expects.
런타임 스택이 예상하는 'flexmlrt-hsi.json' 주석을 수정하세요.

### Estimating Impact
### 영향 추정

To estimate whether edge quantization helps:
에지 양자화가 도움이 되는지 추정하려면 다음을 수행하십시오.

1. **Compile without edge quant** and note the partition report's kernel count
1. **에지 퀀트 없이 컴파일**하고 파티션 보고서의 커널 수를 확인합니다.
2. **Compile with edge quant** (`edge-quantization-in-rt=1`) and compare
2. **에지 퀀트**(`edge-Quantization-in-rt=1`)로 컴파일하고 비교합니다.
3. If kernel count decreases AND no new CpuBecause messages appear, the edge
3. 커널 수가 감소하고 새로운 CpuBecause 메시지가 나타나지 않으면 에지는
quant configuration is viable
퀀트 구성이 가능합니다
4. For performance measurement, run both configurations on the board and
4. 성능 측정을 위해 보드에서 두 구성을 모두 실행하고
compare latency
지연 시간 비교

### Example Configuration with Edge Quantization
### 에지 양자화를 사용한 구성 예

```json
{
  "enable_f32_to_bf16_conversion": true,
  "fe_args": "match-power-of-two-quant-kernel=1 edge-quantization-in-rt=1"
}
```

Or individually controlled:
또는 개별적으로 제어됩니다.
```json
{
  "enable_f32_to_bf16_conversion": true,
  "fe_args": "match-power-of-two-quant-kernel=1 input-quantization-in-rt=1 output-dequantization-in-rt=1"
}
```
