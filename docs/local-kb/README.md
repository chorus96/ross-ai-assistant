<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# AMD Embedded — Locally Deployable RAG Database
# AMD 임베디드 — 로컬로 배포 가능한 RAG 데이터베이스

**A fully air-gapped, self-hosted Retrieval-Augmented Generation (RAG)
**완전히 에어갭이 있는 자체 호스팅 RAG(Retrieval-Augmented Generation)
database for AMD Embedded documentation — the retrieval layer only, with no
AMD Embedded 문서용 데이터베이스 — 검색 계층만 해당, 없음
outbound network calls at runtime.**
런타임 시 아웃바운드 네트워크 호출.**

This document covers everything that is **common regardless of which model
이 문서는 **모델에 관계없이 일반적인 모든 내용을 다룹니다.
answers your questions afterward** — a local LLM on GPU-class hardware, or
나중에 질문에 답변** - GPU급 하드웨어의 로컬 LLM 또는
a frontier cloud model like GitHub Copilot.
GitHub Copilot과 같은 프론티어 클라우드 모델입니다.
That choice is covered in the
그 선택은 다음에서 다룹니다.
two companion documents linked below; this one is only about the RAG
아래에 링크된 두 개의 동반 문서; 이건 단지 RAG에 관한 것뿐이야
database itself: why it exists, how it works, and how to deploy it.
데이터베이스 자체: 존재 이유, 작동 방식 및 배포 방법

The documentation corpus covers the full range of content published by the
문서 자료는 다음에서 게시한 모든 범위의 콘텐츠를 다루고 있습니다.
AMD Embedded team — not just Vivado.
AMD 임베디드 팀 — Vivado만이 아닙니다.
This includes **Vivado**, **Vitis**,
여기에는 **Vivado**, **Vitis**,
**Power Design Manager (PDM)**, **ChipScope**, **System Software**,
**전력 설계 관리자(PDM)**, **ChipScope**, **시스템 소프트웨어**,
**example designs**, **Wiki** pages, and **Answer Records**.
**디자인 예**, **Wiki** 페이지 및 **답변 기록**.

> **Where this page sits in the guide**
> **가이드에서 이 페이지가 있는 위치**
>
> This page is the RAG database: what it is, how it works, and how to
> 이 페이지는 RAG 데이터베이스입니다. 정의, 작동 방식 및 방법
> deploy it on an air-gapped machine.
> 에어 갭 머신에 배포하십시오.
> The rest of the guide:
> 가이드의 나머지 부분:
>
> - [Quick Start](quickstart.md) — stand the stack up on a connected machine
> - [빠른 시작](quickstart.md) — 연결된 시스템에 스택을 세웁니다.
> - [Operate & Troubleshoot](operate.md) — CLI reference, validation, and fixes
> - [운영 및 문제 해결](operate.md) — CLI 참조, 검증 및 수정 사항
> - [Frontier Model](frontier-model.md) — answer with a cloud model (no local GPU)
> - [프론티어 모델](frontier-model.md) — 클라우드 모델로 답변(로컬 GPU 없음)
> - [Local LLM](local-llm.md) — answer with a fully air-gapped local LLM (AMD ROCm)
> - [로컬 LLM](local-llm.md) — 완전히 에어갭된 로컬 LLM(AMD ROCm)으로 답변

---

## 1.
## Overview
## 개요

### 1.1 The Problem
### 1.1 문제

Engineers ask their AI assistant (VS Code, OpenCode, etc.) questions like
엔지니어는 AI 도우미(VS Code, OpenCode 등)에게 다음과 같은 질문을 합니다.
*"Why does Vivado report a hold-time violation on a clock-domain-crossing
*"Vivado가 시계 도메인 교차 시 홀드 타임 위반을 보고하는 이유는 무엇입니까?
path, and how do I fix it?"* A general-purpose AI model was trained on a
경로가 있는데 어떻게 수정하나요?"* 범용 AI 모델은
snapshot of data up to some cutoff date — it may not reflect documentation
특정 마감일까지의 데이터 스냅샷 - 문서가 반영되지 않을 수 있음
published after that cutoff, it often lacks the specific, authoritative
컷오프 이후에 출판된 경우에는 구체적이고 권위 있는 내용이 부족한 경우가 많습니다.
detail found across AMD's documentation set, and worse, when it doesn't
AMD의 문서 세트 전체에서 발견된 세부 정보, 더 나쁜 경우에는 그렇지 않은 경우
actually know something it can still generate a confident,
실제로 자신감을 불러일으킬 수 있는 것을 알고 있습니다.
plausible-sounding, **wrong** answer (commonly called a "hallucination").
그럴듯하게 들리는 **잘못된** 대답(일반적으로 "환각"이라고 함).

This solution fixes that by giving the AI model a **searchable,
이 솔루션은 AI 모델에 **검색 가능,
trustworthy, and current library of real AMD Embedded documentation** to
실제 AMD 임베디드 문서**가 포함된 신뢰할 수 있는 최신 라이브러리
consult before it answers.
답변하기 전에 상담하세요.
The database that makes this possible runs
이를 가능하게 하는 데이터베이스가 실행됩니다.
**entirely on your own machine**, with nothing ever sent to the internet —
**완전히 자신의 컴퓨터에**, 인터넷에 아무것도 전송되지 않음 —
**regardless of which model you use to generate the final answer.**
**최종 답변을 생성하는 데 사용하는 모델에 관계없이.**

