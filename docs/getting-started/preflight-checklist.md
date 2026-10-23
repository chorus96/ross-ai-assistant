<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Pre-Flight Checklist
# 비행 전 체크리스트

Verify your setup is ready before using the AMD Ross Agentic AI Assistant.
AMD Ross Agentic AI Assistant를 사용하기 전에 설정이 준비되었는지 확인하세요.
This checklist works on **Linux** and **Windows**.
이 체크리스트는 **Linux** 및 **Windows**에서 작동합니다.

---

## 1.
## Vivado Installation
## 비바도 설치

| Check<br>확인 | How to Verify<br>확인 방법 |
|-------|--------------|
| Vivado is installed<br>비바도가 설치되었습니다 | Open a terminal and run: `vivado -version`<br>터미널을 열고 실행하세요: `vivado -version` |
| Version is 2026.1 (recommended)<br>버전은 2026.1입니다(권장). | Output should show `Vivado v2026.1` — other versions are architecturally compatible but 2026.1 is tested<br>출력에는 'Vivado v2026.1'이 표시되어야 합니다. 다른 버전은 구조적으로 호환되지만 2026.1은 테스트되었습니다. |
| Vivado is on PATH<br>Vivado가 PATH에 있습니다. | If `vivado -version` fails, source the settings script first: `source /tools/Xilinx/Vivado/2026.1/settings64.sh` (Linux) or run from the Vivado command prompt (Windows)<br>`vivado -version`이 실패하면 먼저 설정 스크립트를 소싱하세요: `source /tools/Xilinx/Vivado/2026.1/settings64.sh`(Linux) 또는 Vivado 명령 프롬프트(Windows)에서 실행 |

---

## 2.
## MCP Server or Extension
## MCP 서버 또는 확장

!!! warning "Choose One — Not Both"
!!! "둘 중 하나만 선택하세요." 경고
Install the **Extension** (VS Code / Cursor) **or** the **MCP Server Binary** (CLI tools).
**확장**(VS 코드/커서) **또는** **MCP 서버 바이너리**(CLI 도구)를 설치합니다.
Not both.
둘 다 아닙니다.

### Option A: Extension (VS Code / Cursor)
### 옵션 A: 확장(VS 코드/커서)

| Check<br>확인 | How to Verify<br>확인 방법 |
|-------|--------------|
| VSIX installed<br>VSIX가 설치됨 | In VS Code: **Extensions** sidebar → search "Vivado" — it should appear in the list<br>VS Code에서: **확장** 사이드바 → "Vivado"를 검색하면 목록에 표시됩니다. |
| Extension version matches<br>확장 버전이 일치합니다. | Extension version should be **v{{ mcp_version }}**<br>확장 버전은 **v{{ mcp_version }}**이어야 합니다. |
| Vivado path configured<br>Vivado 경로 구성됨 | **Settings** → search "Vivado Path" → should point to your Vivado binary<br>**설정** → "Vivado Path" 검색 → Vivado 바이너리를 가리켜야 합니다. |

### Option B: MCP Server Binary (CLI tools)
### 옵션 B: MCP 서버 바이너리(CLI 도구)

| Check<br>확인 | How to Verify<br>확인 방법 |
|-------|--------------|
| Binary downloaded<br>바이너리 다운로드됨 | Check the file exists: `ls ~/bin/vivado-mcp-server-*` (Linux) or check your chosen install directory (Windows)<br>파일이 있는지 확인하십시오: `ls ~/bin/vivado-mcp-server-*`(Linux) 또는 선택한 설치 디렉토리를 확인하십시오(Windows). |
| Binary is executable<br>바이너리는 실행 가능합니다. | Linux: `chmod +x ~/bin/vivado-mcp-server-*` — Windows: no action needed<br>Linux: `chmod +x ~/bin/vivado-mcp-server-*` — Windows: 필요한 조치 없음 |
| Binary responds<br>바이너리 응답 | Run: `echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}'<br>실행: `echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capability":{},"clientInfo":{"name":"test","version":"1.0"}}}' | ./vivado-mcp-server-linux-amd64-{{ mcp_version }}` — should return a JSON response<br>./vivado-mcp-server-linux-amd64-{{ mcp_version }}` — JSON 응답을 반환해야 합니다. |

---

## 3.
## MCP Client Configuration
## MCP 클라이언트 구성

Your AI client needs a config file that tells it where to find the MCP server.
AI 클라이언트에는 MCP 서버를 찾을 수 있는 위치를 알려주는 구성 파일이 필요합니다.
Check the location for your tool:
도구 위치를 확인하세요.

=== "VS Code"
=== "VS 코드"

**File:** `.vscode/mcp.json` (workspace) or `~/.config/Code/User/mcp.json` (global)
**파일:** `.vscode/mcp.json`(작업 공간) 또는 `~/.config/Code/User/mcp.json`(전역)

Verify it contains a `vivado-mcp-server` entry:
'vivado-mcp-server' 항목이 포함되어 있는지 확인하세요.
    ```json
    {
      "servers": {
        "vivado": {
          "command": "/path/to/vivado-mcp-server-linux-amd64-{{ mcp_version }}"
        }
      }
    }
    ```

!!! tip
!!! 팁
If you installed the **VSIX extension**, this config is created automatically — no manual setup needed.
**VSIX 확장**을 설치한 경우 이 구성이 자동으로 생성되므로 수동 설정이 필요하지 않습니다.

