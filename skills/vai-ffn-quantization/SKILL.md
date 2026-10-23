<!-- ko-translation: enabled -->
---
name: vai-ffn-quantization
license: Apache-2.0 WITH LLVM-exception
compatibility: Requires VitisAI 6.3+
metadata:
  version: 6.3.0
  stage: beta
description: 'Identify and quantize Feed-Forward Network (FFN) layers in ONNX models.
  An FFN is any MatMul+Add (weight+bias) linear layer: MLP projections, attention
  QKV projections, attention output projections, and classifier heads. This skill
  detects all FFN patterns, groups them into blocks, and applies per-layer INT8 quantization.'
---
<!--- Copyright (C) 2026 Advanced Micro Devices, Inc. All rights reserved.--->

# FFN Quantization Sub-Skill
# FFN 양자화 하위 스킬

## Access command
## 액세스 명령

This skill must always be accessed through the worker, never in a different way.
이 기술은 항상 작업자를 통해 접근해야 하며 다른 방식으로는 접근할 수 없습니다.
Allowed worker: `vai-mixed-precision-worker`.
허용되는 작업자: `vai-mixed-precision-worker`.

## Parse Arguments
## 인수 구문 분석

This skill expects arguments in this form:
이 기술은 다음 형식의 인수를 기대합니다.

```
--model <onnx_model> [--precision int8|bf16|fp32] [--output <quantized.onnx>] [--list-only] [--layer-filter <regex>]
```

- `--model <path>`: **(MANDATORY)** Path to the input ONNX model (.onnx or .onnxtxt)
- `--model <경로>`: **(필수)** 입력 ONNX 모델의 경로(.onnx 또는 .onnxtxt)
- `--precision <int8|bf16|fp32>`: **(OPTIONAL, default: int8)** Target precision for the detected FFN weights and biases
- `--precision <int8|bf16|fp32>`: **(선택 사항, 기본값: int8)** 감지된 FFN 가중치 및 편향에 대한 목표 정밀도
- `--output <path>`: **(OPTIONAL)** Path for the quantized output model.
- `--output <path>`: **(선택 사항)** 양자화된 출력 모델의 경로입니다.
- Required unless `--list-only` is given.
- `--list-only`가 지정되지 않은 경우 필수입니다.
- `--list-only`: **(OPTIONAL)** Only detect and report FFN layers/blocks; do not modify the model
- `--list-only`: **(선택 사항)** FFN 레이어/블록만 감지하고 보고합니다. 모델을 수정하지 마세요
- `--layer-filter <regex>`: **(OPTIONAL)** Restrict quantization to FFN layers whose MatMul node name matches this regular expression
- `--layer-filter <regex>`: **(선택 사항)** MatMul 노드 이름이 이 정규 표현식과 일치하는 FFN 레이어로 양자화를 제한합니다.

## Scope
## 범위

This skill identifies **all Feed-Forward Network (FFN) layers** in ONNX models
이 기술은 ONNX 모델에서 **모든 FFN(Feed-Forward Network) 레이어**를 식별합니다.
and applies quantization to their weights and biases.
가중치와 편향에 양자화를 적용합니다.
An FFN is any linear
FFN은 선형입니다.
layer represented as a `MatMul` with a 2D weight initializer followed by an
2D 가중치 초기화와 그 뒤에 오는 'MatMul'로 표현되는 레이어
`Add` with a 1D bias initializer.
1D 바이어스 초기화를 사용하여 '추가'합니다.
This includes:
여기에는 다음이 포함됩니다.

- MLP projections (up-projection, down-projection)
- MLP 투영(상향 투영, 하향 투영)
- Attention QKV projections (fused or separate)
- 주의 QKV 투영(융합 또는 분리)
- Attention output projections
- 주의 출력 예측
- Classifier/head linear layers
- 분류기/헤드 선형 레이어

