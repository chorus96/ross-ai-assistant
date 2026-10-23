<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# vivado-simulate-rtl
# vivado-시뮬레이트-rtl

Runs and diagnoses FPGA RTL simulation through Vivado.
Vivado를 통해 FPGA RTL 시뮬레이션을 실행하고 진단합니다.
It launches a bounded
그것은 제한된
compile–elaborate–simulate flow, requires an explicit test contract and
컴파일-정교-시뮬레이션 흐름에는 명시적인 테스트 계약이 필요하며
terminal PASS, classifies failures, and preserves transcripts, waveforms, and
터미널 PASS, 오류 분류, 기록, 파형 및
coverage or activity artifacts.
적용 범위 또는 활동 아티팩트.
A clean compile or simulator exit is not
클린 컴파일 또는 시뮬레이터 종료는 그렇지 않습니다.
functional proof.
기능적 증명.

## Supported simulators
## 지원되는 시뮬레이터

Vivado backends (canonical `export_simulation` / `target_simulator` IDs):
Vivado 백엔드(표준 `export_simulation` / `target_simulator` ID):

| Simulator<br>시뮬레이터 | ID | Host<br>호스트 |
|---|---|---|
| AMD XSim<br>AMD XSim | `xsim`<br>`xsim` | Linux, Windows<br>리눅스, 윈도우 |
| Siemens Questa<br>지멘스 퀘스타 | `questa`<br>'퀘스타' | Linux, Windows<br>리눅스, 윈도우 |
| Siemens ModelSim DE<br>지멘스 모델Sim DE | `modelsim`<br>'모델심' | Linux, Windows<br>리눅스, 윈도우 |
| Synopsys VCS<br>시놉시스 VCS | `vcs`<br>`vcs` | Linux<br>리눅스 |
| Cadence Xcelium<br>케이던스 엑셀리움 | `xcelium`<br>'엑스셀륨' | Linux<br>리눅스 |
| Aldec Riviera-PRO<br>알덱 리비에라-PRO | `riviera`<br>'리비에라' | Linux, Windows<br>리눅스, 윈도우 |
| Aldec Active-HDL<br>Aldec Active-HDL | `activehdl`<br>'활성HDL' | Windows<br>윈도우 |

XSim is the default.
XSim이 기본값입니다.
Do not silently substitute another backend.
다른 백엔드를 자동으로 대체하지 마세요.
Third-party
제3자
tools need a site install, license, and version-matched `compile_simlib`
도구에는 사이트 설치, 라이센스 및 버전 일치 `compile_simlib`가 필요합니다.
libraries.
도서관.
Per-backend setup is in [references/](references/).
백엔드별 설정은 [references/](references/)에 있습니다.

On the Ubuntu 24.04 validation host, behavioral smoke passed for XSim, Questa, VCS, and
Ubuntu 24.04 검증 호스트에서 XSim, Questa, VCS 및
Xcelium.
엑셀리움.
ModelSim and Riviera-PRO were license-blocked.
ModelSim과 Riviera-PRO는 라이센스가 차단되었습니다.
Active-HDL is
활성 HDL은
Windows-only.
Windows 전용.
Ubuntu VCS needs a bash `VCS_HOME` overlay; see
Ubuntu VCS에는 bash `VCS_HOME` 오버레이가 필요합니다. 참조
[references/vcs.md](references/vcs.md).
[참조/vcs.md](참조/vcs.md).

## Example prompts
## 예시 프롬프트

**Behavioral simulation**
**행동 시뮬레이션**
> Open this Vivado project, run `sim_1` with XSim, and prove whether the
> 이 Vivado 프로젝트를 열고 XSim으로 `sim_1`을 실행하고
> testbench passes.
> 테스트벤치를 통과했습니다.
> Quote the terminal marker and give the transcript and
> 터미널 마커를 인용하고 성적표를 제공하고
> waveform paths.
> 파형 경로.

**Third-party backend**
**타사 백엔드**
> Use the configured VCS install and compiled libraries to run `sim_1`.
> 구성된 VCS 설치 및 컴파일된 라이브러리를 사용하여 `sim_1`을 실행합니다.
> Do not
> 하지 마십시오
> fall back to XSim.
> XSim으로 돌아갑니다.
> Report `TEST_PASS` or the first causal error.
> 'TEST_PASS' 또는 첫 번째 원인 오류를 보고하세요.

