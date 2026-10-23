<!-- ko-translation: enabled -->
# Resolution: Uncovered Synth Messages (Not in Dispatch Table)
# 해결 방법: 발견되지 않은 Synth 메시지(디스패치 테이블에 없음)

## Scope
## 범위

This resolution file applies to **any** `[Synth 8-XXX]` message whose ID is NOT
이 해결 파일은 ID가 다음과 같은 **모든** `[Synth 8-XXX]` 메시지에 적용됩니다.
listed in the dispatch table in `message-handlers.md`.
`message-handlers.md`의 디스패치 테이블에 나열되어 있습니다.
These are typically
이들은 일반적으로
synthesis-phase optimization messages (IDs > 1000) that occur after the Verific
Verific 이후에 발생하는 합성 단계 최적화 메시지(ID > 1000)
elaboration front-end has completed.
정교화 프론트엔드가 완료되었습니다.

**Rule**: Even though these messages lack a specific resolution file, they MUST
**규칙**: 이러한 메시지에는 특정 해결 파일이 없더라도 반드시
still be reported with the full format: `###` section, File link, Problematic Code
여전히 전체 형식으로 보고됩니다: `###` 섹션, 파일 링크, 문제 코드
(diff block), and Recommended Fix (diff block) — one per CSV S.No.
(diff 블록) 및 권장 수정 사항(diff 블록) — CSV S.No.당 하나

---

## Common Patterns and Fix Templates
## 공통 패턴 및 수정 템플릿

### Pattern 1: Unused Sequential Element Removed (ID 6014)
### 패턴 1: 사용되지 않은 순차 요소 제거됨(ID 6014)

**Message**: `Unused sequential element <name>_reg was removed.`
**메시지**: `사용되지 않은 순차 요소 <name>_reg가 제거되었습니다.`

**Cause**: A register is assigned but its output `q` is never read in the current
**원인**: 레지스터가 할당되었지만 현재 출력 'q'는 읽히지 않습니다.
instantiation context.
인스턴스화 컨텍스트.
Synthesis optimizes it away.
합성은 그것을 최적화합니다.

**Analysis steps**:
**분석 단계**:
1. Read source at reported file:line — find the register assignment
1. 보고된 file:line에서 소스 읽기 — 레지스터 할당 찾기
2. Trace the register's output — is it supposed to drive something?
2. 레지스터의 출력을 추적합니다. 뭔가를 구동해야 합니까?
3. Check if the module uses parameters that conditionally use the register
3. 모듈이 조건부로 레지스터를 사용하는 매개변수를 사용하는지 확인하십시오.

**Problematic Code template**:
**문제가 있는 코드 템플릿**:
```diff
  // register is assigned but output never consumed
  always @(posedge clk) begin
-   if (enable)
-     unused_reg <= data_in;  // <- output of unused_reg is never read downstream
  end
```

**Recommended Fix options** (choose based on analysis):
**권장 수정 옵션**(분석에 따라 선택):

Option A — Register is intentionally unused (parameter-dependent path):
옵션 A - 레지스터가 의도적으로 사용되지 않습니다(매개변수 종속 경로):
```diff
  // Suppress warning: register is used when REG_Q=1 parameter is set
  always @(posedge clk) begin
+   (* dont_touch = "true" *)
    if (enable)
      unused_reg <= data_in;
  end
```

Option B — Register output should be connected:
옵션 B - 레지스터 출력을 연결해야 합니다.
```diff
  // Connect the register output to its intended consumer
- assign result = intermediate;
+ assign result = unused_reg;
```

Option C — Register is truly dead code, remove it:
옵션 C - 레지스터가 정말 데드 코드이므로 제거하세요.
```diff
  always @(posedge clk) begin
-   if (enable)
-     dead_reg <= data_in;  // <- remove dead register
  end
```

**Decision guidance**:
**결정 지침**:
- If module is parameterized and register is used in some configurations → Option A (waive)
- 모듈이 매개변수화되고 레지스터가 일부 구성에서 사용되는 경우 → 옵션 A(면제)
- If register was meant to be connected but connection is missing → Option B (fix)
- 레지스터를 연결하려고 했는데 연결이 누락된 경우 → 옵션 B(수정)
- If register serves no purpose in any configuration → Option C (remove)
- 어떤 구성에서도 레지스터가 아무런 용도로 사용되지 않는 경우 → 옵션 C(제거)

---

### Pattern 2: Undeclared Symbol Assumed Wire (ID 11241)
### 패턴 2: 선언되지 않은 기호 가정된 와이어(ID 11241)

**Message**: `undeclared symbol '<name>', assumed default net type 'wire'`
**메시지**: `선언되지 않은 기호 '<name>', 기본 네트 유형 'wire'로 가정`

**Cause**: A signal is used (typically in a port connection) without an explicit
**원인**: 명시적인 설명 없이 신호가 사용되었습니다(일반적으로 포트 연결에서).
`wire` or `logic` declaration.
`wire` 또는 `logic` 선언.
Vivado implicitly creates a 1-bit wire.
Vivado는 암시적으로 1비트 와이어를 생성합니다.

**Risk**: If the signal should be wider than 1 bit, implicit declaration causes
**위험**: 신호가 1비트보다 넓어야 하는 경우 암시적 선언으로 인해
silent truncation.
자동 잘림.

