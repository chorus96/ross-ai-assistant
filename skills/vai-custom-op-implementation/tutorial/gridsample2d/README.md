<!-- ko-translation: enabled -->
<!--- Copyright (C) 2026 Advanced Micro Devices, Inc. All rights reserved.--->
# Custom Op example: gridsample2d
# 사용자 정의 작업 예: Gridsample2d

The examples in this directory compute 2D grid sampling of inputs, e.g., like the [ONNX GridSample op](https://onnx.ai/onnx/operators/onnx__GridSample.html).
이 디렉터리의 예는 [ONNX GridSample 작업](https://onnx.ai/onnx/operators/onnx__GridSample.html)과 같은 입력의 2D 그리드 샘플링을 계산합니다.
The examples progress from a simple scalar single-core implementation to a vectorized 16-core distributed version.
예제는 간단한 스칼라 단일 코어 구현에서 벡터화된 16코어 분산 버전으로 진행됩니다.

| Example<br>예 | Cores doing distinct work<br>별개의 작업을 수행하는 코어 | Description<br>설명 |
|---------|-------|-------------|
| `0_scalar_1x1`<br>`0_스칼라_1x1` | 1 | Non-vectorized scalar implementation. Processes one pixel at a time. Simplest starting point.<br>벡터화되지 않은 스칼라 구현. 한 번에 한 픽셀씩 처리합니다. 가장 간단한 출발점. |
| `1_tiled_1x1`<br>`1_tiled_1x1` | 1 | Single-core vectorized implementation. Supports tensor sizes that don't fit in L1 memory via tiling.<br>단일 코어 벡터화 구현. 타일링을 통해 L1 메모리에 맞지 않는 텐서 크기를 지원합니다. |
| `2_tiled_1x4`<br>`2_타일_1x4` | 4 (1x4) | Vectorized implementation distributed across 4 AIE cores.<br>4개의 AIE 코어에 분산된 벡터화된 구현입니다. |
| `3_tiled_4x4`<br>`3_타일_4x4` | 16 (4x4) | Vectorized implementation distributed across all 16 AIE cores.<br>16개 AIE 코어 전체에 분산된 벡터화된 구현입니다. |

Note that the `1x1` in the first two names refers to the *programming model* -- one core's worth of work per kernel call, with no spatial split -- not to how many cores the design occupies.
처음 두 이름의 '1x1'은 디자인이 차지하는 코어 수를 나타내는 것이 아니라 *프로그래밍 모델*(공간 분할 없이 커널 호출당 하나의 코어 작업 가치)을 나타냅니다.
`AieConfig` always describes the full core array and requires every core's input and output ports to be connected, so these two examples broadcast the same tiles to **all** cores, which then all compute the **same** result.
'AieConfig'는 항상 전체 코어 배열을 설명하고 모든 코어의 입력 및 출력 포트가 연결되어야 하므로 이 두 가지 예는 동일한 타일을 **모든** 코어에 브로드캐스트한 다음 모두 **동일한** 결과를 계산합니다.
Only one core's output is meaningful; the rest are redundant copies.
하나의 코어 출력만 의미가 있습니다. 나머지는 중복 복사본입니다.

## What is important to know?
## 아는 것이 중요합니까?
All folders share the same structure, with differences in the kernel (scalar vs vectorized) and the tiling configuration.
모든 폴더는 커널(스칼라 대 벡터화) 및 타일링 구성의 차이를 제외하고 동일한 구조를 공유합니다.

When executing `run.py`, statistics will be printed out.
run.py를 실행하면 통계가 출력됩니다.
These consist of the main statistics cossim (cosine similarity) and medAE (median absolute error), as well as max absolute error and mean absolute error.
이는 주요 통계 cossim(코사인 유사성) 및 medAE(중앙값 절대 오류), 최대 절대 오류 및 평균 절대 오류로 구성됩니다.

There is an `images_input` folder, which contains images to be used for processing.
처리에 사용할 이미지가 포함된 'images_input' 폴더가 있습니다.
Outputs will be under `images_output_temp`.
출력은 `images_output_temp` 아래에 있습니다.
There are 16 different grids, so output naming will be `images_output_temp/outputs_cop_<grid_no>_<image_no>.png` for the custom op implementation, and `images_output_temp/outputs_ref_<grid_no>_<image_no>.png` for the reference implementation.
16개의 서로 다른 그리드가 있으므로 출력 이름 지정은 사용자 정의 작업 구현의 경우 `images_output_temp/outputs_cop_<grid_no>_<image_no>.png`이고 참조 구현의 경우 `images_output_temp/outputs_ref_<grid_no>_<image_no>.png`입니다.

Also note the extra attribute `H_tile` in `.onnxtxt` files.
또한 `.onnxtxt` 파일의 추가 속성 `H_tile`을 참고하세요.
This slices grid/output images into smaller pieces so that larger outputs fit in memory.
이는 더 큰 출력이 메모리에 맞도록 그리드/출력 이미지를 더 작은 조각으로 분할합니다.
For example, `H_tile=5` for a `100x100` grid will create 5 tiles of `20x100` grids to process sequentially.
예를 들어 '100x100' 그리드의 'H_tile=5'는 순차적으로 처리할 '20x100' 그리드의 타일 5개를 생성합니다.