**Failure diagnosis**
**실패 진단**
> This testbench is failing.
> 이 테스트벤치는 실패했습니다.
> Diagnose the first assertion or mismatch, classify
> 첫 번째 주장 또는 불일치를 진단하고 분류합니다.
> it as design, testbench, environment, or timeout, and do not edit the DUT.
> 이를 설계, 테스트벤치, 환경 또는 시간 제한으로 설정하고 DUT를 편집하지 마십시오.

**Self-checking testbench**
**자체 점검 테스트벤치**
> Add a bounded, self-checking testbench for this DUT with a watchdog and a
> 워치독 및
> single `TEST_PASS` / `TEST_FAIL` marker, then run it in XSim.
> 단일 `TEST_PASS` / `TEST_FAIL` 마커를 선택한 다음 XSim에서 실행하세요.

**Coverage closure**
**보험 적용 종료**
> Close XSim statement, branch, condition, and toggle coverage on this
> XSim 문, 분기, 조건을 닫고 이에 대한 적용 범위를 전환합니다.
> regression.
> 회귀.
> Do not exclude reachable logic.
> 도달 가능한 논리를 제외하지 마십시오.
> Show before/after reports.
> 보고서 전후를 표시합니다.

**UVM and seeded regression**
**UVM 및 시드 회귀**
> Run the selected UVM test across these seeds.
> 이러한 시드에 대해 선택한 UVM 테스트를 실행합니다.
> Replay any failing seed exactly
> 실패한 시드를 정확하게 재생
> before diagnosing.
> 진단하기 전에.
> Keep a manifest of plusargs and results per seed.
> plusargs의 매니페스트와 시드당 결과를 유지합니다.

**Assertions and functional coverage**
**어설션 및 기능 적용 범위**
> Run the assertion-aware test, report passed/failed/disabled assertions, and
> 어설션 인식 테스트를 실행하고 통과/실패/비활성화된 어설션을 보고합니다.
> show which functional coverage bins remain uncovered.
> 어떤 기능적 적용 범위가 노출되지 않은 상태로 남아 있는지 보여줍니다.

**Post-implementation timing**
**구현 후 시기**
> After implementation completes, run post-implementation timing simulation of
> 구현이 완료되면 구현 후 타이밍 시뮬레이션을 실행합니다.
> `sim_1` with SDF.
> SDF가 포함된 `sim_1`.
> Prove the functional result; do not treat timing closure as
> 기능적 결과를 증명합니다. 타이밍 클로저를 다음과 같이 취급하지 마십시오.
> simulation PASS.
> 시뮬레이션 패스.

**Activity capture**
**활동 캡처**
> Capture SAIF (or VCD) for this hierarchy after reset, over a bounded
> 제한된 범위에서 재설정 후 이 계층 구조에 대한 SAIF(또는 VCD)를 캡처합니다.
> measurement window.
> 측정 창.
> Require an independent functional PASS before treating
> 치료 전 독립적인 기능적 PASS가 필요합니다.
> the dump as valid.
> 덤프가 유효한 것으로 간주됩니다.

**Portable stimulus**
**휴대용 자극**
> Generate a portable sub-design testbench from this simulation, add an oracle
> 이 시뮬레이션에서 휴대용 하위 설계 테스트벤치를 생성하고 Oracle을 추가합니다.
> and watchdog, and prove replay PASS.
> 및 워치독을 수행하고 리플레이 패스를 증명합니다.

**Guided interactive repair**
**안내 대화형 수리**
> Debug from the waveform first.
> 먼저 파형에서 디버그하세요.
> Stop for an explicit repair / more-evidence /
> 명시적인 수리를 위해 중지 / 추가 증거 /
> stop decision, and do not change DUT RTL until I approve.
> 결정을 중지하고 승인할 때까지 DUT RTL을 변경하지 마십시오.

**Missing third-party setup**
**타사 설정 누락**
> I want Questa for `sim_1`, but the runtime or compiled libraries are not
> `sim_1`용 Questa를 원하지만 런타임이나 컴파일된 라이브러리가 없습니다.
> configured.
> 구성되었습니다.
> Stop before changing `target_simulator` and tell me what is
> `target_simulator`를 변경하기 전에 중지하고 무엇인지 알려주세요.
> missing.
> 실종.

