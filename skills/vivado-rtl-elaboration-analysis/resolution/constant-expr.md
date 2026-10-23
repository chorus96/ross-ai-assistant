<!-- ko-translation: enabled -->
# Constant Expression Errors
# 상수 표현식 오류

**Covers IDs:** 530, 544, 545, 570, 579, 647, 648, 744
**표지 ID:** 530, 544, 545, 570, 579, 647, 648, 744

Grouped fixes for errors where a constant/parameter is required but a variable
상수/매개변수가 필요하지만 변수가 필요한 오류에 대한 그룹화된 수정 사항
or non-constant expression was used.
또는 상수가 아닌 표현식이 사용되었습니다.

---

## VLOG-530: Constant Variable Reassigned
## VLOG-530: 상수 변수가 재할당됨

```
CRITICAL WARNING: [Synth 8-530] constant variable 'DEPTH' is being reassigned [/path/to/file.sv:20]
```

**Fix:** Remove the reassignment — constants/localparams can only be assigned once:
**수정:** 재할당 제거 — 상수/localparams는 한 번만 할당할 수 있습니다.
```diff
  localparam DEPTH = 16;
- DEPTH = 32;                // can't reassign a constant
```

If the value needs to change, use a variable instead:
값을 변경해야 하는 경우 대신 변수를 사용하세요.
```diff
- localparam DEPTH = 16;
+ int DEPTH = 16;            // use variable if reassignment is needed
```

**VHDL equivalent:**
**VHDL에 해당:**
```diff
  constant DEPTH : integer := 16;
- DEPTH := 32;               -- can't reassign a constant
```
Use a variable in a process if reassignment is needed:
재할당이 필요한 경우 프로세스에서 변수를 사용하십시오.
```diff
- constant DEPTH : integer := 16;
+ variable depth : integer := 16;  -- variable allows reassignment
```

---

## VLOG-544 / VLOG-545: Replication Count Issues
## VLOG-544 / VLOG-545: 복제 횟수 문제

```
WARNING: [Synth 8-544] negative replication count [/path/to/file.v:25]
WARNING: [Synth 8-545] zero replication count in concatenation [/path/to/file.v:30]
```

**Fix (544):** Ensure replication count is positive:
**수정(544):** 복제 횟수가 양수인지 확인하세요.
```diff
- assign data = {(WIDTH-16){1'b0}, value};  // WIDTH < 16 → negative count
+ assign data = {(WIDTH > 16 ? WIDTH-16 : 0){1'b0}, value};
```

Or fix the parameter values to ensure positive count.
또는 매개변수 값을 수정하여 양수 개수를 보장합니다.

**Fix (545):** Zero replication creates a zero-width result — remove it:
**수정(545):** 제로 복제로 인해 너비가 0인 결과가 생성됩니다. 이를 제거하세요.
```diff
- assign data = {0{1'b0}, value};   // zero replication — empty
+ assign data = value;               // remove zero-width concat
```

---

## VLOG-570 / VLOG-579 / VHDL-744: Non-Constant Expression
## VLOG-570 / VLOG-579 / VHDL-744: 상수가 아닌 표현

```
ERROR: [Synth 8-570] generate expression is not constant [/path/to/file.v:35]
ERROR: [Synth 8-579] expression must be constant [/path/to/file.v:40]
ERROR: [Synth 8-744] constant required [/path/to/file.vhd:45]
```

These contexts require compile-time constants:
이러한 컨텍스트에는 컴파일 타임 상수가 필요합니다.
- `generate` conditions
- '생성' 조건
- Array/vector sizes
- 배열/벡터 크기
- Parameter values
- 매개변수 값
- Case expressions in `generate case`
- `케이스 생성`의 케이스 표현식

**Fix:** Replace the variable with a `parameter` or `localparam`:
**수정:** 변수를 '매개변수' 또는 'localparam'으로 바꾸세요.
```diff
- integer depth;
- genvar i;
- generate
-   for (i = 0; i < depth; i = i+1) begin  // 'depth' is not constant
+ localparam DEPTH = 8;
+ genvar i;
+ generate
+   for (i = 0; i < DEPTH; i = i+1) begin  // constant — OK
```

**VHDL:**
**VHDL:**
```diff
- variable depth : integer := 8;
- for i in 0 to depth-1 generate    -- 'depth' is not constant
+ constant DEPTH : integer := 8;
+ for i in 0 to DEPTH-1 generate    -- constant — OK
```

---

## VLOG-647 / VLOG-648: Non-Constant Parameter Override
## VLOG-647 / VLOG-648: 상수가 아닌 매개변수 재정의

```
CRITICAL WARNING: [Synth 8-647] hierarchical name in parameter not resolved [/path/to/file.v:50]
CRITICAL WARNING: [Synth 8-648] non-constant value used as parameter override [/path/to/file.v:55]
```

**Fix (647):** Replace hierarchical references with constant values:
**수정(647):** 계층적 참조를 상수 값으로 바꿉니다.
```diff
- sub_mod #(.WIDTH(top.u_other.WIDTH)) u1 (...);  // hierarchical — can't resolve
+ sub_mod #(.WIDTH(8)) u1 (...);                    // use constant value
```

Or pass through parameter chain:
또는 매개변수 체인을 통과합니다.
```diff
+ localparam SUB_WIDTH = 8;
+ sub_mod #(.WIDTH(SUB_WIDTH)) u1 (...);
```

**Fix (648):** Replace variable with constant:
**수정(648):** 변수를 상수로 교체:
```diff
- integer width_var = get_width();
- sub_mod #(.WIDTH(width_var)) u1 (...);  // variable — not constant
+ localparam WIDTH_VAL = 8;
+ sub_mod #(.WIDTH(WIDTH_VAL)) u1 (...);  // constant — OK
```

**VHDL equivalent for non-constant generics:**
**상수가 아닌 제네릭에 해당하는 VHDL:**
```diff
- u1 : entity work.sub_mod
-   generic map (WIDTH => some_signal)  -- signal is not constant
+ u1 : entity work.sub_mod
+   generic map (WIDTH => C_WIDTH)      -- use a constant or generic
```

---
