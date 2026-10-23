<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Recorded Results and Method
# 기록결과 및 방법

These measurements accompany the [ISP skill comparison](../README.md).
이러한 측정은 [ISP 기술 비교](../README.md)와 함께 제공됩니다.
They are preserved observations, not promised results for a new run.
이는 보존된 관찰이지 새 실행에 대한 약속된 결과가 아닙니다.
[`measurements.json`](measurements.json) contains the prompt, exact model identifiers, timings, token counters, and every scored check, including failures.
[`measurements.json`](measurements.json)에는 프롬프트, 정확한 모델 식별자, 타이밍, 토큰 카운터 및 실패를 포함한 모든 채점 검사가 포함되어 있습니다.
It omits machine-specific paths, credentials, and model reasoning.
시스템별 경로, 자격 증명 및 모델 추론을 생략합니다.
Source-record and transcript hashes identify the original evidence; the full transcripts are not bundled here.
원본 기록과 사본 해시는 원본 증거를 식별합니다. 전체 성적표는 여기에 포함되어 있지 않습니다.

## How the Recorded Trials Were Run
## 기록된 시험이 실행된 방법

1. **Prepare an empty design.** A harness created a fresh Vivado 2026.1 session and an empty `benchmark_bd` on `xc2ve3558-sfva1440-2MP-e-S`.
1. **빈 디자인을 준비합니다.** 하네스는 `xc2ve3558-sfva1440-2MP-e-S`에 새로운 Vivado 2026.1 세션과 빈 `benchmark_bd`를 생성했습니다.
1. It checked the live version before starting the timed task.
1. 시간이 지정된 작업을 시작하기 전에 라이브 버전을 확인했습니다.
1. No RTL or configured ISP was supplied.
1. RTL 또는 구성된 ISP가 제공되지 않았습니다.
2. **Select the condition.** The skill condition staged `vivado-ip-configurator` and loaded its Tcl helper.
2. **조건을 선택하세요.** 스킬 조건은 `vivado-ip-configurator`를 단계화하고 Tcl 도우미를 로드했습니다.
2. The baseline had no staged skill or helper.
2. 기준선에는 단계별 기술이나 도우미가 없었습니다.
2. Both had Vivado MCP and AMD documentation-search MCP access.
2. 둘 다 Vivado MCP와 AMD 문서 검색 MCP 액세스 권한을 가졌습니다.
3. **Start a fresh agent.** Each used Codex CLI 0.157.1, a separate `CODEX_HOME`, a new conversation, and medium reasoning.
3. **새 에이전트를 시작합니다.** 각각 Codex CLI 0.157.1, 별도의 'CODEX_HOME', 새로운 대화 및 중간 추론을 사용했습니다.
3. The exact model was `gpt-5.6-luna` or `gpt-5.6-sol` through the AMD gateway.
3. 정확한 모델은 AMD 게이트웨이를 통한 'gpt-5.6-luna' 또는 'gpt-5.6-sol'이었습니다.
3. These were Codex CLI experiments, not Copilot CLI or Claude Code experiments.
3. 이는 Copilot CLI나 Claude Code 실험이 아닌 Codex CLI 실험이었습니다.
4. **Submit the same task.** Only the skill-use instruction and fresh session identifier differed within each pair.
4. **동일한 작업을 제출합니다.** 스킬 사용 지침과 새로운 세션 식별자만 각 쌍 내에서 달랐습니다.
4. No parameter names, expected configuration, result table, or previous conversation was supplied.
4. 매개변수 이름, 예상 구성, 결과 테이블 또는 이전 대화가 제공되지 않았습니다.
4. Each model could query documentation and inspect Vivado.
4. 각 모델은 문서를 쿼리하고 Vivado를 검사할 수 있습니다.
5. **Allow one attempt to finish.** The agent could correct errors during that attempt.
5. **한 번의 완료 시도를 허용합니다.** 에이전트는 해당 시도 중에 오류를 수정할 수 있습니다.
5. There was no coaching or second attempt based on its final score.
5. 최종 점수에 따른 코칭이나 두 번째 시도는 없었습니다.
5. Elapsed model time includes tool calls and recovery.
5. 경과된 모델 시간에는 도구 호출 및 복구가 포함됩니다.
5. It excludes setup and post-run evaluation.
5. 설정 및 실행 후 평가는 제외됩니다.
6. **Grade independently.** The harness read effective values, including defaults, and actual interface pins.
6. **독립적으로 등급 지정.** 하네스는 기본값과 실제 인터페이스 핀을 포함한 유효 값을 읽습니다.
6. It checked the result against the original request, rather than trusting the model's self-report.
6. 모델의 자체 보고를 신뢰하기보다는 원래 요청과 비교하여 결과를 확인했습니다.
6. Feedback from this independent grader was not returned to the model.
6. 이 독립 채점자의 피드백은 모델에 반환되지 않았습니다.
6. All sessions were stopped after evaluation.
6. 평가 후 모든 세션이 중지되었습니다.

