<!-- ko-translation: enabled -->
<!--- Copyright (C) 2026 Advanced Micro Devices, Inc. All rights reserved.--->
# Custom Op example: AffineGrid2D
# 사용자 정의 작업 예: AffineGrid2D

The examples in this directory implement the 2D version of [ONNX AffineGrid](https://onnx.ai/onnx/operators/onnx__AffineGrid.html) operation for VitisAI.
이 디렉터리의 예제는 VitisAI에 대한 [ONNX AffineGrid](https://onnx.ai/onnx/operators/onnx__AffineGrid.html) 작업의 2D 버전을 구현합니다.
The AffineGrid operation generates a sampling grid of normalized coordinates (`[N, H, W, 2]`) from affine transformation matrices (`theta` with shape `[N, 2, 3]`), which can be used for spatial transformer operations.
AffineGrid 작업은 공간 변환기 작업에 사용할 수 있는 아핀 변환 행렬(`[N, 2, 3]` 형태의 `theta`)에서 정규화된 좌표(`[N, H, W, 2]`)의 샘플링 그리드를 생성합니다.
The examples progress from a simple scalar single-core implementation to a vectorized 16-core distributed version.
예제는 간단한 스칼라 단일 코어 구현에서 벡터화된 16코어 분산 버전으로 진행됩니다.

| Example<br>예 | Cores<br>코어 | Description<br>설명 |
|---------|-------|-------------|
| `0_nonvectorized_1x1`<br>`0_비벡터화_1x1` | 1 (1x1) | Non-vectorized scalar implementation. Processes one grid point at a time on a single AIE core. Simplest starting point.<br>벡터화되지 않은 스칼라 구현. 단일 AIE 코어에서 한 번에 하나의 그리드 포인트를 처리합니다. 가장 간단한 출발점. |
| `1_vectorized_1x1`<br>`1_벡터화된_1x1` | 1 (1x1) | Vectorized implementation on a single AIE core. Supports tensor sizes that don't fit in L1 memory via tiling.<br>단일 AIE 코어에서 벡터화된 구현입니다. 타일링을 통해 L1 메모리에 맞지 않는 텐서 크기를 지원합니다. |
| `2_distributed_4x4`<br>`2_분산_4x4` | 16 (4x4) | Vectorized implementation distributed across all 16 AIE cores.<br>16개 AIE 코어 전체에 분산된 벡터화된 구현입니다. |

## What is important to know?
## 아는 것이 중요합니까?

All folders share the same structure, with differences in the kernel (scalar vs vectorized) and the tiling configuration (1x1 vs 4x4).
모든 폴더는 커널(스칼라 대 벡터화)과 타일링 구성(1x1 대 4x4)의 차이를 제외하고 동일한 구조를 공유합니다.

The distribution in `2_distributed_4x4` is **not** along the batch dimension.
`2_distributed_4x4`의 분포는 배치 차원을 따르지 **않습니다**.
Instead, the output grid's H×W spatial points are split evenly across all 16 cores — each core computes `(H_OUT * W_OUT) / 16` grid points.
대신, 출력 그리드의 H×W 공간 포인트는 16개 코어 모두에 균등하게 분할됩니다. 각 코어는 `(H_OUT * W_OUT) / 16` 그리드 포인트를 계산합니다.
The theta matrix is broadcast to every core so that all cores work on the same batch sample simultaneously, and the batch dimension is iterated externally.
세타 행렬은 모든 코어에 브로드캐스트되어 모든 코어가 동일한 배치 샘플에서 동시에 작동하고 배치 차원이 외부에서 반복됩니다.
This approach allows distributing work across all cores even when the batch size is 1.
이 접근 방식을 사용하면 배치 크기가 1인 경우에도 모든 코어에 작업을 분산할 수 있습니다.

When executing `run.py`, statistics will be printed out.
run.py를 실행하면 통계가 출력됩니다.
These consist of the main statistics cossim (cosine similarity) and medAE (median absolute error), as well as max absolute error and mean absolute error.
이는 주요 통계 cossim(코사인 유사성) 및 medAE(중앙값 절대 오류), 최대 절대 오류 및 평균 절대 오류로 구성됩니다.
