<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# ILA Debug — Reference Data
# ILA 디버그 — 참조 데이터

## ILA CONTROL Properties (UG912 — HW_ILA)
## ILA 제어 속성(UG912 — HW_ILA)

| Property<br>재산 | Type<br>유형 | Values<br>가치 | Description<br>설명 |
|----------|------|--------|-------------|
| CONTROL.DATA_DEPTH<br>제어.DATA_DEPTH | int<br>정수 | 1–MAX_DATA_DEPTH (power of 2)<br>1–MAX_DATA_DEPTH(2의 거듭제곱) | Samples per window. DATA_DEPTH × WINDOW_COUNT = MAX_DATA_DEPTH<br>창당 샘플. DATA_DEPTH × WINDOW_COUNT = MAX_DATA_DEPTH |
| CONTROL.WINDOW_COUNT<br>CONTROL.WINDOW_COUNT | int<br>정수 | 1–N | Number of capture windows (multiple trigger events)<br>캡처 창 수(여러 트리거 이벤트) |
| CONTROL.TRIGGER_POSITION<br>제어.TRIGGER_POSITION | int<br>정수 | 0–(DATA_DEPTH-1)<br>0–(DATA_DEPTH-1) | Position of trigger mark. 0=first sample, 512=mid for 1024 depth<br>방아쇠 표시의 위치. 0=첫 번째 샘플, 512=깊이 1024의 경우 중간 |
| CONTROL.TRIGGER_MODE<br>Control.TRIGGER_MODE | enum<br>열거형 | BASIC_ONLY, BASIC_OR_TRIG_IN, ADVANCED_ONLY, ADVANCED_OR_TRIG_IN, TRIG_IN_ONLY<br>BASIC_ONLY, BASIC_OR_TRIG_IN, ADVANCED_ONLY, ADVANCED_OR_TRIG_IN, TRIG_IN_ONLY | Trigger source<br>트리거 소스 |
| CONTROL.TRIGGER_CONDITION<br>Control.TRIGGER_CONDITION | enum<br>열거형 | AND, NAND, OR, NOR<br>AND, NAND, OR, NOR | Boolean equation across participating probes<br>참여 프로브 전체의 부울 방정식 |
| CONTROL.CAPTURE_MODE<br>CONTROL.CAPTURE_MODE | enum<br>열거형 | ALWAYS, BASIC<br>항상, 기본 | ALWAYS=every cycle, BASIC=only when capture condition true<br>ALWAYS=모든 사이클, BASIC=캡처 조건이 true인 경우에만 |
| CONTROL.CAPTURE_CONDITION<br>Control.CAPTURE_CONDITION | enum<br>열거형 | AND, NAND, OR, NOR<br>AND, NAND, OR, NOR | Boolean for capture qualification<br>캡처 자격에 대한 부울 |
| CONTROL.TSM_FILE<br>CONTROL.TSM_FILE | string<br>문자열 | file path<br>파일 경로 | Path to Trigger State Machine (.tsm) file<br>트리거 상태 시스템(.tsm) 파일의 경로 |
| CONTROL.TRIG_OUT_MODE<br>CONTROL.TRIG_OUT_MODE | enum<br>열거형 | DISABLED, TRIGGER_ONLY, TRIG_IN_ONLY, TRIGGER_OR_TRIG_IN<br>비활성화됨, TRIGGER_ONLY, TRIG_IN_ONLY, TRIGGER_OR_TRIG_IN | TRIG_OUT port behavior<br>TRIG_OUT 포트 동작 |

## ILA STATIC Properties (read-only)
## ILA STATIC 속성(읽기 전용)

| Property<br>재산 | Type<br>유형 | Description<br>설명 |
|----------|------|-------------|
| STATIC.MAX_DATA_DEPTH<br>정적.MAX_DATA_DEPTH | int<br>정수 | Maximum samples (set at design time)<br>최대 샘플(디자인 타임에 설정) |
| STATIC.IS_ADVANCED_TRIGGER_MODE_SUPPORTED<br>STATIC.IS_ADVANCED_TRIGGER_MODE_SUPPORTED | bool<br>부울 | Advanced TSM available?<br>고급 TSM을 사용할 수 있습니까? |
| STATIC.IS_BASIC_CAPTURE_MODE_SUPPORTED<br>STATIC.IS_BASIC_CAPTURE_MODE_SUPPORTED | bool<br>부울 | Storage qualifier available?<br>스토리지 한정자를 사용할 수 있습니까? |
| STATIC.IS_TRIG_IN_SUPPORTED<br>STATIC.IS_TRIG_IN_SUPPORTED | bool<br>부울 | TRIG_IN port present?<br>TRIG_IN 포트가 있습니까? |
| STATIC.IS_TRIG_OUT_SUPPORTED<br>STATIC.IS_TRIG_OUT_SUPPORTED | bool<br>부울 | TRIG_OUT port present?<br>TRIG_OUT 포트가 있습니까? |
| STATIC.TSM_COUNTER_0_WIDTH–3_WIDTH<br>STATIC.TSM_COUNTER_0_WIDTH–3_WIDTH | int<br>정수 | TSM counter bit widths<br>TSM 카운터 비트 폭 |

## ILA STATUS Properties (read-only)
## ILA STATUS 속성(읽기 전용)

| Property<br>재산 | Type<br>유형 | Description<br>설명 |
|----------|------|-------------|
| STATUS.CORE_STATUS<br>상태.CORE_상태 | string<br>문자열 | IDLE, ARMED, WAITING_FOR_TRIGGER, TRIGGER_CAPTURED, FULL<br>유휴, ARMED, WAITING_FOR_TRIGGER, TRIGGER_CAPTURED, 전체 |
| STATUS.SAMPLE_COUNT<br>상태.SAMPLE_COUNT | int<br>정수 | Number of samples captured so far<br>지금까지 캡처된 샘플 수 |
| STATUS.IS_TRIGGER_AT_STARTUP<br>상태.IS_TRIGGER_AT_STARTUP | bool<br>부울 | Trigger-at-startup configured?<br>시작 시 트리거가 구성되었습니까? |

## ILA Probe Properties (HW_PROBE, TYPE == ila)
## ILA 프로브 속성(HW_PROBE, TYPE == ila)

