<!-- ko-translation: enabled -->
---
name: hw-ila-debug
description: >-
  Interact with ILA debug cores via vivado-mcp or chipscope-mcp.
  Discover cores, configure triggers,
  arm, capture, upload, export waveform data (CSV/VCD), and analyze results.
  ILA in vivado-mcp supports every device family supported by Vivado.
  chipscope-mcp supports Versal only.
  Use when user asks to "capture ILA data", "set ILA trigger", "trigger on signal",
  "export waveform", "arm ILA", or "trigger immediately".
license: MIT
compatibility: Requires Vivado 2026.1+, hardware with an ILA core, and access to either vivado-mcp or chipscope-mcp.
metadata:
  version: "1.2.2"
  stage: beta
---

<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->
# ILA Debug (EA)
# ILA 디버그(EA)

> **Early Access** — this skill may change before general availability.
> **조기 액세스** — 이 스킬은 일반 출시되기 전에 변경될 수 있습니다.

Interacts with ILA debug cores on live hardware: discover → configure trigger → arm → capture → upload → export → analyze.
라이브 하드웨어에서 ILA 디버그 코어와 상호 작용합니다. 검색 → 트리거 구성 → 준비 → 캡처 → 업로드 → 내보내기 → 분석.

See [REFERENCE.md](REFERENCE.md) for ILA property tables, Tcl command reference, TSM syntax, compare value encoding, and report schema.
ILA 속성 테이블, Tcl 명령 참조, TSM 구문, 비교 값 인코딩 및 보고서 스키마는 [REFERENCE.md](REFERENCE.md)를 참조하세요.

