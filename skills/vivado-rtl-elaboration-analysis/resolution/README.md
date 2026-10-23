<!-- ko-translation: enabled -->
# Elaboration Resolution Guides
# 정교화 해결 가이드

Per-message fix references for Vivado synthesis elaboration (`[Synth 8-XXX]`).
Vivado 합성 정교화를 위한 메시지별 수정 참조(`[Synth 8-XXX]`).
Each guide provides the Vivado message pattern, root cause analysis, fix options
각 가이드는 Vivado 메시지 패턴, 근본 원인 분석, 수정 옵션을 제공합니다.
with before/after code examples, and verification steps.
이전/이후 코드 예시 및 확인 단계를 포함합니다.

## How to Use
## 사용방법

When processing an elaboration message:
상세 메시지를 처리하는 경우:

1. Look up the `msg_id` in the table below
1. 아래 표에서 `msg_id`를 찾아보세요.
2. Load the matching guide with `read_file`
2. `read_file`을 사용하여 일치하는 가이드를 로드합니다.
3. Follow fix instructions, replacing placeholders with actual design names
3. 수정 지침을 따르고 자리 표시자를 실제 디자인 이름으로 바꿉니다.
4. Include the rationale in the generated report
4. 생성된 보고서에 근거를 포함합니다.

## Individual Guides
## 개별 가이드

Complex messages with multiple root cause patterns warranting dedicated files.
전용 파일을 보장하는 여러 근본 원인 패턴이 있는 복잡한 메시지입니다.

| Guide<br>가이드 | Message IDs<br>메시지 ID | Severity<br>심각도 | Description<br>설명 |
|-------|------------|----------|-------------|
| [VLOG-689](VLOG-689.md)<br>[VLOG-689](VLOG-689.md) | 689, 7129 | WARNING<br>경고 | Port width mismatch<br>포트 폭 불일치 |

## Grouped Guides
## 그룹화된 가이드

Messages sharing a common fix pattern, collected into a single file.
공통 수정 패턴을 공유하는 메시지는 단일 파일로 수집됩니다.

| Guide<br>가이드 | Message IDs<br>메시지 ID | Description<br>설명 |
|-------|------------|-------------|
| [case-statement](case-statement.md)<br>[사례 진술](case-statement.md) | 506 | Case statement issues<br>사례 진술 문제 |
| [range-index](range-index.md)<br>[범위-색인](범위-색인.md) | 547, 549 | Range and index errors<br>범위 및 색인 오류 |
| [constant-expr](constant-expr.md)<br>[상수-expr](상수-expr.md) | 647 | Constant expression errors<br>상수 표현 오류 |
| [clock-edge](clock-edge.md)<br>[시계 가장자리](clock-edge.md) | 561 | Clock edge and event control issues<br>클록 에지 및 이벤트 제어 문제 |

## Adding New Guides
## 새 가이드 추가

Create `<VLOG-XXX>.md` or `<VHDL-XXX>.md` (individual) or `<topic>.md` (grouped) with:
다음을 사용하여 `<VLOG-XXX>.md` 또는 `<VHDL-XXX>.md`(개별) 또는 `<topic>.md`(그룹화)를 만듭니다.

1. **Vivado Message** — exact `[Synth 8-XXX]` pattern
1. **Vivado 메시지** — 정확한 `[Synth 8-XXX]` 패턴
2. **Root Cause** — why Vivado emits this
2. **근본 원인** — Vivado가 이것을 내보내는 이유
3. **Fix Options** — before/after code in diff format
3. **수정 옵션** — diff 형식의 코드 이전/이후

See any existing guide as a template.
기존 가이드를 템플릿으로 확인하세요.
