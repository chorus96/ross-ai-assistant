<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# XSim Error Signatures
# XSim 오류 서명

A signature identifies the stage and the first hypothesis.
서명은 단계와 첫 번째 가설을 식별합니다.
It does not
그렇지 않다
identify the root cause.
근본 원인을 식별하십시오.
Pair every signature with the failing construct, then
모든 서명을 실패한 구성과 쌍으로 연결한 다음
research it using
그것을 사용하여 연구
[known-issue-research.md](known-issue-research.md).
[알려진-문제-연구.md](알려진-문제-연구.md).

## Signature table
## 서명 테이블

| Signature<br>서명 | Stage<br>무대 | First hypothesis<br>첫 번째 가설 | Research query seed<br>연구 쿼리 시드 |
|---|---|---|---|
| `[XSIM 43-3316] Signal SIGSEGV received`<br>`[XSIM 43-3316] 신호 SIGSEGV 수신됨` | xelab or runtime<br>xelab 또는 런타임 | front-end or kernel defect on a specific construct<br>특정 구조의 프런트엔드 또는 커널 결함 | signature + construct + `mixed language`<br>서명 + 구문 + '혼합 언어' |
| `[XSIM 43-3294] Signal EXCEPTION_ACCESS_VIOLATION received`<br>`[XSIM 43-3294] 신호 EXCEPTION_ACCESS_VIOLATION 수신됨` | xelab or runtime, Windows<br>xelab 또는 런타임, Windows | Windows-only path of the same class; often stack or memory limit<br>동일한 클래스의 Windows 전용 경로입니다. 종종 스택 또는 메모리 제한 | signature + `Windows` + construct<br>서명 + `Windows` + 구문 |
| `[XSIM 43-3312] Signal SIGABRT received`<br>`[XSIM 43-3312] SIGABRT 신호 수신` | xelab or runtime<br>xelab 또는 런타임 | assertion inside the tool, often VHDL type or bound handling<br>도구 내부의 어설션(종종 VHDL 유형 또는 바운드 처리) | signature + VHDL construct<br>서명 + VHDL 구성 |
| `FATAL_ERROR: Vivado Simulator kernel has discovered an exceptional condition`<br>`치명적인 오류: Vivado 시뮬레이터 커널이 예외 조건을 발견했습니다` | runtime<br>런타임 | kernel defect reached through legal stimulus<br>법적 자극을 통해 도달한 커널 결함 | message + construct + version<br>메시지 + 구성 + 버전 |
| `[XSIM 43-4187] ... is not supported yet for simulation`<br>`[XSIM 43-4187] ... 시뮬레이션에는 아직 지원되지 않습니다` | xvlog or xvhdl<br>xvlog 또는 xvhdl | genuine language-support gap, not a DUT defect<br>DUT 결함이 아닌 실제 언어 지원 격차 | exact construct name + `not supported`<br>정확한 구성 이름 + `지원되지 않음` |
| `[XSIM 43-4452] Linking failed for "xsim.dir/.../dpi.so"`<br>`[XSIM 43-4452] "xsim.dir/.../dpi.so"에 대한 연결 실패` | xsc<br>xsc | host toolchain, gcc version, or missing dev package<br>호스트 툴체인, gcc 버전 또는 누락된 개발 패키지 | signature + `xsc` + distribution<br>서명 + `xsc` + 배포 |
| `[XSIM 43-3254] Could not find hier sig handle for expression`<br>`[XSIM 43-3254] 표현식에 대한 상위 시그니처 핸들을 찾을 수 없습니다.` | runtime or logging<br>런타임 또는 로깅 | hierarchical reference unsupported or optimized away<br>계층적 참조가 지원되지 않거나 최적화되지 않음 | signature + `hierarchical reference`<br>서명 + '계층적 참조' |
| `[XSIM 43-3918] Unable to determine HDL language type of design hierarchy in SDF`<br>`[XSIM 43-3918] SDF에서 디자인 계층 구조의 HDL 언어 유형을 확인할 수 없습니다.` | post-implementation timing<br>구현 후 시기 | SDF scope or language mix at the annotation point<br>주석 지점의 SDF 범위 또는 언어 혼합 | signature + `SDF annotation`<br>서명 + `SDF 주석` |
| `[USF-XSim-62] 'elaborate' step failed with error(s)`<br>`[USF-XSim-62] '정교' 단계가 오류로 인해 실패했습니다.` | xelab<br>xelab | wrapper-level failure; the real cause is in `elaborate.log`<br>래퍼 수준 오류; 진짜 원인은 `elaborate.log`에 있습니다. | read the log first, then research the inner message<br>먼저 로그를 읽고 내부 메시지를 조사하세요. |
| `error while loading shared libraries: libxv_simulator_kernel.so`<br>`공유 라이브러리를 로드하는 중 오류 발생: libxv_simulator_kernel.so` | runtime launch<br>런타임 실행 | environment, not design<br>디자인이 아닌 환경 | run the environment doctor first<br>먼저 환경 의사를 실행하세요 |
| `[VRFC 10-xxxx]`<br>`[VRFC 10-xxxx]` | xvlog or xvhdl<br>xvlog 또는 xvhdl | front-end parse, type, or binding issue<br>프런트 엔드 구문 분석, 유형 또는 바인딩 문제 | exact VRFC id + construct<br>정확한 VRFC ID + 구성 |

