<!-- ko-translation: enabled -->
---
name: hw-vio-debug
description: >-
  Interact with VIO debug cores via vivado-mcp or chipscope-mcp.
  Read input probes, drive output probes, monitor activity, and reset outputs.
  vivado-mcp supports all device families. chipscope-mcp supports Versal only.
  Use when user asks to "read VIO inputs", "drive VIO output", "assert <signal> via VIO",
  "check signal activity", "what is the value of <signal>", or "monitor <probe>".
license: MIT
compatibility: Requires Vivado 2026.1+, hardware with a VIO core, and access to either vivado-mcp or chipscope-mcp.
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
# VIO Debug (EA)
# VIO 디버그(EA)

> **Early Access** — this skill may change before general availability.
> **조기 액세스** — 이 스킬은 일반 출시되기 전에 변경될 수 있습니다.

Interacts with VIO debug cores on live hardware: discover → read inputs → drive outputs → monitor activity.
라이브 하드웨어에서 VIO 디버그 코어와 상호 작용합니다. 검색 → 입력 읽기 → 드라이브 출력 → 활동 모니터링.

See [REFERENCE.md](REFERENCE.md) for VIO property tables and Tcl command reference.
VIO 속성 테이블과 Tcl 명령 참조는 [REFERENCE.md](REFERENCE.md)를 참조하세요.

