<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Using the RAG Database with a Frontier Model (GitHub Copilot)
# Frontier 모델과 함께 RAG 데이터베이스 사용(GitHub Copilot)

**Point a frontier cloud model at your locally-deployed AMD Embedded RAG
**로컬에 배포된 AMD Embedded RAG에서 최첨단 클라우드 모델을 가리킵니다.
database — it runs on the everyday laptop or workstation you already have,
데이터베이스 — 이미 보유하고 있는 일상적인 노트북이나 워크스테이션에서 실행됩니다.
since the answering model is hosted in the cloud and only the retrieval
응답 모델은 클라우드에서 호스팅되고 검색만 가능하므로
layer runs locally.**
레이어는 로컬로 실행됩니다.**

> **Prerequisite:** deploy the RAG database first — see
> **전제 조건:** RAG 데이터베이스를 먼저 배포하십시오.
> [RAG Database Setup](README.md).
> [RAG 데이터베이스 설정](README.md).
> This document
> 이 문서
> assumes `amd-embedded-doc-search` is already running and reachable at
> `amd-embedded-doc-search`가 이미 실행 중이고 다음 위치에서 연결할 수 있다고 가정합니다.
> `http://127.0.0.1:<DOCSEARCH_MCP_PORT>/mcp/doc-search`.
> `http://127.0.0.1:<DOCSEARCH_MCP_PORT>/mcp/doc-search`.

## Who It's For
## 누구를 위한 것인가

This path suits teams that want to run the answering model in the cloud
이 경로는 클라우드에서 응답 모델을 실행하려는 팀에 적합합니다.
rather than host it locally.
로컬에서 호스팅하는 대신
If you have a standard laptop or workstation
표준 노트북이나 워크스테이션이 있는 경우
and already have access to a frontier model — most commonly **GitHub
이미 프론티어 모델에 액세스할 수 있습니다 — 가장 일반적으로 **GitHub
Copilot** in VS Code — you can get fully grounded, cited answers against the
VS Code의 Copilot** — 완전히 근거가 있고 인용된 답변을 얻을 수 있습니다.
entire AMD Embedded documentation corpus (Vivado, Vitis, PDM, ChipScope,
전체 AMD 임베디드 문서 자료(Vivado, Vitis, PDM, ChipScope,
System Software, example designs, Wiki, Answer Records).
시스템 소프트웨어, 예제 디자인, Wiki, Answer Records).
The RAG database
RAG 데이터베이스
(embedding model, vector search, MCP server) still runs 100% locally; only
(임베딩 모델, 벡터 검색, MCP 서버)는 여전히 로컬에서 100% 실행됩니다. 만
the final answer-generation step is handled by the cloud model you already
최종 답변 생성 단계는 이미 클라우드 모델에 의해 처리됩니다.
have a subscription/sign-in for.
구독/로그인이 있습니다.

