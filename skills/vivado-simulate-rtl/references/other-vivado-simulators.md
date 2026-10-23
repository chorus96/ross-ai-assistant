<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Other Vivado-Supported Simulators
# 기타 Vivado 지원 시뮬레이터

Release snapshot and index.
스냅샷 및 인덱스를 출시합니다.
Each backend has a dedicated reference linked from
각 백엔드에는 다음에서 링크된 전용 참조가 있습니다.
`SKILL.md`.
`SKILL.md`.
Treat versions as a snapshot, not a permanent promise.
버전을 영구적인 약속이 아닌 스냅샷으로 취급하십시오.
Before use,
사용하기 전에,
query `Vivado:vivado_doc_search` for the installed release's UG900 supported
지원되는 설치된 릴리스의 UG900에 대해 `Vivado:vivado_doc_search` 쿼리
simulators and UG973 compatible third-party tools.
시뮬레이터 및 UG973 호환 타사 도구.

## Vivado 2026.1 snapshot
## 비바도 2026.1 스냅샷

| Simulator<br>시뮬레이터 | Minimum listed version<br>최소 나열된 버전 | UG973 host support<br>UG973 호스트 지원 | Canonical ID<br>정식 ID | Reference<br>참고자료 |
|---|---:|---|---|---|
| Siemens ModelSim DE<br>지멘스 모델Sim DE | 2025.3 | Red Hat Linux, Windows<br>레드햇 리눅스, 윈도우 | `modelsim`<br>'모델심' | [modelsim.md](modelsim.md)<br>[모델심.md](모델심.md) |
| Synopsys VCS<br>시놉시스 VCS | X-2025.06-SP2<br>X-2025.06-SP2 | Red Hat Linux<br>레드햇 리눅스 | `vcs`<br>`vcs` | [vcs.md](vcs.md)<br>[vcs.md](vcs.md) |
| Cadence Xcelium<br>케이던스 엑셀리움 | 25.09.001 | Red Hat Linux<br>레드햇 리눅스 | `xcelium`<br>'엑스셀륨' | [xcelium.md](xcelium.md)<br>[xcelium.md](xcelium.md) |
| Aldec Riviera-PRO<br>알덱 리비에라-PRO | 2024.10 | Red Hat Linux, Windows<br>레드햇 리눅스, 윈도우 | `riviera`<br>'리비에라' | [riviera.md](riviera.md)<br>[리비에라.md](리비에라.md) |
| Aldec Active-HDL<br>Aldec Active-HDL | 16.0 | Windows<br>윈도우 | `activehdl`<br>'활성HDL' | [activehdl.md](activehdl.md)<br>[액티브hdl.md](액티브hdl.md) |

Questa remains [questa.md](questa.md).
Questa는 [questa.md](questa.md)로 유지됩니다.
XSim remains [xsim.md](xsim.md).
XSim은 [xsim.md](xsim.md)로 유지됩니다.

These are minimum compatible versions in UG973; AMD states testing is limited
이는 UG973의 최소 호환 버전입니다. AMD는 테스트가 제한되어 있다고 말합니다.
to current versions.
현재 버전으로.
A host OS supported by Vivado is not automatically a
Vivado가 지원하는 호스트 OS는 자동으로
supported host for every third-party simulator.
모든 타사 시뮬레이터에 대해 지원되는 호스트.

## Shared rules
## 공유 규칙

For every third-party backend:
모든 타사 백엔드의 경우:

1. verify current simulator/version/platform support in UG900 and UG973;
1. UG900 및 UG973에서 현재 시뮬레이터/버전/플랫폼 지원을 확인합니다.
2. ask for the site-specific installation and license setup;
2. 사이트별 설치 및 라이센스 설정을 요청합니다.
3. validate the executable-reported version;
3. 실행 가능한 보고 버전의 유효성을 검사합니다.
4. validate version-matched AMD simulation libraries from `compile_simlib`;
4. `compile_simlib`에서 버전이 일치하는 AMD 시뮬레이션 라이브러리를 검증합니다.
5. inspect project, simulation set, top, language mix, IP, and mode;
5. 프로젝트, 시뮬레이션 세트, 상단, 언어 혼합, IP 및 모드를 검사합니다.
6. have Vivado generate scripts with the canonical ID;
6. Vivado가 표준 ID를 사용하여 스크립트를 생성하도록 합니다.
7. run generated compile, elaborate, and simulate scripts in order;
7. 생성된 컴파일, 정교화, 시뮬레이션 스크립트를 순서대로 실행합니다.
8. require an explicit functional PASS and preserve transcript, waveform,
8. 명시적인 기능 통과가 필요하며 기록, 파형,
scripts, library mapping, seed, defines, plusargs, and generics.
스크립트, 라이브러리 매핑, 시드, 정의, plusargs 및 제네릭.

Never invent commands when generated scripts exist.
생성된 스크립트가 존재할 때 명령을 작성하지 마십시오.
Never silently fall back to
절대로 조용히 물러서지 마세요.
XSim after a requested backend fails.
요청한 백엔드가 실패한 후 XSim.
Do not copy Questa property prefixes to
Questa 속성 접두어를 복사하지 마십시오.
another simulator.
또 다른 시뮬레이터.

```tcl
report_property [current_project]
report_property [get_filesets <simset>]
export_simulation -help
compile_simlib -help
```

## Official sources
## 공식 출처

- [UG900 supported simulators](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Supported-Simulators)
- [UG900 지원 시뮬레이터](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Supported-Simulators)
- [UG900 export_simulation](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/export_simulation)
- [UG900 내보내기_시뮬레이션](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/export_simulation)
- [UG900 third-party simulator setup](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Simulator-Settings-for-Third-Party-Tools)
- [UG900 타사 시뮬레이터 설정](https://docs.amd.com/r/en-US/ug900-vivado-logic-simulation/Simulator-Settings-for-Third-Party-Tools)
- [UG973 compatible third-party tools](https://docs.amd.com/r/en-US/ug973-vivado-release-notes-install-license/Compatible-Third-Party-Tools)
- [UG973 호환 타사 도구](https://docs.amd.com/r/en-US/ug973-vivado-release-notes-install-license/Compatible-Third-Party-Tools)
