<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->
# edge-detection — RGB edge detection reference
# edge-Detection — RGB 가장자리 감지 참조

End-to-end test design for the LLM skill flow.
LLM 기술 흐름을 위한 엔드투엔드 테스트 설계.
The MATLAB algorithm here is an **RGB edge detector** with three operating modes (luminance, per-channel, max-gradient).
여기서 MATLAB 알고리즘은 세 가지 작동 모드(휘도, 채널별, 최대 그라데이션)를 갖춘 **RGB 가장자리 감지기**입니다.

- **Environment:** Claude Code + `claude-opus-4.8` model, Vitis HLS 2026.1 / Vivado 2026.1
- **환경:** Claude Code + `claude-opus-4.8` 모델, Vitis HLS 2026.1 / Vivado 2026.1

> **Audience:** first-time users of the skill flow.
> **대상:** 스킬 흐름을 처음 사용하는 사용자입니다.
> This design exercises the LLM pipeline against a Sobel-style stencil computation with multiple algorithmic variants.
> 이 디자인은 여러 알고리즘 변형을 사용하여 Sobel 스타일 스텐실 계산에 대해 LLM 파이프라인을 실행합니다.

---

## What edge detection does
## 가장자리 감지 기능

```
  ┌────────────┐   gradient kernels     ┌────────────────────┐  threshold   ┌────────────────┐
  │ RGB image  │ ─────────────────────► │ Gradient magnitude │ ───────────► │ Binary edges   │
  │            │   (Sobel-style 3×3)    │  per pixel         │              │ (logical mask) │
  └────────────┘                        └────────────────────┘              └────────────────┘
```

Each pixel's edge response is computed from a **3×3 neighbourhood** — a stencil computation.
각 픽셀의 가장자리 응답은 **3×3 이웃**(스텐실 계산)에서 계산됩니다.
The 3 modes differ in how they combine the R/G/B channels:
3가지 모드는 R/G/B 채널을 결합하는 방식이 다릅니다.

| Mode<br>모드 | What it does<br>기능 | Good for<br>다음에 좋습니다 |
|---|---|---|
| `luminance` (default)<br>'휘도'(기본값) | Convert RGB → grayscale, then 1× edge detection<br>RGB → 회색조로 변환한 다음 1× 가장자리 감지 | Standard edges, fastest<br>표준 엣지, 가장 빠른 속도 |
| `perChannel`<br>`채널별`          | Detect on each R/G/B separately, combine gradients<br>각 R/G/B를 개별적으로 감지하고 그라디언트를 결합합니다. | Color edges (red↔green, etc.)<br>색상 가장자리 (빨간색 ⇔ 녹색 등) |
| `maxGradient`<br>`최대 그래디언트`         | Take max gradient across channels per pixel<br>픽셀당 채널 전체에서 최대 그래디언트를 사용합니다. | Most sensitive<br>가장 민감한 |

---

## Files
## 파일

| File<br>파일 | Role<br>역할 |
|---|---|
| [`rgbEdgeDetector.m`](rgbEdgeDetector.m)<br>[`rgbEdgeDetector.m`](rgbEdgeDetector.m)         | Main edge detection function (all 3 modes)<br>메인 에지 검출 기능(3개 모드 모두) |
| [`testbench_edgeDetector.m`](testbench_edgeDetector.m)<br>[`testbench_edgeDetector.m`](testbench_edgeDetector.m) | Verification suite with test coverage for all modes<br>모든 모드에 대한 테스트 범위를 갖춘 검증 제품군 |
| [`edge_detection_runme.m`](edge_detection_runme.m)<br>[`edge_Detection_runme.m`](edge_Detection_runme.m) | Entry-point script — runs the testbench end-to-end<br>진입점 스크립트 — 테스트벤치를 처음부터 끝까지 실행합니다. |

---

## How to run the design
## 디자인을 실행하는 방법

**Design targets:**
**설계 대상:**

| Parameter<br>매개변수 | Value<br>가치 |
|---|---|
| Throughput<br>처리량  | **4400 FPS**<br>**4400FPS** |
| Clock<br>시계       | **3.3 ns** (≈ 303 MHz)<br>**3.3ns**(≒ 303MHz) |
| FPGA part<br>FPGA 부분   | `xczu9eg-ffvb1156-2-e`<br>`xczu9eg-ffvb1156-2-e` |

In Claude Code, from this design directory:
Claude Code의 이 디자인 디렉토리에서:

```
/matlab-to-cpp  rgbEdgeDetector  4400 FPS  part=xczu9eg-ffvb1156-2-e  clock=3.3
```

(`<design_name>` = the `.m` basename without extension.)
(`<design_name>` = 확장자가 없는 `.m` 기본 이름입니다.)

The skill chain will:
스킬 체인은 다음을 수행합니다.

1. [`/matlab-to-cpp`](../../skills/matlab-to-cpp/SKILL.md) — generate `golden/`, `sample_based/`, `frame_based/` with C++ ports verified against the MATLAB golden.
1. [`/matlab-to-cpp`](../../skills/matlab-to-cpp/SKILL.md) — MATLAB Golden에 대해 검증된 C++ 포트를 사용하여 `golden/`, `sample_based/`, `frame_based/`를 생성합니다.
2. [`/hls-architect`](../../skills/hls-architect/SKILL.md) — produce `rearchitect/v1/`.
2. [`/hls-architect`](../../skills/hls-architect/SKILL.md) — `rearchitect/v1/`을 생성합니다.
2. Likely a hierarchical `compute()`: gradient → magnitude → threshold as separate stages.
2. 아마도 계층적 `compute()`: 기울기 → 크기 → 임계값을 별도의 단계로 사용합니다.
3. [`/hls-optimize`](../../skills/hls-optimize/SKILL.md) — iterate pragmas until 4400 FPS @ 303 MHz is met.
3. [`/hls-optimize`](../../skills/hls-optimize/SKILL.md) — 303MHz에서 4400FPS가 충족될 때까지 pragma를 반복합니다.

