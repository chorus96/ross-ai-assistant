<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="docs/images/amd-ross-lockup-dark.png">
    <source media="(prefers-color-scheme: light)" srcset="docs/images/amd-ross-lockup-light.png">
    <img alt="AMD Ross" src="docs/images/amd-ross-lockup-light.png" width="280">
  </picture>
</p>

# AMD Ross™ Agentic AI Assistant
# AMD Ross™ Agentic AI 어시스턴트

![Agent Skills](https://img.shields.io/badge/Agent_Skills-Standard-7B2D8E)
[![GitHub Copilot](https://img.shields.io/badge/GitHub_Copilot-Compatible-007ACC)](https://github.com/features/copilot)
[![Cursor](https://img.shields.io/badge/Cursor-Compatible-000000)](https://cursor.com)
[![Claude Code](https://img.shields.io/badge/Claude_Code-Compatible-F07535)](https://www.anthropic.com/claude-code)
[![OpenAI Codex](https://img.shields.io/badge/OpenAI_Codex-Compatible-412991)](https://openai.com/codex/)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue)](LICENSE)

Agent skills for AMD FPGA/SoC development.
AMD FPGA/SoC 개발을 위한 에이전트 기술.

**New here?** Start with the [Getting Started](docs/getting-started/) guide.
**새로운 내용이 있습니까?** [시작하기](docs/getting-started/) 가이드부터 시작하세요.
See [docs](docs/) for FAQ, MCP reference, and support.
FAQ, MCP 참조 및 지원은 [문서](docs/)를 참조하세요.

## Quick Start
## 빠른 시작

We recommend cloning this repository locally.
이 저장소를 로컬로 복제하는 것이 좋습니다.
It includes not only skills, but also docs and examples.
여기에는 기술뿐만 아니라 문서와 예제도 포함되어 있습니다.
In the install commands below, replace `/path/to/ross-ai-assistant` with the path to your local clone.
아래 설치 명령에서 `/path/to/ross-ai-assistant`를 로컬 복제본의 경로로 바꾸세요.

### Prerequisites
### 전제조건

Install **one** of the following from the [AMD Ross Agentic AI Assistant download page](https://www.amd.com/en/support/downloads/ross-agentic-ai.html):
[AMD Ross Agentic AI Assistant 다운로드 페이지](https://www.amd.com/en/support/downloads/ross-agentic-ai.html)에서 다음 중 **하나**를 설치하세요.

| Component<br>구성 요소 | Description<br>설명 | Follow-up installation<br>후속 설치 |
|-----------|-------------|------------------------|
| Vivado AI Extension (VS Code / Cursor)<br>Vivado AI 확장(VS 코드/커서) | Includes the Vivado MCP server, the skills in this repository, and the `amd-doc-search` MCP configuration<br>Vivado MCP 서버, 이 저장소의 기술 및 `amd-doc-search` MCP 구성을 포함합니다. | None<br>없음 |
| Vivado MCP Server (Linux or Windows)<br>Vivado MCP 서버(Linux 또는 Windows) | The Vivado MCP server: Tcl command execution, project management, and synthesis/implementation<br>Vivado MCP 서버: Tcl 명령 실행, 프로젝트 관리 및 합성/구현 | Configure the [`amd-doc-search` MCP server](#install-the-amd-doc-search-mcp-server), then install the skills in this repo as a [plugin](#install-as-a-plugin) or [individually](#install-skills-interactive)<br>[`amd-doc-search` MCP 서버](#install-the-amd-doc-search-mcp-server)를 구성한 다음 이 저장소의 기술을 [플러그인](#install-as-a-plugin) 또는 [개별적으로](#install-skills-interactive)로 설치합니다. |

Use the MCP server for CLI clients, or the VS Code / Cursor extension for IDE setups — not both.
CLI 클라이언트에는 MCP 서버를 사용하고, IDE 설정에는 VS Code/Cursor 확장을 사용하세요. 둘 다 사용할 수는 없습니다.

### Install the amd-doc-search MCP server
### amd-doc-search MCP 서버 설치

The `amd-doc-search` MCP server searches AMD FPGA and Adaptive SoC documentation: user guides and device collaterals, and tool usage for Vivado, Vitis, embedded system software, and related products.
'amd-doc-search' MCP 서버는 AMD FPGA 및 Adaptive SoC 문서(사용자 가이드, 장치 보조 자료, Vivado, Vitis, 임베디드 시스템 소프트웨어 및 관련 제품에 대한 도구 사용)를 검색합니다.

The Vivado AI Extension already includes this configuration.
Vivado AI Extension에는 이미 이 구성이 포함되어 있습니다.
With the standalone Vivado MCP Server, add the server yourself.
독립형 Vivado MCP 서버를 사용하면 서버를 직접 추가할 수 있습니다.

For detailed installation instructions, see [https://ross.amd.com/chat#setup](https://ross.amd.com/chat#setup).
자세한 설치 지침은 [https://ross.amd.com/chat#setup](https://ross.amd.com/chat#setup)을 참조하세요.

> **Local knowledge base.** If you want documentation search to stay on your own machine, use the [local knowledge base](docs/local-kb/README.md) instead of the hosted `amd-doc-search` service.
> **로컬 지식 베이스.** 문서 검색을 자신의 컴퓨터에 유지하려면 호스팅된 `amd-doc-search` 서비스 대신 [로컬 지식 베이스](docs/local-kb/README.md)를 사용하세요.

### Install as a plugin
### 플러그인으로 설치

This repository is also an agent plugin: your local clone can be installed as a plugin, which adds every skill in one step.
이 저장소는 에이전트 플러그인이기도 합니다. 로컬 복제본을 플러그인으로 설치하여 모든 기술을 한 단계로 추가할 수 있습니다.
Updating is then a `git pull` of the clone.
업데이트는 클론의 `git pull`입니다.

Once installed, invoke the skills by name, for example `/hls-optimize`, and they call each other by those same names.
설치한 후에는 `/hls-optimize`와 같은 이름으로 기술을 호출하면 동일한 이름으로 서로를 호출합니다.
Some clients list them under the plugin's name, `amd-ross-agentic-ai-assistant`.
일부 클라이언트는 플러그인 이름 `amd-ross-agentic-ai-assistant` 아래에 이를 나열합니다.

> **Choose one install method.** If you install the plugin, do not also install the skills with `npx skills add` or by copying them.
> **설치 방법 중 하나를 선택하세요.** 플러그인을 설치하는 경우 'npx Skill add'나 복사하여 스킬도 설치하지 마세요.
> Every skill would then be loaded twice.
> 그러면 모든 스킬이 두 번 로드됩니다.

The plugin contains skills only.
플러그인에는 스킬만 포함되어 있습니다.
The Vivado MCP Server or the Vivado AI Extension from [Prerequisites](#prerequisites) is still installed separately.
Vivado MCP Server 또는 [Prerequisites](#prerequisites)의 Vivado AI Extension은 여전히 ​​별도로 설치됩니다.

See [Install as a plugin](docs/getting-started/install-plugin.md) for the steps in your AI client and how to update.
AI 클라이언트의 단계와 업데이트 방법은 [플러그인으로 설치](docs/getting-started/install-plugin.md)를 참조하세요.

### Install skills (interactive)
### 스킬 설치(대화형)

Install [Node.js](https://nodejs.org/) to use the `npx skills` commands below.
아래 `npx Skill` 명령을 사용하려면 [Node.js](https://nodejs.org/)를 설치하세요.

Point `npx skills add` at your local clone (or an extracted release package):
로컬 복제본(또는 추출된 릴리스 패키지)에서 `npx Skill add`를 지정합니다.

```bash
npx skills add /path/to/ross-ai-assistant
```

This installs skills into the current workspace.
그러면 현재 작업 영역에 기술이 설치됩니다.
Add `--global` to install into your home directory (`~/.claude/skills/`) so they are available across all workspaces.
`--global`을 추가하여 홈 디렉터리(`~/.claude/skills/`)에 설치하면 모든 작업 공간에서 사용할 수 있습니다.

### Install all skills
### 모든 스킬을 설치하세요

```bash
npx skills add /path/to/ross-ai-assistant --all
```

### Install all skills globally
### 전역적으로 모든 스킬 설치

```bash
npx skills add /path/to/ross-ai-assistant --all --global
```

### Install a specific skill
### 특정 스킬을 설치하세요

```bash
npx skills add /path/to/ross-ai-assistant --skill hls-optimize
```

### List available skills
### 사용 가능한 스킬 나열

```bash
npx skills add /path/to/ross-ai-assistant --list
```

### Manual installation (no Node.js)
### 수동 설치(Node.js 없음)

Copy desired skill folders to your agent's skills directory:
원하는 기술 폴더를 상담원의 기술 디렉터리에 복사합니다.

```bash
cp -r /path/to/ross-ai-assistant/skills/hls-optimize ~/.claude/skills/
```

## Skills
## 기술

### Vivado
### 비바도

These skills require the [Vivado MCP Server](docs/reference/vivado-mcp-tools.md) for live Vivado interaction.
이러한 기술은 라이브 Vivado 상호 작용을 위해 [Vivado MCP 서버](docs/reference/vivado-mcp-tools.md)가 필요합니다.

| Skill<br>스킬                                                         | Description<br>설명                                                                    | Examples<br>예                                                             |
|---------------------------------------------------------------|--------------------------------------------------------------------------------|----------------------------------------------------------------------|
| [vivado-simulate-rtl](skills/vivado-simulate-rtl/)<br>[vivado-simulate-rtl](기술/vivado-simulate-rtl/)            | Run and diagnose RTL simulation across XSim and supported third-party simulators<br>XSim 및 지원되는 타사 시뮬레이터에서 RTL 시뮬레이션을 실행하고 진단합니다. | -                                                                    |
| [vivado-rtl-lint](skills/vivado-rtl-lint/)<br>[vivado-rtl-lint](기술/vivado-rtl-lint/)                    | Run Vivado's RTL linter and report design issues with prioritized, code-level fixes<br>Vivado의 RTL Linter를 실행하고 우선순위가 지정된 코드 수준 수정으로 설계 문제를 보고합니다. | [multi-violation](examples/rtl-lint/multi-violation/)<br>[다중 위반](예/rtl-lint/다중 위반/)                |
| [vivado-timing-methodology-checks](skills/vivado-timing-methodology-checks/)<br>[vivado-timing-methodology-checks](skills/vivado-timing-methodology-checks/) | Run timing methodology checks and analyze violations<br>타이밍 방법론 검사 실행 및 위반 분석            | [multi-violation](examples/timing-methodology-checks/multi-violation/)<br>[다중 위반](예/timing-methodology-checks/다중 위반/) |
| [vivado-revision-control](skills/vivado-revision-control/)<br>[vivado-revision-control](skills/vivado-revision-control/)    | Manage Vivado project files under revision control<br>개정 제어 하에 Vivado 프로젝트 파일 관리                             | [export-ipi-project](examples/vivado-revision-control/export-ipi-project/)<br>[내보내기-ipi-프로젝트](예/vivado-개정-제어/내보내기-ipi-프로젝트/) |
| [vivado-ip-configurator](skills/vivado-ip-configurator/)<br>[vivado-ip-configurator](기술/vivado-ip-configurator/) | Configure one Vivado IP cell from a natural-language description<br>자연어 설명으로 하나의 Vivado IP 셀 구성 | — |

### Vitis HLS
### 바이티스 HLS

| Skill<br>스킬                                                  | Description<br>설명                                                                    | Examples<br>예                                                                                                                                                           |
|--------------------------------------------------------|--------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| [hls-matlab-to-cpp](skills/hls-matlab-to-cpp/)<br>[hls-matlab-to-cpp](기술/hls-matlab-to-cpp/)        | Convert MATLAB sample-based code to synthesizable C++ for HLS<br>MATLAB 샘플 기반 코드를 HLS용 합성 가능한 C++로 변환                  | [edge-detection](examples/hls-matlab-to-cpp/edge-detection/), [matlab-kernel-gemm](examples/hls-matlab-to-cpp/matlab-kernel-gemm/), [matlab-kernel-svd](examples/hls-matlab-to-cpp/matlab-kernel-svd/)<br>[에지 감지](examples/hls-matlab-to-cpp/edge-Detection/), [matlab-kernel-gemm](examples/hls-matlab-to-cpp/matlab-kernel-gemm/), [matlab-kernel-svd](examples/hls-matlab-to-cpp/matlab-kernel-svd/) |
| [hls-architect](skills/hls-architect/)<br>[hls-architect](기술/hls-architect/)                 | Convert input code to multi-stage HLS dataflow architecture<br>입력 코드를 다단계 HLS 데이터 흐름 아키텍처로 변환                    | —                                                                                                                                                                  |
| [hls-optimize](skills/hls-optimize/)<br>[hls-최적화](기술/hls-최적화/)                   | Iteratively optimize HLS kernel against target performance/resource criteria<br>목표 성능/리소스 기준에 대해 HLS 커널을 반복적으로 최적화합니다.   | [globaltonemapping](examples/hls-optimize/globaltonemapping/)<br>[globaltonemapping](examples/hls-optimize/globaltonemapping/)                                                                                                      |

These three skills are the primary entry points for HLS development.
이 세 가지 기술은 HLS 개발의 기본 진입점입니다.
Each skill automatically invokes specialized helper skills (dataflow checks, burst inference, array partitioning, report extraction, etc.) as needed — you don't need to install or invoke helpers separately.
각 기술은 필요에 따라 전문 도우미 기술(데이터 흐름 검사, 버스트 추론, 배열 분할, 보고서 추출 등)을 자동으로 호출합니다. 도우미를 별도로 설치하거나 호출할 필요가 없습니다.

**Build skill:** [hls-run-flow](skills/hls-run-flow/) runs the HLS compilation pipeline (csim, csynth, cosim, implementation) and is called by the core skills above or can be invoked directly.
**빌드 기술:** [hls-run-flow](skills/hls-run-flow/)는 HLS 컴파일 파이프라인(csim, csynth, cosim, 구현)을 실행하고 위의 핵심 기술에 의해 호출되거나 직접 호출될 수 있습니다.
Example: [intro-matmul](examples/hls-run-flow/intro-matmul/).
예: [intro-matmul](예/hls-run-flow/intro-matmul/).

### Hardware Debug
### 하드웨어 디버그

These skills debug live FPGA/SoC hardware via Vivado Hardware Manager Tcl.
이러한 기술은 Vivado Hardware Manager Tcl을 통해 라이브 FPGA/SoC 하드웨어를 디버그합니다.

| Skill<br>스킬                                        | Description<br>설명                                                                    | Examples<br>예                                                                                                                                                                                                  |
|----------------------------------------------|--------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| [hw-ila-debug](skills/hw-ila-debug/)<br>[hw-ila-디버그](기술/hw-ila-debug/)         | Interact with ILA debug cores on live hardware: trigger, capture, and export waveform data<br>라이브 하드웨어에서 ILA 디버그 코어와 상호 작용: 파형 데이터 트리거, 캡처 및 내보내기 | [axi-protocol-capture](examples/hw-ila-debug/axi-protocol-capture/)<br>[axi-프로토콜-캡처](예/hw-ila-debug/axi-프로토콜-캡처/)                                                                                                                                     |
| [hw-vio-debug](skills/hw-vio-debug/)<br>[hw-vio-디버그](기술/hw-vio-debug/)         | Read input probes and drive output probes on live hardware via VIO<br>VIO를 통해 라이브 하드웨어에서 입력 프로브를 읽고 출력 프로브를 구동합니다.             | [axi-register-rw](examples/hw-vio-debug/axi-register-rw/)<br>[axi-register-rw](예제/hw-vio-debug/axi-register-rw/)                                                                                                                                               |

### Vitis AI
### 바이티스 AI

These skills compile and optimize ONNX models for AMD NPU hardware with the VAIML compiler.
이러한 기술은 VAIML 컴파일러를 사용하여 AMD NPU 하드웨어용 ONNX 모델을 컴파일하고 최적화합니다.
Start with an orchestrator skill and let it load implementation skills and [worker subagents](agents/) as needed.
오케스트레이터 기술로 시작하여 필요에 따라 구현 기술과 [작업자 하위 에이전트](agents/)를 로드하도록 합니다.

| Skill<br>스킬 | Description<br>설명 | Examples<br>예 |
|-------|-------------|----------|
| [vai-quantization-guide](skills/vai-quantization-guide/)<br>[vai-양자화-가이드](skills/vai-양자화-가이드/) | Mixed-precision quantization for AMD NPU (VINT8, BF16, hybrid)<br>AMD NPU(VINT8, BF16, 하이브리드)에 대한 혼합 정밀도 양자화 | [resnet50-quantize-vint8-bf16](examples/vai-quantization-guide/resnet50-quantize-vint8-bf16/), [vit-encoder-full-offload](examples/vai-quantization-guide/vit-encoder-full-offload/)<br>[resnet50-Quantize-vint8-bf16](예제/vai-양자화-가이드/resnet50-Quantize-vint8-bf16/), [vit-encoder-full-offload](예제/vai-양자화-가이드/vit-encoder-full-offload/) |
| [vai-flag-configuration](skills/vai-flag-configuration/)<br>[vai-플래그-구성](skills/vai-flag-configuration/) | Iteratively tune `vitisai_config.json` compiler flags<br>'vitsai_config.json' 컴파일러 플래그를 반복적으로 조정합니다. | — |
| [vai-custom-op](skills/vai-custom-op/)<br>[vai-custom-op](스킬/vai-custom-op/) | Orchestrate parallel custom AIE operator development<br>병렬 맞춤형 AIE 연산자 개발 조율 | [multi-input-multi-node-custom-op](examples/vai-custom-op/multi-input-multi-node-custom-op/)<br>[다중 입력-다중 노드-사용자 정의-op](예/vai-custom-op/다중 입력-다중-노드-사용자 정의-op/) |

Helper skills (`vai-custom-op-implementation`, `vai-fe-args`, `vai-perf-analysis`, and the other `vai-*` folders) are invoked by those orchestrators and by the workers in `agents/`.
도우미 기술(`vai-custom-op-implementation`, `vai-fe-args`, `vai-perf-analytic` 및 기타 `vai-*` 폴더)은 해당 오케스트레이터와 `agents/`의 작업자에 의해 호출됩니다.