An FFN layer consists of:
FFN 레이어는 다음으로 구성됩니다.
- A `MatMul` node where one input is a 2D graph initializer (weight)
- 하나의 입력이 2D 그래프 이니셜라이저(가중치)인 `MatMul` 노드
- An `Add` node immediately downstream whose other input is a 1D initializer (bias)
- 다른 입력이 1D 이니셜라이저(바이어스)인 'Add' 노드 바로 다운스트림
- Optionally followed by an activation function (GELU, ReLU, SiLU, etc.)
- 선택적으로 활성화 함수(GELU, ReLU, SiLU 등)가 뒤따릅니다.

FFNs are grouped into **blocks** based on their structural role in the model:
FFN은 모델에서의 구조적 역할에 따라 **블록**으로 그룹화됩니다.

| Block Type<br>블록 유형 | FFNs per Block<br>블록당 FFN | Description<br>설명 |
|------------|----------------|-------------|
| **MLP block**<br>**MLP 블록** | 2 | Up-projection (with activation) + down-projection (no activation)<br>상향 투영(활성화 있음) + 하향 투영(활성화 없음) |
| **Attention block**<br>**주의 차단** | 2 | QKV in-projection (no activation) + output projection (no activation)<br>QKV 내투영(활성화 없음) + 출력 투사(활성화 없음) |
| **Tail**<br>**꼬리** | 1–2 | Standalone linear layers (classifiers, heads) at model start/end<br>모델 시작/끝의 독립형 선형 레이어(분류자, 헤드) |

## What This Skill Does
## 이 기술의 역할

1. **Pattern Detection**: Scans an ONNX graph for **all** MatMul+Add patterns
1. **패턴 감지**: **모든** MatMul+패턴 추가에 대한 ONNX 그래프를 스캔합니다.
where the MatMul has a 2D weight initializer and the downstream Add has a
여기서 MatMul에는 2D 가중치 초기화가 있고 다운스트림 Add에는
1D bias initializer.
1D 바이어스 초기화.
Does NOT filter by dimension ratio or structural role —
차원 비율이나 구조적 역할을 기준으로 필터링하지 않습니다. —
every such pattern is an FFN.
이러한 모든 패턴은 FFN입니다.

2. **Activation Detection**: For each FFN, checks whether the Add output feeds
2. **활성화 감지**: 각 FFN에 대해 출력 피드 추가 여부를 확인합니다.
directly into an activation function (Gelu, Relu, Silu, etc.).
활성화 함수(Gelu, Relu, Silu 등)에 직접 적용됩니다.

3. **Block Grouping**: Groups FFNs into blocks (MLP blocks, attention blocks,
3. **블록 그룹화**: FFN을 블록(MLP 블록, 주의 블록,
tail) based on weight shapes and activation presence.
꼬리) 무게 모양과 활성화 여부를 기반으로 합니다.

4. **Weight/Bias Extraction**: Reports the weight and bias initializer names,
4. **가중치/편향 추출**: 가중치 및 편향 이니셜라이저 이름을 보고합니다.
shapes, and current quantization state for each detected FFN layer.
감지된 각 FFN 레이어에 대한 모양 및 현재 양자화 상태.

5. **Quantization Application**: Inserts QuantizeLinear/DequantizeLinear nodes
5. **양자화 애플리케이션**: QuantizeLinear/DeQuantizeLinear 노드 삽입
around FFN weights and biases to quantize them to INT8 (symmetric,
FFN 가중치와 편향을 중심으로 INT8(대칭,
per-tensor).
텐서당).
VINT8 on AMD NPU is always per-tensor quantization.
AMD NPU의 VINT8은 항상 텐서별 양자화입니다.

## Supported FFN Patterns
## 지원되는 FFN 패턴

The detection handles these structural variants:
감지는 다음과 같은 구조적 변형을 처리합니다.

