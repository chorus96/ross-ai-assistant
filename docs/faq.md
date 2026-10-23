<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Frequently Asked Questions
# 자주 묻는 질문

## General
## 일반

**What is the Vivado MCP Server?**
**Vivado MCP 서버란 무엇입니까?**

The MCP (Model Context Protocol) Server is a bridge that connects AI agents to a running Vivado session.
MCP(Model Context Protocol) 서버는 AI 에이전트를 실행 중인 Vivado 세션에 연결하는 브리지입니다.
It translates agent requests into Vivado Tcl commands and returns structured results, enabling AI-assisted FPGA design workflows.
에이전트 요청을 Vivado Tcl 명령으로 변환하고 구조화된 결과를 반환하여 AI 지원 FPGA 설계 워크플로를 활성화합니다.

**Which Vivado versions are supported?**
**어떤 Vivado 버전이 지원되나요?**

Vivado 2020.2 or later.
비바도 2020.2 이상.
2026.1 is the version the flow is tested against, and the MCP server communicates through Vivado's standard Tcl interface, so it is architecturally compatible with released versions either side of that.
2026.1은 흐름이 테스트되는 버전이며 MCP 서버는 Vivado의 표준 Tcl 인터페이스를 통해 통신하므로 그 양쪽에서 릴리스된 버전과 아키텍처적으로 호환됩니다.
Individual skills and examples note their own minimum supported versions where they differ.
개별 기술과 예제는 서로 다른 경우 최소 지원 버전을 명시합니다.

**Does the AI modify my design files directly?**
**AI가 내 디자인 파일을 직접 수정하나요?**

The AI agent executes Vivado Tcl commands through the MCP server.
AI 에이전트는 MCP 서버를 통해 Vivado Tcl 명령을 실행합니다.
It can modify your project within the Vivado session (adding IP, changing constraints, etc.), but it does not directly edit HDL source files unless you explicitly ask it to through your IDE's file editing capabilities.
Vivado 세션 내에서 프로젝트를 수정할 수 있지만(IP 추가, 제약 조건 변경 등), IDE의 파일 편집 기능을 통해 명시적으로 요청하지 않는 한 HDL 소스 파일을 직접 편집하지 않습니다.

**Do HLS skills need the MCP server?**
**HLS 기술에는 MCP 서버가 필요합니까?**

No.
HLS skills (matlab-to-cpp, hls-architect, hls-optimize) work directly with your source code and HLS project files.
HLS 기술(matlab-to-cpp, hls-architect, hls-optimize)은 소스 코드 및 HLS 프로젝트 파일에서 직접 작동합니다.
They provide code generation, architecture guidance, and optimization recommendations without requiring a live Vivado/HLS session for basic usage.
기본 사용을 위해 라이브 Vivado/HLS 세션이 필요 없이 코드 생성, 아키텍처 지침 및 최적화 권장 사항을 제공합니다.
The `hls-run-flow` build skill invokes Vitis HLS for compilation via standard command-line tools (`v++`, `vitis-run`), not MCP.
`hls-run-flow` 빌드 기술은 MCP가 아닌 표준 명령줄 도구(`v++`, `vitis-run`)를 통해 컴파일하기 위해 Vitis HLS를 호출합니다.

**Does this workflow with AMD Ross™ AI assistant require an additional license?**
**AMD Ross™ AI 어시스턴트를 사용하는 이 워크플로우에는 추가 라이선스가 필요합니까?**

No additional license is required.
추가 라이센스가 필요하지 않습니다.
You only need the applicable licenses for the AMD tools you use (for example Vivado or Vitis HLS).
사용하는 AMD 도구(예: Vivado 또는 Vitis HLS)에 해당하는 라이선스만 필요합니다.

**Which coding agents and models can I use with AMD skills?**
**AMD 기술에는 어떤 코딩 에이전트와 모델을 사용할 수 있나요?**