`43-3316`, `43-3294`, `43-3312`, and the kernel fatal are the same family seen
`43-3316`, `43-3294`, `43-3312` 및 치명적인 커널은 동일한 계열입니다.
from different host platforms and stages.
다양한 호스트 플랫폼과 단계에서.
Treat them identically: capture the
동일하게 취급: 캡처
stack text, identify the construct, arbitrate across simulators, reduce, and
텍스트 스택, 구성 식별, 시뮬레이터 간 조정, 축소 및
research.
연구.

## Handling rules
## 취급 규칙

A crash is always classified `environment` for the purposes of this skill's
충돌은 이 스킬의 목적에 따라 항상 '환경'으로 분류됩니다.
verdict, because the tool failed rather than the DUT.
DUT보다는 도구가 실패했기 때문입니다.
Never report a crash as
충돌을 다음과 같이 보고하지 마십시오.
a design FAIL, and never edit RTL to dodge one without saying so.
디자인이 실패하고 RTL을 편집하여 그렇게 말하지 않고 회피하지 마십시오.

An unsupported-construct message is also `environment`.
지원되지 않는 구성 메시지도 '환경'입니다.
The source may be
출처는 다음과 같습니다.
fully legal.
완전히 합법적입니다.
Confirm with the LRM or another simulator before suggesting any
제안하기 전에 LRM이나 다른 시뮬레이터를 통해 확인하세요.
source change.
소스 변경.

`USF-XSim-62` and similar wrapper errors carry no diagnostic content.
'USF-XSim-62' 및 유사한 래퍼 오류에는 진단 내용이 없습니다.
Open the
열기
stage log named in the message and quote the first inner error instead.
메시지에 이름이 지정된 스테이지 로그를 작성하고 대신 첫 번째 내부 오류를 인용하세요.

## Workflow from a signature
## 서명의 작업 흐름

1. Capture the complete stage log, tool version banner, and host OS.
1. 전체 단계 로그, 도구 버전 배너 및 호스트 OS를 캡처합니다.
2. Identify the construct at the reported location.
2. 보고된 위치에서 구조물을 식별합니다.
3. Run [cross-simulator arbitration](cross-simulator-arbitration.md) when other
3. 다른 경우 [cross-simulator 중재](cross-simulator-arbitration.md)를 실행합니다.
backends are available.
백엔드를 사용할 수 있습니다.
4. Reduce with [minimal-reproducer.md](minimal-reproducer.md).
4. [minimal-reproducer.md](minimal-reproducer.md)로 줄입니다.
5. Research with [known-issue-research.md](known-issue-research.md).
5. [known-issue-research.md](known-issue-research.md)로 조사하세요.
6. If the failure is new in this release, run
6. 이번 릴리스에서 오류가 새로운 경우 다음을 실행합니다.
[version-bisect.md](version-bisect.md).
[버전-bisect.md](버전-bisect.md).
7. Report cause, workaround, verification result, and residual risk.
7. 원인, 해결 방법, 검증 결과 및 잔여 위험을 보고합니다.

## Official sources
## 공식 출처

- [UG900 Logic Simulation](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation)
- [UG900 로직 시뮬레이션](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation)
- [Xilinx Simulation Solution Center (AR 58795)](https://adaptivesupport.amd.com/s/article/58795?language=en_US)
- [Xilinx 시뮬레이션 솔루션 센터(AR 58795)](https://adaptivesupport.amd.com/s/article/58795?언어=en_US)