**Routing:** Try Vivado MCP (`vivado_execute` + Hardware Manager Tcl, below) first if a Vivado MCP session is available — it works across all device families.
**라우팅:** Vivado MCP 세션을 사용할 수 있는 경우 먼저 Vivado MCP(아래 `vivado_execute` + 하드웨어 관리자 Tcl)를 사용해 보세요. 이는 모든 장치 제품군에서 작동합니다.
Fall back to chipscope-mcp (`chipscope_ila_core`/`chipscope_ila_capture`) if Vivado MCP is not installed.
Vivado MCP가 설치되지 않은 경우 Chipscope-mcp(`chipscope_ila_core`/`chipscope_ila_capture`)로 대체됩니다.
See [Chipscope-mcp Fallback](#chipscope-mcp-fallback) below.
아래의 [Chipscope-mcp 폴백](#chipscope-mcp-fallback)을 참조하세요.

## Tools Used
## 사용된 도구

**Primary path — Vivado MCP** (plus the agent's own file tools):
**기본 경로 — Vivado MCP** (에이전트 자체 파일 도구 포함):

| Tool<br>도구 | Purpose<br>목적 |
|------|---------|
| `vivado_execute`<br>'vivado_execute' | Run all Tcl commands (`get_hw_ilas`, `run_hw_ila`, etc.). Every Tcl block MUST be run via `vivado_execute(session_id=..., command=<tcl>)`.<br>모든 Tcl 명령(`get_hw_ilas`, `run_hw_ila` 등)을 실행합니다. 모든 Tcl 블록은 `vivado_execute(session_id=..., command=<tcl>)`를 통해 실행되어야 합니다. |
| `vivado_doc_search`<br>`vivado_doc_search` | Look up unfamiliar ILA properties, probe compare value syntax, TSM syntax. No `session_id` needed.<br>익숙하지 않은 ILA 속성을 찾아보고 비교 값 구문, TSM 구문을 검색하세요. 'session_id'가 필요하지 않습니다. |
| `vivado_status`<br>`vivado_status` | Check Vivado session health.<br>Vivado 세션 상태를 확인하세요. |
| `vivado_list_sessions`<br>`vivado_list_sessions` | Find an active session to use (get its `session_id`).<br>사용할 활성 세션을 찾습니다(`session_id` 가져오기). |
| Agent file tools<br>에이전트 파일 도구 | Write output files (report_data.json, REPORT.md, CSV/VCD exports).<br>출력 파일을 작성합니다(report_data.json, REPORT.md, CSV/VCD 내보내기). |

**Fallback path — chipscope-mcp** (when Vivado MCP is unavailable): `chipscope_session`, `chipscope_device`, `chipscope_scan`, `chipscope_ila_core`, `chipscope_ila_capture`.
**대체 경로 — Chipscope-mcp**(Vivado MCP를 사용할 수 없는 경우): `chipscope_session`, `chipscope_device`, `chipscope_scan`, `chipscope_ila_core`, `chipscope_ila_capture`.
See [Chipscope-mcp Fallback](#chipscope-mcp-fallback).
[Chipscope-mcp 폴백](#chipscope-mcp-fallback)을 참조하세요.

**No terminal commands, no Python.** Everything goes through Vivado MCP or, as fallback, chipscope-mcp.
**터미널 명령도 없고 Python도 없습니다.** 모든 것이 Vivado MCP를 통과하거나 대체적으로 Chipscope-mcp를 통과합니다.

---

## Session Setup: Vivado Version Must Match hw_server
## 세션 설정: Vivado 버전은 hw_server와 일치해야 합니다.

- `vivado_list_sessions` → check for an active session.
- `vivado_list_sessions` → 활성 세션을 확인하세요.
- Before reusing one, run `version -short` and compare to the target `hw_server`'s version.
- 재사용하기 전에 `version -short`를 실행하고 대상 `hw_server`의 버전과 비교하세요.
- Mismatch → `ERROR: [Labtools 27-3366] Cannot support older hw_server version`.
- 불일치 → '오류: [Labtools 27-3366] 이전 hw_server 버전을 지원할 수 없습니다.'
- **On mismatch: do not retry or restart automatically.** `vivado_start` does not pick a
- **불일치 시: 재시도하지 않거나 자동으로 다시 시작하지 마세요.** `vivado_start`는
matching Vivado version for you — an omitted `vivado_path` just resolves from `PATH`,
Vivado 버전과 일치합니다. 생략된 `vivado_path`는 `PATH`에서 확인됩니다.
which is often the same mismatched install.
이는 종종 동일하게 일치하지 않는 설치입니다.
Report the exact error, the session's
정확한 오류를 보고하세요. 세션의
version, and the target `hw_server`'s version to the user, and ask them to point at
버전과 대상 `hw_server`의 버전을 사용자에게 알려주고 다음 위치를 가리키도록 요청합니다.
the correct Vivado install (`vivado_path`) or the correct `hw_server`.
올바른 Vivado 설치(`vivado_path`) 또는 올바른 `hw_server`.
- This only applies to an **already-running, independently-started** `hw_server`.
- 이는 **이미 실행 중이고 독립적으로 시작된** `hw_server`에만 적용됩니다.
- A
`hw_server` that Vivado auto-starts itself when connecting to a locally attached board
로컬로 연결된 보드에 연결할 때 Vivado가 자동으로 시작되는 'hw_server'
is launched from that same Vivado install, so it cannot be version-mismatched — see
동일한 Vivado 설치에서 시작되므로 버전이 일치하지 않을 수 있습니다.
"No hw_server" below.
아래에 "hw_server 없음"이 있습니다.
- Only reuse a session once its version matches.
- 버전이 일치하는 경우에만 세션을 재사용하세요.

---

## Efficiency Guidelines
## 효율성 지침

- **Pass `session_id`** to every `vivado_execute` call — get it from `vivado_list_sessions` if unknown.
- **`session_id`**를 모든 `vivado_execute` 호출에 전달합니다. 알 수 없는 경우 `vivado_list_sessions`에서 가져옵니다.
- **Verify session version against `hw_server` before reuse, and report a mismatch to the user rather than retrying** — see [Session Setup](#session-setup-vivado-version-must-match-hw_server) above.
- **재사용하기 전에 'hw_server'에 대해 세션 버전을 확인하고, 재시도보다는 사용자에게 불일치를 보고합니다** — 위의 [세션 설정](#session-setup-vivado-version-must-match-hw_server)을 참조하세요.
- **Batch Tcl** — discover all ILA cores in one `vivado_execute`, not separate calls.
- **Batch Tcl** — 별도의 호출이 아닌 하나의 `vivado_execute`로 모든 ILA 코어를 검색합니다.
- **Write reports to file** — do not dump full waveform data in chat.
- **파일에 보고서 쓰기** — 채팅에서 전체 파형 데이터를 덤프하지 않습니다.
- Give a short summary only.
- 간략한 요약만 제공하세요.
- **Use `vivado_doc_search`** for unfamiliar probe properties, TSM syntax, or compare value encoding.
- **익숙하지 않은 프로브 속성, TSM 구문 또는 비교 값 인코딩의 경우 `vivado_doc_search`**를 사용하세요.
- **Do NOT** use `display_hw_ila_data` — it opens a GUI window.
- **사용하지 마세요** `display_hw_ila_data`를 사용하면 GUI 창이 열립니다.
- Use `write_hw_ila_data` + `list_hw_samples` for data extraction.
- 데이터 추출에는 `write_hw_ila_data` + `list_hw_samples`를 사용하세요.
- **Do NOT** use `shell` commands or Vivado Tcl file I/O.
- **쉘` 명령이나 Vivado Tcl 파일 I/O를 사용하지 마십시오**.
- Use agent file tools.
- 에이전트 파일 도구를 사용하십시오.
- **Do NOT** retry failed Tcl with different syntax.
- **하지 마세요** 다른 구문으로 실패한 Tcl을 다시 시도하세요.
- Search docs first, then report the error.
- 먼저 문서를 검색한 후 오류를 신고하세요.

---

## Capture & Analysis Workflow
## 캡처 및 분석 워크플로우

### Step 1: Discover ILA Cores & Probes
### 1단계: ILA 코어 및 프로브 검색

Run via `vivado_execute(session_id=<id>, command=<tcl below>)`:
`vivado_execute(session_id=<id>, command=<tcl below>)`를 통해 실행:

```tcl
puts "=== ILA CORES ==="
foreach ila [get_hw_ilas] {
  puts "ILA:$ila"
  puts "  DATA_DEPTH:[get_property CONTROL.DATA_DEPTH $ila]"
  puts "  MAX_DEPTH:[get_property STATIC.MAX_DATA_DEPTH $ila]"
  puts "  TRIGGER_MODE:[get_property CONTROL.TRIGGER_MODE $ila]"
  puts "  ADV_TRIGGER:[get_property STATIC.IS_ADVANCED_TRIGGER_MODE_SUPPORTED $ila]"
  puts "  TRIG_IN:[get_property STATIC.IS_TRIG_IN_SUPPORTED $ila]"
  puts "  CAPTURE_QUAL:[get_property STATIC.IS_BASIC_CAPTURE_MODE_SUPPORTED $ila]"
  puts "  STATUS:[get_property STATUS.CORE_STATUS $ila]"
  puts "  WINDOW_COUNT:[get_property CONTROL.WINDOW_COUNT $ila]"
  puts "  === PROBES ==="
  foreach p [get_hw_probes -of_objects $ila] {
    puts "  PROBE:$p WIDTH:[get_property PROBE_PORT_BIT_COUNT $p] TYPE:[get_property TYPE $p]"
  }
}
```

**No ILA found** → STOP.
**ILA를 찾을 수 없습니다** → 중지합니다.
Guide: "No ILA cores detected.
가이드: "ILA 코어가 감지되지 않습니다.
Ensure the device was programmed with a bitstream containing ILA cores and the correct .ltx probes file is associated."
ILA 코어가 포함된 비트스트림으로 장치가 프로그래밍되었고 올바른 .ltx 프로브 파일이 연결되어 있는지 확인하세요."

On the chipscope-mcp fallback path:
Chipscope-mcp 대체 경로에서:
- A zero-ILA reading before `chipscope_scan(action="scan", ltx_path=...)` has run is not authoritative — don't treat it as "no ILA found" yet.
- `chipscope_scan(action="scan", ltx_path=...)`가 실행되기 전의 제로 ILA 읽기는 신뢰할 수 없습니다. 아직 "ILA를 찾을 수 없음"으로 처리하지 마세요.
- After scanning, if the server reports a `debug_ready` field, treat `debug_ready: true` as confirmation.
- 스캔 후 서버가 `debug_ready` 필드를 보고하면 `debug_ready: true`를 확인으로 처리합니다.
- If `debug_ready` is absent, fall back to the scan's own reported core count instead.
- `debug_ready`가 없으면 대신 스캔에서 보고된 자체 코어 수로 대체합니다.

### Step 2: Configure Trigger
### 2단계: 트리거 구성

**Basic trigger** — map user's NL request to probe compare values:
**기본 트리거** — 비교 값을 조사하기 위해 사용자의 NL 요청을 매핑합니다.

```tcl
set ila [lindex [get_hw_ilas] 0]
# TRIGGER_MODE is read-only on Versal (axis_ila) — use catch to skip gracefully
catch {set_property CONTROL.TRIGGER_MODE BASIC_ONLY $ila}
set_property CONTROL.TRIGGER_CONDITION AND $ila
set_property CONTROL.TRIGGER_POSITION 512 $ila
set_property CONTROL.DATA_DEPTH 1024 $ila
set_property TRIGGER_COMPARE_VALUE eq1'b1 [get_hw_probes <probe_name> -of_objects $ila]
```

**Compare value syntax:** `eq<width>'<radix><value>` — see [REFERENCE.md](REFERENCE.md) for full encoding (=, !=, <, >, R, F, B, N, X).
**값 구문 비교:** `eq<width>'<radix><value>` — 전체 인코딩(=, !=, <, >, R, F, B, N, X)은 [REFERENCE.md](REFERENCE.md)를 참조하세요.

**Advanced trigger (TSM)** — for multi-phase triggers (e.g., AXI transactions).
**고급 트리거(TSM)** — 다단계 트리거용(예: AXI 트랜잭션)
See [REFERENCE.md](REFERENCE.md#trigger-state-machine-tsm-syntax) for the full grammar, rejection rules, and worked examples:
전체 문법, 거부 규칙 및 작업 예제는 [REFERENCE.md](REFERENCE.md#trigger-state-machine-tsm-syntax)를 참조하세요.
1. Generate a `.tsm` file with state machine code
1. 상태 기계 코드를 사용하여 `.tsm` 파일 생성
2. Write it using agent file tools
2. 에이전트 파일 도구를 사용하여 작성
3. Set: `set_property CONTROL.TRIGGER_MODE ADVANCED_ONLY $ila`
3. 설정: `set_property CONTROL.TRIGGER_MODE ADVANCED_ONLY $ila`
4. Set: `set_property CONTROL.TSM_FILE <path> $ila`
4. 설정: `set_property CONTROL.TSM_FILE <경로> $ila`
5. Compile-check: `run_hw_ila -compile_only $ila`
5. 컴파일 확인: `run_hw_ila -compile_only $ila`

**Capture condition** (storage qualifier):
**캡처 조건**(저장 한정자):
```tcl
set_property CONTROL.CAPTURE_MODE BASIC $ila
set_property CONTROL.CAPTURE_CONDITION AND $ila
set_property CAPTURE_COMPARE_VALUE eq1'b1 [get_hw_probes <probe> -of_objects $ila]
```

### Step 3: Arm, Wait & Upload
### 3단계: 준비, 대기 및 업로드

```tcl
run_hw_ila $ila
wait_on_hw_ila $ila
upload_hw_ila_data $ila
```

For immediate capture (no trigger, "aliveness" check):
즉시 캡처의 경우(트리거 없음, "활성" 확인):
```tcl
run_hw_ila -trigger_now $ila
wait_on_hw_ila $ila
upload_hw_ila_data $ila
```

**⚠️ `wait_on_hw_ila` blocks until trigger fires.** If the trigger condition is never met, the agent will hang.
**⚠️ `wait_on_hw_ila`는 트리거가 실행될 때까지 차단됩니다.** 트리거 조건이 충족되지 않으면 에이전트가 정지됩니다.
For conditions that may not fire, use `run_hw_ila -trigger_now` first to verify probe connectivity.
실행되지 않을 수 있는 조건의 경우 먼저 'run_hw_ila -trigger_now'를 사용하여 프로브 연결을 확인하세요.

**⚠️ `wait_on_hw_ila -timeout N` is in MINUTES** (not seconds).
**⚠️`wait_on_hw_ila -timeout N`은 MINUTES**(초 아님) 단위입니다.
Use `-timeout 1` for a 1-minute cap.
1분으로 제한하려면 '-timeout 1'을 사용하세요.
Omitting `-timeout` waits indefinitely.
`-timeout`을 생략하면 무기한 대기합니다.

**⚠️ Versal DPC arm latency.** On Versal (axis_ila via DPC JTAG), there is a brief latency between `run_hw_ila` returning and the ILA hardware being fully armed.
**⚠️ Versal DPC arm 대기 시간.** Versal(DPC JTAG를 통한 axis_ila)에서는 'run_hw_ila' 반환과 ILA 하드웨어가 완전히 무장되는 사이에 짧은 대기 시간이 있습니다.
If the triggering event is driven by VIO in the same Vivado session, issue the `commit_hw_vio` that fires the trigger in a **separate `vivado_execute` call** — not in the same command as `run_hw_ila`.
트리거링 이벤트가 동일한 Vivado 세션에서 VIO에 의해 구동되는 경우 `run_hw_ila`와 동일한 명령이 아닌 **별도의 `vivado_execute` 호출**에서 트리거를 실행하는 `commit_hw_vio`를 실행하세요.
The natural inter-call latency (~100 ms) is sufficient for the ILA to settle into WAITING_FOR_TRIGGER state.
ILA가 WAITING_FOR_TRIGGER 상태에 도달하는 데에는 자연스러운 호출 간 대기 시간(~100ms)이면 충분합니다.

### Step 4: Export & Analyze Data
### 4단계: 데이터 내보내기 및 분석

```tcl
write_hw_ila_data -csv_file vivado_agentic_ai_reports/hw-ila-debug/capture_data [upload_hw_ila_data $ila]
```

Export formats: `-csv_file` (spreadsheet), `-vcd_file` (waveform viewer), or native `.ila` (Vivado reload).
내보내기 형식: `-csv_file`(스프레드시트), `-vcd_file`(파형 뷰어) 또는 기본 `.ila`(Vivado reload).

To read specific probe sample values programmatically:
특정 프로브 샘플 값을 프로그래밍 방식으로 읽으려면:
```tcl
list_hw_samples [get_hw_probes <probe_name> -of_objects $ila]
```

### Step 5: Write Output Files — MANDATORY after every successful capture
### 5단계: 출력 파일 쓰기 - 캡처가 성공할 때마다 필수

**This step is not optional and does not depend on the user asking for a report.** Any prompt that results in a completed ILA capture — including "display captured data", "show the waveform", "upload and display", or "arm and capture" — MUST produce all three output files before any chat summary is given.
**이 단계는 선택 사항이 아니며 보고서를 요청하는 사용자에 따라 달라지지 않습니다.** "캡처된 데이터 표시", "파형 표시", "업로드 및 표시" 또는 "장착 및 캡처"를 포함하여 ILA 캡처가 완료되는 모든 프롬프트는 채팅 요약이 제공되기 전에 세 가지 출력 파일을 모두 생성해야 합니다.

Write every file below under `vivado_agentic_ai_reports/hw-ila-debug/`.
`vivado_agentic_ai_reports/hw-ila-debug/` 아래에 아래의 모든 파일을 작성하세요.
The
는
file tool creates missing parent directories, so do not shell out to `mkdir`
파일 도구는 누락된 상위 디렉토리를 생성하므로 `mkdir`로 쉘아웃하지 마십시오.
(this skill forbids terminal commands).
(이 스킬은 터미널 명령을 금지합니다).

| File<br>파일 | Format<br>형식 | Content<br>내용 |
|------|--------|---------|
| `capture_data.csv`<br>`capture_data.csv` | CSV<br>CSV | Raw waveform data from ILA (via `write_hw_ila_data`)<br>ILA의 원시 파형 데이터(`write_hw_ila_data`를 통해) |
| `report_data.json`<br>`보고서_데이터.json` | JSON<br>JSON | Structured capture metadata + analysis (schema in [REFERENCE.md](REFERENCE.md))<br>구조적 캡처 메타데이터 + 분석([REFERENCE.md](REFERENCE.md)의 스키마) |
| `REPORT.md`<br>`보고서.md` | Markdown<br>마크다운 | Executive summary: trigger config, capture stats, signal observations<br>요약: 트리거 구성, 캡처 통계, 신호 관찰 |

**Prompt mapping** — these user phrases all require files to be written before summarizing in chat:
**프롬프트 매핑** — 이러한 사용자 문구는 모두 채팅에서 요약하기 전에 파일을 작성해야 합니다.

| User says<br>사용자가 말합니다 | Agent does<br>에이전트는 |
|---|---|
| "display the captured ILA data"<br>"캡처된 ILA 데이터를 표시합니다" | upload → export CSV → write JSON + REPORT.md → chat summary<br>업로드 → CSV 내보내기 → JSON + REPORT.md 작성 → 채팅 요약 |
| "show me the waveform"<br>"파형 보여줘" | upload → export CSV → write JSON + REPORT.md → chat summary<br>업로드 → CSV 내보내기 → JSON + REPORT.md 작성 → 채팅 요약 |
| "arm and capture"<br>"팔과 포획" | arm → wait → upload → export → write JSON + REPORT.md → chat summary<br>arm → 대기 → 업로드 → 내보내기 → JSON + REPORT.md 작성 → 채팅 요약 |
| "upload and display"<br>"업로드 및 표시" | upload → export → write JSON + REPORT.md → chat summary<br>업로드 → 내보내기 → JSON + REPORT.md 작성 → 채팅 요약 |

**After writing files, confirm in chat:** "Files written to `vivado_agentic_ai_reports/hw-ila-debug/`: `capture_data.csv`, `report_data.json`, `REPORT.md`."
**파일을 작성한 후 채팅에서 확인하세요.** "`vivado_agentic_ai_reports/hw-ila-debug/`에 작성된 파일: `capture_data.csv`, `report_data.json`, `REPORT.md`."

**Where to find the report:** `vivado_agentic_ai_reports/hw-ila-debug/REPORT.md` in the working directory where the agent session was started.
**보고서를 찾을 수 있는 위치:** 에이전트 세션이 시작된 작업 디렉터리의 `vivado_agentic_ai_reports/hw-ila-debug/REPORT.md`
If the report was not generated, the capture data is still uploaded (from Step 3) — write the output files from that existing upload without re-arming.
보고서가 생성되지 않은 경우 캡처 데이터는 계속 업로드됩니다(3단계). 다시 준비하지 않고 기존 업로드의 출력 파일을 작성합니다.
Only retry from Step 3 if the capture itself failed (e.g. trigger never fired, upload returned no samples).
캡처 자체가 실패한 경우에만 3단계부터 다시 시도하세요(예: 트리거가 실행되지 않았거나 업로드가 샘플을 반환하지 않음).


---

## Design-Specific Rules
## 디자인별 규칙

**All outputs MUST use ACTUAL values.**
**모든 출력은 실제 값을 사용해야 합니다.**

| Rule<br>규칙 | Wrong<br>틀렸어 | Correct<br>맞다 |
|------|-------|---------|
| Probe value<br>프로브 값 | "signal is high"<br>"신호가 높다" | "axi_arvalid (hw_ila_1/probe3): 1'b1 at sample 512"<br>"axi_arvalid(hw_ila_1/probe3): 샘플 512의 1'b1" |
| Trigger config<br>트리거 구성 | "set up trigger"<br>"트리거 설정" | "TRIGGER_MODE: BASIC_ONLY, CONDITION: AND, probe3 == eq1'b1, POSITION: 512/1024"<br>"TRIGGER_MODE: BASIC_ONLY, 조건: AND, 프로브3 == eq1'b1, 위치: 512/1024" |
| Core info<br>핵심 정보 | "found ILA"<br>"ILA를 찾았어요" | "hw_ila_1: 6 probes, depth 4096, adv trigger supported"<br>"hw_ila_1: 6개 프로브, 깊이 4096, 고급 트리거 지원" |

---

## Error Handling
## 오류 처리

| Error<br>오류 | Symptom<br>증상 | Action<br>액션 |
|-------|---------|--------|
| No ILA cores<br>ILA 코어 없음 | `get_hw_ilas` empty<br>`get_hw_ilas` 비어 있음 | Check: bitstream has debug cores? .ltx file associated?<br>확인: 비트스트림에 디버그 코어가 있습니까? .ltx 파일이 연결되어 있나요? |
| No hw_server<br>아니요 hw_server | `get_hw_servers` empty<br>`get_hw_servers` 비어 있음 | Guide: `connect_hw_server -url localhost:3121`<br>가이드: `connect_hw_server -url localhost:3121` |
| Trigger never fires<br>트리거가 실행되지 않음 | `wait_on_hw_ila` hangs<br>`wait_on_hw_ila`가 중단됨 | Use `run_hw_ila -trigger_now` to verify connectivity first<br>`run_hw_ila -trigger_now`를 사용하여 먼저 연결을 확인하세요. |
| Probe not found<br>프로브를 찾을 수 없습니다 | `get_hw_probes` returns empty<br>`get_hw_probes`가 비어 있음을 반환합니다. | Check: .ltx file matches bitstream? `vivado_doc_search` for probe naming<br>확인: .ltx 파일이 비트스트림과 일치합니까? 프로브 이름 지정을 위한 `vivado_doc_search` |
| TSM compile error<br>TSM 컴파일 오류 | `run_hw_ila -compile_only` fails<br>`run_hw_ila -compile_only` 실패 | Check the exact rule in [REFERENCE.md](REFERENCE.md#trigger-state-machine-tsm-syntax) first (it lists the specific rejection error strings); `vivado_doc_search` for anything not covered there<br>먼저 [REFERENCE.md](REFERENCE.md#trigger-state-machine-tsm-syntax)에서 정확한 규칙을 확인하세요(특정 거부 오류 문자열이 나열되어 있음). 거기에 포함되지 않은 내용에 대한 `vivado_doc_search` |
| TSM `arm()` rejected (chipscope-mcp path)<br>TSM `arm()` 거부됨(chipscope-mcp 경로) | `chipscope_ila_core(action="arm", trigger_state_machine=...)` returns `success: false` with a parser error<br>`chipscope_ila_core(action="arm", Trigger_state_machine=...)`는 파서 오류와 함께 `success: false`를 반환합니다. | Match the exact error string against [REFERENCE.md](REFERENCE.md#trigger-state-machine-tsm-syntax)'s "Rejected patterns" table first<br>먼저 [REFERENCE.md](REFERENCE.md#trigger-state-machine-tsm-syntax)의 "거부된 패턴" 테이블과 정확한 오류 문자열을 일치시키세요. |

---

## Examples
## 예

**"Capture whatever is on the ILA right now"** → Discover ILA → `run_hw_ila -trigger_now` → `wait_on_hw_ila` → `upload_hw_ila_data` → export CSV → summarize.
**"지금 ILA에 있는 모든 것을 캡처하세요"** → ILA 검색 → `run_hw_ila -trigger_now` → `wait_on_hw_ila` → `upload_hw_ila_data` → CSV 내보내기 → 요약.

**"Trigger when AXI RVALID goes high"** → Find probe for `axi_rvalid` → `set_property TRIGGER_COMPARE_VALUE eq1'b1` → arm → wait → upload → export.
**"AXI RVALID가 높아지면 트리거"** → `axi_rvalid`에 대한 프로브 찾기 → `set_property TRIGGER_COMPARE_VALUE eq1'b1` → arm → 대기 → 업로드 → 내보내기.

**"Trigger on AXI read transaction (address then data then last)"** → Generate 3-state TSM → write `.tsm` file → set `ADVANCED_ONLY` → arm → capture.
**"AXI 읽기 트랜잭션에서 트리거(주소, 데이터, 마지막)"** → 3-상태 TSM 생성 → '.tsm' 파일 쓰기 → 'ADVANCED_ONLY' 설정 → arm → 캡처.

---

## Chipscope-mcp Fallback
## Chipscope-mcp 대체

Use this path only when Vivado MCP is not installed.
Vivado MCP가 설치되지 않은 경우에만 이 경로를 사용하십시오.
Session lifecycle per VIVADO-PATTERNS.md Pattern 19 (vivado-skill-creator):
VIVADO-PATTERNS.md 패턴 19(vivado-skill-creator)에 따른 세션 수명 주기:

```
chipscope_session(action="connect", hw_server_url="TCP:<host>:3121", cs_server_url="TCP:<host>:3042")
chipscope_device(action="select", device_selector="0")
chipscope_scan(action="scan", ltx_path="<path-to-design>.ltx")
```

**ILA — discover, arm, capture:**
**ILA — 발견, 준비, 캡처:**
```
chipscope_ila_core(action="list")
chipscope_ila_core(action="probes", ila_name="<ila-name>")
chipscope_ila_core(action="set_trigger", ila_name="<ila-name>", probe_trigger_values={"<probe>": ["==", "0x...1"]})
chipscope_ila_core(action="arm", ila_name="<ila-name>", trigger_immediately=False, window_count=<n>, window_size=<n>, trigger_position=<n>)
# For a multi-phase (TSM) trigger instead of a single probe compare, pass
# trigger_state_machine=<program text> to arm() -- see REFERENCE.md's TSM
# section for grammar, rejection rules, and worked examples.
chipscope_ila_capture(action="wait", ila_name="<ila-name>", timeout_minutes=<n>)
chipscope_ila_capture(action="upload", ila_name="<ila-name>")
chipscope_ila_capture(action="export", ila_name="<ila-name>", file_path="vivado_agentic_ai_reports/hw-ila-debug/capture_data.csv", export_format="CSV")
```

Confirmed tool schema notes:
확인된 도구 스키마 참고 사항:
- `chipscope_ila_core` actions: `list/status/probes/reset_probes/set_trigger/set_capture/arm/trigger_immediate`
- `chipscope_ila_core` 작업: `list/status/probes/reset_probes/set_trigger/set_capture/arm/trigger_immediate`
- `chipscope_ila_capture` actions: `wait/upload/get_data/export`
- `chipscope_ila_capture` 작업: `기다림/업로드/get_data/내보내기`
- Use `ila_name=` (not `core=`)
- `ila_name=`(`core=` 아님)을 사용하세요.
- `export`'s `file_path` is required — always pass it explicitly under `vivado_agentic_ai_reports/hw-ila-debug/` so it lands where Step 5 requires
- `export`의 `file_path`가 필요합니다. 항상 `vivado_agentic_ai_reports/hw-ila-debug/` 아래에 명시적으로 전달하여 5단계에서 필요한 위치에 도달하도록 합니다.

**NEVER auto-program PDI.** Stop and ask before `chipscope_device(action="program")`.
**PDI를 자동 프로그래밍하지 마세요.** `chipscope_device(action="program")` 전에 멈추고 물어보세요.

---

## Integration
## 통합

**Complementary:** `hw-vio-debug` drives stimulus → ILA captures response (combined debug session)
**보완적:** `hw-vio-debug`가 자극을 유도 → ILA가 응답을 캡처함(결합된 디버그 세션)
**Downstream:** Export CSV/VCD for offline analysis in external waveform viewers
**다운스트림:** 외부 파형 뷰어에서 오프라인 분석을 위해 CSV/VCD 내보내기

---

## Metadata
## 메타데이터

**Keywords:** ILA, trigger, capture, waveform, probe, debug, CSV, VCD, TSM, state machine, compare value
**키워드:** ILA, 트리거, 캡처, 파형, 프로브, 디버그, CSV, VCD, TSM, 상태 머신, 값 비교