Use a current coding agent (Cursor, Claude Code, Copilot, or similar) that can load Agent Skills and call tools (including AMD MCP when the skill needs a live Vivado session).
에이전트 스킬을 로드하고 도구(스킬에 라이브 Vivado 세션이 필요한 경우 AMD MCP 포함)를 호출할 수 있는 최신 코딩 에이전트(Cursor, Claude Code, Copilot 또는 유사)를 사용하십시오.
Use a current, large-context coding model your company already allows — not a small or chat-only model.
소규모 또는 채팅 전용 모델이 아닌, 회사에서 이미 허용하는 현재의 대규모 컨텍스트 코딩 모델을 사용하세요.

Current GPT-5 Sol–class and Claude Opus 5–class coding models provide the capabilities these skills require.
현재 GPT-5 Sol 클래스 및 Claude Opus 5 클래스 코딩 모델은 이러한 기술에 필요한 기능을 제공합니다.

## Setup & Configuration
## 설정 및 구성

**Can I use the MCP server without the VS Code extension?**
**VS Code 확장 없이 MCP 서버를 사용할 수 있나요?**

Yes.
그렇습니다.
The MCP server works with any MCP-compatible client: Cursor, Claude Code, GitHub Copilot CLI, Codex CLI, and others.
MCP 서버는 Cursor, Claude Code, GitHub Copilot CLI, Codex CLI 등 모든 MCP 호환 클라이언트와 작동합니다.
The VS Code extension adds convenience features but isn't required.
VS Code 확장은 편의 기능을 추가하지만 필수는 아닙니다.

**Do I need to start Vivado myself first?**
**먼저 Vivado를 직접 시작해야 하나요?**

No.
The `vivado_start` tool launches Vivado for you, in Tcl mode by default — just ask the agent to start a session.
`vivado_start` 도구는 기본적으로 Tcl 모드에서 Vivado를 시작합니다. 에이전트에게 세션 시작을 요청하기만 하면 됩니다.
`vivado_connect` is for the other direction: attaching to a Vivado instance that is already running with its Tcl webserver.
`vivado_connect`는 다른 방향을 위한 것입니다. 즉, Tcl 웹 서버로 이미 실행 중인 Vivado 인스턴스에 연결하는 것입니다.

**Can I run Vivado on a different machine?**
**Vivado를 다른 머신에서 실행할 수 있나요?**

Yes.
그렇습니다.
Ask the agent to start the session remotely: `vivado_ssh` runs it on a named host, and `vivado_lsf` submits it to an LSF cluster node.
에이전트에게 세션을 원격으로 시작하도록 요청합니다. `vivado_ssh`는 명명된 호스트에서 세션을 실행하고 `vivado_lsf`는 이를 LSF 클러스터 노드에 제출합니다.
Both are Linux only, and both need the working directory to be on a filesystem the remote machine reaches at the same path.
둘 다 Linux 전용이며 원격 시스템이 동일한 경로에 도달하는 파일 시스템에 작업 디렉터리가 있어야 합니다.
`vivado_lsf` with `action='check'` reports what is missing from an LSF setup.
`action='check'`가 포함된 `vivado_lsf`는 LSF 설정에서 누락된 내용을 보고합니다.

**I see "Connection refused" — what's wrong?**
**"연결 거부됨"이 표시됩니다. 무엇이 문제인가요?**

Usually the session is gone, or was never started.
일반적으로 세션이 사라졌거나 시작되지 않았습니다.
Ask the agent to list sessions (`vivado_list_sessions`) and start one if the list is empty.
에이전트에게 세션(`vivado_list_sessions`)을 나열하고 목록이 비어 있으면 세션을 시작하도록 요청하세요.
If a session is listed but unreachable, `vivado_cleanup` with `operation='health'` reports its state and `operation='cleanup_stale'` clears dead entries.
세션이 나열되어 있지만 연결할 수 없는 경우 `Operation='health'`인 `vivado_cleanup`은 해당 상태를 보고하고 `Operation='cleanup_stale'`은 죽은 항목을 지웁니다.

**How do I update to a new version?**
**새 버전으로 어떻게 업데이트하나요?**

