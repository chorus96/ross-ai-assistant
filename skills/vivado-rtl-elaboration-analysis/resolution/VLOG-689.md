<!-- ko-translation: enabled -->
# VLOG-689: Port Width Mismatch
# VLOG-689: 포트 너비 불일치

## Message Format
## 메시지 형식

```
WARNING: [Synth 8-689] width (N) of port connection '<port>' does not match port width (M) of module '<module>' [file:line]
```

## Root Cause
## 근본 원인

The signal connected to a port has a different bit-width than the port expects.
포트에 연결된 신호의 비트 폭은 포트에서 예상하는 것과 다릅니다.
This is commonly caused by:
이는 일반적으로 다음으로 인해 발생합니다.

1. **Hardcoded signal dimensions** in a parameterized module — signal uses `[31:0]`
1. 매개변수화된 모듈의 **하드코딩된 신호 크기** — 신호는 `[31:0]`을 사용합니다.
instead of `[PARAM-1:0]`
`[PARAM-1:0]` 대신
2. **Parameter override at instantiation** — parent passes a smaller value than
2. **인스턴스화 시 매개변수 재정의** — 상위 항목이 다음보다 작은 값을 전달합니다.
the default, making the submodule port narrower than the connecting signal
기본적으로 서브모듈 포트를 연결 신호보다 좁게 만듭니다.
3. **Typed unpacked arrays** — signal is declared as `typedef [N:0] array` but
3. **압축되지 않은 형식화된 배열** — 신호는 `typedef [N:0] 배열`로 선언되지만
submodule expects fewer elements
하위 모듈에는 더 적은 수의 요소가 필요합니다.

## Critical Analysis Steps
## 중요 분석 단계

**MANDATORY: Trace parameters from the top-level instantiation downward.**
**필수: 최상위 인스턴스화부터 아래쪽으로 매개변수를 추적합니다.**

Do NOT assume parameter defaults are the elaborated values.
매개변수 기본값이 정교한 값이라고 가정하지 마십시오.
The actual width depends
실제 너비는 다음과 같습니다.
on the full hierarchy:
전체 계층 구조에서:

1. **Find the instantiation** of the reported module in the file at the reported line
1. 보고된 줄의 파일에서 보고된 모듈의 **인스턴스화**를 찾습니다.
2. **Read the parameter overrides** at instantiation (`.PARAM(value)`)
2. **인스턴스화 시 매개변수 재정의** 읽기(`.PARAM(value)`)
3. **Trace `value` upward** — is it a parameter of the parent?
3. **'값'을 위쪽으로 추적** - 상위 항목의 매개변수인가요?
3. What does the
3. 무엇을 하는가?
grandparent pass?
조부모님 패스?
4. **Compute the actual port width** using the traced parameter value
4. **추적된 매개변수 값을 사용하여 실제 포트 너비를 계산**
5. **Read the signal declaration** at the connection site
5. **연결 지점에서 신호 선언을 읽으세요**
6. **If the signal uses a typedef**, find the typedef in the package and **count
6. **신호가 typedef를 사용하는 경우** 패키지에서 typedef를 찾고 **count
the actual bits** (sum all fields in a `struct packed`).
실제 비트**(`struct Packed`의 모든 필드를 합산함).
Do NOT assume the
가정하지 마십시오
width from Vivado's numbers alone — verify by reading the source.
Vivado의 숫자만으로 너비 — 소스를 읽어 확인하십시오.
7. **Check all consumers** of the signal — search for all occurrences in the module
7. **신호의 모든 소비자 확인** — 모듈에서 발생하는 모든 항목을 검색합니다.
to see if other instantiations or assignments use the full range
다른 인스턴스화 또는 할당이 전체 범위를 사용하는지 확인
8. **Compare** actual signal width vs actual port width
8. **비교** 실제 신호 폭과 실제 포트 폭

### Example: Parameter Tracing
### 예: 매개변수 추적

```
ethernet_comms (HDLC_NO_OF_CHANNELS = 16)   ← top sets 16
  └─ ethernet_hdlc_top (HDLC_NO_OF_CHANNELS = 16)  ← inherited
       ├─ signal: ohif_sel [31:0]  ← HARDCODED 32 elements = 512 bits
       │    └─ typedef: eth_hdlc_ohif_sel_tdef (struct packed, 16 bits) ← VERIFY in pkg
       └─ hdlc_decoder_top_v4 (NB_CHANNELS = 16)
            └─ port: iv_ohif_sel [NB_CHANNELS-1:0][15:0] = 16×16 = 256 bits
```

Vivado reports: "width (512) does not match port width (256)" — **correct warning**.
Vivado 보고: "너비(512)가 포트 너비(256)와 일치하지 않습니다." — **올바른 경고**.

