<!-- ko-translation: enabled -->
<!--- Copyright (C) 2026 Advanced Micro Devices, Inc. All rights reserved.--->
# Tutorials and examples
# 튜토리얼 및 예시

This directory contains examples and tutorials to demonstrate various aspects of the custom ops
이 디렉터리에는 사용자 지정 작업의 다양한 측면을 보여주는 예제와 튜토리얼이 포함되어 있습니다.
flow, and serve as a basis to implement your own custom ops.
흐름을 제공하고 자신만의 맞춤 작업을 구현하기 위한 기반으로 사용됩니다.

| Example<br>예             | Description<br>설명                                                                                           |
|---------------------|-------------------------------------------------------------------------------------------------------|
| [negate/0_untiled](/docs/30_custom_ops/tutorial/negate/0_untiled)<br>[부정/0_untiled](/docs/30_custom_ops/tutorial/negate/0_untiled)  | Simplest way to implement a unary negate operation. Runs on a single AIE core and only supports tensors that fit in L1.<br>단항 부정 연산을 구현하는 가장 간단한 방법입니다. 단일 AIE 코어에서 실행되며 L1에 맞는 텐서만 지원합니다.                                                  |
| [negate/1_tiled](/docs/30_custom_ops/tutorial/negate/1_tiled)<br>[negate/1_tiled](/docs/30_custom_ops/tutorial/negate/1_tiled)    | Expands on the previous example by using tiling to support tensor sizes that don't fit in L1 memory.<br>L1 메모리에 맞지 않는 텐서 크기를 지원하기 위해 타일링을 사용하여 이전 예제를 확장합니다. |
| [negate/2_distributed](/docs/30_custom_ops/tutorial/negate/2_distributed)<br>[negate/2_distributed](/docs/30_custom_ops/tutorial/negate/2_distributed) | Extends tiling to make full use of the 4x4 AIE overlay, distributing the negate operation across all 16 cores.<br>타일링을 확장하여 4x4 AIE 오버레이를 최대한 활용하고 16개 코어 전체에 부정 작업을 분산시킵니다.    |
| [negate/3_distributed_and_vectorized](/docs/30_custom_ops/tutorial/negate/3_distributed_and_vectorized)<br>[negate/3_distributed_and_Vectorized](/docs/30_custom_ops/tutorial/negate/3_distributed_and_Vectorized)    | Expands on the previous example by optimizing the kernel using pipelining, aliasing information and vectorization.<br>파이프라이닝, 앨리어싱 정보 및 벡터화를 사용하여 커널을 최적화하여 이전 예를 확장합니다. |
| [negate/4_distributed_vectorized_2stamp](/docs/30_custom_ops/tutorial/negate/4_distributed_vectorized_2stamp)<br>[negate/4_distributed_Vectorized_2stamp](/docs/30_custom_ops/tutorial/negate/4_distributed_Vectorized_2stamp)    | Expands on the previous example by increasing parallelism by using 2 stamps.<br>2개의 스탬프를 사용하여 병렬성을 높여 이전 예를 확장합니다. |
| [negate/5_distributed_vectorized_6stamp](/docs/30_custom_ops/tutorial/negate/5_distributed_vectorized_6stamp)<br>[negate/5_distributed_Vectorized_6stamp](/docs/30_custom_ops/tutorial/negate/5_distributed_Vectorized_6stamp)    | Expands on the previous example by increasing parallelism with 6 stamps.<br>6개의 스탬프로 병렬성을 늘려 이전 예를 확장합니다. |
| [negate/6_split_channel](/docs/30_custom_ops/tutorial/negate/6_split_channel)<br>[negate/6_split_channel](/docs/30_custom_ops/tutorial/negate/6_split_channel)    | Reads the input from DDR in two halves, each over a distinct shim DMA channel.<br>각각 고유한 심 DMA 채널을 통해 DDR의 입력을 두 부분으로 읽습니다. |
| [negate/7_reuse_4channel](/docs/30_custom_ops/tutorial/negate/7_reuse_4channel)<br>[negate/7_reuse_4channel](/docs/30_custom_ops/tutorial/negate/7_reuse_4channel)    | Reads the input from DDR in four parts, each over a distinct shim DMA channel.<br>각각 고유한 심 DMA 채널을 통해 DDR의 입력을 네 부분으로 읽습니다. |
| [mul/0_untiled](/docs/30_custom_ops/tutorial/mul/0_untiled)<br>[mul/0_untiled](/docs/30_custom_ops/tutorial/mul/0_untiled)    | Simple binary operation: computes the element-wise product of two vectors. Runs on a single AIE core and only supports tensors that fit in L1.<br>단순 이진 연산: 두 벡터의 요소별 곱을 계산합니다. 단일 AIE 코어에서 실행되며 L1에 맞는 텐서만 지원합니다. |
| [mul/1_tiled](/docs/30_custom_ops/tutorial/mul/1_tiled)<br>[mul/1_tiled](/docs/30_custom_ops/tutorial/mul/1_tiled)    | Expands on the previous example by using tiling to support tensor sizes that don't fit in L1 memory. Runs on a single AIE core.<br>L1 메모리에 맞지 않는 텐서 크기를 지원하기 위해 타일링을 사용하여 이전 예제를 확장합니다. 단일 AIE 코어에서 실행됩니다. |
| [mul/2_distributed](/docs/30_custom_ops/tutorial/mul/2_distributed)<br>[mul/2_distributed](/docs/30_custom_ops/tutorial/mul/2_distributed)    | Expands on the previous example by parallelizing ("distributing") the computations across all AIE cores.<br>모든 AIE 코어에 걸쳐 계산을 병렬화("분산")하여 이전 예를 확장합니다. |
| [mul/3_distributed_2stamp](/docs/30_custom_ops/tutorial/mul/3_distributed_2stamp)<br>[mul/3_distributed_2stamp](/docs/30_custom_ops/tutorial/mul/3_distributed_2stamp)    | Expand on the previous example by parallelizing across both AIE cores and 2 stamps.<br>AIE 코어와 2개의 스탬프를 모두 병렬화하여 이전 예를 확장합니다. |
| [mul/4_distributed_6stamp](/docs/30_custom_ops/tutorial/mul/4_distributed_6stamp)<br>[mul/4_distributed_6stamp](/docs/30_custom_ops/tutorial/mul/4_distributed_6stamp)    | Expand on the previous example by parallelizing across 6 stamps<br>6개의 스탬프를 병렬화하여 이전 예를 확장합니다. |
| [mul/5_performant_overlay](/docs/30_custom_ops/tutorial/mul/5_performant_overlay)<br>[mul/5_performant_overlay](/docs/30_custom_ops/tutorial/mul/5_performant_overlay)     | Demonstrates how to tile the `2_distributed` example on the "performant" overlay<br>'성능' 오버레이에 '2_distributed' 예시를 타일링하는 방법을 보여줍니다. |
| [conv1d/0_distributed](/docs/30_custom_ops/tutorial/conv1d/0_distributed)<br>[conv1d/0_distributed](/docs/30_custom_ops/tutorial/conv1d/0_distributed)    | 1-D single-channel convolution distributed over the full 4x4 AIE overlay, with column-broadcast IFM, row-broadcast WTS, and an asynchronous (long-lived) WTS buffer. Worked example for the [tiling guide](/docs/30_custom_ops/tiling.md).<br>열 브로드캐스트 IFM, 행 브로드캐스트 WTS 및 비동기(장기 지속) WTS 버퍼를 사용하여 전체 4x4 AIE 오버레이에 분산된 1D 단일 채널 컨볼루션입니다. [타일링 가이드](/docs/30_custom_ops/tiling.md)에 대한 작업된 예입니다. |
| [gemm](/docs/30_custom_ops/tutorial/gemm)<br>[gemm](/docs/30_custom_ops/tutorial/gemm)    | **GEMM + bias** on the 8x8x8 `aie::mmul` -- the matmul / linear / pointwise-conv archetype (a 1x1 conv *is* a GEMM: out-ch x in-ch x spatial, no halo, no sliding window). Demonstrates pre-blocking both operands into 8x8 mmul tiles, **per-row weight distribution** (`SpatialDistribute`), **bias folded into the weights** (one extra reduction tile, no separate bias DMA), multi-memtile L3->L2 staging with a wide DMA burst, and **weight streaming** from L2 when weights don't fit in L1.<br>8x8x8 `aie::mmul`의 **GEMM + 바이어스** -- matmul / 선형 / 점별 변환 원형(1x1 변환 *은* GEMM: out-ch x in-ch x 공간, 후광 없음, 슬라이딩 윈도우 없음). 두 피연산자를 8x8 mmul 타일로 사전 차단하는 방법, **행별 가중치 분포**(`SpatialDistribute`), **가중치에 접힌 바이어스**(하나의 추가 감소 타일, 별도의 바이어스 DMA 없음), 넓은 DMA 버스트를 사용한 다중 멤타일 L3->L2 스테이징, 가중치가 L1에 맞지 않을 때 L2에서 **가중치 스트리밍**을 보여줍니다. |
| [conv2d/conv3x3_int8_2stamp](/docs/30_custom_ops/tutorial/conv2d/conv3x3_int8_2stamp)<br>[conv2d/conv3x3_int8_2stamp](/docs/30_custom_ops/tutorial/conv2d/conv3x3_int8_2stamp)    | **Spatial 3x3 stride-1 INT8 convolution** on the 4x8x8 `aie::mmul` (`int8 x int8 -> int32`). The full-conv regime: **halo rows** + an in-register **sliding window** (`shuffle_down_fill`), a **multi-call partial-sum reduction over input-channel depth** (OFM async ratio), and **per-output-channel int8 requantization** with a folded pre-scaled int32 bias. Reproduces a native VAIML conv layer bit-exactly.<br>**공간 3x3 스트라이드-1 INT8 컨볼루션** 4x8x8 `aie::mmul`(`int8 x int8 -> int32`). 전체 변환 체제: **후광 행** + 레지스터 내 **슬라이딩 윈도우**(`shuffle_down_fill`), **입력 채널 깊이에 대한 다중 호출 부분 합 감소**(OFM 비동기 비율), 접힌 사전 스케일링된 int32 바이어스를 사용한 **출력 채널별 int8 재양자화**. 기본 VAIML 변환 레이어를 비트 정확하게 재현합니다. |
| [topk/0_untiled](/docs/30_custom_ops/tutorial/topk/0_untiled)<br>[topk/0_untiled](/docs/30_custom_ops/tutorial/topk/0_untiled)    | Simple TopK operation that selects the largest K elements. Runs on a single AIE core and only supports tensors that fit in L1.<br>가장 큰 K 요소를 선택하는 간단한 TopK 작업입니다. 단일 AIE 코어에서 실행되며 L1에 맞는 텐서만 지원합니다. |
| [topk/1_tiled](/docs/30_custom_ops/tutorial/topk/1_tiled)<br>[topk/1_tiled](/docs/30_custom_ops/tutorial/topk/1_tiled)    | A tiled TopK implementation that supports tensor sizes that don't fit in L1 memory and demonstrates asynchronous kernel outputs.<br>L1 메모리에 맞지 않는 텐서 크기를 지원하고 비동기 커널 출력을 보여주는 타일형 TopK 구현입니다. |
| [affinegrid2d/0_nonvectorized_1x1](/docs/30_custom_ops/tutorial/affinegrid2d/0_nonvectorized_1x1)<br>[affinegrid2d/0_nonVectorized_1x1](/docs/30_custom_ops/tutorial/affinegrid2d/0_nonVectorized_1x1)    | A non-vectorized AffineGrid2D implementation running on a single AIE core. Simplest starting point for AffineGrid2D.<br>단일 AIE 코어에서 실행되는 벡터화되지 않은 AffineGrid2D 구현입니다. AffineGrid2D의 가장 간단한 시작점입니다.|
| [affinegrid2d/1_vectorized_1x1](/docs/30_custom_ops/tutorial/affinegrid2d/1_vectorized_1x1)<br>[affinegrid2d/1_Vectorized_1x1](/docs/30_custom_ops/tutorial/affinegrid2d/1_Vectorized_1x1)    | A tiled and vectorized AffineGrid2D implementation that supports tensor sizes that don't fit in L1 memory.<br>L1 메모리에 맞지 않는 텐서 크기를 지원하는 타일링 및 벡터화된 AffineGrid2D 구현입니다.|
| [affinegrid2d/2_distributed_4x4](/docs/30_custom_ops/tutorial/affinegrid2d/2_distributed_4x4)<br>[affinegrid2d/2_distributed_4x4](/docs/30_custom_ops/tutorial/affinegrid2d/2_distributed_4x4)    | Expands on the previous example by distributing the AffineGrid2D operation across all 16 AIE cores.<br>AffineGrid2D 작업을 16개 AIE 코어 전체에 분산하여 이전 예를 확장합니다.|
| [topk_indices/0_untiled](/docs/30_custom_ops/tutorial/topk_indices/0_untiled)<br>[topk_indices/0_untiled](/docs/30_custom_ops/tutorial/topk_indices/0_untiled)    | TopK operation that returns the indices of the K largest elements from a 1D input tensor. Runs on a single AIE core and fits the entire input in L1 memory.<br>1D 입력 텐서에서 K개의 가장 큰 요소의 인덱스를 반환하는 TopK 연산입니다. 단일 AIE 코어에서 실행되며 전체 입력을 L1 메모리에 맞춥니다. |
| [reducemax](reducemax/README.md)<br>[리듀스맥스](리듀스맥스/README.md)           | Custom implementation of ONNX ReduceMax. Two variants show how the reduction axis drives the tiling: a strided outer-axis max, and a de-interleaved inner-most-axis max.<br>ONNX ReduceMax의 사용자 정의 구현. 두 가지 변형은 축소 축이 타일링을 구동하는 방법을 보여줍니다. 스트라이드된 외부 축 최대값과 디인터리브된 가장 안쪽 축 최대값입니다. |
| [gridsample2d/0_scalar_1x1](/docs/30_custom_ops/tutorial/gridsample2d/0_scalar_1x1)<br>[gridsample2d/0_scalar_1x1](/docs/30_custom_ops/tutorial/gridsample2d/0_scalar_1x1)           | Non-vectorized scalar GridSample2D implementation running on a single AIE core. Simplest starting point for GridSample2D.<br>단일 AIE 코어에서 실행되는 벡터화되지 않은 스칼라 GridSample2D 구현입니다. GridSample2D의 가장 간단한 시작점입니다. |
| [gridsample2d/1_tiled_1x1](/docs/30_custom_ops/tutorial/gridsample2d/1_tiled_1x1)<br>[gridsample2d/1_tiled_1x1](/docs/30_custom_ops/tutorial/gridsample2d/1_tiled_1x1)           | Single-core tiled GridSample2D implementation. Supports tensor sizes that don't fit in L1 memory.<br>단일 코어 타일형 GridSample2D 구현. L1 메모리에 맞지 않는 텐서 크기를 지원합니다. |
| [gridsample2d/2_tiled_1x4](/docs/30_custom_ops/tutorial/gridsample2d/2_tiled_1x4)<br>[gridsample2d/2_tiled_1x4](/docs/30_custom_ops/tutorial/gridsample2d/2_tiled_1x4)           | Vectorized GridSample2D implementation running on 4 AIE cores.<br>4개의 AIE 코어에서 실행되는 벡터화된 GridSample2D 구현입니다. |
| [gridsample2d/3_tiled_4x4](/docs/30_custom_ops/tutorial/gridsample2d/3_tiled_4x4)<br>[gridsample2d/3_tiled_4x4](/docs/30_custom_ops/tutorial/gridsample2d/3_tiled_4x4)           | Expands on the previous example by distributing the grid sampling operation across all 16 AIE cores.<br>16개 AIE 코어 전체에 그리드 샘플링 작업을 분산하여 이전 예를 확장합니다. |
| [spacetodepth](/docs/30_custom_ops/tutorial/spacetodepth)<br>[공간깊이](/docs/30_custom_ops/tutorial/spaceto깊이)           | INT8 SpaceToDepth (`blocksize=2`), showing a custom op that changes the DDR data layout (`auto_pad`, `ddr_tensor_layout`). Also demonstrates **PDLL matching**: the input model is a stock, unmodified ONNX `SpaceToDepth`, and the custom op is created by the compiler from a `match_on` pattern instead of by hand-editing the model.<br>INT8 SpaceToDepth(`blocksize=2`), DDR 데이터 레이아웃(`auto_pad`, `ddr_tensor_layout`)을 변경하는 사용자 정의 작업을 표시합니다. 또한 **PDLL 일치**를 보여줍니다. 입력 모델은 수정되지 않은 ONNX 'SpaceToDepth'이며 사용자 지정 작업은 모델을 직접 편집하는 대신 컴파일러가 'match_on' 패턴에서 생성합니다. |
| [addmul](addmul/README.md)<br>[addmul](addmul/README.md)           | Custom implementation of `out = (A + B) * C` as a **two-phase** op whose phase boundary lives in DDR. Shows `create_l3_scratch()` / `create_phase()`, two separate L1 input buffers reached over the column- and row-broadcast families (one kernel argument each, no packed pointer arithmetic), and `setTemporalIterations()` to stream a tensor through chunk-sized memory tiles.<br>위상 경계가 DDR에 있는 **2단계** 작업으로 'out = (A + B) * C'를 맞춤 구현합니다. `create_l3_scratch()` / `create_phase()`, 열 및 행 브로드캐스트 계열(각각 하나의 커널 인수, 패킹된 포인터 연산 없음)에 도달한 두 개의 개별 L1 입력 버퍼, 청크 크기 메모리 타일을 통해 텐서를 스트리밍하는 `setTemporalIterations()`를 보여줍니다. |