Both Luna runs selected `xilinx.com:ip:visp_ss:2.0`.
두 Luna는 모두 선택된 `xilinx.com:ip:visp_ss:2.0`을 실행합니다.
Sol with the skill selected that subsystem; Sol without the skill selected `xilinx.com:ip:visp:2.0`, the lower-level hardened ISP.
해당 서브시스템을 선택한 스킬을 가진 솔; Sol은 기술 없이 하위 수준으로 강화된 ISP인 `xilinx.com:ip:visp:2.0`을 선택했습니다.
The prompt did not require the wrapper.
프롬프트에는 래퍼가 필요하지 않았습니다.
The common checks accept either when it realizes the requested interface and configuration.
일반적인 검사에서는 요청된 인터페이스와 구성을 인식할 때 둘 중 하나를 수락합니다.

### Isolation
### 격리

The trials ran sequentially on the same Linux host.
평가판은 동일한 Linux 호스트에서 순차적으로 실행되었습니다.
Every completed trial had a separate agent workspace, conversation, and Vivado session.
완료된 모든 평가판에는 별도의 상담원 작업 공간, 대화 및 Vivado 세션이 있었습니다.

For **Luna**, both the agent and its dedicated MCP/Vivado server were sandboxed with bubblewrap.
**Luna**의 경우 에이전트와 전용 MCP/Vivado 서버 모두 버블랩으로 샌드박스 처리되었습니다.
The host filesystem was read-only, `/home` was hidden except for the trial's writable workspace and the skill mount in the enabled condition, and PID, temporary, and runtime directories were separated.
호스트 파일 시스템은 읽기 전용이었고, 체험판의 쓰기 가능한 작업 공간과 활성화된 조건의 스킬 마운트를 제외하고 '/home'은 숨겨졌으며, PID, 임시, 런타임 디렉터리는 분리되었습니다.
Each trial had a separate server process and private discovery/session state.
각 평가판에는 별도의 서버 프로세스와 비공개 검색/세션 상태가 있었습니다.
Environment variables were allowlisted, and only the server received a read-only license configuration file.
환경 변수가 허용 목록에 추가되었으며 서버만 읽기 전용 라이선스 구성 파일을 받았습니다.
Live Tcl probes confirmed that repository tests, the grader, and previous Sol results were inaccessible.
라이브 Tcl 프로브는 저장소 테스트, 그레이더 및 이전 Sol 결과에 액세스할 수 없음을 확인했습니다.

The earlier **Sol** pair had agent-side filesystem separation, fresh Vivado sessions, and a run-specific MCP server, but that server could access the host filesystem and was shared sequentially between the two conditions.
이전 **Sol** 쌍에는 에이전트 측 파일 시스템 분리, 새로운 Vivado 세션 및 실행별 MCP 서버가 있었지만 해당 서버는 호스트 파일 시스템에 액세스할 수 있었고 두 조건 간에 순차적으로 공유되었습니다.
Its recorded tool calls showed no answer-key or other-trial reads, but the setup did not prevent all such access.
기록된 도구 호출에는 답변 키나 기타 시험 읽기가 표시되지 않았지만 설정으로 인해 이러한 액세스가 모두 차단되지는 않았습니다.
The Sol rows therefore are not a perfectly controlled isolation-matched comparison with Luna.
따라서 Sol 행은 Luna와 완벽하게 제어된 격리 일치 비교가 아닙니다.

