<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Known-Issue Research
# 알려진 문제 연구

Turn a reproducible failure into a documented cause and a verified workaround
재현 가능한 오류를 문서화된 원인과 검증된 해결 방법으로 전환
before calling it an unfixable tool defect.
그것을 고칠 수 없는 도구 결함이라고 부르기 전에.
Classification alone leaves the
분류만으로는
user blocked.
사용자가 차단되었습니다.

Use `Vivado:vivado_doc_search`.
'Vivado:vivado_doc_search'를 사용하세요.
It searches AMD/Xilinx Answer Records, known
알려진 AMD/Xilinx Answer Records를 검색합니다.
issues, user guides, wikis, and GitHub, and returns source URLs.
이슈, 사용자 가이드, 위키, GitHub를 검색하고 소스 URL을 반환합니다.

## When to research
## 연구할 시기

Always, for:
항상:

- a simulator crash, fatal, or internal error;
- 시뮬레이터 충돌, 치명적 또는 내부 오류;
- an unsupported-construct or not-yet-supported message;
- 지원되지 않는 구성 또는 아직 지원되지 않는 메시지
- a library, link, DPI, or host-environment failure;
- 라이브러리, 링크, DPI 또는 호스트 환경 오류
- a result that differs between XSim and another simulator;
- XSim과 다른 시뮬레이터 간에 결과가 다릅니다.
- a failure that appeared after a tool-version change.
- 도구 버전 변경 후 발생한 오류입니다.

Research before declaring `environment` final, before filing a defect, and
'환경' 최종 선언 전, 결함 접수 전, 연구하고
before telling the user there is no workaround.
사용자에게 해결 방법이 없다고 말하기 전에.

## Build the query
## 쿼리 작성

- Quote the exact message identifier and text, for example
- 예를 들어 정확한 메시지 식별자와 텍스트를 인용하세요.
`XSIM 43-3316 Signal SIGSEGV received`.
'XSIM 43-3316 신호 SIGSEGV 수신됨'.
- Add the distinguishing construct, not just the signature: `mixed language
- 서명뿐만 아니라 구별되는 구문을 추가하십시오. `혼합 언어
hierarchical reference`, `GT cell VHDL and Verilog`, `VHDL 2008 nested
계층적 참조`, `GT 셀 VHDL 및 Verilog`, `VHDL 2008 중첩
record`, `unpacked array concatenation`.
레코드`, `압축해제된 배열 연결`.
- Name the stage: `xvlog`, `xvhdl`, `xelab`, `xsim`, `xsc`, `xcrg`,
- 스테이지 이름 지정: `xvlog`, `xvhdl`, `xelab`, `xsim`, `xsc`, `xcrg`,
`compile_simlib`, `export_simulation`.
`compile_simlib`, `export_simulation`.
- Scope with `tool` and `version` when release behavior matters.
- 릴리스 동작이 중요한 경우 `tool` 및 `version`을 사용하여 범위를 지정하세요.
- Omit `family`
- '가족' 생략
unless the failure is device-specific.
오류가 장치별로 발생하지 않는 한.
- Run two or three differently phrased queries before concluding that nothing
- 아무 것도 없다고 결론을 내리기 전에 두세 가지 다르게 표현된 쿼리를 실행하세요.
is documented.
문서화되어 있습니다.
A signature-only query and a construct-only query return
서명 전용 쿼리 및 구문 전용 쿼리 반환
different results.
다른 결과.

## Read the results
## 결과 읽기

Answer Records and Known Issues are authoritative.
답변 기록 및 알려진 문제는 신뢰할 수 있습니다.
Record the article number,
품목번호를 기록하고,
affected versions, fixed-in release, mechanism, and workaround.
영향을 받는 버전, 수정된 릴리스, 메커니즘 및 해결 방법.
User guides
사용자 가이드
establish supported behavior.
지원되는 동작을 설정합니다.
Wiki, blog, and GitHub results are corroboration
Wiki, 블로그, GitHub 결과는 확증입니다.
only.
만.