| Property<br>재산 | Type<br>유형 | R/W | Description<br>설명 |
|----------|------|-----|-------------|
| TRIGGER_COMPARE_VALUE<br>TRIGGER_COMPARE_VALUE | string<br>문자열 | R/W | Trigger match pattern (see encoding below)<br>트리거 일치 패턴(아래 인코딩 참조) |
| CAPTURE_COMPARE_VALUE<br>CAPTURE_COMPARE_VALUE | string<br>문자열 | R/W | Capture qual match pattern<br>자격 일치 패턴 캡처 |
| COMPARATOR_COUNT<br>COMPARATOR_COUNT | int<br>정수 | R/O | Number of match units on this probe<br>이 프로브의 일치 단위 수 |
| PROBE_PORT<br>PROBE_PORT | int<br>정수 | R/O | Physical probe port number<br>물리적 프로브 포트 번호 |
| PROBE_PORT_BIT_COUNT<br>PROBE_PORT_BIT_COUNT | int<br>정수 | R/O | Bit width<br>비트 폭 |
| NAME<br>이름 | string<br>문자열 | R/O | HDL net name (from .ltx)<br>HDL 네트 이름(.ltx에서) |

## Compare Value Encoding
## 값 인코딩 비교

Format: `<operator><width>'<radix><value>`
형식: `<연산자><너비>'<기수><값>`

**Operators:**
**운영자:**

| Operator<br>운영자 | Meaning<br>의미 |
|----------|---------|
| `eq` | Equal (=)<br>같음(=) |
| `neq`<br>'네크' | Not equal (!=)<br>같지 않음(!=) |
| `lt` | Less than (<)<br>미만(<) |
| `lteq`<br>`lteq` | Less than or equal (≤)<br>작거나 같음(≤) |
| `gt` | Greater than (>)<br>보다 큼(>) |
| `gteq`<br>`gteq` | Greater than or equal (≥)<br>크거나 같음(≥) |

**Radix:** `b` (binary), `h` (hex), `o` (octal), `u` (unsigned), `s` (signed)
**기수:** `b`(2진수), `h`(16진수), `o`(8진수), `u`(부호 없음), `s`(부호 있음)

**Special bit values (binary only):**
**특수 비트 값(바이너리에만 해당):**

| Char<br>숯 | Meaning<br>의미 |
|------|---------|
| `X` | Don't care — probe does NOT participate in condition<br>상관없음 - 프로브가 조건에 참여하지 않음 |
| `R` | Rising edge (0→1)<br>상승 에지(0→1) |
| `F` | Falling edge (1→0)<br>하강에지(1→0) |
| `B` | Both edges (any transition)<br>양쪽 가장자리(모든 전환) |
| `N` | No transition (stable)<br>전환 없음(안정) |
| `L` | Opposite of R<br>R의 반대 |
| `S` | Opposite of F<br>F의 반대 |

**Examples:**
**예:**
- `eq1'b1` — probe equals 1
- `eq1'b1` — 프로브는 1과 같습니다.
- `eq4'hA` — 4-bit probe equals 0xA
- `eq4'hA` — 4비트 프로브는 0xA와 같습니다.
- `eq1'bR` — rising edge on 1-bit probe
- `eq1'bR` — 1비트 프로브의 상승 에지
- `neq8'hFF` — 8-bit probe not equal to 0xFF
- `neq8'hFF` — 0xFF와 같지 않은 8비트 프로브
- `gt16'u1000` — 16-bit unsigned probe > 1000
- `gt16'u1000` — 16비트 부호 없는 프로브 > 1000
- `eq2'bXX` — don't care (probe not participating)
- `eq2'bXX` — 상관없음(프로브가 참여하지 않음)

## Trigger State Machine (TSM) Syntax
## 트리거 상태 머신(TSM) 구문

TSM programs define multi-state advanced triggers for `ADVANCED_ONLY`/
TSM 프로그램은 `ADVANCED_ONLY`/에 대한 다중 상태 고급 트리거를 정의합니다.
`ADVANCED_OR_TRIG_IN` mode.
'ADVANCED_OR_TRIG_IN' 모드.
The same language and rules apply on both
두 언어 모두에 동일한 언어와 규칙이 적용됩니다.
routing paths this skill uses: vivado-mcp's `.tsm` file
이 스킬이 사용하는 라우팅 경로: vivado-mcp의 `.tsm` 파일
(`CONTROL.TSM_FILE`, compile-checked with `run_hw_ila -compile_only`) and
(`Control.TSM_FILE`, `run_hw_ila -compile_only`로 컴파일 확인) 및
chipscope-mcp's `chipscope_ila_core(action="arm", trigger_state_machine=<program text>)`.
Chipscope-mcp의 `chipscope_ila_core(action="arm", Trigger_state_machine=<프로그램 텍스트>)`.

### Grammar
### 문법