If you installed the extension from the Marketplace, it updates itself.
Marketplace에서 확장 프로그램을 설치한 경우 자동으로 업데이트됩니다.
For a manual install, download the latest VSIX and reinstall it; a standalone MCP server binary is replaced in place.
수동 설치의 경우 최신 VSIX를 다운로드하여 다시 설치하세요. 독립형 MCP 서버 바이너리가 제자리에서 교체됩니다.

**How do I share the Vivado MCP server among multiple users in my organization?**
**내 조직의 여러 사용자가 Vivado MCP 서버를 어떻게 공유합니까?**

Place the MCP server binary in a shared location accessible by all users (e.g., a network drive or shared filesystem).
모든 사용자가 액세스할 수 있는 공유 위치(예: 네트워크 드라이브 또는 공유 파일 시스템)에 MCP 서버 바이너리를 배치합니다.
Each user then configures their own `mcp.json` file — as documented in the [Getting Started](getting-started/) guides for their IDE or CLI — pointing the `"command"` field to the shared binary path.
그런 다음 각 사용자는 IDE 또는 CLI에 대한 [시작하기](getting-started/) 가이드에 설명된 대로 자신의 `mcp.json` 파일을 구성하고 `"command"` 필드를 공유 바이너리 경로로 지정합니다.

No per-user installation is needed; only the `mcp.json` configuration differs per user.
사용자별 설치가 필요하지 않습니다. 사용자마다 `mcp.json` 구성만 다릅니다.