> **Already using the Vivado MCP Server's built-in `vivado_doc_search`
> **이미 Vivado MCP 서버에 내장된 `vivado_doc_search`를 사용하고 있습니다.
> tool?** That tool queries AMD's hosted knowledge base live over the
> 도구?** 해당 도구는 AMD가 호스팅하는 지식 기반을 실시간으로 쿼리합니다.
> internet, so with an internet connection it's already current and needs
> 인터넷이 연결되어 있으면 이미 최신 상태이고 필요합니다.
> no local deployment.
> 로컬 배포가 없습니다.
> This path isn't about replacing it — see
> 이 경로는 교체에 관한 것이 아닙니다.
> [Why use local RAG with a cloud model?](#why-local-rag)
> [클라우드 모델과 함께 로컬 RAG를 사용하는 이유는 무엇입니까?](#why-local-rag)
> for where it helps.
> 그것이 어디에 도움이 되는지.

## Not Air-Gapped
## 에어 갭이 없음

Use this path only where a policy-approved outbound connection to GitHub
GitHub에 대한 정책 승인 아웃바운드 연결이 있는 경우에만 이 경로를 사용하십시오.
Copilot is acceptable.
부조종사는 허용됩니다.
The documentation **retrieval** stays local, but
문서 **검색**은 로컬로 유지되지만
**answer generation** does not.
**답변 생성**은 그렇지 않습니다.

| | RAG database ([core doc](README.md))<br>RAG 데이터베이스([핵심 문서](README.md)) | This document (frontier model)<br>이 문서(프론티어 모델) |
|---|---|---|
| Runs locally, no outbound calls<br>로컬로 실행되며 발신 통화는 불가능합니다. | ✅ Always<br>✅ 항상 | ✅ Retrieval side only<br>✅ 검색면만 해당 |
| Overall workflow air-gapped<br>전체 워크플로 에어갭 | ✅ If paired with a [local LLM](local-llm.md)<br>✅ [로컬 LLM](local-llm.md)과 페어링된 경우 | ❌ **No** — your question and the retrieved passages are sent to GitHub Copilot to generate the answer<br>❌ **아니요** — 귀하의 질문과 검색된 구절은 답변을 생성하기 위해 GitHub Copilot으로 전송됩니다 |
| Requires GPU / local-LLM hardware<br>GPU/local-LLM 하드웨어 필요 | — | ❌ No — any machine that runs VS Code and Copilot works<br>❌ 아니요 - VS Code 및 Copilot을 실행하는 모든 시스템이 작동합니다. |

If you need a fully air-gapped, zero-outbound deployment, use the
완전히 에어갭이 있고 아웃바운드가 없는 배포가 필요한 경우 다음을 사용하세요.
[Local LLM](local-llm.md) path instead.
대신 [로컬 LLM](local-llm.md) 경로를 사용하세요.

## Why Local RAG
## 왜 로컬 RAG인가?

The key idea is **decoupling**: this solution keeps *where documentation is
핵심 아이디어는 **분리**입니다. 이 솔루션은 *문서가 있는 곳에 보관합니다.
retrieved* (always local — the RAG database) separate from *which model
*어떤 모델과 별도로 검색됨*(항상 로컬 - RAG 데이터베이스)
writes the answer* (your choice — cloud or local).
답변*을 작성합니다(귀하의 선택 – 클라우드 또는 로컬).
Because the two halves
왜냐하면 두 개의 반쪽이
are independent, you can mix and match them, and swap either one later
독립적이므로 혼합하여 사용할 수 있으며 나중에 둘 중 하나를 교체할 수 있습니다.
without redoing the other.
다른 하나를 다시 실행하지 않고.
Pairing the **local** RAG database with a
**로컬** RAG 데이터베이스를
**cloud** frontier model is useful in two common cases:
**클라우드** 프론티어 모델은 다음과 같은 두 가지 일반적인 경우에 유용합니다.

- **Evaluation / validation.** Before investing in GPU hardware and a local
- **평가/검증.** GPU 하드웨어 및 로컬에 투자하기 전에
LLM, confirm that the local documentation snapshot and its retrieval
LLM, 로컬 문서 스냅샷 및 검색 확인
quality are good — using a model you already trust (Copilot).
품질이 좋습니다. 이미 신뢰하는 모델(Copilot)을 사용합니다.
You
당신
validate the retrieval half once, independently of whichever LLM you
어떤 LLM과 관계없이 검색 절반을 한 번만 검증합니다.
eventually pair with it.
결국 그것과 짝을 이룬다.
- **IT policy.** If your network blocks AMD's documentation endpoints but
- **IT 정책.** 네트워크가 AMD의 문서 엔드포인트를 차단하지만
permits your LLM provider, the local RAG database still gives the
LLM 제공업체를 허용하더라도 로컬 RAG 데이터베이스는 여전히
assistant grounded AMD documentation: the docs come from your machine,
보조 기반 AMD 문서: 문서는 컴퓨터에서 제공됩니다.
and only answer generation goes to the (allowed) cloud model.
답변 생성만 (허용된) 클라우드 모델로 이동합니다.

> **Note vs. the hosted tool.** With general internet access, the Vivado
> **호스팅된 도구와 참고하세요.** 일반적인 인터넷 액세스를 통해 Vivado는
> MCP Server's hosted `vivado_doc_search` queries AMD's knowledge base
> MCP 서버에서 호스팅되는 'vivado_doc_search'는 AMD의 지식 기반을 쿼리합니다.
> live, so this local database isn't a fresher data source that replaces
> 따라서 이 로컬 데이터베이스는 대체하는 최신 데이터 소스가 아닙니다.
> it — the value is the decoupling above.
> 그것 — 값은 위의 디커플링입니다.
> And once a machine is fully
> 그리고 기계가 완전히 작동되면
> disconnected, the hosted tool is unreachable, so a
> 연결이 끊어지면 호스팅된 도구에 연결할 수 없으므로
> [local LLM](local-llm.md) becomes the way to keep documentation search
> [local LLM](local-llm.md)은 문서 검색을 유지하는 방법이 됩니다.
> working.
> 일하고 있습니다.

## Setup
## 설정

### Step 1 — Confirm RAG
### 1단계 - RAG 확인

```bash
amd-embedded-doc-search ps
```

All three containers (Weaviate, llama.cpp, MCP server) should show as
세 개의 컨테이너(Weaviate, llama.cpp, MCP 서버)는 모두 다음과 같이 표시되어야 합니다.
running.
달리기.
Note the port from `~/.config/amd-embedded-doc-search/.env`
`~/.config/amd-embedded-doc-search/.env`의 포트를 참고하세요.
(`DOCSEARCH_MCP_PORT`, default `8080`).
(`DOCSEARCH_MCP_PORT`, 기본값 `8080`).

### Step 2 — VS Code
### 2단계 - VS 코드

- Download: <https://code.visualstudio.com/download>
- 다운로드: <https://code.visualstudio.com/download>
- Sign in with your GitHub account that has Copilot access (the built-in
- Copilot 액세스 권한이 있는 GitHub 계정으로 로그인합니다(기본 제공
Copilot Chat experience — no extra extension install needed on current
Copilot Chat 환경 — 현재 추가 확장 설치가 필요하지 않습니다.
VS Code versions).
VS 코드 버전).
- Open the Chat view and confirm a Copilot model (e.g.
- 채팅 보기를 열고 Copilot 모델을 확인합니다(예:
- GPT or Claude,
- GPT 또는 클로드,
depending on what your Copilot plan/model picker offers) is selected.
Copilot 계획/모델 선택기에서 제공하는 항목에 따라)가 선택됩니다.

This is the standard, already-familiar Copilot sign-in flow — nothing
이는 이미 익숙한 표준 Copilot 로그인 흐름입니다.
RAG-specific here.
RAG 전용입니다.
If your organization already has GitHub Copilot rolled
조직에 이미 GitHub Copilot이 포함되어 있는 경우
out, you likely have nothing to do in this step.
이 단계에서는 할 일이 없을 가능성이 높습니다.

### Step 3 — Add MCP
### 3단계 - MCP 추가

Create `.vscode/mcp.json` in your workspace (or your user `mcp.json` to
작업 공간(또는 사용자 `mcp.json`)에 `.vscode/mcp.json`을 생성하여
make it available everywhere):
어디에서나 사용할 수 있도록 하세요):

```json
{
  "servers": {
    "amd-embedded-doc-search": {
      "type": "http",
      "url": "http://127.0.0.1:8080/mcp/doc-search"
    }
  }
}
```

Replace `8080` with your actual `DOCSEARCH_MCP_PORT` if you configured a
구성한 경우 `8080`을 실제 `DOCSEARCH_MCP_PORT`로 바꾸세요.
different port.
다른 포트.
Reload VS Code when prompted.
메시지가 표시되면 VS Code를 다시 로드합니다.

### Step 4 — Verify
### 4단계 - 확인

1. Open Chat, switch to **Agent** mode.
1. 채팅을 열고 **에이전트** 모드로 전환하세요.
2. Confirm your usual Copilot model is selected (GPT/Claude/etc., whatever
2. 평소 Copilot 모델(GPT/Claude/etc. 등)이 선택되었는지 확인하세요.
your plan offers — tool calling is supported natively by all current
귀하의 계획은 현재 모든 제품에서 기본적으로 도구 호출을 지원합니다.
Copilot models).
부조종사 모델).
3. Confirm the `amd-embedded-doc-search` tool is listed in the tools picker (🛠 icon).
3. 'amd-embedded-doc-search' 도구가 도구 선택기(🛠 아이콘)에 나열되어 있는지 확인하세요.
4. Ask: *"Search the documentation for AXI SmartConnect."* (or any Vivado,
4. 질문: *"AXI SmartConnect에 대한 문서를 검색하세요."* (또는 Vivado,
Vitis, PDM, ChipScope, System Software, example-design, Wiki, or Answer
Vitis, PDM, ChipScope, 시스템 소프트웨어, 예제 디자인, Wiki 또는 Answer
Record topic)
녹음 주제)

