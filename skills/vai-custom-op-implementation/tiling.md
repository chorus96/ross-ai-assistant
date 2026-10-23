<!-- ko-translation: enabled -->
<!--- Copyright (C) 2026 Advanced Micro Devices, Inc. All rights reserved.--->
# Tiling and DMA configuration
# 타일링 및 DMA 구성

AI Engines have the capability to process very large amounts of data despite a limited memory, thanks to their parallelism.
AI 엔진은 병렬성 덕분에 제한된 메모리에도 불구하고 매우 많은 양의 데이터를 처리할 수 있는 능력을 갖추고 있습니다.
The key to programming the AIEs to obtain performance is to make sure each AIE core is efficiently used.
성능을 얻기 위해 AIE를 프로그래밍하는 핵심은 각 AIE 코어가 효율적으로 사용되는지 확인하는 것입니다.
A single operator is able to be distributed over several AIE cores at once; one then needs to partition the operator's work so that each core can work in parallel, under the constraints posed by the limited memory capacity of the AIE cores and the architecture's data movement abilities.
단일 운영자는 동시에 여러 AIE 코어에 분산될 수 있습니다. 그런 다음 AIE 코어의 제한된 메모리 용량과 아키텍처의 데이터 이동 능력으로 인한 제약 조건 하에서 각 코어가 병렬로 작동할 수 있도록 운영자의 작업을 분할해야 합니다.

The process of adapting a custom op to partition its work is called tiling.
작업을 분할하기 위해 사용자 지정 작업을 조정하는 프로세스를 타일링이라고 합니다.
There are two tasks to be performed:
수행할 작업은 두 가지입니다.
* first, decompose the original problem and make the kernel work on a sub-problem;
* 먼저 원래 문제를 분해하고 커널이 하위 문제에서 작동하도록 만듭니다.
* second, allocate memory buffers and program data movements and kernel calls across the AIE grid, to process the entire problem.
* 둘째, AIE 그리드 전반에 걸쳐 메모리 버퍼와 프로그램 데이터 이동 및 커널 호출을 할당하여 전체 문제를 처리합니다.

This documentation will walk you through both of the above steps.
이 문서에서는 위의 두 단계를 모두 안내합니다.

**Related material**
**관련자료**

- End-to-end custom op flow (config, tiling script API, overlays): [Custom Ops Flow](README.md).
- 엔드투엔드 사용자 정의 작업 흐름(구성, 타일링 스크립트 API, 오버레이): [사용자 정의 작업 흐름](README.md).
- Kernel-side coding (what runs on each core, `lp_params`, memory): [AIE Kernel Programming Guide for Custom Ops](kernel_development.md).
- 커널 측 코딩(각 코어에서 실행되는 것, `lp_params`, 메모리): [Custom Ops를 위한 AIE 커널 프로그래밍 가이드](kernel_development.md).
- Small examples that grow from untiled → tiled → distributed: [Custom op tutorial index](tutorial/README.md).
- 경작 → 타일 → 분산에서 성장하는 작은 예: [사용자 정의 작업 튜토리얼 색인](tutorial/README.md).

## Table of contents
## 목차