| Pattern<br>패턴 | Description<br>설명 |
|---------|-------------|
| `MatMul → Add`<br>`MatMul → 추가` | Basic FFN: weight matmul followed by bias add<br>기본 FFN: 가중치 계산 후 편향 추가 |
| `MatMul → Add → Activation`<br>`MatMul → 추가 → 활성화` | FFN followed by an activation (GELU, ReLU, SiLU, etc.)<br>FFN에 이어 활성화(GELU, ReLU, SiLU 등) |
| `MatMul → Q → DQ → Add`<br>`MatMul → Q → DQ → 추가` | Already-quantized FFN (INT8 Q/DQ between matmul and bias)<br>이미 양자화된 FFN(matmul과 바이어스 간 INT8 Q/DQ) |
| `MatMul → BFPQuantizeDequantize → Add`<br>`MatMul → BFPQuantizeDeQuantize → 추가` | MX/BFP-quantized FFN<br>MX/BFP 양자화 FFN |
| `Gemm` (with bias)<br>'Gemm'(편향 있음) | Fused MatMul+Add as a single Gemm node<br>Fused MatMul+단일 Gemm 노드로 추가 |
| `Gemm → Q → DQ → Reshape → Q → DQ → Activation` | Quantized FFN with activation behind Q/
| `Gemm → Q → DQ → Reshape → Q → DQ → 활성화` | Q/ 뒤에서 활성화되는 양자화된 FFN
DQ/Reshape chain |
DQ/체인 변형 |
| `Add → Q → DQ → Reshape → Q → DQ → Activation`<br>`추가 → Q → DQ → 변형 → Q → DQ → 활성화` | Quantized bias Add with activation behind pass-through ops<br>통과 작업 뒤에 활성화된 양자화된 바이어스 추가 |

### Activation Detection Through Q/DQ Chains
### Q/DQ 체인을 통한 활성화 감지

In quantized (QDQ) models, activation functions are not the direct consumer
양자화(QDQ) 모델에서 활성화 함수는 직접적인 소비자가 아닙니다.
of the bias Add/Gemm output.
바이어스 Add/Gemm 출력.
Instead, there are intermediate pass-through
대신 중간 통과가 있습니다.
nodes (QuantizeLinear, DequantizeLinear, Reshape, Cast, BFPQuantizeDequantize)
노드(QuantizeLinear, DeQuantizeLinear, Reshape, Cast, BFPQuantizeDeQuantize)
between the FFN output and the activation.
FFN 출력과 활성화 사이.
The detection traces forward through
탐지는 다음을 통해 추적됩니다.
single-consumer chains of these pass-through ops (up to 8 hops) to find
이러한 통과 작업(최대 8홉)의 단일 소비자 체인을 찾아
activations that logically belong to the FFN layer.
논리적으로 FFN 계층에 속하는 활성화.

## Detection Algorithm
## 탐지 알고리즘

An FFN is identified by three criteria (all must hold):
FFN은 세 가지 기준으로 식별됩니다(모두 충족해야 함).

1. **Has a 2D weight initializer**: One input to the MatMul is a graph
1. **2D 가중치 초기화 기능 있음**: MatMul에 대한 입력 중 하나는 그래프입니다.
initializer with exactly 2 dimensions `[in_features, out_features]`.
정확히 2차원 `[in_features, out_features]`를 갖는 초기화 프로그램입니다.
This excludes attention score MatMuls (Q×K^T, attn×V) where both
이는 주의 점수 MatMuls(Q×K^T, attn×V)를 제외합니다.
inputs are runtime tensors.
입력은 런타임 텐서입니다.
2. **Has a bias Add immediately downstream**: The MatMul output (possibly
2. **바이어스가 있음 다운스트림에 즉시 추가**: MatMul 출력(아마도
through Q/DQ) feeds into an Add whose other input is a 1D initializer.
Q/DQ를 통해)는 다른 입력이 1D 초기화인 Add에 공급됩니다.
3. **Weight is 2D**: Not 3D/4D (which would indicate a batched matmul).
3. **무게는 2D입니다**: 3D/4D가 아닙니다(일괄 처리된 매트물을 나타냄).