Expected: Copilot calls the `vivado_doc_search` tool and answers using the RAG
예상: Copilot이 'vivado_doc_search' 도구를 호출하고 RAG를 사용하여 응답합니다.
database, typically citing the source document(s).
일반적으로 원본 문서를 인용합니다.

### Step 5 — Custom Agent
### 5단계 - 맞춤형 에이전트

By default, VS Code's Agent mode exposes its **full built-in tool belt**
기본적으로 VS Code의 에이전트 모드는 **완전히 내장된 도구 벨트**를 노출합니다.
(terminal, file edit, search, etc.) in every request, alongside whatever
(터미널, 파일 편집, 검색 등) 모든 요청에서 무엇이든 함께
MCP tools you've added.
추가한 MCP 도구.
For RAG-only doc-search queries, none of those
RAG 전용 문서 검색 쿼리의 경우 해당 항목 중 어느 것도 없습니다.
extra tools are needed, but their schemas are still sent to the model on
추가 도구가 필요하지만 해당 스키마는 여전히 모델로 전송됩니다.
every turn.
매 턴마다.

VS Code supports **custom agents** (`.agent.md` files) that restrict which
VS Code는 다음을 제한하는 **사용자 지정 에이전트**(`.agent.md` 파일)를 지원합니다.
tools are available for a given persona.
특정 페르소나에 대해 도구를 사용할 수 있습니다.
Create:
만들다:

```text
~/.copilot/agents/amd-embedded-doc-search.agent.md
```

(A per-VS Code-profile user location; `.github/agents/amd-embedded-doc-search.agent.md`
(VS Code 프로필별 사용자 위치, `.github/agents/amd-embedded-doc-search.agent.md`
in a workspace is the team-shared equivalent if you'd rather commit it to
작업 공간에서는 커밋하려는 경우 팀이 공유하는 것과 동일합니다.
a repo.)
저장소.)

```markdown
---
name: "AMD Embedded Doc Search — Adaptive"
description: "Fast AMD documentation RAG lookup. Use when the user asks about Vivado, Vitis, FPGA design, timing closure, IP configuration, hardware debugging, or any AMD Embedded topic."
tools: [amd-embedded-doc-search/*]
---
You are a documentation search assistant for AMD Embedded (Vivado, Vitis, Power Design Manager, ChipScope, System Software, example designs, Wiki pages, and Answer Records).

## Rules

1. IMMEDIATELY call `vivado_doc_search` with the user's question — do not ask clarifying questions first.
2. Start with `limit: 5` for every question, including a simple A-versus-B comparison. Use `limit: 10` only when the user explicitly requests a broad/deep answer, the request has three or more distinct subtopics, or the first search does not contain enough evidence. Prefer a second focused search over retrieving a large, loosely relevant result set.
3. Scale the answer to the task, not to the model:
    - For a simple fact or direct how-to question, be concise (about 150 words is a target, not a hard cap).
    - For a multi-part, diagnostic, design, or explicitly detailed request, provide the complete supported explanation. Use headings, steps, tables, and short code or Tcl examples when they improve clarity.
    - Never omit a necessary qualification, prerequisite, limitation, or troubleshooting step merely to meet a word target.
4. Do NOT fabricate information. Ground factual claims in the search results and cite the relevant source URL with the claim. Do not add a specific command, constraint type, parameter, implementation pattern, or rationale unless a returned source explicitly supports it. If a source only says to update constraints, report that without inferring how; state when the retrieved material does not provide the detail.
5. If the first results do not support a complete answer, are ambiguous, or conflict, run one or more narrower follow-up searches before responding.
6. If the tool returns no relevant results, say so and suggest a rephrased query.
7. Do not assume the active model's identity or capability. This workflow must work with both local and frontier models by favoring relevance and evidence over a fixed output limit.

## Output Format

- Answer sized to the user's question, with key facts and actionable guidance
- Source URLs next to the claims they support when practical
- A mandatory short **Sources** list at the end containing every URL used; do not finish the answer until the list is present
```

The `tools: [amd-embedded-doc-search/*]` line is the important part — it restricts this
`tools: [amd-embedded-doc-search/*]` 줄은 중요한 부분입니다. 이를 제한합니다.
agent to **only** the doc-search MCP server's tools, dropping the
에이전트는 MCP 서버의 문서 검색 도구 **만** 사용하고
terminal, file-edit, and other built-in tool schemas from the prompt
프롬프트에서 터미널, 파일 편집 및 기타 내장 도구 스키마
entirely.
완전히.