Reconciled from two sources that disagree in three places (see inline notes
세 군데에서 의견이 일치하지 않는 두 소스를 조정했습니다(인라인 메모 참조).
below the block): the official
블록 아래): 공식
[Vivado UG908 "Trigger State Machine Language Grammar"](https://docs.amd.com/r/en-US/ug908-vivado-programming-debugging/Trigger-State-Machine-Language-Grammar)
[Vivado UG908 "트리거 상태 기계 언어 문법"](https://docs.amd.com/r/en-US/ug908-vivado-programming-debugging/Trigger-State-Machine-Language-Grammar)
page, and chipscope-mcp's internal chipscopy LLM trigger-generation prompt
페이지 및 Chipscope-mcp의 내부 칩 복사 LLM 트리거 생성 프롬프트
template (`chipscopy_llm/prompt_builder.py`).
템플릿(`chipscopy_llm/prompt_builder.py`).

```
'THING' = THING is a terminal
{<thing>} = 0 or more thing
[<thing>] = 0 or 1 thing

<program>           ::= <state_list>
<state_list>        ::= <state_list> <state> | <state>
<state>             ::= 'STATE' <state_label> ':' <if_condition> | <action_block>
<action_block>      ::= <action_list> 'GOTO' <state_label> ';'
                       | <action_list> 'TRIGGER' ';'
                       | 'GOTO' <state_label> ';'
                       | 'TRIGGER' ';'
<action_list>       ::= <action_statement> | <action_list> <action_statement>
<action_statement>  ::= 'SET_FLAG' <flag_name> ';'
                       | 'CLEAR_FLAG' <flag_name> ';'
                       | 'INCREMENT_COUNTER' <counter_name> ';'
                       | 'RESET_COUNTER' <counter_name> ';'
<if_condition>      ::= 'IF' '(' <condition> ')' 'THEN' <action_block>
                         ['ELSEIF' '(' <condition> ')' 'THEN' <action_block>]
                         'ELSE' <action_block>
                         'ENDIF'
<condition>         ::= <probe_match_list> | <counter_match> | <probe_counter_match>
<probe_counter_match> ::= '(' <probe_counter_match> ')'
                       | <probe_match_list> <boolean_logic_op> <counter_match>
                       | <counter_match> <boolean_logic_op> <probe_match_list>
<probe_match_list>  ::= '(' <probe_match> ')' | <probe_match>
<probe_match>       ::= <probe_match_list> <boolean_logic_op> <probe_match_list>
                       | <probe_name> <compare_op> <constant>
                       | <constant> <compare_op> <probe_name>
<counter_match>     ::= '(' <counter_match> ')'
                       | <counter_name> <counter_compare_op> <constant>
                       | <constant> <counter_compare_op> <counter_name>
<constant>          ::= <integer_constant> | <hex_constant> | <binary_constant>
<compare_op>        ::= '==' | '!=' | '>' | '>=' | '<' | '<='
<counter_compare_op> ::= '==' | '!='
<boolean_logic_op>  ::= '&&' | '||'

<probe_name>        ::= [A-Z_\[\]<>/\.][A-Z_0-9\[\]<>/\.]+
<state_label>       ::= [A-Z_][A-Z_0-9]+
<flag_name>         ::= \$FLAG[0-3]
<counter_name>      ::= \$COUNTER[0-3]
<hex_constant>      ::= <integer>*'h<hex_digit>+
<binary_constant>   ::= <integer>*'b<binary_digit>+
<integer_constant>  ::= <integer>*'u<integer_digit>+
<integer>           ::= <digit>+
<hex_digit>         ::= [0-9ABCDEFBN_]
<binary_digit>      ::= [01XRFBN_]
<digit>             ::= [0-9]
```

The language is case-insensitive.
언어는 대소문자를 구분하지 않습니다.
`#` starts a comment; everything from `#`
`#`은 주석을 시작합니다. `#`의 모든 것
to end of line is ignored.
줄 끝까지는 무시됩니다.

**Where this reconciles two disagreeing sources:**
**이로 인해 서로 다른 두 출처가 조화를 이루는 경우:**
- UG908's published grammar page writes `<actionblock>` (no underscore)
- UG908의 게시된 문법 페이지에는 `<actionblock>`(밑줄 없음)이 기록되어 있습니다.
inside `<if_condition>` but `<action_block>` everywhere else — the same
`<if_condition>` 내부에 있지만 다른 모든 곳에서는 `<action_block>` — 동일
nonterminal, an inconsistency in AMD's own published page.
비터미널, AMD 자체 게시 페이지의 불일치.
Normalized to
정규화됨
`<action_block>` here.
여기 `<action_block>`이 있습니다.
- UG908's grammar page shares the full 6-operator `<compare_op>` for
- UG908의 문법 페이지는 6개 연산자 `<compare_op>` 전체를 공유합니다.
`<counter_match>`, but UG908's own separate "Counter Conditions" page
`<counter_match>`, 그러나 UG908의 별도 "카운터 조건" 페이지
states only `==`/`!=` are valid for counters — matching chipscopy's
상태는 `==`/`!=`만 카운터에 유효합니다 — 칩 카피와 일치
internal grammar's dedicated `<counter_compare_op>`, used here since it
내부 문법 전용 `<counter_compare_op>`, 이후 여기서 사용됨
matches the documented enforced behavior, not the raw grammar page.
원시 문법 페이지가 아닌 문서화된 강제 동작과 일치합니다.
- UG908's `<probe_name>` omits `.` from the allowed charset; chipscopy's
- UG908의 `<probe_name>`은 허용된 문자 세트에서 `.`를 생략합니다. 칩스코피
internal grammar includes it.
내부 문법에 포함되어 있습니다.
The wider (chipscopy) charset is used here
여기에서는 더 넓은(칩스코피) 문자 세트가 사용됩니다.
so a hierarchical probe name containing `.` is not misread as invalid.
따라서 '.'을 포함하는 계층적 프로브 이름은 유효하지 않은 것으로 잘못 읽혀지지 않습니다.

### Rules that actually cause rejections
### 실제로 거부를 유발하는 규칙

These are the specific failure modes chipscope-mcp#1289 reproduced on real
실제 환경에서 재현된 특정 오류 모드인 Chipscope-mcp#1289는 다음과 같습니다.
hardware (6 distinct rejected attempts, one real `arm()` call each) plus the
하드웨어(6개의 거부된 시도, 각각 하나의 실제 `arm()` 호출) 및
grammar's own structural constraints.
문법 자체의 구조적 제약.
Each rule names its source.
각 규칙은 해당 소스의 이름을 지정합니다.

- **`state <name>:` is mandatory** — a bare `<name>:` without the `state`
- **`state <name>:`은 필수입니다** — `state`가 없는 순수한 `<name>:`
keyword is rejected (`mismatched input '<name>' expecting STATE`).
키워드가 거부되었습니다(`상태가 예상되는 입력 '<이름>'이 일치하지 않습니다`).
*(Source: chipscope-mcp#1289 failure #1.)*
*(출처: Chipscope-mcp#1289 실패 #1.)*
- **Only non-entry states need an incoming `goto` to be reachable.** The
- **진입하지 않은 상태에만 도달하려면 수신 `goto`가 필요합니다.**
first-declared (entry) state needs no incoming `goto` — it is where
처음 선언된(진입) 상태에는 들어오는 `goto`가 필요하지 않습니다.
execution starts.
실행이 시작됩니다.
A single-state program, or a state that only ever
단일 주 프로그램, 또는 오직 한 주에서만
`goto`s itself, is valid — see the first two Examples below, one of which
`goto` 자체는 유효합니다. 아래의 처음 두 예를 참조하세요. 그 중 하나는
is hardware-proven in chipscope-mcp's own MCP test suite.
Chipscope-mcp의 자체 MCP 테스트 제품군에서 하드웨어로 입증되었습니다.
A *non-entry*
A *비입장*
state with no path back to it from the entry state is rejected
진입 상태에서 돌아갈 경로가 없는 상태는 거부됩니다.
(`The following states are not reachable: <name>.`). *(Source:
(`다음 상태에 연결할 수 없습니다: <이름>.`). *(출처:
chipscope-mcp `tests/mcp/test_mcp_ila.py::test_advanced_trigger_state_machine`,
칩스코프-mcp `tests/mcp/test_mcp_ila.py::test_advanced_trigger_state_machine`,
hardware-proven on vmk180/vpk120/vck190, plus chipscopy's own canonical
vmk180/vpk120/vck190에서 하드웨어 검증 및 Chipscopy의 자체 표준
single-state trigger-immediate example. chipscope-mcp#1289's failure #3
단일 상태 트리거 즉시 예시. Chipscope-mcp#1289의 실패 #3
— "trigger state declared first, nothing goes to it later" — is this rule
— "트리거 상태가 먼저 선언되고 나중에 아무 것도 진행되지 않습니다" — 이 규칙입니다
applied to a non-entry state.)*
미진입 상태에 적용됩니다.)*
- **At least one state must contain a `trigger;` action.** A program with no
- **적어도 하나의 상태에는 '트리거' 동작이 포함되어야 합니다.** 트리거가 없는 프로그램
reachable `trigger;` is rejected explicitly
도달 가능한 `트리거;`가 명시적으로 거부되었습니다.
(`The state machine need at least one trigger action.`). *(Source:
('상태 머신에는 최소한 하나의 트리거 작업이 필요합니다.') *(출처:
chipscope-mcp#1289 failures #2 and #6.)*
Chipscope-mcp#1289 오류 #2 및 #6.)*
- **A `goto` naming an undeclared state is rejected by name**
- **선언되지 않은 상태의 이름을 지정하는 `goto`는 이름으로 거부됩니다**
(`Unknown state "<name>".`) — this includes forward-declaring a state with
(`알 수 없는 상태 "<name>".`) — 여기에는 상태를 다음과 같이 전달 선언하는 것이 포함됩니다.
`state X;` (no colon, no body): that's not valid syntax at all
`state X;`(콜론 없음, 본문 없음): 전혀 유효한 구문이 아닙니다.
(`no viable alternative at input 'state X;'`). *(Source: chipscope-mcp#1289
('입력 '상태 X'에는 실행 가능한 대안이 없습니다;'). *(출처: Chipscope-mcp#1289
failures #2, #5, #6.)*
실패 #2, #5, #6.)*
- **At most one `elseif` per `if`.** *(Source: chipscopy `prompt_builder.py`,
- **`if`당 최대 하나의 `elseif`.** *(출처:chipscopy `prompt_builder.py`,
stated explicitly — "Only a single elseif is allowed in the if statement" —
명시적으로 언급됨 — "if 문에는 단일 elseif만 허용됩니다." —
and confirmed by the grammar's own `[...]` optional-and-singular slot.)*
문법 자체 `[...]` 선택 및 단수 슬롯으로 확인됩니다.)*
- **No nested `if`.** An `<action_block>` (what follows `then`/`else`/
- **중첩된 `if`가 없습니다.** `<action_block>`(`then`/`else`/ 뒤에 오는 내용)
`elseif`) can only be `goto`, `trigger`, or a counter/flag action — never
`elseif`)는 `goto`, `trigger` 또는 카운터/플래그 작업만 가능합니다.
another `if`. *(Source: chipscopy `prompt_builder.py`, stated explicitly —
또 다른 '만약'. *(출처: Chipscopy `prompt_builder.py`, 명시적으로 명시됨 —
"No nested if statements are allowed" — and confirmed by the grammar
"중첩된 if 문은 허용되지 않습니다." - 문법으로 확인됨
having no `if`-producing symbol inside `<action_block>`.)*
`<action_block>` 안에 `if` 생성 기호가 없습니다.)*
- **Counter conditions support only `==`/`!=`.** Probe conditions
- **카운터 조건은 `==`/`!=`만 지원합니다.** 프로브 조건
additionally support `<`, `<=`, `>`, `>=`. *(Source: chipscopy
추가로 `<`, `<=`, `>`, `>=`를 지원합니다. *(출처: 칩카피
`prompt_builder.py`, and UG908's "Counter Conditions" page — see the
`prompt_builder.py` 및 UG908의 "카운터 조건" 페이지 —
grammar-reconciliation note above for why this differs from UG908's raw
이것이 UG908의 원시와 다른 이유에 대한 위의 문법 조정 참고 사항
grammar page.)*
문법 페이지.)*
- **Each counter has exactly one comparator** — a given `$counterN` can
- **각 카운터에는 정확히 하나의 비교기가 있습니다** — 주어진 `$counterN`은
appear in a counter condition only once in the whole program. *(Source:
전체 프로그램에서 한 번만 카운터 조건으로 나타납니다. *(출처:
UG908 "Counter Conditions" page.)* **Each PROBE port has a compile-time
UG908 "카운터 조건" 페이지.)* **각 PROBE 포트에는 컴파일 시간이 있습니다.
comparator limit, and a probe attached to that port can appear in a probe
비교기 제한 및 해당 포트에 연결된 프로브가 프로브에 나타날 수 있습니다.
condition only that many times total.** Two sources disagree on the
총 횟수만큼만 조건을 적용합니다.** 두 소스의 의견이 일치하지 않습니다.
limit: UG908's "Debug Probe Conditions" page says 1-16; chipscopy's
제한: UG908의 "디버그 프로브 조건" 페이지에 1-16이 표시됩니다. 칩스코피
internal prompt text says 1-4.
내부 프롬프트 텍스트에 1-4가 표시됩니다.
Both are stated here rather than picking
여기에는 둘 다 선택하는 것이 아니라 둘 다 명시되어 있습니다.
one — the design's actual limit is queryable via this same file's `ILA
하나 - 디자인의 실제 한계는 동일한 파일의 `ILA를 통해 쿼리 가능합니다.
Design-Time Core Properties` table (`C_ALL_PROBE_SAME_MU_CNT`, range
디자인 타임 핵심 속성` 테이블(`C_ALL_PROBE_SAME_MU_CNT`, 범위
1-16, agreeing with UG908).
1-16, UG908에 동의함).
- **A condition's `<bit_width>` must equal the actual width of the probe or
- **조건의 `<bit_width>`는 프로브의 실제 너비와 동일해야 합니다.
counter it compares.** Counters are always 16 bits
카운터를 비교합니다.** 카운터는 항상 16비트입니다.
(`$counter0 == 16'u100`, not `$counter0 == 8'u100`). *(Source: UG908
(`$counter0 == 16'u100`, `$counter0 == 8'u100`이 아님). *(출처: UG908
"Debug Probe Conditions" page — "bit width is the width of the probe.")*
"디버그 프로브 조건" 페이지 — "비트 너비는 프로브의 너비입니다.")*
- **Probe vector indices (`probe[0]`, `probe[1]`) are not valid TSM operands.**
- **프로브 벡터 인덱스(`probe[0]`, `probe[1]`)는 유효한 TSM 피연산자가 아닙니다.**
Collapse to a single multi-bit compare against the whole probe instead:
대신 전체 프로브에 대한 단일 다중 비트 비교로 축소합니다.
`(slot_0_r_dout[0] == 1'b1) && (slot_0_r_dout[1] == 1'b1)` must be written
`(slot_0_r_dout[0] == 1'b1) && (slot_0_r_dout[1] == 1'b1)`을 작성해야 합니다.
`(slot_0_r_dout == 2'b11)`. *(Source: chipscopy `prompt_builder.py` — this
`(slot_0_r_dout == 2'b11)`. *(출처:chipscopy `prompt_builder.py` — 이
rule is not stated in UG908's pages found during research; flagged here as
연구 중에 발견된 UG908의 페이지에는 규칙이 명시되어 있지 않습니다. 여기에 플래그가 지정됨
a practical rule from chipscope-mcp's own internal tooling, not an
Chipscope-mcp 자체 내부 툴링의 실제 규칙입니다.
official grammar citation.)*
공식 문법 인용.)*
- **Prefer don't-care (`X`) bits over `&&`/`||` of the same probe** when only
- **동일한 프로브의 `&&`/`||`보다 상관 없음(`X`) 비트를 선호**
some bits matter: `(sig == 3'bX11)` over
일부 비트는 중요합니다: `(sig == 3'bX11)` 이상
`((sig == 3'b011) || (sig == 3'b111))`. *(Source: chipscopy
`((sig == 3'b011) || (sig == 3'b111))`. *(출처: 칩카피
`prompt_builder.py`, stated explicitly with a BAD/GOOD example pair.)*
BAD/GOOD 예제 쌍으로 명시적으로 명시된 `prompt_builder.py`.)*

### Optional: enumerated compare values
### 선택 사항: 열거된 비교 값

UG908's "Advanced Trigger" page shows a form that compares against a
UG908의 "고급 트리거" 페이지는 다음과 비교되는 양식을 보여줍니다.
design-declared enumeration name instead of a numeric literal, e.g.
숫자 리터럴 대신 디자인으로 선언된 열거형 이름입니다.
`fast_ila_slice5_fb == 5'eFIFTEEN`.
`fast_ila_slice5_fb == 5'eFIFTEEN`.
This is **not** part of the primary
이것은 기본의 일부가 **아닙니다**
grammar above (neither UG908's own grammar page nor chipscopy's internal
위의 문법(UG908의 자체 문법 페이지나 Chipscopy의 내부 페이지가 아님)
grammar defines an `'e` radix or enum production) — it only works when the
문법은 `'e` 기수 또는 열거형 생성을 정의합니다) — 이는 다음과 같은 경우에만 작동합니다.
target signal has enumerated values defined in the design's waveform/bus
대상 신호에는 설계의 파형/버스에 정의된 열거 값이 있습니다.
configuration.
구성.
Most probes should use the ordinary bit/hex/decimal literal
대부분의 프로브는 일반 비트/16진수/10진수 리터럴을 사용해야 합니다.
forms in the primary grammar instead; reach for this only if a design is
대신 기본 문법으로 구성됩니다. 디자인이 다음과 같은 경우에만 이를 달성할 수 있습니다.
known to declare enum values for the probe in question.
문제의 프로브에 대한 열거형 값을 선언하는 것으로 알려져 있습니다.

```
state my_state0:
  if (fast_ila_slice5_fb == 5'eFIFTEEN) then
    set_flag $flag1;
    goto my_state0;
  elseif (fast_ila_slice5_fb == 5'eTWELVE) then
    trigger;
  else
    clear_flag $flag1;
    goto my_state0;
  endif
```

*(Source: UG908 "Advanced-Trigger" page, byte-exact.)*
*(출처: UG908 "고급 트리거" 페이지, 바이트 단위)*

### Examples
### 예

**Trigger immediately** (single state, no condition, no `goto` at all —
**즉시 트리거**(단일 상태, 조건 없음, `goto` 전혀 없음 —
valid: the entry state needs no incoming `goto`):
유효함: 항목 상태에는 들어오는 `goto`가 필요하지 않습니다.):
```
state state0:
  trigger;
```
*(Source: chipscopy `prompt_builder.py`, byte-exact.)*
*(출처: 칩카피 `prompt_builder.py`, 바이트 단위.)*

**Single state with a self-loop** — hardware-proven on real hardware
**자체 루프가 있는 단일 상태** — 실제 하드웨어에서 하드웨어 검증됨
(vmk180, vpk120, vck190) via chipscope-mcp's own MCP test suite, confirming
(vmk180, vpk120, vck190) Chipscope-mcp의 자체 MCP 테스트 스위트를 통해 확인
a single-state program with a conditional and a self-referencing `goto` is
조건부 및 자체 참조 'goto'를 사용하는 단일 상태 프로그램은
valid:
유효한:
```
state s0:
  if (<probe> == 1'b1) then
    trigger;
  else
    goto s0;
  endif
```
*(Source: chipscope-mcp `tests/mcp/test_mcp_ila.py::test_advanced_trigger_state_machine`,
*(출처: Chipscope-mcp `tests/mcp/test_mcp_ila.py::test_advanced_trigger_state_machine`,
`success: true, mode: "advanced"`.)*
`성공: true, 모드: "고급"`.)*

**One-way branch** (unconditional `goto`, no `if`):
**단방향 분기**(무조건 `goto`, `if` 없음):
```
state my_state0:
  goto my_state1;
state my_state1:
  trigger;
```
*(Source: first line byte-exact from chipscopy `prompt_builder.py`; the
*(출처: 칩 카피 `prompt_builder.py`의 첫 번째 줄 바이트와 정확함;
second state's body is added here to make the fragment a complete, runnable
두 번째 상태의 본문이 여기에 추가되어 조각을 완전하고 실행 가능하게 만듭니다.
two-state program.)*
2개 주 프로그램.)*

**Confirmed working on real hardware** — vmk180, ILA core
**실제 하드웨어에서 작동 확인됨** — vmk180, ILA 코어
`chipscopy_i/counters/ila_slow_counter_0`, chipscope-mcp#1289 (`arm()`
`chipscopy_i/counters/ila_slow_counter_0`, Chipscope-mcp#1289 (`arm()`
returned `mode: "advanced"`, trigger fired on the 2nd state's threshold, not
`mode: "advanced"`를 반환했습니다. 두 번째 상태의 임계값에서 트리거가 실행되었습니다.
the 1st, verified against uploaded sample data):
첫 번째, 업로드된 샘플 데이터에 대해 확인됨):
```
state wait_first:
  if (chipscopy_i/counters/slow_counter_0_Q_1 > 32'hb2d05e00) then
    goto wait_second;
  else
    goto wait_first;
  endif

state wait_second:
  if (chipscopy_i/counters/slow_counter_0_Q_1 > 32'hee6b2800) then
    trigger;
  else
    goto wait_second;
  endif
```
*(Source: chipscope-mcp#1289 issue body, byte-exact.)*
*(출처: Chipscope-mcp#1289 문제 본문, 바이트 단위.)*

**Counter + trigger on the 4th rising edge of a probe:**
**프로브의 네 번째 상승 에지에서 카운터 + 트리거:**
```
state wait_for_4th_rising_edge_of_abc:
  if ((abc == 1'bR) && ($counter0 == 16'h0003)) then
    reset_counter $counter0;
    trigger;
  elseif (abc == 1'bR) then
    increment_counter $counter0;
    goto wait_for_4th_rising_edge_of_abc;
  else
    goto wait_for_4th_rising_edge_of_abc;
  endif
```
*(Source: chipscopy `prompt_builder.py`, byte-exact.)*
*(출처: 칩카피 `prompt_builder.py`, 바이트 단위.)*

**AXI write of specific data to a specific address** (two-phase: address
**특정 데이터를 특정 주소에 AXI 쓰기**(2단계: 주소
then data; hierarchical probe names, don't-care bits for handshake signals):
그런 다음 데이터; 계층적 프로브 이름, 핸드셰이크 신호에 대해 신경 쓰지 않는 비트):
```
state wait_for_write_addr:
  if ((slot_0_aw_dout == 2'b11) && (chipscopy_i/noc_tg_bc/noc_bc_axis_ila_0/SLOT_0_AXI_awaddr == 64'h80000000)) then
    goto wait_for_write_data;
  else
    goto wait_for_write_addr;
  endif

state wait_for_write_data:
  if ((slot_0_w_dout == 3'b11X) && (chipscopy_i/noc_tg_bc/noc_bc_axis_ila_0/SLOT_0_AXI_wdata == 32'h12345678)) then
    trigger;
  else
    goto wait_for_write_data;
  endif
```
*(Source: rewritten from chipscopy `prompt_builder.py` for readability,
*(출처: 가독성을 위해 칩 카피 `prompt_builder.py`에서 다시 작성됨,
collapsing indexed-bit ANDs into single multi-bit compares per the
인덱스 비트 AND를 단일 다중 비트 비교로 축소
don't-care rule above.
위의 상관 없음 규칙.
Address phase: source constrains both
주소 단계: 소스는 두 가지 모두를 제한합니다.
`slot_0_aw_dout[0]` and `[1]` to `1'b1`, no free bit — `2'b11`, unchanged
`slot_0_aw_dout[0]` 및 `[1]`에서 `1'b1`까지, 여유 비트 없음 — `2'b11`, 변경되지 않음
from source.
소스에서.
Data phase: source constrains `slot_0_w_dout[1]` and `[2]` to
데이터 단계: 소스는 `slot_0_w_dout[1]` 및 `[2]`를 다음으로 제한합니다.
`1'b1`, bit `[0]` free — `3'b11X` (bit2=1, bit1=1, bit0=X).)*
`1'b1`, 비트 `[0]` 사용 가능 — `3'b11X` (비트2=1, 비트1=1, 비트0=X).)*

### Rejected patterns seen in practice (chipscope-mcp#1289)
### 실제로 나타나는 거부된 패턴(chipscope-mcp#1289)

Six distinct real `arm()` calls against live hardware, each rejected with a
라이브 하드웨어에 대한 6개의 서로 다른 실제 `arm()` 호출, 각각 거부됨
different parser error — recognize these exact strings and fix the
다른 파서 오류 — 이러한 정확한 문자열을 인식하고 수정합니다.
corresponding rule above instead of guessing a new variant. "What differed"
새로운 변형을 추측하는 대신 위의 해당 규칙을 따르세요. "무엇이 달랐나"
is quoted verbatim from the issue; it describes what the reporter changed,
문제에서 그대로 인용되었습니다. 기자가 무엇을 바꿨는지 설명하고,
not a proven root cause in every row.
모든 행에서 입증된 근본 원인은 아닙니다.

| What differed from a working program (quoted from #1289)<br>실제 프로그램과 다른 점 (#1289에서 인용) | Server error<br>서버 오류 |
|---|---|
| No `state` keyword before the label<br>라벨 앞에 `state` 키워드가 없습니다. | `mismatched input 'wait_a' expecting STATE`<br>`상태가 일치하지 않는 입력 'wait_a'' |
| Final state was bare `trigger;` with no `if/endif` wrapper<br>최종 상태는 'if/endif' 래퍼가 없는 '트리거;'였습니다. | `Unknown state "wait_b".` + `The state machine need at least one trigger action.`<br>`알 수 없는 상태 "wait_b".` + `상태 머신에는 적어도 하나의 트리거 작업이 필요합니다.` |
| Reordered states, trigger state declared first<br>재정렬된 상태, 트리거 상태가 먼저 선언됨 | `The following states are not reachable: wait_b.`<br>`다음 상태에 연결할 수 없습니다: wait_b.` |
| Single-state version<br>단일 상태 버전 | `The following states are not reachable: wait_a.`<br>`다음 상태에 연결할 수 없습니다: wait_a.` |
| Added bare `state X;` forward declarations<br>기본 `상태 X;` 전방 선언을 추가했습니다. | `no viable alternative at input 'state wait_b;'`<br>`입력 '상태 wait_b'에 실행 가능한 대안이 없습니다.` |
| Renamed states, final state still bare `trigger;`<br>상태 이름이 변경되었지만 최종 상태에는 여전히 '트리거'가 남아 있습니다. | `Unknown state "phase_two".` + `The state machine need at least one trigger action.`<br>`알 수 없는 상태 "phase_two".` + `상태 머신에는 적어도 하나의 트리거 작업이 필요합니다.` |

**Note on the "Single-state version" row:** this reference's own Examples
**"단일 상태 버전" 행에 대한 참고 사항:** 이 참조의 자체 예
section above shows two hardware-proven single-state programs (one with no
위 섹션에서는 하드웨어로 입증된 두 개의 단일 상태 프로그램을 보여줍니다(하나는
`goto` at all, one with a self-loop), so a single state is not
'goto'는 전혀 사용되지 않으며 자체 루프가 있음) 단일 상태는 그렇지 않습니다.
categorically rejected.
단호히 거절했습니다.
The reporter's exact source for this specific
이 특정 내용에 대한 기자의 정확한 출처
attempt is not available, so its precise cause is not restated as a general
시도가 불가능하므로 정확한 원인은 일반적인 것으로 다시 기술되지 않습니다.
rule here — treat it as an open question if it resurfaces, not as
여기에서 규칙을 따르십시오. 다시 표면화되면 공개 질문으로 취급하지 말고 공개 질문으로 처리하십시오.
"single-state programs fail."
"단일 상태 프로그램이 실패합니다."

### Sources
### 출처

- [UG908 "Trigger State Machine Language Grammar"](https://docs.amd.com/r/en-US/ug908-vivado-programming-debugging/Trigger-State-Machine-Language-Grammar)
- [UG908 "트리거 상태 기계 언어 문법"](https://docs.amd.com/r/en-US/ug908-vivado-programming-debugging/Trigger-State-Machine-Language-Grammar)
- [UG908 "Conditional Branching"](https://docs.amd.com/r/en-US/ug908-vivado-programming-debugging/Conditional-Branching)
- [UG908 "조건부 분기"](https://docs.amd.com/r/en-US/ug908-vivado-programming-debugging/Conditional-Branching)
- [UG908 "Counters"](https://docs.amd.com/r/en-US/ug908-vivado-programming-debugging/Counters) / [UG908 "Counter Conditions"](https://docs.amd.com/r/en-US/ug908-vivado-programming-debugging/Counter-Conditions)
- [UG908 "카운터"](https://docs.amd.com/r/en-US/ug908-vivado-programming-debugging/Counters) / [UG908 "카운터 조건"](https://docs.amd.com/r/en-US/ug908-vivado-programming-debugging/Counter-Conditions)
- [UG908 "Flags"](https://docs.amd.com/r/en-US/ug908-vivado-programming-debugging/Flags)
- [UG908 "플래그"](https://docs.amd.com/r/en-US/ug908-vivado-programming-debugging/Flags)
- [UG908 "Debug Probe Conditions"](https://docs.amd.com/r/en-US/ug908-vivado-programming-debugging/Debug-Probe-Conditions)
- [UG908 "디버그 프로브 조건"](https://docs.amd.com/r/en-US/ug908-vivado-programming-debugging/Debug-Probe-Conditions)
- [UG908 "Advanced Trigger"](https://docs.amd.com/r/en-US/ug908-vivado-programming-debugging/Advanced-Trigger)
- [UG908 "고급 트리거"](https://docs.amd.com/r/en-US/ug908-vivado-programming-debugging/Advanced-Trigger)
- chipscope-mcp's internal chipscopy LLM trigger-generation prompt template (`chipscopy_llm/prompt_builder.py`)
- Chipscope-mcp의 내부 Chipscopy LLM 트리거 생성 프롬프트 템플릿(`chipscopy_llm/prompt_builder.py`)
- chipscope-mcp `tests/mcp/test_mcp_ila.py::test_advanced_trigger_state_machine` (hardware-proven)
- Chipscope-mcp `tests/mcp/test_mcp_ila.py::test_advanced_trigger_state_machine`(하드웨어 입증)
- chipscope-mcp#1289 (confirmed-working example, 6-failure table)
- Chipscope-mcp#1289(확인된 작동 예, 6-실패 테이블)

## ILA Design-Time Core Properties (for netlist insertion reference)
## ILA 디자인 타임 핵심 속성(넷리스트 삽입 참조용)

| Property<br>재산 | Values<br>가치 | Description<br>설명 |
|----------|--------|-------------|
| C_DATA_DEPTH<br>C_DATA_DEPTH | 1024–131072 | Sample buffer depth<br>샘플 버퍼 깊이 |
| C_NUM_OF_PROBES<br>C_NUM_OF_PROBES개 | 1–1024 | Number of probe ports<br>프로브 포트 수 |
| C_PROBE\<n\>_WIDTH<br>C_PROBE\<n\>_WIDTH | 1–4096 | Width of probe port n<br>프로브 포트 n의 너비 |
| C_ADV_TRIGGER<br>C_ADV_TRIGGER | true/false<br>참/거짓 | Enable advanced trigger / TSM<br>고급 트리거/TSM 활성화 |
| C_EN_STRG_QUAL<br>C_EN_STRG_QUAL | true/false<br>참/거짓 | Enable capture condition (storage qualifier)<br>캡처 조건 활성화(스토리지 한정자) |
| C_TRIGIN_EN<br>C_TRIGIN_EN | true/false<br>참/거짓 | Enable TRIG_IN port<br>TRIG_IN 포트 활성화 |
| C_TRIGOUT_EN<br>C_TRIGOUT_EN | true/false<br>참/거짓 | Enable TRIG_OUT port<br>TRIG_OUT 포트 활성화 |
| C_INPUT_PIPE_STAGES<br>C_INPUT_PIPE_STAGES | 0–6 | Pipeline stages on probe inputs (timing)<br>프로브 입력의 파이프라인 단계(타이밍) |
| C_ALL_PROBE_SAME_MU_CNT<br>C_ALL_PROBE_SAME_MU_CNT | 1–16 | Match units per probe<br>프로브당 일치 단위 |
| C_MEMORY_TYPE<br>C_MEMORY_TYPE | 0 (BRAM), 1 (URAM)<br>0(BRAM), 1(URAM) | Storage primitive (Versal only)<br>저장 기본 요소(Versal에만 해당) |

## Tcl Command Quick Reference — ILA
## Tcl 명령 빠른 참조 — ILA

| Command<br>명령 | Purpose<br>목적 |
|---------|---------|
| `get_hw_ilas`<br>`get_hw_ilas` | List all ILA debug cores on current device<br>현재 장치의 모든 ILA 디버그 코어 나열 |
| `current_hw_ila`<br>`현재_hw_ila` | Get/set the current ILA<br>현재 ILA 가져오기/설정 |
| `get_hw_probes -of_objects $ila`<br>`get_hw_probes -of_objects $ila` | List probes on an ILA<br>ILA에 프로브 나열 |
| `set_property CONTROL.* $ila`<br>`set_property Control.* $ila` | Configure trigger/capture settings<br>트리거/캡처 설정 구성 |
| `set_property TRIGGER_COMPARE_VALUE <val> $probe`<br>`set_property TRIGGER_COMPARE_VALUE <val> $probe` | Set trigger match on a probe<br>프로브에서 트리거 일치 설정 |
| `run_hw_ila $ila`<br>`run_hw_ila $ila` | Arm ILA for trigger event<br>트리거 이벤트를 위한 Arm ILA |
| `run_hw_ila -trigger_now $ila`<br>`run_hw_ila -trigger_now $ila` | Trigger immediately (aliveness check)<br>즉시 트리거(활성 확인) |
| `run_hw_ila -compile_only $ila`<br>`run_hw_ila -compile_only $ila` | Compile-check TSM without arming<br>무장 없이 TSM 컴파일 확인 |
| `run_hw_ila -file <path> $ila`<br>`run_hw_ila -파일 <경로> $ila` | Export trigger config for startup<br>시작을 위한 트리거 구성 내보내기 |
| `wait_on_hw_ila $ila`<br>`wait_on_hw_ila $ila` | Block until capture complete<br>캡처가 완료될 때까지 차단 |
| `upload_hw_ila_data $ila`<br>`upload_hw_ila_data $ila` | Pull captured data from device<br>장치에서 캡처된 데이터 가져오기 |
| `write_hw_ila_data -csv_file <file> $data`<br>`write_hw_ila_data -csv_file <파일> $data` | Export as CSV<br>CSV로 내보내기 |
| `write_hw_ila_data -vcd_file <file> $data`<br>`write_hw_ila_data -vcd_file <파일> $data` | Export as VCD<br>VCD로 내보내기 |
| `write_hw_ila_data <file> $data`<br>`write_hw_ila_data <파일> $data` | Export as native .ila<br>기본 .ila로 내보내기 |
| `read_hw_ila_data <file>`<br>`read_hw_ila_data <파일>` | Load previously saved .ila file<br>이전에 저장한 .ila 파일 로드 |
| `list_hw_samples $probe`<br>`list_hw_samples $프로브` | List sample values for a probe<br>프로브의 샘플 값 나열 |
| `create_hw_probe -map {...} <name> $ila`<br>`create_hw_probe -map {...} <이름> $ila` | Create custom probe from physical bits<br>물리적 비트에서 맞춤형 프로브 생성 |
| `reset_hw_ila $ila`<br>`reset_hw_ila $ila` | Reset all CONTROL properties to defaults<br>모든 CONTROL 속성을 기본값으로 재설정 |

## report_data.json Schema
## report_data.json 스키마

```json
{
  "metadata": {
    "skill": "hw-ila-debug", "version": "1.0.0",
    "mode": "ila", "timestamp": "<ISO8601>",
    "device": "<part>", "ila_core": "hw_ila_1"
  },
  "ila_config": {
    "data_depth": 1024, "max_depth": 4096,
    "trigger_mode": "BASIC_ONLY", "trigger_condition": "AND",
    "trigger_position": 512, "window_count": 1,
    "capture_mode": "ALWAYS"
  },
  "probes": [
    { "name": "axi_arvalid", "width": 1, "type": "ila",
      "trigger_compare": "eq1'b1", "port": 3 }
  ],
  "capture": {
    "status": "TRIGGER_CAPTURED", "sample_count": 1024,
    "export_file": "capture_data.csv", "export_format": "csv"
  },
  "observations": ["axi_arvalid asserted at sample 512", "..."],
  "recommendations": []
}
```

## References
## 참고자료

- **UG908**: Vivado Programming and Debugging — hw_ila, hw_probe Tcl commands
- **UG908**: Vivado 프로그래밍 및 디버깅 — hw_ila, hw_probe Tcl 명령
- **UG912**: Vivado Properties Reference — HW_ILA, HW_PROBE properties
- **UG912**: Vivado 속성 참조 — HW_ILA, HW_PROBE 속성
- **UG835**: Vivado Tcl Commands Reference — run_hw_ila, write_hw_ila_data, etc.
- **UG835**: Vivado Tcl 명령 참조 — run_hw_ila, write_hw_ila_data 등
- **UG936**: Vivado Tutorial: Programming and Debugging — step-by-step ILA lab
- **UG936**: Vivado 튜토리얼: 프로그래밍 및 디버깅 — 단계별 ILA 랩
- **PG172**: Integrated Logic Analyzer v6.2 Product Guide — ILA core parameters, ports
- **PG172**: 통합 로직 분석기 v6.2 제품 가이드 - ILA 핵심 매개변수, 포트
- **PG357**: ILA with AXI4-Stream Interface Product Guide — Versal ILA
- **PG357**: AXI4-Stream 인터페이스가 포함된 ILA 제품 가이드 - Versal ILA
- **UG908 "Trigger State Machine Language Grammar"**: full TSM BNF — see this file's "Trigger State Machine (TSM) Syntax" section for the reconciled grammar
- **UG908 "Trigger State Machine Language Grammar"**: 전체 TSM BNF — 조정된 문법은 이 파일의 "TSM(Trigger State Machine) 구문" 섹션을 참조하세요.
- **chipscope-mcp's internal chipscopy LLM trigger-generation prompt template**: source for several TSM examples and rules above
- **chipscope-mcp의 내부 칩 복사 LLM 트리거 생성 프롬프트 템플릿**: 위의 여러 TSM 예제 및 규칙에 대한 소스
- **chipscope-mcp `tests/mcp/test_mcp_ila.py::test_advanced_trigger_state_machine`**: hardware-proven single-state TSM example, vmk180/vpk120/vck190
- **chipscope-mcp `tests/mcp/test_mcp_ila.py::test_advanced_trigger_state_machine`**: 하드웨어로 입증된 단일 상태 TSM 예, vmk180/vpk120/vck190
