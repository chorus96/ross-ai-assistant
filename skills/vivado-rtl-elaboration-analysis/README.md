<!-- ko-translation: enabled -->
# RTL Elaboration Analysis Skill
# RTL 정교화 분석 기술

## What This Skill Does
## 이 기술의 역할

Analyzes Vivado synthesis elaboration diagnostics (`[Synth 8-XXX]` errors, critical
Vivado 합성 정교 진단(`[Synth 8-XXX]` 오류, 중요 분석)
warnings, and warnings) and ties every finding back to your actual RTL.
경고 및 경고) 모든 결과를 실제 RTL에 다시 연결합니다.
It explains
설명한다
root causes at the reported file and line, then proposes concrete RTL-level fixes.
보고된 파일 및 라인의 근본 원인을 파악한 다음 구체적인 RTL 수준 수정 사항을 제안합니다.

The report style is: summary first, hotspots next, then a per-message fix section for
보고서 스타일은 요약 먼저, 핫스팟 다음, 메시지별 수정 섹션입니다.
every actionable diagnostic.
실행 가능한 모든 진단.

## Operating Modes
## 작동 모드

1. **Log Analysis Mode (primary):** Parse `vivado.log`, `runme.log`, or
1. **로그 분석 모드(기본):** `vivado.log`, `runme.log` 구문 분석 또는
`synth_1/runme.log`, then inspect the referenced RTL and propose fixes.
`synth_1/runme.log`, 참조된 RTL을 검사하고 수정 사항을 제안합니다.
2. **Session Mode (secondary):** Use an active Vivado session to run elaboration and
2. **세션 모드(보조):** 활성 Vivado 세션을 사용하여 정교화를 실행하고
analyze results in real time.
결과를 실시간으로 분석합니다.

## Quick Usage
## 빠른 사용법