Select it from the agents dropdown in the Chat view
채팅 보기의 상담원 드롭다운에서 선택하세요.
(`@amd-embedded-doc-search`) instead of the default Agent mode when you
(`@amd-embedded-doc-search`)를 사용하는 경우 기본 에이전트 모드 대신
only need documentation lookups.
문서 조회만 필요합니다.

#### Measuring Speedup
#### 속도 향상 측정

If you've noticed `@amd-embedded-doc-search` responding faster than asking the same
같은 질문을 하는 것보다 `@amd-embedded-doc-search`가 더 빠르게 응답하는 것을 본 경우
question in default Agent mode, here's the general idea, then the direct
기본 에이전트 모드에서의 질문은 다음과 같습니다. 일반적인 아이디어는 다음과 같습니다.
measurement behind it:
그 뒤에 측정:

**The general idea.** A custom agent with `tools: [amd-embedded-doc-search/*]` calls
**일반적인 개념.** `tools: [amd-embedded-doc-search/*]` 호출을 사용하는 사용자 정의 에이전트
**only** the tool(s) matching that pattern — nothing else is ever placed
**오직** 해당 패턴과 일치하는 도구 — 다른 것은 배치되지 않습니다.
in front of the model.
모델 앞에서.
Default Agent mode, by contrast, hands the model
이와 대조적으로 기본 에이전트 모드는 모델을 전달합니다.
*everything* your IDE/CLI knows about on every single turn: its own full
*모든 것* IDE/CLI가 매 턴마다 알고 있는 것: 자체 전체
built-in tool belt (terminal, file edit, search, etc.) **plus every tool
내장된 도구 벨트(터미널, 파일 편집, 검색 등) **및 모든 도구
from every other MCP server you happen to have configured** — regardless
구성한 다른 모든 MCP 서버에서 ** — 관계없이
of whether your question needs any of them.
귀하의 질문에 그 중 하나가 필요한지 여부.
A workspace may have no other
작업 공간에는 다른 것이 없을 수 있습니다
MCP servers configured, or several (other product servers, language
구성된 MCP 서버 또는 여러 개(다른 제품 서버, 언어
servers, or your own tools).
서버 또는 자체 도구).
A general prompt pays for *all of them* on
일반 프롬프트는 *모두*에 대한 비용을 지불합니다.
every turn, while a scoped custom agent pays for none of them.
매 턴마다 범위가 지정된 맞춤 에이전트는 그 중 어느 것도 비용을 지불하지 않습니다.

**1.
Retrieval itself is not the bottleneck — it's fast and constant.**
검색 자체는 병목 현상이 아니라 빠르고 지속적입니다.**
Timing five real `tools/call` round trips straight against the `amd-embedded-doc-search`
`amd-embedded-doc-search`에 대해 5개의 실제 `도구/호출` 왕복 시간을 측정합니다.
MCP server (bypassing any client/model, `curl` directly to
MCP 서버(클라이언트/모델을 우회하고 `curl`을 직접
`http://127.0.0.1:8080/mcp/doc-search`) gives:
`http://127.0.0.1:8080/mcp/doc-search`)는 다음을 제공합니다.

| Query<br>쿼리 | Round-trip time<br>왕복 시간 |
|---|---|
| "AXI SmartConnect configuration"<br>"AXI SmartConnect 구성" | 31 ms |
| "hold time violation clock domain crossing"<br>"유지 시간 위반 클록 도메인 교차" | 23 ms |
| "incremental compile UG904"<br>"증분 컴파일 UG904" | 20 ms |
| "report_utilization BRAM URAM"<br>"report_utilization BRAM URAM" | 22 ms |
| "GT link training debug ILA"<br>"GT 링크 훈련 디버그 ILA" | 21 ms |

This ~20-30 ms embed-and-search step happens identically no matter which
이 ~20-30ms 삽입 및 검색 단계는 어떤 경우에도 동일하게 발생합니다.
agent or model calls it. **The speed difference you're seeing is not
에이전트나 모델이 전화를 겁니다. **지금 보고 있는 속도 차이는 실제와 다릅니다.
retrieval — it's how much gets sent to the model *before* your question,
검색 — 질문 *전에* 모델에 전송되는 양입니다.
on every single turn.**
매 턴마다.**

