<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Vivado MCP Server
# 비바도 MCP 서버

The Vivado MCP Server connects your AI agent to a live Vivado session.
Vivado MCP 서버는 AI 에이전트를 라이브 Vivado 세션에 연결합니다.
The agent can start Vivado, run Tcl commands, read logs, search AMD documentation, and drive synthesis/implementation — all through the MCP protocol.
에이전트는 MCP 프로토콜을 통해 Vivado를 시작하고, Tcl 명령을 실행하고, 로그를 읽고, AMD 문서를 검색하고, 드라이브 합성/구현을 수행할 수 있습니다.

```
Your AI Client  ──MCP protocol──>  Vivado MCP Server  ──Tcl──>  Vivado
```

## Prerequisites
## 전제조건

- **Vivado 2026.1** installed and on your `PATH`
- **Vivado 2026.1**이 `PATH`에 설치되어 있음
- **One of the following** (not both):
- **다음 중 하나**(둘 다는 아님):
    - **Vivado AI Assistant Extension** VSIX — for IDE setups (VS Code / Cursor).
    - **Vivado AI Assistant Extension** VSIX — IDE 설정용(VS 코드/커서).
    - The extension bundles the MCP server.
    - 확장은 MCP 서버를 번들로 제공합니다.
    - **Vivado MCP Server** binary — for CLI setups (Claude Code, Copilot CLI, Codex CLI).
    - **Vivado MCP 서버** 바이너리 — CLI 설정용(Claude Code, Copilot CLI, Codex CLI).
    - Download from the EA release page.
    - EA 릴리스 페이지에서 다운로드하세요.

> **Version Compatibility:** Vivado **2026.1** is the tested baseline.
> **버전 호환성:** Vivado **2026.1**은 테스트된 기준입니다.
> The MCP server communicates through Vivado's standard Tcl interface and is architecturally compatible with all released versions.
> MCP 서버는 Vivado의 표준 Tcl 인터페이스를 통해 통신하며 아키텍처적으로 출시된 모든 버전과 호환됩니다.
> Individual skills and examples note their minimum supported versions.
> 개별 기술과 예제에는 지원되는 최소 버전이 명시되어 있습니다.

---

## VS Code & Cursor
## VS 코드 및 커서

The VSIX extension includes both the Vivado AI Assistant UI **and** the MCP server — no separate binary install or MCP configuration needed.
VSIX 확장에는 Vivado AI Assistant UI **및** MCP 서버가 모두 포함되어 있으므로 별도의 바이너리 설치나 MCP 구성이 필요하지 않습니다.

### Install the Vivado AI Assistant Extension
### Vivado AI Assistant 확장 프로그램 설치

**From VSIX file:**
**VSIX 파일에서:**

1. Download the Vivado AI Extension `.vsix` file from the EA release page
1. EA 릴리스 페이지에서 Vivado AI 확장 `.vsix` 파일을 다운로드하세요.
2. Open VS Code (or Cursor)
2. VS Code(또는 커서) 열기
3. Go to **Extensions** sidebar > click `...` > **Install from VSIX...**
3. **확장** 사이드바로 이동 > `...` 클릭 > **VSIX에서 설치...**
4. Select the downloaded `.vsix` file
4. 다운로드한 `.vsix` 파일을 선택하세요.

**From command line:**
**명령줄에서:**

```bash
# VS Code
code --install-extension vivado-ai-extension-<version>.vsix

# Cursor
cursor --install-extension vivado-ai-extension-<version>.vsix
```

### Configure Vivado Path
### Vivado 경로 구성

After installing the extension, set the path to your Vivado executable:
확장 기능을 설치한 후 Vivado 실행 파일의 경로를 설정하세요.

1. Open **Settings** (Ctrl+,)
1. **설정** 열기(Ctrl+,)
2. Search for **Vivado Path**
2. **Vivado 경로**를 검색하세요.
3. Enter the full path to your Vivado binary (e.g., `/tools/Xilinx/Vivado/2026.1/bin/vivado`)
3. Vivado 바이너리의 전체 경로를 입력하세요(예: `/tools/Xilinx/Vivado/2026.1/bin/vivado`)

> If Vivado is already on your system `PATH`, this step is optional — the extension will auto-detect it.
> Vivado가 이미 시스템 'PATH'에 있는 경우 이 단계는 선택 사항입니다. 확장 프로그램이 이를 자동 감지합니다.
> However, explicitly setting the path avoids issues when multiple versions are installed.
> 그러나 경로를 명시적으로 설정하면 여러 버전이 설치될 때 문제가 발생하지 않습니다.

### Start Vivado
### 비바도 시작

You don't need to manually launch Vivado — the AI agent can start it for you.
Vivado를 수동으로 시작할 필요가 없습니다. AI 에이전트가 대신 시작할 수 있습니다.
Simply ask in the chat panel:
채팅 패널에서 간단히 물어보세요:

> *"Start a Vivado session"*
> *"Vivado 세션 시작"*