One signature can have several documented mechanisms.
하나의 서명에는 여러 문서화된 메커니즘이 있을 수 있습니다.
`XSIM 43-3316` alone
'XSIM 43-3316' 단독
maps to at least mixed-language hierarchical references, a GT cell
최소한 혼합 언어 계층적 참조인 GT 셀에 매핑됩니다.
instantiated from both VHDL and Verilog, and stack overflow from an oversized
VHDL과 Verilog 모두에서 인스턴스화되었으며, 대형 서버에서 스택 오버플로가 발생했습니다.
concatenation.
연결.
The construct in the failing source, not the signal name,
신호 이름이 아닌 실패한 소스의 구성,
selects which one applies.
어느 것이 적용되는지 선택합니다.

Check version applicability before trusting a fix claim.
수정 사항 청구를 신뢰하기 전에 버전 적용 가능성을 확인하세요.
An Answer Record from
의 답변 기록
an older release may still describe the correct mechanism while its "fixed in"
이전 릴리스는 "수정"된 동안에도 여전히 올바른 메커니즘을 설명할 수 있습니다.
statement no longer matches the installed version, especially when the current
명령문은 더 이상 설치된 버전과 일치하지 않습니다. 특히 현재 버전이
failure is a regression.
실패는 퇴행이다.

## Verify before reporting
## 신고하기 전에 확인하세요

A documented workaround is a hypothesis until it runs.
문서화된 해결 방법은 실행되기 전까지는 가설입니다.

1. Extract the candidate workaround.
1. 후보 해결 방법을 추출합니다.
2. Apply it to a copy.
2. 사본에 적용하십시오.
2. Do not modify the user's project without approval.
2. 승인 없이 사용자의 프로젝트를 수정하지 마십시오.
3. Rerun the same bounded test with the same contract.
3. 동일한 계약으로 동일한 제한된 테스트를 다시 실행합니다.
4. Compare against the original first causal error.
4. 원래의 첫 번째 원인 오류와 비교하십시오.
5. Record whether it resolved, partially resolved, or did not apply.
5. 해결되었는지, 부분적으로 해결되었는지, 적용되지 않았는지 기록합니다.

State plainly when a workaround changes semantics, for example forcing a
해결 방법으로 의미가 변경되는 경우(예: 강제 실행)를 명확하게 명시하세요.
single-language flow, disabling an optimization, or splitting an expression.
단일 언어 흐름, 최적화 비활성화 또는 표현식 분할.
The user needs to know what they are trading away.
사용자는 자신이 무엇을 거래하고 있는지 알아야 합니다.

## Prohibited
## 금지

- Presenting an Answer Record workaround as verified without rerunning it.
- 재실행하지 않고 검증된 답변 기록 해결 방법을 제시합니다.
- Downloading or applying a tactical patch or tarball without explicit
- 명시적인 내용 없이 전술 패치나 타르볼을 다운로드하거나 적용하는 행위
approval.
승인.
- Claiming no known issue exists after a single query.
- 단일 쿼리 후에는 알려진 문제가 없다고 주장합니다.
- Letting an older Answer Record override evidence observed in the installed
- 설치된 답변 기록에서 관찰된 증거를 이전 응답 기록보다 우선시하도록 허용
release.
릴리스.
- Inventing an article number or URL.
- 기사 번호나 URL을 만들어내는 행위.
- Cite only what the search returned.
- 검색 결과만 인용하십시오.

## Report
## 신고

```text
Signature and stage:
Search queries used:
Answer Record / document (id + URL):
Affected versions / fixed-in:
Documented mechanism:
Applies to this case: yes | no | partially
Workaround tried:
Result after rerun:
Semantics changed by the workaround:
Remaining risk:
```

## Official sources
## 공식 출처

- [UG900 Logic Simulation](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation)
- [UG900 로직 시뮬레이션](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation)
- [Xilinx Simulation Solution Center (AR 58795)](https://adaptivesupport.amd.com/s/article/58795?language=en_US)
- [Xilinx 시뮬레이션 솔루션 센터(AR 58795)](https://adaptivesupport.amd.com/s/article/58795?언어=en_US)
- [compile_simlib design assistant (AR 63904)](https://adaptivesupport.amd.com/s/article/63904?language=en_US)
- [compile_simlib 디자인 도우미(AR 63904)](https://adaptivesupport.amd.com/s/article/63904?언어=en_US)