No dimension-ratio filtering is applied.
차원 비율 필터링이 적용되지 않습니다.
All MatMul+Add patterns meeting
모든 MatMul+Add 패턴 미팅
these criteria are FFNs regardless of their structural role (MLP, attention
이러한 기준은 구조적 역할(MLP, 주의력)에 관계없이 FFN입니다.
projection, classifier head, etc.).
투영, 분류기 헤드 등).

### Previous error (corrected)
### 이전 오류(수정됨)

Earlier versions of this skill used dimension-ratio heuristics (e.g.,
이 기술의 이전 버전에서는 차원 비율 휴리스틱을 사용했습니다(예:
`out_features / in_features ∈ {2, 3, 4, 8}`) and structural context to
`out_features / in_features ∈ {2, 3, 4, 8}`) 및 구조적 컨텍스트
exclude attention projections from FFN detection.
FFN 감지에서 주의 예측을 제외합니다.
This was wrong — attention
이건 틀렸어 - 주의
QKV projections (`dim → 3*dim`) and output projections (`dim → dim`) are
QKV 투영(`dim → 3*dim`) 및 출력 투영(`dim → 희미함`)은 다음과 같습니다.
equally valid FFNs.
동등하게 유효한 FFN.
The ratio heuristic also failed for fused QKV projections
융합된 QKV 예측에 대해서도 비율 휴리스틱이 실패했습니다.
(ratio 3) and square projections (ratio 1).
(비율 3) 및 정사각형 투영(비율 1).

## Usage Examples
## 사용 예

### FFN detection and quantization
### FFN 감지 및 양자화

```bash
# List all detected FFN layers without modifying the model
python ffn_quantize.py --input model.onnx --list-only

# Quantize all FFN weights and biases to INT8
python ffn_quantize.py --input model.onnx --output model_ffn_int8.onnx --precision int8

# Quantize only specific layers (by regex on node name)
python ffn_quantize.py --input model.onnx --output model_ffn_int8.onnx \
    --precision int8 --layer-filter "mlp|ffn"

# Dequantize FFN layers back to float (for re-quantization experiments)
python ffn_quantize.py --input model_quant.onnx --output model_ffn_fp32.onnx --precision fp32
```

### FFN block extraction and reporting
### FFN 블록 추출 및 보고

```bash
# Report all FFN blocks in a model
python extract_ffns.py --input model.onnx

# JSON output
python extract_ffns.py --input model.onnx --format json

# Filter by node name pattern
python extract_ffns.py --input model.onnx --layer-filter "mlp"
```

## Example: ViT Encoder (vit_encoder_b1_s256_d512_m2048_h16_l12)
## 예: ViT 인코더(vit_encoder_b1_s256_d512_m2048_h16_l12)

This model has **48 FFNs** organized into 24 blocks:
이 모델에는 **48개의 FFN**이 24개의 블록으로 구성되어 있습니다.

| Block Type<br>블록 유형 | Count<br>개수 | FFNs/Block<br>FFN/블록 | FFN 1<br>FFN 1 | FFN 2<br>FFN 2 | Total FFNs<br>총 FFN |
|------------|-------|------------|-------|-------|------------|
| MLP block<br>MLP 블록 | 12 | 2 | up-proj [512→2048] + Gelu<br>up-proj [512→2048] + 겔루 | down-proj [2048→512]<br>다운프로젝트 [2048→512] | 24 |
| Attention block<br>주의 차단 | 12 | 2 | QKV in-proj [512→1536]<br>QKV 인프로젝트 [512→1536] | out-proj [512→512]<br>아웃프로젝트 [512→512] | 24 |