> **Important distinction up front:** *this* database is air-gapped no
> **앞서 중요한 차이점:** *이* 데이터베이스는 에어 갭(air-gap)입니다. 아니요
> matter what.
> 무슨 상관이야.
> Whether your *overall workflow* is air-gapped depends on
> *전체 워크플로우*가 에어 갭인지 여부는 다음에 따라 다릅니다.
> which model you pick to generate answers — see
> 답변을 생성하기 위해 어떤 모델을 선택하는지 — 참조
> [§1.4](#14-answering-model).
> [§1.4](#14-응답 모델).

### 1.2 Key Concepts
### 1.2 주요 개념

<details>
<summary><strong>New to Agentic AI, RAG, or embeddings? Expand for a plain-language primer</strong></summary>

These are the building blocks used throughout this document, explained
다음은 이 문서 전반에 걸쳐 사용되는 빌딩 블록입니다.
without assuming prior knowledge.
사전 지식을 가정하지 않고.

**Large Language Model (LLM).** An LLM (Qwen, Gemma, GPT, Claude, etc.) is
**대형 언어 모델(LLM).** LLM(Qwen, Gemma, GPT, Claude 등)은 다음과 같습니다.
software trained on enormous amounts of text that can read a question and
질문을 읽고 읽을 수 있는 엄청난 양의 텍스트에 대해 훈련된 소프트웨어
generate a human-like answer.
인간과 같은 답변을 생성합니다.
Think of it as an extremely well-read intern:
매우 잘 읽은 인턴이라고 생각하십시오.
fluent, fast, and good at explaining things — but its knowledge is frozen at
유창하고 빠르며 설명을 잘하지만 지식은 얼어붙어 있습니다.
whatever point it was trained, and by itself it cannot look anything up.
어떤 지점에서 훈련을 받았든 그 자체로는 아무것도 찾아볼 수 없습니다.
This document treats the LLM as a **pluggable component** — it doesn't
이 문서에서는 LLM을 **플러그형 구성 요소**로 취급합니다.
matter whether it runs on your laptop's GPU or in a cloud data center; the
노트북의 GPU에서 실행되는지, 클라우드 데이터 센터에서 실행되는지는 중요합니다. 는
RAG database talks to it the same way either way.
RAG 데이터베이스는 어느 쪽이든 동일한 방식으로 통신합니다.

**Agentic AI / Agent mode.** Ordinarily, chatting with an LLM is one-way:
**에이전트 AI / 에이전트 모드.** 일반적으로 LLM과의 채팅은 단방향입니다.
you ask, it answers purely from what it memorized during training.
당신이 물으면 그것은 훈련 중에 기억한 것만으로 대답합니다.
*Agentic AI* means the model can take actions instead of just talking — it
*에이전트 AI*는 모델이 단순히 말하는 대신 행동을 취할 수 있음을 의미합니다.
can decide "I don't actually know this, let me use a tool to find out,"
"실제로는 이것을 모릅니다. 도구를 사용하여 알아 보겠습니다."라고 결정할 수 있습니다.
invoke that tool, read the result, and use it to write a better answer.
해당 도구를 호출하고 결과를 읽고 이를 사용하여 더 나은 답변을 작성하세요.
This capability is usually exposed to you as **"Agent mode"** in tools like
이 기능은 일반적으로 다음과 같은 도구에서 **"에이전트 모드"**로 표시됩니다.
VS Code or OpenCode.
VS 코드 또는 OpenCode.

**Tool calling.** The mechanism that makes Agentic AI possible.
**도구 호출.** Agentic AI를 가능하게 하는 메커니즘입니다.
A *tool* is
*도구*는
simply a function the model is told about — e.g. "here is a tool called
단순히 모델이 설명하는 기능입니다. "여기에 라는 도구가 있습니다.
`doc-search` that takes a question and returns documentation passages." A
질문을 받아 문서 구절을 반환하는 '문서 검색'입니다." 에이
tool-calling-capable model recognizes when a tool would help, calls it
도구 호출 가능 모델은 도구가 도움이 될 때를 인식하고 이를 호출합니다.
automatically mid-conversation, and folds the result into its answer.
대화 중에 자동으로 결과를 답변으로 접습니다.
Every
매
mainstream frontier model (GitHub Copilot's models, Claude, GPT) supports
주류 프론티어 모델(GitHub Copilot의 모델, Claude, GPT)은 다음을 지원합니다.
this natively; local models vary in how reliably they do it, which is why
이것은 기본적으로; 로컬 모델은 얼마나 안정적으로 수행하는지에 따라 다릅니다.
model choice matters for local deployments (see the
로컬 배포에서는 모델 선택이 중요합니다(참조:
[Local LLM document](local-llm.md)).
[로컬 LLM 문서](local-llm.md)).

**Retrieval-Augmented Generation (RAG).** This is the overall pattern the
**RAG(Retrieval-Augmented Generation).** 이는 전체 패턴입니다.
whole solution implements, and where the "RAG" in this document's title
전체 솔루션 구현 및 이 문서 제목의 "RAG"
comes from.
에서 온다.
Instead of relying only on what the LLM memorized during
LLM이 학습한 내용에만 의존하기보다는
training, RAG **retrieves** relevant, real passages from a trusted source
훈련, RAG는 신뢰할 수 있는 소스로부터 관련성 있는 실제 구절을 **검색**합니다.
right before the model answers, and inserts them into the conversation as
모델이 응답하기 직전에 다음과 같이 대화에 삽입합니다.
extra context — the model's generation is *augmented* with retrieved
추가 컨텍스트 — 모델 생성이 검색된 내용으로 *증강*됩니다.
facts.
사실.
The result: answers are grounded in real, current, sourced
결과: 답변은 실제, 현재, 출처에 근거합니다.
documentation rather than the model's frozen memory, and you can trace an
모델의 고정된 메모리가 아닌 문서화를 통해 추적할 수 있습니다.
answer back to the exact document it came from.
그것이 나온 정확한 문서에 답하십시오.

**Embeddings & vector databases.** Computers can't directly compare the
**임베딩 및 벡터 데이터베이스.** 컴퓨터는 데이터를 직접 비교할 수 없습니다.
*meaning* of two pieces of text — meaning has to be turned into numbers
*의미* 두 텍스트 조각 — 의미는 숫자로 변환되어야 함
first.
먼저.
An **embedding model** (this solution ships with a choice of two —
**임베딩 모델**(이 솔루션은 두 가지 중 하나를 선택할 수 있습니다.
see [§1.5](#15-embedding-packages)) converts a chunk of
[§1.5](#15-embedding-packages) 참조)는
text into a long list of numbers (a "vector") that captures its semantic
의미를 포착하는 긴 숫자 목록("벡터")으로 텍스트를 변환합니다.
meaning: two passages about similar topics end up with similar vectors,
의미: 유사한 주제에 관한 두 구절은 유사한 벡터로 끝납니다.
even if they don't share any of the same words.
같은 단어를 공유하지 않더라도.
A **vector database** (this
**벡터 데이터베이스**(이것은
solution uses Weaviate) stores hundreds of thousands of these vectors and can instantly
솔루션은 Weaviate를 사용하여 수십만 개의 벡터를 저장하고 즉시
find the ones closest in meaning to a new query's vector.
새 쿼리의 벡터에 가장 가까운 의미를 찾습니다.
This is what lets
이것이 바로
the assistant find the *right* documentation passage even when your
보조자는 귀하가
question is phrased completely differently from the manual.
질문은 매뉴얼과 완전히 다르게 표현되어 있습니다.

**How is an embedding model different from the LLM that answers you?**
**임베딩 모델은 답변을 제공하는 LLM과 어떻게 다릅니까?**
This is the most common point of confusion, so it's worth being explicit:
이는 가장 흔히 혼동되는 지점이므로 명확하게 설명할 가치가 있습니다.
this solution always uses **two different models, for two different
이 솔루션은 항상 **두 가지 서로 다른 모델을 사용합니다.
jobs** — and only one of them ("the LLM") is the one you get to choose
직업** - 그 중 하나만("LLM") 선택할 수 있습니다.
between local and frontier.
지역과 국경 사이.

| | Embedding model (e.g. EmbeddingGemma, Qwen3-Embedding)<br>임베딩 모델(예: EmbeddingGemma, Qwen3-Embedding) | The LLM that answers you (local *or* frontier)<br>귀하에게 응답하는 LLM(지역 *또는* 국경) |
|---|---|---|
| **Job**<br>**직업** | Turns text into a vector of numbers<br>텍스트를 숫자 벡터로 변환합니다. | Reads text and generates new text<br>텍스트를 읽고 새 텍스트를 생성합니다. |
| **Output**<br>**출력** | A list of ~hundreds of numbers — not human-readable<br>사람이 읽을 수 없는 ~수백 개의 숫자 목록 | A conversational, human-readable answer<br>사람이 읽을 수 있는 대화식 답변 |
| **Size**<br>**크기** | Small (300M-600M parameters, a few hundred MB to ~1.2 GB)<br>작음(300M~600M 매개변수, 수백 MB~~1.2GB) | Varies — tens of GB if local, unknown/irrelevant if a cloud frontier model<br>다양함 - 로컬인 경우 수십 GB, 클라우드 프론티어 모델인 경우 알 수 없음/관련 없음 |
| **Can it chat or reason?**<br>**채팅이나 추론이 가능합니까?** | No — it has no notion of a "conversation," it only measures similarity of meaning<br>아니요. "대화"라는 개념이 없으며 의미의 유사성만 측정합니다. | Yes — this is what actually writes the answer you read<br>예 — 이것이 실제로 읽은 답변을 쓰는 것입니다 |
| **Can it search millions of documents fast?**<br>**수백만 개의 문서를 빠르게 검색할 수 있나요?** | Yes — that's its entire purpose<br>예 - 그게 전부 목적이에요 | Not practically — far too slow/expensive to compare raw text this way<br>실제로는 그렇지 않습니다. 이런 식으로 원시 텍스트를 비교하기에는 너무 느리거나 비용이 많이 듭니다. |
| **Where it runs**<br>**실행 위치** | Always local — inside the Docker stack (§2.2)<br>항상 로컬 - Docker 스택 내부(§2.2) | Your choice — see [§1.4](#14-answering-model)<br>귀하의 선택 - [§1.4](#14-응답 모델) 참조 |

In short: the **embedding model is the librarian's index card system** — it
간단히 말해서 **임베딩 모델은 사서의 색인 카드 시스템**입니다.
doesn't understand your question in the way a person does, but it's
당신의 질문을 다른 사람이 이해하는 방식으로 이해하지는 못하지만,
extremely fast at finding "which documents are about the same topic as
"어떤 문서가 다음과 같은 주제에 관한 것인지 매우 빠르게 찾아냅니다.
this." **Whichever LLM you choose is the person who actually reads what
이거." **어떤 LLM을 선택하든 실제로 내용을 읽는 사람은
the librarian hands over and explains it to you in plain English.** Neither
사서가 건네주고 쉬운 영어로 설명해줍니다.**
one can do the other's job — which is exactly why RAG wires the two
하나는 다른 하나의 작업을 수행할 수 있습니다. 이것이 바로 RAG가 두 개를 연결하는 이유입니다.
together instead of relying on just one model.
하나의 모델에만 의존하는 것이 아니라 함께 협력하는 것입니다.
This document is entirely
이 문서는 전적으로
about the librarian (the embedding model + vector database + MCP server);
사서 정보(임베딩 모델 + 벡터 데이터베이스 + MCP 서버)
it never assumes anything about which "person" (LLM) you've chosen.
이는 귀하가 선택한 "사람"(LLM)에 대해 어떠한 가정도 하지 않습니다.

**Model Context Protocol (MCP).** A standard, model-agnostic way for AI
**MCP(Model Context Protocol).** 모델에 구애받지 않는 표준 AI 방식
clients (VS Code, OpenCode, Claude, etc.) to discover and call *external
*외부 검색 및 호출을 위한 클라이언트(VS Code, OpenCode, Claude 등)
tools*, regardless of what language or platform those tools are built in.
도구*(해당 도구가 내장된 언어나 플랫폼에 관계 없음)
Rather than every AI application needing custom, one-off integration code
맞춤형 일회성 통합 코드가 필요한 모든 AI 애플리케이션이 아닌
per tool, MCP defines a common protocol: a tool "server" advertises what it
도구별로 MCP는 공통 프로토콜을 정의합니다. 도구 "서버"는 도구가 무엇인지 광고합니다.
can do, and any MCP-compatible client can call it the same way.
할 수 있으며 모든 MCP 호환 클라이언트는 동일한 방식으로 호출할 수 있습니다.
In this
이에
solution, the `amd-embedded-doc-search` MCP server exposes exactly one tool —
솔루션인 'amd-embedded-doc-search' MCP 서버는 정확히 하나의 도구를 노출합니다.
searching the AMD Embedded documentation corpus. **Any** MCP-compatible
AMD Embedded 문서 자료를 검색합니다. **모든** MCP 호환
client/model combination can use it, local or cloud.
클라이언트/모델 조합은 로컬 또는 클라우드에서 사용할 수 있습니다.

</details>

### 1.3 Worked Example
### 1.3 실제 사례

Suppose you ask: *"Why does Vivado report a hold-time violation on a
당신이 다음과 같이 질문한다고 가정해보자: *"Vivado가 왜 보류 시간 위반을 보고합니까?
clock-domain-crossing path, and how do I fix it?"*
시계 영역 교차 경로, 어떻게 수정하나요?"*

![Flow diagram: You ask a question, which passes through the AI Client, the LLM, the doc-search MCP server, the embedding model, and the Weaviate vector database, before a grounded, cited answer flows back to you](diagrams/how-it-fits-together.svg)

Step by step, that diagram is:
단계별로 해당 다이어그램은 다음과 같습니다.

1. Your AI client (VS Code Agent mode or OpenCode) sends the question to
1. AI 클라이언트(VS Code Agent 모드 또는 OpenCode)가 질문을 보냅니다.
the LLM you've configured — local or frontier, it makes no difference
귀하가 구성한 LLM — 지역이든 변경 지역이든 상관없습니다.
to the steps that follow.
다음 단계로 이동하세요.
The LLM recognizes it doesn't have enough
LLM은 충분하지 않다는 것을 인식합니다.
specific information, and calls the `vivado_doc_search` tool over MCP —
특정 정보를 제공하고 MCP를 통해 `vivado_doc_search` 도구를 호출합니다.
instead of guessing.
추측하는 대신.
2. The MCP server forwards your question text to Weaviate.
2. MCP 서버는 귀하의 질문 텍스트를 Weaviate로 전달합니다.
2. Weaviate's
2. 위비에이트의
configured vectorizer sends the query to the local llama.cpp embedding
구성된 벡터화 도구는 쿼리를 로컬 llama.cpp 임베딩으로 보냅니다.
service, which converts it into a vector.
이를 벡터로 변환하는 서비스입니다.
Weaviate then performs a
Weaviate는 다음을 수행합니다.
hybrid search — keyword and vector similarity — against the
하이브리드 검색(키워드 및 벡터 유사성)
pre-computed vectors already stored in its database.
사전 계산된 벡터는 이미 데이터베이스에 저장되어 있습니다.
The corpus was
말뭉치는
built from AMD Embedded's Vivado, Vitis, PDM, ChipScope, System
AMD Embedded의 Vivado, Vitis, PDM, ChipScope, 시스템으로 구축
Software, example designs, Wiki, and Answer Records content.
소프트웨어, 예제 디자인, Wiki 및 Answer Records 콘텐츠.
3. The matching documentation passages are returned through the MCP server
3. 일치하는 문서 구절이 MCP 서버를 통해 반환됩니다.
back to the LLM.
LLM으로 돌아갑니다.
4. The LLM reads those passages and writes a final answer, grounded in —
4. LLM은 해당 구절을 읽고 다음에 근거하여 최종 답변을 작성합니다.
and typically citing — the actual documentation, instead of relying on
일반적으로 인용 — 의존하는 대신 실제 문서
whatever it vaguely remembers from training.
훈련에서 막연하게 기억하는 것은 무엇이든.

The retrieval step itself (step 2: vectorize → hybrid search → return
검색 단계 자체(2단계: 벡터화 → 하이브리드 검색 → 반환)
passages) is consistently fast — typically well under a second, since the embedding
통로)는 지속적으로 빠릅니다. 일반적으로 임베딩 이후 1초 미만입니다.
model is small (~300 MB) and the vector search is a local, in-memory
모델은 작고(~300MB) 벡터 검색은 로컬, 메모리 내입니다.
lookup.
조회.
Total round-trip time then depends on which LLM generates the
총 왕복 시간은 어떤 LLM이 생성하는지에 따라 달라집니다.
answer — see the [Frontier Model](frontier-model.md) or
대답 - [프론티어 모델](frontier-model.md)을 참조하거나
[Local LLM](local-llm.md) guide for that half of the picture.
[로컬 LLM](local-llm.md) 그림의 절반에 대한 안내입니다.

### 1.4 Answering Model
### 1.4 응답 모델

Everything above — vector database, embedding model, and MCP server — always
위의 모든 것(벡터 데이터베이스, 임베딩 모델, MCP 서버)은 항상
runs locally.
로컬로 실행됩니다.
Your only choice is **where the model that writes the final
유일한 선택은 **최종 결과를 작성하는 모델이 어디인지입니다.
answer runs**:
응답 실행**:

![Diagram: the AI Client splits into two layers — the Answering Model (frontier cloud or local) via a chat/tool-calling API, and the RAG Database via MCP over HTTP — only the RAG Database side is guaranteed to need no Internet connection](diagrams/two-layers.svg)

The RAG database (deployed below) is the same for both options — only the
RAG 데이터베이스(아래 배포)는 두 옵션 모두 동일합니다.
answering model differs:
응답 모델이 다릅니다.

| | **[Frontier model](frontier-model.md)** (Option A)<br>**[프론티어 모델](frontier-model.md)** (옵션 A) | **[Local LLM](local-llm.md)** (Option B)<br>**[로컬 LLM](local-llm.md)** (옵션 B) |
|---|---|---|
| Answering LLM runs<br>LLM 실행 응답 | Cloud (e.g. GitHub Copilot in VS Code)<br>클라우드(예: VS Code의 GitHub Copilot) | On your machine (`llama-server`)<br>귀하의 컴퓨터에서(`llama-server`) |
| Hardware<br>하드웨어 | Any laptop — **no local GPU**<br>모든 노트북 — **로컬 GPU 없음** | GPU-class (ROCm or Vulkan capable; this guide is written and tested for **AMD ROCm**)<br>GPU 클래스(ROCm 또는 Vulkan 지원, 이 가이드는 **AMD ROCm**에 대해 작성 및 테스트되었습니다.) |
| End-to-end air-gapped?<br>종단 간 에어 갭? | **No** — the question + retrieved passages go to the cloud model<br>**아니요** — 질문 + 검색된 구절이 클라우드 모델로 이동합니다. | **Yes** — zero outbound network<br>**예** — 아웃바운드 네트워크 없음 |
| Choose when<br>언제 선택하세요 | Fastest start, or no local GPU<br>가장 빠른 시작 또는 로컬 GPU 없음 | Full air-gap is required<br>완전한 에어갭이 필요합니다. |

### 1.5 Embedding Packages
### 1.5 패키지 삽입

This solution is delivered as a self-contained package **per embedding
이 솔루션은 **임베딩당 독립형 패키지로 제공됩니다.
model**.
모델**.
You receive **one** archive for the embedding model you choose.
선택한 임베딩 모델에 대해 **하나** 아카이브를 받습니다.
The archive name changes from build to build — use the file you were given,
아카이브 이름은 빌드마다 변경됩니다. 제공된 파일을 사용하십시오.
not a fixed filename.
고정된 파일 이름이 아닙니다.
Both variants are fully self-hosted and air-gapped,
두 변종 모두 완전히 자체 호스팅되고 에어갭이 있습니다.
and both install and operate **identically**; every command in this guide
**동일하게** 설치하고 작동합니다. 이 가이드의 모든 명령
(`amd-embedded-doc-search configure`, `amd-embedded-doc-search start`, etc.)
(`amd-embedded-doc-search 구성`, `amd-embedded-doc-search start` 등)
works the same regardless of which you were given.
당신이 무엇을 받았는지에 관계없이 동일하게 작동합니다.

The embedding model **can affect retrieval quality**, but a head-to-head
임베딩 모델은 **검색 품질에 영향을 미칠 수 있지만**
comparison for this corpus is **not yet published** (planned for a future
이 자료에 대한 비교는 **아직 게시되지 않았습니다**(향후 계획됨)
release).
릴리스).
Until then, if you get to choose, decide on **license** or
그때까지는 선택이 가능하다면 **라이센스**를 결정하거나
**download size** — otherwise use the one supplied to you.
**다운로드 크기** — 그렇지 않으면 제공된 크기를 사용하세요.

| | `embeddinggemma-300m`<br>`임베딩젬마-300m` | `qwen3-embedding-0.6b`<br>`qwen3-임베딩-0.6b` |
|---|---|---|
| Embedding model<br>임베딩 모델 | Google EmbeddingGemma-300M (Q8_0)<br>Google EmbeddingGemma-300M(Q8_0) | Qwen3-Embedding-0.6B (f16)<br>Qwen3-임베딩-0.6B(f16) |
| License<br>라이센스 | Gemma Terms of Use<br>젬마 이용약관 | Apache 2.0<br>아파치 2.0 |
| Download size<br>다운로드 크기 | ~3.1 GB | ~4.4 GB |

For **setup**, the two are interchangeable — the same commands and file
**설정**의 경우 두 가지가 서로 호환됩니다. 동일한 명령과 파일
layout apply; only the archive filename and the model file under `models/`
레이아웃 적용; `models/` 아래의 아카이브 파일 이름과 모델 파일만
differ (see [§2.4](#24-file-layout) and
다르며([§2.4](#24-file-layout 참조) 및
[§3.1 Step 1](#31-phase-1)).
[§3.1 단계 1](#31-단계-1)).

---

## 2.
## Architecture
## 건축

### 2.1 Docker Basics
### 2.1 도커 기본

Before looking at the stack itself, it helps to clear up a common point of
스택 자체를 보기 전에, 다음과 같은 공통점을 정리하는 데 도움이 됩니다.
confusion: **"Docker" is not a container — it's the platform that runs
혼란: **"Docker"는 컨테이너가 아닙니다. 실행되는 플랫폼입니다.
containers.** A single Docker installation on one machine can run many
컨테이너.** 한 시스템에 단일 Docker를 설치하면 여러 개를 실행할 수 있습니다.
containers at the same time, each fully isolated from the others.
동시에 컨테이너는 서로 완전히 격리됩니다.

- A **container** is a lightweight, self-contained package: an application
- **컨테이너**는 경량의 독립형 패키지입니다.
plus everything it needs to run (libraries, configuration, dependencies) —
그리고 실행에 필요한 모든 것(라이브러리, 구성, 종속성) —
but *not* a full operating system.
그러나 완전한 운영 체제는 *아닙니다*.
Think of it like a shipping container
컨테이너라고 생각하시면 됩니다
(where Docker gets its name and logo): a standardized box that can be
(Docker의 이름과 로고가 표시됨): 표준화된 상자입니다.
built once, moved anywhere, and run the same way regardless of what's
한 번 구축하고, 어디로든 이동하고, 무엇이든지 동일한 방식으로 실행됩니다.
inside or what machine it's placed on.
내부 또는 어떤 기계에 배치되어 있는지.
- **Docker** (or more precisely the *Docker Engine*) is the software
- **Docker**(또는 더 정확하게는 *Docker Engine*)는 소프트웨어입니다.
installed on your machine that knows how to build, start, stop, and
빌드하고, 시작하고, 중지하고, 수행하는 방법을 아는 컴퓨터에 설치됩니다.
isolate these containers.
이 용기를 격리하십시오.
- **Docker Compose** is a tool for describing a *group* of related
- **Docker Compose**는 관련 *그룹*을 설명하는 도구입니다.
containers — which images to run, how they're networked together, which
컨테이너 - 실행할 이미지, 네트워크 연결 방식,
ports they expose — in a single file (`docker-compose.yml`), and starting
노출되는 포트 — 단일 파일(`docker-compose.yml`)에서 시작
or stopping all of them together with one command.
또는 하나의 명령으로 모두 함께 중지할 수 있습니다.
A group of containers
컨테이너 그룹
managed this way is often called a **stack** or **project**.
이런 방식으로 관리되는 것을 **스택** 또는 **프로젝트**라고 합니다.

A single Docker installation can absolutely run multiple containers side by
단일 Docker 설치로 여러 컨테이너를 동시에 실행할 수 있습니다.
side — and that's exactly what happens here.
측면 — 그리고 그것이 바로 여기서 일어나는 일입니다.
This solution's
이 솔루션의
`amd-embedded-doc-search` stack is one Docker Compose project made up of **three separate
`amd-embedded-doc-search` 스택은 **세 개의 개별 프로젝트로 구성된 하나의 Docker Compose 프로젝트입니다.
containers** (Weaviate, llama.cpp, and the MCP server), each running in its
컨테이너**(Weaviate, llama.cpp 및 MCP 서버)는 각각 자체에서 실행됩니다.
own isolated process, started and stopped together, but independently
자체 격리된 프로세스가 함께 시작되고 중지되지만 독립적으로 수행됩니다.
replaceable and independently restartable if one of them needs attention.
주의가 필요한 경우 교체 가능하고 독립적으로 다시 시작할 수 있습니다.

### 2.2 Docker Stack
### 2.2 도커 스택

![Diagram: the MCP server sends searches to Weaviate; Weaviate requests query embeddings from llama.cpp and performs hybrid search.
![다이어그램: MCP 서버는 Weaviate로 검색을 보냅니다. Weaviate는 llama.cpp에서 쿼리 임베딩을 요청하고 하이브리드 검색을 수행합니다.
A one-time importer loads the snapshot into Weaviate on first start.](diagrams/docker-stack.svg)
일회성 임포터는 처음 시작할 때 Weaviate에 스냅샷을 로드합니다.](diagrams/docker-stack.svg)

| Container<br>컨테이너 | Image<br>이미지 | Role<br>역할 |
|-----------|-------|------|
| **Weaviate**<br>**Weaviate** | `cr.weaviate.io/semitechnologies/weaviate:1.38.2`<br>`cr.weaviate.io/semitechnologies/weaviate:1.38.2` | Vector database and query orchestrator. Its `text2vec-openai` vectorizer calls llama.cpp for query embeddings, then Weaviate performs hybrid keyword/vector search over the imported corpus.<br>벡터 데이터베이스 및 쿼리 조정자. 'text2vec-openai' 벡터화 도구는 쿼리 임베딩을 위해 llama.cpp를 호출한 다음 Weaviate는 가져온 코퍼스에 대해 하이브리드 키워드/벡터 검색을 수행합니다. |
| **llama.cpp**<br>**라마.cpp** | `ghcr.io/ggml-org/llama.cpp:server`<br>`ghcr.io/ggml-org/llama.cpp:서버` | Long-running embedding service. Loads EmbeddingGemma-300M or Qwen3-Embedding-0.6B at startup and converts query text to vectors through `/v1/embeddings`. Small footprint (a few hundred MB to ~1.2 GB model, CPU-only by default).<br>장기 실행 임베딩 서비스. 시작 시 EmbeddingGemma-300M 또는 Qwen3-Embedding-0.6B를 로드하고 `/v1/embeddings`를 통해 쿼리 텍스트를 벡터로 변환합니다. 작은 설치 공간(수백 MB ~ 1.2GB 모델, 기본적으로 CPU 전용). |
| **MCP Server**<br>**MCP 서버** | `vivado.amd.com/doc-search-mcp-server:0.9.0`<br>`vivado.amd.com/doc-search-mcp-server:0.9.0` | Exposes the `vivado_doc_search` tool over MCP HTTP transport (named `vivado_doc_search` for client compatibility, but it searches the entire AMD Embedded corpus, not Vivado alone). It sends search requests to Weaviate and returns its results.<br>MCP HTTP 전송을 통해 `vivado_doc_search` 도구를 노출합니다(클라이언트 호환성을 위해 `vivado_doc_search`로 명명되었지만 Vivado 단독이 아닌 전체 AMD 임베디드 코퍼스를 검색합니다). Weaviate에 검색 요청을 보내고 결과를 반환합니다. |
| **Snapshot importer** *(first start only)*<br>**스냅샷 가져오기** *(처음 시작만 해당)* | snapshot image bundled in the package<br>패키지에 번들로 포함된 스냅샷 이미지 | One-shot job container. Waits for Weaviate, creates/configures the collection, and loads the pre-vectorized documentation snapshot. It exits after successful import. The image tag follows that package build.<br>원샷 작업 컨테이너. Weaviate를 기다리고, 컬렉션을 생성/구성하고, 사전 벡터화된 문서 스냅샷을 로드합니다. 가져오기 성공 후 종료됩니다. 이미지 태그는 해당 패키지 빌드를 따릅니다. |

None of these containers is an answering LLM — this stack is **retrieval
이러한 컨테이너 중 어느 것도 응답 LLM이 아닙니다. 이 스택은 **검색입니다.
only**.
만**.
The snapshot importer is a one-shot job, not a fourth long-running
스냅샷 가져오기는 4번째 장기 실행이 아닌 일회성 작업입니다.
service.
서비스.
The model that reads the retrieved passages and writes an answer
검색된 지문을 읽고 답을 쓰는 모델
lives outside this stack entirely (see [§1.4](#14-answering-model)).
이 스택 외부에 완전히 존재합니다([§1.4](#14-응답 모델) 참조).

### 2.3 Components
### 2.3 구성요소

- **Embedding model**: one of two self-hosted options depending on the
- **임베딩 모델**: 두 가지 자체 호스팅 옵션 중 하나
package you were supplied (see [§1.5](#15-embedding-packages)):
제공된 패키지([§1.5](#15-embedding-packages) 참조):
Google EmbeddingGemma-300M, quantized to Q8_0 (GGUF),
Google EmbeddingGemma-300M, Q8_0(GGUF)으로 양자화,
`models/embeddinggemma-300M-Q8_0.gguf` (~330 MB); or Qwen3-Embedding-0.6B,
`models/embeddinggemma-300M-Q8_0.gguf` (~330MB); 또는 Qwen3-Embedding-0.6B,
f16 (GGUF), `models/Qwen3-Embedding-0.6B-f16.gguf` (~1.2 GB).
f16(GGUF), `models/Qwen3-Embedding-0.6B-f16.gguf`(~1.2GB).
Either
둘 중 하나
converts natural-language queries into vectors for similarity search.
유사성 검색을 위해 자연어 쿼리를 벡터로 변환합니다.
Runs via llama.cpp's built-in HTTP server — no Python, no GPU required.
llama.cpp의 내장 HTTP 서버를 통해 실행됩니다. Python이나 GPU가 필요하지 않습니다.
- **Documentation database**: Weaviate, loaded with a curated,
- **문서 데이터베이스**: 선별된 내용이 포함된 Weaviate
pre-vectorized snapshot of the **full AMD Embedded documentation set**:
**전체 AMD 임베디드 문서 세트**의 사전 벡터화된 스냅샷:
Vivado, Vitis, Power Design Manager, ChipScope, System Software, example
Vivado, Vitis, Power Design Manager, ChipScope, 시스템 소프트웨어, 예
designs, Wiki pages, and Answer Records — shipped as a Weaviate snapshot
디자인, Wiki 페이지 및 답변 기록 — Weaviate 스냅샷으로 제공
and imported automatically on first `amd-embedded-doc-search start`.
첫 번째 `amd-embedded-doc-search start` 시 자동으로 가져옵니다.
During queries,
쿼리하는 동안,
Weaviate calls the configured local llama.cpp embedding endpoint and then
Weaviate는 구성된 로컬 llama.cpp 임베딩 엔드포인트를 호출한 다음
performs hybrid keyword/vector search.
하이브리드 키워드/벡터 검색을 수행합니다.
- **MCP server**: exposes `vivado_doc_search` at
- **MCP 서버**: 'vivado_doc_search'를 다음 위치에 노출합니다.
`http://127.0.0.1:<port>/mcp/doc-search` over the Model Context Protocol —
Model Context Protocol을 통한 `http://127.0.0.1:<port>/mcp/doc-search` —
usable by any MCP-compatible client (VS Code Agent mode, OpenCode, etc.),
모든 MCP 호환 클라이언트(VS Code Agent 모드, OpenCode 등)에서 사용할 수 있습니다.
whichever model that client is talking to.
그 클라이언트가 어떤 모델과 대화하고 있든 상관없습니다.
It contacts Weaviate, not the
Weaviate에 연결되지만
embedding service directly.
직접 임베딩 서비스를 제공합니다.
- **The answering LLM** (external to this stack, by design): this stack
- **응답 LLM**(설계상 이 스택 외부): 이 스택
provides **retrieval only** — you supply a separate model to generate
**검색만** 제공 — 생성할 별도의 모델을 제공합니다.
answers, either a [frontier cloud model](frontier-model.md)
답변은 [프론티어 클라우드 모델](frontier-model.md) 중 하나입니다.
or a [local LLM](local-llm.md).
또는 [로컬 LLM](local-llm.md).
Whichever you pick, it
어느 쪽을 선택하든지
**must support tool calling** so it can invoke the `vivado_doc_search` tool.
**도구 호출을 지원해야** `vivado_doc_search` 도구를 호출할 수 있습니다.

  > **Why not a code-focused model?** Code models tend to answer from
  > **코드 중심 모델이 아닌 이유는 무엇입니까?** 코드 모델은
  > training data instead of calling the search tool, producing plausible
  > 검색 도구를 호출하는 대신 데이터를 훈련하여 그럴듯한 결과를 생성합니다.
  > but potentially wrong answers.
  > 그러나 잠재적으로 잘못된 답변입니다.
  > Use a tool-calling/instruction-following
  > 도구 호출/지시 따르기 사용
  > model so the assistant actually searches the docs.
  > 모델을 사용하면 보조자가 실제로 문서를 검색할 수 있습니다.
  > This applies to both
  > 이는 두 가지 모두에 적용됩니다.
  > frontier and local model choices.
  > 프론티어 및 지역 모델 선택.

### 2.4 File Layout
### 2.4 파일 레이아웃

```text
~/.config/amd-embedded-doc-search/   ← Generated by `amd-embedded-doc-search configure`
├── docker-compose.yaml            ← Stack definition
├── class-config.json              ← Vectorizer configuration
└── .env                           ← Ports, API keys, model settings

Package contents (the archive is flat — no wrapping folder):
├── amd-embedded-doc-search        ← CLI binary (Linux)
├── amd-embedded-doc-search.exe    ← CLI binary (Windows)
├── README.md                      ← Generic amd-embedded-doc-search CLI reference
├── NOTICE                         ← Third-party licenses
├── models/
│   └── <embedding-model>.gguf     ← Embedding model (embeddinggemma-300M-Q8_0.gguf
│                                     or Qwen3-Embedding-0.6B-f16.gguf, per §1.5)
└── images/                        ← Pre-loaded Docker images
    ├── weaviate.tar                ← Weaviate vector database
    ├── llama.cpp.tar                ← llama.cpp embedding server (serves /v1/embeddings)
    ├── doc-search-mcp-server.tar    ← MCP server (exposes the vivado_doc_search tool)
    └── snapshot.tar                 ← Pre-vectorized documentation snapshot (a one-time job imports it into Weaviate on first start)
```

> **Note:** this guide is distributed alongside the archive, not inside it
> **참고:** 이 가이드는 아카이브 내부가 아닌 아카이브와 함께 배포됩니다.
> — the tarball itself only contains the items listed above.
> — 타르볼 자체에는 위에 나열된 항목만 포함되어 있습니다.

---

## 3.
## Deployment
## 배포

This guide is split into two phases, done in order, and only covers
이 가이드는 두 단계로 나누어 순서대로 수행되며 다음 내용만 다룹니다.
deploying the **RAG database itself**.
**RAG 데이터베이스 자체** 배포.
Whichever answering-model option
어떤 응답 모델 옵션이든
you pick ([frontier](frontier-model.md) or
([frontier](frontier-model.md)를 선택하거나
[local](local-llm.md)) has its own client-connection steps in
[local](local-llm.md))에는 자체 클라이언트 연결 단계가 있습니다.
its own document — come back here first.
자체 문서 — 먼저 여기로 돌아오세요.

- **Phase 1 — 🌐 Connected machine.** Download everything into a *transfer
- **1단계 — 🌐 연결된 머신.** 모든 것을 *전송으로 다운로드합니다.
bundle* (a folder you move across the air gap).
번들*(공극을 가로질러 이동하는 폴더).
Nothing is installed
아무것도 설치되지 않았습니다
here.
여기.
- **Phase 2 — 🔒 Air-gapped machine.** Install and configure everything
- **2단계 — 🔒 에어갭 머신.** 모든 것을 설치하고 구성합니다.
from that transfer bundle.
해당 전송 번들에서.

> **Same machine for both phases?
> **두 단계 모두에 동일한 기계가 있습니까?
> That's fine.** The two phases describe
> 괜찮습니다.** 두 단계에서는 다음과 같이 설명합니다.
> *network states*, not necessarily *two different machines*.
> *네트워크 상태*, 반드시 *두 개의 서로 다른 시스템*일 필요는 없습니다.
> If your
> 만약 당신의
> policy allows it, you can temporarily connect the target machine to the
> 정책에서 허용하는 경우 대상 시스템을 임시로 연결할 수 있습니다.
> Internet, complete Phase 1 directly on it (skipping the transfer-bundle
> 인터넷에서 직접 1단계를 완료하세요(이전 번들 건너뛰기)
> copy step), then disconnect it (unplug the cable, disable Wi-Fi, remove
> 복사 단계) 그런 다음 연결을 끊습니다(케이블 분리, Wi-Fi 비활성화, 제거
> the route — whatever "air-gapped" means in your environment) before
> 경로 — 귀하의 환경에서 "air-gapped"가 의미하는 모든 것)
> starting Phase 2.
> 2단계 시작.

> **Deploying with a frontier model instead?** The RAG database itself is
> **대신 프론티어 모델을 사용하여 배포하시겠습니까?** RAG 데이터베이스 자체는
> still deployed exactly as below (it never talks to the internet), but
> 여전히 아래와 같이 정확하게 배포되지만(인터넷과 통신하지 않음)
> your *overall* environment won't be air-gapped once you connect a
> 일단 연결하면 *전체* 환경에 에어갭이 발생하지 않습니다.
> frontier model as the answering LLM — see
> LLM에 응답하는 프론티어 모델 — 참조
> [Frontier Model](frontier-model.md)
> [프론티어 모델](frontier-model.md)
> for that (much shorter) path, which doesn't require Phase 1's
> 1단계가 필요하지 않은 (훨씬 더 짧은) 경로의 경우
> Docker/LLM download steps below beyond the MCP package itself.
> MCP 패키지 자체를 넘어서는 Docker/LLM 다운로드 단계는 다음과 같습니다.

Decide up front which client you will use (VS Code or OpenCode), because it
어떤 클라이언트(VS Code 또는 OpenCode)를 사용할지 미리 결정하세요.
changes what you download:
다운로드하는 내용이 변경됩니다.

- **Option A — VS Code (recommended).** Most engineers already have it
- **옵션 A - VS Code(권장).** 대부분의 엔지니어는 이미 VS Code를 보유하고 있습니다.
installed, it has a familiar GUI with a built-in Chat/Agent view.
설치하면 채팅/에이전트 보기가 내장된 친숙한 GUI가 있습니다.
Works
작품
with either a frontier model (sign in normally) or a local model via
프론티어 모델(일반적으로 로그인) 또는 로컬 모델을 통해
**Bring Your Own Key (BYOK)** — no extra extension required either way.
**BYOK(Bring Your Own Key)** — 어느 쪽이든 추가 확장이 필요하지 않습니다.
- **Option B — OpenCode (optional).** A terminal-based AI client.
- **옵션 B — OpenCode(선택 사항).** 터미널 기반 AI 클라이언트.
- Install
- 설치
it only if you specifically prefer a CLI-driven workflow over an IDE.
IDE보다 CLI 기반 워크플로를 특별히 선호하는 경우에만 사용하세요.

| You want<br>당신이 원하는 | Download in Phase 1<br>1단계에서 다운로드 | Install in Phase 2<br>2단계에서 설치 |
|----------|---------------------|--------------------|
| AMD Embedded Documentation MCP (required, this document)<br>AMD 임베디드 문서 MCP(필수, 이 문서) | Step 1<br>1단계 | Step 2<br>2단계 |
| **VS Code** (recommended)<br>**VS 코드**(권장) | Step 2<br>2단계 | See client docs below<br>아래 클라이언트 문서를 참조하세요. |
| **OpenCode** (optional)<br>**오픈코드**(선택사항) | Step 3<br>3단계 | See client docs below<br>아래 클라이언트 문서를 참조하세요. |
| A local LLM (only if using the [Local LLM](local-llm.md) option)<br>로컬 LLM([로컬 LLM](local-llm.md) 옵션을 사용하는 경우에만) | See that document<br>해당 문서를 참조하세요 | See that document<br>해당 문서를 참조하세요 |

### 3.1 Phase 1
### 3.1 1단계

> **Getting the files across the air gap:** gather everything into one
> **공극을 통해 파일 가져오기:** 모든 것을 하나로 모으기
> *transfer bundle* and move it using whatever transfer method your
> *번들 전송*을 수행하고 원하는 전송 방법을 사용하여 이동합니다.
> environment approves — a USB drive, a guarded/one-way file transfer, an
> 환경 승인 - USB 드라이브, 보호된/단방향 파일 전송,
> approved secure share, etc.
> 승인된 보안 공유 등

Suggested folder layout:
권장되는 폴더 레이아웃:

```text
transfer/
├── MCP/                      (required)
│   └── <package>.tar.gz
│
├── Docker/                   (only if the target has no Docker)
│   ├── docker-<version>.tgz            (Engine static tarball — Linux)
│   └── docker-compose-linux-x86_64     (Compose plugin binary — Linux)
│                                        (or: Docker Desktop installer on Windows)
│
├── VSCode/                   (recommended)
│   └── code.deb  (or code.rpm / VSCodeSetup.exe)
│
├── OpenCode/                 (optional)
│   └── opencode.tar.gz
│
└── Lemonade/                 (only for the Local LLM option — see local-llm.md)
```

**Step 1 — Copy the AMD Embedded Documentation MCP package (required)**
**1단계 - AMD Embedded Documentation MCP 패키지 복사(필수)**

Copy the `amd-embedded-doc-search` package supplied to you into `transfer/MCP/`.
제공된 `amd-embedded-doc-search` 패키지를 `transfer/MCP/`에 복사합니다.
The archive name changes between builds.
아카이브 이름은 빌드 간에 변경됩니다.
It identifies the bundled embedding
번들 임베딩을 식별합니다.
model (see [§1.5](#15-embedding-packages)); either variant works, so use the
모델([§1.5](#15-embedding-packages) 참조); 두 변형 모두 작동하므로 다음을 사용하십시오.
file you were given.
당신이 준 파일.

> **Where do I get this package?**
> **이 패키지는 어디서 구할 수 있나요?**
>
> Use the AMD Embedded Local Knowledge Base package supplied for your
> 귀하의 컴퓨터에 제공된 AMD Embedded Local Knowledge Base 패키지를 사용하십시오.
> entitlement.
> 자격.
> Pick **one** variant (they differ only by the bundled
> **한 가지** 변형을 선택하세요(번들만 다릅니다).
> embedding model, per [§1.5](#15-embedding-packages)),
> [§1.5](#15-embedding-packages)에 따라 임베딩 모델,
> obtain it while you still have network access, and copy it into
> 네트워크에 액세스할 수 있는 동안 이를 확보하여 다음 위치에 복사하십시오.
> `transfer/MCP/`.
> `이전/MCP/`.
> You cannot fetch it later from the air-gapped side.
> 나중에 에어갭이 있는 쪽에서는 가져올 수 없습니다.

`amd-embedded-doc-search` is Docker-based (Weaviate + an embedding model + the MCP
`amd-embedded-doc-search`는 Docker 기반입니다(Weaviate + 임베딩 모델 + MCP
server), so **the target machine must have Docker**.
서버)이므로 **대상 머신에 Docker가 있어야 합니다**.
If it doesn't already
아직 그렇지 않은 경우
have Docker, add it to `transfer/Docker/`:
Docker가 있으면 `transfer/Docker/`에 추가하세요.

- **Linux (recommended):** the **Engine static tarball** (`docker-<version>.tgz`)
- **Linux(권장):** **엔진 정적 타르볼**(`docker-<version>.tgz`)
from <https://download.docker.com/linux/static/stable/> plus the **Compose
<https://download.docker.com/linux/static/stable/> 및 **Compose에서
plugin binary** (`docker-compose-linux-x86_64`) from
플러그인 바이너리** (`docker-compose-linux-x86_64`)
  <https://github.com/docker/compose/releases>. The static tarball is
self-contained, so it avoids the offline `.deb`/`.rpm` dependency problem.
독립적이므로 오프라인 `.deb`/`.rpm` 종속성 문제를 방지합니다.
- **Windows:** the Docker Desktop installer, from
- **Windows:** Docker Desktop 설치 프로그램, 출처:
  <https://docs.docker.com/desktop/setup/install/windows-install/>.

You do **not** need to stage any container images — the `amd-embedded-doc-search` package
컨테이너 이미지(`amd-embedded-doc-search` 패키지)를 스테이징할 필요가 **없습니다**
already bundles them and loads them on first start.
이미 번들로 묶어서 처음 시작할 때 로드합니다.

**Step 2 — Download VS Code (recommended)**
**2단계 - VS Code 다운로드(권장)**

Download the installer for the **target machine's** OS into `transfer/VSCode/`:
**대상 머신** OS용 설치 프로그램을 `transfer/VSCode/`에 다운로드합니다.

- Download page: <https://code.visualstudio.com/download>
- 다운로드 페이지: <https://code.visualstudio.com/download>
- Windows: `VSCodeSetup.exe` · Ubuntu/Debian: `code.deb` · RHEL/Fedora: `code.rpm`
- Windows: `VSCodeSetup.exe` · Ubuntu/Debian: `code.deb` · RHEL/Fedora: `code.rpm`

**Step 3 — Download OpenCode (optional)**
**3단계 - OpenCode 다운로드(선택 사항)**

Only if you plan to use OpenCode instead of VS Code:
VS Code 대신 OpenCode를 사용하려는 경우에만 다음을 수행하세요.

- Releases: <https://github.com/sst/opencode/releases>
- 릴리스: <https://github.com/sst/opencode/releases>
- Docs: <https://opencode.ai/docs/>
- 문서: <https://opencode.ai/docs/>

**Step 4 — Confirm the transfer bundle is complete**
**4단계 - 이전 번들이 완료되었는지 확인**

Before leaving the connected machine (you cannot download anything after
연결된 기기를 떠나기 전(이후에는 아무것도 다운로드할 수 없습니다)
this point):
이 점):

- ✅ `transfer/MCP/` — the AMD Embedded Documentation MCP package
- ✅ `transfer/MCP/` — AMD 임베디드 문서 MCP 패키지
- ✅ `transfer/Docker/` — Docker Engine + Compose (only if the target has no Docker)
- ✅ `transfer/Docker/` — Docker 엔진 + Compose(대상에 Docker가 없는 경우에만)
- ✅ `transfer/VSCode/` — VS Code installer (if using VS Code)
- ✅ `transfer/VSCode/` — VS Code 설치 프로그램(VS Code를 사용하는 경우)
- ✅ `transfer/OpenCode/` — OpenCode package (if using OpenCode)
- ✅ `transfer/OpenCode/` — OpenCode 패키지(OpenCode를 사용하는 경우)

If you plan to run a local LLM too, also gather the items listed in
로컬 LLM도 운영할 계획이라면 다음에 나열된 항목도 수집하세요.
[Local LLM](local-llm.md)'s transfer-bundle
[로컬 LLM](local-llm.md)의 편입 번들
section before leaving the connected machine.
연결된 기기를 떠나기 전에 섹션을 참조하세요.

Now move the bundle to the air-gapped machine and continue with Phase 2.
이제 번들을 에어 갭 머신으로 이동하고 2단계를 계속합니다.

### 3.2 Phase 2
### 3.2 2단계

> **Known air-gap gotchas — skim these first**
> **알려진 에어 갭 문제 — 먼저 훑어보세요**
>
> These failures show up only on locked-down, offline, or NFS-backed hosts —
> 이러한 오류는 잠긴 호스트, 오프라인 또는 NFS 지원 호스트에서만 나타납니다.
> not on a normal connected laptop.
> 일반 연결된 노트북에는 없습니다.
> Each is *symptom → cause → what to do*;
> 각각은 *증상 → 원인 → 조치사항*입니다.
> adapt the specifics to your environment.
> 세부 사항을 환경에 맞게 조정하세요.
>
> - **`mcp` container stuck in `Created`** → the one-time documentation import
> - **`mcp` 컨테이너가 `Created`에 멈춤** → 일회성 문서 가져오기
> is still running (~10 min) → wait; watch it with
> 여전히 실행 중입니다(~10분) → 기다리세요. 그걸로 지켜봐
> `docker logs -f amd-embedded-doc-search-weaviate-docs-import-1`.
> `docker 로그 -f amd-embedded-doc-search-weaviate-docs-import-1`.
> It starts on its own.
> 그것은 스스로 시작됩니다.
> - **Client can't reach the server (`HTTP 000`)** → `localhost` resolved to
> - **클라이언트가 서버에 연결할 수 없습니다(`HTTP 000`)** → `localhost`가 다음으로 확인되었습니다.
> IPv6 `::1`, but Docker publishes on IPv4 → use **`127.0.0.1`** in the URL.
> IPv6 `::1`, 그러나 Docker는 IPv4에 게시 → URL에 **`127.0.0.1`**을 사용합니다.
> - **`permission denied … mkdir …/models`** → your home is on NFS with
> - **`권한이 거부되었습니다 ... mkdir .../models`** → 집이 NFS에 있습니다.
> `root_squash` (check `stat -f -c '%T' "$HOME"` → `nfs`) → keep deployment
> `root_squash`(`stat -f -c '%T' "$HOME"` → `nfs` 확인) → 배포 유지
> state on a **local disk**, not your NFS home — the quickest fix is
> NFS 홈이 아닌 **로컬 디스크**에 상태를 저장합니다. 가장 빠른 해결 방법은 다음과 같습니다.
> `export XDG_CONFIG_HOME=/local/path` before running any
> 실행하기 전에 `XDG_CONFIG_HOME=/local/path` 내보내기
> `amd-embedded-doc-search` command; it redirects config/data generation
> `amd-embedded-doc-search` 명령; 구성/데이터 생성을 리디렉션합니다.
> away from `~/.config` entirely.
> `~/.config`에서 완전히 벗어나세요.
> - **`configure` fails with `could not open a new TTY`** → its interactive UI
> - **`configure`가 `새 TTY를 열 수 없습니다`**와 함께 실패 → 대화형 UI
> needs a **real terminal** → run it in an interactive shell (or drive it
> **실제 터미널**이 필요합니다 → 대화형 셸에서 실행(또는 구동)
> through a `pty` wrapper for automation).
> 자동화를 위한 'pty' 래퍼를 통해).
> - **Offline `.deb`/`.rpm` install fails on dependencies** → package managers
> - **종속성으로 인한 오프라인 `.deb`/`.rpm` 설치 실패** → 패키지 관리자
> don't resolve deps offline → gather each package **with its dependencies**
> deps를 오프라인으로 해결하지 마세요 → 각 패키지를 **종속성과 함께** 수집하세요
> on the connected machine, or use **self-contained builds** (tarballs/static
> 연결된 머신에서 실행하거나 **자체 포함 빌드**(타르볼/정적)를 사용하세요.
> binaries).
> 바이너리).
> - **`docker ps` → `permission denied … docker.sock`** → you need **non-sudo
> - **`docker ps` → `권한이 거부되었습니다. docker.sock`** → **non-sudo가 필요합니다.
> Docker access** (the `docker` group) → on a secured host this is an admin
> Docker 액세스**('docker' 그룹) → 보안 호스트에서는 관리자입니다.
> action; ask your sysadmin.
> 행동; 시스템 관리자에게 물어보세요.

**Step 1 — Install Docker (only if the machine doesn't already have it)**
**1단계 - Docker 설치(머신에 아직 Docker가 없는 경우에만)**

`amd-embedded-doc-search` is a Docker-based stack (Weaviate + an embedding model + the MCP
`amd-embedded-doc-search`는 Docker 기반 스택(Weaviate + 임베딩 모델 + MCP)입니다.
server), so the target machine needs Docker. **Skip this step entirely if
서버)이므로 대상 머신에는 Docker가 필요합니다. **다음 경우에는 이 단계를 완전히 건너뛰세요.
`docker ps` already works.**
`docker ps`는 이미 작동합니다.**

#### Linux (static tarball)
#### Linux(정적 타르볼)


Extract the Engine binaries onto your `PATH`:
`PATH`에 엔진 바이너리를 추출합니다.

```bash
tar -xvzf transfer/Docker/docker-*.tgz
sudo cp docker/* /usr/bin/
```

Install the Compose plugin binary so `docker compose` works:
`docker compose`가 작동하도록 Compose 플러그인 바이너리를 설치합니다.

```bash
sudo mkdir -p /usr/local/lib/docker/cli-plugins
sudo cp transfer/Docker/docker-compose-linux-x86_64 \
    /usr/local/lib/docker/cli-plugins/docker-compose
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
```

The static tarball does **not** include a systemd service file — create one
정적 타르볼은 시스템 서비스 파일을 포함하지 **않습니다** — 하나 생성
so Docker starts on boot:
Docker는 부팅 시 시작됩니다.

```bash
sudo tee /etc/systemd/system/docker.service > /dev/null <<'EOF'
[Unit]
Description=Docker Application Container Engine
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/bin/dockerd
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

Then enable and start Docker:
그런 다음 Docker를 활성화하고 시작합니다.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now docker
```

Add your user to the `docker` group so you can run Docker without `sudo`,
`sudo` 없이 Docker를 실행할 수 있도록 `docker` 그룹에 사용자를 추가하세요.
then **reboot** — a log out/log in is *not* sufficient for the new group
그런 다음 **재부팅** — 새 그룹에는 로그아웃/로그인만으로는 충분하지 *않습니다*
membership to take effect:
멤버십이 적용됩니다:

```bash
sudo groupadd docker
sudo usermod -aG docker $USER
sudo reboot
```

#### Windows (Docker Desktop)
#### Windows(도커 데스크탑)

Run the **Docker Desktop** installer from `transfer/Docker/` (download it on
`transfer/Docker/`에서 **Docker Desktop** 설치 프로그램을 실행합니다(다운로드:
the connected machine from
연결된 기계에서
<https://docs.docker.com/desktop/setup/install/windows-install/>). Docker
Desktop bundles the Engine and the Compose plugin and runs them as a
Desktop은 Engine과 Compose 플러그인을 번들로 묶어서 실행합니다.
background service, so none of the Linux binary-copy, systemd, or
백그라운드 서비스이므로 Linux 바이너리 복사본, systemd 또는
`docker`-group steps are needed.
`docker` 그룹 단계가 필요합니다.

When installation finishes and the Docker Desktop dashboard shows **Engine
설치가 완료되고 Docker Desktop 대시보드에 **Engine이 표시되면
running**, continue.
실행**, 계속하세요.

Confirm Docker is working before continuing (Linux: no `sudo` needed):
계속하기 전에 Docker가 작동하는지 확인하세요(Linux: `sudo` 필요 없음).

```bash
docker ps
```

**Step 2 — Install the AMD Embedded Documentation MCP server (required)**
**2단계 - AMD Embedded Documentation MCP 서버 설치(필수)**

Extract the package you copied into `transfer/MCP/`, then check the environment
`transfer/MCP/`에 복사한 패키지를 추출한 후 환경을 확인하세요.
and bring the stack up.
그리고 스택을 올려보세요.
Unlike older drops, the archive is **flat** — it
이전 드롭과 달리 아카이브는 **플랫**입니다.
extracts directly into the current directory, with no wrapping folder to `cd`
`cd`에 대한 래핑 폴더 없이 현재 디렉토리로 직접 추출합니다.
into:
에:

```bash
tar -xvzf transfer/MCP/<package>.tar.gz
./amd-embedded-doc-search doctor            # verify Docker is ready
./amd-embedded-doc-search configure --start # generate config and start the stack
```

> **On Windows**
> **Windows의 경우**
>
> The `tar` command above is the same, but run the CLI with the `.exe` build
> 위의 `tar` 명령은 동일하지만 `.exe` 빌드로 CLI를 실행합니다.
> from PowerShell — e.g.
> PowerShell에서 - 예:
> `.\amd-embedded-doc-search.exe doctor` and
> `.\amd-embedded-doc-search.exe doctor` 및
> `.\amd-embedded-doc-search.exe configure --start`.
> `.\amd-embedded-doc-search.exe 구성 --start`.

The MCP port is chosen during `configure` and saved in
MCP 포트는 '구성' 중에 선택되어 다음 위치에 저장됩니다.
`~/.config/amd-embedded-doc-search/.env` as `DOCSEARCH_MCP_PORT` (default `8080`):
`~/.config/amd-embedded-doc-search/.env`를 `DOCSEARCH_MCP_PORT`로(기본값 `8080`):

```text
http://127.0.0.1:<DOCSEARCH_MCP_PORT>/mcp/doc-search
```

Full command reference, flags, validation, and troubleshooting:
전체 명령 참조, 플래그, 검증 및 문제 해결:
**[Operate & Troubleshoot](operate.md)**.
**[작동 및 문제 해결](operate.md)**.

**Step 3 — Connect an answering model**
**3단계 - 응답 모델 연결**

Continue with whichever fits your hardware and network policy:
하드웨어 및 네트워크 정책에 맞는 것을 계속 진행하세요.

- [Frontier Model](frontier-model.md) —
- [프론티어 모델](frontier-model.md) —
no local-LLM hardware required, works on any laptop
로컬 LLM 하드웨어가 필요하지 않으며 모든 노트북에서 작동합니다.
- [Local LLM](local-llm.md) — fully
- [로컬 LLM](local-llm.md) — 전체
air-gapped end to end, requires GPU-class hardware
엔드 투 엔드(air-gapped end-to-end), GPU급 하드웨어 필요

---

## 4.
## Next Steps
## 다음 단계

The RAG database is now deployed and validated.
이제 RAG 데이터베이스가 배포되고 검증되었습니다.
Pick your answering model:
응답 모델을 선택하세요.

- **No GPU / just want to try it quickly?** →
- **GPU가 없습니다. 빨리 사용해 보고 싶으신가요?** →
[Frontier Model](frontier-model.md)
[프론티어 모델](frontier-model.md)
(GitHub Copilot in VS Code, a few minutes of setup)
(VS Code의 GitHub Copilot, 설정에 몇 분 소요)
- **Need a fully air-gapped end-to-end deployment?** →
- **완전히 에어갭이 있는 엔드투엔드 배포가 필요합니까?** →
[Local LLM](local-llm.md) (AMD ROCm,
[로컬 LLM](local-llm.md) (AMD ROCm,
includes local-LLM setup and VRAM planning)
로컬 LLM 설정 및 VRAM 계획 포함)

---

<p class="sphinxhide" align="center"><sub>Copyright © 2026 Advanced Micro Devices, Inc</sub></p>
<p class="sphinxhide" align="center"><sup><a href="https://www.amd.com/en/corporate/copyright">Terms and Conditions</a></sup></p>