**Routing:** Try Vivado MCP (`vivado_execute` + Hardware Manager Tcl, below) first if a Vivado MCP session is available — it works across all device families.
**라우팅:** Vivado MCP 세션을 사용할 수 있는 경우 먼저 Vivado MCP(아래 `vivado_execute` + 하드웨어 관리자 Tcl)를 사용해 보세요. 이는 모든 장치 제품군에서 작동합니다.
Fall back to chipscope-mcp (`chipscope_vio`) if Vivado MCP is not installed.
Vivado MCP가 설치되지 않은 경우 Chipscope-mcp(`chipscope_vio`)로 대체됩니다.
See [Chipscope-mcp Fallback](#chipscope-mcp-fallback) below.
아래의 [Chipscope-mcp 폴백](#chipscope-mcp-fallback)을 참조하세요.

## Tools Used
## 사용된 도구

**Primary path — Vivado MCP** (plus the agent's own file tools):
**기본 경로 — Vivado MCP** (에이전트 자체 파일 도구 포함):

| Tool<br>도구 | Purpose<br>목적 |
|------|---------|
| `vivado_execute`<br>'vivado_execute' | Run all Tcl commands (`get_hw_vios`, `commit_hw_vio`, etc.). Every Tcl block MUST be run via `vivado_execute(session_id=..., command=<tcl>)`.<br>모든 Tcl 명령(`get_hw_vios`, `commit_hw_vio` 등)을 실행합니다. 모든 Tcl 블록은 `vivado_execute(session_id=..., command=<tcl>)`를 통해 실행되어야 합니다. |
| `vivado_doc_search`<br>`vivado_doc_search` | Look up unfamiliar VIO properties. No `session_id` needed.<br>익숙하지 않은 VIO 속성을 찾아보세요. 'session_id'가 필요하지 않습니다. |
| `vivado_status`<br>`vivado_status` | Check Vivado session health.<br>Vivado 세션 상태를 확인하세요. |
| `vivado_list_sessions`<br>`vivado_list_sessions` | Find an active session to use (get its `session_id`).<br>사용할 활성 세션을 찾습니다(`session_id` 가져오기). |
| Agent file tools<br>에이전트 파일 도구 | Write output files (vio_snapshot.json, REPORT.md).<br>출력 파일(vio_snapshot.json, REPORT.md)을 작성합니다. |

**Fallback path — chipscope-mcp** (when Vivado MCP is unavailable): `chipscope_session`, `chipscope_device`, `chipscope_scan`, `chipscope_vio`.
**대체 경로 — Chipscope-mcp** (Vivado MCP를 사용할 수 없는 경우): `chipscope_session`, `chipscope_device`, `chipscope_scan`, `chipscope_vio`.
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
- **Batch Tcl** — discover all VIO cores in one `vivado_execute`, not separate calls.
- **Batch Tcl** — 별도의 호출이 아닌 하나의 `vivado_execute`로 모든 VIO 코어를 검색합니다.
- **Write reports to file** — do not dump full probe data in chat.
- **파일에 보고서 작성** - 채팅에서 전체 조사 데이터를 덤프하지 마세요.
- Give a short summary only.
- 간략한 요약만 제공하세요.
- **Use `vivado_doc_search`** for unfamiliar probe properties.
- **낯선 프로브 속성에 대해서는 `vivado_doc_search`**를 사용하세요.
- **Do NOT** use `shell` commands or Vivado Tcl file I/O.
- **쉘` 명령이나 Vivado Tcl 파일 I/O를 사용하지 마십시오**.
- Use agent file tools.
- 에이전트 파일 도구를 사용하십시오.
- **Do NOT** retry failed Tcl with different syntax.
- **하지 마세요** 다른 구문으로 실패한 Tcl을 다시 시도하세요.
- Search docs first, then report the error.
- 먼저 문서를 검색한 후 오류를 신고하세요.

---

## Monitor & Control Workflow
## 모니터링 및 제어 워크플로

### Step 1: Discover VIO Cores & Probes
### 1단계: VIO 코어 및 프로브 검색

```tcl
puts "=== VIO CORES ==="
foreach vio [get_hw_vios] {
  puts "VIO:$vio INSTANCE:[get_property INSTANCE_NAME $vio] REFRESH:[get_property CORE_REFRESH_RATE_MS $vio]"
  puts "  === INPUT PROBES ==="
  foreach p [get_hw_probes -of_objects $vio -filter {TYPE == vio_input}] {
    refresh_hw_vio $vio
    puts "  IN:$p VALUE:[get_property INPUT_VALUE $p] ACTIVITY:[get_property ACTIVITY_VALUE $p]"
  }
  puts "  === OUTPUT PROBES ==="
  foreach p [get_hw_probes -of_objects $vio -filter {TYPE == vio_output}] {
    puts "  OUT:$p VALUE:[get_property OUTPUT_VALUE $p]"
  }
}
```

**No VIO found** → STOP.
**VIO를 찾을 수 없습니다** → 중지합니다.
Guide: "No VIO cores detected.
가이드: "VIO 코어가 감지되지 않습니다.
Ensure the device was programmed with a bitstream containing VIO cores and the correct .ltx probes file is associated."
장치가 VIO 코어를 포함하는 비트스트림으로 프로그래밍되었고 올바른 .ltx 프로브 파일이 연결되어 있는지 확인하세요."

**⚠️ After programming or reprogramming,** VIO output probe software values may retain stale values from the previous session.
**⚠️ 프로그래밍 또는 재프로그래밍 후** VIO 출력 프로브 소프트웨어 값은 이전 세션의 오래된 값을 유지할 수 있습니다.
Always sync before driving outputs:
출력을 구동하기 전에 항상 동기화하십시오.
```tcl
refresh_hw_vio -update_output_values [get_hw_vios hw_vio_1]
```
or reset all outputs to design-time initial values:
또는 모든 출력을 디자인 타임 초기 값으로 재설정합니다.
```tcl
reset_hw_vio_outputs [get_hw_vios hw_vio_1]
```
This ensures the software state matches hardware, so that subsequent `set_property OUTPUT_VALUE` + `commit_hw_vio` creates proper signal transitions (e.g., 0→1 edges for pulse-triggered designs).
이렇게 하면 소프트웨어 상태가 하드웨어와 일치하므로 후속 `set_property OUTPUT_VALUE` + `commit_hw_vio`가 적절한 신호 전환을 생성합니다(예: 펄스 트리거 설계의 경우 0→1 에지).

### Step 2: Read Input Values
### 2단계: 입력 값 읽기

```tcl
refresh_hw_vio [get_hw_vios hw_vio_1]
foreach p [get_hw_probes -of_objects [get_hw_vios hw_vio_1] -filter {TYPE == vio_input}] {
  puts "$p INPUT:[get_property INPUT_VALUE $p] ACTIVITY:[get_property ACTIVITY_VALUE $p]"
}
```

### Step 3: Drive Output Values
### 3단계: 출력 값 구동

Map user's NL request to probe + value:
사용자의 NL 요청을 프로브 + 값으로 매핑합니다.

```tcl
set_property OUTPUT_VALUE 1 [get_hw_probes <probe_name> -of_objects [get_hw_vios hw_vio_1]]
commit_hw_vio [get_hw_vios hw_vio_1]
```

**Reset all outputs** to initial values:
**모든 출력**을 초기값으로 재설정:
```tcl
reset_hw_vio_outputs [get_hw_vios hw_vio_1]
```

**Clear activity detectors:**
**활동 감지기 지우기:**
```tcl
reset_hw_vio_activity [get_hw_vios hw_vio_1]
```

### Step 4: Write Output Files
### 4단계: 출력 파일 쓰기

**⚠️ Write ALL files before giving any summary.** Use agent file tools.
**⚠️ 요약을 제공하기 전에 모든 파일을 작성하세요.** 에이전트 파일 도구를 사용하세요.

Write every file below under `vivado_agentic_ai_reports/hw-vio-debug/`.
`vivado_agentic_ai_reports/hw-vio-debug/` 아래에 아래의 모든 파일을 작성하세요.
The
는
file tool creates missing parent directories, so do not shell out to `mkdir`
파일 도구는 누락된 상위 디렉토리를 생성하므로 `mkdir`로 쉘아웃하지 마십시오.
(this skill forbids terminal commands).
(이 스킬은 터미널 명령을 금지합니다).

| File<br>파일 | Format<br>형식 | Content<br>내용 |
|------|--------|---------|
| `vio_snapshot.json`<br>`vio_snapshot.json` | JSON<br>JSON | All probe names, directions, current values, activity<br>모든 프로브 이름, 방향, 현재 값, 활동 |
| `REPORT.md`<br>`보고서.md` | Markdown<br>마크다운 | VIO core status, probe values table, actions taken<br>VIO 코어 상태, 프로브 값 테이블, 취한 조치 |

---

## Design-Specific Rules
## 디자인별 규칙

**All outputs MUST use ACTUAL values.**
**모든 출력은 실제 값을 사용해야 합니다.**

| Rule<br>규칙 | Wrong<br>틀렸어 | Correct<br>맞다 |
|------|-------|---------|
| VIO output<br>VIO 출력 | "asserted reset"<br>"재설정 주장" | "Set OUTPUT_VALUE=1 on rst_n (hw_vio_1), committed"<br>"rst_n(hw_vio_1)에 OUTPUT_VALUE=1을 설정하고 커밋됨" |
| Core info<br>핵심 정보 | "found VIO"<br>"VIO를 찾았습니다" | "hw_vio_1: 4 input probes, 2 output probes, refresh 500ms"<br>"hw_vio_1: 입력 프로브 4개, 출력 프로브 2개, 새로 고침 500ms" |

---

## Error Handling
## 오류 처리

| Error<br>오류 | Symptom<br>증상 | Action<br>액션 |
|-------|---------|--------|
| No VIO cores<br>VIO 코어 없음 | `get_hw_vios` empty<br>`get_hw_vios` 비어 있음 | Check: bitstream has debug cores? .ltx file associated?<br>확인: 비트스트림에 디버그 코어가 있습니까? .ltx 파일이 연결되어 있나요? |
| No hw_server<br>아니요 hw_server | `get_hw_servers` empty<br>`get_hw_servers` 비어 있음 | Guide: `connect_hw_server -url localhost:3121`<br>가이드: `connect_hw_server -url localhost:3121` |
| VIO out-of-sync<br>VIO가 동기화되지 않음 | Status shows "Outputs out-of-sync"<br>상태가 "출력이 동기화되지 않음"으로 표시됩니다. | Run: `refresh_hw_vio -update_output_values [get_hw_vios]`<br>실행: `refresh_hw_vio -update_output_values [get_hw_vios]` |

---

## Examples
## 예

**"What is the VIO reset signal?"** → Discover VIO → `refresh_hw_vio` → read `INPUT_VALUE` of reset probe → report value + activity.
**"VIO 재설정 신호는 무엇입니까?"** → VIO 검색 → `refresh_hw_vio` → 재설정 프로브의 `INPUT_VALUE` 읽기 → 값 + 활동 보고.

**"Assert reset for 1 cycle"** → `set_property OUTPUT_VALUE 1` on reset probe → `commit_hw_vio` → `set_property OUTPUT_VALUE 0` → `commit_hw_vio`.
**"1주기 동안 재설정을 주장합니다"** → 재설정 프로브의 `set_property OUTPUT_VALUE 1` → `commit_hw_vio` → `set_property OUTPUT_VALUE 0` → `commit_hw_vio`.

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

**VIO — read inputs, drive outputs:**
**VIO — 입력 읽기, 출력 구동:**
```
chipscope_vio(action="read", vio_name="<vio-name>")
chipscope_vio(action="write", vio_name="<vio-name>", probe_values={"<probe>": 1})
```

Exact action/parameter names must be confirmed against the live chipscope-mcp tool schema before use — do not assume the names above are final without checking `chipscope-mcp/src/chipscope_mcp/tools/chipscope_vio.py`.
사용하기 전에 라이브 Chipscope-mcp 도구 스키마에 대해 정확한 작업/매개변수 이름을 확인해야 합니다. `chipscope-mcp/src/chipscope_mcp/tools/chipscope_vio.py`를 확인하지 않고 위의 이름이 최종 이름이라고 가정하지 마십시오.

**NEVER auto-program PDI.** Stop and ask before `chipscope_device(action="program")`.
**PDI를 자동 프로그래밍하지 마세요.** `chipscope_device(action="program")` 전에 멈추고 물어보세요.

---

## Integration
## 통합

**Complementary:** Drives stimulus for `hw-ila-debug` capture (combined debug session)
**보완적:** `hw-ila-debug` 캡처에 대한 자극을 유도합니다(결합된 디버그 세션).

---

## Metadata
## 메타데이터

**Keywords:** VIO, probe, input, output, activity, monitor, drive, reset
**키워드:** VIO, 프로브, 입력, 출력, 활동, 모니터, 드라이브, 재설정