- **MLP blocks**: One FFN with activation (Gelu), one without
- **MLP 블록**: 활성화된 FFN(Gelu) 1개, 활성화되지 않은 FFN 1개
- **Attention blocks**: Both FFNs without activation
- **주의 차단**: 활성화되지 않은 두 FFN 모두
- **Tail**: None (model ends with LayerNormalization)
- **꼬리**: 없음(모델이 LayerNormalization으로 끝남)
- **Attention score MatMuls** (Q×K^T, attn×V): 24 nodes excluded — both
- **주의 점수 MatMuls** (Q×K^T, attn×V): 24개 노드 제외 — 둘 다
inputs are runtime tensors, no weight initializer
입력은 런타임 텐서이며 가중치 초기화는 없습니다.

## Integration with Mixed-Precision Workflow
## 혼합 정밀도 워크플로와 통합

This skill is a building block for the broader mixed-precision strategy:
이 기술은 더 광범위한 혼합 정밀도 전략을 위한 구성 요소입니다.

1. Run pattern detection to identify FFN layers
1. 패턴 감지를 실행하여 FFN 레이어 식별
2. Quantize FFN weights/biases to INT8 (they are typically robust to quantization)
2. FFN 가중치/편향을 INT8로 양자화합니다(일반적으로 양자화에 강력함).
3. If accuracy degrades, selectively promote specific FFN layers to BF16
3. 정확도가 저하되면 특정 FFN 레이어를 선택적으로 BF16으로 승격합니다.
4. Combine with the precision choices selected for attention layers (INT8 for FFN)
4. Attention 레이어(FFN의 경우 INT8)에 대해 선택한 정밀 선택과 결합

## Residual-Add Exclusion (mandatory for ViT encoders)
## 잔여 추가 제외(ViT 인코더의 경우 필수)

From the ViT encoder quantization sweep (12 shapes, d512–d2048,
ViT 인코더 양자화 스윕(12개 모양, d512–d2048,
s256–s1024): **baseline per-tensor INT8 on the full encoder collapses to
s256–s1024): **전체 인코더의 텐서당 기준선 INT8은 다음으로 축소됩니다.
0.10% top-1**.
0.10% 상위 1**.
No INT8 scheme recovers it — asymmetric, float-scale,
INT8 체계는 이를 복구하지 않습니다(비대칭, 부동 규모,
SmoothQuant+BiasCorrection all stay ≤0.20%.
SmoothQuant+BiasCorrection은 모두 0.20% 이하로 유지됩니다.
Only more bits (INT16) or
추가 비트(INT16) 또는
**excluding the deep residual-stream Adds** from INT8 does.
**INT8의 깊은 잔류 스트림 추가 제외**는 그렇습니다.

The cause is entirely the 13 residual-stream Add outputs
원인은 전적으로 13개의 잔여 스트림 Add 출력입니다.
(`add_12`..`add_24`), which carry outlier activations (max |x| ≈ 277–281).
(`add_12`..`add_24`), 이상값 활성화(최대 |x| ≒ 277–281)를 수행합니다.
These interleave into two kinds — attention residual Adds (odd indices) and
이것들은 두 가지 종류, 즉 어텐션 잔여 추가(홀수 지수)와
MLP residual Adds (even indices, deep) — and **both branches must be
MLP 잔여 추가(심지어 인덱스, 깊은) - **두 분기 모두 다음과 같아야 합니다.
excluded**: dropping only one branch collapses accuracy back to ≈0%.
제외됨**: 분기 하나만 삭제하면 정확도가 다시 0%로 축소됩니다.

| Excluded → BF16 set<br>제외 → BF16 세트             | count<br>카운트 | top-1 (2k)<br>상위 1개(2k) |
|----------------------------------|------:|-----------:|
| 13 Adds (`add_12..24`)<br>13개 추가(`add_12..24`)           | 13    |   68.30 %  |
| only 12 attention Adds (odd)<br>관심 12개만 추가(이상)     | 12    |   0.05 %   |
| only 7 MLP Adds (even, deep)<br>단 7개의 MLP 추가(짝수, 깊은)    | 7     |   0.15 %   |

Late residual Adds (20–23) are the most sensitive; early ones tolerate INT8
늦은 잔여 추가(20-23)가 가장 민감합니다. 초기 사람들은 INT8을 용인합니다
better — consistent with residual magnitude growing with depth.
더 좋음 — 깊이에 따라 증가하는 잔여 규모와 일치합니다.

### BF16 casting pattern for excluded Adds
### 제외된 추가에 대한 BF16 캐스팅 패턴

FP32, INT16, and BF16 all recover equally for the excluded Adds.
FP32, INT16 및 BF16은 모두 제외된 추가에 대해 동일하게 복구됩니다.
Use BF16
BF16을 사용하세요
(via cast-style QDQ) for deployment efficiency:
(캐스트 스타일 QDQ를 통해) 배포 효율성을 위해:

**Detection** — for each residual Add, inspect what consumes its output:
**탐지** — 각 잔여 추가에 대해 출력을 소비하는 항목을 검사합니다.
```python
cons_optypes = {c.op_type for c in consumers.get(add_node.output[0], [])}
# {'QuantizeLinear'} → still INT8, needs conversion
# {'Cast'}           → already BF16, skip
```

**Fix** — replace each `QuantizeLinear` with `Cast(to=BFLOAT16)` (single
**수정** — 각 `QuantizeLinear`를 `Cast(to=BFLOAT16)`로 교체(단일)
input: the Add output; drop scale/zero_point inputs) and each downstream
입력: 추가 출력; 드롭 스케일/zero_point 입력) 및 각 다운스트림
`DequantizeLinear` with `Cast(to=FLOAT)`.
`Cast(to=FLOAT)`를 사용한 `DeQuantizeLinear`.
Delete the now-unused
현재 사용하지 않는 삭제
`_scale`/`_zero_point` initializers.
`_scale`/`_zero_point` 초기화 프로그램.
Fix stale `value_info` dtype entries.
오래된 `value_info` dtype 항목을 수정합니다.

