<!-- ko-translation: enabled -->
---
name: vai-aie-kernel-development
license: Apache-2.0 WITH LLVM-exception
compatibility: Requires VitisAI 6.3+
metadata:
  version: 6.3.0
  stage: beta
description: Comprehensive AIE kernel development guide covering architecture-level
  optimization techniques, result analysis, and compiler-independent patterns. Covers
  loop splitting for register pressure, in-place updates, signedness optimization,
  VLIW slot awareness, result analysis (disassembly, DM maps, calltree, profiling),
  and general kernel coding practices. Ships aie_api_references.md, the detailed AIE
  API primitive catalog (per-dtype support, vectorization, accumulator types, math-pattern
  to API lookup). For Peano compiler-specific pragmas, loop hints, software pipelining
  details, and optimization remarks, see the vai-aie-compiler-oriented-optimizations
  skill.
---
<!--- Copyright (C) 2026 Advanced Micro Devices, Inc. All rights reserved.--->

# AIE Kernel Development Guide
# AIE 커널 개발 가이드

## Access command
## 액세스 명령

This skill must always be accessed through the worker, never in a different way.
이 기술은 항상 작업자를 통해 접근해야 하며 다른 방식으로는 접근할 수 없습니다.
Allowed worker: `vai-custom-op-worker`.
허용된 작업자: `vai-custom-op-worker`.

## Description
## 설명

This guide covers compiler-independent techniques and analysis methods for
이 가이드에서는 컴파일러 독립적인 기술과 분석 방법을 다룹니다.
writing high-performance AIE kernels.
고성능 AIE 커널 작성.
It applies regardless of whether
여부와 상관없이 적용됩니다.
you are using Peano or any other AIE compiler backend.
Peano 또는 기타 AIE 컴파일러 백엔드를 사용하고 있습니다.

For Peano-specific compiler pragmas, loop hints, software pipelining
Peano 관련 컴파일러 pragma, 루프 힌트, 소프트웨어 파이프라이닝용
(pre-RA vs post-RA), loop versioning, and reading optimization remarks,
(RA 이전 vs. RA), 루프 버전 관리 및 최적화 설명 읽기,
see the **vai-aie-compiler-oriented-optimizations** skill.
**vai-aie-컴파일러 지향 최적화** 기술을 참조하세요.


## 1.
## Architecture Fundamentals
## 아키텍처 기초

AIE cores are VLIW (Very Long Instruction Word) processors.
AIE 코어는 VLIW(Very Long Instruction Word) 프로세서입니다.
Key properties:
주요 속성:

- **VLIW slots**: Each cycle can issue multiple operations in parallel
- **VLIW 슬롯**: 각 주기는 여러 작업을 병렬로 실행할 수 있습니다.
(load, store, scalar, vector, etc.) if they fit into different slots.
(로드, 저장, 스칼라, 벡터 등) 서로 다른 슬롯에 맞는 경우.
Filling more slots per cycle = higher throughput.
주기당 더 많은 슬롯을 채우는 것 = 더 높은 처리량.
- **In-order execution**: No out-of-order engine.
- **순서대로 실행**: 순서가 잘못된 엔진이 없습니다.
- The compiler must
- 컴파일러는 다음을 수행해야 합니다.
statically schedule instructions to hide latencies.
대기 시간을 숨기기 위한 지침을 정적으로 예약합니다.
- **Zero-overhead loops (ZOL)**: Hardware loop support that eliminates
- **제로 오버헤드 루프(ZOL)**: 하드웨어 루프 지원으로
branch overhead.
지점 오버헤드.
The compiler converts eligible loops automatically.
컴파일러는 적합한 루프를 자동으로 변환합니다.
- **Multiple memory banks**: Data memory (DM) is banked (A, B, C, D).
- **다중 메모리 뱅크**: 데이터 메모리(DM)가 뱅크됩니다(A, B, C, D).
Two loads or a load and a store can execute simultaneously if they
두 개의 로드 또는 로드와 저장이 동시에 실행될 수 있습니다.
access different banks.
다른 은행에 접속하세요.
- **Software pipelining**: The most important optimization for inner
- **소프트웨어 파이프라이닝**: 내부를 위한 가장 중요한 최적화
loops.
루프.
It overlaps consecutive iterations so that instructions from
연속적인 반복이 겹쳐지므로
multiple iterations are in-flight simultaneously.
여러 반복이 동시에 진행 중입니다.

The compiler needs developer hints (restrict, bank annotations, trip
컴파일러에는 개발자 힌트(제한, 은행 주석, 트립)가 필요합니다.
counts, pipelining directives) to produce optimal schedules.
카운트, 파이프라인 지시문)을 사용하여 최적의 일정을 생성합니다.


