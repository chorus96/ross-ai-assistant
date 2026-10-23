<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Operate & Troubleshoot the RAG Stack
# RAG 스택 작동 및 문제 해결

Command reference for the `amd-embedded-doc-search` CLI, plus validation and
'amd-embedded-doc-search' CLI에 대한 명령 참조와 검증 및
troubleshooting for the local RAG database.
로컬 RAG 데이터베이스 문제 해결.
For first-time deployment, see
처음 배포하는 경우 다음을 참조하세요.
[RAG Database Setup](README.md).
[RAG 데이터베이스 설정](README.md).

Examples use the command name alone.
예제에서는 명령 이름만 사용합니다.
From the extracted package directory,
추출된 패키지 디렉터리에서
run `./amd-embedded-doc-search` instead, unless that binary is already on
해당 바이너리가 이미 켜져 있지 않은 한 `./amd-embedded-doc-search`를 대신 실행하세요.
your `PATH`.
당신의 'PATH'.

## Prerequisites
## 전제조건

Install Docker Desktop (Windows) or Docker Engine + Compose plugin
Docker Desktop(Windows) 또는 Docker Engine + Compose 플러그인 설치
(Linux): <https://docs.docker.com/get-started/get-docker/>
(리눅스): <https://docs.docker.com/get-started/get-docker/>

```console
$ docker ps
$ docker compose version
```

| Requirement<br>요구 사항 | Why<br>왜? |
|-------------|-----|
| Docker Engine + Compose plugin<br>Docker 엔진 + Compose 플러그인 | All three services run as containers<br>세 가지 서비스 모두 컨테이너로 실행됩니다. |
| ~8 GB free disk<br>~8GB의 여유 디스크 | Weaviate snapshot + model + container images<br>스냅샷 + 모델 + 컨테이너 이미지 Weaviate |
| An answering model (frontier or local)<br>응답 모델(프론티어 또는 지역) | Generates answers from retrieved docs<br>검색된 문서에서 답변을 생성합니다. |
| An MCP client (VS Code or OpenCode)<br>MCP 클라이언트(VS Code 또는 OpenCode) | User interface for chat<br>채팅용 사용자 인터페이스 |

## Environment Check
## 환경점검

```console
$ amd-embedded-doc-search doctor
```

Checks the Docker CLI, daemon, and Compose plugin; prints an exact
Docker CLI, 데몬, Compose 플러그인을 확인합니다. 정확한 내용을 인쇄합니다
remediation command for anything missing.
누락된 내용에 대한 수정 명령입니다.
Safe to run any time — installs
언제든지 안전하게 실행할 수 있습니다. — 설치
nothing.
아무것도.

## Quick Start
## 빠른 시작

```console
$ amd-embedded-doc-search doctor       # 1. check environment
$ amd-embedded-doc-search configure    # 2. generate deployment files (interactive)
$ amd-embedded-doc-search start        # 3. bring the stack up
```

Or in one step: `amd-embedded-doc-search configure --start`.
또는 한 단계로 `amd-embedded-doc-search 구성 --start`를 수행합니다.

> **First start takes time.** `start` (and `configure --start`) runs
> **처음 시작하는 데 시간이 걸립니다.** `start`(및 `configure --start`)가 실행됩니다.
> `docker compose up -d`, which returns almost immediately — but the
> `docker compose up -d`는 거의 즉시 반환되지만
> containers keep initializing in the background afterward: Weaviate has
> 컨테이너는 나중에 백그라운드에서 계속 초기화됩니다. Weaviate가
> to import the pre-built snapshot (the full AMD Embedded documentation
> 사전 구축된 스냅샷을 가져오려면(전체 AMD Embedded 문서
> corpus) and llama.cpp has to load the embedding model.
> 말뭉치) 및 llama.cpp는 임베딩 모델을 로드해야 합니다.
> This can take a
> 이 작업에는 시간이 걸릴 수 있습니다.
> few minutes depending on disk speed, even though the images themselves
> 이미지 자체가 디스크 속도에 따라 다르지만 몇 분
> are already pre-loaded from `images/`.
> `images/`에서 이미 사전 로드되어 있습니다.
> Run `amd-embedded-doc-search ps` or
> `amd-embedded-doc-search ps`를 실행하거나
> `amd-embedded-doc-search logs --follow` to confirm all three containers are healthy
> 세 개의 컨테이너가 모두 정상인지 확인하려면 `amd-embedded-doc-search 로그 --follow`를 사용하세요.
> before connecting a client — querying too early can return empty or
> 클라이언트에 연결하기 전에 — 너무 일찍 쿼리하면 비어 있거나
> failed results.
> 실패한 결과.

