<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Getting Started
# 시작하기

Set up the AMD Ross Agentic AI Assistant in your preferred environment.
원하는 환경에서 AMD Ross Agentic AI Assistant를 설정하세요.
Follow the guides below in order — install agent skills first, then configure the MCP servers you need.
아래 가이드를 순서대로 따르세요. 먼저 에이전트 기술을 설치한 다음 필요한 MCP 서버를 구성하세요.

## Prerequisites
## 전제조건

**For Vivado AI Assistant (MCP Server):**
**Vivado AI Assistant(MCP 서버)의 경우:**

- **Vivado 2026.1** installed and on your `PATH`
- **Vivado 2026.1**이 `PATH`에 설치되어 있음
- **One of the following** (not both):
- **다음 중 하나**(둘 다는 아님):
    - **Vivado AI Assistant Extension** VSIX — for IDE setups (VS Code / Cursor).
    - **Vivado AI Assistant Extension** VSIX — IDE 설정용(VS 코드/커서).
    - The extension bundles the MCP server.
    - 확장은 MCP 서버를 번들로 제공합니다.
    - **Vivado MCP Server** binary — for CLI setups (Claude Code, Copilot CLI, Codex).
    - **Vivado MCP 서버** 바이너리 — CLI 설정용(Claude Code, Copilot CLI, Codex).
    - Install to a directory like `~/tools/`.
    - `~/tools/`와 같은 디렉토리에 설치하세요.
- A running Vivado session or a design checkpoint (DCP) to work with
- 실행 중인 Vivado 세션 또는 작업할 디자인 체크포인트(DCP)

**For Vitis HLS AI Assistant (Agent Skills):**
**Vitis HLS AI Assistant(에이전트 기술)의 경우:**

- **Vitis Unified IDE** installed and licensed — after sourcing `settings64.sh`, `v++ --version` and `vitis-run --version` should both work
- **Vitis Unified IDE** 설치 및 라이센스 — `settings64.sh`, `v++ --version` 및 `vitis-run --version`을 소싱한 후 둘 다 작동해야 합니다.
- A skill-capable AI client: VS Code, Cursor, Claude Code, Codex CLI, or GitHub Copilot CLI
- 기술이 가능한 AI 클라이언트: VS Code, Cursor, Claude Code, Codex CLI 또는 GitHub Copilot CLI
- *(Optional)* The Vivado MCP Server, for AMD-doc-grounded answers via `vivado_doc_search`
- *(선택 사항)* `vivado_doc_search`를 통한 AMD 문서 기반 답변을 위한 Vivado MCP 서버

> **Version Compatibility:** Vivado **2026.1** and Vitis **2026.1** are the tested versions.
> **버전 호환성:** Vivado **2026.1** 및 Vitis **2026.1**은 테스트된 버전입니다.
> The MCP server communicates with Vivado through its standard Tcl interface, so it is architecturally compatible with all released versions of Vivado.
> MCP 서버는 표준 Tcl 인터페이스를 통해 Vivado와 통신하므로, 출시된 모든 Vivado 버전과 구조적으로 호환됩니다.
> Individual skills and examples note their own minimum supported versions where they differ (see each skill's `SKILL.md` and each example's `README.md`).
> 개별 스킬과 예시는 서로 다른 경우 최소 지원 버전을 명시합니다(각 스킬의 'SKILL.md' 및 각 예시의 'README.md' 참조).

## Setup Guides
## 설정 가이드

> Cursor, Claude Code and Codex users can install step 1's skills as a plugin instead — see [Install as a plugin](../../README.md#install-as-a-plugin).
> Cursor, Claude Code 및 Codex 사용자는 대신 1단계의 스킬을 플러그인으로 설치할 수 있습니다. [플러그인으로 설치](../../README.md#install-as-a-plugin)를 참조하세요.
> The plugin contains skills only; the MCP servers in the remaining steps are still installed separately.
> 플러그인에는 기술만 포함되어 있습니다. 나머지 단계의 MCP 서버는 여전히 별도로 설치됩니다.

| Step<br>단계 | Guide<br>가이드 | What it covers<br>다루는 내용 |
|------|-------|----------------|
| 1 | [Install Agent Skills](../../README.md#quick-start)<br>[에이전트 스킬 설치](../../README.md#quick-start) | `npx skills add` — install HLS and Vivado methodology skills<br>`npx Skill add` — HLS 및 Vivado 방법론 스킬 설치 |
| 2 | [Vivado MCP Server](vivado-mcp.md)<br>[비바도 MCP 서버](vivado-mcp.md) | VSIX extension (VS Code / Cursor) or standalone binary (CLI tools)<br>VSIX 확장(VS 코드/커서) 또는 독립 실행형 바이너리(CLI 도구) |
| 3 | [HLS Quick Start](hls-quickstart.md)<br>[HLS 빠른 시작](hls-quickstart.md) | Vitis HLS skills — no MCP server needed<br>Vitis HLS 기술 — MCP 서버가 필요하지 않습니다 |
| 4 | [Pre-Flight Checklist](preflight-checklist.md)<br>[비행 전 체크리스트](preflight-checklist.md) | Verify everything works end-to-end<br>모든 것이 처음부터 끝까지 작동하는지 확인하세요. |

## Architecture Overview
## 아키텍처 개요

**Vivado AI Assistant** uses the MCP protocol to give your AI agent live tool access:
**Vivado AI Assistant**는 MCP 프로토콜을 사용하여 AI 에이전트에 라이브 도구 액세스 권한을 부여합니다.

```
Your AI Client  ──MCP protocol──>  MCP Server  ──Tcl──>  Vivado
```

1. **Your client** (IDE or CLI) connects to the MCP server via stdio
1. **클라이언트**(IDE 또는 CLI)는 stdio를 통해 MCP 서버에 연결됩니다.
2. **The MCP server** translates AI agent requests into tool commands
2. **MCP 서버**는 AI 에이전트 요청을 도구 명령으로 변환합니다.
3. **Vivado** executes the commands and returns results through the same chain
3. **Vivado**는 명령을 실행하고 동일한 체인을 통해 결과를 반환합니다.

**Vitis HLS AI Assistant** uses agent skills — no MCP server required:
**Vitis HLS AI Assistant**는 에이전트 기술을 사용합니다. MCP 서버가 필요하지 않습니다.

```
Your AI Client  ──reads──>  Agent Skills (SKILL.md)  ──runs──>  v++ / vitis-run
```

1. **Your client** (IDE or CLI) loads skill files from your workspace or `~/.claude/skills/`
1. **클라이언트**(IDE 또는 CLI)는 작업공간 또는 `~/.claude/skills/`에서 기술 파일을 로드합니다.
2. **The skills** encode HLS methodology — pragma strategies, flow commands, analysis patterns
2. **기술**은 HLS 방법론을 인코딩합니다 — pragma 전략, 흐름 명령, 분석 패턴
3. **Vitis tools** (`v++`, `vitis-run`) are invoked directly by the agent in your terminal
3. **Vitis 도구**(`v++`, `vitis-run`)는 터미널의 에이전트에 의해 직접 호출됩니다.

## After Setup
## 설정 후

Once configured, try the [HLS Quick Start](hls-quickstart.md) or the example designs in the `examples/` directory.
구성한 후에는 [HLS 빠른 시작](hls-quickstart.md) 또는 `examples/` 디렉터리의 예제 디자인을 사용해 보세요.

---