### Dual-branch nuance: LayerNorm branch stays INT8
### 이중 분기 뉘앙스: LayerNorm 분기는 INT8로 유지됩니다.

Each residual Add output feeds two branches: one into `LayerNormalization`
각 잔여 추가 출력은 두 개의 분기를 제공합니다. 하나는 `LayerNormalization`에 제공됩니다.
(the next FFN/MHA sublayer input) and one carrying the residual stream
(다음 FFN/MHA 하위 계층 입력) 및 잔여 스트림을 전달하는 것
forward to the next Add. **Only the residual-carry branch should be BF16.**
다음 추가로 전달됩니다. **잔차 캐리 분기만 BF16이어야 합니다.**
The LayerNorm-feeding branch should remain INT8, matching the model's general
LayerNorm 공급 분기는 모델의 일반 사항과 일치하여 INT8로 유지되어야 합니다.
precision recipe.
정밀레시피.
If step 1 of graph surgery over-converts both branches,
그래프 수술의 1단계에서 두 가지 모두 과도하게 변환되면
step 6 (`restore_int8_layernorm_branch.py`) must revert just the
6단계(`restore_int8_layernorm_branch.py`)에서는
LayerNorm-feeding branch by recovering the original scale/zero_point from
원래 scale/zero_point를 복구하여 LayerNorm 공급 분기
pre-surgery model state and replacing the Cast pair back to
수술 전 모델 상태 및 Cast 쌍을 다시 교체
QuantizeLinear/DequantizeLinear.
QuantizeLinear/DeQuantizeLinear.

### Quark configuration for BF16 Add exclusion
### BF16용 Quark 구성 제외 추가

