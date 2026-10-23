<!-- ko-translation: enabled -->
---
name: vai-aie-compiler-oriented-optimizations
license: Apache-2.0 WITH LLVM-exception
compatibility: Requires VitisAI 6.3+
description: Performance optimization guide for custom AIE kernel development with Peano. Covers restrict pointers, DM bank annotations, loop pragmas, loop hints, software pipelining (pre-RA vs post-RA), loop versioning, function structure, pointer increments, sub-32-bit limitations, vector alignment, type conversion chains, and reading backend optimization hints from the compiler (remarks via -Rpass*/-fsave-optimization-record and warnings such as -Wpass-failed and the aie-multi-slot-pseudo missing-memory-bank hint). Consult when writing or reviewing AIE kernel C++ code for throughput, when diagnosing pipelining failures or high II, or when interpreting build-log warnings and missed-opportunity remarks from the AIE backend.
metadata:
  version: 6.3.0
  stage: beta
---

<!--
This file is licensed under the Apache License v2.0 with LLVM Exceptions.
이 파일은 LLVM 예외가 포함된 Apache 라이센스 v2.0에 따라 라이센스가 부여됩니다.
See https://llvm.org/LICENSE.txt for license information.
라이선스 정보는 https://llvm.org/LICENSE.txt를 참조하세요.
SPDX-License-Identifier: Apache-2.0 WITH LLVM-exception
SPDX-License-Identifier: LLVM 예외가 포함된 Apache-2.0

(c) Copyright 2026 Advanced Micro Devices, Inc. or its affiliates
(c) Copyright 2026 Advanced Micro Devices, Inc. 또는 그 계열사
-->

# Performance Optimization Guide for Custom AIE Kernel Development
# 맞춤형 AIE 커널 개발을 위한 성능 최적화 가이드

This guide documents actionable techniques for writing high-performance
이 가이드는 고성능 문서 작성을 위한 실행 가능한 기술을 문서화합니다.
AIE kernels, with compiler context explaining why each technique works.
각 기술이 작동하는 이유를 설명하는 컴파일러 컨텍스트가 포함된 AIE 커널.
It draws from real optimization work across AIE kernels (Conv2D, GEMM,
이는 AIE 커널(Conv2D, GEMM,
MaxPool, Slice, Transpose, etc.) and applies to the Peano compiler.
MaxPool, Slice, Transpose 등) 및 Peano 컴파일러에 적용됩니다.



## 1.
## Architecture Context
## 아키텍처 컨텍스트

AIE cores are VLIW (Very Long Instruction Word) processors with an
AIE 코어는 VLIW(Very Long Instruction Word) 프로세서입니다.
exposed pipeline.
노출된 파이프라인.
Key properties that affect kernel performance:
커널 성능에 영향을 미치는 주요 속성:

- **VLIW slots**: Each cycle can issue multiple operations in parallel
- **VLIW 슬롯**: 각 주기는 여러 작업을 병렬로 실행할 수 있습니다.
(load, store, scalar, vector, etc.) if they fit into different slots.
(로드, 저장, 스칼라, 벡터 등) 서로 다른 슬롯에 맞는 경우.
Filling more slots per cycle = higher throughput.
주기당 더 많은 슬롯을 채우는 것 = 더 높은 처리량.
- **In-order execution**: There is no out-of-order engine.
- **순서대로 실행**: 비순차적 엔진이 없습니다.
- The compiler
- 컴파일러
must statically schedule instructions to hide latencies.
지연 시간을 숨기려면 지침을 정적으로 예약해야 합니다.
- **Zero-overhead loops (ZOL)**: Hardware loop support that eliminates
- **제로 오버헤드 루프(ZOL)**: 하드웨어 루프 지원으로
branch overhead.
지점 오버헤드.
The compiler converts eligible loops automatically.
컴파일러는 적합한 루프를 자동으로 변환합니다.
- **Multiple memory banks**: Data memory is banked (A, B, C, D).
- **다중 메모리 뱅크**: 데이터 메모리가 뱅크됩니다(A, B, C, D).
- Two
- 2개
loads or a load and a store can execute simultaneously if they access
로드 또는 로드와 저장소가 액세스하는 경우 동시에 실행할 수 있습니다.
different banks.
다른 은행.
- **Software pipelining**: The most important optimization for inner
- **소프트웨어 파이프라이닝**: 내부를 위한 가장 중요한 최적화
loops.
루프.
It overlaps consecutive iterations so that instructions from
연속적인 반복이 겹쳐지므로
iteration N execute while iterations N-(NS-1), ..., N-1 are still
반복 N은 반복 N-(NS-1), ..., N-1이 아직 진행 중인 동안 실행됩니다.
in-flight (where NS is the number of pipeline stages).
진행 중(여기서 NS는 파이프라인 단계 수)

The compiler needs developer hints (restrict, bank annotations, trip
컴파일러에는 개발자 힌트(제한, 은행 주석, 트립)가 필요합니다.
counts, pipelining directives) to produce optimal schedules.
카운트, 파이프라인 지시문)을 사용하여 최적의 일정을 생성합니다.
Without
없이
them, it must be conservative.
보수적이어야 합니다.



## 2.
## Restrict Pointers
## 포인터 제한

### What
### 무엇
The `__restrict` qualifier tells the compiler that a pointer does not
`__restrict` 한정자는 컴파일러에게 포인터가
alias any other pointer in scope.
범위 내의 다른 포인터에 별칭을 지정합니다.
This is the single most impactful
이것이 가장 임팩트 있는 싱글이다
optimization annotation for AIE kernels.
AIE 커널에 대한 최적화 주석입니다.

### Why it matters
### 왜 중요한가요?
Without `__restrict`, the compiler must assume any store could modify
`__restrict`가 없으면 컴파일러는 모든 저장소가 수정할 수 있다고 가정해야 합니다.
the memory a subsequent load reads from.
후속 로드가 읽는 메모리입니다.
This forces it to:
이로 인해 다음이 수행됩니다.
- Insert memory barriers between stores and loads
- 저장소와 로드 사이에 메모리 장벽 삽입
- Prevent reordering across store/load pairs
- 저장소/로드 쌍 전체에서 재정렬 방지
- Limit or disable software pipelining
- 소프트웨어 파이프라인 제한 또는 비활성화

With `__restrict`, the compiler knows loads and stores to different
`__restrict`를 사용하면 컴파일러는 로드 및 저장을 알고 있습니다.
pointers are independent, enabling aggressive instruction reordering
포인터는 독립적이므로 공격적인 명령어 재정렬이 가능합니다.
and pipelining.
그리고 파이프라이닝.

### Common pitfalls
### 일반적인 함정

**Casting away restrict.**
**캐스트 어웨이는 제한됩니다.**
A frequent mistake is declaring function parameters with `__restrict`,
빈번한 실수는 `__restrict`를 사용하여 함수 매개변수를 선언하는 것입니다.
then immediately casting them to unqualified pointers inside the
그런 다음 즉시 이를 내부의 정규화되지 않은 포인터로 캐스팅합니다.
function body.
기능 본체.
The cast discards the restrict information.
캐스트는 제한 정보를 삭제합니다.

```cpp
// BAD: restrict is lost after the cast
void foo(dtype *__restrict ifm_ptr, dtype *__restrict ofm_ptr, ...) {
    dtype __aie_dm_resource_a *input_ptr =
        (dtype __aie_dm_resource_a *)ifm_ptr;       // restrict is gone
    dtype __aie_dm_resource_b *output_ptr =
        (dtype __aie_dm_resource_b *)ofm_ptr;        // restrict is gone
    ...
}
```

```cpp
// GOOD: restrict-qualified pointers as parameters, no stripping casts
void foo(
    dtype __aie_dm_resource_a *__restrict input_ptr,
    dtype __aie_dm_resource_b *__restrict ofm_load_ptr,
    dtype __aie_dm_resource_b *__restrict ofm_write_ptr,
    ...) {
    // Use input_ptr, ofm_load_ptr, ofm_write_ptr directly
}
```

**Restrict on buffer references vs raw pointers.**
**버퍼 참조와 원시 포인터를 제한합니다.**
ADF kernel entry points receive `adf::input_buffer_conf` and
ADF 커널 진입점은 `adf::input_buffer_conf`를 수신하고
`adf::output_buffer_conf` references.
`adf::output_buffer_conf` 참조.
Marking these references as
이러한 참조를 다음으로 표시
`__restrict` is **not sufficient** -- the restrict qualifier on a
`__restrict`는 **충분하지 않습니다** -- 제한 한정자는
reference to a buffer object does not propagate to the underlying data
버퍼 개체에 대한 참조는 기본 데이터로 전파되지 않습니다.
pointer returned by `.data()`.
`.data()`에 의해 반환된 포인터입니다.

The function that contains the hot loop must receive raw
핫 루프를 포함하는 함수는 원시 데이터를 수신해야 합니다.
`dtype *__restrict` pointers, not buffer references.
버퍼 참조가 아닌 `dtype *__restrict` 포인터.
Call `.data()`
`.data()` 호출
at the boundary (in the entry-point wrapper) and pass the resulting
(진입점 래퍼의) 경계에서 결과를 전달합니다.
restrict-qualified pointer into the core function:
핵심 함수에 대한 제한 한정 포인터:

```cpp
// BAD: restrict on buffer ref does not help the loop
void my_kernel(
    adf::input_buffer_conf<int8_t, ...> &__restrict ifm,
    adf::output_buffer_conf<int8_t, ...> &__restrict ofm) {

    // .data() returns a plain pointer -- restrict from the ref is gone
    auto *in = ifm.data();
    auto *out = ofm.data();
    for (int i = 0; i < N; i++) {
        out[i] = in[i] + 1;   // compiler assumes in/out may alias
    }
}
```

```cpp
// GOOD: .data() called in wrapper, restrict on raw pointers
void my_kernel(
    adf::input_buffer_conf<int8_t, ...> &__restrict ifm,
    adf::output_buffer_conf<int8_t, ...> &__restrict ofm) {

    my_kernel_core(
        (int8_t *__restrict)ifm.data(),
        (int8_t *__restrict)ofm.data());
}

__attribute__((noinline)) void my_kernel_core(
    int8_t *__restrict in,
    int8_t *__restrict out) {

    for (int i = 0; i < N; i++) {
        out[i] = in[i] + 1;   // compiler knows in/out don't alias
    }
}
```

The key rule: **the function containing the loop must receive
핵심 규칙: **루프를 포함하는 함수는 다음을 수신해야 합니다.
`__restrict` pointers as parameters, not buffer references**.
`__restrict` 포인터는 버퍼 참조가 아닌 매개변수로 사용됩니다**.
The
는
`.data()` extraction and restrict cast belong in the wrapper that
`.data()` 추출 및 제한 캐스트는 래퍼에 속합니다.
calls it.
그것을 부른다.

### Pattern: inline wrapper + noinline core
### 패턴: 인라인 래퍼 + noinline 코어

LLVM does not support scoped restrict: `__restrict` on function
LLVM은 범위 제한을 지원하지 않습니다: 함수에 대한 `__restrict`
parameters creates a no-alias guarantee that only holds at the
매개변수는
function boundary.
기능 경계.
When the function is inlined, this scope is
함수가 인라인되면 이 범위는
lost and the inlined code inherits the caller's alias information.
손실되고 인라인 코드는 호출자의 별칭 정보를 상속합니다.

