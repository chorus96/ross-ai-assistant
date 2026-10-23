<!-- ko-translation: enabled -->
# Case Statement Errors
# 사례 설명 오류

**Covers IDs:** 150, 506, 507, 508, 746, 750, 814, 838, 849
**표지 ID:** 150, 506, 507, 508, 746, 750, 814, 838, 849

Grouped fixes for case statement issues — duplicate, overlapping, unreachable, and
사례 설명 문제에 대한 그룹화된 수정 사항(중복, 중복, 접근 불가,
out-of-range choices.
범위를 벗어난 선택.

---

## VLOG-150: Multiple Defaults
## VLOG-150: 다중 기본값

```
ERROR: [Synth 8-150] multiple defaults in case statement [/path/to/file.v:50]
```

**Fix:** Keep one `default` block, merge the logic:
**수정:** 하나의 '기본' 블록을 유지하고 로직을 병합합니다.
```diff
  case (sel)
    2'b00: out = a;
    2'b01: out = b;
    default: out = c;
-   default: out = d;        // remove second default
  endcase
```

---

## VLOG-506: Case Item Never Executed
## VLOG-506: 케이스 항목이 실행되지 않음

```
WARNING: [Synth 8-506] case item 'xx' is never executed [/path/to/file.v:55]
```

**Fix:** Remove the dead case item or fix its value:
**수정:** 데드 케이스 항목을 제거하거나 해당 값을 수정하세요.
```diff
  case (sel)     // sel is 2-bit
    2'b00: out = a;
    2'b01: out = b;
    2'b10: out = c;
    2'b11: out = d;
-   2'bxx: out = e;    // never matches in synthesis — remove
  endcase
```

For `casex`/`casez` patterns, `x`/`z` are wildcards but plain `case` treats them as
`casex`/`casez` 패턴의 경우 `x`/`z`는 와일드카드이지만 일반 `case`는 이를 다음과 같이 처리합니다.
unmatchable values.
일치하지 않는 값.

---

## VLOG-507: Unreachable Case Item
## VLOG-507: 연결할 수 없는 케이스 항목

```
WARNING: [Synth 8-507] unreachable case item [/path/to/file.v:60]
```

**Fix:** A previous item already covers this value.
**수정:** 이전 항목에 이미 이 값이 포함되어 있습니다.
Remove or reorder:
제거 또는 재정렬:
```diff
  casez (sel)
    4'b1???: out = a;    // matches everything starting with 1
-   4'b1100: out = b;    // unreachable — already matched above
    4'b0???: out = c;
  endcase
```

---

## VLOG-508: Overlapping Case Items
## VLOG-508: 케이스 항목 중복

```
WARNING: [Synth 8-508] overlapping case item values detected [/path/to/file.v:65]
```

**Fix:** Make case items mutually exclusive:
**수정:** 케이스 항목을 상호 배타적으로 만듭니다.
```diff
  casez (sel)
-   4'b1???: out = a;
-   4'b11??: out = b;     // overlaps with 1???
+   4'b10??: out = a;     // non-overlapping
+   4'b11??: out = b;     // non-overlapping
    default: out = c;
  endcase
```

Or use `priority case` / `unique case` (SystemVerilog) for intended priority encoding.
또는 의도된 우선순위 인코딩에 '우선순위 케이스' / '고유 케이스'(SystemVerilog)를 사용하십시오.

---

## VHDL-746: Overlapping Case Choice
## VHDL-746: 중복 케이스 선택

```
ERROR: [Synth 8-746] overlapping case choice [/path/to/file.vhd:70]
```

**Fix:** Remove or modify the overlapping choice:
**수정:** 겹치는 선택 항목을 제거하거나 수정합니다.
```diff
  case sel is
    when "00" to "10" => out <= a;
-   when "01"         => out <= b;    -- overlaps with "00" to "10"
+   when "11"         => out <= b;    -- non-overlapping value
    when others       => out <= c;
  end case;
```

---

## VHDL-750 / VHDL-814: Duplicate Choice
## VHDL-750 / VHDL-814: 중복 선택

```
ERROR: [Synth 8-750] duplicate choice in aggregate [/path/to/file.vhd:75]
ERROR: [Synth 8-814] previously used choice [/path/to/file.vhd:80]
```

**Fix:** Remove the duplicate choice value:
**수정:** 중복된 선택 값을 제거합니다.
```diff
  case state is
    when IDLE => next <= RUN;
    when RUN  => next <= DONE;
-   when RUN  => next <= ERR;     -- duplicate — remove
    when DONE => next <= IDLE;
    when others => next <= IDLE;
  end case;
```

---

## VHDL-838 / VLOG-849: Choice/Width Out of Range
## VHDL-838 / VLOG-849: 선택/폭이 범위를 벗어났습니다.

```
WARNING: [Synth 8-838] case choice value '16' is out of range [/path/to/file.vhd:85]
WARNING: [Synth 8-849] case expression and choice width mismatch [/path/to/file.vhd:90]
```

**Fix (838):** Adjust choice to be within the select expression's range:
**수정(838):** 선택 표현식의 범위 내에 있도록 선택 항목을 조정합니다.
```diff
  -- sel is integer range 0 to 7
  case sel is
    when 0 => out <= a;
-   when 16 => out <= b;     -- out of range (0 to 7)
+   when 7  => out <= b;     -- within range
    when others => out <= c;
  end case;
```

**Fix (849):** Match choice width to case expression width:
**수정(849):** 선택 폭을 대소문자 표현 폭과 일치시킵니다.
```diff
  -- sel is std_logic_vector(3 downto 0) — 4 bits
  case sel is
-   when "00000" => out <= a;     -- 5 bits, should be 4
+   when "0000"  => out <= a;     -- 4 bits — matches sel width
    when others => out <= c;
  end case;
```

---
