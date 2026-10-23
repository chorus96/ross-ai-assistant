<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Quick Start — Deploy the Local RAG Database
# 빠른 시작 - 로컬 RAG 데이터베이스 배포

The fastest way to stand up the Local RAG documentation database on a
로컬 RAG 문서 데이터베이스를 구축하는 가장 빠른 방법
**connected machine** (internet available) — three steps.
**연결된 머신**(인터넷 사용 가능) — 3단계.
If you need a
당신이 필요로하는 경우
fully air-gapped install using a transfer bundle, use
전송 번들을 사용하여 완전 에어갭 설치, 사용
[RAG Database Setup](README.md) instead.
[RAG 데이터베이스 설정](README.md) 대신.

> **What you'll end up with**
> **결국은 어떻게 될까요**
>
> A local `amd-embedded-doc-search` stack (vector database + embedding model + MCP
> 로컬 `amd-embedded-doc-search` 스택(벡터 데이터베이스 + 임베딩 모델 + MCP
> server) running in Docker, exposing a `vivado_doc_search` tool your AI
> 서버) Docker에서 실행되어 'vivado_doc_search' 도구를 AI에 노출
> assistant can call — with no documentation queries leaving your machine.
> 보조자가 전화를 걸 수 있습니다. 컴퓨터에서 문서에 대한 문의가 필요하지 않습니다.

## a.
## Install Docker
## 도커 설치

Install Docker Desktop (Windows) or Docker Engine + Compose plugin
Docker Desktop(Windows) 또는 Docker Engine + Compose 플러그인 설치
(Linux): <https://docs.docker.com/get-started/get-docker/>
(리눅스): <https://docs.docker.com/get-started/get-docker/>

Verify it's working:
작동하는지 확인하세요.

```bash
docker ps
docker compose version
```

## b.
## Download and extract the package
## 패키지 다운로드 및 추출

Download **one** variant of the AMD Embedded Local Knowledge Base package
AMD Embedded Local Knowledge Base 패키지 **한 가지** 변형 다운로드
supplied for your entitlement (the two variants differ only by the bundled
귀하의 자격에 대해 제공됩니다(두 가지 변형은 번들로 제공되는 버전만 다릅니다).
embedding model).
임베딩 모델).
The archive name changes from build to build, so extract
아카이브 이름은 빌드마다 변경되므로 추출하십시오.
the file you downloaded:
다운로드한 파일:

```bash
tar xzf <package>.tar.gz
```

This unpacks the `amd-embedded-doc-search` CLI, the embedding `models/`, and the
그러면 `amd-embedded-doc-search` CLI, 임베딩 `models/` 및
pre-loaded Docker `images/` into the current directory.
Docker `images/`를 현재 디렉터리에 미리 로드했습니다.

## c.
## Deploy
## 배포

From the extracted folder:
추출된 폴더에서:

```bash
./amd-embedded-doc-search doctor              # confirm Docker is ready
./amd-embedded-doc-search configure --start   # generate config and bring the stack up
```

`configure` is interactive — the defaults are fine:
`configure`는 대화형입니다. 기본값은 괜찮습니다:

1. **Collection** — the document snapshot to import (name depends on the package build)
1. **컬렉션** — 가져올 문서 스냅샷(이름은 패키지 빌드에 따라 다름)
2. **Embedding model** — the bundled model
2. **임베딩 모델** — 번들 모델
3. **MCP port** — the first free port at or above `8080`
3. **MCP 포트** — `8080` 이상의 첫 번째 사용 가능한 포트

> **Optional: GPU acceleration**
> **선택사항: GPU 가속**
>
> If the machine has a supported AMD GPU (ROCm), add `--gpu` to
> 머신에 지원되는 AMD GPU(ROCm)가 있는 경우 `--gpu`를 추가하세요.
> `configure` to run the embedding model on the GPU, which speeds up the
> GPU에서 임베딩 모델을 실행하도록 '구성'하여 속도를 높입니다.
> one-time import.
> 일회성 수입.

