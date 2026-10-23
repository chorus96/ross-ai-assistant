<!-- ko-translation: enabled -->
---
name: vivado-ip-configurator
description: >
  Use this skill for configuring ANY Vivado IP block via natural language. Trigger whenever
  the user asks to configure, customize, parameterize, or set up an IP core — including
  AXI peripherals, processing systems (Zynq PS, Versal CIPS), memory controllers, DSP blocks,
  clocking, resets, DMA, interconnects, or any IP with CONFIG.* properties. Also trigger when
  the user provides a high-level description of desired IP behavior and expects the correct
  set_property -dict to be generated. This skill uses a two-tier approach: Tier 1 builds the
  configuration from documentation, Tier 2 recovers from errors using Vivado's own feedback.
license: MIT
compatibility: Requires Vivado 2026.1, Vivado MCP and AMD documentation search MCP. Python 3.10+ is resolved automatically, preferring Vivado's bundled interpreter.
metadata:
  version: "0.1.0"
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

# IP Configurator — Doc-Driven Configuration with Error Recovery
# IP 구성자 — 오류 복구 기능을 갖춘 문서 기반 구성

## Python quick reference and local API
## Python 빠른 참조 및 로컬 API

In the Vivado session, `ipcfg::help <word>` prints each matching helper's call shape,
Vivado 세션에서 `ipcfg::help <word>`는 일치하는 각 도우미의 호출 형태를 인쇄합니다.
purpose and success/failure results, generated from the loaded code; use it instead of
로드된 코드에서 생성된 목적 및 성공/실패 결과 대신에 그것을 사용
reading `lib/*.tcl`.
`lib/*.tcl`을 읽는 중입니다.
The [local Tcl API inventory](../vivado-ipi-assembler/REFERENCE.md#local-tcl-api-inventory) adds
[로컬 Tcl API 인벤토리](../vivado-ipi-assembler/REFERENCE.md#local-tcl-api-inventory)는 다음을 추가합니다.
gates and mutation ownership.
게이트 및 돌연변이 소유권.
Do not infer a helper's
도우미의 말을 추론하지 마세요.
contract from its name or parse every return as a status line.
이름에서 계약을 체결하거나 모든 반품을 상태 줄로 구문 분석합니다.

**Offline scripts:** use `lib/vivado_python.sh` (`vivado_python.cmd` on Windows).
**오프라인 스크립트:** `lib/vivado_python.sh`(Windows에서는 `vivado_python.cmd`)를 사용하세요.
The skill should try the installed Vivado's bundled Python automatically before
이 스킬은 설치된 Vivado의 번들 Python을 자동으로 시도해야 합니다.
an existing host interpreter.
기존 호스트 인터프리터.
Customers do not need a separate Python installation,
고객은 별도의 Python 설치가 필요하지 않으며,
notebook changes, or a mandatory policy flag.
노트북 변경 또는 필수 정책 플래그.
Use the active Vivado environment;
활성 Vivado 환경을 사용하십시오.
the Linux launcher also discovers `vivado` on PATH.
Linux 실행 프로그램은 PATH에서 `vivado`도 발견합니다.
If discovery fails, locate the
검색에 실패하면
installed Vivado and set `IPCFG_VIVADO_ROOT` to its root, not its executable.
Vivado를 설치하고 `IPCFG_VIVADO_ROOT`를 실행 파일이 아닌 루트로 설정합니다.
Do not ask customers to install Python as the first remedy.
고객에게 첫 번째 해결 방법으로 Python을 설치하도록 요청하지 마세요.
Do not start Vivado merely to run Python. **Inside an already loaded
단지 Python을 실행하기 위해 Vivado를 시작하지 마십시오. **이미 로드된 내부
Vivado session**, reuse `ipcfg::python_exec`; do not build another resolver or use
Vivado 세션**, `ipcfg::python_exec` 재사용; 다른 리졸버를 구축하거나 사용하지 마십시오
`exec [ipcfg::python_exe] ...`, which bypasses the environment wrapper.
환경 래퍼를 우회하는 `exec [ipcfg::python_exe] ...`.

Offline Linux invocation (replace both absolute paths; keep the quotes):
오프라인 Linux 호출(두 절대 경로 모두 교체, 따옴표 유지):

```sh
/bin/sh "/absolute/workspace/.claude/skills/vivado-ip-configurator/lib/vivado_python.sh" \
  "/absolute/workspace with spaces/outputs/check.py" "argument with spaces"
```

This starts Python, not Vivado.
그러면 Vivado가 아닌 Python이 시작됩니다.
A nonzero child exit remains nonzero.
0이 아닌 하위 종료는 0이 아닌 상태로 유지됩니다.
Each shell
각 껍질
invocation probes its runtime afresh; the once-per-process cache below belongs to
호출은 런타임을 새로 조사합니다. 아래의 프로세스당 한 번 캐시는 다음에 속합니다.
the loaded Tcl helper.
로드된 Tcl 도우미.
The offline launchers reject multiple compatible bundles
오프라인 실행 프로그램은 여러 호환 가능한 번들을 거부합니다.
as ambiguous; choose an approved `IPCFG_PYTHON` override in that case.
모호하다; 이 경우 승인된 `IPCFG_PYTHON` 재정의를 선택하세요.

| Existing helper<br>기존 도우미 | Return / use<br>반품/이용 |
|---|---|
| `ipcfg::python_status`<br>`ipcfg::python_status` | `PYTHON:OK source=... version=... executable=...`, or returned `PYTHON_RUNTIME_FAIL:...` text; check the prefix, not just Tcl success.<br>`PYTHON:OK 소스=... 버전=... 실행 파일=...` 또는 `PYTHON_RUNTIME_FAIL:...` 텍스트를 반환했습니다. Tcl 성공뿐만 아니라 접두사도 확인하세요. |
| `ipcfg::python_runtime`<br>`ipcfg::python_runtime` | Dict with `executable`, `root`, `source`, `version`, `score`; resolves lazily, then reuses the cache. Raises on resolution failure.<br>`실행 파일`, `루트`, `소스`, `버전`, `점수`로 사전을 지정합니다. 느리게 해결된 다음 캐시를 재사용합니다. 해결 실패 시 발생합니다. |
| `ipcfg::python_exe`<br>`ipcfg::python_exe` | Selected executable path only, not an environment-aware invocation. Raises on resolution failure.<br>환경 인식 호출이 아닌 실행 가능한 경로만 선택했습니다. 해결 실패 시 발생합니다. |
| `ipcfg::python_exec args...`<br>`ipcfg::python_exec 인수...` | Child output as a string; pass the script path and each argument separately. Resolution, import and nonzero-exit failures raise Tcl errors.<br>문자열로 하위 출력; 스크립트 경로와 각 인수를 별도로 전달합니다. 해결, 가져오기 및 0이 아닌 종료 실패로 인해 Tcl 오류가 발생합니다. |
| `ipcfg::python_script script args...`<br>`ipcfg::python_script 스크립트 인수...` | Preferred script entry point: requires an existing absolute `.py` path before invoking `python_exec`. Invalid paths raise `PYTHON_SCRIPT_FAIL`; child errors remain Tcl errors.<br>선호하는 스크립트 진입점: `python_exec`를 호출하기 전에 기존 절대 `.py` 경로가 필요합니다. 잘못된 경로는 `PYTHON_SCRIPT_FAIL`을 발생시킵니다. 하위 오류는 Tcl 오류로 유지됩니다. |

Once per session, before mutations, record the interpreter and required stdlib
세션당 한 번, 돌연변이가 발생하기 전에 인터프리터와 필수 stdlib를 기록합니다.
imports in the workflow evidence (after loading the helper).
워크플로 증거에서 가져옵니다(도우미를 로드한 후).
The skill performs
스킬이 수행됩니다
this preflight; do not add it to the user's notebook or require them to run it:
이 비행 전; 사용자 노트북에 추가하거나 실행하도록 요구하지 마세요.

```tcl
set python_status [ipcfg::python_status]
puts $python_status
if {![string match {PYTHON:OK *} $python_status]} { error $python_status }
puts [ipcfg::python_exec -c {import argparse, base64, hashlib, json, pathlib, re, sys, typing; print("PYTHON_IMPORTS:OK")}]
```

Run a script with list-safe arguments and absolute paths, including paths with spaces:
공백이 있는 경로를 포함하여 목록 안전 인수와 절대 경로를 사용하여 스크립트를 실행합니다.

```tcl
set script {/absolute/workspace/.claude/skills/vivado-spec-generator/scripts/validate_spec.py}
set arguments [list {/absolute/workspace with spaces/outputs/design_spec.json}]
puts [ipcfg::python_script $script {*}$arguments]
```

Replace example paths with the actual workspace paths.
예제 경로를 실제 작업공간 경로로 바꿉니다.
A stdlib preflight does not
stdlib 프리플라이트는 그렇지 않습니다.
prove optional packages are installed; add only imports required by the chosen script.
선택적 패키지가 설치되어 있음을 증명합니다. 선택한 스크립트에 필요한 가져오기만 추가하세요.
Keep errors visible and stop that stage on failure.
오류를 계속 표시하고 실패 시 해당 단계를 중지합니다.

The Tcl resolver prefers an explicit `IPCFG_PYTHON` override, then the newest usable Vivado
Tcl 리졸버는 명시적인 `IPCFG_PYTHON` 재정의를 선호한 다음 사용 가능한 최신 Vivado를 선호합니다.
bundle, then host `python3`/`python`.
번들로 묶은 다음 `python3`/`python`을 호스트하세요.
An invalid override fails, rather than silently
자동으로 발생하지 않고 잘못된 재정의가 실패합니다.
falling back.
뒤로 넘어짐.
Set the override to an approved interpreter path before preflight.
비행 전 승인된 통역사 경로로 재정의를 설정하세요.
The cache key includes platform, `IPCFG_PYTHON`, `IPCFG_VIVADO_ROOT`, `XILINX_VIVADO`,
캐시 키에는 플랫폼 `IPCFG_PYTHON`, `IPCFG_VIVADO_ROOT`, `XILINX_VIVADO`,
`VIVADO_PATH`, `PATH` and `IPCFG_REQUIRE_BUNDLED`; unchanged keys reuse the runtime without repeat probes.
`VIVADO_PATH`, `PATH` 및 `IPCFG_REQUIRE_BUNDLED`; 변경되지 않은 키는 반복 조사 없이 런타임을 재사용합니다.
`IPCFG_REQUIRE_BUNDLED=1` is optional for controlled tests or an explicitly requested
'IPCFG_REQUIRE_BUNDLED=1'은 제어된 테스트의 경우 선택 사항이거나 명시적으로 요청된 경우입니다.
bundle-only policy; do not enable it by default.
번들 전용 정책; 기본적으로 활성화하지 마십시오.
It disables implicit host fallback,
암시적 호스트 폴백을 비활성화합니다.
but an explicit `IPCFG_PYTHON` override still takes precedence.
그러나 명시적인 `IPCFG_PYTHON` 재정의는 여전히 우선 적용됩니다.
Do not routinely call `python_reset` or re-source libraries to rediscover Python.
Python을 재발견하기 위해 `python_reset`이나 리소스 라이브러리를 일상적으로 호출하지 마세요.
The wrapper temporarily clears `PYTHONHOME`, `PYTHONPATH`, `PYTHONSTARTUP` and
래퍼는 일시적으로 `PYTHONHOME`, `PYTHONPATH`, `PYTHONSTARTUP` 및
`PYTHONINSPECT`, adjusts the child runtime library path (`LD_LIBRARY_PATH` on Linux,
`PYTHONINSPECT`는 하위 런타임 라이브러리 경로(Linux의 경우 `LD_LIBRARY_PATH`,
`PATH` on Windows), and restores prior values or absence on success **and failure**.
Windows에서는 `PATH`), 성공 **및 실패** 시 이전 값 또는 부재를 복원합니다.
Preserve that per-invocation restoration; never permanently unset Vivado's environment.
호출별 복원을 유지합니다. Vivado의 환경을 영구적으로 설정 해제하지 마세요.

## Compact output and retained evidence
## 압축된 출력 및 보관된 증거

Keep successful tool responses under roughly 1 KiB of helper-owned text.
도우미 소유 텍스트의 약 1KiB 미만으로 성공적인 도구 응답을 유지하세요.
Do not
하지 마십시오
print whole CONFIG snapshots, nested PS dictionaries, full configuration diffs,
전체 CONFIG 스냅샷, 중첩된 PS 사전, 전체 구성 차이점을 인쇄합니다.
ownership inventories, caches, or parameter-dependency dumps.
소유권 인벤토리, 캐시 또는 매개변수 종속성 덤프.
Keep them in Tcl
Tcl에 보관하세요
variables and persist large results with `ipcfg::write_evidence` before returning
변수를 사용하고 반환하기 전에 `ipcfg::write_evidence`를 사용하여 큰 결과를 유지합니다.
only counts, required readbacks, and the evidence receipt.
개수, 필수 다시 읽어오기 및 증거 수령만 가능합니다.
End tool batches with
도구 배치 종료
a short `puts`, not a `set` whose return value exposes a large dictionary.
반환 값이 큰 사전을 노출하는 `set`이 아닌 짧은 `puts`입니다.

Choose an existing writable evidence directory under the workflow's output
워크플로 출력에서 기존 쓰기 가능한 증거 디렉터리를 선택합니다.
directory before mutations.
돌연변이 이전의 디렉토리.
Use a fresh filename for each record:
각 레코드에 새로운 파일 이름을 사용하십시오.

For MCP sessions, bind the exact `log_file` from the startup receipt once after
MCP 세션의 경우 시작 영수증에서 정확한 `log_file`을 한 번 바인딩합니다.
loading the library: `set ::ipcfg::native_log_path <absolute-session-log>`.
라이브러리 로딩: `set ::ipcfg::native_log_path <absolute-session-log>`.
Do not guess a shared `vivado.log` path.
공유 `vivado.log` 경로를 추측하지 마세요.
This process-level binding survives project
이 프로세스 수준 바인딩은 프로젝트에서 지속됩니다.
transitions; never point it at another session.
전환; 다른 세션을 가리키지 마십시오.
The wrapper preserves each operation's
래퍼는 각 작업의
native log byte range in a new `.native.log` file and rejects replacement/truncation.
새로운 `.native.log` 파일의 기본 로그 바이트 범위를 지정하고 대체/잘림을 거부합니다.
This is required for operations such as validation that can remove earlier messages
이는 이전 메시지를 제거할 수 있는 유효성 검사와 같은 작업에 필요합니다.
from `write_messages`.
'write_messages'에서.
Without a bound log, counter/export mismatches fail closed.
바인딩된 로그가 없으면 카운터/내보내기 불일치가 실패 처리됩니다.
Native severity counters are also retained.
기본 심각도 카운터도 유지됩니다.
With a bound log, excess WARNING counts
바인딩된 로그를 사용하면 초과 WARNING 횟수가 발생합니다.
are recorded as `unprinted_warning_count`: Vivado validation can increment this counter
'unprinted_warning_count'로 기록됩니다: Vivado 검증은 이 카운터를 증가시킬 수 있습니다
without printing a warning.
경고를 인쇄하지 않고.
Printed ignored-write warnings still fail; ERROR/CRITICAL
인쇄된 무시된 쓰기 경고는 여전히 실패합니다. 오류/위험
counter discrepancies always fail.
카운터 불일치는 항상 실패합니다.
Counts cover the current process, not child runs.
개수는 하위 실행이 아닌 현재 프로세스를 포함합니다.
Keep capture windows inside one project; closing/reopening resets message history.
하나의 프로젝트 내에 캡처 창을 유지합니다. 닫거나 다시 열면 메시지 기록이 재설정됩니다.

```tcl
set evidence_file [file join $evidence_dir "config.[pid].[clock clicks].tcldict"]
puts [ipcfg::write_evidence $evidence_file $details]
```

`write_evidence {path data}` requires an absolute path and refuses to overwrite
`write_evidence {path data}`에는 절대 경로가 필요하며 덮어쓰기를 거부합니다.
existing files.
기존 파일.
It returns a dict with `status RECORDED`, `bytes`, and
`status RECORDED`, `bytes`가 포함된 사전을 반환합니다.
`evidence_path`; the UTF-8 file is a Tcl dict containing `schema_version` and
`증거_경로`; UTF-8 파일은 `schema_version`을 포함하는 Tcl 사전이며
the complete `data`.
완전한 '데이터'.
Read it as data with `read`/`dict get`, never `source` it.
`read`/`dict get`을 사용하여 데이터로 읽으세요. 절대로 `source`하지 마세요.
For configuration calls, use automatic capture rather than manually assembling
구성 호출의 경우 수동으로 조립하는 대신 자동 캡처를 사용합니다.
the evidence.
증거.
It retains the return value, Tcl catch options, and native output:
반환 값, Tcl catch 옵션 및 기본 출력을 유지합니다.

```tcl
set evidence_file [file join $evidence_dir "require.[pid].[clock clicks].tcldict"]
set receipt [ipcfg::capture $evidence_file \
  [list ipcfg::require $cell $desired $reason] {REQ:pinned+applied}]
puts $receipt
if {[dict get $receipt status] ne "OK"} {error "CONFIGURE_FAIL:see $evidence_file"}
```

`capture {path script {expect {}}}` executes once in the caller's scope.
`capture {path script {expect {}}}`는 호출자의 범위에서 한 번 실행됩니다.
It writes
그것은 쓴다
the full result and `-errorinfo`/`-errorcode` automatically, plus immutable
전체 결과와 `-errorinfo`/`-errorcode`가 자동으로 표시되며 변경할 수 없습니다.
`.before.log`/`.after.log` snapshots from `write_messages -severity OUTPUT`.
`.before.log`/`.after.log` 스냅샷은 `write_messages -severity OUTPUT`의 스냅샷입니다.
The receipt includes status, Tcl code, warning/critical/error counts, paths,
영수증에는 상태, Tcl 코드, 경고/중요/오류 수, 경로,
elapsed milliseconds and a short excerpt.
경과된 밀리초 및 짧은 발췌문입니다.
Missing capture support, a changed
캡처 지원 누락, 변경됨
message history without a bound `::ipcfg::native_log_path`, or evidence I/O failure is not
경계 `::ipcfg::native_log_path`가 없는 메시지 기록 또는 I/O 실패가 아니라는 증거
a successful configuration.
성공적인 구성.
With the native log bound, messages come from that log's
기본 로그 바인딩을 사용하면 메시지는 해당 로그에서 나옵니다.
byte range, so Vivado inserting late messages into its earlier history is recorded as
바이트 범위이므로 이전 기록에 늦은 메시지를 삽입하는 Vivado는 다음과 같이 기록됩니다.
`history_reordered 1`, not a failure.
`history_reordered 1`, 실패가 아닙니다.
When the result misses `expect`, the excerpt starts
결과에 `expect`가 누락되면 발췌가 시작됩니다.
`result does not match expected {..}: got '..'` - fix the expected prefix or the call.
`결과가 예상된 {..}과 일치하지 않습니다: '..'을 얻었습니다.` - 예상되는 접두사 또는 호출을 수정합니다.
It never retries the script.
스크립트를 다시 시도하지 않습니다.
An existing evidence file is rejected before execution, so a
기존 증거 파일은 실행 전에 거부되므로
rerun needs a new evidence file name.
재실행에는 새로운 증거 파일 이름이 필요합니다.

Supply accepted status prefixes for status-returning calls; the shared
상태 반환 호출에 허용되는 상태 접두사를 제공합니다. 공유된
`ipcfg::result_status` predicate rejects FAIL/PENDING/PARTIAL/malformed results
`ipcfg::result_status` 조건자는 FAIL/PENDING/PARTIAL/잘못된 결과를 거부합니다.
and accepts NA only when explicitly requested.
명시적으로 요청한 경우에만 NA를 허용합니다.
With no `expect`, OK describes
'기대' 없이 OK 설명
Tcl/native-message execution only, not the meaning of an arbitrary return value.
Tcl/네이티브 메시지 실행만 가능하며 임의 반환 값의 의미는 아닙니다.
Every new native ERROR or CRITICAL WARNING fails capture, even when Tcl returned
모든 새로운 기본 ERROR 또는 CRITICAL WARNING은 Tcl이 반환된 경우에도 캡처에 실패합니다.
zero.
제로.
WARNING lines indicating ignored/disabled writes also fail.
무시/비활성화된 쓰기를 나타내는 WARNING 줄도 실패합니다.
Ordinary
보통
warnings remain counted and visible.
경고는 계속 계산되고 표시됩니다.
Required hardware readbacks still apply.
필수 하드웨어 리드백이 계속 적용됩니다.
Do not wrap expected-failure discovery experiments as successful configuration;
예상되는 실패 발견 실험을 성공적인 구성으로 포장하지 마십시오.
retain their FAIL evidence and interpret it only as probe feedback.
FAIL 증거를 유지하고 이를 프로브 피드백으로만 해석합니다.

Managed `ipiasm::assemble_from_plan` calls capture each operation automatically;
관리되는 `ipiasm::assemble_from_plan` 호출은 각 작업을 자동으로 캡처합니다.
do not double-wrap them.
이중 포장하지 마십시오.
Stage results are persisted automatically, including
다음을 포함하여 단계 결과가 자동으로 유지됩니다.
compiler failures.
컴파일러 오류.
For standalone configuration, use `capture` for each meaningful
독립형 구성의 경우 의미 있는 각 항목에 '캡처'를 사용하세요.
operation or a driver whose final return is its checked status.
최종 반환이 확인된 상태인 운전자 또는 운전자.
Do not fetch or
가져오지 않거나
print entire logs after clean success; the same call supplies counts and paths.
완전한 성공 후 전체 로그를 인쇄합니다. 동일한 호출이 개수와 경로를 제공합니다.

Use `source -notrace` for helper and generated-script loads in Vivado.
Vivado에서 헬퍼 및 생성된 스크립트 로드를 위해 `source -notrace`를 사용하세요.
This removes
이는 제거됩니다
source-code echo, not warnings/errors ([AMD AR 62335](https://adaptivesupport.amd.com/s/article/62335?language=en_US)).
경고/오류가 아닌 소스 코드 에코([AMD AR 62335](https://adaptivesupport.amd.com/s/article/62335?언어=en_US)).
Do not use `-quiet`, global message suppression, or redirect output to `/dev/null`
`-quiet`, 전역 메시지 억제를 사용하거나 출력을 `/dev/null`로 리디렉션하지 마세요.
to meet the output budget.
출력 예산을 충족합니다.
Preserve the native Vivado log.
네이티브 Vivado 로그를 보존합니다.
`capture` inspects
`capture` 검사
native messages without suppressing their console output; use MCP `verbosity="digest"`
콘솔 출력을 억제하지 않는 기본 메시지 MCP `verbosity="digest"`를 사용하세요.
for large native reports and keep the returned log path.
대규모 기본 보고서의 경우 반환된 로그 경로를 유지하세요.
`write_evidence` alone
`write_evidence`만 사용
archives supplied data, not console output.
콘솔 출력이 아닌 제공된 데이터를 보관합니다.
Avoid repeated full rule-registry dumps:
반복되는 전체 규칙 레지스트리 덤프를 방지합니다.
reuse a captured registry for an unchanged session and inspect only the relevant row.
변경되지 않은 세션에 대해 캡처된 레지스트리를 재사용하고 관련 행만 검사합니다.

Successful `board_bind` returns hook and ownership counts; all ownership values
성공적인 `board_bind`는 후크 및 소유권 수를 반환합니다. 모든 소유권 가치
remain available through `ipcfg::board_owned`.
`ipcfg::board_owned`를 통해 계속 사용할 수 있습니다.
Print only relevant entries on
관련 항목만 인쇄
inspection or archive the full result.
전체 결과를 검사하거나 보관합니다.
Failure diagnostics are not shortened by
고장 진단은 다음과 같이 단축되지 않습니다.
the helper.
도우미.

## Design workflow and scratch experiments
## 디자인 워크플로 및 스크래치 실험

For real designs, describe a **requirements checklist**, **configuration readback** and
실제 설계의 경우 **요구사항 체크리스트**, **구성 다시 읽기** 및
**observed hardware interfaces**.
**관찰된 하드웨어 인터페이스**.
Benchmark grades are not design acceptance criteria.
벤치마크 등급은 설계 승인 기준이 아닙니다.
When called during system planning, use the assembler's scratch project and preserve its
시스템 계획 중에 호출되면 어셈블러의 스크래치 프로젝트를 사용하고 해당 프로젝트를 보존합니다.
delivery-project boundary.
배달 프로젝트 경계.
Do not change the target part or create another session.
대상 파트를 변경하거나 다른 세션을 생성하지 마세요.

For each uncertain capability, record the expected observable result before testing.
각각의 불확실한 기능에 대해 테스트하기 전에 예상되는 관찰 가능한 결과를 기록하십시오.
Use the existing creation, automation, parameter-discovery and readback helpers; batch
기존 생성, 자동화, 매개변수 검색 및 다시 읽어들이기 도우미를 사용하세요. 배치
related operations.
관련 작업.
Check actual interface protocol/direction and pin width as well as
실제 인터페이스 프로토콜/방향, 핀 너비 등을 확인하세요.
CONFIG values.
구성 값.
A stored count does not prove a vector grew.
저장된 개수는 벡터가 증가했음을 증명하지 않습니다.
For connection-derived
연결 파생의 경우
properties, make a minimal documented connection and inspect the result after propagation;
속성을 사용하여 최소한으로 문서화된 연결을 만들고 전파 후 결과를 검사합니다.
do not force ownership of a derived property based on a guessed workaround.
추측된 해결 방법을 기반으로 파생 속성의 소유권을 강요하지 마십시오.

Capture defaults before configuration when a before/after comparison is needed.
전후 비교가 필요한 경우 구성 전 기본값을 캡처합니다.
A cell
셀
created and configured in one call is already configured, not a default baseline.
한 번의 호출로 생성되고 구성된 것은 기본 기준이 아니라 이미 구성되어 있습니다.
Defaults
기본값
that satisfy documented requirements are valid and do not require an artificial change.
문서화된 요구 사항을 충족하는 것은 유효하며 인위적인 변경이 필요하지 않습니다.
Scratch success must be rechecked in the delivery design.
스크래치 성공 여부는 배송 디자인에서 다시 확인해야 합니다.
Allow at most 10 total attempts
최대 10번의 시도를 허용합니다.
per IP, including the initial attempt, shared across parameter configuration, automation,
매개변수 구성, 자동화 전반에 걸쳐 공유되는 초기 시도를 포함한 IP당
stub connections and nested helpers.
스텁 연결 및 중첩된 도우미.
Do not restart the count on a nested call or resume.
중첩된 호출에서 카운트를 다시 시작하거나 재개하지 마세요.
Record the attempt count and diagnostic evidence in the run output.
실행 출력에 시도 횟수와 진단 증거를 기록합니다.
After the tenth failed
열번째 실패 후
attempt, stop mutating that IP and ask for user intervention before any further attempt.
시도하는 경우 해당 IP 변경을 중지하고 추가 시도 전에 사용자 개입을 요청하십시오.

### Persistent attempt enforcement
### 지속적인 시도 시행

An explicitly selected external independent-audit controller may supply
명시적으로 선택된 외부 독립 감사 컨트롤러는 다음을 제공할 수 있습니다.
`ipcfg::repair_guard` for bounded corrective transactions after initial construction.
초기 구성 후 제한된 수정 트랜잭션을 위한 `ipcfg::repair_guard`.
In that mode, allow at most ten persistent corrective transactions per logical IP,
해당 모드에서는 논리 IP당 최대 10개의 지속적인 수정 트랜잭션을 허용합니다.
rechecking independent requirements after each and stopping on PASS.
각 요구 사항을 다시 확인하고 PASS에서 중지합니다.
The controller
컨트롤러
must own the project ledger lock, reserve before mutation, enforce declared operations
프로젝트 원장 잠금을 소유해야 하며, 변경 전에 예약하고, 선언된 작업을 시행해야 합니다.
and stable identities, and refuse exhausted legacy ledgers without migration approval.
안정적인 신원을 보장하고 마이그레이션 승인 없이 소진된 레거시 원장을 거부합니다.
Do not set this hook directly or use it to bypass a refusal.
이 후크를 직접 설정하거나 거부를 우회하는 데 사용하지 마십시오.
Without that controller,
해당 컨트롤러가 없으면
the legacy mutation-count policy below remains in force.
아래의 기존 돌연변이 수 정책은 계속 유효합니다.

The configuration helpers use `ipcfg::attempt_call` and persist counts plus diagnostic
구성 도우미는 `ipcfg::attempt_call`을 사용하고 카운트와 진단을 유지합니다.
history in `<project-directory>/.ipcfg/attempts.tcldict`, keyed by BD and full cell path.
BD 및 전체 셀 경로로 입력된 `<project-directory>/.ipcfg/attempts.tcldict`의 기록.
Creation/configuration, board binding, parameter repair, automation, stub recovery and
생성/구성, 보드 바인딩, 매개변수 복구, 자동화, 스텁 복구 및
mutating sub-key discovery share a hard limit of ten attempts.
하위 키 검색을 변경하면 시도 횟수가 10번으로 제한됩니다.
Each actual mutation
각각의 실제 돌연변이
operation counts, including successful ones and fallback attempts; forwarding calls and
성공적인 작업 및 대체 시도를 포함한 작업 횟수 통화 착신 전환 및
empty/idempotent operations do not add a count.
비어 있음/멱등성 작업은 개수를 추가하지 않습니다.
A temporary range probe and its required
임시 범위 프로브 및 필수
restoration form one attempt.
복원은 한 번의 시도로 이루어집니다.
Library reload, assembler initialization and a new Tcl
라이브러리 재로드, 어셈블러 초기화 및 새로운 Tcl
process on the same project do not reset the ledger.
동일한 프로젝트의 프로세스는 원장을 재설정하지 않습니다.

`CONFIGURE_FAIL:ATTEMPT_LIMIT` requires user intervention, not another recovery path.
`CONFIGURE_FAIL:ATTEMPT_LIMIT`에는 다른 복구 경로가 아닌 사용자 개입이 필요합니다.
An unreadable ledger or existing lock fails closed with `ATTEMPT_STATE`; preserve the
읽을 수 없는 원장 또는 기존 잠금이 `ATTEMPT_STATE`로 닫히지 못했습니다. 보존하다
files and investigate before asking the user to authorize recovery.
사용자에게 복구 승인을 요청하기 전에 파일을 검토하고 조사하세요.
There is no automatic
자동은 없습니다
reset or budget extension.
재설정 또는 예산 확장.
Preserve the project directory when resuming.
재개할 때 프로젝트 디렉터리를 유지합니다.

This is helper-level enforcement, not a global Vivado command interceptor.
이는 글로벌 Vivado 명령 인터셉터가 아닌 도우미 수준 시행입니다.
Direct Tcl
직접 Tcl
writes and assembler infrastructure writes outside these helpers are not automatically
이러한 도우미 외부의 쓰기 및 어셈블러 인프라 쓰기는 자동으로 수행되지 않습니다.
counted.
계산되었습니다.
Do not use those paths to bypass exhaustion.
피로를 우회하기 위해 이러한 경로를 사용하지 마십시오.
A different project, BD or cell
다른 프로젝트, BD 또는 셀
identity has a separate ledger entry; do not rename or recreate context to obtain retries.
신원에는 별도의 원장 항목이 있습니다. 재시도를 얻기 위해 컨텍스트의 이름을 바꾸거나 컨텍스트를 다시 생성하지 마십시오.

## Overview
## 개요

This skill configures Vivado IP blocks from natural-language prompts using a two-tier approach
이 기술은 2계층 접근 방식을 사용하여 자연어 프롬프트에서 Vivado IP 블록을 구성합니다.
that treats Vivado as the source of truth.
Vivado를 진실의 근원으로 취급합니다.
It does NOT rely on pre-built parameter databases
사전 구축된 매개변수 데이터베이스에 의존하지 않습니다.
or exhaustive JSON dumps of IP configuration spaces.
또는 IP 구성 공간의 철저한 JSON 덤프.

**Tier 1 — Doc-Driven First Attempt:**
**계층 1 - 문서 기반 첫 번째 시도:**
1. Search AMD documentation for the IP's configuration parameters
1. IP의 구성 매개변수에 대해서는 AMD 문서를 검색하세요.
2. Build a `set_property -dict` from the docs
2. 문서에서 `set_property -dict`를 빌드하세요.
3. Execute it via `vivado_execute`
3. `vivado_execute`를 통해 실행하세요.

**Tier 2 — Error-Driven Recovery (only if Tier 1 fails):**
**계층 2 - 오류 기반 복구(계층 1이 실패한 경우에만):**
1. Parse the error message from the failed `set_property -dict`
1. 실패한 `set_property -dict`의 오류 메시지를 구문 분석합니다.
2. Search docs for the specific parameters/constraints mentioned in the error
2. 오류에 언급된 특정 매개변수/제약조건에 대한 문서를 검색하세요.
3. Build a corrected `-dict` and retry
3. 수정된 `-dict`를 빌드하고 다시 시도하세요.

Key principle: `set_property -dict` is **atomic for hard errors** — if it throws, no
주요 원칙: `set_property -dict`는 **하드 오류에 대해 원자적**입니다. 오류가 발생하면 아니요
parameters are changed, and the error messages are rich and specific.
매개변수가 변경되고 오류 메시지가 풍부하고 구체적입니다.

**Caveat (critical):** In IP Integrator (BD) mode, an *unknown* CONFIG parameter does
**주의사항(중요):** BD(IP Integrator) 모드에서는 *알 수 없는* CONFIG 매개변수가
**not** throw.
**던지지 마세요**.
Vivado emits a non-fatal `CRITICAL WARNING [BD 41-1276] ... does not exist`
Vivado는 치명적이지 않은 'CRITICAL WARNING [BD 41-1276] ... 존재하지 않습니다'를 표시합니다.
(or `[IP_Flow 19-7090] Invalid parameter ...
(또는 `[IP_Flow 19-7090] 잘못된 매개변수...
Ignoring`), `catch` returns 0, and the script
Ignoring`), `catch`는 0을 반환하고 스크립트는
would falsely report success.
성공을 거짓으로 보고할 것입니다.
Therefore Tier 1 must **verify** that the parameters actually
따라서 Tier 1에서는 매개변수가 실제로
took effect (see Step 3, Phase 3 verification) and the agent must **scan the execute output**
적용되었으며(3단계, 3단계 확인 참조) 에이전트는 **실행 출력을 스캔**해야 합니다.
for these warning patterns before declaring SUCCESS — never trust the `SUCCESS:` line alone.
SUCCESS를 선언하기 전에 이러한 경고 패턴에 대해 — 절대 'SUCCESS:' 줄만 신뢰하지 마세요.

## Operating Modes
## 작동 모드

This skill runs in one of two modes.
이 스킬은 두 가지 모드 중 하나로 실행됩니다.
Choose the mode at the start of the session and keep it.
세션 시작 시 모드를 선택하고 유지하세요.

| Mode<br>모드 | When<br>언제 | Behavior<br>행동 |
|---|---|---|
| `benchmark` (default)<br>`벤치마크`(기본값) | Following `execution_protocol.md` or any automated/blind run<br>`execution_protocol.md` 또는 자동/블라인드 실행을 따르십시오. | Deterministic: never ask the user. **Must** call `vivado_execute`, write the artifact, then finish. Skip exploratory reads.<br>결정적: 사용자에게 묻지 마십시오. **반드시** `vivado_execute`를 호출하고 아티팩트를 작성한 다음 완료해야 합니다. 탐색적 읽기를 건너뜁니다. |
| `design`<br>'디자인' | Helping a user build a real, working design<br>사용자가 실제로 작동하는 디자인을 구축하도록 지원 | May **ask clickable clarifying questions** when doc search is inconclusive, and proactively uses Block Automation / Configurable Example Designs to flesh out the design.<br>문서 검색이 결론을 내리지 못한 경우 **클릭 가능한 명확한 질문**을 하고 블록 자동화/구성 가능한 예제 디자인을 적극적으로 사용하여 디자인을 구체화할 수 있습니다. |

If the mode is not stated, infer `benchmark` when a protocol/blind-run file is driving you;
모드가 명시되지 않은 경우 프로토콜/블라인드 실행 파일이 사용자를 구동할 때 '벤치마크'를 추론하세요.
otherwise assume `design`.
그렇지 않으면 '디자인'을 가정합니다.
Steps below are tagged **(both)**, **(benchmark)**, or **(design)**
아래 단계에는 **(둘 다)**, **(벤치마크)** 또는 **(디자인)** 태그가 지정되어 있습니다.
where the behavior differs.
행동이 다른 곳.
Everything not tagged applies to both.
태그가 지정되지 않은 모든 항목은 둘 다에 적용됩니다.

## Always report coverage (REQUIRED, both modes)
## 항상 적용 범위 보고(필수, 두 모드 모두)

You MUST tell the user, in plain language, whether the IP was **fully parameterized** from
IP가 **완전히 매개변수화**되었는지 여부를 일반 언어로 사용자에게 알려야 합니다.
the prompt — and if not, **exactly which part(s) of the prompt could not be applied, and why**.
프롬프트 — 그렇지 않은 경우 **정확히 프롬프트의 어떤 부분을 적용할 수 없었으며 그 이유**.
Never silently grade `partial` / `creation_only` / `negative`.
`partial` / `creation_only` / `negative`를 자동으로 평가하지 마세요.

Drive this from the Step 1.5 requirement ledger.
1.5단계 요구 사항 원장에서 이를 구동합니다.
Every atomic requirement ends in one outcome:
모든 원자적 요구 사항은 하나의 결과로 끝납니다.
- `applied` — set and verified (stuck + intent realized)
- `applied` — 설정 및 확인됨(멈춤 + 의도 실현)
- `default` — already correct by default (`value_src=default`, acceptable)
- `default` — 기본적으로 이미 정확함(`value_src=default`, 허용됨)
- `unapplied:<reason>` — could not be parameterized; `<reason>` is one of
- `unapplied:<reason>` — 매개변수화할 수 없습니다. `<이유>`는 다음 중 하나입니다.
`runtime-register-only`, `integration-derived`, `gated-no-enabler`, `not-a-config-param`,
`런타임 등록 전용`, `통합 파생`, `gated-no-enabler`, `not-a-config-param`,
`value-out-of-range`, `wrong-ip-for-capability`.
'값이 범위를 벗어났습니다', '기능에 대한 IP가 잘못되었습니다'.

Emit the coverage report whenever the result is not fully applied:
결과가 완전히 적용되지 않을 때마다 적용 범위 보고서를 내보냅니다.

```tcl
puts [ipcfg::coverage_report $ledger]   ;# COVERAGE:FULL | COVERAGE:PARTIAL + the unapplied list
```

Then say to the user, verbatim about the gaps: "I could **not** fully parameterize `<IP>` from
그런 다음 사용자에게 공백에 대해 다음과 같이 말합니다. "'<IP>'를 완전히 매개변수화할 수 **없습니다**.
the prompt.
프롬프트.
Applied: <list>.
적용됨: <목록>.
Could NOT apply: '<the exact prompt phrase>' because <reason>."
적용할 수 없습니다: '<정확한 프롬프트 문구>' 왜냐하면 <이유> 때문입니다."
Quote the actual prompt fragment for each gap — do not paraphrase it away.
각 공백에 대해 실제 프롬프트 부분을 인용하세요. 다른 말로 바꾸어 표현하지 마세요.

## MCP Tools Used
## 사용된 MCP 도구

Only THREE tools are needed:
세 가지 도구만 필요합니다:

| Tool<br>도구 | Purpose<br>목적 | When<br>언제 |
|------|---------|------|
| `vivado_doc_search`<br>`vivado_doc_search` | Find IP parameters, valid values, dependencies<br>IP 매개변수, 유효한 값, 종속성 찾기 | Tier 1 (always), Tier 2 (on error)<br>Tier 1(항상), Tier 2(오류 시) |
| `vivado_execute`<br>'vivado_execute' | Run `set_property -dict` and read-back commands<br>`set_property -dict` 및 다시 읽기 명령 실행 | Both tiers<br>두 계층 모두 |
| `vivado_log_messages`<br>`vivado_log_messages` | Additional diagnostics if error string is ambiguous<br>오류 문자열이 모호한 경우 추가 진단 | Tier 2 (rare)<br>2등급(희귀) |

## Workflow
## 작업 흐름

### Step 0: Part resolution (guarded — before anything else)
### 0단계: 부분 해결(보호 - 무엇보다 먼저)

Many IPs only exist on specific device families. **A part swap is destructive to a
많은 IP는 특정 장치 제품군에만 존재합니다. **부품 교환은 제품에 파괴적입니다.
design-in-progress**: existing IP may be incompatible with the new part, will need
설계 진행 중**: 기존 IP가 새 부품과 호환되지 않을 수 있습니다.
upgrading, and may then mismatch the original prompts.
업그레이드하면 원래 프롬프트와 일치하지 않을 수 있습니다.
So treat swapping as **opt-in and
따라서 교환을 **선택으로 처리하고
last-resort**, not routine:
최후의 수단**, 일상적이지 않음:

1. **Check availability on the current part first** — `ipcfg::vlnv_ok <vlnv>`.
1. **현재 부품의 가용성을 먼저 확인하세요** — `ipcfg::vlnv_ok <vlnv>`.
1. If the IP is
1. IP가 다음과 같은 경우
available, do **not** swap.
가능합니다. 교환하지 **마세요**.
2. **Only if unavailable**, consider a swap.
2. **사용할 수 없는 경우에만** 교환을 고려하세요.
2. `ipcfg::ensure_part <part>` is **guarded**: if
2. `ipcfg::ensure_part <part>`는 **보호됩니다**: if
other `bd_cells` already exist it returns `WARN:SWAP_BLOCKED:...` instead of swapping.
다른 `bd_cells`이 이미 존재하면 교체 대신 `WARN:SWAP_BLOCKED:...`를 반환합니다.
   - **(design)** On `WARN:SWAP_BLOCKED`, **ask the user** (clickable) whether to swap,
   - **(설계)** `WARN:SWAP_BLOCKED`에서 **사용자에게** (클릭 가능) 교체 여부를 묻고,
warning that existing IP may need upgrading and could mismatch their intent.
기존 IP를 업그레이드해야 할 수도 있고 의도와 일치하지 않을 수도 있음을 경고합니다.
Only call
통화만
`ipcfg::ensure_part <part> 1` (force) after they confirm.
`ipcfg::ensure_part <part> 1`(강제) 확인 후.
   - **(benchmark)** The BD holds only the throwaway test cell, so the guard passes and the
   - **(벤치마크)** BD는 일회용 테스트 셀만 보유하고 있어서 가드는 통과하고
swap proceeds; restore the part in cleanup.
스왑 수익; 부품을 정리하여 복원합니다.
3. Never swap silently mid-design.
3. 설계 도중에 조용히 교체하지 마십시오.

**Step 0a — Doc search for device family (combined with parameter search):**
**0a단계 — 장치 제품군에 대한 문서 검색(매개변수 검색과 결합):**

Include device/family keywords in the Tier 1 doc search so a single call covers both
단일 통화로 두 가지 모두를 다룰 수 있도록 Tier 1 문서 검색에 장치/제품군 키워드를 포함합니다.
part compatibility and CONFIG parameters:
부품 호환성 및 CONFIG 매개변수:

```
vivado_doc_search("<IP name> supported devices configuration parameters")
```

From the results, extract:
결과에서 다음을 추출합니다.
- Which device families support this IP (Zynq-7000, Zynq US+, UltraScale+, Versal, etc.)
- 이 IP를 지원하는 장치 제품군(Zynq-7000, Zynq US+, UltraScale+, Versal 등)
- The CONFIG.* property names (used in Step 3)
- CONFIG.* 속성 이름(3단계에서 사용됨)

**Step 0b — Determine if a part swap is needed:**
**0b단계 - 부품 교체가 필요한지 확인합니다.**

If the doc search indicates the IP requires a different device family than the current
문서 검색에서 해당 IP에 현재와 다른 장치 제품군이 필요하다고 나타나는 경우
project part, select a representative part from the static fallback map below.
프로젝트 부분인 경우 아래 정적 대체 맵에서 대표 부분을 선택합니다.
The AI
AI
should prefer doc-search results when they name a specific family; the map is a fallback.
특정 가족의 이름을 지을 때 문서 검색 결과를 선호해야 합니다. 지도는 대체입니다.

Static fallback map (IP -> required part):
정적 대체 맵(IP -> 필수 부분):

| IP | Required Part<br>필수부분 | Why<br>왜? |
|---|---|---|
| `processing_system7`<br>`처리_시스템7` | `xc7z020clg484-1`<br>`xc7z020clg484-1` | Zynq-7000 PS<br>Zynq-7000PS |
| `versal_cips`<br>`versal_cips` | `xcvc1902-vsva2197-2MP-e-S`<br>`xcvc1902-vsva2197-2MP-e-S` | Versal CIPS<br>버설 CIPS |
| `axi_noc`<br>`axi_noc` | `xcvc1902-vsva2197-2MP-e-S`<br>`xcvc1902-vsva2197-2MP-e-S` | Versal NoC<br>버설 NoC |
| `mrmac`<br>'mrmac' | `xcvc1902-vsva2197-2MP-e-S`<br>`xcvc1902-vsva2197-2MP-e-S` | Versal Multi-Rate MAC<br>Versal 다중 속도 MAC |
| `xdma`<br>`xdma` | `xcvu9p-flga2104-2L-e`<br>`xcvu9p-flga2104-2L-e` | Needs GT quads (Virtex US+)<br>GT 쿼드 필요(Virtex US+) |
| `cmac_usplus`<br>'cmac_usplus' | `xcvu9p-flga2104-2L-e`<br>`xcvu9p-flga2104-2L-e` | 100G needs GT quads<br>100G에는 GT 쿼드가 필요합니다 |
| `gt_quad_base`<br>`gt_quad_base` | `xcvc1902-vsva2197-2MP-e-S`<br>`xcvc1902-vsva2197-2MP-e-S` | Versal GT Quad IP<br>Versal GT 쿼드 IP |
| `hbm`<br>'hbm' | `xcvu37p-fsvh2892-2L-e`<br>`xcvu37p-fsvh2892-2L-e` | HBM-equipped Virtex US+<br>HBM 탑재 Virtex US+ |
| `rfdc`<br>`rfdc` | `xczu28dr-ffvg1517-2-e`<br>`xczu28dr-ffvg1517-2-e` | RFSoC<br>RFSoC |
| `axi_ethernet` (10G/25G)<br>`axi_ethernet`(10G/25G) | `xcvu9p-flga2104-2L-e`<br>`xcvu9p-flga2104-2L-e` | 10G+ needs GT<br>10G+에는 GT가 필요합니다 |

**Step 0c — Part swap TCL (inline in the same script):**
**0c단계 — 부품 교체 TCL(동일 스크립트의 인라인):**

When a swap is actually warranted (Step 0 guard passed, or user-confirmed in design mode),
교체가 실제로 보장되는 경우(0단계 가드 통과 또는 설계 모드에서 사용자 확인)
it is embedded at the top of the configuration TCL via the guarded `ipcfg::ensure_part`
이는 보호된 `ipcfg::ensure_part`를 통해 구성 TCL의 맨 위에 포함됩니다.
(pass `1` to force only after confirmation), and restored after cleanup.
(확인 후에만 강제로 적용하려면 '1'을 전달), 정리 후 복원됩니다.
See the template
템플릿 보기
in Step 3.
3단계에서.

**Step 0d — Capability pre-check (IP vs. intent):**
**0d단계 - 기능 사전 확인(IP 대 의도):**

Before building the dict, sanity-check that the chosen IP can actually deliver the
사전을 작성하기 전에 선택한 IP가 실제로
requested capability.
요청한 기능.
Many prompts name a feature that belongs to a *different* IP, and
많은 프롬프트에서는 *다른* IP에 속한 기능을 지정합니다.
the BD will silently ignore the unsupported settings rather than error clearly.
BD는 오류를 명확히 하기보다는 지원되지 않는 설정을 자동으로 무시합니다.
When the
때
doc search shows the requested rate/interface/mode is outside the IP's envelope, do **not**
문서 검색에 요청된 속도/인터페이스/모드가 IP 범위를 벗어난 것으로 표시됩니다. **하지 마세요**
attempt a doomed config — instead **resolve it as a negative**, exactly like a
불운한 구성을 시도하십시오. 대신에 **부정적인 것으로 해결**하십시오.
`VALUE_OUT_OF_RANGE` recovery:
`VALUE_OUT_OF_RANGE` 복구:

1. Apply the **closest valid** setting the named IP *does* support (e.g. on `axi_ethernet`,
1. 명명된 IP가 지원 *지원*하는 **가장 가까운 유효** 설정을 적용합니다(예: `axi_ethernet`,
fall back to `1000BaseX`/`SGMII` when 10G/25G was requested).
10G/25G가 요청되면 `1000BaseX`/`SGMII`로 대체됩니다.
2. Grade `negative`, and in `fidelity_note` state both the IP's documented limit **and** the
2. '부정' 등급, 'fidelity_note'에 IP의 문서화된 제한 **및** 둘 다를 명시합니다.
correct alternative IP for the requested capability.
요청된 기능에 대한 올바른 대체 IP입니다.
3. This is a **resolved** outcome (`tier2_success: true`), not an unrecovered failure — a
3. 이는 복구되지 않은 실패가 아닌 **해결된** 결과(`tier2_success: true`)입니다.
provably-impossible-on-the-named-IP request is a legitimate negative, never a fall-through
명명된 IP 요청은 증명 불가능할 정도로 합법적인 부정이며 결코 실패하지 않습니다.
fail. (If the request becomes impossible only via a Vivado error rather than the doc, the
실패하다. (문서가 아닌 Vivado 오류로만 요청이 불가능하게 된 경우,
same resolution applies once the `VALUE_OUT_OF_RANGE`/`PARAM_NOT_FOUND` error confirms it.)
`VALUE_OUT_OF_RANGE`/`PARAM_NOT_FOUND` 오류가 확인되면 동일한 해결 방법이 적용됩니다.)

> Note: this path keeps fidelity to the *named* IP.
> 참고: 이 경로는 *이름이 지정된* IP에 대한 충실도를 유지합니다.
> Substituting a different IP to deliver
> 다른 IP로 대체하여 전달
> the capability (e.g.
> 능력(예를 들어
> `axi_ethernet` → `xxv_ethernet`) is intentionally **out of scope** —
> `axi_ethernet` → `xxv_ethernet`)은 의도적으로 **범위를 벗어났습니다** —
> it would change the IP the user asked for.
> 사용자가 요청한 IP가 변경됩니다.
> Only do that if the user explicitly asks to
> 사용자가 명시적으로 요청한 경우에만 그렇게 하세요.
> "use whatever IP achieves X."
> "X를 달성하는 모든 IP를 사용하십시오."

General signals that the prompt targets the wrong IP:
프롬프트가 잘못된 IP를 대상으로 한다는 일반 신호는 다음과 같습니다.
- A line rate / data rate above the IP's documented maximum (escalate to the faster IP).
- IP의 문서화된 최대값을 초과하는 회선 속도/데이터 속도(더 빠른 IP로 에스컬레이션)
- An interface the IP doesn't expose in BD (e.g. an AXI4 memory-mapped interface on an IP
- IP가 BD에서 노출하지 않는 인터페이스(예: IP의 AXI4 메모리 매핑 인터페이스)
whose doc lists only a "Native" interface — that needs a separate controller/bridge IP).
해당 문서에는 별도의 컨트롤러/브리지 IP가 필요한 "네이티브" 인터페이스만 나열되어 있습니다.
- A mode named as if it were a `CONFIG` enum but documented as a separate IP or domain.
- 'CONFIG' 열거형인 것처럼 이름이 지정되었지만 별도의 IP 또는 도메인으로 문서화된 모드입니다.

The mapping of which IP provides a capability comes from the doc search, not from a static
어떤 IP가 기능을 제공하는지에 대한 매핑은 정적 검색이 아닌 문서 검색에서 나옵니다.
table — let the documentation name the right IP, then grade `negative` with that name in
테이블 — 문서에 올바른 IP 이름을 지정하고 해당 이름으로 '부정적' 등급을 지정합니다.
`fidelity_note`.
'fidelity_note'.

### Step 1: Parse the intent
### 1단계: 의도 분석

Extract from the user's prompt:
사용자 프롬프트에서 추출합니다.
- **IP type** — which IP core (e.g., AXI GPIO, AXI DMA, Zynq PS, MIPI CSI-2 RX)
- **IP 유형** — IP 코어(예: AXI GPIO, AXI DMA, Zynq PS, MIPI CSI-2 RX)
- **Key requirements** — the functional parameters (width, channels, modes, protocols)
- **주요 요구 사항** — 기능 매개변수(폭, 채널, 모드, 프로토콜)
- **Cell name** — if specified, otherwise use a sensible default
- **셀 이름** — 지정된 경우, 그렇지 않으면 적절한 기본값을 사용합니다.

**A capability may live in a standalone IP OR be integrated into a larger system IP.**
**기능은 독립형 IP에 있거나 더 큰 시스템 IP에 통합될 수 있습니다.**
The same feature can be delivered either by a dedicated IP or by a sub-block of a
동일한 기능은 전용 IP 또는 하위 블록을 통해 제공될 수 있습니다.
processing-system / platform IP, configured through a nested `*_CONFIG` dictionary.
중첩된 `*_CONFIG` 사전을 통해 구성된 처리 시스템/플랫폼 IP.
Let doc
문서를 보자
search name where the capability lives; when both paths exist, prefer the one the
해당 기능이 있는 이름을 검색하세요. 두 경로가 모두 존재하는 경우 다음 중 하나를 선호합니다.
surrounding design / example design actually uses.
주변 디자인 / 실제 사용하는 디자인 예시입니다.
Picking the standalone IP when the
독립형 IP 선택
device delivers the feature inside its system IP (or vice-versa) is an
장치가 시스템 IP 내부에 기능을 전달하는 경우(또는 그 반대)는
**`unapplied:wrong-ip-for-capability`** outcome — keep the identification doc-grounded, not
**`unapplied:wrong-ip-for-capability`** 결과 — 신분증을 문서 기반으로 유지하세요.
name-based.
이름 기반.

**When to ask the user (design mode):** If `vivado_doc_search` is inconclusive **and** the
**사용자에게 질문하는 경우(디자인 모드):** `vivado_doc_search`가 결론에 이르지 못하고 **그리고**
unknown materially changes the IP choice or the configuration (e.g. which IP/sub-block
알 수 없음으로 인해 IP 선택이나 구성이 실질적으로 변경됩니다(예: 어떤 IP/하위 블록이
delivers a capability, or an ambiguous requirement), ask a short **clickable** question
기능을 제공하거나 요구 사항이 모호한 경우) 짧은 **클릭 가능한** 질문을 하세요.
rather than guessing or emitting a vague/formulaic answer.
모호하고 정형화된 답변을 추측하거나 발표하는 대신.
In **benchmark mode**, never ask
**벤치마크 모드**에서는 묻지 마세요.
— make the documented deterministic choice and record the assumption in the ledger.
— 문서화된 결정론적 선택을 하고 원장에 가정을 기록합니다.

### Step 1.5: Requirement ledger (decompose the prompt)
### 1.5단계: 요구사항 원장(프롬프트 분해)

Before any config, decompose the prompt into a list of **atomic requirements** — one
구성하기 전에 프롬프트를 **원자적 요구 사항** 목록으로 분해합니다.
per named feature, toggle, mode, count, width, rate, or protocol the user asked for.
사용자가 요청한 명명된 기능, 토글, 모드, 개수, 너비, 속도 또는 프로토콜별로.
Carry this ledger through Tier 1 and Tier 2 as a coverage contract:
적용 범위 계약으로 Tier 1 및 Tier 2를 통해 이 원장을 수행합니다.

| Requirement (from prompt)<br>요구사항(프롬프트에서) | CONFIG param<br>CONFIG 매개변수 | Doc evidence (why this param)<br>문서 증거(이 매개변수가 있는 이유) | Expected observable change<br>예상되는 관찰 가능한 변화 |
|---|---|---|---|
| "user-data-type filtering on"<br>"사용자 데이터 유형 필터링 켜기" | `CONFIG.<X>`<br>`구성.<X>` | doc line stating X does this<br>X가 이 작업을 수행한다는 내용의 문서 라인 | default 0 -> 1<br>기본값 0 -> 1 |
| "2 channels"<br>"2채널" | `CONFIG.<Y>`<br>`구성.<Y>` | doc line stating Y is channel count<br>Y가 채널 수임을 나타내는 문서 라인 | default 1 -> 2<br>기본값 1 -> 2 |

Rules:
규칙:
- **Every requirement must map to a specific CONFIG param with a doc citation.** A
- **모든 요구사항은 문서 인용과 함께 특정 CONFIG 매개변수에 매핑되어야 합니다.** A
requirement with no mapped param is an *uncovered* requirement — never grade `full`
매핑된 매개변수가 없는 요구 사항은 *공개* 요구 사항입니다. '전체' 등급을 지정하지 마세요.
while any requirement is uncovered.
모든 요구 사항이 밝혀지는 동안.
- A compound noun phrase is often **multiple** requirements (e.g. "dual memory
- 복합 명사구는 종종 **여러** 요구 사항을 갖습니다(예: "이중 메모리"
controller with discrete components" = controller-count *and* component-type).
개별 구성 요소가 있는 컨트롤러" = 컨트롤러 수 *및* 구성 요소 유형).
Split
분할
them; a single param rarely covers a compound phrase.
그들; 단일 매개변수가 복합 문구를 다루는 경우는 거의 없습니다.
- The "Expected observable change" column feeds the Step 3 intent audit — record the
- "관찰 가능한 변경 예상" 열은 3단계 의도 감사를 제공합니다.
default you expect to move away from, so a no-op (value already at the target by
기본값에서 벗어날 것으로 예상되므로 아무 작업도 수행하지 않습니다(값이 이미 목표에 도달함).
default) is detectable.
기본값)을 감지할 수 있습니다.

### Step 2: Doc search (Tier 1)
### 2단계: 문서 검색(Tier 1)

**Consult the learned cache FIRST (0 MCP calls).** Before any doc search, check whether a
**학습된 캐시를 먼저 참조하십시오(MCP 호출 0개).** 문서 검색 전에
prior run already discovered where this feature lives:
이전 실행에서 이 기능이 어디에 있는지 이미 발견했습니다.
```tcl
set hit [ipcfg::cache_get <ip> "<feature phrase>"]   ;# "" on miss
```
On a **hit**, you get the `param`, `shape`, `enabler`, and `value_src` that were earned in a
**적중** 시에 획득한 `param`, `shape`, `enabler` 및 `value_src`를 얻습니다.
previous run.
이전 실행.
Do a cheap `get_property` existence re-verify (the param still exists for this
저렴한 `get_property` 존재 여부를 다시 확인하십시오(이에 대한 매개변수는 여전히 존재합니다).
IP version), then go straight to applying your value — skip the doc search.
IP 버전) 그런 다음 바로 가치 적용으로 이동하세요. 문서 검색을 건너뛰세요.
On a **miss**,
**실수** 시,
fall through to the doc search below; once you have *confirmed* a mapping (it stuck and
아래 문서 검색으로 넘어갑니다. 일단 매핑을 *확인*하면(멈추고
`verify_intent`/`audit_intent` passed), **write it back**:
`verify_intent`/`audit_intent` 통과), **다시 작성**:
```tcl
ipcfg::cache_put <ip> "<feature phrase>" CONFIG.<NAME> <scalar|comma-list|nested> \
                 <enabler_param|null> <user|default> "<doc citation>" <ip_version>
```
This kills the run-to-run "found-then-lost" variance and cuts repeat doc searches.
이렇게 하면 실행 간 "발견 후 손실" 차이가 사라지고 반복적인 문서 검색이 중단됩니다.
The cache
캐시
stores **discovery facts only — never the prompt's expected value** (there is no value field;
**검색 사실만 저장합니다. 프롬프트의 예상 값은 저장하지 않습니다**(값 필드가 없습니다.
see `cache/README.md`), so it is safe to consult and populate during a blind run.
'cache/README.md' 참조), 따라서 블라인드 실행 중에 참조하고 채우는 것이 안전합니다.

Call `vivado_doc_search` with a targeted query.
타겟 쿼리로 'vivado_doc_search'를 호출하세요.
Combine part-family and parameter
부품 패밀리와 매개변수 결합
discovery into a single call when possible:
가능한 경우 단일 호출로 검색:

```
vivado_doc_search("<IP name> supported devices CONFIG parameters <key feature>")
```

From the results, identify:
결과에서 다음을 확인하십시오.
- Which device families support this IP (feeds Step 0b)
- 이 IP를 지원하는 장치 제품군(0b단계 피드)
- The correct CONFIG.* property names
- 올바른 CONFIG.* 속성 이름
- Valid values and ranges
- 유효한 값 및 범위
- Any noted dependencies between parameters
- 매개변수 간의 알려진 종속성

**Doc-grounded mapping (anti-synonym guard) — required for every requirement.**
**문서 기반 매핑(동의어 방지 보호) — 모든 요구 사항에 필요합니다.**
Pick a parameter because its *documented description* matches the feature's **semantics**,
*문서화된 설명*이 기능의 **의미**와 일치하므로 매개변수를 선택하세요.
NOT because the parameter **name** looks related.
매개변수 **name**이 관련되어 있는 것처럼 보이기 때문이 아닙니다.
Param names routinely share keywords
매개변수 이름은 정기적으로 키워드를 공유합니다.
while implementing different functions, so name pattern-matching silently picks the wrong
다른 기능을 구현하는 동안 이름 패턴 일치가 자동으로 잘못된 항목을 선택합니다.
one (it will set fine and stick, producing a false success that Tier 2 never sees).
하나(잘 설정되고 고정되어 Tier 2가 결코 볼 수 없는 잘못된 성공을 생성합니다).

- When several candidate params share a keyword from the prompt, read each candidate's
- 여러 후보 매개변수가 프롬프트에서 키워드를 공유하는 경우 각 후보의 매개변수를 읽습니다.
doc description and choose the one whose description states the requested function.
문서 설명을 선택하고 요청된 기능에 대한 설명이 있는 문서를 선택하세요.
- Record the deciding doc line in the ledger's "Doc evidence" column — if you cannot cite
- 인용할 수 없는 경우 원장의 "문서 증거" 열에 결정적인 문서 라인을 기록합니다.
why a param implements the requirement, you have not grounded it yet; search again.
매개변수가 요구사항을 구현하는 이유는 아직 근거가 없습니다. 다시 검색해 보세요.
- Illustrative (do not generalize the specific names): a phrase like "user-data-type
- 예시(특정 이름을 일반화하지 마세요): "user-data-type"과 같은 문구
filtering" can collide with distinct params for "embedded non-image data", a "video
필터링"은 "내장된 비이미지 데이터", "비디오"에 대한 고유 매개변수와 충돌할 수 있습니다.
format / ISP bridge", and an actual "user-data-type filter" — only one matches the
형식 / ISP 브리지" 및 실제 "사용자 데이터 유형 필터" - 하나만 일치합니다.
intent.
의도.
Disambiguate by description, not by which name contains "data" or "filter".
이름에 "데이터" 또는 "필터"가 포함되어 있는 것이 아니라 설명으로 구분하십시오.
- **Map to the COMPLETE matching set, not the first match.** A single requirement is often
- **첫 번째 일치가 아닌 전체 일치 집합에 매핑됩니다.** 단일 요구 사항은 종종
realized by *several* sibling params, and missing one silently under-configures the IP.
*여러* 형제 매개변수에 의해 실현되고 하나가 누락되면 자동으로 IP가 과소 구성됩니다.
Enumerate the full neighborhood for the feature keyword with the scratch-cell introspection
스크래치 셀 검사를 통해 기능 키워드에 대한 전체 이웃을 열거합니다.
helper, then doc-grant each candidate:
도우미, 그런 다음 각 후보자에게 문서를 부여합니다.
  ```tcl
  puts [ipcfg::discover_params <vlnv> <keyword>]   ;# DISCOVER:CONFIG.<...> CONFIG.<...> ...
  ```
Validated: `ipcfg::discover_params xilinx.com:ip:mipi_csi2_rx_subsystem lane` returns BOTH
검증됨: `ipcfg::discover_params xilinx.com:ip:mipi_csi2_rx_subsystem Lane`은 둘 다 반환합니다.
`CONFIG.CMN_NUM_LANES` **and** `CONFIG.C_DPHY_LANES` (plus active-lane siblings) — a "N
`CONFIG.CMN_NUM_LANES` **및** `CONFIG.C_DPHY_LANES`(활성 레인 형제 포함) - "N
lanes" requirement must set the ones the docs say apply, not just whichever name you hit
차선" 요구 사항은 어떤 이름을 누르는 것이 아니라 문서에 적용되는 항목을 설정해야 합니다.
first.
먼저.
Write confirmed discoveries to the learned cache (`ipcfg::cache_put`).
확인된 발견을 학습된 캐시(`ipcfg::cache_put`)에 씁니다.

**On Vivado 2026.1+ (VIVADO-23126), prefer native param discovery for FLAT-param IPs.**
**Vivado 2026.1+(VIVADO-23126)에서는 FLAT 매개변수 IP에 대한 기본 매개변수 검색을 선호합니다.**
Gate with `ipcfg::has_native dump_param_deps`.
`ipcfg::has_native dump_param_deps`로 게이트하세요.
When 1, replace the per-IP `vivado_doc_search`
1인 경우 IP별 `vivado_doc_search`를 교체합니다.
with one `ipcfg::param_deps <scratch_ipname>` (dump to file) + targeted `ipcfg::param_block`
하나의 `ipcfg::param_deps <scratch_ipname>`(파일로 덤프) + 대상 `ipcfg::param_block`
reads — this carries explicit Range/Enabled/Disabled per param and cut token footprint ~87%
읽기 - 이는 매개변수당 명시적인 범위/활성화/비활성화를 전달하고 토큰 공간을 ~87%로 줄입니다.
and MCP round-trips ~30% on the rave2 Versal subset. **Always** read targeted blocks, never
MCP 왕복은 rave2 Versal 하위 집합에서 ~30%입니다. **항상** 대상 블록을 읽습니다. 읽지 않습니다.
the whole dump.
전체 덤프.
For **PS/CIPS-class** IPs (`ps_wizard`, `versal_cips`) the actionable params
**PS/CIPS 클래스** IP(`ps_wizard`, `versal_cips`)의 경우 실행 가능한 매개변수
live in a nested dict (`PS11_CONFIG_INTERNAL`, `PS_PMC_CONFIG`); use `ipcfg::param_dict` to
중첩된 사전(`PS11_CONFIG_INTERNAL`, `PS_PMC_CONFIG`)에 살고 있습니다. `ipcfg::param_dict`를 사용하여
list sub-keys + current values.
하위 키 + 현재 값을 나열합니다.
`dump_param_deps` does NOT carry per-sub-key range/gating, so
`dump_param_deps`는 하위 키별 범위/게이트를 전달하지 않으므로
for the handful of sub-keys a prompt actually touches use the **set-and-observe** helpers
소수의 하위 키에 대해 프롬프트가 실제로 터치되는 경우 **설정 및 관찰** 도우미를 사용하세요.
instead of a doc search: `ipcfg::probe_subkey_range` (push an out-of-range sentinel — non-destructive,
문서 검색 대신: `ipcfg::probe_subkey_range`(범위를 벗어난 감시 장치 푸시 - 비파괴적,
the customizer auto-restores; run with `capture_log=true` and parse the legal set with
사용자 정의 프로그램이 자동 복원됩니다. `capture_log=true`로 실행하고 다음으로 합법적인 세트를 구문 분석합니다.
`ipcfg::range_from_log`, since the `is out of range { ... }` text only reaches the log)
`ipcfg::range_from_log`, `is out of range { ... }` 텍스트는 로그에만 도달하므로)
and `ipcfg::resolve_subkeys` (apply legal inputs, read back the resolved `*_INTERNAL` to see what
및 `ipcfg::resolve_subkeys`(적법한 입력을 적용하고 해결된 `*_INTERNAL`을 다시 읽어서
was derived/gated).
파생/게이트됨).
Both are version-INDEPENDENT (customizer feedback, not a 23126 command), so
둘 다 버전 독립적입니다(23126 명령이 아닌 사용자 정의 피드백).
they are the range/gating fallback on BOTH 2025.2 and 2026.1.
이는 2025.2 및 2026.1 모두의 범위/게이트 대체입니다.
When `has_native` is 0 (2025.2 and
`has_native`가 0인 경우(2025.2 및
mislabeled builds), keep using `discover_params` + doc search for discovery unchanged.
레이블이 잘못 지정된 빌드), `discover_params` + 검색을 위한 문서 검색을 변경 없이 계속 사용하세요.

**Backend resilience:** If `vivado_doc_search` returns a non-JSON / HTML error payload
**백엔드 복원력:** `vivado_doc_search`가 JSON/HTML이 아닌 오류 페이로드를 반환하는 경우
(e.g. an HTTP 404 page) rather than documentation, treat it as a *backend failure*, not as
(예: HTTP 404 페이지) 문서가 아닌 *백엔드 실패*로 처리합니다.
"no parameters found." Retry once against the alternate doc-search server if one is
"매개변수를 찾을 수 없습니다." 대체 문서 검색 서버가 있는 경우 해당 서버에 대해 한 번 재시도합니다.
available; if doc search stays unavailable, fall back to **error-feedback-only mode** —
사용 가능; 문서 검색을 계속 사용할 수 없으면 **오류 피드백 전용 모드**로 돌아갑니다. —
attempt a best-effort dict from the prompt and let Tier 2's Vivado error messages drive the
프롬프트에서 최선의 dict를 시도하고 Tier 2의 Vivado 오류 메시지가
correction.
수정.
Never grade a prompt down just because the doc backend was unreachable.
문서 백엔드에 접근할 수 없다는 이유만으로 프롬프트의 등급을 낮추지 마세요.

### Step 2.5: Source the helper library (ONCE per session)
### 2.5단계: 도우미 라이브러리 소싱(세션당 한 번)

All repetitive TCL (part swap, VLNV check, cell creation, dict apply with error
모든 반복 TCL(부품 교환, VLNV 검사, 셀 생성, 오류가 있는 dict 적용)
classification, stuck-value verification, value-format introspection, block automation,
분류, 정체된 값 검증, 값 형식 검사, 블록 자동화,
stub connections, cleanup) lives in a generic, **IP-agnostic** helper library shipped with
스텁 연결, 정리)는 다음과 함께 제공되는 일반적인 **IP 독립적** 도우미 라이브러리에 있습니다.
this skill: `lib/ipcfg.tcl`.
이 기술은 `lib/ipcfg.tcl`입니다.
Source it exactly once per Vivado session, after a block
블록 후 Vivado 세션당 정확히 한 번 소싱하세요.
design is open:
디자인이 공개되었습니다:

```
vivado_execute("source -notrace <skill_dir>/lib/ipcfg.tcl")
```

The helpers contain **no** IP names, VLNVs, or parameter names — every IP-specific value is
헬퍼에는 IP 이름, VLNV 또는 매개변수 이름이 **없음** 포함되어 있습니다. 모든 IP 관련 값은
passed in by you (discovered from doc search + Vivado feedback).
귀하가 전달한 것입니다(문서 검색 + Vivado 피드백에서 발견).
Mutation helpers generally
일반적으로 돌연변이 도우미
return status text; queries may return a dict, list, path or multiline display text.
상태 텍스트 반환; 쿼리는 사전, 목록, 경로 또는 여러 줄 표시 텍스트를 반환할 수 있습니다.
Use the [local API return contracts](../vivado-ipi-assembler/REFERENCE.md#local-tcl-api-inventory),
[로컬 API 반환 계약](../vivado-ipi-assembler/REFERENCE.md#local-tcl-api-inventory)을 사용하세요.
not a universal `SUCCESS:` parser.
보편적인 `SUCCESS:` 파서가 아닙니다.
Repeated source is guarded in the current library;
반복 소스는 현재 라이브러리에서 보호됩니다.
never hot-reload changed code into an older live session.
변경된 코드를 이전 라이브 세션으로 핫 리로드하지 마세요.
See
참조
[assembler lifecycle](../vivado-ipi-assembler/SKILL.md#setup-and-lifecycle) before resuming or
[어셈블러 라이프사이클](../vivado-ipi-assembler/SKILL.md#setup-and-lifecycle) 재개 전 또는
transitioning from scratch to delivery; `init` resets gates and is not a reload check.
처음부터 배송까지 전환; `init`은 게이트를 재설정하며 다시 로드 확인이 아닙니다.

| Helper<br>도우미 | Purpose<br>목적 |
|---|---|
| `ipcfg::ensure_part <part> ?force?`<br>`ipcfg::ensure_part <부분> ?force?` | **Guarded** swap (close/reopen bd). Returns `SWAP:old->new`, `NOSWAP:p`, or `WARN:SWAP_BLOCKED:...` when other cells exist; pass `force=1` only after user confirmation<br>**보호됨** 스왑(BD 닫기/다시 열기). 다른 셀이 존재하는 경우 `SWAP:old->new`, `NOSWAP:p` 또는 `WARN:SWAP_BLOCKED:...`를 반환합니다. 사용자 확인 후에만 `force=1`을 전달합니다. |
| `ipcfg::restore_part <orig>`<br>`ipcfg::restore_part <orig>` | Swap back to the original part<br>원래 부품으로 다시 교체 |
| `ipcfg::vlnv_ok <vlnv_prefix>`<br>`ipcfg::vlnv_ok <vlnv_prefix>` | 1/0 — is the version-free VLNV available for the current part<br>1/0 — 현재 부품에 사용 가능한 버전 없는 VLNV입니다. |
| `ipcfg::create_cell <vlnv> <cell>`<br>`ipcfg::create_cell <vlnv> <셀>` | Idempotent `create_bd_cell` with error classification. Checks the ipdef's `SUPPORTED_PARTS` before creating anything. An IP this part cannot instantiate returns `CONFIGURE_FAIL:WRONG_PART:<vlnv> is not supported on <part> ...; same-function IP this part supports: {..}` without spending an attempt; pick from that list (`create_cell_cfg` does the same)<br>오류 분류가 포함된 멱등성 'create_bd_cell'입니다. 아무것도 생성하기 전에 ipdef의 `SUPPORTED_PARTS`를 확인합니다. 이 부분이 인스턴스화할 수 없는 IP는 `CONFIGURE_FAIL:WRONG_PART:<vlnv>가 <part>에서 지원되지 않음을 반환합니다 ...; 동일한 기능 IP 이 부분은 다음을 지원합니다: {..}` 시도를 소비하지 않고; 해당 목록에서 선택하세요(`create_cell_cfg`도 동일합니다) |
| `ipcfg::create_cell_cfg <vlnv> <cell> <dict>`<br>`ipcfg::create_cell_cfg <vlnv> <셀> <dict>` | **Preferred:** create **and** configure in ONE call via `create_bd_cell -set_param` (skips init-to-default + separate apply); falls back to create+apply_dict if unsupported<br>**선호:** `create_bd_cell -set_param`을 통해 한 번의 호출로 **만들고** 구성합니다(기본값으로 초기화 건너뛰기 + 별도 적용). 지원되지 않는 경우 create+apply_dict로 대체됩니다. |
| `ipcfg::apply_dict <cell> <dict>`<br>`ipcfg::apply_dict <셀> <dict>` | `set_property -dict` with `PARAM_NOT_FOUND/VALUE_OUT_OF_RANGE/READ_ONLY/NOT_SUPPORTED/PARAM_DISABLED` classification (use to add/fix params after `create_cell_cfg`)<br>`PARAM_NOT_FOUND/VALUE_OUT_OF_RANGE/READ_ONLY/NOT_SUPPORTED/PARAM_DISABLED` 분류가 포함된 `set_property -dict`(`create_cell_cfg` 이후 매개변수를 추가/수정하는 데 사용) |
| `ipcfg::reconcile_disabled <cell> <dict>`<br>`ipcfg::reconcile_disabled <셀> <dict>` | For gated/disabled keys: returns `OMIT:{...}` (current value already equals intent — drop them) and `DIFFER:{...}` (real problem — only these need an enabler)<br>게이트/비활성화 키의 경우: `OMIT:{...}`(현재 값이 이미 의도와 동일하므로 삭제) 및 `DIFFER:{...}`(실제 문제 - 활성화 코드만 필요함)을 반환합니다. |
| `ipcfg::is_system_intent <text>`<br>`ipcfg::is_system_intent <텍스트>` | Heuristic: does the prompt describe a **subsystem** (integrated MC, PCIe controller, PS PL-clocks/peripherals)? Returns `SYSTEM:{reasons}` or `STANDALONE`. A `SYSTEM` verdict → try automation-first; if no rule + gated params, report integration-derived honestly<br>휴리스틱: 프롬프트가 **하위 시스템**(통합 MC, PCIe 컨트롤러, PS PL 클럭/주변 장치)을 설명합니까? `SYSTEM:{reasons}` 또는 `STANDALONE`을 반환합니다. 'SYSTEM' 판정 → 자동화 우선 시도; 규칙이 없고 게이트 매개변수가 있는 경우 통합 파생을 정직하게 보고합니다. |
| `ipcfg::rule_for_vlnv <dump> <vlnv>`<br>`ipcfg::rule_for_vlnv <덤프> <vlnv>` | Parse a captured `::bd::util_cmd rules dump` to get the Designer-Assistance rule for a cell's VLNV → `RULE:xilinx.com:bd_rule:<short> desc={...}` or `RULE:none` (version-free or full VLNV ok)<br>캡처된 `::bd::util_cmd 규칙 덤프`를 구문 분석하여 셀의 VLNV에 대한 Designer-Assistance 규칙을 가져옵니다 → `RULE:xilinx.com:bd_rule:<short> desc={...}` 또는 `RULE:none`(버전이 없거나 전체 VLNV 가능) |
| `ipcfg::run_automation <cell> <rule> ?config? ?baseline?`<br>`ipcfg::run_automation <셀> <규칙> ?config? ?기준?` | Apply a rule (id `xilinx.com:bd_rule:<short>` from the dump) and report what it changed: `AUTOMATION:OK changed={k old new class ...} added_cells={...} default_options={k v ..}`. Before applying it checks `<config>` against the rule's **live option schema** (below) without mutating the design: omitted keys are filled with the rule's GUI defaults and listed in `default_options`; `CONFIGURE_FAIL:UNKNOWN_AUTOMATION_KEY:<keys>` = keys the rule would silently ignore; `CONFIGURE_FAIL:ILLEGAL_AUTOMATION_VALUE:<k=v legal={..}>` = value not legal in the current board/design context. Both list the full schema, so one repair can be complete. If the value you need is not in the legal list, this rule cannot provide it here — take another route (e.g. configure the IP directly with board interfaces) instead of retrying. Also classifies `MISSING_AUTOMATION_KEY`, `NEEDS_BOARD_PRESET`, `PRESET_OVERRIDDEN`<br>규칙(덤프의 ID `xilinx.com:bd_rule:<short>`)을 적용하고 변경된 사항을 보고합니다: `AUTOMATION:OK 변경됨={k old new class ...} added_cells={...} default_options={k v ..}`. 적용하기 전에 디자인을 변경하지 않고 규칙의 **라이브 옵션 스키마**(아래)에 대해 `<config>`를 확인합니다. 생략된 키는 규칙의 GUI 기본값으로 채워지고 `default_options`에 나열됩니다. `CONFIGURE_FAIL:UNKNOWN_AUTOMATION_KEY:<키>` = 규칙이 자동으로 무시하는 키입니다. `CONFIGURE_FAIL:ILLEGAL_AUTOMATION_VALUE:<k=v legal={..}>` = 현재 보드/설계 컨텍스트에서 값이 유효하지 않습니다. 둘 다 전체 스키마를 나열하므로 한 번의 복구가 완료될 수 있습니다. 필요한 값이 적법한 목록에 없으면 이 규칙은 이를 여기에서 제공할 수 없습니다. 다시 시도하는 대신 다른 경로(예: 보드 인터페이스로 IP를 직접 구성)를 사용하십시오. 또한 `MISSING_AUTOMATION_KEY`, `NEEDS_BOARD_PRESET`, `PRESET_OVERRIDDEN`을 분류합니다. |
| `ipcfg::automation_rule_options <cell> <rule>`<br>`ipcfg::automation_rule_options <셀> <규칙>` | **Preferred schema source.** Calls the loaded rule's own `get_rule_options` (what its GUI shows; no design mutation) → `RULE_OPTIONS:keys={..} defaults={k v ..} legal={k {..} ..} schema=<k=default legal=.. [disabled]; ..>`. Values are case-sensitive (`None`, not `none`/`0`). Needs the BD open and the target cell created. Measured 2026.1: `axi_noc2` keys are `num_axi_tg num_aximm_ext pl2noc_psx pl2noc_apm num_axi_bram lpddr5_mc_type num_mc_lpddr5 ddr5_mc_type num_mc_ddr5 noc_clk` (no `mc_type`/`num_mc`)<br>**기본 스키마 소스.** 로드된 규칙 자체 `get_rule_options`(GUI가 표시하는 것, 디자인 변경 없음)를 호출합니다. → `RULE_OPTIONS:keys={..} defaults={k v ..} legal={k {..} ..}schema=<k=default legal=.. [disabled]; ..>`. 값은 대소문자를 구분합니다(`None`, `none`/`0` 아님). BD를 열고 타겟 셀을 생성해야 합니다. 2026.1 측정: `axi_noc2` 키는 `num_axi_tg num_aximm_ext pl2noc_psx pl2noc_apm num_axi_bram lpddr5_mc_type num_mc_lpddr5 ddr5_mc_type num_mc_ddr5 noc_clk`입니다(no `mc_type`/`num_mc`) |
| `ipcfg::automation_config_schema <rule_file>`<br>`ipcfg::automation_config_schema <규칙_파일>` | Offline fallback when the rule is not loaded: parse a rule's `.tcl` (+ sibling `utils.tcl`) for `-config` keys + defaults → `SCHEMA:keys={..} defaults={k v ..} file=<path>`. Can be stale or unreadable for newer releases; prefer `automation_rule_options`<br>규칙이 로드되지 않은 경우 오프라인 폴백: `-config` 키 + 기본값에 대한 규칙의 `.tcl`(+ 형제 `utils.tcl`)을 구문 분석합니다 → `SCHEMA:keys={..} defaults={k v ..} file=<path>`. 최신 릴리스에서는 오래되었거나 읽을 수 없을 수 있습니다. 'automation_rule_options'를 선호합니다. |
| `ipcfg::preset_drift <cell>`<br>`ipcfg::preset_drift <셀>` | Compare a board-bound cell with the registered board's preset → exactly one of `PRESET:OK n=<keys> presets=<procs>`, `PRESET_DRIFT:<n> <key>=<actual>(preset <value>) ...`, `PRESET_CHECK_ERROR:<why>`. Test `PRESET:OK*`; there is no `PRESET_DRIFT:OK`<br>보드에 바인딩된 셀을 등록된 보드의 사전 설정 → 정확히 `PRESET:OK n=<keys> Presets=<procs>`, `PRESET_DRIFT:<n> <key>=<actual>(preset <value>) ...`, `PRESET_CHECK_ERROR:<why>` 중 하나와 비교합니다. `PRESET:OK*`를 테스트하세요. `PRESET_DRIFT:OK`가 없습니다. |
| `ipcfg::pinned_reset <cell>`<br>`ipcfg::pinned_reset <셀>` | Pinned `ipcfg::require` values that no longer hold → `""` or `PINNED_RESET:<n> <key>=<cur>(want <v>) ... restore={k v ..}`. `board_bind` and `ipiasm::connect_board_intf` append it to their result when a preset overwrote pinned spec values; pass the whole `restore` dict to ONE `ipcfg::require`, then `audit_requirements`<br>더 이상 보유하지 않는 `ipcfg::require` 값 → `""` 또는 `PINNED_RESET:<n> <key>=<cur>(want <v>) ... 복원={k v ..}`를 고정했습니다. 사전 설정이 고정된 사양 값을 덮어썼을 때 `board_bind` 및 `ipiasm::connect_board_intf`를 결과에 추가합니다. 전체 `restore` dict를 하나의 `ipcfg::require`에 전달한 다음 `audit_requirements`를 전달합니다. |
| `ipcfg::source_reported <file>`<br>`ipcfg::source_reported <파일>` | `source` a stage script. First refuses a script that calls a command that does not exist in this session (`SCRIPT_CHECK_FAIL:<file> ... {names}`, nothing run); on failure prints `SCRIPT_FAIL:<file>: <message>` and `SCRIPT_TRACE:<file/line trace>` before re-raising. Use for every stage/probe script<br>스테이지 스크립트를 `소스`합니다. 먼저 이 세션에 존재하지 않는 명령(`SCRIPT_CHECK_FAIL:<file> ... {names}`, 아무것도 실행되지 않음)을 호출하는 스크립트를 거부합니다. 실패하면 다시 발생하기 전에 `SCRIPT_FAIL:<file>: <message>` 및 `SCRIPT_TRACE:<file/line Trace>`를 인쇄합니다. 모든 단계/프로브 스크립트에 사용 |
| `ipcfg::script_check <file>`<br>`ipcfg::script_check <파일>` | The same unknown-command check without running the script → `SCRIPT_CHECK:OK` or `SCRIPT_CHECK_FAIL`<br>스크립트를 실행하지 않고 동일한 알 수 없는 명령 확인 → `SCRIPT_CHECK:OK` 또는 `SCRIPT_CHECK_FAIL` |
| `ipcfg::help ?<word or glob>?`<br>`ipcfg::help ?<단어 또는 덩어리>?` | Call shape, purpose and `ok:`/`fail:` results of matching `ipcfg::`/`ipiasm::` helpers, generated from the loaded code<br>로드된 코드에서 생성된 일치하는 `ipcfg::`/`ipiasm::` 도우미의 호출 형태, 목적 및 `ok:`/`fail:` 결과 |
| `ipcfg::snapshot_all <cell>`<br>`ipcfg::snapshot_all <셀>` | Capture the **entire** `CONFIG.*` dict in one pass (baseline for `config_diff`/`run_automation`)<br>한 번의 패스로 **전체** `CONFIG.*` 딕셔너리 캡처(`config_diff`/`run_automation`의 기준선) |
| `ipcfg::config_diff <cell> <baseline> ?reqmap?`<br>`ipcfg::config_diff <셀> <기준> ?reqmap?` | Full-config diff after an apply. Requested keys → `EXACT/RESOLVED/REVERTED`; other moved keys → `CHANGED` (IP side effect). Returns `{key old new class ...}`<br>적용 후 전체 구성 차이점. 요청된 키 → `EXACT/RESOLVED/REVERTED`; 다른 이동된 키 → `CHANGED`(IP 부작용). `{key old new class ...}`를 반환합니다. |
| `ipcfg::classify_change <want> <base> <cur> ?validset? ?tolerance?`<br>`ipcfg::classify_change <원함> <base> <cur> ?validset? ?관용?` | One-key verdict: `EXACT` / `RESOLVED` (numeric and within the `tolerance` the requirement supplied: `<abs>` or `<n>%`; never without one) / `NEAREST` (a **miss**: the IP snapped to a nearby legal value, reported as a diagnostic) / `REVERTED` (snapped to default/elsewhere); `UNCHANGED`/`CHANGED` when `want=""`<br>단일 키 판정: `EXACT` / `RESOLVED`(숫자 및 `허용치` 내에서 요구 사항 제공: `<abs>` 또는 `<n>%`; 하나 없이는 절대 없음) / `NEAREST`(**miss**: IP가 가까운 합법적인 값으로 스냅되고 진단으로 보고됨) / `REVERTED`(기본값/다른 위치로 스냅됨); `want=""`인 경우 `변경되지 않음`/`변경됨` |
| `ipcfg::discover_params <vlnv> <keyword>`<br>`ipcfg::discover_params <vlnv> <키워드>` | Scoped introspection on a throwaway `create_ip` cell → `DISCOVER:CONFIG.<...> ...` (the **complete** param neighborhood matching a feature keyword). The one sanctioned `list_property` use; maps a requirement to the full sibling set (e.g. `CMN_NUM_LANES` **and** `C_DPHY_LANES`)<br>일회용 `create_ip` 셀에 대한 범위 검사 → `DISCOVER:CONFIG.<...> ...`(기능 키워드와 일치하는 **완전한** 매개변수 환경) 승인된 `list_property` 사용; 요구사항을 전체 형제 세트(예: `CMN_NUM_LANES` **및** `C_DPHY_LANES`)에 매핑합니다. |
| `ipcfg::native_caps` / `ipcfg::has_native <feature>`<br>`ipcfg::native_caps` / `ipcfg::has_native <기능>` | **Version gate (2026.1+).** Detects, once per session, whether the native VIVADO-23126 commands exist (by **command presence**, not version string — some `2026.1_*` builds report `2025.2.0`). `feature` = `dump_param_deps` \<br>**버전 게이트(2026.1+).** 기본 VIVADO-23126 명령이 존재하는지 여부를 세션당 한 번씩 감지합니다(버전 문자열이 아닌 **명령 존재**로 - 일부 `2026.1_*` 빌드 보고서 `2025.2.0`). `기능` = `dump_param_deps` \| `can_connect` → `0/1`. All native helpers below return `*_NA` when 0 so you fall back to the reactive path<br>`can_connect` → `0/1`. 아래의 모든 기본 도우미는 0일 때 `*_NA`를 반환하므로 반응 경로로 돌아갑니다. |
| `ipcfg::param_deps <ipname> ?file?`<br>`ipcfg::param_deps <ip 이름> ?파일?` | **2026.1+ fast param discovery.** `::debug::dump_param_deps` (resolved Range/Enabled/Disabled + dependency graph) written to a FILE → `PARAM_DEPS:<file>:<bytes>` or `PARAM_DEPS_NA`. For **flat-param IPs** this REPLACES the per-IP `vivado_doc_search` (measured ~50–72 KB) — confirmed ~87% fewer tokens on the rave2 subset. `ipname` must be a managed-IP instance (`create_ip` first); BD cell names are rejected<br>**2026.1+ 빠른 매개변수 검색.** FILE → `PARAM_DEPS:<file>:<bytes>` 또는 `PARAM_DEPS_NA`에 기록된 `::debug::dump_param_deps`(해결된 범위/활성화/비활성화 + 종속성 그래프). **평면 매개변수 IP**의 경우 이는 IP당 'vivado_doc_search'(~50~72KB 측정)를 대체합니다. — rave2 하위 집합에서 ~87% 더 적은 토큰을 확인했습니다. `ipname`은 관리형 IP 인스턴스여야 합니다(`create_ip` 먼저). BD 셀 이름이 거부됩니다. |
| `ipcfg::param_block <file> <param>`<br>`ipcfg::param_block <파일> <매개변수>` | **Targeted read** of ONE param's block from a `param_deps` dump (Range/Enabled/Disabled/Default). ALWAYS use this — never inline the whole dump (172 KB ps_wizard … 665 KB axi_noc); a blanket read is ~2× worse than 2025.2<br>`param_deps` 덤프에서 하나의 매개변수 블록에 대한 **대상 읽기**(범위/활성화/비활성화/기본값). 항상 이것을 사용하십시오. 전체 덤프를 인라인하지 마십시오(172KB ps_wizard … 665KB axi_noc). 포괄적인 읽기는 2025.2보다 ~2배 더 나쁩니다. |
| `ipcfg::param_dict <file> <dictparam> ?subkey?`<br>`ipcfg::param_dict <파일> <dictparam> ?하위 키?` | **Nested-dict introspection** for PS/CIPS-class IPs. `dump_param_deps` exposes the real sub-keys only inside one resolved dict value (`ps_wizard` `PS11_CONFIG_INTERNAL` = a ~1315-key flat Tcl dict). `subkey=""`→`PD_KEYS:<all top-level keys>`; else `PD:<k>=<value>`. Recovers sub-key **names + current/default values** with NO doc search. **Limit:** per-sub-key valid-range/gating is NOT in the dump → use the two set-and-observe helpers below<br>PS/CIPS 클래스 IP에 대한 **중첩 사전 검사**. `dump_param_deps`는 하나의 해결된 dict 값(`ps_wizard` `PS11_CONFIG_INTERNAL` = ~1315 키 플랫 Tcl dict) 내에서만 실제 하위 키를 노출합니다. `subkey=""`→`PD_KEYS:<모든 최상위 키>`; 그렇지 않으면 `PD:<k>=<값>`. 문서 검색 없이 하위 키 **이름 + 현재/기본값**을 복구합니다. **제한:** 하위 키당 유효 범위/게이팅이 덤프에 없습니다 → 아래 두 개의 설정 및 관찰 도우미를 사용하세요. |
| `ipcfg::probe_subkey_range <cell> <dictparam> <subkey> ?sentinel?`<br>`ipcfg::probe_subkey_range <셀> <dictparam> <하위 키> ?sentinel?` | **Per-sub-key RANGE via set-and-observe** (fills the gap `dump_param_deps` leaves). Pushes an out-of-range sentinel for ONE sub-key on a BD cell's dict user-param; the customizer rejects it and **auto-restores** (non-destructive). Returns `PSR_OUT_OF_RANGE:<k>` (validated input — the legal set is in the captured log) \<br>**설정 및 관찰을 통한 하위 키별 범위**(`dump_param_deps`가 남는 간격을 채웁니다.) BD 셀의 dict user-param에 있는 하나의 하위 키에 대해 범위를 벗어난 감시 장치를 푸시합니다. 맞춤설정 도구는 이를 거부하고 **자동 복원**(비파괴)합니다. `PSR_OUT_OF_RANGE:<k>` 반환(검증된 입력 - 합법적인 세트가 캡처된 로그에 있음) \| `PSR_NOTVALIDATED` (free-form input, no range) \<br>`PSR_NOTVALIDATED`(자유 형식 입력, 범위 없음) \| `PSR_NOKEY` (derived/ignored, not user-settable) \<br>`PSR_NOKEY`(파생/무시, 사용자 설정 불가) \| `PSR_ERR`. **Run with `capture_log=true`** — the legal-set text ("`is out of range { ... }`") only reaches the message log, NOT `$e`/`errorInfo`. **Version-independent** (2025.2 + 2026.1). Proven on 2026.1 `ps_wizard` (`PS_USE_PMCPL_CLK0 → { 0,1 }`)<br>'PSR_ERR'. **`capture_log=true`로 실행** — 합법적으로 설정된 텍스트("`is out of range { ... }`")는 `$e`/`errorInfo`가 아닌 메시지 로그에만 도달합니다. **버전 독립적**(2025.2 + 2026.1). 2026.1 `ps_wizard`에서 입증됨(`PS_USE_PMCPL_CLK0 → { 0,1 }`) |
| `ipcfg::range_from_log <console> <subkey>`<br>`ipcfg::range_from_log <콘솔> <하위 키>` | **Parser** for the `probe_subkey_range` companion: scan the `capture_log` payload for the sub-key's range line → `RANGE:<k>={ 0,1 }` \<br>`probe_subkey_range` 컴패니언에 대한 **파서**: 하위 키의 범위 라인에 대한 `capture_log` 페이로드를 스캔 → `RANGE:<k>={ 0,1 }` \| `RANGE_NONE:<k>`. Stateless (same pattern as `find_disabled`/`autofix_apply` taking the console)<br>`범위_없음:<k>`. 상태 비저장(콘솔을 사용하는 `find_disabled`/`autofix_apply`와 동일한 패턴) |
| `ipcfg::resolve_subkeys <cell> <userparam> <override> ?internalparam? ?wantkeys?`<br>`ipcfg::resolve_subkeys <셀> <사용자 매개변수> <재정의> ?internalparam? ?열쇠를 원해?` | **Gating/derivation via set-and-observe.** Applies a legal `{subkey value ...}` override, reads back the resolved `*_INTERNAL` dict → `RSK:<k v ...>` (or `RSK_KEYS:<n>`). Shows what an input derives/gates (proven: req 250 MHz → `ACT_FREQMHZ 249.997 DIVISOR0 4 SRCSEL NPLL BUF 1`). **Applies a real config change** (snapshot `CONFIG.<userparam>` first if rollback needed). Version-independent<br>**설정 및 관찰을 통한 게이팅/파생.** 유효한 `{하위 키 값 ...}` 재정의를 적용하고 해결된 `*_INTERNAL` dict → `RSK:<k v ...>`(또는 `RSK_KEYS:<n>`)를 다시 읽습니다. 입력이 무엇을 파생/게이트하는지 보여줍니다(검증됨: req 250 MHz → `ACT_FREQMHZ 249.997 DIVISOR0 4 SRCSEL NPLL BUF 1`). **실제 구성 변경 사항을 적용합니다**(롤백이 필요한 경우 'CONFIG.<userparam>' 스냅샷을 먼저 적용). 버전 독립적 |
| `ipcfg::coverage_report <ledger>`<br>`ipcfg::coverage_report <원장>` | Build the REQUIRED partial-coverage disclosure from the requirement ledger (`COVERAGE:FULL<br>요구 사항 원장(`COVERAGE:FULL)에서 필수 부분 적용 범위 공개를 작성합니다.|PARTIAL` + unapplied prompt parts)<br>PARTIAL` + 미적용 프롬프트 부분) |
| `ipcfg::snapshot <cell> <keys>`<br>`ipcfg::스냅샷 <셀> <키>` | Capture **baseline default** values BEFORE config (call right after create); returns `{key val ...}` for `verify_intent`<br>구성 전 **기준 기본값** 값을 캡처합니다(생성 직후 호출). `verify_intent`에 대해 `{key val ...}`을 반환합니다. |
| `ipcfg::verify_stuck <cell> <dict>`<br>`ipcfg::verify_stuck <셀> <dict>` | Returns the list of keys that did **not** take effect (missing **or** value-reverted; a `*_CONFIG` key is compared sub-key by sub-key only when you pass a key/value dict and the readback is one; scalar `*_CONFIG` params compare as scalars). Empty list = all stuck<br>적용되지 **않은** 키 목록을 반환합니다(누락 **또는** 값이 복원됨; `*_CONFIG` 키는 키/값 사전을 전달하고 다시 읽어들이는 경우에만 하위 키별로 하위 키를 비교합니다. 스칼라 `*_CONFIG` 매개변수는 스칼라로 비교합니다). 빈 목록 = 모두 붙어 있음 |
| `ipcfg::verify_intent <cell> <reqmap> ?flag_no_change? ?tolerances?`<br>`ipcfg::verify_intent <셀> <reqmap> ?flag_no_change? ?공차?` | Intent audit: given `{key want baseline ...}`, flags keys that **reverted**, landed on a **nearest-legal** value, or are **suspect-no-change** (value == default). Acceptance is exact; a difference passes only within a tolerance given for that key in `tolerances` (`{key <abs>\<br>인텐트 감사: `{키가 기준선을 원합니다 ...}`가 주어지면 **되돌린** 키, **가장 합법적인** 값에 도달한 키 또는 **의심스러운 변경 없음**(값 == 기본값)인 키에 플래그를 지정합니다. 수락은 정확합니다. 차이는 `tolerances`(`{key <abs>\)의 해당 키에 대해 지정된 허용 오차 내에서만 통과됩니다.|<n>% ...}`). Pass `flag_no_change=0` to return only hard misses once doc grounding confirms a default legitimately satisfies intent (`value_src=default`). Empty list = intent realized<br><n>% ...}`). 문서 기반이 기본값이 합법적으로 의도를 충족하는지 확인하면(`value_src=default`) 하드 미스만 반환하려면 `flag_no_change=0`을 전달합니다. 빈 목록 = 의도 실현 |
| `ipcfg::audit_intent <cell> <reqmap> ?tolerances?`<br>`ipcfg::audit_intent <셀> <reqmap> ?공차?` | Reports buckets: `EXACT:{...} RESOLVED:{k(=cur,want=w,tolerance=t) ...} BAD:{...}`; `RESOLVED` exists only for keys given a tolerance<br>보고서 버킷: `정확함:{...} 해결됨:{k(=cur,want=w,tolerance=t) ...} 불량:{...}`; '해결됨'은 허용 오차가 지정된 키에만 존재합니다. |
| `ipcfg::find_disabled <output>`<br>`ipcfg::find_disabled <출력>` | Scans raw execute output for non-fatal `disabled parameter`/`[BD 41-721]` warnings (gated params that `catch` returns 0 for). Returns `DISABLED:...` or `""`<br>치명적이지 않은 `disabled 매개변수`/`[BD 41-721]` 경고(`catch`가 0을 반환하는 게이트 매개변수)에 대한 원시 실행 출력을 검색합니다. `DISABLED:...` 또는 `""`를 반환합니다. |
| `ipcfg::param_format <cell> <key>`<br>`ipcfg::param_format <셀> <키>` | Reads a key back so you can learn its **shape** (comma list, nested `{{...}}`, etc.)<br>키를 다시 읽어 **모양**(쉼표 목록, 중첩된 `{{...}}` 등)을 학습할 수 있습니다. |
| `ipcfg::try_automation <cell> <rule> ?config?`<br>`ipcfg::try_automation <셀> <규칙> ?config?` | Generic `apply_bd_automation -rule <rule>` (Designer Assistance)<br>일반 `apply_bd_automation -rule <rule>`(디자이너 지원) |
| `ipcfg::add_stub <cell> <pin> <intf_vlnv> ?prop? ?val?`<br>`ipcfg::add_stub <셀> <핀> <intf_vlnv> ?prop? ?발?` | Boundary interface port for connection-derived params; mode auto-mirrors the pin<br>연결 파생 매개변수를 위한 경계 인터페이스 포트. 모드는 핀을 자동으로 미러링합니다. |
| `ipcfg::cleanup <cell> ?orig_part?`<br>`ipcfg::cleanup <셀> ?orig_part?` | Deletes `STUB_*` ports, the cell, and restores the part if given<br>'STUB_*' 포트와 셀을 삭제하고, 제공된 경우 해당 부분을 복원합니다. |
| `ipcfg::cache_get <ip> <feature>`<br>`ipcfg::cache_get <ip> <기능>` | **Consult first** (0 MCP calls): returns the earned discovery entry (param/shape/enabler/value_src/doc/version) as JSON, or `""` on miss<br>**먼저 문의**(0 MCP 호출): 획득한 검색 항목(param/shape/enabler/value_src/doc/version)을 JSON으로 반환하거나 누락 시 `""`를 반환합니다. |
| `ipcfg::cache_put <ip> <feature> <param> <shape> <enabler> <value_src> <doc> <ipver>`<br>`ipcfg::cache_put <ip> <feature> <param> <shape> <enabler> <value_src> <doc> <ipver>` | **Write back** a confirmed mapping. Stores discovery facts only — **never** the prompt's expected value (blind-safe)<br>확인된 매핑을 **다시 작성**하세요. 검색 사실만 저장 — 프롬프트의 예상 값 **절대로** 저장하지 않음(블라인드-세이프) |
| `ipcfg::cache_dump`<br>`ipcfg::캐시_덤프` | Print the whole learned cache (debug)<br>학습된 캐시 전체를 인쇄합니다(디버그). |

**Concise per-prompt template** (replaces the verbose phase-by-phase script — the phases
**간결한 프롬프트별 템플릿**(상세한 단계별 스크립트를 대체합니다.
below are still the mental model, but the helpers collapse each to one line):
아래는 여전히 멘탈 모델이지만 헬퍼는 각각 한 줄로 축소됩니다.

```tcl
set cell <cell_name>
set orig [get_property PART [current_project]]
# Phase 0: only if vlnv_ok fails on the current part; ensure_part is guarded (see Step 0)
if {![ipcfg::vlnv_ok <vendor:library:name>]} {
    puts [ipcfg::ensure_part <required_part>]   ;# may return WARN:SWAP_BLOCKED -> confirm in design mode
}
# Phase 1: VLNV check
if {![ipcfg::vlnv_ok <vendor:library:name>]} { puts "CONFIGURE_FAIL:VLNV_NOT_FOUND"; ipcfg::restore_part $orig; return }
# Phase 2+3 in ONE call: create + parameterize (snapshot defaults first for the intent audit)
set d [list CONFIG.PARAM_A {a} CONFIG.PARAM_B {b}]
puts "APPLY:[ipcfg::create_cell_cfg <vendor:library:name> $cell $d]"
set bad [ipcfg::verify_stuck $cell $d]
if {[llength $bad] != 0} {
    puts "STUCK_FAIL:$bad"
} else {
  puts "READBACK:OK $cell; verify required interfaces and feature behavior before SUCCESS"
}
# Finally, emit coverage from the requirement ledger (REQUIRED when not fully applied):
# ledger = {"<prompt phrase>" CONFIG.PARAM_A applied  "<prompt phrase>" CONFIG.PARAM_B unapplied:integration-derived ...}
puts [ipcfg::coverage_report $ledger]
```

Note: `create_cell_cfg` is the single-call fast path.
참고: `create_cell_cfg`는 단일 호출 빠른 경로입니다.
When you need ordered/multi-pass
주문/다중패스가 필요한 경우
configuration (enablers first — see Phase 3), create with the enabler dict, then use
구성(인에이블러 먼저 - 3단계 참조), 인에이블러 dict로 생성한 다음 사용
`ipcfg::apply_dict` for the dependent pass.
종속 패스의 경우 `ipcfg::apply_dict`입니다.
For a default-versus-configured intent audit, create a separate default scratch cell or
기본 대 구성된 의도 감사의 경우 별도의 기본 스크래치 셀을 생성하거나
capture the baseline before `apply_dict`, then call `verify_intent`/`audit_intent` with
`apply_dict` 이전의 기준선을 캡처한 다음 `verify_intent`/`audit_intent`를 호출하세요.
that real baseline.
그 진짜 기준선.
Never use a post-configuration snapshot as defaults.
구성 후 스냅샷을 기본값으로 사용하지 마십시오.

`verify_stuck` is strictly stronger than the old existence-only loop: it catches the IPI
`verify_stuck`은 기존 존재 전용 루프보다 엄격하게 강력합니다. IPI를 포착합니다.
**silent revert** case where a key exists but the value snapped back (e.g. a peripheral
**자동 되돌리기** 키가 존재하지만 값이 다시 스냅되는 경우(예: 주변 장치)
`ENABLE` that reverts to `0`), which a `get_property` existence check misses.
'0'으로 되돌아가는 'ENABLE'), 'get_property' 존재 확인이 누락되었습니다.
You still scan
아직도 스캔 중이신가요?
the raw output for warning patterns as a backstop, but `STUCK_FAIL:` is the primary signal.
경고 패턴에 대한 원시 출력은 백스톱으로 사용되지만 'STUCK_FAIL:'이 기본 신호입니다.

**Intent audit (the non-error failure mode) — required.** `verify_stuck` only proves the
**의도 감사(오류가 아닌 실패 모드) - 필수입니다.** `verify_stuck`은
params *you chose* persisted; it cannot tell whether you chose the *right* param for a named
*당신이 선택한* 매개변수는 지속됩니다; 이름이 지정된 매개변수에 *올바른* 매개변수를 선택했는지 여부는 알 수 없습니다.
feature.
특징.
The most dangerous benchmark failure is a **wrong-but-valid** param: it exists, it
가장 위험한 벤치마크 실패는 **잘못되었지만 유효한** 매개변수입니다.
accepts your value, it sticks — no error, no `STUCK_FAIL`, no warning — so error-driven
귀하의 가치를 받아들이면 그대로 유지됩니다. 오류도 없고 `STUCK_FAIL`도 없고 경고도 없습니다. 따라서 오류가 발생합니다.
Tier 2 never fires and the prompt silently scores as a false success.
Tier 2는 실행되지 않으며 프롬프트는 자동으로 거짓 성공으로 평가됩니다.
To catch this without
이것을 잡지 않고 잡으려면
any IP-specific knowledge, `snapshot` the baseline *before* configuring, then run
IP 관련 지식, 구성 *전* 기준선 '스냅샷'을 실행한 다음 실행합니다.
`verify_intent` *after*:
`verify_intent` *후*:

- A requirement that means **enable/select/turn-on** must produce an **observable change
- **활성화/선택/켜기**를 의미하는 요구 사항은 **관찰 가능한 변경을 생성해야 합니다.
from the default**.
기본값에서 **.
If the param you set was *already* at the requested value by default
설정한 매개변수가 기본적으로 요청된 값에 *이미* 있는 경우
(`suspect-no-change`), you almost certainly set the wrong param — the param that truly
(`suspect-no-change`), 잘못된 매개변수를 설정한 것이 거의 확실합니다.
implements the feature is still sitting at its default.
기능을 구현하면 여전히 기본값으로 유지됩니다.
- A non-empty `verify_intent` list is a **new Tier-2 trigger** (`INTENT_FAIL`) even though
- 비어 있지 않은 `verify_intent` 목록은 **새로운 Tier-2 트리거**(`INTENT_FAIL`)입니다.
nothing errored: go back to Step 2, re-read the doc descriptions for the suspect
오류 없음: 2단계로 돌아가서 피의자에 대한 문서 설명을 다시 읽으세요.
requirement, remap to the correctly-grounded param, and re-apply.
요구 사항을 충족하려면 올바르게 기반된 매개변수에 다시 매핑하고 다시 적용하세요.
- Cross-check against the Step 1.5 ledger: every requirement must be observably realized,
- 1.5단계 원장을 대조하여 확인합니다. 모든 요구사항은 관찰 가능하게 실현되어야 합니다.
not merely "set without error."
단순히 "오류 없이 설정"하는 것이 아닙니다.

The remainder of Step 3 documents the phases the helpers implement, plus the
3단계의 나머지 부분에서는 도우미가 구현하는 단계와
error/format/automation/stub recovery patterns.
오류/형식/자동화/스텁 복구 패턴.
Use the helpers; drop to raw TCL only for an
도우미를 사용하십시오. 에 대해서만 원시 TCL로 드롭
IP-specific quirk a helper does not cover.
도우미가 다루지 않는 IP 관련 문제입니다.

### Step 3: Build and execute the -dict
### 3단계: -dict 빌드 및 실행

Compose a SINGLE TCL script with four phases: part swap (if needed), VLNV validation,
부품 교체(필요한 경우), VLNV 검증,
cell creation, and configuration.
셀 생성 및 구성.
Use `catch` around every mutating command so errors
모든 변경 명령 주위에 `catch`를 사용하여 오류가 발생하도록 하세요.
produce parseable output instead of aborting the script.
스크립트를 중단하는 대신 구문 분석 가능한 출력을 생성합니다.

```tcl
# /tmp/configure_<ip_name>.tcl

# --- Phase 0: Part swap (only if IP needs a different device family) ---
# AI: set _need_swap to 1 and _target_part to the required part when
# the doc search or fallback map indicates a part change is needed.
# Omit this entire block when no swap is needed.
set _orig_part [get_property PART [current_project]]
set _target_part "<required_part>"
set _need_swap [expr {$_orig_part ne $_target_part}]
if {$_need_swap} {
    close_bd_design [current_bd_design]
    set_property PART $_target_part [current_project]
    open_bd_design [get_files benchmark_bd.bd]
    puts "PART_SWAP:$_orig_part->$_target_part"
}

# --- Phase 1: VLNV pre-validation ---
set _vlnv_matches [get_ipdefs -filter {VLNV =~ "<vendor:library:name>:*"}]
if {$_vlnv_matches eq ""} {
    if {$_need_swap} {
        close_bd_design [current_bd_design]
        set_property PART $_orig_part [current_project]
        open_bd_design [get_files benchmark_bd.bd]
    }
    puts "CONFIGURE_FAIL:VLNV_NOT_FOUND:<vendor:library:name> not available for $_target_part"
    return
}

# --- Phase 2: Create cell ---
if {[llength [get_bd_cells <cell_name>]] == 0} {
    if {[catch {create_bd_cell -type ip -vlnv <vendor:library:name> <cell_name>} _err]} {
        if {$_need_swap} {
            close_bd_design [current_bd_design]
            set_property PART $_orig_part [current_project]
            open_bd_design [get_files benchmark_bd.bd]
        }
        puts "CONFIGURE_FAIL:CREATE_ERROR:$_err"
        return
    }
}

# --- Phase 3: Apply configuration with structured error handling ---
set _prop_dict [list \
    CONFIG.PARAM_A {value_a} \
    CONFIG.PARAM_B {value_b} \
    CONFIG.PARAM_C {value_c} \
]
# Empty-dict guard: set_property -dict {} throws "Missing name/value pair".
# For "default settings" prompts where intent == defaults, skip configuration.
if {[llength $_prop_dict] == 0} {
    puts "SUCCESS: <cell_name> created with default configuration"
    return
}
if {[catch {set_property -dict $_prop_dict [get_bd_cells <cell_name>]} _cfg_err]} {
    set _err_type "UNKNOWN"
    if {[string match {*does not exist*} $_cfg_err]}      {set _err_type "PARAM_NOT_FOUND"}
    if {[string match {*is out of the range*} $_cfg_err]}  {set _err_type "VALUE_OUT_OF_RANGE"}
    if {[string match {*read-only*} $_cfg_err]}            {set _err_type "READ_ONLY"}
    if {[string match {*It is read-only*} $_cfg_err]}      {set _err_type "READ_ONLY"}
    if {[string match {*not supported*} $_cfg_err]}        {set _err_type "NOT_SUPPORTED"}
    puts "CONFIGURE_FAIL:${_err_type}:$_cfg_err"
    return
}

# --- Phase 3 verification: catch SILENTLY-IGNORED params (IPI CRITICAL WARNINGs) ---
# In BD mode an unknown CONFIG key emits a non-fatal CRITICAL WARNING and catch returns 0,
# so the dict above can "succeed" without applying. Verify every key actually exists on the
# cell. get_property on a specific, already-set key is verification (not parameter discovery),
# so it does not violate the "no list_property/report_property" rule.
set _missing {}
foreach {_k _v} $_prop_dict {
    if {[catch {get_property $_k [get_bd_cells <cell_name>]}]} {
        lappend _missing $_k
    }
}
if {[llength $_missing] > 0} {
    puts "CONFIGURE_FAIL:PARAM_NOT_FOUND:silently ignored (do not exist): $_missing"
    return
}

puts "SUCCESS: <cell_name> configured"
```

**Output-scan backstop:** Even with the verification loop above, after `vivado_execute`
**출력 스캔 백스톱:** 위의 확인 루프를 사용하더라도 'vivado_execute' 이후
returns, scan its full stdout for `Cannot set the parameter`, `does not exist`,
반환되면 '매개변수를 설정할 수 없습니다', '존재하지 않습니다'에 대한 전체 표준 출력을 검색합니다.
`Invalid parameter`, or `Ignoring`.
`잘못된 매개변수` 또는 `무시`.
If any appear, treat the result as
나타나는 경우 결과를 다음과 같이 처리하십시오.
`CONFIGURE_FAIL:PARAM_NOT_FOUND` regardless of whether a `SUCCESS:` line was printed — the
`SUCCESS:` 줄이 인쇄되었는지 여부에 관계없이 `CONFIGURE_FAIL:PARAM_NOT_FOUND` —
warning may reference a parameter the loop could not catch (e.g. a key inside a nested dict).
경고는 루프가 포착할 수 없는 매개변수(예: 중첩된 사전 내부의 키)를 참조할 수 있습니다.

**Nested-dict params:** When a CONFIG value is itself a dictionary (e.g.
**Nested-dict 매개변수:** CONFIG 값 자체가 사전인 경우(예:
`CONFIG.CPM_CONFIG {...}`, `CONFIG.PS_PMC_CONFIG {...}`, or a transceiver-wizard config),
`CONFIG.CPM_CONFIG {...}`, `CONFIG.PS_PMC_CONFIG {...}` 또는 트랜시버 마법사 구성),
invalid *sub-keys* are silently dropped with `[IP_Flow 19-7090] ...
잘못된 *하위 키*는 `[IP_Flow 19-7090] ...을 사용하여 자동으로 삭제됩니다.
Ignoring`.
무시`.
The
는
top-level `get_property` check above will pass even though sub-keys were ignored.
하위 키가 무시되더라도 위의 최상위 `get_property` 검사는 통과됩니다.
`ipcfg::verify_stuck` already inspects `*_CONFIG` keys sub-key by sub-key, so a non-empty
`ipcfg::verify_stuck`은 이미 `*_CONFIG` 키를 하위 키별로 검사하므로 비어 있지 않은
`STUCK_FAIL` list names the exact sub-keys that did not apply.
`STUCK_FAIL` 목록은 적용되지 않은 정확한 하위 키의 이름을 지정합니다.

**STUCK recovery — value-format introspection (generic, no per-IP table):** The most common
**STUCK 복구 — 값 형식 검사(일반, IP별 테이블 없음):** 가장 일반적인
reason a key reverts (rather than erroring) is that your **value shape** does not match what
오류가 아닌 키가 되돌아가는 이유는 **값 형태**가 무엇과 일치하지 않기 때문입니다.
the IP expects.
IP가 기대합니다.
This is fully recoverable without any IP-specific knowledge:
이는 IP 관련 지식 없이도 완전히 복구 가능합니다.

1. Read the current value back with `ipcfg::param_format <cell> <key>` to learn the shape.
1. `ipcfg::param_format <cell> <key>`로 현재 값을 다시 읽어 모양을 알아보세요.
2. **Mirror that shape**, substituting your target value:
2. **그 모양을 미러링**하여 목표 값을 대체합니다.
   - **Comma-list params** (e.g.
   - **쉼표 목록 매개변수**(예:
   - Versal `clk_wizard`): `CLKOUT_USED` reads back as
   - Versal `clk_wizard`): `CLKOUT_USED`는 다음과 같이 다시 읽습니다.
`true,false,false,...` and `CLKOUT_REQUESTED_OUT_FREQUENCY` as a comma list — supply one
`true,false,false,...` 및 `CLKOUT_REQUESTED_OUT_FREQUENCY`를 쉼표 목록으로 사용 — 하나 제공
comma-separated string per the read-back length, **not** per-clock keys.
시계별 키가 **아님** 다시 읽기 길이별로 쉼표로 구분된 문자열입니다.
   - **Nested list-of-pairs sub-keys** (e.g.
   - **중첩된 쌍 목록 하위 키**(예:
   - Versal `versal_cips` peripherals): a peripheral
   - Versal `versal_cips` 주변 장치): 주변 장치
sub-key reads back as `{{ENABLE 0} {IO {PS_MIO 0 ..
하위 키는 `{{ENABLE 0} {IO {PS_MIO 0 ..
1}}}`.
To enable it, re-send the
활성화하려면
**same structure** with `ENABLE 1` (keep the read-back `IO {...}` routing).
**동일한 구조**와 `ENABLE 1`(다시 읽기 `IO {...}` 라우팅 유지).
Sending a
보내는 중
flat `{ENABLE 1}` is silently rejected.
플랫 `{ENABLE 1}`은 자동으로 거부됩니다.
   - **Scalar mismatch**: just supply the documented value/units the read-back implies.
   - **스칼라 불일치**: 다시 읽기에서 암시하는 문서화된 값/단위를 제공하면 됩니다.
3. Re-apply and re-run `verify_stuck`.
3. 'verify_stuck'을 다시 적용하고 다시 실행하세요.
3. Only after a format-mirrored retry still fails do you
3. 포맷 미러링 재시도가 여전히 실패한 후에야
grade `creation_only`/`partial`.
등급 `creation_only`/`partial`.

This single read-back-then-mirror loop replaces any per-IP value-format table — it works for
이 단일 읽기 후 미러 루프는 모든 IP별 값 형식 테이블을 대체합니다.
any param whose value snapped back, because the IP itself tells you the shape it wants.
IP 자체가 원하는 모양을 알려주기 때문에 값이 다시 스냅된 모든 매개변수입니다.

**Exact clock outputs — search, do not hand-tune:** When a spec requires exact (integer-Hz,
**정확한 클럭 출력 - 검색, 수동 조정하지 않음:** 사양에서 정확한(정수-Hz,
zero or small tolerance) outputs from a vector-`CLKOUT_*` clock wizard, call the bounded search
0 또는 작은 허용오차) 벡터-`CLKOUT_*` 시계 마법사의 출력, 경계 검색 호출
instead of editing requested rates by hand:
요청된 요율을 직접 편집하는 대신:

```tcl
ipcfg::search_clock_config <wizard_vlnv> <source_port_or_pin> {
    {id <name> hz <exact_hz> nominal_hz <nominal_hz> ?tolerance_hz <n>?} ...
} <absolute_evidence_dir> -max_generators <N allowed by the design> ?-min_attempts 10? ?-max_attempts 24?
```

It tries at least `-min_attempts` distinct configurations (every one if fewer exist) before
이전에 적어도 `-min_attempts` 고유 구성(더 적은 경우 모두)을 시도합니다.
returning `CLOCK_SEARCH:FAIL`, varying: `PRIM_IN_FREQ` propagated vs explicit, grouping of targets
`CLOCK_SEARCH:FAIL` 반환, 다양함: `PRIM_IN_FREQ` 전파 및 명시적, 대상 그룹화
across up to `-max_generators` wizards, MBUFGCE divided taps for exact ÷2/÷4/÷8 relations,
최대 `-max_generators` 마법사에 걸쳐 MBUFGCE는 정확한 ¼2/¶4/¶8 관계를 위해 탭을 나눕니다.
nominal/exact/kHz requested-rate encoding, output order, and 1×/2× source companions.
공칭/정확/kHz 요청 속도 인코딩, 출력 순서 및 1×/2× 소스 동반.
By default
기본적으로
(`-isolate 1`) every attempt runs in a temporary block design fed by a port at the live source
(`-isolate 1`) 모든 시도는 라이브 소스의 포트에서 제공하는 임시 블록 디자인에서 실행됩니다.
rate, so unrelated unconnected or undriven cells in your design cannot fail every candidate; the
따라서 설계에서 관련이 없거나 연결되지 않거나 구동되지 않는 셀이 모든 후보를 실패할 수는 없습니다. 는
temporary design is removed and only the winning configuration is placed in your design, wired
임시 디자인은 제거되고 가장 효과적인 구성만 디자인에 배치되고 연결됩니다.
to the real source.
실제 소스에.
Its rates there are proven only after your design validates and propagates:
그 비율은 귀하의 설계가 검증되고 전파된 후에만 입증됩니다.
re-read the placed outputs then.
그런 다음 배치된 출력을 다시 읽으십시오.
Each attempt uses fresh cells, validates, and reads back
시도할 때마다 새로운 셀을 사용하고 유효성을 검사한 후 다시 읽습니다.
propagated `FREQ_HZ`; failed cells are deleted (`::ipcfg::last_clock_search` has the evidence).
'FREQ_HZ' 전파; 실패한 셀은 삭제됩니다(`::ipcfg::last_clock_search`에 증거가 있음).

**Reading the placed clocks back — only after a clean validate.** IP integrator propagates
**배치된 시계를 다시 읽기 - 깨끗한 검증 후에만 가능.** IP 통합자가 전파됨
parameters only while `validate_bd_design` runs (UG994), and a validate that reports errors does
'validate_bd_design'이 실행되는 동안에만 매개변수를 사용하며(UG994) 오류를 보고하는 유효성 검사는
not settle the placed wizards: their input clock keeps `FREQ_HZ.VALUE_SRC=DEFAULT` (100 MHz) and
배치된 마법사를 확정하지 않음: 해당 입력 클럭은 `FREQ_HZ.VALUE_SRC=DEFAULT`(100MHz)를 유지하고
every output reads a rate derived from it — a false mismatch, not a bad configuration.
모든 출력은 여기서 파생된 비율을 읽습니다. 이는 잘못된 구성이 아니라 잘못된 불일치입니다.
Measured
측정됨
2026.1: outputs off by ~1.5 kHz after a validate that failed on unrelated undriven clocks, exact
2026.1: 관련되지 않은 구동되지 않은 클럭에서 실패한 검증 후 ~1.5kHz까지 출력이 꺼짐, 정확함
after a clean one.
깨끗한 후에.
So: connect the placed outputs to their consumers (in a scratch probe, drive
따라서 배치된 출력을 소비자에게 연결합니다(스크래치 프로브에서 드라이브
every remaining clock input so the design can validate), run `validate_bd_design`, then
디자인이 검증될 수 있도록 나머지 모든 클럭 입력) `validate_bd_design`을 실행한 다음
`ipcfg::clock_placement_status` → `CLOCK_PLACEMENT:OK n=<k> {..}` |
`ipcfg::clock_placement_status` → `CLOCK_PLACEMENT:OK n=<k> {..}` |
`CLOCK_PLACEMENT:PENDING inputs={..} ...
`CLOCK_PLACEMENT:PENDING 입력={..} ...
Blocking: <undriven clocks>` (not yet propagated: fix the
차단: <구동되지 않은 시계>`(아직 전파되지 않음: 수정
blockers, never re-search) | `CLOCK_PLACEMENT:FAIL <n> {id=<actual>(want <hz>) ..}` (propagated and
차단제, 다시 검색하지 않음) | `CLOCK_PLACEMENT:FAIL <n> {id=<실제>(<hz> 필요) ..}`(전파 및
wrong: a real failure) | `CLOCK_PLACEMENT:ERROR:<why>`.
틀렸다: 진짜 실패) | `CLOCK_PLACEMENT:ERROR:<이유>`.
Do not rerun the search with `-isolate 0`
'-isolate 0'을 사용하여 검색을 다시 실행하지 마세요.
to work around a PENDING readback: the caller design's unrelated errors then fail every candidate.
PENDING 읽기를 해결하려면 호출자 디자인의 관련 없는 오류가 모든 후보에 실패합니다.
Measured on 2026.1 `clkx5_wiz` with a non-round source rate: writing `PRIM_IN_FREQ` explicitly
2026.1 'clkx5_wiz'에서 반올림되지 않은 소스 속도로 측정됨: 'PRIM_IN_FREQ'를 명시적으로 작성
leaves outputs a few Hz off, while propagation lets the wizard hit exact rates.
출력을 몇Hz 정도 끄는 반면 전파를 통해 마법사는 정확한 속도에 도달할 수 있습니다.
The search never
검색은 절대 안됨
widens a tolerance or changes a target; a `FAIL` under the stated generator limit is evidence
허용 범위를 넓히거나 목표를 변경합니다. 명시된 발전기 한도 미만의 '실패'는 증거입니다.
to report, not permission to raise the limit unless the spec allows it.
보고할 수는 있지만 사양에서 허용하지 않는 한 한도를 높일 수 있는 권한은 없습니다.

**Board-preset automations must not override the preset:** when `ipcfg::run_automation`
**보드 사전 설정 자동화는 사전 설정을 재정의해서는 안 됩니다.** `ipcfg::run_automation`인 경우
is called with `board_preset Yes`, it compares the cell against the registered board's
`board_preset Yes`로 호출되면 셀을 등록된 보드의 셀과 비교합니다.
`preset.xml` (`ipcfg::preset_drift <cell>`, including hierarchical sub-keys such as
`preset.xml`(`ipcfg::preset_drift <cell>`, 다음과 같은 계층적 하위 키 포함)
`CONFIG.PS11_CONFIG(PS_NUM_FABRIC_RESETS)`) and returns `CONFIGURE_FAIL:PRESET_OVERRIDDEN` if any
`CONFIG.PS11_CONFIG(PS_NUM_FABRIC_RESETS)`) 및 `CONFIGURE_FAIL:PRESET_OVERRIDDEN`이 있는 경우 반환
preset value changed.
사전 설정 값이 변경되었습니다.
Other rule options can shadow preset values: on 2026.1 `ps_wizard`,
다른 규칙 옵션은 사전 설정 값을 섀도잉할 수 있습니다. 2026.1 `ps_wizard`에서는
`pl_resets`/`pl_clocks` overwrite the preset's fabric reset/clock settings.
`pl_resets`/`pl_clocks`는 프리셋의 패브릭 재설정/시계 설정을 덮어씁니다.
Those option keys are
해당 옵션 키는
required by the rule, so pass the preset's own values (read them from the drift report) rather
규칙에서 요구하므로 사전 설정의 자체 값을 전달하는 대신(드리프트 보고서에서 읽어옴)
than dropping the keys or hand-writing the parameter afterwards.
키를 삭제하거나 나중에 매개변수를 직접 쓰는 것보다.

**Apply a board-preset automation once per cell.** On 2026.1 `ps_wizard`, re-running the board
**셀당 한 번씩 보드 사전 설정 자동화를 적용합니다.** 2026.1 `ps_wizard`에서 보드를 다시 실행합니다.
rule on a PS whose preset is already applied changes no parameter but deletes every PS address
사전 설정이 이미 적용된 PS의 규칙은 매개변수를 변경하지 않지만 모든 PS 주소를 삭제합니다.
space (216 -> 0 on a Versal PS); the loss survives save/reopen, and validation, re-setting the
공간(Versal PS에서는 216 -> 0); 손실은 저장/다시 열기, 유효성 검사, 재설정 후에도 유지됩니다.
PS configuration or recreating the PS do not restore it.
PS를 구성하거나 PS를 다시 생성해도 PS가 복원되지 않습니다.
`run_automation` therefore returns
따라서 `run_automation`은 다음을 반환합니다.
`SKIP:board automation already applied` for a preset-OK cell that has address spaces, and
주소 공간이 있는 사전 설정 OK 셀에 대해 `SKIP:보드 자동화가 이미 적용되었습니다`, 그리고
`CONFIGURE_FAIL:ADDRESS_SPACES_LOST` if any automation removes spaces.
자동화가 공백을 제거하는 경우 `CONFIGURE_FAIL:ADDRESS_SPACES_LOST`
Wire later NoC or PL
나중에 NoC 또는 PL 연결
interfaces with the connect helpers, not another automation pass.
다른 자동화 패스가 아닌 연결 도우미와 인터페이스합니다.

**Exact by default; tolerance only from the requirement:** Some params are *requests* the IP
**기본적으로 정확합니다. 요구 사항에서만 허용되는 허용 범위:** 일부 매개변수는 IP *요청*입니다.
quantizes to the nearest achievable value (PLL/MMCM frequencies, divider ratios, lane counts
달성 가능한 가장 가까운 값으로 양자화합니다(PLL/MMCM 주파수, 분배기 비율, 레인 수
rounded to a legal set).
유효한 세트로 반올림됨)
A read-back of 720 for a required 719.98, or 4095 for a required
필수 719.98의 경우 720, 필수의 경우 4095 다시 읽기
4096, is a **miss** (`NEAREST`), not a pass.
4096은 통과가 아닌 **실패**(`NEAREST`)입니다.
Accept a difference only when the DesignSpec
DesignSpec이 지정된 경우에만 차이를 허용합니다.
states a tolerance for that value, and pass it explicitly: `ipcfg::require $cell $d $reason
해당 값에 대한 허용 오차를 명시하고 이를 명시적으로 전달합니다: `ipcfg::require $cell $d $reason
{CONFIG.<KEY> 0.1%}` (or `<abs>`); `audit_requirements` then reports `RESOLVED` within it.
{CONFIG.<KEY> 0.1%}`(또는 `<abs>`); 'audit_requirements'는 그 안에 'RESOLVED'를 보고합니다.
On `NEAREST`, the achieved value tells you what the IP can do: change the inputs that
'NEAREST'에서 달성된 값은 IP가 무엇을 할 수 있는지 알려줍니다. 즉, 입력을 변경하는 것입니다.
quantize it (e.g. the clock search) or escalate.
이를 양자화(예: 시계 검색)하거나 에스컬레이션합니다.
Do not widen a tolerance to make it pass.
통과하기 위해 공차를 넓히지 마십시오.
Record an accepted in-tolerance value as `applied-resolved:<achieved>` so request vs achieved
허용된 허용 오차 값을 'applied-resolved:<achieved>'로 기록하여 요청과 달성 여부를 확인합니다.
is disclosed.
공개됩니다.
Use a full-config baseline + diff to see
전체 구성 기준 + 차이점을 사용하여 확인하세요.
both the resolution and any IP side effects in one shot:
해상도와 IP 부작용을 모두 한 번에 해결:
```tcl
set base [ipcfg::snapshot_all $cell]
ipcfg::apply_dict $cell $d
set diff [ipcfg::config_diff $cell $base $d]
set evidence_file [file join $evidence_dir "diff.[pid].[clock clicks].tcldict"]
puts [ipcfg::write_evidence $evidence_file $diff]
```

Inspect requested changes and unexpected side effects from `diff`; do not print
`diff`에서 요청된 변경 사항과 예상치 못한 부작용을 검사합니다. 인쇄하지 마세요
the full value dictionaries.
전체 값 사전.
The recording receipt is not an intent verdict.
녹음 영수증은 의도적인 판결이 아닙니다.

**Automation-first for SYSTEM intent (do this BEFORE fighting gated CONFIG):**
**시스템 의도에 대한 자동화 우선(Gated CONFIG와 싸우기 전에 이 작업을 수행하십시오):**
Run `ipcfg::is_system_intent <prompt>` first.
먼저 `ipcfg::is_system_intent <프롬프트>`를 실행하세요.
If it returns `SYSTEM:{...}` (integrated
`SYSTEM:{...}`(통합)을 반환하는 경우
memory controllers, a PCIe controller, PS PL-clocks/peripherals), the prompt describes a
메모리 컨트롤러, PCIe 컨트롤러, PS PL 클럭/주변 장치), 프롬프트에서는 다음을 설명합니다.
*subsystem* that Vivado realizes through **block automation** / a **Configurable Example
Vivado가 **블록 자동화**를 통해 구현하는 *하위 시스템* / **구성 가능한 예시
Design**, not through plain standalone `CONFIG.*`.
디자인**, 단순한 독립 실행형 'CONFIG.*'를 통하지 않습니다.
Fighting the gated CONFIG there wastes
게이트된 CONFIG와 싸우는 것은 낭비입니다.
calls (validated: `NUM_MC`/`NUM_MCP` on a bare `axi_noc2` cell are **disabled** and every
호출(검증됨: 베어 `axi_noc2` 셀의 `NUM_MC`/`NUM_MCP`는 **비활성화**되고 모든
`set_property` is silently ignored with `[IP_Flow 19-3374] ... disabled parameter ...
`[IP_Flow 19-3374] ... 비활성화된 매개변수 ...에서는 `set_property`가 자동으로 무시됩니다.
ignored`).
무시`).
Do **not** hardcode which IPs need it.
필요한 IP를 하드코딩하지 **하지 마세요**.
Instead:
대신:

1. **Enumerate the loaded rules directly from Vivado** (the authoritative source — `get_bd_automation_rules` does *not* exist, but the internal rule manager does dump the registry).
1. **Vivado에서 직접 로드된 규칙을 열거합니다**(권한 있는 소스인 `get_bd_automation_rules`는 존재하지 *않지만* 내부 규칙 관리자가 레지스트리를 덤프합니다).
1. Run, in one execute, the rule dump and read the printed `[DBG] RulesMap:` block:
1. 한 번에 규칙 덤프를 실행하고 인쇄된 `[DBG] RulesMap:` 블록을 읽습니다.
   ```tcl
   ::bd::util_cmd rules dump
   # prints: "<vlnv>":["<description>", "<rule-short-name>", "<rules.tcl path>"]
   ```
The dump is a C-level message (it does **not** return a value and cannot be captured
덤프는 C 수준 메시지입니다(값을 반환하지 **않음** 및 캡처할 수 없음).
in-proc), so read it from the execute **output**.
in-proc) 실행 **출력**에서 읽어보세요.
Find the row whose key matches your
키와 일치하는 행을 찾으세요.
cell's VLNV; the **2nd quoted field is the rule short-name**, and the rule id is
세포의 VLNV; **두 번째로 인용된 필드는 규칙 짧은 이름**이고 규칙 ID는 다음과 같습니다.
`xilinx.com:bd_rule:<short>` (validated: `axi_noc2`, `ps_wizard`, `versal_cips`,
`xilinx.com:bd_rule:<short>`(검증됨: `axi_noc2`, `ps_wizard`, `versal_cips`,
`pcie_versal`, `microblaze`, … all have rules).
`pcie_versal`, `microblaze`, … 모두 규칙이 있습니다).
If you have the dump text in a Tcl var,
Tcl var에 덤프 텍스트가 있는 경우,
`ipcfg::rule_for_vlnv $dump <vlnv>` extracts the id for you.
`ipcfg::rule_for_vlnv $dump <vlnv>`는 ID를 추출합니다.
The 3rd field is the rule's
세 번째 필드는 규칙의 필드입니다.
`rules.tcl`/`bd.tcl` path — open it (or `vivado_doc_search`) to learn the rule's `-config`
`rules.tcl`/`bd.tcl` 경로 — 이를 열어(또는 `vivado_doc_search`) 규칙의 `-config`를 알아보세요.
option names.
옵션 이름.
Also check whether the IP ships a **Configurable Example Design**.
또한 IP가 **구성 가능한 예제 디자인**을 제공하는지 확인하세요.
2. **Run the automation and read what it changed** so you can confirm the result against intent.
2. **자동화를 실행하고 변경된 내용을 읽으면** 의도와 대조하여 결과를 확인할 수 있습니다.
2. Automation uses
2. 자동화 용도
the same 10-total-attempt budget per IP as parameter configuration; it does not get
매개변수 구성과 동일한 IP당 총 10회 시도 예산; 그것은 얻지 못한다
a separate recovery budget.
별도의 복구 예산.
Pass the rule's options — a **bare apply is often a no-op** for subsystem
규칙의 옵션을 전달합니다. **기본 적용은 하위 시스템에 대해 작동하지 않는 경우가 많습니다**
features (validated: `apply_bd_automation -rule xilinx.com:bd_rule:axi_noc2` with no
기능(검증됨: `apply_bd_automation -rule xilinx.com:bd_rule:axi_noc2` 없음
`-config` leaves `NUM_MC` at 0; the MC options must be supplied):
`-config`는 `NUM_MC`를 0으로 유지합니다. MC 옵션을 제공해야 함):
   ```tcl
   set base [ipcfg::snapshot_all $cell]
  set automation [ipcfg::run_automation $cell xilinx.com:bd_rule:<short> {<opt> <val> ...} $base]
  set evidence_file [file join $evidence_dir "automation.[pid].[clock clicks].tcldict"]
  puts [ipcfg::write_evidence $evidence_file $automation]
  if {![string match AUTOMATION:OK* $automation]} {error [string range $automation 0 799]}
   ```
Read the result: the `changed` keys (and any `added_cells`) are what automation set — match
결과를 읽어보세요. `changed` 키(및 `add_cells`)는 자동화 세트와 일치합니다.
them to the requirement instead of guessing gated CONFIG. (`ipcfg::try_automation` still
게이트된 CONFIG를 추측하는 대신 요구 사항에 따라 설정합니다. (`ipcfg::try_automation`은 아직
exists for a fire-only apply when you do not need the change report.)
변경 보고서가 필요하지 않은 경우 화재 전용 적용을 위해 존재합니다.)
3. **(design)** When multiple rules / example designs could apply, ask the user (clickable)
3. **(디자인)** 여러 규칙/예시 디자인을 적용할 수 있는 경우 사용자에게 질문합니다(클릭 가능).
which to apply rather than guessing.
추측하기보다는 적용하는 것이 좋습니다.

Per the give-up gate, you may not grade a requirement `integration-derived` until **either**
포기 게이트에 따라 **둘 중 하나**가 될 때까지 '통합 파생' 요구 사항을 채점할 수 없습니다.
the rule dump shows no rule for the VLNV (and no example design realizes it) **or** the rule
규칙 덤프에는 VLNV에 대한 규칙이 표시되지 않습니다(그리고 이를 실현하는 예제 디자인도 없음) **또는** 규칙
was applied with its options and its reported changes did not produce the value.
옵션과 함께 적용되었으며 보고된 변경 사항으로 인해 값이 생성되지 않았습니다.
A `SYSTEM` intent
'SYSTEM' 인텐트
whose params are disabled with **no** rule/enabler is legitimately integration-derived —
**no** 규칙/활성화로 매개변수가 비활성화된 매개변수는 합법적으로 통합에서 파생되었습니다.
report it via the coverage report, do not fake a value.
적용 범위 보고서를 통해 보고하고 값을 위조하지 마십시오.

For `versal_cips`, note (validated) that PL-clock frequencies, `PS_NUM_FABRIC_RESETS`, and
`versal_cips`의 경우 PL 클록 주파수 `PS_NUM_FABRIC_RESETS` 및
`PS_USE_PMCPL_CLK*` **do** stick directly inside `CONFIG.PS_PMC_CONFIG`; automation is for
`PS_USE_PMCPL_CLK*` **do**는 `CONFIG.PS_PMC_CONFIG` 안에 직접 붙이십시오. 자동화는
generating the external clk/reset **wiring**, not for setting those values.
해당 값을 설정하기 위한 것이 아니라 외부 clk/reset **배선**을 생성합니다.
Always prefer the
항상 선호하는
direct nested-dict path when `verify_stuck` confirms it stuck, and fall back to automation
'verify_stuck'이 중단되었음을 확인한 경우 직접 중첩된 사전 경로를 확인하고 자동화로 돌아갑니다.
(per docs) only for connectivity the dict cannot express.
(문서별) 연결에 대해서만 dict가 표현할 수 없습니다.

**Phase 3 procedure — enabler-first, multi-pass (the default, not an exception):**
**3단계 절차 — Enabler 우선, 다중 패스(기본값, 예외 없음):**
Treat ordered configuration as the *normal* path, not a special case.
순서가 지정된 구성을 특별한 경우가 아닌 *일반* 경로로 처리합니다.
Many params are
많은 매개변수는
**gated**: they only become settable once a parent param selects a mode/type/topology or
**게이트**: 상위 매개변수가 모드/유형/토폴로지를 선택한 후에만 설정 가능해집니다.
turns a feature on.
기능을 켭니다.
Setting a gated child before its enabler makes the child silently
활성화자 앞에 게이트된 하위를 설정하면 하위가 자동으로 설정됩니다.
**disabled/ignored** (see `PARAM_DISABLED` recovery in Step 5).
**비활성화/무시**(5단계의 `PARAM_DISABLED` 복구 참조)
So, by default:
따라서 기본적으로 다음과 같습니다.

1. **Pass 1 — enablers first:** set the params that *select or enable* something —
1. **패스 1 — 활성화 요소 먼저:** 무언가를 *선택하거나 활성화*하는 매개변수 설정 —
mode/type/device-selection, topology, channel/controller counts, and feature
모드/유형/장치 선택, 토폴로지, 채널/컨트롤러 수 및 기능
`ENABLE` toggles.
'활성화'가 전환됩니다.
These are the params other params depend on.
이는 다른 매개변수가 의존하는 매개변수입니다.
2. **Pass 2 — dependents:** set the params that only make sense once Pass 1 applied.
2. **패스 2 - 종속 항목:** 패스 1이 적용된 후에만 의미가 있는 매개변수를 설정합니다.
3. **Verify:** run `verify_stuck` + `verify_intent` after Pass 2.
3. **확인:** 패스 2 이후 `verify_stuck` + `verify_intent`를 실행합니다.

How to tell which params are enablers (IP-agnostic): the doc lists them as
어떤 매개변수가 활성화자인지 확인하는 방법(IP에 구애받지 않음): 문서에는 해당 매개변수가 다음과 같이 나열되어 있습니다.
prerequisites/parents, or the prompt's compound phrase implies a selection before a
전제 조건/부모 또는 프롬프트의 복합 문구는
detail (e.g. "select device type X, then configure its N controllers" — the device-type
세부 정보(예: "장치 유형 X를 선택한 다음 N 컨트롤러를 구성합니다" — 장치 유형
selection is the enabler that un-gates the controller params).
선택은 컨트롤러 매개변수를 게이트 해제하는 활성화 도구입니다.
When unsure, set the
확실하지 않은 경우 다음을 설정하세요.
most "selecting/enabling"-sounding params in Pass 1 and let `find_disabled` /
패스 1에서 대부분의 "선택/활성화" 소리 매개변수를 사용하고 `find_disabled`를 허용합니다.
`PARAM_DISABLED` feedback confirm or correct the ordering.
`PARAM_DISABLED` 피드백으로 주문을 확인하거나 수정합니다.

Concrete IP examples that need this ordering (illustrative, not exhaustive — RFDC mixer
이 순서가 필요한 구체적인 IP 예(완전하지 않은 예시 - RFDC 믹서)
type requires tile/slice enables; `v_tc` generator sizes require `max_lines_per_frame`).
유형에는 타일/슬라이스 활성화가 필요합니다. `v_tc` 생성기 크기에는 `max_lines_per_frame`이 필요합니다.)
Split Phase 3 into ordered passes within the same script:
동일한 스크립트 내에서 3단계를 순서가 지정된 패스로 분할합니다.

```tcl
# --- Phase 3a: Enable parent features first ---
set_property -dict [list \
    CONFIG.ENABLE_PARENT {1} \
    CONFIG.PARENT_MODE  {some_mode} \
] [get_bd_cells <cell_name>]

# --- Phase 3b: Set dependent params (now valid) ---
set _prop_dict [list \
    CONFIG.CHILD_PARAM_A {value_a} \
    CONFIG.CHILD_PARAM_B {value_b} \
]
if {[catch {set_property -dict $_prop_dict [get_bd_cells <cell_name>]} _cfg_err]} {
    # ... error handling as above ...
}
```

IPs known to require multi-pass ordering:
다중 패스 주문이 필요한 것으로 알려진 IP:

| IP | Pass 1 (enables)<br>패스 1(활성화) | Pass 2 (dependent)<br>패스 2(종속) |
|---|---|---|
| `rfdc`<br>`rfdc` | Tile enable, slice enable, sampling rate<br>타일 활성화, 슬라이스 활성화, 샘플링 속도 | Mixer type, NCO freq, decimation<br>믹서 유형, NCO 주파수, 데시메이션 |

Note: `v_tc` GEN_* timing parameters (VACTIVE, HACTIVE, frame sizes) are
참고: `v_tc` GEN_* 타이밍 매개변수(VACTIVE, HACTIVE, 프레임 크기)는
runtime AXI-Lite register values programmed by the VTC driver, not design-time
디자인 타임이 아닌 VTC 드라이버에 의해 프로그래밍된 런타임 AXI-Lite 레지스터 값
CONFIG properties.
구성 속성.
Only `enable_generation` and `max_lines_per_frame` are
'enable_ Generation' 및 'max_lines_per_frame'만 가능합니다.
design-time params.
디자인 타임 매개변수.

**Phase 4 — Stub connections for connection-derived params (generic):** Some parameters are
**4단계 — 연결 파생 매개변수(일반)에 대한 스텁 연결:** 일부 매개변수는 다음과 같습니다.
read-only in BD because their value propagates from an interface connection, not
BD에서는 해당 값이 인터페이스 연결에서 전파되기 때문에 읽기 전용입니다.
`set_property`.
`세트_속성`.
After Phases 1-3 succeed, attach a boundary interface port with
1~3단계가 성공하면 경계 인터페이스 포트를 연결합니다.
`ipcfg::add_stub` — it auto-detects the pin's interface VLNV requirement (you pass the VLNV)
`ipcfg::add_stub` — 핀의 인터페이스 VLNV 요구 사항을 자동 감지합니다(VLNV를 전달함).
and **mirrors the pin's mode** (an external port attached to a `Slave` pin must itself be a
**핀의 모드를 미러링합니다**('슬레이브' 핀에 연결된 외부 포트는 그 자체가
`Slave` port; passing the wrong mode yields `[BD 41-172] modes ... do not match`):
'슬레이브' 포트; 잘못된 모드를 전달하면 `[BD 41-172] 모드 ... 일치하지 않음`이 발생합니다.):

```tcl
puts [ipcfg::add_stub $cell <pin_name> <intf_vlnv> ?<CONFIG_prop> <val>?]
# e.g. drive an AXI-Stream slave input so the cell adopts the connected width:
puts [ipcfg::add_stub $cell S_AXIS xilinx.com:interface:axis_rtl:1.0 TDATA_NUM_BYTES 8]
```

**Important nuance (validated):** for many AXI-Stream *receive* inputs the width parameter is
**중요한 뉘앙스(검증됨):** 많은 AXI-Stream *수신* 입력의 경우 너비 매개변수는 다음과 같습니다.
**read-only on the boundary port too** (`[BD 41-737] ...
**경계 포트에서도 읽기 전용입니다** (`[BD 41-737] ...
TDATA_NUM_BYTES ... read-only`) —
TDATA_NUM_BYTES ... 읽기 전용`) —
the width is inherited from whatever master is connected at integration time, and a bare
너비는 통합 시 연결된 모든 마스터에서 상속되며 기본
boundary port has no master to define it.
경계 포트에는 이를 정의할 마스터가 없습니다.
`add_stub` tolerates this (the property set is
`add_stub`은 이를 허용합니다(속성 세트는
wrapped in `catch`): the connection still elaborates, but you **cannot force** the width
`catch`로 래핑됨): 연결은 여전히 정교하지만 너비를 **강제할 수는 없습니다**
standalone.
독립형.
When the target width param is still wrong after the stub, that feature is
스텁 후에도 대상 너비 매개변수가 여전히 잘못된 경우 해당 기능은 다음과 같습니다.
genuinely **integration-derived** — grade `partial` and record the param in `runtime_params`,
진정으로 **통합 파생** — '부분' 등급을 매기고 'runtime_params'에 매개변수를 기록합니다.
do not keep fighting it.
계속 싸우지 마십시오.
Use stubs to *enable elaboration / drive settable port widths*, not
스텁을 사용하여 *정교화를 활성화하고 설정 가능한 포트 너비를 구동*합니다.
to fake values the datapath must supply.
값을 가짜로 만들려면 데이터 경로가 제공해야 합니다.

`ipcfg::cleanup <cell> ?orig_part?` removes all `STUB_*` ports (and their nets), deletes the
`ipcfg::cleanup <cell> ?orig_part?`는 모든 `STUB_*` 포트(및 해당 네트)를 제거하고
cell, and restores the part if you pass the original — so a single call replaces the manual
원본을 전달하면 부품을 복원하므로 한 번의 전화로 매뉴얼을 대체할 수 있습니다.
stub/port/part teardown.
스텁/포트/부분 분해.

**When no part swap is needed**, omit Phase 0 entirely and remove the `_need_swap`
**부품 교체가 필요하지 않은 경우** 0단계를 완전히 생략하고 `_need_swap`을 제거하세요.
restore blocks — the script simplifies to Phases 1-3 (and optionally Phase 4) only.
블록 복원 — 스크립트는 1~3단계(및 선택적으로 4단계)로만 단순화됩니다.

**Cleanup TCL (always runs after recording results):**
**정리 TCL(항상 결과 기록 후 실행):**

```tcl
# One call deletes STUB_* ports + the cell, and restores the part if it was swapped.
# Pass $orig only when a swap happened; omit (or pass "") otherwise.
puts [ipcfg::cleanup <cell_name> $orig]
```

This replaces the manual `delete_bd_objs` / part-restore blocks.
이는 수동 `delete_bd_objs` / 부분 복원 블록을 대체합니다.
It is safe to call even when
전화해도 안전해요
no stubs or swap were used (the port/part steps are no-ops).
스텁이나 스왑은 사용되지 않았습니다(포트/부분 단계는 작동하지 않습니다).

Execute:
실행:
```
vivado_execute("source -notrace /tmp/configure_<cell_name>.tcl")
```

### Step 4: Handle result
### 4단계: 결과 처리

Parse the output line.
출력 라인을 구문 분석합니다.
Possible prefixes:
가능한 접두사:

| Output Prefix<br>출력 접두사 | Meaning<br>의미 | Next Step<br>다음 단계 |
|---|---|---|
| `SUCCESS:`<br>`성공:` | All parameters applied<br>모든 매개변수가 적용됨 | Run the self-audit below before reporting<br>신고 전 아래 자체 감사를 실행하세요. |
| `STUCK_FAIL:...`<br>`걸림 실패:...` | A key reverted / was silently ignored<br>키가 되돌려졌거나 자동으로 무시되었습니다. | Tier 2: format-mirror retry or `PARAM_DISABLED` recovery<br>계층 2: 포맷 미러 재시도 또는 `PARAM_DISABLED` 복구 |
| `INTENT_FAIL:...`<br>`의도_실패:...` | A key is `suspect-no-change` (likely wrong param)<br>키는 'suspect-no-change'입니다(잘못된 매개변수일 가능성이 있음). | Tier 2: remap via doc descriptions, re-apply<br>Tier 2: 문서 설명을 통해 다시 매핑하고 다시 적용 |
| `CONFIGURE_FAIL:VLNV_NOT_FOUND:...`<br>`CONFIGURE_FAIL:VLNV_NOT_FOUND:...` | IP not available even after part swap<br>부품 교체 후에도 IP를 사용할 수 없음 | Record failure — no retry<br>기록 실패 - 재시도 없음 |
| `CONFIGURE_FAIL:<err_type>:...`<br>`CONFIGURE_FAIL:<err_type>:...` | Configuration error<br>구성 오류 | Enter Tier 2 using `<err_type>`<br>`<err_type>`을 사용하여 Tier 2에 진입 |

Note: `PART_SWAP:` lines are informational — record the target part in results as
참고: `PART_SWAP:` 줄은 정보 제공용입니다. 결과의 대상 부분을 다음과 같이 기록합니다.
`part_swapped`.
'부분 교환'.

**Self-audit before declaring SUCCESS (required):** A `SUCCESS:` line is *necessary but
**SUCCESS 선언 전 자체 감사(필수):** 'SUCCESS:' 줄은 *필요하지만
not sufficient*.
충분하지 않습니다*.
Before reporting, confirm all of:
보고하기 전에 다음 사항을 모두 확인하세요.
1. `ipcfg::verify_stuck` returned empty (no revert).
1. `ipcfg::verify_stuck`이 비어 있는 상태로 반환되었습니다(되돌리기 없음).
2. `ipcfg::verify_intent` flagged no **hard reverts**.
2. `ipcfg::verify_intent`에 **하드 되돌리기**가 없음으로 표시되었습니다.
2. A `suspect-no-change` (value already
2. '의심스러운 변화 없음'(값은 이미
equalled the default) is a **soft** flag, not an automatic failure: re-read the doc
기본값과 같음)은 자동 실패가 아닌 **소프트** 플래그입니다. 문서를 다시 읽으세요.
description for that requirement.
해당 요구사항에 대한 설명입니다.
If the param you set is the doc-correct one and its
설정한 매개변수가 문서에 맞는 매개변수이고 그 매개변수가
default legitimately satisfies the intent, **accept it** and record `value_src=default`
기본값은 의도를 합법적으로 충족하고 **수락**하고 `value_src=default`를 기록합니다.
in the ledger (a user/answer not changing a param does not make the default wrong).
원장에서 (매개 변수를 변경하지 않은 사용자/답변은 기본값을 잘못 만들지 않습니다).
Only
만
treat it as `INTENT_FAIL` when a *competing* param better matches the doc description —
*경쟁* 매개변수가 문서 설명과 더 잘 일치하는 경우 `INTENT_FAIL`로 처리합니다. —
then remap and re-apply. (Re-run `verify_intent` with `flag_no_change=0` once you have
그런 다음 다시 매핑하고 다시 적용하십시오. ('flag_no_change=0'을 사용하여 'verify_intent'를 다시 실행하세요.
confirmed the grounding, so only hard reverts remain.)
접지가 확인되었으므로 하드 복귀만 남습니다.)
**Exception — on/enable requirements:** when the requirement asks to turn something on,
**예외 — 요구 사항 켜기/활성화:** 요구 사항에서 무언가를 켜도록 요청하는 경우,
enable or include it, and the param you chose already held that value by default, do
활성화하거나 포함하고 선택한 매개변수는 이미 기본적으로 해당 값을 보유하고 있습니다.
not accept the default yet.
아직 기본값을 수락하지 마세요.
Run `ipcfg::discover_params <vlnv> <keyword>` with the
`ipcfg::discover_params <vlnv> <keyword>`를 실행하세요.
feature keyword (e.g.
기능 키워드(예:
`REG_IF`) and read each match's description: one phrase often
`REG_IF`) 각 일치 항목의 설명을 읽습니다. 한 문구가 자주 사용됩니다.
covers one param per sub-block (controller, PHY, ...).
하위 블록(컨트롤러, PHY 등)당 하나의 매개변수를 다룹니다.
Set every match the description
모든 일치 항목을 설명으로 설정
supports; accept `value_src=default` only when no sibling applies.
지원; 형제가 적용되지 않는 경우에만 `value_src=default`를 허용합니다.
3. `ipcfg::find_disabled <raw_output>` returned `""` (no non-fatal gated-param warning).
3. `ipcfg::find_disabled <raw_output>`이 `""`를 반환했습니다(치명적이지 않은 Gated-param 경고 없음).
4. **Ledger coverage:** every requirement from Step 1.5 is mapped to a param that is
4. **원장 적용 범위:** 1.5단계의 모든 요구 사항은 다음과 같은 매개변수에 매핑됩니다.
confirmed by 1–3.
1~3으로 확인됩니다.
Any uncovered or suspect requirement downgrades the result and
발견되지 않았거나 의심스러운 요구 사항이 있으면 결과가 다운그레이드되고
triggers the matching Tier-2 recovery instead of reporting `full`.
'전체'를 보고하는 대신 일치하는 Tier-2 복구를 트리거합니다.

### Step 5: Error recovery (Tier 2)
### 5단계: 오류 복구(계층 2)

The `CONFIGURE_FAIL` line contains a classified error type.
'CONFIGURE_FAIL' 줄에는 분류된 오류 유형이 포함되어 있습니다.
Each type has a
각 유형에는
different recovery strategy:
다양한 복구 전략:

| Error Type<br>오류 유형 | Vivado Message Pattern<br>Vivado 메시지 패턴 | Recovery<br>회복 |
|---|---|---|
| `VLNV_NOT_FOUND`<br>`VLNV_NOT_FOUND` | IP unavailable after part swap<br>부품 교체 후 IP를 사용할 수 없음 | Try doc search for correct part, swap again if plausible; otherwise record failure<br>올바른 부품을 찾기 위해 문서 검색을 시도하고 그럴듯하면 다시 교체하십시오. 그렇지 않으면 기록 실패 |
| `PARAM_NOT_FOUND`<br>`PARAM_NOT_FOUND` | `[BD 41-1276] Cannot set the parameter X ... does not exist` (hard error) **or** the non-fatal `[BD 41-1276] ... does not exist` / `[IP_Flow 19-7090] Invalid parameter ... Ignoring` caught by the Phase 3 verification loop / output scan<br>`[BD 41-1276] 매개변수 X를 설정할 수 없습니다 ... 존재하지 않습니다`(하드 오류) **또는** 치명적이지 않은 `[BD 41-1276] ... 존재하지 않습니다` / `[IP_Flow 19-7090] 잘못된 매개변수 ... 무시`가 3단계 확인 루프/출력 스캔에 의해 포착되었습니다. | Drop unknown params; doc-search for correct names; rebuild dict<br>알 수 없는 매개변수를 삭제하세요. 정확한 이름을 위한 문서 검색; 딕셔너리 다시 작성 |
| `VALUE_OUT_OF_RANGE`<br>`VALUE_OUT_OF_RANGE` | `[IP_Flow 19-3461] Value 'V' is out of the range ... Valid values are - A, B, C`<br>`[IP_Flow 19-3461] 값 'V'가 범위를 벗어났습니다... 유효한 값은 - A, B, C입니다` | Parse valid values from the error text; pick closest match; rebuild dict<br>오류 텍스트에서 유효한 값을 구문 분석합니다. 가장 가까운 일치 항목을 선택하십시오. 딕셔너리 다시 작성 |
| `READ_ONLY`<br>`읽기 전용` | `[BD 41-737] Cannot set the parameter X ... It is read-only`<br>`[BD 41-737] X 매개변수를 설정할 수 없습니다. 읽기 전용입니다` | Strip read-only params and re-run — **no doc search needed** (saves 1 MCP call)<br>읽기 전용 매개변수를 제거하고 다시 실행 — **문서 검색이 필요하지 않음**(1개의 MCP 호출 저장) |
| `NOT_SUPPORTED`<br>`지원되지 않음` | `[BD 5-683] VLNV ... is not supported for the current part`<br>`[BD 5-683] VLNV ... 현재 부분에서는 지원되지 않습니다` | Record failure — feature unavailable on this part<br>기록 실패 - 이 부분에서는 기능을 사용할 수 없습니다. |
| `PARAM_DISABLED`<br>`PARAM_DISABLED` | `disabled parameter ... ignored` / `[BD 41-721]` (often **non-fatal** — caught by `ipcfg::find_disabled` or the output scan, not a thrown error)<br>`disabled 매개변수 ... 무시됨` / `[BD 41-721]` (종종 **치명적이지 않음** — `ipcfg::find_disabled` 또는 출력 스캔에 의해 포착되며 오류가 발생하지 않음) | **Not terminal, not `creation_only`.** Find + set the gating parent first, then re-apply (see below)<br>**터미널이 아니고 'creation_only'가 아닙니다.** 먼저 게이팅 상위 항목을 찾아 설정한 다음 다시 적용하세요(아래 참조). |
| `INTENT_FAIL`<br>`의도_실패` | *No Vivado message* — `verify_intent` flagged a **hard revert**, or a `suspect-no-change` that doc re-check shows a *competing* param better matches<br>*Vivado 메시지 없음* — `verify_intent`가 **하드 되돌리기**를 플래그 지정했거나 문서 재확인에서 *경쟁* 매개변수가 더 잘 일치함을 보여주는 `suspect-no-change`를 표시했습니다. | Re-read doc descriptions; remap the suspect requirement to the correctly-grounded param and re-apply. If the param is doc-correct and the default satisfies intent, accept as `value_src=default` (not a failure)<br>문서 설명을 다시 읽으세요. 의심되는 요구 사항을 올바르게 기반된 매개변수에 다시 매핑하고 다시 적용합니다. 매개변수가 문서에 정확하고 기본값이 의도를 충족하는 경우 'value_src=default'로 수락합니다(실패 아님). |
| `UNKNOWN`<br>'알 수 없음' | Anything else<br>다른 것 | Call `vivado_log_messages` for detail, then doc-search<br>자세한 내용은 `vivado_log_messages`를 호출한 다음 문서 검색을 통해 확인하세요. |

**For `VLNV_NOT_FOUND` after no part swap was attempted:** The IP may need a
**부품 교환을 시도하지 않은 후 `VLNV_NOT_FOUND`의 경우:** IP에
different device family.
다른 장치 제품군.
Use `vivado_doc_search("<ip_name> supported devices")` to
`vivado_doc_search("<ip_name> 지원 장치")`를 사용하여
find the correct family, select a part from the fallback map, and retry with a part
올바른 제품군을 찾고 대체 맵에서 부품을 선택한 다음 부품으로 다시 시도하세요.
swap.
교환.
This is the one case where Tier 2 can recover from a VLNV failure.
이는 Tier 2가 VLNV 오류로부터 복구할 수 있는 한 가지 사례입니다.

**For `READ_ONLY` errors:** The read-only parameter often already has the desired
**`READ_ONLY` 오류의 경우:** 읽기 전용 매개변수에는 이미 원하는 값이 이미 있는 경우가 많습니다.
default value (e.g., `C_EXT_RESET_HIGH` defaults to active-low).
기본값(예: `C_EXT_RESET_HIGH`의 기본값은 활성-낮음)입니다.
Drop it from the
에서 삭제하세요.
dict and retry immediately — no doc search, no wasted MCP call:
dict 및 즉시 재시도 — 문서 검색 없음, 낭비되는 MCP 통화 없음:

```tcl
# Tier 2 retry — read-only params stripped
set_property -dict [list \
    CONFIG.PARAM_A {value_a} \
    CONFIG.PARAM_C {value_c} \
] [get_bd_cells <cell_name>]
puts "SUCCESS: <cell_name> configured (read-only params skipped)"
```

**For `VALUE_OUT_OF_RANGE` errors:** Vivado's error text includes the valid set, so this
**`VALUE_OUT_OF_RANGE` 오류의 경우:** Vivado의 오류 텍스트에는 유효한 세트가 포함되어 있으므로
is the **default zero-doc-search recovery** — never spend an MCP doc call here.
**기본 제로 문서 검색 복구**입니다. 여기서 MCP 문서 호출을 사용하지 마십시오.
Parse the
구문 분석
list directly:
직접 나열:
```
Valid values are - 6, 8, 10, 12
```
Then decide deterministically:
그런 다음 결정론적으로 결정합니다.
- If the requested value maps to a member of the set (or a clear nearest member that still
- 요청된 값이 집합의 구성원(또는 여전히 가장 가까운 구성원)에 매핑되는 경우
satisfies intent), apply it and grade `full`.
의도를 만족) 적용하고 'full' 등급을 매깁니다.
- If the requested value is **provably outside** the valid set with no equivalent (e.g. a
- 요청된 값이 상응하는 값이 없는 유효한 세트 **외부**인 경우(예:
4-tap scaler when the minimum is 6), apply the closest valid value and grade `negative`,
최소값이 6인 경우 4탭 스케일러), 가장 가까운 유효한 값과 '음수' 등급을 적용하고,
recording the valid set in `fidelity_note`.
'fidelity_note'에 유효한 세트를 기록합니다.

**For `PARAM_DISABLED` errors (gated parameter) — reconcile first, then find the enabler:**
**`PARAM_DISABLED` 오류(게이트 매개변수)의 경우 — 먼저 조정한 다음 인에이블러를 찾으세요.**
A disabled/ignored parameter means the param is real but currently *gated*.
비활성화/무시된 매개변수는 매개변수가 실제이지만 현재 *게이트됨*을 의미합니다.
This is
이것은
**recoverable** and must never be graded `creation_only` on first sight.
**복구 가능**하며 첫눈에 'creation_only' 등급을 받아서는 안 됩니다.
Recovery
회복
(IP-agnostic):
(IP에 구애받지 않음):

1. **Detect it even when non-fatal.** A gated param frequently does not throw — Vivado
1. **치명적이지 않은 경우에도 감지합니다.** 게이트 매개변수는 자주 발생하지 않습니다 — Vivado
emits a non-fatal warning and `catch` returns 0.
치명적이지 않은 경고를 내보내고 'catch'는 0을 반환합니다.
Always run
항상 실행
`ipcfg::find_disabled <raw_output>` (and the output scan) after apply; a `DISABLED:...`
적용 후 `ipcfg::find_disabled <raw_output>`(및 출력 스캔); '비활성화됨:...'
result is the trigger, not just a thrown `CONFIGURE_FAIL:PARAM_DISABLED`.
결과는 단순히 `CONFIGURE_FAIL:PARAM_DISABLED`가 발생한 것이 아니라 트리거입니다.
2. **Reconcile before hunting an enabler.** Run `ipcfg::reconcile_disabled <cell> <dict>`:
2. **인에이블러를 찾기 전에 조정하십시오.** `ipcfg::reconcile_disabled <cell> <dict>`를 실행하십시오.
   - **`OMIT:{...}`** — these gated keys already hold the value you wanted (the read-only /
   - **`생략:{...}`** — 이러한 게이트 키는 이미 원하는 값을 보유하고 있습니다(읽기 전용 /
gated value equals your intent). **Drop them and move on** — no enabler search, no
게이트된 가치는 귀하의 의도와 동일합니다). **삭제하고 계속 진행** — 지원 요소 검색 없음, 없음
re-apply.
다시 신청하세요.
This saves MCP calls and is the common case.
이는 MCP 호출을 저장하며 일반적인 경우입니다.
   - **`DIFFER:{...}`** — only these are genuine problems (gated value ≠ intent).
   - **`다름:{...}`** — 이는 실제 문제입니다(제한된 값 ≠ 의도).
   - Continue
   - 계속
to step 3 for the `DIFFER` set only.
'DIFFER' 세트의 경우에만 3단계로 진행하세요.
3. **Find the enabler via doc search (DIFFER keys only):**
3. **문서 검색을 통해 Enabler 찾기(다른 키만 해당):**
   ```
   vivado_doc_search("<IP name> <gated_param_or_feature> prerequisite enable depends on")
   ```
Look for the parent param that selects the mode/type/device/topology which un-gates it.
게이트를 해제하는 모드/유형/장치/토폴로지를 선택하는 상위 매개변수를 찾으세요.
4. **Set the enabler first, then the still-differing gated params** — the enabler-first
4. **인에이블러를 먼저 설정한 다음 여전히 다른 게이트 매개변수를 설정합니다** — 인에이블러 우선
multi-pass from Phase 3:
3단계의 다중 패스:
   ```tcl
   # Pass 1: the enabler the doc named (mode/type/device-selection/feature toggle)
   puts "ENABLER:[ipcfg::apply_dict $cell [list CONFIG.<ENABLER> {<value>}]]"
   # Pass 2: re-apply the DIFFER params (now valid)
   puts "GATED:[ipcfg::apply_dict $cell [list CONFIG.<GATED_A> {a}]]"
   puts "STUCK:[ipcfg::verify_stuck $cell [list CONFIG.<GATED_A> {a}]]"
   ```
5. **Re-verify** with `verify_stuck` + `verify_intent`.
5. **다시 확인** `verify_stuck` + `verify_intent`를 사용하세요.
5. Only if a `DIFFER` param still will
5. `DIFFER` 매개변수가 여전히
not take *after* the enabler is confirmed set do you grade `partial` (see the tightened
*인에이블러가 확인된 후* 사용하지 않음 '부분' 등급을 지정합니까(강화된 내용 참조)
give-up gate in Step 4), and record that exact prompt fragment as `unapplied:gated-no-enabler`.
4단계의 포기 게이트) 정확한 프롬프트 부분을 'unapplied:gated-no-enabler'로 기록하세요.

This generalizes the validated "live inputs only appear once an IO-type selector is set"
이는 검증된 "라이브 입력은 IO 유형 선택기가 설정된 후에만 나타납니다"를 일반화합니다.
case and any "device-type selection un-gates the controller/channel params" pattern — the
경우 및 모든 "장치 유형 선택은 컨트롤러/채널 매개변수를 해제합니다" 패턴 —
agent discovers the specific enabler from docs, the procedure stays IP-independent.
에이전트가 문서에서 특정 Enabler를 발견하면 절차는 IP 독립적으로 유지됩니다.

**For `PARAM_NOT_FOUND` errors:** This is the main case requiring a doc search.
**`PARAM_NOT_FOUND` 오류의 경우:** 문서 검색이 필요한 주요 사례입니다.
The parameter name in the error is the one to search for:
오류의 매개변수 이름은 다음과 같이 검색할 이름입니다.
```
vivado_doc_search("<ip_name> <param_from_error> valid parameter names")
```

**For `PARAM_NOT_FOUND` in BD — standalone IP fallback:** Some CONFIG params
**BD의 `PARAM_NOT_FOUND`의 경우 — 독립형 IP 폴백:** 일부 CONFIG 매개변수
exist in standalone mode (`create_ip`) but are hidden in BD mode (`create_bd_cell`).
독립형 모드(`create_ip`)에는 존재하지만 BD 모드(`create_bd_cell`)에서는 숨겨져 있습니다.
When a `PARAM_NOT_FOUND` error occurs for a param that doc search confirms is
문서 검색에서 확인한 매개변수에 대해 'PARAM_NOT_FOUND' 오류가 발생하는 경우
valid for the IP, try the standalone path as a validation mechanism:
IP에 유효한 경우 유효성 검사 메커니즘으로 독립 실행형 경로를 사용해 보세요.

```tcl
# Tier 2 fallback: standalone IP validation
create_ip -vlnv <vlnv> -module_name <name>_standalone -dir /tmp
set_property -dict [list \
    CONFIG.PARAM_A {value_a} \
    CONFIG.PARAM_B {value_b} \
] [get_ips <name>_standalone]
puts "STANDALONE_OK:<id>"
# Cleanup standalone IP
remove_files [get_files /tmp/<name>_standalone/<name>_standalone.xci]
file delete -force /tmp/<name>_standalone
```

Testing showed that originally hypothesized BD-restricted params
테스트 결과 원래 가설은 BD 제한 매개변수인 것으로 나타났습니다.
(`axi_timer` C_ONE_TIMER_ONLY, `axis_interconnect` ARB_ALGORITHM,
(`axis_timer` C_ONE_TIMER_ONLY, `axis_interconnect` ARB_ALGORITHM,
`axis_switch` DECODER_BASE) are actually not configurable in ANY mode
`axis_switch` DECODER_BASE)는 실제로 ANY 모드에서 구성할 수 없습니다.
for current IP versions.
현재 IP 버전의 경우.
The standalone fallback remains available for
독립 실행형 폴백은 계속해서 사용할 수 있습니다.
future cases where a param genuinely exists in standalone but not BD.
매개변수가 독립 실행형에는 실제로 존재하지만 BD는 존재하지 않는 미래의 경우입니다.

The fidelity grade for standalone-validated configs is `full_standalone`.
독립형 검증 구성의 충실도 등급은 `full_standalone`입니다.

**For runtime-only features — annotation instead of failure:** Some requested
**런타임 전용 기능 — 실패 대신 주석:** 일부 요청됨
features are controlled by software register writes, not design-time CONFIG
기능은 디자인 타임 CONFIG가 아닌 소프트웨어 레지스터 쓰기에 의해 제어됩니다.
properties.
속성.
When doc search returns a register map instead of CONFIG params,
문서 검색이 CONFIG 매개변수 대신 레지스터 맵을 반환하는 경우,
emit a structured annotation:
구조화된 주석을 내보냅니다.

```
RUNTIME_CONFIG:<id>:<feature>=<value>:<register>[<bits>]@<offset>
```

Examples:
예:
- `RUNTIME_CONFIG:60:CPOL=1,CPHA=1:SPICR[4:3]@0x60`
- `RUNTIME_CONFIG:60:CPOL=1,CPHA=1:SPICR[4:3]@0x60`
- `RUNTIME_CONFIG:100:SPEED_P0=100G,FEC=RS-FEC:PORT0_CONFIGURATION_REV1[7:0]@0x0100`
- `RUNTIME_CONFIG:100:SPEED_P0=100G,FEC=RS-FEC:PORT0_CONFIGURATION_REV1[7:0]@0x0100`

The benchmark result gets a `runtime_params` field:
벤치마크 결과는 `runtime_params` 필드를 얻습니다.

```json
{
  "runtime_params": {
    "CPOL": {"value": 1, "register": "SPICR", "bit": 3, "offset": "0x60"},
    "CPHA": {"value": 1, "register": "SPICR", "bit": 4, "offset": "0x60"}
  }
}
```

IPs with known runtime-only features:
알려진 런타임 전용 기능이 있는 IP:

| IP | Runtime Feature<br>런타임 기능 | Register<br>등록 | Notes<br>메모 |
|---|---|---|---|
| `axi_quad_spi`<br>`axi_quad_spi` | CPOL, CPHA<br>CPOL, CPHA | SPICR (0x60)<br>스파이커(0x60) | SPI clock polarity/phase<br>SPI 클록 극성/위상 |
| `mrmac`<br>'mrmac' | Port speed, FEC, PTP<br>포트 속도, FEC, PTP | Per-port config registers<br>포트별 구성 레지스터 | All port config is runtime<br>모든 포트 구성은 런타임입니다. |
| `axi_ethernet`<br>`축_이더넷` | DMA enable<br>DMA 활성화 | External — DMA is a separate IP<br>외부 — DMA는 별도의 IP입니다. | Not a register; requires companion IP<br>레지스터가 아닙니다. 컴패니언 IP가 필요합니다 |
| `v_tc`<br>`v_tc` | GEN_* timing params<br>GEN_* 타이밍 매개변수 | Generation registers (0x60-0x84)<br>생성 레지스터(0x60-0x84) | 1080p60 timing is set by VTC driver<br>1080p60 타이밍은 VTC 드라이버에 의해 설정됩니다. |

When runtime params are identified, the design-time CONFIG params that ARE
런타임 매개변수가 식별되면 디자인 타임 CONFIG 매개변수는
available should still be set via `set_property -dict`.
available은 여전히 `set_property -dict`를 통해 설정되어야 합니다.
The fidelity grade
충실도 등급
becomes `full` (with `runtime_params` populated) since the design-time config
디자인 타임 구성 이후 `full`(`runtime_params`가 채워짐)이 됩니다.
is complete and the runtime portion is documented.
완료되었으며 런타임 부분이 문서화되었습니다.

**Give-up gate — do NOT grade `creation_only`/`partial` prematurely.** You may never
**포기 관문 - `creation_only`/`partial` 등급을 미리 채점하지 마세요.**
conclude a parameter is integration-derived, connection-derived, gated, or runtime-only
매개변수가 통합 파생, 연결 파생, 게이트형 또는 런타임 전용이라고 결론을 내립니다.
**from reasoning alone**.
**추론만으로**.
Before downgrading a requirement, you MUST have evidence of all:
요구 사항을 다운그레이드하기 전에 다음 사항에 대한 증거가 있어야 합니다.
1. **You actually attempted the literal value with `set_property` and it failed/reverted.**
1. **실제로 `set_property`를 사용하여 리터럴 값을 시도했지만 실패/복귀했습니다.**
A param you merely *assumed* is derived does not count.
단순히 *가정* 파생되었다고 가정한 매개변수는 포함되지 않습니다.
In particular:
특히:
   - **Try the global / top-level key form, not just per-instance keys.** Many counts and
   - **단순히 인스턴스별 키가 아닌 전역/최상위 키 형식을 사용해 보세요.** 개수가 많고
widths are set by a single global `CONFIG.<X>` even when per-port `CONFIG.<Sn>_<X>`
너비는 포트별 `CONFIG.<Sn>_<X>`인 경우에도 단일 전역 `CONFIG.<X>`에 의해 설정됩니다.
keys are read-only (e.g. a NoC data width applied once globally rather than per slave
키는 읽기 전용입니다(예: 슬레이브당이 아닌 전체적으로 한 번 적용되는 NoC 데이터 너비
port).
포트).
Attempt the global key before declaring the value connection-derived.
연결 파생 값을 선언하기 전에 전역 키를 시도하십시오.
   - Run `verify_stuck` to confirm it genuinely reverted, not that you skipped it.
   - 건너뛴 것이 아니라 정말로 되돌려졌는지 확인하려면 `verify_stuck`을 실행하세요.
2. **Enabler search attempted** (if disabled/ignored): you ran the `PARAM_DISABLED`
2. **인에이블러 검색 시도**(비활성화/무시된 경우): `PARAM_DISABLED`를 실행했습니다.
reconciliation (below) and, for any `DIFFER` key, doc-searched + set the gating parent
조정(아래) 및 'DIFFER' 키에 대해 문서 검색 + 게이팅 상위 설정
first — and the child *still* would not take.
먼저 — 그런데 그 아이는 *여전히* 받아들이려고 하지 않았습니다.
3. **Block Automation checked** (for integration/connectivity features): you ran
3. **블록 자동화 확인**(통합/연결 기능용): 실행했습니다.
`ipcfg::is_system_intent` and, if `SYSTEM`, enumerated the loaded rules with
`ipcfg::is_system_intent`, `SYSTEM`인 경우 로드된 규칙을 다음과 같이 열거합니다.
`::bd::util_cmd rules dump` (→ `ipcfg::rule_for_vlnv`) and either no rule exists for the
`::bd::util_cmd 규칙 덤프`(→ `ipcfg::rule_for_vlnv`) 및
VLNV (and no example design realizes it) or `ipcfg::run_automation` was applied
VLNV(및 이를 실현하는 예제 디자인 없음) 또는 `ipcfg::run_automation`이 적용되었습니다.
**with the rule's options** and its reported changes did not realize the value.
**규칙의 옵션** 및 보고된 변경 사항을 사용하여 값을 실현하지 못했습니다.
Integration-derived
통합 파생
is only valid after automation could not deliver it.
자동화가 이를 전달할 수 없는 경우에만 유효합니다.
4. **Doc confirms no design-time path.** A doc search shows the feature is genuinely
4. **문서에서는 디자인 타임 경로가 없음을 확인했습니다.** 문서 검색에 따르면 이 기능은 실제로 존재하는 것으로 나타났습니다.
runtime-register-only or integration/connection-derived (not just absent from your dict).
런타임 등록 전용 또는 통합/연결 파생(딕셔너리에 없는 것이 아니라).

If any of 1–4 is missing, continue evidence-driven recovery only while the shared attempt
1~4 중 하나라도 누락된 경우 공유 시도 동안에만 증거 중심 복구를 계속합니다.
budget remains.
예산이 남아있습니다.
Budget exhaustion takes precedence over this give-up gate: preserve the
예산 소진이 포기 관문보다 우선합니다.
unresolved evidence and ask for user intervention rather than claiming a proven limitation.
해결되지 않은 증거를 발견하고 입증된 제한 사항을 주장하기보다는 사용자 개입을 요청합니다.
Whatever you finally cannot apply MUST appear in the Step 1.5 ledger as `unapplied:<reason>`
최종적으로 적용할 수 없는 항목은 1.5단계 원장에 `적용되지 않음:<이유>`로 표시되어야 합니다.
and in the `ipcfg::coverage_report` output (see "Always report coverage").
그리고 `ipcfg::coverage_report` 출력에서 확인하세요("항상 적용 범위 보고" 참조).

**Maximum total attempts per IP: 10, including the initial attempt.** Share the count
**IP당 최대 총 시도 횟수: 초기 시도를 포함하여 10회.** 횟수 공유
across all recovery paths.
모든 복구 경로에 걸쳐.
After the tenth failed attempt, stop changes to that IP,
10번째 시도 실패 후 해당 IP에 대한 변경을 중지합니다.
report the attempt history and diagnostics, and ask for user intervention.
시도 이력 및 진단을 보고하고 사용자 개입을 요청합니다.

## Performance Targets
## 성과 목표

| Metric<br>미터법 | Target<br>대상 |
|--------|--------|
| Tier 1 success rate (easy prompts)<br>Tier 1 성공률(쉬운 프롬프트) | >90% |
| Tier 1 success rate (medium prompts)<br>Tier 1 성공률(중간 메시지) | >70% |
| Tier 1 success rate (hard prompts)<br>Tier 1 성공률(하드 프롬프트) | >40% |
| Tier 2 recovery rate (when Tier 1 fails)<br>Tier 2 복구율(Tier 1이 실패하는 경우) | >80% |
| True fidelity rate (full + full_standalone)<br>실제 충실도 비율(full + full_standalone) | >90% |
| Total MCP calls (Tier 1, no part swap)<br>총 MCP 통화(계층 1, 부품 교환 없음) | 2-3 |
| Total MCP calls (Tier 1, with part swap)<br>총 MCP 통화(Tier 1, 부품 교체 포함) | 3-4 |
| Total MCP calls (Tier 2 recovery)<br>총 MCP 통화(계층 2 복구) | 4-6 |
| Total MCP calls (Phase 4 stub connections)<br>총 MCP 호출(4단계 스텁 연결) | +1 (added to Tier 1 or 2)<br>+1(Tier 1 또는 2에 추가됨) |
| Total MCP calls (standalone fallback)<br>총 MCP 호출(독립형 대체) | +1 (Tier 2 only)<br>+1(티어 2에만 해당) |

## Fidelity Grades
## 충실도 등급

| Grade<br>등급 | Meaning<br>의미 |
|-------|---------|
| `full`<br>'가득' | CONFIG params match prompt intent; runtime params documented if applicable<br>CONFIG 매개변수는 프롬프트 의도와 일치합니다. 해당되는 경우 런타임 매개변수를 문서화합니다. |
| `full_standalone`<br>`완전 독립형` | Config validated via `create_ip` (standalone); param unavailable in BD mode<br>'create_ip'(독립형)을 통해 구성이 검증되었습니다. BD 모드에서는 매개변수를 사용할 수 없습니다. |
| `partial`<br>'부분적' | IP configured but some features not settable via CONFIG or connections<br>IP가 구성되었지만 일부 기능은 CONFIG 또는 연결을 통해 설정할 수 없습니다. |
| `negative`<br>'부정적' | Prompt asked for values outside Vivado's valid range<br>Vivado의 유효 범위를 벗어난 값을 요청하는 프롬프트 |
| `creation_only`<br>`창조_만` | IP created but key requested features aren't CONFIG properties<br>IP가 생성되었지만 주요 요청 기능이 CONFIG 속성이 아닙니다. |

## What NOT to do
## 하지 말아야 할 것

- **Do not query parameters one at a time** — always use `set_property -dict` for batch ops
- **한 번에 하나씩 매개변수를 쿼리하지 마세요** — 일괄 작업에는 항상 `set_property -dict`를 사용하세요.
- **Do not try to read the full parameter space** before configuring — go straight to the
- **구성하기 전에 전체 매개변수 공간을 읽으려고 하지 마십시오** — 바로
-dict. **Exception (sanctioned):** a *keyword-scoped* `ipcfg::discover_params <vlnv>
-dict. **예외(승인됨):** *키워드 범위* `ipcfg::discover_params <vlnv>
  <keyword>` on a throwaway scratch cell, used to map a requirement to its COMPLETE sibling
set.
설정합니다.
That is targeted neighborhood discovery, not a blind full dump, and is the only
이는 블라인드 전체 덤프가 아닌 타겟 동네 검색이며 유일한 방법입니다.
permitted `list_property` use.
`list_property` 사용이 허용됩니다.
- **Do not exceed 10 total attempts per IP, including the initial attempt** — stop and ask for user intervention
- **초기 시도를 포함하여 IP당 총 시도 횟수가 10회를 초과하지 마십시오** — 중지하고 사용자 개입을 요청합니다.
- **Do not hardcode IP versions** — use version-free VLNVs
- **IP 버전을 하드코딩하지 마세요** — 버전이 없는 VLNV를 사용하세요.
- **Do not issue individual `set_property` calls** for each parameter — one `-dict`, one call
- **각 매개변수에 대해 개별 `set_property` 호출을 실행하지 마세요** — 하나의 `-dict`, 하나의 호출
- **Do not use Phase 4 stubs when the param IS settable via CONFIG** — stubs are only for
- **CONFIG를 통해 매개변수를 설정할 수 있는 경우 4단계 스텁을 사용하지 마세요** — 스텁은
genuinely connection-derived read-only params
실제로 연결에서 파생된 읽기 전용 매개변수
- **Do not mix standalone and BD validation** in the same script — standalone is a separate path
- **독립 실행형과 BD 유효성 검사를 동일한 스크립트에 혼합하지 마십시오** — 독립 실행형은 별도의 경로입니다.
- **Do not trust a `catch` of `0` (or a `SUCCESS:` line) alone in BD mode** — unknown params
- **BD 모드에서 `0`(또는 `SUCCESS:` 줄)의 `catch`만 신뢰하지 마세요** — 알 수 없는 매개변수
emit non-fatal CRITICAL WARNINGs; always run the Phase 3 verification loop and scan output
치명적이지 않은 중요 경고를 발산합니다. 항상 3단계 검증 루프를 실행하고 출력을 스캔합니다.
- **Do not configure an IP that the docs show cannot deliver the requested capability** —
- **문서에서 요청한 기능을 제공할 수 없는 IP는 구성하지 마세요** —
grade `negative` and name the correct IP instead of forcing an ignored setting
무시된 설정을 강제하는 대신 '부정적' 등급을 지정하고 올바른 IP 이름을 지정하세요.
- **Do not spend a doc-search call on a `VALUE_OUT_OF_RANGE` error** — the valid set is in
- **`VALUE_OUT_OF_RANGE` 오류에 대해 문서 검색 호출을 낭비하지 마세요** — 유효한 세트는
the error text; parse it directly
오류 텍스트; 직접 분석해 보세요
- **Do not hand-roll the repetitive TCL** (part swap, create, dict+verify, stub, cleanup) —
- **반복적인 TCL을 직접 작성하지 마세요**(부분 교환, 생성, dict+검증, 스텁, 정리) —
call the `ipcfg::*` helpers from `lib/ipcfg.tcl` (sourced once per session); drop to raw
`lib/ipcfg.tcl`에서 `ipcfg::*` 도우미를 호출합니다(세션당 한 번씩 소스 제공). 원시로 드롭
TCL only for a quirk no helper covers
도우미가 포함되지 않은 특이한 TCL 전용
- **Do not declare success on an empty `catch` in BD mode** — a non-empty `STUCK_FAIL` list
- **BD 모드에서 빈 `catch`에 성공을 선언하지 마세요** — 비어 있지 않은 `STUCK_FAIL` 목록
from `ipcfg::verify_stuck` means a key reverted; recover before grading `full`
`ipcfg::verify_stuck`은 키가 되돌려졌다는 의미입니다. '전체' 등급을 매기기 전에 복구
- **Do not guess a reverted param's value shape** — read it back with `ipcfg::param_format`
- **되돌린 매개변수의 값 형태를 추측하지 마세요** — `ipcfg::param_format`으로 다시 읽어보세요
and mirror the structure (comma list / nested `{{...}}`) instead of a per-IP format table
IP별 형식 테이블 대신 구조(쉼표 목록/중첩 `{{...}}`)를 미러링합니다.
- **Do not keep fighting a width that is read-only on the boundary port too** — it is
- **경계 포트에서도 읽기 전용인 너비와 계속 싸우지 마세요** — 그렇습니다.
integration-derived; grade `partial` and record it in `runtime_params`
통합 파생; `partial` 등급을 매기고 `runtime_params`에 기록합니다.
- **Do not pick a param by name similarity** — ground every requirement in the param's
- **이름 유사성으로 매개변수를 선택하지 마세요** — 매개변수의 모든 요구사항을 근거로 합니다.
documented *description*; same-keyword params often implement different functions
문서화된 *설명*; 동일한 키워드 매개변수는 종종 다른 기능을 구현합니다.
- **Do not trust a stuck value that equals the default** for an enable/select requirement —
- **활성화/선택 요구 사항에 대한 기본값과 동일한 고정 값을 신뢰하지 마십시오** —
`verify_intent` flags it `suspect-no-change`; it usually means you set the wrong param
`verify_intent`는 이를 `suspect-no-change`로 표시합니다. 이는 일반적으로 잘못된 매개변수를 설정했음을 의미합니다.
- **Do not grade `creation_only` on a disabled/ignored param** before running the
- **비활성화/무시된 매개변수에 대해 `creation_only` 등급을 매기지 마세요**
`PARAM_DISABLED` reconciliation (`ipcfg::reconcile_disabled`; only `DIFFER` keys need an
`PARAM_DISABLED` 조정(`ipcfg::reconcile_disabled`; `DIFFER` 키에만
enabler — `OMIT` keys already hold the wanted value)
인에이블러 - 'OMIT' 키에 이미 원하는 값이 포함되어 있음)
- **Do not declare SUCCESS on the `SUCCESS:` line alone** — also require empty
- **`SUCCESS:` 줄에서만 SUCCESS를 선언하지 마세요** — 또한 비어 있어야 합니다.
`verify_intent` (hard reverts), empty `find_disabled`, and full Step 1.5 ledger coverage
`verify_intent`(하드 되돌리기), 빈 `find_disabled` 및 전체 1.5단계 원장 적용 범위
- **Do not declare a param "derived"/"runtime"/"gated" from reasoning** — you must have an
- **추론에서 매개변수 "파생"/"런타임"/"게이트"를 선언하지 마세요** —
actual failed `set_property` (including the global/top-level key form), enumerated the
실제로 실패한 `set_property`(전역/최상위 키 형식 포함), 열거된
loaded automation rules (`::bd::util_cmd rules dump` → `ipcfg::rule_for_vlnv`), and a doc
로드된 자동화 규칙(`::bd::util_cmd 규칙 덤프` → `ipcfg::rule_for_vlnv`) 및 문서
citation first (see the give-up gate)
먼저 인용(포기 게이트 참조)
- **Do not swap parts silently mid-design** — `ipcfg::ensure_part` is guarded; on
- **설계 도중에 조용히 부품을 교체하지 마세요** — `ipcfg::ensure_part`는 보호됩니다. 에
`WARN:SWAP_BLOCKED` confirm with the user (design mode) before forcing
`WARN:SWAP_BLOCKED`를 강제하기 전에 사용자(디자인 모드)에게 확인하세요.
- **Do not treat `suspect-no-change` as an automatic failure** — a doc-correct param whose
- **'suspect-no-change'를 자동 실패로 처리하지 마십시오** — 문서에 맞는 매개변수
default satisfies intent is `value_src=default`, which is acceptable
기본 만족 의도는 'value_src=default'이며 이는 허용됩니다.
- **Do not finish without disclosing coverage** — when the IP is not fully parameterized,
- **범위를 공개하지 않고 끝내지 마십시오** — IP가 완전히 매개변수화되지 않은 경우,
emit `ipcfg::coverage_report` and tell the user the exact prompt fragment(s) you could not
`ipcfg::coverage_report`를 내보내고 사용자에게 전달하지 못한 정확한 프롬프트 조각을 알려줍니다.
apply and why
신청하고 그 이유
- **Do not create-then-configure in two calls when one works** — prefer
- **하나가 작동할 때 두 호출로 생성 후 구성하지 마세요** — 선호
`ipcfg::create_cell_cfg` (single `create_bd_cell -set_param`); fall back to two-step only
`ipcfg::create_cell_cfg`(단일 `create_bd_cell -set_param`); 2단계로만 돌아가기
when needed for ordered/multi-pass config
주문/다중 패스 구성이 필요한 경우