## Configure
## 구성

1. **Collection** — the document snapshot to import (the AMD Embedded
1. **컬렉션** — 가져올 문서 스냅샷(AMD Embedded
documentation corpus package supplied to you)
귀하에게 제공된 문서 코퍼스 패키지)
2. **Embedding model** — the vectorizer to use for semantic search
2. **임베딩 모델** — 의미 검색에 사용할 벡터화 도구
3. **MCP port** — host port for the MCP server (defaults to the first free
3. **MCP 포트** — MCP 서버용 호스트 포트(기본값은 첫 번째 무료 포트)
port at or above `8080`)
'8080' 이상의 포트)

For embedding models served via the OpenAI API you'll also be asked for a
OpenAI API를 통해 제공되는 임베딩 모델의 경우
**base URL** and **API key** — these are stored in `.env` and never echoed
**기본 URL** 및 **API 키** — `.env`에 저장되며 에코되지 않습니다.
to the terminal.
터미널로.
Weaviate's own API key is auto-generated on first run and
Weaviate 자체 API 키는 최초 실행 시 자동 생성되며,
reused automatically on subsequent `configure` runs; you never need to
후속 `configure` 실행 시 자동으로 재사용됩니다. 넌 절대 그럴 필요 없어
provide it.
그것을 제공하십시오.

All generated files are written to `~/.config/amd-embedded-doc-search/`:
생성된 모든 파일은 `~/.config/amd-embedded-doc-search/`에 기록됩니다.

| File<br>파일 | Purpose<br>목적 |
| --- | --- |
| `docker-compose.yaml`<br>`docker-compose.yaml` | stack definition<br>스택 정의 |
| `class-config.json`<br>`클래스 구성.json` | vectorizer config mounted by the importer container<br>임포터 컨테이너에 의해 마운트된 벡터화기 구성 |
| `.env`<br>`.env` | persisted settings (model, ports, API keys)<br>지속 설정(모델, 포트, API 키) |

Key flags:
주요 플래그:

| Flag<br>깃발 | Default<br>기본값 | Description<br>설명 |
| --- | --- | --- |
| `--gpu`<br>`--gpu` | `false`<br>'거짓' | Enable AMD ROCm GPU acceleration for the embedding model<br>임베딩 모델에 대해 AMD ROCm GPU 가속 활성화 |
| `--llama-image`<br>`--라마-이미지` | `ghcr.io/ggml-org/llama.cpp:server`<br>`ghcr.io/ggml-org/llama.cpp:서버` | llama.cpp image (self-hosted models only)<br>llama.cpp 이미지(자체 호스팅 모델만 해당) |
| `--mcp-image`<br>`--mcp-이미지` | `vivado.amd.com/doc-search-mcp-server:0.9.0`<br>`vivado.amd.com/doc-search-mcp-server:0.9.0` | MCP server image<br>MCP 서버 이미지 |
| `--mcp-port`<br>`--mcp-포트` | first free port ≥ `8080`<br>첫 번째 자유 포트 ≥ `8080` | Host port for the MCP server<br>MCP 서버용 호스트 포트 |
| `--start`<br>`--시작` | `false`<br>'거짓' | Start the stack after writing configuration files<br>구성 파일을 작성한 후 스택 시작 |
| `--weaviate-image`<br>`--weaviate-이미지` | `cr.weaviate.io/semitechnologies/weaviate:1.38.2`<br>`cr.weaviate.io/semitechnologies/weaviate:1.38.2` | Weaviate image<br>위비 이미지 |

## Lifecycle
## 수명주기

