<!-- ko-translation: enabled -->
<!--- Copyright (C) 2026 Advanced Micro Devices, Inc. All rights reserved.--->
# Custom ReduceMax Implementation
# 사용자 정의 ReduceMax 구현

This directory contains a custom implementation of the [ONNX ReduceMax](https://onnx.ai/onnx/operators/onnx__ReduceMax.html) operation.
이 디렉터리에는 [ONNX ReduceMax](https://onnx.ai/onnx/operators/onnx__ReduceMax.html) 작업의 사용자 지정 구현이 포함되어 있습니다.
The implementation demonstrates how to create custom operators with specialized tiling strategies for different reduction axes, including the case where the reduced axis is the inner-most (contiguous) dimension.
구현에서는 축소 축이 가장 안쪽(인접) 차원인 경우를 포함하여 다양한 축소 축에 대한 특수 타일링 전략을 사용하여 사용자 정의 연산자를 생성하는 방법을 보여줍니다.

## Overview
## 개요

The custom ReduceMax operation takes the maximum along a given axis of a bfloat16 tensor.
사용자 정의 ReduceMax 작업은 bfloat16 텐서의 지정된 축을 따라 최대값을 취합니다.
Two reduction layouts are shown, each in its own subdirectory:
두 가지 축소 레이아웃이 표시되며 각각은 자체 하위 디렉터리에 있습니다.
- **`reduce_max_axis_0` (Outer Dimension)**: Reduces along the outer axis of a `[4, 2, 31360]` tensor, taking the max across the strided outer planes.
- **`reduce_max_axis_0`(외부 차원)**: `[4, 2, 31360]` 텐서의 외부 축을 따라 감소하여 스트라이드된 외부 평면에서 최대값을 취합니다.
- **`reduce_max_axis_2` (Inner-most Dimension)**: Reduces along the inner-most axis (axis 2) of a `[31360, 2, 4]` tensor, taking the max over the contiguous run of elements per output.
- **`reduce_max_axis_2`(가장 안쪽 차원)**: `[31360, 2, 4]` 텐서의 가장 안쪽 축(축 2)을 따라 감소하여 출력당 연속 요소 실행에 대한 최대값을 취합니다.

Both layouts reduce to the same number of independent outputs (`N = 62720`) and distribute that work across the 16 cores of the 4x4 AIE overlay.
두 레이아웃 모두 동일한 수의 독립 출력(`N = 62720`)으로 감소하고 해당 작업을 4x4 AIE 오버레이의 16개 코어에 분산합니다.

## Architecture
## 건축

### Core Components
### 핵심 구성요소

**Kernel Implementation** (`custom_reducemax*.cpp`)
**커널 구현** (`custom_reducemax*.cpp`)
   - Kernel implementation tailored to the reduction-axis layout (strided max vs. contiguous de-interleaved max)
   - 축소 축 레이아웃에 맞춰진 커널 구현(스트라이드 최대값 대 연속 디인터리브 최대값)

**Tiling Strategy** (`custom_reducemax_tiling.py`)
**타일링 전략** (`custom_reducemax_tiling.py`)
   - Memory hierarchy management (L3 -> L2 -> L1)
   - 메모리 계층 관리(L3 -> L2 -> L1)
   - Core distribution across 4x4 AIE overlay
   - 4x4 AIE 오버레이 전반에 걸친 코어 배포

**Configuration** (`custom_reducemax.yaml`)
**구성** (`custom_reducemax.yaml`)
   - Links kernel and tiling implementations
   - 커널 및 타일링 구현 연결
   - Defines the ADF wrapper signature
   - ADF 래퍼 서명을 정의합니다.

## Implementation Details
## 구현 세부정보

### Axis 0 Reduction (Outer Dimension)
### 축 0 축소(외형 치수)

**Input Shape**: [4, 2, 31360] -> **Output Shape**: [1, 2, 31360]
**입력 형태**: [4, 2, 31360] -> **출력 형태**: [1, 2, 31360]

- **Strategy**: The reduced axis (extent `R = 4`) is the strided outer dimension; the `N = 2 x 31360 = 62720` output positions are independent.
- **전략**: 축소된 축('R = 4' 범위)은 스트라이드된 외부 차원입니다. `N = 2 x 31360 = 62720` 출력 위치는 독립적입니다.
- Distribute `N` across the cores and keep the `R`-way max inside each core.
- 코어 전체에 'N'을 배포하고 각 코어 내부에 'R' 방향 최대값을 유지합니다.
- **Memory Layout**:
- **메모리 레이아웃**:
  - L2: View the input as plane-major `[R, N]`; stream `16 x T` output positions per rotation.
  - L2: 입력을 평면 메이저 `[R, N]`으로 봅니다. 회전당 '16 x T' 출력 위치 스트림.
  - L1: Broadcast each column's `[R, ROWS*T]` slice to its 4 cores; each core selects its own `[R, T]` slice via its row index.
  - L1: 각 열의 `[R, ROWS*T]` 슬라이스를 4개 코어에 브로드캐스팅합니다. 각 코어는 행 인덱스를 통해 자체 `[R, T]` 슬라이스를 선택합니다.
- **Core Distribution**: Each core takes the max of `R` strided planes for its `T` output positions; the OFM is collected per-core with `SpatialDistribute2D`.
- **코어 배포**: 각 코어는 'T' 출력 위치에 대해 최대 'R' 스트라이드 평면을 사용합니다. OFM은 `SpatialDistribute2D`를 사용하여 코어별로 수집됩니다.

### Axis 2 Reduction (Inner-most Dimension)
### 축 2 축소(가장 안쪽 차원)

**Input Shape**: [31360, 2, 4] -> **Output Shape**: [31360, 2, 1]
**입력 형태**: [31360, 2, 4] -> **출력 형태**: [31360, 2, 1]

- **Strategy**: The reduced axis (extent `K = 4`) is the contiguous inner-most dimension.
- **전략**: 축소된 축('K = 4' 범위)은 인접한 가장 안쪽 차원입니다.
- Flattened, this is `N = 31360 x 2 = 62720` independent reductions, each over `K = 4` contiguous elements (`out[g] = max_k in[K*g + k]`).
- 평면화하면 이는 각각 `K = 4` 연속 요소(`out[g] = max_k in[K*g + k]`)에 대한 `N = 31360 x 2 = 62720` 독립적인 감소입니다.
- Distribute the `N` reductions across the cores.
- 코어 전체에 'N' 감소를 분배합니다.
- **Memory Layout**:
- **메모리 레이아웃**:
  - L2: Stream the flat tensor in 16-core slabs (`16 x T x K` input elements -> `16 x T` outputs per rotation).
  - L2: 16코어 슬래브에서 플랫 텐서를 스트리밍합니다('16 x T x K' 입력 요소 -> 회전당 '16 x T' 출력).
  - L1: Broadcast each column's `[ROWS, T*K]` slice to its 4 cores; each core selects its own `[T*K]` slice via its row index.
  - L1: 각 열의 `[ROWS, T*K]` 슬라이스를 4개 코어에 브로드캐스팅합니다. 각 코어는 행 인덱스를 통해 자체 `[T*K]` 슬라이스를 선택합니다.
- **Core Distribution**: Each core handles `T` reductions per call.
- **코어 배포**: 각 코어는 호출당 `T` 감소를 처리합니다.
- The kernel loads a wide contiguous bf16 vector, de-interleaves it into the `K` components, combines them with a pairwise `aie::max` tree, and stores the result, avoiding a scalar tail.
- 커널은 넓은 연속 bf16 벡터를 로드하고 이를 `K` 구성 요소로 디인터리브한 다음 쌍별 `aie::max` 트리와 결합하고 스칼라 테일을 피하면서 결과를 저장합니다.
