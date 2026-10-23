<!-- ko-translation: enabled -->
<!--- Copyright (C) 2026 Advanced Micro Devices, Inc. All rights reserved.--->
# AIE Kernel Programming
# AIE 커널 프로그래밍

This guide provides an overview on writing AIE kernel code for [custom operations in VAIML](README.md).
이 가이드는 [VAIML의 사용자 정의 작업](README.md)을 위한 AIE 커널 코드 작성에 대한 개요를 제공합니다.
The kernel is the core computational component that executes on AIE hardware and defines the actual operation logic.
커널은 AIE 하드웨어에서 실행되고 실제 작동 논리를 정의하는 핵심 계산 구성 요소입니다.

## Table of Contents
## 목차

- [Overview](#overview)
- [개요](#overview)
- [Kernel Fundamentals](#kernel-fundamentals)
- [커널 기초](#kernel-fundamentals)
- [Basic Kernel Examples](#basic-kernel-examples)
- [기본 커널 예](#basic-kernel-examples)
- [Performance Optimization](#performance-optimization)
- [성능 최적화](#performance-optimization)
- [Advanced Techniques](#advanced-techniques)
- [고급 기술](#advanced-techniques)
- [Reference Links](#reference-links)
- [참조 링크](#reference-links)

## Overview
## 개요

The AIE kernel is a C++ function that executes on AIE cores and performs the actual computation for your custom operation.
AIE 커널은 AIE 코어에서 실행되고 사용자 지정 작업에 대한 실제 계산을 수행하는 C++ 함수입니다.
The kernel code runs on every AIE core of the overlay in parallel.
커널 코드는 오버레이의 모든 AIE 코어에서 병렬로 실행됩니다.
Each AIE core has its own
각 AIE 코어에는 고유한
memory space containing the L1 buffers, heap and stack.
L1 버퍼, 힙 및 스택을 포함하는 메모리 공간입니다.
The kernel function is called whenever all synchronous buffers are ready (i.e. input buffers
커널 함수는 모든 동기 버퍼가 준비될 때마다 호출됩니다(예: 입력 버퍼
have received their data and output buffers have free space available).
데이터를 수신했으며 출력 버퍼에 사용 가능한 여유 공간이 있습니다).

## Kernel Fundamentals
## 커널 기초

### Namespace
### 네임스페이스

The kernel must be implemented within the `custom_ops` namespace.
커널은 `custom_ops` 네임스페이스 내에서 구현되어야 합니다.
### Basic Kernel Structure
### 기본 커널 구조

Every AIE kernel follows a standard pattern:
모든 AIE 커널은 표준 패턴을 따릅니다.

```cpp
#include <adf.h>

namespace custom_ops {

// Define the size of layer parameters array
static constexpr int my_kernel_lp_size = 1;

template <typename dtype_input, typename dtype_output>
__attribute__((noinline)) void my_kernel(
    adf::input_buffer_conf<dtype_input, adf::bpc_sync_0d> &__restrict input,
    adf::output_buffer_conf<dtype_output, adf::bpc_sync_0d> &__restrict output,
    const uint32_t (&lp_params)[my_kernel_lp_size]
) {
    // Kernel implementation
}

} // namespace custom_ops
```

### Kernel Signature Components
### 커널 서명 구성 요소

#### Template Parameters
#### 템플릿 매개변수
The template arguments correspond exactly to the data types of each buffer argument in order, ignoring the `lp_params` parameter.
템플릿 인수는 'lp_params' 매개변수를 무시하고 순서대로 각 버퍼 인수의 데이터 유형과 정확히 일치합니다.

For a kernel with two inputs and one output, the signature is:
두 개의 입력과 하나의 출력이 있는 커널의 경우 서명은 다음과 같습니다.
```cpp
template <typename dtype_a, typename dtype_b, typename dtype_c>
void my_kernel(
    adf::input_buffer_conf<dtype_a, adf::bpc_sync_0d> &__restrict input_a,
    adf::input_buffer_conf<dtype_b, adf::bpc_sync_0d> &__restrict input_b,
    adf::output_buffer_conf<dtype_c, bpc_sync_0d> &__restrict output_c,
    const uint32_t (&lp_params)[lp_size],
);
```

The template parameters are:
템플릿 매개변수는 다음과 같습니다.
- `dtype_a`: Data type of first buffer (`input_a`)
- `dtype_a`: 첫 번째 버퍼의 데이터 유형(`input_a`)
- `dtype_b`: Data type of second buffer (`input_b`)
- `dtype_b`: 두 번째 버퍼의 데이터 유형(`input_b`)
- `dtype_c`: Data type of third buffer (`output_c`)
- `dtype_c`: 세 번째 버퍼의 데이터 유형(`output_c`)

#### Function Parameters
#### 기능 매개변수
1. **Input Buffers**: ADF buffer configurations for input data
1. **입력 버퍼**: 입력 데이터를 위한 ADF 버퍼 구성
2. **Layer Parameters**: Array of runtime parameters from tiler (does not affect template signature)
2. **레이어 매개변수**: 타일러의 런타임 매개변수 배열(템플릿 서명에 영향을 주지 않음)
3. **Output Buffers**: ADF buffer configurations for output data
3. **출력 버퍼**: 출력 데이터를 위한 ADF 버퍼 구성

#### Buffer Synchronization Modes
#### 버퍼 동기화 모드
- `adf::bpc_sync_0d`: Synchronous access (use this for most cases)
- `adf::bpc_sync_0d`: 동기 액세스(대부분의 경우 이것을 사용)
- `adf::bpc_async_0d`: Asynchronous access
- `adf::bpc_async_0d`: 비동기 액세스

More details on alternative options for synchronization besides `bpc_sync_0d` are [here](https://docs.amd.com/r/en-US/ug1079-ai-engine-kernel-coding/Asynchronous-Buffer-Port-Access).
'bpc_sync_0d' 외에 동기화를 위한 대체 옵션에 대한 자세한 내용은 [여기](https://docs.amd.com/r/en-US/ug1079-ai-engine-kernel-coding/Asynchronous-Buffer-Port-Access)에서 확인하세요.

Refer to the [tiling section on asynchronous buffers](tiling.md#keeping-buffers-across-kernel-calls-asynchronous-buffers) for information on how to use them.
사용 방법에 대한 자세한 내용은 [비동기 버퍼의 타일링 섹션](tiling.md#keeping-buffers-across-kernel-calls-asynchronous-buffers)을 참조하세요.

### Layer Parameters (LPs)
### 레이어 매개변수(LP)

Layer parameters are how the tiler passes runtime configuration to your kernel.
레이어 매개변수는 타일러가 런타임 구성을 커널에 전달하는 방법입니다.
Think of them as function arguments that tell your kernel how much data to process and how it's organized.
처리할 데이터의 양과 구성 방법을 커널에 알려주는 함수 인수로 생각하십시오.

```python
# In tiler (Python) - this is handled by the tiling component
lp_params = [total_elements, tile_height, tile_width]
tiling.set_kernel(lp_params)
```

```cpp
// In kernel (C++) - your kernel receives these values
static constexpr int my_kernel_lp_size = 3;  // Must match number of parameters
// ...
int total_elements = (int)lp_params[0];  // How many elements to process
int tile_height = (int)lp_params[1];     // Height of data tile
int tile_width = (int)lp_params[2];      // Width of data tile
```

**Important**: The `lp_size` constant must exactly match the number of parameters your tiler sends.
**중요**: `lp_size` 상수는 타일러가 전송하는 매개변수 수와 정확히 일치해야 합니다.

## Basic Kernel Examples
## 기본 커널 예

### Example 1: Simple Unary Operation (Negate)
### 예제 1: 단순 단항 연산(부정)

This example shows a basic kernel that negates each input element.
이 예에서는 각 입력 요소를 무효화하는 기본 커널을 보여줍니다.
This is a good starting point for understanding kernel structure.
이는 커널 구조를 이해하기 위한 좋은 출발점입니다.

```cpp
#include <adf.h>

namespace custom_ops {

// Define how many layer parameters this kernel expects
static constexpr int negate_kernel_lp_size = 1;

template <typename dtype_ifm, typename dtype_ofm>
__attribute__((noinline)) void negate_kernel(
    adf::input_buffer_conf<dtype_ifm, adf::bpc_sync_0d> &__restrict ifm,
    adf::output_buffer_conf<dtype_ofm, adf::bpc_sync_0d> &__restrict ofm,
    const uint32_t (&lp_params)[negate_kernel_lp_size]
) {
    // Get total number of elements from tiler
    int trip_count = (int)lp_params[0];

    // Get raw data pointers from ADF buffers
    dtype_ifm *in = ifm.data();
    dtype_ofm *out = ofm.data();

    // Process elements one by one (scalar approach)
    for (int i = 0; i < trip_count; i++) {
        *out = -(*in);  // Negate the input value
        in++;           // Move to next input element
        out++;          // Move to next output element
    }
}

} // namespace custom_ops
```

**Key points for beginners**:
**초보자를 위한 핵심 포인트**:
- Template parameters `dtype_ifm` and `dtype_ofm` will be the actual data types (e.g., `bfloat16`)
- 템플릿 매개변수 `dtype_ifm` 및 `dtype_ofm`은 실제 데이터 유형이 됩니다(예: `bfloat16`)
- `trip_count` tells you how many elements to process
- `trip_count`는 처리할 요소 수를 알려줍니다.
- Use `.data()` to get raw pointers from ADF buffers
- ADF 버퍼에서 원시 포인터를 얻으려면 `.data()`를 사용하세요.
- This is a scalar implementation - we'll show vectorized versions later
- 이것은 스칼라 구현입니다. 나중에 벡터화된 버전을 보여드리겠습니다.

### Example 2: Binary Operation (Element-wise Multiplication)
### 예제 2: 이진 연산(요소별 곱셈)

This example shows how to handle two input buffers for operations like element-wise multiplication.
이 예에서는 요소별 곱셈과 같은 연산을 위해 두 개의 입력 버퍼를 처리하는 방법을 보여줍니다.

```cpp
#include <adf.h>

namespace custom_ops {

static constexpr int mul_kernel_lp_size = 1;

template <typename dtype_ifm, typename dtype_wts, typename dtype_ofm>
__attribute__((noinline)) void mul_kernel(
    adf::input_buffer_conf<dtype_ifm, adf::bpc_sync_0d> &__restrict a,
    adf::input_buffer_conf<dtype_wts, adf::bpc_sync_0d> &__restrict b,
    adf::output_buffer_conf<dtype_ofm, adf::bpc_sync_0d> &__restrict out,
    const uint32_t (&lp_params)[mul_kernel_lp_size]
) {
    dtype_ifm * a_data = a.data();
    dtype_wts * b_data = b.data();
    dtype_ofm * out_data = out.data();

    uint32_t size = lp_params[0];

    // Element-wise multiplication: out[i] = a[i] * b[i]
    for (uint32_t i = 0; i < size; ++i) {
        out_data[i] = a_data[i] * b_data[i];
    }
}

} // namespace custom_ops
```

## Performance Optimization
## 성능 최적화

### Vectorization
### 벡터화

Transform scalar operations into vector operations to perform multiple operations within the same cycle.
스칼라 연산을 벡터 연산으로 변환하여 동일한 주기 내에서 여러 연산을 수행합니다.
The [AIE high level API](https://download.amd.com/docnav/aiengine/xilinx2025_1/aiengine_api/aie_api/doc/topics.html) simplifies the task of implementing vector algorithms on AIE.
[AIE 상위 수준 API](https://download.amd.com/docnav/aiengine/xilinx2025_1/aiengine_api/aie_api/doc/topics.html)는 AIE에서 벡터 알고리즘을 구현하는 작업을 단순화합니다.
In particular, several arithmetic operations have built-in vector support; they are documented on [this page](https://download.amd.com/docnav/aiengine/xilinx2025_1/aiengine_api/aie_api/doc/group__group__arithmetic.html).
특히, 여러 산술 연산에는 벡터 지원 기능이 내장되어 있습니다. [이 페이지](https://download.amd.com/docnav/aiengine/xilinx2025_1/aiengine_api/aie_api/doc/group__group__arithmetic.html)에 문서화되어 있습니다.

In the example below, we use the following steps:
아래 예에서는 다음 단계를 사용합니다.
1) Create the vector iterators `in` and `out`.
1) 벡터 반복기 `in` 및 `out`을 만듭니다.
1) `aie::begin_restrict_vector<16>(ptr)` interprets `ptr` as a pointer to an array of vectors with `16` elements each.
1) `aie::begin_restrict_Vector<16>(ptr)`는 `ptr`을 각각 `16` 요소가 있는 벡터 배열에 대한 포인터로 해석합니다.
2) Reduce the loop iteration count by a factor of 16 because each iteration now processes 16 elements.
2) 이제 각 반복이 16개의 요소를 처리하므로 루프 반복 횟수를 16배로 줄입니다.
3) Use `*in` to load a vector of 16 elements from the input buffer.
3) 입력 버퍼에서 16개 요소의 벡터를 로드하려면 '*in'을 사용하세요.
4) Use `in++` to advance the input iterator by 16 elements to the next vector.
4) 입력 반복자를 16개 요소만큼 다음 벡터로 이동하려면 'in++'를 사용하세요.
5) Use `aie::neg` to negate all 16 elements in the vector.
5) 벡터의 16개 요소를 모두 무효화하려면 `aie::neg`를 사용하세요.
6) Use `*out` to store a vector of 16 elements to the output buffer.
6) 출력 버퍼에 16개 요소의 벡터를 저장하려면 `*out`을 사용하세요.
7) Use `out++` to advance the output iterator by 16 elements to the next vector.
7) 'out++'를 사용하여 출력 반복기를 16개 요소씩 다음 벡터로 전진시킵니다.

```cpp
#include <adf.h>
#include "aie_api/aie.hpp"

namespace custom_ops {

static constexpr int negate_kernel_lp_size = 1;

template <typename dtype_ifm, typename dtype_ofm>
__attribute__((noinline)) void negate_kernel_vectorized(
    adf::input_buffer_conf<dtype_ifm, bpc_sync_0d> &__restrict ifm,
    adf::output_buffer_conf<dtype_ofm, bpc_sync_0d> &__restrict ofm,
    const uint32_t (&lp_params)[negate_kernel_lp_size]
) {
    int trip_count = (int)lp_params[0];

    // Create vector iterators. Addresses must be aligned to 32 bytes.
    auto in = aie::cbegin_restrict_vector<16>(ifm.data());
    auto out = aie::begin_restrict_vector<16>(ofm.data());

    // Process 16 elements at a time
    for (int i = 0; i < trip_count / 16; i++) {
        aie::vector<dtype_ifm, 16> vin = *in++;
        aie::vector<dtype_ofm, 16> negated = aie::neg(vin);
        *out++ = negated;
    }
}

} // namespace custom_ops
```

Always ensure that the addresses for vector loads/stores are aligned to at least the vector size.
벡터 로드/저장을 위한 주소가 최소한 벡터 크기에 맞게 정렬되어 있는지 항상 확인하십시오.
Otherwise they will silently produce wrong results.
그렇지 않으면 조용히 잘못된 결과가 나올 것입니다.
See [here](https://download.amd.com/docnav/aiengine/xilinx2025_1/aiengine_api/aie_api/doc/group__group__memory.html) for details on alignment requirements.
정렬 요구 사항에 대한 자세한 내용은 [여기](https://download.amd.com/docnav/aiengine/xilinx2025_1/aiengine_api/aie_api/doc/group__group__memory.html)를 참조하세요.
Alternatively, `aie::vector::load_unaligned` and `aie::vector::store_unaligned` allow to load/store from unaligned addresses
또는 `aie::벡터::load_unaligned` 및 `aie::벡터::store_unaligned`를 사용하면 정렬되지 않은 주소에서 로드/저장할 수 있습니다.
but with a performance penalty.
하지만 성능 저하가 있습니다.

### Loop Pipelining
### 루프 파이프라이닝

Use compiler hints to optimize loop performance.
루프 성능을 최적화하려면 컴파일러 힌트를 사용하십시오.
`AIE_PIPELINE` enables hardware
`AIE_PIPELINE`을 사용하면 하드웨어를 활성화할 수 있습니다.
loop pipelining, making use of the AIE zero-overhead hardware loops.
AIE 제로 오버헤드 하드웨어 루프를 활용하는 루프 파이프라인.
`AIE_LOOP_MIN_ITERATION_COUNT(x)` is used to promise that the loop will always
`AIE_LOOP_MIN_ITERATION_COUNT(x)`는 루프가 항상 실행되도록 약속하는 데 사용됩니다.
perform at least `x` iterations, which is necessary for the compiler to
컴파일러가 다음을 수행하는 데 필요한 최소한 `x` 반복을 수행합니다.
produce optimized code for the loop.
루프에 최적화된 코드를 생성합니다.

```cpp
// Compiler-specific macros for loop optimization
#define AIE_LOOP_MIN_ITERATION_COUNT(x) \
    _Pragma(__STRINGIFY(clang loop min_iteration_count(x)))
#define AIE_PIPELINE _Pragma("clang loop pipeline(disable)")

// Optimized loop with pipelining hints
AIE_LOOP_MIN_ITERATION_COUNT(16)
AIE_PIPELINE
for (int i = 0; i < trip_count; i++) {
    aie::vector<bfloat16, negate_vec_size> vin = *in++;
    aie::vector<bfloat16, negate_vec_size> negated = aie::neg(vin);
    *out++ = negated;
}
```

The compiler might fail to pipeline if the loop condition or loop expression
루프 조건 또는 루프 표현식이 다음과 같은 경우 컴파일러는 파이프라인에 실패할 수 있습니다.
are complicated.
복잡합니다.

### Aliasing
### 앨리어싱

**Critical for Performance**: The computational code must be separated into a dedicated function with `__restrict` on its pointer arguments.
**성능에 중요**: 계산 코드는 포인터 인수에 `__restrict`를 사용하여 전용 함수로 분리되어야 합니다.
This allows the single-core compiler to see correct aliasing information and perform optimal vectorization and loop optimizations.
이를 통해 단일 코어 컴파일러는 올바른 앨리어싱 정보를 확인하고 최적의 벡터화 및 루프 최적화를 수행할 수 있습니다.

```cpp
#include <adf.h>
#include "aie_api/aie.hpp"

namespace custom_ops {

static constexpr int negate_kernel_lp_size = 1;

// Separate computational function with __restrict pointers
template <typename dtype_ifm, typename dtype_ofm>
static void compute(
    dtype_ifm *__restrict input_ptr,
    dtype_ofm *__restrict output_ptr,
    int size
) {
    constexpr int vec_size = 16;

    auto in = aie::cbegin_restrict_vector<vec_size>(input_ptr);
    auto out = aie::begin_restrict_vector<vec_size>(output_ptr);

    AIE_LOOP_MIN_ITERATION_COUNT(16)
    AIE_PIPELINE
    for (int i = 0; i < size / vec_size; i++) {
        aie::vector<dtype_ifm, vec_size> vin = *in++;
        aie::vector<dtype_ofm, vec_size> negated = aie::neg(vin);
        *out++ = negated;
    }
}

// Main kernel function calls the computational function
template <typename dtype_ifm, typename dtype_ofm>
__attribute__((noinline)) void negate_kernel_optimized(
    adf::input_buffer_conf<dtype_ifm, bpc_sync_0d> &__restrict ifm,
    adf::output_buffer_conf<dtype_ofm, bpc_sync_0d> &__restrict ofm,
    const uint32_t (&lp_params)[negate_kernel_lp_size]
) {
    int trip_count = (int)lp_params[0];

    // Extract pointers and call computational function
    dtype_ifm *input_ptr = ifm.data();
    dtype_ofm *output_ptr = ofm.data();

    compute(input_ptr, output_ptr, trip_count);
}
}
```

### Banking
### 은행 업무

The AIE Core contains two load units which allows it to load two vectors from memory in the same cycle
AIE 코어에는 동일한 주기에 메모리에서 두 개의 벡터를 로드할 수 있는 두 개의 로드 장치가 포함되어 있습니다.
provided that both pointers are in different banks.
단, 두 포인터가 서로 다른 뱅크에 있어야 합니다.
On AIE2P and AIE2PS, the banks span the following addresses in L1:
AIE2P 및 AIE2PS에서 뱅크는 L1의 다음 주소에 걸쳐 있습니다.
| Bank Name<br>은행명                | Address Range<br>주소 범위       | Size<br>크기   |
|--------------------------|---------------------|--------|
| `__aie_dm_resource_a`<br>`__aie_dm_resource_a`    | `0x0000 - 0x3FFFF`<br>`0x0000 - 0x3FFFF`  | 16 KB  |
| `__aie_dm_resource_b`<br>`__aie_dm_resource_b`    | `0x4000 - 0x7FFFF`<br>`0x4000 - 0x7FFFF`  | 16 KB  |
| `__aie_dm_resource_c`<br>`__aie_dm_resource_c`    | `0x8000 - 0xBFFFF`<br>`0x8000 - 0xBFFFF`  | 16 KB  |
| `__aie_dm_resource_d`<br>`__aie_dm_resource_d`    | `0xC000 - 0xFFFFF`<br>`0xC000 - 0xFFFFF`  | 16 KB  |

Each bank is 16 KB, for a total of 64 KB.
각 뱅크는 16KB이므로 총 64KB입니다.
We can help the compiler to generate more
우리는 컴파일러가 더 많은 것을 생성하도록 도울 수 있습니다
efficient code by adding the `__aie_dm_resource_a` annotations where known:
알려진 경우 `__aie_dm_resource_a` 주석을 추가하여 효율적인 코드를 생성하세요.

```cpp
static void compute(
    bfloat16 __aie_dm_resource_a *__restrict input_ptr,
    // Signal that input2_ptr is in the same bank as input_ptr
    bfloat16 __aie_dm_resource_a *__restrict input2_ptr,
    bfloat16 *__restrict output_ptr,
    int size
) {
 ...
}
```
On pointers that are not annotated, the compiler will assume that those never conflict
주석이 지정되지 않은 포인터에서 컴파일러는 해당 포인터가 충돌하지 않는다고 가정합니다.
with any other loads and schedule them in the same cycle.
다른 부하와 함께 사용하고 동일한 주기로 예약합니다.
This is always safe; the hardware will detect conflicts at runtime and stall
이는 항상 안전합니다. 하드웨어는 런타임 및 정지 시 충돌을 감지합니다.
one of the loads if necessary.
필요한 경우 부하 중 하나입니다.
The bank annotations don't affect stores as there is only a single store unit.
단일 매장 단위만 있으므로 은행 주석은 매장에 영향을 주지 않습니다.

## Advanced Techniques
## 고급 기술

### Determining version of the AI Engine
### AI 엔진 버전 확인
Use the `__AIE_ARCH__` macro or the constexpr `aie::arch::version` variable to determine the target version of the AIE during
`__AIE_ARCH__` 매크로 또는 constexpr `aie::arch::version` 변수를 사용하여 AIE의 대상 버전을 결정합니다.
compilation.
편집.
| `__AIE_ARCH__` Value<br>`__AIE_ARCH__` 값 | `aie::arch::version` value<br>`aie::arch::version` 값 | Architecture Name<br>아키텍처 이름 | Devices<br>장치   |
|----------------------|----------------------------|-------------------|-----------|
| 21                   | `aie::arch::XDNA2`<br>`aie::아치::XDNA2`         | AIE2P<br>AIE2P             | Strix<br>스트릭스     |
| 22                   | `aie::arch::AIE_MLv2`<br>`aie::아치::AIE_MLv2`      | AIE2PS<br>AIE2PS            | Telluride<br>텔루라이드 |

## Reference Links
## 참조 링크

### AIE API Documentation
### AIE API 문서
- [AIE API User Guide](https://download.amd.com/docnav/aiengine/xilinx2025_1/aiengine_api/aie_api/doc/index.html) - Complete API reference
- [AIE API 사용자 가이드](https://download.amd.com/docnav/aiengine/xilinx2025_1/aiengine_api/aie_api/doc/index.html) - 전체 API 참조
- [Basic Types](https://download.amd.com/docnav/aiengine/xilinx2025_1/aiengine_api/aie_api/doc/group__group__basic__types.html) - Vector and accumulator types
- [기본 유형](https://download.amd.com/docnav/aiengine/xilinx2025_1/aiengine_api/aie_api/doc/group__group__basic__types.html) - 벡터 및 누산기 유형
- [Arithmetic Operations](https://download.amd.com/docnav/aiengine/xilinx2025_1/aiengine_api/aie_api/doc/group__group__arithmetic.html) - Mathematical operations
- [산술 연산](https://download.amd.com/docnav/aiengine/xilinx2025_1/aiengine_api/aie_api/doc/group__group__arithmetic.html) - 수학적 연산
- [Memory Operations](https://download.amd.com/docnav/aiengine/xilinx2025_1/aiengine_api/aie_api/doc/group__group__memory.html) - Load/store operations
- [메모리 작업](https://download.amd.com/docnav/aiengine/xilinx2025_1/aiengine_api/aie_api/doc/group__group__memory.html) - 로드/저장 작업

### AIE Programming Resources
### AIE 프로그래밍 리소스
- [AIE Kernel Programming User Guide](https://docs.amd.com/r/en-US/ug1079-ai-engine-kernel-coding/Introduction-to-Scalar-and-Vector-Programming) - Programming guide
- [AIE 커널 프로그래밍 사용자 가이드](https://docs.amd.com/r/en-US/ug1079-ai-engine-kernel-coding/Introduction-to-Scalar-and-Vector-Programming) - 프로그래밍 가이드

### Custom Ops Resources
### 맞춤형 운영 리소스
- [Custom Ops Main Documentation](README.md) - Overview and configuration
- [Custom Ops 주요 문서](README.md) - 개요 및 구성
- [Custom Ops Tutorial Examples](tutorial/README.md) - Practical examples
- [Custom Ops 튜토리얼 예제](tutorial/README.md) - 실제 예제
