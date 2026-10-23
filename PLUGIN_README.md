<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# AMD Ross™ Agentic AI Assistant
# AMD Ross™ Agentic AI 어시스턴트

Agent skills for AMD FPGA/SoC development.
AMD FPGA/SoC 개발을 위한 에이전트 기술.

**New here?** Start with the [Getting Started](https://github.com/Xilinx/ross-ai-assistant/tree/main/docs/getting-started) guide.
**여기에 처음 오셨나요?** [시작하기](https://github.com/Xilinx/ross-ai-assistant/tree/main/docs/getting-started) 가이드로 시작하세요.
See the [docs](https://github.com/Xilinx/ross-ai-assistant/tree/main/docs) for the FAQ, MCP reference, and support.
FAQ, MCP 참조 및 지원은 [문서](https://github.com/Xilinx/ross-ai-assistant/tree/main/docs)를 참조하세요.

## Requirements
## 요구사항

The Vivado skills need the Vivado MCP server, either installed on its own or through the Vivado AI Extension for VS Code / Cursor.
Vivado 기술에는 자체적으로 설치되거나 VS 코드/커서용 Vivado AI 확장을 통해 설치되는 Vivado MCP 서버가 필요합니다.
Both are available from the [AMD Ross Agentic AI Assistant download page](https://www.amd.com/en/support/downloads/ross-agentic-ai.html).
둘 다 [AMD Ross Agentic AI Assistant 다운로드 페이지](https://www.amd.com/en/support/downloads/ross-agentic-ai.html)에서 다운로드할 수 있습니다.

## Using the skills
## 기술 사용

Invoke a skill by name, for example `/hls-optimize`.
이름으로 스킬을 호출합니다(예: `/hls-optimize`).
Skills call each other by those same names.
스킬은 서로를 같은 이름으로 부릅니다.
Some clients list them under the plugin's name, `amd-ross-agentic-ai-assistant`.
일부 클라이언트는 플러그인 이름 `amd-ross-agentic-ai-assistant` 아래에 이를 나열합니다.

Install these skills one way only.
이러한 기술은 한 가지 방법으로만 설치하십시오.
If you also installed them with `npx skills add` or by copying them, every skill is loaded twice; see [Troubleshooting](https://github.com/Xilinx/ross-ai-assistant/blob/main/docs/getting-started/install-plugin.md#troubleshooting).
`npx Skill add`를 사용하거나 복사하여 설치한 경우 모든 스킬이 두 번 로드됩니다. [문제 해결](https://github.com/Xilinx/ross-ai-assistant/blob/main/docs/getting-started/install-plugin.md#troubleshooting)을 참조하세요.

## More
## 더보기

- [Skills and examples](https://github.com/Xilinx/ross-ai-assistant#skills)
- [기술 및 예시](https://github.com/Xilinx/ross-ai-assistant#skills)
- [Vivado MCP tools reference](https://github.com/Xilinx/ross-ai-assistant/blob/main/docs/reference/vivado-mcp-tools.md)
- [Vivado MCP 도구 참조](https://github.com/Xilinx/ross-ai-assistant/blob/main/docs/reference/vivado-mcp-tools.md)
- [Vitis AI worker subagents](https://github.com/Xilinx/ross-ai-assistant/tree/main/agents)
- [Vitis AI 작업자 하위 에이전트](https://github.com/Xilinx/ross-ai-assistant/tree/main/agents)
- [FAQ](https://github.com/Xilinx/ross-ai-assistant/blob/main/docs/faq.md) and [support](https://github.com/Xilinx/ross-ai-assistant/blob/main/docs/support.md)
- [FAQ](https://github.com/Xilinx/ross-ai-assistant/blob/main/docs/faq.md) 및 [지원](https://github.com/Xilinx/ross-ai-assistant/blob/main/docs/support.md)