The agent will use the `vivado_start` MCP tool to launch Vivado automatically.
에이전트는 'vivado_start' MCP 도구를 사용하여 Vivado를 자동으로 시작합니다.

**Alternatively, start Vivado manually and connect:**
**또는 Vivado를 수동으로 시작하고 연결하세요.**

```bash
vivado -mode tcl
```

Then, in the Vivado Tcl console, enable the web server:
그런 다음 Vivado Tcl 콘솔에서 웹 서버를 활성화합니다.

```tcl
webserver -start -port 8088 -key none
```

Ask the agent to connect:
상담원에게 연결을 요청하세요.

> *"Connect to my Vivado session at http://localhost:8088"*
> *"http://localhost:8088에서 내 Vivado 세션에 연결합니다"*

> **Note:** HLS skills (matlab-to-cpp, hls-architect, hls-optimize) do not require a running Vivado session.
> **참고:** HLS 기술(matlab-to-cpp, hls-architect, hls-optimize)에는 실행 중인 Vivado 세션이 필요하지 않습니다.
> They provide code generation and optimization guidance directly.
> 코드 생성 및 최적화 지침을 직접 제공합니다.

### Verify
### 확인

=== "VS Code"
=== "VS 코드"

    1. Open the **Copilot Chat** panel (Ctrl+Alt+B)
    1. **Copilot Chat** 패널 열기(Ctrl+Alt+B)
    2. Switch to **Agent** mode (click the mode selector)
    2. **에이전트** 모드로 전환(모드 선택기 클릭)
    3. Type: `List the available MCP tools`
    3. 유형: `사용 가능한 MCP 도구 나열`
    4. You should see Vivado-related tools like `vivado_execute`, `vivado_connect`, etc.
    4. `vivado_execute`, `vivado_connect` 등과 같은 Vivado 관련 도구가 표시되어야 합니다.

=== "Cursor"
=== "커서"

    1. Open **Cursor Chat** (Ctrl+L) or **Composer** (Ctrl+I)
    1. **커서 채팅**(Ctrl+L) 또는 **작곡가**(Ctrl+I)를 엽니다.
    2. Switch to **Agent** mode
    2. **에이전트** 모드로 전환
    3. Ask: `What Vivado MCP tools are available?`
    3. 질문: '어떤 Vivado MCP 도구를 사용할 수 있나요?'
    4. The agent should list tools like `vivado_execute`, `vivado_connect`, etc.
    4. 에이전트는 `vivado_execute`, `vivado_connect` 등과 같은 도구를 나열해야 합니다.

> **Tip:** Cursor auto-discovers MCP servers from the extension — no restart needed.
> **팁:** 커서는 확장 프로그램에서 MCP 서버를 자동 검색하므로 다시 시작할 필요가 없습니다.

### Troubleshooting
### 문제 해결

| Issue<br>이슈 | Solution<br>솔루션 |
|-------|----------|
| Chat panel not visible<br>채팅 패널이 표시되지 않음 | VS Code: Ctrl+Alt+B. Cursor: Ctrl+L<br>VS 코드: Ctrl+Alt+B. 커서: Ctrl+L |
| Vivado MCP tools missing from tools list<br>도구 목록에서 Vivado MCP 도구가 누락되었습니다. | Click **Configure tools** (next to model selection) and ensure Vivado MCP tools are listed. If not, reload the window (Ctrl+Shift+P > `Developer: Reload Window`)<br>**Configure tools**(모델 선택 옆)를 클릭하고 Vivado MCP 도구가 나열되어 있는지 확인하세요. 그렇지 않은 경우 창을 다시 로드합니다(Ctrl+Shift+P > `개발자: 창 다시 로드`). |
| "Connection refused" or Vivado MCP disconnect<br>"연결이 거부되었습니다" 또는 Vivado MCP 연결 끊김 | Reload the window and verify MCP tools are available. Check **Output > Vivado MCP** for errors<br>창을 다시 로드하고 MCP 도구를 사용할 수 있는지 확인하십시오. **Output > Vivado MCP**에서 오류를 확인하세요. |
| Extension not loading<br>확장 프로그램이 로드되지 않음 | Verify VS Code >= 1.114 or latest Cursor. Check **Output > Vivado MCP** for errors<br>VS Code >= 1.114 또는 최신 커서를 확인하세요. **Output > Vivado MCP**에서 오류를 확인하세요. |
| Skills not loading<br>스킬이 로딩되지 않습니다 | **Settings** > search **skills**. Ensure **Chat: Use Agent Skills** is checked and **Chat: Agent Skills Locations** includes `.claude/skills` (workspace) or `~/.claude/skills` (home). Verify the paths contain folders with `SKILL.md` files<br>**설정** > **스킬**을 검색하세요. **채팅: 상담원 기술 사용**이 선택되어 있고 **채팅: 상담원 기술 위치**에 `.claude/skills`(작업 공간) 또는 `~/.claude/skills`(홈)가 포함되어 있는지 확인하세요. 경로에 'SKILL.md' 파일이 있는 폴더가 포함되어 있는지 확인하세요. |

