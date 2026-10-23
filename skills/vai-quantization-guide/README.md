<!-- ko-translation: enabled -->
<!--- Copyright (C) 2026 Advanced Micro Devices, Inc. All rights reserved.--->
# Mixed Precision Support
# 혼합 정밀 지원

## Table of Contents
## 목차

- [Introduction](#introduction)
- [소개](#소개)
- [Supported Patterns](#supported-patterns)
- [지원되는 패턴](#supported-patterns)
  - [Non-quantized (int8)](#non-quantized-int8)
  - [비양자화(int8)](#non-양자화-int8)
  - [Pure Quantized (int8)](#pure-quantized-int8)
  - [순수 양자화(int8)](#pure-Quantized-int8)
  - [Quantized Core with Floating-Point Tail](#quantized-core-with-floating-point-tail)
  - [부동 소수점 테일이 있는 양자화 코어](#Quantized-core-with-floating-point-tail)
  - [Floating-Point Head with Quantized Core](#floating-point-head-with-quantized-core)
  - [양자화된 코어가 있는 부동 소수점 헤드](#floating-point-head-with-Quantized-core)
  - [Floating-Point Head and Tail with Quantized Core](#floating-point-head-and-tail-with-quantized-core)
  - [양자화된 코어가 있는 부동 소수점 헤드 및 테일](#floating-point-head-and-tail-with-Quantized-core)
  - [Quantized Head and Tail with Floating-Point Middle](#quantized-head-and-tail-with-floating-point-middle)
  - [부동 소수점 중간이 있는 양자화된 머리와 꼬리](#Quantized-head-and-tail-with-floating-point-middle)
  - [Four-Island Models](#four-island-models)
  - [4섬 모델](#4섬 모델)
  - [MX6 Models](#mx6-models)
  - [MX6 모델](#mx6-모델)
  - [Extended Quantization (EQDQ) Models](#extended-quantization-eqdq-models)
  - [확장 양자화(EQDQ) 모델](#extended-양자화-eqdq-models)
- [Configuration](#configuration)
- [구성](#configuration)
  - [Compiler Flags](#compiler-flags)
  - [컴파일러 플래그](#compiler-flags)
  - [VitisAI Execution Provider Config](#vitisai-execution-provider-config)
  - [VitisAI 실행 제공자 구성](#vitisai-execution-provider-config)
- [FP16 Support (AIE2ps Only)](#fp16-support-aie2ps-only)
- [FP16 지원(AIE2ps 전용)](#fp16-support-aie2ps-only)
- [BF16 and ONNXRuntime](#bf16-and-onnxruntime)
- [BF16 및 ONNXRuntime](#bf16-and-onnxruntime)
- [Examples](#examples)
- [예](#예제)
  - [Running the Examples](#running-the-examples)
  - [예제 실행](#running-the-examples)
  - [Running on a Board](#running-on-a-board)
  - [보드 위에서 달리다](#running-on-a-board)
  - [Example Reference](#example-reference)
  - [예시 참고](#example-reference)
- [Limitations and Known Issues](#limitations-and-known-issues)
- [제한사항 및 알려진 문제](#limitations-and-known-issues)

---

## Introduction
## 소개

Mixed precision support enables VAIML to compile and execute models that combine
혼합 정밀도 지원을 통해 VAIML은 결합된 모델을 컴파일하고 실행할 수 있습니다.
different numeric data types within a single graph.
단일 그래프 내의 다양한 숫자 데이터 유형.
A typical mixed-precision
전형적인 혼합 정밀도
model contains quantized (int8) subgraphs — expressed via `QuantizeLinear` /
모델에는 'QuantizeLinear'를 통해 표현되는 양자화된(int8) 하위 그래프가 포함되어 있습니다. /
`DequantizeLinear` (QDQ) patterns — alongside floating-point subgraphs in
QDQ(DeQuantizeLinear) 패턴 — 부동 소수점 하위 그래프와 함께
`float32`, `bfloat16`, or `float16`.
`float32`, `bfloat16` 또는 `float16`.

This is useful for models where:
이는 다음과 같은 모델에 유용합니다.
- Only part of the graph is quantized (e.g. a quantized backbone with a
- 그래프의 일부만 양자화됩니다(예: 양자화된 백본
floating-point detection head).
부동 소수점 감지 헤드).
- Certain operations require higher precision for numerical accuracy.
- 특정 작업에서는 수치 정확도를 위해 더 높은 정밀도가 필요합니다.
- Different layers have been independently quantized to int8 with
- 서로 다른 레이어는 int8로 독립적으로 양자화되었습니다.
floating-point transitions between them.
그들 사이의 부동 소수점 전환.

The VAIML compiler recognizes QDQ patterns and preserves them as quantized
VAIML 컴파일러는 QDQ 패턴을 인식하고 이를 양자화된 패턴으로 유지합니다.
int8 operators on AIE, while converting the surrounding floating-point
주변 부동 소수점을 변환하는 동안 AIE의 int8 연산자
operations to the target AIE data type (typically bfloat16).
대상 AIE 데이터 유형(일반적으로 bfloat16)에 대한 작업입니다.

---

## Supported Patterns
## 지원되는 패턴

The following sections describe the precision patterns that the compiler
다음 섹션에서는 컴파일러가 제공하는 정밀도 패턴을 설명합니다.
supports, grouped by how the floating-point and quantized regions are
부동 소수점 및 양자화된 영역이 어떻게 그룹화되어 있는지 지원합니다.
arranged.
배열.
Each pattern has a corresponding ONNX example model in the
각 패턴에는 해당 ONNX 예제 모델이 있습니다.
[`models/`](/docs/10_onnx_support/mixed_precision/models) directory.
[`models/`](/docs/10_onnx_support/mixed_precision/models) 디렉터리.

### Non Quantized (int8)
### 비양자화(int8)

Operations that support the int8 tensor element type are not quantized with QDQ patterns.
int8 텐서 요소 유형을 지원하는 연산은 QDQ 패턴으로 양자화되지 않습니다.
Instead, they are represented as pure/raw int8 ops.
대신 순수/원시 int8 작업으로 표시됩니다.
All operations run as int8 on AIE.
모든 작업은 AIE에서 int8로 실행됩니다.

In the example illustrated below, op1 is a raw int8 op but op2 is a QDQ op.
아래 예시에서 op1은 원시 int8 연산이지만 op2는 QDQ 연산입니다.

```
input(int8) → op1 → DQ → op2 → Q → output(int8)
```

| Model<br>모델 | Inputs<br>입력 | Outputs<br>출력 |
|-------|--------|---------|
| [`raw_int8.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/raw_int8/raw_int8.onnxtxt)<br>[`raw_int8.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/raw_int8/raw_int8.onnxtxt) | `int8[16,32,64]`, `int8[32,16,64]`<br>`int8[16,32,64]`, `int8[32,16,64]` | `int8[16,32,64]`<br>`int8[16,32,64]` |

### Pure Quantized (int8)
### 순수 양자화(int8)

The entire graph is quantized with QDQ patterns.
전체 그래프는 QDQ 패턴으로 양자화됩니다.
All operations run as int8
모든 작업은 int8로 실행됩니다.
on AIE.
AIE에서.

```
input(int8) → DQ → op → Q → DQ → op → Q → output(int8)
```

| Model<br>모델 | Inputs<br>입력 | Outputs<br>출력 |
|-------|--------|---------|
| [`vint8.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/vint8/vint8.onnxtxt)<br>[`vint8.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/vint8/vint8.onnxtxt) | `int8[16,32,64]`, `int8[32,16,64]`<br>`int8[16,32,64]`, `int8[32,16,64]` | `int8[16,32,64]`<br>`int8[16,32,64]` |

### Quantized Core with Floating-Point Tail
### 부동 소수점 테일을 갖춘 양자화 코어

The model starts quantized and transitions to floating-point at the output.
모델은 양자화를 시작하고 출력에서 부동 소수점으로 전환됩니다.
The quantized subgraph runs as int8 on AIE; the floating-point tail runs at
양자화된 하위 그래프는 AIE에서 int8로 실행됩니다. 부동 소수점 꼬리는 다음에서 실행됩니다.
the target FP precision.
목표 FP 정밀도.

```
input(int8) → DQ → [quantized ops] → Q → DQ → [fp ops] → output(fp)
```

| Model<br>모델 | Outputs<br>출력 | Device<br>장치 | Description<br>설명 |
|-------|---------|--------|-------------|
| [`vint8_fp32_tobf16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/vint8_fp32_tobf16/vint8_fp32_tobf16.onnxtxt)<br>[`vint8_fp32_tobf16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/vint8_fp32_tobf16/vint8_fp32_tobf16.onnxtxt) | `float[16,32,128]`<br>`플로트[16,32,128]` | all<br>모두 | FP32 tail<br>FP32 테일 |
| [`vint8_fp32_tofp16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/vint8_fp32_tofp16/vint8_fp32_tofp16.onnxtxt)<br>[`vint8_fp32_tofp16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/vint8_fp32_tofp16/vint8_fp32_tofp16.onnxtxt) | `float[16,32,128]`<br>`플로트[16,32,128]` | **ve2** | FP32 tail<br>FP32 테일 |
| [`vint8_bf16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/vint8_bf16/vint8_bf16.onnxtxt)<br>[`vint8_bf16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/vint8_bf16/vint8_bf16.onnxtxt) | `bfloat16[16,32,128]`<br>`bfloat16[16,32,128]` | all<br>모두 | BF16 tail<br>BF16 꼬리 |
| [`vint8_fp16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/vint8_fp16/vint8_fp16.onnxtxt)<br>[`vint8_fp16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/vint8_fp16/vint8_fp16.onnxtxt) | `float16[16,32,128]`<br>`float16[16,32,128]` | **ve2** | FP16 tail (AIE2ps only)<br>FP16 테일(AIE2ps만 해당) |

### Floating-Point Head with Quantized Core
### 양자화된 코어가 있는 부동 소수점 헤드

The model starts with floating-point operations and transitions to quantized
모델은 부동 소수점 연산으로 시작하여 양자화 연산으로 전환됩니다.
int8 for the output.
출력의 경우 int8입니다.

```
input(fp) → [fp ops] → Q → DQ → [quantized ops] → Q → output(int8)
```

| Model<br>모델 | Inputs<br>입력 | Device<br>장치 | Description<br>설명 |
|-------|--------|--------|-------------|
| [`fp32_vint8_tobf16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/fp32_vint8_tobf16/fp32_vint8_tobf16.onnxtxt)<br>[`fp32_vint8_tobf16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/fp32_vint8_tobf16/fp32_vint8_tobf16.onnxtxt) | `float[16,32,64]`<br>`플로트[16,32,64]` | all<br>모두 | FP32 head<br>FP32 헤드 |
| [`fp32_vint8_tofp16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/fp32_vint8_tofp16/fp32_vint8_tofp16.onnxtxt)<br>[`fp32_vint8_tofp16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/fp32_vint8_tofp16/fp32_vint8_tofp16.onnxtxt) | `float[16,32,64]`<br>`플로트[16,32,64]` | **ve2** | FP32 head<br>FP32 헤드 |
| [`bf16_vint8.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/bf16_vint8/bf16_vint8.onnxtxt)<br>[`bf16_vint8.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/bf16_vint8/bf16_vint8.onnxtxt) | `bfloat16[16,32,64]`<br>`bfloat16[16,32,64]` | all<br>모두 | BF16 head<br>BF16 헤드 |
| [`fp16_vint8.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/fp16_vint8/fp16_vint8.onnxtxt)<br>[`fp16_vint8.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/fp16_vint8/fp16_vint8.onnxtxt) | `float16[16,32,64]`<br>`float16[16,32,64]` | **ve2** | FP16 head (AIE2ps only)<br>FP16 헤드(AIE2ps만 해당) |

### Floating-Point Head and Tail with Quantized Core
### 양자화된 코어가 있는 부동 소수점 헤드 및 테일

The model has floating-point regions at both ends with a quantized int8 core
모델의 양쪽 끝에는 양자화된 int8 코어가 있는 부동 소수점 영역이 있습니다.
in the middle.
중간에.

```
input(fp) → [fp ops] → Q → DQ → [quantized ops] → Q → DQ → [fp ops] → output(fp)
```

| Model<br>모델 | Inputs<br>입력 | Outputs<br>출력 | Device<br>장치 | Description<br>설명 |
|-------|--------|---------|--------|-------------|
| [`fp32_vint8_fp32_tobf16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/fp32_vint8_fp32_tobf16/fp32_vint8_fp32_tobf16.onnxtxt)<br>[`fp32_vint8_fp32_tobf16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/fp32_vint8_fp32_tobf16/fp32_vint8_fp32_tobf16.onnxtxt) | fp32 | fp32 | all<br>모두 | FP32 head + tail<br>FP32 머리 + 꼬리 |
| [`fp32_vint8_fp32_tofp16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/fp32_vint8_fp32_tofp16/fp32_vint8_fp32_tofp16.onnxtxt)<br>[`fp32_vint8_fp32_tofp16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/fp32_vint8_fp32_tofp16/fp32_vint8_fp32_tofp16.onnxtxt) | fp32 | fp32 | **ve2** | FP32 head + tail<br>FP32 머리 + 꼬리 |
| [`bf16_vint8_bf16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/bf16_vint8_bf16/bf16_vint8_bf16.onnxtxt)<br>[`bf16_vint8_bf16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/bf16_vint8_bf16/bf16_vint8_bf16.onnxtxt) | bf16 | bf16 | all<br>모두 | BF16 head + tail<br>BF16 머리 + 꼬리 |
| [`fp16_vint8_fp16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/fp16_vint8_fp16/fp16_vint8_fp16.onnxtxt)<br>[`fp16_vint8_fp16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/fp16_vint8_fp16/fp16_vint8_fp16.onnxtxt) | fp16 | fp16 | **ve2** | FP16 head + tail (AIE2ps only)<br>FP16 헤드 + 테일(AIE2ps 전용) |

### Quantized Head and Tail with Floating-Point Middle
### 부동 소수점 중간이 있는 양자화된 머리와 꼬리

The model has quantized int8 regions at the input and output with
모델은 입력 및 출력에서 int8 영역을 양자화했습니다.
floating-point operations in between.
그 사이의 부동 소수점 연산.

```
input(fp/int8) → Q → DQ → [quantized ops] → Q → DQ → [fp ops] → Q → DQ → [quantized ops] → Q → output(int8)
```

| Model<br>모델 | Middle Type<br>중간 유형 | Device<br>장치 | Description<br>설명 |
|-------|-------------|--------|-------------|
| [`vint8_fp32_vint8_tobf16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/vint8_fp32_vint8_tobf16/vint8_fp32_vint8_tobf16.onnxtxt)<br>[`vint8_fp32_vint8_tobf16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/vint8_fp32_vint8_tobf16/vint8_fp32_vint8_tobf16.onnxtxt) | fp32 | all<br>모두 | FP32 middle<br>FP32 중간 |
| [`vint8_fp32_vint8_tofp16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/vint8_fp32_vint8_tofp16/vint8_fp32_vint8_tofp16.onnxtxt)<br>[`vint8_fp32_vint8_tofp16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/vint8_fp32_vint8_tofp16/vint8_fp32_vint8_tofp16.onnxtxt) | fp32 | **ve2** | FP32 middle<br>FP32 중간 |
| [`vint8_bf16_vint8.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/vint8_bf16_vint8/vint8_bf16_vint8.onnxtxt)<br>[`vint8_bf16_vint8.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/vint8_bf16_vint8/vint8_bf16_vint8.onnxtxt) | bf16 | all<br>모두 | BF16 middle<br>BF16 중간 |
| [`vint8_fp16_vint8.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/vint8_fp16_vint8/vint8_fp16_vint8.onnxtxt)<br>[`vint8_fp16_vint8.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/vint8_fp16_vint8/vint8_fp16_vint8.onnxtxt) | fp16 | **ve2** | FP16 middle (AIE2ps only)<br>FP16 중간(AIE2ps만 해당) |

### Four-Island Models
### 4섬 모델

Models with four or more alternating quantized/floating-point regions.
4개 이상의 교대 양자화/부동 소수점 영역이 있는 모델.
These
이것들
require `match-power-of-two-quant-kernel=1` in the compiler configuration.
컴파일러 구성에 `match-power-of-two-Quant-kernel=1`이 필요합니다.

```
input → [region1] → [region2] → [region3] → [region4] → output
```

| Model<br>모델 | Device<br>장치 | Description<br>설명 |
|-------|--------|-------------|
| [`fp32_vint8_four_islands_tobf16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/fp32_vint8_four_islands_tobf16/fp32_vint8_four_islands_tobf16.onnxtxt)<br>[`fp32_vint8_four_islands_tobf16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/fp32_vint8_four_islands_tobf16/fp32_vint8_four_islands_tobf16.onnxtxt) | all<br>모두 | FP32/int8 four-island (to-bf16 conversion)<br>FP32/int8 4섬(bf16으로 변환) |
| [`fp32_vint8_four_islands_tofp16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/fp32_vint8_four_islands_tofp16/fp32_vint8_four_islands_tofp16.onnxtxt)<br>[`fp32_vint8_four_islands_tofp16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/fp32_vint8_four_islands_tofp16/fp32_vint8_four_islands_tofp16.onnxtxt) | **ve2** | FP32/int8 four-island (to-fp16 conversion)<br>FP32/int8 4섬(fp16으로 변환) |
| [`vint8_fp32_four_islands_tobf16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/vint8_fp32_four_islands_tobf16/vint8_fp32_four_islands_tobf16.onnxtxt)<br>[`vint8_fp32_four_islands_tobf16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/vint8_fp32_four_islands_tobf16/vint8_fp32_four_islands_tobf16.onnxtxt) | all<br>모두 | Int8/fp32 four-island (to-bf16 conversion)<br>Int8/fp32 4섬(bf16으로 변환) |
| [`vint8_fp32_four_islands_tofp16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/vint8_fp32_four_islands_tofp16/vint8_fp32_four_islands_tofp16.onnxtxt)<br>[`vint8_fp32_four_islands_tofp16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/vint8_fp32_four_islands_tofp16/vint8_fp32_four_islands_tofp16.onnxtxt) | **ve2** | Int8/fp32 four-island (to-fp16 conversion)<br>Int8/fp32 4섬(fp16으로 변환) |

### MX6 Models
### MX6 모델

Models with MX6 convolution and vint8 fake quantized operations or floating-point
MX6 컨볼루션 및 vint8 가짜 양자화 연산 또는 부동 소수점을 사용하는 모델
regions.
지역.
These require `match-conv-mx6=1` (enabled by default) and depending on the model `match-gemm-mx6=1`.
여기에는 `match-conv-mx6=1`(기본적으로 활성화됨)이 필요하며 모델에 따라 `match-gemm-mx6=1`이 필요합니다.
`fp16` is not supported with these models as the force-conversion floating-point type.
`fp16`은 강제 변환 부동 소수점 유형으로 이러한 모델에서 지원되지 않습니다.

| Model<br>모델 | Device<br>장치 | Description<br>설명 |
|-------|--------|-------------|
| [`mx6_vint8.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/mx6_vint8/mx6_vint8.onnxtxt)<br>[`mx6_vint8.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/mx6_vint8/mx6_vint8.onnxtxt) | **ve2** | MX6/int8 multi-island (to-bf16 conversion)<br>MX6/int8 다중 아일랜드(bf16으로 변환) |
| [`mx6_fp32_tobf16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/mx6_fp32_tobf16/mx6_fp32_tobf16.onnxtxt)<br>[`mx6_fp32_tobf16.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/mx6_fp32_tobf16/mx6_fp32_tobf16.onnxtxt) | **ve2** | MX6/FP32 multi-island (to-fp16 conversion)<br>MX6/FP32 다중 아일랜드(fp16으로 변환) |

### Extended Quantization (EQDQ) Models
### 확장 양자화(EQDQ) 모델

Models using `com.amd.quark` Extended QuantizeLinear / DequantizeLinear (EQDQ)
`com.amd.quark` 확장 QuantizeLinear / DeQuantizeLinear(EQDQ)를 사용하는 모델
operators.
연산자.
These operators represent quantization from the AMD Quark quantizer
이러한 연산자는 AMD Quark 양자화기의 양자화를 나타냅니다.
and are compiled to bf16/f16 kernels on AIE (they are NOT matched as int8
AIE에서 bf16/f16 커널로 컴파일됩니다(int8과 일치하지 않습니다).
operators).
연산자).
EQDQ models do not require `enable_f32_to_bf16_conversion` since
EQDQ 모델에는 `enable_f32_to_bf16_conversion`이 필요하지 않습니다.
the EQDQ operators already signal the target precision.
EQDQ 연산자는 이미 목표 정밀도를 나타냅니다.

```
input(fp) → EQ → EDQ → [bf16/f16 ops] → EQ → EDQ → output(fp)
```

| Model<br>모델 | Device<br>장치 | Description<br>설명 |
|-------|--------|-------------|
| [`eqdq.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/eqdq/eqdq.onnxtxt)<br>[`eqdq.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/eqdq/eqdq.onnxtxt) | all<br>모두 | Pure EQDQ — Conv + Relu both wrapped in Extended QDQ<br>순수 EQDQ — 확장 QDQ로 래핑된 Conv + Relu |
| [`vint8_eqdq.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/vint8_eqdq/vint8_eqdq.onnxtxt)<br>[`vint8_eqdq.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/vint8_eqdq/vint8_eqdq.onnxtxt) | all<br>모두 | Mixed vint8/EQDQ — 2 vint8 islands (Conv) + 2 EQDQ islands (Sigmoid, Add)<br>혼합 vint8/EQDQ — 2개의 vint8 아일랜드(Conv) + 2개의 EQDQ 아일랜드(Sigmoid, Add) |
| [`conv_vint8_eqdq.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/conv_vint8_eqdq/conv_vint8_eqdq.onnxtxt)<br>[`conv_vint8_eqdq.onnxtxt`](/docs/10_onnx_support/mixed_precision/models/conv_vint8_eqdq/conv_vint8_eqdq.onnxtxt) | all<br>모두 | QDQ Conv+Relu (vint8) followed by EQDQ Conv+Relu, output cast via Q/DQ<br>QDQ Conv+Relu(vint8) 다음에 EQDQ Conv+Relu, Q/DQ를 통해 출력 캐스팅 |

---

## Configuration
## 구성

### Compiler Flags
### 컴파일러 플래그

The following flags control mixed precision behavior in the VAIML compiler:
다음 플래그는 VAIML 컴파일러의 혼합 정밀도 동작을 제어합니다.

| Flag<br>깃발 | Description<br>설명 |
|------|-------------|
| `enable_f32_to_bf16_conversion`<br>`enable_f32_to_bf16_conversion` | Converts floating-point (fp32) operations to bfloat16 on AIE. Required for bf16 and fp32 models where the AIE executes in bfloat16.<br>AIE에서 부동 소수점(fp32) 연산을 bfloat16으로 변환합니다. AIE가 bfloat16에서 실행되는 bf16 및 fp32 모델에 필요합니다. |
| `enable_f32_to_f16_conversion`<br>`enable_f32_to_f16_conversion` | Converts floating-point (fp32) operations to float16 on AIE. Required for fp16 models on AIE2ps.<br>AIE에서 부동 소수점(fp32) 연산을 float16으로 변환합니다. AIE2ps의 fp16 모델에 필요합니다. |
| `group_args: "mixed_precision"`<br>`그룹_인수: "혼합_정밀도"` | Enables the consolidated mixed-precision flow, which includes QDQ-aware FP conversion (the compiler preserves quantized subgraphs and only converts the floating-point portions).<br>QDQ 인식 FP 변환을 포함하는 통합 혼합 정밀도 흐름을 활성화합니다(컴파일러는 양자화된 하위 그래프를 유지하고 부동 소수점 부분만 변환함). |
| `device: "ve2"`<br>`장치: "ve2"` | Targets the AIE2ps platform, which is required for FP16 support.<br>FP16 지원에 필요한 AIE2ps 플랫폼을 대상으로 합니다. |

Internally, the `group_args: "mixed_precision"` flag enables:
내부적으로 `group_args: "mixed_precision"` 플래그는 다음을 활성화합니다.
- `match-power-of-two-quant-kernel=1` — matches MLLIB QuantizeLinear kernel for a BF16/F16 -> Int8 xten_nn.quantize operator when
- `match-power-of-two-Quant-kernel=1` — 다음 경우에 BF16/F16 -> Int8 xten_nn.Quantize 연산자에 대한 MLLIB QuantizeLinear 커널과 일치합니다.
operation is a power-of-two scale and has zero zero_point.
연산은 2의 거듭제곱 규모이며 zero_point가 0입니다.

### VitisAI Execution Provider Config
### VitisAI 실행 공급자 구성

Standard configuration for mixed precision models:
혼합 정밀 모델의 표준 구성:

```json
{
  "passes": [
    {
      "name": "vaiml_partition",
      "plugin": "vaip-pass_vaiml_partition",
      "vaiml_config": {
        "keep_outputs": true,
        "optimize_level": 2,
        "enable_f32_to_bf16_conversion": true,
        "logging_level": "info",
        "group_args": "mixed_precision"
      }
    }
  ],
  "target": "VAIML",
  "targets": [
    { "name": "VAIML", "pass": ["vaiml_partition"] }
  ]
}
```

For **AIE2ps (ve2)** targets, add `"device": "ve2"` to the
**AIE2ps (ve2)** 대상의 경우 `"device": "ve2"`를 추가합니다.
`vaiml_config` section.
`vaiml_config` 섹션.
See [`vitisai_config_ve2.json`](/docs/10_onnx_support/mixed_precision/vitisai_config_ve2.json).
[`vitisai_config_ve2.json`](/docs/10_onnx_support/mixed_precision/vitsai_config_ve2.json)을 참조하세요.

---

## FP16 Support (AIE2ps Only)
## FP16 지원(AIE2ps만 해당)

Float16 (FP16) operations are natively supported only on the AIE2ps architecture.
Float16(FP16) 작업은 기본적으로 AIE2ps 아키텍처에서만 지원됩니다.
This corresponds to device `ve2` in the VAIML
이는 VAIML의 've2' 장치에 해당합니다.
configuration.
구성.

**NOTE**: All `*_tofp16` or fp16-including models require a `ve2` target.
**참고**: 모든 `*_tofp16` 또는 fp16 포함 모델에는 `ve2` 타겟이 필요합니다.

When running the test suite, FP16 tests are **automatically skipped** unless
테스트 모음을 실행할 때 FP16 테스트는 다음과 같은 경우를 제외하고 **자동으로 건너뜁니다**.
the `--device ve2` flag is passed:
`--device ve2` 플래그가 전달됩니다:

```bash
# Skip fp16 tests (default):
python scripts/run_tests.py --stage compile

# Include fp16 tests (ve2 board required):
python scripts/run_tests.py --device ve2 --stage exec
```

---

## BF16 and ONNXRuntime
## BF16 및 ONNXRuntime

**ONNXRuntime does not natively support bfloat16.** When executing mixed
**ONNXRuntime은 기본적으로 bfloat16을 지원하지 않습니다.** 혼합 실행 시
precision models through the VitisAI Execution Provider in ONNXRuntime, the
ONNXRuntime의 VitisAI 실행 공급자를 통한 정밀 모델
**fp32 variant** of the model is used in place of the bf16 variant.
모델의 **fp32 변형**은 bf16 변형 대신 사용됩니다.
The VAIML
VAIML
compiler performs the fp32-to-bf16 conversion automatically when
컴파일러는 다음과 같은 경우 자동으로 fp32에서 bf16으로의 변환을 수행합니다.
`enable_f32_to_bf16_conversion` is enabled.
'enable_f32_to_bf16_conversion'이 활성화되었습니다.

The mapping of bf16 models to their fp32 ORT substitutes:
bf16 모델을 fp32 ORT 대체품으로 매핑:

| BF16 Model<br>BF16 모델 | ORT FP32 Substitute<br>ORT FP32 대체 |
|------------|---------------------|
| `vint8_bf16.onnx`<br>`vint8_bf16.onnx` | `vint8_fp32_tobf16.onnx`<br>`vint8_fp32_tobf16.onnx` |
| `bf16_vint8.onnx`<br>`bf16_vint8.onnx` | `fp32_vint8_tobf16.onnx`<br>`fp32_vint8_tobf16.onnx` |
| `bf16_vint8_bf16.onnx`<br>`bf16_vint8_bf16.onnx` | `fp32_vint8_fp32_tobf16.onnx`<br>`fp32_vint8_fp32_tobf16.onnx` |
| `vint8_bf16_vint8.onnx`<br>`vint8_bf16_vint8.onnx` | `vint8_fp32_vint8_tobf16.onnx`<br>`vint8_fp32_vint8_tobf16.onnx` |

---

## Examples
## 예

All example models are located in the [`models/`](/docs/10_onnx_support/mixed_precision/models) directory with both
모든 예제 모델은 [`models/`](/docs/10_onnx_support/mixed_precision/models) 디렉토리에 있습니다.
`.onnx` (binary) and `.onnxtxt` (human-readable) formats.
`.onnx`(바이너리) 및 `.onnxtxt`(사람이 읽을 수 있는) 형식입니다.

### Running the Examples
### 예제 실행

#### Prerequisites
#### 전제조건

Activate the Vitis AI environment.
Vitis AI 환경을 활성화합니다.
Inside the Vitis AI Docker container it is
Vitis AI Docker 컨테이너 내부에는
already provisioned and nothing needs sourcing; otherwise source its activate
이미 프로비저닝되었으며 소싱이 필요하지 않습니다. 그렇지 않으면 활성화 소스를 제공합니다.
script:
스크립트:
```bash
source <path-to-vitis-ai-activate-script>
```

If you have a board available, also source XRT:
사용 가능한 보드가 있는 경우 XRT 소스도 제공합니다.
```bash
source /opt/xilinx/xrt/setup.sh
```

Recommended environment variables for verbose error output:
자세한 오류 출력을 위해 권장되는 환경 변수:
```bash
export DEBUG_VAIML_PARTITION=1
export FLEXML_PRINT_VITISTOOLS_OUTPUT=1
```

#### List available tests
#### 사용 가능한 테스트 나열

```bash
python scripts/run_tests.py --list
```

#### Compile only (no board required)
#### 컴파일만 가능(보드 필요 없음)

```bash
# All stx-compatible tests:
python scripts/run_tests.py --stage compile

# A specific test:
python scripts/run_tests.py --test vint8_fp32_tobf16 --stage compile

# Include ve2/fp16 tests:
python scripts/run_tests.py --device ve2 --stage compile
```

#### Compile and execute (board required)
#### 컴파일 및 실행(보드 필요)

```bash
python scripts/run_tests.py --stage exec

# With custom tolerances:
python scripts/run_tests.py --stage exec --rtol 0.02 --atol 0.02
```

### Running on a Board
### 보드 위에서 달리기

By default, use the `run_on_board.py` script provided by the `vai-custom-op-implementation` skill.
기본적으로 `vai-custom-op-implementation` 스킬에서 제공하는 `run_on_board.py` 스크립트를 사용합니다.
It handles SSH, file syncing, environment setup, custom-op registration, and board locking for Telluride targets, and can run locally or remotely.
SSH, 파일 동기화, 환경 설정, 맞춤형 작업 등록, Telluride 대상에 대한 보드 잠금을 처리하고 로컬 또는 원격으로 실행할 수 있습니다.
If the user tells you to do board runs another way, follow their instructions and use their commands.
사용자가 다른 방법으로 보드 런을 하라고 하면 지시에 따라 명령을 사용하세요.

### Example Reference
### 예시 참조

| Test Name<br>테스트 이름 | Category<br>카테고리 | Model Structure<br>모델 구조 | Device<br>장치 | What It Demonstrates<br>그것이 보여주는 것 |
|-----------|----------|-----------------|--------|----------------------|
| `raw_int8`<br>`원시_int8` | int8 only<br>int8만 | `int8 → int8`<br>`int8 → int8` | all<br>모두 | Non-quantized int8 model — supported ops in raw int8<br>양자화되지 않은 int8 모델 — 원시 int8에서 지원되는 작업 |
| `vint8`<br>'vint8' | int8 only<br>int8만 | `int8 → int8`<br>`int8 → int8` | all<br>모두 | Fully quantized baseline — all ops in QDQ patterns run as int8 on AIE<br>완전히 양자화된 기준선 — QDQ 패턴의 모든 작업은 AIE에서 int8로 실행됩니다. |
| `vint8_fp32_tobf16`<br>`vint8_fp32_tobf16` | int8 → fp<br>int8 → fp | `int8 → fp32`<br>`int8 → fp32` | all<br>모두 | Quantized core with fp32 output tail (force-converted bf16 operations)<br>fp32 출력 테일이 있는 양자화 코어(강제 변환된 bf16 작업) |
| `vint8_fp32_tofp16`<br>`vint8_fp32_tofp16` | int8 → fp<br>int8 → fp | `int8 → fp32`<br>`int8 → fp32` | **ve2** | Quantized core with fp32 output tail (force-converted fp16 operations)<br>fp32 출력 테일이 있는 양자화 코어(강제로 변환된 fp16 작업) |
| `vint8_bf16`<br>`vint8_bf16` | int8 → fp<br>int8 → fp | `int8 → bf16`<br>`int8 → bf16` | all<br>모두 | Same as above but output in bf16; ORT uses fp32 substitute<br>위와 동일하지만 bf16으로 출력됩니다. ORT는 fp32 대체품을 사용합니다. |
| `vint8_fp16`<br>`vint8_fp16` | int8 → fp<br>int8 → fp | `int8 → fp16`<br>`int8 → fp16` | **ve2** | Same as above but output in fp16; requires AIE2ps<br>위와 동일하지만 fp16으로 출력됩니다. AIE2ps가 필요합니다 |
| `fp32_vint8_tobf16`<br>`fp32_vint8_tobf16` | fp → int8<br>fp → int8 | `fp32 → int8`<br>`fp32 → int8` | all<br>모두 | FP32 input head feeding into quantized core (force-converted bf16 operations)<br>양자화된 코어로 공급되는 FP32 입력 헤드(강제 변환된 bf16 작업)|
| `fp32_vint8_tofp16`<br>`fp32_vint8_tofp16` | fp → int8<br>fp → int8 | `fp32 → int8`<br>`fp32 → int8` | **ve2** | FP32 input head feeding into quantized core (force-converted fp16 operations)<br>양자화된 코어로 공급되는 FP32 입력 헤드(강제 변환된 fp16 작업) |
| `bf16_vint8`<br>`bf16_vint8` | fp → int8<br>fp → int8 | `bf16 → int8`<br>`bf16 → int8` | all<br>모두 | BF16 input head; ORT uses fp32 substitute<br>BF16 입력 헤드; ORT는 fp32 대체품을 사용합니다. |
| `fp16_vint8`<br>`fp16_vint8` | fp → int8<br>fp → int8 | `fp16 → int8`<br>`fp16 → int8` | **ve2** | FP16 input head; requires AIE2ps<br>FP16 입력 헤드; AIE2ps가 필요합니다 |
| `fp32_vint8_fp32_tobf16`<br>`fp32_vint8_fp32_tobf16` | fp → int8 → fp<br>fp → int8 → fp | `fp32 → int8 → fp32`<br>`fp32 → int8 → fp32` | all<br>모두 | FP32 head + tail with quantized middle (multi-input model)<br>FP32 헤드 + 중앙이 양자화된 테일(다중 입력 모델) |
| `fp32_vint8_fp32_tofp16`<br>`fp32_vint8_fp32_tofp16` | fp → int8 → fp<br>fp → int8 → fp | `fp32 → int8 → fp32`<br>`fp32 → int8 → fp32` | **ve2** | FP32 head + tail with quantized middle (multi-input model)<br>FP32 헤드 + 중앙이 양자화된 테일(다중 입력 모델) |
| `bf16_vint8_bf16`<br>`bf16_vint8_bf16` | fp → int8 → fp<br>fp → int8 → fp | `bf16 → int8 → bf16`<br>`bf16 → int8 → bf16` | all<br>모두 | BF16 head + tail; ORT uses fp32 substitute<br>BF16 머리 + 꼬리; ORT는 fp32 대체품을 사용합니다. |
| `fp16_vint8_fp16`<br>`fp16_vint8_fp16` | fp → int8 → fp<br>fp → int8 → fp | `fp16 → int8 → fp16`<br>`fp16 → int8 → fp16` | **ve2** | FP16 head + tail; requires AIE2ps<br>FP16 머리 + 꼬리; AIE2ps가 필요합니다 |
| `mx6_vint8`<br>`mx6_vint8` | multi-island<br>다중 섬 | `... int8 → mx6 → int8 ...`<br>`... int8 → mx6 → int8 ...` | **ve2** | MX6 operators + int8 quantized operators<br>MX6 연산자 + int8 양자화 연산자 |
| `mx6_fp32_tobf16`<br>`mx6_fp32_tobf16` | multi-island<br>다중 섬 | `... fp32 → mx6 → fp32 ...`<br>`... fp32 → mx6 → fp32 ...` | **ve2** | MX6 operators + fp32 operators (force-converted bf16 operations)<br>MX6 연산자 + fp32 연산자(강제 변환된 bf16 작업) |
| `vint8_fp32_vint8_tobf16`<br>`vint8_fp32_vint8_tobf16` | int8 → fp → int8<br>int8 → fp → int8 | `int8 → fp32 → int8`<br>`int8 → fp32 → int8` | all<br>모두 | Quantized head and tail with fp32 middle (multi-input model)<br>fp32 중간을 갖춘 양자화된 머리와 꼬리(다중 입력 모델) |
| `vint8_fp32_vint8_tofp16`<br>`vint8_fp32_vint8_tofp16` | int8 → fp → int8<br>int8 → fp → int8 | `int8 → fp32 → int8`<br>`int8 → fp32 → int8` | **ve2** | Quantized head and tail with fp32 middle (multi-input model)<br>fp32 중간을 갖춘 양자화된 머리와 꼬리(다중 입력 모델) |
| `vint8_bf16_vint8`<br>`vint8_bf16_vint8` | int8 → fp → int8<br>int8 → fp → int8 | `int8 → bf16 → int8`<br>`int8 → bf16 → int8` | all<br>모두 | Same with bf16 middle; ORT uses fp32 substitute<br>bf16 중간과 동일합니다. ORT는 fp32 대체품을 사용합니다. |
| `vint8_fp16_vint8`<br>`vint8_fp16_vint8` | int8 → fp → int8<br>int8 → fp → int8 | `int8 → fp16 → int8`<br>`int8 → fp16 → int8` | **ve2** | Same with fp16 middle; requires AIE2ps<br>fp16 중간과 동일합니다. AIE2ps가 필요합니다 |
| `fp32_vint8_four_islands_tobf16`<br>`fp32_vint8_four_islands_tobf16` | 4 islands<br>4개의 섬 | `fp32 → int8 → fp32 → int8`<br>`fp32 → int8 → fp32 → int8` | all<br>모두 | FP32/int8 four-island (to-bf16 conversion)<br>FP32/int8 4섬(bf16으로 변환) |
| `fp32_vint8_four_islands_tofp16`<br>`fp32_vint8_four_islands_tofp16` | 4 islands<br>4개의 섬 | `fp32 → int8 → fp32 → int8`<br>`fp32 → int8 → fp32 → int8` | **ve2** | FP32/int8 four-island (to-fp16 conversion)<br>FP32/int8 4섬(fp16으로 변환) |
| `vint8_fp32_four_islands_tobf16`<br>`vint8_fp32_four_islands_tobf16` | 4 islands<br>4개의 섬 | `int8 → fp32 → int8 → fp32`<br>`int8 → fp32 → int8 → fp32` | all<br>모두 | Int8/fp32 four-island (to-bf16 conversion)<br>Int8/fp32 4섬(bf16으로 변환) |
| `vint8_fp32_four_islands_tofp16`<br>`vint8_fp32_four_islands_tofp16` | 4 islands<br>4개의 섬 | `int8 → fp32 → int8 → fp32`<br>`int8 → fp32 → int8 → fp32` | **ve2** | Int8/fp32 four-island (to-fp16 conversion)<br>Int8/fp32 4섬(fp16으로 변환) |
| `eqdq`<br>`eqdq` | eqdq only<br>eqdq만 | `fp → eqdq → fp`<br>`fp → eqdq → fp` | all<br>모두 | Pure EQDQ — Conv + Relu wrapped in Extended QDQ (bf16 kernels)<br>순수 EQDQ — 확장 QDQ(bf16 커널)로 래핑된 Conv + Relu |
| `vint8_eqdq`<br>`vint8_eqdq` | multi-island<br>다중 섬 | `int8 → eqdq → int8 → eqdq → int8`<br>`int8 → eqdq → int8 → eqdq → int8` | all<br>모두 | 2 vint8 Conv islands + 2 EQDQ islands (Sigmoid, Add)<br>2개의 vint8 Conv 아일랜드 + 2개의 EQDQ 아일랜드(Sigmoid, Add) |
| `conv_vint8_eqdq`<br>`conv_vint8_eqdq` | int8 → eqdq<br>int8 → eqdq | `int8(conv+relu) → eqdq(conv+relu)`<br>`int8(conv+relu) → eqdq(conv+relu)` | all<br>모두 | QDQ Conv+Relu (vint8) then EQDQ Conv+Relu<br>QDQ Conv+Relu(vint8) 이후 EQDQ Conv+Relu |

---

## Limitations and Known Issues
## 제한 사항 및 알려진 문제

- **BF16 not supported in ONNXRuntime**: BF16 models cannot be loaded directly
- **BF16은 ONNXRuntime에서 지원되지 않습니다**: BF16 모델은 직접 로드할 수 없습니다.
by ORT.
ORT로.
Use the fp32 variant with `enable_f32_to_bf16_conversion` instead.
대신 `enable_f32_to_bf16_conversion`과 함께 fp32 변형을 사용하세요.
- **FP16 requires AIE2ps**: Float16 operations are only supported on the
- **FP16에는 AIE2ps가 필요합니다**: Float16 작업은
`ve2` device.
've2' 장치.
Tests are skipped automatically on other platforms.
다른 플랫폼에서는 테스트를 자동으로 건너뜁니다.
- **Tolerance considerations**: Comparisons between fp32 CPU reference and
- **공차 고려사항**: fp32 CPU 레퍼런스와
bf16/fp16 AIE execution use relaxed tolerances (default: 1% relative, 0.015
bf16/fp16 AIE 실행은 완화된 허용 오차를 사용합니다(기본값: 상대 1%, 0.015).
absolute) due to reduced floating-point precision.
절대) 부동 소수점 정밀도가 감소했기 때문입니다.
