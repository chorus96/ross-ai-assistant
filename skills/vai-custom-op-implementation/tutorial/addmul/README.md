<!-- ko-translation: enabled -->
<!--- Copyright (C) 2026 Advanced Micro Devices, Inc. All rights reserved.--->
# addmul
# 추가

Custom implementation of `out = (A + B) * C`: a **two-phase** op whose phase
`out = (A + B) * C`의 맞춤 구현: 단계가 있는 **2단계** 작업
boundary lives in DDR.
경계는 DDR에 존재합니다.

| Example<br>예 | Description<br>설명 |
|---|---|
| [0_l3_scratch](/docs/30_custom_ops/tutorial/addmul/0_l3_scratch)<br>[0_l3_scratch](/docs/30_custom_ops/tutorial/addmul/0_l3_scratch) | Two sequential phases on the same 16 cores, tied together by an L3 scratch buffer. Shows `create_l3_scratch()`, `create_phase()`, two separate L1 input buffers reached over the two broadcast families, and `setTemporalIterations()` for streaming a tensor through chunk-sized memory tiles.<br>동일한 16개 코어의 두 순차적 단계는 L3 스크래치 버퍼로 함께 연결됩니다. `create_l3_scratch()`, `create_phase()`, 두 개의 브로드캐스트 계열을 통해 도달한 두 개의 개별 L1 입력 버퍼, 청크 크기 메모리 타일을 통해 텐서를 스트리밍하기 위한 `setTemporalIterations()`를 보여줍니다. |

## Why two phases
## 왜 2단계인가?

The intermediate `A + B` is 128 KB against a 64 KB L1 data memory, so it cannot
중간 `A + B`는 64KB L1 데이터 메모리에 대해 128KB이므로
stay in the cores between the add and the multiply.
덧셈과 곱셈 사이의 핵심에 머물러라.
Splitting the op into two
작업을 두 개로 나누기
phases lets the add retire completely, spill its result to a DDR scratch
단계를 통해 추가가 완전히 종료되고 결과가 DDR 스크래치로 유출됩니다.
buffer, and have the multiply read it back.
버퍼를 생성하고 곱셈이 이를 다시 읽도록 합니다.
`AieConfig.create_l3_scratch()`
`AieConfig.create_l3_scratch()`
registers that buffer as an extra external buffer, whose memory the runtime
해당 버퍼를 추가 외부 버퍼로 등록합니다. 이 버퍼의 메모리는 런타임입니다.
provides; it is neither an operand nor a result of the op.
제공하다; 이는 피연산자도 아니고 연산의 결과도 아닙니다.
The lowering pairs
낮추는 쌍
writer and reader by `TensorVar` variable id, so the *same* `TensorVar` must be
작성자와 리더는 `TensorVar` 변수 ID에 의해 결정되므로 *동일한* `TensorVar`는 다음과 같아야 합니다.
used at both ends.
양쪽 끝에서 사용됩니다.

## Two operands, two broadcast families
## 두 개의 피연산자, 두 개의 브로드캐스트 계열

A core has exactly two DMA input ports, fed by two different broadcast
코어에는 정확히 두 개의 DMA 입력 포트가 있으며 두 개의 서로 다른 브로드캐스트에 의해 공급됩니다.
channels: one per column and one per row.
채널: 열당 하나, 행당 하나.
The op gives each operand its own L1
op는 각 피연산자에게 자체 L1을 제공합니다.
buffer and its own family -- the first rides the column broadcast, the second
버퍼와 그 자체의 가족 - 첫 번째는 칼럼 방송을 타고, 두 번째는
the row broadcast -- so the kernel takes two `input_buffer_conf` arguments
행 브로드캐스트 - 따라서 커널은 두 개의 `input_buffer_conf` 인수를 사용합니다.
rather than one packed buffer with pointer arithmetic.
포인터 산술을 사용하는 하나의 압축된 버퍼가 아닌

Core `(col, row)` therefore reads `col_l1[row]` and `row_l1[col]`, which name
따라서 코어 `(col, row)`는 `col_l1[row]` 및 `row_l1[col]`을 읽습니다.
the same global elements.
동일한 전역 요소.
The kernel picks its slice with
커널은 다음을 사용하여 슬라이스를 선택합니다.
`aie::tile::current().id()`.
`aie::tile::current().id()`.

Note that a buffer must sit within one memory tile of every DMA that reads it:
버퍼는 이를 읽는 모든 DMA의 하나의 메모리 타일 내에 있어야 합니다.
memory tiles reach their immediate neighbours, not the whole array.
메모리 타일은 전체 배열이 아닌 바로 이웃에 도달합니다.
The
는
column-broadcast channels source from memory tile 1 and the row-broadcast ones
열 브로드캐스트 채널은 메모리 타일 1과 행 브로드캐스트 채널에서 소스됩니다.
from memory tile 3, which is what fixes where each operand's L2 slab may live.
메모리 타일 3에서 각 피연산자의 L2 슬래브가 존재할 수 있는 위치를 수정합니다.

## Streaming, not staging
## 스테이징이 아닌 스트리밍

Each phase moves its tensor through the array in `ITERS` chunks.
각 단계는 `ITERS` 청크의 배열을 통해 텐서를 이동합니다.
The iteration
반복
count lives entirely on the L3 side, where `TileBy()` splits the tensor into
count는 전적으로 L3 측에 존재하며, 여기서 `TileBy()`는 텐서를 다음으로 분할합니다.
tiles; every L2 slab holds a single tile and is an identity pattern -- one tile
타일; 모든 L2 슬래브는 단일 타일을 보유하며 ID 패턴입니다. 즉, 하나의 타일
in, one tile out.
안으로, 하나의 타일이 나옵니다.

What re-arms a slab's DMA once per tile is `setTemporalIterations()`, which
타일당 한 번씩 슬래브의 DMA를 재설정하는 것은 `setTemporalIterations()`입니다.
becomes its `adf::repetition_count`.
`adf::repetition_count`가 됩니다.
This is separate from `num_buffers`, which
이는 `num_buffers`와는 별개입니다.
only says how many slots the slab ping-pongs between, not how often they turn
슬래브 탁구가 얼마나 자주 회전하는지가 아니라 슬롯 사이에 몇 개의 슬롯이 있는지만 나타냅니다.
over.
끝났다.
Without it the shim pushes the whole tensor at a memory tile that accepts
그것이 없으면 심은 허용되는 메모리 타일에서 전체 텐서를 푸시합니다.
one chunk, and the design deadlocks with every worker parked in
한 덩어리, 그리고 모든 작업자가 주차된 설계 교착 상태
`FW_STATE_WORKER_POST_WORK_BARRIER`.
'FW_STATE_WORKER_POST_WORK_BARRIER'.

The two sides of a transfer need agree on neither shape nor rank -- only on how
이적의 양측은 형태나 순위에 동의할 필요가 없습니다.
many elements make up one tile.
많은 요소가 하나의 타일을 구성합니다.