## 2.
## Vectorization and the AIE API
## 벡터화 및 AIE API

Two techniques compound to give most of the speedup on AIE cores:
AIE 코어에서 대부분의 속도 향상을 제공하기 위해 두 가지 기술이 결합되었습니다.

- **Vectorization** uses the vector datapath that scalar code leaves idle.
- **벡터화**는 스칼라 코드가 유휴 상태로 유지되는 벡터 데이터 경로를 사용합니다.
Speedup scales roughly with the vector width for the chosen element type
속도 향상은 선택한 요소 유형의 벡터 너비에 따라 대략적으로 조정됩니다.
(e.g.
32-way for int8 mul on AIE-ML), provided the loop is also software
AIE-ML의 int8 mul에 대한 32방향(루프도 소프트웨어인 경우)
pipelined and not memory bound.
파이프라인이 있고 메모리에 바인딩되어 있지 않습니다.
- **Library calls** delegate to implementations written by the AIE API
- **라이브러리 호출**은 AIE API로 작성된 구현을 위임합니다.
authors.
저자.
They are vector-aware, tuned per architecture (AIE1/AIE2/AIE-ML),
벡터를 인식하고 아키텍처(AIE1/AIE2/AIE-ML)에 따라 조정됩니다.
and well tested across element types and accumulator widths.
요소 유형과 누산기 너비에 걸쳐 잘 테스트되었습니다.

Prefer the AIE API over hand-rolled intrinsics: it gives you vectorization
직접 작성한 내장 함수보다 AIE API를 선호합니다. 벡터화 기능을 제공합니다.
and portability across AIE variants in one call, and the compiler schedules
한 번의 호출로 AIE 변형 전반에 걸쳐 이식성을 제공하고 컴파일러는 일정을 계획합니다.
it better than ad-hoc intrinsic sequences.
임시 내장 시퀀스보다 낫습니다.
Only fall back to raw intrinsics
원시 내장 함수로만 대체
when profiling shows the API call is the bottleneck and a custom schedule
프로파일링에서 API 호출에 병목 현상이 발생하고 사용자 지정 일정이 표시되는 경우
can demonstrably do better.
분명히 더 잘할 수 있습니다.

**Start from the bundled reference.** Before writing or optimizing any kernel
**번들에 포함된 참조에서 시작하세요.** 커널을 작성하거나 최적화하기 전에
math, read [aie_api_references.md](aie_api_references.md) next to this file: it
수학, 이 파일 옆에 있는 [aie_api_references.md](aie_api_references.md)를 읽어보세요.
catalogs the AIE API primitives (core, fused/advanced, data movement/shuffle,
AIE API 기본 요소(코어, 융합/고급, 데이터 이동/셔플,
`aie::mmul`) with their dtype support, lane width, and accumulator type, plus a
`aie::mmul`)의 dtype 지원, 레인 너비 및 누산기 유형과
math-pattern -> API table.
수학 패턴 -> API 테이블.
Use it to pick the idiomatic fused primitive instead
대신 관용적 융합 기본 요소를 선택하는 데 사용하세요.
of hand-rolling a LUT or polynomial.
LUT 또는 다항식을 수동으로 롤링하는 것입니다.

