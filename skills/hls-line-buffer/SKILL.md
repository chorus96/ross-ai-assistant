<!-- ko-translation: enabled -->
---
name: hls-line-buffer
description: Fill the HLS line-buffer and window-buffer gap for stencil kernels, including TOTAL_ITER drain and verification. Use when generated buffer code misses the drain or the check.
license: MIT
metadata:
  author: "Sai Akhil Ayyagari"
  version: "1.0.0"
---
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# HLS Line Buffer
# HLS 라인 버퍼

The LLM can generate a line buffer and window buffer structure on its own.
LLM은 라인 버퍼와 윈도우 버퍼 구조를 자체적으로 생성할 수 있습니다.
This guide fills the gap on what it consistently gets wrong.
이 가이드는 지속적으로 잘못되는 부분에 대한 공백을 메워줍니다.

**Before generating code, understand the flow below, then produce code that matches it.**
**코드를 생성하기 전, 아래의 흐름을 이해하신 후 그에 맞는 코드를 생성해 주세요.**

**Constants** (derive before writing code):
**상수**(코드 작성 전 파생):
- `KRAD = KSIZE / 2` — kernel radius (e.g.
- `KRAD = KSIZE / 2` — 커널 반경(예:
- KSIZE=15 -> KRAD=7)
- KSIZE=15 -> KRAD=7)
- `NLB = KSIZE - 1` — number of line buffer rows
- `NLB = KSIZE - 1` — 라인 버퍼 행 수
- `WARMUP = KRAD * COLS + KRAD` — iterations before first valid output pixel
- `WARMUP = KRAD * COLS + KRAD` — 첫 번째 유효한 출력 픽셀 이전의 반복
- `TOTAL_ITER = ROWS * COLS + WARMUP` — total loop iterations
- `TOTAL_ITER = ROWS * COLS + WARMUP` — 총 루프 반복

---

## 1 — Run for TOTAL_ITER, not ROWS x COLS
## 1 — ROWS x COLS가 아닌 TOTAL_ITER에 대해 실행

After the last image pixel, the line buffer still holds KRAD rows of real data that have never been centered in the window.
마지막 이미지 픽셀 이후 라인 버퍼는 창 중앙에 배치되지 않은 실제 데이터의 KRAD 행을 계속 보유합니다.
Without extra iterations those rows are never output.
추가 반복이 없으면 해당 행은 출력되지 않습니다.

Fix: run the compute loop for TOTAL_ITER iterations.
수정: TOTAL_ITER 반복에 대해 컴퓨팅 루프를 실행합니다.
Read the actual pixel from the stream when `n < ROWS * COLS`, otherwise use zero — all in one loop:
n < ROWS * COLS`일 때 스트림에서 실제 픽셀을 읽습니다. 그렇지 않으면 0을 사용합니다 — 모두 하나의 루프에서:

```cpp
U8 pixel = (n < num_pixels) ? pixel_stream.read() : 0;
```

The zeros flush the last rows through so every pixel gets centered and output.
0은 마지막 행을 플러시하여 모든 픽셀이 중앙에 배치되고 출력됩니다.
Use **one loop** — not two separate loops (two loops add an FSM transition and cost ~16 cycles).
두 개의 별도 루프가 아닌 **하나의 루프**를 사용하십시오. 두 개의 루프는 FSM 전환을 추가하고 최대 16사이클의 비용이 듭니다.

---

## 2 — Verify the LLM applied step 1 correctly
## 2 - LLM이 1단계를 올바르게 적용했는지 확인

Check these before running csim/cosim:
csim/cosim을 실행하기 전에 다음을 확인하십시오.

- `TOTAL_ITER = ROWS * COLS + WARMUP` — not just `ROWS * COLS`
- `TOTAL_ITER = ROWS * COLS + WARMUP` - `ROWS * COLS`뿐만 아니라
- `WARMUP = KRAD * COLS + KRAD` — uses the same COLS as col_ptr wrapping (if col_ptr wraps at COLS_POW2, WARMUP must also use COLS_POW2)
- `WARMUP = KRAD * COLS + KRAD` — col_ptr 래핑과 동일한 COLS를 사용합니다(col_ptr이 COLS_POW2에서 래핑하는 경우 WARMUP도 COLS_POW2를 사용해야 함).
- Zero tail is inline in the loop (`n < num_pixels` ternary), not a second loop
- 제로 테일은 두 번째 루프가 아닌 루프(`n < num_pixels` 삼항)에서 인라인입니다.
- Output is gated on `n >= WARMUP` — no output written during warmup
- 출력은 'n >= WARMUP'으로 게이트됩니다. — 워밍업 중에는 출력이 기록되지 않습니다.


---

## Output Contract
## 출력 계약

### When Asked to Generate/Implement Code with Line Buffer
### 라인 버퍼를 사용하여 코드 생성/구현을 요청하는 경우

If the user asks you to **generate**, **implement**, or **create** code using a line buffer:
사용자가 라인 버퍼를 사용하여 코드 **생성**, **구현** 또는 **생성**을 요청하는 경우:

1. Generate the complete implementation following Steps 1-2 above
1. 위의 1~2단계에 따라 전체 구현을 생성합니다.
2. **If line buffer is successfully applied:** you must output the
2. **라인 버퍼가 성공적으로 적용되면:** 다음을 출력해야 합니다.

```
LINE_BUFFER_VERDICT: GENERATED_PASS
```
or if it did not apply, you must
또는 해당 사항이 적용되지 않은 경우 반드시
```
LINE_BUFFER_VERDICT: GENERATED_FAILED
```