- [AIE Architecture in VAIML](#aie-architecture-in-vaiml)
- [VAIML의 AIE 아키텍처](#aie-architecture-in-vaiml)
  - [AIE Components](#aie-components)
  - [AIE 구성요소](#aie-composites)
  - [AIE topology and VAIML overlay](#aie-topology-and-vaiml-overlay)
  - [AIE 토폴로지 및 VAIML 오버레이](#aie-topology-and-vaiml-overlay)
- [Tiling process example](#tiling-process-example)
- [타일링 처리 예시](#tiling-process-example)
  - [Step 1: decompose the problem; implement each tile in C++](#step-1-decompose-the-problem-implement-each-tile-in-c)
  - [1단계: 문제를 분해합니다. C++로 각 타일 구현](#step-1-decompose-the-problem-implement-each-tile-in-c)
    - [Purpose](#purpose)
    - [목적](#목적)
    - [Apply loop tiling to the kernel](#apply-loop-tiling-to-the-kernel)
    - [커널에 루프 타일링 적용](#apply-loop-tiling-to-the-kernel)
    - [Extract key information about the tile's inputs and outputs](#extract-key-information-about-the-tiles-inputs-and-outputs)
    - [타일의 입력 및 출력에 대한 주요 정보 추출](#extract-key-information-about-the-tiles-inputs-and-outputs)
    - [Create an AIE kernel with the right prototype](#create-an-aie-kernel-with-the-right-prototype)
    - [올바른 프로토타입으로 AIE 커널 생성](#create-an-aie-kernel-with-the-right-prototype)
    - [Optimization: re-use weights across calls](#optimization-re-use-weights-across-calls)
    - [최적화: 호출 전반에 걸쳐 가중치 재사용](#optimization-re-use-weights-across-calls)
  - [Step 2: Allocate memory on the AIE grid and program data movement](#step-2-allocate-memory-on-the-aie-grid-and-program-data-movement)
  - [2단계: AIE 그리드에 메모리 할당 및 데이터 이동 프로그램](#step-2-allocate-memory-on-the-aie-grid-and-program-data-movement)
    - [Abstract calculation of data movement](#abstract-calculation-of-data-movement)
    - [데이터 이동의 추상 계산](#abstract-calculation-of-data-movement)
    - [Writing the tiling script](#writing-the-tiling-script)
    - [타일링 스크립트 작성](#writing-the-tiling-script)
- [Guidance for kernel tiling](#guidance-for-kernel-tiling)
- [커널 타일링 지침](#guidance-for-kernel-tiling)
  - [Tiling and data flow analysis](#tiling-and-data-flow-analysis)
  - [타일링 및 데이터 흐름 분석](#tiling-and-data-flow-analytic)
    - [Loop tiling](#loop-tiling)
    - [루프 타일링](#loop-tiling)
    - [Tiling of inputs and outputs, data flow analysis](#tiling-of-inputs-and-outputs-data-flow-analysis)
    - [입력 및 출력 타일링, 데이터 흐름 분석](#tiling-of-inputs-and-outputs-data-flow-analytic)
    - [Cost floors and the DDR re-read factor](#cost-floors-and-the-ddr-re-read-factor)
    - [비용 하한선 및 DDR 다시 읽기 계수](#cost-floors-and-the-ddr-re-read-factor)
  - [Kernel interface](#kernel-interface)
  - [커널 인터페이스](#kernel-interface)
  - [Buffer usage](#buffer-usage)
  - [버퍼 사용량](#buffer-usage)
  - [Keeping buffers across kernel calls (asynchronous buffers)](#keeping-buffers-across-kernel-calls-asynchronous-buffers)
  - [커널 호출 전반에 걸쳐 버퍼 유지(비동기 버퍼)](#keeping-buffers-across-kernel-calls-asynchronous-buffers)
  - [Random accesses and absolute positioning within tensors](#random-accesses-and-absolute-positioning-within-tensors)
  - [텐서 내 무작위 액세스 및 절대 위치 지정](#random-accesses-and-absolute-positioning-within-tensors)
- [Guidance for AIE data movement](#guidance-for-aie-data-movement)
- [AIE 데이터 이동 지침](#guidance-for-aie-data-movement)
  - [Assignment of AIE cores to tiles](#assignment-of-aie-cores-to-tiles)
  - [타일에 AIE 코어 할당](#designment-of-aie-cores-to-tiles)
- [Python tiling script reference](#python-tiling-script-reference)
- [Python 타일링 스크립트 참조](#python-tiling-script-reference)
  - [Interface and provided information](#interface-and-provided-information)
  - [인터페이스 및 제공정보](#interface-and-provided-information)
    - [The `getTiling()` function](#the-gettiling-function)
    - [`getTiling()` 함수](#the-gettiling-function)
    - [Obtaining ONNX tensor shapes in the tiling script](#obtaining-onnx-tensor-shapes-in-the-tiling-script)
    - [타일링 스크립트에서 ONNX 텐서 모양 얻기](#obtaining-onnx-tensor-shapes-in-the-tiling-script)
    - [Using ONNX attributes in the tiling script](#using-onnx-attributes-in-the-tiling-script)
    - [타일링 스크립트에서 ONNX 속성 사용](#using-onnx-attributes-in-the-tiling-script)
  - [Buffer definition](#buffer-definition)
  - [버퍼 정의](#buffer-definition)
    - [External buffers](#external-buffers)
    - [외부 버퍼](#external-buffers)
    - [AIE buffers](#aie-buffers)
    - [AIE 버퍼](#aie-buffers)
  - [Buffer tiling and traversal order transformation](#buffer-tiling-and-traversal-order-transformation)
  - [버퍼 타일링 및 순회 순서 변환](#buffer-tiling-and-traversal-order-transformation)
  - [Kernel, number of calls and parameters](#kernel-number-of-calls-and-parameters)
  - [커널, 호출 수 및 매개변수](#kernel-number-of-calls-and-parameters)
  - [Data Movement Setters](#data-movement-setters)
  - [데이터 이동 설정자](#data-movement-setters)
    - [L3 <-> L2 transfers: `set_l3_to_l2_transfer`, `set_l2_to_l3_transfer`](#l3---l2-transfers-set_l3_to_l2_transfer-set_l2_to_l3_transfer)
    - [L3 <-> L2 전송: `set_l3_to_l2_transfer`, `set_l2_to_l3_transfer`](#l3---l2-transfers-set_l3_to_l2_transfer-set_l2_to_l3_transfer)
    - [L2 -> L1 transfers: `set_l2_to_l1_transfer`](#l2---l1-transfers-set_l2_to_l1_transfer)
    - [L2 -> L1 전송: `set_l2_to_l1_transfer`](#l2---l1-transfers-set_l2_to_l1_transfer)
    - [L1 -> L2 transfers: `set_l1_to_l2_transfer`](#l1---l2-transfers-set_l1_to_l2_transfer)
    - [L1 -> L2 전송: `set_l1_to_l2_transfer`](#l1---l2-transfers-set_l1_to_l2_transfer)
  - [Kernel Ratios](#kernel-ratios)
  - [커널 비율](#커널 비율)
  - [Tiling Script Design Tips](#tiling-script-design-tips)
  - [타일링 스크립트 디자인 팁](#tiling-script-design-tips)
    - [Validation and common pitfalls](#validation-and-common-pitfalls)
    - [검증 및 일반적인 함정](#validation-and-common-pitfalls)
- [Troubleshooting tiling](#troubleshooting-tiling)
- [타일링 문제 해결](#troubleshooting-tiling)
  - [Frequently encountered errors](#frequently-encountered-errors)
  - [자주 발생하는 오류](#frequently-encountered-errors)
    - [`IO Buffer 'mk[i][j].in[N]' is required to have a total size which is multiple of 4 bytes` (aiecompiler 77-23295)](#io-buffer-mkijinn-is-required-to-have-a-total-size-which-is-multiple-of-4-bytes-aiecompiler-77-23295)
    - [`IO 버퍼 'mk[i][j].in[N]'은 4바이트의 배수인 총 크기를 가져야 합니다`(aiecompiler 77-23295)](#io-buffer-mkijinn-is-required-to-have-a-total-size-which-is-multiple-of-4-bytes-aiecompiler-77-23295)
  - [Compiler flags and environment variables](#compiler-flags-and-environment-variables)
  - [컴파일러 플래그 및 환경 변수](#compiler-flags-and-environment-variables)
  - [Using Python printout](#using-python-printout)
  - [Python 출력 사용](#using-python-printout)
  - [Checking the produced data movement header](#checking-the-produced-data-movement-header)
  - [생성된 데이터 이동 헤더 확인](#checking-the-produced-data-movement-header)

## AIE Architecture in VAIML
## VAIML의 AIE 아키텍처

### AIE Components
### AIE 구성 요소

The AIE is organized as a grid composed of three components:
AIE는 세 가지 구성 요소로 구성된 그리드로 구성됩니다.
* AIE Cores, that perform actual arithmetic operations,
* 실제 산술 연산을 수행하는 AIE 코어,
* Memory tiles, with a larger capacity then individual AIE cores,
* 개별 AIE 코어보다 용량이 더 큰 메모리 타일,
* Data movement engines (also called DMA engines), that transfer data between cores, memory and the external memory.
* 코어, 메모리 및 외부 메모리 간에 데이터를 전송하는 데이터 이동 엔진(DMA 엔진이라고도 함).

Programming the AIE requires a kernel, which each AIE core executes, as well as configurations for each of the DMA engines.
AIE를 프로그래밍하려면 각 AIE 코어가 실행하는 커널과 각 DMA 엔진에 대한 구성이 필요합니다.
This guide will explain how to obtain both from an original kernel implementation.
이 가이드에서는 원래 커널 구현에서 두 가지를 모두 얻는 방법을 설명합니다.

### AIE topology and VAIML overlay
### AIE 토폴로지 및 VAIML 오버레이

The AIE architecture is seen as a distributed memory hierarchy: AIE cores as the innermost level of memory (L1); memory tiles as the second level (L2); DDR (external memory) as the third and outermost level (L3).
AIE 아키텍처는 분산 메모리 계층 구조로 간주됩니다. AIE 코어는 메모리의 가장 안쪽 수준(L1)입니다. 두 번째 레벨(L2)인 메모리 타일; 세 번째이자 가장 바깥쪽 레벨(L3)인 DDR(외부 메모리).

**Programming model:** The C++ code that is written for the kernel gets executed on AIE cores themselves.
**프로그래밍 모델:** 커널용으로 작성된 C++ 코드는 AIE 코어 자체에서 실행됩니다.
In VAIML, the same kernel code gets replicated on all the AIE cores of a grid.
VAIML에서는 동일한 커널 코드가 그리드의 모든 AIE 코어에 복제됩니다.
The kernel only has access to the L1 memory it is provided with when it is called.
커널은 호출될 때 제공되는 L1 메모리에만 액세스할 수 있습니다.
There is no communication channel with the rest of the AIE grid available to the AIE cores during kernel execution.
커널 실행 중에 AIE 코어에 사용 가능한 나머지 AIE 그리드와의 통신 채널이 없습니다.

**Data movement:** DMA engines perform the memory transfers between the three levels of the memory hierarchy. **The DMAs are not configurable at runtime; the compiler must determine all data movement statically.** In particular, the data transfers performed by the DMAs cannot depend on the data itself.
**데이터 이동:** DMA 엔진은 메모리 계층 구조의 세 가지 수준 간에 메모리 전송을 수행합니다. **DMA는 런타임에 구성할 수 없습니다. 컴파일러는 모든 데이터 이동을 정적으로 결정해야 합니다.** 특히 DMA에 의해 수행되는 데이터 전송은 데이터 자체에 의존할 수 없습니다.
Random access by the kernel is only possible within the L1 buffers.
커널에 의한 무작위 접근은 L1 버퍼 내에서만 가능합니다.

**DMA source and destination points:** DMAs support reading from / writing to various points in the AIE grid.
**DMA 소스 및 대상 지점:** DMA는 AIE 그리드의 다양한 지점에서 읽기/쓰기를 지원합니다.
In VAIML, the access patterns are configurable, but **the source/destination points of each DMA are fixed** and consistent across the entire model.
VAIML에서는 액세스 패턴을 구성할 수 있지만 **각 DMA의 소스/대상 지점은 고정**되어 전체 모델에서 일관됩니다.
The configuration of source/destination is fixed by an *overlay*.
소스/대상 구성은 *오버레이*로 고정됩니다.

**List of supported overlays:** The AIE architecture consist in one or more grids of processors, each of which has a 4x4 layout.
**지원되는 오버레이 목록:** AIE 아키텍처는 각각 4x4 레이아웃을 갖는 하나 이상의 프로세서 그리드로 구성됩니다.
Depending on the target hardware platform, three overlay variants are available:
대상 하드웨어 플랫폼에 따라 세 가지 오버레이 변형을 사용할 수 있습니다.

- **`rai_1x4x4`** -- used on **AIE2 (STX)** platforms.
- **`rai_1x4x4`** -- **AIE2(STX)** 플랫폼에서 사용됩니다.
- This lets the operator use one 4x4 grid, i.e.
- 이를 통해 운영자는 하나의 4x4 그리드를 사용할 수 있습니다.
- 16 processors, simultaneously.
- 동시에 16개의 프로세서.
- **`rai_2x4x4`** -- same as `rai_1x4x4` but with 2 stamps.
- **`rai_2x4x4`** -- `rai_1x4x4`와 동일하지만 스탬프가 2개 있습니다.
- Used on **AIE2 (STX)** platforms.
- **AIE2(STX)** 플랫폼에서 사용됩니다.
- This lets the operator use two 4x4 grids, i.e.
- 이를 통해 운영자는 두 개의 4x4 그리드를 사용할 수 있습니다.
- 32 processors, simultaneously.
- 동시에 32개의 프로세서.
- **`aie2p_6x4x4`** -- used on **AIE2PS** platforms.
- **`aie2p_6x4x4`** -- **AIE2PS** 플랫폼에서 사용됩니다.
- This lets the operator use six 4x4 grids, i.e.
- 이를 통해 운영자는 6개의 4x4 그리드를 사용할 수 있습니다.
- 96 processors, simultaneously.
- 96개의 프로세서를 동시에 지원합니다.

All overlays share the same 4x4 core-tile grid and the same connection topology between memtiles and core tiles.
모든 오버레이는 동일한 4x4 코어 타일 그리드와 멤타일과 코어 타일 간의 동일한 연결 토폴로지를 공유합니다.
The diagrams below show the connections for each overlay (one stamp):
아래 다이어그램은 각 오버레이(스탬프 1개)에 대한 연결을 보여줍니다.

| AIE2 (STX) overlay (`rai_1x4x4`)<br>AIE2(STX) 오버레이(`rai_1x4x4`) | AIE2PS overlay (`aie2p_6x4x4`, stamp 0)<br>AIE2PS 오버레이(`aie2p_6x4x4`, 스탬프 0) |
|---|---|
| ![AIE2 1x4x4 overlay](resources/AIE-1x4x4-overlay.svg)<br>![AIE2 1x4x4 오버레이](resources/AIE-1x4x4-overlay.svg) | ![AIE2PS 6x4x4 overlay](resources/AIE-6x4x4-overlay-stamp0.svg)<br>![AIE2PS 6x4x4 오버레이](resources/AIE-6x4x4-overlay-stamp0.svg) |

**L3 to L2 channels:** On the `rai_1x4x4` and `aie2p_6x4x4` overlays, there are six L3 to L2 channels that can be used for inputs; each pair of channels targets a different memory tile.
**L3~L2 채널:** `rai_1x4x4` 및 `aie2p_6x4x4` 오버레이에는 입력에 사용할 수 있는 6개의 L3~L2 채널이 있습니다. 각 채널 쌍은 서로 다른 메모리 타일을 대상으로 합니다.
That tile and its neighbor may be used as write addresses for L3 to L2 transfers; this means:
해당 타일과 그 이웃은 L3에서 L2로의 전송을 위한 쓰기 주소로 사용될 수 있습니다. 이는 다음을 의미합니다.

| Channel (on above schematic) | Allowed memtiles (`rai_1x4x4`) | Allowed memtiles (`aie2p_6x4x4`)
| 채널(위 회로도에 있음) | 허용되는 메모리(`rai_1x4x4`) | 허용되는 메모리(`aie2p_6x4x4`)
|------|-----------------------|-----------------------|
| 0, 1 | MT(0) or MT(1)<br>MT(0) 또는 MT(1)        | MT(0) or MT(1)<br>MT(0) 또는 MT(1)        |
| 2, 3 | MT(1), MT(2) or MT(3)<br>MT(1), MT(2) 또는 MT(3) | MT(1), MT(2) or MT(3)<br>MT(1), MT(2) 또는 MT(3) |
| 4, 5 | MT(2) or MT(3)<br>MT(2) 또는 MT(3)        | MT(1), MT(2) or MT(3)<br>MT(1), MT(2) 또는 MT(3) |

**L3 to L2 channels:** Two L2 to L3 channels are at your disposal for L2 to L3 transfers (from the AIE memory tiles back to DDR).
**L3 - L2 채널:** L2 - L3 전송(AIE 메모리 타일에서 다시 DDR로)을 위해 2개의 L2 - L3 채널을 사용할 수 있습니다.
On `rai_1x4x4`, `rai_2x4x4` and `aie2p_6x4x4`, these go from memory tile MT(2), and can read back from MT(1), MT(2) or MT(3).
`rai_1x4x4`, `rai_2x4x4` 및 `aie2p_6x4x4`에서는 메모리 타일 MT(2)에서 이동하고 MT(1), MT(2) 또는 MT(3)에서 다시 읽을 수 있습니다.

**L2 to L1 channels:** Eight channels are provided for L2 to L1 transfers, i.e. from memory tiles to individual AIE cores.
**L2에서 L1 채널로:** L2에서 L1로의 전송, 즉 메모리 타일에서 개별 AIE 코어로의 전송을 위해 8개의 채널이 제공됩니다.
Four of them target the four columns of the AIE, four target the four rows of the AIE stamp.
그 중 4개는 AIE의 4개 열을 대상으로 하고, 4개는 AIE 스탬프의 4개 행을 대상으로 합니다.

| Channel (on above schematic)<br>채널(위 회로도에 있음) | Physically connected tile<br>물리적으로 연결된 타일 | Data origin memory tile<br>데이터 원본 메모리 타일 | Target cores<br>대상 코어 |
|---|-------|-----------------|-----------------------|
| 0 | MT(1) | MT(0),  MT(1) or MT(2)<br>MT(0), MT(1) 또는 MT(2) | CT(0,0), CT(1,0), CT(2,0), CT(3,0)<br>CT(0,0), CT(1,0), CT(2,0), CT(3,0) |
| 1 | MT(1) | MT(0),  MT(1) or MT(2)<br>MT(0), MT(1) 또는 MT(2) | CT(0,1), CT(1,1), CT(2,1), CT(3,1)<br>CT(0,1), CT(1,1), CT(2,1), CT(3,1) |
| 2 | MT(1) | MT(0),  MT(1) or MT(2)<br>MT(0), MT(1) 또는 MT(2) | CT(0,2), CT(1,2), CT(2,2), CT(3,2)<br>CT(0,2), CT(1,2), CT(2,2), CT(3,2) |
| 3 | MT(1) | MT(0),  MT(1) or MT(2)<br>MT(0), MT(1) 또는 MT(2) | CT(0,3), CT(1,3), CT(2,3), CT(3,3)<br>CT(0,3), CT(1,3), CT(2,3), CT(3,3) |
| 4 | MT(3) | MT(2) or MT(3)<br>MT(2) 또는 MT(3) | CT(0,0), CT(0,1), CT(0,2), CT(0,3)<br>CT(0,0), CT(0,1), CT(0,2), CT(0,3) |
| 5 | MT(3) | MT(2) or MT(3)<br>MT(2) 또는 MT(3) | CT(1,0), CT(1,1), CT(1,2), CT(1,3)<br>CT(1,0), CT(1,1), CT(1,2), CT(1,3) |
| 6 | MT(3) | MT(2) or MT(3)<br>MT(2) 또는 MT(3) | CT(2,0), CT(2,1), CT(2,2), CT(2,3)<br>CT(2,0), CT(2,1), CT(2,2), CT(2,3) |
| 7 | MT(3) | MT(2) or MT(3)<br>MT(2) 또는 MT(3) | CT(3,0), CT(3,1), CT(3,2), CT(3,3)<br>CT(3,0), CT(3,1), CT(3,2), CT(3,3) |

These are broadcast channels: every target core of a channel receives the same data, and it is not possible to filter the data based on the intended destination core, or selectively enable writes.
이는 브로드캐스트 채널입니다. 채널의 모든 대상 코어는 동일한 데이터를 수신하며 의도한 대상 코어를 기반으로 데이터를 필터링하거나 선택적으로 쓰기를 활성화할 수 없습니다.

**L1 to L2 channels:** Four channels are provided, one per column, for the return of data from AIE cores back to the memory tiles.
**L1~L2 채널:** AIE 코어에서 메모리 타일로 데이터를 반환하기 위해 열당 하나씩 4개의 채널이 제공됩니다.
These all physically target memory tile MT(1), and can therefore write into memory tiles MT(0), MT(1) or MT(2).
이들은 모두 메모리 타일 MT(1)을 물리적으로 대상으로 하므로 메모리 타일 MT(0), MT(1) 또는 MT(2)에 쓸 수 있습니다.

| Channel (on above schematic)<br>채널(위 회로도에 있음) | Origin cores<br>오리진 코어 |
|---|-----------------------|
| 0 | CT(0,0), CT(1,0), CT(2,0), CT(3,0)<br>CT(0,0), CT(1,0), CT(2,0), CT(3,0) |
| 1 | CT(0,1), CT(1,1), CT(2,1), CT(3,1)<br>CT(0,1), CT(1,1), CT(2,1), CT(3,1) |
| 2 | CT(0,2), CT(1,2), CT(2,2), CT(3,2)<br>CT(0,2), CT(1,2), CT(2,2), CT(3,2) |
| 3 | CT(0,3), CT(1,3), CT(2,3), CT(3,3)<br>CT(0,3), CT(1,3), CT(2,3), CT(3,3) |

These channels benefit from a packet merging system: dispatching and addressing of the data on the L2 side is possible based on the origin core.
이러한 채널은 패킷 병합 시스템의 이점을 활용합니다. 즉, 원본 코어를 기반으로 L2 측의 데이터 디스패치 및 주소 지정이 가능합니다.
Therefore, these channels may be used as unicast from L2 to L1.
따라서 이러한 채널은 L2에서 L1까지 유니캐스트로 사용될 수 있습니다.

## Tiling process example
## 타일링 처리 예

### Step 1: decompose the problem; implement each tile in C++
### 1단계: 문제를 분해합니다. C++로 각 타일 구현

#### Purpose
#### 목적

This step covers the tiling of the C++ kernel itself.
이 단계에서는 C++ 커널 자체의 타일링을 다룹니다.
Its goal is to go from a basic version of the kernel to a tiled version, and to obtain the expression of which data one kernel call needs to compute one tile of output.
그 목표는 커널의 기본 버전에서 타일 버전으로 이동하고 하나의 커널 호출이 하나의 출력 타일을 계산하는 데 필요한 데이터 표현을 얻는 것입니다.
This abstract information is turned into a concrete script in Step 2 (see [Step 2: Allocate memory on the AIE grid and program data movement](#step-2-allocate-memory-on-the-aie-grid-and-program-data-movement)).
이 추상 정보는 2단계에서 구체적인 스크립트로 변환됩니다([2단계: AIE 그리드에 메모리 할당 및 프로그램 데이터 이동](#step-2-allocate-memory-on-the-aie-grid-and-program-data-movement) 참조).

This essentially consists in:
이는 기본적으로 다음으로 구성됩니다.
* applying loop tiling to the kernel,
* 커널에 루프 타일링을 적용하고,
* extracting the inner loops to describe one tile, and wrap it into an AIE kernel,
* 하나의 타일을 설명하기 위해 내부 루프를 추출하고 이를 AIE 커널로 래핑합니다.
* determining the "tiling pattern", which is a symbolic relation between a tile of output and tiles of input.
* 출력 타일과 입력 타일 사이의 상징적 관계인 "타일링 패턴"을 결정합니다.

From this 1-dimensional, single-channel convolution:
이 1차원 단일 채널 컨볼루션에서:
```cpp
static inline void conv1d(
  const float* __restrict in, // Input feature map (IFM)
  const float* __restrict wts, // Kernel (weights)
  float* __restrict out, // Output feature map (OFM)
  std::size_t L_in, // Length of the IFM (elements)
  std::size_t K // Kernel size
) {
  const std::size_t L_out = L_in - K + 1;
  for (std::size_t o = 0; o < L_out; ++o) {
    float acc = 0.f;
    for (std::size_t k = 0; k < K; ++k)
      acc += in[o + k] * kernel[k];
    out[o] = acc;
  }
}
```

one wants to get the following:
우리는 다음을 얻고 싶어합니다:
```cpp
// Size of the lp_params that the tiling defines:
//   [0] = T  -- tile size (number of output elements per kernel call)
//   [1] = K  -- kernel (filter) size
//   [2] = calls_per_wts -- number of kernel calls during which the
//                          asynchronous WTS buffer must stay live
static constexpr int conv1d_kernel_lp_size = 3;

template <typename dtype_ifm, typename dtype_wts, typename dtype_ofm>
__attribute__((noinline)) void conv1d_kernel(
    adf::input_buffer_conf<dtype_ifm, adf::bpc_sync_0d> &__restrict ifm,
    adf::input_buffer_conf<dtype_wts, adf::bpc_async_0d> &__restrict wts,
    adf::output_buffer_conf<dtype_ofm, adf::bpc_sync_0d> &__restrict ofm,
    const uint32_t (&lp_params)[conv1d_kernel_lp_size]) {

  const uint32_t T = lp_params[0];
  const uint32_t K = lp_params[1];
  const uint32_t calls_per_wts = lp_params[2];

  static uint16_t core_row = aie::tile::current().id().row % 4;
  const uint32_t ifm_offset = T * core_row;

  // Buffer acquisition: the WTS buffer is acquired on the first call after a rotation.
  static uint32_t wts_call_count = 0;
  if (wts_call_count == 0) {
    wts.acquire();
  }

  auto *ifm_data = (dtype_ifm *__restrict)(ifm.data());
  auto *wts_data = (dtype_wts *__restrict)(wts.data());
  auto *ofm_data = (dtype_ofm *__restrict)(ofm.data());

  // Actual computation: one tile of output is computed for each core.
  for (uint32_t j = 0; j < T; ++j) {
    float acc = 0.f;
    for (uint32_t k = 0; k < K; ++k) {
      acc += float(ifm_data[ifm_offset + j + k]) * float(wts_data[k]);
    }
    ofm_data[j] = dtype_ofm(acc);
  }

  // Buffer release: the WTS buffer is released after the last call.
  wts_call_count += 1;
  if (wts_call_count == calls_per_wts) {
    wts.release();
    wts_call_count = 0;
  }
}
```
as well as the following data flow information:
다음 데이터 흐름 정보도 포함됩니다.

```
 * Output tiles (buffer `ofm`) of size T
 * Tiles are indexed from i = 0 to i = L_out / T - 1
 * Input tiles (buffer `ifm`) of size T + K
 * For the tile indexed by i:
   * The tile of output produced is out[T * i] to out[T * (i+1) - 1],
   * One tile of input is needed, from in[T * i] to in[T * (i+1) + K - 1].
 * Kernel weights (buffer `wts`) of size K, need to stay unchanged across all kernel calls.
```

#### Apply loop tiling to the kernel
#### 커널에 루프 타일링 적용

Most kernels that we seek to map to the AI Engine consist of loop nests iterating over certain arrays, applying a mathematical function at each iteration.
우리가 AI 엔진에 매핑하려는 대부분의 커널은 특정 배열을 반복하는 루프 중첩으로 구성되어 각 반복마다 수학 함수를 적용합니다.
In most cases, these loops can be broken down piece-wise into an inner loop that describes a piece, and an outer loop that goes across pieces.
대부분의 경우 이러한 루프는 조각을 설명하는 내부 루프와 조각을 가로지르는 외부 루프로 조각별로 나눌 수 있습니다.
Such pieces are called tiles, and the process is called loop tiling.
이러한 조각을 타일이라고 하며, 그 과정을 루프 타일링이라고 합니다.

A simple C++ implementation of the kernel that runs on the CPU is an easy starting point.
CPU에서 실행되는 커널의 간단한 C++ 구현은 쉬운 시작점입니다.
For example: one can tile the following 1-dimensional convolution:
예를 들어 다음과 같은 1차원 컨볼루션을 타일링할 수 있습니다.
```cpp
static inline void conv1d_kernel(
  const float* __restrict in, // Input feature map (IFM)
  const float* __restrict kernel, // Kernel (weights)
  float* __restrict out, // Output feature map (OFM)
  std::size_t L_in, // Length of the IFM (elements)
  std::size_t K // Kernel size
) {
  const std::size_t L_out = L_in - K + 1;
  for (std::size_t o = 0; o < L_out; ++o) {
    float acc = 0.f;
    for (std::size_t k = 0; k < K; ++k)
      acc += in[o + k] * kernel[k];
    out[o] = acc;
  }
}
```
by splitting the `L_out` dimension into chunks of size `T`:
'L_out' 차원을 'T' 크기의 덩어리로 분할하여:
```cpp
static inline void conv1d_kernel(
  const float* __restrict in, // Input feature map (IFM)
  const float* __restrict kernel, // Kernel (weights)
  float* __restrict out, // Output feature map (OFM)
  std::size_t L_in, // Length of the IFM (elements)
  std::size_t K, // Kernel size
  std::size_t T // OFM tile size
) {
  // K is the kernel size
  // L_in is the input length
  const std::size_t L_out = L_in - K + 1;
  for (std::size_t i = 0; i < L_out / T; ++i) {
    for (std::size_t j = 0; j < T; ++j) {
      std::size_t o = T * i + j;
      float acc = 0.f;
      for (std::size_t k = 0; k < K; ++k)
        acc += in[o + k] * kernel[k];
      out[o] = acc;
    }
  }
}
```

This breakdown can be visualized as the breakdown of the output feature map into chunks of size T each.
이 분석은 출력 특징 맵을 각각 T 크기의 청크로 분석하여 시각화할 수 있습니다.
Notably, it can be seen that each OFM tile of size `T` needs an IFM window of width `T + K - 1`; this relation determine how much data needs to live in each AIE core for computations to happen.
특히, 'T' 크기의 각 OFM 타일에는 'T + K - 1' 너비의 IFM 창이 필요하다는 것을 알 수 있습니다. 이 관계는 계산을 수행하기 위해 각 AIE 코어에 얼마나 많은 데이터가 있어야 하는지를 결정합니다.

![OFM partitioned into output tiles of width T](resources/1d-conv-ofm-tiles.svg)

#### Extract key information about the tile's inputs and outputs
#### 타일의 입력 및 출력에 대한 주요 정보 추출

Each AI Engine core will compute one output tile from the above obtained set of tiles.
각 AI Engine 코어는 위에서 얻은 타일 세트에서 하나의 출력 타일을 계산합니다.
The outer loop is therefore going to be parallel: multiple tiles will be computed at once.
따라서 외부 루프는 병렬이 됩니다. 즉, 여러 타일이 한 번에 계산됩니다.
```cpp
static inline void conv1d_outer_loops(
  const float* __restrict in, // Input feature map (IFM)
  const float* __restrict kernel, // Kernel (weights)
  float* __restrict out, // Output feature map (OFM)
  std::size_t L_in, // Length of the IFM (elements)
  std::size_t K, // Kernel size
  std::size_t T // OFM tile size
) {
  const std::size_t L_out = L_in - K + 1;
  for (std::size_t i = 0; i < L_out / T; ++i) {
    // Compute tile number i
    conv1d_kernel(in, kernel, out, L_in, K, i);
  }
}
```

The inner loop computes one output tile.
내부 루프는 하나의 출력 타일을 계산합니다.
It is what each AIE core will execute:
각 AIE 코어가 실행하는 작업은 다음과 같습니다.
```cpp
static inline void conv1d_kernel(
  const float* __restrict in, // Input feature map (IFM)
  const float* __restrict kernel, // Kernel (weights)
  float* __restrict out, // Output feature map (OFM)
  std::size_t K, // Kernel size
  std::size_t T, // OFM tile size
  std::size_t i // Tile index
) {
  for (std::size_t j = 0; j < T; ++j) {
    std::size_t o = T * i + j;
    float acc = 0.f;
    for (std::size_t k = 0; k < K; ++k)
      acc += in[o + k] * kernel[k];
    out[o] = acc;
  }
}
```

**Bufferization:** In the above code, the `in` and `out` arrays are directly indexed from the AIE core.
**버퍼화:** 위 코드에서 'in' 및 'out' 배열은 AIE 코어에서 직접 인덱싱됩니다.
Per the architecture section (see [AIE Architecture in VAIML](#aie-architecture-in-vaiml)), the AIE cores cannot make direct access to external memories.
아키텍처 섹션([VAIML의 AIE 아키텍처](#aie-architecture-in-vaiml) 참조)에 따라 AIE 코어는 외부 메모리에 직접 액세스할 수 없습니다.
If `in` and `out` are very large arrays, the above code will not work on the AIE.
'in'과 'out'이 매우 큰 배열인 경우 위 코드는 AIE에서 작동하지 않습니다.

The solution is to compute the output tile in a local buffer whose size is small, and likewise retrieve a small window of the input, that is only as large as required to compute the output tile.
해결책은 크기가 작은 로컬 버퍼에서 출력 타일을 계산하고 마찬가지로 출력 타일을 계산하는 데 필요한 만큼만 큰 입력의 작은 창을 검색하는 것입니다.

From the example above, one can determine that the input required to compute one tile is the `K` elements of the kernel, and a window of `T + K` consecutive elements from `in`.
위의 예에서 하나의 타일을 계산하는 데 필요한 입력은 커널의 `K` 요소이고 `in`의 `T + K` 연속 요소의 창임을 확인할 수 있습니다.

**Relation between output and input tiles:** To compute one tile of the output, the kernel needs a specific tile of the input.
**출력 타일과 입력 타일 간의 관계:** 출력의 타일 하나를 계산하려면 커널에 입력의 특정 타일이 필요합니다.
This relation is computed by analyzing the tiled loops, and answering the following questions:
이 관계는 타일 루프를 분석하고 다음 질문에 답하여 계산됩니다.
* What is the output indexed on?
* 출력은 무엇에 색인되어 있습니까?
* When the loop indexing the output traverses one tile, which loops are also iterated on?
* 출력을 인덱싱하는 루프가 하나의 타일을 통과할 때 어떤 루프도 반복됩니까?
* Throughout the loops traversed to compute one output tile, which inputs are accessed?
* 하나의 출력 타일을 계산하기 위해 통과하는 루프 전체에서 어떤 입력에 액세스합니까?

From the example above, to compute `out[T * i + j]` from `j = 0` to `j = T-1`, the window of elements from `in` that is required is `in[T * i]` to `in[T * (i + 1) + K]`.
위의 예에서 `j = 0`에서 `j = T-1`까지 `out[T * i + j]`를 계산하려면 필요한 `in`에서 요소의 창은 `in[T * i]`에서 `in[T * (i + 1) + K]`입니다.

With this relation determined, the outer loop can be expressed as memory copies:
이 관계가 결정되면 외부 루프는 메모리 복사본으로 표현될 수 있습니다.
```cpp
static inline void conv1d_outer_loops(
  const float* __restrict in, // Input feature map (IFM)
  const float* __restrict kernel, // Kernel (weights)
  float* __restrict out, // Output feature map (OFM)
  std::size_t L_in, // Length of the IFM (elements)
  std::size_t K, // Kernel size
  std::size_t T // OFM tile size
) {
  float *in_buf = (float *)malloc((T + K) * sizeof(float));
  float *out_buf = (float *)malloc(T * sizeof(float));
  for (std::size_t i = 0; i < L_out / T; ++i) {
    // Copy one tile of input (in[T * i] -> in[T * (i+1) + K - 1])
    memcpy(in_buf, in + (T * i), (T + K) * sizeof(float));
    // Perform the computation
    conv1d_kernel(in_buf, kernel, out_buf, L_in, K, T, i);
    // Copy one tile of output (out[T * i] -> out[T * (i+1) - 1])
    memcpy(out + (T * i), T * sizeof(float));
  }
}
```
From this outer loop, we can obtain the following data flow information:
이 외부 루프에서 다음과 같은 데이터 흐름 정보를 얻을 수 있습니다.
```
 * Input tiles of size T + K
 * Kernel weights of size K.
 * Output tiles of size T
 * Tiles are indexed from i = 0 to i = L_out / T - 1
 * For the tile indexed by i:
   * The tile of output produced is out[T * i] to out[T * (i+1) - 1],
   * One tile of input is needed, from in[T * i] to in[T * (i+1) + K - 1].
```

We have extracted sufficient information from the outer loop to compute the buffer allocation and data movement on the AI Engine. **The outer loop never gets compiled to AIE binary**, and will no longer be used from this point on in the tutorial.
AI 엔진의 버퍼 할당 및 데이터 이동을 계산하기 위해 외부 루프에서 충분한 정보를 추출했습니다. **외부 루프는 AIE 바이너리로 컴파일되지 않으며** 튜토리얼의 이 시점부터는 더 이상 사용되지 않습니다.

The kernel is adapted to work locally on the buffers to produce its output, removing the need for the `L_in` argument:
커널은 출력을 생성하기 위해 버퍼에서 로컬로 작동하도록 조정되어 'L_in' 인수가 필요하지 않습니다.
```cpp
static inline void conv1d_kernel(
  const float* __restrict in, // Input feature map (IFM)
  const float* __restrict kernel, // Kernel (weights)
  float* __restrict out, // Output feature map (OFM)
  std::size_t K, // Kernel size
  std::size_t T // OFM tile size
) {
  for (std::size_t j = 0; j < T; ++j) {
    float acc = 0.f;
    for (std::size_t k = 0; k < K; ++k)
      acc += in[j + k] * kernel[k];
    out[j] = acc;
  }
}
```

#### Create an AIE kernel with the right prototype
#### 올바른 프로토타입으로 AIE 커널 생성

The 1-dimensional convolution takes two input buffers.
1차원 컨볼루션은 두 개의 입력 버퍼를 사용합니다.
It therefore needs to have the following prototype (see [Kernel interface](#kernel-interface)):
따라서 다음과 같은 프로토타입이 필요합니다([커널 인터페이스](#kernel-interface) 참조).
```cpp
// 2 input parameters: K (Kernel size), T (OFM tile size)
static constexpr int conv1d_kernel_lp_size = 2;

template <typename dtype_ifm, typename dtype_wts, typename dtype_ofm>
__attribute__((noinline)) void conv1d_kernel(
  adf::input_buffer_conf<dtype_ifm, adf::bpc_sync_0d> &__restrict ifm,
  adf::input_buffer_conf<dtype_wts, adf::bpc_sync_0d> &__restrict wts,
  adf::output_buffer_conf<dtype_ofm, adf::bpc_sync_0d> &__restrict ofm,
  const uint32_t (&lp_params)[conv1d_kernel_lp_size]) {
```

which means that this function:
이는 이 기능이 다음을 의미합니다.
```cpp
static inline void conv1d_kernel(
  const float* __restrict in, // Input feature map (IFM)
  const float* __restrict kernel, // Kernel (weights)
  float* __restrict out, // Output feature map (OFM)
  std::size_t K, // Kernel size
  std::size_t T // OFM tile size
) {
  for (std::size_t j = 0; j < T; ++j) {
    float acc = 0.f;
    for (std::size_t k = 0; k < K; ++k)
      acc += in[j + k] * kernel[k];
    out[j] = acc;
  }
}
```
becomes:
다음과 같이 됩니다:
```cpp
// 2 input parameters: K (Kernel size), T (OFM tile size)
static constexpr int conv1d_kernel_lp_size = 2;

template <typename dtype_ifm, typename dtype_wts, typename dtype_ofm>
__attribute__((noinline)) void conv1d_kernel(
  adf::input_buffer_conf<dtype_ifm, adf::bpc_sync_0d> &__restrict ifm,
  adf::input_buffer_conf<dtype_wts, adf::bpc_sync_0d> &__restrict wts,
  adf::output_buffer_conf<dtype_ofm, adf::bpc_sync_0d> &__restrict ofm,
  const uint32_t (&lp_params)[conv1d_kernel_lp_size]) {
{
  // Buffers allocated from the outside
  auto *in = (float *__restrict)(ifm.data());
  auto *kernel = (float *__restrict)(wts.data());
  auto *out = (float *__restrict)(ofm.data());

  // Former numeric arguments to conv1d_kernel
  const uint32_t T = lp_params[0];
  const uint32_t K = lp_params[1];

  // Actual computation: one tile of output is computed for each core.
  for (std::size_t j = 0; j < T; ++j) {
    float acc = 0.f;
    for (std::size_t k = 0; k < K; ++k)
      acc += in[j + k] * kernel[k];
    out[o] = acc;
  }
}
```

The data flow information is augmented with the buffer names:
데이터 흐름 정보는 버퍼 이름으로 확장됩니다.
```
 * Input tiles (buffer `ifm`) of size T + K
 * Kernel weights (buffer `wts`) of size K.
 * Output tiles (buffer `ofm`) of size T
 * Tiles are indexed from i = 0 to i = L_out / T - 1
 * For the tile indexed by i:
   * The tile of output produced is out[T * i] to out[T * (i+1) - 1],
   * One tile of input is needed, from in[T * i] to in[T * (i+1) + K - 1].
```

#### Optimization: re-use weights across calls
#### 최적화: 호출 전반에 걸쳐 가중치 재사용

In the above kernel, all the buffer arguments are *synchronous*, i.e. their mode of operation is `adf::bpc_sync_0d`:
위 커널에서 모든 버퍼 인수는 *동기*입니다. 즉, 작동 모드는 `adf::bpc_sync_0d`입니다.
```cpp
  adf::input_buffer_conf<float, adf::bpc_sync_0d> &__restrict ifm,
  adf::input_buffer_conf<float, adf::bpc_sync_0d> &__restrict wts,
  adf::output_buffer_conf<float, adf::bpc_sync_0d> &__restrict ofm,
```
The buffers `ifm`, `wts` and `ofm` are in principle double-buffers: during a kernel call, the AIE core operates on one half of the buffer, while the other half contains next tile inputs (or previous tile results for `ofm`).
`ifm`, `wts` 및 `ofm` 버퍼는 원칙적으로 이중 버퍼입니다. 커널 호출 중에 AIE 코어는 버퍼의 절반에서 작동하고 나머지 절반에는 다음 타일 입력(또는 `ofm`에 대한 이전 타일 결과)이 포함됩니다.

In synchronous mode, the buffers are rotated on every call: one needs to read the weights from DDR on subsequent calls.
동기 모드에서는 버퍼가 모든 호출에서 회전됩니다. 즉, 후속 호출에서 DDR의 가중치를 읽어야 합니다.
Since the weights are the same for every tile of OFM, reading them more than once is redundant.
OFM의 모든 타일에 대한 가중치는 동일하므로 두 번 이상 읽는 것은 중복됩니다.
To maximize I/O efficiency, the `wts` input buffer should stay unchanged across kernel calls.
I/O 효율성을 최대화하려면 'wts' 입력 버퍼가 커널 호출 전반에 걸쳐 변경되지 않고 유지되어야 합니다.

To make this happen, we need to:
이를 실현하려면 다음을 수행해야 합니다.
* change the mode of operation of the `wts` argument to `adf::bpc_async_0d`:
* `wts` 인수의 작동 모드를 `adf::bpc_async_0d`로 변경합니다.
```cpp
  adf::input_buffer_conf<dtype_wts, adf::bpc_async_0d> &__restrict wts,
```
* provide the number of kernel calls (see [Kernel Ratios](#kernel-ratios)) across which buffer rotation does not happen, and after which a rotation will happen; this adds an additional entry into `lp_params`, whose size becomes 3:
* 버퍼 회전이 발생하지 않고 그 이후에 회전이 발생하는 커널 호출 수([커널 비율](#kernel-ratios) 참조)를 제공합니다. 그러면 크기가 3이 되는 `lp_params`에 추가 항목이 추가됩니다.
```cpp
// 3 input parameters: K (Kernel size), T (OFM tile size), calls_per_wts (number of calls to the kernel)
static constexpr int conv1d_kernel_lp_size = 3;

// template <typename dtype_ifm, typename dtype_wts, typename dtype_ofm>
// __attribute__((noinline)) void conv1d_kernel(...) {
// [...]
  const uint32_t calls_per_wts = lp_params[2];
```
The value of `calls_per_wts` is the total number of calls to the kernel.
`calls_per_wts` 값은 커널에 대한 총 호출 수입니다.
It is dependent on the size of the input.
입력 크기에 따라 다릅니다.
It will be computed in the tiling script.
타일링 스크립트에서 계산됩니다.
* acquire the buffer on the first call, and release it on the last call:
* 첫 번째 호출에서 버퍼를 획득하고 마지막 호출에서 해제합니다.
```cpp
// template <typename dtype_ifm, typename dtype_wts, typename dtype_ofm>
// __attribute__((noinline)) void conv1d_kernel(...) {
    // Buffer acquisition: the WTS buffer is acquired on the first call after a rotation.
    static uint32_t wts_call_count = 0;
    if (wts_call_count == 0) {
      wts.acquire();
    }

    // Get the buffer pointers _after_ acquisition
    auto *ifm_data = (dtype_ifm *__restrict)(ifm.data());
    auto *wts_data = (dtype_wts *__restrict)(wts.data());
    auto *ofm_data = (dtype_ofm *__restrict)(ofm.data());

    // Actual computation: one tile of output is computed for each core.
    // [...]

    // Buffer release: the WTS buffer is released after the last call.
    wts_call_count += 1;
    if (wts_call_count == calls_per_wts) {
        wts.release();
        wts_call_count = 0;
    }
}
```

This gives the final kernel:
이것은 최종 커널을 제공합니다:
```cpp
// 3 input parameters: K (Kernel size), T (OFM tile size), calls_per_wts (number of calls to the kernel)
static constexpr int conv1d_kernel_lp_size = 3;

template <typename dtype_ifm, typename dtype_wts, typename dtype_ofm>
__attribute__((noinline)) void conv1d_kernel(
    adf::input_buffer_conf<dtype_ifm, adf::bpc_sync_0d> &__restrict ifm,
    adf::input_buffer_conf<dtype_wts, adf::bpc_async_0d> &__restrict wts,
    adf::output_buffer_conf<dtype_ofm, adf::bpc_sync_0d> &__restrict ofm,
    const uint32_t (&lp_params)[conv1d_kernel_lp_size]) {

    // Former numeric arguments to conv1d_kernel
    const uint32_t T = lp_params[0];
    const uint32_t K = lp_params[1];
    const uint32_t calls_per_wts = lp_params[2];

    // Buffer acquisition: the WTS buffer is acquired on the first call after a rotation.
    static uint32_t wts_call_count = 0;
    if (wts_call_count == 0) {
      wts.acquire();
    }

    // Get the buffer pointers _after_ acquisition
    auto *ifm_data = (dtype_ifm *__restrict)(ifm.data());
    auto *wts_data = (dtype_wts *__restrict)(wts.data());
    auto *ofm_data = (dtype_ofm *__restrict)(ofm.data());

    // Actual computation: one tile of output is computed for each core.
    for (std::size_t j = 0; j < T; ++j) {
        float acc = 0.f;
        for (std::size_t k = 0; k < K; ++k)
            acc += in[j + k] * kernel[k];
        out[o] = acc;
    }

    // Buffer release: the WTS buffer is released after the last call.
    wts_call_count += 1;
    if (wts_call_count == calls_per_wts) {
        wts.release();
        wts_call_count = 0;
    }
}
```

and the asynchronous buffer information is added to the data flow relation:
비동기 버퍼 정보가 데이터 흐름 관계에 추가됩니다.
```
 * Input tiles (buffer `ifm`) of size T + K
 * Kernel weights (buffer `wts`) of size K, need to stay unchanged across all kernel calls.
 * Output tiles (buffer `ofm`) of size T
 * Tiles are indexed from i = 0 to i = L_out / T - 1
 * For the tile indexed by i:
   * The tile of output produced is out[T * i] to out[T * (i+1) - 1],
   * One tile of input is needed, from in[T * i] to in[T * (i+1) + K - 1].
```

### Step 2: Allocate memory on the AIE grid and program data movement
### 2단계: AIE 그리드에 메모리 할당 및 데이터 이동 프로그램

Step 1 has created a kernel and obtained data flow information that apply to each individual tiles.
1단계에서는 커널을 생성하고 각 개별 타일에 적용되는 데이터 흐름 정보를 얻었습니다.
The AIE consists of a grid of processors with determined communication channels (see [AIE Architecture in VAIML](#aie-architecture-in-vaiml)).
AIE는 결정된 통신 채널을 갖춘 프로세서 그리드로 구성됩니다([VAIML의 AIE 아키텍처](#aie-architecture-in-vaiml) 참조).

The next step is to program the data movement on the AI Engine.
다음 단계는 AI 엔진에서 데이터 이동을 프로그래밍하는 것입니다.
To that effect, one needs to write a script called the *tiling script*, taking into account the AIE's topology specifics (see [AIE topology and VAIML overlay](#aie-topology-and-vaiml-overlay)).
이를 위해서는 AIE의 토폴로지 세부 사항을 고려하여 *타일링 스크립트*라는 스크립트를 작성해야 합니다([AIE 토폴로지 및 VAIML 오버레이](#aie-topology-and-vaiml-overlay) 참조).

To build such a tiling script, we need to:
이러한 타일링 스크립트를 작성하려면 다음을 수행해야 합니다.
* select a AIE core for each tile:
* 각 타일에 대해 AIE 코어를 선택합니다.
* determine which data is needed to feed *all the cores* in the AIE grid,
* AIE 그리드의 *모든 코어*에 공급하는 데 필요한 데이터를 결정하고,
* configure the DMAs to transfer data in and out of the AIE memory tiles and cores.
* AIE 메모리 타일 및 코어 안팎으로 데이터를 전송하도록 DMA를 구성합니다.

#### Abstract calculation of data movement
#### 데이터 이동의 추상 계산

Before implementing the script, we need to calculate the data movement and memory allocation using the data flow information.
스크립트를 구현하기 전에 데이터 흐름 정보를 이용하여 데이터 이동 및 메모리 할당을 계산해야 합니다.

From our kernel and outer loops, we determined the following data flow information:
커널 및 외부 루프에서 다음과 같은 데이터 흐름 정보를 결정했습니다.
```
 * Input tiles (buffer `ifm`) of size T + K
 * Kernel weights (buffer `wts`) of size K, need to stay unchanged across all kernel calls.
 * Output tiles (buffer `ofm`) of size T
 * Tiles are indexed from i = 0 to i = L_out / T - 1
 * For the tile indexed by i:
   * The tile of output produced is out[T * i] to out[T * (i+1) - 1],
   * One tile of input is needed, from in[T * i] to in[T * (i+1) + K - 1].
```

Producing `out[T * i]` to `out[T * (i+1) - 1]` requires:
`out[T * i]`를 `out[T * (i+1) - 1]`로 생성하려면 다음이 필요합니다.
* the entire kernel `kernel[0]` to `kernel[K-1]`, and
* 전체 커널 `kernel[0]`을 `kernel[K-1]`로, 그리고
* the inputs `in[T * i]` to `in[T * (i+1) + K - 1]`.
* `in[T * i]`를 `in[T * (i+1) + K - 1]`에 입력합니다.

We now focus on the `in` input distribution.
이제 'in' 입력 분포에 중점을 둡니다.

The `in` tensor goes through the IFM channel.
'in' 텐서는 IFM 채널을 통과합니다.
According to the overlay (see [AIE topology and VAIML overlay](#aie-topology-and-vaiml-overlay)), IFMs are broadcast along columns of 4 AIE cores: this means that along an AIE column, all the AIE cores will receive the same input data, regardless of how much of it they actually use.
오버레이([AIE 토폴로지 및 VAIML 오버레이](#aie-topology-and-vaiml-overlay) 참조)에 따르면 IFM은 4개의 AIE 코어 열을 따라 브로드캐스트됩니다. 즉, AIE 열을 따라 모든 AIE 코어는 실제로 사용하는 데이터의 양에 관계없이 동일한 입력 데이터를 수신합니다.

The IFM broacasting scheme is shown in blue in the following figure:
IFM 방송 구성표는 다음 그림에서 파란색으로 표시됩니다.

![Column 0 IFM broadcast + per-core OFM tiles for 1-D conv (k=3)](resources/AIE-1x4x4-overlay-1d-conv-lanes.svg)

Two consecutive tiles `i` and `i+1` require the following input:
두 개의 연속 타일 'i' 및 'i+1'에는 다음 입력이 필요합니다.
* Tile `i`: `in[T * i]` to `in[T * (i+1) + K - 1]`
* 타일 `i`: `in[T * i]`에서 `in[T * (i+1) + K - 1]`로
* Tile `i+1`: `in[T * (i+1)]` to `in[T * (i+2) + K - 1]`
* 타일 `i+1`: `in[T * (i+1)]` ~ `in[T * (i+2) + K - 1]`

These two inputs overlap on the `k` elements `in[T * (i+1)]` to `in[T * (i+1) + K - 1]`.
이 두 입력은 `k` 요소 `in[T * (i+1)]`에서 `in[T * (i+1) + K - 1]`까지 겹칩니다.
By broadcasting `in[T * i]` to `in[T * (i+2) + K - 1]` through the IFM channel, we satisfy the input requirements of both tiles.
IFM 채널을 통해 `in[T * i]`를 `in[T * (i+2) + K - 1]`로 브로드캐스팅함으로써 두 타일의 입력 요구 사항을 모두 충족합니다.
Since each column has 4 AIE cores, it is sufficient to broadcast `in[T * i]` to `in[T * (i+4) + K - 1]` through so that each core can process its tile, which makes a total volume of `4*T + k` elements.
각 열에는 4개의 AIE 코어가 있으므로 `in[T * i]`를 `in[T * (i+4) + K - 1]`로 브로드캐스트하면 각 코어가 타일을 처리할 수 있으므로 총 `4*T + k` 요소 볼륨이 됩니다.

The cores' respective OFMs are then concatenated to obtain the final output, as is shown in green on the schematic; there is no broadcast of the OFMs.
그러면 회로도에 녹색으로 표시된 것처럼 코어의 각 OFM이 연결되어 최종 출력을 얻습니다. OFM은 방송되지 않습니다.

Let's now generalize this scheme to 4 columns:
이제 이 구성표를 4개 열로 일반화해 보겠습니다.
| Column<br>칼럼 | Tiles produced<br>생산된 타일 |
| --- | --- |
| 0 | 0, 1, 2, 3 |
| 1 | 4, 5, 6, 7 |
| 2 | 8, 9, 10, 11 |
| 3 | 12, 13, 14, 15 |

Should there be more than 16 tiles, this pattern will be repeated with tile indices modulo 16: the general formula is that cores will be assigned tiles `i % 16`, `(i + 1) % 16`, up to `(i + 15) % 16`.
타일이 16개보다 많으면 이 패턴은 타일 인덱스 모듈로 16으로 반복됩니다. 일반 공식은 코어에 `i % 16`, `(i + 1) % 16`, 최대 `(i + 15) % 16` 타일이 할당된다는 것입니다.

We now have enough information to determine the **entire data movement on the AIE**:
이제 **AIE의 전체 데이터 이동**을 결정하는 데 충분한 정보가 있습니다.

Each 4x4 grid produces **16 consecutive tiles** of the output:
각 4x4 그리드는 출력의 **16개 연속 타일**을 생성합니다.
* Each column (numbered `C`) needs `4*T + k` elements, starting from `in[T * (i + 4*C)]` to `in[T * (i + 4*C + 4) + K - 1]`.
* 각 열(`C` 번호)에는 `in[T * (i + 4*C)]`에서 시작하여 `in[T * (i + 4*C + 4) + K - 1]`까지 `4*T + k` 요소가 필요합니다.
* In total, to process tiles `i` through `i + 15`, we need to transfer elements `in[T * i]` to `in[T * (i + 16) + K - 1]` into the AIE.
* 전체적으로 `i`부터 `i + 15`까지 타일을 처리하려면 `in[T * i]`에서 `in[T * (i + 16) + K - 1]` 요소를 AIE로 전송해야 합니다.
* Tiles `i` through `i + 15` produce elements `out[T * i]` to `out[T * (i + 16) - 1]` (a total of `T * 16` elements).
* `i`부터 `i + 15`까지 타일은 `out[T * i]`부터 `out[T * (i + 16) - 1]`까지의 요소(총 `T * 16` 요소)를 생성합니다.

#### Writing the tiling script
#### 타일링 스크립트 작성

Let's write a script that configures the DMA transfers based on the information computed above, using Tensor Expressions.
Tensor Expressions를 사용하여 위에서 계산된 정보를 기반으로 DMA 전송을 구성하는 스크립트를 작성해 보겠습니다.
The full script lives in `tutorial/conv1d/0_distributed/custom_op_conv1d/custom_conv1d_tiling.py`.
전체 스크립트는 `tutorial/conv1d/0_distributed/custom_op_conv1d/custom_conv1d_tiling.py`에 있습니다.

* Unpack the op interface and name the overlay constants.
* op 인터페이스의 압축을 풀고 오버레이 상수의 이름을 지정합니다.
* The tile size `T` is set here.
* 타일 ​​크기 `T`가 여기에서 설정됩니다.
```py
    ifm, wts, ofm = opInterface

    OVERLAY_COLS = 4
    OVERLAY_ROWS = 4
    N_CORES = OVERLAY_COLS * OVERLAY_ROWS

    # Tile size (one core's OFM contribution).
    T = 64
```

* Get the stamp to be configured:
* 구성할 스탬프를 가져옵니다.
```py
    nb_stamps = len(tiling)
    # Configure each stamp.
    for stamp in tiling:
        spu = stamp[0]
    # or configure only one stamp:
    spu = tiling[0][0]
```

* Get the `L_in` and `K` parameters, as well as `L_out` (length of the output).
* 'L_in' 및 'K' 매개변수와 'L_out'(출력 길이)을 가져옵니다.
* We get *padded* tensors (the compiler pads the innermost ONNX dim to a 32-bit boundary), and check the consistency of the input and output lengths:
* *패딩된* 텐서를 얻고(컴파일러는 가장 안쪽 ONNX 치수를 32비트 경계로 패딩함) 입력 및 출력 길이의 일관성을 확인합니다.
```py
    (L_in,) = ifm.getPaddedShape()
    (K,) = wts.getPaddedShape()
    (L_out,) = ofm.getPaddedShape()
    # K = K_padded. K_padded is always even (innermost ONNX dim is rounded
    # up to a 32-bit boundary, which for bfloat16 means pairs of elements), so
    # `K` is also even and `4*T + K` is even -> per-core IO buffers
    # satisfy the aiecompiler 4-byte total-size constraint. We need at least
    # K - 1 K elements for the convolution; one extra trailing element is
    # transferred but unused by the kernel.
    K = K
    assert K % 2 == 0, f"K must be even for bf16 4-byte alignment (got {K})"

    # The last DDR window ends at (calls_per_wts - 1) * (N_CORES * T) +
    # (N_CORES * T + K) == L_out + K. Make sure the padded IFM is big
    # enough to back that read; the compiler's auto-padding usually provides
    # the missing bytes (e.g. L_in = 4103 -> padded 4104 covers a K of 8).
    assert L_in >= L_out + K, (
        f"Padded L_in ({L_in}) must be >= L_out + K "
        f"({L_out} + {K} = {L_out + K}) so the last IFM window fits"
    )
```

* Calculate the number of kernel calls, to provide the `calls_per_wts` parameter, necessary for the asynchronous buffer rotation (see [Optimization: re-use weights across calls](#optimization-re-use-weights-across-calls)).
* 비동기 버퍼 회전에 필요한 'calls_per_wts' 매개변수를 제공하기 위해 커널 호출 수를 계산합니다([최적화: 호출 간 가중치 재사용](#optimization-re-use-weights-across-calls) 참조).
* `NT` is the total number of `T`-element OFM tiles; one L2 rotation feeds 16 of them (one per core), so we run `wts_call_count` rotations and the WTS buffer stays live for the whole compilation:
* 'NT'는 'T' 요소 OFM 타일의 총 개수입니다. 하나의 L2 회전은 그 중 16개(코어당 하나)를 공급하므로 `wts_call_count` 회전을 실행하고 WTS 버퍼는 전체 컴파일 동안 라이브 상태를 유지합니다.
```py
    # Total number of OFM tiles. Must be a multiple of the 4x4 grid size so
    # that each L2 rotation covers exactly 16 tiles, one per core.
    NT = L_out // T
    assert NT * T == L_out, f"L_out ({L_out}) must be a multiple of T ({T})"
    assert NT % N_CORES == 0, (
        f"NT ({NT}) must be a multiple of {N_CORES} so the workload distributes"
        " evenly across the 4x4 grid"
    )

    # Number of L3->L2 rotations.
    # This is the number of consecutive kernel calls during which the WTS buffer
    # stays constant. The whole filter is reused for every output tile, so it
    # can remain live for the entire computation.
    calls_per_wts = NT // N_CORES
```

* Create the L2 (memtile) buffers (see [Buffer definition](#buffer-definition)):
* L2(멤타일) 버퍼를 만듭니다([버퍼 정의](#buffer-definition) 참조).
  * `ifm` is placed in tile MT(0) for column-wise distribution to L1; it is double-buffered and sized for 16 tiles;
  * 'ifm'은 L1에 대한 열별 배포를 위해 타일 MT(0)에 배치됩니다. 이중 버퍼링되며 16개 타일에 해당하는 크기입니다.
  * `wts` is placed in tile MT(3) for row-wise distribution to L1; it is single-buffered (it stays constant) on memtile column 3;
  * 'wts'는 L1에 대한 행별 배포를 위해 타일 MT(3)에 배치됩니다. Memtile 열 3에서는 단일 버퍼링됩니다(일정하게 유지됨).
  * `ofm` is placed in tile MT(2), shaped `[OVERLAY_COLS, OVERLAY_ROWS, T]` so as to fit 16 output tiles:
  * `ofm`은 16개의 출력 타일에 맞도록 `[OVERLAY_COLS, OVERLAY_ROWS, T]` 모양의 타일 MT(2)에 배치됩니다.
```py
    ifm_l2 = tensor_expr.TensorVar.make(
        locations=[
            tensor_expr.Location(0, 0, 0x00000),
            tensor_expr.Location(0, 0, 0x10000),
        ],
        shape=[N_CORES * T + K],
        type=ifm.getDType(),
    )
    wts_l2 = tensor_expr.TensorVar.make(
        locations=[tensor_expr.Location(3, 0, 0x00000)],
        shape=[K],
        type=wts.getDType(),
    )
    ofm_l2 = tensor_expr.TensorVar.make(
        locations=[
            tensor_expr.Location(2, 0, 0x00000),
            tensor_expr.Location(2, 0, 0x10000),
        ],
        shape=[OVERLAY_COLS, OVERLAY_ROWS, T],
        type=ofm.getDType(),
    )
```

* Create the per-core L1 buffers (see [Buffer definition](#buffer-definition)):
* 코어당 L1 버퍼를 생성합니다([버퍼 정의](#buffer-definition) 참조).
  * `ifm`: each core in a column receives `4*T + K` IFM elements (broadcast) and uses its row index to pick its own slice.
  * `ifm`: 열의 각 코어는 `4*T + K` IFM 요소(브로드캐스트)를 수신하고 해당 행 인덱스를 사용하여 자체 슬라이스를 선택합니다.
  * `wts` is the full filter, single-buffered (no rotation is performed across kernel calls).
  * `wts`는 단일 버퍼링된 전체 필터입니다(커널 호출 전반에 걸쳐 회전이 수행되지 않음).
  * `ofm` is one `T`-element tile per core, double-buffered.
  * 'ofm'은 이중 버퍼링된 코어당 하나의 'T' 요소 타일입니다.
```py
    ifm_l1 = tensor_expr.TensorVar.make(
        [OVERLAY_ROWS * T + K],
        ifm.getDType(),
        tensor_expr.BufferingStrategy.DoubleBuffered,
    )
    wts_l1 = tensor_expr.TensorVar.make(
        [K],
        wts.getDType(),
        tensor_expr.BufferingStrategy.SingleBuffered,
    )
    ofm_l1 = tensor_expr.TensorVar.make(
        [T],
        ofm.getDType(),
        tensor_expr.BufferingStrategy.DoubleBuffered,
    )
```

* Configure the L3<->L2 transfers (see [Data movement setters](#data-movement-setters)):
* L3<->L2 전송을 구성합니다([데이터 이동 설정자](#data-movement-setters) 참조).
  * The IFM DDR pointer is tiled into `wts_call_count` overlapping windows of `N_CORES * T + K` elements, and the next window starts `N_CORES * T` later.
  * IFM DDR 포인터는 'N_CORES * T + K' 요소의 창과 겹치는 'wts_call_count'에 타일링되고 다음 창은 나중에 'N_CORES * T'를 시작합니다.
This is done using the `Tile` function (see [Buffer tiling and traversal order transformation](#buffer-tiling-and-traversal-order-transformation)).
이는 `Tile` 기능을 사용하여 수행됩니다([버퍼 타일링 및 순회 순서 변환](#buffer-tiling-and-traversal-order-transformation) 참조).
Note that only `T + K - 1` elements are used by the kernel, but the transfer needs to be a multiple of 32 bits; having asserted that `K` is even, we need to transfer one extra element.
커널에서는 'T + K - 1' 요소만 사용되지만 전송은 32비트의 배수여야 합니다. 'K'가 짝수라고 주장했으므로 하나의 추가 요소를 전송해야 합니다.
  * The WTS is transferred in full once.
  * WTS는 전체가 한 번 전송됩니다.
  * The OFM L2 is reshaped from `[OVERLAY_COLS, OVERLAY_ROWS, T]` down to a flat `N_CORES * T`-element block before being written back to DDR:
  * OFM L2는 DDR에 다시 기록되기 전에 `[OVERLAY_COLS, OVERLAY_ROWS, T]`에서 플랫 `N_CORES * T` 요소 블록으로 재구성됩니다.
```py
    # Get a DDR channel targeting MT(0)
    ifm_l3_l2 = [ch for ch in spu.get_l3_to_l2_channels() if ch.mem_tile_port.tile_col == 0][0]
    # Transfer a tile of IFM
    spu.set_l3_to_l2_transfer(
        ifm_l3_l2,
        ifm.getTensorVar().Tile(
            index=0,
            stride=N_CORES * T,
            tileLen=N_CORES * T + K,
            numTiles=wts_call_count,
        ),
        ifm_l2,
    )

    # Get a DDR channel targeting MT(3)
    wts_l3_l2 = [ch for ch in spu.get_l3_to_l2_channels() if ch.mem_tile_port.tile_col == 3][0]

    # The whole (padded) WTS is transferred once and reused.
    spu.set_l3_to_l2_transfer(wts_l3_l2, wts.getTensorVar(), wts_l2)

    # Get a L2 to L3 channel
    ofm_l2_l3 = spu.get_l2_to_l3_channels()[0]
    # OFM DDR is written back in 16*T-element blocks.
    spu.set_l2_to_l3_transfer(
        ofm_l2_l3,
        ofm_l2.Reshape([N_CORES * T]),
        ofm.getTensorVar().TileBy(index=0, tileLen=N_CORES * T),
    )
```

* Configure the L2 -> L1 IFM transfer.
* L2 -> L1 IFM 전송을 구성합니다.
* The L2 IFM is split into `OVERLAY_COLS` overlapping column tiles of `4*T + K` elements (see [Buffer tiling and traversal order transformation](#buffer-tiling-and-traversal-order-transformation)); each column tile is broadcast to its column's 4 cores:
* L2 IFM은 '4*T + K' 요소의 'OVERLAY_COLS' 중첩 열 타일로 분할됩니다([버퍼 타일링 및 순회 순서 변환](#buffer-tiling-and-traversal-order-transformation) 참조). 각 열 타일은 해당 열의 4개 코어로 브로드캐스트됩니다.
```py
    ifm_per_column = ifm_l2.Tile(
        index=0,
        stride=OVERLAY_ROWS * T,
        tileLen=OVERLAY_ROWS * T + K,
        numTiles=OVERLAY_COLS,
    )
    for col, ch in enumerate(spu.get_l2_to_l1_channels(broadcast=tensor_expr.broadcast_on.COLUMNS)):
        spu.set_l2_to_l1_transfer(ch, ifm_per_column[col], ifm_l1)
```

* Configure the L2 -> L1 WTS transfer.
* L2 -> L1 WTS 전송을 구성합니다.
* The full WTS buffer is broadcast once to every row, so both sides of `set_l2_to_l1_transfer` are lists of `OVERLAY_ROWS` entries:
* 전체 WTS 버퍼는 모든 행에 한 번 브로드캐스트되므로 `set_l2_to_l1_transfer`의 양쪽은 `OVERLAY_ROWS` 항목 목록입니다.
```py
    for ch in spu.get_l2_to_l1_channels(broadcast=tensor_expr.broadcast_on.ROWS):
        spu.set_l2_to_l1_transfer(
            ch,
            wts_l2,
            wts_l1,
            ratio=math.ceil(NT / N_CORES)
        )
```
Note the `ratio` argument we used: `NT / N_CORES` is the number of kernel calls, therefore the kernel can hold the same weights for the entire duration of the op, without needing to transfer them again (see [Kernel ratios](#kernel-ratios)).
우리가 사용한 `ratio` 인수에 주목하세요. `NT / N_CORES`는 커널 호출 수이므로 커널은 가중치를 다시 전송할 필요 없이 전체 작업 기간 동안 동일한 가중치를 유지할 수 있습니다([커널 비율](#kernel-ratios) 참조).

* Configure the L1 -> L2 OFM collection.
* L1 -> L2 OFM 컬렉션을 구성합니다.
* `ofm_l2` is already shaped `[OVERLAY_COLS, OVERLAY_ROWS, T]`, so `SpatialDistribute2D` cuts those two outermost dimensions and each core writes its `[T]` slice into one `(col, row)` location:
* `ofm_l2`는 이미 `[OVERLAY_COLS, OVERLAY_ROWS, T]` 모양이므로 `SpatialDistribute2D`는 가장 바깥쪽 두 차원을 자르고 각 코어는 `[T]` 슬라이스를 하나의 `(열, 행)` 위치에 씁니다.
```py
    ofm_spatial_dist = ofm_l2.SpatialDistribute2D(
        dimA=0, dimB=1, tileACount=OVERLAY_COLS, tileBCount=OVERLAY_ROWS
    )
    for col, ch in enumerate(spu.get_l2_to_l1_channels(broadcast=tensor_expr.broadcast_on.ROWS)):
      spu.set_l1_to_l2_transfer(
          ch,
          ofm_l1,
          ofm_spatial_dist[col],
      )
```

Finally, we configure how the kernel is called.
마지막으로 커널 호출 방법을 구성합니다.
The `lp_params` (`T`, `K`, and `calls_per_wts`) are passed in the same `set_kernel` call:
`lp_params`(`T`, `K` 및 `calls_per_wts`)는 동일한 `set_kernel` 호출에 전달됩니다.
```py
    # Set the kernel's arguments to be the L1 buffers we wrote into
    ph.set_kernel_arguments([ifm_l1, wts_l1, ofm_l1])
    # Set the kernel parameters (`lp_params`)
    ph.set_kernel_params([T, K, calls_per_wts])
    # Give the number of kernel calls
    ph.set_kernel_nb_calls(NT / N_CORES)
    # Give the compiler the name of the kernel to call and the file that implements it
    ph.set_kernel_function_name("conv1d_kernel")
    ph.set_kernel_impl(Path("conv1d.cpp"))
```

## Guidance for kernel tiling
## 커널 타일링 지침

This section provides general advice with respect to creating the tiled kernel and extracting the relevant data flow information.
이 섹션에서는 타일 커널 생성 및 관련 데이터 흐름 정보 추출과 관련된 일반적인 조언을 제공합니다.

### Tiling and data flow analysis
### 타일링 및 데이터 흐름 분석

Tiling consists in the subdivision of an iteration domain into smaller pieces, in such a way that each piece can be executed on a processor and its (usually small) local memory.
타일링은 반복 영역을 더 작은 조각으로 세분화하여 각 조각이 프로세서와 해당 로컬 메모리(보통 작은)에서 실행될 수 있도록 하는 것입니다.
It also enables parallelism, in that multiple smaller pieces are executed in parallel.
또한 여러 개의 작은 조각이 병렬로 실행된다는 점에서 병렬성을 가능하게 합니다.

#### Loop tiling
#### 루프 타일링

Starting from an existing code, one can mechanically apply loop tiling.
기존 코드에서 시작하여 루프 타일링을 기계적으로 적용할 수 있습니다.
A technique that is commonly applicable to obtain tiling from an existing loop nest is to:
기존 루프 중첩에서 타일링을 얻기 위해 일반적으로 적용할 수 있는 기술은 다음과 같습니다.

1. apply *strip-mining*: split a dimension into blocks of a certain size; then
1. *스트립 마이닝* 적용: 차원을 특정 크기의 블록으로 분할합니다. 그럼
```cpp
for (std::size_t i = 0; i < N; ++i)
  for (std::size_t j = 0; j < K; ++j)
    // Iteration (i, j)
```
becomes
된다
```cpp
for (std::size_t i = 0; i < N; ++i)
  for (std::size_t j0 = 0; j0 < K / T; ++j0) // K % T == 0
    for (std::size_t j1 = 0; j1 < T; ++j1)
      // Iteration (i, T * j0 + j1)
```
2. apply *loop inversion*: move the outer loop `j0` to outside the loop nest; then
2. *루프 반전* 적용: 외부 루프 `j0`을 루프 중첩 외부로 이동합니다. 그럼
```cpp
for (std::size_t i = 0; i < N; ++i)
  for (std::size_t j0 = 0; j0 < K / T; ++j0) // K % T == 0
    for (std::size_t j1 = 0; j1 < T; ++j1)
      // Iteration (i, T * j0 + j1)
```
becomes
된다
```cpp
for (std::size_t j0 = 0; j0 < K / T; ++j0) // K % T == 0
  // One tile
  for (std::size_t i = 0; i < N; ++i)
    for (std::size_t j1 = 0; j1 < T; ++j1)
      // Iteration (i, T * j0 + j1)
```

**These operations are not always legal.** The legality of these operations depends on the loop bounds, and what the loop is executing.
**이러한 작업이 항상 합법적인 것은 아닙니다.** 이러한 작업의 적법성은 루프 범위와 루프가 실행 중인 내용에 따라 달라집니다.
For conceptual information about tiling and its legality, refer to online resources such as the [Wikipedia sheet on loop tiling](https://en.wikipedia.org/wiki/Loop_tiling).
타일링 및 해당 적법성에 대한 개념 정보는 [루프 타일링에 대한 Wikipedia 시트](https://en.wikipedia.org/wiki/Loop_tiling)와 같은 온라인 리소스를 참조하세요.

#### Tiling of inputs and outputs, data flow analysis
#### 입력 및 출력 타일링, 데이터 흐름 분석

Although tiling is an operation on the kernel's *iteration domain*, i.e. its loop nest, tiling can be computed starting from the outputs (resp. inputs).
타일링은 커널의 *반복 도메인*, 즉 루프 중첩에 대한 작업이지만 타일링은 출력(각각 입력)에서 시작하여 계산될 수 있습니다.
Instead of tiling with an arbitrary size `T` like in the above Loop tiling section (see [Loop tiling](#loop-tiling)), one needs to break down an output (resp. input) array, and determine how big the iteration tiles that produce each piece of the output (resp. consume each piece of the input) are.
위의 루프 타일링 섹션([루프 타일링](#loop-tiling) 참조)처럼 임의 크기 'T'로 타일링하는 대신 출력(각 입력) 배열을 분해하고 각 출력 조각을 생성하는(각각 입력의 각 조각을 소비하는) 반복 타일의 크기를 결정해야 합니다.
Then, one can determine the size of the associated inputs (resp. outputs).
그런 다음 관련 입력(각 출력)의 크기를 결정할 수 있습니다.

The *data flow analysis* consists in building a formal relation between the inputs and the outputs of the kernel, that encodes, for each output element that gets computed, which input elements are needed to compute it.
*데이터 흐름 분석*은 계산되는 각 출력 요소에 대해 이를 계산하는 데 필요한 입력 요소를 인코딩하는 커널의 입력과 출력 간의 형식적인 관계를 구축하는 것으로 구성됩니다.

**Example:** tiling the following loop from the outputs.
**예:** 출력에서 ​​다음 루프를 타일링합니다.
```cpp
for (std::size_t i = 0; i < 128; ++i)
  for (std::size_t j = 0; j < 1024; ++j) {
    out[i][j] = in[j + 2] * j;
  }
```
We want to make blocks of `out` of size 4*8: `out[0][0]` to `out[0][7]`, `out[1][0]` to `out[1][7]`, `out[0][8]` to `out[0][15]`, etc.; all the way to `out[127][1023]`.
우리는 크기 4*8의 `out` 블록을 만들고 싶습니다: `out[0][0]`에서 `out[0][7]`, `out[1][0]`에서 `out[1][7]`, `out[0][8]`에서 `out[0][15]`까지; `out[127][1023]`까지.

Symbolically expressed, these blocks are the rectangles from `out[4*i0][8*j0]` to `out[4*i0+3][8*j0+7]`, where `i0` and `j0` are between 0 and respectively 128 / 4 = 32 and 1024 / 8 = 128.
기호로 표현하면, 이 블록은 `out[4*i0][8*j0]`에서 `out[4*i0+3][8*j0+7]`까지의 직사각형입니다. 여기서 `i0`과 `j0`은 0과 각각 128/4 = 32 및 1024/8 = 128 사이입니다.

There is only one iteration that contributes to `out[i][j]`, which is the `(i, j)`-th.
`out[i][j]`에 기여하는 반복은 `(i, j)`번째뿐입니다.
Therefore, we tile the `i` loop with size 4 and the `j` loop with size 8: the loop `i` is broken into two loops `i0` and `i1` where `i = 4*i0 + i1`, and `j = 8*j0 + j1` likewise.
따라서 `i` 루프를 크기 4로 타일링하고 `j` 루프를 크기 8로 타일링합니다. `i` 루프는 두 개의 루프 `i0`과 `i1`로 나뉩니다. 여기서 `i = 4*i0 + i1` 및 `j = 8*j0 + j1`도 마찬가지입니다.

```cpp
for (std::size_t i0 = 0; i0 < 32; ++i0)
  for (std::size_t j0 = 0; j0 < 128; ++j0)
    // One tile
    for (std::size_t i1 = 0; i1 < 4; ++i1)
      for (std::size_t j1 = 0; j1 < 8; ++j1) {
        // Iteration (4*i0 + i1, 8*j0 + j1)
        out[4*i0 + i1][8*j0 + j1] += in[4*i0 + i1 + 2] * (8*j0 + j1);
      }
```
To compute `out[4*i0][8*j0]` to `out[4*i0+3][8*j0+7]`, one needs a full iteration of the `i1` and `j1` loop.
`out[4*i0][8*j0]`을 `out[4*i0+3][8*j0+7]`로 계산하려면 `i1` 및 `j1` 루프의 전체 반복이 필요합니다.
Throughout this iteration, `in[4*i0 + 2]` to `in[4*i0 + 3 + 2]` are needed.
이 반복 전체에서 `in[4*i0 + 2]`에서 `in[4*i0 + 3 + 2]`가 필요합니다.
We then need to tile `in` with tile size 4, starting at offset 2.
그런 다음 오프셋 2에서 시작하여 타일 크기 4로 'in'을 타일링해야 합니다.

The above analysis can be written as the following data flow relation:
위의 분석은 다음과 같은 데이터 흐름 관계로 작성될 수 있습니다.
```
 * Input tiles of size 4, starting at offset 2
 * Output tiles of size 4*8
 * Tiles are indexed from i0 = 0 to i0 = 32, j0 = 0 to j0 = 128
 * For the tile indexed by (i0, j0):
   * The tile of output produced is out[4*i0][8*j0] to out[4*i0+3][8*j0+7],
   * One tile of input is needed, from in[4*i0 + 2] to in[4*i0 + 3 + 2].
```

Take note of this relation that is needed to build the tiling script: see Step 2: Allocate memory on the AIE grid and program data movement (see [Step 2: Allocate memory on the AIE grid and program data movement](#step-2-allocate-memory-on-the-aie-grid-and-program-data-movement)) for an example, and Python tiling script reference (see [Python tiling script reference](#python-tiling-script-reference)) for the reference.
타일링 스크립트를 작성하는 데 필요한 이 관계를 기록해 두십시오. 예시는 2단계: AIE 그리드에 메모리 할당 및 프로그램 데이터 이동([2단계: AIE 그리드에 메모리 할당 및 프로그램 데이터 이동](#step-2-allocate-memory-on-the-aie-grid-and-program-data-movement) 참조) 및 Python 타일링 스크립트 참조([Python 타일링 스크립트 참조)를 참조하세요. 참조](#python-tiling-script-reference))를 참조하세요.

#### Cost floors and the DDR re-read factor
#### 비용 하한선 및 DDR 다시 읽기 요소

The data-flow relation above tells you *which* input tiles each output tile
위의 데이터 흐름 관계는 *어떤* 입력 타일이 각 출력 타일인지 알려줍니다.
needs.
필요합니다.
At this point, you can compute whether your kernel is intrinsically memory or compute-bound.
이 시점에서 커널이 본질적으로 메모리인지 또는 컴퓨팅 바인딩인지 계산할 수 있습니다.
This will help you determine whether to optimize the kernel or tiling.
이는 커널을 최적화할지 타일링을 최적화할지 결정하는 데 도움이 됩니다.

**1.Compute the memory transfer and compute time floor.**
**1.메모리 전송을 계산하고 시간 층을 계산합니다.**
```
DMA floor      = sum(tensor volumes, each counted ONCE) / effective DDR BW
compute floor  = total MACs / (peak MACs per cycle per core * cores) / clock
achievable    ~= max(DMA floor, compute floor)
```

The larger floor names the regime (DMA-bound vs compute-bound), i.e. what is
더 큰 층은 체제(DMA 바인딩 대 컴퓨팅 바인딩)를 명명합니다.
worth optimizing.
최적화할 가치가 있습니다.

For instance, taking a Conv 1x1 + Add + ReLU with a 1x384x32x32 shape and bf16 type:
예를 들어, 1x384x32x32 모양과 bf16 유형의 Conv 1x1 + Add + ReLU를 사용하면 다음과 같습니다.
IFM + WTS + OFM = 768 KB + 294 KB + 768 KB ~= **1.83 MB**; at ~12 GB/s, the DMA
IFM + WTS + OFM = 768KB + 294KB + 768KB ~= **1.83MB**; ~12GB/s에서 DMA
floor is ~156 us.
바닥은 ~ 156 us입니다.

The compute floor for the same op: a 1x1 conv is `Cout * Cin` MACs per pixel, so
동일한 작업에 대한 컴퓨팅 바닥: 1x1 변환은 픽셀당 `Cout * Cin` MAC이므로
`384 * 384 * 32 * 32` ~= **151 M MACs** (the Add and ReLU add one elementwise pass
`384 * 384 * 32 * 32` ~= **151M MACs** (Add 및 ReLU는 하나의 요소별 패스를 추가합니다.
each over 393 K elements -- negligible).
각각 393,000개 이상의 요소(무시할 수 있음)
On one 4x4 stamp at 512 bf16 MACs per
512 bf16 MAC의 4x4 스탬프 1개당
cycle per core (one `aie::mmul<8,8,8>` issue; see
코어당 주기(`aie::mmul<8,8,8>` 문제 1개, 참조)
[the conv2d tutorial](/docs/30_custom_ops/tutorial/conv2d/conv3x3_int8_2stamp)), that is
[conv2d 튜토리얼](/docs/30_custom_ops/tutorial/conv2d/conv3x3_int8_2stamp)), 즉
`151e6 / (512 * 16)` ~= 18.4 K cycles, i.e. **~15 us** at a 1.25 GHz core clock.
`151e6 / (512 * 16)` ~= 18.4K 주기, 즉 1.25GHz 코어 클럭에서 **~15us**.

Compute is therefore ~10x cheaper than the transfers, so the op is DMA-bound: in
따라서 컴퓨팅은 전송보다 ~10배 저렴하므로 작업은 DMA에 바인딩됩니다.
this case, tiling is to be optimized rather than the kernel. (Both the ~12 GB/s
이 경우 커널보다는 타일링이 최적화됩니다. (둘 다 ~12GB/s
and the clock are platform figures -- substitute your target's.)
시계는 플랫폼 숫자입니다. 목표의 숫자로 대체하세요.)

**2.
Compute each tensor's DDR re-read factor.**
각 텐서의 DDR 다시 읽기 계수를 계산합니다.**

A tensor that is reused across an outer loop, but is not kept on-chip across that
외부 루프에서 재사용되지만 해당 루프 전체에서 온칩으로 유지되지 않는 텐서
loop, re-crosses DDR once per pass.
루프는 패스당 한 번씩 DDR을 다시 교차합니다.
Its DDR traffic is:
DDR 트래픽은 다음과 같습니다.

```
tensor DDR traffic = volume * reread
    reread = product of the .Repeat(...) factors applied at the L3<->L2 transfer
             (repeats at L2->L1 do not touch DDR and do not count)
```

A `reread` above 1 on a sizeable tensor means the design pays a multiple of the
크기가 큰 텐서에서 1보다 큰 '다시 읽기'는 설계가 다음의 배수를 지불한다는 의미입니다.
floor.
바닥.
This is a structural property of the dataflow -- the wrong operand is
이는 데이터 흐름의 구조적 속성입니다. 잘못된 피연산자는
being re-streamed -- rather than a tile-size knob to tune later.
나중에 조정할 타일 크기의 손잡이가 아닌 다시 스트리밍됩니다.
The fix is the
수정 사항은
classic stationarity choice: keep the smaller reused operand resident in L2, and
고전적인 정상성 선택: L2에 상주하는 더 작은 재사용 피연산자를 유지하고
hold the other in L1 across the inner loop, so that neither re-crosses DDR and
내부 루프를 가로질러 L1에 다른 하나를 유지하여 DDR을 다시 교차하지 않고
the total traffic falls back to the floor.
총 트래픽이 다시 바닥으로 떨어집니다.
Operands are held across calls using
피연산자는 다음을 사용하여 통화 전반에 걸쳐 유지됩니다.
asynchronous buffers, described in [Keeping buffers across kernel calls (asynchronous buffers)](#keeping-buffers-across-kernel-calls-asynchronous-buffers).
[커널 호출 전반에 걸쳐 버퍼 유지(비동기 버퍼)](#keeping-buffers-across-kernel-calls-asynchronous-buffers)에 설명된 비동기 버퍼.

For instance, on the same conv: a tiling that streams the IFM once per
예를 들어, 동일한 변환에서: IFM을 1회당 한 번씩 스트리밍하는 타일링
output-channel block re-reads it `n_blocks = 6` times, so IFM traffic is
출력 채널 블록은 이를 `n_blocks = 6`번 다시 읽으므로 IFM 트래픽은
768 KB x 6 = 4.5 MB and the total is 5.6 MB, or 3.1x the floor (measured
768KB x 6 = 4.5MB이고 총 크기는 5.6MB, 즉 바닥의 3.1배(측정치)입니다.
~484 us).
Making the weights resident in L2 and holding the IFM in L1 brings
가중치를 L2에 상주시키고 IFM을 L1에 유지하면
every `reread` back to 1, for a total of 1.83 MB, at the floor (measured
바닥에서 총 1.83MB에 대해 '다시 읽기'를 할 때마다 1로 다시 돌아갑니다(측정됨).
~152 us).
Same operator, same kernel: a purely structural change worth 4.3x.
동일한 연산자, 동일한 커널: 4.3배 가치가 있는 순전히 구조적 변경입니다.

A tiling whose re-read factor exceeds 1 on a sizeable tensor is therefore best
따라서 크기가 큰 텐서에서 다시 읽기 인수가 1을 초과하는 타일링이 가장 좋습니다.
corrected before compiling, since no amount of later parameter tuning recovers
이후의 매개변수 조정이 복구되지 않으므로 컴파일하기 전에 수정되었습니다.
the traffic it costs.
트래픽 비용이 발생합니다.

### Kernel interface
### 커널 인터페이스

There are two function prototypes accepted for custom ops.
사용자 정의 작업에는 두 가지 함수 프로토타입이 허용됩니다.
* For functions that require one input buffer:
* 하나의 입력 버퍼가 필요한 함수의 경우:
```cpp
template <typename dtype_ifm, typename dtype_ofm>
__attribute__((noinline)) void kernel_name(
  adf::input_buffer_conf<dtype_ifm, /*adf::bpc_sync_0d or bpc_async_0d*/> &__restrict input_a,
  adf::output_buffer_conf<dtype_ofm, /*adf::bpc_sync_0d or bpc_async_0d*/> &__restrict output,
  const uint32_t (&lp_params)[NB_PARAMS]);
```
* For functions that require two input buffers:
* 두 개의 입력 버퍼가 필요한 함수의 경우:
```cpp
template <typename dtype_ifm, typename dtype_wts, typename dtype_ofm>
__attribute__((noinline)) void kernel_name(
  adf::input_buffer_conf<dtype_ifm, /*adf::bpc_sync_0d or bpc_async_0d*/> &__restrict input_a,
  adf::input_buffer_conf<dtype_wts, /*adf::bpc_sync_0d or bpc_async_0d*/> &__restrict input_b,
  adf::output_buffer_conf<dtype_ofm, /*adf::bpc_sync_0d or bpc_async_0d*/> &__restrict output,
  const uint32_t (&lp_params)[NB_PARAMS]);
```

The `input_a` and `input_b` arguments are the L1 buffers that are passed to the AIE core.
'input_a' 및 'input_b' 인수는 AIE 코어에 전달되는 L1 버퍼입니다.
Per the overlay, there are at most two of them.
오버레이당 최대 2개가 있습니다.

**Remarks:**
**비고:**
* The kernel function name is free to choose, and must be set in the tiling script using `set_kernel_function_name`; likewise, the kernel's implementation file must be set using `set_kernel_impl` (see [Writing the tiling script](#writing-the-tiling-script)).
* 커널 함수 이름은 자유롭게 선택할 수 있으며 `set_kernel_function_name`을 사용하여 타일링 스크립트에서 설정해야 합니다. 마찬가지로 커널의 구현 파일은 `set_kernel_impl`을 사용하여 설정해야 합니다([타일링 스크립트 작성](#writing-the-tiling-script) 참조).
* Each stamp can use a different kernel file and call a different function.
* 각 스탬프는 다른 커널 파일을 사용하고 다른 함수를 호출할 수 있습니다.
* The templating is mandatory, even if the types are fixed by the kernel.
* 유형이 커널에 의해 고정된 경우에도 템플릿은 필수입니다.
* `lp_params` can contain an arbitrary number of parameters.
* `lp_params`에는 임의 개수의 매개변수가 포함될 수 있습니다.
* Their type is necessarily `uint32_t`, but you may cast and reinterpret these as necessary within the kernel.
* 해당 유형은 반드시 'uint32_t'이지만 커널 내에서 필요에 따라 이를 캐스팅하고 재해석할 수 있습니다.
* Choosing `adf::bpc_sync_0d` or `adf::bpc_async_0d` depends on whether double-buffering rotation should happen every call.
* `adf::bpc_sync_0d` 또는 `adf::bpc_async_0d` 선택은 호출마다 이중 버퍼 회전이 발생해야 하는지 여부에 따라 달라집니다.
* For those L1 buffers given to the kernel (using `set_kernel_arguments`) whose type is `adf::bpc_async_0d`, the L2 to L1 or L1 to L2 transfers (`set_l2_to_l1_transfer`, `set_l1_to_l2_transfer`) must bear a `ratio` argument with the ratios.
* 유형이 `adf::bpc_async_0d`인 커널에 제공된 L1 버퍼(`set_kernel_arguments` 사용)의 경우 L2에서 L1로 또는 L1에서 L2로의 전송(`set_l2_to_l1_transfer`, `set_l1_to_l2_transfer`)은 비율과 함께 `ratio` 인수를 포함해야 합니다.
* See [Keeping buffers across kernel calls (asynchronous buffers)](#keeping-buffers-across-kernel-calls-asynchronous-buffers).
* [커널 호출 전반에 걸쳐 버퍼 유지(비동기 버퍼)](#keeping-buffers-across-kernel-calls-asynchronous-buffers)를 참조하세요.
* Per the overlay (see [AIE Topology and VAIML overlay](#aie-topology-and-vaiml-overlay)), both the `input_a` and `input_b` buffers are populated using broadcast channels, either column-wise or row-wise, from L2 memories.
* 오버레이에 따라([AIE 토폴로지 및 VAIML 오버레이](#aie-topology-and-vaiml-overlay) 참조) `input_a` 및 `input_b` 버퍼는 모두 L2 메모리에서 열 방향 또는 행 방향으로 브로드캐스트 채널을 사용하여 채워집니다.
* Every core on either a row or a column will therefore have the same contents.
* 따라서 행이나 열의 모든 코어는 동일한 내용을 갖습니다.
* The buffers across a row or column are nevertheless physically independent, and changes in one core will not be reflected in other cores.
* 그럼에도 불구하고 행이나 열의 버퍼는 물리적으로 독립적이며 한 코어의 변경 사항이 다른 코어에 반영되지 않습니다.
* See the [Writing the tiling script](#writing-the-tiling-script) section.
* [타일링 스크립트 작성](#writing-the-tiling-script) 섹션을 참조하세요.


### Buffer usage
### 버퍼 사용량

One cannot directly access the `input_a`, `input_b` and `output` buffer objects.
`input_a`, `input_b` 및 `output` 버퍼 개체에 직접 액세스할 수 없습니다.
Pointers are obtained using the `data()` method:
포인터는 `data()` 메서드를 사용하여 얻습니다.

```cpp
template <typename dtype_ifm, typename dtype_wts, typename dtype_ofm>
__attribute__((noinline)) void kernel_name(
  adf::input_buffer_conf<dtype_ifm, /*adf::bpc_sync_0d or bpc_async_0d*/> &__restrict input_a,
  adf::input_buffer_conf<dtype_wts, /*adf::bpc_sync_0d or bpc_async_0d*/> &__restrict input_b,
  adf::output_buffer_conf<dtype_ofm, /*adf::bpc_sync_0d or bpc_async_0d*/> &__restrict output,
  const uint32_t (&lp_params)[NB_PARAMS]) {

  auto *input_a_data = (dtype_ifm *__restrict)(input_a.data());
  auto *input_b_data = (dtype_wts *__restrict)(input_b.data());
  auto *output_data = (dtype_ofm *__restrict)(output.data());
}
```

### Keeping buffers across kernel calls (asynchronous buffers)
### 커널 호출 전반에 걸쳐 버퍼 유지(비동기 버퍼)

The asynchronous buffer feature allows a buffer to be reused across several kernel calls, instead of being reloaded or re-emitted every single call.
비동기 버퍼 기능을 사용하면 호출할 때마다 다시 로드하거나 다시 내보내는 대신 여러 커널 호출에서 버퍼를 재사용할 수 있습니다.
This is useful for advanced tiling and scheduling scenarios, such as overlapping data for multi-tile reductions or broadcasts.
이는 다중 타일 축소 또는 브로드캐스트를 위한 데이터 중복과 같은 고급 타일링 및 예약 시나리오에 유용합니다.

To use asynchronous buffers, you must explicitly declare, for each buffer port, how many kernel calls the buffer should remain unchanged.
비동기 버퍼를 사용하려면 각 버퍼 포트에 대해 버퍼가 변경되지 않고 유지되어야 하는 커널 호출 수를 명시적으로 선언해야 합니다.
This is done by specifying the number of kernel calls per buffer in the `ratio` argument on the L2 to L1 (or L1 to L2) transfer in your tiling script.
이는 타일링 스크립트에서 L2에서 L1로(또는 L1에서 L2로) 전송 시 `ratio` 인수에 버퍼당 커널 호출 수를 지정하여 수행됩니다.
For example, if a buffer will be used for 4 kernel calls before being updated, you would set its kernel ratio to 4.
예를 들어 업데이트되기 전에 버퍼가 4개의 커널 호출에 사용되는 경우 커널 비율을 4로 설정합니다.

On the C++ kernel side, the kernel is responsible for acquiring and releasing asynchronous buffers for the required duration.
C++ 커널 측에서 커널은 필요한 기간 동안 비동기 버퍼를 획득하고 해제하는 역할을 담당합니다.

The kernel must:
커널은 다음을 수행해야 합니다.
* explicitly *acquire* the buffer before calling `.data()`,
* `.data()`를 호출하기 전에 명시적으로 버퍼를 *획득*합니다.
* process the correct number of calls (kernel ratio) before releasing the buffer is considered released,
* 버퍼를 해제한 것으로 간주되기 전에 올바른 호출 수(커널 비율)를 처리합니다.
* *release* the buffer after the right number of kernel calls are elapsed.
* 적절한 커널 호출 횟수가 경과된 후에 버퍼를 *해제*합니다.

The VAIML runtime guarantees that the buffer will stay unchanged for the promised number of calls (as set in the `ratio` parameter of `set_l2_to_l1_transfer`), and it is critical that your kernel code honors this contract for correctness.
VAIML 런타임은 약속된 호출 수(`set_l2_to_l1_transfer`의 `ratio` 매개변수에 설정된 대로) 동안 버퍼가 변경되지 않고 유지되도록 보장하며, 커널 코드가 정확성을 위해 이 계약을 준수하는 것이 중요합니다.

**Example:** to declare the second input of a kernel to be kept for 4 calls, one needs to:
**예:** 4번의 호출 동안 유지되도록 커널의 두 번째 입력을 선언하려면 다음을 수행해야 합니다.

- In the tiling script (Python), kernel ratios are passed through `set_l2_to_l1_transfer`:
- 타일링 스크립트(Python)에서 커널 비율은 `set_l2_to_l1_transfer`를 통해 전달됩니다.
  ```python
  spu.set_l2_to_l1_transfer(
    l2_to_l1_channel, l2_buf, l1_buf, ratio=4 # buffer stays constant for 4 kernel invocations
  )
  ```
- The kernel's prototype on the second input needs to use `adf::bpc_async_0d` as a mode.
- 두 번째 입력의 커널 프로토타입은 'adf::bpc_async_0d'를 모드로 사용해야 합니다.
- In the kernel (C++), use the buffer’s acquire/release API, and ensure you process exactly the number of calls promised by `kernel_ratios` before changing the asynchronous buffer.
- 커널(C++)에서는 버퍼의 획득/해제 API를 사용하고 비동기 버퍼를 변경하기 전에 'kernel_ratios'에서 약속한 호출 수를 정확하게 처리하는지 확인하세요.
```cpp
template <typename dtype_a, typename dtype_b, typename dtype_c>
void my_kernel(
  adf::input_buffer_conf<dtype_a, adf::bpc_sync_0d> &__restrict input_a,
  adf::input_buffer_conf<dtype_b, adf::bpc_async_0d> &__restrict input_b,
  adf::output_buffer_conf<dtype_c, bpc_sync_0d> &__restrict output,
  const uint32_t (&lp_params)[lp_size]
);
```

Buffer acquisition is done as follows:
버퍼 획득은 다음과 같이 수행됩니다.

```cpp
template <typename dtype_a, typename dtype_b, typename dtype_c>
void my_kernel(
  adf::input_buffer_conf<dtype_a, adf::bpc_sync_0d> &__restrict input_a,
  adf::input_buffer_conf<dtype_b, adf::bpc_async_0d> &__restrict input_b,
  adf::output_buffer_conf<dtype_c, bpc_sync_0d> &__restrict output,
  const uint32_t (&lp_params)[lp_size]
) {
  static uint32_t tile_count = 0;
  if(tile_count == 0) {
    input_b.acquire();
  }
  // Only use input_b after acquisition.
  auto *input_b_data = input_b.data();

  // ... processing code ...

  // Update tile count and release the buffer if we've reached 4 uses of it
  tile_count += 1;
  if(tile_count == 4) {
    input_b.release();
    tile_count = 0;
  }

}
```

**Remark**: It is possible to use a run-time parameter (using `lp_params`) to give the kernel ratios from the tiling script to the kernel.
**참고**: 런타임 매개변수(`lp_params` 사용)를 사용하여 타일링 스크립트의 커널 비율을 커널에 제공할 수 있습니다.
These can then be dependent on the op instance rather than statically encoded.
그런 다음 정적으로 인코딩되지 않고 op 인스턴스에 종속될 수 있습니다.
```cpp
template <typename dtype_a, typename dtype_b, typename dtype_c>
void my_kernel(
  adf::input_buffer_conf<dtype_a, adf::bpc_sync_0d> &__restrict input_a,
  adf::input_buffer_conf<dtype_b, adf::bpc_async_0d> &__restrict input_b,
  adf::output_buffer_conf<dtype_c, bpc_sync_0d> &__restrict output,
  const uint32_t (&lp_params)[lp_size]
) {
  uint32_t calls_per_b_tile = lp_params[0];

  // ...

  // Release the buffer if we've reached all uses of it
  tile_count += 1;
  if(tile_count == calls_per_b_tile) {
    input_b.release();
    tile_count = 0;
  }

}
```

### Random accesses and absolute positioning within tensors
### 텐서 내 무작위 액세스 및 절대 위치 지정

Kernels on the AIE are not allowed to access tensors stored in L2 or L3 memory directly.
AIE의 커널은 L2 또는 L3 메모리에 저장된 텐서에 직접 액세스할 수 없습니다.
Each kernel call operates only on the buffers it receives, which correspond to individual tiles (subsections) of the overall tensor.
각 커널 호출은 전체 텐서의 개별 타일(하위 섹션)에 해당하는 수신 버퍼에서만 작동합니다.
The position of a tile within the global tensor is not automatically provided to the kernel.
전역 텐서 내의 타일 위치는 커널에 자동으로 제공되지 않습니다.

If a kernel requires random access to the entire input or output tensor (for example, accessing elements outside its assigned tile), it is sometimes possible to rework the algorithm to avoid such random accesses, and make them local to the current tile.
커널이 전체 입력 또는 출력 텐서에 대한 무작위 액세스를 요구하는 경우(예: 할당된 타일 외부의 요소에 액세스), 이러한 무작위 액세스를 방지하기 위해 알고리즘을 재작업하고 이를 현재 타일에 로컬로 만들 수 있는 경우가 있습니다.

In some cases (data-dependent accesses, such as indices encoded in an operator's input), one can obtain random accesses to an input array by traversing it entirely more than once, using the `Repeat` function in the tiling script (see [Buffer tiling and traversal order transformation](#buffer-tiling-and-traversal-order-transformation)), and keeping track of the current position within the tensor.
일부 경우(연산자의 입력에 인코딩된 인덱스와 같은 데이터 종속 액세스) 타일링 스크립트의 '반복' 기능을 사용하여([버퍼 타일링 및 순회 순서 변환](#buffer-tiling-and-traversal-order-transformation) 참조) 입력 배열을 완전히 두 번 이상 순회하고 텐서 내의 현재 위치를 추적하여 입력 배열에 대한 무작위 액세스를 얻을 수 있습니다.

For instance, the following example tracks the position of a tile within the global tensor using `tile_size` and `nb_tiles` provided as run-time parameters.
예를 들어, 다음 예시에서는 런타임 매개변수로 제공된 'tile_size' 및 'nb_tiles'를 사용하여 전역 텐서 내 타일의 위치를 ​​추적합니다.
```cpp
// Example: Tracking tile position using a static counter
// Traverse input_b for every tile of input_a and output
void kernel_example(
  adf::input_buffer_conf<dtype_a, adf::bpc_sync_0d> &__restrict input_a,
  adf::input_buffer_conf<dtype_b, adf::bpc_async_0d> &__restrict input_b,
  adf::output_buffer_conf<dtype_c, bpc_sync_0d> &__restrict output,
  const uint32_t (&lp_params)[lp_size]) {

  static uint32_t tile_b_index = 0; // persists across invocations
  uint32_t tile_size = lp_params[0];
  uint32_t nb_tiles = lp_params[1];
  uint32_t global_start = tile_index * tile_size;
  uint32_t global_end   = global_start + tile_size;

  if(tile_b_index == 0) {
    input_a.acquire();
    output.acquire();
  }

  auto *input_a_data = (uint32_t *__restrict)(input_a.data());
  auto *input_b_data = (uint32_t *__restrict)(input_b.data());
  auto *output_data = (dtype_ofm *__restrict)(output.data());

  // ... process tile knowing that:
  // tile_b_index * tile_size <= indices of input_b_data < (1 + tile_b_index) * tile_size

  ++tile_b_index; // Update for next invocation
  if(tile_b_index == nb_tiles) {
    tile_b_index = 0;
    input_a.release();
    output.release();
  }
}
```

Traversing a buffer multiple times is an expensive operation.
버퍼를 여러 번 탐색하는 것은 비용이 많이 드는 작업입니다.
Random accesses to the global tensor are therefore in general discouraged.
따라서 전역 텐서에 대한 무작위 액세스는 일반적으로 권장되지 않습니다.

## Guidance for AIE data movement
## AIE 데이터 이동 지침

### Assignment of AIE cores to tiles
### 타일에 AIE 코어 할당

In order to compute one tile of outputs, an AIE core needs to be provided with specific input data.
하나의 출력 타일을 계산하려면 AIE 코어에 특정 입력 데이터가 제공되어야 합니다.
Core assignment to tiles can be done arbitrarily, as long as the following constraints are satisfied:
다음 제약 조건이 충족되는 한 타일에 대한 코어 할당을 임의로 수행할 수 있습니다.
* **Overlay topology**.
* **오버레이 토폴로지**.
* The overlay constrains which routes the data can take.
* 오버레이는 데이터가 사용할 수 있는 경로를 제한합니다.
On each 4x4 set of cores, the IFM is routed **broadcast down each column**, and weights are **broadcast down each row**.
각 4x4 코어 세트에서 IFM은 **각 열 아래로 브로드캐스트** 라우팅되고, 가중치는 **각 행 아래로 브로드캐스트**됩니다.
The L1 (in-core) buffers for IFM and WTS on each core need to be as big as all the data that gets transferred.
각 코어의 IFM 및 WTS용 L1(코어 내) 버퍼는 전송되는 모든 데이터만큼 커야 합니다.
* **L1 buffer sizing**: all L1 buffers mut fit within L1 capacity, accounting for ping/pong (double-buffering, which doubles the size of each buffer) and the reserved stack/heap region.
* **L1 버퍼 크기**: 모든 L1 버퍼는 ping/pong(각 버퍼의 크기를 두 배로 늘리는 이중 버퍼링) 및 예약된 스택/힙 영역을 고려하여 L1 용량 내에 맞습니다.
* **L2 Buffer sizing**.
* **L2 버퍼 크기 조정**.
* The L2 buffers need to hold enough data such that each tile along each column (resp. row) has the IFM (resp.
* L2 버퍼는 각 열(각 행)의 각 타일이 IFM(각 행)을 가질 수 있도록 충분한 데이터를 보유해야 합니다.
* WTS) data it needs to produce its output; but, the total buffer size must not exceed the L2 buffer capacity.
* WTS) 출력을 생성하는 데 필요한 데이터입니다. 단, 총 버퍼 크기는 L2 버퍼 용량을 초과할 수 없습니다.

![`AIE-1x4x4-overlay.svg`](resources/AIE-1x4x4-overlay.svg)

**Heuristic**: in general, one will try to maximize the reuse of IFM within a column and WTS within a row: if, like in the example (see [Tiling process example](#tiling-process-example)), tiles are overlapping, then the overlapping portions should be used more than once across a column.
**경험적 방법**: 일반적으로 열 내에서는 IFM, 행 내에서는 WTS의 재사용을 최대화하려고 합니다. 예([타일링 프로세스 예](#tiling-process-example) 참조)와 같이 타일이 겹치는 경우 겹치는 부분은 열 전체에서 두 번 이상 사용해야 합니다.
This is sometimes not possible (e.g. with element-wise operations).
이는 때로는 불가능합니다(예: 요소별 작업의 경우).

To determine exactly how much stack and heap are needed by your kernel, refer to [this section](README.md#determining-the-stack-and-heap-size) of the general custom ops guide.
커널에 필요한 스택과 힙의 양을 정확히 확인하려면 일반 사용자 정의 작업 가이드의 [이 섹션](README.md#determining-the-stack-and-heap-size)을 참조하세요.

## Python tiling script reference
## Python 타일링 스크립트 참조

This section is a reference for the Python tiling script API: data-movement
이 섹션은 Python 타일링 스크립트 API: 데이터 이동에 대한 참조입니다.
guidelines, the `getTiling()` interface, buffer and setter primitives, kernel
지침, `getTiling()` 인터페이스, 버퍼 및 설정자 기본 요소, 커널
ratios, design tips, and troubleshooting.
비율, 디자인 팁 및 문제 해결.
It complements the worked example in
이는 작업된 예제를 보완합니다.
Step 2 (see [Step 2: Allocate memory on the AIE grid and program data movement](#step-2-allocate-memory-on-the-aie-grid-and-program-data-movement)).
2단계([2단계: AIE 그리드에 메모리 할당 및 프로그램 데이터 이동](#step-2-allocate-memory-on-the-aie-grid-and-program-data-movement) 참조).

The tiling script's role is to define:
타일링 스크립트의 역할은 다음을 정의하는 것입니다.
* the locations and sizes of buffers at each memory level,
* 각 메모리 레벨의 버퍼 위치와 크기,
* the data movement between memory levels.
* 메모리 레벨 간의 데이터 이동.

This section describes the Python interface provided by VAIML to build such scripts.
이 섹션에서는 이러한 스크립트를 빌드하기 위해 VAIML에서 제공하는 Python 인터페이스에 대해 설명합니다.

#### Interface and provided information
#### 인터페이스 및 제공정보

The tiling script needs to be defined in a Python file, requiring a `getTiling()` function.
타일링 스크립트는 Python 파일에 정의되어야 하며 `getTiling()` 함수가 필요합니다.

##### The `getTiling()` function
##### `getTiling()` 함수

The `getTiling()` function receives an `OperatorInfo` object that contains information on, e.g., the shapes of the inputs and outputs, and a tiling script configuration object that **must** be named `tiling`, and that is of type `AieConfig`:
`getTiling()` 함수는 입력 및 출력의 모양 등에 대한 정보가 포함된 `OperatorInfo` 객체와 `tiling`이라는 이름을 **반드시** 지정해야 하며 `AieConfig` 유형인 타일링 스크립트 구성 객체를 수신합니다.
```py
def getTiling(opInfo: tensor_expr.OperatorInfo, tiling: tensor_expr.AieConfig):
  ...
```
The `tiling` object(s) contain(s) the accessors that set the configurations of DMA engines (see [Data Movement Setters](#data-movement-setters)).
`tiling` 객체에는 DMA 엔진의 구성을 설정하는 접근자가 포함되어 있습니다([데이터 이동 설정자](#data-movement-setters) 참조).

Configure as many stamps as your operator distributes over: stamps you leave untouched are trimmed on return (at least one is always kept), so a list-form script that only configures `tiling[0]` behaves like the single-stamp form.
운영자가 배포하는 만큼 많은 스탬프를 구성합니다. 손대지 않은 채로 남겨둔 스탬프는 반환 시 잘립니다(최소 하나는 항상 유지됨). 따라서 '타일링[0]'만 구성하는 목록 형식 스크립트는 단일 스탬프 형식처럼 작동합니다.

Example:
예:
```python
import tensor_expr
def getTiling(opInterface: tensor_expr.OperatorInfo, tiling: tensor_expr.AieConfig):
    # one entry per stamp; a single-stamp overlay gives len(tiling) == 1
    for stamp in tiling:
        # Get the single phase for this stamp
        spu = stamp[0]
        # rest of the code below using spu
```

Example of requiring a 6x4x4 overlay:
6x4x4 오버레이가 필요한 예:
```python
import tensor_expr
def getTiling(opInterface: tensor_expr.OperatorInfo, tiling: tensor_expr.AieConfig):
    # rest of the code below
    assert(len(tiling) == 6), "expected 6 stamps"
```

**Phases**: each stamp is able to hold multiple *phases*, which designate individual steps taken by the custom op.
**단계**: 각 스탬프는 사용자 지정 작업에서 수행하는 개별 단계를 지정하는 여러 *단계*를 보유할 수 있습니다.
Each stamp initially contains a single `StampPhaseUnit`, which is accessible by `tiling[0][0]` (or `stamp[0]` if you iterate `for stamp in tiling`).
각 스탬프에는 처음에 `tiling[0][0]`(또는 `for stamp in tiling`을 반복하는 경우 `stamp[0]`)으로 액세스할 수 있는 단일 `StampPhaseUnit`이 포함되어 있습니다.

In the rest of this guide, we designate that `StampPhaseUnit` of a stamp by `spu`:
이 가이드의 나머지 부분에서는 `spu`로 스탬프의 `StampPhaseUnit`을 지정합니다.
```py
    spu = tiling[0][0]
```

This `spu` is the object that configures the data transfers and the kernel.
이 'spu'는 데이터 전송과 커널을 구성하는 개체입니다.

##### Obtaining ONNX tensor shapes in the tiling script
##### 타일링 스크립트에서 ONNX 텐서 모양 얻기

The compiler automatically aligns the innermost dimension of the ONNX tensors to 32 bits.
컴파일러는 자동으로 ONNX 텐서의 가장 안쪽 차원을 32비트로 정렬합니다.
It is possible to obtain the original ONNX model shape using `modelShape` for operands and result:
피연산자 및 결과에 `modelShape`를 사용하여 원래 ONNX 모델 모양을 얻을 수 있습니다.

**Example:**
**예:**
```python
    # The shape in the ONNX model is [N, C, H, W].
    onnxShape = operand.modelShape
    # The DDR shape is [H, C, N, W].
    ddrUnpaddedShape = operand.getShape()
    # The DDR padded shape is [H+2, C+3, N, W].
    ddrShape = operand.getPaddedShape()
```

##### Using ONNX attributes in the tiling script
##### 타일링 스크립트에서 ONNX 특성 사용

Each ONNX operation has [attributes](https://onnx.ai/onnx/intro/concepts.html#input-output-node-initializer-attributes).
각 ONNX 작업에는 [속성](https://onnx.ai/onnx/intro/concepts.html#input-output-node-initializer-attributes)이 있습니다.
The ONNX operation attributes are passed to the tiling script through the op interface's `attributes` field.
ONNX 작업 속성은 op 인터페이스의 `attributes` 필드를 통해 타일링 스크립트에 전달됩니다.

Example: considering this operator
예: 이 연산자를 고려해보세요
```
<
   ir_version: 7,
   opset_import: ["" : 13, "mydomain" : 1]
>
test (float[32,64] x) => (float[32,1] res) {
   res = mydomain.myreducemax <data_type: string = "float32", shape: ints = [32, 1], axis = 1> (x)
}
```
then the `axis` attribute can be accessed by the tiling script as follows:
그러면 다음과 같이 타일링 스크립트를 통해 `axis` 속성에 액세스할 수 있습니다.
```python
    axis_attr = opInterface.attributes["axis"]
```

#### Buffer definition
#### 버퍼 정의

Buffers are defined as `TensorVar` objects with the correct properties and size.
버퍼는 올바른 속성과 크기를 가진 `TensorVar` 객체로 정의됩니다.

The main buffer-construction entry point is `tensor_expr.TensorVar.make(...)`.
주요 버퍼 생성 진입점은 `tensor_expr.TensorVar.make(...)`입니다.
The most common forms are:
가장 일반적인 형태는 다음과 같습니다.

- `tensor_expr.TensorVar.make([d0, d1], type)` or `tensor_expr.TensorVar.make(shape=[d0, d1], type=type)` for a shape-only buffer when the compiler owns the placement, such as DDR/L3 helper buffers.
- 컴파일러가 DDR/L3 도우미 버퍼와 같은 배치를 소유한 경우 모양 전용 버퍼의 경우 `tensor_expr.TensorVar.make([d0, d1], type)` 또는 `tensor_expr.TensorVar.make(shape=[d0, d1], type=type)`.
- `tensor_expr.TensorVar.make([Location(...), ...], shape, type)` for manual placement when you want to pick the addresses yourself (the dtype keyword is `type`).
- 주소를 직접 선택하려는 경우 수동 배치를 위한 `tensor_expr.TensorVar.make([Location(...), ...], Shape, type)`(dtype 키워드는 `type`입니다).
- `tensor_expr.TensorVar.make(shape, type, tensor_expr.BufferingStrategy.SingleBuffered)` or `tensor_expr.TensorVar.make(shape, type, tensor_expr.BufferingStrategy.DoubleBuffered)` for automatic L1 placement.
- 자동 L1 배치를 위한 `tensor_expr.TensorVar.make(shape, type, tensor_expr.BufferingStrategy.SingleBuffered)` 또는 `tensor_expr.TensorVar.make(shape, type, tensor_expr.BufferingStrategy.DoubleBuffered)`.

Example:
예:

```python
# Shape-only helper buffer, type manually specified
l3_var = tensor_expr.TensorVar.make([d0, d1], tensor_expr.OperandType.Int8)

# Manual placement, type obtained from the ONNX model
ifm_mk_var = tensor_expr.TensorVar.make(
    [tensor_expr.Location(ifm_ping_addr), tensor_expr.Location(ifm_pong_addr)],
    [d0, d1],
    ifm.getDType(),
)

# Automatic L1 placement
ofm_mk_var = tensor_expr.TensorVar.make(
    [d0, d1],
    ofm.getDType(),
    tensor_expr.BufferingStrategy.DoubleBuffered,
)
```

**Remarks**:
**비고**:
- TensorVar shapes are expressed in **elements**, not bytes.
- TensorVar 모양은 바이트가 아닌 **요소**로 표현됩니다.
- `Location` addresses are **byte** addresses.
- `위치` 주소는 **바이트** 주소입니다.
- Dtype size impacts both buffer size and alignment (e.g., bfloat16 = 2 bytes/element).
- Dtype 크기는 버퍼 크기와 정렬 모두에 영향을 미칩니다(예: bfloat16 = 2바이트/요소).

##### External buffers
##### 외부 버퍼

External buffers ("L3") are located in the global memory, The compiler owns the DDR placement and size.
외부 버퍼("L3")는 전역 메모리에 위치하며, 컴파일러는 DDR 배치와 크기를 소유합니다.
The tiling script cannot affect these buffers.
타일링 스크립트는 이러한 버퍼에 영향을 미칠 수 없습니다.

There are two common L3 use cases:
두 가지 일반적인 L3 사용 사례가 있습니다.
- **Real op inputs/outputs in DDR:** use `getTensorVar()` on the operand or result.
- **DDR의 실제 연산 입력/출력:** 피연산자 또는 결과에 `getTensorVar()`를 사용합니다.
- This is the most common form in custom-op tiling scripts because it gives you the actual model tensor in DDR.
- 이는 DDR의 실제 모델 텐서를 제공하므로 사용자 정의 최적화 스크립트에서 가장 일반적인 형식입니다.
- **Helper or test buffers in DDR:** use `tensor_expr.TensorVar.make(shape=[...])` when you want a shape-only TensorVar and do not want to specify an address yourself.
- **DDR의 도우미 또는 테스트 버퍼:** 모양 전용 TensorVar를 원하고 주소를 직접 지정하고 싶지 않은 경우 `tensor_expr.TensorVar.make(shape=[...])`를 사용하세요.

Example of L3 buffers for input feature map (ifm) and output feature map (ofm):
입력 기능 맵(ifm) 및 출력 기능 맵(ofm)에 대한 L3 버퍼의 예:
```python
    ifm, ofm = opInterface # For a binary op, opInterface unpacks into ifm, wts, ofm
    ifm_ddr_var = ifm.getTensorVar()
    ofm_ddr_var = ofm.getTensorVar()
```

##### AIE buffers
##### AIE 버퍼

Buffers located within the AIE are to be created in memory tiles (L2) and AIE cores (L1).
AIE 내에 위치한 버퍼는 메모리 타일(L2)과 AIE 코어(L1)에 생성됩니다.

**L2 buffers**: these buffers, located in AIE memory tiles, sit between DDR and the cores and serve as coarser-grain intermediate buffers.
**L2 버퍼**: AIE 메모리 타일에 위치한 이 버퍼는 DDR과 코어 사이에 위치하며 더 거친 중간 버퍼 역할을 합니다.
They are created using `TensorVar.make(...)`.
TensorVar.make(...)`를 사용하여 생성됩니다.

* Placement: the allocation of these buffers is manual; the location needs to be explicitly provided.
* 배치: 이러한 버퍼의 할당은 수동입니다. 위치를 명시적으로 제공해야 합니다.
* Placement constraints (see [AIE topology and VAIML overlay](#aie-topology-and-vaiml-overlay)):
* 배치 제약 조건([AIE 토폴로지 및 VAIML 오버레이](#aie-topology-and-vaiml-overlay) 참조):
  * Buffers whose contents are distributed to L1 across columns must be placed in mem tile 0, 1, or 2.
  * 내용이 여러 열에 걸쳐 L1에 배포되는 버퍼는 메모리 타일 0, 1 또는 2에 배치되어야 합니다.
  * Buffers whose contents are distributed to L1 across rows must be placed in mem tile 2, or 3.
  * 내용이 행 전체에 걸쳐 L1에 배포되는 버퍼는 메모리 타일 2 또는 3에 배치되어야 합니다.
  * Buffers used for L1 to L2 transfers must be placed in mem tile 0, 1, or 2.
  * L1에서 L2로의 전송에 사용되는 버퍼는 메모리 타일 0, 1 또는 2에 배치되어야 합니다.
* Capacity and layout:
* 용량 및 레이아웃:
  * Each overlay column has one L2 mem tile of size 512 KB.
  * 각 오버레이 열에는 크기가 512KB인 하나의 L2 mem 타일이 있습니다.
  * L2 buffers can be up to 1 MB; such buffers span the current and the next column of mem tiles.
  * L2 버퍼는 최대 1MB까지 가능합니다. 이러한 버퍼는 mem 타일의 현재 열과 다음 열에 걸쳐 있습니다.
* Single/double-buffering: both single- and double-buffering (ping-pong or PIPO) are supported.
* 단일/이중 버퍼링: 단일 및 이중 버퍼링(핑퐁 또는 PIPO)이 모두 지원됩니다.
* Overlapping of L2 buffers is currently not supported.
* L2 버퍼 중첩은 현재 지원되지 않습니다.
* Path performance:
* 경로 성능:
  * For column-wise L2 to L1 data distribution, placing data in L2 column 1 is the fastest (preferred), with columns 0 and 2 available at lower bandwidth.
  * L2에서 L1로의 열 단위 데이터 배포의 경우 L2 열 1에 데이터를 배치하는 것이 가장 빠르며(선호) 열 0과 2는 더 낮은 대역폭에서 사용할 수 있습니다.
  * For row-wise L2 to L1 data distribution, placing data in L2 column 3 is the fastest (preferred), with column 2 as a slower fallback.
  * 행별 L2에서 L1로의 데이터 배포의 경우 L2 열 3에 데이터를 배치하는 것이 가장 빠르며(선호) 열 2는 더 느린 대체입니다.
  * Prefer placing buffers accordingly when performance matters.
  * 성능이 중요한 경우 그에 따라 버퍼를 배치하는 것이 좋습니다.

**Using single- and double-buffering**: The number of buffers is determined by the number of provided addresses in the `location` parameter of `TensorExpr.make()`; each location is a `Location(...)` object.
**단일 및 이중 버퍼링 사용**: 버퍼 수는 `TensorExpr.make()`의 `location` 매개변수에 제공된 주소 수에 따라 결정됩니다. 각 위치는 `Location(...)` 객체입니다.

**Remark**: Double-buffering doubles the buffer size.
**참고**: 이중 버퍼링은 버퍼 크기를 두 배로 늘립니다.

Example (single memory tile):
예(단일 메모리 타일):
```python
ifm_ping_addr = 0x0       # PING buffer location
ifm_pong_addr = 0x3000    # PONG buffer location

ifm_mem_var = tensor_expr.TensorVar.make(
    locations=[
        tensor_expr.Location(0, 0, ifm_ping_addr),
        tensor_expr.Location(0, 0, ifm_pong_addr)
    ],
    shape=[L2_SHAPE],
    type=ifm.getDType(),
)
```

In larger tiling scripts, the same API is used to place different tensors on different memtiles:
더 큰 타일링 스크립트에서는 동일한 API를 사용하여 다양한 밈타일에 다양한 텐서를 배치합니다.

```python
ifm_mem_var = tensor_expr.TensorVar.make(
    locations=[
        tensor_expr.Location(0, 0, 0),
        tensor_expr.Location(0, 0, 0x60000),
    ],
    shape=[L2_M, L2_K],
    type=ifm.getDType(),
)
ofm_mem_var = tensor_expr.TensorVar.make(
    locations=[
        tensor_expr.Location(1, 0, 0x40000),
        tensor_expr.Location(2, 0, 0x20000),
    ],
    shape=[L2_K * L2_M],
    type=ofm.getDType(),
)
```

**Number of temporal iterations**: for those kernels that will iterate more than once, double-buffer rotations serve as synchronization barriers between iterations.
**시간적 반복 횟수**: 두 번 이상 반복되는 커널의 경우 이중 버퍼 회전은 반복 간의 동기화 장벽 역할을 합니다.
The number of rotations that are necessary for each buffer must be specified.
각 버퍼에 필요한 회전수를 지정해야 합니다.

The number of buffer rotations can be computed as follows:
버퍼 회전 수는 다음과 같이 계산할 수 있습니다.
* for L2 buffers used in L3 to L2 transfers: it is the ratio between the volume of data read from L3 and the volume of data written to L2.
* L3에서 L2로의 전송에 사용되는 L2 버퍼의 경우 L3에서 읽은 데이터 볼륨과 L2에 쓴 데이터 볼륨 간의 비율입니다.
* for L1 buffers used in L2 to L1 transfers, it is (both must agree):
* L2에서 L1로의 전송에 사용되는 L1 버퍼의 경우 다음과 같습니다(둘 다 동의해야 함).
  * the ratio between the volume of data read from L2 and the volume of data written to L1, and
  * L2에서 읽은 데이터 양과 L1에 쓴 데이터 양의 비율
  * the number of kernel calls divided by the kernel ratio for that L1 buffer (which is 1 if the argument is not asynchronous, see [Kernel ratios](#kernel-ratios)).
  * 커널 호출 수를 해당 L1 버퍼에 대한 커널 비율로 나눈 값입니다(인수가 비동기식이 아닌 경우 1입니다. [커널 비율](#kernel-ratios) 참조).

**Example:** computing the number of L3 to L2 transfers
**예:** L3에서 L2로의 전송 횟수 계산
```python
# The L3 tensor has size 64 * 32
l3_var = tensor_expr.TensorVar.make([64, 32], dtype)
# The L2 tensor is a tile of size 4 * 4
l2_var = tensor_expr.TensorVar.make([4, 4], dtype)
# Make tiles of size 4 * 4 of the L3 variable
l3_var_tiled = l3_var.TileBy([4, 4])
# Describe the L3 tensor tile by tile
spu.set_l3_to_l2_transfer(ifm_l3_l2_ch, l3_var_tiled, l2_var)
```
The total volume of data read from L3 is 2048 elements, and 16 elements per tile are written into L2.
L3에서 읽은 데이터의 총량은 2048개 요소이며, 타일당 16개 요소가 L2에 기록됩니다.
There will be 128 tiles of size 4x4 transferred from L3 to L2.
L3에서 L2로 전송되는 4x4 크기의 타일 128개가 있습니다.
You therefore need to specify:
따라서 다음을 지정해야 합니다.
```py
l2_var.setTemporalIterations(128)
```

**L1 buffers**: these buffers are located within the AIE cores and are the only ones the kernel has direct access to.
**L1 버퍼**: 이 버퍼는 AIE 코어 내에 있으며 커널이 직접 액세스할 수 있는 유일한 버퍼입니다.

* **Sizing**: these buffers need to be manually sized.
* **크기 조정**: 이러한 버퍼의 크기는 수동으로 조정해야 합니다.
* **Placement**: manual and automatic placement are available:
* **배치**: 수동 및 자동 배치가 가능합니다.
  * for manual placement, use the `tensor_expr.TensorVar.make(locations, shape, type)` constructor,
  * 수동 배치의 경우 `tensor_expr.TensorVar.make(locations, Shape, type)` 생성자를 사용하세요.
  * for automatic placement, use the `tensor_expr.TensorVar.make(shape, type, buffering)` constructor.
  * 자동 배치를 위해서는 `tensor_expr.TensorVar.make(shape, type, buffering)` 생성자를 사용하세요.
* **Single/double-buffering**: both single- and double-buffering are supported.
* **단일/이중 버퍼링**: 단일 버퍼링과 이중 버퍼링이 모두 지원됩니다.
* This must be specified:
* 다음과 같이 지정해야 합니다.
  * if manually placing the buffers, by the number of provided addresses in the `locations` argument (1 for single-buffering, 2 for double-buffering),
  * 버퍼를 수동으로 배치하는 경우 'locations' 인수에 제공된 주소 수(단일 버퍼링의 경우 1, 이중 버퍼링의 경우 2)
  * if using automatic placement, by selecting a strategy (`tensor_expr.BufferingStrategy.SingleBuffered`, `tensor_expr.BufferingStrategy.DoubleBuffered`).
  * 자동 배치를 사용하는 경우 전략(`tensor_expr.BufferingStrategy.SingleBuffered`, `tensor_expr.BufferingStrategy.DoubleBuffered`)을 선택합니다.

```python
# Manual L1 placement
ifm_mk_var = tensor_expr.TensorVar.make(
    [
        tensor_expr.Location(0, 0, ifm_ping_addr),
        tensor_expr.Location(0, 0, ifm_pong_addr),
    ],
    L1_SHAPE,
    ifm.getDType(),
)

# Automatic L1 placement
ofm_mk_var = tensor_expr.TensorVar.make(
    L1_SHAPE,
    ofm.getDType(),
    tensor_expr.BufferingStrategy.DoubleBuffered,
)
```

**Remarks**:
**비고**:
* Automatic L1 placement uses the tensor shape and dtype to size buffers correctly.
* 자동 L1 배치는 텐서 모양과 dtype을 사용하여 버퍼 크기를 올바르게 조정합니다.
* If the whole data fits into L1/L2, there is no need to use a ping-pong buffer.
* 전체 데이터가 L1/L2에 맞으면 핑퐁 버퍼를 사용할 필요가 없습니다.

**Constraints on locations**: Use explicit `Location(...)` values when you want full manual control over the L1 layout.
**위치 제약**: L1 레이아웃을 완전히 수동으로 제어하려면 명시적인 `Location(...)` 값을 사용하세요.
The following constraints apply:
다음 제약조건이 적용됩니다.
- **Overlapping:** Buffers should not overlap in one memory.
- **겹침:** 버퍼는 하나의 메모리에서 겹쳐서는 안 됩니다.
- **Total buffer size:** Total buffer size (number of elements × dtype size) must not exceed available memory.
- **총 버퍼 크기:** 총 버퍼 크기(요소 수 × dtype 크기)는 사용 가능한 메모리를 초과할 수 없습니다.
- **Mem tile location:** ifm and ofm must be placed in mem tile 0, 1 or 2, while wts must be placed in either mem tile 1, 2 or 3.
- **Mem 타일 위치:** ifm 및 ofm은 mem 타일 0, 1 또는 2에 배치되어야 하고, wts는 mem 타일 1, 2 또는 3에 배치되어야 합니다.
- **Reserved space**: The region `0xA000 - 0xDFFF` (16 KB) in L1 is reserved for stack & heap.
- **예약된 공간**: L1의 `0xA000 - 0xDFFF`(16KB) 영역은 스택 및 힙용으로 예약되어 있습니다.

#### Buffer tiling and traversal order transformation
#### 버퍼 타일링 및 순회 순서 변환

Buffers defined as `TensorVar` objects are traversed in their canonical order, from the innermost dimensions to the outermost.
'TensorVar' 객체로 정의된 버퍼는 가장 안쪽 차원에서 가장 바깥쪽 차원까지 표준 순서에 따라 순회됩니다.
It is possible to change the traversal order by composing operations (`TensorExpr`) on top of a `TensorVar`.
`TensorVar` 위에 작업(`TensorExpr`)을 구성하여 순회 순서를 변경할 수 있습니다.

The following transformations are available on the buffer and its traversal order:
버퍼 및 해당 순회 순서에 대해 다음 변환을 사용할 수 있습니다.

| Operation<br>작동 | Applies to<br>적용대상 | Effect<br>효과 | Yields<br>수확량 |
| --- | --- | --- | --- |
| `Reshape`<br>'모양 바꾸기' | Buffers<br>버퍼 | Returns a traversal order of the buffer as if its shape was the desired shape.<br>버퍼의 모양이 원하는 모양인 것처럼 버퍼의 순회 순서를 반환합니다. | Traversal order<br>순회 순서 |
| `Squash`<br>'스쿼시' | Buffers<br>버퍼 | Reshapes the buffer to a single dimension whose size is the product of all dimensions' sizes. Returns the canonical traversal order.<br>모든 차원의 크기를 곱한 크기의 단일 차원으로 버퍼의 형태를 변경합니다. 표준 순회 순서를 반환합니다. | Traversal order<br>순회 순서 |
| `Tile`<br>'타일' | Buffers, traversal orders<br>버퍼, 순회 순서 | Alters the traversal order (if a buffer, its canonical traversal order) so that dimension `index` is traversed as tiles of size `tileLen`, each tile starting at `k * stride` where `k` is the tile number. Creates an new outermost dimension that indexes the tiles.<br>순회 순서(버퍼인 경우 표준 순회 순서)를 변경하여 `index` 차원이 `tileLen` 크기의 타일로 순회하도록 하며, 각 타일은 `k * stride`에서 시작합니다. 여기서 `k`는 타일 번호입니다. 타일을 인덱싱하는 새로운 가장 바깥쪽 차원을 만듭니다. | Traversal order<br>순회 순서 |
| `TileBy`<br>'타일바이' | Buffers, traversal orders<br>버퍼, 순회 순서 | Specialized version of `Tile` that creates non-overlapping tiles of size `size`.<br>'size' 크기의 겹치지 않는 타일을 생성하는 'Tile'의 특수 버전입니다. | Traversal order<br>순회 순서 |
| `TileTo`<br>`타일로` | Buffers, traversal orders<br>버퍼, 순회 순서 | Specialized version of `Tile` that creates `numTiles` non-overlapping tiles of equal size.<br>동일한 크기의 겹치지 않는 'numTiles' 타일을 생성하는 'Tile'의 특수 버전입니다. | Traversal order<br>순회 순서 |
| `Transpose`<br>`조옮김` | Buffers, traversal orders<br>버퍼, 순회 순서 | Alters the traversal order (if a buffer, its canonical traversal order) by permuting the order of which its dimensions are traversed.<br>해당 차원이 순회되는 순서를 변경하여 순회 순서(버퍼인 경우 표준 순회 순서)를 변경합니다. | Traversal order<br>순회 순서 |
| `RightPadTo`<br>`RightPadTo` | Buffers, traversal orders<br>버퍼, 순회 순서 | Alters the traversal order (if a buffer, its canonical traversal order) by adding zero-padding after the last element of the selected dimension before the next dimension is traversed.<br>다음 차원을 순회하기 전에 선택한 차원의 마지막 요소 뒤에 제로 패딩을 추가하여 순회 순서(버퍼인 경우 표준 순회 순서)를 변경합니다. | Traversal order<br>순회 순서 |
| `Slice`<br>'슬라이스' | Buffers, traversal orders<br>버퍼, 순회 순서 | Alters the traversal order (if a buffer, its canonical traversal order) by traversing only elements from `offset` to `offset+length`.<br>`오프셋`에서 `오프셋+길이`까지 요소만 순회하여 순회 순서(버퍼인 경우 표준 순회 순서)를 변경합니다. | Traversal order<br>순회 순서 |
| `Repeat`<br>'반복' | Buffers, traversal orders<br>버퍼, 순회 순서 | Alters the traversal order (if a buffer, its canonical traversal order) by traversing it again the specified number of times.<br>지정된 횟수만큼 다시 순회하여 순회 순서(버퍼인 경우 표준 순회 순서)를 변경합니다. | Traversal order<br>순회 순서 |
| `SpatialDistribute`<br>`공간분산` | Buffers, traversal orders<br>버퍼, 순회 순서 | Cuts the traversal order into `tileCount` equal blocks on the specified `dimension`, and returns a list with each piece as a traversal order.<br>순회 순서를 지정된 `dimension`의 `tileCount` 동일한 블록으로 자르고 각 조각이 순회 순서로 포함된 목록을 반환합니다. | List of traversal orders<br>순회 순서 목록 |
| `SpatialDistribute2D`<br>`SpatialDistribute2D` | Buffers, traversal orders<br>버퍼, 순회 순서 | Cuts the traversal order into `tileCountA`, `tileCountB` equal blocks on the specified dimensions `dimA`, `dimB`. Returns a list of lists, with pieces along `dimA` on the outer lists and along `dimB` on the inner lists.<br>순회 순서를 지정된 차원 `dimA`, `dimB`의 `tileCountA`, `tileCountB` 동일한 블록으로 자릅니다. 외부 목록의 'dimA'와 내부 목록의 'dimB'에 따른 조각으로 목록 목록을 반환합니다. | List of lists of traversal orders<br>순회 순서 목록 목록 |

The reference API for all available calls is located in [the Tensor Expression API reference](tiling_api/TensorExpr.md).
사용 가능한 모든 호출에 대한 참조 API는 [Tensor Expression API 참조](tiling_api/TensorExpr.md)에 있습니다.

**Remarks:**
**비고:**
* Before tiling a tensor, you can zero-pad the tensor to align it to a multiple of the tile size using `RightPadTo`:
* 텐서를 타일링하기 전에 'RightPadTo'를 사용하여 텐서를 제로 패딩하여 타일 크기의 배수로 정렬할 수 있습니다.
```py
    var = tensor_expr.TensorVar.make(
        locations=[tensor_expr.Location(0, 0, 0)],
        shape=[7, 6],
        type=tensor_expr.OperandType.Int8,
    )
    # Pad var to make it a multiple of 4 on the inner dimension
    pad_var = var.RightPadTo(index=1, length=8)
    tiled_var = pad_var.TileBy(1, 4) # yields a traversal order of shape [2, 7, 4]
```
* `RightPadTo` cannot be used on L3 tensors.
* 'RightPadTo'는 L3 텐서에서 사용할 수 없습니다.
* Zero-padding is only available on L2 tensors.
* 제로 패딩은 L2 텐서에서만 사용할 수 있습니다.
* `Tile` supports overlapping tiles.
* '타일'은 겹치는 타일을 지원합니다.
* `TileBy` and `TileTo` always create tiles of equal size.
* `TileBy` 및 `TileTo`는 항상 동일한 크기의 타일을 만듭니다.

**Example**: tile a 64x32 tensor into 2-dimensional tiles of size 4x4 each.
**예**: 64x32 텐서를 각각 4x4 크기의 2차원 타일로 타일링합니다.
After this operation, the buffer is traversed as 16x8 tiles of size 4x4.
이 작업 후에 버퍼는 4x4 크기의 16x8 타일로 이동됩니다.

```python
# L3 variable, in its canonical access order.
# The outermost dimension (last in enumeration order) has size 16,
# the innermost (first in enumeration order) has size 32.
l3_var = tensor_expr.TensorVar.make([64, 32])
# Tile the innermost dimension with size 4. An outermost dimension of size 32 / 4 = 8 is created.
l3_var.TileBy(1, 4).getShape()
# Result: [8, 64, 4]
# Then tile the outermost dimension (that is now the second outermost, so it has index 1) by size 4.
l3_var.TileBy(1, 4).TileBy(1, 4).getShape()
# Result: [16, 8, 4, 4]

```

#### Kernel, number of calls and parameters
#### 커널, 호출 수 및 매개변수

It is necessary to specify the following information about the kernel:
커널에 대해 다음 정보를 지정해야 합니다.
* Its arguments, in the form of L1 buffers (`TensorExpr`) (`set_kernel_arguments`).
* L1 버퍼(`TensorExpr`)(`set_kernel_arguments`) 형식의 인수입니다.
* The C++ kernel has an additional argument, `lp_params` to pass the parameters;
* C++ 커널에는 매개변수를 전달하기 위한 추가 인수 'lp_params'가 있습니다.
* The values of the `lp_params` parameters (`set_kernel_params`);
* `lp_params` 매개변수의 값(`set_kernel_params`);
* The number of times the kernel is called (`set_kernel_nb_calls`);
* 커널이 호출된 횟수(`set_kernel_nb_calls`);
* The location of the kernel implementation and name of the function to be called (`set_kernel_function_name`, `set_kernel_impl`).
* 커널 구현 위치 및 호출할 함수 이름(`set_kernel_function_name`, `set_kernel_impl`).

**Example**:
**예**:
```py
    spu.set_kernel_arguments([ifm_mk, ofm_mk])
    spu.set_kernel_params([tileSize])
    spu.set_kernel_nb_calls(2*nb_temporal_iterations)
    spu.set_kernel_function_name("sample_op")
    spu.set_kernel_impl(Path("sample_op.cpp"))
```

**Computing the number of kernel calls**: The number of kernel calls is in general computable using the information from the data transfers:
**커널 호출 수 계산**: 커널 호출 수는 일반적으로 데이터 전송 정보를 사용하여 계산할 수 있습니다.
* how many tiles are transferred from L3 to L2 (L2 buffer rotation count), or from L2 to L3,
* L3에서 L2로(L2 버퍼 회전 횟수) 또는 L2에서 L3으로 전송되는 타일 수
* for each tile, how many iterations the data transferred into L1 needs to stay constant (also called "Kernel ratio", see [Kernel ratios](#kernel-ratios)).
* 각 타일에 대해 L1으로 전송된 데이터가 일정하게 유지되어야 하는 반복 횟수입니다("커널 비율"이라고도 함, [커널 비율](#kernel-ratios) 참조).
The number of kernel calls, in most cases, is the product of the number of tiles coming from L3 to L2, by the kernel ratio of the L1 buffer.
대부분의 경우 커널 호출 수는 L3에서 L2로 오는 타일 수와 L1 버퍼의 커널 비율을 곱한 것입니다.
If there are multiple L1 buffers, the number of kernel calls must be consistent whichever input / output is used.
L1 버퍼가 여러 개 있는 경우 어떤 입력/출력이 사용되든 커널 호출 수가 일관되어야 합니다.

**Example:**
**예:**
For instance, take the following IFM tiling:
예를 들어 다음 IFM 타일링을 사용합니다.

![Tiling example](resources/mk-rep-calc.svg)

* The L3 buffer of shape [5, 2, 8, 4, 64] is split into 5x2 = 10 L2 tiles of shape [8, 4, 64],
* [5, 2, 8, 4, 64] 모양의 L3 버퍼는 5x2 = 10개의 [8, 4, 64] 모양의 L2 타일로 분할됩니다.
* Each L2 tile of shape [8, 4, 64] is reshaped into [2, 4, 4, 64], then split into 2x4x4 = 32 L1 tiles of shape [64],
* [8, 4, 64] 모양의 각 L2 타일은 [2, 4, 4, 64]로 모양이 변경된 다음 2x4x4 = 32개의 [64] 모양의 L1 타일로 분할됩니다.
* The L1 are then spatially distributed: each of the 4x4 AIE cores processes one L1 tile per kernel call.
* 그런 다음 L1은 공간적으로 분산됩니다. 각 4x4 AIE 코어는 커널 호출당 하나의 L1 타일을 처리합니다.
* Every L2 tile is therefore processed in 2 temporal steps (the first 16 tiles, then the next 16).
* 따라서 모든 L2 타일은 2개의 시간적 단계(처음 16개 타일, 다음 16개 타일)로 처리됩니다.

Therefore the number of kernel calls, for each core, is:
따라서 각 코어에 대한 커널 호출 수는 다음과 같습니다.
```
l3_shape = 5 * 2 * 8 * 4 * 64  # product of dimensions
l2_shape = 8 * 4 * 64          # product of dimensions
l1_shape = 64
nb_parallel_tiles = 4 * 4

nb_calls_per_core = (l3_shape / l2_shape) * (l2_shape / (l1_shape * nb_parallel_tiles)) = 20
```

One therefore sets the kernel calls:
따라서 커널 호출을 설정합니다.
```py
    spu.set_kernel_nb_calls(20)
```

**Passing extra parameters to the kernel via `lp_params`**
**`lp_params`를 통해 커널에 추가 매개변수 전달**

There may be decisions in the tiling script that need to be propagated to the kernel, for example tile sizes or loop counts.
타일링 스크립트에는 타일 크기나 루프 수와 같이 커널에 전파해야 하는 결정이 있을 수 있습니다.
You can use the `lp_params` argument in `set_kernel_params()` to communicate these values directly.
`set_kernel_params()`에서 `lp_params` 인수를 사용하여 이러한 값을 직접 전달할 수 있습니다.

**Example**:
**예**:
```python
    # Direct passing of an attribute value from the ONNX op to the kernel parameters
    axis = opInterface.attributes["axis"]
    # Passing of a computed value to the kernel parameters
    loop_trip_count = total_iterations / nb_cores
    lp_params = [axis, loop_trip_count]
    spu.set_kernel_params(lp_params)
```
The values given by the above snippet are then obtained in the C++ kernel:
위의 코드 조각에서 제공한 값은 C++ 커널에서 가져옵니다.
```cpp
constexpr int nb_lp_params = 2;
void negate_kernel(
adf::input_buffer_conf<dtype_ifm_0, bpc_sync_0d> &__restrict ifm0, adf::output_buffer_conf<dtype_ofm_0, bpc_sync_0d> &__restrict ofm,
const uint32_t (&layer_params)[nb_lp_params]) {
  uint32_t axis = lp_params[0];
  uint32_t loop_trip_count = lp_params[1];
  // [...]
}
```

#### Data Movement Setters
#### 데이터 이동 설정자

The data movement setters configure the overlay's DMA channels (see [AIE topology and VAIML overlay](#aie-topology-and-vaiml-overlay)).
데이터 이동 설정자는 오버레이의 DMA 채널을 구성합니다([AIE 토폴로지 및 VAIML 오버레이](#aie-topology-and-vaiml-overlay) 참조).
If using a Nx4x4 overlay, each setter configures the channels of one 4x4 grid.
Nx4x4 오버레이를 사용하는 경우 각 설정자는 하나의 4x4 그리드의 채널을 구성합니다.

These setters expect to be given a buffer and its traversal order, as `TensorExpr` objects (or lists of such).
이러한 setter에는 `TensorExpr` 객체(또는 그러한 목록)와 같은 버퍼와 해당 순회 순서가 제공될 것으로 예상합니다.

The following schematic shows the data flow in each 4x4 grid of AI Engine processors, and what each setter configures:
다음 도식은 AI Engine 프로세서의 각 4x4 그리드의 데이터 흐름과 각 설정자가 구성하는 내용을 보여줍니다.

![Data movement flow: L3 → L2 → L1 kernel → L2 → L3](resources/data-movement-flow.svg)

See also [the DMA engine documentation](https://docs.amd.com/r/en-US/ug1603-ai-engine-ml-kernel-graph/Memory-and-DMA-Programming) for details on which features each DMA engine supports.
각 DMA 엔진이 지원하는 기능에 대한 자세한 내용은 [DMA 엔진 설명서](https://docs.amd.com/r/en-US/ug1603-ai-engine-ml-kernel-graph/Memory-and-DMA-Programming)를 참조하세요.

**Remark:** Direct streaming into the AIE cores from/to L3 (skipping L2) is not supported; explicit data movement through L2 is required.
**참고:** L3 간 AIE 코어로의 직접 스트리밍(L2 건너뛰기)은 지원되지 않습니다. L2를 통한 명시적인 데이터 이동이 필요합니다.

##### L3 <-> L2 transfers: `set_l3_to_l2_transfer`, `set_l2_to_l3_transfer`
##### L3 <-> L2 전송: `set_l3_to_l2_transfer`, `set_l2_to_l3_transfer`

These functions configure the data transfers between the DDR and the memory tiles.
이러한 기능은 DDR과 메모리 타일 간의 데이터 전송을 구성합니다.

Example:
예:
```py
    ifm, wts, ofm = opInterface
    # Obtain L3 to L2 channels: for the ifm parameter, select tile 0; for the wts, select tile 3
    ifm_l3_l2_ch = [ch for ch in spu.get_l3_to_l2_channels() if ch.mem_tile_port.tile_col == 0][0]
    wts_l3_l2_ch = [ch for ch in spu.get_l3_to_l2_channels() if ch.mem_tile_port.tile_col == 3][0]

    spu.set_l3_to_l2_transfer(ifm_l3_l2_ch, ifm.getTensorVar().TileBy([8*4*4]), ifm_l2_var)
    spu.set_l3_to_l2_transfer(wts_l3_l2_ch, wts.getTensorVar(), wts_l2_var)

    # Obtain L2 to L3 channel (these are from tile 2)
    ofm_l3_l2_ch = spu.get_l2_to_l3_channels()[0]
    spu.set_l2_to_l3_transfer(wts_l3_l2_ch, ofm_l2_var, ofm.getTensorVar().TileBy([8*4*4]))
```

**Remark:** Only a single buffer access pattern (`TensorExpr`) on both the read and write sides is supported.
**비고:** 읽기 및 쓰기 측 모두에서 단일 버퍼 액세스 패턴(`TensorExpr`)만 지원됩니다.

##### L2 -> L1 transfers: `set_l2_to_l1_transfer`
##### L2 -> L1 전송: `set_l2_to_l1_transfer`

The `set_l2_to_l1_transfer` function configures the data movement from L2 buffers to L1 buffers of each core of the AIE, row-wise or column-wise.
'set_l2_to_l1_transfer' 함수는 행 방향 또는 열 방향으로 AIE의 각 코어의 L2 버퍼에서 L1 버퍼로의 데이터 이동을 구성합니다.
All cores along either a row or a column, depending on the selected channel, receive the same data in their L1 buffers (see [AIE topology and VAIML overlay](#aie-topology-and-vaiml-overlay)).
선택한 채널에 따라 행 또는 열에 있는 모든 코어는 L1 버퍼에서 동일한 데이터를 수신합니다([AIE 토폴로지 및 VAIML 오버레이](#aie-topology-and-vaiml-overlay) 참조).

This setter supports:
이 세터는 다음을 지원합니다.
* one read-side, one write-side access pattern (`TensorExpr`); in this case, the same write pattern is applied to all the cores targeted by the channel.
* 하나는 읽기 측, 하나는 쓰기 측 액세스 패턴(`TensorExpr`)입니다. 이 경우 채널의 대상이 되는 모든 코어에 동일한 쓰기 패턴이 적용됩니다.
* one read-side access pattern and a list of write-side access patterns, one per core targeted by the channel.
* 하나의 읽기 측 액세스 패턴과 쓰기 측 액세스 패턴 목록(채널 대상 코어당 하나씩).
* Each channel targets 4 cores.
* 각 채널은 4개의 코어를 대상으로 합니다.

Example:
예:
```py
    # Obtain L2 to L1 channels targeting the columns
    ifm_l2_l1_channels = spu.get_l2_to_l1_channels(broadcast=tensor_expr.broadcast_on.COLUMNS)
    # There is one channel per column; we transfer one part of the IFM to each column.
    # Since the column-wise channels start from tile 1, ifm_l2_var must be placed in
    # tile 0, 1 or 2.
    for column_index, column_channel in enumerate(ifm_l2_l1_channels):
        spu.set_l2_to_l1_transfer(column_channel, ifm_l2_var.TileTo([4])[column_index], ifm_l1_var)

    # Obtain L2 to L1 channels targeting the rows
    wts_l2_l1_channels = spu.get_l2_to_l1_channels(broadcast=tensor_expr.broadcast_on.ROWS)
    # There is one channel per row; we transfer one part of the weights to each row.
    # Since the row-wise channels start from tile 3, wts_l2_var must be placed in
    # tile 2 or 3.
    for row_index, row_channel in enumerate(wts_l2_l1_channels):
        spu.set_l2_to_l1_transfer(row_channel, wts_l2_var.TileTo([4])[row_index], wts_l1_var)
```

##### L1 -> L2 transfers: `set_l1_to_l2_transfer`
##### L1 -> L2 전송: `set_l1_to_l2_transfer`

The `set_l1_to_l2_transfer` function configures the data movement from L1 buffers in AIE cores to L2 buffers.
'set_l1_to_l2_transfer' 함수는 AIE 코어의 L1 버퍼에서 L2 버퍼로의 데이터 이동을 구성합니다.

This setter configures, for a given L1 to L2 channel, the individual read access patterns from each core, and the corresponding write access pattern into an L2 buffer.
이 설정기는 지정된 L1~L2 채널에 대해 각 코어의 개별 읽기 액세스 패턴과 해당 쓰기 액세스 패턴을 L2 버퍼에 구성합니다.
There is one L2 to L1 channel per column in a stamp.
스탬프에는 열당 하나의 L2~L1 채널이 있습니다.
This setter supports:
이 세터는 다음을 지원합니다.
* one read-side, a list of write-side access patterns (`TensorExpr`), one per core at the origin of the channel; in this case, the same read-from-L1 pattern is applied to all the cores targeted by the channel; and each core has its own write-to-L2 pattern.
* 하나는 읽기 측, 쓰기 측 액세스 패턴 목록(`TensorExpr`), 채널 원본의 코어당 하나입니다. 이 경우 동일한 L1 읽기 패턴이 채널의 대상이 되는 모든 코어에 적용됩니다. 각 코어에는 자체 L2 쓰기 패턴이 있습니다.
* a list of read-side access patterns and a list of write-side access patterns, one per core at the origin of the channel; in this case, each core has its own read and write pattern.
* 읽기 측 액세스 패턴 목록과 쓰기 측 액세스 패턴 목록(채널 원본에서 코어당 하나씩) 이 경우 각 코어에는 자체 읽기 및 쓰기 패턴이 있습니다.

**Example 1:** simple spatial distribution over rows and columns of a 1-D buffer
**예 1:** 1차원 버퍼의 행과 열에 대한 간단한 공간 분포
```py
    ofm_l1_l2 = spu.get_l1_to_l2_channels()
    # From an L2 TensorVar, make a COLS * ROWS grid, then using SpatialDistribute2D,
    # split the grid into COLS * ROWS blocks of equal size. Each block contains
    # the result of one specific core
    # Spatial distribution is done first on columns, then on rows, yielding an
    # array (along columns) of 4 arrays (along rows).
    distributed_ofm_l2 = ofm_mem.Reshape(
        [OVERLAY_COLS, OVERLAY_ROWS, tileSize]
    ).SpatialDistribute2D(
        dimA=0, dimB=1, tileACount=OVERLAY_COLS, tileBCount=OVERLAY_ROWS
    )
    for col, ch in enumerate(ofm_l1_l2):
        spu.set_l1_to_l2_transfer(
            ch,
            ofm_mk,
            distributed_ofm_l2[col], # Array of 4 blocks, one per row (i.e. per core)
        )
```

**Example:**
**예:**
Consider an OFM whose L2 buffer is shaped `[32, 8, 3]`.
L2 버퍼가 '[32, 8, 3]' 모양인 OFM을 생각해 보세요.
We divide the buffer for OFM writes:
OFM 쓰기를 위해 버퍼를 나눕니다.
- The outermost dimension (size 32) is split into 4 chunks of size 8 → *giving 4 columns*.
- 가장 바깥쪽 차원(크기 32)은 크기 8의 4개 청크로 분할됩니다 → *4개의 열 제공*.
- The next (second outermost) dimension is split into 4 chunks of size 2 → *giving 4 rows*,
- 다음(두 번째로 가장 바깥쪽) 차원은 크기 2의 4개 청크로 분할됩니다 → *4개 행 제공*,
- The innermost dimension stays 3.
- 가장 안쪽 차원은 3으로 유지됩니다.

![Spatial distribution](resources/automatic-spatial-distribution.svg)

Using `SpatialDistribute2D`, we divide the OFM into 16 tiles, and create a pattern for each of them:
`SpatialDistribute2D`를 사용하여 OFM을 16개의 타일로 나누고 각 타일에 대한 패턴을 만듭니다.
```py
# Get 16 access patterns as a list of 4 lists
ofm_spatial_dist = ofm_l2.SpatialDistribute2D(
    dimA=0, dimB=1, tileACount=4, tileBCount=4
)
```
Each core at position `(col, row)` in the grid will write to the slice
그리드의 '(열, 행)' 위치에 있는 각 코어는 슬라이스에 씁니다.
```
L2[8*col : 8*(col+1), 2*row : 2*(row+1), 0 : 3 (entire slice)]
```
or otherwise written, as the following list of slices (one per core, where `*` represents the entire range 0 to 3):
또는 다음과 같은 슬라이스 목록으로 작성됩니다(코어당 하나, 여기서 '*'는 0에서 3까지의 전체 범위를 나타냄).
```python
[
    # [[column 0], [column 1], [column 2], [column 3]]
    [[0:8, 0:2, *], [0:8, 2:4, *], [0:8, 4:6, *], [0:8, 6:8, *]], # row 0
    [[8:16, 0:2, *], [8:16, 2:4, *], [8:16, 4:6, *], [8:16, 6:8, *]], # row 1
    [[16:24, 0:2, *], [16:24, 2:4, *], [16:24, 4:6, *], [16:24, 6:8, *]], # row 2
    [[24:32, 0:2, *], [24:32, 2:4, *], [24:32, 4:6, *], [24:32, 6:8, *]], # row 3
]
```
Each slice is of size `[8, 2, 3]`.
각 슬라이스의 크기는 `[8, 2, 3]`입니다.
They can therefore be traversed by defining an L1 variable of dimension `[8, 2, 3]`:
따라서 차원 `[8, 2, 3]`의 L1 변수를 정의하여 탐색할 수 있습니다.
```py
ofm_l1 = tensor_expr.TensorVar.make(
    [8, 2, 3],
    ofm.getDType(),
    tensor_expr.BufferingStrategy.DoubleBuffered,
)
for col, ch in enumerate(spu.get_l1_to_l2_channels())
    # Set for each column, the 4 writes corresponding to the cores along that column
    spu.set_l1_to_l2_transfer(ch, ofm_l1, ofm_spatial_dist[col])
```

#### Kernel Ratios
#### 커널 비율

The *kernel ratios* need to be specified for those inputs and outputs which, in the kernel, have been defined as asynchronous.
커널에서 비동기식으로 정의된 입력 및 출력에 대해 *커널 비율*을 지정해야 합니다.

The ratio for a given input is defined as the number of calls of the kernel between double-buffer rotations.
주어진 입력에 대한 비율은 이중 버퍼 회전 사이의 커널 호출 수로 정의됩니다.
The kernel:
커널:
* **must** acquire the buffer on the first call after a double buffer rotation,
* **반드시** 이중 버퍼 회전 후 첫 번째 호출에서 버퍼를 획득해야 합니다.
* **must** release the buffer before the next double-buffer rotation,
* **반드시** 다음 이중 버퍼 회전 전에 버퍼를 해제해야 합니다.
* **must** hold the buffer for exactly the number of calls declared in the tiling script.
* **반드시** 타일링 스크립트에 선언된 호출 수만큼 버퍼를 보유해야 합니다.

Kernel ratios are passed using the `ratio` argument of the `set_l2_to_l1_transfer` and `set_l1_to_l2_transfer` functions:
커널 비율은 `set_l2_to_l1_transfer` 및 `set_l1_to_l2_transfer` 함수의 `ratio` 인수를 사용하여 전달됩니다.
```py
    for column_index, column_channel in enumerate(ifm_l2_l1_channels):
        spu.set_l2_to_l1_transfer(column_channel, ifm_l2_var.TileTo([4])[column_index], ifm_l1_var, ratio=3) # L1 variable is rotated every 3 kernel calls

    for row_index, row_channel in enumerate(wts_l2_l1_channels):
        spu.set_l2_to_l1_transfer(row_channel, wts_l2_var.TileTo([4])[row_index], wts_l1_var, ratio=7) # L1 variable is rotated every 7 kernel calls
```

**Remarks:**
**비고:**
* Failure to specify kernel ratios of asynchronous inputs or outputs is an error.
* 비동기 입력 또는 출력의 커널 비율을 지정하지 않으면 오류가 발생합니다.
* Kernel ratios may, directly or indirectly, depend on the size of the input tensors; for instance, a kernel may hold an L1 input buffer constant (e.g., a small weight array) while another large-size tiled input tensor is being traversed to another L1 buffer.
* 커널 비율은 입력 텐서의 크기에 따라 직접 또는 간접적으로 달라질 수 있습니다. 예를 들어, 다른 대형 타일 입력 텐서가 다른 L1 버퍼로 이동하는 동안 커널은 L1 입력 버퍼 상수(예: 작은 가중치 배열)를 보유할 수 있습니다.

**Important note**: the number of kernel calls, the kernel ratios and the number of iterations for each L2 buffer must be in agreement.
**중요 사항**: 각 L2 버퍼에 대한 커널 호출 수, 커널 비율 및 반복 횟수가 일치해야 합니다.
Otherwise, your custom op will stall.
그렇지 않으면 사용자 정의 작업이 중단됩니다.

Example:
예:
```py
    ifm_mem = tensor_expr.TensorVar.make(
        locations=[tensor_expr.Location(0, 0, 0), tensor_expr.Location(0, 0, 0x10000)],
        shape=[OVERLAY_COLS, OVERLAY_ROWS, tileSize],
        type=ifm.getDType(),
    )
    # Number of buffer rotations
    ifm_mem.setTemporalIterations(nb_temporal_iterations)
    ifm_l2_l1 = spu.get_l2_to_l1_channels(broadcast=tensor_expr.broadcast_on.COLUMNS)
    for col, ch in enumerate(ifm_l2_l1):
        spu.set_l2_to_l1_transfer(
            ch,
            ifm_mem.TileTo(index=0, numTiles=OVERLAY_COLS)[col],
            ifm_mk,
            ratio=2, # 2 uses per buffer rotation
        )
    # Total number of kernel calls : 2 uses per rotation * number of rotations
    spu.set_kernel_nb_calls(2*nb_temporal_iterations)
```

### Tiling Script Design Tips
### 타일링 스크립트 디자인 팁

The following guidance complements the sections above.
다음 지침은 위 섹션을 보완합니다.

#### Validation and common pitfalls
#### 검증 및 일반적인 함정
- Ensure ping/pong buffers do not overlap and remain outside the reserved L1 stack/heap region `0xA000 - 0xDFFF` (16 KB).
- 핑/퐁 버퍼가 겹치지 않고 예약된 L1 스택/힙 영역 '0xA000 - 0xDFFF'(16KB) 외부에 남아 있는지 확인하세요.
- Align tile sizes with kernel vectorization to avoid scalar tails or misalignment penalties.
- 스칼라 테일이나 정렬 불량 페널티를 방지하려면 타일 크기를 커널 벡터화와 정렬하세요.
- Use the [logging/keep-outputs options](README.md#customop-execution-and-debugging-flags) to debug tiling and data movement.
- 타일링 및 데이터 이동을 디버그하려면 [로깅/출력 유지 옵션](README.md#customop-execution-and-debugging-flags)을 사용하세요.

## Troubleshooting tiling
## 타일링 문제 해결

### Frequently encountered errors
### 자주 발생하는 오류

#### `IO Buffer 'mk[i][j].in[N]' is required to have a total size which is multiple of 4 bytes` (aiecompiler 77-23295)
#### `IO 버퍼 'mk[i][j].in[N]'은 4바이트의 배수인 총 크기를 가져야 합니다.`(aiecompiler 77-23295)

The DMA access granularity of the AI Engine is 32 bits.
AI 엔진의 DMA 액세스 단위는 32비트입니다.
Therefore, `aiecompiler` rejects any IO buffer whose byte size is not a whole multiple of 4.
따라서 'aiecompiler'는 바이트 크기가 4의 배수가 아닌 모든 IO 버퍼를 거부합니다.
For `bfloat16` buffers for instance, this means an **even element count** is required.
예를 들어 `bfloat16` 버퍼의 경우 이는 **짝수 요소 개수**가 필요함을 의미합니다.

Although the compiler auto-pads the innermost dimension of the ONNX operator input to a 32-bit boundary (see [Obtaining ONNX tensor shapes in the tiling script](#obtaining-onnx-tensor-shapes-in-the-tiling-script)), one still needs to make sure the accessed data is aligned to 32 bits per element.
컴파일러는 ONNX 연산자 입력의 가장 안쪽 차원을 32비트 경계로 자동 채웁니다([타일링 스크립트에서 ONNX 텐서 모양 얻기](#obtaining-onnx-tensor-shapes-in-the-tiling-script) 참조). 하지만 여전히 액세스된 데이터가 요소당 32비트로 정렬되어 있는지 확인해야 합니다.

**Example:** aligning an L1 buffer to 32 bits
**예:** L1 버퍼를 32비트로 정렬
```python
(K,) = wts.getPaddedShape()      # Channel WTS is of type bf16
assert K % 2 == 0, "An even number of weights is required"
ifm_l1_shape = [OVERLAY_ROWS * T + K - 1]   # Wrong: exact size, but not 32-bit aligned
ifm_l1_shape = [OVERLAY_ROWS * T + K]       # Right: bigger than needed, 32-bit aligned
```

### Compiler flags and environment variables
### 컴파일러 플래그 및 환경 변수

The same environment variables as in the [general custom ops documentation](README.md#customop-execution-and-debugging-flags) are used to debug the tiling.
[일반 사용자 정의 작업 문서](README.md#customop-execution-and-debugging-flags)와 동일한 환경 변수가 타일링을 디버깅하는 데 사용됩니다.
Errors in the tiling script are shown in the compiler's standard output.
타일링 스크립트의 오류는 컴파일러의 표준 출력에 표시됩니다.

### Using Python printout
### Python 인쇄물 사용

During compilation, the compiler runs your tiling script and calls `getTiling()`.
컴파일하는 동안 컴파일러는 타일링 스크립트를 실행하고 `getTiling()`을 호출합니다.
Any output from `print()` (or other writes to standard output) in that script is emitted on the **compiler’s standard output**—the same stream as the rest of the compile log—not in a separate file.
해당 스크립트에 있는 `print()`의 모든 출력(또는 표준 출력에 대한 기타 쓰기)은 별도의 파일이 아닌 **컴파일러의 표준 출력**(컴파일 로그의 나머지 부분과 동일한 스트림)으로 내보내집니다.

Use this to sanity-check shapes, tile sizes, loop counts, and `lp_params` before you inspect generated headers or run simulation.
생성된 헤더를 검사하거나 시뮬레이션을 실행하기 전에 이를 사용하여 모양, 타일 크기, 루프 수 및 `lp_params` 상태를 확인하세요.
For example:
예를 들면:

```python
def getTiling(opInterface: tensor_expr.OperatorInfo, tiling: tensor_expr.AieConfig):
    ifm, wts, ofm = opInterface
    print(f"IFM shape: {ifm.getShape()}")
    print(f"OFM shape: {ofm.getShape()}")
    # ... configure tiling ...
    print(f"LP params: {lp_params}")
```

Recompile after changing the script; the printout appears on the next compile.
스크립트를 변경한 후 다시 컴파일하십시오. 인쇄물은 다음 컴파일에 나타납니다.
If you capture compile output to a log file, search that log for your messages.
컴파일 출력을 로그 파일로 캡처하는 경우 해당 로그에서 메시지를 검색하세요.

For ground-truth DMA configuration after a successful compile, use the generated tiling header described in the section on checking the data movement header (see [Checking the produced data movement header](#checking-the-produced-data-movement-header)).
성공적인 컴파일 후 실측 DMA 구성을 위해 데이터 이동 헤더 확인 섹션에 설명된 생성된 타일링 헤더를 사용합니다([생성된 데이터 이동 헤더 확인](#checking-the-produced-data-movement-header) 참조).

### Checking the produced data movement header
### 생성된 데이터 이동 헤더 확인

Upon compilation, the `<compilation output dir>/custom_ops` directory contains the generated tiling headers.
컴파일 시 `<컴파일 출력 디렉토리>/custom_ops` 디렉토리에는 생성된 타일링 헤더가 포함됩니다.
There is one for your custom op, whose name will be something like `mydomain_myop__myop_0_multi_layer_overlay.h`.
사용자 정의 작업을 위한 이름이 `mydomain_myop__myop_0_multi_layer_overlay.h`와 같은 것이 있습니다.

Example:
예:
```cpp
#ifndef _MYDOMAIN_MYNEGATE__MYNEGATE_NEGATED_0_MULTI_LAYER_OVERLAY_
#define _MYDOMAIN_MYNEGATE__MYNEGATE_NEGATED_0_MULTI_LAYER_OVERLAY_
#include "L2/include/ml_adf/common.h"
#include "L2/include/ml_adf/ml_adf_overlay_4x4.h"
#include "L2/include/ml_adf/tg_utils.h"
#include "L2/include/ml_adf/tiling_def_4x4.h"
#include <adf.h>

namespace custom_ops {
constexpr int mydomain_mynegate_0_lp_size = 1;

struct mydomain_mynegate_0_params_t {
  std::vector<ml_adf_overlay_4x4::tiling_params_t> tiling_params;
};
} // namespace custom_ops

namespace custom_ops {
template <typename dtype_ifm_0, typename dtype_ofm_0>
void negate_kernel(
adf::input_buffer_conf<dtype_ifm_0, bpc_sync_0d> &__restrict ifm0, adf::output_buffer_conf<dtype_ofm_0, bpc_sync_0d> &__restrict ofm, const uint32_t (&layer_params)[mydomain_mynegate_0_lp_size]);

// One class per stamp and per phase. Multi-stamp kernels will have several of these.
template <int stack_location, int stack_size, int heap_size>
class mydomain_mynegate_0_stamp0_phase0 : public adf::graph {
public:
  // L2 variables. Check these match with the number of `TensorVar` in L2 you created and used (unused L2 variables in the Python script do not appear)
  adf::shared_buffer<bfloat16> shared_buffer_0;
  adf::shared_buffer<bfloat16> shared_buffer_1;
  pktorderedmerge<4> merge_0;
  pktorderedmerge<4> merge_1;
  pktorderedmerge<4> merge_2;
  pktorderedmerge<4> merge_3;
  adf::kernel mk[4][4];

  mydomain_mynegate_0_stamp0_phase0(adf::external_buffer<bfloat16> &external_var_0, adf::external_buffer<bfloat16> &external_var_1) {
    using namespace adf;
    const std::vector<uint32_t> lp_param = {128};

    // Internal shared buffers declaration
    // L2 variable for the IFM
    shared_buffer_0 = adf::shared_buffer<bfloat16>::create({128}, /*numWrites*/1, /*numReads*/4); // This L2 variable is written to once and read 4 times, once per column. You can check how many `set_l3_to_l2_transfer` and `set_l2_to_l1_transfer` calls use this variable to verify.
    adf::location<adf::buffer>(shared_buffer_0) = mllib_graphs::construct_mem_loc<0>(std::vector<std::vector<uint32_t>>{{0, 0, 0}}); // Placement of the L2 variable, check that this corresponds to the placement you specified
    adf::num_buffers(shared_buffer_0) = 1; // Single-buffering (count how many Locations you gave to the TensorVar)
    // L2 variable for the OFM
    shared_buffer_1 = adf::shared_buffer<bfloat16>::create({128}, /*numWrites*/4, /*numReads*/1); // One write per column, but one single read (L2 to L3)
    adf::location<adf::buffer>(shared_buffer_1) = mllib_graphs::construct_mem_loc<0>(std::vector<std::vector<uint32_t>>{{1, 0, 0}});
    adf::num_buffers(shared_buffer_1) = 1;

    // adf.compute -> kernels
    mk[0][0] = adf::kernel::create(negate_kernel<bfloat16, bfloat16>); // Name of your kernel and template parameters
    adf::source(mk[0][0]) = "/path/to/custom_op/custom_op.cpp";
    adf::repetition_count(mk[0][0]) = 1; // set_kernel_nb_calls sets this
    adf::runtime<adf::ratio>(mk[0][0]) = 0.8;
    adf::location<adf::kernel>(mk[0][0]) = adf::tile(0, 0);
    adf::async(mk[0][0].in[0]); // First argument of the kernel is async
    adf::async(mk[0][0].in[1]); // the last argument - designates `lp_params` - is always async.
    adf::initial_value(mk[0][0].in[1]) = lp_param;
    adf::async_repetition(mk[0][0].in[0]) = 2; // Kernel ratio for the first argument
    adf::location<adf::stack>(mk[0][0]) = {adf::address(0, 0, stack_location)};
    adf::stack_size(mk[0][0]) = stack_size;
    adf::heap_size(mk[0][0]) = heap_size;
    mk[0][1] = adf::kernel::create(negate_kernel<bfloat16, bfloat16>);
    // [...]
    // The above kernel settings repeated 15 times (one setting per core)

    // Connections
    // L3 to L2 transfer
    // 1:1 bind external_var_0.out[0] -> shared_buffer_0.in[0]
    // The two locations below come from the L3 to L2 channel you have selected (from get_l3_to_l2_channels())
    adf::location<adf::dma>(external_var_0.out[0]) = adf::dma_channel(adf::shim_tile, 0, 0, 0);
    adf::location<adf::dma>(shared_buffer_0.in[0]) = adf::dma_channel(adf::memory_tile, 0, 0, 0);
    adf::connect(external_var_0.out[0], shared_buffer_0.in[0]);
    // The access patterns, generated from the TensorExpr expressions.
    adf::read_access(external_var_0.out[0]) = {adf::tiling({.buffer_dimension = {128},
                                                            .tiling_dimension = {128},
                                                            .offset = {0},
                                                            .tile_traversal = {}
                                                           })};
    adf::write_access(shared_buffer_0.in[0]) = {adf::tiling({.buffer_dimension = {128},
                                                             .tiling_dimension = {128},
                                                             .offset = {0},
                                                             .tile_traversal = {}
                                                            })};
    // L2 to L1 transfer
    // 1:N dispatch shared_buffer_0.out[0] -> {mk[0][0].in[0], mk[0][1].in[0], mk[0][2].in[0], mk[0][3].in[0]}
    adf::location<adf::dma>(shared_buffer_0.out[0]) = adf::dma_channel(adf::memory_tile, 1, 0, 0); // Column-wise channel distribution starts from column 1, even if your variable is placed in column 0 or 2
    // The access pattern you have specified using Tensor Expressions
    adf::read_access(shared_buffer_0.out[0]) = {adf::tiling({.buffer_dimension = {128},
                                                             .tiling_dimension = {128},
                                                             .offset = {0},
                                                             .tile_traversal = {}
                                                            })};
    // Specification of the L1 buffers
    adf::dimensions(mk[0][0].in[0]) = {128};
    adf::single_buffer(mk[0][0].in[0]);
    adf::location<adf::buffer>(mk[0][0].in[0]) = {adf::address(0, 0, 0)};
    adf::location<adf::dma>(mk[0][0].in[0]) = adf::dma_channel(adf::aie_tile, 0, 0, 0);
    adf::connect(shared_buffer_0.out[0], mk[0][0].in[0]);
    adf::dimensions(mk[0][1].in[0]) = {128};
    adf::single_buffer(mk[0][1].in[0]);
    adf::location<adf::buffer>(mk[0][1].in[0]) = {adf::address(0, 1, 0)};
    adf::location<adf::dma>(mk[0][1].in[0]) = adf::dma_channel(adf::aie_tile, 0, 1, 0);
    adf::connect(shared_buffer_0.out[0], mk[0][1].in[0]);
    adf::dimensions(mk[0][2].in[0]) = {128};
    adf::single_buffer(mk[0][2].in[0]);
    adf::location<adf::buffer>(mk[0][2].in[0]) = {adf::address(0, 2, 0)};
    adf::location<adf::dma>(mk[0][2].in[0]) = adf::dma_channel(adf::aie_tile, 0, 2, 0);
    adf::connect(shared_buffer_0.out[0], mk[0][2].in[0]);
    adf::dimensions(mk[0][3].in[0]) = {128};
    adf::single_buffer(mk[0][3].in[0]);
    adf::location<adf::buffer>(mk[0][3].in[0]) = {adf::address(0, 3, 0)};
    adf::location<adf::dma>(mk[0][3].in[0]) = adf::dma_channel(adf::aie_tile, 0, 3, 0);
    adf::connect(shared_buffer_0.out[0], mk[0][3].in[0]);

    // [...]
    // The above 1:N dispatch repeated 3 times

    // N:1 merge merge_0 {mk[0][0].out[0], mk[0][1].out[0], mk[0][2].out[0], mk[0][3].out[0]} -> shared_buffer_1.in[0]
    merge_0 = pktorderedmerge<4>::create();
    // Configuration of the L1 read side
    adf::dimensions(mk[0][0].out[0]) = {128}; // Dimensions of the TensorVar
    adf::single_buffer(mk[0][0].out[0]); // If your L1 variable uses SingleBuffering
    adf::location<adf::buffer>(mk[0][0].out[0]) = {adf::address(0, 0, 20480)}; // L1 automatic placement - check if you manually placed the variable
    adf::location<adf::dma>(mk[0][0].out[0]) = adf::dma_channel(adf::aie_tile, 0, 0, 0); // Core tile origin of the buffer
    adf::connect(mk[0][0].out[0], merge_0.in[0]); // There is one merge per channel
    // [...]
    // The above configuration repeated 3 times (once per core in the column)
    // Configuration of the L2 write side
    adf::connect<adf::pktstream>(merge_0.out[0], shared_buffer_1.in[0]);
    adf::write_access(shared_buffer_1.in[0]) = {adf::tiling({.buffer_dimension = {128},
                                                             .tiling_dimension = {128},
                                                             .offset = {0},
                                                             .tile_traversal = {},
                                                             .packet_port_id = 0
                                                            }),
                                                adf::tiling({.buffer_dimension = {128},
                                                             .tiling_dimension = {128},
                                                             .offset = {0},
                                                             .tile_traversal = {},
                                                             .packet_port_id = 1
                                                            }),
                                                adf::tiling({.buffer_dimension = {128},
                                                             .tiling_dimension = {128},
                                                             .offset = {0},
                                                             .tile_traversal = {},
                                                             .packet_port_id = 2
                                                            }),
                                                adf::tiling({.buffer_dimension = {128},
                                                             .tiling_dimension = {128},
                                                             .offset = {0},
                                                             .tile_traversal = {},
                                                             .packet_port_id = 3
                                                            })};
    adf::location<adf::dma>(shared_buffer_1.in[0]) = adf::dma_channel(adf::memory_tile, 1, 0, 0);

    // [...]
    // The above N:1 merge repeated 3 times

    // The OFM writeback to L3
    // 1:1 bind shared_buffer_1.out[0] -> external_var_1.in[0]
    adf::location<adf::dma>(shared_buffer_1.out[0]) = adf::dma_channel(adf::memory_tile, 0, 0, 0);
    adf::location<adf::dma>(external_var_1.in[0]) = adf::dma_channel(adf::shim_tile, 0, 0, 0);
    adf::connect(shared_buffer_1.out[0], external_var_1.in[0]);
    adf::read_access(shared_buffer_1.out[0]) = {adf::tiling({.buffer_dimension = {128},
                                                             .tiling_dimension = {128},
                                                             .offset = {0},
                                                             .tile_traversal = {}
                                                            })};
    adf::write_access(external_var_1.in[0]) = {adf::tiling({.buffer_dimension = {128},
                                                            .tiling_dimension = {128},
                                                            .offset = {0},
                                                            .tile_traversal = {}
                                                           })};
  }
}; // class mydomain_mynegate_0_stamp0_phase0
} // namespace custom_ops

namespace custom_ops {
template <typename dtype_var_0, typename dtype_var_1, int overlay_id, int kernel_cols,
          int kernel_rows, int stack_location, int stack_size, int heap_size,
          int ifm_ddr_port_logical_index = 0,
          int ofm_ddr_port_logical_index = 0, int overlay_col_offset = 0,
          int num_ifm_shim_ch = 1, int num_ofm_shim_ch = 1>
class mydomain_mynegate_overlay : public adf::graph {
public:
  using mydomain_mynegate_0_params_t = struct mydomain_mynegate_0_params_t;

  mydomain_mynegate_0_stamp0_phase0<stack_location, stack_size, heap_size> stamp0_phase0;

  mydomain_mynegate_overlay(adf::external_buffer<dtype_var_0> &external_var_0, adf::external_buffer<dtype_var_1> &external_var_1, const mydomain_mynegate_0_params_t & /*params_t unused*/)
      : stamp0_phase0(external_var_0, external_var_1) {
 }
};
} // namespace custom_ops

using namespace ml_adf_overlay_4x4;
std::vector<ml_adf_overlay_4x4::tiling_params_t> mydomain_mynegate__mynegate_negated_0_multi_layer_overlay = {};

#endif // _MYDOMAIN_MYNEGATE__MYNEGATE_NEGATED_0_MULTI_LAYER_OVERLAY_
```

A few useful sanity checks against the tiling script:
타일링 스크립트에 대한 몇 가지 유용한 온전성 검사:

- The dimensions and `Location(col, row, offset)` arguments given to the L2 `TensorVar.make(...)` calls should match the shapes in `adf::shared_buffer<>.create({...})` (in elements) and `adf::location` statements at the beginning of each stamp/phase class declaration.
- L2 `TensorVar.make(...)` 호출에 제공된 차원 및 `Location(col, row, offset)` 인수는 각 스탬프/단계 클래스 선언 시작 부분에 있는 `adf::shared_buffer<>.create({...})`(요소 내) 및 `adf::location` 문의 모양과 일치해야 합니다.
- The shapes are to be taken after any `Reshape`/`Tile` expressions.
- 모양은 'Reshape'/'Tile' 표현식 다음에 가져와야 합니다.
- The dimensions of the L1 `TensorVar`s are to be found in `adf::dimensions(mk[i][j].{in,out}[0]) = {...}` statements.
- L1 `TensorVar`의 차원은 `adf::dimensions(mk[i][j].{in,out}[0]) = {...}` 문에서 찾을 수 있습니다.
- Also check the `adf::location<adf::buffer>(mk[i][j].{in,out}[0]) = {adf::address(0, 0, [addr])}` statements.
- 또한 `adf::location<adf::buffer>(mk[i][j].{in,out}[0]) = {adf::address(0, 0, [addr])}` 문도 확인하세요.
- The kernel-call and L2-rotation counts you have computed (see [Kernel Ratios](#kernel-ratios)) appear in `adf::repetition_count(mk[0][0]) = [kernel call count]`, `adf::repetition_count(shared_buffer_0) = [L2 buffer rotation count]`.
- 계산한 커널 호출 및 L2 회전 횟수([커널 비율](#kernel-ratios) 참조)는 `adf::repetition_count(mk[0][0]) = [커널 호출 횟수]`, `adf::repetition_count(shared_buffer_0) = [L2 버퍼 회전 횟수]`에 나타납니다.
- The kernel ratios appear for each argument/result in `adf::async_repetition(mk[0][0].{in,out}[arg_or_res_number]) = [value]`.
- 각 인수/결과에 대한 커널 비율은 `adf::async_repetition(mk[0][0].{in,out}[arg_or_res_number]) = [value]`에 나타납니다.