Neither setup was a separate VM: the host kernel, installed tools, and network were shared.
두 설정 모두 별도의 VM이 아니었습니다. 호스트 커널, 설치된 도구 및 네트워크가 공유되었습니다.
These are documented experimental boundaries, not a claim of absolute isolation.
이는 문서화된 실험적 경계이지 절대적인 격리를 주장하는 것이 아닙니다.

### Setup and Grading Corrections
### 설정 및 등급 수정

- An early Sol skill attempt was interrupted when its MCP server received a termination signal.
- MCP 서버가 종료 신호를 수신했을 때 초기 솔 스킬 시도가 중단되었습니다.
- Shared discovery state had allowed IDE clients to connect.
- 공유 검색 상태로 인해 IDE 클라이언트 연결이 허용되었습니다.
- That attempt was excluded as an infrastructure abort, preserved, and replaced after fixing discovery isolation.
- 해당 시도는 인프라 중단으로 제외되고 검색 격리를 수정한 후 보존 및 교체되었습니다.
- There was one completed skill attempt used in the table.
- 표에는 완료된 스킬 시도가 1회 사용되었습니다.
- Luna's first preparation failed because the sandbox hid its license configuration.
- 샌드박스가 라이선스 구성을 숨겨서 Luna의 첫 번째 준비가 실패했습니다.
- Mounting that file read-only fixed setup before either task-model attempt; no Luna task attempt was repeated.
- 작업 모델을 시도하기 전에 해당 파일을 읽기 전용 고정 설정으로 마운트합니다. Luna 작업 시도가 반복되지 않았습니다.
- Sol's initial grader assumed the subsystem IP and its pin naming.
- Sol의 초기 그레이더는 하위 시스템 IP와 해당 핀 이름을 가정했습니다.
- It was corrected to recognize the lower-level ISP and its different data-pin names.
- 하위 ISP와 그에 따른 다양한 데이터핀 이름을 인식하도록 수정되었습니다.
- Two wrapper-only default checks were removed from both Sol scores, leaving 113 common checks.
- 2개의 래퍼 전용 기본 검사가 두 Sol 점수 모두에서 제거되어 113개의 공통 검사가 남았습니다.
- Regrading used saved designs and captured evidence; the model designs were not repaired.
- 사용된 저장된 디자인과 캡처된 증거를 재등급화합니다. 모델 디자인은 수리되지 않았습니다.
- Luna used this corrected contract unchanged from the start.
- 루나는 처음부터 수정된 계약을 그대로 사용했습니다.

## What the Score Measures
## 점수가 측정하는 것

The 113 checks cover IP identity, enabled tiles, independent-input mode, live-input counts, disabled multi-pass settings, active input formats and pixels per clock, interface presence/protocol/direction, physical data-pin widths, shared input-3 settings, selected default output widths, and saved-design presence.
113개 검사에는 IP ID, 활성화된 타일, 독립 입력 모드, 실시간 입력 개수, 비활성화된 다중 패스 설정, 활성 입력 형식 및 클록당 픽셀, 인터페이스 존재/프로토콜/방향, 물리적 데이터 핀 너비, 공유 입력 3 설정, 선택된 기본 출력 너비 및 저장된 디자인 존재 여부가 포함됩니다.

Some checks overlap: one missing interface causes its protocol, direction, and width checks to fail.
일부 검사는 중복됩니다. 하나의 누락된 인터페이스로 인해 프로토콜, 방향 및 너비 검사가 실패합니다.
The score is useful for comparing these runs, but is not a probability of correctness.
점수는 이러한 실행을 비교하는 데 유용하지만 정확성의 확률은 아닙니다.
A pass requires all checks; 84% is still a failed configuration.
패스에는 모든 확인이 필요합니다. 84%는 여전히 실패한 구성입니다.

The independent checks found:
발견된 독립적인 검사는 다음과 같습니다.

