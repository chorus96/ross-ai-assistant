<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# MCP Tool Reference
# MCP 도구 참조

Complete reference for all tools provided by the Vivado MCP Server.
Vivado MCP 서버가 제공하는 모든 도구에 대한 전체 참조입니다.
These are the capabilities available to any MCP-compatible AI agent (VS Code Copilot, Cursor, Claude Code, etc.) when connected to the server.
이는 서버에 연결될 때 모든 MCP 호환 AI 에이전트(VS Code Copilot, Cursor, Claude Code 등)에서 사용할 수 있는 기능입니다.

---

## Quick Reference
## 빠른 참조

| Tool<br>도구 | Category<br>카테고리 | Description<br>설명 |
|------|----------|-------------|
| [`vivado_start`](#vivado_start)<br>[`vivado_start`](#vivado_start) | Session<br>세션 | Launch a new Vivado session<br>새로운 Vivado 세션 시작 |
| [`vivado_stop`](#vivado_stop)<br>[`vivado_stop`](#vivado_stop) | Session<br>세션 | Stop a Vivado session<br>Vivado 세션 중지 |
| [`vivado_connect`](#vivado_connect)<br>[`vivado_connect`](#vivado_connect) | Session<br>세션 | Reconnect to an existing Vivado instance<br>기존 Vivado 인스턴스에 다시 연결 |
| [`vivado_list_sessions`](#vivado_list_sessions)<br>[`vivado_list_sessions`](#vivado_list_sessions) | Session<br>세션 | List all active sessions<br>모든 활성 세션 나열 |
| [`vivado_cleanup`](#vivado_cleanup)<br>[`vivado_cleanup`](#vivado_cleanup) | Session<br>세션 | Clean up stale sessions and orphaned processes<br>오래된 세션 및 고아 프로세스 정리 |
| [`vivado_execute`](#vivado_execute)<br>[`vivado_execute`](#vivado_execute) | Execution<br>실행 | Execute Tcl commands in Vivado<br>Vivado에서 Tcl 명령 실행 |
| [`vivado_status`](#vivado_status)<br>[`vivado_status`](#vivado_status) | Monitoring<br>모니터링 | Check session health and run progress<br>세션 상태 확인 및 진행 상황 확인 |
| [`vivado_log_messages`](#vivado_log_messages)<br>[`vivado_log_messages`](#vivado_log_messages) | Monitoring<br>모니터링 | Parse log for errors, warnings, and messages<br>오류, 경고 및 메시지에 대한 로그 구문 분석 |
| [`vivado_history`](#vivado_history)<br>[`vivado_history`](#vivado_history) | Monitoring<br>모니터링 | Get or search command history<br>명령 기록 가져오기 또는 검색 |
| [`vivado_display`](#vivado_display)<br>[`vivado_display`](#vivado_display) | GUI<br>GUI | Manage displays, screenshots, and virtual displays (Linux only)<br>디스플레이, 스크린샷, 가상 디스플레이 관리(Linux에만 해당) |
| [`vivado_lsf`](#vivado_lsf)<br>[`vivado_lsf`](#vivado_lsf) | Remote<br>원격 | Run Vivado on LSF cluster nodes (Linux only)<br>LSF 클러스터 노드에서 Vivado 실행(Linux에만 해당) |
| [`vivado_ssh`](#vivado_ssh)<br>[`vivado_ssh`](#vivado_ssh) | Remote<br>원격 | Run Vivado on remote machines via SSH (Linux only)<br>SSH를 통해 원격 머신에서 Vivado 실행(Linux에만 해당) |
| [`vivado_todos`](#vivado_todos)<br>[`vivado_todos`](#vivado_todos) | Workflow<br>작업 흐름 | Track multi-step task progress in IDE sidebar<br>IDE 사이드바에서 다단계 작업 진행 상황 추적 |
| [`vivado_client_info`](#vivado_client_info)<br>[`vivado_client_info`](#vivado_client_info) | Workflow<br>작업 흐름 | Get MCP client capabilities<br>MCP 클라이언트 기능 얻기 |

Documentation search is a **separate MCP server**, `amd-doc-search`, rather than
문서 검색은 **별도의 MCP 서버**, `amd-doc-search`입니다.
a tool of this one — see [`vivado_doc_search`](#vivado_doc_search) below.
이 도구 — 아래 [`vivado_doc_search`](#vivado_doc_search)를 참조하세요.

Which tools are present depends on the platform and the configuration.
어떤 도구가 제공되는지는 플랫폼과 구성에 따라 다릅니다.
`vivado_display`, `vivado_lsf` and `vivado_ssh` are Linux only, and the last two
`vivado_display`, `vivado_lsf` 및 `vivado_ssh`는 Linux 전용이며 마지막 두 개는
are registered only when LSF or SSH is enabled and usable on the machine.
LSF 또는 SSH가 활성화되어 머신에서 사용할 수 있는 경우에만 등록됩니다.
The
는
server reports which tools it skipped, and why, at startup; your IDE's tool list
서버는 시작 시 건너뛴 도구와 그 이유를 보고합니다. IDE의 도구 목록
shows what the running server actually exposes.
실행 중인 서버가 실제로 노출하는 내용을 보여줍니다.

---

## Session Management
## 세션 관리

### `vivado_start`
### 'vivado_start'

Launch a new Vivado session.
새로운 Vivado 세션을 시작하세요.
Always starts in Tcl mode by default — GUI mode is only used when the user explicitly requests graphical interaction.
기본적으로 항상 Tcl 모드에서 시작합니다. GUI 모드는 사용자가 명시적으로 그래픽 상호 작용을 요청할 때만 사용됩니다.
The server configures a Tcl webserver and smart proxy for command execution.
서버는 명령 실행을 위해 Tcl 웹 서버와 스마트 프록시를 구성합니다.

| Parameter<br>매개변수 | Type<br>유형 | Required<br>필수 | Description<br>설명 |
|-----------|------|----------|-------------|
| `working_dir`<br>`working_dir` | string<br>문자열 | **Yes**<br>**예** | Working directory for the Vivado project. Must exist and be writable<br>Vivado 프로젝트의 작업 디렉토리입니다. 존재하고 쓰기 가능해야 합니다. |
| `session_type`<br>`세션_유형` | enum<br>열거형 | **Yes**<br>**예** | `ipi` for block design workflows, `general` for everything else<br>블록 설계 작업 흐름은 'ipi', 그 외 모든 항목은 'general' |
| `gui_mode`<br>`gui_mode` | boolean<br>부울 | No | `true` for GUI mode with bundled VNC display. Default: `false` (Tcl mode)<br>번들 VNC 디스플레이가 있는 GUI 모드의 경우 'true'입니다. 기본값: `false`(Tcl 모드) |
| `display_mode`<br>`디스플레이_모드` | enum<br>열거형 | No | Display handling: `auto` (default), `vnc`, `x11`, `none`<br>디스플레이 처리: `auto`(기본값), `vnc`, `x11`, `none` |
| `display`<br>'디스플레이' | string<br>문자열 | No | X11 display string (e.g. `:99`). Required when `display_mode=x11`<br>X11 표시 문자열(예: `:99`). `display_mode=x11`인 경우 필수 |
| `port`<br>'항구' | number<br>번호 | No | Webserver port. Default: auto-assigned<br>웹서버 포트. 기본값: 자동 할당 |
| `vivado_path`<br>`vivado_path` | string<br>문자열 | No | Path to Vivado executable. Auto-resolved from PATH if omitted<br>Vivado 실행 파일의 경로입니다. 생략된 경우 PATH에서 자동 해결 |
| `env`<br>'환경' | object<br>물체 | No | Extra environment variables for the Vivado process<br>Vivado 프로세스를 위한 추가 환경 변수 |
| `external`<br>'외부' | boolean<br>부울 | No | Allow external connections to the webserver. Default: `false`<br>웹 서버에 대한 외부 연결을 허용합니다. 기본값: `false` |
| `key`<br>'열쇠' | string<br>문자열 | No | Security key for the webserver. Default: `none`<br>웹서버의 보안 키입니다. 기본값: '없음' |
| `link_todos`<br>`link_todos` | boolean<br>부울 | No | Move todos from the `global` session onto this one<br>'글로벌' 세션에서 할일을 이 세션으로 이동하세요. |

**Tips:**
**팁:**
- Sessions start in Tcl mode (headless) by default.
- 세션은 기본적으로 Tcl 모드(헤드리스)에서 시작됩니다.
- Request GUI mode only when you need visual interaction (block designs, schematics).
- 시각적 상호 작용(블록 설계, 회로도)이 필요한 경우에만 GUI 모드를 요청하십시오.
- You can start in Tcl mode and switch to GUI later with `start_gui` — no restart needed.
- Tcl 모드에서 시작하고 나중에 `start_gui`를 사용하여 GUI로 전환할 수 있습니다. 다시 시작할 필요가 없습니다.
- In GUI mode, the server spins up a VNC display automatically with a VNC connection string.
- GUI 모드에서 서버는 VNC 연결 문자열을 사용하여 VNC 디스플레이를 자동으로 회전합니다.
- Use `ipi` session type for Block Design (IP Integrator) workflows, `general` for everything else.
- 블록 설계(IP Integrator) 워크플로우에는 'ipi' 세션 유형을 사용하고, 그 외 모든 것에는 '일반'을 사용합니다.

---

### `vivado_stop`
### 'vivado_stop'

Gracefully stop a Vivado session.
Vivado 세션을 정상적으로 중지합니다.
Saves state, closes the project, and terminates the Vivado process.
상태를 저장하고, 프로젝트를 닫고, Vivado 프로세스를 종료합니다.

| Parameter<br>매개변수 | Type<br>유형 | Required<br>필수 | Description<br>설명 |
|-----------|------|----------|-------------|
| `session_id`<br>`세션_ID` | string<br>문자열 | **Yes**<br>**예** | Session to stop<br>중지할 세션 |
| `close_vivado`<br>`close_vivado` | boolean<br>부울 | No | Close Vivado application. Default: `true`<br>비바도 애플리케이션을 닫습니다. 기본값: `true` |

Set `close_vivado=false` to detach the MCP server while keeping Vivado running.
Vivado를 계속 실행하는 동안 MCP 서버를 분리하려면 `close_vivado=false`를 설정하세요.

---

### `vivado_connect`
### 'vivado_connect'

Reconnect to an existing Vivado instance that already has a running Tcl webserver.
이미 실행 중인 Tcl 웹 서버가 있는 기존 Vivado 인스턴스에 다시 연결합니다.

| Parameter<br>매개변수 | Type<br>유형 | Required<br>필수 | Description<br>설명 |
|-----------|------|----------|-------------|
| `webserver_url`<br>`웹서버_URL` | string<br>문자열 | **Yes**<br>**예** | URL of the Vivado webserver (e.g. `http://localhost:8088`)<br>Vivado 웹서버의 URL(예: `http://localhost:8088`) |
| `proxy_addr`<br>`프록시_주소` | string<br>문자열 | No | Smart proxy address (`host:port`). Preferred for better performance<br>스마트 프록시 주소(`host:port`). 더 나은 성능을 위해 선호됨 |
| `session_id`<br>`세션_ID` | string<br>문자열 | No | Reconnect to a known session by ID<br>ID로 알려진 세션에 다시 연결 |
| `pid`<br>'pid' | integer<br>정수 | No | Verify process is alive before connecting<br>연결하기 전에 프로세스가 활성 상태인지 확인하세요. |
| `working_dir`<br>`working_dir` | string<br>문자열 | No | Working directory reference<br>작업 디렉토리 참조 |

---

### `vivado_list_sessions`
### `vivado_list_sessions`

List all active Vivado sessions with detailed metadata.
자세한 메타데이터와 함께 모든 활성 Vivado 세션을 나열합니다.
No parameters required.
매개변수가 필요하지 않습니다.

Returns for each session: `session_id`, `session_type`, `deployment_type` (`local`, `ssh`), `vivado_version`, `vivado_edition`, `is_gui_mode`, `hostname`, `uptime_seconds`, `proxy_addr`, and SSH details (`ssh_host`, `ssh_user`).
각 세션에 대한 반환: `session_id`, `session_type`, `deployment_type`(`local`, `ssh`), `vivado_version`, `vivado_edition`, `is_gui_mode`, `hostname`, `uptime_seconds`, `proxy_addr` 및 SSH 세부정보(`ssh_host`, `ssh_user`).

---

### `vivado_cleanup`
### `vivado_cleanup`

Clean up stale sessions and orphaned processes.
오래된 세션과 고아 프로세스를 정리합니다.

| Parameter<br>매개변수 | Type<br>유형 | Required<br>필수 | Description<br>설명 |
|-----------|------|----------|-------------|
| `operation`<br>'작전' | enum<br>열거형 | **Yes**<br>**예** | See operations below<br>아래 작업을 참조하세요. |
| `session_id`<br>`세션_ID` | string<br>문자열 | No | Target specific session<br>타겟 특정 세션 |
| `job_ids`<br>`작업_ID` | array<br>배열 | No | LSF job IDs for `kill_orphaned_lsf`<br>`kill_orphaned_lsf`에 대한 LSF 작업 ID |

**Operations:**
**작업:**

| Operation<br>작동 | Description<br>설명 |
|-----------|-------------|
| `stats`<br>'통계' | Session count and statistics<br>세션 수 및 통계 |
| `health`<br>'건강' | Check all session health<br>모든 세션 상태 확인 |
| `cleanup_stale`<br>`cleanup_stale` | Remove dead sessions from tracking<br>추적에서 죽은 세션 제거 |
| `kill_orphans`<br>`kill_orphans` | Kill Vivado processes started by MCP but not tracked<br>MCP에 의해 시작되었지만 추적되지 않은 Vivado 프로세스 종료 |
| `kill_all`<br>'모두 죽여라' | Kill all MCP-started Vivado processes<br>MCP로 시작된 모든 Vivado 프로세스를 종료합니다. |
| `recover`<br>'회복' | Reconnect to persisted sessions<br>지속 세션에 다시 연결 |
| `list_orphaned_lsf`<br>`list_orphaned_lsf` | List LSF jobs not tracked by any session (Linux, LSF only)<br>어떤 세션에서도 추적되지 않는 LSF 작업 나열(Linux, LSF만 해당) |
| `kill_orphaned_lsf`<br>`kill_orphaned_lsf` | Kill those jobs, by `job_ids` (Linux, LSF only)<br>`job_ids`로 해당 작업을 종료합니다(Linux, LSF에만 해당). |
| `remove_non_recoverable`<br>`remove_non_recoverable` | Remove LSF sessions that cannot be reconnected to<br>다시 연결할 수 없는 LSF 세션을 제거합니다. |
| `kill_non_recoverable`<br>`kill_non_recoverable` | Kill those sessions' jobs (Linux, LSF only)<br>해당 세션의 작업을 종료합니다(Linux, LSF에만 해당). |

> **Local processes: safe by design** — the process operations (`kill_orphans`,
> **로컬 프로세스: 설계상 안전** — 프로세스 작업(`kill_orphans`,
> `kill_all`) only affect Vivado processes started by the MCP server.
> `kill_all`)은 MCP 서버에 의해 시작된 Vivado 프로세스에만 영향을 미칩니다.
> They will
> 그들은 그럴 것이다
> never touch Vivado instances you launched manually.
> 수동으로 시작한 Vivado 인스턴스를 절대 건드리지 마세요.
>
> **LSF jobs: check the list first.** `list_orphaned_lsf` reports jobs *not
> **LSF 작업: 목록을 먼저 확인하세요.** `list_orphaned_lsf`는 작업을 보고합니다.
> tracked by any session*, which is a wider set than "started by this server".
> "이 서버에 의해 시작됨"보다 더 넓은 집합인 모든 세션*에 의해 추적됩니다.
> Run it and read the result before `kill_orphaned_lsf`; pass explicit `job_ids`
> 이를 실행하고 `kill_orphaned_lsf` 전에 결과를 읽으십시오. 명시적인 `job_ids` 전달
> rather than relying on the confirmation prompt, which not every client
> 모든 클라이언트가 그렇지는 않은 확인 프롬프트에 의존하는 대신
> supports.
> 지원합니다.

---

## Command Execution
## 명령 실행

### `vivado_execute`
### 'vivado_execute'

Execute Tcl commands in Vivado.
Vivado에서 Tcl 명령을 실행합니다.
This is the primary tool for all Vivado operations.
이는 모든 Vivado 작업을 위한 기본 도구입니다.

| Parameter<br>매개변수 | Type<br>유형 | Required<br>필수 | Description<br>설명 |
|-----------|------|----------|-------------|
| `command`<br>'명령' | string<br>문자열 | **Yes**<br>**예** | Tcl command(s) to execute. Batch with semicolons<br>실행할 Tcl 명령입니다. 세미콜론이 포함된 일괄 처리 |
| `session_id`<br>`세션_ID` | string<br>문자열 | **Yes**<br>**예** | Target session<br>대상 세션 |
| `mode`<br>'모드' | enum<br>열거형 | No | `auto` (default), `proxy`, or `webserver`<br>`auto`(기본값), `proxy` 또는 `webserver` |
| `capture_log`<br>`캡처_로그` | boolean<br>부울 | No | Capture log output. Default: `true`<br>로그 출력을 캡처합니다. 기본값: `true` |
| `echo`<br>'에코' | boolean<br>부울 | No | Show command in Vivado console. Default: `true`<br>Vivado 콘솔에 명령을 표시합니다. 기본값: `true` |
| `verbosity`<br>'장황함' | enum<br>열거형 | No | How much command output is returned to the agent. Does not affect the Vivado terminal, `vivado.log` or history<br>에이전트에 반환되는 명령 출력의 양입니다. Vivado 터미널, 'vivado.log' 또는 기록에 영향을 주지 않습니다. |

**Batching commands:** Multiple setup commands can be combined with semicolons:
**일괄 처리 명령:** 여러 설정 명령을 세미콜론으로 결합할 수 있습니다.
```tcl
add_files src/top.v; set_property top top [current_fileset]; update_compile_order -fileset sources_1
```

**Standalone commands** (must appear first or alone):
**독립 실행형 명령**(먼저 나타나거나 단독으로 나타나야 함):
- Design flow: `synth_design`, `place_design`, `route_design`, `opt_design`, `phys_opt_design`, `write_bitstream`
- 디자인 흐름: `synth_design`, `place_design`, `route_design`, `opt_design`, `phys_opt_design`, `write_bitstream`
- Run management: `launch_runs`, `wait_on_run`, `reset_runs`
- 실행 관리: `launch_runs`, `wait_on_run`, `reset_runs`
- Project: `open_project`, `close_project`, `open_checkpoint`, `read_checkpoint`, `source`
- 프로젝트: `open_project`, `close_project`, `open_checkpoint`, `read_checkpoint`, `source`
- GUI: `start_gui`, `stop_gui`
- GUI: `start_gui`, `stop_gui`

**Synthesis & Implementation flow:**
**합성 및 구현 흐름:**
```tcl
-- Step 1: launch_runs synth_1 -jobs 8
-- Step 2: wait_on_run synth_1
-- Step 3: launch_runs impl_1 -to_step write_device_image -jobs 8
-- Step 4: wait_on_run impl_1
```

**Long-running commands:** Commands like synthesis or implementation can take minutes to hours.
**장기 실행 명령:** 합성 또는 구현과 같은 명령은 몇 분에서 몇 시간까지 걸릴 수 있습니다.
The server hands off execution and monitors progress in the background.
서버는 실행을 중단하고 백그라운드에서 진행 상황을 모니터링합니다.
The agent automatically polls `vivado_status` to track progress.
에이전트는 자동으로 'vivado_status'를 폴링하여 진행 상황을 추적합니다.
You can check `vivado_log_messages` at any time to see errors or warnings as they appear.
언제든지 `vivado_log_messages`를 확인하여 나타나는 오류나 경고를 확인할 수 있습니다.

---

## Monitoring
## 모니터링

### `vivado_status`
### `vivado_status`

Check session health, monitor synthesis/implementation runs, and recover stuck processes.
세션 상태를 확인하고, 합성/구현 실행을 모니터링하고, 중단된 프로세스를 복구하세요.

| Parameter<br>매개변수 | Type<br>유형 | Required<br>필수 | Description<br>설명 |
|-----------|------|----------|-------------|
| `session_id`<br>`세션_ID` | string<br>문자열 | **Yes**<br>**예** | Target session<br>대상 세션 |
| `action`<br>'액션' | enum<br>열거형 | No | `session` (default), `runs`, `health`, `reset_run`, `cancel_and_relaunch`<br>`session`(기본값), `runs`, `health`, `reset_run`, `cancel_and_relaunch` |
| `run_name`<br>`실행_이름` | string<br>문자열 | No | Run to reset (for `reset_run`)<br>재설정을 위해 실행(`reset_run`의 경우) |
| `jobs`<br>'일자리' | integer<br>정수 | No | Parallel jobs (for `cancel_and_relaunch`)<br>병렬 작업(`cancel_and_relaunch`의 경우) |

Can be called while a command is still running — it doesn't interfere with execution.
명령이 실행되는 동안 호출할 수 있으며 실행을 방해하지 않습니다.

---

### `vivado_log_messages`
### `vivado_log_messages`

Parse the Vivado log file for structured error, warning, critical warning, and info messages.
구조화된 오류, 경고, 중요한 경고 및 정보 메시지에 대해 Vivado 로그 파일을 구문 분석합니다.

| Parameter<br>매개변수 | Type<br>유형 | Required<br>필수 | Description<br>설명 |
|-----------|------|----------|-------------|
| `session_id`<br>`세션_ID` | string<br>문자열 | **Yes**<br>**예** | Target session<br>대상 세션 |
| `filter`<br>'필터' | string<br>문자열 | No | `errors`, `warnings`, `critical`, `info`, `all`. Default: errors + critical + warnings<br>'오류', '경고', '중요', '정보', '모두'. 기본값: 오류 + 위험 + 경고 |
| `max_messages`<br>`최대 메시지` | number<br>번호 | No | Limit per category. Default: all<br>카테고리별로 제한됩니다. 기본값: 모두 |

Can be called at any time, even during long-running commands.
장기 실행 명령 중에도 언제든지 호출할 수 있습니다.

---

### `vivado_history`
### 'vivado_history'

Get or search the complete command history for a session.
세션에 대한 전체 명령 기록을 가져오거나 검색합니다.

| Parameter<br>매개변수 | Type<br>유형 | Required<br>필수 | Description<br>설명 |
|-----------|------|----------|-------------|
| `session_id`<br>`세션_ID` | string<br>문자열 | **Yes**<br>**예** | Target session, or `all` to search across all sessions<br>대상 세션 또는 모든 세션을 검색하려면 'all' |
| `search_query`<br>`검색_쿼리` | string<br>문자열 | No | Filter by command text or output (case-insensitive)<br>명령 텍스트 또는 출력으로 필터링(대소문자 구분) |
| `limit`<br>'한계' | integer<br>정수 | No | Max entries. Default: 50<br>최대 항목 수 기본값: 50 |
| `since_id`<br>`since_id` | integer<br>정수 | No | Only entries after this history ID<br>이 기록 ID 이후의 항목만 |

---

## GUI & Display
## GUI 및 디스플레이

### `vivado_display`
### 'vivado_display'

> **Linux only.** `vivado_display` is not supported on Windows.
> **Linux에만 해당.** `vivado_display`는 Windows에서 지원되지 않습니다.

Manage X11 displays, capture screenshots, and run virtual displays (TigerVNC Xvnc) for headless GUI operation.
헤드리스 GUI 작업을 위해 X11 디스플레이를 관리하고, 스크린샷을 캡처하고, 가상 디스플레이(TigerVNC Xvnc)를 실행합니다.

| Parameter<br>매개변수 | Type<br>유형 | Required<br>필수 | Description<br>설명 |
|-----------|------|----------|-------------|
| `action`<br>'액션' | enum<br>열거형 | **Yes**<br>**예** | `check`, `screenshot`, `virtual_start`, `virtual_stop`, `virtual_status`<br>`확인`, `스크린샷`, `virtual_start`, `virtual_stop`, `virtual_status` |
| `session_id`<br>`세션_ID` | string<br>문자열 | Conditional<br>조건부 | Required for `screenshot`<br>'스크린샷'에 필요 |
| `display_num`<br>`디스플레이_번호` | number<br>번호 | No | Display number for `virtual_start`<br>`virtual_start`에 대한 표시 번호 |
| `width`<br>'너비' | number<br>번호 | No | Screen width. Default: 1920<br>화면 너비. 기본값: 1920 |
| `height`<br>'높이' | number<br>번호 | No | Screen height. Default: 1080<br>화면 높이. 기본값: 1080 |

**Native display vs. virtual display:** By default, the agent may open the GUI on a virtual VNC display rather than your native X11 display.
**기본 디스플레이와 가상 디스플레이 비교:** 기본적으로 에이전트는 기본 X11 디스플레이가 아닌 가상 VNC 디스플레이에서 GUI를 열 수 있습니다.
To ensure Vivado opens on your display:
디스플레이에서 Vivado가 열리도록 하려면:

1. Check your display: `echo $DISPLAY` in your terminal (e.g.
1. 디스플레이를 확인하세요: 터미널에서 `echo $DISPLAY`(예:
1. `:0`, `localhost:10.0`)
1. `:0`, `localhost:10.0`)
2. Tell the agent explicitly: *"Open the GUI on my native display"* or *"Use display :0"*
2. 상담원에게 다음과 같이 명시적으로 알립니다. *"기본 디스플레이에서 GUI를 엽니다."* 또는 *"디스플레이 사용:0"*
3. Or pass `display_mode='x11'` when starting Vivado via `vivado_start` or `vivado_ssh`
3. 또는 `vivado_start` 또는 `vivado_ssh`를 통해 Vivado를 시작할 때 `display_mode='x11'`을 전달하세요.

---

## Research
## 연구

### `vivado_doc_search`
### `vivado_doc_search`

> **A separate MCP server.** `vivado_doc_search` is provided by `amd-doc-search`,
> **별도의 MCP 서버.** `vivado_doc_search`는 `amd-doc-search`에서 제공됩니다.
> not by the Vivado MCP server.
> Vivado MCP 서버가 아닙니다.
> The extension registers both, so it is there
> 확장 프로그램은 둘 다 등록하므로 거기에 있습니다.
> automatically; an MCP-only setup has to add `amd-doc-search` itself, and the
> 자동으로; MCP 전용 설정은 'amd-doc-search' 자체를 추가해야 하며
> tool is simply absent until it does.
> 도구는 그렇게 될 때까지 존재하지 않습니다.

Search the AMD/Xilinx documentation database using hybrid search (BM25 keyword matching + vector semantic search).
하이브리드 검색(BM25 키워드 일치 + 벡터 의미 검색)을 사용하여 AMD/Xilinx 문서 데이터베이스를 검색합니다.

| Parameter<br>매개변수 | Type<br>유형 | Required<br>필수 | Description<br>설명 |
|-----------|------|----------|-------------|
| `query`<br>'쿼리' | string<br>문자열 | **Yes**<br>**예** | Search query<br>검색어 |
| `top_k`<br>`top_k` | number<br>번호 | No | Number of results. Default: 25<br>결과 수. 기본값: 25 |
| `alpha`<br>'알파' | number<br>번호 | No | Semantic vs keyword weight (0.0-1.0). Default: 0.80<br>의미론적 대 키워드 가중치(0.0-1.0). 기본값: 0.80 |

Use lower alpha values (0.3-0.5) for exact Tcl command lookups.
정확한 Tcl 명령 조회를 위해서는 더 낮은 알파 값(0.3-0.5)을 사용하십시오.
Use higher values (0.7-0.9) for conceptual questions.
개념적 질문에는 더 높은 값(0.7-0.9)을 사용하십시오.

---

## Remote Deployment
## 원격 배포

### `vivado_lsf`
### `vivado_lsf`

> **Linux only**, and registered only when LSF is enabled and the LSF client
> **Linux에만 해당**, LSF가 활성화되고 LSF 클라이언트가 활성화된 경우에만 등록됩니다.
> commands are usable on the machine.
> 명령은 머신에서 사용할 수 있습니다.
> If the tool is absent, run `action='check'`
> 도구가 없으면 `action='check'`를 실행하세요.
> expectations against your setup — the server logs the reason it skipped it.
> 설정에 대한 기대 — 서버는 이를 건너뛴 이유를 기록합니다.

Manage Vivado sessions on LSF cluster nodes.
LSF 클러스터 노드에서 Vivado 세션을 관리합니다.

| Parameter<br>매개변수 | Type<br>유형 | Required<br>필수 | Description<br>설명 |
|-----------|------|----------|-------------|
| `action`<br>'액션' | enum<br>열거형 | **Yes**<br>**예** | `check`, `queues`, `start`, `status`, `kill`<br>`check`, `queues`, `start`, `status`, `kill` |
| `working_dir`<br>`working_dir` | string<br>문자열 | Conditional<br>조건부 | Project directory on a shared filesystem. Required for `start`<br>공유 파일 시스템의 프로젝트 디렉터리입니다. '시작'에 필수 |
| `session_type`<br>`세션_유형` | enum<br>열거형 | Conditional<br>조건부 | `general` or `ipi`. Required for `start`<br>'일반' 또는 'ipi'. '시작'에 필수 |
| `queue`<br>'큐' | string<br>문자열 | No | Queue to submit to. Unset auto-detects a suitable one<br>제출할 대기열입니다. 설정되지 않은 경우 적합한 항목을 자동 감지합니다. |
| `submit_mode`<br>`제출_모드` | enum<br>열거형 | No | `auto` (default), `interactive`, `batch`<br>`auto`(기본값), `interactive`, `batch` |
| `memory`<br>'기억' | string<br>문자열 | No | Memory request with a unit, e.g. `8g`, `16GB`, `8192m`. A bare number reads as GB<br>단위를 사용한 메모리 요청(예: '8g', '16GB', '8192m'. 숫자는 GB로 표시됩니다. |
| `slots`<br>`슬롯` | number<br>번호 | No | CPU slots. Vivado caps its own threading at 8<br>CPU 슬롯. Vivado는 자체 스레딩을 8로 제한합니다. |
| `os_type`<br>`os_type` | string<br>문자열 | No | OS/machine type, e.g. `rhel8`. Dropped if the cluster defines no such resource<br>OS/머신 유형(예: 'rhel8'. 클러스터가 해당 리소스를 정의하지 않으면 삭제됩니다. |
| `gui_mode`<br>`gui_mode` | boolean<br>부울 | No | Launch with GUI on the LSF node. Default: `false`<br>LSF 노드에서 GUI로 시작합니다. 기본값: `false` |
| `display_mode`<br>`디스플레이_모드` | enum<br>열거형 | No | `auto`, `vnc`, `x11`, `none`. Default: `none` for Tcl sessions<br>'자동', 'vnc', 'x11', '없음'. 기본값: Tcl 세션의 경우 `none` |
| `display`<br>'디스플레이' | string<br>문자열 | No | X11 display for the remote node. Required when `display_mode='x11'`<br>원격 노드에 대한 X11 표시입니다. `display_mode='x11'`인 경우 필수 |
| `vivado_path`<br>`vivado_path` | string<br>문자열 | No | Only set when the user gives an explicit path; otherwise auto-resolved<br>사용자가 명시적인 경로를 제공하는 경우에만 설정됩니다. 그렇지 않으면 자동 해결됨 |
| `job_id`<br>`작업_ID` | string<br>문자열 | No | LSF job ID, for `status` / `kill`<br>LSF 작업 ID, `status` / `kill`용 |
| `session_id`<br>`세션_ID` | string<br>문자열 | No | Session to look the job up by, for `status` / `kill`<br>`status` / `kill`에 대해 작업을 조회하는 세션 |

**Start here on a new machine.** `action='check'` verifies every prerequisite —
**여기서 새 머신에서 시작하세요.** `action='check'`는 모든 전제 조건을 확인합니다 —
client commands, cluster reachability, queue access, shared filesystems — and
클라이언트 명령, 클러스터 연결 가능성, 대기열 액세스, 공유 파일 시스템 — 및
reports what to fix.
수정해야 할 사항을 보고합니다.
`action='queues'` lists the queues you can actually submit
`action='queues'`는 실제로 제출할 수 있는 대기열을 나열합니다.
to, with their interactive/batch policy and slot limits; use it before choosing
대화형/일괄 정책 및 슬롯 제한을 사용하여 선택하기 전에 사용해 보세요
`queue` rather than guessing a name.
이름을 추측하는 것보다 '큐'.

As with SSH, the working directory must be on a filesystem the cluster node
SSH와 마찬가지로 작업 디렉터리는 클러스터 노드의 파일 시스템에 있어야 합니다.
reaches at the same path.
같은 경로에 도달합니다.

---

### `vivado_ssh`
### `vivado_ssh`

> **Linux only.** `vivado_ssh` is not supported on Windows.
> **Linux에만 해당.** `vivado_ssh`는 Windows에서 지원되지 않습니다.

Start and manage Vivado sessions on remote machines via SSH.
SSH를 통해 원격 머신에서 Vivado 세션을 시작하고 관리하세요.
The remote machine must share the same filesystem (NFS) as the local machine.
원격 시스템은 로컬 시스템과 동일한 파일 시스템(NFS)을 공유해야 합니다.

| Parameter<br>매개변수 | Type<br>유형 | Required<br>필수 | Description<br>설명 |
|-----------|------|----------|-------------|
| `action`<br>'액션' | enum<br>열거형 | **Yes**<br>**예** | `start`, `status`, or `kill`<br>`start`, `status` 또는 `kill` |
| `ssh_host`<br>`ssh_host` | string<br>문자열 | Conditional<br>조건부 | Remote hostname. Required for `start`<br>원격 호스트 이름. '시작'에 필수 |
| `working_dir`<br>`working_dir` | string<br>문자열 | Conditional<br>조건부 | Project directory on shared filesystem. Required for `start`<br>공유 파일 시스템의 프로젝트 디렉터리입니다. '시작'에 필수 |
| `session_type`<br>`세션_유형` | enum<br>열거형 | Conditional<br>조건부 | `general` or `ipi`. Required for `start`<br>'일반' 또는 'ipi'. '시작'에 필수 |
| `gui_mode`<br>`gui_mode` | boolean<br>부울 | No | Launch with GUI. Default: `false`<br>GUI로 시작하세요. 기본값: `false` |
| `display_mode`<br>`디스플레이_모드` | enum<br>열거형 | No | Display management: `auto`, `vnc`, `x11`, `none`. Default: `auto`<br>디스플레이 관리: `auto`, `vnc`, `x11`, `none`. 기본값: `자동` |
| `ssh_user`<br>`ssh_user` | string<br>문자열 | No | SSH username. Default: current user<br>SSH 사용자 이름. 기본값: 현재 사용자 |
| `ssh_key`<br>`ssh_key` | string<br>문자열 | No | Path to SSH private key<br>SSH 개인 키의 경로 |
| `display`<br>'디스플레이' | string<br>문자열 | No | X11 display for the remote node. Required when `display_mode='x11'`<br>원격 노드에 대한 X11 표시입니다. `display_mode='x11'`인 경우 필수 |
| `vivado_path`<br>`vivado_path` | string<br>문자열 | No | Only set when the user gives an explicit path; otherwise auto-resolved<br>사용자가 명시적인 경로를 제공하는 경우에만 설정됩니다. 그렇지 않으면 자동 해결됨 |
| `session_id`<br>`세션_ID` | string<br>문자열 | No | Session ID (for `status` / `kill`)<br>세션 ID(`status` / `kill`용) |

> SSH deployment requires the working directory to be on a shared filesystem (NFS) accessible at the same path on both machines.
> SSH 배포를 위해서는 작업 디렉터리가 두 시스템의 동일한 경로에서 액세스할 수 있는 공유 파일 시스템(NFS)에 있어야 합니다.

---

## Workflow
## 작업 흐름

### `vivado_todos`
### `vivado_todos`

Track multi-step task progress with a structured todo list.
구조화된 할 일 목록을 통해 다단계 작업 진행 상황을 추적하세요.
Tasks appear in the IDE sidebar.
작업은 IDE 사이드바에 나타납니다.

| Parameter<br>매개변수 | Type<br>유형 | Required<br>필수 | Description<br>설명 |
|-----------|------|----------|-------------|
| `operation`<br>'작전' | enum<br>열거형 | **Yes**<br>**예** | `plan`, `update`, `read`, `clear`, `show`<br>`계획`, `업데이트`, `읽기`, `지우기`, `표시` |
| `tasks`<br>'과제' | array<br>배열 | Conditional<br>조건부 | Task list for `plan`<br>`계획`에 대한 작업 목록 |
| `title`<br>'제목' | string<br>문자열 | Conditional<br>조건부 | Plan title. Required for `plan`<br>계획 제목. '계획'에 필수 |
| `id` | string<br>문자열 | Conditional<br>조건부 | Task ID. Required for `update`<br>작업 ID. '업데이트'에 필요 |
| `status`<br>'상태' | enum<br>열거형 | Conditional<br>조건부 | `active`, `done`, `skipped`, `failed`. Required for `update`<br>'활성', '완료', '건너뛰기', '실패'. '업데이트'에 필요 |
| `reason`<br>'이유' | string<br>문자열 | Conditional<br>조건부 | Why the task was skipped or failed. **Required** for `update` with status `skipped` or `failed`<br>작업을 건너뛰거나 실패한 이유 **건너뛰기` 또는 `실패` 상태의 `업데이트`에 필수** |
| `summary`<br>'요약' | string<br>문자열 | No | What was accomplished, for `update` with status `done`<br>'완료' 상태의 '업데이트'에 대해 수행된 작업 |
| `files`<br>`파일` | array<br>배열 | No | Files modified or created during the task<br>작업 중에 수정되거나 생성된 파일 |
| `artifacts`<br>'유물' | array<br>배열 | No | Output files: bitstreams, reports<br>출력 파일: 비트스트림, 보고서 |
| `session_id`<br>`세션_ID` | string<br>문자열 | No | Session the plan belongs to<br>계획이 속한 세션 |

---

### `vivado_client_info`
### `vivado_client_info`

Get information about the connected MCP client and its capabilities.
연결된 MCP 클라이언트 및 해당 기능에 대한 정보를 가져옵니다.
No parameters.
매개변수가 없습니다.
Returns client name, version, and supported features (GUI, terminal, sidebar).
클라이언트 이름, 버전 및 지원되는 기능(GUI, 터미널, 사이드바)을 반환합니다.

---

## Parallel Execution
## 병렬 실행

**Read-only tools** (`vivado_status`, `vivado_log_messages`, `vivado_history`, `vivado_doc_search`, `vivado_list_sessions`) can all run in parallel — even while a command is executing.
**읽기 전용 도구**(`vivado_status`, `vivado_log_messages`, `vivado_history`, `vivado_doc_search`, `vivado_list_sessions`)는 명령이 실행되는 동안에도 모두 병렬로 실행할 수 있습니다.

**Command execution** (`vivado_execute`) is sequential within a single session because Vivado's Tcl interpreter is single-threaded.
**명령 실행**(`vivado_execute`)은 Vivado의 Tcl 인터프리터가 단일 스레드이기 때문에 단일 세션 내에서 순차적입니다.
However, commands on **different sessions** can run in parallel.
그러나 **다른 세션**의 명령은 병렬로 실행될 수 있습니다.

---

## Disabling Tools
## 도구 비활성화

If a tool causes unwanted behavior, you can disable individual MCP tools in your IDE.
도구로 인해 원치 않는 동작이 발생하는 경우 IDE에서 개별 MCP 도구를 비활성화할 수 있습니다.

**VS Code / GitHub Copilot:** In the Chat view, select the **Configure Tools** button (wrench icon) in the chat input field.
**VS Code / GitHub Copilot:** 채팅 보기의 채팅 입력 필드에서 **도구 구성** 버튼(렌치 아이콘)을 선택합니다.
Toggle specific tools on or off.
특정 도구를 켜거나 끕니다.
To disable an entire MCP server, right-click it in the **MCP SERVERS - INSTALLED** section and select **Disable**.
전체 MCP 서버를 비활성화하려면 **MCP SERVERS - INSTALLED** 섹션에서 해당 서버를 마우스 오른쪽 버튼으로 클릭하고 **Disable**을 선택합니다.

**Cursor:** Use the tools picker in the chat input.
**커서:** 채팅 입력에서 도구 선택기를 사용하세요.
Toggle individual tools on or off before sending your prompt.
메시지를 보내기 전에 개별 도구를 켜거나 끕니다.

**GitHub Copilot CLI:**
**GitHub Copilot CLI:**
```bash
copilot --deny-tool='vivado(vivado_display)'
```

**Claude Code:** Add a deny rule to your `~/.claude/settings.json` or `.claude/settings.json`:
**Claude 코드:** `~/.claude/settings.json` 또는 `.claude/settings.json`에 거부 규칙을 추가합니다.
```json
{
  "permissions": {
    "deny": ["mcp__vivado__vivado_display"]
  }
}
```

**OpenAI Codex CLI:** Add `disabled_tools` to the server entry in `~/.codex/config.toml`:
**OpenAI Codex CLI:** `~/.codex/config.toml`의 서버 항목에 `disabled_tools`를 추가합니다.
```toml
[mcp_servers.vivado-mcp]
command = "/path/to/vivado-mcp-server"
args = ["--stdio-bridge"]
disabled_tools = ["vivado_display"]
```

---