| Command<br>명령 | What it does<br>기능 |
|---------|-------------|
| `amd-embedded-doc-search doctor`<br>`AMD-임베디드-문서-검색 의사` | Read-only health check of Docker environment<br>Docker 환경의 읽기 전용 상태 확인 |
| `amd-embedded-doc-search configure [--start]`<br>`amd-embedded-doc-search 구성 [--start]` | Generate/update deployment files (interactive)<br>배포 파일 생성/업데이트(대화형) |
| `amd-embedded-doc-search start [--follow]`<br>`amd-embedded-doc-search 시작 [--follow]` | Start the stack (`docker compose up -d`)<br>스택 시작(`docker compose up -d`) |
| `amd-embedded-doc-search stop`<br>`amd-임베디드-문서-검색 중지` | Stop containers, keep data volumes<br>컨테이너 중지, 데이터 볼륨 유지 |
| `amd-embedded-doc-search ps`<br>`amd-임베디드-문서-검색 ps` | Show container status<br>컨테이너 상태 표시 |
| `amd-embedded-doc-search logs [--follow]`<br>`amd-embedded-doc-search 로그 [--follow]` | View container logs<br>컨테이너 로그 보기 |
| `amd-embedded-doc-search stats`<br>`amd-embedded-doc-search 통계` | Live CPU/memory/IO per container<br>컨테이너당 라이브 CPU/메모리/IO |
| `amd-embedded-doc-search purge [--yes]`<br>`amd-embedded-doc-search 제거 [--yes]` | **Destructive**: remove all data and config<br>**파괴적**: 모든 데이터 및 구성 제거 |