## Common Pitfall: Typed Unpacked Arrays
## 일반적인 함정: 형식화된 압축되지 않은 배열

When a signal is declared with a typedef:
typedef를 사용하여 신호를 선언하는 경우:
```systemverilog
my_typedef [31:0] my_signal;   // 32-element unpacked array
```

You **cannot** use bit-slicing like `my_signal[255:0]`.
`my_signal[255:0]`과 같은 비트 슬라이싱을 사용할 수 **없습니다**.
This is an unpacked array,
이것은 압축이 풀린 배열입니다.
not a flat bit vector.
플랫 비트 벡터가 아닙니다.
Valid operations:
유효한 작업:
- Array slice: `my_signal[15:0]` (elements 0–15)
- 배열 슬라이스: `my_signal[15:0]`(요소 0-15)
- Single element: `my_signal[0]`
- 단일 요소: `my_signal[0]`

## Fix Strategy
## 전략 수정

### Case 1: Signal shared with other consumers (most common in practice)
### 사례 1: 다른 소비자와 공유되는 신호(실제로 가장 일반적임)

Before changing the declaration, **check all consumers** of the signal.
선언을 변경하기 전에 신호의 **모든 소비자**를 확인하세요.
If another
다른 경우
module (e.g., MPI register interface) connects to all 32 elements individually,
모듈(예: MPI 레지스터 인터페이스)은 32개 요소 모두에 개별적으로 연결됩니다.
the declaration MUST stay `[31:0]`.
선언은 `[31:0]`을 유지해야 합니다.
Fix at the port connection using an array slice:
어레이 슬라이스를 사용하여 포트 연결을 수정합니다.

```diff
- .iv_ohif_sel  (ohif_sel),
+ .iv_ohif_sel  (ohif_sel[HDLC_NO_OF_CHANNELS-1:0]),
```

**Real-world example:** `ethernet_hdlc_mpi` connects `ohif_sel[0]`..`ohif_sel[31]`
**실제 예:** `ethernet_hdlc_mpi`는 `ohif_sel[0]`..`ohif_sel[31]`을 연결합니다.
individually.
개별적으로.
The decoder/encoder only needs channels 0–15.
디코더/인코더에는 채널 0~15만 필요합니다.
Shrinking the array
배열 축소
would break MPI.
MPI가 깨질 수 있습니다.
The fix is an array slice at the decoder/encoder ports only.
수정 사항은 디코더/인코더 포트에만 있는 배열 슬라이스입니다.

### Case 2: Signal only used by the mismatched port
### 사례 2: 일치하지 않는 포트에서만 사용되는 신호

If the signal has NO other consumers that need the full range, parameterize
신호에 전체 범위가 필요한 다른 소비자가 없는 경우 매개변수화
the declaration:
선언:

```diff
- eth_hdlc_ohif_sel_tdef  [31:0]  ohif_sel;
+ eth_hdlc_ohif_sel_tdef  [HDLC_NO_OF_CHANNELS-1:0]  ohif_sel;
```

**IMPORTANT:** Only do this after confirming no other module/port needs the full
**중요:** 전체 모듈/포트가 필요하지 않다는 것을 확인한 후에만 이 작업을 수행하십시오.
range.
범위.
Check all instances of the signal name in the module.
모듈에서 신호 이름의 모든 인스턴스를 확인하십시오.

### Case 3: Signal too narrow (widen it)
### 사례 3: 신호가 너무 좁음(확대)

```diff
- wire [15:0] fill_level;    // 16 bits
+ wire [16:0] fill_level;    // 17 bits to match FIFO port
```

## Verification
## 검증

After applying fix:
수정 사항을 적용한 후:
1. Re-run elaboration (`synth_design -rtl`)
1. 정교화 재실행(`synth_design -rtl`)
2. Confirm the [Synth 8-689] warning is gone for this port
2. 이 포트에 대한 [Synth 8-689] 경고가 사라졌는지 확인하세요.
3. Check no new errors introduced (e.g., if other connections to the same signal
3. 새로운 오류가 발생하지 않았는지 확인하십시오(예: 동일한 신호에 다른 연결이 있는 경우).
now have a mismatch)
이제 불일치가 발생했습니다)

## Related Messages
## 관련 메시지

- [Synth 8-514] — Similar port width mismatch (positional connections)
- [Synth 8-514] - 유사한 포트 너비 불일치(위치 연결)
- [Synth 8-7129] — Port unconnected or has no load (may appear after width fix
- [Synth 8-7129] — 포트가 연결되지 않았거나 로드가 없습니다(너비 수정 후에 나타날 수 있음).
if upper bits were the only load)
상위 비트가 유일한 로드인 경우)
