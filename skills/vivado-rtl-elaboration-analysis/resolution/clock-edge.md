<!-- ko-translation: enabled -->
# Clock Edge and Event Control Issues
# 클록 에지 및 이벤트 제어 문제

Resolution guide for clock/edge-related elaboration messages.
클럭/에지 관련 상세 메시지에 대한 해결 가이드입니다.

## Covered Messages
## 해당 메시지

| ID | Severity<br>심각도 | Message Pattern<br>메시지 패턴 |
|----|----------|----------------|
| 560 | ERROR<br>오류 | mixed level and edge expression list<br>혼합 레벨 및 에지 표현식 목록 |
| 561 | ERROR<br>오류 | no clock in event control<br>이벤트 제어에 시계가 없습니다. |
| 562 | ERROR<br>오류 | ambiguous clock in event control<br>이벤트 제어의 모호한 시계 |
| 563 | WARNING<br>경고 | more than one event control statement — only the first is synthesizable<br>둘 이상의 이벤트 제어문 - 첫 번째만 합성 가능 |

---

## [Synth 8-560] Mixed level and edge expression list
## [Synth 8-560] 레벨 및 에지 혼합 표현 목록

### Vivado Message
### 비바도 메시지
```
ERROR: [Synth 8-560] mixed level and edge expression list [file:line]
```

### Root Cause
### 근본 원인
The sensitivity list mixes edge-triggered signals (e.g., `posedge clk`) with
민감도 목록은 에지 트리거 신호(예: `posedge clk`)를 다음과 혼합합니다.
level-sensitive signals (e.g., `data`).
레벨에 민감한 신호(예: '데이터').
This is not synthesizable — an always
이것은 합성할 수 없습니다. 항상
block must be **either** sequential (all edges) or combinational (all levels).
블록은 **순차적**(모든 가장자리)이거나 조합적(모든 수준)이어야 합니다.

### Fix Options
### 수정 옵션

**Option A — Sequential block (remove level signals):**
**옵션 A - 순차 차단(레벨 신호 제거):**
```diff
- always @(posedge clk or data) begin  // <- mixed edge + level
+ always @(posedge clk) begin          // <- edge-only (sequential)
    if (data)
      q <= d;
  end
```

**Option B — Async reset pattern (edge for both clock and reset):**
**옵션 B - 비동기 재설정 패턴(클럭 및 재설정 모두에 대한 에지):**
```diff
- always @(posedge clk or rst) begin   // <- 'rst' is level, not edge
+ always @(posedge clk or posedge rst) begin  // <- edge for both
    if (rst)
      q <= '0;
    else
      q <= d;
  end
```

**VHDL equivalent:**
**VHDL에 해당:**
```diff
- process(clk, data)   -- level-sensitive data mixed with clocked logic
+ process(clk)         -- sequential: clock only
  begin
    if rising_edge(clk) then
      if data = '1' then
        q <= d;
      end if;
    end if;
  end process;
```

### Rationale
### 이론적 근거
Synthesis requires deterministic register inference.
합성에는 결정론적 레지스터 추론이 필요합니다.
Mixed level/edge prevents
혼합 레벨/에지 방지
the tool from deciding whether to create a flip-flop or a latch.
플립플롭을 생성할지 래치를 생성할지 결정하는 도구는 없습니다.

---

## [Synth 8-561] No clock in event control
## [Synth 8-561] 이벤트 제어에 시계가 없습니다.

### Vivado Message
### 비바도 메시지
```
ERROR: [Synth 8-561] no clock in event control [file:line]
```

### Root Cause
### 근본 원인
An always block uses edge notation (`posedge`/`negedge`) in the sensitivity list,
항상 블록은 민감도 목록에서 가장자리 표기법(`posedge`/`negedge`)을 사용합니다.
but no identifiable clock signal was found.
그러나 식별 가능한 클럭 신호는 발견되지 않았습니다.
Typically caused by an empty or
일반적으로 비어 있거나
malformed sensitivity list.
잘못된 민감도 목록.

### Fix Options
### 수정 옵션

**Option A — Add clock signal:**
**옵션 A - 클럭 신호 추가:**
```diff
- always @(posedge) begin              // <- no signal name after posedge
+ always @(posedge clk) begin          // <- specify the clock signal
    q <= d;
  end
```

**Option B — Convert to combinational if no clock intended:**
**옵션 B — 시계를 의도하지 않은 경우 조합으로 변환:**
```diff
- always @(posedge) begin
+ always @(*) begin                    // <- combinational, no clock needed
    q = d;                             // <- blocking assignment for comb
  end
```

### Rationale
### 이론적 근거
An edge-triggered block requires a clock signal for register inference.
에지 트리거 블록에는 레지스터 추론을 위한 클럭 신호가 필요합니다.
Without
없이
a clock, the synthesis tool cannot determine what drives the registers.
클럭이 있는 경우 합성 도구는 레지스터를 구동하는 것이 무엇인지 결정할 수 없습니다.

---

## [Synth 8-562] Ambiguous clock in event control
## [Synth 8-562] 이벤트 제어의 시계가 모호함

### Vivado Message
### 비바도 메시지
```
ERROR: [Synth 8-562] ambiguous clock in event control [file:line]
```

