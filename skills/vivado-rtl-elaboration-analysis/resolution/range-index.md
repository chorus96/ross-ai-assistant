<!-- ko-translation: enabled -->
# Range and Index Errors
# 범위 및 인덱스 오류

**Covers IDs:** 547, 548, 549, 742, 745, 766, 769, 803, 806, 825, 827, 869
**표지 ID:** 547, 548, 549, 742, 745, 766, 769, 803, 806, 825, 827, 869

Grouped fixes for out-of-range indices, slice direction mismatches, null ranges,
범위를 벗어난 인덱스, 슬라이스 방향 불일치, Null 범위에 대한 그룹화된 수정 사항
and integer overflow errors.
및 정수 오버플로 오류.

---

## VLOG-547 / VHDL-742: Index Out of Range
## VLOG-547 / VHDL-742: 인덱스가 범위를 벗어났습니다.

```
WARNING: [Synth 8-547] index value '8' is out of range for 'data[7:0]' [/path/to/file.v:30]
ERROR: [Synth 8-742] index value '8' out of range for 'data' [/path/to/file.vhd:35]
```

**Fix:** Adjust the index to be within the declared range:
**수정:** 선언된 범위 내에 있도록 인덱스를 조정합니다.
```diff
  // data is [7:0]
- assign bit = data[8];       // out of range
+ assign bit = data[7];       // within range
```

Or widen the declaration if the higher index is needed:
또는 더 높은 인덱스가 필요한 경우 선언을 넓힙니다.
```diff
- wire [7:0] data;
+ wire [8:0] data;            // widen to accommodate index 8
```

---

## VLOG-548: Part-Select Out of Range
## VLOG-548: 부품 선택이 범위를 벗어났습니다.

```
ERROR: [Synth 8-548] part-select [15:8] out of range for 'data[7:0]' [/path/to/file.v:35]
```

**Fix:** Adjust the slice to fit within declaration bounds:
**수정:** 선언 범위 내에 맞게 슬라이스를 조정합니다.
```diff
  // data is [7:0]
- assign byte1 = data[15:8];    // out of range
+ assign byte1 = data[7:0];     // within range
```

Or widen the source:
또는 소스를 넓히십시오.
```diff
- wire [7:0]  data;
+ wire [15:0] data;
```

---

## VLOG-549 / VHDL-745: Slice Direction Mismatch
## VLOG-549 / VHDL-745: 슬라이스 방향 불일치

```
ERROR: [Synth 8-549] part-select direction mismatch for 'data' [/path/to/file.v:40]
ERROR: [Synth 8-745] slice direction mismatch for 'data' [/path/to/file.vhd:45]
```

**Fix:** Match the slice direction to the declaration:
**수정:** 슬라이스 방향을 선언과 일치시킵니다.

**Verilog** (declared `[7:0]` — descending):
**Verilog** (`[7:0]` 선언 — 내림차순):
```diff
- assign nibble = data[0:3];    // ascending — mismatch
+ assign nibble = data[3:0];    // descending — matches declaration
```

**VHDL** (declared `downto`):
**VHDL**('downto'로 선언됨):
```diff
- nibble <= data(0 to 3);       -- ascending — mismatch
+ nibble <= data(3 downto 0);   -- downto — matches declaration
```

---

## VHDL-766 / VHDL-803 / VHDL-806 / VHDL-825: Value Out of Range
## VHDL-766 / VHDL-803 / VHDL-806 / VHDL-825: 값이 범위를 벗어났습니다.

```
ERROR: [Synth 8-766] expression out of range [/path/to/file.vhd:50]
ERROR: [Synth 8-803] integer value '300' out of type range [/path/to/file.vhd:55]
ERROR: [Synth 8-806] integer overflow [/path/to/file.vhd:60]
ERROR: [Synth 8-825] value out of allowable range [/path/to/file.vhd:65]
```

**Fix:** Clamp the value to the type's range or widen the type:
**수정:** 유형 범위에 맞게 값을 제한하거나 유형을 넓힙니다.
```diff
  -- signal counter : integer range 0 to 255
- counter <= 300;                 -- out of range
+ counter <= 255;                 -- clamp to max
```

Or widen the type:
또는 유형을 넓힙니다.
```diff
- signal counter : integer range 0 to 255;
+ signal counter : integer range 0 to 511;
```

---

## VHDL-769: Null Range
## VHDL-769: 널 범위

```
CRITICAL WARNING: [Synth 8-769] null range detected [/path/to/file.vhd:70]
```

A null range occurs when the left bound equals or crosses the right bound in the
Null 범위는 왼쪽 경계가 오른쪽 경계와 같거나 교차할 때 발생합니다.
wrong direction (e.g., `7 to 0` for a `downto` type, or `0 downto 7` for a `to` type).
잘못된 방향입니다(예: `downto` 유형의 경우 `7 to 0`, `to` 유형의 경우 `0 downto 7`).

**Fix:** Correct the range direction:
**수정:** 범위 방향 수정:
```diff
- signal data : std_logic_vector(0 downto 7);   -- null range!
+ signal data : std_logic_vector(7 downto 0);   -- correct direction
```

Or fix swapped bounds:
또는 교체된 경계를 수정합니다.
```diff
- signal data : std_logic_vector(0 to 0);       -- only 1 bit? check intent
+ signal data : std_logic_vector(7 downto 0);   -- intended 8-bit signal
```

---

## VHDL-827: Aggregate Range Direction Mismatch
## VHDL-827: 집계 범위 방향 불일치

```
ERROR: [Synth 8-827] aggregate range direction mismatch [/path/to/file.vhd:80]
```

**Fix:** Match the aggregate range direction to the target type:
**수정:** 집계 범위 방향을 대상 유형과 일치시킵니다.
```diff
  -- target is std_logic_vector(7 downto 0)
- data <= (0 to 7 => '0');         -- 'to' doesn't match 'downto'
+ data <= (7 downto 0 => '0');     -- matches target direction
```

Or use positional/named association:
또는 위치/이름이 지정된 연관을 사용하십시오.
```diff
+ data <= (others => '0');          -- direction-agnostic
```

---

## VHDL-869: Out of Bounds Slice
## VHDL-869: 범위를 벗어난 슬라이스

```
ERROR: [Synth 8-869] slice is out of bounds [/path/to/file.vhd:85]
```

**Fix:** Adjust the slice to fit within the array bounds:
**수정:** 배열 경계 내에 맞게 슬라이스를 조정합니다.
```diff
  -- data is std_logic_vector(7 downto 0)
- nibble <= data(11 downto 8);     -- out of bounds
+ nibble <= data(7 downto 4);      -- within bounds
```

---
