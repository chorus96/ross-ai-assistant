<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->
# ResNet50 — Quantize to VINT8+BF16 vs. the BF16 Design
# ResNet50 — VINT8+BF16 대 BF16 설계로 양자화

**Quantize** a ResNet50 directly to a mixed **VINT8+BF16** design and show it
ResNet50을 혼합 **VINT8+BF16** 설계로 직접 **양자화**하고 표시합니다.
**matches the accuracy of the all-BF16 design** while running faster, and is
**모든 BF16 설계의 정확성과 일치**하면서 더욱 빠르게 작동합니다.
**clearly more accurate than the full-VINT8 design** — using fake quantization
**완전한 VINT8 디자인보다 확실히 더 정확함** — 가짜 양자화 사용
with QuantizeLinear and DequantizeLinear for **VINT8**.
**VINT8**에 대해 QuantizeLinear 및 DeQuantizeLinear를 사용합니다.

The model is a ResNet50 fine-tuned on CIFAR-10 (input `1x3x32x32`, 10 classes),
모델은 CIFAR-10('1x3x32x32' 입력, 10개 클래스)에서 미세 조정된 ResNet50입니다.
the same one used by the Vitis-AI
Vitis-AI에서 사용하는 것과 동일한 것
[`resnet50_bf16_cifar10`](https://github.com/amd/Vitis-AI/blob/release/6.3/versal_2ve/examples/tutorials/resnet50_bf16_cifar10)
[`resnet50_bf16_cifar10`](https://github.com/amd/Vitis-AI/blob/release/6.3/versal_2ve/examples/tutorials/resnet50_bf16_cifar10)
tutorial, whose weights this example downloads.
이 예제에서 다운로드하는 가중치의 튜토리얼입니다.

## The three designs
## 세 가지 디자인

| Design<br>디자인 | How it is built<br>구축 방법 | Role<br>역할 |
|--------|-----------------|------|
| **BF16** (reference)<br>**BF16** (참고) | FP32 ONNX compiled with `enable_f32_to_bf16_conversion`. This *is* the [`resnet50_bf16_cifar10` BF16 design](https://github.com/amd/Vitis-AI/blob/release/6.3/versal_2ve/examples/tutorials/resnet50_bf16_cifar10).<br>'enable_f32_to_bf16_conversion'으로 컴파일된 FP32 ONNX. 이것은 *[`resnet50_bf16_cifar10` BF16 디자인](https://github.com/amd/Vitis-AI/blob/release/6.3/versal_2ve/examples/tutorials/resnet50_bf16_cifar10)입니다. | Accuracy reference (BF16 ≈ FP32).<br>정확도 기준(BF16 ≒ FP32). |
| **Full VINT8**<br>**전체 VINT8** | Quark symmetric INT8 (`XInt8Spec`) on every layer.<br>모든 레이어에 Quark 대칭 INT8(`XInt8Spec`)이 있습니다. | Fast, but loses accuracy.<br>빠르지만 정확성이 떨어집니다. |
| **VINT8+BF16** (this design)<br>**VINT8+BF16** (이 디자인) | Quark INT8 **except** the accuracy-critical layers, which are excluded → FP32 → compiler BF16.<br>Quark INT8 **정확도가 중요한 레이어 제외** → FP32 → 컴파일러 BF16. | Matches BF16 accuracy, faster than all-BF16.<br>모든 BF16보다 빠른 BF16 정확도와 일치합니다. |

## Goal
## 목표

Derive a VINT8+BF16 design that **does not lose accuracy versus the BF16
**BF16에 비해 정확도가 떨어지지 않는 VINT8+BF16 설계 도출
reference** and is **more accurate than the full-VINT8 design**, while still
참조**이며 **전체 VINT8 디자인보다 더 정확합니다**.
compiling to a **single NPU partition**.
**단일 NPU 파티션**으로 컴파일 중입니다.

## Quantization approach with force-conversion
## 강제 변환을 통한 양자화 접근 방식

BF16 regions are produced as follows: Accuracy-critical layers are **excluded from
BF16 영역은 다음과 같이 생성됩니다. 정확도가 중요한 레이어는 **제외됩니다.
INT8** (they stay FP32 — standard `QuantizeLinear`/`DequantizeLinear` everywhere
INT8** (그들은 어디에서나 FP32를 유지합니다 — 표준 `QuantizeLinear`/`DeQuantizeLinear`
else), and the VAIML compiler converts those FP32 islands to **BF16** via
else), VAIML 컴파일러는 다음을 통해 해당 FP32 섬을 **BF16**으로 변환합니다.
`enable_f32_to_bf16_conversion`.
'enable_f32_to_bf16_conversion'.

## Skills Used
## 사용된 스킬

- **vai-quantization-guide** — quantizes the FP32 model, selects the VINT8+BF16
- **vai-Quantization-guide** — FP32 모델을 양자화하고 VINT8+BF16을 선택합니다.
layer assignment, and validates accuracy against the BF16 reference.
레이어 할당을 수행하고 BF16 참조에 대한 정확성을 검증합니다.
- The quantization uses a **Quark JSON config** ([`input/vint8_bf16.quark.json`](input/vint8_bf16.quark.json)):
- 양자화는 **Quark JSON 구성**([`input/vint8_bf16.quark.json`](input/vint8_bf16.quark.json))을 사용합니다.
a global `XInt8Spec` plus an `exclude` list of the accuracy-critical layers.
전역 `XInt8Spec`과 정확도가 중요한 레이어의 `제외` 목록.
The skill writes this config (from its sensitivity analysis) and applies the
스킬은 민감도 분석을 통해 이 구성을 작성하고
exclusions via `QConfig.exclude` — the excluded layers stay FP32 and the
`QConfig.exclude`를 통한 제외 - 제외된 레이어는 FP32로 유지되고
compiler converts them to BF16.
컴파일러는 이를 BF16으로 변환합니다.
No custom quantization script is needed.
사용자 정의 양자화 스크립트가 필요하지 않습니다.

## Prerequisites
## 전제조건

- A **Vitis AI** environment (Quark, the VAIML `flexml` compiler, the
- **Vitis AI** 환경(Quark, VAIML `flexml` 컴파일러,
`VitisAIExecutionProvider`).
`VitisAIExecutionProvider`).
Provisioned inside the Vitis AI Docker container.
Vitis AI Docker 컨테이너 내부에 프로비저닝됩니다.
- `torch` + `torchvision` and the CIFAR-10 dataset (reference model, calibration
- `torch` + `torchvision` 및 CIFAR-10 데이터 세트(참조 모델, 교정
tensors, top-1 check).
텐서, 상위 1개 확인).
- Network access to the public [amd/Vitis-AI](https://github.com/amd/Vitis-AI/tree/release/6.3/versal_2ve/examples/tutorials/resnet50_bf16_cifar10)
- 대중에 대한 네트워크 액세스 [amd/Vitis-AI](https://github.com/amd/Vitis-AI/tree/release/6.3/versal_2ve/examples/tutorials/resnet50_bf16_cifar10)
repository to download the pretrained weights, or a local clone of it.
사전 훈련된 가중치 또는 해당 가중치의 로컬 복제본을 다운로드할 수 있는 저장소입니다.
- For the board step: a **VE2** board reachable via `/path/to/run_on_board`.
- 보드 단계: `/path/to/run_on_board`를 통해 연결할 수 있는 **VE2** 보드.

## Starting Point
## 출발점

Reproduction files in [`input/`](input/):
[`input/`](input/)의 재생산 파일:

| File<br>파일 | Role<br>역할 |
|------|------|
| [`fetch_model.py`](input/fetch_model.py)<br>[`fetch_model.py`](input/fetch_model.py) | Downloads the CIFAR-10 ResNet50 weights from the `resnet50_bf16_cifar10` tutorial and exports the FP32 ONNX — the BF16/quantization source.<br>'resnet50_bf16_cifar10' 튜토리얼에서 CIFAR-10 ResNet50 가중치를 다운로드하고 BF16/양자화 소스인 FP32 ONNX를 내보냅니다. |
| [`vint8_bf16.quark.json`](input/vint8_bf16.quark.json)<br>[`vint8_bf16.quark.json`](input/vint8_bf16.quark.json) | Quark config for the **VINT8+BF16** design — global `XInt8Spec` + an `exclude` list (the BF16-head layers, kept FP32 → compiler BF16). Consumed by the skill's `mixed_precision_quantize.py`. The skill can also produce this file.<br>**VINT8+BF16** 디자인을 위한 Quark 구성 — 전역 `XInt8Spec` + `제외` 목록(BF16 헤드 레이어, FP32 유지 → 컴파일러 BF16). 스킬의 `mixed_precision_Quantize.py`에 의해 소비됩니다. 스킬로 이 파일을 생성할 수도 있습니다. |
| [`full_vint8.quark.json`](input/full_vint8.quark.json)<br>[`full_vint8.quark.json`](input/full_vint8.quark.json) | Quark config for the **full VINT8** baseline (global `XInt8Spec`, no exclusions). The skill can also produce this file.<br>**전체 VINT8** 기준선에 대한 Quark 구성(전역 `XInt8Spec`, 제외 없음) 스킬로 이 파일을 생성할 수도 있습니다. |
| [`make_calib.py`](input/make_calib.py)<br>[`make_calib.py`](input/make_calib.py) | Exports CIFAR-10 calibration tensors as `ifm_*.npy`.<br>CIFAR-10 교정 텐서를 'ifm_*.npy'로 내보냅니다. |
| [`eval_top1.py`](input/eval_top1.py)<br>[`eval_top1.py`](input/eval_top1.py) | Top-1 / top-k accuracy on the CIFAR-10 test set (CPU). With `--reference`, also reports logit fidelity against the BF16 reference: argmax agreement, cosine similarity and PSNR.<br>CIFAR-10 테스트 세트(CPU)의 Top-1/top-k 정확도. `--reference`를 사용하면 BF16 참조에 대한 로짓 충실도(argmax 일치, 코사인 유사성 및 PSNR)도 보고합니다. |
| [`pin_batch1.py`](input/pin_batch1.py)<br>[`pin_batch1.py`](input/pin_batch1.py) | Pins the dynamic batch dim to 1 (VAIML needs static shapes).<br>동적 배치 밝기를 1로 고정합니다(VAIML에는 정적 모양이 필요함). |
| [`vitisai_config_bf16.json`](input/vitisai_config_bf16.json)<br>[`vitisai_config_bf16.json`](input/vitsai_config_bf16.json) | BF16 **reference** compile config (`enable_f32_to_bf16_conversion`, no Quark) — the `resnet50_bf16_cifar10` tutorial config.<br>BF16 **참조** 컴파일 구성(`enable_f32_to_bf16_conversion`, Quark 없음) - `resnet50_bf16_cifar10` 튜토리얼 구성. |
| [`vitisai_config_vint8_bf16.json`](input/vitisai_config_vint8_bf16.json)<br>[`vitisai_config_vint8_bf16.json`](input/vitsai_config_vint8_bf16.json) | VINT8+BF16 compile config (`ve2`, `enable_f32_to_bf16_conversion`, `group_args: "mixed_precision"`).<br>VINT8+BF16 컴파일 구성(`ve2`, `enable_f32_to_bf16_conversion`, `group_args: "mixed_precision"`). |
| [`calib/ifm_*.npy`](input/calib)<br>[`calib/ifm_*.npy`](입력/calib) | 8 sample calibration tensors (regenerate the full set with `make_calib.py`).<br>8개의 샘플 교정 텐서(`make_calib.py`를 사용하여 전체 세트를 재생성). |

## How to Run
## 달리는 방법

**Follow [`prompt.md`](prompt.md)**, or run the steps directly:
**[`prompt.md`](prompt.md)**를 따르거나 다음 단계를 직접 실행하세요.

```bash
# 0. FP32 reference model + CIFAR-10 data, then calibration tensors:
python input/fetch_model.py                     # -> models/resnet_trained_for_cifar10.onnx
python input/make_calib.py --num 64 --download  # -> calib/ifm_*.npy
```

`fetch_model.py` pulls `models/resnet_trained_for_cifar10.pt` from the
`fetch_model.py`는 `models/resnet_trained_for_cifar10.pt`를 가져옵니다.
`resnet50_bf16_cifar10` tutorial in [amd/Vitis-AI](https://github.com/amd/Vitis-AI/tree/release/6.3/versal_2ve/examples/tutorials/resnet50_bf16_cifar10)
[amd/Vitis-AI](https://github.com/amd/Vitis-AI/tree/release/6.3/versal_2ve/examples/tutorials/resnet50_bf16_cifar10)의 `resnet50_bf16_cifar10` 튜토리얼
(branch `release/6.3`).
('release/6.3' 분기).
No credentials are needed.
자격 증명이 필요하지 않습니다.
If you already have a clone
이미 클론이 있는 경우
of the repository, point at it directly and skip the download:
저장소를 직접 가리키고 다운로드를 건너뜁니다.
`python input/fetch_model.py --weights <clone>/versal_2ve/examples/tutorials/resnet50_bf16_cifar10/models/resnet_trained_for_cifar10.pt`
`python input/fetch_model.py --weights <clone>/versal_2ve/examples/tutorials/resnet50_bf16_cifar10/models/resnet_trained_for_cifar10.pt`

```bash
# 1. Full VINT8 (fast, lower accuracy) — Quark config with no exclusions:
python scripts/mixed_precision_quantize.py \
    --input models/resnet_trained_for_cifar10.onnx --output rn_vint8.onnx \
    --config input/full_vint8.quark.json --calibration-data input/calib/ifm_*.npy

# 2. VINT8+BF16 design — same config plus an `exclude` list for the BF16 head:
python scripts/mixed_precision_quantize.py \
    --input models/resnet_trained_for_cifar10.onnx --output rn_vint8_bf16.onnx \
    --config input/vint8_bf16.quark.json --calibration-data input/calib/ifm_*.npy

# 3. Accuracy and fidelity. The FP32 model is the BF16 reference; `--reference`
#    adds agreement / cosine similarity / PSNR of the logits against it:
python input/eval_top1.py --model models/resnet_trained_for_cifar10.onnx --num 10000
python input/eval_top1.py --model rn_vint8.onnx      --num 10000 --reference models/resnet_trained_for_cifar10.onnx
python input/eval_top1.py --model rn_vint8_bf16.onnx --num 10000 --reference models/resnet_trained_for_cifar10.onnx

# 4. Compile all three for the NPU (pin batch first):
python input/pin_batch1.py models/resnet_trained_for_cifar10.onnx rn_fp32_b1.onnx
python input/pin_batch1.py rn_vint8.onnx      rn_vint8_b1.onnx
python input/pin_batch1.py rn_vint8_bf16.onnx rn_vint8_bf16_b1.onnx
python scripts/compile.py rn_fp32_b1.onnx       --vitisai-config input/vitisai_config_bf16.json      --cache-dir cache_bf16ref
python scripts/compile.py rn_vint8_b1.onnx      --vitisai-config input/vitisai_config_vint8_bf16.json --cache-dir cache_int8
python scripts/compile.py rn_vint8_bf16_b1.onnx --vitisai-config input/vitisai_config_vint8_bf16.json --cache-dir cache_bf16
```

## Recorded Run
## 기록된 달리기

### 1.
### Which layers stay BF16 (sensitivity)
### BF16(감도)을 유지하는 레이어

A per-stage sweep (keep one stage in BF16, INT8 elsewhere) shows the accuracy
스테이지별 스윕(BF16, INT8에서 한 스테이지 유지)은 정확도를 보여줍니다.
loss is concentrated in the **early stages** — quantizing the raw,
손실은 **초기 단계**에 집중되어 있습니다. 즉, 원시 데이터를 양자화하고
high-dynamic-range input feature maps to INT8 is what costs top-1:
INT8에 대한 높은 동적 범위 입력 기능 매핑은 상위 1위 비용입니다.

| BF16 region (rest INT8)<br>BF16 지역(나머지 INT8) | Top-1 (500 imgs)<br>상위 1위(이미지 500개) |
|-------------------------|------------------|
| — (full VINT8)<br>— (전체 VINT8)          | 80.6% |
| `conv1 + layer1`<br>`conv1 + 레이어1`        | 82.0% |
| **`conv1 + layer1 + layer2`**<br>**`conv1 + 레이어1 + 레이어2`** | **83.0%** |

So the design keeps an **INT8 core/tail with a BF16 head** (`conv1`, `layer1`,
따라서 설계에서는 **INT8 코어/테일과 BF16 헤드**(`conv1`, `layer1`,
`layer2`) — the "Floating-Point Head with Quantized Core" pattern.
`layer2`) — "양자화된 코어가 있는 부동 소수점 헤드" 패턴.

### 2.
### Accuracy — VINT8+BF16 matches the BF16 reference
### 정확도 — VINT8+BF16은 BF16 참조와 일치합니다.

| Design<br>디자인 | Top-1 (CIFAR-10 test, 500 imgs)<br>Top-1(CIFAR-10 테스트, 500개 이미지) | Δ vs BF16 ref<br>Δ 대 BF16 참조 |
|--------|--------------------------------|----------------|
| **BF16 (reference)**<br>**BF16(참고)** | **83.0%** | — |
| Full VINT8<br>풀 VINT8 | 80.6% | −2.4 |
| **VINT8+BF16 (this design)**<br>**VINT8+BF16(본 디자인)** | **83.0%** | **0.0** |

The VINT8+BF16 design **loses nothing** versus the BF16 reference and is **+2.4
VINT8+BF16 디자인은 BF16 레퍼런스에 비해 **손실이 없으며 **+2.4입니다.
points** over full VINT8.
전체 VINT8에 대한 포인트**.

### 3.
### Compilation — all three fully offload to NPU
### 컴파일 — 세 가지 모두 NPU로 완전히 오프로드됩니다.

| Design<br>디자인 | Operators<br>연산자 | Supported by VAIML<br>VAIML에서 지원 | NPU partitions<br>NPU 파티션 | CPU partitions<br>CPU 파티션 |
|--------|-----------|--------------------|----------------|----------------|
| BF16 (reference)<br>BF16(참고) | 124 | 124 (100%) | **1** | 0 |
| Full VINT8<br>풀 VINT8 | 400 | 400 (100%) | **1** | 0 |
| VINT8+BF16 (this design)<br>VINT8+BF16(이 디자인) | 281 | 281 (100%) | **1** | 0 |

The VINT8+BF16 operation census confirms:
VINT8+BF16 운영 인구조사는 다음을 확인합니다.

```
kernel:Conv2d       (i8, i8, i16) -> (i8)   , 30   # INT8 core + tail
kernel:Conv2DBf16   (bf16,bf16,bf16)->(bf16), 24   # BF16 head (compiler-converted FP32)
kernel:AddBf16      (bf16, bf16) -> (bf16)  , 16   # residual adds in the BF16 head
kernel:DequantizeLinear (i8) -> (bf16)      , 18   # INT8 <-> BF16 island boundaries
kernel:QuantizeLinear   (bf16) -> (i8)      , 10
```

### 4.
### Board comparison — BF16 vs INT8 vs VINT8+BF16
### 보드 비교 — BF16 vs INT8 vs VINT8+BF16

All three are single-partition NPU binaries.
세 가지 모두 단일 파티션 NPU 바이너리입니다.
On a VE2 board, compare
VE2 보드에서 비교
latency and accuracy with the bundled script:
번들 스크립트를 사용한 대기 시간 및 정확성:

```bash
python /path/to/run_on_board.py \
    --board-type telluride --boardhost <telluride-host> --board-user <user> \
    -p <project-dir> -- rn_vint8_bf16_b1.onnx --cache-dir cache_bf16 --num-runs 10
```

<!-- BOARD_RESULTS: filled in from the recorded VE2 board run (bf16 vs int8 vs vint8+bf16 latency) -->
_Board latency numbers are recorded here once the run completes.
_실행이 완료되면 보드 대기 시간 수치가 여기에 기록됩니다.
Expectation:
기대:
full VINT8 is fastest, all-BF16 is slowest, and the VINT8+BF16 design sits close
전체 VINT8이 가장 빠르고, 모든 BF16이 가장 느리며, VINT8+BF16 디자인이 유사합니다.
to VINT8 (only a BF16 head) while matching BF16 accuracy._
BF16 정확도와 일치하면서 VINT8(BF16 헤드만)로._

## Expected Behavior
## 예상되는 동작

- The VINT8+BF16 design **matches the BF16 reference accuracy** and is **more
- VINT8+BF16 설계는 **BF16 기준 정확도**와 일치하며 **더욱 우수합니다.
accurate than full VINT8**.
전체 VINT8**보다 정확합니다.
- All three designs compile to a **single NPU partition (100% on AIE)**.
- 세 가지 디자인 모두 **단일 NPU 파티션(AIE에서는 100%)**으로 컴파일됩니다.
- No experimental / Extended-QDQ operators appear — the BF16 regions come
- 실험적/확장형 QDQ 연산자가 나타나지 않습니다. BF16 영역이 나타납니다.
entirely from INT8-exclusion + compiler `enable_f32_to_bf16_conversion`.
전적으로 INT8 제외 + 컴파일러 `enable_f32_to_bf16_conversion`에서 가능합니다.