=== "Cursor"
=== "커서"

**File:** `.cursor/mcp.json` (workspace) or `~/.cursor/mcp.json` (global)
**파일:** `.cursor/mcp.json`(작업 공간) 또는 `~/.cursor/mcp.json`(전역)

Same JSON format as VS Code above.
위의 VS Code와 동일한 JSON 형식입니다.

=== "Claude Desktop"
=== "클로드 데스크탑"

**File:**
**파일:**

    - Linux: `~/.config/Claude/claude_desktop_config.json`
    - 리눅스: `~/.config/Claude/claude_desktop_config.json`
    - Windows: `%APPDATA%\Claude\claude_desktop_config.json`
    - Windows: `%APPDATA%\Claude\claude_desktop_config.json`

=== "Claude Code"
=== "클로드 코드"

**File:** `.mcp.json` (project) or `~/.claude.json` (user)
**파일:** `.mcp.json`(프로젝트) 또는 `~/.claude.json`(사용자)

Verify with: `cat .mcp.json | grep vivado`
다음으로 확인하세요: `cat .mcp.json | 그렙 비바도`

=== "Windsurf"
=== "윈드서핑"

**File:** `~/.codeium/windsurf/mcp_config.json`
**파일:** `~/.codeium/windsurf/mcp_config.json`

=== "GitHub Copilot CLI"
=== "GitHub Copilot CLI"

MCP servers are passed at invocation:
MCP 서버는 호출 시 전달됩니다.
    ```bash
    gh copilot --mcp-server vivado=/path/to/vivado-mcp-server-linux-amd64-{{ mcp_version }}
    ```

=== "Codex CLI"
=== "코덱스 CLI"

**File:** `~/.codex/config.toml` or `.codex/config.toml` (project)
**파일:** `~/.codex/config.toml` 또는 `.codex/config.toml`(프로젝트)

    ```toml
    [mcp.vivado]
    command = "/path/to/vivado-mcp-server-linux-amd64-{{ mcp_version }}"
    ```

---

## 4.
## AI Client Installed
## AI 클라이언트 설치됨

| Client<br>클라이언트 | How to Check<br>확인 방법 |
|--------|-------------|
| **VS Code**<br>**VS 코드** | `code --version`<br>`코드 --버전` |
| **Cursor**<br>**커서** | `cursor --version`<br>`커서 --버전` |
| **Claude Code**<br>**클로드 코드** | `claude --version`<br>`클로드 --버전` |
| **GitHub Copilot CLI**<br>**GitHub Copilot CLI** | `gh copilot --version`<br>`gh 부조종사 --버전` |
| **Codex CLI**<br>**코덱스 CLI** | `codex --version`<br>`코덱스 --버전` |
| **Windsurf**<br>**윈드서핑** | `windsurf --version`<br>`윈드서프 --버전` |
| **Claude Desktop**<br>**클로드 데스크탑** | Open the app — check the MCP icon in the chat input area<br>앱 열기 - 채팅 입력 영역에서 MCP 아이콘을 확인하세요. |

---

## 5.
## Knowledge Base
## 기술 자료

The Knowledge Base provides RAG-powered documentation search via `vivado_doc_search`.
기술 자료는 `vivado_doc_search`를 통해 RAG 기반 문서 검색을 제공합니다.
By default it is hosted by **AMD** and publicly accessible — no VPN or local setup required.
기본적으로 **AMD**에서 호스팅되며 공개적으로 액세스할 수 있습니다. VPN이나 ​​로컬 설정이 필요하지 않습니다.
The MCP server connects to it automatically.
MCP 서버가 자동으로 연결됩니다.
For privacy, telemetry, and air-gapped options, see the [FAQ](../faq.md#security-privacy-and-the-knowledge-base) and the [local knowledge base](../local-kb/README.md).
개인정보 보호, 원격 측정 및 에어갭 옵션에 대해서는 [FAQ](../faq.md#security-privacy-and-the-knowledge-base) 및 [로컬 기술 자료](../local-kb/README.md)를 참조하세요.

| Check<br>확인 | How to Verify<br>확인 방법 |
|-------|--------------|
| `vivado_doc_search` responds<br>`vivado_doc_search`가 응답합니다. | In your AI chat, ask: *"Search AMD docs for async clock domain crossings"* — you should get cited results from UG903, UG949, etc.<br>AI 채팅에서 다음과 같이 질문하세요. *"비동기 클럭 도메인 교차에 대한 AMD 문서 검색"* — UG903, UG949 등에서 인용된 결과를 받아야 합니다. |

---

## Quick Smoke Test
## 빠른 연기 테스트

After verifying all the above, run this in your AI chat to confirm end-to-end:
위의 사항을 모두 확인한 후 AI 채팅에서 다음을 실행하여 엔드투엔드를 확인하세요.

> *"What version of Vivado do I have?"*
> *"내가 가지고 있는 Vivado 버전은 무엇인가요?"*

If the agent calls `vivado_execute` and returns your Vivado version, everything is working.
에이전트가 `vivado_execute`를 호출하고 Vivado 버전을 반환하면 모든 것이 작동하는 것입니다.

<p class="sphinxhide" align="center"><sub>Copyright © 2026 Advanced Micro Devices, Inc</sub></p>
<p class="sphinxhide" align="center"><sup><a href="https://www.amd.com/en/corporate/copyright">Terms and Conditions</a></sup></p>