## Setup (Linux)
## 설정(리눅스)
This description assumes that we are running on Ubuntu 24.04 or later
이 설명에서는 Ubuntu 24.04 이상에서 실행되고 있다고 가정합니다.

Source the virtual environment you want to test, e.g.:
테스트하려는 가상 환경을 소싱하세요. 예:
```
source <ryzen_ai_dir>/lnx64/bin/activate
```

If you have a Strix machine, source the XRT installation.
Strix 시스템이 있는 경우 XRT 설치를 소싱하세요.
Skip
건너뛰기
this step if you only want to compile but not run the model.
모델을 실행하지 않고 컴파일만 하려는 경우 이 단계를 수행하세요.
```
source /opt/xilinx/xrt/setup.sh
```

Not strictly required, but highly recommended are:
엄격하게 요구되는 것은 아니지만 적극 권장되는 사항은 다음과 같습니다.
```
export DEBUG_VAIML_PARTITION=1
export FLEXML_PRINT_VITISTOOLS_OUTPUT=1
```

This will make sure error messages from VAIML are shown on the console.
이렇게 하면 VAIML의 오류 메시지가 콘솔에 표시됩니다.

## Run an example
## 예제 실행

Copy all example files to some location, we call it `<mydir>` here.
모든 예제 파일을 특정 위치에 복사합니다. 여기서는 `<mydir>`이라고 합니다.

To execute it, use:
이를 실행하려면 다음을 사용하십시오.
```
cd <mydir>
python run.py
```

This will compile the ONNX model, run it on the AIE and output results.
그러면 ONNX 모델이 컴파일되어 AIE에서 실행되고 결과가 출력됩니다.
For details, see the python script that is called.
자세한 내용은 호출되는 Python 스크립트를 참조하세요.