For anything it does not cover, read the headers themselves -- its
여기서 다루지 않는 내용은 헤더 자체를 읽어보세요.
["discover from the headers"](aie_api_references.md#if-its-not-here-discover-from-the-headers)
["헤더에서 검색"](aie_api_references.md#if-its-not-here-discover-from-the-headers)
section is the single source of truth for resolving the header directory out of
섹션은 헤더 디렉터리를 확인하기 위한 단일 정보 소스입니다.
the active environment and for which header answers which question.
활성 환경과 헤더가 어떤 질문에 대답하는지.
One rule
하나의 규칙
worth repeating here: scope every search to that directory, because a `find` or
여기서 반복할 가치가 있습니다. 모든 검색 범위를 해당 디렉토리로 지정하세요. 왜냐하면 `find` 또는
`grep -r` from the env root or prefix walks a tree large enough to stall.
env 루트 또는 접두사에서 `grep -r`은 멈출 만큼 큰 트리를 탐색합니다.

## 3.
## VLIW Slot Awareness
## VLIW 슬롯 인식

Understanding which instructions occupy which VLIW slots is essential
어떤 명령어가 어떤 VLIW 슬롯을 차지하는지 이해하는 것이 중요합니다.
for identifying parallelism opportunities and slot conflicts.
병렬화 기회와 슬롯 충돌을 식별하기 위한 것입니다.

### VLIW Slot Overview
### VLIW 슬롯 개요

A typical AIE core exposes parallel issue slots that can be roughly grouped
일반적인 AIE 코어는 대략적으로 그룹화할 수 있는 병렬 문제 슬롯을 노출합니다.
as follows.
다음과 같이.
Consult the architecture reference for your specific target
특정 대상에 대한 아키텍처 참조를 확인하세요.
device for the exact slot table and instruction set.
정확한 슬롯 테이블과 명령어 세트를 위한 장치입니다.

| Slot<br>슬롯     | Role<br>역할     | Typical Instructions<br>일반적인 지침                                  |
|----------|----------|-------------------------------------------------------|
| Load A<br>로드 A   | Load<br>로드     | Vector and scalar loads, pointer-add updates<br>벡터 및 스칼라 로드, 포인터 추가 업데이트          |
| Load B<br>로드 B   | Load<br>로드     | Second load port, unpack/move<br>두 번째 로드 포트, 포장 풀기/이동                         |
| Store<br>매장    | Store<br>매장    | Vector store, pack, shift-round-saturate, convert<br>벡터 저장, 팩, 이동-라운드-포화, 변환     |
| Scalar<br>스칼라   | Scalar<br>스칼라   | Integer ALU, compare, branch, lock acquire/release<br>정수 ALU, 비교, 분기, 잠금 획득/해제    |
| Move<br>이동     | Move<br>이동     | Vector shuffle/select/shift, accumulator move<br>벡터 셔플/선택/이동, 누산기 이동         |
| Move imm<br>움직여요 | Move imm<br>움직여요 | Immediate moves, immediate branches<br>즉시 이동, 즉시 분기                   |
| Vec MAC<br>Vec 맥  | Vec MAC<br>Vec 맥  | Vector multiply-accumulate, add, subtract, negate<br>벡터 곱셈-누산, 더하기, 빼기, 부정     |
| Mat MAC<br>매트 맥  | Mat MAC<br>매트 맥  | Matrix multiply-accumulate (integer / float variants)<br>행렬 곱셈-누산(정수/부동 소수점 변형) |

### Conflict Detection
### 충돌 감지

If two instructions in the same loop body need the same slot, they
동일한 루프 본문에 있는 두 명령어에 동일한 슬롯이 필요한 경우
cannot execute in the same cycle.
동일한 주기에서 실행할 수 없습니다.
This increases the minimum initiation
이는 최소 시작을 증가시킵니다.
interval (MII).
간격(MII).

Example: Two VLD instructions both need slot A or B.
예: 두 VLD 명령어 모두 슬롯 A 또는 B가 필요합니다.
If the loop body
루프 본체의 경우
has 4 VLD operations, and only slots A and B can issue loads, the
4개의 VLD 작업이 있고 슬롯 A와 B만 로드를 실행할 수 있습니다.
resource MII is at least 2.
리소스 MII는 2 이상입니다.

### Common AI/ML Operation Mappings
### 일반적인 AI/ML 작업 매핑

| Operation<br>작동     | Instruction Sequence<br>지시 순서                     |
|--------------|------------------------------------------|
| Softmax<br>소프트맥스      | VEXP2 + VADDR + VINV<br>VEXP2 + VADDR + VINV                     |
| ReLU<br>ReLU         | VMAX_LT<br>VMAX_LT                                  |
| MatMul<br>MatMul       | MMAC (supports sparse)<br>MMAC(스파스 지원)                    |
| Convolution<br>컨볼루션  | VMOV.conv + MMAC<br>VMOV.conv + MMAC                          |
| Quantization<br>양자화 | VSRS + VCLAMP<br>VSRS + VCLAMP                             |


## 4.
## Register Pressure: Loop Splitting and Small Temporaries
## 레지스터 압력: 루프 분할 및 작은 임시

When a loop is under register pressure, the compiler wastes instructions
루프에 레지스터 압력이 가해지면 컴파일러는 명령어를 낭비합니다.
on spill-to-stack and reload within the inner loop.
유출되어 스택에 쌓이고 내부 루프 내에서 다시 로드됩니다.
This inflates II
이것은 II를 팽창시킨다
and reduces throughput.
그리고 처리량을 줄입니다.

### Detection
### 탐지
- Look for stack spills in the generated assembly (`.lst` file)
- 생성된 어셈블리(`.lst` 파일)에서 스택 유출을 찾습니다.
- Stack spill instructions appear as stores to DM addresses in the
- 스택 유출 지침은 다음의 DM 주소에 대한 저장소로 나타납니다.
system region
시스템 지역
- If the inner loop has more vector variables alive than physical
- 내부 루프에 물리적 변수보다 더 많은 벡터 변수가 살아있는 경우
registers, spills are inevitable
기록부, 유출은 불가피하다

### Solution: Split the Loop
### 해결책: 루프 분할

Split the loop into two passes, using DM for intermediate results:
중간 결과를 위해 DM을 사용하여 루프를 두 개의 패스로 분할합니다.

```cpp
// BEFORE: single loop with high register pressure
for (int i = 0; i < N; i++) {
    // load input, load weights, compute MAC, apply activation,
    // quantize, store -- too many live registers
}

// AFTER: two loops, each with lower pressure
// Pass 1: compute and store intermediate
for (int i = 0; i < N; i++) {
    auto result = compute(input[i], weights[i]);
    intermediate[i] = result;  // DM write
}
// Pass 2: post-process from intermediate
for (int i = 0; i < N; i++) {
    auto val = intermediate[i];  // DM read
    output[i] = activate_and_quantize(val);
}
```

Each resulting loop has fewer live registers and can be pipelined more
각 결과 루프에는 라이브 레지스터 수가 적고 더 많이 파이프라인될 수 있습니다.
effectively.
효과적으로.
The DM read/write cost is typically cheaper than the
DM 읽기/쓰기 비용은 일반적으로 DM보다 저렴합니다.
spill/reload overhead in a bloated single loop.
부풀어 오른 단일 루프에서 오버 헤드를 흘리거나 다시로드합니다.

### Solution: Keep Temporaries Small (per-row / 8-lane)
### 해결 방법: 임시 파일을 작게 유지(행당/8레인)

The other lever on the same problem: instead of restructuring the loop, shrink
같은 문제에 대한 또 다른 수단: 루프를 재구성하는 대신 축소
how much state is live inside it.
그 안에 얼마나 많은 상태가 살고 있는지.
A kernel that holds a whole tile in registers
레지스터에 전체 타일을 보유하는 커널
spills; one that walks the tile a row at a time does not.
유출; 한 번에 한 줄씩 타일을 걷는 사람은 그렇지 않습니다.

```cpp
// HEAVY: a full 64-lane tile stays live across the computation
aie::vector<bfloat16, 64> tile = aie::load_v<64>(in);
tile = do_stuff(tile);
aie::store_v(out, tile);

// LIGHT: one 8-lane row live at a time (small live set)
for (int r = 0; r < 8; ++r) {
    aie::vector<bfloat16, 8> row = aie::load_v<8>(in + r * 8);
    row = do_stuff(row);
    aie::store_v(out + r * 8, row);
}
```

The same idea applies to everything else the kernel keeps around: prefer named
커널이 유지하는 다른 모든 것에도 동일한 아이디어가 적용됩니다. 이름이 지정된 것을 선호합니다.
locals over addressable vector arrays (indexing forces the compiler to give the
주소 지정이 가능한 벡터 배열에 대한 로컬(인덱싱은 컴파일러가
array a DM home), keep reductions in registers, and drop dynamic/temporary
DM 홈 배열), 레지스터 감소 유지, 동적/임시 삭제
allocations and static scratch/LUTs, which consume heap.
힙을 소비하는 할당 및 정적 스크래치/LUT.

**Reach for this only when you actually need to cut stack/register usage.**
**실제로 스택/레지스터 사용량을 줄여야 하는 경우에만 이 방법을 사용하세요.**
Narrower vectors mean more iterations and less work per instruction, so it trades
벡터가 좁을수록 반복 횟수가 늘어나고 명령어당 작업량이 줄어듭니다.
throughput for pressure relief.
압력 완화를 위한 처리량.
Confirm from the `.lst` (spills) or a stack
`.lst`(유출) 또는 스택에서 확인하세요.
overflow that pressure is the real problem before shrinking anything.
무엇이든 축소하기 전에 압력이 진짜 문제인 것 같습니다.


## 5.
## In-Place Updates
## 내부 업데이트

When a kernel must read and write the same buffer (in-place operation),
커널이 동일한 버퍼를 읽고 써야 하는 경우(내부 작업),
use two separate pointers with different bank hints and `__restrict`
서로 다른 은행 힌트와 `__restrict`를 사용하여 두 개의 별도 포인터를 사용하세요.
to allow dense inner loops:
조밀한 내부 루프를 허용하려면 다음을 수행하십시오.

```cpp
// BAD: single pointer for read-modify-write -- compiler assumes alias
for (int i = 0; i < N; i++) {
    buf[i] = process(buf[i]);
}

// GOOD: two pointers with bank hints and restrict
void inplace_core(
    dtype __aie_dm_resource_a *__restrict read_ptr,
    dtype __aie_dm_resource_b *__restrict write_ptr,
    int count) {

    for (int i = 0; i < count; i++) {
        auto val = aie::load_v<VEC_LEN>(read_ptr);
        read_ptr = byte_incr(read_ptr, VEC_BYTES);
        auto result = process(val);
        store_v(write_ptr, result);
        write_ptr = byte_incr(write_ptr, VEC_BYTES);
    }
}

// Call with same base address, different bank hints
inplace_core(
    (dtype __aie_dm_resource_a *__restrict)buf,
    (dtype __aie_dm_resource_b *__restrict)buf,
    N);
```

The "fake" bank hints tell the compiler to schedule the load and
"가짜" 뱅크 힌트는 컴파일러에게 로드를 예약하고
store in parallel (different bank ports), even though they point to
병렬로 저장(다른 뱅크 포트)
the same physical memory.
동일한 물리적 메모리.
Combined with `__restrict`, this enables
`__restrict`와 결합하면 다음이 가능해집니다.
full pipelining.
전체 파이프라인.


## 6.
## Signedness Optimization
## 부호화 최적화

When both signed and unsigned integer types need the same processing,
부호 있는 정수 유형과 부호 없는 정수 유형 모두 동일한 처리가 필요한 경우
avoid duplicating the kernel.
커널 복제를 피하십시오.
Use a single signed type signature with
단일 서명 유형 서명을 사용하십시오.
runtime signedness control:
런타임 부호성 제어:

```cpp
// GOOD: single function handles both int8 and uint8
void both_signedness(
    int8_t *__restrict ifm,
    int8_t *__restrict ofm,
    bool signed_fm) {

    // Use aie::to_fixed_sign() to handle signedness at runtime
    auto vec = aie::load_v<VEC_LEN>(ifm);
    auto result = aie::to_fixed_sign(vec, /* sign_vector */, shift);
    // ... process ...
}
```

This halves program memory (PM) usage compared to maintaining separate
이는 별도의 유지 관리에 비해 프로그램 메모리(PM) 사용량을 절반으로 줄입니다.
signed/unsigned kernel variants.
서명된/서명되지 않은 커널 변형.


## 7.
## Restrict Pointers
## 포인터 제한

The `__restrict` qualifier tells the compiler that a pointer does not
`__restrict` 한정자는 컴파일러에게 포인터가
alias any other pointer in scope.
범위 내의 다른 포인터에 별칭을 지정합니다.
This is the single most impactful
이것이 가장 임팩트 있는 싱글이다
optimization annotation for AIE kernels.
AIE 커널에 대한 최적화 주석입니다.

### Key Rules
### 주요 규칙

- **The function containing the hot loop must receive `__restrict`
- **핫 루프를 포함하는 함수는 `__restrict`를 수신해야 합니다.
pointers as parameters**, not buffer references.
버퍼 참조가 아닌 매개변수**로서의 포인터.
The `.data()`
`.data()`
extraction and restrict cast belong in the wrapper that calls it.
추출 및 제한 캐스트는 이를 호출하는 래퍼에 속합니다.
- Never cast away `__restrict` when applying bank annotations --
- 은행 주석을 적용할 때 `__restrict`를 버리지 마세요 --
combine them in the same declaration.
동일한 선언으로 결합합니다.
- Apply `__restrict` to all function pointer parameters that don't
- 그렇지 않은 모든 함수 포인터 매개변수에 `__restrict`를 적용합니다.
alias each other, including parameter struct references.
매개변수 구조체 참조를 포함하여 서로 별칭을 지정합니다.

### Pattern: Inline Wrapper + Noinline Core
### 패턴: 인라인 래퍼 + Noinline 코어

```cpp
// Public API: INLINE, does type dispatch and pointer casting
template <typename dtype>
INLINE void my_kernel(
    adf::input_buffer_conf<dtype, ...> &__restrict ifm,
    adf::output_buffer_conf<dtype, ...> &__restrict ofm,
    params_t &params) {
    my_kernel_core<dtype>(
        (dtype __aie_dm_resource_a *__restrict)ifm.data(),
        (dtype __aie_dm_resource_b *__restrict)ofm.data(),
        params);
}

// Core function: NOINLINE, receives properly qualified pointers
template <typename dtype>
__attribute__((noinline)) void my_kernel_core(
    dtype __aie_dm_resource_a *__restrict ifm_ptr,
    dtype __aie_dm_resource_b *__restrict ofm_ptr,
    params_t &__restrict params) {
    // Hot loop here -- compiler sees full restrict + bank info
}
```


## 8.
## DM Bank Annotations
## DM 은행 주석

AIE data memory is banked.
AIE 데이터 메모리가 뱅크됩니다.
Annotating pointers with their bank
은행과 함께 포인터에 주석 달기
assignment enables simultaneous memory operations in the same VLIW
할당을 통해 동일한 VLIW에서 동시 메모리 작업이 가능합니다.
bundle.
번들.

### Syntax
### 구문
```cpp
dtype __aie_dm_resource_a *ptr;    // Bank A
dtype __aie_dm_resource_b *ptr;    // Bank B
dtype __aie_dm_resource_c *ptr;    // Bank C
dtype __aie_dm_resource_d *ptr;    // Bank D
```

### Typical Assignments
### 일반적인 과제

| Data<br>데이터         | Bank<br>은행 | Rationale<br>이론적 근거                                    |
|-------------|------|----------------------------------------------|
| Input (IFM)<br>입력(IFM) | A    | Loads via load port A<br>로드 포트 A를 통해 로드                        |
| Weights<br>가중치     | B    | Separate bank enables parallel loads<br>별도의 뱅크로 병렬 로드 가능         |
| Output (OFM)<br>출력(OFM)| B    | Stores via store port B, parallel with A<br>A와 병렬로 저장소 포트 B를 통해 저장     |
| Bias / TDM<br>바이어스/TDM  | C/D  | Secondary data, flexible placement<br>보조 데이터, 유연한 배치           |

### Same-Bank Conflict
### 동일은행 갈등

If two pointers loaded in the same iteration are on the same bank,
동일한 반복에 로드된 두 포인터가 동일한 뱅크에 있는 경우
loads serialize.
로드가 직렬화됩니다.
Always put concurrently accessed data on different
동시에 액세스되는 데이터는 항상 다른 위치에 저장
banks.
은행.


## 9.
## Vector Load/Store Alignment
## 벡터 로드/저장 정렬

Vector loads and stores require address alignment to the vector width.
벡터 로드 및 저장에는 벡터 너비에 대한 주소 정렬이 필요합니다.
Misaligned accesses cause undefined behavior or hardware exceptions.
잘못 정렬된 액세스로 인해 정의되지 않은 동작이나 하드웨어 예외가 발생합니다.

| Vector Width<br>벡터 폭 | Alignment Required<br>정렬 필요 |
|--------------|-------------------|
| 256-bit<br>256비트      | 32-byte aligned<br>32바이트 정렬   |
| 512-bit<br>512비트      | 64-byte aligned<br>64바이트 정렬   |
| 1024-bit<br>1024비트     | 128-byte aligned<br>128바이트 정렬  |

For unaligned streaming access, use FIFO intrinsics
정렬되지 않은 스트리밍 액세스의 경우 FIFO 내장 함수를 사용하세요.
(`fifo_ld_pop` / `fifo_st_push`) which handle alignment internally.
(`fifo_ld_pop` / `fifo_st_push`) 내부적으로 정렬을 처리합니다.


## 10.
## Sub-32-Bit Load/Store Limitations
## 32비트 미만 로드/저장 제한

Loads and stores narrower than 32 bits (`int8`, `int16`, `bfloat16`
32비트(`int8`, `int16`, `bfloat16`보다 좁은 값을 로드하고 저장합니다.
scalars) trigger hardware read-modify-write, making them significantly
스칼라)는 하드웨어 읽기-수정-쓰기를 트리거하여 하드웨어를 크게 만듭니다.
more expensive than vector-width operations.
벡터 너비 연산보다 비용이 더 많이 듭니다.

**Rule**: Always use vector-width loads and stores (`aie::load_v`,
**규칙**: 항상 벡터 너비 로드 및 저장(`aie::load_v`,
`store_v`) in performance-critical loops.
`store_v`) 성능이 중요한 루프에 있습니다.
Avoid scalar sub-word
스칼라 하위 단어 방지
memory operations inside hot loops.
핫 루프 내부의 메모리 작업.


## 11.
## Result Analysis
## 결과 분석

### Compile Artifacts
### 아티팩트 컴파일

In `Work/aie/{X}_{Y}/Release/` (single core tests typically use core
`Work/aie/{X}_{Y}/Release/`에서(단일 코어 테스트는 일반적으로 코어를 사용합니다)
(0,0)):

| File<br>파일               | Contents<br>목차                                          |
|-------------------|---------------------------------------------------|
| `{X}_{Y}.lst`<br>`{X}_{Y}.lst`     | Disassembled assembly -- inspect for slot usage,<br>분해된 어셈블리 - 슬롯 사용 여부를 검사합니다.   |
|                   | spills, and instruction count per loop<br>유출 및 루프당 명령어 수             |
| `{X}_{Y}.map`<br>`{X}_{Y}.map`     | Compile-time DM allocation for stack/heap<br>스택/힙에 대한 컴파일 타임 DM 할당          |
| `{X}_{Y}.calltree`<br>`{X}_{Y}.calltree`| Function call tree with per-function summary<br>함수별 요약이 포함된 함수 호출 트리       |
|                   | (instruction count, stack depth, PM size)<br>(명령어 개수, 스택 깊이, PM 크기)          |

### Profiling Results
### 프로파일링 결과

The file `profile_instr_{x}_{y}.xml` contains the aiesim profiling
`profile_instr_{x}_{y}.xml` 파일에는 aiesim 프로파일링이 포함되어 있습니다.
result from the Instruction Set Simulator (ISS).
ISS(명령어 세트 시뮬레이터)의 결과입니다.

**Important**: The `{x}_{y}` coordinates in profile files count
**중요**: 프로필 파일 수의 `{x}_{y}` 좌표
differently than `Work/aie/` folders -- they include the mem tile
`Work/aie/` 폴더와는 다르게 -- mem 타일을 포함합니다.
row, so the most typical value is `(0, 1)` not `(0, 0)`.
행이므로 가장 일반적인 값은 `(0, 0)`이 아닌 `(0, 1)`입니다.

### Using the Calltree for QoR Analysis
### QoR 분석을 위해 콜트리 사용

The `.calltree` file provides per-function metrics that are critical
`.calltree` 파일은 중요한 기능별 측정항목을 제공합니다.
for QoR (Quality of Results) analysis:
QoR(결과 품질) 분석의 경우:

- **Column 2 (`stack_desc`)**: Stack usage per function -- take the
- **열 2(`stack_desc`)**: 함수별 스택 사용량 -
maximum across multi-core runs
멀티 코어 실행 시 최대값
- **Column 6 (`func_desc`)**: Program memory (PM) size per function
- **열 6(`func_desc`)**: 함수당 프로그램 메모리(PM) 크기
-- take the maximum across multi-core runs
-- 멀티 코어 실행 전반에 걸쳐 최대한 활용

These values should be compared against metadata expectations.
이러한 값은 메타데이터 기대치와 비교되어야 합니다.
Automate the
자동화
extraction with a small script that parses the `.calltree` columns when
'.calltree' 열을 구문 분석하는 작은 스크립트를 사용하여 추출
running this analysis at scale.
이 분석을 대규모로 실행합니다.

### Using the .map File for Heap Analysis
### 힙 분석을 위해 .map 파일 사용

The `.map` file contains the DMb memory map section.
`.map` 파일에는 DMB 메모리 맵 섹션이 포함되어 있습니다.
Parse it to
그것을 구문 분석
determine heap usage by summing referenced symbols, rounded to
참조 기호를 합산하여 힙 사용량을 결정하고 반올림합니다.
4-byte alignment.
4바이트 정렬.
Take the maximum across multi-core runs.
멀티 코어 실행 전반에 걸쳐 최대한 활용하세요.

### Quick Diagnosis from Artifacts
### 아티팩트를 통한 빠른 진단

| Symptom<br>증상                     | Check<br>확인                                | Likely Cause<br>가능한 원인                    |
|----------------------------|--------------------------------------|---------------------------------|
| High instruction count<br>높은 명령어 수     | `.lst` inner loop body<br>`.lst` 내부 루프 본문               | Spills, slot conflicts, or<br>유출, 슬롯 충돌 또는      |
|                            |                                      | unoptimized conversion chains<br>최적화되지 않은 전환 체인   |
| Large stack size<br>큰 스택 크기           | `.calltree` column 2<br>`.calltree` 열 2                 | Too many live variables,<br>라이브 변수가 너무 많습니다.        |
|                            |                                      | split the loop<br>루프를 분할                  |
| PM bloat<br>오후 부풀어 오름                   | `.calltree` column 6<br>`.calltree` 열 6                 | Inlined code, loop versioning<br>인라인 코드, 루프 버전 관리   |
|                            |                                      | duplication<br>복제                     |
| Cycle count >> expected<br>사이클 수 >> 예상    | `profile_instr` vs theoretical II<br>`profile_instr` 대 이론 II    | Memory stalls, lock contention,<br>메모리 정지, 잠금 경합, |
|                            |                                      | DMA latency<br>DMA 대기 시간                     |


## 12.
## Innermost Loop Bound Factors
## 가장 안쪽 루프 결합 요인

Three factors can bound innermost loop optimization:
세 가지 요소가 가장 안쪽 루프 최적화를 제한할 수 있습니다.

1. **Slot conflicts**: Prevents VLIW bundling.
1. **슬롯 충돌**: VLIW 번들링을 방지합니다.
1. Check the VLIW slot
1. VLIW 슬롯 확인
table (Section 3) to identify which instructions compete for the
어떤 지침이 경쟁하는지 식별하는 표(섹션 3)
same slot.
같은 슬롯.

2. **Register pressure**: Check the number of registers used, taking
2. **레지스터 압력**: 사용된 레지스터 수를 확인하고
instruction latency into account.
명령 대기 시간을 고려합니다.
Stack spills in the `.lst` file
`.lst` 파일에서 스택 유출
indicate register overuse.
레지스터 과다 사용을 나타냅니다.
Consider loop splitting (Section 4).
루프 분할을 고려하십시오(섹션 4).

3. **Insufficient loop range**: The loop range annotation guarantees
3. **불충분한 루프 범위**: 루프 범위 주석은 다음을 보장합니다.
the compiler can safely pipeline.
컴파일러는 안전하게 파이프라인할 수 있습니다.
Without it, the compiler must
그것이 없으면 컴파일러는 다음을 수행해야 합니다.
be conservative about prologue/epilogue peeling.
프롤로그/에필로그 필링에 대해 보수적이어야 합니다.


## 13.
## AIE Core Coding Rules
## AIE 핵심 코딩 규칙

These rules apply to all AIE kernel code regardless of compiler:
이러한 규칙은 컴파일러에 관계없이 모든 AIE 커널 코드에 적용됩니다.

- AIE intrinsics and AIE API (`aie::` calls) often hold both unsigned
- AIE 내장 함수 및 AIE API(`aie::` 호출)는 종종 서명되지 않은 두 가지를 모두 보유합니다.
and signed vectors in the same container type (e.g., `v32int16` for
동일한 컨테이너 유형의 부호 있는 벡터(예: 'v32int16'
both `int16` and `uint16`), with signedness controlled by an extra
`int16` 및 `uint16` 모두), 부호는 추가 항목에 의해 제어됨
argument.
논쟁.
Do not expand such abstractions.
그러한 추상화를 확장하지 마십시오.
- `printf` from `stdio.h` can be used for debugging, but it can cause
- `stdio.h`의 `printf`를 디버깅에 사용할 수 있지만 이로 인해
compilation failure in innermost loops.
가장 안쪽 루프에서 컴파일 오류가 발생했습니다.
Use it outside hot loops or
핫 루프 외부에서 사용하거나
guard with `#ifdef __X86SIM__`.
`#ifdef __X86SIM__`으로 보호하세요.
- Use the AIE API (`aie::` namespace) instead of raw intrinsics when
- 다음과 같은 경우 원시 내장 함수 대신 AIE API(`aie::` 네임스페이스)를 사용하세요.
possible -- it is more portable and readable.
가능합니다. 이식성이 뛰어나고 읽기 쉽습니다.
- Avoid `me_primitive` calls.
- `me_primitive` 호출을 피하세요.


## 14.
## Related Skills
## 관련 스킬

- **vai-aie-compiler-oriented-optimizations** -- Peano-specific pragmas
- **vai-aie-compiler-지향-최적화** -- Peano 특정 pragma
(`AIE_PREPARE_FOR_POSTPIPELINING`, `AIE_LOOP_RANGE`, `AIE_LOOP_HINT`,
(`AIE_PREPARE_FOR_POSTPIPELINING`, `AIE_LOOP_RANGE`, `AIE_LOOP_HINT`,
etc.), software pipelining details (pre-RA vs post-RA), loop
등), 소프트웨어 파이프라인 세부 정보(RA 이전 및 RA 이후), 루프
versioning (`VERSIONED_LOOP`), and reading optimization remarks.
버전 관리(`VERSIONED_LOOP`) 및 최적화 설명 읽기.
This guide is authored and maintained by the AIE compiler developers
이 가이드는 AIE 컴파일러 개발자가 작성하고 유지 관리합니다.
and serves as the **source of truth** for all compiler-oriented
모든 컴파일러 지향의 **진실** 소스 역할을 합니다.
optimization techniques.
최적화 기술.
It should not be overlooked when optimizing
최적화 시 간과해서는 안 될 점
kernel performance -- always consult it alongside this skill.
커널 성능 - 항상 이 기술과 함께 참조하십시오.