> **First start often takes about 10 minutes**
> **처음 시작하는 데 보통 10분 정도 걸립니다**
>
> On first run the stack imports the documentation snapshot and loads the
> 처음 실행 시 스택은 문서 스냅샷을 가져오고
> embedding model.
> 임베딩 모델.
> The `mcp` container can stay in `Created` until that
> `mcp` 컨테이너는 그 때까지 `Created` 상태로 유지될 수 있습니다.
> import finishes.
> 수입이 끝났습니다.
> Follow progress with `./amd-embedded-doc-search logs --follow`, and
> `./amd-embedded-doc-searchlogs --follow`를 사용하여 진행 상황을 따르고
> confirm all containers are healthy with `./amd-embedded-doc-search ps` before
> 전에 `./amd-embedded-doc-search ps`를 사용하여 모든 컨테이너가 정상인지 확인하세요.
> connecting a client — querying too early can return empty results.
> 클라이언트 연결 — 너무 일찍 쿼리하면 빈 결과가 반환될 수 있습니다.

Confirm the stack is up:
스택이 작동 중인지 확인합니다.

```bash
./amd-embedded-doc-search ps
```

## Connect your AI assistant
## AI 어시스턴트 연결

Note the MCP port (printed at the end of `configure`, and stored in
MCP 포트를 기록해 두십시오(`configure` 끝에 인쇄되어 다음 위치에 저장됨).
`~/.config/amd-embedded-doc-search/.env` as `DOCSEARCH_MCP_PORT`, default `8080`).
`~/.config/amd-embedded-doc-search/.env`를 `DOCSEARCH_MCP_PORT`로, 기본값은 `8080`).
Add
추가
this to your MCP client configuration — for VS Code, create
이를 MCP 클라이언트 구성에 적용합니다. VS Code의 경우
`.vscode/mcp.json`:
`.vscode/mcp.json`:

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

Replace `8080` with your actual port if different, then reload the client.
'8080'을 실제 포트로 바꾼 후 클라이언트를 다시 로드하세요.
In **Agent mode**, ask a Vivado / Vitis / PDM / ChipScope question and
**에이전트 모드**에서 Vivado / Vitis / PDM / ChipScope 질문을 하고
confirm the assistant calls the `vivado_doc_search` tool.
어시스턴트가 'vivado_doc_search' 도구를 호출하는지 확인하세요.

## Everyday commands
## 일상적인 명령

| Command<br>명령 | What it does<br>기능 |
|---|---|
| `./amd-embedded-doc-search ps`<br>`./amd-embedded-doc-search ps` | Show container status<br>컨테이너 상태 표시 |
| `./amd-embedded-doc-search logs --follow`<br>`./amd-embedded-doc-search 로그 --follow` | Stream container logs<br>스트림 컨테이너 로그 |
| `./amd-embedded-doc-search stop`<br>`./amd-embedded-doc-search 중지` | Stop containers (data is preserved)<br>컨테이너 중지(데이터는 보존됨) |
| `./amd-embedded-doc-search start`<br>`./amd-embedded-doc-search 시작` | Start the stack again<br>스택을 다시 시작하세요 |
| `./amd-embedded-doc-search purge --yes`<br>`./amd-embedded-doc-search 제거 --예` | Remove all data and config (start over)<br>모든 데이터 및 구성 제거(다시 시작) |

## Where to go next
## 다음엔 어디로 갈까

- **Fully air-gapped install** (transfer bundle, two phases) → [RAG Database Setup](README.md)
- **완전한 에어갭 설치**(번들 전송, 2단계) → [RAG 데이터베이스 설정](README.md)
- **Use it with a cloud model** (no GPU needed) → [Frontier Model (Cloud)](frontier-model.md)
- **클라우드 모델과 함께 사용** (GPU 필요 없음) → [Frontier Model(Cloud)](frontier-model.md)
- **Use it with a local, offline LLM** → [Local LLM (Air-Gapped)](local-llm.md)
- **로컬, 오프라인 LLM과 함께 사용** → [로컬 LLM(Air-Gapped)](local-llm.md)

---

<p class="sphinxhide" align="center"><sub>Copyright © 2026 Advanced Micro Devices, Inc</sub></p>
<p class="sphinxhide" align="center"><sup><a href="https://www.amd.com/en/corporate/copyright">Terms and Conditions</a></sup></p>