> **Tip:** For questions, tips, and product updates, use [AMD Adaptive Support](https://adaptivesupport.amd.com/s/?language=en_US).
> **팁:** 질문, 팁, 제품 업데이트는 [AMD Adaptive Support](https://adaptivesupport.amd.com/s/?언어=en_US)를 이용하세요.

## Skills
## 기술

**What is a SKILL.md file?**
**SKILL.md 파일이란 무엇입니까?**

A SKILL.md is a structured instruction file that teaches an AI agent how to perform a specific FPGA design task.
SKILL.md는 AI 에이전트에게 특정 FPGA 설계 작업을 수행하는 방법을 가르치는 구조화된 지침 파일입니다.
It contains step-by-step workflows, interpretation guides, and fix recommendations.
여기에는 단계별 워크플로, 해석 가이드 및 수정 권장 사항이 포함되어 있습니다.
Agents read these files to gain domain expertise.
상담원은 도메인 전문 지식을 얻기 위해 이러한 파일을 읽습니다.

**Where do I put skill files?**
**스킬 파일은 어디에 넣나요?**

Place them under `.claude/skills/` in your workspace.
작업 공간의 `.claude/skills/` 아래에 배치하세요.
The AI agent automatically discovers and reads them from that location.
AI 에이전트는 해당 위치에서 자동으로 검색하고 읽습니다.

```
your-workspace/
└── .claude/
    └── skills/
        └── your-skill/
            └── SKILL.md
```

**Can I write my own skills?**
**스킬을 직접 작성할 수 있나요?**

Yes.
그렇습니다.
Skills follow the open [Agent Skills](https://agentskills.io) standard: each is a `SKILL.md` Markdown file with YAML frontmatter (at minimum a `name` and `description`) followed by the instructions.
기술은 공개 [에이전트 기술](https://agentskills.io) 표준을 따릅니다. 각 기술은 YAML 머리말(최소 `이름` 및 `설명`)과 지침이 포함된 `SKILL.md` 마크다운 파일입니다.
See the standard for the full format, and browse the `skills/` folder in this package for working examples to model yours on.
전체 형식에 대한 표준을 확인하고, 모델링할 작업 예제를 보려면 이 패키지의 `skills/` 폴더를 찾아보세요.

**What's the difference between user-facing and helper skills?**
**사용자 대상 기술과 도우미 기술의 차이점은 무엇입니까?**

User-facing skills (matlab-to-cpp, hls-architect, hls-optimize) are what you invoke directly by asking the agent.
사용자 대면 기술(matlab-to-cpp, hls-architect, hls-optimize)은 에이전트에 요청하여 직접 호출하는 것입니다.
Helper skills (dataflow checks, burst inference, array partitioning, report extraction, etc.) are automatically invoked by core skills as needed — you don't need to know about them or install them separately.
도우미 기술(데이터 흐름 검사, 버스트 추론, 배열 분할, 보고서 추출 등)은 필요에 따라 핵심 기술에 의해 자동으로 호출됩니다. 이에 대해 알거나 별도로 설치할 필요가 없습니다.

**What does a skill stage of Beta mean?**
**베타의 스킬 단계는 무엇을 의미하나요?**

Beta indicates that the skill has undergone preliminary validation and testing but is still maturing.
베타는 해당 기술이 예비 검증 및 테스트를 거쳤지만 아직 성숙 단계에 있음을 나타냅니다.
Functionality, APIs, and implementation details may evolve in future releases.
기능, API 및 구현 세부 사항은 향후 릴리스에서 발전할 수 있습니다.

## Troubleshooting
## 문제 해결

**The agent isn't using the skill I expected**
**상담원이 내가 기대했던 기술을 사용하지 않습니다**

Make sure the SKILL.md is in the correct location (`.claude/skills/skill-name/SKILL.md`) and that you reference the skill by name in your prompt.
SKILL.md가 올바른 위치(`.claude/skills/skill-name/SKILL.md`)에 있고 프롬프트에서 이름으로 스킬을 참조하는지 확인하세요.
You can also ask the agent: "What skills are available?"
상담원에게 "어떤 기술을 사용할 수 있나요?"라고 물어볼 수도 있습니다.

**Vivado commands are failing through the agent**
**Vivado 명령이 에이전트를 통해 실패합니다**

Check the Vivado Tcl console for error messages.
Vivado Tcl 콘솔에서 오류 메시지를 확인하세요.
Common causes: design not open, wrong design state (e.g., trying to route before placing), or missing source files.
일반적인 원인: 디자인이 열려 있지 않거나, 잘못된 디자인 상태(예: 배치하기 전에 라우팅을 시도함) 또는 소스 파일이 누락되었습니다.

**The agent seems slow**
**에이전트가 느린 것 같습니다**

Complex Vivado operations (synthesis, implementation) take time regardless of the AI layer.
복잡한 Vivado 작업(합성, 구현)은 AI 계층에 관계없이 시간이 걸립니다.
The MCP server streams results, but you'll still wait for Vivado to finish.
MCP 서버는 결과를 스트리밍하지만 여전히 Vivado가 완료될 때까지 기다립니다.
For faster iteration, use targeted operations like `synth_design -lint` instead of full synthesis.
더 빠른 반복을 위해 전체 합성 대신 `synth_design -lint`와 같은 대상 작업을 사용하세요.

## Security, Privacy, and the Knowledge Base
## 보안, 개인 정보 보호 및 기술 자료

Ross AI assistant combines three things that are often confused: **your AI client and model** (Copilot, Claude, Cursor, etc.), **AMD MCP servers and skills** (local tool access), and **documentation search** (hosted or local knowledge base).
Ross AI 보조자는 종종 혼동되는 세 가지를 결합합니다. **AI 클라이언트 및 모델**(Copilot, Claude, Cursor 등), **AMD MCP 서버 및 기술**(로컬 도구 액세스), **문서 검색**(호스팅 또는 로컬 지식 기반).
Privacy depends on which layer you are asking about.
개인정보 보호는 귀하가 요청하는 레이어에 따라 다릅니다.

### Privacy: what stays on your machine
### 개인 정보 보호: 컴퓨터에 남아 있는 내용

**Does Vivado MCP upload my RTL, project, or Tcl to AMD?**
**Vivado MCP는 내 RTL, 프로젝트 또는 Tcl을 AMD에 업로드합니까?**

No.
The Vivado MCP Server runs on a machine you control.
Vivado MCP 서버는 귀하가 제어하는 시스템에서 실행됩니다.
It talks to Vivado locally and returns results to your AI client.
로컬로 Vivado와 통신하고 AI 클라이언트에 결과를 반환합니다.
Your design files, netlists, constraints, bitstreams, and the text of Tcl commands issued through `vivado_execute` are **not** uploaded to AMD as part of normal MCP operation.
'vivado_execute'를 통해 실행된 디자인 파일, 넷리스트, 제약 조건, 비트스트림 및 Tcl 명령 텍스트는 일반적인 MCP 작업의 일부로 AMD에 업로드되지 **않습니다**.

**Does the Vivado MCP Server collect telemetry?**
**Vivado MCP 서버는 원격 측정을 수집합니까?**

No.
The Vivado MCP Server does not send telemetry or usage data back to AMD.
Vivado MCP 서버는 원격 측정 또는 사용 데이터를 AMD로 다시 보내지 않습니다.
The Vivado MCP server communicates only with the local Vivado installation and does not transmit design information, prompts, or user interactions to AMD.
Vivado MCP 서버는 로컬 Vivado 설치와만 통신하며 설계 정보, 프롬프트 또는 사용자 상호 작용을 AMD로 전송하지 않습니다.

**Does the Documentation Search MCP collect telemetry?**
**문서 검색 MCP는 원격 분석을 수집합니까?**

No.
It does not send telemetry from your machine.
컴퓨터에서 원격 분석을 보내지 않습니다.
The hosted documentation service keeps server-side usage statistics only: which documentation resources were accessed, and aggregate usage counts.
호스팅된 문서 서비스는 서버측 사용 통계(액세스된 문서 리소스 및 집계된 사용 횟수)만 유지합니다.
Those statistics do not include user prompts, search queries, design content, or personally identifiable design information.
이러한 통계에는 사용자 프롬프트, 검색어, 디자인 콘텐츠 또는 개인 식별이 가능한 디자인 정보가 포함되지 않습니다.

**Are user prompts or design data collected?**
**사용자 프롬프트나 설계 데이터가 수집되나요?**

No.
User prompts, search queries, RTL code, design content, and other customer intellectual property are not included in those server-side usage statistics.
사용자 프롬프트, 검색 쿼리, RTL 코드, 디자인 콘텐츠 및 기타 고객 지적 재산은 해당 서버측 사용 통계에 포함되지 않습니다.

**Does AMD train AI models on my prompts or designs?**
**AMD는 내 프롬프트나 디자인에 따라 AI 모델을 교육합니까?**

AMD does not provide the foundation LLM and does **not** use your prompts, inputs, or outputs to train AMD models.
AMD는 기초 LLM을 제공하지 않으며 AMD 모델을 교육하기 위해 프롬프트, 입력 또는 출력을 사용하지 **않습니다**.
Ross AI assistant supplies skills, MCP tools, and optional documentation search — it plugs into **your** LLM configuration.
Ross AI 도우미는 기술, MCP 도구 및 선택적 문서 검색을 제공하며 **귀하의** LLM 구성에 연결됩니다.
What your model provider retains or trains on is governed by **that provider's** enterprise agreement and policies, not by the Ross AI assistant license.
모델 공급자가 유지하거나 교육하는 내용은 Ross AI 보조 라이선스가 아닌 **해당 공급자**의 기업 계약 및 정책에 따라 결정됩니다.

### Privacy: cloud knowledge base (`vivado_doc_search`)
### 개인 정보 보호: 클라우드 지식 기반(`vivado_doc_search`)

**What is the hosted knowledge base?**
**호스팅된 지식 베이스란 무엇입니까?**

`vivado_doc_search` comes from a **separate MCP server**, `amd-doc-search`, rather than from the Vivado MCP server.
`vivado_doc_search`는 Vivado MCP 서버가 아닌 **별도의 MCP 서버**, `amd-doc-search`에서 옵니다.
When your agent calls it, that server queries AMD's **hosted documentation index** over the network.
에이전트가 이를 호출하면 해당 서버는 네트워크를 통해 AMD의 **호스팅 문서 색인**을 쿼리합니다.
The index contains **published AMD documentation** (user guides, programming guides, Answer Records, and related web content) — not your project files.
색인에는 프로젝트 파일이 아닌 **게시된 AMD 문서**(사용자 가이드, 프로그래밍 가이드, 응답 기록 및 관련 웹 콘텐츠)가 포함되어 있습니다.

The extension registers both servers, so the tool is there automatically.
확장은 두 서버를 모두 등록하므로 도구가 자동으로 존재합니다.
An MCP-only setup adds `amd-doc-search` itself, and the tool is simply absent until it does — which is also the simplest way to make sure no documentation query leaves your machine.
MCP 전용 설정에는 `amd-doc-search` 자체가 추가되며, 도구는 추가될 때까지 존재하지 않습니다. 이는 문서 쿼리가 컴퓨터에서 나가지 않도록 하는 가장 간단한 방법이기도 합니다.

**Does AMD store my documentation search questions?**
**AMD는 내 문서 검색 질문을 저장합니까?**

No.
AMD does **not** store the text of your `vivado_doc_search` queries on the cloud knowledge base service.
AMD는 클라우드 지식 기반 서비스에 'vivado_doc_search' 쿼리 텍스트를 저장하지 **않습니다**.

**Which MCP tools reach AMD's network?**
**어떤 MCP 도구가 AMD 네트워크에 연결됩니까?**

Only `vivado_doc_search`, and it belongs to the `amd-doc-search` server rather than the Vivado one.
`vivado_doc_search`만 해당되며 Vivado 서버가 아닌 `amd-doc-search` 서버에 속합니다.
Every tool the Vivado MCP server provides communicates between your local Vivado session and your MCP client only.
Vivado MCP 서버가 제공하는 모든 도구는 로컬 Vivado 세션과 MCP 클라이언트 사이에서만 통신합니다.

### Privacy: air-gapped deployments
### 개인정보 보호: 에어갭 배포

**Can we avoid sending documentation searches to AMD's cloud?**
**문서 검색을 AMD의 클라우드로 보내는 것을 피할 수 있습니까?**

Yes.
그렇습니다.
Deploy the [AMD Embedded Local Knowledge Base](local-kb/README.md) (offline RAG Docker) on your network and configure the MCP server to use that local endpoint for `vivado_doc_search`.
네트워크에 [AMD 임베디드 로컬 기술 자료](local-kb/README.md)(오프라인 RAG Docker)를 배포하고 `vivado_doc_search`에 해당 로컬 엔드포인트를 사용하도록 MCP 서버를 구성합니다.
Documentation retrieval then stays on your infrastructure.
그러면 문서 검색이 인프라에 유지됩니다.

**Is local knowledge base + a cloud LLM fully air-gapped?**
**로컬 지식 기반 + 클라우드 LLM이 완전히 에어갭되어 있나요?**

No.
If you use GitHub Copilot or another cloud model, your question and the documentation passages the model reads can still go to that provider.
GitHub Copilot 또는 다른 클라우드 모델을 사용하는 경우 모델이 읽는 질문과 문서 구절은 해당 공급자에게 계속 전달될 수 있습니다.
A **fully** air-gapped workflow requires both local documentation search **and** a local tool-calling LLM.
**완전히** 에어갭이 있는 워크플로에는 로컬 문서 검색 **과** 로컬 도구 호출 LLM이 모두 필요합니다.
See [Using the RAG database with a frontier model](local-kb/frontier-model.md) and [Local LLM deployment](local-kb/local-llm.md).
[프론티어 모델과 함께 RAG 데이터베이스 사용](local-kb/frontier-model.md) 및 [로컬 LLM 배포](local-kb/local-llm.md)를 참조하세요.

**Our IT policy blocks cloud AI or proprietary data leaving the site — what should we use?**
**저희 IT 정책은 클라우드 AI 또는 독점 데이터가 사이트에서 나가는 것을 차단합니다. 무엇을 사용해야 합니까?**

| Concern<br>우려 | Typical approach<br>일반적인 접근 방식 |
|--------|------------------|
| Doc search must not call AMD cloud<br>문서 검색에서 AMD 클라우드를 호출하면 안 됩니다. | [Local knowledge base](local-kb/README.md)<br>[로컬 지식 베이스](local-kb/README.md) |
| LLM must not leave the site<br>LLM은 사이트를 떠나서는 안 됩니다. | Local LLM with tool-calling support<br>도구 호출 지원이 포함된 로컬 LLM |
| Vivado automation<br>비바도 자동화 | Vivado MCP (local); restrict which MCP tools are enabled if needed<br>비바도 MCP(로컬); 필요한 경우 활성화할 MCP 도구를 제한합니다. |

### Privacy: AMD NDA documentation
### 개인 정보 보호: AMD NDA 문서

**Does the hosted knowledge base include AMD NDA or EA-restricted documents?**
**호스팅된 기술 자료에 AMD NDA 또는 EA 제한 문서가 포함되어 있습니까?**

No.
The hosted index is **generally published** AMD documentation only.
호스팅된 인덱스는 **일반적으로 게시**된 AMD 문서에만 해당됩니다.
AMD content that is under NDA, Early Access, or another restricted program is **not** in that public index.
NDA, Early Access 또는 기타 제한된 프로그램이 적용되는 AMD 콘텐츠는 해당 공개 색인에 **아닙니다**.

If your program entitles you to restricted AMD documents, keep those files under your own access controls (for example your EA download workspace).
프로그램에서 제한된 AMD 문서에 대한 권한을 부여하는 경우 해당 파일을 자체 액세스 제어(예: EA 다운로드 작업 공간)에 보관하십시오.
Do not assume `vivado_doc_search` against AMD's cloud can retrieve them.
AMD 클라우드에 대한 'vivado_doc_search'가 이를 검색할 수 있다고 가정하지 마십시오.

### Privacy: your design data and export control
### 개인정보 보호: 설계 데이터 및 수출 통제

**Does AMD's knowledge base hold my company's RTL or other customer-owned files?**
**AMD의 지식 기반에는 회사의 RTL 또는 기타 고객 소유 파일이 보관되어 있습니까?**

No.
Your project files, RTL, constraints, and similar artifacts are **your** content.
귀하의 프로젝트 파일, RTL, 제약 조건 및 유사한 아티팩트가 **귀하의** 콘텐츠입니다.
AMD does not ingest them into the hosted or local AMD documentation knowledge bases.
AMD는 이를 호스팅되거나 로컬 AMD 문서 지식 기반으로 수집하지 않습니다.
You remain the data owner and decide whether any of that material is placed in a prompt, a workspace the agent can read, or a cloud LLM.
귀하는 데이터 소유자로 남아 해당 자료를 프롬프트, 상담원이 읽을 수 있는 작업 영역 또는 클라우드 LLM에 배치할지 여부를 결정합니다.

**Who is responsible for export-controlled (ITAR/EAR) design data?**
**수출 통제(ITAR/EAR) 설계 데이터는 누가 책임지나요?**

You are.
당신은 그렇습니다.
Ross AI assistant does not change your obligation to handle controlled goods correctly.
Ross AI 비서는 통제된 상품을 올바르게 처리해야 하는 귀하의 의무를 변경하지 않습니다.
Do not submit export-controlled design content to cloud LLM services unless your compliance team has approved that path.
규정 준수 팀이 해당 경로를 승인하지 않는 한 수출 통제 설계 콘텐츠를 클라우드 LLM 서비스에 제출하지 마세요.

### Still have questions?
### 아직도 질문이 있으신가요?

For product support, see [Support](support.md) and [AMD Adaptive Support](https://adaptivesupport.amd.com/s/?language=en_US).
제품 지원은 [지원](support.md) 및 [AMD 적응형 지원](https://adaptivesupport.amd.com/s/?언어=en_US)을 참조하세요.

---
