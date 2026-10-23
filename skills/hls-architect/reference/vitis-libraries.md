<!-- ko-translation: enabled -->
---
description: Vitis Libraries study guide — browse L1 source code to learn HLS coding patterns. DO NOT include library headers in user designs. Study, then write your own.
---
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Pattern References — Study, Then Write Your Own Code
# 패턴 참조 — 연구 후 자신만의 코드 작성

**Rule for all references**: DO NOT include or link any headers or source files in user designs.
**모든 참조에 대한 규칙**: 사용자 디자인에 헤더나 소스 파일을 포함하거나 링크하지 마십시오.
Use them as **pattern references only** — read the implementation, understand the HLS coding style, then write equivalent code from scratch in the user's own kernel.
**패턴 참조 전용**으로 사용하세요. 구현을 읽고 HLS 코딩 스타일을 이해한 다음 사용자 자체 커널에서 처음부터 동등한 코드를 작성하세요.

---

## Vitis Libraries (Production AMD/Xilinx HLS Code)
## Vitis 라이브러리(프로덕션 AMD/Xilinx HLS 코드)

**Repository**: https://github.com/Xilinx/Vitis_Libraries/tree/2025.1
**저장소**: https://github.com/Xilinx/Vitis_Libraries/tree/2025.1

Production-quality HLS implementations covering vision, DSP, linear algebra, data compression, and more.
비전, DSP, 선형 대수학, 데이터 압축 등을 다루는 프로덕션 품질의 HLS 구현입니다.
Study L1 source code to learn streaming interfaces, pragma usage, AXI/stream bridging, and memory optimization patterns.
L1 소스 코드를 연구하여 스트리밍 인터페이스, pragma 사용, AXI/스트림 브리징 및 메모리 최적화 패턴을 알아보세요.

---

## How to Look Up Library Code
## 도서관 코드를 찾는 방법

1. **Browse the L1 include directory** for your domain (see Library Domains table below):
1. 도메인에 대한 **L1 포함 디렉터리 찾아보기**(아래 라이브러리 도메인 표 참조):
`https://github.com/Xilinx/Vitis_Libraries/tree/2025.1/<domain>/L1/include/`
`https://github.com/Xilinx/Vitis_Libraries/tree/2025.1/<도메인>/L1/include/`

2. **Read the raw source** to study the implementation:
2. **원시 소스를 읽어** 구현을 연구하세요.
`https://raw.githubusercontent.com/Xilinx/Vitis_Libraries/2025.1/<domain>/L1/include/hw/<file>.hpp`
`https://raw.githubusercontent.com/Xilinx/Vitis_Libraries/2025.1/<도메인>/L1/include/hw/<파일>.hpp`

3. **Extract the pattern** and write your own kernel from scratch following that style
3. **패턴을 추출**하고 해당 스타일에 따라 처음부터 자신만의 커널을 작성합니다.

---

**Focus on L1** (individual HLS kernels for C-sim/csynth/cosim).
**L1에 중점**(C-sim/csynth/cosim용 개별 HLS 커널)
L2/L3 are for complete XCLBIN/multi-kernel apps.
L2/L3은 완전한 XCLBIN/다중 커널 앱용입니다.

---

## Library Domains
## 도서관 도메인

| Domain<br>도메인 | Path<br>경로 | Key Primitives<br>주요 기본 요소 |
|---|---|---|
| **DSP**<br>**DSP** | `dsp/L1/include/hw/`<br>`dsp/L1/include/hw/` | FFT (1D, 2D), FIR filters, DDS, matrix multiply (GeMM)<br>FFT(1D, 2D), FIR 필터, DDS, 행렬 곱셈(GeMM) |
| **Vision**<br>**비전** | `vision/L1/include/`<br>`비전/L1/포함/` | filter2D, resize, threshold, morphology, Harris, FAST, line buffer, window buffer<br>filter2D, 크기 조정, 임계값, 형태학, Harris, FAST, 라인 버퍼, 창 버퍼 |
| **Utils / Streams**<br>**유틸리티/스트림** | `utils/L1/include/xf_utils_hw/`<br>`utils/L1/include/xf_utils_hw/` | AXI↔stream bridges, stream split/combine/dup/reorder/sync<br>AXI⇔스트림 브릿지, 스트림 분할/결합/복제/재순서/동기화 |
| **Data Mover**<br>**데이터 이동자** | `data_mover/L1/include/xf_data_mover/`<br>`data_mover/L1/include/xf_data_mover/` | Load master→stream, store stream→master, 4D data movers<br>마스터→스트림 로드, 스트림 저장→마스터, 4D 데이터 이동기 |
| **BLAS**<br>**블라** | `blas/L1/include/`<br>`blas/L1/include/` | GEMM, GEMV, DOT, AXPY — matrix/vector ops<br>GEMM, GEMV, DOT, AXPY — 행렬/벡터 연산 |
| **Solver**<br>**해결사** | `solver/L1/include/`<br>`해석기/L1/포함/` | QR, SVD, Cholesky, linear system solver<br>QR, SVD, Cholesky, 선형 시스템 솔버 |
| **Data Compression**<br>**데이터 압축** | `data_compression/L1/include/`<br>`데이터 압축/L1/포함/` | LZ4, Snappy, Zlib, Zstd<br>LZ4, Snappy, Zlib, Zstd |
| **Security**<br>**보안** | `security/L1/include/`<br>`보안/L1/포함/` | AES, SHA, RSA, ECC<br>AES, SHA, RSA, ECC |
| **HPC**<br>**HPC** | `hpc/L1/include/`<br>`hpc/L1/포함/` | GEMM variants, systolic arrays<br>GEMM 변형, 수축기 배열 |


