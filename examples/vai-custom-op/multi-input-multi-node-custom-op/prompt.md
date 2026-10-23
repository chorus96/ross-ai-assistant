<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: Apache-2.0 WITH LLVM-exception
SPDX-License-Identifier: LLVM 예외가 포함된 Apache-2.0
-->
# Multi-Input, Multi-Node — Nine Ops and Three Inputs In, One Custom Op Out
# 다중 입력, 다중 노드 — 9개의 Ops 및 3개의 입력 입력, 1개의 맞춤형 Op Out

Quick-start prompt for the **custom op** skill (`/vai-custom-op`): hand it a
**custom op** 스킬(`/vai-custom-op`)에 대한 빠른 시작 프롬프트:
nine-node graph with **three real inputs**, plus a
**실제 입력 3개**와
`vitisai_config.json`, and let it generate everything else — kernel, tiling, op
`vitsai_config.json`, 그리고 커널, 타일링, op 등 다른 모든 것을 생성하도록 합니다.
config, the rewritten model, and the `custom_ops` section of the compile config.
config, 재작성된 모델, 컴파일 구성의 'custom_ops' 섹션.

The only two files you are provided with are in [`input/`](input/).
제공되는 파일 두 개는 [`input/`](input/)에 있습니다.

## The prompt
## 프롬프트

```
/vai-custom-op --model input/model.onnx \
    --vitisai-config input/vitisai_config.json \
    --ops Gemm_0,gemm_output_reshape,Slice_4,Exp,Mul_2,Slice_1,Mul_1,Add_1,Concat_5:myop

Create a single custom op that fuses every op between the tensors gemm_input
and concat_output, i.e. the Gemm plus the whole
Reshape / Slice / Exp / Mul / Add / Concat chain that follows it.

```

## Notes
## 메모

- **`--ops <nodes>:myop`** — the node list is a single cluster, so all nine
- **`--ops <nodes>:myop`** — 노드 목록은 단일 클러스터이므로 9개 모두
nodes collapse into one custom op.
노드는 하나의 사용자 정의 작업으로 축소됩니다.
The spec matches on **node name**.
사양은 **노드 이름**과 일치합니다.
- **Three inputs, two buffers** — this is the point of the example.
- **3개의 입력, 2개의 버퍼** — 이것이 예시의 핵심입니다.
- A custom op
- 맞춤형 작전
sees at most two core-tile buffers (IFM and WTS), so the three inputs cannot
최대 2개의 코어 타일 버퍼(IFM 및 WTS)를 볼 수 있으므로 3개의 입력은
each get their own.
각자 자신의 것을 얻습니다.
The prompt tells the skill to solve it in the tiling, not
프롬프트는 스킬이 아닌 타일링에서 해결하도록 지시합니다.
by adding `Concat`/`Reshape` nodes to the graph, which would cost a DDR round
'Concat'/'Reshape' 노드를 그래프에 추가하면 DDR 라운드 비용이 발생합니다.
trip.
여행.
- **`scale` fans out to both `Mul_1` and `Mul_2`**, so whatever packing the
- **`scale`은 `Mul_1`과 `Mul_2` 모두로 팬아웃되므로 무엇이든 포장할 수 있습니다.
skill picks has to serve both branches.
스킬 선택은 두 지점을 모두 제공해야 합니다.
- **Device**: `ve2`, from the supplied `vitisai_config.json`.
- **장치**: 제공된 `vitsai_config.json`의 `ve2`.
- The skill does not
- 스킬은 안됨
change the model-wide knobs; it only appends the `custom_ops` section.
모델 전체의 손잡이를 변경합니다. `custom_ops` 섹션만 추가합니다.
- **Board**: the prompt does not name a board, so it works unchanged whatever
- **보드**: 프롬프트는 보드 이름을 지정하지 않으므로 변경되지 않고 작동합니다.
your setup is.
귀하의 설정은 다음과 같습니다.
Export `BOARDHOST` and `BOARD_USER`, plus either
`BOARDHOST` 및 `BOARD_USER` 및 둘 중 하나 내보내기
`BOARD_PASSWORD` or `BOARD_KEY`, and the skill picks them up when it calls
`BOARD_PASSWORD` 또는 `BOARD_KEY`, 호출 시 스킬이 이를 선택합니다.
`run_on_board.py`.
`run_on_board.py`.
Without a reachable board the skill will warn that correctness rests on
접근 가능한 보드가 없으면 기술은 정확성이 다음에 달려 있음을 경고합니다.
x86sim alone and that the Phase 5 optimization loop degrades to static
x86sim만 사용하면 5단계 최적화 루프가 정적으로 저하됩니다.
metrics.
측정항목.