**2.
The tool-schema overhead is real, and it can be large.** Every MCP
도구 스키마 오버헤드는 실제이며 클 수 있습니다.** 모든 MCP
server you have configured — not just `amd-embedded-doc-search` — advertises its full
'amd-embedded-doc-search'뿐만 아니라 귀하가 구성한 서버는 전체 내용을 광고합니다.
set of tool schemas (and often a block of usage instructions) to the model
모델에 대한 도구 스키마 세트(종종 사용 지침 블록)
on every single turn, whether or not your question needs any of them.
귀하의 질문에 필요한지 여부에 관계없이 매 턴마다.
Scoping an agent to `tools: [amd-embedded-doc-search/*]` means only `amd-embedded-doc-search`'s own
에이전트 범위를 `tools: [amd-embedded-doc-search/*]`로 지정하면 `amd-embedded-doc-search` 자신의 에이전트만 의미합니다.
(small) schema is ever sent; default Agent mode pays the token cost of
(작은) 스키마가 전송됩니다. 기본 에이전트 모드는 토큰 비용을 지불합니다.
*every* configured tool, every time, regardless of what else happens to be
*모든* 구성된 도구, 다른 일이 발생하더라도 항상
installed in a given workspace.
특정 작업 공간에 설치됩니다.
The more tools/MCP servers you have
보유하고 있는 도구/MCP 서버가 많을수록
configured, the bigger that gap gets.
구성할수록 그 격차는 더 커집니다.