- **Luna with skill: 18 failed checks.** Eight first-ISP paths on tiles 0 and 2 had incorrect 16-bit formats and eight corresponding ports were 64 instead of 48 bits wide.
- **기술을 갖춘 Luna: 18번의 검사 실패.** 타일 0과 2의 첫 번째 ISP 경로 8개에는 잘못된 16비트 형식이 있었고 해당 포트 8개는 너비가 48비트가 아닌 64였습니다.
- Two shared input-3 format checks also failed.
- 두 개의 공유 입력-3 형식 검사도 실패했습니다.
- All 15 interfaces were present.
- 15개의 인터페이스가 모두 존재했습니다.
- **Luna without skill: 46 failed checks.** All three tiles remained in Basic mode, nine independent input interfaces were absent, and active input formats/widths did not satisfy the requested RAW12 paths.
- **기술이 없는 루나: 46번의 검사 실패.** 타일 3개 모두 기본 모드에 남아 있었고, 9개의 독립적인 입력 인터페이스가 없었고, 활성 입력 형식/폭이 요청된 RAW12 경로를 충족하지 못했습니다.
- The six visible inputs were all 64 bits wide, including tile 1 inputs reported by the model as RAW12.
- 모델에서 RAW12로 보고된 타일 1 입력을 포함하여 표시되는 6개의 입력은 모두 64비트 너비였습니다.
- **Sol, both conditions: zero failed checks.** This does not mean the designs were identical.
- **솔, 두 조건 모두: 실패한 검사가 0개입니다.** 이는 설계가 동일하다는 의미는 아닙니다.
- They used different IP layers and differed in additional defaults, including virtual-channel values and clock precision.
- 그들은 서로 다른 IP 계층을 사용했으며 가상 채널 값 및 시계 정밀도를 포함한 추가 기본값이 달랐습니다.
- Full-system wiring and hardware operation were not tested.
- 전체 시스템 배선 및 하드웨어 작동은 테스트되지 않았습니다.

### Why the Skill's Checker Did Not Guarantee Success
### 스킬 체커가 성공을 보장하지 않는 이유

The skill has error-recovery helpers and requires an intent audit.
이 기술에는 오류 복구 도우미가 있으며 의도 감사가 필요합니다.
In the Luna skill trace, the agent used the helpers to recover from rejected configuration, but did not call `verify_intent` or `audit_intent`.
Luna 스킬 추적에서 에이전트는 도우미를 사용하여 거부된 구성을 복구했지만 'verify_intent' 또는 'audit_intent'를 호출하지 않았습니다.
It manually read values back and accepted the wrong per-tile interpretation of the user's requirement.
수동으로 값을 다시 읽고 사용자 요구 사항에 대한 잘못된 타일별 해석을 수락했습니다.

Even a correctly invoked checker relies on the agent supplying a correct requirement-to-parameter mapping.
올바르게 호출된 검사기라도 올바른 요구사항-매개변수 매핑을 제공하는 에이전트에 의존합니다.
Comparing live values against incorrectly chosen expected values can pass.
잘못 선택된 예상 값과 실제 값을 비교하면 통과할 수 있습니다.
This example demonstrates both the benefit of guidance on port-enablement dependencies and the need for independent acceptance criteria.
이 예는 포트 활성화 종속성에 대한 지침의 이점과 독립적인 승인 기준의 필요성을 모두 보여줍니다.

## Time, Tool Calls, and Tokens
## 시간, 도구 호출 및 토큰

| Luna metric<br>루나 미터법 | With skill<br>실력으로 | Without skill<br>기술이 없으면 |
|---|---:|---:|
| Model time<br>모델 시간 | 241.44 s | 214.42 s |
| Setup through grading<br>채점을 통한 설정 | 250.62 s | 223.62 s |
| Tcl batches<br>Tcl 배치 | 13 | 26 |
| Nonzero Tcl results<br>0이 아닌 Tcl 결과 | 2 | 6 |
| Native Vivado ERROR lines<br>네이티브 Vivado ERROR 라인 | 6 | 22 |
| Documentation searches<br>문서 검색 | 5 | 2 |
| Total input tokens<br>총 입력 토큰 | 1,662,423 | 1,626,525 |
| Cached input, included above<br>위에 포함된 캐시된 입력 | 1,580,728 | 1,562,459 |
| Non-cached input<br>캐시되지 않은 입력 | 81,695 | 64,066 |
| Output tokens<br>출력 토큰 | 10,505 | 7,547 |