**Problematic Code template**:
**문제가 있는 코드 템플릿**:
```diff
  // Signal used without explicit declaration
  module_inst u_inst (
-   .o_data(undeclared_signal)  // <- no declaration; assumed 1-bit wire
  );
```

**Recommended Fix**:
**권장 수정**:
```diff
+ wire [WIDTH-1:0] undeclared_signal;  // <- add explicit declaration with correct width
+
  module_inst u_inst (
    .o_data(undeclared_signal)
  );
```

**Analysis steps**:
**분석 단계**:
1. Find the port connection using the undeclared signal
1. 선언되지 않은 신호를 사용하여 포트 연결 찾기
2. Look up the module definition to determine the port width
2. 포트 너비를 결정하려면 모듈 정의를 찾아보세요.
3. Add explicit wire declaration with matching width
3. 너비가 일치하는 명시적 와이어 선언 추가

---

### Pattern 3: Case Statement Not Full / No Default (ID 155)
### 패턴 3: 사례 설명이 완전하지 않음/기본값 없음(ID 155)

**Message**: `case statement is not full and has no default`
**메시지**: `case 문이 꽉 차 있지 않고 기본값이 없습니다`

**Cause**: A `case` statement in a combinational block does not cover all possible
**원인**: 조합 블록의 'case' 문은 가능한 모든 것을 다루지 않습니다.
input values and lacks a `default` branch.
값을 입력하고 '기본' 분기가 없습니다.
This infers a latch.
이것은 래치를 추론합니다.

**Problematic Code template**:
**문제가 있는 코드 템플릿**:
```diff
  always_comb begin
    case (sel)
      2'b00: out = a;
      2'b01: out = b;
-     // missing 2'b10, 2'b11 and no default  // <- infers latch
    endcase
  end
```

**Recommended Fix**:
**권장 수정**:
```diff
  always_comb begin
    case (sel)
      2'b00: out = a;
      2'b01: out = b;
+     default: out = '0;  // <- prevents latch inference
    endcase
  end
```

---

### Pattern 4: Informational / Progress Messages (IDs 6155, 6157, 7075, 7078, 7079, 3876)
### 패턴 4: 정보/진행 메시지(ID 6155, 6157, 7075, 7078, 7079, 3876)

**Messages**: "synthesizing module", "done synthesizing module", "Helper process",
**메시지**: "합성 모듈", "합성 모듈 완료", "도우미 프로세스",
"Multithreading enabled", "$readmem file read"
"멀티스레딩 활성화", "$readmem 파일 읽기"

**Action**: These are purely informational.
**조치**: 이는 순전히 정보 제공용입니다.
They are classified as INFO severity and
INFO 심각도로 분류됩니다.
excluded from the report.
보고서에서 제외됩니다.
No Problematic Code or Recommended Fix needed.
문제가 있는 코드나 권장 수정이 필요하지 않습니다.

**Rule**: Only process INFO messages if they indicate a code quality issue
**규칙**: 코드 품질 문제를 나타내는 경우에만 INFO 메시지를 처리하세요.
(e.g., 11241, 155).
Progress/system INFOs are skipped.
진행률/시스템 정보를 건너뜁니다.

---

## Reporting Format (MANDATORY)
## 보고 형식(필수)

For each non-INFO message whose ID is not in the dispatch table, write:
ID가 디스패치 테이블에 없는 각 비INFO 메시지에 대해 다음을 작성하십시오.

```markdown
### [Synth 8-XXXX] <message text>
**CSV S.No:** <number>
**File:** [filename.sv](CSV_File_path#Lnn)
**Message:** <full message text>
**Root Cause:** <explain why this message occurs based on reading the source>

**Problematic Code:**
\`\`\`diff
  context line from read_file
- problematic line  // <- explanation
  context line
\`\`\`

**Recommended Fix:**
\`\`\`diff
  context line
+ fixed/improved line  // <- what changed and why
  context line
\`\`\`

**Rationale:** <why this fix is correct, or why waiving is appropriate>
```

**NEVER skip the diff blocks.** Even if the recommendation is "waive this warning",
**절대 diff 블록을 건너뛰지 마세요.** 권장 사항이 "이 경고 무시"인 경우에도
show the code and explain why waiving is appropriate using a suppress directive:
코드를 보여주고 억제 지시문을 사용하여 포기가 적절한 이유를 설명하십시오.
```diff
+ (* synthesis, keep *)  // Intentionally unused in this configuration
```

---

## Dispatch Table Addition
## 디스패치 테이블 추가

Add the following to `message-handlers.md` under a new section:
새 섹션 아래 `message-handlers.md`에 다음을 추가합니다.

```markdown
### Synthesis Optimization Phase (Not Verific Front-End)

| ID | Severity | Short Name | Fix Strategy |
|----|----------|------------|--------------|
| 155 | info/warning | case not full | Add `default` branch → `resolution/uncovered-synth-messages.md` |
| 6014 | warning | unused register removed | Connect output or waive → `resolution/uncovered-synth-messages.md` |
| 11241 | info/warning | implicit wire | Add explicit declaration → `resolution/uncovered-synth-messages.md` |
```
