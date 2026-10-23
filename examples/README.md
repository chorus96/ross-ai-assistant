<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->
# Examples
# 예

Reference designs and walkthroughs for agentic FPGA workflows.
에이전트 FPGA 워크플로를 위한 참조 디자인 및 연습입니다.
Each example demonstrates a task or a comparison of skill-guided and direct tool use.
각 예는 작업을 보여주거나 기술 안내 및 직접 도구 사용을 비교합니다.

---

## Available examples
## 사용 가능한 예

| Example<br>예 | Input<br>입력 | Skills used<br>사용된 스킬 | Target<br>대상 |
|---------|-------|-------------|--------|
| [`hls-intro-matmul/`](hls-intro-matmul/)<br>[`hls-intro-matmul/`](hls-intro-matmul/) | C++ triple-nested loop<br>C++ 삼중 중첩 루프 | `/hls-architect`, `/hls-optimize`<br>`/hls-architect`, `/hls-optimize` | 15 GOPS @ 3.3 ns<br>15GOPS @ 3.3ns |
| [`edge-detection/`](edge-detection/)<br>[`가장자리 감지/`](가장자리 감지/) | MATLAB Sobel RGB edge detector<br>MATLAB Sobel RGB 가장자리 감지기 | `/matlab-to-cpp`, `/hls-architect`, `/hls-optimize`<br>`/matlab-to-cpp`, `/hls-architect`, `/hls-optimize` | 4400 FPS @ 3.3 ns<br>4400FPS @ 3.3ns |
| [`globaltonemapping/`](globaltonemapping/)<br>[`globaltonemapping/`](globaltonemapping/) | C++ Vitis Vision L1 baseline<br>C++ Vitis Vision L1 베이스라인 | `/hls-optimize`<br>`/hls-최적화` | 2x latency reduction<br>2배의 지연 시간 감소 |
| [`vivado-ip-configurator/isp-skill-comparison/`](vivado-ip-configurator/isp-skill-comparison/)<br>[`vivado-ip-configurator/isp-skill-comparison/`](vivado-ip-configurator/isp-skill-comparison/) | High-level ISP configuration request<br>상위 수준 ISP 구성 요청 | `vivado-ip-configurator`, compared with direct MCP use<br>'vivado-ip-configurator', 직접 MCP 사용과 비교 | 15 independent streaming inputs; recorded Luna and Sol results<br>15개의 독립적인 스트리밍 입력; Luna 및 Sol 결과 기록 |

---

## hls-intro-matmul
## hls-소개-matmul

**C++ → HLS, no MATLAB required.** A 64×64×64 matrix multiplication kernel optimized from a simple triple-nested loop to a pipelined HLS implementation.
**C++ → HLS, MATLAB 필요 없음.** 간단한 삼중 중첩 루프에서 파이프라인 HLS 구현까지 최적화된 64×64×64 행렬 곱셈 커널.

```
cd hls-intro-matmul/
/hls-architect throughput=15 GOPS part=xczu9eg-ffvb1156-2-e clock=3.3
```

Good starting point if you have an existing C++ algorithm and want to accelerate it with HLS.
기존 C++ 알고리즘이 있고 이를 HLS로 가속화하려는 경우 좋은 출발점이 됩니다.

---

## edge-detection
## 가장자리 감지

**MATLAB → C++ → HLS end-to-end.** An RGB Sobel-style edge detector with three operating modes (luminance, per-channel, max-gradient).
**MATLAB → C++ → HLS 엔드 투 엔드.** 세 가지 작동 모드(휘도, 채널별, 최대 그라데이션)를 갖춘 RGB Sobel 스타일 가장자리 감지기입니다.
Exercises the full skill chain including MATLAB golden capture, C++ port, architecture design, and pragma optimization.
MATLAB 골든 캡처, C++ 포트, 아키텍처 설계 및 pragma 최적화를 포함한 전체 기술 체인을 연습합니다.

```
cd edge-detection/
/matlab-to-cpp  rgbEdgeDetector  4400 FPS  part=xczu9eg-ffvb1156-2-e  clock=3.3
```

Good second example if you are coming from a MATLAB algorithm and need the full conversion flow.
MATLAB 알고리즘을 사용하고 전체 변환 흐름이 필요한 경우 좋은 두 번째 예입니다.

---

## globaltonemapping
## 글로벌 톤 매핑

**Optimize an existing HLS design.** A Vitis Vision L1 global tone mapping kernel (HDR → LDR).
**기존 HLS 디자인을 최적화합니다.** Vitis Vision L1 글로벌 톤 매핑 커널(HDR → LDR).
Starting point is the unoptimized NPPC=1 baseline; the goal is 2x latency reduction.
시작점은 최적화되지 않은 NPPC=1 기준선입니다. 목표는 대기 시간을 2배로 줄이는 것입니다.

```
cd globaltonemapping/
/hls-optimize  xf_gtm_accel.cpp  Reduce true-latency by 2x while using no more than 1.5x the resources
```

Good example if you already have an HLS design and want to tune pragmas to meet a performance target.
이미 HLS 설계가 있고 성능 목표를 달성하기 위해 pragma를 조정하려는 경우 좋은 예입니다.

---

## Loading skills
## 로딩 스킬

Skills live in [`../skills/`](../skills/).
스킬은 [`../skills/`](../skills/)에 있습니다.
Load them with `--plugin-dir` when launching Claude Code:
Claude Code를 시작할 때 `--plugin-dir`을 사용하여 로드하세요.

```bash
claude --plugin-dir /path/to/hls-llm-skills/skills
```