**Simulator crash with a documented workaround**
**문서화된 해결 방법으로 인한 시뮬레이터 충돌**
> `xelab` dies with `[XSIM 43-3316] Signal SIGSEGV received` but Questa
> `xelab`은 `[XSIM 43-3316] 신호 SIGSEGV 수신`으로 종료되지만 Questa
> elaborates the same sources.
> 동일한 소스를 자세히 설명합니다.
> Find the Answer Record, try the workaround, and
> 답변 기록을 찾아 해결 방법을 시도해 보세요.
> tell me whether it actually fixed it and what it costs.
> 실제로 문제가 해결되었는지 여부와 비용은 얼마인지 알려주세요.

**Design defect or simulator defect**
**설계 결함 또는 시뮬레이터 결함**
> XSim and VCS disagree on this testbench.
> XSim과 VCS는 이 테스트벤치에 동의하지 않습니다.
> Run both on identical sources and
> 동일한 소스에서 둘 다 실행하고
> settings and tell me which one is wrong, with a citation.
> 설정을 확인하고 인용과 함께 어느 것이 잘못되었는지 알려주세요.

**Reportable defect package**
**보고 가능한 결함 패키지**
> There is no workaround.
> 해결 방법이 없습니다.
> Reduce this to the smallest case that keeps the same
> 이것을 동일하게 유지하는 가장 작은 케이스로 줄이세요.
> signature and give me a package I can file.
> 서명하고 제가 제출할 수 있는 패키지를 주세요.

**Host preflight**
**호스트 프리플라이트**
> Simulation will not start on this machine.
> 이 컴퓨터에서는 시뮬레이션이 시작되지 않습니다.
> Check the host, libraries,
> 호스트, 라이브러리를 확인하세요.
> toolchain, and license before running anything.
> 무엇이든 실행하기 전에 툴체인 및 라이센스를 확인하세요.

**Regression bisect**
**회귀 이등분**
> This passed in an older Vivado.
> 이것은 더 오래된 Vivado에서 전달되었습니다.
> Find the release that broke it and give me a
> 문제가 발생한 릴리스를 찾아 나에게 알려주세요.
> fallback.
> 대체.

**Performance**
**성능**
> Our XSim run is slow.
> XSim 실행이 느립니다.
> Measure where the time goes per stage before changing
> 변경하기 전에 단계별로 시간이 어디로 가는지 측정
> anything.
> 무엇이든.

**Headless evidence**
**머리 없는 증거**
> No display on this CI box.
> 이 CI 상자에는 표시되지 않습니다.
> Prove the regression from the transcript and a
> 성적표로부터 회귀를 증명하고
> VCD diff, and tell me the first divergence time.
> VCD 차이를 확인하고 첫 번째 분기 시간을 알려주세요.

## Host check
## 호스트 확인

```bash
python3 <skill-dir>/scripts/sim_env_doctor.py \
  --vivado <path-to-vivado> --clibs <compiled-library-dir> --simulator <id>
```

Read-only.
읽기 전용입니다.
Reports Vivado and XSim tool presence, simulator-kernel library
Vivado 및 XSim 도구 존재 여부, 시뮬레이터 커널 라이브러리 보고
resolution, DPI toolchain, `/bin/sh` implementation, compiled-library
해상도, DPI 툴체인, `/bin/sh` 구현, 컴파일된 라이브러리
contents, license variables, workspace writability, and display availability.
콘텐츠, 라이센스 변수, 작업 공간 쓰기 가능성 및 디스플레이 가용성.
Exits nonzero on a hard failure.
심각한 오류가 발생하면 0이 아닌 값으로 종료됩니다.

## Evaluations
## 평가

The evaluation suite (scenarios, fixtures, and graders) is maintained with the
평가 세트(시나리오, 고정 장치 및 채점자)는 다음과 같이 유지됩니다.
skill's test suites in its source repository, not in the distributed skill.
스킬의 테스트 스위트는 분산 스킬이 아닌 소스 저장소에 있습니다.