---

## Claude Code
## 클로드 코드

### Install the MCP Server Binary
### MCP 서버 바이너리 설치

Download the standalone binary from the EA release page, make it executable, and move it to a convenient location:
EA 릴리스 페이지에서 독립 실행형 바이너리를 다운로드하여 실행 가능하게 만들고 편리한 위치로 이동합니다.

```bash
chmod +x vivado-mcp-server-linux-amd64-<version>
mv vivado-mcp-server-linux-amd64-<version> ~/tools/vivado-mcp-server
```

### Register the Server
### 서버 등록

Register the Vivado MCP server at **user scope** so it's available in every workspace:
모든 워크스페이스에서 사용할 수 있도록 **사용자 범위**에서 Vivado MCP 서버를 등록하세요.

```bash
claude mcp add vivado-mcp --scope user --transport stdio \
  --env VIVADO_PATH=/path/to/Vivado/<version>/bin/vivado \
  -- /path/to/vivado-mcp-server \
  --stdio-bridge
```

This stores the config in `~/.claude.json`:
그러면 `~/.claude.json`에 구성이 저장됩니다.

```json
{
  "mcpServers": {
    "vivado-mcp": {
      "command": "/path/to/vivado-mcp-server",
      "args": ["--stdio-bridge"],
      "type": "stdio",
      "env": {
        "VIVADO_PATH": "/path/to/Vivado/<version>/bin/vivado"
      }
    }
  }
}
```

> **VIVADO_PATH:** Set to the full path of your Vivado executable.
> **VIVADO_PATH:** Vivado 실행 파일의 전체 경로로 설정하세요.
> If Vivado is already on your system `PATH`, you can omit this — the MCP server will auto-detect it.
> Vivado가 이미 시스템 `PATH`에 있는 경우 이를 생략할 수 있습니다. MCP 서버가 이를 자동 감지합니다.

**Scoping options:**
**범위 지정 옵션:**
- `--scope user` — available in all projects (recommended for the MCP server)
- `--scope user` — 모든 프로젝트에서 사용 가능(MCP 서버에 권장)
- `--scope local` (default) — only in the current project
- `--scope local`(기본값) — 현재 프로젝트에서만
- `--scope project` — shared with your team via `.mcp.json` in source control
- `--scope project` — 소스 제어에서 `.mcp.json`을 통해 팀과 공유

### Verify
### 확인

```bash
cd your-design-workspace/
claude
# Then ask: "List the available MCP tools"
```

---

## GitHub Copilot CLI
## GitHub 코파일럿 CLI

### Install
### 설치

```bash
curl -fsSL https://gh.io/copilot-install | bash

# Or via npm
npm install -g @github/copilot
```

### Configure MCP
### MCP 구성

From inside an interactive `copilot` session, use the `/mcp add` slash command and fill in the server details.
대화형 `copilot` 세션 내에서 `/mcp add` 슬래시 명령을 사용하고 서버 세부 정보를 입력합니다.

The config is stored in `~/.copilot/mcp-config.json`:
구성은 `~/.copilot/mcp-config.json`에 저장됩니다.

```json
{
  "mcpServers": {
    "vivado-mcp": {
      "command": "/path/to/vivado-mcp-server",
      "args": ["--stdio-bridge"],
      "type": "stdio",
      "env": {
        "VIVADO_PATH": "/path/to/Vivado/<version>/bin/vivado"
      }
    }
  }
}
```

### Verify
### 확인

```bash
cd your-design-workspace/
copilot
# Then ask: "List the available MCP tools"
```

---

## OpenAI Codex CLI
## OpenAI 코덱스 CLI

### Install
### 설치

```bash
npm install -g @openai/codex
```

### Configure MCP
### MCP 구성

Register the Vivado MCP server globally:
Vivado MCP 서버를 전역적으로 등록하십시오:

```bash
codex mcp add vivado-mcp \
  --env VIVADO_PATH=/path/to/Vivado/<version>/bin/vivado \
  -- /path/to/vivado-mcp-server \
  --stdio-bridge
```

This stores the config in `~/.codex/config.toml`:
그러면 `~/.codex/config.toml`에 구성이 저장됩니다.

```toml
[mcp_servers.vivado-mcp]
command = "/path/to/vivado-mcp-server"
args = ["--stdio-bridge"]

[mcp_servers.vivado-mcp.env]
VIVADO_PATH = "/path/to/Vivado/<version>/bin/vivado"
```

> **Note:** Codex CLI uses TOML, not JSON.
> **참고:** Codex CLI는 JSON이 아닌 TOML을 사용합니다.
> The CLI and IDE extension share this configuration.
> CLI와 IDE 확장은 이 구성을 공유합니다.

### Verify
### 확인

```bash
cd your-design-workspace/
codex
# Then ask: "List the available MCP tools"
```

---
