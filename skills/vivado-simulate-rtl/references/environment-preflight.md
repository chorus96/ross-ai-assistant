<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Environment Preflight
# 환경 프리플라이트

Catch host, library, license, and toolchain problems before spending a long
오랜 시간을 투자하기 전에 호스트, 라이브러리, 라이센스 및 툴체인 문제를 파악하세요.
simulation on them.
그들에 대한 시뮬레이션.
Many reported simulation failures never reach the design.
보고된 많은 시뮬레이션 실패는 설계에 도달하지 못합니다.

## Run the doctor
## 의사를 실행

The doctor ships with this skill at `scripts/sim_env_doctor.py`, next to
의사는 옆에 있는 `scripts/sim_env_doctor.py`에 이 기술을 제공합니다.
this `references/` directory.
이 `references/` 디렉토리입니다.
Run it from the skill directory's location in
다음의 스킬 디렉터리 위치에서 실행하세요.
the current agent's install; there is no need to search for it.
현재 에이전트의 설치; 그것을 검색할 필요가 없습니다.

```bash
python3 <skill-dir>/scripts/sim_env_doctor.py --json
```

Useful options:
유용한 옵션:

- `--vivado <path>` to check a specific install instead of `PATH`;
- `PATH` 대신 특정 설치를 확인하려면 `--vivado <path>`;
- `--clibs <dir>` to validate a compiled-library directory;
- `--clibs <dir>`은 컴파일된 라이브러리 디렉토리를 검증합니다.
- `--simulator <id>` to check a third-party backend's license variables.
- 타사 백엔드의 라이선스 변수를 확인하려면 `--simulator <id>`를 사용하세요.

The script is read-only.
스크립트는 읽기 전용입니다.
It exits nonzero when a hard failure is found and
심각한 오류가 발견되면 0이 아닌 값으로 종료됩니다.
prints each check as `pass`, `warn`, or `fail` with the evidence it used.
사용된 증거와 함께 각 검사를 '통과', '경고' 또는 '실패'로 인쇄합니다.

## What it checks
## 검사 내용

- Vivado and XSim executables resolve, and the reported version banner.
- Vivado 및 XSim 실행 파일이 해결되고 보고된 버전 배너가 표시됩니다.
- Shared-library closure of the simulator kernel, so a missing `.so` is named
- 시뮬레이터 커널의 공유 라이브러리 폐쇄로 인해 누락된 '.so' 이름이 지정됩니다.
before launch rather than at runtime.
런타임이 아닌 실행 전.
- `LD_LIBRARY_PATH` entries that exist and are readable.
- 존재하고 읽을 수 있는 `LD_LIBRARY_PATH` 항목입니다.
- Host C/C++ toolchain presence and version for DPI and `xsc`.
- DPI 및 `xsc`에 대한 호스트 C/C++ 툴체인 존재 및 버전.
- `/bin/sh` implementation, which breaks Synopsys wrappers when it is dash.
- 대시인 경우 Synopsys 래퍼를 중단시키는 `/bin/sh` 구현입니다.
- Compiled-library directory existence, contents, and any recorded tool
- 컴파일된 라이브러리 디렉토리 존재, 내용 및 기록된 도구
version, so a `compile_simlib` mismatch is visible.
버전이므로 `compile_simlib` 불일치가 표시됩니다.
- License environment variables for the selected third-party simulator.
- 선택한 타사 시뮬레이터의 라이선스 환경 변수입니다.
- Writable working directory and free space for waveform and coverage output.
- 쓰기 가능한 작업 디렉토리와 파형 및 커버리지 출력을 위한 여유 공간.
- Display availability, which matters only for GUI requests.
- GUI 요청에만 중요한 표시 가용성입니다.

## Interpreting results
## 결과 해석

A `fail` is an `environment` classification with evidence attached.
'실패'는 증거가 첨부된 '환경' 분류입니다.
Report the
보고하다
exact missing prerequisite.
정확히 누락된 전제 조건.
Do not edit RTL, and do not substitute another
RTL을 편집하지 말고 다른 것으로 대체하지 마십시오.
simulator to route around it.
주위를 라우팅하는 시뮬레이터입니다.

A compiled-library mismatch is the most common silent cause of strange
컴파일된 라이브러리 불일치는 이상한 오류가 발생하는 가장 일반적인 원인입니다.
elaboration errors.
정교 오류.
Libraries must be recompiled after any Vivado version or
라이브러리는 Vivado 버전 이후에 다시 컴파일되어야 합니다.
update change, and after installing a new third-party simulator.
업데이트 변경 및 새로운 타사 시뮬레이터 설치 후.

When a check fails in a way that is not obviously a local misconfiguration,
명백한 로컬 구성 오류가 아닌 방식으로 검사가 실패하는 경우
research it with [known-issue-research.md](known-issue-research.md) before
전에 [known-issue-research.md](known-issue-research.md)로 조사해 보세요.
concluding it is site-specific.
결론은 사이트에 따라 다릅니다.

## Manual checks the script cannot make
## 스크립트가 수행할 수 없는 수동 검사

Ask the user, or confirm in Vivado:
사용자에게 물어보거나 Vivado에서 확인하세요.

```tcl
report_property [current_project]
get_property compxlib.<simulator>_compiled_library_dir [current_project]
export_simulation -help
compile_simlib -help
```

Never guess site paths, license servers, or module names.
사이트 경로, 라이센스 서버 또는 모듈 이름을 추측하지 마십시오.
Never run
절대 뛰지 마세요
`compile_simlib` without approval; it is long and it writes to shared areas.
승인 없이 `compile_simlib`; 길이가 길고 공유 영역에 씁니다.

## Official sources
## 공식 출처

- [UG973 compatible third-party tools](https://docs.amd.com/r/en-US/ug973-vivado-release-notes-install-license/Compatible-Third-Party-Tools)
- [UG973 호환 타사 도구](https://docs.amd.com/r/en-US/ug973-vivado-release-notes-install-license/Compatible-Third-Party-Tools)
- [How to compile Vivado simulation libraries (AR 64083)](https://adaptivesupport.amd.com/s/article/64083?language=en_US)
- [Vivado 시뮬레이션 라이브러리 컴파일 방법(AR 64083)](https://adaptivesupport.amd.com/s/article/64083?언어=en_US)
- [compile_simlib design assistant (AR 63904)](https://adaptivesupport.amd.com/s/article/63904?language=en_US)
- [compile_simlib 디자인 도우미(AR 63904)](https://adaptivesupport.amd.com/s/article/63904?언어=en_US)