### Root Cause
### 근본 원인
Multiple signals with `posedge`/`negedge` in the sensitivity list, and the tool
민감도 목록에 `posedge`/`negedge`가 있는 여러 신호 및 도구
cannot determine which is the clock vs. async control.
시계 제어와 비동기 제어 중 어느 것을 결정할 수 없습니다.
This happens when:
이는 다음과 같은 경우에 발생합니다.
- Two or more edge events with no clear if/else structure to disambiguate
- 명확하게 구분할 수 있는 명확한 if/else 구조가 없는 두 개 이상의 엣지 이벤트
- Missing priority check for the async signal
- 비동기 신호에 대한 우선순위 확인 누락

### Fix Options
### 수정 옵션

**Option A — Use standard async reset pattern (reset checked first in if):**
**옵션 A - 표준 비동기 재설정 패턴 사용(if에서 재설정이 먼저 확인됨):**
```diff
  always @(posedge clk or posedge rst) begin
-   q <= d;                            // <- no if/else, ambiguous
+   if (rst)                           // <- async signal checked FIRST
+     q <= '0;
+   else
+     q <= d;                          // <- clocked path in else
  end
```

**Option B — Remove extra edge (keep single clock):**
**옵션 B - 추가 가장자리 제거(단일 시계 유지):**
```diff
- always @(posedge clk or posedge enable) begin  // <- two edges, ambiguous
+ always @(posedge clk) begin                    // <- single clock
    if (enable)
      q <= d;
  end
```

**VHDL equivalent:**
**VHDL에 해당:**
```diff
  process(clk, rst)
  begin
-   q <= d;                         -- ambiguous: no rising_edge/reset check
+   if rst = '1' then               -- async reset checked first
+     q <= '0';
+   elsif rising_edge(clk) then     -- clock in elsif
+     q <= d;
+   end if;
  end process;
```

### Rationale
### 이론적 근거
When multiple edges are present, the synthesis tool expects a standard async
여러 가장자리가 있는 경우 합성 도구는 표준 비동기를 기대합니다.
reset pattern: the async signal is checked first (highest priority `if`), and
패턴 재설정: 비동기 신호가 먼저 확인됩니다(가장 높은 우선순위 `if`).
the clock path is in the `else`/`elsif rising_edge()` branch.
클럭 경로는 `else`/`elsifrising_edge()` 분기에 있습니다.
Any other structure
다른 구조
creates ambiguity about which signal is the clock.
어떤 신호가 시계인지 모호해집니다.

---

## [Synth 8-563] More than one event control statement
## [Synth 8-563] 하나 이상의 이벤트 제어문

### Vivado Message
### 비바도 메시지
```
WARNING: [Synth 8-563] more than one event control statement in this 'always' block — only the first is synthesizable [file:line]
```

### Root Cause
### 근본 원인
The always block contains multiple `@(...)` event controls.
Always 블록에는 여러 개의 `@(...)` 이벤트 컨트롤이 포함되어 있습니다.
Only the first
첫 번째만
determines the sensitivity.
감도를 결정합니다.
The rest are simulation-only constructs (delays,
나머지는 시뮬레이션 전용 구성(지연,
wait statements) that synthesis ignores.
대기 문) 합성이 무시합니다.

### Fix Options
### 수정 옵션

**Option A — Remove internal event controls:**
**옵션 A - 내부 이벤트 컨트롤 제거:**
```diff
  always @(posedge clk) begin
    a <= b;
-   @(posedge clk);                 // <- second event control (ignored)
-   c <= d;                         // <- this assignment is unpredictable
+   c <= d;                         // <- moves to same clock edge
  end
```

**Option B — Split into separate always blocks for multi-cycle:**
**옵션 B — 다중 주기를 위해 별도의 Always 블록으로 분할:**
```diff
- always @(posedge clk) begin
-   a <= b;
-   @(posedge clk);                 // <- waiting for next clock (sim only)
-   c <= a;
- end
+ always @(posedge clk) begin
+   a <= b;
+ end
+
+ // Pipeline stage 2: one cycle later
+ always @(posedge clk) begin
+   c <= a;                          // <- a is already registered, so c = b delayed 2 cycles
+ end
```

### Rationale
### 이론적 근거
Synthesis flattens all logic into a single clock domain evaluation.
합성은 모든 로직을 단일 클록 도메인 평가로 평면화합니다.
Multiple event
다중 이벤트
controls within one always block are a simulation timing construct with no hardware
하나의 Always 블록 내의 컨트롤은 하드웨어가 없는 시뮬레이션 타이밍 구성입니다.
equivalent.
동등하다.
Use separate always blocks or registered pipelines for multi-cycle behavior.
다중 주기 동작을 위해 별도의 항상 블록 또는 등록된 파이프라인을 사용합니다.

---

## References
## 참고자료

- UG901 Ch.4 — Combinational vs Sequential Coding
- UG901 Ch.4 — 조합 코딩과 순차 코딩
- IEEE 1364-2005 §9.7.5 — Event Control
- IEEE 1364-2005 §9.7.5 — 이벤트 제어
- IEEE 1800-2017 §9.4.2 — Event Control
- IEEE 1800-2017 §9.4.2 — 이벤트 제어
- IEEE 1076-2008 §11.3 — Process Sensitivity
- IEEE 1076-2008 §11.3 — 프로세스 민감도