This matters when the same pointer is passed for both reading and
이는 읽기와 읽기 모두에 동일한 포인터가 전달될 때 중요합니다.
writing (e.g., `foo_core(ifm_ptr, ofm_ptr, ofm_ptr)` where
쓰기(예: `foo_core(ifm_ptr, ofm_ptr, ofm_ptr)`, 여기서
`ofm_ptr` serves as both load and store pointer).
'ofm_ptr'은 로드 및 저장 포인터 역할을 합니다.
Without
없이
`noinline`, the compiler sees both pointers originate from the
`noinline`, 컴파일러는 두 포인터가 모두
same value and must assume they alias.
동일한 값을 가지며 별칭을 가정해야 합니다.
This forces the pipeliner
이로 인해 파이프라인이 강제로
to stall each iteration's load until the previous iteration's
이전 반복이 완료될 때까지 각 반복의 로드를 중지합니다.
store completes, preventing it from overlapping iterations and
저장이 완료되어 반복이 중복되는 것을 방지하고
inflating II.
팽창 Ⅱ.

```cpp
// BAD: only 2 pointers -- ofm used for both load and store.
//      If inlined, compiler sees one ofm pointer and assumes
//      the load and store alias, preventing iteration overlap.
template <typename dtype>
void foo(
    dtype IFM_DM_BANK *__restrict ifm_ptr,
    dtype OFM_DM_BANK *__restrict ofm_ptr,
    params_t &__restrict params) {
    AIE_LOOP_RANGE(4,)
    for (int i = 0; i < params.count; i++) {
        auto in_vec = aie::load_v<32>(ifm_ptr);
        auto ofm_vec = aie::load_v<32>(ofm_ptr);
        auto result = aie::max(in_vec, ofm_vec);
        aie::store_v(ofm_ptr, result);
        ifm_ptr = add_2d_byte(ifm_ptr, params.ifm_iter);
        ofm_ptr = add_2d_byte(ofm_ptr, params.ofm_iter);
    }
}
```

```cpp
// GOOD: noinline core preserves restrict scope at function boundary
template <typename dtype>
__attribute__((noinline)) void foo_core(
    dtype IFM_DM_BANK *__restrict ifm_ptr,
    dtype OFM_DM_BANK *__restrict ofm_load_ptr,
    dtype OFM_DM_BANK *__restrict ofm_write_ptr,
    params_t &__restrict params) {
    AIE_LOOP_RANGE(4,)
    for (int i = 0; i < params.count; i++) {
        auto in_vec = aie::load_v<32>(ifm_ptr);
        auto ofm_vec = aie::load_v<32>(ofm_load_ptr);
        auto result = aie::max(in_vec, ofm_vec);
        aie::store_v(ofm_write_ptr, result);
        ifm_ptr = add_2d_byte(ifm_ptr, params.ifm_iter);
        ofm_load_ptr = add_2d_byte(ofm_load_ptr, params.ofm_iter);
        ofm_write_ptr = add_2d_byte(ofm_write_ptr, params.ofm_iter);
    }
}

// Inline wrapper: casts pointers and forwards
template <uint8_t batch_size, typename dtype>
INLINE void foo(
    adf::input_buffer_conf<dtype, ...> &__restrict ifm,
    adf::output_buffer_conf<dtype, ...> &__restrict ofm,
    params_t &params) {
    foo_core<dtype>(
        (dtype IFM_DM_BANK *__restrict)ifm.data(),
        (dtype OFM_DM_BANK *__restrict)ofm.data(),
        (dtype OFM_DM_BANK *__restrict)ofm.data(),
        params);
}
```

### Where to apply restrict
### 제한 신청 장소
- All function pointer parameters that don't alias each other
- 서로 별칭을 지정하지 않는 모든 함수 포인터 매개변수
- Parameter struct references (`gemm_params_t &__restrict param`)
- 매개변수 구조체 참조(`gemm_params_t &__restrict param`)
- Setup function arguments (`const uint32_t *__restrict lp`)
- 함수 인수 설정(`const uint32_t *__restrict lp`)


## 3.
## DM Bank Annotations
## DM 은행 주석

### What
### 무엇
AIE data memory is organized into banks.
AIE 데이터 메모리는 뱅크로 구성됩니다.
Each bank supports
각 은행이 지원하는
only a single memory access per cycle.
사이클당 단일 메모리 액세스만 가능합니다.
The VLIW bundle has
VLIW 번들은
two memory slots (slot A and slot B), so to fill both slots
2개의 메모리 슬롯(슬롯 A 및 슬롯 B)을 사용하여 두 슬롯을 모두 채웁니다.
in the same cycle, the two accesses must target different
동일한 주기에서 두 액세스는 서로 다른 목표를 지정해야 합니다.
banks.
은행.
Annotating pointers with their bank assignment tells
은행 할당으로 포인터에 주석을 달면 다음과 같습니다.
the compiler which bank each access uses, enabling it to
각 액세스가 사용하는 뱅크 컴파일러를 통해 다음을 수행할 수 있습니다.
schedule both memory slots in parallel.
두 메모리 슬롯을 병렬로 예약합니다.
Accessing the
액세스
same bank from both slots causes a hardware stall.
두 슬롯의 동일한 뱅크로 인해 하드웨어 정지가 발생합니다.

### Syntax
### 구문
```cpp
// Direct annotation
dtype __aie_dm_resource_a *ptr;    // Bank A
dtype __aie_dm_resource_b *ptr;    // Bank B
dtype __aie_dm_resource_c *ptr;    // Bank C
dtype __aie_dm_resource_d *ptr;    // Bank D
dtype __aie_dm_resource_bd *ptr;   // Bank B or D (flexible)

// Macro shorthand
#define DM_BANK(x) __aie_dm_resource_##x
dtype DM_BANK(a) *ptr;

// Conventional aliases (defined per kernel in *_params.h)
#define IFM_DM_BANK __aie_dm_resource_a
#define WT_DM_BANK  __aie_dm_resource_a
#define OFM_DM_BANK __aie_dm_resource_b
```

### Typical bank assignments
### 일반적인 은행 할당

| Data<br>데이터         | Bank<br>은행 | Rationale<br>이론적 근거                                        |
|-------------|------|--------------------------------------------------|
| Input (IFM)<br>입력(IFM) | A    | Dedicated input bank for parallel access with outputs<br>출력과의 병렬 액세스를 위한 전용 입력 뱅크 |
| Weights<br>가중치     | B    | Separate bank from IFM enables parallel loads<br>IFM과 별도의 뱅크로 병렬 로드 가능    |
| Output (OFM)<br>출력(OFM)| B    | Separate bank from inputs enables parallel store + load<br>입력과 별도의 뱅크로 병렬 저장 + 로드 가능|
| Bias / TDM<br>바이어스/TDM  | C/D  | Secondary data, flexible placement<br>보조 데이터, 유연한 배치               |
| Accumulators<br>어큐뮬레이터| C/D  | Partial sums loaded/stored independently<br>독립적으로 로드/저장된 부분 합계          |


### Common pitfall: same-bank conflict
### 일반적인 함정: 동일 은행 간의 충돌

If two memory operations in the same VLIW bundle target the
동일한 VLIW 번들의 두 메모리 작업이
same bank, the hardware inserts a stall cycle to serialize
동일한 뱅크에서 하드웨어는 직렬화를 위해 정지 사이클을 삽입합니다.
the accesses.
액세스.
In a pipelined loop this stall inflates II.
파이프라인 루프에서 이 실속은 II를 팽창시킵니다.

```cpp
// BAD: both input pointers on bank A -- loads are serialized
static void compute(
    bfloat16 __aie_dm_resource_a *__restrict input_ptr,
    bfloat16 __aie_dm_resource_a *__restrict input2_ptr,
    bfloat16 *__restrict output_ptr,
    int size
)
```

```cpp
// GOOD: input pointers on different banks -- parallel loads
static void compute(
    bfloat16 __aie_dm_resource_a *__restrict input_ptr,
    bfloat16 __aie_dm_resource_b *__restrict input2_ptr,
    bfloat16 *__restrict output_ptr,
    int size
)
```



### Common pitfall: casting away bank annotations
### 일반적인 함정: 은행 주석을 없애는 것

Similar to the restrict pitfall in Section 2, pointer casts can
섹션 2의 제한 함정과 유사하게 포인터 캐스트는 다음을 수행할 수 있습니다.
silently strip DM bank qualifiers.
DM 은행 한정자를 자동으로 제거합니다.
When an intermediate pointer is
중간 포인터가 있을 때
created via cast without preserving the bank annotation, the compiler
뱅크 주석을 보존하지 않고 캐스트를 통해 생성됨, 컴파일러
loses bank information for that pointer and cannot schedule parallel
해당 포인터에 대한 은행 정보를 잃어 병렬을 예약할 수 없습니다.
memory operations.
메모리 작업.

```cpp
// BAD: cast drops the bank annotation -- compiler loses bank info
bfloat16 IFM_DM_BANK *__restrict p_in = ...;
v32bfloat16 *p_in_vec = (v32bfloat16 *)p_in;  // IFM_DM_BANK is gone
```

```cpp
// GOOD: cast preserves the bank annotation
bfloat16 IFM_DM_BANK *__restrict p_in = ...;
v32bfloat16 IFM_DM_BANK *p_in_vec = (v32bfloat16 IFM_DM_BANK *)p_in;
```

This is easy to miss when changing pointer element types (e.g.,
이는 포인터 요소 유형을 변경할 때 놓치기 쉽습니다(예:
scalar to vector) for FIFO or vector load operations.
FIFO 또는 벡터 로드 작업의 경우 스칼라에서 벡터로).
Always carry
항상 휴대하세요
the bank qualifier through every cast in the chain.
체인의 모든 캐스트를 통한 은행 예선.

When a load reaches the multi-slot pseudo materializer without a DM
DM 없이 로드가 다중 슬롯 의사 구체화 장치에 도달하는 경우
bank qualifier, the compiler still picks a slot for that load, but
뱅크 한정자가 있어도 컴파일러는 여전히 해당 로드에 대한 슬롯을 선택하지만
it has to fall back to a bank-blind heuristic.
은행맹인 경험적 방법으로 돌아가야 합니다.
The chosen slot is
선택한 슬롯은
often suboptimal and can prevent the pipeliner from finding the best II.
종종 차선책이며 파이프라인이 최상의 II를 찾지 못하게 할 수 있습니다.
The compiler reports this as a missed-opportunity hint -- a stderr
컴파일러는 이를 놓친 기회 힌트(stderr)로 보고합니다.
warning plus an `-Rpass-missed=aie-multi-slot-pseudo` remark.
경고와 `-Rpass-missed=aie-multi-slot-pseudo` 설명이 추가됩니다.
See
참조
Section 14 for how to read these hints.
이 힌트를 읽는 방법은 섹션 14를 참조하세요.


## 4.
## Store Width and Layout
## 매장 너비 및 레이아웃

How a hot loop writes its results out can have a large effect on
핫 루프가 결과를 기록하는 방식은 다음에 큰 영향을 미칠 수 있습니다.
throughput.
처리량.
The following sections cover three independent techniques for
다음 섹션에서는 세 가지 독립적인 기술을 다룹니다.
making stores cheaper.
매장을 저렴하게 만드는 것.

### Sub-32-bit stores
### 32비트 미만 저장소

Stores narrower than 32 bits (e.g., `int8`, `int16`, `bfloat16`
32비트보다 좁은 저장(예: `int8`, `int16`, `bfloat16`
scalars) are very expensive on AIE.
스칼라)은 AIE에서 매우 비쌉니다.
Part-word loads are no worse than
부분 단어 로드는 다음보다 나쁘지 않습니다.
full-width loads, but sub-word stores force the memory hardware to
전체 너비를 로드하지만 하위 단어 저장은 메모리 하드웨어를 강제로 로드합니다.
perform a full-width read-modify-write internally.
내부적으로 전체 너비 읽기-수정-쓰기를 수행합니다.

- **Performance**: Sub-word stores are significantly more expensive
- **성능**: 하위 단어 스토어는 훨씬 더 비쌉니다.
than 32-bit or vector-width stores because the hardware must read
하드웨어가 읽어야 하기 때문에 32비트 또는 벡터 너비 저장소보다
the surrounding word, merge the sub-word value, and write it back.
주변 단어, 하위 단어 값을 병합하고 다시 작성합니다.
The read-modify-write sequence occupies multiple processor pipeline
읽기-수정-쓰기 시퀀스는 여러 프로세서 파이프라인을 차지합니다.
stages, inflating II and reducing throughput in pipelined loops.
II를 확장하고 파이프라인 루프의 처리량을 줄입니다.
A
single sub-word store in a loop body can prevent the pipeliner from
루프 본문에 단일 하위 단어 저장소가 있으면 파이프라인이
achieving the resource-limited MII.
자원이 제한된 MII를 달성합니다.
- **Alias analysis**: The read-modify-write destroys alias information
- **별칭 분석**: 읽기-수정-쓰기는 별칭 정보를 파괴합니다.
because the hardware touches the entire containing word, not just
하드웨어는 단어뿐만 아니라 포함된 단어 전체에 닿기 때문입니다.
the addressed byte(s).
주소가 지정된 바이트.
The compiler must treat sub-word stores
컴파일러는 하위 단어 저장소를 처리해야 합니다.
conservatively, limiting reordering and pipelining opportunities.
보수적으로 재정렬 및 파이프라인 기회를 제한합니다.

**Rule**: Stores must be at least 32 bits (4 bytes) wide to
**규칙**: 저장소는 폭이 32비트(4바이트) 이상이어야 합니다.
avoid the read-modify-write penalty.
읽기-수정-쓰기 페널티를 피하세요.
Use vector-width stores
벡터 너비 저장소 사용
(`store_v`) in performance-critical loops.
(`store_v`) 성능이 중요한 루프에 있습니다.
Pack data into
데이터를 다음으로 압축
vectors before storing.
저장하기 전의 벡터.
Avoid scalar sub-word stores inside
내부에 스칼라 하위 단어 저장소를 피하세요.
hot loops.
핫 루프.

```cpp
// BAD: scalar int8 stores -- each st.s8 instruction incurs a
//      6-cycle pipeline drain (read-modify-write penalty)
int8_t *ofm = ...;
for (int i = 0; i < 32; i++) {
    ofm[i] = result;  // st.s8: 7 cycles per iteration
}
```

```cpp
// GOOD (bulk data): vector store -- single store, no penalty
aie::vector<int8_t, 32> result_vec = aie::broadcast<int8_t, 32>(result);
aie::store_v(ofm_ptr, result_vec);
ofm_ptr = byte_incr(ofm_ptr, 32);
```

```cpp
// GOOD (few values): pack int8 results into a vector, store once
//      Avoids per-element st.s8; one vector store at the end
aie::vector<int8_t, 4> packed;
packed[0] = a;
packed[1] = b;
packed[2] = c;
packed[3] = d;
aie::store_v(ofm_ptr, packed);  // single 32-bit store
```

### Consider storing to the destination layout directly instead of shuffling
### 셔플링 대신 대상 레이아웃에 직접 저장하는 것을 고려하세요.

When results are computed in one register layout but memory needs another,
하나의 레지스터 레이아웃에서 결과가 계산되지만 메모리에 다른 레지스터 레이아웃이 필요한 경우,
the natural code shuffles/interleaves into the target layout and stores
자연스러운 코드가 대상 레이아웃에 섞이거나 인터리브되어 저장됩니다.
contiguously.
연속적으로.
If the shuffle exists *only* to feed a contiguous store, it
셔플이 인접한 저장소에 공급하기 위해 *만* 존재하는 경우
may be redundant: the store's addressing can often encode the layout for
중복될 수 있습니다. 상점의 주소 지정은 종종 레이아웃을 인코딩할 수 있습니다.
free, letting you drop the shuffle and store each sub-vector directly to
무료로 셔플을 삭제하고 각 하위 벡터를 직접 저장할 수 있습니다.
its destination offset.
대상 오프셋입니다.

This is a **tradeoff, not a rule** -- it swaps shuffle work for store work.
이는 **규칙이 아닌 절충**입니다. 즉, 셔플 작업을 매장 작업으로 바꿉니다.
Dropping the shuffle wins when the write-out path is shuffle-bound; keeping
쓰기 경로가 셔플 바인딩된 경우 셔플을 삭제하면 승리합니다. 유지
it wins when stores are the bottleneck, since direct storing issues *more,
직접 저장 문제가 발생하므로 매장에 병목 현상이 발생하면 승리합니다.
narrower* stores in place of *fewer, wider* ones.
*더 적고 넓은* 매장 대신 더 좁은* 매장을 사용하세요.
There is no layout-
레이아웃이 없습니다.
independent answer -- measure both versions' `BundleCount` (Section 14) and
독립적인 답변 - 두 버전의 `BundleCount`(섹션 14)를 모두 측정하고
achieved II on the actual target and memory layout.
실제 타겟과 메모리 레이아웃에서 II를 달성했습니다.

Two constraints gate whether the direct-store form is worth measuring:
직접 매장 형태가 측정할 가치가 있는지 여부를 결정하는 두 가지 제약 조건은 다음과 같습니다.

- **Each sub-vector store must stay >=32 bits wide** (the rule above).
- **각 하위 벡터 저장소는 폭이 32비트 이상이어야 합니다**(위 규칙).
- If a
- 만약
sub-vector would be narrower than 32 bits, the read-modify-write penalty
하위 벡터가 32비트보다 좁아지면 읽기-수정-쓰기 패널티가 발생합니다.
almost always outweighs the saved shuffles -- keep the shuffle.
거의 항상 저장된 셔플보다 큽니다. 셔플을 유지하세요.
- **Extract indices must be compile-time constants.** A runtime `extract<>`
- **추출 색인은 컴파일 타임 상수여야 합니다.** 런타임 `extract<>`
index may generate a branchy dynamic extract.
인덱스는 가지가 있는 동적 추출을 생성할 수 있습니다.
Hand-unroll the stores over
상점을 손으로 펼칩니다.
the statically known count and guard with a `static_assert` so a layout
정적으로 알려진 카운트와 'static_assert'를 사용한 가드이므로 레이아웃
change fails loudly instead of silently regressing.
변화는 조용히 퇴보하는 대신 큰 소리로 실패합니다.

### Fold a power-of-two output scale into the conversion shift
### 2의 거듭제곱 출력 스케일을 변환 시프트로 접습니다.

When results are scaled by a power of two on the way out, apply the scale
도중에 결과가 2의 거듭제곱으로 조정되면 해당 규모를 적용합니다.
as the shift of the accumulator->vector conversion (SRS) rather than as a
누산기->벡터 변환(SRS)의 이동이 아닌
separate multiply.
별도의 곱하기.
`to_vector<T_out>(shift)` already performs a
`to_Vector<T_out>(shift)`는 이미
shift-round-saturate; passing `SCALE_SHIFT` there is equivalent to a
교대-포화; `SCALE_SHIFT`를 전달하면
trailing multiply by `1 << SCALE_SHIFT`, and removes the multiply
후행에 `1 << SCALE_SHIFT`를 곱하고 곱셈을 제거합니다.
entirely.
완전히.
On `int32` output each `aie::mul` otherwise expands to a
`int32` 출력에서 각 `aie::mul`은 그렇지 않으면 a로 확장됩니다.
`vmul`+`vmac` sequence per accumulator tile.
누산기 타일당 `vmul`+`vmac` 시퀀스.

```cpp
// BAD: convert, then multiply by the scale
auto v = C.to_vector<int32_t>();
v = aie::mul(v, scale).to_vector<int32_t>();   // scale == 1 << SCALE_SHIFT

// GOOD: fold the power-of-two scale into the SRS shift, no multiply
auto v = C.to_vector<int32_t>(SCALE_SHIFT);
```

**Constraint:** valid only when the scale is exactly a power of two
**제약조건:** 척도가 정확히 2의 거듭제곱인 경우에만 유효합니다.
(`scale == 1 << SCALE_SHIFT`); an arbitrary runtime scale still needs the
(`규모 == 1 << SCALE_SHIFT`); 임의의 런타임 규모에는 여전히
multiply.
곱하다.

Both techniques compose in the same write-out path.
두 기술 모두 동일한 쓰기 경로로 구성됩니다.
On a shuffle-bound
셔플 바운드에서
kernel, folding the scale and dropping the shuffle each removed bundles from
커널, 스케일을 접고 제거된 각 번들을 셔플에서 삭제합니다.
the write-out block with II unchanged; on a store-bound path the shuffle-drop
II가 변경되지 않은 쓰기 블록; 상점 경계 경로에서 셔플 드롭
may not help -- measure both.
도움이 되지 않을 수도 있습니다. 두 가지를 모두 측정하세요.

## 5.
## Loop Pragmas Reference
## 루프 프라그마 참조

All macros are defined in `utils.h` (available in the mllib/L1/include/common/utils.h).
모든 매크로는 `utils.h`(mllib/L1/include/common/utils.h에서 사용 가능)에 정의되어 있습니다.

### Macro table
### 매크로 테이블

The table below lists the `AIE_*` wrapper macros and their underlying
아래 표에는 `AIE_*` 래퍼 매크로와 그 기본이 나열되어 있습니다.
`#pragma clang loop` expansions.
`#pragma clang loop` 확장.

| AIE_* macro<br>AIE_* 매크로                       | `#pragma clang loop` option<br>`#pragma clang loop` 옵션        | Purpose<br>목적                                                         |
|-----------------------------------|------------------------------------|----------------------------------------------|
| `AIE_LOOP_MIN_ITERATION_COUNT(N)`<br>`AIE_LOOP_MIN_ITERATION_COUNT(N)` | `min_iteration_count(N)`<br>`최소 반복 횟수(N)`           | Minimum trip count; used by the pipeliner and unroller<br>최소 운행 횟수 파이프라인과 언롤러에서 사용됨           |
| `AIE_LOOP_MAX_ITERATION_COUNT(N)`<br>`AIE_LOOP_MAX_ITERATION_COUNT(N)` | `max_iteration_count(N)`<br>`max_iteration_count(N)`           | Maximum trip count; enables bounded-count optimizations<br>최대 운행 횟수 제한된 개수의 최적화를 가능하게 합니다.          |
| `AIE_LOOP_RANGE(min,)` / `AIE_LOOP_RANGE(min, max)`<br>`AIE_LOOP_RANGE(최소,)` / `AIE_LOOP_RANGE(최소, 최대)` | (both combined)<br>(둘 다 합쳐서)       | Convenience macro combining min and optional max<br>최소값과 선택적 최대값을 결합한 편의 매크로         |
| `AIE_PREPARE_FOR_POSTPIPELINING`<br>`AIE_PREPARE_FOR_POSTPIPELINING`  | `pipeline(disable)`<br>`파이프라인(비활성화)`                | Disable the pre-RA pipeliner; the post-RA pipeliner takes over (see [Section 7](#7-software-pipelining))<br>사전 RA 파이프라인을 비활성화합니다. 포스트 RA 파이프라인이 인계받습니다([섹션 7](#7-software-pipelining) 참조).  |
| `AIE_PREPARE_FOR_PIPELINING`<br>`AIE_PREPARE_FOR_PIPELINING`      | (no-op)<br>(무작동)                            | Legacy; explicitly allows software pipelining (this is the default if a minimum iteration count is provided)<br>레거시; 소프트웨어 파이프라인을 명시적으로 허용합니다(최소 반복 횟수가 제공되는 경우 기본값입니다). |
| `AIE_TRY_INITIATION_INTERVAL(N)`<br>`AIE_TRY_INITIATION_INTERVAL(N)`  | `pipeline_initiation_interval(N)`<br>`파이프라인_시작_간격(N)`  | Suggest a target II to the pipeliner<br>파이프라인에 타겟 II를 제안합니다.                            |
| `AIE_LOOP_NO_UNROLL`<br>`AIE_LOOP_NO_UNROLL`              | `unroll(disable)`<br>`언롤(비활성화)`                  | Prevent loop unrolling<br>루프 풀림 방지                                          |
| `AIE_LOOP_UNROLL_FULL`<br>`AIE_LOOP_UNROLL_FULL`            | `unroll(full)`<br>`풀기(전체)`                     | Fully unroll the loop<br>루프를 완전히 펼칩니다.                                           |
| `AIE_LOOP_UNROLL(N)`<br>`AIE_LOOP_UNROLL(N)`              | `unroll_count(N)`<br>`unroll_count(N)`                  | Unroll by factor N<br>N 요소로 펼치기                                              |
| `AIE_LOOP_HINT(key, value)`<br>`AIE_LOOP_HINT(키, 값)`       | `hint(key, value)`<br>`힌트(키, 값)`                 | Pass arbitrary per-loop metadata to specific compiler passes (see [Section 6](#6-loop-hints))<br>임의의 루프별 메타데이터를 특정 컴파일러 패스에 전달합니다([섹션 6](#6-loop-hints) 참조).   |
| `AIE_KEEP_SWLOOP`<br>`AIE_KEEP_SWLOOP`                 | (TBD)<br>(미정)                              | Keep the software loop structure (prevent ZOL conversion)<br>소프트웨어 루프 구조 유지(ZOL 변환 방지)       |
| `AIE_NO_PIPELINING`<br>`AIE_NO_PIPELINING`               | (TBD)<br>(미정)                              | Disable all software pipelining for the loop<br>루프에 대한 모든 소프트웨어 파이프라인 비활성화                    |

The macros that matter most for kernel performance are
커널 성능에 가장 중요한 매크로는 다음과 같습니다.
`AIE_PREPARE_FOR_POSTPIPELINING`, `AIE_TRY_INITIATION_INTERVAL`,
`AIE_PREPARE_FOR_POSTPIPELINING`, `AIE_TRY_INITIATION_INTERVAL`,
`AIE_LOOP_RANGE`, and `AIE_LOOP_HINT`.
`AIE_LOOP_RANGE` 및 `AIE_LOOP_HINT`.

### Placement
### 배치
Place the `AIE_*` macros immediately before the `for` statement:
`for` 문 바로 앞에 `AIE_*` 매크로를 배치하세요.

```cpp
AIE_PREPARE_FOR_PIPELINING
AIE_LOOP_RANGE(4,)
for (int i = 0; i < param.inner_count; i++) {
    // loop body
}
```




## 6.
## Loop Hints
## 루프 힌트

### What
### 무엇
`AIE_LOOP_HINT(key, value)` is a mechanism to pass arbitrary
`AIE_LOOP_HINT(key, value)`는 임의의 값을 전달하는 메커니즘입니다.
key-value metadata to specific compiler passes on a per-loop basis.
특정 컴파일러에 대한 키-값 메타데이터는 루프별로 전달됩니다.

### How it works
### 작동 원리
On Peano, `AIE_LOOP_HINT(key, value)` expands to:
Peano에서는 `AIE_LOOP_HINT(key, value)`가 다음으로 확장됩니다.
```
#pragma clang loop hint(key, value)
```
This attaches `!llvm.loop.hint.<key>` metadata to the loop's back-edge
`!llvm.loop.hint.<key>` 메타데이터를 루프의 뒤쪽 가장자리에 연결합니다.
branch in LLVM IR.
LLVM IR의 분기.
The compiler's `LoopOptionOverrides` infrastructure
컴파일러의 `LoopOptionOverrides` 인프라
reads this metadata and overrides the corresponding `cl::opt` default
이 메타데이터를 읽고 해당 `cl::opt` 기본값을 재정의합니다.
for that specific loop only.
해당 특정 루프에 대해서만.

### Precedence
### 우선순위
1. Explicit command-line flag (e.g., `--aie-gpr-realloc=true`) --
1. 명시적 명령줄 플래그(예: `--aie-gpr-realloc=true`) --
highest priority, always wins
최우선 순위, 항상 승리
2. Per-loop metadata (`AIE_LOOP_HINT(aie-gpr-realloc, 1)`)
2. 루프별 메타데이터(`AIE_LOOP_HINT(aie-gpr-realloc, 1)`)
3. `cl::opt` compiled-in default -- lowest priority
3. `cl::opt` 컴파일된 기본값 - 가장 낮은 우선순위

### Example: enabling register reallocation for a hot loop
### 예: 핫 루프에 대한 레지스터 재할당 활성화
```cpp
AIE_LOOP_HINT(aie-gpr-realloc, 1)
AIE_LOOP_HINT(aie-realloc-loopaware, 1)
AIE_PREPARE_FOR_PIPELINING
AIE_LOOP_RANGE(6,)
for (int i = 0; i < count; i++) {
    // This specific loop gets register reallocation even if the
    // global default is off
}
```

### Available keys
### 사용 가능한 키
Any `cl::opt` ArgStr defined in the AIE backend can be used as a key.
AIE 백엔드에 정의된 모든 `cl::opt` ArgStr을 키로 사용할 수 있습니다.
The value is either an integer (for bool/int/enum options) or a string.
값은 정수(bool/int/enum 옵션의 경우) 또는 문자열입니다.

The most common use case is tuning the WAW (Write-After-Write)
가장 일반적인 사용 사례는 WAW(Write-After-Write)를 조정하는 것입니다.
register rewriter, which renames registers in pipelined loops to
파이프라인 루프의 레지스터 이름을 다음으로 바꾸는 레지스터 리라이터(Register Rewriter)
eliminate WAW hazards.
WAW 위험을 제거합니다.
Available options:
사용 가능한 옵션:

| Key<br>열쇠                          | Type<br>유형    | Default<br>기본값 | Effect<br>효과                                                     |
|------------------------------|---------|---------|-------------------------------------------------------------|
| `aie-gpr-realloc`<br>`aie-gpr-realloc`           | 0/1     | 0       | Enable GPR register reallocation in addition to vector regs<br>벡터 레지스터 외에 GPR 레지스터 재할당 활성화 |
| `aie-realloc-loopaware`<br>`aie-realloc-loopaware`     | 0/1     | 0       | Prime the LRU queue so allocations wrap around loop boundary<br>할당이 루프 경계를 둘러싸도록 LRU 대기열을 프라이밍합니다.|
| `aie-aggressive-realloc`<br>`aie-공격적-realloc`    | 0/1     | 0       | Aggressively de-allocate live-through registers<br>적극적으로 라이브 스루 레지스터 할당 취소             |
| `aie-reg-rewrite-mode`<br>`aie-reg-다시 쓰기 모드`      | string<br>문자열  | auto<br>자동    | Rewriting mode: `basic`, `auto`, `latencyaware`, `swpaware`, `swpaware-auto`<br>재작성 모드: `basic`, `auto`, `latencyaware`, `swpaware`, `swpaware-auto` |
| `aie-waw-reg-rewrite-min-lat`<br>`aie-waw-reg-rewrite-min-lat`| int<br>정수    | 3       | Minimum operand latency considered for WAW rewriting<br>WAW 재작성을 위해 고려되는 최소 피연산자 대기 시간        |
| `aie-realloc-ii-bias`<br>`aie-realloc-ii-bias`       | int<br>정수     | 0       | MinII bias for swpaware mode<br>swpaware 모드에 대한 MinII 바이어스                                |

Example: enabling GPR reallocation with loop-aware priming on a
예: 루프 인식 프라이밍으로 GPR 재할당 활성화
specific hot loop:
특정 핫 루프:
```cpp
AIE_LOOP_HINT(aie-gpr-realloc, 1)
AIE_LOOP_HINT(aie-realloc-loopaware, 1)
AIE_LOOP_HINT(aie-reg-rewrite-mode, swpaware)
```

### Disabling per-loop overrides
### 루프별 재정의 비활성화
For debugging, pass `--aie-ignore-loop-hints` to `llc`
디버깅하려면 `--aie-ignore-loop-hints`를 `llc`에 전달하세요.
to ignore all `AIE_LOOP_HINT` metadata globally.
모든 'AIE_LOOP_HINT' 메타데이터를 전역적으로 무시합니다.


## 7.
## Software Pipelining
## 소프트웨어 파이프라이닝

### What the pipeliner does
### 파이프라인이 하는 일
Software pipelining overlaps consecutive iterations of a loop body.
소프트웨어 파이프라이닝은 루프 본문의 연속 반복과 겹칩니다.
Instead of completing iteration N before starting N+1, the pipeliner
N+1을 시작하기 전에 반복 N을 완료하는 대신 파이프라인은
interleaves them so that, e.g., loads for iteration N+2 happen while
예를 들어 반복 N+2에 대한 로드가 발생하도록 인터리브합니다.
iteration N is computing and iteration N-1 is storing.
반복 N은 계산 중이고 반복 N-1은 저장 중입니다.

This fills more VLIW slots per cycle and hides long latencies (memory,
이는 주기당 더 많은 VLIW 슬롯을 채우고 긴 대기 시간(메모리,
multiply-accumulate).
곱셈-누적).

### Pre-RA vs post-RA pipeliner
### Pre-RA vs Post-RA 파이프라인
Peano has two software pipeliners:
Peano에는 두 가지 소프트웨어 파이프라인이 있습니다.
- **Pre-RA pipeliner** (MachinePipeliner): Runs before register
- **Pre-RA 파이프라인**(MachinePipeliner): 등록 전 실행
allocation.
할당.
It has more scheduling freedom but may produce
일정의 자유도가 더 높지만
schedules that cause high register pressure and spilling.
높은 레지스터 압력과 유출을 유발하는 일정.
It can also create a guarded prologue (see `AIE_KEEP_SWLOOP`).
또한 보호된 프롤로그를 생성할 수도 있습니다(`AIE_KEEP_SWLOOP` 참조).
- **Post-RA pipeliner**: Runs after register allocation.
- **Post-RA 파이프라인**: 레지스터 할당 후에 실행됩니다.
- Register
- 등록
assignments are fixed, so it avoids spill-related issues.
할당이 고정되어 유출 관련 문제를 방지합니다.

The compiler automatically selects which pipeliner to use for each
컴파일러는 각 파이프라인에 사용할 파이프라인을 자동으로 선택합니다.
loop.
루프.
There is no guarantee that a particular pipeliner was used.
특정 파이프라인이 사용되었다는 보장은 없습니다.

`AIE_PREPARE_FOR_POSTPIPELINING` disables the pre-RA pipeliner for
`AIE_PREPARE_FOR_POSTPIPELINING`은 사전 RA 파이프라인을 비활성화합니다.
that loop (via `#pragma clang loop pipeline(disable)`), guaranteeing
해당 루프(`#pragma clang looppipeline(disable)`를 통해)는 다음을 보장합니다.
the post-RA pipeliner will be the one to attempt pipelining.
포스트 RA 파이프라인이 파이프라이닝을 시도하는 사람이 될 것입니다.
This is
이것은
the only way to ensure the post-RA pipeliner runs on a given loop.
이는 RA 이후 파이프라인이 특정 루프에서 실행되도록 하는 유일한 방법입니다.

When to use `AIE_PREPARE_FOR_POSTPIPELINING`:
`AIE_PREPARE_FOR_POSTPIPELINING`을 사용하는 경우:
- The pipelined loop has spill code in the loop body
- 파이프라인 루프의 루프 본문에 유출 코드가 있습니다.
- The pipelined loop contains sequences of register moves
- 파이프라인 루프에는 일련의 레지스터 이동이 포함됩니다.
("register-fifos")
("등록-fifos")

### Key concepts
### 주요 개념

Software pipelining splits the loop body into stages of equal length
소프트웨어 파이프라이닝은 루프 본문을 동일한 길이의 단계로 분할합니다.
that are then executed in parallel in pipeline fashion.
그런 다음 파이프라인 방식으로 병렬로 실행됩니다.
Each stage
각 스테이지
takes II cycles, and NS stages overlap in flight.
II 사이클을 취하고 NS 단계가 비행 중에 겹칩니다.

**II (Initiation Interval)**: The number of cycles between starting
**II(개시 간격)**: 시작 사이의 주기 수
two consecutive iterations.
두 번의 연속 반복.
Lower II = higher throughput.
낮은 II = 더 높은 처리량.
An II of 1
1의 II
means a new iteration starts every cycle.
매 사이클마다 새로운 반복이 시작된다는 의미입니다.

**MII (Minimum Initiation Interval)**: The theoretical lower bound,
**MII(최소 개시 간격)**: 이론적 하한,
determined by:
다음에 의해 결정됩니다:
- **Resource MII**: Determined by the most-used VLIW slot.
- **리소스 MII**: 가장 많이 사용되는 VLIW 슬롯에 따라 결정됩니다.
Each slot can execute one operation per cycle; the slot
각 슬롯은 주기당 하나의 작업을 실행할 수 있습니다. 슬롯
with the highest demand sets the Resource MII.
수요가 가장 높은 리소스 MII를 설정합니다.
For example,
예를 들어,
if one iteration uses 2 slot-A ops, 1 slot-B op, 1 ALU op,
한 번의 반복이 슬롯 A 작업 2개, 슬롯 B 작업 1개, ALU 작업 1개를 사용하는 경우,
and 1 store, the bottleneck is slot A (2 uses), so
1개 매장에서 병목 현상은 슬롯 A(2개 사용)이므로
Resource MII = 2.
자원 MII = 2.
- **Recurrence MII**: Determined by data dependence cycles within the
- **반복 MII**: 데이터 의존성 주기에 따라 결정됩니다.
loop (e.g., accumulator feedback chains)
루프(예: 누산기 피드백 체인)

MII = max(Resource MII, Recurrence MII).
MII = 최대(리소스 MII, 반복 MII).
The pipeliner tries to find a
파이프라인은 다음을 찾으려고 합니다.
schedule at MII; if it fails it increments II and retries.
MII의 일정; 실패하면 II가 증가하고 재시도됩니다.

**NS (Number of Stages)**: How many overlapping iterations are in
**NS(단계 수)**: 중복되는 반복 횟수
flight.
비행.
More stages = more overlap but also more prologue/epilogue
더 많은 단계 = 더 많은 중복과 더 많은 프롤로그/에필로그
code and higher register pressure.
코드 및 더 높은 레지스터 압력.

**Minimum trip count requirement**: The loop must execute at least NS
**최소 트립 횟수 요구 사항**: 루프는 최소한 NS를 실행해야 합니다.
times for the pipelined schedule to be valid.
파이프라인 일정이 유효한 시간입니다.
This is why
이것이 바로 이유이다
`AIE_LOOP_RANGE` is important -- it tells the pipeliner the loop will
`AIE_LOOP_RANGE`는 중요합니다. 파이프라인에 루프가 실행될 것임을 알려줍니다.
iterate enough times.
충분히 반복하세요.

### Trip count and AIE_LOOP_RANGE
### 이동 횟수 및 AIE_LOOP_RANGE

```cpp
// Tell the compiler this loop executes at least 4 times
AIE_LOOP_RANGE(4,)
for (int i = 0; i < param.count; i++) { ... }

// Tell the compiler this loop executes between 4 and 16 times
AIE_LOOP_RANGE(4, 16)
for (int i = 0; i < param.count; i++) { ... }
```

If you set the minimum too low, the pipeliner may produce suboptimal code.
최소값을 너무 낮게 설정하면 파이프라인이 차선의 코드를 생성할 수 있습니다.
If you set it too high
너무 높게 설정하면
(higher than the actual runtime count), the program will be incorrect.
(실제 런타임 카운트보다 높음) 프로그램이 올바르지 않게 됩니다.

### Suggesting a target II
### 타겟 제안 II
`AIE_TRY_INITIATION_INTERVAL(N)` tells the postpipeliner to try
`AIE_TRY_INITIATION_INTERVAL(N)`은 포스트파이프라이너에게 시도하라고 지시합니다.
harder to reach II = N.
II = N에 도달하기가 더 어렵습니다.
Normally the pipeliner tries a sequence of
일반적으로 파이프라인은 다음의 시퀀스를 시도합니다.
heuristics at each candidate II and moves on if they fail.
각 후보 II에 대해 휴리스틱을 적용하고 실패하면 계속 진행합니다.
When a
언제
target II is set, the pipeliner additionally invokes its exact solver
타겟 II가 설정되면 파이프라인은 정확한 솔버를 추가로 호출합니다.
at that II, significantly increasing the chance of finding a valid
II에서는 유효한 것을 찾을 확률이 크게 높아집니다.
schedule at the cost of longer compile time.
컴파일 시간이 길어지는 대신 일정을 계획하세요.

```cpp
AIE_TRY_INITIATION_INTERVAL(4)
AIE_PREPARE_FOR_PIPELINING
AIE_LOOP_RANGE(8,)
for (int i = 0; i < count; i++) { ... }
```

The target II is not a guarantee.
타겟 II는 보장되지 않습니다.
If no valid schedule exists at the
유효한 일정이 없는 경우
requested II (due to resource or recurrence constraints), the
요청 II(자원 또는 반복 제약으로 인해)
pipeliner continues to higher II values.
파이프라인은 계속해서 더 높은 II 값을 유지합니다.
Use optimization remarks to
최적화 설명을 사용하여
verify the achieved II.
II 달성을 확인합니다.


## 8.
## Loop Versioning
## 루프 버전 관리

### The problem
### 문제
Software pipelining needs a minimum trip count (typically 4-8,
소프트웨어 파이프라인에는 최소 트립 횟수가 필요합니다(일반적으로 4-8,
depending on NS).
NS에 따라 다름).
If the loop count is a runtime parameter that might
루프 카운트가 다음과 같은 런타임 매개변수인 경우
be small, the pipeliner must either:
크기가 작으면 파이프라인은 다음 중 하나를 수행해야 합니다.
- Refuse to pipeline (safe but slow for large counts)
- 파이프라인 거부(안전하지만 대량의 경우 느림)
- Generate guarded prologue/epilogue (code size overhead)
- 보호된 프롤로그/에필로그 생성(코드 크기 오버헤드)

### Prefer AIE_LOOP_RANGE over loop versioning
### 루프 버전 관리보다 AIE_LOOP_RANGE를 선호합니다.
Loop versioning duplicates the entire loop body, which **doubles
루프 버전 관리는 전체 루프 본문을 복제합니다.
program memory (PM) usage** for that loop.
해당 루프에 대한 프로그램 메모리(PM) 사용량**입니다.
Before reaching for
도달하기 전에
`VERSIONED_LOOP`, consider whether you can simply increase
`VERSIONED_LOOP`, 단순히 늘릴 수 있는지 고려하세요.
`AIE_LOOP_RANGE` to guarantee a sufficient minimum iteration count.
충분한 최소 반복 횟수를 보장하는 `AIE_LOOP_RANGE`입니다.

If the kernel's calling convention or tiling guarantees the loop
커널의 호출 규칙이나 타일링이 루프를 보장하는 경우
always executes at least N times, set `AIE_LOOP_RANGE(N,)` and avoid
항상 최소 N번 실행하고 `AIE_LOOP_RANGE(N,)`을 설정하고 피하세요.
versioning entirely.
완전히 버전 관리.
Loop versioning should only be used when the trip
루프 버전 관리는 여행할 때만 사용해야 합니다.
count genuinely varies below the pipelining threshold at runtime.
개수는 실제로 런타임 시 파이프라인 임계값 아래로 달라집니다.

### The solution: VERSIONED_LOOP
### 해결책: VERSIONED_LOOP
Split the loop into two runtime paths:
루프를 두 개의 런타임 경로로 분할합니다.
- **High-count path**: The trip count is >= MinIters, so pipelining is
- **많은 경로**: 이동 횟수는 >= MinIters이므로 파이프라인은
enabled with appropriate `AIE_LOOP_RANGE` and pipelining pragmas.
적절한 `AIE_LOOP_RANGE` 및 파이프라이닝 프라그마를 사용하여 활성화됩니다.
- **Low-count path**: The trip count is < MinIters, so pipelining is
- **낮은 횟수의 경로**: 이동 횟수는 < MinIters이므로 파이프라인은
disabled and the loop runs as simple scalar code.
비활성화되고 루프는 간단한 스칼라 코드로 실행됩니다.

```cpp
// Define the loop body as a lambda
auto body = [&]() __attribute__((always_inline)) {
    target_vec = aie::load_v<VEC_LEN>(ofm_load_ptr);
    ifm_vec = aie::load_v<VEC_LEN>(ifm_load_ptr);
    store_v(ofm_write_ptr, aie::max(target_vec, ifm_vec));
    ofm_load_ptr = byte_incr(ofm_load_ptr, byte_incr_val);
    ofm_write_ptr = byte_incr(ofm_write_ptr, byte_incr_val);
};

// 6 = minimum iterations for pipelining to be profitable
VERSIONED_LOOP(6, param.inner_loop_count, body);
```

### The VERSIONED_LOOP macro
### VERSIONED_LOOP 매크로
Defined in `utils.h`:
`utils.h`에 정의됨:
```cpp
#define VERSIONED_LOOP(MinIters, count, body, ...) \
    do { \
        if ((count) >= (MinIters)) { \
            __VA_ARGS__ \
            AIE_PREPARE_FOR_PIPELINING \
            AIE_LOOP_RANGE(MinIters,) \
            for (int _vl_i = 0; _vl_i < (count); _vl_i++) { \
                (body)(); \
            } \
        } else { \
            AIE_NO_PREPARE_FOR_PIPELINING \
            AIE_LOOP_RANGE(1,) \
            AIE_LOOP_NO_UNROLL \
            for (int _vl_i = 0; _vl_i < (count); _vl_i++) { \
                (body)(); \
            } \
        } \
    } while(0)
```

The optional trailing `...` argument accepts extra pragma macros for
선택적 후행 '...' 인수는 다음에 대한 추가 pragma 매크로를 허용합니다.
the high-count path (e.g., `AIE_LOOP_NO_UNROLL`).
개수가 많은 경로(예: `AIE_LOOP_NO_UNROLL`)

### Choosing MinIters
### MinIter 선택
MinIters must be >= the number of pipeline stages (NS) the compiler
MinIters는 컴파일러의 파이프라인 단계(NS) 수보다 커야 합니다.
will find.
찾을 것입니다.
This depends on the loop body complexity:
이는 루프 본문의 복잡성에 따라 다릅니다.
- Simple load-compute-store: NS ~ 3-4, use MinIters = 4
- 단순 로드-계산-저장: NS ~ 3-4, MinIters = 4 사용
- Complex MAC with multiple loads: NS ~ 5-7, use MinIters = 6-8
- 여러 로드가 있는 복잡한 MAC: NS ~ 5-7, MinIters = 6-8 사용
- Check optimization remarks for the actual NS
- 실제 NS에 대한 최적화 설명을 확인하세요.



### Lambda pattern details
### 람다 패턴 세부정보
The body must be a named lambda, not an inline lambda in the macro
본문은 매크로의 인라인 람다가 아니라 명명된 람다여야 합니다.
call.
전화.
This avoids preprocessor issues with commas in template
이는 템플릿의 쉼표와 관련된 전처리기 문제를 방지합니다.
arguments:
인수:

```cpp
// BAD: commas in template args confuse the preprocessor
VERSIONED_LOOP(6, count, [&]() {
    aie::max<aie::vector<int8_t, 32>, aie::vector<int8_t, 32>>(...);
});

// GOOD: named lambda
auto body = [&]() __attribute__((always_inline)) {
    aie::max<aie::vector<int8_t, 32>, aie::vector<int8_t, 32>>(...);
};
VERSIONED_LOOP(6, count, body);
```


## 9.
## Function Structure for Pipelining
## 파이프라이닝의 함수 구조

### NOINLINE core, INLINE wrapper
### NOINLINE 코어, INLINE 래퍼

See [Section 2, Pattern: inline wrapper + noinline core](#pattern-inline-wrapper--noinline-core)
[섹션 2, 패턴: 인라인 래퍼 + noinline 코어](#pattern-inline-wrapper--noinline-core)를 참조하세요.
for the full explanation of why `noinline` is required.
'noinline'이 필요한 이유에 대한 자세한 설명은 여기를 참조하세요.
In short:
간단히 말해서:
LLVM does not support scoped restrict, so inlining a function
LLVM은 범위 제한을 지원하지 않으므로 함수 인라인
loses the `__restrict` guarantees on its parameters.
매개변수에 대한 `__restrict` 보장을 잃습니다.
When the same
같은 때
buffer is passed for both load and store, this forces the pipeliner
로드와 저장 모두에 버퍼가 전달되므로 파이프라인이 강제로 실행됩니다.
to assume aliasing and prevents it from overlapping iterations.
앨리어싱을 가정하고 반복이 겹치는 것을 방지합니다.

The core function containing the hot loop must be `NOINLINE`.
핫 루프를 포함하는 핵심 기능은 'NOINLINE'이어야 합니다.
The
는
public API function should be a thin `INLINE` wrapper that casts
공개 API 함수는 캐스트하는 얇은 'INLINE' 래퍼여야 합니다.
pointers and forwards them to the core.
포인터를 코어에 전달합니다.

## 10.
## Pointer Increment Patterns
## 포인터 증가 패턴

### Post-increment vs indexed access
### 사후 증분과 인덱싱된 액세스
Post-increment addressing maps directly to AIE hardware addressing
증분 후 주소 지정은 AIE 하드웨어 주소 지정에 직접 매핑됩니다.
modes, saving an instruction in the loop body:
모드, 루프 본문에 명령어 저장:

```cpp
// BAD: indexed access -- requires separate address computation
aie::vector<bfloat16, 64> in_vec = aie::vector<bfloat16, 64>(ptrA[i]);
ptrB[i] = out_vec.to_native();
```

```cpp
// GOOD: post-increment -- maps to hardware postinc mode
aie::vector<bfloat16, 64> in_vec = aie::vector<bfloat16, 64>(*ptrA++);
*ptrB++ = out_vec.to_native();
```

### 2D/3D hardware iterators
### 2D/3D 하드웨어 반복자
AIE provides hardware address generators with 2D and 3D striding.
AIE는 2D 및 3D 스트라이딩 기능을 갖춘 하드웨어 주소 생성기를 제공합니다.
Use the iterator macros to leverage them:
반복자 매크로를 사용하여 활용하세요.

```cpp
SETUP_ITR3D_COUNTERS(iterator)

// Inside the loop:
input_ptr = INCR_ITR3D_PARAM(input_ptr, iterator, params)
```

These macros use `add_3d_byte` / `add_2d_byte` intrinsics that map to
이 매크로는 다음에 매핑되는 `add_3d_byte` / `add_2d_byte` 내장 함수를 사용합니다.
the hardware's multi-dimensional address generation unit.
하드웨어의 다차원 주소 생성 장치입니다.
The hardware
하드웨어
handles wrap and stride in zero cycles.
제로 사이클에서 랩과 스트라이드를 처리합니다.

### FIFO reads
### FIFO 읽기
For unaligned streaming access, use FIFO intrinsics:
정렬되지 않은 스트리밍 액세스의 경우 FIFO 내장 함수를 사용하세요.
```cpp
fifo_state_t fA;
fA.pos = 0;
aie::vector<bfloat16, 32> v0 = fifo_ld_pop(pInput, fA);
```
FIFO operations manage alignment internally and map to efficient
FIFO 작업은 내부적으로 정렬을 관리하고 효율적으로 매핑합니다.
hardware primitives.
하드웨어 프리미티브.


## 11.
## Vector Load/Store Alignment
## 벡터 로드/저장 정렬

Vector loads and stores require the address to be aligned to the
벡터 로드 및 저장에는 주소가
vector width.
벡터 폭.
Misaligned accesses cause undefined behavior or
잘못 정렬된 액세스로 인해 정의되지 않은 동작이 발생하거나
hardware exceptions.
하드웨어 예외.

### Alignment requirements
### 정렬 요구 사항
- 256-bit (32-byte) vectors: address must be 32-byte aligned
- 256비트(32바이트) 벡터: 주소는 32바이트로 정렬되어야 합니다.
- 512-bit (64-byte) vectors: address must be 64-byte aligned
- 512비트(64바이트) 벡터: 주소는 64바이트로 정렬되어야 합니다.
- 1024-bit (128-byte) vectors: address must be 128-byte aligned
- 1024비트(128바이트) 벡터: 주소는 128바이트로 정렬되어야 합니다.

### Ensuring alignment
### 정렬 보장
- **Buffer allocation**: ADF buffers allocated via `adf::location`
- **버퍼 할당**: `adf::location`을 통해 할당된 ADF 버퍼
are naturally aligned.
자연스럽게 정렬됩니다.
Stack-allocated vectors should use
스택 할당 벡터는 다음을 사용해야 합니다.
`alignas`:
`alignas`:
  ```cpp
  alignas(64) int8_t local_buf[256];
  ```
- **Pointer arithmetic**: When using `byte_incr` or manual pointer
- **포인터 연산**: `byte_incr` 또는 수동 포인터를 사용할 때
offsets, ensure the increment preserves alignment.
오프셋을 사용하면 증분이 정렬을 유지하는지 확인하세요.
Incrementing by
증분 기준
the vector width in bytes always preserves alignment if the base
바이트 단위의 벡터 너비는 기본이 다음과 같은 경우 항상 정렬을 유지합니다.
was aligned.
정렬되었습니다.
- **Parameter-driven offsets**: If a runtime parameter controls a
- **매개변수 기반 오프셋**: 런타임 매개변수가
pointer offset, verify the parameter is a multiple of the vector
포인터 오프셋, 매개변수가 벡터의 배수인지 확인하세요.
alignment.
정렬.
Misaligned offsets from parameters are a common source
매개변수의 잘못 정렬된 오프셋이 일반적인 원인입니다.
of hard-to-debug crashes.
디버그하기 어려운 충돌이 발생했습니다.
- **FIFO reads**: FIFO intrinsics (`fifo_ld_pop`) handle unaligned
- **FIFO 읽기**: FIFO 내장 함수(`fifo_ld_pop`) 핸들이 정렬되지 않았습니다.
access internally and are safe for arbitrary alignment.
내부적으로 접근할 수 있으며 임의 정렬에 안전합니다.

### Diagnosing alignment issues
### 정렬 문제 진단
Misaligned vector accesses typically manifest as simulation hangs or
잘못 정렬된 벡터 액세스는 일반적으로 시뮬레이션이 중단되거나
incorrect data (silent corruption).
잘못된 데이터(자동 손상).
If a kernel produces wrong results
커널이 잘못된 결과를 생성하는 경우
only for certain parameter combinations, check whether the failing
특정 매개변수 조합에 대해서만 실패 여부를 확인하십시오.
cases produce misaligned pointers.
경우에는 잘못 정렬된 포인터가 생성됩니다.

## 12.
## Advanced Loop Optimization Techniques
## 고급 루프 최적화 기술

### Prefer compiler pipelining over manual unrolling
### 수동 언롤링보다 컴파일러 파이프라인을 선호합니다.
Manual loop unrolling (duplicating the loop body N times, reducing the
수동 루프 풀기(루프 본문을 N번 복제하여
trip count by N) increases code size and prevents the post-RA
N에 의한 트립 횟수) 코드 크기를 늘리고 사후 RA를 방지합니다.
pipeliner from working effectively.
파이프라인이 효과적으로 작동하지 못하게 됩니다.
The pipeliner needs a single-body
파이프라인에는 단일 본체가 필요합니다.
loop to overlap iterations.
반복을 겹치는 루프.

Instead of manual unrolling, use `AIE_LOOP_RANGE` with a minimum
수동 언롤링 대신 최소한 `AIE_LOOP_RANGE`를 사용하세요.
equal to the former unroll factor, and let the compiler pipeline:
이전 언롤링 요소와 동일하고 컴파일러 파이프라인이 다음을 수행하도록 합니다.

```cpp
// BAD: manual 8x unroll -- pipeliner cannot overlap iterations
for (int i = 0; i < count; i++) {
    vec1 = load_v<16>(in_ptr); INCR_ITR2D_PARAM(in_ptr, ...)
    vec2 = load_v<16>(in_ptr); INCR_ITR2D_PARAM(in_ptr, ...)
    // ... 6 more copies ...
    store_v(out_ptr, vec1); INCR_ITR2D_PARAM(out_ptr, ...)
    store_v(out_ptr, vec2); INCR_ITR2D_PARAM(out_ptr, ...)
    // ... 6 more copies ...
}

// GOOD: single body, pipelined -- compiler overlaps iterations
AIE_PREPARE_FOR_PIPELINING
AIE_LOOP_RANGE(8,)
for (int i = 0; i < count * 8; i++) {
    vec = load_v<16>(in_ptr); INCR_ITR2D_PARAM(in_ptr, ...)
    store_v(out_ptr, vec);    INCR_ITR2D_PARAM(out_ptr, ...)
}
```

Multiply the trip count by the former unroll factor and set
이동 횟수에 이전 언롤 인수를 곱하여 설정합니다.
`AIE_LOOP_RANGE` to guarantee the compiler the new minimum.
`AIE_LOOP_RANGE`는 컴파일러에게 새로운 최소값을 보장합니다.

### Avoid manually peeling the first iteration
### 첫 번째 반복을 수동으로 벗기지 마세요.

A related anti-pattern is **hand-peeling the first loop iteration** --
관련된 안티 패턴은 **첫 번째 루프 반복을 손으로 벗기기**입니다.
pre-loading inputs and issuing the first computation *before* the loop, then
입력을 미리 로드하고 루프 *전에* 첫 번째 계산을 실행한 다음
starting the loop at the second iteration.
두 번째 반복에서 루프를 시작합니다.
This steals iterations from the
이는 다음에서 반복을 훔칩니다.
pipeliner, which is the toolchain's expert at loop peeling: it already
루프 필링의 툴체인 전문가인 파이프라인: 이미
constructs an optimal prologue and decides how many iterations to peel.
최적의 프롤로그를 구성하고 벗겨낼 반복 횟수를 결정합니다.
Hand-peeling cannot lower the achieved II and only takes that decision away
핸드 필링은 달성된 II를 낮출 수 없으며 그 결정을 취소할 뿐입니다.
from the component best equipped to make it.
그것을 만들기 위해 가장 잘 갖춰진 구성 요소로부터.

Prefer a single loop from `i = 0` with pre-zeroed accumulators.
사전에 0이 지정된 누산기를 사용하여 'i = 0'의 단일 루프를 선호합니다.
Seeding an
시딩
accumulator with `zero + first_product` is numerically identical to the
`zero + first_product`를 갖는 누산기는 숫자상으로 동일합니다.
peeled version, so no correctness is lost:
벗겨진 버전이므로 정확성이 손실되지 않습니다.

```cpp
// BAD: first iteration peeled before the loop, which starts at i = 1.
acc = first_product(...);                       // duplicated body
for (unsigned i = 1; i < n; ++i) acc = mac(acc, ...);

// GOOD: single body from i = 0, accumulator pre-zeroed.
acc = aie::zeros<...>();
for (unsigned i = 0; i < n; ++i) acc = mac(acc, ...);
```

**Exception -- deliberate peeling in nested loops.** When an inner
**예외 - 중첩된 루프에서 의도적으로 벗겨짐.**
software-pipelined loop's prologue/epilogue is too short to hide a
소프트웨어 파이프라인 루프의 프롤로그/에필로그가 너무 짧아서
long-latency instruction in the *outer* loop, peeling a few inner iterations
*외부* 루프의 지연 시간이 긴 명령, 몇 번의 내부 반복 제거
up into the outer loop lets them run in parallel with that outer-loop latency.
외부 루프로 올라가면 외부 루프 대기 시간과 병렬로 실행될 수 있습니다.
This is a deliberate scheduling trade, not accidental duplication -- justify it
이는 우발적인 중복이 아닌 의도적인 일정 조정 거래입니다. 이를 정당화하십시오.
with a measured II/latency improvement (Section 14), since the default guidance
측정된 II/지연 시간 개선(섹션 14), 기본 지침 이후
above still holds for the common single-loop case.
위의 내용은 일반적인 단일 루프 사례에도 적용됩니다.

### AIE_LOOP_UNROLL for controlled unrolling
### 제어된 언롤링을 위한 AIE_LOOP_UNROLL
When the loop body is too small for the pipeliner to fill all VLIW
파이프라인이 모든 VLIW를 채우기에 루프 본문이 너무 작은 경우
slots, controlled unrolling can help by giving the pipeliner a larger
슬롯, 제어된 언롤링은 파이프라인에 더 큰 슬롯을 제공하여 도움이 될 수 있습니다.
loop body to schedule.
루프 본문을 예약합니다.
Use `AIE_LOOP_UNROLL(N)` together with
`AIE_LOOP_UNROLL(N)`을 다음과 함께 사용하세요.
pipelining pragmas:
파이프라이닝 프라그마:

```cpp
AIE_LOOP_UNROLL(2)
AIE_PREPARE_FOR_PIPELINING
AIE_LOOP_RANGE(4,)
for (int w = 0; w < w_loop_cnt; w++) {
    // The compiler unrolls 2x, then pipelines the unrolled body
}
```

This is different from manual unrolling: the compiler still sees a
이는 수동 언롤링과 다릅니다. 컴파일러는 여전히
loop and can pipeline the unrolled body.
루프를 만들고 펼쳐진 몸체를 파이프라인할 수 있습니다.
The trip count does not
운행횟수는 그렇지 않습니다
change (the compiler adjusts internally).
변경합니다(컴파일러는 내부적으로 조정합니다).

### Minimize type conversion chains
### 유형 변환 체인 최소화
Type conversion chains that bounce through intermediate types add
중간 유형을 통과하는 유형 변환 체인 추가
instructions to the loop body, increasing II.
루프 본문에 명령을 전달하고 II를 증가시킵니다.
Use the most direct
가장 직접적인 방법을 사용하세요.
conversion available:
변환 가능:

```cpp
// BAD: int8 -> acc32 -> int32 -> float -> accfloat -> bfloat16
//   5 conversion steps, each adding instructions to the loop body
aie::vector<int8, 16> in_int8 = *it_input++;
aie::accum<acc32, 16> in_acc32;
in_acc32.template from_vector(in_int8);
aie::vector<int32, 16> in_int32 = in_acc32.to_vector<int32>();
aie::vector<float, 16> in_float = aie::to_float(in_int32, shift);
aie::accum<accfloat, 16> in_accfloat;
in_accfloat.template from_vector(in_float);
auto result = in_accfloat.to_vector<bfloat16>();

// GOOD: int8 -> float -> bfloat16
//   2 conversion steps using direct API
aie::vector<int8, 16> in_int8 = *it_input++;
aie::vector<float, 16> in_float = aie::to_float(in_int8, shift);
auto result = aie::accum<accfloat, 16>(in_float).to_vector<bfloat16>();
```

Check the AIE API documentation for direct conversion functions
직접 변환 기능에 대해서는 AIE API 문서를 확인하세요.
(`aie::to_float`, `aie::exp2`, etc.) before writing multi-step
(`aie::to_float`, `aie::exp2` 등) 다단계를 작성하기 전에
conversion chains.
변환 체인.

### Stay in the native data type
### 기본 데이터 유형 유지
When processing data that will be converted later, delay the
나중에 변환될 데이터를 처리할 때
conversion and operate in the native type as long as possible.
변환하고 가능한 한 네이티브 유형으로 작동합니다.
This
이
reduces the number of instructions in the loop body:
루프 본문의 명령 수를 줄입니다.

```cpp
// BAD: convert int8 -> float for max reduction
//   float max is more expensive, and conversion adds instructions
for (int i = 0; i < elem_iters; i++) {
    aie::vector<int8, 16> in_int8 = *input_max++;
    // ... 5-step conversion to bfloat16 ...
    max_vec = aie::max(max_vec, converted_vec);
}

// GOOD: find max directly in int8 domain
aie::vector<int8, 16> max_vec_int8 = aie::broadcast<int8, 16>((int8)-128);
AIE_PREPARE_FOR_PIPELINING
AIE_LOOP_RANGE(4,)
for (int i = 0; i < elem_iters; i++) {
    aie::vector<int8, 16> in_int8 = *input_max++;
    max_vec_int8 = aie::max(max_vec_int8, in_int8);
}
int8 max_value = aie::reduce_max(max_vec_int8);
// Convert only the final scalar result
```

### Early exit for zero-trip-count loops
### 제로 트립 카운트 루프의 조기 종료
Always guard loops that may have zero iterations:
반복 횟수가 0일 수 있는 루프를 항상 보호하세요.

```cpp
// GOOD: skip all setup when nothing to do
if (param.trim_loops == 0)
    return;
// ... iterator setup ...
AIE_LOOP_RANGE(8,)
for (int i = 0; i < param.trim_loops * 8; i++) { ... }
```

This avoids unnecessary prologue/epilogue execution and simplifies
이는 불필요한 프롤로그/에필로그 실행을 방지하고 단순화합니다.
the compiler's analysis.
컴파일러의 분석.


## 13.
## Coding for Peano
## Peano를 위한 코딩

### Always use AIE_* macros
### 항상 AIE_* 매크로를 사용하세요
Never use raw `#pragma clang loop` directives in kernel code.
커널 코드에서 원시 `#pragma clang loop` 지시문을 사용하지 마십시오.
Use the
사용
`AIE_*` macros from `utils.h` which expand correctly for Peano.
Peano에 대해 올바르게 확장되는 `utils.h`의 `AIE_*` 매크로.

### Template argument deduction
### 템플릿 인수 추론
`aie::concat` accepts a tuple of vectors, but when
`aie::concat`은 벡터의 튜플을 허용하지만,
`std::make_tuple` deduces its template arguments from
`std::make_tuple`은 템플릿 인수를 다음에서 추론합니다.
expressions like `shuffle(...)`, Clang may infer a
`shuffle(...)`과 같은 표현식을 사용하면 Clang이 다음을 추론할 수 있습니다.
different type than `aie::concat` expects (e.g., a native
`aie::concat`이 예상하는 것과 다른 유형(예: 기본
vector type instead of `aie::vector`), causing overload
`aie::벡터` 대신 벡터 유형)으로 인해 과부하가 발생함
resolution to fail.
실패 결심.
Specifying the tuple element types
튜플 요소 유형 지정
explicitly avoids this:
명시적으로 이를 방지합니다.

```cpp
// BAD: implicit template args -- may fail to compile on Peano
auto result = aie::concat(std::make_tuple(
    shuffle(v0, v1, T16_4x16_lo),
    shuffle(v0, v1, T16_4x16_hi)));

```

```cpp
// GOOD: explicit template types -- compiles cleanly on Peano
auto result = aie::concat(
    std::make_tuple<aie::vector<bfloat16, 32>,
                    aie::vector<bfloat16, 32>>(
        shuffle(v0, v1, T16_4x16_lo),
        shuffle(v0, v1, T16_4x16_hi)));
```

### DM bank syntax
### DM 은행 구문
Always use `__aie_dm_resource_*` qualifiers for memory bank
메모리 뱅크에는 항상 `__aie_dm_resource_*` 한정자를 사용하세요.
annotations:
주석:

```cpp
v128mx6_unaligned __aie_dm_resource_bd *pW = ...;
v16accfloat __aie_dm_resource_b *pB = ...;
```


## 14.
## Reading Optimization Hints (Remarks and Warnings)
## 최적화 힌트 읽기(설명 및 경고)

Optimization remarks and warnings are the primary tool for evaluating
최적화 설명 및 경고는 평가를 위한 기본 도구입니다.
how well the compiler pipelined a loop and for diagnosing performance
컴파일러가 루프를 얼마나 잘 파이프라인하고 성능을 진단하는지
issues.
문제.
They report the achieved II, stage count, prologue/epilogue
달성한 II, 스테이지 수, 프롤로그/에필로그를 보고합니다.
cost, and whether a loop became a zero-overhead loop, and they flag
비용, 루프가 오버헤드 없는 루프가 되었는지 여부, 그리고 플래그를 지정합니다.
missed opportunities the compiler could not act on (e.g. a load
컴파일러가 조치를 취할 수 없는 기회를 놓쳤습니다(예: 로드
without a memory-bank annotation).
메모리 뱅크 주석 없이).

The AIE backend exposes this information through two channels:
AIE 백엔드는 두 가지 채널을 통해 이 정보를 노출합니다.

- **Remarks channel**: structured records emitted by AIE backend
- **비고 채널**: AIE 백엔드에서 내보낸 구조화된 레코드
passes (`postpipeliner`, `aie-hardware-loops`, `aie-asm-printer`,
통과(`postpipeliner`, `aie-hardware-loops`, `aie-asm-printer`,
`aie-multi-slot-pseudo`).
`aie-다중 슬롯-의사`).
Each remark has a kind: applied
각 댓글에는 종류가 있습니다. 적용됨
(`Passed`), missed opportunity (`Missed`), or informational
('통과'), 기회를 놓쳤습니다('놓쳤습니다') 또는 정보 제공
(`Analysis`).
(`분석`).
Remarks are off by default -- they only appear when
설명은 기본적으로 꺼져 있습니다. 다음 경우에만 표시됩니다.
a corresponding `-Rpass*` / `-pass-remarks*` flag is given, or via
해당 `-Rpass*` / `-pass-remarks*` 플래그가 제공되거나 다음을 통해
the YAML optimization record.
YAML 최적화 레코드.
- **Warnings channel**: diagnostics that always reach the build log
- **경고 채널**: 항상 빌드 로그에 도달하는 진단
without any remark flag.
코멘트 플래그 없이.
Two sources matter for kernel authors:
커널 작성자에게는 두 가지 소스가 중요합니다.
the default-on `-Wpass-failed` warnings for dropped IR-level
IR 수준 삭제에 대한 기본 '-Wpass-failed' 경고
pragmas, and the AIE backend's own `WithColor::warning` calls
pragmas 및 AIE 백엔드의 자체 `WithColor::warning` 호출
(currently the multi-slot materializer's missing-memory-bank
(현재 다중 슬롯 Materializer의 누락된 메모리 뱅크
warning).
경고).

The remaining subsections cover the existing llc / YAML workflow
나머지 하위 섹션에서는 기존 LLC/YAML 워크플로를 다룹니다.
first, then the clang-driven view, then the warnings channel, and
먼저 clang 기반 보기, 경고 채널,
finally the new `aie-multi-slot-pseudo` missing-memory-bank hint.
마지막으로 새로운 'aie-multi-slot-pseudo' 메모리 뱅크 누락 힌트입니다.

### Enabling remarks
### 비고 활성화
Pass these flags to `llc` (or via the build system's compiler flags):
다음 플래그를 `llc`에 전달합니다(또는 빌드 시스템의 컴파일러 플래그를 통해).

```bash
# Write all remarks to stdout (common for piping into FileCheck)
llc -pass-remarks-output=- \
    -pass-remarks-filter='postpipeliner|aie-hardware-loops|aie-asm-printer' \
    -mtriple=aie2p input.ll -o output.s

# Write to a YAML file for offline analysis
llc -pass-remarks-output=remarks.yaml \
    -pass-remarks-filter='postpipeliner|aie-hardware-loops|aie-asm-printer' \
    -mtriple=aie2p input.ll -o output.s
```

### Postpipeliner remarks
### 포스트파이프라인 설명
The postpipeliner emits a `!Passed` remark when it finds a valid
포스트파이프라이너는 유효한 것을 찾으면 `!Passed` 설명을 내보냅니다.
software-pipelined schedule.
소프트웨어 파이프라인 일정.
The remark contains everything needed to
설명에는 필요한 모든 내용이 포함되어 있습니다.
evaluate the quality of the schedule:
일정의 질을 평가합니다.

```yaml
--- !Passed
Pass:            postpipeliner
Name:            schedule
Function:        foo
Args:
  - String:          Schedule found
  - II:              '18'              # Initiation interval
  - NS:              '2'               # Number of pipeline stages
  - Loop:            for.body          # The loop basic block
  - Prologue:        entry             # Block where prologue was inserted
  - PrologueBundles: '18'             # VLIW bundles in the prologue
  - Epilogue:        ''                # Block where epilogue was inserted
  - EpilogueBundles: '9'              # VLIW bundles in the epilogue
...
```

**Fields explained:**
**필드 설명:**
- **II**: Cycles between starting consecutive iterations.
- **II**: 연속 반복 시작 사이를 순환합니다.
- Lower = better.
- 더 낮음 = 더 좋습니다.
This is the most important number for throughput.
이는 처리량에 있어 가장 중요한 수치입니다.
- **NS**: Number of overlapping iterations in flight.
- **NS**: 비행 중 중복되는 반복 횟수입니다.
- More stages =
- 추가 단계 =
more overlap, but also more prologue/epilogue overhead and higher
겹치는 부분이 많지만 프롤로그/에필로그 오버헤드도 더 많고 더 높습니다.
register pressure.
압력을 등록합니다.
- **Loop**: The basic block that contains the pipelined kernel.
- **루프**: 파이프라인 커널을 포함하는 기본 블록입니다.
- **Prologue / PrologueBundles**: Where the prologue (pipeline ramp-up)
- **Prologue / PrologueBundles**: 프롤로그(파이프라인 확장)가 있는 위치
was placed and how many VLIW bundles it costs.
배치되었으며 VLIW 번들의 비용은 얼마입니까?
This is one-time
이는 일회성입니다.
setup overhead before the loop enters steady state.
루프가 정상 상태에 들어가기 전에 오버헤드를 설정합니다.
- **Epilogue / EpilogueBundles**: Where the epilogue (pipeline drain)
- **에필로그 / 에필로그번들**: 에필로그(파이프라인 드레인)가 있는 위치
was placed and how many VLIW bundles it costs.
배치되었으며 VLIW 번들의 비용은 얼마입니까?

**If pipelining fails**, no `!Passed` remark is emitted for that loop.
**파이프라인이 실패하면** 해당 루프에 대해 '!Passed' 설명이 내보내지지 않습니다.
The absence of a remark indicates the pipeliner could not find a
설명이 없으면 파이프라인이
schedule.
일정.
Common causes:
일반적인 원인:
- Minimum trip count too low for the stage count (add `AIE_LOOP_RANGE`)
- 단계 수에 비해 최소 이동 횟수가 너무 적습니다(`AIE_LOOP_RANGE` 추가).
- Resource conflicts that prevent any valid II
- 유효한 II를 방해하는 리소스 충돌
- Register pressure too high (loop body too complex)
- 등록 압력이 너무 높음(루프 본체가 너무 복잡함)

### Evaluating pipeline quality
### 파이프라인 품질 평가
Use the remark fields to compute the total cycle cost of the loop:
루프의 총 주기 비용을 계산하려면 설명 필드를 사용하십시오.

```
total_cycles = PrologueBundles + (trip_count * II) + EpilogueBundles
```

For high trip counts, the prologue and epilogue are amortized and II
여행 횟수가 많을 경우 프롤로그와 에필로그가 상각되고 II
dominates.
지배한다.
For small trip counts, the overhead matters more.
이동 횟수가 적은 경우 오버헤드가 더 중요합니다.

**Is my II good?** Compare the achieved II against the Minimum
**내 II가 좋은가요?** 달성한 II와 최소값을 비교하세요.
Initiation Interval (MII):
개시 간격(MII):
- **Resource MII**: Count the bottleneck resource.
- **리소스 MII**: 병목 현상이 발생한 리소스를 계산합니다.
- For example, if the
- 예를 들어,
loop body needs 8 loads and the architecture has 2 load ports,
루프 본체에는 8개의 로드가 필요하고 아키텍처에는 2개의 로드 포트가 있습니다.
Resource MII >= 4.
리소스 MII >= 4.
- **Recurrence MII**: Determined by feedback chains (e.g., accumulator
- **반복 MII**: 피드백 체인(예: 누산기)에 의해 결정됩니다.
dependency loops).
종속성 루프).
- **MII** = max(Resource MII, Recurrence MII).
- **MII** = 최대(리소스 MII, 반복 MII).
- An achieved II equal to MII is optimal.
- MII와 동일한 달성된 II가 최적입니다.
- II = MII + 1 or +2 is
- II = MII + 1 또는 +2는
typical.
전형적인.
II >> MII suggests a scheduling bottleneck worth
II >> MII는 일정 병목 현상이 발생할 가치가 있음을 제안합니다.
investigating.
조사 중입니다.

**Is the prologue/epilogue cost acceptable?** Large prologue/epilogue
**프롤로그/에필로그 비용이 허용되나요?** 큰 프롤로그/에필로그
bundles indicate many pipeline stages.
번들은 많은 파이프라인 단계를 나타냅니다.
If the trip count is small, the
운행 횟수가 적은 경우,
overhead may negate the pipelining benefit.
오버헤드가 파이프라인 이점을 무효화할 수 있습니다.
Consider:
고려 사항:
- Using `VERSIONED_LOOP` to avoid pipelining for low trip counts
- 낮은 이동 횟수에 대한 파이프라인을 피하기 위해 `VERSIONED_LOOP` 사용
- Reducing loop body complexity (fewer instructions per iteration)
- 루프 본문 복잡성 감소(반복당 명령 수 감소)
- Checking if `AIE_LOOP_RANGE` accurately reflects the minimum trip
- `AIE_LOOP_RANGE`가 최소 이동 거리를 정확하게 반영하는지 확인
count
카운트

### Hardware loop remarks
### 하드웨어 루프 설명
The `aie-hardware-loops` pass reports whether each loop was converted
'aie-hardware-loops' 단계는 각 루프가 변환되었는지 여부를 보고합니다.
to a zero-overhead loop (ZOL):
제로 오버헤드 루프(ZOL)로:

```yaml
--- !Analysis
Pass:            aie-hardware-loops
Name:            analysis
Function:        foo
Args:
  - LoopID:          '0'
  - BasicBlock:      for.body
  - Zero-Overhead-Loop: 'true'
```

ZOL conversion eliminates the branch instruction at the end of the
ZOL 변환은 마지막에 분기 명령을 제거합니다.
loop body, freeing a VLIW slot and removing the branch latency.
루프 본문을 통해 VLIW 슬롯을 해제하고 분기 대기 시간을 제거합니다.
This
이
is both a PM and a performance improvement.
PM이자 성능 개선입니다.

The compiler generates a hardware loop when the minimum iteration
컴파일러는 최소 반복이 발생하면 하드웨어 루프를 생성합니다.
count is known (via `AIE_LOOP_RANGE`).
개수는 `AIE_LOOP_RANGE`를 통해 알려져 있습니다.
When the minimum is known and
최소값을 알고 있는 경우
greater than 1, the compiler can also remove the loop guard (the
1보다 크면 컴파일러는 루프 가드를 제거할 수도 있습니다(
branch that skips the loop when the trip count is zero), saving
트립 횟수가 0일 때 루프를 건너뛰는 분기), 저장
additional PM and cycles.
추가 PM 및 주기.

**Current limitation**: the loop guard is only removed when the
**현재 제한**: 루프 가드는 다음과 같은 경우에만 제거됩니다.
minimum iteration count is strictly greater than 1.
최소 반복 횟수는 1보다 엄격하게 큽니다.
Setting
설정
`AIE_LOOP_RANGE(2,)` or higher enables this optimization.
`AIE_LOOP_RANGE(2,)` 이상에서는 이 최적화가 가능합니다.

If `Zero-Overhead-Loop` is `false` or the remark is absent, the loop
'Zero-Overhead-Loop'이 'false'이거나 설명이 없으면 루프는
uses a software branch.
소프트웨어 분기를 사용합니다.
Common reasons ZOL conversion fails: the loop
ZOL 변환이 실패하는 일반적인 이유: 루프
has multiple exits, or the trip count cannot be determined.
출구가 여러 개이거나 운행 횟수를 확인할 수 없습니다.

### Assembly-level remarks
### 어셈블리 수준 설명
The `aie-asm-printer` pass reports the code size of each basic block:
`aie-asm-printer` 패스는 각 기본 블록의 코드 크기를 보고합니다.

```yaml
--- !Analysis
Pass:            aie-asm-printer
Name:            analysis
Function:        foo
Args:
  - BasicBlock:      for.body
  - BundleCount:     '4'
  - ByteCount:       '64'
```

- **BundleCount**: Number of VLIW bundles (one bundle = one cycle).
- **BundleCount**: VLIW 번들의 수(1개의 번들 = 1사이클).
For a pipelined loop, the BundleCount of the loop body equals II.
파이프라인 루프의 경우 루프 본문의 BundleCount는 II와 같습니다.
- **ByteCount**: Total instruction bytes.
- **ByteCount**: 총 명령어 바이트입니다.
- Useful for tracking program
- 추적 프로그램에 유용
memory (PM) usage.
메모리(PM) 사용량.

### Combining remarks for full picture
### 전체 그림에 대한 설명 결합
For a complete performance picture, enable all three remark passes and
완전한 성능 그림을 얻으려면 세 가지 설명 패스를 모두 활성화하고
read them together:
함께 읽어보세요:

1. **aie-hardware-loops**: Is it a ZOL? (eliminates branch overhead)
1. **aie-hardware-loops**: ZOL인가요? (브랜치 오버헤드 제거)
2. **postpipeliner**: What II / NS was achieved?
2. **포스트파이프라인**: 어떤 II/NS가 달성되었나요?
2. What is the
2. 무엇입니까?
prologue/epilogue cost?
프롤로그/에필로그 비용?
3. **aie-asm-printer**: What is the BundleCount of each block?
3. **aie-asm-printer**: 각 블록의 BundleCount는 얼마입니까?
(confirms II and shows non-loop overhead)
(II를 확인하고 비루프 오버헤드를 보여줌)

Example workflow:
예시 워크플로우:
```bash
llc -pass-remarks-output=- \
    -pass-remarks-filter='postpipeliner|aie-hardware-loops|aie-asm-printer' \
    -mtriple=aie2p my_kernel.ll -o my_kernel.s
```
Then grep for `!Passed` to find pipelined loops and check their II.
그런 다음 ``!Passed'를 찾아 파이프라인 루프를 찾고 II를 확인하세요.

### aiesim cycle-accurate verification
### aiesim 사이클의 정확한 검증
After optimizing, verify with cycle-accurate simulation:
최적화 후 정확한 주기 시뮬레이션을 통해 검증합니다.
```bash
make run XPART=<device> TARGET=aiesim
```
Check the cycle count report and compare with the theoretical optimum:
사이클 카운트 보고서를 확인하고 이론적 최적과 비교하십시오.
```
optimal_cycles = PrologueBundles + (trip_count * II) + EpilogueBundles
```
If the measured cycle count significantly exceeds this, look for
측정된 사이클 수가 이 값을 크게 초과하는 경우 다음을 찾으십시오.
stalls caused by memory bank conflicts, lock contention, or DMA
메모리 뱅크 충돌, 잠금 경합 또는 DMA로 인한 정지
latency that the remarks don't capture.
발언이 포착하지 못하는 대기 시간.


### Backend remark passes and how to view them
### 백엔드 설명 패스 및 이를 보는 방법

The AIE backend currently emits remarks from four passes.
AIE 백엔드는 현재 4개의 패스에서 설명을 내보냅니다.
Each remark
각 발언
has a kind -- `Passed` (transformation applied), `Missed` (missed
종류가 있습니다 -- `Passed`(변환 적용됨), `Missed`(누락됨)
opportunity), or `Analysis` (informational) -- and is surfaced by a
기회) 또는 '분석'(정보 제공) -- 그리고
matching flag: `-Rpass=`, `-Rpass-missed=`, or `-Rpass-analysis=` for
일치 플래그: `-Rpass=`, `-Rpass-missed=` 또는 `-Rpass-analytic=`
a clang kernel build, with the equivalent `-pass-remarks=`,
동등한 `-pass-remarks=`를 사용하는 clang 커널 빌드,
`-pass-remarks-missed=`, `-pass-remarks-analysis=` for `llc`.
`llc`의 경우 `-pass-remarks-missed=`, `-pass-remarks-analytic=`.
The
는
earlier subsections show the llc / YAML workflow; the same remarks
이전 하위 섹션에서는 llc/YAML 워크플로를 보여줍니다. 같은 말
are reachable directly from a clang-driven kernel build using the
다음을 사용하여 clang 기반 커널 빌드에서 직접 접근할 수 있습니다.
flags in the last column below.
아래 마지막 열의 플래그입니다.

| Pass<br>패스                    | Name<br>이름                  | Kind<br>종류     | Reports<br>보고서                                                                | Flag to enable<br>활성화할 플래그                            |
|-------------------------|-----------------------|----------|------------------------------------------------------------------------|-------------------------------------------|
| `postpipeliner`<br>'포스트파이프라이너'         | `schedule`<br>'일정'            | Passed<br>합격   | II, NS, prologue/epilogue bundles when a schedule is found<br>II, NS, 일정 발견 시 프롤로그/에필로그 번들             | `-Rpass=postpipeliner`<br>`-Rpass=포스트파이프라이너`                    |
| `postpipeliner`<br>'포스트파이프라이너'         | `schedule`<br>'일정'            | Missed<br>놓친   | Reason pipelining failed (e.g. `Longest circuit does not fit II`)<br>파이프라인이 실패한 이유(예: '가장 긴 회로가 II에 맞지 않음')      | `-Rpass-missed=postpipeliner`<br>`-Rpass-missed=postpipeliner`             |
| `aie-hardware-loops`<br>`aie-하드웨어 루프`    | `analysis`<br>'분석'            | Analysis<br>분석 | Per-loop ZOL conversion result (`Zero-Overhead-Loop: true/false`)<br>루프별 ZOL 변환 결과(`Zero-Overhead-Loop: true/false`)      | `-Rpass-analysis=aie-hardware-loops`<br>`-Rpass-분석=aie-하드웨어-루프`      |
| `aie-asm-printer`<br>`aie-asm-프린터`       | `analysis`<br>'분석'            | Analysis<br>분석 | Per-block `BundleCount` and `ByteCount`<br>블록별 `BundleCount` 및 `ByteCount`                                | `-Rpass-analysis=aie-asm-printer`<br>`-Rpass-분석=aie-asm-프린터`         |
| `aie-multi-slot-pseudo`<br>`aie-다중 슬롯-의사` | `missing-memory-bank`<br>'잃어버린 메모리 뱅크' | Missed<br>놓친   | Load reached the multi-slot materializer without a DM bank annotation<br>DM 뱅크 주석 없이 로드가 다중 슬롯 Materializer에 도달했습니다.  | `-Rpass-missed=aie-multi-slot-pseudo`<br>`-Rpass-missed=aie-다중 슬롯-의사`     |

`-Rpass-missed` is the only way (besides the YAML record) to see
`-Rpass-missed`는 YAML 레코드 외에 볼 수 있는 유일한 방법입니다.
`postpipeliner`'s "no schedule found" / "longest circuit does not
`postpipeliner`의 "일정을 찾을 수 없습니다" / "가장 긴 회로가 없습니다"
fit II" remarks -- they are not emitted as warnings.
fit II" 설명 - 경고로 표시되지 않습니다.

For the YAML record (recommended for CI / postmortem analysis),
YAML 레코드의 경우(CI/사후 분석에 권장)
clang exposes `-fsave-optimization-record`,
clang은 `-fsave-optimization-record`를 노출합니다.
`-foptimization-record-file=<path>`, and
`-foptimization-record-file=<경로>` 및
`-foptimization-record-passes=<regex>`; these write the same YAML
`-foptimization-record-passes=<regex>`; 이들은 동일한 YAML을 작성합니다.
format as `llc -pass-remarks-output=` documented earlier, and the
이전에 문서화된 `llc -pass-remarks-output=` 형식으로, 그리고
`-passes=` flag corresponds to `llc -pass-remarks-filter=`.
`-passes=` 플래그는 `llc -pass-remarks-filter=`에 해당합니다.

Recommended kernel-dev defaults:
권장되는 커널 개발 기본값:

```bash
clang ... \
  -Rpass=postpipeliner \
  -Rpass-missed='postpipeliner|aie-multi-slot-pseudo' \
  -Rpass-analysis='aie-hardware-loops|aie-asm-printer'
```

### Warnings channel
### 경고 채널

Warnings reach the build log without any remark flag.
경고는 설명 플래그 없이 빌드 로그에 도달합니다.

- **`-Wpass-failed`** (on by default): emitted by the IR-level
- **`-Wpass-failed`** (기본적으로 켜져 있음): IR 수준에서 방출됩니다.
optimizer when an explicitly requested transformation cannot be
명시적으로 요청된 변환을 수행할 수 없는 경우 최적화 프로그램
applied.
적용되었습니다.
This is what fires when a `#pragma clang loop` directive
이것은 `#pragma clang loop` 지시문이 실행될 때 실행되는 것입니다.
(or an `AIE_*` macro that expands to one) is dropped, e.g.
(또는 하나로 확장되는 `AIE_*` 매크로)가 삭제됩니다.
`loop not vectorized: the optimizer was unable to perform the
`루프가 벡터화되지 않았습니다: 최적화 프로그램이 다음을 수행할 수 없습니다.
requested transformation; the transformation might be disabled or
변형을 요청했습니다. 변환이 비활성화되거나
specified as part of an unsupported transformation ordering`.
지원되지 않는 변환 순서'의 일부로 지정되었습니다.
Treat any `-Wpass-failed` warning as evidence that a pragma was
'-Wpass-failed' 경고를 pragma가 실패했다는 증거로 처리합니다.
silently ignored.
조용히 무시했습니다.
- **AIE backend warnings via `WithColor::warning`**: the AIE backend
- **`WithColor::warning`을 통한 AIE 백엔드 경고**: AIE 백엔드
also writes warnings directly to stderr, independent of `-Rpass*`
또한 `-Rpass*`와 관계없이 stderr에 직접 경고를 기록합니다.
and of `-Wpass-failed`.
그리고 `-Wpass-failed`.
The first such warning in tree is the
트리에서 첫 번째 경고는 다음과 같습니다.
multi-slot materializer's missing-memory-bank warning, covered in
다중 슬롯 Materializer의 메모리 뱅크 누락 경고
the next subsection.
다음 하위 섹션.

Note: `postpipeliner` missed schedules are **only** on the remarks
참고: 'postpipeliner' 놓친 일정은 비고에만 **만** 있습니다.
channel -- they do not warn by default.
채널 - 기본적으로 경고하지 않습니다.
Use
사용
`-Rpass-missed=postpipeliner` (or the YAML record) to see them.
`-Rpass-missed=postpipeliner`(또는 YAML 레코드)를 사용하여 확인하세요.

### Missing memory bank annotations (`aie-multi-slot-pseudo`)
### 메모리 뱅크 주석 누락(`aie-multi-slot-pseudo`)

**What slot materialization does.** A multi-slot pseudo is a load
**슬롯 구체화의 역할.** 다중 슬롯 의사는 로드입니다.
opcode that has not yet been bound to a specific issue slot.
아직 특정 이슈 슬롯에 바인딩되지 않은 opcode입니다.
The
는
`aie-multi-slot-pseudo` pass picks a concrete slot-bound opcode for
`aie-multi-slot-pseudo` 패스는 구체적인 슬롯 바인딩 opcode를 선택합니다.
each such load before the post-RA pipeliner runs.
RA 이후 파이프라인이 실행되기 전에 이러한 각 로드가 실행됩니다.
Picking the right
올바른 선택
slot matters because two loads in the same VLIW bundle must use
동일한 VLIW 번들의 두 로드는 슬롯을 사용해야 하기 때문에 슬롯이 중요합니다.
different load slots, and each load slot has affinity to specific
서로 다른 로드 슬롯이 있으며 각 로드 슬롯은 특정 항목에 대한 선호도를 갖습니다.
memory banks.
메모리 뱅크.
The materializer uses the load's DM bank annotation
Materializer는 로드의 DM 뱅크 주석을 사용합니다.
to map it to a slot whose ports match that bank, so that consecutive
포트가 해당 뱅크와 일치하는 슬롯에 매핑하여 연속적으로
loads on different banks can issue in parallel.
서로 다른 뱅크의 로드가 동시에 발생할 수 있습니다.

**What this hint means.** It fires when the materializer encounters
**이 힌트의 의미.** 구체화자가 다음을 만날 때 실행됩니다.
a load whose pointer carries no DM bank annotation (see Section 3
포인터가 DM 뱅크 주석을 전달하지 않는 로드(섹션 3 참조)
for the qualifier list).
예선 목록의 경우).
It is a **missed-opportunity** hint, not a
그것은 **놓친 기회** 힌트이지,
functional failure: the materializer still picks a slot-bound opcode
기능적 실패: 구체화자는 여전히 슬롯 바인딩된 opcode를 선택합니다.
for the load, but it has to fall back to a bank-blind heuristic.
하지만 은행맹인 휴리스틱으로 돌아가야 합니다.
The chosen slot is often suboptimal
선택한 슬롯은 종종 차선책입니다.
-- the load lands in the wrong slot and blocks the load ordering
-- 로드가 잘못된 슬롯에 배치되어 로드 순서가 차단됩니다.
the pipeliner would otherwise use, raising the achieved II.
그렇지 않으면 파이프라인은 이를 사용하여 달성된 II를 높입니다.

What the kernel author sees:
커널 작성자가 보는 것:

- Always, on stderr (no flag required):
- 항상 stderr에서(플래그가 필요하지 않음):
  ```
  warning: No memory bank assigned to load in function '<fn>',
  block '<bb>' at <file>:<line>:<col>: <MI dump>
  ```
- With `-Rpass-missed=aie-multi-slot-pseudo` (or in the YAML record),
- `-Rpass-missed=aie-multi-slot-pseudo`(또는 YAML 레코드에서)를 사용하면
a `!Missed` remark anchored to the same debug location:
동일한 디버그 위치에 고정된 `!Missed` 설명:
  ```yaml
  --- !Missed
  Pass:            aie-multi-slot-pseudo
  Name:            missing-memory-bank
  Function:        <fn>
  Args:
    - String: "No memory bank assigned to load in function '<fn>', block '<bb>' at <loc>"
  ```

How to fix: annotate the offending pointer with a DM bank qualifier
해결 방법: 문제가 있는 포인터에 DM 뱅크 한정자로 주석을 답니다.
(`__aie_dm_resource_a` / `_b` / `_c` / `_d`) -- see Section 3 for
(`__aie_dm_resource_a` / `_b` / `_c` / `_d`) -- 자세한 내용은 섹션 3을 참조하세요.
the qualifier list and parallel-access rules.
한정자 목록 및 병렬 액세스 규칙.
If the load comes from
부하가 다음에서 오는 경우
a helper or cast, check that the bank qualifier is preserved through
도우미 또는 캐스트를 통해 은행 한정자가 보존되는지 확인하십시오.
every intermediate pointer and is not stripped by a cast (the same
모든 중간 포인터는 캐스트로 제거되지 않습니다(동일
pitfall as Section 2 "Casting away restrict" applies to bank
섹션 2 "캐스팅 제한"이 은행에 적용되므로 함정
qualifiers).
예선).

Ignoring the hint will not break the kernel, but the suboptimal slot
힌트를 무시해도 커널은 손상되지 않지만 최적이 아닌 슬롯
choice typically shows up later as a higher achieved II in the
선택은 일반적으로 나중에 더 높은 성취 II로 나타납니다.
`postpipeliner` remark, or as a `BundleCount` that exceeds the
`postpipeliner` 설명 또는
resource MII reported by `aie-asm-printer`.
`aie-asm-printer`에 의해 보고된 리소스 MII입니다.


## 15.
## Optimization Checklist
## 최적화 체크리스트

Use this checklist when writing or reviewing kernel code:
커널 코드를 작성하거나 검토할 때 이 체크리스트를 사용하십시오:

1. All pointer parameters use `__restrict` where they don't alias
1. 모든 포인터 매개변수는 별칭이 아닌 `__restrict`를 사용합니다.
2. No pointer casts strip `__restrict` or DM bank qualifiers
2. 포인터가 스트립 `__restrict` 또는 DM 뱅크 한정자를 캐스팅하지 않습니다.
3. Core loop function is `NOINLINE`; public wrapper is `INLINE`
3. 핵심 루프 기능은 'NOINLINE'입니다. 공개 래퍼는 'INLINE'입니다.
4. Innermost loop has `AIE_LOOP_RANGE` with accurate minimum trip
4. 가장 안쪽 루프에는 정확한 최소 이동 거리를 갖는 'AIE_LOOP_RANGE'가 있습니다.
count
카운트
5. `AIE_PREPARE_FOR_POSTPIPELINING` is present on pipelined inner
5. 파이프라인 내부에 `AIE_PREPARE_FOR_POSTPIPELINING`이 있습니다.
loops
루프
6. DM bank annotations match the data flow (inputs and outputs on
6. DM 뱅크 주석은 데이터 흐름(입력 및 출력)과 일치합니다.
different banks for parallel access)
병렬 액세스를 위한 다른 은행)
7. Variable trip count loops use `VERSIONED_LOOP` or manual
7. 가변 트립 카운트 루프는 `VERSIONED_LOOP` 또는 수동을 사용합니다.
versioning
버전 관리
8. Post-increment addressing (`*ptr++`) preferred over indexed
8. 색인화보다 후증가 주소 지정(`*ptr++`)이 선호됩니다.
access (`ptr[i]`) in hot loops
핫 루프의 액세스(`ptr[i]`)
9. Accumulator structs use and proper `alignas`
9. 누산기 구조체 사용 및 적절한 'alignas'
10. Optimization hints checked: achieved II compared with target and
10. 최적화 힌트 확인: 목표와 비교하여 II 달성
ZOL conversion confirmed for each pipelined inner loop; scan
각 파이프라인 내부 루프에 대해 ZOL 변환이 확인되었습니다. 스캔
the build log for `-Wpass-failed` warnings and any
`-Wpass-failed` 경고에 대한 빌드 로그 및 기타
`aie-multi-slot-pseudo` missing-memory-bank warnings, and review
'aie-multi-slot-pseudo' 메모리 뱅크 누락 경고 및 검토
`-Rpass-missed=postpipeliner` output for missed schedules
누락된 일정에 대한 `-Rpass-missed=postpipeliner` 출력
11. Parameter structs passed by `__restrict` reference
11. `__restrict` 참조에 의해 전달된 매개변수 구조체
12. `AIE_*` macros used instead of raw `#pragma clang loop`
12. 원시 `#pragma clang loop` 대신 `AIE_*` 매크로가 사용되었습니다.
13. `std::make_tuple` uses explicit template types where type
13. `std::make_tuple`은 명시적인 템플릿 유형을 사용합니다.
deduction is ambiguous
공제가 모호하다
14. `AIE_LOOP_HINT` used for per-loop compiler tuning when global
14. 글로벌일 때 루프별 컴파일러 튜닝에 사용되는 `AIE_LOOP_HINT`
defaults are suboptimal
기본값은 차선책입니다
