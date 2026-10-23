<!-- ko-translation: enabled -->
<!--- Copyright (C) 2026 Advanced Micro Devices, Inc. All rights reserved.--->
# AIE API Reference (detailed)
# AIE API 참조(상세)

A detailed reference for the AIE API (`aie::` namespace, from
AIE API에 대한 자세한 참조(`aie::` 네임스페이스, 출처:
`<aie_api/aie.hpp>`) primitives used in kernels -- giving, for each primitive,
`<aie_api/aie.hpp>`) 커널에 사용되는 프리미티브 -- 각 프리미티브에 대해 다음을 제공합니다.
the **per-dtype support, whether it vectorizes (and at what lane width), the
**dtype별 지원, 벡터화 여부(및 레인 너비),
accumulator it lands in, and the idiomatic usage**.
누산기 및 관용적 사용법 **.
This is the depth the raw
이것은 원시의 깊이입니다
header doc-comments do not surface at a glance, so an agent can answer three
헤더 문서-댓글은 한눈에 드러나지 않으므로 상담원은 세 가지에 답변할 수 있습니다.
questions without reading the whole header tree:
전체 헤더 트리를 읽지 않고 질문:

1. **Does this primitive exist for my dtype?** (bf16 / fp32 = `accfloat` / int8)
1. **이 기본 요소가 내 dtype에 존재합니까?** (bf16 / fp32 = `accfloat` / int8)
2. **Is it vectorized, and what does it accumulate into?**
2. **벡터화되어 있으며 무엇으로 축적되나요?**
3. **What is the idiomatic, fused primitive for my math pattern?**
3. **내 수학 패턴에 대한 관용적 융합 기본 요소는 무엇입니까?**
(so we don't write `mul`+`sub` where `msc_square` exists)
(그래서 우리는 `msc_square`가 존재하는 곳에 `mul`+`sub`를 쓰지 않습니다)

**Scope.** Populated for **reduction / normalization** kernels today (LayerNorm,
**범위.** 현재 **감소/정규화** 커널을 위해 채워집니다(LayerNorm,
RMSNorm, Softmax statistics, `Reduce*`), covering both the **core** primitives
RMSNorm, Softmax 통계, `Reduce*`), **핵심** 프리미티브를 모두 포함
(load/store, arithmetic, reductions) and the **fused / advanced** ones
(로드/저장, 산술, 축소) 및 **융합/고급** 항목
(`mac_square`, `msc_square`, `reduce_add_v`, `invsqrt`, `to_float`/`to_fixed`).
(`mac_square`, `msc_square`, `reduce_add_v`, `invsqrt`, `to_float`/`to_fixed`).
It also covers **data-movement / shuffle / de-interleave** primitives
또한 **데이터 이동/셔플/디인터리브** 기본 요소도 다룹니다.
(`load_unaligned_v`, `filter_even`/`filter_odd`, `shuffle_up_fill` /
(`load_unaligned_v`, `filter_even`/`filter_odd`, `shuffle_up_fill` /
`shuffle_down_fill`, `broadcast`) needed by **windowed / strided spatial** ops
**창/스트라이드 공간** 작업에 필요한 `shuffle_down_fill`, `broadcast`)
(pooling, conv, stride-2 resample) where the output straddles even/odd lanes or
(풀링, 전환, 스트라이드-2 리샘플) 출력이 짝수/홀수 레인에 걸쳐 있거나
a tap is not vector-aligned.
탭이 벡터 정렬되지 않았습니다.
It also covers the **matrix-multiply** primitive
또한 **행렬 곱셈** 기본 요소도 다룹니다.
`aie::mmul` (GEMM / matmul / linear / 1x1-conv), which is the one non-vector
벡터가 아닌 `aie::mmul` (GEMM / matmul / 선형 / 1x1-conv)
datapath here.
여기 데이터 경로가 있습니다.
The structure is generic and can be extended to other op
구조는 일반적이며 다른 작업으로 확장될 수 있습니다.
families.
가족.

This reference is part of the `/vai-custom-op` skill (kernel authoring, see Phase 2c)
이 참조는 `/vai-custom-op` 기술의 일부입니다(커널 작성, 2c단계 참조).
and complements `/vai-aie-compiler-oriented-optimizations` (compiler/pipelining
`/vai-aie-compiler-지향-최적화`(컴파일러/파이프라인)을 보완합니다.
hints).
힌트).
It is not an exhaustive signature dump -- for the full overload set of
이는 완전한 서명 덤프가 아닙니다.
any primitive, [discover it from the headers](#if-its-not-here-discover-from-the-headers).
모든 기본 요소, [헤더에서 검색](#if-its-not-here-discover-from-the-headers).

---

## How to use this reference
## 이 참조를 사용하는 방법

1. Start from your **math pattern**, not an API name -> [Pattern -> API](#pattern--api).
1. API 이름 -> [패턴 -> API](#pattern--api)가 아닌 **수학 패턴**에서 시작하세요.
2. Confirm dtype support / vectorization / accumulator in the
2. dtype 지원/벡터화/누산기를 확인하세요.
[API catalog](#api-catalog).
[API 카탈로그](#api-catalog).
3. Sanity-check against the [precision & vectorization rules](#precision--vectorization-rules).
3. [정밀도 및 벡터화 규칙](#precision--벡터화-규칙)에 대한 온전성 검사를 수행합니다.
4. If the primitive you need is not here ->
4. 필요한 기본 요소가 여기에 없으면 ->
[discover it from the headers](#if-its-not-here-discover-from-the-headers).
[헤더에서 찾아보세요](#if-its-not-here-discover-from-the-headers).
Then consider adding it back here for the next agent.
그런 다음 다음 상담원을 위해 여기에 다시 추가하는 것을 고려해 보세요.

---

## Precision & vectorization rules
## 정밀도 및 벡터화 규칙

Generic rules that decide *how* you call these primitives, independent of family:
계열과 관계없이 이러한 기본 요소를 호출하는 *방법*을 결정하는 일반 규칙:

- **Accumulate bf16 in `accfloat`, never in bf16.** Use
- **`accfloat`에 bf16을 축적하고, 절대로 bf16에 축적하지 마세요.** 사용
`aie::accum<accfloat, N>`; widen with `acc.from_vector(vec)` and read back with
`aie::accum<accfloat, N>`; `acc.from_Vector(vec)`로 확장하고 다음으로 다시 읽습니다.
`acc.to_vector<T>()`.
`acc.to_Vector<T>()`.
Summing many bf16 values in bf16 loses precision fast.
bf16에서 많은 bf16 값을 합산하면 정밀도가 빠르게 손실됩니다.
- **Prefer the fused accumulate primitive over `add`/`mac` of a hand-built term.**
- **손으로 만든 용어의 `add`/`mac`보다 융합된 누적 기본 형식을 선호합니다.**
`mac_square(acc, v)` and `msc_square(acc, v)` keep the square in the
`mac_square(acc, v)` 및 `msc_square(acc, v)`는 사각형을
accumulator datapath; the naive form spends extra vector ops and a wider
누산기 데이터 경로; 순진한 형태는 추가 벡터 연산을 소비하고 더 넓은 범위를 차지합니다.
dependency chain.
종속성 체인.
- **Stay in the vector datapath for stats when you can.** Dropping to a scalar
- **가능한 경우 통계를 보려면 벡터 데이터 경로를 유지하세요.** 스칼라로 떨어지기
(`reduce_add` -> scalar `mean`/`var`/`invsqrt`) creates a serial
(`reduce_add` -> 스칼라 `mean`/`var`/`invsqrt`)는 직렬을 생성합니다.
reduce->invsqrt critical path.
감소->invsqrt 중요 경로.
`reduce_add_v` + vector `invsqrt` keeps the
`reduce_add_v` + 벡터 `invsqrt`는
chain in vector registers. (Whether this is worth it depends on whether the op
벡터 레지스터의 체인. (이것이 가치가 있는지 여부는 op가
is compute-bound vs DMA-bound -- measure with `/vai-perf-analysis` first.)
계산 바인딩과 DMA 바인딩입니다. 먼저 '/vai-perf-analytic'으로 측정하세요.)
- **Batching a transcendental across rows still stays in the FPU.** When you
- **행 전체에 걸쳐 초월 일괄 처리는 여전히 FPU에 유지됩니다.**
need each row's stat in its own lane (e.g. one `invsqrt` over R rows), place
각 행의 통계가 자체 레인에 필요합니다(예: R 행에 대해 하나의 `invsqrt`).
row r's reduced scalar into lane r with the `vector::set(value, idx)` member
`벡터::set(value, idx)` 멤버를 사용하여 행 r의 축소된 스칼라를 레인 r로 변환합니다.
(or `vec[idx] = value`) / a shift-in, *not* by writing `reduce_add` scalars to
(또는 `vec[idx] = value`) / 시프트 인, `reduce_add` 스칼라를 작성하여 *아님*
a scratch array and reloading.
스크래치 배열 및 다시 로드.
The scratch-array round-trip leaves the vector
스크래치 배열 왕복은 벡터를 남깁니다.
FPU, adds L1 traffic, and grows the stack -- it defeats the batching it was
FPU, L1 트래픽 추가 및 스택 확장 - 이전의 일괄 처리를 무효화합니다.
meant to enable.
활성화한다는 의미입니다.
- **Match lane width to dtype** (same tuning the main skill uses): bf16 = 32
- **차선 너비를 dtype에 일치**(주 스킬이 사용하는 것과 동일한 튜닝): bf16 = 32
lanes, int8 = 64, int32 = 16.
레인, int8 = 64, int32 = 16.
Size tiles to an exact multiple so there is no
타일의 크기를 정확한 배수로 조정하여
scalar remainder.
스칼라 나머지.
- **Scalar `float` math is emulated -- keep arithmetic in vectors.** The scalar
- **스칼라 `float` 수학이 에뮬레이트됩니다. 산술을 벡터로 유지합니다.** 스칼라
unit has no FPU: a plain `float` `*`, `/`, `int->float` cast, or compare lowers
유닛에는 FPU가 없습니다: 일반 `float` `*`, `/`, `int->float` 캐스트 또는 비교 로우
to a soft-float library call (`__mulsf3`, `__floatsisf`, `__gesf2`, ...).
소프트 플로트 라이브러리 호출(`__mulsf3`, `__floatsisf`, `__gesf2`, ...).
Those
그
calls are un-inlinable and force the compiler to spill live values to the stack
호출은 인라인화할 수 없으며 컴파일러가 실시간 값을 스택에 유출하도록 강제합니다.
around each one, so scalar float costs both cycles and stack frame.
스칼라 플로트는 사이클과 스택 프레임 모두에 비용이 듭니다.
Do the math
수학을 해보세요
on `aie::vector` (even a small lane count) and reserve scalars for lane
`aie::벡터`(작은 차선 개수라도)에서 차선에 대한 스칼라를 예약합니다.
moves (`v.get`/`v.set`) and index math.
이동(`v.get`/`v.set`) 및 인덱스 수학.
Diagnostic: `grep` the kernel
진단: 커널 `grep`
`.calltree` for `__*sf*` symbols -- nonzero means scalar float is still there.
`__*sf*` 기호에 대한 `.calltree` -- 0이 아닌 경우 스칼라 부동 소수점이 여전히 존재한다는 의미입니다.
- **int8 in/out needs explicit (de)quant**: `aie::to_float(v.unpack(), shift)` on
- **int8 입력/출력에는 명시적인 (비)퀀트가 필요합니다**: `aie::to_float(v.unpack(), Shift)` on
the way in, `aie::to_fixed<int8>(acc.to_vector<bf16>(), out_shift)` on the way
들어오는 길, `aie::to_fixed<int8>(acc.to_Vector<bf16>(), out_shift)` 가는 길
out.
밖으로.
There is no implicit int8<->float conversion.
암시적 int8<->float 변환은 없습니다.
- **For complex math (`exp` / `exp2`, and similar):** if the output is int8,
- **복잡한 수학(`exp` / `exp2` 등):** 출력이 int8인 경우,
implement it with an exact LUT -- it is both more accurate and as fast as the
정확한 LUT로 구현하세요. 더 정확하고 빠릅니다.
HW math.
하드웨어 수학.
If the output is bf16/fp32, use the matching AIE API (`aie::exp2`,
출력이 bf16/fp32인 경우 일치하는 AIE API(`aie::exp2`,
etc.).
등).
Details: [Complex Math functions implementation](#complex-math-functions-implementation).
세부정보: [복잡한 수학 함수 구현](#complex-math-functions-implementation).
- **Prefer aligned `load_v` over `load_unaligned_v`.** Vector loads want a
- **`load_unaligned_v`보다 정렬된 `load_v`를 선호합니다.** 벡터 로드는
vector-aligned address.
벡터 정렬된 주소입니다.
When a tap is not aligned (a halo/shifted window in a
탭이 정렬되지 않은 경우(후광/이동된 창이
stencil), `load_unaligned_v` works but the compiler lowers it to an aligned
스텐실), `load_unaligned_v`는 작동하지만 컴파일러는 이를 정렬된 값으로 낮춥니다.
load + a `vshift`, so it is not free.
로드 + `vshift`이므로 무료가 아닙니다.
Where possible, keep the base
가능하면 베이스를 유지하세요.
vector-aligned and derive shifted lanes with `shuffle_*` from vectors you have
벡터 정렬을 수행하고 가지고 있는 벡터에서 `shuffle_*`을 사용하여 이동된 차선을 파생시킵니다.
already loaded, or lay out an L1 scratch with a left pad so the shifted base
이미 로드되었거나 왼쪽 패드로 L1 스크래치를 배치하여 베이스가 이동되도록 합니다.
stays aligned (see the [de-interleave example](#windowed--strided-de-interleave-examples)).
정렬된 상태를 유지합니다([디인터리브 예시](#windowed--strided-de-interleave-examples) 참조).

---

## Pattern -> API
## 패턴 -> API

Map the math you need to the idiomatic primitive. (`acc` = `aie::accum<accfloat,N>`.)
필요한 수학을 관용적 기본 요소에 매핑합니다. (`acc` = `aie::accum<accfloat,N>`.)

| You need...<br>당신은 필요합니다 ...                              | Use<br>사용                                              | Do NOT hand-roll as<br>손으로 굴리지 마십시오.            |
|------------------------------------------|--------------------------------------------------|--------------------------------|
| sum of a vector into fp32 acc<br>fp32 acc에 대한 벡터의 합            | `acc = aie::add(acc, v)`<br>`acc = aie::add(acc, v)`                          | scalar accumulate loop<br>스칼라 누적 루프         |
| **sum of squares** into fp32 acc<br>**제곱합**을 fp32 acc로         | **`acc = aie::mac_square(acc, v)`**<br>**`acc = aie::mac_square(acc, v)`**              | `aie::mac(acc, v, v)`<br>`aie::mac(acc, v, v)`          |
| **variance `E[x^2] - mean^2`**<br>**분산`E[x^2] - 평균^2`**           | **`aie::msc_square(Ex2_acc, mean_vec)`**<br>**`aie::msc_square(Ex2_acc, 평균_vec)`**         | scalar `sumsq*inv - mean*mean`<br>스칼라 `sumsq*inv - 평균*평균` |
| horizontal reduce, keep result in vector<br>수평 축소, 결과를 벡터로 유지 | **`aie::reduce_add_v(acc.to_vector<float>())`**<br>**`aie::reduce_add_v(acc.to_Vector<float>())`**  | `aie::reduce_add` (-> scalar)<br>`aie::reduce_add` (-> 스칼라)  |
| `1/sqrt(x)` over a vector<br>벡터에 대한 `1/sqrt(x)`                | **`aie::invsqrt(v)`**<br>**`aie::invsqrt(v)`**                            | Newton iters / scalar `1/sqrt`<br>뉴턴 반복/스칼라 `1/sqrt` |
| clamp variance to `>= eps` (stability)<br>분산을`>=eps`로 고정(안정성)   | **`aie::max(var_eps, broadcast(eps))`**<br>**`aie::max(var_eps, 방송(eps))`**          | branchy scalar guard<br>가지가 있는 스칼라 가드           |
| affine `a*x + b` fused<br>아핀 `a*x + b` 융합                   | `aie::mac(aie::accum(b_vec), x_vec, a_vec)`<br>`aie::mac(aie::accum(b_vec), x_vec, a_vec)`      | separate `mul` then `add`<br>`mul`을 분리한 다음 `add`를 분리합니다.      |
| **matrix multiply / GEMM tile** (contract two 8x8 tiles)<br>**행렬 곱셈/GEMM 타일**(2개의 8x8 타일 계약) | **`aie::mmul<8,8,8,TA,TB>` + `.mul`/`.mac`**<br>**`aie::mmul<8,8,8,TA,TB>` + `.mul`/`.mac`** | a doubly-nested lane-wise `aie::mac` loop (mmul issues a full 8x8x8 tile step = 512 MACs)<br>이중으로 중첩된 레인별 `aie::mac` 루프(mmul은 전체 8x8x8 타일 단계 = 512 MAC를 발행함) |
| int8 -> float dequant (with shift)<br>int8 -> float 디퀀트(시프트 포함)       | **`aie::to_float(v.unpack(), shift)`** (vector only)<br>**`aie::to_float(v.unpack(), Shift)`** (벡터 전용) | manual `* scale`, or a **scalar** `to_float` (silently returns 0)<br>수동 `* scale` 또는 **scalar** `to_float`(자동으로 0을 반환) |
| float -> int8 quant (with shift)<br>float -> int8 퀀트(shift 포함)         | **`aie::to_fixed<int8>(v, out_shift)`** (vector only)<br>**`aie::to_fixed<int8>(v, out_shift)`** (벡터만 해당) | manual round/clamp/cast, or a **scalar** `to_fixed` (silently returns 0)<br>수동 라운드/클램프/캐스트 또는 **스칼라** `to_fixed`(자동으로 0을 반환) |
| **split even/odd lanes** (stride-2 gather, K-reduction)<br>**짝수/홀수 레인 분할**(스트라이드-2 수집, K-감소) | **`aie::filter_even(v)` / `aie::filter_odd(v)`**<br>**`aie::filter_even(v)` / `aie::filter_odd(v)`** | scalar strided index loop<br>스칼라 스트라이드 인덱스 루프 |
| **shifted / halo tap** (offset not vector-aligned)<br>**이동됨/후광 탭**(오프셋이 벡터 정렬되지 않음) | **`aie::load_unaligned_v<N>(ptr)`** or `shuffle_*` of a loaded vec<br>**`aie::load_unaligned_v<N>(ptr)`** 또는 로드된 vec의 `shuffle_*` | `load_v` at a misaligned addr (reads wrong data)<br>잘못 정렬된 주소의 `load_v`(잘못된 데이터 읽기) |
| **shift lanes by n, fill vacated lanes** (pad/halo carry)<br>**n만큼 차선 이동, 빈 차선 채우기**(패드/후광 캐리) | **`aie::shuffle_up_fill(v, fill, n)` / `shuffle_down_fill`**<br>**`aie::shuffle_up_fill(v, fill, n)` / `shuffle_down_fill`** | scratch store at +/-1 offset (unaligned)<br>+/-1 오프셋에 스크래치 저장(정렬되지 않음) |
| **pad sentinel** for max/min windows (`-inf`/`+inf`)<br>**패드 센티널** 최대/최소 창(`-inf`/`+inf`) | **`aie::broadcast<bf16,N>((bfloat16)-3.0e38f)`**<br>**`aie::broadcast<bf16,N>((bfloat16)-3.0e38f)`** | per-lane scalar store of the pad<br>패드의 레인별 스칼라 저장 |

---

## API catalog
## API 카탈로그

Columns: **dtypes** (in), **vec** (vectorized?), **accum** (lands in), **lanes**,
열: **dtypes**(in), **vec**(벡터화?), **accum**(착지), **lanes**,
**notes** (idiomatic usage / why), **header**.
**참고**(관용적 사용법/이유), **헤더**.

### Core primitives
### 핵심 프리미티브

The everyday load/store, arithmetic, and reduction ops.
일상적인 로드/저장, 산술 및 감소 작업입니다.
Included here with the
여기에 포함됨
per-dtype / vectorization / accumulator detail rather than just the name.
단순한 이름이 아닌 dtype별/벡터화/누산기 세부정보입니다.

| API<br>API | math<br>수학 | dtypes<br>dtypes | vec<br>vec | accum<br>축적 | lanes<br>차선 | notes<br>메모 | header<br>헤더 |
|---|---|---|---|---|---|---|---|
| `aie::load_v<N>(ptr)` / `aie::store_v(ptr, v)`<br>`aie::load_v<N>(ptr)` / `aie::store_v(ptr, v)` | vector load / store<br>벡터 로드/저장 | any<br>어떤 | yes<br>응 | n/a | N | address must be vector-aligned (see compiler skill ch.11)<br>주소는 벡터 정렬되어야 합니다(컴파일러 기술 ch.11 참조). | `detail/*ld_st*.hpp`<br>`세부정보/*ld_st*.hpp` |
| `aie::broadcast<T,N>(s)`<br>`aie::broadcast<T,N>(들)` | splat scalar to N lanes<br>스칼라를 N 레인으로 표시 | any<br>어떤 | yes<br>응 | n/a | N | build a vector constant (e.g. `eps`, `inv_D`, or a `-inf`/`LOWEST` pad sentinel for max/min windows)<br>벡터 상수 구축(예: `eps`, `inv_D` 또는 최대/최소 창에 대한 `-inf`/`LOWEST` 패드 센티널) | `detail/broadcast*.hpp`<br>`세부정보/방송*.hpp` |
| `aie::add(acc, v)` / `aie::add(v, v)`<br>`aie::add(acc, v)` / `aie::add(v, v)` | add<br>추가하다 | bf16, fp32, int<br>bf16, fp32, 정수 | yes<br>응 | `accfloat` (acc form)<br>`accfloat`(acc 형식) | 16/32/64 | acc form widens bf16 into fp32 -- use for reduction sums<br>acc 형식은 bf16을 fp32로 확장합니다 - 축소 합계에 사용 | `detail/add*.hpp`<br>`세부정보/추가*.hpp` |
| `aie::sub(a, b)`<br>`aie::sub(a, b)` | subtract<br>빼다 | bf16, fp32, int<br>bf16, fp32, 정수 | yes<br>응 | `accfloat`/vec<br>`accfloat`/vec | 16/32/64 | -- | `detail/sub*.hpp`<br>`세부정보/하위*.hpp` |
| `aie::mul(a, b)`<br>`aie::mul(a, b)` | multiply (-> acc)<br>곱하기 (-> acc) | bf16, fp32, int<br>bf16, fp32, 정수 | yes<br>응 | `accfloat`<br>'acfloat' | 16/32 | returns an accumulator; narrow with `.to_vector<T>()`<br>누산기를 반환합니다. `.to_Vector<T>()`로 범위를 좁힙니다. | `detail/mul*.hpp`<br>`세부정보/mul*.hpp` |
| `aie::mac(acc, a, b)`<br>`aie::mac(acc, a, b)` | `acc += a*b`<br>`acc += a*b` | bf16, fp32, int<br>bf16, fp32, 정수 | yes<br>응 | `accfloat`<br>'acfloat' | 16/32 | fused multiply-accumulate; basis of affine `a*x+b`<br>융합된 곱셈-축적; 아핀 `a*x+b`의 기초 | `detail/mul*.hpp`<br>`세부정보/mul*.hpp` |
| `aie::max(a,b)` / `aie::min(a,b)`<br>`aie::max(a,b)` / `aie::min(a,b)` | elementwise max/min<br>요소별 최대/최소 | any<br>어떤 | yes<br>응 | vec<br>vec | 16/32/64 | also used as branchless clamp (e.g. `max(var,eps)`); bf16 lowers to `vmax_lt.bf16`; basis of pooling reductions<br>분기 없는 클램프로도 사용됩니다(예: `max(var,eps)`). bf16은 `vmax_lt.bf16`으로 낮아집니다. 풀링 감소의 기초 | `detail/max_min.hpp`<br>`세부정보/max_min.hpp` |
| `aie::abs(v)` / `aie::neg(v)`<br>`aie::abs(v)` / `aie::neg(v)` | abs / negate<br>절대 / 부정 | any<br>어떤 | yes<br>응 | vec<br>vec | 16/32/64 | -- | `detail/abs.hpp` / `detail/neg.hpp`<br>`세부사항/abs.hpp` / `세부사항/neg.hpp` |
| `aie::reduce_add(v)` / `reduce_max` / `reduce_min`<br>`aie::reduce_add(v)` / `reduce_max` / `reduce_min` | horizontal reduce -> **scalar**<br>수평 축소 -> **스칼라** | bf16, fp32, int<br>bf16, fp32, 정수 | yes<br>응 | scalar<br>스칼라 | -> 1 | pulls onto scalar unit; prefer `reduce_add_v` to stay vector<br>스칼라 단위를 끌어당깁니다. 벡터를 유지하려면 `reduce_add_v`를 선호합니다. | `detail/reduce*.hpp`<br>`세부정보/축소*.hpp` |
| `v.set(val, idx)` / `v.get(idx)` / `v[idx]`<br>`v.set(val, idx)` / `v.get(idx)` / `v[idx]` | write / read a **single lane**<br>**단일 레인** 쓰기/읽기 | any<br>어떤 | n/a | n/a | 1 lane<br>1차선 | member fns (arg order is **value-first**: `set(val, idx)`); use to place a scalar at a lane (e.g. row->lane batching). **No** free `aie::insert(vec,idx,scalar)`; the `v.insert(idx, subvec)` member is *sub-vector-at-partition-index*, not scalar-at-lane<br>멤버 fns(인수 순서는 **값 우선**: `set(val, idx)`입니다.); 레인에 스칼라를 배치하는 데 사용됩니다(예: 행->레인 일괄 처리). **아니요** 무료 `aie::insert(vec,idx,scalar)`; `v.insert(idx, subvec)` 멤버는 *sub-Vector-at-partition-index*이며, scalar-at-lane이 아닙니다. | `vector.hpp`<br>`벡터.hpp` |
| `tanh(v)` / `exp2(v)`<br>`tanh(v)` / `exp2(v)` | transcendentals<br>초월적인 | bf16, fp32 (acc)<br>bf16, fp32(acc) | yes<br>응 | n/a | 16/32 | HW-accelerated per arch; use the library form, never `std::` (no stdc++ on AIE)<br>아치당 HW 가속; 라이브러리 형식을 사용하고 `std::`는 사용하지 마세요(AIE에서는 stdc++ 없음). | `detail/elementary*.hpp`<br>`세부정보/기본*.hpp` |

### Fused / advanced primitives
### 융합/고급 프리미티브

The easy-to-miss or hand-rolled-by-accident ones.
놓치기 쉬운 것이나 사고로 손으로 굴린 것.
These are the highest-value
이는 가장 높은 가치를 지닌
entries -- prefer them over the naive composition shown in the last column.
항목 - 마지막 열에 표시된 순진한 구성보다 선호합니다.

| API<br>API | math<br>수학 | dtypes<br>dtypes | vec<br>vec | accum<br>축적 | lanes<br>차선 | notes (vs hand-rolled)<br>노트(손으로 말은 것과 비교) | header<br>헤더 |
|---|---|---|---|---|---|---|---|
| `aie::mac_square(acc, v)`<br>`aie::mac_square(acc, v)` | `acc += v*v`<br>`acc += v*v` | bf16, fp32<br>bf16, fp32 | yes<br>응 | `accfloat`<br>'acfloat' | 16/32 | square+accumulate in 1 op; tighter dep chain than `mac(v,v)`<br>정사각형+1개 연산으로 누적; `mac(v,v)`보다 더 긴밀한 심층 체인 | `detail/mul*.hpp`<br>`세부정보/mul*.hpp` |
| `aie::msc_square(acc, v)`<br>`aie::msc_square(acc, v)` | `acc - v*v`<br>`acc - v*v` | bf16, fp32<br>bf16, fp32 | yes<br>응 | `accfloat`<br>'acfloat' | 16/32 | the whole `E[x^2]-mean^2` variance step, fused & vectorized; vs scalar `sumsq*inv - mean*mean`<br>전체 `E[x^2]-평균^2` 분산 단계, 융합 및 벡터화; 대 스칼라`sumsq*inv - 평균*평균` | `detail/mul*.hpp`<br>`세부정보/mul*.hpp` |
| `aie::reduce_add_v(v)`<br>`aie::reduce_add_v(v)` | horizontal sum, **returns vector**<br>수평 합, **벡터를 반환** | fp32, bf16<br>fp32, bf16 | yes<br>응 | n/a | lane->lane<br>차선->차선 | keeps reduction in vector regs (no scalar FPU round-trip); vs `reduce_add`<br>벡터 등록 감소를 유지합니다(스칼라 FPU 왕복 없음). vs `reduce_add` | `detail/reduce*.hpp`<br>`세부정보/축소*.hpp` |
| `aie::invsqrt(v)`<br>`aie::invsqrt(v)` | `1/sqrt(v)`<br>`1/sqrt(v)` | bf16, fp32<br>bf16, fp32 | yes<br>응 | n/a | 16/32 | HW reciprocal-sqrt; vs scalar `1.0f/aie::sqrt(v)` or Newton<br>HW 상호-sqrt; 대 스칼라 `1.0f/aie::sqrt(v)` 또는 뉴턴 | `detail/sqrt.hpp` / `elementary`<br>`detail/sqrt.hpp` / `elementary` |
| `aie::inv(v)`<br>`aie::inv(v)` | `1/v` | bf16, fp32<br>bf16, fp32 | yes<br>응 | n/a | 16/32 | HW reciprocal; avoids scalar divide<br>HW 상호; 스칼라 분할 방지 | `detail/*inv*.hpp`<br>`세부정보/*inv*.hpp` |
| `aie::to_float(v, shift)`<br>`aie::to_float(v, Shift)` | int->float w/ shift<br>int->float(shift 포함) | int8/16/32<br>정수8/16/32 | **vector only**<br>**벡터만** | n/a | 16/32/64 | fused dequant; pair with `.unpack()` for int8. **No scalar overload** -- a scalar arg silently returns 0 (no error)<br>융합된 디퀀트; int8의 경우 `.unpack()`과 쌍을 이룹니다. **스칼라 오버로드 없음** -- 스칼라 인수가 자동으로 0을 반환합니다(오류 없음). | `detail/to_float*.hpp`<br>`detail/to_float*.hpp` |
| `aie::to_fixed<T>(v, shift)`<br>`aie::to_fixed<T>(v, Shift)` | float->int w/ shift<br>float->int(shift 포함) | bf16, fp32<br>bf16, fp32 | **vector only**<br>**벡터만** | n/a | 16/32 | fused round+scale+clamp to int output. **No scalar overload** -- a scalar arg silently returns 0 (no error)<br>융합된 라운드+스케일+클램프를 int 출력에 적용합니다. **스칼라 오버로드 없음** -- 스칼라 인수가 자동으로 0을 반환합니다(오류 없음). | `detail/to_fixed*.hpp`<br>`detail/to_fixed*.hpp` |
| `acc.from_vector(v)` / `acc.to_vector<T>()`<br>`acc.from_Vector(v)` / `acc.to_Vector<T>()` | widen / narrow<br>넓히다 / 좁히다 | any<br>어떤 | yes<br>응 | `accfloat`/...<br>'acfloat'/... | N | the canonical bf16<->fp32-acc bridge<br>표준 bf16<->fp32-acc 브리지 | `aie.hpp` (accum)<br>`aie.hpp`(누적) |

### Data movement, shuffle & de-interleave primitives
### 데이터 이동, 셔플 및 디인터리브 프리미티브

Lane-rearrangement and non-aligned access.
차선 재배치 및 비정렬 접근.
These are the primitives that make
이들은 만드는 기본 요소입니다.
**windowed / strided spatial** ops (pooling, stride-2 resample, stencils)
**창/스트라이드 공간** 작업(풀링, 스트라이드-2 리샘플, 스텐실)
vectorizable: an odd window with an even stride makes the output straddle even
벡터화 가능: 스트라이드가 짝수인 홀수 창은 출력을 짝수로 만듭니다.
and odd input lanes, so you must de-interleave, and halo taps land at
홀수 입력 레인이므로 디인터리브를 수행해야 하며 후광 탭이
non-vector-aligned offsets.
벡터 정렬되지 않은 오프셋.
All lower to `vshuffle` / `vshift` on the vector
모두 벡터의 `vshuffle` / `vshift`로 낮음
unit (visible as such in the Peano `.lst`).
단위(Peano `.lst`에서 볼 수 있음).

| API<br>API | math<br>수학 | dtypes<br>dtypes | vec<br>vec | accum<br>축적 | lanes<br>차선 | notes (idiomatic usage / why)<br>메모(관용적 사용법/이유) | header<br>헤더 |
|---|---|---|---|---|---|---|---|
| `aie::load_unaligned_v<N>(ptr, aligned_elems=1)`<br>`aie::load_unaligned_v<N>(ptr, alignment_elems=1)` | vector load from a non-vector-aligned address<br>벡터 정렬되지 않은 주소에서 벡터 로드 | any<br>어떤 | yes<br>응 | n/a | N | for a shifted/halo tap whose offset is not a multiple of the vector width (e.g. the `A[2*wo-1]` left tap of a stride-2 pool). Lowers to aligned load + `vshift`, so **not free** -- prefer keeping the base aligned and deriving shifts via `shuffle_*`, or pad the L1 scratch so the base stays aligned<br>오프셋이 벡터 폭의 배수가 아닌 시프트/후광 탭의 경우(예: 스트라이드-2 풀의 `A[2*wo-1]` 왼쪽 탭) 정렬된 로드 + `vshift`로 낮추어 **자유롭지 않음** - 베이스 정렬을 유지하고 `shuffle_*`을 통해 시프트를 파생시키는 것을 선호하거나 베이스가 정렬된 상태를 유지하도록 L1 스크래치를 패딩합니다. | `aie.hpp` (`load_unaligned_v`) / `detail/*ld_st*.hpp`<br>`aie.hpp` (`load_unaligned_v`) / `detail/*ld_st*.hpp` |
| `aie::filter_even(v, chunk=1)` / `aie::filter_odd(v, chunk=1)`<br>`aie::filter_even(v, 청크=1)` / `aie::filter_odd(v, 청크=1)` | keep even- / odd-indexed lanes (de-interleave)<br>짝수/홀수 인덱스 레인 유지(디인터리브) | bf16, int, fp32<br>bf16, 정수, fp32 | yes<br>응 | n/a | N -> N/2 | the stride-2 de-interleave: `filter_even([a0,a1,a2,a3,...]) = [a0,a2,...]`. Use for pooling/conv with stride 2 (output = even lanes) and for grouped inner-axis reductions (two stages split a 4-way group into 4 lane-streams -- see `reducemax/reduce_max_axis_2`). Optional `chunk` keeps groups of `chunk` consecutive elements together. Lowers to `vshuffle`<br>stride-2 디인터리브: `filter_even([a0,a1,a2,a3,...]) = [a0,a2,...]`. 스트라이드 2(출력 = 짝수 레인)의 풀링/변환 및 그룹화된 내부 축 감소(2단계가 4방향 그룹을 4개의 레인 스트림으로 분할합니다 -- `reducemax/reduce_max_axis_2` 참조)에 사용합니다. 선택적 `chunk`는 `chunk` 연속 요소 그룹을 함께 유지합니다. 'vshuffle'로 낮춥니다. | `aie.hpp` (`filter_even`/`filter_odd`) / `detail/filter*.hpp`<br>`aie.hpp` (`filter_even`/`filter_odd`) / `detail/filter*.hpp` |
| `aie::shuffle_up_fill(v, fill, n)` / `aie::shuffle_down_fill(v, fill, n)`<br>`aie::shuffle_up_fill(v, fill, n)` / `aie::shuffle_down_fill(v, fill, n)` | shift lanes up/down by `n`, filling the vacated lanes from `fill`<br>'n'만큼 차선을 위/아래로 이동하고, 'fill'에서 비어 있는 차선을 채웁니다. | any<br>어떤 | yes<br>응 | n/a | N | carry a halo/pad element across a vector boundary without an unaligned access: e.g. build the `A[2*wo-1]` tap as `shuffle_up_fill(odd_lanes, prev_or_pad, 1)`. `fill` supplies the entering lanes (use a `broadcast(LOWEST)` for a max-pool pad). Lowers to `vshuffle`<br>정렬되지 않은 액세스 없이 벡터 경계를 넘어 후광/패드 요소를 전달합니다. `A[2*wo-1]` 탭을 `shuffle_up_fill(odd_lanes, prev_or_pad, 1)`로 빌드합니다. 'fill'은 진입 차선을 제공합니다(최대 풀 패드에는 'broadcast(LOWEST)' 사용). 'vshuffle'로 낮춥니다. | `aie.hpp` (`shuffle_*`) / `detail/shuffle*.hpp`<br>`aie.hpp` (`shuffle_*`) / `detail/shuffle*.hpp` |
| `aie::shuffle_up(v,n)` / `aie::shuffle_down(v,n)` (+ `_rotate` / `_replicate` variants)<br>`aie::shuffle_up(v,n)` / `aie::shuffle_down(v,n)` (+ `_rotate` / `_replicate` 변형) | shift lanes by `n` (zero/undef, rotate, or edge-replicate fill)<br>n만큼 차선 이동(0/비정의, 회전 또는 가장자리 복제 채우기) | any<br>어떤 | yes<br>응 | n/a | N | same as the `_fill` forms but with a fixed fill policy: `_rotate` wraps around, `_replicate` repeats the edge lane. Use when you don't need a specific pad value<br>`_fill` 형식과 동일하지만 고정된 채우기 정책이 있습니다. `_rotate`는 감싸고 `_replicate`는 가장자리 레인을 반복합니다. 특정 패드값이 필요하지 않을 때 사용 | `aie.hpp` (`shuffle_*`) / `detail/shuffle*.hpp`<br>`aie.hpp` (`shuffle_*`) / `detail/shuffle*.hpp` |

Notes:
참고:
- `mac_square` / `msc_square` are the matched pair for any
- `mac_square` / `msc_square`는 모든 항목에 대해 일치하는 쌍입니다.
mean/variance reduction (LayerNorm, RMSNorm, BatchNorm stats).
평균/분산 감소(LayerNorm, RMSNorm, BatchNorm 통계).
- `reduce_add_v` vs `reduce_add`: the `_v` form returns a vector (broadcast the
- `reduce_add_v` 대 `reduce_add`: `_v` 형식은 벡터를 반환합니다(방송
lane you want); the bare form returns a scalar and pulls you onto the scalar
원하는 차선); 베어 폼은 스칼라를 반환하고 스칼라 위로 끌어당깁니다.
unit.
단위.
- `filter_even`/`filter_odd` return a **half-width** vector (N -> N/2).
- `filter_even`/`filter_odd`는 **반폭** 벡터(N -> N/2)를 반환합니다.
- To keep a
- 유지하려면
full-width store, produce two halves and recombine, or size the loop so each
전체 너비 저장, 두 개의 반쪽을 생성하고 재결합하거나 루프 크기를 조정하여 각각
iteration consumes a full input vector and emits one half-width output vector
반복은 전체 입력 벡터를 소비하고 하나의 절반 너비 출력 벡터를 방출합니다.
(the natural shape for stride-2 pooling: 16 input lanes -> 8 outputs).
(스트라이드-2 풀링의 자연스러운 형태: 16개 입력 레인 -> 8개 출력)

---

## Complex Math functions implementation
## 복잡한 수학 함수 구현

To implement a complex mathematical function, prefer a LUT if using 8-bit inputs.
복잡한 수학 함수를 구현하려면 8비트 입력을 사용하는 경우 LUT를 선호합니다.
It will in general be faster and less expensive than using the floating-point hardware.
일반적으로 부동 소수점 하드웨어를 사용하는 것보다 더 빠르고 저렴합니다.

**Op-specific advice: ** for `exp` and `exp2`, pick your `exp` by the **output** dtype:
**작업 관련 조언: ** `exp` 및 `exp2`의 경우 **출력** dtype으로 `exp`를 선택하세요.

- **bf16 / fp32 output -> use HW `aie::exp2`.** Write `exp(x)` as
- **bf16 / fp32 출력 -> HW `aie::exp2` 사용.** `exp(x)`를 다음과 같이 씁니다.
`exp2(x*log2(e))`.
`exp2(x*log2(e))`.
It is one fast vector op.
하나의 빠른 벡터 연산입니다.
It is only ~bf16-accurate
그것은 단지 ~bf16-정확하다
(~12 bits), but that is good enough for a bf16/fp32 output.
(~12비트), 하지만 bf16/fp32 출력에는 충분합니다.
- **int8 output -> use an exact lookup table (LUT).** The int8 input has only
- **int8 출력 -> 정확한 조회 테이블(LUT)을 사용합니다.** int8 입력에는
**256 possible values**, so a table of `exp` is *exact*.
**256개의 가능한 값**이므로 `exp` 테이블은 *정확*합니다.
HW `exp2` is too
HW `exp2`도 마찬가지입니다.
rough here -- its small error flips int8 output bits.
대략적인 내용은 작은 오류가 int8 출력 비트를 뒤집는 것입니다.

> **Do NOT implement int8 `exp` with a polynomial / range-reduction approximation.**
> **다항식/범위 축소 근사를 사용하여 int8 `exp`를 구현하지 마세요.**

**For int8 input, how the LUT works:**
**int8 입력의 경우 LUT 작동 방식:**

- A full 256-entry fp32 table is 1 KB and does **not** fit the core's
- 전체 256개 항목 fp32 테이블은 1KB이며 코어에 맞지 **않습니다**
static-data budget.
정적 데이터 예산.
So store **two 16-entry tables** (128 B) and multiply them:
따라서 **2개의 16개 항목 테이블**(128B)을 저장하고 이를 곱합니다.
`exp(scale*q) = exp(hi part) * exp(lo part)`.
`exp(scale*q) = exp(hi 부분) * exp(lo 부분)`.
Align each table to
각 테이블을 다음과 같이 정렬합니다.
`alignas(aie::vector_decl_align)` (64 B) if vector-loaded.
벡터가 로드된 경우 `alignas(aie::Vector_decl_align)` (64 B).
- At runtime, build the full table **once into an oversized OFM ping/pong L1 buffer allocated by the tiler/ADF**
- 런타임 시 전체 테이블을 **타일러/ADF가 할당한 대형 OFM 핑/퐁 L1 버퍼에 한 번만 작성**
and look values up in the loop.
루프에서 값을 찾아보세요.
- Do the lookup with **`aie::parallel_lookup`** -- a vectorised HW gather,
- **`aie::parallel_lookup`**으로 조회를 수행합니다. -- 벡터화된 HW 수집입니다.
16 lanes/op.
16레인/op.
Its 4 parallel accesses need **4 interleaved copies across two DM
4개의 병렬 액세스에는 두 개의 DM에 걸쳐 **4개의 인터리브 복사본이 필요합니다.
banks**, so it only works when the table lives in the framework buffer above,
뱅크**이므로 테이블이 위의 프레임워크 버퍼에 있을 때만 작동합니다.
never in `.rodata`. **Even/odd bank layout:** each 64 B table line is really
결코 `.rodata`에는 없습니다. **짝수/홀수 뱅크 레이아웃:** 각 64B 테이블 라인은 실제로
two 32 B halves (an "even" half and an "odd" half), and `parallel_lookup`
두 개의 32 B 반쪽("짝수" 반쪽과 "홀수" 반쪽) 및 `병렬_조회`
reads both at once -- so the same values must sit in **both** halves.
두 가지를 동시에 읽습니다. 따라서 동일한 값이 **둘 다** 절반에 있어야 합니다.
Write
쓰기
each entry as **8 floats copied into both halves**
각 항목은 **8개의 부동 소수점으로 두 부분으로 복사됨**
(`concat(e.extract<8>(0), e.extract<8>(0))`).
(`concat(e.extract<8>(0), e.extract<8>(0))`).
If you copy in 4-float chunks
4-float 청크로 복사하는 경우
instead, lanes read the wrong half: it still compiles, but the board gives
대신, 레인은 잘못된 절반을 읽습니다. 여전히 컴파일되지만 보드는 다음을 제공합니다.
wrong answers.
잘못된 대답.
- Keep everything **fp32** in between (the exp values, the sum, the 1/sum).
- 그 사이에 모든 **fp32**를 유지하세요(exp 값, 합계, 1/합계).
Using bf16 here loses too much and flips output bits.
여기서 bf16을 사용하면 너무 많은 손실이 발생하고 출력 비트가 뒤집힙니다.

For int8, the LUT wins on **both** counts: it runs about as fast as HW `exp2`
int8의 경우 LUT는 **두 가지** 모두에서 승리합니다. HW `exp2`만큼 빠르게 실행됩니다.
(a plain table lookup, no transcendental in the loop) **and** it is exact, so it
(일반 테이블 조회, 루프에 초월 없음) **그리고** 정확하므로
is both **faster and more accurate** than the alternatives.
다른 대안보다 **빠르고 정확**합니다.

> Rule of thumb: **bf16 out -> `exp2`; int8 out -> exact LUT; never a polynomial
> 경험 법칙: **bf16 out -> `exp2`; int8 출력 -> 정확한 LUT; 결코 다항식은 아니다
> for int8** (slower than the LUT and buys no accuracy).
> int8**의 경우(LUT보다 느리고 정확도를 구매하지 않음)

---

## LUT table using aie::parallel_lookup
## aie::parallel_lookup을 사용하는 LUT 테이블

Use the following information about `aie::parallel_lookup`:
`aie::parallel_lookup`에 대해 다음 정보를 사용하세요.

- **Construct the lookup in the SAME function as `.fetch()`.** Its addressing
- **`.fetch()`와 같은 동일한 함수에서 조회를 구성합니다.** 주소 지정
state lives in registers and does NOT survive being passed across a (noinline)
상태는 레지스터에 존재하며 (noinline)을 통과해도 살아남지 않습니다.
call boundary.
통화 경계.
Rebuilding it at the fetch site is free.
가져오기 사이트에서 재구축하는 것은 무료입니다.
- **Build the table in a NOINLINE helper called from the wrapper, not the hot
- **핫이 아닌 래퍼에서 호출되는 NOINLINE 도우미에서 테이블을 구축합니다.
core.**
코어.**
- **Build once per physical buffer.** The L1 tail is double-buffered and never
- **물리적 버퍼당 한 번 빌드됩니다.** L1 테일은 이중 버퍼링되며 절대로 빌드되지 않습니다.
DMA-drained: cache the (up to 2) ping/pong addresses and skip the rebuild on
DMA 배수: ping/pong 주소(최대 2개)를 캐시하고 재구축을 건너뜁니다.
later invocations.
나중에 호출합니다.

**It's memory-heavy -- decide if you need it.** `parallel_lookup<4>` keeps 4
**메모리가 많이 듭니다. 필요한지 결정하세요.** `parallel_lookup<4>`은 4를 유지합니다.
bank-interleaved copies of the table, and double-buffering doubles that again
뱅크 인터리브된 테이블 복사본과 이중 버퍼링으로 다시 두 배로 늘어납니다.
(vs a single flat copy for a scalar-gather table).
(스칼라 수집 테이블의 단일 플랫 복사본과 비교)
It only pays when the lookup
조회할 때만 비용이 청구됩니다.
is the hot op (gather-bound, e.g. channel-vectorized).
핫 연산(수집 바인딩, 예: 채널 벡터화)입니다.
If you vectorize the other
다른 것을 벡터화하면
axis (positions) so sum/reciprocal/normalize are full-width vector work, a scalar
축(위치)이므로 합계/역수/정규화는 전폭 벡터 작업, 스칼라입니다.
gather hides behind that arithmetic -- use it and save memory.
그 산술 뒤에 숨겨진 것을 모으십시오 - 그것을 사용하고 메모리를 절약하십시오.

---

## Matrix-multiply (GEMM) with `aie::mmul`
## `aie::mmul`을 사용한 행렬 곱셈(GEMM)

The one **non-vector** primitive: `aie::mmul<M,K,N,TA,TB>` contracts an `MxK`
하나의 **비벡터** 프리미티브: `aie::mmul<M,K,N,TA,TB>`는 `MxK`를 축소합니다.
A-tile with a `KxN` B-tile into an `MxN` accumulator via the hardware matrix
하드웨어 매트릭스를 통해 'KxN' B 타일이 있는 A 타일을 'MxN' 누산기로 변환
mode.
모드.
`bf16 x bf16 -> fp32` runs **8x8x8** natively (512 MACs/issue, ~10x a
`bf16 x bf16 -> fp32`는 기본적으로 **8x8x8**을 실행합니다(512 MAC/문제, ~10x a)
lane-wise `aie::mac`).
차선별 `aie::mac`).
Use it for matmul / linear / 1x1-conv; skip it when the
matmul/linear/1x1-conv에 사용하세요. 그럴 때는 건너뛰세요.
reduction is tiny (`K < 8`, e.g. depthwise), shapes force heavy 8-padding, or
감소가 작거나(`K < 8`, 예: 깊이 기준) 모양에 무거운 8-패딩이 적용되거나
the op isn't matmul-shaped.
op는 matmul 모양이 아닙니다.
Full worked kernels + tiling shipped with the
완전히 작동된 커널 + 타일링이 함께 제공됨
`vai-custom-op-implementation` skill -- the gemm tutorial (bf16 bias-folded
`vai-custom-op-implementation` 스킬 -- gemm 튜토리얼(bf16 bias-folded
GEMM = 1x1 conv) and the conv2d tutorial
GEMM = 1x1 전환) 및 conv2d 튜토리얼
`conv3x3_int8_2stamp` (int8 spatial conv: halo, sliding window, multi-call depth
`conv3x3_int8_2stamp`(int8 공간 변환: 후광, 슬라이딩 윈도우, 다중 호출 깊이
reduction, requant).
감소, 재수량).

| API<br>API | math<br>수학 | notes<br>메모 | header<br>헤더 |
|---|---|---|---|
| `aie::mmul<M,K,N,TA,TB>`<br>`aie::mmul<M,K,N,TA,TB>` | `MxN` tile accumulator contracting over `K`<br>'K'에 비해 축소되는 'MxN' 타일 누산기 | bf16xbf16->fp32 / int combos (fp32 matrix mode limited); A=`size_A=M*K` `a[mi*K+ki]`, B=`size_B=K*N` `b[ki*N+ni]`; holds 2 cml regs so ~4 live tiles fill the file<br>bf16xbf16->fp32 / int 콤보(fp32 매트릭스 모드 제한); A=`size_A=M*K` `a[mi*K+ki]`, B=`size_B=K*N` `b[ki*N+ni]`; 2 cml reg를 보유하므로 ~4개의 라이브 타일이 파일을 채웁니다. | `detail/*/mmul*.hpp`<br>`세부정보/*/mmul*.hpp` |
| `C.mul(a,b)` / `C.mac(a,b)`<br>`C.mul(a,b)` / `C.mac(a,b)` | `C = a.b` (seed) / `C += a.b`<br>`C = a.b`(시드) / `C += a.b` | operands are `aie::vector` of `size_A`/`size_B`; `mul` on the first reduction step, `mac` after<br>피연산자는 `size_A`/`size_B`의 `aie::벡터`입니다. 첫 번째 감소 단계에서는 'mul', 그 이후에는 'mac' | same<br>같은 |
| `C.to_accum()` / `C.to_vector<T>()`<br>`C.to_accum()` / `C.to_Vector<T>()` | read `MxN` result<br>`MxN` 결과 읽기 | narrow for the store; `res.extract<N>(mi)` pulls output row `mi`<br>가게에 비해 좁다. `res.extract<N>(mi)`는 출력 행 `mi`를 가져옵니다. | same<br>같은 |

Idioms (see the tutorial kernel): pre-block operands to contiguous 8x8 tiles on
관용어(튜토리얼 커널 참조): 연속된 8x8 타일에 대한 사전 블록 피연산자
the L2->L1 DMA (one `load_v` each); cap live accumulators at ~4 and loop
L2->L1 DMA(각각 하나의 'load_v') 라이브 누산기를 ~4로 제한하고 루프
sub-blocks rather than growing them; roll the reduction loop
하위 블록을 성장시키는 대신 감소 루프를 굴려 라.
(`#pragma clang loop unroll(disable)`) to keep operand loads out of the stack;
(`#pragma clang loop unroll(disable)`) 피연산자 로드를 스택에서 유지합니다.
fold bias as one extra reduction tile `mac`'d against a constant-`1.0` operand.
상수 '1.0' 피연산자에 대해 하나의 추가 축소 타일 'mac'로 바이어스를 접습니다.

---

## Idiomatic / fused choices
## 관용적/융합된 선택

If you find yourself coding the right column, stop and use the left:
오른쪽 열을 코딩하는 경우 중지하고 왼쪽을 사용하세요.

| Use this<br>이것을 사용하십시오 | instead of<br>대신에 | gain<br>이득 |
|---|---|---|
| `aie::mac_square(acc, v)`<br>`aie::mac_square(acc, v)` | `aie::mac(acc, v, v)`<br>`aie::mac(acc, v, v)` | idiomatic; one square op, tighter chain<br>관용적인; 하나의 사각형 op, 더 단단한 체인 |
| `aie::msc_square(Ex2, mean)`<br>`aie::msc_square(Ex2, 평균)` | scalar `sumsq*inv - mean*mean`<br>스칼라 `sumsq*inv - 평균*평균` | variance stays vectorized & fused<br>분산은 벡터화 및 융합된 상태로 유지됩니다. |
| `aie::reduce_add_v(...)`<br>`aie::reduce_add_v(...)` | `aie::reduce_add(...)` + scalar math<br>`aie::reduce_add(...)` + 스칼라 수학 | avoids scalar-FPU serialization<br>스칼라 FPU 직렬화 방지 |
| `vec.set(reduce_add(v), r)` / `vec[r] = reduce_add(v)` (row->lane for batched `invsqrt`)<br>`vec.set(reduce_add(v), r)` / `vec[r] = Reduce_add(v)` (일괄 `invsqrt`에 대한 행->레인) | scratch array `buf[r]=...` then reload<br>스크래치 배열 `buf[r]=...` 후 다시 로드 | keeps per-row stats in the FPU; no L1/stack round-trip (there is no free `aie::insert`; `vector::insert` is sub-vector-at-partition, not scalar-at-lane)<br>FPU에 행별 통계를 유지합니다. L1/스택 왕복 없음(무료 `aie::insert`가 없습니다. `벡터::삽입`은 스칼라 앳 레인이 아닌 파티션 하위 벡터입니다.) |
| `aie::invsqrt(v)`<br>`aie::invsqrt(v)` | `1.0f / aie::sqrt(v)` or Newton<br>`1.0f / aie::sqrt(v)` 또는 뉴턴 | single HW op, no divide<br>단일 HW 운영, 분할 없음 |
| `aie::max(var_eps, eps)`<br>`aie::max(var_eps, eps)` | `if (var < eps) var = eps;`<br>`if (var <eps) var = eps;` | branchless, vectorized stability clamp<br>분기 없는 벡터화된 안정성 클램프 |
| `aie::filter_even(v)` / `aie::filter_odd(v)`<br>`aie::filter_even(v)` / `aie::filter_odd(v)` | scalar strided-index copy for stride-2 output<br>stride-2 출력을 위한 스칼라 strided-index 복사 | one `vshuffle` de-interleaves 16 lanes vs 8 scalar loads/stores<br>하나의 `vshuffle`은 16개 레인 대 8개의 스칼라 로드/저장을 디인터리브합니다. |
| `aie::store_v(out, vec)` of an assembled result<br>조립된 결과의 `aie::store_v(out, vec)` | per-element scalar `out[i]=...` bf16 stores<br>요소별 스칼라 `out[i]=...` bf16 저장 | avoids the sub-32-bit (bf16) store read-modify-write penalty<br>하위 32비트(bf16) 저장 읽기-수정-쓰기 페널티를 방지합니다. |
| `aie::shuffle_up_fill(od, pad, 1)` for a halo tap<br>후광 탭의 경우 `aie::shuffle_up_fill(od, pad, 1)` | `aie::load_unaligned_v` at a `-1` offset per iteration<br>반복당 `-1` 오프셋에서 `aie::load_unaligned_v` | keeps the base aligned; no loop-carried unaligned load<br>베이스 정렬을 유지합니다. 루프 운반 비정렬 부하 없음 |

---

## Windowed / strided de-interleave example
## 윈도우/스트라이드 디인터리브 예시

**Key insight:** an odd window (k=3) with an even stride (s=2) makes the output
**주요 통찰력:** 짝수 스트라이드(s=2)가 있는 홀수 창(k=3)이 출력을 만듭니다.
straddle even and odd input lanes, so a stride-2 pool
짝수 및 홀수 입력 레인에 걸쳐 있으므로 스트라이드 2 풀
(`out[wo] = max(A[2*wo-1], A[2*wo], A[2*wo+1])`, `A` = vertically-maxed row) is a
(`out[wo] = max(A[2*wo-1], A[2*wo], A[2*wo+1])`, `A` = 수직 최대 행)은
**de-interleave**: `filter_even`/`filter_odd` split the taps and a shifted load
**디인터리브**: `filter_even`/`filter_odd`는 탭과 이동된 로드를 분할합니다.
supplies the third.
세 번째를 공급합니다.

```cpp
static constexpr int VEC = 16;                    // bf16 lanes
// `A` is a 32B-aligned L1 scratch with a left pad (A[-1]=LOWEST, pad >= VEC so
// `A-2` stays vector-aligned). Emit 8 outputs/iter, no scalar tail.
aie::vector<bfloat16, VEC> v  = aie::load_v<VEC>(A + b * VEC);
aie::vector<bfloat16, VEC> vm = aie::load_unaligned_v<VEC>(A + b * VEC - 2);
auto o = aie::max(aie::max(aie::filter_even(v),   // A[2*wo]   center tap
                           aie::filter_odd(v)),   // A[2*wo+1] right tap
                           aie::filter_odd(vm));  // A[2*wo-1] left tap (8-wide)
aie::store_v(out + b * (VEC / 2), o);
```

To avoid the per-iteration unaligned load, derive the left tap from lanes you
반복당 정렬되지 않은 로드를 방지하려면 다음 차선에서 왼쪽 탭을 파생시킵니다.
already have with `aie::shuffle_up_fill(od, prev_od_or_pad, 1)` (costs a
이미 `aie::shuffle_up_fill(od, prev_od_or_pad, 1)`을 사용하고 있습니다(비용은
loop-carried value).
루프 전달 값).
Confirm the choice from the loop's initiation interval in
루프의 시작 간격에서 선택을 확인합니다.
the `.lst`, and only invest here if the op is compute-bound -- large-IFM pooling
`.lst`, 작업이 컴퓨팅에 국한된 경우에만 여기에 투자합니다 -- 대규모 IFM 풀링
is usually DMA-dominated.
일반적으로 DMA가 지배적입니다.

---

## If it's not here: discover from the headers
## 여기에 없으면 헤더에서 검색하세요.

This catalog is not exhaustive.
이 카탈로그는 완전한 것이 아닙니다.
To confirm a primitive exists, see its full
기본 요소가 존재하는지 확인하려면 전체 내용을 참조하세요.
overload set, and pick the right accumulator/return type, read the **active
오버로드를 설정하고 올바른 누산기/반환 유형을 선택하고 **active를 읽으세요.
environment's** `aie_api` headers.
환경** `aie_api` 헤더.

**Resolve the header directory first.** The headers ship inside the active Python
**먼저 헤더 디렉터리를 확인하세요.** 헤더는 활성 Python 내부에 제공됩니다.
environment, under `<site-packages>/include/aie_api/` (`sysconfig`'s `purelib`
환경, `<site-packages>/include/aie_api/`(`sysconfig`의 `purelib`
gives you site-packages).
사이트 패키지를 제공합니다).
Do **not** hardcode an absolute path -- it moves per
절대 경로를 하드코딩하지 **마세요**.
release/host.
릴리스/호스트.
Resolve it once from the environment in effect for this run, keep
이 실행에 적용되는 환경에서 한 번 해결하고 유지합니다.
it in `$AIE_API_DIR`, and reuse it for every lookup.
`$AIE_API_DIR`에 저장하고 모든 조회에 재사용하세요.

**Scope every search to `$AIE_API_DIR`.** Never `find` or `grep -r` from the env
**모든 검색 범위를 `$AIE_API_DIR`로 지정하세요.** 절대로 환경에서 `find` 또는 `grep -r`을 사용하지 마세요.
root or its prefix: that tree is large enough that the search stalls and gets
루트 또는 해당 접두사: 해당 트리는 검색이 중단되고 검색이 중단될 만큼 충분히 큽니다.
backgrounded or aborted.
배경이 지정되었거나 중단되었습니다.
In the unlikely case that site-packages does not
사이트 패키지가 그렇지 않은 경우는 거의 없습니다.
resolve, a **depth-capped** `find` under the env prefix (`sys.prefix`) is the
해결, env 접두사(`sys.prefix`) 아래의 **깊이 제한** `find`는
only acceptable fallback.
허용되는 대체만 가능합니다.

**Where to read what:**
**읽을 내용:**

- `aie.hpp` -- the public entry header (declarations + doc comments); pulls in
- `aie.hpp` -- 공개 항목 헤더(선언 + 문서 주석); 끌어당긴다
vectors, accumulators, and the free-function API (`aie::add`, `aie::mul`,
벡터, 누산기 및 자유 함수 API(`aie::add`, `aie::mul`,
`aie::mac_square`, `aie::reduce_add`, ...).
`aie::mac_square`, `aie::reduce_add`, ...).
Grep a primitive name here to get
여기에서 기본 이름을 Grep하여 가져오세요.
its overload set and doc block.
오버로드 세트 및 문서 블록.
- `vector.hpp`, `accum.hpp` -- the `aie::vector<T, N>` / `aie::accum<Acc, N>`
- `벡터.hpp`, `accum.hpp` -- `aie::벡터<T, N>` / `aie::accum<Acc, N>`
storage types; check which `(T, N)` pairs the target arch supports.
저장 유형; 타겟 아치가 지원하는 `(T, N)` 쌍을 확인하세요.
- `mmul.hpp`, `sliding_mul.hpp` -- matrix multiply and sliding-window multiplies
- `mmul.hpp`, `sliding_mul.hpp` -- 행렬 곱셈 및 슬라이딩 윈도우 곱셈
(the right primitives for conv / matmul kernels).
(conv / matmul 커널에 적합한 기본 요소).
- `operators.hpp`, `utils.hpp` -- overloaded operators and helpers (broadcast,
- `operators.hpp`, `utils.hpp` -- 오버로드된 연산자 및 도우미(브로드캐스트,
zeros, iterators).
0, 반복자).
- `detail/` -- the implementations, one header per op class (`add.hpp`, `mul.hpp`,
- `detail/` -- 구현, 작업 클래스당 하나의 헤더(`add.hpp`, `mul.hpp`,
`max_min.hpp`, `neg.hpp`, `lut.hpp`, `linear_approx.hpp`, `elementary.hpp`,
`max_min.hpp`, `neg.hpp`, `lut.hpp`, `linear_about.hpp`, `elementary.hpp`,
...; `ls` it to see which op families ship at all), with architecture-specific
...; 어떤 op 제품군이 출시되는지 확인하려면 'ls'를 사용하세요), 아키텍처별
variants under `detail/aie2p/`, `detail/aie2ps/`, etc.
`detail/aie2p/`, `detail/aie2ps/` 등의 변형
Read it to see what a
읽어보면 어떤 내용인지 알 수 있다
call lowers to and where a transcendental is specialized per arch; never
초월이 아치별로 특화된 곳으로 하강을 호출합니다. 결코
`#include` it directly.
직접 `#include`하세요.

```bash
grep -nE "mac_square|msc_square|reduce_add_v|invsqrt" "$AIE_API_DIR/aie.hpp"
ls "$AIE_API_DIR/detail"                 # which op classes exist
# Where is it defined / specialized per arch? Recursive grep, scoped.
grep -rln "tanh" "$AIE_API_DIR"          # -> detail/elementary.hpp, detail/aie2p/elementary.hpp
```

Write the call from what the header declares (exact name, vector vs accum
헤더가 선언한 것(정확한 이름, 벡터 대 누적)에서 호출을 작성합니다.
argument types, return type).
인수 유형, 반환 유형).
If a primitive is missing, **compose it from
기본 요소가 누락된 경우 **에서 구성합니다.
existing ops** (e.g. sigmoid from `tanh`, `mac` chains, `aie::inv` /
기존 ops** (예: `tanh`, `mac` 체인, `aie::inv`의 시그모이드 /
`aie::invsqrt`) before falling back to a scalar `std::`-style loop.
`aie::invsqrt`) 스칼라 `std::` 스타일 루프로 되돌아가기 전에.