Token counts are cumulative CLI-reported usage over repeated model calls.
토큰 수는 반복적인 모델 호출에 대한 누적 CLI 보고 사용량입니다.
They are not the size of one prompt; most input was cached.
하나의 프롬프트 크기가 아닙니다. 대부분의 입력이 캐시되었습니다.
Reasoning output is included in the reported output total and is not added again.
추론 출력은 보고된 출력 총계에 포함되며 다시 추가되지 않습니다.
Native ERROR lines and failed Tcl batches are different measurements: one rejected operation can emit several messages, and helpers can capture a failure while the outer Tcl call returns successfully.
기본 ERROR 줄과 실패한 Tcl 배치는 서로 다른 측정값입니다. 하나의 거부된 작업이 여러 메시지를 내보낼 수 있으며 도우미는 외부 Tcl 호출이 성공적으로 반환되는 동안 실패를 캡처할 수 있습니다.

### Illustrative Model Cost
### 예시적인 모델 비용

Using the repository's configured Luna rates at the time of analysis gives approximately **$0.0646 with the skill** and **$0.0563 without it**: about $0.0083, or 15%, more for the skill condition.
분석 당시 리포지토리에 구성된 Luna 요율을 사용하면 스킬이 있는 경우 약 $0.0646**, 스킬이 없는 경우 **$0.0563**: 스킬 조건의 경우 약 $0.0083 또는 15%가 더 많습니다.

The rates used were $0.20 per million ordinary input tokens, $0.02 for cache reads, $0.25 for cache writes, and $1.20 for output.
사용된 요금은 일반 입력 토큰 백만 개당 0.20달러, 캐시 읽기의 경우 0.02달러, 캐시 쓰기의 경우 0.25달러, 출력의 경우 1.20달러였습니다.
The calculation treats cache reads and writes as subsets of total input: `ordinary input = total input − cache reads − cache writes`.
계산에서는 캐시 읽기 및 쓰기를 총 입력의 하위 집합으로 처리합니다('일반 입력 = 총 입력 − 캐시 읽기 − 캐시 쓰기').
The recorded cache-write counts were 81,599 and 63,958 respectively.
기록된 캐시 쓰기 횟수는 각각 81,599 및 63,958이었습니다.

These are **estimates from local configuration, not verified AMD gateway charges or a current price quote**.
이는 **로컬 구성의 추정치이며 확인된 AMD 게이트웨이 요금이나 현재 가격 견적이 아닙니다**.
They exclude preflight, local compute, licenses, and MCP service costs.
비행 전, 로컬 컴퓨팅, 라이선스 및 MCP 서비스 비용은 제외됩니다.
The rates and arithmetic inputs are preserved in `measurements.json`; substitute your actual billing terms before making a cost decision.
비율과 산술 입력은 `measurements.json`에 보존됩니다. 비용 결정을 내리기 전에 실제 청구 조건을 대체하세요.

## Interpreting the Value
## 가치 해석

For Luna, the skill condition delivered the required independent ports and improved the score by about 25 percentage points with modest additional time and estimated model cost.
Luna의 경우 기술 조건은 필요한 독립 포트를 제공했으며 적절한 추가 시간과 예상 모델 비용으로 점수를 약 25% 포인트 향상시켰습니다.
It still failed the overall task.
여전히 전체 작업에 실패했습니다.
For Sol, direct MCP use already achieved 100% and was faster.
Sol의 경우 직접 MCP 사용이 이미 100%를 달성했으며 속도도 더 빨랐습니다.

The demonstrated value is therefore specific to the model and requirement.
따라서 입증된 값은 모델 및 요구 사항에 따라 다릅니다.
Use this example to understand the workflow and failure modes, then measure repeated trials on your own tasks before making a reliability or cost claim.
이 예를 사용하여 작업흐름과 실패 모드를 이해한 다음 신뢰성이나 비용 청구를 하기 전에 자신의 작업에 대한 반복적인 시도를 측정하십시오.
