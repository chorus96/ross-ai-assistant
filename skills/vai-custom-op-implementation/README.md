<!-- ko-translation: enabled -->
<!--- Copyright (C) 2026 Advanced Micro Devices, Inc. All rights reserved.--->
# Custom Ops Flow
# 맞춤 작업 흐름

## Table of Contents
## 목차

- [Introduction](#introduction)
- [소개](#소개)
  - [Several useful resources](#several-useful-resources)
  - [여러 유용한 리소스](#several-useful-resources)
- [Using Custom Ops](#using-custom-ops)
- [커스텀 Ops 사용](#using-custom-ops)
  - [Vitis AI Config](#vitis-ai-config)
  - [Vitis AI 구성](#vitis-ai-config)
  - [Custom Op Config](#custom-op-config)
  - [사용자 정의 작업 구성](#custom-op-config)
    - [Tiling and kernel](#tiling-and-kernel)
    - [타일링 및 커널](#tiling-and-kernel)
    - [Matching an unmodified model with PDLL](#matching-an-unmodified-model-with-pdll)
    - [수정되지 않은 모델을 PDLL과 매칭](#matching-an-unmodified-model-with-pdll)
    - [Heap and stack size](#heap-and-stack-size)
    - [힙 및 스택 크기](#heap-and-stack-size)
    - [Kernel signature declaration](#kernel-signature-declaration)
    - [커널 서명 선언](#kernel-signature-declaration)
      - [Floating-point data types](#floating-point-data-types)
      - [부동 소수점 데이터 유형](#floating-point-data-types)
    - [Data layout specification](#data-layout-specification)
    - [데이터 레이아웃 사양](#data-layout-사양)
      - [Automatic padding for vectorization](#automatic-padding-for-vectorization)
      - [벡터화를 위한 자동 패딩](#automatic-padding-for-Vectorization)
      - [Reducing layout discrepancies](#reducing-layout-discrepancies)
      - [레이아웃 불일치 줄이기](#reducing-layout-discrepancies)
  - [Using Custom Ops with ONNX Runtime](#using-custom-ops-with-onnx-runtime)
  - [ONNX 런타임과 함께 Custom Ops 사용](#using-custom-ops-with-onnx-runtime)
- [Developing Custom Ops](#developing-custom-ops)
- [Custom Ops 개발](#developing-custom-ops)
  - [Subgraph extraction](#subgraph-extraction)
  - [하위 그래프 추출](#subgraph-extraction)
  - [Custom Op Development](#custom-op-development)
  - [커스텀 Op 개발](#custom-op-development)
    - [Debugging Tiling](#debugging-tiling)
    - [디버깅 타일링](#debugging-tiling)
    - [Board](#board)
    - [보드](#보드)
  - [Kernel Code](#kernel-code)
  - [커널 코드](#kernel-code)
- [Appendix](#appendix)
- [부록](#appendix)
  - [CustomOp execution and debugging flags](#customop-execution-and-debugging-flags)
  - [Customop 실행 및 디버깅 플래그](#customop-execution-and-debugging-flags)
  - [ADF header](#adf-header)
  - [ADF 헤더](#adf-header)
  - [Determining the stack and heap size](#determining-the-stack-and-heap-size)
  - [스택 및 힙 크기 결정](#determining-the-stack-and-heap-size)

---

## Introduction
## 소개

The Custom Ops Flow is a methodology for extending VAIML to support user-defined operations.
Custom Ops Flow는 사용자 정의 작업을 지원하기 위해 VAIML을 확장하는 방법론입니다.
Unlike the traditional flow, which relies on a fixed set of pre-built kernels, the Custom Ops Flow gives you the flexibility to define your own kernel, tailored to your specific application needs and performance goals.
고정된 사전 구축 커널 세트에 의존하는 기존 흐름과 달리 Custom Ops Flow는 특정 애플리케이션 요구 사항과 성능 목표에 맞게 맞춤화된 자체 커널을 정의할 수 있는 유연성을 제공합니다.

This approach is designed for users with some experience in AIE hardware and programming.
이 접근 방식은 AIE 하드웨어 및 프로그래밍에 대한 경험이 있는 사용자를 위해 설계되었습니다.
It provides you with more control over the logic and integration of your operator, while still maintaining a layer of abstraction from the hardware.
이는 하드웨어의 추상화 계층을 유지하면서 운영자의 논리 및 통합에 대한 더 많은 제어를 제공합니다.
This lets you focus on your operator’s functionality, while ensuring compatibility with the VAIML framework and overlay constraints.
이를 통해 VAIML 프레임워크 및 오버레이 제약 조건과의 호환성을 보장하면서 운영자의 기능에 집중할 수 있습니다.

Custom Ops Flow is especially useful when:
맞춤 작업 흐름은 다음과 같은 경우에 특히 유용합니다.
- You need an ONNX operation that VAIML does not natively support.
- VAIML이 기본적으로 지원하지 않는 ONNX 작업이 필요합니다.
- Your application has performance requirements that call for a more specialized implementation than what VAIML currently supports.
- 귀하의 애플리케이션에는 현재 VAIML이 지원하는 것보다 더 전문적인 구현을 요구하는 성능 요구 사항이 있습니다.

To implement a custom operator, you’ll need several key components:
사용자 정의 연산자를 구현하려면 몇 가지 주요 구성 요소가 필요합니다.

[**Custom Op Config**](#custom-op-config)
[**사용자 정의 작업 구성**](#custom-op-config)
  > Configuration file that specifies, e.g., the paths to all files that are required to make a custom op usable within VAIML.
  > 예를 들어 VAIML 내에서 사용자 정의 작업을 사용할 수 있도록 만드는 데 필요한 모든 파일의 경로를 지정하는 구성 파일입니다.
  > This file acts as the central reference for your custom op and ensures all resources are available during compilation.
  > 이 파일은 사용자 정의 작업에 대한 중앙 참조 역할을 하며 컴파일 중에 모든 리소스를 사용할 수 있도록 보장합니다.

[**AIE Kernel Implementation**](#kernel-code)
[**AIE 커널 구현**](#kernel-code)
  > Defines the computation for your operator.
  > 연산자에 대한 계산을 정의합니다.
  > This kernel runs on the AIE hardware and should be designed to efficiently process your input tensors.
  > 이 커널은 AIE 하드웨어에서 실행되며 입력 텐서를 효율적으로 처리하도록 설계되어야 합니다.

[**Tiling**](tiling.md)
[**타일링**](tiling.md)
  > Describes how your tensors are partitioned and mapped onto the AIE hardware.
  > 텐서를 분할하고 AIE 하드웨어에 매핑하는 방법을 설명합니다.
  > The tiler defines the data movement and tiling strategy, which is important for achieving good performance.
  > 타일러는 우수한 성능을 달성하는 데 중요한 데이터 이동 및 타일링 전략을 정의합니다.

Each component plays a specific role in the compilation and execution process.
각 구성 요소는 컴파일 및 실행 프로세스에서 특정 역할을 수행합니다.
The config ties everything together, the tiler manages data movement, the kernel performs the computation, and the header handles integration with the AIE overlay and memory.
구성은 모든 것을 하나로 묶고, 타일러는 데이터 이동을 관리하고, 커널은 계산을 수행하고, 헤더는 AIE 오버레이 및 메모리와의 통합을 처리합니다.
Further details and guidance is provided for each step/component below.
각 단계/구성 요소에 대한 자세한 내용과 지침은 아래에 제공됩니다.

---

### Several useful resources:
### 몇 가지 유용한 리소스:

- [Custom Op Usage Examples](tutorial/README.md)
- [사용자 정의 Op 사용 예](tutorial/README.md)
- [RyzenAI Docs](https://ryzenai.docs.amd.com/en/latest/)
- [RyzenAI 문서](https://ryzenai.docs.amd.com/en/latest/)
- [AI Engine Documentation Landing page](https://docs.amd.com/p/ug1720-ai-engine-landing)
- [AI 엔진 문서 랜딩 페이지](https://docs.amd.com/p/ug1720-ai-engine-landing)
- [AIE-ML Architecture user guide](https://docs.amd.com/r/en-US/am020-versal-aie-ml/AIE-ML-Array-Overview)
- [AIE-ML 아키텍처 사용자 가이드](https://docs.amd.com/r/en-US/am020-versal-aie-ml/AIE-ML-Array-Overview)
- [AIE Kernel & Graph programming user guide](https://docs.amd.com/r/en-US/ug1079-ai-engine-kernel-coding/Overview?tocId=_gsmjbSrB9YCl4WkKcFCww)
- [AIE 커널 및 그래프 프로그래밍 사용자 가이드](https://docs.amd.com/r/en-US/ug1079-ai-engine-kernel-coding/Overview?tocId=_gsmjbSrB9YCl4WkKcFCww)
- [Vitis AIE Development Tutorials](https://github.com/Xilinx/Vitis-Tutorials/tree/2025.1/AI_Engine_Development/AIE-ML)
- [Vitis AIE 개발 튜토리얼](https://github.com/Xilinx/Vitis-Tutorials/tree/2025.1/AI_Engine_Development/AIE-ML)

## Using Custom Ops
## 커스텀 작업 사용

This section describes how to configure and use custom ops within the VAIML framework.
이 섹션에서는 VAIML 프레임워크 내에서 사용자 정의 작업을 구성하고 사용하는 방법을 설명합니다.

### Vitis AI Config
### 바이티스 AI 구성

Onnx models are compiled for AIE by using onnxruntime with the [VitisAIExecutionProvider](https://ryzenai.docs.amd.com/en/latest/modelrun.html).
Onnx 모델은 [VitisAIExecutionProvider](https://ryzenai.docs.amd.com/en/latest/modelrun.html)와 함께 onnxruntime을 사용하여 AIE용으로 컴파일됩니다.
The Vitis AI Config provides settings for the VitisAIExecutionProvider.
Vitis AI Config는 VitisAIExecutionProvider에 대한 설정을 제공합니다.
One part of this is the configuration for VAIML, which includes the Custom Ops flow.
그 중 한 부분은 Custom Ops 흐름을 포함하는 VAIML 구성입니다.

The following section describes the custom ops:
다음 섹션에서는 사용자 지정 작업에 대해 설명합니다.
```json
"custom_ops": {
    "<custom_op_name>": {
      "op_config": "<custom_op_config>"
    }
}
```

The first part, `<custom_op_name>`, is the name of the custom op as it appears in your onnx model.
첫 번째 부분 `<custom_op_name>`은 onnx 모델에 나타나는 사용자 정의 작업의 이름입니다.

> Note: Parts in `<>` need to be replaced by actual values.
> 참고: `<>` 부분은 실제 값으로 교체해야 합니다.

You can have multiple custom ops; add one entry per op you want to register.
여러 사용자 정의 작업을 가질 수 있습니다. 등록하려는 작업당 하나의 항목을 추가하세요.
Find all details on the `<custom_op_config>` in the [Custom Op Config Section](#custom-op-config).
[Custom Op Config 섹션](#custom-op-config)에서 `<custom_op_config>`에 대한 모든 세부정보를 찾아보세요.

A full Vitis AI Config (`vitisai_config.json`) including the custom ops section looks like this:
사용자 정의 작업 섹션을 포함하는 전체 Vitis AI 구성(`vitsai_config.json`)은 다음과 같습니다.
```json
{
  "passes": [
    {
      "name": "vaiml_partition2",
      "plugin": "vaip-pass_vaiml_partition",
      "vaiml_config": {
        "keep_outputs": true,
        "optimize_level": 2,
        "enable_f32_to_bf16_conversion": true,
        "logging_level": "info",
        "fe_args": "small-tensor-threshold-unwrapping=0",
        "experiment_features": [
          "KeepOrphanNodes"
        ],
        "custom_ops": {
          "mydomain.myop": {
            "op_config": "my_custom_op.yaml"
          }
        }
      }
    }
  ],
  "target": "VAIML",
  "targets": [
    {
      "name": "VAIML",
      "pass": [
        "vaiml_partition2"
      ]
    }
  ]
}
```

Find the full documentation on the Vitis AI Execution Provider Options [here](https://ryzenai.docs.amd.com/en/latest/modelrun.html#vitisai-ep-provider-options).
Vitis AI 실행 공급자 옵션에 대한 전체 문서는 [여기](https://ryzenai.docs.amd.com/en/latest/modelrun.html#vitisai-ep-provider-options)에서 찾아보세요.

### Custom Op Config
### 맞춤형 작업 구성

The custom op configuration contains references to all custom operator implementation-related files and kernel properties.
사용자 정의 작업 구성에는 모든 사용자 정의 연산자 구현 관련 파일 및 커널 속성에 대한 참조가 포함되어 있습니다.

#### Tiling and kernel
#### 타일링과 커널

The configuration, i.e., the `<custom_op_config>` referenced in the [Vitis AI Config](#vitis-ai-config), has the following mandatory content:
[Vitis AI Config](#vitis-ai-config)에서 참조하는 구성, 즉 `<custom_op_config>`에는 다음과 같은 필수 내용이 있습니다.
```yaml
tiling: <my_tiler.py>
kernel: <my_adf_wrapper.cpp>
```

The `tiling` entry points to the python code that describes how to tile the inputs.
'tiling' 항목은 입력을 타일링하는 방법을 설명하는 Python 코드를 가리킵니다.
The [tiling guide](tiling.md) explains how to create this file.
[타일링 가이드](tiling.md)에 이 파일을 만드는 방법이 설명되어 있습니다.

The `kernel` entry points to the C++ file that implements the functionality of the custom op ([Kernel Code](#kernel-code)).
`커널` 항목은 사용자 정의 작업([커널 코드](#kernel-code))의 기능을 구현하는 C++ 파일을 가리킵니다.

#### Matching an unmodified model with PDLL
#### 수정되지 않은 모델을 PDLL과 일치

By default, the input model must already contain your custom op, which means editing the ONNX file to replace the original operation with, e.g., `mydomain.myop`.
기본적으로 입력 모델에는 사용자 정의 작업이 이미 포함되어 있어야 합니다. 즉, ONNX 파일을 편집하여 원래 작업을 `mydomain.myop`으로 대체하는 것을 의미합니다.

Alternatively, you can let the compiler create the custom op for you by matching a pattern in the *unmodified* model.
또는 *수정되지 않은* 모델의 패턴을 일치시켜 컴파일러가 사용자 정의 작업을 생성하도록 할 수 있습니다.
Add a `match_on` entry pointing to a [PDLL](https://mlir.llvm.org/docs/PDLL/) pattern file:
[PDLL](https://mlir.llvm.org/docs/PDLL/) 패턴 파일을 가리키는 `match_on` 항목을 추가합니다.
```yaml
match_on:
  original_model: <my_pattern.pdll>
```

The pattern is applied to the original model, before ONNX shape inference.
패턴은 ONNX 모양 추론 전에 원본 모델에 적용됩니다.
Use `input_shape_inferred:` instead of `original_model:` to run it *after* shape inference, so the matched op (and hence the created custom op) carries statically-ranked types.
모양 추론 *이후* 실행하려면 `original_model:` 대신 `input_shape_inferred:`를 사용하세요. 그러면 일치하는 작업(따라서 생성된 사용자 지정 작업)이 정적으로 순위가 지정된 유형을 전달합니다.
This matters when the op you match has no `value_info` in the model and would otherwise be unranked.
이는 일치하는 작업의 모델에 `value_info`가 없고 그렇지 않으면 순위가 지정되지 않을 때 중요합니다.

A pattern that rewrites a stock `SpaceToDepth` into `mydomain.myspacetodepth` looks like this:
스톡 `SpaceToDepth`를 `mydomain.myspacetolength`로 다시 작성하는 패턴은 다음과 같습니다.
```
#include "shared/custom_ops.pdll"

Pattern match_spacetodepth with benefit(0) {
    let s2d = op<onnx.SpaceToDepth>(input: Value<inType: Type>) {blocksize = attr<"2 : si64">} -> (outType: Type);
    isTopLevel(s2d);
    replace s2d with createCustomOp(s2d, (input), "mydomain.myspacetodepth");
}
```

The `isTopLevel` constraint and the `createCustomOp` rewrite come from the compiler-provided include, which is resolved out of `<install-dir>/data/Transform/pdll`.
`isTopLevel` 제약 조건과 `createCustomOp` 재작성은 `<install-dir>/data/Transform/pdll`에서 해결되는 컴파일러 제공 포함에서 나옵니다.
The op name passed to `createCustomOp` must match the key used in [the Vitis AI configuration file](#vitis-ai-config).
`createCustomOp`에 전달된 작업 이름은 [Vitis AI 구성 파일](#vitis-ai-config)에 사용된 키와 일치해야 합니다.

Constraining the match (here, `blocksize = 2`) keeps the rewrite limited to what your kernel actually implements; anything that does not match is left alone and falls back to the regular VAIML lowering.
일치를 제한하면(여기서는 `blocksize = 2`) 커널이 실제로 구현하는 것만으로 재작성이 제한됩니다. 일치하지 않는 모든 항목은 그대로 남아 있으며 일반 VAIML 낮추기로 돌아갑니다.

Because the model keeps its original, standard ONNX operation, it also stays runnable outside VAIML, and no custom-op schema registration is needed in ONNX Runtime (see [Using Custom Ops with ONNX Runtime](#using-custom-ops-with-onnx-runtime)).
모델은 원래의 표준 ONNX 작업을 유지하므로 VAIML 외부에서도 실행 가능한 상태로 유지되며 ONNX Runtime에서는 custom-op 스키마 등록이 필요하지 않습니다([ONNX Runtime에서 Custom Ops 사용](#using-custom-ops-with-onnx-runtime) 참조).
The [spacetodepth tutorial](/docs/30_custom_ops/tutorial/spacetodepth) is a worked example.
[spaceto깊이 튜토리얼](/docs/30_custom_ops/tutorial/spaceto깊이)은 실제 예제입니다.

#### Heap and stack size
#### 힙 및 스택 크기

You can specify the stack and heap size requirements of your kernel.
커널의 스택 및 힙 크기 요구 사항을 지정할 수 있습니다.

```yaml
stack_size: <num>
heap_size: <num>
```
The total stack and heap size for the whole model must not exceed 16 KB and is influenced by the stack and heap size requested by each operator.
전체 모델의 총 스택 및 힙 크기는 16KB를 초과할 수 없으며 각 연산자가 요청하는 스택 및 힙 크기의 영향을 받습니다.
As a rule of thumb, the sum of `stack_size` and `heap_size` should be below `12288` to leave enough margin.
경험상 충분한 여백을 남기려면 `stack_size`와 `heap_size`의 합이 `12288` 미만이어야 합니다.
It is safe to set these values above the actual kernel requirement, but this limits the amount of memory that tilers will accept to allocate to L1 buffers.
이러한 값을 실제 커널 요구 사항보다 높게 설정하는 것이 안전하지만 이로 인해 타일러가 L1 버퍼에 할당할 수 있는 메모리 양이 제한됩니다.
You can find out how much stack and heap your kernel actually requires, using the guidance from [the appendix](#determining-the-stack-and-heap-size).
[부록](#determining-the-stack-and-heap-size)의 지침을 사용하여 커널에 실제로 필요한 스택과 힙의 양을 확인할 수 있습니다.
If left unspecified, VAIML uses the default stack and heap size.
지정하지 않으면 VAIML은 기본 스택 및 힙 크기를 사용합니다.

#### Kernel signature declaration
#### 커널 서명 선언

Elements from the signature of your kernel need to be given as a `signature` specification as shown below.
커널 서명의 요소는 아래와 같이 `서명` 사양으로 제공되어야 합니다.
Note that if the custom op has two inputs, a second `async` entry must be provided under `inputs`.
사용자 정의 작업에 두 개의 입력이 있는 경우 두 번째 'async' 항목이 'inputs' 아래에 제공되어야 합니다.
```yaml
signature:
  name: <my_kernel>
  inputs:
    - async: [false|true]
  outputs:
    - async: [false|true]
```

The name of the kernel function in the kernel definition ([Kernel Code](#kernel-code)) is specified by `name`.
커널 정의([Kernel Code](#kernel-code))의 커널 함수 이름은 `name`으로 지정됩니다.

If one entry in `inputs` and one in `outputs` is used, the corresponding C++ signature is:
'inputs' 항목 하나와 'outputs' 항목 하나가 사용되는 경우 해당 C++ 서명은 다음과 같습니다.
```cpp
template <typename dtype_ifm, typename dtype_ofm>
__attribute__((noinline)) void
<my_kernel>(adf::input_buffer_conf<dtype_ifm, bpc_(a)?sync_0d> &__restrict ifm,
            adf::output_buffer_conf<dtype_ofm, bpc_(a)?sync_0d> &__restrict ofm,
            const uint32_t (&lp_params)[negate_kernel_lp_size])
```

For each input and output, the `async` property specifies the synchronization option used for the corresponding buffer port, which is `bpc_sync_0d` if `false` and `bpc_async_0d` if `true`.
각 입력 및 출력에 대해 'async' 속성은 해당 버퍼 포트에 사용되는 동기화 옵션을 지정합니다. 이는 'false'인 경우 'bpc_sync_0d'이고 'true'인 경우 'bpc_async_0d'입니다.

> **NOTE:** For advanced use cases, the user might manually write ADF code.
> **참고:** 고급 사용 사례의 경우 사용자가 수동으로 ADF 코드를 작성할 수 있습니다.
> This is not recommended.
> 이는 권장되지 않습니다.
> The path to the ADF header can be specified using the `ml_adf_header` config key, see [ADF Header](#adf-header) in the [Appendix](#appendix).
> ADF 헤더 경로는 `ml_adf_header` 구성 키를 사용하여 지정할 수 있습니다. [부록](#appendix)의 [ADF 헤더](#adf-header)를 참조하세요.

##### Floating-point data types
##### 부동 소수점 데이터 유형

VAIML does not support 32 bit floating point (`float`) operations; by default, it converts the tensors to the natively supported `bfloat16` type instead.
VAIML은 32비트 부동 소수점('float') 연산을 지원하지 않습니다. 기본적으로 텐서는 기본적으로 지원되는 `bfloat16` 유형으로 대신 변환됩니다.

If the input to your ONNX custom op is of `float` type, then `bfloat16` data is passed to the kernel.
ONNX 사용자 정의 작업에 대한 입력이 `float` 유형인 경우 `bfloat16` 데이터가 커널에 전달됩니다.

#### Data layout specification
#### 데이터 레이아웃 사양

The kernel is allowed to require a different data layout in DDR than the one specified in the ONNX operation.
커널은 ONNX 작업에 지정된 것과 다른 DDR의 데이터 레이아웃을 요구할 수 있습니다.
The DDR data layout can differ from the ONNX one in the following ways:
DDR 데이터 레이아웃은 다음과 같은 점에서 ONNX와 다를 수 있습니다.
  * a transposition: swapping two or more tensor dimensions,
  * 전치: 두 개 이상의 텐서 차원을 교환합니다.
  * a "vectorization dimension": splitting up a dimension into blocks of a specified size, and moving the dimension traversing the inside of the blocks as the innermost.
  * "벡터화 차원": 차원을 지정된 크기의 블록으로 분할하고 블록 내부를 가장 안쪽으로 이동하는 차원을 이동합니다.

To communicate this requirement to the compiler, you can extend the kernel signature section to include the data layouts as it appears in the ONNX model and the layout as it is expected by the kernel implementation.
이 요구 사항을 컴파일러에 전달하기 위해 커널 서명 섹션을 확장하여 ONNX 모델에 나타나는 데이터 레이아웃과 커널 구현에서 예상되는 레이아웃을 포함할 수 있습니다.
This is achieved using the `onnx_tensor_layout` and `ddr_tensor_layout` attributes on each input and/or output.
이는 각 입력 및/또는 출력의 `onnx_tensor_layout` 및 `ddr_tensor_layout` 속성을 사용하여 달성됩니다.

The ONNX tensor layout (`onnx_tensor_layout`) must be a sequence of capital letters (`A` to `Z`), optionally separated by spaces.
ONNX 텐서 레이아웃(`onnx_tensor_layout`)은 대문자(`A`~`Z`)의 시퀀스여야 하며 선택적으로 공백으로 구분됩니다.

The DDR tensor layout (`ddr_tensor_layout`) must be a permutation of the ONNX tensor layout, and optionally:
DDR 텐서 레이아웃(`ddr_tensor_layout`)은 ONNX 텐서 레이아웃의 순열이어야 하며 선택적으로 다음을 수행해야 합니다.
* one of the dimensions may be followed by `/` and an integer, which is the vectorization factor; if vectorization is applied, the vectorization factor must:
* 차원 중 하나 뒤에 `/`와 벡터화 인자인 정수가 올 수 있습니다. 벡터화가 적용되는 경우 벡터화 요소는 다음과 같아야 합니다.
  * be a power of two, and
  * 2의 거듭제곱이 되고,
  * be repeated at the end of the layout (materializing the innermost dimension).
  * 레이아웃 끝에서 반복됩니다(가장 안쪽 차원을 구체화).
* spaces are admissible to separate dimensions.
* 공간은 별도의 차원으로 허용됩니다.

`ddr_tensor_layout` must match the following regular expression:
`ddr_tensor_layout`은 다음 정규 표현식과 일치해야 합니다:
```
[A-Z ]*([A-Z]/([0-9]+))?[A-Z ]+([0-9]*)
```

Any transposition (permutation of the ONNX data layout in the DDR data layout) is valid.
모든 전치(DDR 데이터 레이아웃의 ONNX 데이터 레이아웃 순열)가 유효합니다.
Only one of the dimensions may be "vectorized".
차원 중 하나만 "벡터화"할 수 있습니다.
The size of this dimension in the ONNX model must be a multiple of the vectorization factor.
ONNX 모델에서 이 차원의 크기는 벡터화 인자의 배수여야 합니다.

Both the DDR or ONNX tensor layout must be specified, or none.
DDR 또는 ONNX 텐서 레이아웃을 모두 지정하거나 지정하지 않아야 합니다.
Absent a specification, it is assumed that the kernel's data layout matches the ONNX layout.
사양이 없으면 커널의 데이터 레이아웃이 ONNX 레이아웃과 일치하는 것으로 간주됩니다.

Example:
예:

```yaml
signature:
  name: my_kernel
  inputs:
  - async: false
    onnx_tensor_layout: "N C H W"
    ddr_tensor_layout: "H C N/8 W 8"
  outputs:
  - async: false
    onnx_tensor_layout: "N C H W"
    ddr_tensor_layout: "H C N/8 W 8"
```

In the above example, the dimension `N` is split into blocks of size 8.
위의 예에서 'N' 차원은 크기가 8인 블록으로 분할됩니다.

##### Automatic padding for vectorization
##### 벡터화를 위한 자동 패딩

Using the vectorization feature requires the shape of the tensor, on the vectorized dimension, to be a multiple of the vectorization factor.
벡터화 기능을 사용하려면 벡터화 차원에서 텐서의 모양이 벡터화 인자의 배수가 되어야 합니다.
ONNX tensors will automatically be padded to satisfy this requirement.
ONNX 텐서는 이 요구 사항을 충족하기 위해 자동으로 채워집니다.
In the above example, if the ONNX tensor has a shape of `[12, a, b, c]`, it will be padded to `[16, a, b, c]` so that the dimension `N` can be vectorized with a factor of 8.
위의 예에서 ONNX 텐서의 모양이 `[12, a, b, c]`인 경우 `[16, a, b, c]`에 패딩되어 차원 `N`이 8배로 벡터화될 수 있습니다.

##### Reducing layout discrepancies
##### 레이아웃 불일치 줄이기

Ops before and after a custom op may have different L3 layout requirements than the custom op itself.
사용자 정의 작업 전후의 작업은 사용자 정의 작업 자체와 L3 레이아웃 요구 사항이 다를 수 있습니다.
In that case, VAIML automatically inserts data movement operations to resolve the layout discrepancies between ops.
이 경우 VAIML은 데이터 이동 작업을 자동으로 삽입하여 작업 간의 레이아웃 불일치를 해결합니다.
If this is found to be the case, remarks will be emitted with the layout of the producer or consumer op, and that of the custom op.
이러한 경우가 발견되면 생산자 또는 소비자 작업의 레이아웃과 사용자 지정 작업의 레이아웃과 함께 설명이 내보내집니다.
For instance, you may encounter this remark for discrepancies with the producer op:
예를 들어 생산자 작업과의 불일치에 대해 다음과 같은 설명이 나타날 수 있습니다.
```
INFO: [VAIML-COMPILE 1000] loc(callsite(unknown at "CompilerGeneratedLoc")): Custom op MyUnary (mydomain.myop) operand 0 uses a different memory layout than its producer, possibly causing suboptimal data movement. This op: ONNX tensor layout = "A B C D", DDR Layout = "C B A/8 D 8"; producer op: ONNX layout = "A B C", DDR layout = "A B C".
```
or that one for discrepancies with the consumer:
또는 소비자와의 불일치에 대한 것:
```
INFO: [VAIML-COMPILE 1000] loc(callsite(unknown at "CompilerGeneratedLoc")): Custom op MyUnary (mydomain.myop) result 0 uses a different memory layout than its consumer, possibly causing suboptimal data movement. This op: ONNX tensor layout = "A B C D", DDR Layout = "C B A/8 D 8"; consumer op: ONNX layout = "A B C D", DDR layout = "A B C D".
```

If possible, you may change your kernel's data layout requirements.
가능하다면 커널의 데이터 레이아웃 요구 사항을 변경할 수 있습니다.
Doing so may improve the overall performance of your design by reducing the amount of data movement between ops.
이렇게 하면 작업 간 데이터 이동량이 줄어들어 디자인의 전반적인 성능이 향상될 수 있습니다.

### Using Custom Ops with ONNX Runtime
### ONNX 런타임과 함께 Custom Ops 사용

Assume you have a small model like the one shown below as `onnxtxt` (with a corresponding `<model.onnx>`):
아래와 같은 'onnxtxt'(해당하는 `<model.onnx>` 포함)와 같은 작은 모델이 있다고 가정합니다.
```
<
    ir_version: 7,
    opset_import: ["" : 13, "mydomain" : 1]
>
test(float[2, 64] in) => (float[2, 64] result) {
    result = mydomain.myop<data_type = "float32", shape = [2, 64]>(in)
}
```

> Note: The onnxtxt format is not supported as input format, we use it here for readability purposes.
> 참고: onnxtxt 형식은 입력 형식으로 지원되지 않으며 여기서는 가독성을 위해 사용합니다.
> If you would like to convert an onnxtxt to onnx (or the other way around) you can use [this conversion script](/scripts/utils/convertonnx.py).
> onnxtxt를 onnx로(또는 그 반대) 변환하려면 [이 변환 스크립트](/scripts/utils/convertonnx.py)를 사용할 수 있습니다.
> Be aware that the format does have its limitations right now though, especially for `bfloat16` not every model can be parsed properly by the onnx python package.
> 하지만 현재 형식에는 제한이 있다는 점에 유의하세요. 특히 `bfloat16`의 경우 onnx Python 패키지에서 모든 모델을 올바르게 구문 분석할 수 있는 것은 아닙니다.

This model contains the custom operation `mydomain.myop`, which we want to run on AIE.
이 모델에는 AIE에서 실행하려는 사용자 지정 작업 `mydomain.myop`이 포함되어 있습니다.
Make sure that the model contains the data type and shape for all custom ops.
모델에 모든 사용자 정의 작업에 대한 데이터 유형과 형태가 포함되어 있는지 확인하십시오.
VAIML only supports static shapes.
VAIML은 정적 모양만 지원합니다.

The name given to the ONNX op (here `mydomain.myop`) must match:
ONNX 작업(여기서는 `mydomain.myop`)에 지정된 이름은 다음과 일치해야 합니다.
* the name and domain given to ONNX Runtime for registration (here the domain is `mydomain` and the name is `myop`),
* 등록을 위해 ONNX Runtime에 지정된 이름과 도메인(여기서 도메인은 'mydomain'이고 이름은 'myop'입니다)
* the op name given in [the Vitis AI configuration file](#vitis-ai-config),
* [Vitis AI 구성 파일](#vitis-ai-config)에 지정된 작업 이름,
* the file name of the kernel configuration YAML file (if provided), for instance `mydomain.myop.yaml`.
* 커널 구성 YAML 파일(제공된 경우)의 파일 이름입니다(예: `mydomain.myop.yaml`).


The following skeleton shows how you can start from python to run your ONNX model including custom ops with ONNX runtime and the `VitisAIExecutionProvider`:
다음 뼈대에서는 Python에서 시작하여 ONNX 런타임 및 `VitisAIExecutionProvider`를 사용하는 사용자 지정 작업을 포함하여 ONNX 모델을 실행하는 방법을 보여줍니다.
```python
import sys
import numpy as np
import onnxruntime as ort
from onnxruntime_custom_ops import (
    register_dynamic_custom_ops_to_onnxruntime,
    vaiml_custom_op_schema,
)


def main() -> int:
    # Register the custom ops
    register_dynamic_custom_ops_to_onnxruntime(
        [vaiml_custom_op_schema(domain="mydomain", name="myop", nb_inputs=1, nb_outputs=1)]
    )

    print("Creating onnx session using VitisAIExecutionProvider...")
    onnx_session = ort.InferenceSession(
        "<model.onnx>",
        providers=["VitisAIExecutionProvider"],
        provider_options=[
            {
                "config_file": "vitisai_config.json",
            }
        ],
    )

    # Generate some random inputs for your model (choose a shape and type consistent with your model).
    my_input = np.random.rand(2, 64).astype(np.float32)

    # Use the generated input to run your model (adjust `in` to your model input name).
    outputs = onnx_session.run(None, {"in": my_input})
    print(f"Model outputs:\n{outputs}")

    return 0


if __name__ == "__main__":
    sys.exit(main())
```

## Developing Custom Ops
## 맞춤형 운영 개발

The CustomOp development methodology which can be divided into 3 stages:
3단계로 나눌 수 있는 CustomOp 개발 방법론:

- [Subgraph extraction](#subgraph-extraction)
- [하위 그래프 추출](#subgraph-extraction)
- [Custom Op Development](#custom-op-development)
- [커스텀 Op 개발](#custom-op-development)
- Verification
- 검증


![Development Diagram](resources/development_diagram.svg)

It is very important to start developing custom ops by extracting the subgraph with the ONNX operations ([Operation Extraction](#subgraph-extraction)) that we want to convert to custom ops.
커스텀 연산으로 변환하려는 ONNX 연산([Operation Extraction](#subgraph-extraction))으로 하위 그래프를 추출하여 커스텀 연산 개발을 시작하는 것이 매우 중요합니다.
Reducing the size of the model will make verifying the custom op faster while helping with debugging.
모델 크기를 줄이면 디버깅에 도움이 되는 동시에 사용자 정의 작업을 더 빠르게 확인할 수 있습니다.

The extracted subgraph (Model A) serves as a golden reference that we can use to verify the custom op(s) against Model B.
추출된 하위 그래프(모델 A)는 모델 B에 대해 사용자 지정 작업을 확인하는 데 사용할 수 있는 주요 참조 역할을 합니다.

After creating both Model A and B, the actual custom op development can start.
모델 A와 B를 모두 생성한 후 실제 맞춤형 작업 개발을 시작할 수 있습니다.
There is tooling (see [here](/scripts)) that automates the process of both compiling and running models on the board.
보드에서 모델을 컴파일하고 실행하는 프로세스를 자동화하는 도구([여기](/scripts) 참조)가 있습니다.

The last step missing after having tested the custom op standalone is to integrate it into the final model.
맞춤형 작업 독립 실행형을 테스트한 후 누락된 마지막 단계는 이를 최종 모델에 통합하는 것입니다.
That last stage is beyond the scope of this document.
마지막 단계는 이 문서의 범위를 벗어납니다.


### Subgraph extraction
### 하위 그래프 추출

To create Model A - the subgraph containing the original operations that will be converted to a custom op - you can use the `onnx-cut` tool.
사용자 정의 작업으로 변환될 원래 작업을 포함하는 하위 그래프인 모델 A를 생성하려면 `onnx-cut` 도구를 사용할 수 있습니다.

```<
    ir_version: 7,
    opset_import: ["" : 13]
>
test(float[1, 4] A, float[1, 4] B, float[1, 4] C, float[1, 4] D) => (float[1, 4] result) {
    mul_out = Mul(A, B)
    add_out = Add(C, D)
    result = Sub(mul_out, add_out)
}
```

Using the model described above, one can extract the `Sub` operation into a subgraph one would with `./scripts/onnx_cut.py --model ./mul_add_sub.onnx  --input-name mul_out add_out --output-names result --output model_a.onnx`.
위에서 설명한 모델을 사용하면 `./scripts/onnx_cut.py --model ./mul_add_sub.onnx --input-name mul_out add_out --output-names result --output model_a.onnx`를 사용하여 `Sub` 작업을 하위 그래프로 추출할 수 있습니다.
Running this command results in:
이 명령을 실행하면 다음과 같은 결과가 나타납니다.

```
<
   ir_version: 7,
   opset_import: ["" : 13],
   producer_name: "onnx.utils.extract_model"
>
Extracted from {test} (float[1,4] mul_out, float[1,4] add_out) => (float[1,4] result)
   <float[1,4] mul_out, float[1,4] add_out>
{
   result = Sub (mul_out, add_out)
}
```

### Custom Op Development
### 맞춤형 운영 개발

This part of the guide is only deals with the development methodology.
가이드의 이 부분은 개발 방법론만 다룹니다.
To understand how to develop a kernel, several guides have been made available:
커널 개발 방법을 이해하기 위해 다음과 같은 몇 가지 가이드가 제공됩니다.
   - Check [kernel development](kernel_development.md) to learn how to write optimized kernels.
   - 최적화된 커널을 작성하는 방법을 알아보려면 [커널 개발](kernel_development.md)을 확인하세요.
   - Check [this](tiling.md) guide to learn how to writing tiling.
   - 타일링 작성 방법을 알아보려면 [this](tiling.md) 가이드를 확인하세요.

Before starting to develop a custom op from scratch, it generally makes sense to start by modifying one of the existing examples.
처음부터 사용자 정의 작업 개발을 시작하기 전에 일반적으로 기존 예제 중 하나를 수정하여 시작하는 것이 좋습니다.
They can be found [here](tutorial/README.md).
[여기](tutorial/README.md)에서 찾을 수 있습니다.
The tutorial examples try to be broad enough to capture the different types of ops like element-wise ops or reductions.
튜토리얼 예제는 요소별 작업 또는 감소와 같은 다양한 유형의 작업을 포착할 수 있을 만큼 충분히 광범위합니다.

The first thing to do when developing a custom op is to check for functional correctness.
사용자 정의 작업을 개발할 때 가장 먼저 해야 할 일은 기능의 정확성을 확인하는 것입니다.
It is very important to use x86 simulation for this purpose, because the simulator makes it possible to perform printf debug, allowing to debug any problem with tiling or numeric mismatches.
이 목적을 위해 x86 시뮬레이션을 사용하는 것이 매우 중요합니다. 시뮬레이터를 사용하면 printf 디버그를 수행하여 타일링이나 숫자 불일치와 관련된 모든 문제를 디버깅할 수 있기 때문입니다.

A normal compile already emits the artifacts required to run the simulator alongside the hardware artifacts, so one cache serves both. x86 simulation is then selected at run time by handing `run.py` a copy of the vitisai config that adds `"runner_type": "x86sim"` to its `vaiml_config`:
일반 컴파일은 이미 하드웨어 아티팩트와 함께 시뮬레이터를 실행하는 데 필요한 아티팩트를 생성하므로 하나의 캐시가 두 가지 모두를 제공합니다. x86 시뮬레이션은 `vaiml_config`에 `"runner_type": "x86sim"`을 추가하는 vitisai 구성 복사본을 `run.py`에 전달하여 런타임에 선택됩니다.

```
# Source the venv
source <ryzen_ai_dir>/lnx64/bin/activate

# Compile the model with custom ops (model b)
python scripts/compile.py CustomReduceMax_axis_2.onnx \
    --vitisai-config vitisai_config.json

# Run the x86 simulation against that same cache, using the config copy
# that sets "runner_type": "x86sim"
python scripts/run.py CustomReduceMax_axis_2.onnx \
    --vitisai-config vitisai_config_x86sim.json \
    --input-dir reference_data --output-dir x86sim_outputs --num-runs 1
```

Do not compile with the x86sim config: `runner_type` is stored in the cache, and a later board run that simply omits the key will fall back to the stored `x86sim` and fail.
x86sim 구성으로 컴파일하지 마십시오. `runner_type`은 캐시에 저장되며 단순히 키를 생략하는 이후 보드 실행은 저장된 `x86sim`으로 대체되어 실패합니다.


#### Debugging Tiling
#### 디버깅 타일링

To analyze a tiling strategy for a custom op, the best way is to go to its compilation directory and search for the header file that contains the tiling implementation for the custom op (there is only one per custom op).
사용자 정의 작업에 대한 타일링 전략을 분석하는 가장 좋은 방법은 해당 컴파일 디렉터리로 이동하여 사용자 정의 작업에 대한 타일링 구현이 포함된 헤더 파일을 검색하는 것입니다(사용자 정의 작업당 하나만 있음).

Example:
예:

When compiling a custom op with name `mycustomop`, its tiling could be found under `<compilation_directory>/vaiml_par_0/0/backend/gen_tg_adf/_mycustomop_res.h`.
이름이 `mycustomop`인 사용자 정의 작업을 컴파일할 때 해당 타일링은 `<compilation_directory>/vaiml_par_0/0/backend/gen_tg_adf/_mycustomop_res.h`에서 찾을 수 있습니다.

Documentation on this header file is structured is available [online](https://docs.amd.com/r/en-US/ug1603-ai-engine-ml-kernel-graph/Tiling-Parameters-Specification).
구조화된 이 헤더 파일에 대한 문서는 [온라인](https://docs.amd.com/r/en-US/ug1603-ai-engine-ml-kernel-graph/Tiling-Parameters-Specification)에서 확인할 수 있습니다.

#### Board
#### 보드

The last stage of custom op development is to test on the board to make sure there aren't any numeric mismatches.
맞춤형 작전 개발의 마지막 단계는 보드에서 테스트하여 숫자 불일치가 없는지 확인하는 것입니다.

### Kernel Code
### 커널 코드

The kernel code is what will be executed on the AIE cores, and hence is responsible to do the computation of the operation.
커널 코드는 AIE 코어에서 실행되므로 작업 계산을 담당합니다.
Each AIE core executes the same kernel code.
각 AIE 코어는 동일한 커널 코드를 실행합니다.
Layer Parameters (LPs, often also called Run-Time Parameters or RTPs) can be used to communicate information to the kernel.
계층 매개변수(LP, 런타임 매개변수 또는 RTP라고도 함)는 커널에 정보를 전달하는 데 사용될 수 있습니다.
LPs are filled with values that are defined in the [tiling script](tiling.md#kernel-number-of-calls-and-parameters).
LP는 [타일링 스크립트](tiling.md#kernel-number-of-calls-and-parameters)에 정의된 값으로 채워집니다.

A typical kernel signature looks like this:
일반적인 커널 서명은 다음과 같습니다.
```C++
template <typename dtype_input, typename dtype_output>
__attribute__((noinline)) void my_kernel(
    adf::input_buffer_conf<dtype_input, bpc_sync_0d> &__restrict in,
    adf::output_buffer_conf<dtype_output, bpc_sync_0d> &__restrict out,
    const uint32_t (&lp_params)[<lp_size>]);
```
The kernel receives buffers to inputs and output, and additionally the LPs.
커널은 입력 및 출력에 대한 버퍼와 추가로 LP를 수신합니다.

The `<lp_size>` needs to be replaced by the number of LPs that you have provided in your tiler.
`<lp_size>`는 타일러에 제공한 LP 수로 대체되어야 합니다.

For simple cases, a kernel will receive a tile of the input(s), and compute its outputs based on its inputs and RTPs.
간단한 경우 커널은 입력 타일을 수신하고 입력 및 RTP를 기반으로 출력을 계산합니다.

For more details, see the [kernel documentation](kernel_development.md).
자세한 내용은 [커널 문서](kernel_development.md)를 참조하세요.

## Appendix
## 부록

### CustomOp execution and debugging flags
### CustomOp 실행 및 디버깅 플래그

```json
"logging_level": "info"
```
This enables the debug output from FlexML (where the custom ops are processed).
이를 통해 FlexML(사용자 지정 작업이 처리되는 곳)의 디버그 출력이 활성화됩니다.
Useful for debugging.
디버깅에 유용합니다.

```json
"keep_outputs": true
```
Makes sure FlexML keeps all intermediate output files.
FlexML이 모든 중간 출력 파일을 유지하는지 확인합니다.
Useful for debugging.
디버깅에 유용합니다.

```json
"fe_args": "small-tensor-threshold-unwrapping=0"
```
There is a heuristic that operations on very small tensors are executed on CPU.
매우 작은 텐서에 대한 작업이 CPU에서 실행되는 경험적 방법이 있습니다.
If you have small tensor sizes, set the threshold for this to zero to run those operations on AIE instead.
텐서 크기가 작은 경우 임계값을 0으로 설정하여 대신 AIE에서 해당 작업을 실행하세요.

```json
"optimize_level": 2
```
Controls the level of optimization applied by the compiler.
컴파일러가 적용하는 최적화 수준을 제어합니다.
It might try different things like different overlays, increasing the number of connection from the AIE to DDR.
다양한 오버레이와 같은 다양한 작업을 시도하여 AIE에서 DDR로의 연결 수를 늘릴 수 있습니다.
Changing this flag has wide repercussions and might cause failures.
이 플래그를 변경하면 광범위한 영향을 미치며 오류가 발생할 수 있습니다.
This should not be modified.
이는 수정하면 안 됩니다.

```json
"tp_size": 1
```
A *stamp* is the basic replicable block of an overlay: one full grid of AIE cores together with its memtile connections, and it is the unit that a single tiling object describes.
*스탬프*는 오버레이의 복제 가능한 기본 블록입니다. AIE 코어의 전체 그리드와 멤타일 연결이 결합된 하나의 전체 그리드이며 단일 타일링 객체가 설명하는 단위입니다.
`tp_size: N` provisions the overlay with N stamps, i.e., N times the compute of one stamp.
`tp_size: N`은 N개의 스탬프로 오버레이를 프로비저닝합니다. 즉, 하나의 스탬프 계산의 N배입니다.
If not given, assume 1.
주어지지 않으면 1로 가정합니다.
The number of cores per stamp is a property of the overlay, see the [tiling guide](tiling.md#aie-topology-and-vaiml-overlay).
스탬프당 코어 수는 오버레이의 속성입니다. [타일링 가이드](tiling.md#aie-topology-and-vaiml-overlay)를 참조하세요.


### ADF header
### ADF 헤더

The ADF header is a header file where the data movement and scheduling of AIE kernels is done.
ADF 헤더는 AIE 커널의 데이터 이동 및 스케줄링이 수행되는 헤더 파일입니다.
This is the file used to integrate your operator with the overall AIE graph.
이는 연산자를 전체 AIE 그래프와 통합하는 데 사용되는 파일입니다.
When you provide a tiling script, this file is generated for you and contains the data movement instructions the tiling script requested.
타일링 스크립트를 제공하면 이 파일이 생성되고 타일링 스크립트가 요청한 데이터 이동 지침이 포함됩니다.

You can provide your own manually-written ADF header.
직접 작성한 ADF 헤더를 제공할 수 있습니다.
To do so, its name needs to be supplied in the custom op config's optional `ml_adf_header` field, alongside the kernel YAML:
그렇게 하려면 커널 YAML과 함께 사용자 정의 op 구성의 선택적 `ml_adf_header` 필드에 해당 이름을 제공해야 합니다.
```yaml
ml_adf_header: <my_header.h>
kernel_yaml_dir: <dir_to_mydomain.mycustomop.yaml>
```

Providing ADF header yourself is possible if:
다음과 같은 경우 ADF 헤더를 직접 제공할 수 있습니다.
* you already have a custom-made ADF overlay header, or
* 이미 맞춤형 ADF 오버레이 헤더가 있거나
* you need to modify the automatically-generated ADF header.
* 자동으로 생성된 ADF 헤더를 수정해야 합니다.
* Doing so stops the auto-generation of ADF headers for the given custom op.
* 이렇게 하면 해당 사용자 정의 작업에 대한 ADF 헤더 자동 생성이 중지됩니다.
In this case, you can start from the automatically generated header.
이 경우 자동으로 생성된 헤더에서 시작할 수 있습니다.
Assuming that `mydomain.mycustomop` is the name of the custom op in ONNX, you can find the generated ADF header under `custom_ops` inside your results/artifact folder, with the name `mydomain_mycustomop_0_multi_layer_overlay.h`.
`mydomain.mycustomop`이 ONNX의 사용자 정의 작업 이름이라고 가정하면 결과/아티팩트 폴더 내 `custom_ops`에서 `mydomain_mycustomop_0_multi_layer_overlay.h`라는 이름으로 생성된 ADF 헤더를 찾을 수 있습니다.
The generated kernel YAML is located next to it in the same directory.
생성된 커널 YAML은 동일한 디렉터리의 옆에 있습니다.

### Determining the stack and heap size
### 스택 및 힙 크기 결정

You can find out how much stack and heap are actually used by your kernel to fine-tune these values.
이러한 값을 미세 조정하기 위해 커널이 실제로 얼마나 많은 스택과 힙을 사용하는지 확인할 수 있습니다.
For that, you need to access some of the compilation artifacts.
이를 위해서는 일부 컴파일 아티팩트에 액세스해야 합니다.
In order to do so, you should set the VAIP cache directory to a working directory instead of the default temporary:
이렇게 하려면 VAIP 캐시 디렉터리를 기본 임시 디렉터리 대신 작업 디렉터리로 설정해야 합니다.

```py
    onnx_session = ort.InferenceSession(
        "<model.onnx>",
        providers=["VitisAIExecutionProvider"],
        provider_options=[
            {
                "config_file": "vitisai_config.json",
                "cache_dir": "/path/to/cache",
                "cacheKey": "model",
            }
        ],
    )
```

Once compilation is complete, the compilation artifacts in `/path/to/cache/model` will contain information on the stack and heap size.
컴파일이 완료되면 `/path/to/cache/model`의 컴파일 아티팩트에 스택 및 힙 크기에 대한 정보가 포함됩니다.

* Stack: the file `/path/to/cache/model/vaiml_par_0/0/aiecompiler/Work/aie/0_0/Release/0_0.calltree` contains a table like the following:
* 스택: `/path/to/cache/model/vaiml_par_0/0/aiecompiler/Work/aie/0_0/Release/0_0.calltree` 파일에는 다음과 같은 테이블이 포함되어 있습니다.
```
Call tree stack and functions sizes:

stack stack stack call  func  func  function name
      desc  level level       desc
----- ----- ----- ----- ----- ----- --------------------------------------------------------------
    0   320     0     0    32  2080 __start
   64   320     1     1   112  2048     _main_init
  256   256     2     2  1536  1936         main
    0     0     3     3   192   192             _ZN3adf11block_writeEPKNS_7reg_valEj
    0     0     3     3   208   208             _ZN10custom_ops10mul_kernel... // Your function name, mangled

Maximum call level : 3
Maximum stack level: 3
Maximum stack size : 320
```
where the stack used by your kernel and the total maximum size (to be used in the YAML) can be found.
커널에서 사용하는 스택과 YAML에서 사용되는 총 최대 크기를 찾을 수 있습니다.

* Heap: The file `/path/to/cache/model/vaiml_par_0/0/aiecompiler/Work/aie/0_0/Release/0_0.map` contains a table like the following:
* 힙: `/path/to/cache/model/vaiml_par_0/0/aiecompiler/Work/aie/0_0/Release/0_0.map` 파일에는 다음과 같은 테이블이 포함되어 있습니다.
```
     VMA      LMA     Size Align Out     In      Symbol
       0        0        0     1 . = 0x0
       0        0      820    16 .text
       0        0       20    16         :(.text)
       0        0       20     1                 __start
      20       20       70    16         :(.text._main_init)
      20       20        0     1                 .Lfunc_begin1
      20       20       70     1                 _main_init
  // Look for the following line, that mentions your C++ function (starts by its mangled name)
      90       90       d0    16         Work/aie/pm_reload_analysis0/Release/pm_reload_analysis0.o:(.text._ZN10custom_ops10mul_kernel...)
      90       90        0     1                 .Lfunc_begin1
      90       90       d0     1                 void custom_ops::mul_kernel... // Name of your kernel
      e0       e0        0     1                 .LBB1_2
     140      140        0     1                 .L_LEnd0
     150      150        0     1                 .LBB1_3
     160      160      600    16         Work/aie/0_0/Release/0_0.o:(.text.main)
     [...]
     760      760       c0    16         Work/aie/0_0/Release/0_0.o:(.text._ZN3adf11block_writeEPKNS_7reg_valEj)
     [...]
// Objects in the .bss section (e.g. static or global variables) will be shown here.
   7b500    7b500      480     4 .bss.lcpPing
     [...]
```
Here the heap size needed by the kernel is `0xd0` (= 208 bytes) + 0 bytes from the .bss section; the total heap size to give the YAML is the sum of the `.text` and `.bss` sections, here `0x820` + `0x480` = `0xCA0` = 3232 bytes.
여기서 커널에 필요한 힙 크기는 `0xd0`(= 208바이트) + .bss 섹션의 0바이트입니다. YAML에 제공되는 총 힙 크기는 `.text` 및 `.bss` 섹션의 합입니다. 여기서는 `0x820` + `0x480` = `0xCA0` = 3232바이트입니다.