**3.
Why fewer input tokens means a measurably faster response, even for
입력 토큰이 적으면 응답 속도가 훨씬 빨라지는 이유는 무엇입니까?
a cloud frontier model.** Every transformer-based LLM — local or cloud —
클라우드 프론티어 모델.** 모든 변환기 기반 LLM(로컬 또는 클라우드)
processes its entire input (system prompt + tool schemas + conversation +
전체 입력을 처리합니다(시스템 프롬프트 + 도구 스키마 + 대화 +
your question) in a "prefill" pass before generating the first output
귀하의 질문) 첫 번째 출력을 생성하기 전에 "미리 채우기"단계에서
token, and prefill cost scales with input token count.
토큰 및 사전 채우기 비용은 입력 토큰 수에 따라 조정됩니다.
This is
이것은
architecture, not a local-LLM quirk — the same mechanism measured directly
로컬 LLM의 특징이 아닌 아키텍처 — 직접 측정된 동일한 메커니즘
against a local model in
국내 모델 대비
[Local LLM §4.1 (step A.6)](local-llm.md#41-vs-code)
[로컬 LLM §4.1(A.6 단계)](local-llm.md#41-vs-code)
(-73% prompt tokens → -56% total request time) is what's driving the
(프롬프트 토큰 -73% → 총 요청 시간 -56%)이
speedup you're seeing in Copilot.
Copilot에서 볼 수 있는 속도 향상.
A larger tool belt also gives the model
더 큰 공구 벨트도 모델에 제공됩니다.
more to reason about, occasionally triggering an unnecessary tool call
더 많은 이유를 생각해 볼 필요가 있으며 때때로 불필요한 도구 호출이 발생합니다.
before it settles on `vivado_doc_search` — each detour costs a full extra network
`vivado_doc_search`에 정착하기 전에 — 각 우회에는 전체 추가 네트워크 비용이 듭니다.
round trip, not just tokens.
토큰뿐만 아니라 왕복 여행.

> **Note:** GitHub Copilot's cloud endpoint doesn't expose per-request
> **참고:** GitHub Copilot의 클라우드 엔드포인트는 요청별 정보를 노출하지 않습니다.
> prompt-token/timing logs the way a self-hosted `llama-server` does, so
> 프롬프트 토큰/타이밍은 자체 호스팅 `llama-server`가 수행하는 방식으로 기록하므로
> an exact "X seconds vs.
> 정확한 "X초 대
> Y seconds" Copilot-side number isn't available
> Y초" 부조종사 측 번호를 사용할 수 없습니다.
> the same way the local-model number above was obtained.
> 위의 지역 모델 번호를 얻은 것과 같은 방법입니다.
> If you want one,
> 당신이 하나를 원한다면,
> the reliable way is a manual side-by-side: ask the same question via
> 신뢰할 수 있는 방법은 수동으로 나란히 하는 것입니다. 다음을 통해 동일한 질문을 하십시오.
> `@amd-embedded-doc-search` and via default Agent mode, several times
> `@amd-embedded-doc-search` 및 기본 에이전트 모드를 통해 여러 번
> each, and compare elapsed time.
> 각각, 경과 시간을 비교합니다.

## Validation
## 검증

**MCP server:** `amd-embedded-doc-search ps` — all three containers up.
**MCP 서버:** `amd-embedded-doc-search ps` — 세 개의 컨테이너가 모두 작동합니다.

**Client:** ask in VS Code Agent mode (with a Copilot model selected):
**클라이언트:** VS Code Agent 모드에서 질문합니다(Copilot 모델이 선택됨).

```text
Search UG904 for incremental compile.
```
- ✅ The `vivado_doc_search` tool is invoked.
- ✅ `vivado_doc_search` 도구가 호출됩니다.
- ✅ Documentation is returned, with source citations.
- ✅ 출처 인용과 함께 문서가 반환됩니다.

**This path is not air-gapped (expected):**
**이 경로는 에어갭이 아닙니다(예상):**

- The machine needs a network path to GitHub Copilot.
- 머신에는 GitHub Copilot에 대한 네트워크 경로가 필요합니다.
- A successful
- 성공적인
`ping` is normal here.
여기서는 'ping'이 정상입니다.
- Documentation search itself stays on this machine, but the question and
- 문서 검색 자체는 이 컴퓨터에 남아 있지만 질문과
the retrieved passages are sent to GitHub Copilot to generate the answer.
검색된 구절은 GitHub Copilot으로 전송되어 답변을 생성합니다.

## Troubleshooting
## 문제 해결

| Symptom<br>증상 | Likely cause / fix<br>가능한 원인/수정 사항 |
|---------|--------------------|
| `vivado_doc_search` tool not offered<br>'vivado_doc_search' 도구는 제공되지 않습니다. | Chat must be in **Agent** mode; confirm `.vscode/mcp.json` is valid and reload VS Code.<br>채팅은 **에이전트** 모드여야 합니다. `.vscode/mcp.json`이 유효한지 확인하고 VS Code를 다시 로드하세요. |
| MCP server not reachable<br>MCP 서버에 연결할 수 없습니다 | Run `amd-embedded-doc-search ps` and `amd-embedded-doc-search doctor` (see [Operate & Troubleshoot](operate.md)); confirm the URL matches `http://127.0.0.1:<DOCSEARCH_MCP_PORT>/mcp/doc-search`.<br>`amd-embedded-doc-search ps` 및 `amd-embedded-doc-search doctor`를 실행합니다([운영 및 문제 해결](operate.md) 참조). URL이 `http://127.0.0.1:<DOCSEARCH_MCP_PORT>/mcp/doc-search`와 일치하는지 확인하세요. |
| Copilot answers without calling the tool<br>Copilot은 도구를 호출하지 않고 응답합니다. | Some prompts don't clearly signal the model needs current/internal info — rephrase to explicitly ask it to search the documentation, or use the `@amd-embedded-doc-search` custom agent (Step 5) which forces an immediate tool call.<br>일부 프롬프트는 모델에 현재/내부 정보가 필요하다는 것을 명확하게 나타내지 않습니다. 문서를 검색하도록 명시적으로 요청하도록 바꾸거나 즉각적인 도구 호출을 강제하는 `@amd-embedded-doc-search` 사용자 정의 에이전트(5단계)를 사용하십시오. |
| Copilot chat unavailable / no models in picker<br>Copilot 채팅을 사용할 수 없음/선택기에 모델이 없음 | This is a GitHub Copilot sign-in/licensing issue unrelated to `amd-embedded-doc-search` — confirm your GitHub account has an active Copilot plan.<br>이는 'amd-embedded-doc-search'와 관련이 없는 GitHub Copilot 로그인/라이선스 문제입니다. GitHub 계정에 활성 Copilot 계획이 있는지 확인하세요. |

## Next Steps
## 다음 단계

- Need the RAG database steps again? →
- RAG 데이터베이스 단계가 다시 필요합니까? →
[RAG Database Setup](README.md)
[RAG 데이터베이스 설정](README.md)
- Need a fully air-gapped deployment instead (no outbound calls at all)? →
- 대신 완전히 에어갭 배포가 필요합니까(아웃바운드 통화가 전혀 없음)? →
[Local LLM](local-llm.md)
[로컬 LLM](local-llm.md)

---

<p class="sphinxhide" align="center"><sub>Copyright © 2026 Advanced Micro Devices, Inc</sub></p>
<p class="sphinxhide" align="center"><sup><a href="https://www.amd.com/en/corporate/copyright">Terms and Conditions</a></sup></p>