Analyze RTL plus a log:
RTL과 로그를 분석합니다.
- `Analyze /path/to/my_rtl and /path/to/vivado.log; identify RTL root causes and fixes`
- `/path/to/my_rtl 및 /path/to/vivado.log를 분석합니다. RTL 근본 원인 식별 및 수정'

Analyze a pasted diagnostic:
붙여넣은 진단을 분석합니다.
- `Analyze this [Synth 8-xxx] message against the RTL at the reported file and line`
- `보고된 파일 및 라인의 RTL에 대해 이 [Synth 8-xxx] 메시지를 분석합니다.`

Analyze with an active Vivado session:
활성 Vivado 세션으로 분석:
- `Run RTL elaboration in Vivado and analyze all Synth 8-XXX messages`
- `Vivado에서 RTL 정교화를 실행하고 모든 Synth 8-XXX 메시지를 분석하세요`

## Report Style
## 보고서 스타일

Expected output is organized like a synthesis analysis report:
예상 결과는 종합 분석 보고서처럼 구성됩니다.
1. Summary of severity counts and actionability
1. 심각도 수 및 실행 가능성 요약
2. Hotspot tables by RTL file and source location
2. RTL 파일 및 소스 위치별 핫스팟 테이블
3. Per-message analysis with cause, context, and recommended fix
3. 원인, 컨텍스트, 권장 수정 사항을 포함한 메시지별 분석
4. Final recommendations and next steps
4. 최종 권장 사항 및 다음 단계

Supporting references included in this package:
이 패키지에 포함된 지원 참조 자료:
- `report-format.md` — report structure, diff rules, and JSON schema
- `report-format.md` — 보고서 구조, 차이점 규칙 및 JSON 스키마
- `tcl-reference.md` — Vivado TCL flow for elaboration
- `tcl-reference.md` — 정교화를 위한 Vivado TCL 흐름
- `resolution-guide.md` — dispatcher and resolution policy
- `solution-guide.md` — 디스패처 및 해상도 정책
- `resolution/` — message-to-guide map
- `solution/` — 메시지-가이드 맵

## Structured JSON Input
## 구조화된 JSON 입력

When structured JSON diagnostics are available, the skill consumes them directly.
구조화된 JSON 진단을 사용할 수 있는 경우 기술은 이를 직접 사용합니다.
Your RTL and diagnostics remain the source of truth.
RTL 및 진단은 정보의 소스로 유지됩니다.

Minimum message fields:
최소 메시지 필드:
- `severity`
- '심각도'
- `msg_id`
- `msg_id`
- `message_text`
- `메시지_텍스트`
- `rtl_source_file`
- `rtl_source_file`
- `rtl_source_line`
- `rtl_source_line`

## Directory Layout
## 디렉토리 레이아웃

```
vivado-rtl-elaboration-analysis/
├── SKILL.md
├── README.md
├── message-handlers.md
├── resolution-guide.md
├── report-format.md
├── tcl-reference.md
├── parse_elab_messages.py
├── csv_to_per_file_parts.py
└── resolution/
```

## Core Commands
## 핵심 명령

Parse elaboration messages from a Vivado log:
Vivado 로그에서 상세 메시지를 구문 분석합니다.
- `python3 parse_elab_messages.py <log_file> <output_csv>`
- `python3 pars_elab_messages.py <log_file> <output_csv>`

Split a messages CSV into per-file parts:
메시지 CSV를 파일별 부분으로 분할합니다.
- `python3 csv_to_per_file_parts.py <elab_messages.csv> <output_dir>`
- `python3 csv_to_per_file_parts.py <elab_messages.csv> <output_dir>`

## Runbook
## 런북

### A) Log Analysis Flow (Primary)
### 가) 로그 분석 흐름(1차)

Prerequisites:
전제 조건:
- RTL/project sources
- RTL/프로젝트 소스
- A synthesis/elaboration log (`runme.log` or `vivado.log`), a diagnostic CSV, or a
- 종합/작성 로그(`runme.log` 또는 `vivado.log`), 진단 CSV 또는
pasted message
붙여넣은 메시지

Steps:
단계:
1. Parse messages from the log when available:
1. 가능한 경우 로그에서 메시지를 구문 분석합니다.
	- `python3 parse_elab_messages.py <log_file> reports/rtl-elaboration-analysis/messages.csv`
	- `python3 pars_elab_messages.py <log_file> 보고서/rtl-elaboration-analytic/messages.csv`
2. (If actionable messages > 50) split by file for the extended flow:
2. (실행 가능한 메시지 > 50인 경우) 확장 흐름을 위해 파일별로 분할:
	- `python3 csv_to_per_file_parts.py reports/rtl-elaboration-analysis/messages.csv reports/rtl-elaboration-analysis/`
	- `python3 csv_to_per_file_parts.py 보고서/rtl-elaboration-analytic/messages.csv 보고서/rtl-elaboration-analytic/`
3. Inspect the actual RTL around every reported file and line before proposing changes.
3. 변경 사항을 제안하기 전에 보고된 모든 파일과 라인에 대한 실제 RTL을 검사하십시오.
4. Generate the report and include a focused validation command for the project.
4. 보고서를 생성하고 프로젝트에 대한 집중적인 검증 명령을 포함합니다.

Expected outputs:
예상 출력:
- `reports/rtl-elaboration-analysis/messages.csv`
- `보고서/rtl-elaboration-analytic/messages.csv`
- `reports/rtl-elaboration-analysis/messages_by_file.txt` (extended flow)
- `reports/rtl-elaboration-analytic/messages_by_file.txt`(확장 흐름)
- `reports/rtl-elaboration-analysis/messages_by_file_part*.txt` (extended flow)
- `reports/rtl-elaboration-analytic/messages_by_file_part*.txt`(확장 흐름)

Validation checklist:
검증 체크리스트:
1. `messages.csv` exists and has a header plus data rows
1. `messages.csv`가 존재하며 헤더와 데이터 행이 있습니다.
2. The non-INFO row count matches the expected actionable count
2. INFO가 아닌 행 개수가 예상되는 실행 가능한 개수와 일치합니다.
3. A resolution checklist is generated in the report workflow (`resolution_checklist.md`)
3. 보고서 워크플로우(`solution_checklist.md`)에서 해결 체크리스트가 생성됩니다.
4. The final report follows the rule: one non-INFO message row → one `###` section
4. 최종 보고서는 INFO가 아닌 메시지 행 1개 → `###` 섹션 1개라는 규칙을 따릅니다.

### B) Session Flow (Secondary)
### B) 세션 흐름(보조)

Prerequisites:
전제 조건:
- An active Vivado session with the project and RTL loaded
- 프로젝트와 RTL이 로드된 활성 Vivado 세션

Steps:
단계:
1. Run RTL elaboration through the Vivado session.
1. Vivado 세션을 통해 RTL 정교화를 실행하세요.
2. Collect the complete elaboration diagnostics.
2. 완전한 정교 진단을 수집합니다.
3. Inspect the actual RTL and produce the same report as Log Analysis Mode.
3. 실제 RTL을 검사하고 로그 분석 모드와 동일한 보고서를 생성합니다.

### C) Missing Input Handling
### C) 누락된 입력 처리

If no log and no active Vivado session are available:
로그가 없고 활성 Vivado 세션을 사용할 수 없는 경우:
1. Ask for a `runme.log`/`vivado.log`, a diagnostic CSV, or a pasted `[Synth 8-XXX]` message.
1. `runme.log`/`vivado.log`, 진단용 CSV 또는 붙여넣은 `[Synth 8-XXX]` 메시지를 요청하세요.
2. If RTL is available, inspect the referenced sources directly once a diagnostic is provided.
2. RTL을 사용할 수 있는 경우 진단이 제공되면 참조 소스를 직접 검사하십시오.

## Expected Output
## 예상 출력

- Per-message root cause tied to the actual RTL location
- 실제 RTL 위치와 연결된 메시지별 근본 원인
- Tier classification (T1/T2/T3)
- 계층 분류(T1/T2/T3)
- Recommended RTL-level fixes
- 권장 RTL 수준 수정 사항
- Priority ordering and hotspot summary
- 우선순위 정렬 및 핫스팟 요약
