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

Quick-start prompt for the **mixed-precision** skill (`/vai-quantization-guide`):
**혼합 정밀도** 기술에 대한 빠른 시작 프롬프트(`/vai-Quantization-guide`):
**quantize** a ResNet50 straight to a mixed **VINT8+BF16** design and show it
ResNet50을 혼합 **VINT8+BF16** 디자인으로 바로 **양자화**하고 보여줍니다.
**matches the accuracy of the all-BF16 design** while being faster, and is
**모든 BF16 설계의 정확성**과 일치하면서도 더 빠릅니다.
**clearly more accurate than the full-VINT8 design**.
**전체 VINT8 디자인보다 확실히 더 정확합니다**.

The all-BF16 model — from the Vitis-AI
Vitis-AI의 모든 BF16 모델
[`resnet50_bf16_cifar10` tutorial](https://github.com/amd/Vitis-AI/blob/release/6.3/versal_2ve/examples/tutorials/resnet50_bf16_cifar10)
[`resnet50_bf16_cifar10` 튜토리얼](https://github.com/amd/Vitis-AI/blob/release/6.3/versal_2ve/examples/tutorials/resnet50_bf16_cifar10)
— is the **accuracy reference** (a BF16 model is numerically close to the
— **정확도 기준**입니다(BF16 모델은 수치상으로
non-quantized FP32 model).
양자화되지 않은 FP32 모델).

## The prompt
## 프롬프트

```
Quantize the ResNet50 model to a mixed VINT8+BF16 design with the mixed-precision
skill, starting from the FP32/BF16 model. The all-BF16 design from the
Vitis-AI resnet50_bf16_cifar10 tutorial is the accuracy reference.
Goal: match the BF16 design's top-1 accuracy while running faster than all-BF16,
and be clearly more accurate than the full-VINT8 design.

Quantize to INT8 but keep the accuracy-critical layers in floating point (excluded
from INT8) so the VAIML compiler converts those islands to BF16 via
enable_f32_to_bf16_conversion. Ensure that the design is fully offloaded, that is, it achieves a single NPU partition.

/vai-quantization-guide --model input/models/resnet_trained_for_cifar10.onnx \
    --device ve2 --target-accuracy "top1>=bf16-1%" \
    --calibration-data input/calib/ifm_0.npy input/calib/ifm_1.npy input/calib/ifm_2.npy
```

After quantization, compile all three designs (BF16 reference, full VINT8, and
양자화 후 세 가지 디자인(BF16 참조, 전체 VINT8 및
VINT8+BF16) for the NPU, confirm full offload, and run on the board to compare
VINT8+BF16) NPU의 경우 전체 오프로드를 확인하고 보드에서 실행하여 비교합니다.
latency and accuracy.
대기 시간과 정확성.

## Notes
## 메모

- **Model**: a ResNet50 fine-tuned on CIFAR-10 (input `1x3x32x32`, 10 classes) —
- **모델**: CIFAR-10에서 미세 조정된 ResNet50('1x3x32x32' 입력, 10개 클래스) —
the model shipped with the Vitis-AI `resnet50_bf16_cifar10` tutorial.
Vitis-AI 'resnet50_bf16_cifar10' 튜토리얼과 함께 제공되는 모델입니다.
`input/fetch_model.py` downloads its weights and exports the FP32 ONNX.
`input/fetch_model.py`는 가중치를 다운로드하고 FP32 ONNX를 내보냅니다.
- **Reference (BF16)**: the FP32 ONNX compiled with
- **참조(BF16)**: 다음으로 컴파일된 FP32 ONNX
`enable_f32_to_bf16_conversion` (no Quark) — this *is* the
`enable_f32_to_bf16_conversion`(Quark 없음) — 이것은 *이*
`resnet50_bf16_cifar10` design.
'resnet50_bf16_cifar10' 디자인.
Its accuracy equals the FP32 model's.
정확도는 FP32 모델과 동일합니다.
- **Device**: `ve2`.
- **장치**: `ve2`.
- **Accuracy metric**: **top-1** on the CIFAR-10 test set.
- **정확도 지표**: CIFAR-10 테스트 세트의 **상위 1**입니다.