> **Switching between embedding-model packages
> **임베딩 모델 패키지 간 전환
> ([RAG Database Setup §1.5](README.md#15-embedding-packages))
> ([RAG 데이터베이스 설정 §1.5](README.md#15-embedding-packages))
> on the same machine?
> 같은 기계에서?
> Use `purge`, not `stop`.** `amd-embedded-doc-search stop`
> `stop`이 아닌 `purge`를 사용하세요.** `amd-embedded-doc-search stop`
> intentionally preserves the Weaviate data volume, so if you re-run
> 의도적으로 Weaviate 데이터 볼륨을 보존하므로 다시 실행하면
> `amd-embedded-doc-search configure` pointed at a *different* embedding-model package
> *다른* 임베딩 모델 패키지를 가리키는 `amd-embedded-doc-search 구성`
> (e.g. going from `embeddinggemma-300m` to `qwen3-embedding-0.6b`), the
> (예: `embeddinggemma-300m`에서 `qwen3-embedding-0.6b`로 이동)
> importer sees the collection already exists, skips loading the new
> 수입자는 컬렉션이 이미 존재하는 것을 확인하고 새 컬렉션 로드를 건너뜁니다.
> snapshot, and the old vectors are reused as-is.
> 스냅샷, 이전 벡터는 그대로 재사용됩니다.
> Because the two models
> 두 모델이 있기 때문에
> produce vectors of different lengths (EmbeddingGemma-300M = 768 dims,
> 다양한 길이의 벡터 생성(EmbeddingGemma-300M = 768개,
> Qwen3-Embedding-0.6B = 1024 dims), any subsequent search fails with an
> Qwen3-Embedding-0.6B = 1024 희미함), 후속 검색은 다음과 같이 실패합니다.
> error like `vector lengths don't match`.
> '벡터 길이가 일치하지 않습니다'와 같은 오류가 발생했습니다.
> Run `amd-embedded-doc-search purge --yes`
> `amd-embedded-doc-search purge --yes`를 실행하세요.
> before `amd-embedded-doc-search configure` whenever you change embedding-model
> 임베딩 모델을 변경할 때마다 `amd-embedded-doc-search 구성` 전에
> packages, so the stack starts from a clean volume and imports the correct
> 패키지이므로 스택은 깨끗한 볼륨에서 시작하여 올바른 볼륨을 가져옵니다.
> data.
> 데이터.
> On a clean volume, first-time import re-embeds the full document
> 깨끗한 볼륨에서 처음 가져올 때 전체 문서를 다시 포함합니다.
> corpus using the bundled model and can take several minutes (observed:
> 번들 모델을 사용하는 코퍼스는 몇 분 정도 걸릴 수 있습니다(관찰:
> ~10 minutes for ~714k chunks on CPU) — this is expected, one-time work,
> CPU에서 ~714k 청크의 경우 ~10분) — 예상되는 일회성 작업입니다.
> not a hang.
> 멈추지 않습니다.

## Global Flags
## 글로벌 플래그

| Flag<br>깃발 | Description<br>설명 |
| --- | --- |
| `--accessible`<br>`--접근 가능` | Use accessible (screen-reader-friendly) TUI forms — pass before any subcommand, e.g. `amd-embedded-doc-search --accessible configure`<br>접근 가능한(화면 판독기 친화적인) TUI 양식을 사용합니다. 예를 들어 하위 명령 앞에 전달합니다. `amd-embedded-doc-search --접근 가능 구성` |

## Connect a Client
## 클라이언트 연결

> **Not automatic.** `amd-embedded-doc-search configure`/`start`/`ps` only manage the
> **자동이 아닙니다.** `amd-embedded-doc-search 구성`/`start`/`ps`는
> Docker stack itself — they never touch VS Code's or OpenCode's config
> Docker 스택 자체 - VS Code 또는 OpenCode의 구성을 건드리지 않습니다.
> files.
> 파일.
> You still add this block yourself to the client's MCP config
> 이 블록을 클라이언트의 MCP 구성에 직접 추가하면 됩니다.
> (`.vscode/mcp.json`, etc.) — see
> (`.vscode/mcp.json` 등) — 참조
> [Frontier Model](frontier-model.md)
> [프론티어 모델](frontier-model.md)
> or [Local LLM](local-llm.md) for the exact
> 또는 정확한 내용을 보려면 [로컬 LLM](local-llm.md)
> client steps for your chosen model option.
> 선택한 모델 옵션에 대한 클라이언트 단계.

```json
"amd-embedded-doc-search": {
  "type": "http",
  "url": "http://127.0.0.1:<DOCSEARCH_MCP_PORT>/mcp/doc-search"
}
```

Port defaults to `8080`, stored in `~/.config/amd-embedded-doc-search/.env` as
포트 기본값은 `8080`이며 `~/.config/amd-embedded-doc-search/.env`에 다음과 같이 저장됩니다.
`DOCSEARCH_MCP_PORT`.
`DOCSEARCH_MCP_PORT`.

## Validation
## 검증

**MCP server:**
**MCP 서버:**

```bash
amd-embedded-doc-search ps
```
- ✅ The `amd-embedded-doc-search` containers are up.
- ✅ `amd-embedded-doc-search` 컨테이너가 작동 중입니다.

**Client:** ask in VS Code Agent mode or in OpenCode:
**클라이언트:** VS Code Agent 모드 또는 OpenCode에서 요청:

```text
Search UG904 for incremental compile.
```
- ✅ The `vivado_doc_search` tool is invoked.
- ✅ `vivado_doc_search` 도구가 호출됩니다.
- ✅ Documentation is returned.
- ✅ 서류가 반송되었습니다.

This confirms the RAG database itself is healthy, independent of which
이를 통해 RAG 데이터베이스 자체가 정상임을 확인합니다.
answering model you're using.
사용 중인 응답 모델입니다.
If this works but the *overall* answer
이것이 작동하지만 *전체* 대답이
quality or connectivity has problems, check the model-specific validation
품질이나 연결에 문제가 있는 경우 모델별 유효성 검사를 확인하세요.
checklist in
체크리스트
[Frontier Model](frontier-model.md) or
[프론티어 모델](frontier-model.md) 또는
[Local LLM](local-llm.md).
[로컬 LLM](local-llm.md).

## Troubleshooting
## 문제 해결

| Symptom<br>증상 | Likely cause / fix<br>가능한 원인/수정 사항 |
|---------|--------------------|
| MCP tool not offered<br>MCP 도구는 제공되지 않습니다 | Chat must be in **Agent** mode; confirm `.vscode/mcp.json` (or equivalent) is valid and reload the client.<br>채팅은 **에이전트** 모드여야 합니다. `.vscode/mcp.json`(또는 이에 상응하는 항목)이 유효한지 확인하고 클라이언트를 다시 로드하세요. |
| MCP server not reachable<br>MCP 서버에 연결할 수 없습니다 | Run `amd-embedded-doc-search ps` and `amd-embedded-doc-search doctor`; confirm the client URL matches `http://127.0.0.1:<DOCSEARCH_MCP_PORT>/mcp/doc-search`.<br>`amd-embedded-doc-search ps` 및 `amd-embedded-doc-search doctor`를 실행하세요. 클라이언트 URL이 `http://127.0.0.1:<DOCSEARCH_MCP_PORT>/mcp/doc-search`와 일치하는지 확인하세요. |
| Search fails with `vector lengths don't match` (e.g. `768 vs 1024`)<br>'벡터 길이가 일치하지 않습니다'(예: '768 대 1024')로 인해 검색이 실패합니다. | You switched embedding-model packages ([RAG Database Setup §1.5](README.md#15-embedding-packages)) using `amd-embedded-doc-search stop` instead of `amd-embedded-doc-search purge`, so the old package's vectors are still in the data volume. Run `amd-embedded-doc-search purge --yes` then `amd-embedded-doc-search configure --start` to reimport cleanly with the new model.<br>`amd-embedded-doc-search purge` 대신 `amd-embedded-doc-search stop`을 사용하여 포함 모델 패키지([RAG 데이터베이스 설정 §1.5](README.md#15-embedding-packages))를 전환했으므로 이전 패키지의 벡터가 여전히 데이터 볼륨에 있습니다. `amd-embedded-doc-search purge --yes`를 실행한 다음 `amd-embedded-doc-search 구성 --start`를 실행하여 새 모델로 깔끔하게 다시 가져옵니다. |
| MCP client gets `Connection reset by peer` right after a fresh import finishes<br>MCP 클라이언트는 새로운 가져오기가 완료된 직후 '피어에 의한 연결 재설정'을 받습니다. | Expected and harmless: the `amd-embedded-doc-search-mcp-1` container restarts itself once the importer finishes populating Weaviate, and needs ~15-30 seconds to come back up. Wait briefly (`docker ps --filter name=amd-embedded-doc-search-mcp-1` until `STATUS` shows it's been up for at least 15-30s) and retry the request.<br>예상 가능하고 무해함: 가져오기 도구가 Weaviate 채우기를 완료하면 'amd-embedded-doc-search-mcp-1' 컨테이너가 자동으로 다시 시작되며, 다시 시작하는 데 ~15~30초가 필요합니다. 잠시 기다렸다가(`STATUS`에 최소 15~30초 동안 작동 중이라고 표시될 때까지 `docker ps --filter name=amd-embedded-doc-search-mcp-1`) 요청을 다시 시도하세요. |
| Model gives fewer retries than expected on a RAG query<br>모델은 RAG 쿼리에서 예상보다 적은 수의 재시도를 제공합니다. | Check the retrieved chunks — RAG corpus contamination can cause a model to give up early on a poor-quality search result; this is a data issue, not necessarily a model issue.<br>검색된 청크 확인 — RAG 코퍼스 오염으로 인해 모델이 품질이 낮은 검색 결과로 인해 조기에 포기될 수 있습니다. 이는 데이터 문제이지 반드시 모델 문제는 아닙니다. |

For model-connection issues specifically (model not listed, tool calling
특히 모델 연결 문제의 경우(모델이 나열되지 않음, 도구 호출
not working, utility models, VRAM, etc.), see the troubleshooting section
작동하지 않음, 실용신안, VRAM 등), 문제 해결 섹션을 참조하세요.
in [Frontier Model](frontier-model.md)
[프론티어 모델](frontier-model.md)
or [Local LLM](local-llm.md).
또는 [로컬 LLM](local-llm.md).

<p class="sphinxhide" align="center"><sub>Copyright © 2026 Advanced Micro Devices, Inc</sub></p>
<p class="sphinxhide" align="center"><sup><a href="https://www.amd.com/en/corporate/copyright">Terms and Conditions</a></sup></p>
