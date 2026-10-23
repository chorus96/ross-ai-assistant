<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->
# hls-intro-matmul — Matrix Multiplication C++ → HLS
# hls-intro-matmul — 행렬 곱셈 C++ → HLS

Introductory example for the **C++ → HLS** skill flow.
**C++ → HLS** 기술 흐름에 대한 소개 예입니다.
This design starts with a simple triple-nested loop matrix multiplication and demonstrates how the skills transform it into an optimized HLS implementation.
이 디자인은 간단한 삼중 중첩 루프 행렬 곱셈으로 시작하여 기술을 통해 이를 최적화된 HLS 구현으로 변환하는 방법을 보여줍니다.

- **Environment:** Claude Code + `claude-opus-4.8` model, Vitis HLS 2026.1 / Vivado 2026.1
- **환경:** Claude Code + `claude-opus-4.8` 모델, Vitis HLS 2026.1 / Vivado 2026.1

> **Audience:** Users starting with existing C++ algorithms who want to accelerate them with HLS.
> **대상:** HLS로 가속화하려는 기존 C++ 알고리즘으로 시작하는 사용자입니다.
> This design exercises the HLS-architect skill flow without requiring MATLAB.
> 이 디자인은 MATLAB 없이도 HLS 아키텍처 기술 흐름을 실행합니다.

---

## What matrix multiplication does
## 행렬 곱셈의 기능

Matrix multiplication computes `C = A × B` where:
행렬 곱셈은 `C = A × B`를 계산합니다. 여기서:
- **A** is an `M × K` matrix
- **A**는 'M × K' 행렬입니다.
- **B** is a `K × N` matrix
- **B**는 'K × N' 행렬입니다.
- **C** is an `M × N` matrix (output)
- **C**는 'M × N' 행렬(출력)입니다.

```
  ┌──────────┐   ┌──────────┐   ┌──────────┐
  │ A (M×K)  │ × │ B (K×N)  │ = │ C (M×N)  │
  └──────────┘   └──────────┘   └──────────┘

  C[i][j] = Σ A[i][k] × B[k][j]  (for k = 0 to K-1)
```

The baseline implementation uses three nested loops — a classic compute pattern that HLS can heavily optimize through tiling, pipelining, and array partitioning.
기본 구현에서는 HLS가 타일링, 파이프라이닝 및 배열 파티셔닝을 통해 크게 최적화할 수 있는 클래식 컴퓨팅 패턴인 세 개의 중첩 루프를 사용합니다.

---

## Files
## 파일

| File<br>파일 | Role<br>역할 |
|---|---|
| [`kernel.cpp`](kernel.cpp)<br>[`kernel.cpp`](kernel.cpp) | Baseline kernel — simple triple-nested loop implementation<br>기준 커널 — 간단한 삼중 중첩 루프 구현 |
| [`kernel.hpp`](kernel.hpp)<br>[`kernel.hpp`](kernel.hpp) | Header file defining `MAX_SIZE=4096` and the `kernel()` function signature<br>`MAX_SIZE=4096` 및 `kernel()` 함수 서명을 정의하는 헤더 파일 |
| [`main.cpp`](main.cpp)<br>[`main.cpp`](main.cpp) | Testbench — generates random matrices, runs the kernel, verifies against reference using ULP-based float comparison<br>테스트벤치 — 무작위 행렬 생성, 커널 실행, ULP 기반 부동 소수점 비교를 사용하여 참조에 대해 확인 |

---

## How to run the design
## 디자인을 실행하는 방법

**Design targets:**
**설계 대상:**

| Parameter<br>매개변수 | Value<br>가치 |
|---|---|
| Throughput goal<br>처리량 목표 | **15 GOPS**<br>**15GOPS** |
| FPGA part<br>FPGA 부분 | xczu9eg-ffvb1156-2-e<br>xczu9eg-ffvb1156-2-e |
| Clock period<br>시계 주기 | 3.3 ns |
| Matrix size<br>매트릭스 크기 | **64×64×64** (ops per call = 2×64³ = 524 288 MAC ops)<br>**64×64×64**(호출당 작업 = 2×64³ = 524 288 MAC 작업) |

In Claude Code, from this design directory:
Claude Code의 이 디자인 디렉토리에서:

/hls-architect throughput=15 GOPS part=xczu9eg-ffvb1156-2-e clock=3.3 matrix size(M,N,K) = 64,64,64
/hls-architect 처리량=15 GOPS 부분=xczu9eg-ffvb1156-2-e 클럭=3.3 매트릭스 크기(M,N,K) = 64,64,64


The skill chain will:
스킬 체인은 다음을 수행합니다.

1. [`/hls-architect`](../../skills/hls-architect/SKILL.md) — generate HLS DATAFLOW architecture (load → compute → store)
1. [`/hls-architect`](../../skills/hls-architect/SKILL.md) — HLS DATAFLOW 아키텍처 생성(로드 → 컴퓨팅 → 저장)
2. [`/hls-optimize`](../../skills/hls-optimize/SKILL.md) — iterate pragmas until 15 GOPS throughput target is met
2. [`/hls-optimize`](../../skills/hls-optimize/SKILL.md) — 15GOPS 처리량 목표가 충족될 때까지 pragma를 반복합니다.