Set `BF16QDQToCast = True` in `extra_options` and pass the 13 Add output
`extra_options`에서 `BF16QDQToCast = True`를 설정하고 13 Add 출력을 전달합니다.
tensor names under `ExtendedQuantType.QBFloat16` in `MixedPrecisionTensor`.
'MixedPrecisionTensor'의 'ExtendedQuantType.QBFloat16' 아래에 있는 텐서 이름.
Always set `DedicatedQDQPair = True` explicitly (AIESW-30603) — block
항상 `DedicatedQDQPair = True`를 명시적으로 설정합니다(AIESW-30603) — 블록
floating-point presets don't enable it by default, and without it the VAIML
부동 소수점 사전 설정은 기본적으로 이를 활성화하지 않으며, 이것이 없으면 VAIML
frontend over-reports unsupported ops in fused clusters.
프런트엔드는 융합된 클러스터에서 지원되지 않는 작업을 과도하게 보고합니다.

## >2GB Model Handling
## >2GB 모델 처리

Models exceeding ~2GB (e.g. d2048 shapes) trigger two separate crashes in
2GB를 초과하는 모델(예: d2048 모양)은 두 번의 별도 충돌을 유발합니다.
Quark 0.12's `quantize_static`:
Quark 0.12의 'Quarkize_static':

1. **Auto-detect crash**: `ByteSize()` is called before the external-data
1. **충돌 자동 감지**: 'ByteSize()'는 외부 데이터가 호출되기 전에 호출됩니다.
flag can be set, itself triggering the protobuf 2GB overflow.
플래그가 설정되어 자체적으로 protobuf 2GB 오버플로를 트리거할 수 있습니다.
2. **Re-inline crash**: `check_onnx_model` → `create_infer_session_for_onnx_model`
2. **재인라인 충돌**: `check_onnx_model` → `create_infer_session_for_onnx_model`
re-inlines the cached model without forwarding `use_external_data_format`.
`use_external_data_format`을 전달하지 않고 캐시된 모델을 다시 인라인합니다.

**Fix**: auto-detect when external tensor bytes exceed ~90% of protobuf's 2GB
**수정**: 외부 텐서 바이트가 protobuf 2GB의 ~90%를 초과하면 자동 감지
limit (`--external-data auto|on|off`), pre-set `use_external_data_format=True`,
제한(`--external-data auto|on|off`), 사전 설정 `use_external_data_format=True`,
and monkeypatch `create_infer_session_for_onnx_model` to force the
그리고 Monkeypatch `create_infer_session_for_onnx_model`을 사용하여 강제로
external-data branch for any in-memory `ModelProto`.
메모리 내 'ModelProto'에 대한 외부 데이터 분기입니다.

For >2GB shapes, Quark never fuses `MatMul+bias-Add → Gemm`, so the unfused
2GB가 넘는 모양의 경우 Quark는 `MatMul+bias-Add → Gemm`을 융합하지 않으므로 융합되지 않은
boundary fix (`fix_mha_boundary_unfused.py`) is needed instead of the
대신 경계 수정(`fix_mha_boundary_unfused.py`)이 필요합니다.
fused-case fix.
융합 케이스 수정.

## Key Insight from Production Quantization
## 생산 수량화의 주요 통찰력

From ViT encoder quantization experience: all FFN layers (MLP projections
ViT 인코더 양자화 경험에서: 모든 FFN 레이어(MLP 프로젝션)
AND attention projections) are reliably quantizable to INT8 without accuracy
AND 주의 예측)은 정확도 없이 INT8로 안정적으로 수량화할 수 있습니다.
loss.
손실.
The accuracy-sensitive components are the residual-stream Adds (which
정확도에 민감한 구성요소는 잔여 스트림 추가(이것은
**must** be excluded to BF16 — see above) and the attention score
**반드시** BF16에서 제외되어야 함 - 위 참조) 및 관심 점수
computations (runtime×runtime MatMuls) — not the weight MatMuls.
계산(런타임×런타임 MatMuls) — MatMuls 가중치가 아닙니다.
This makes
이것은
INT8 FFN quantization a safe default that delivers performance without
INT8 FFN 양자화는 문제 없이 성능을 제공하는 안전한 기본값입니다.
sacrificing accuracy, provided the residual-Add exclusion is applied.
잔여 추가 제외가 적용되는 경우 정확도가 희생됩니다.
