<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->
# globaltonemapping — HDR to LDR tone mapping baseline
# globaltonemapping - HDR에서 LDR로의 톤 매핑 기준선

Vitis Vision (xf_opencv) L1 baseline example for **Global Tone Mapping (GTM)** - converts High Dynamic Range images to Low Dynamic Range displays.
**GTM(Global Tone Mapping)**을 위한 Vitis Vision(xf_opencv) L1 기준 예 - 높은 동적 범위 이미지를 낮은 동적 범위 디스플레이로 변환합니다.

- **Environment:** Claude Code + `claude-opus-4.8` model, Vitis HLS 2026.1 / Vivado 2026.1
- **환경:** Claude Code + `claude-opus-4.8` 모델, Vitis HLS 2026.1 / Vivado 2026.1

> **Baseline design:** NPPC=1, no optimizations.
> **기본 설계:** NPPC=1, 최적화 없음.
> This is the starting point for `/hls-optimize` skill.
> 이것이 `/hls-optimize` 스킬의 출발점입니다.

---

## Files
## 파일

| File<br>파일 | Role<br>역할 |
|---|---|
| [`xf_gtm_accel.cpp`](xf_gtm_accel.cpp)<br>[`xf_gtm_accel.cpp`](xf_gtm_accel.cpp)   | Top-level HLS kernel with AXI interfaces<br>AXI 인터페이스를 갖춘 최상위 HLS 커널 |
| [`xf_gtm_tb.cpp`](xf_gtm_tb.cpp)<br>[`xf_gtm_tb.cpp`](xf_gtm_tb.cpp)         | C testbench — OpenCV reference comparison<br>C 테스트벤치 — OpenCV 참조 비교 |
| [`xf_config_params.h`](xf_config_params.h)<br>[`xf_config_params.h`](xf_config_params.h) | Design parameters (HEIGHT=168, WIDTH=256, **NPPC=1**)<br>설계 매개변수(높이=168, 너비=256, **NPPC=1**) |
| [`ltm_input_s.png`](ltm_input_s.png)<br>[`ltm_input_s.png`](ltm_input_s.png)     | Test input image (168×256 HDR PNG)<br>테스트 입력 이미지(168×256 HDR PNG) |
| [`hls_config.tmpl`](hls_config.tmpl)<br>[`hls_config.tmpl`](hls_config.tmpl)     | HLS configuration template<br>HLS 구성 템플릿 |
| [`gen_config.sh`](gen_config.sh)<br>[`gen_config.sh`](gen_config.sh)         | Script to generate hls_config.cfg<br>hls_config.cfg 생성 스크립트 |

---

## Prerequisites
## 전제조건

Set OpenCV paths in your environment:
환경에서 OpenCV 경로를 설정합니다.
```bash
export OPENCV_INCLUDE=/path/to/opencv/include/opencv4
export OPENCV_LIB=/path/to/opencv/lib
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$OPENCV_LIB
```

Then generate the HLS configuration:
그런 다음 HLS 구성을 생성합니다.
```bash
./gen_config.sh
```

---

## How to run the design
## 디자인을 실행하는 방법

In Claude Code, from this design directory:
Claude Code의 이 디자인 디렉토리에서:

```
/hls-optimize  xf_gtm_accel.cpp  Reduce true-latency by 2x while using no more than 1.5x the resources
```

The skill chain will:
스킬 체인은 다음을 수행합니다.

1. [`/hls-optimize`](../../skills/hls-optimize/SKILL.md) — analyze baseline, apply optimizations (parallel processing with NPPC=2/4, pragmas), iterate until 2× latency reduction met
1. [`/hls-optimize`](../../skills/hls-optimize/SKILL.md) — 기준 분석, 최적화 적용(NPPC=2/4, pragma를 사용한 병렬 처리), 2배의 대기 시간 감소가 충족될 때까지 반복
