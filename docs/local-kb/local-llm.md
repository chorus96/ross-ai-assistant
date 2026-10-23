<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Fully Air-Gapped Local LLM Deployment (AMD ROCm)
# 완전히 Air-Gapped 로컬 LLM 배포(AMD ROCm)

**Deploy a local, tool-calling-capable LLM alongside the RAG database for
**다음을 위해 RAG 데이터베이스와 함께 도구 호출이 가능한 로컬 LLM을 배포합니다.
a fully air-gapped, zero-outbound-network deployment end to end.**
완전한 에어갭, 제로 아웃바운드 네트워크 배포 엔드투엔드.**

> **Deployment model:** Lemonade is a model manager and can download a ROCm
> **배포 모델:** Lemonade는 모델 관리자이며 ROCm을 다운로드할 수 있습니다.
> `llama.cpp` backend; it is not the backend itself.
> `llama.cpp` 백엔드; 백엔드 자체는 아닙니다.
> The reproducible,
> 재현 가능한,
> air-gapped procedure in this document stages that complete backend and runs
> 이 문서의 에어갭 절차는 백엔드를 완료하고 실행하는 단계입니다.
> `llama-server` directly.
> `llama-server`를 직접 사용합니다.
> Lemonade server mode remains optional and must use
> 레모네이드 서버 모드는 선택 사항으로 남아 있으며 다음을 사용해야 합니다.
> the exact cache layout for the staged Lemonade version.
> 단계적 레모네이드 버전의 정확한 캐시 레이아웃.

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

Use this document if your environment requires that **no traffic ever
**트래픽이 전혀 없는 환경이 필요한 경우 이 문서를 사용하세요.
leaves the machine** — not even to a frontier model's cloud API.
기계**를 떠납니다. 심지어 개척자 모델의 클라우드 API에도 해당되지 않습니다.
This
이
requires GPU-class hardware capable of running a modern, tool-calling LLM
최신 도구 호출 LLM을 실행할 수 있는 GPU급 하드웨어가 필요합니다.
at usable speed: a supported AMD GPU running ROCm, with enough VRAM for
사용 가능한 속도: ROCm을 실행하는 지원되는 AMD GPU, 충분한 VRAM 포함
your chosen model.
당신이 선택한 모델.

> **Note on the host machine.** The host that runs the *answering LLM* is
> **호스트 시스템에 대한 참고 사항.** *응답 LLM*을 실행하는 호스트는 다음과 같습니다.
> a separate x86 machine with a supported AMD GPU — it is unrelated to the
> 지원되는 AMD GPU를 갖춘 별도의 x86 시스템 — 이는
> AMD FPGA or Adaptive SoC you are designing for.
> 귀하가 설계하고 있는 AMD FPGA 또는 적응형 SoC.
> Adjust the paths, ports,
> 경로, 포트 조정
> and flags below to match your machine.
> 그리고 아래에 귀하의 컴퓨터와 일치하는 플래그를 지정하세요.

If you don't have hardware like this (e.g.
이와 같은 하드웨어가 없는 경우(예:
a standard laptop), see
표준 노트북), 참조
[Frontier Model](frontier-model.md)
[프론티어 모델](frontier-model.md)
instead — it reuses the same RAG database with no local-LLM hardware
대신 — 로컬 LLM 하드웨어 없이 동일한 RAG 데이터베이스를 재사용합니다.
required, at the cost of not being fully air-gapped.
완전히 에어 갭이 발생하지 않는 대가로 필요합니다.

---

## 1.
## Transfer Bundle
## 번들 전송

If you're building the transfer bundle described in
에 설명된 전송 번들을 빌드하는 경우
[RAG Database Setup §3.1](README.md#31-phase-1),
[RAG 데이터베이스 설정 §3.1](README.md#31-phase-1),
add a local-LLM folder before leaving the connected machine:
연결된 시스템을 떠나기 전에 local-LLM 폴더를 추가하십시오.

```text
transfer/
└── Lemonade/                 (only if the target has no local LLM)
    ├── lemonade-installer-<version>.<package>
    ├── llamacpp-rocm/         (complete ROCm inference runtime — see below)
    ├── MANIFEST.sha256
    └── models/
        ├── Qwen3.6-35B-A3B-GGUF/
        │   ├── Qwen3.6-35B-A3B-UD-Q4_K_XL.gguf
        │   └── mmproj-F16.gguf
        └── gemma-4-26B-A4B-it-GGUF/
            ├── gemma-4-26B-A4B-it-UD-Q4_K_M.gguf
            └── mmproj-F16.gguf
```

**Download the Lemonade installer package** for the target's exact Linux
**레모네이드 설치 프로그램 패키지를 다운로드**하여 대상의 정확한 Linux를 확인하세요.
distribution, architecture, and Lemonade version.
배포, 아키텍처 및 레모네이드 버전.
The
는
[Lemonade installation guide](https://lemonade-server.ai/docs/guide/install/)
[레모네이드 설치 가이드](https://lemonade-server.ai/docs/guide/install/)
selects the package for each platform; it is not itself an installer binary.
각 플랫폼에 대한 패키지를 선택합니다. 그 자체로는 설치 프로그램 바이너리가 아닙니다.
Record the package URL and version in the transfer manifest.
전송 매니페스트에 패키지 URL과 버전을 기록합니다.

**Download the GPU inference backend (ROCm).** The Lemonade installer does
**GPU 추론 백엔드(ROCm)를 다운로드하세요.** 레모네이드 설치 프로그램은
**not** include the GPU inference engine.
**아님** GPU 추론 엔진은 포함되지 않습니다.
Lemonade downloads the ROCm
레모네이드는 ROCm을 다운로드합니다.
`llama.cpp` backend (`llama-server` and its ROCm libraries) from the internet
인터넷의 `llama.cpp` 백엔드(`llama-server` 및 해당 ROCm 라이브러리)
the first time it runs a model — which fails on an air-gapped host.
모델을 처음 실행하면 에어갭 호스트에서 실패합니다.
Capture it
캡처
now, on a connected staging machine with the same Linux distribution,
이제 동일한 Linux 배포판을 사용하는 연결된 스테이징 머신에서
architecture, ROCm compatibility, and GPU family as the target:
아키텍처, ROCm 호환성 및 GPU 제품군을 대상으로 합니다.

1. Install Lemonade and load any model **once** so it fetches the ROCm backend
1. Lemonade를 설치하고 모든 모델을 **한 번** 로드하여 ROCm 백엔드를 가져옵니다.
into its cache, then locate the downloaded `llamacpp/rocm` backend folder
캐시에 넣은 다음 다운로드한 `llamacpp/rocm` 백엔드 폴더를 찾습니다.
(its path depends on the install method — check Lemonade's cache/bin dir).
(경로는 설치 방법에 따라 다릅니다. Lemonade의 캐시/bin 디렉토리를 확인하세요.)
2. Copy the **complete contents** of that `rocm/` folder — `llama-server`, its
2. 해당 `rocm/` 폴더의 **전체 내용**을 복사합니다 — `llama-server`,
shared libraries, and subdirectories — into
공유 라이브러리 및 하위 디렉토리 —
`transfer/Lemonade/llamacpp-rocm/`.
`이동/레모네이드/llamacpp-rocm/`.
Do not copy only the executable.
실행 파일만 복사하지 마십시오.
3. Confirm the staged binary can start and record its version:
3. 준비된 바이너리가 시작되고 해당 버전을 기록할 수 있는지 확인합니다.
   ```bash
   transfer/Lemonade/llamacpp-rocm/llama-server --version
   ```

> **Confirm the backend location**
> **백엔드 위치 확인**
>
> Where Lemonade caches the downloaded backend depends on the Lemonade
> Lemonade가 다운로드한 백엔드를 캐시하는 위치는 Lemonade에 따라 다릅니다.
> release and install method.
> 릴리스 및 설치 방법.
> Do not invent a Lemonade cache path on the
> 레모네이드 캐시 경로를 만들지 마세요.
> target.
> 목표.
> This guide installs the staged runtime independently at
> 이 가이드에서는 단계별 런타임을 독립적으로 설치합니다.
> `/opt/local-llm/runtime/` for direct `llama-server` use (see
> `llama-server`를 직접 사용하려면 `/opt/local-llm/runtime/`(참조
> [§3.1](#31-runtime)).
> [§3.1](#31-런타임)).
> If you choose Lemonade server mode, transfer and
> 레모네이드 서버 모드를 선택하면 전송 및
> restore its cache tree in the exact layout produced by the matching
> 매칭에 의해 생성된 정확한 레이아웃으로 캐시 트리를 복원합니다.
> Lemonade release.
> 레모네이드 출시.

Lemonade runs **GGUF** model files hosted on Hugging Face — you download
Lemonade는 Hugging Face에서 호스팅되는 **GGUF** 모델 파일을 실행합니다. 다운로드하면 됩니다.
the files on the connected machine; Lemonade picks them up on the
연결된 컴퓨터의 파일 레모네이드는 그들을 데리러 간다.
air-gapped machine afterward.
나중에 에어 갭 머신.

The listed multimodal model variants include the model itself plus a small
나열된 다중 모드 모델 변형에는 모델 자체와 작은 모델이 포함됩니다.
`mmproj` companion.
`mmproj` 동반자.
Download exactly these files into
다음 파일을 정확히 다운로드하세요.
`transfer/Lemonade/models/` (keep each model in its own subfolder, since
`transfer/Lemonade/models/`(각 모델을 자체 하위 폴더에 보관합니다.
the `mmproj-F16.gguf` files share the same name across repos):
`mmproj-F16.gguf` 파일은 저장소 전체에서 동일한 이름을 공유합니다.)

| Model<br>모델 | Repo<br>레포 | File<br>파일 | Size<br>크기 |
|---|---|---|---|
| **Qwen3.6-35B-A3B** (recommended)<br>**Qwen3.6-35B-A3B** (권장) | [`unsloth/Qwen3.6-35B-A3B-GGUF`](https://huggingface.co/unsloth/Qwen3.6-35B-A3B-GGUF)<br>[`unsloth/Qwen3.6-35B-A3B-GGUF`](https://huggingface.co/unsloth/Qwen3.6-35B-A3B-GGUF) | `Qwen3.6-35B-A3B-UD-Q4_K_XL.gguf` + `mmproj-F16.gguf`<br>`Qwen3.6-35B-A3B-UD-Q4_K_XL.gguf` + `mmproj-F16.gguf` | ~22 GB + ~0.9 GB<br>~22GB + ~0.9GB |
| **Gemma-4-26B-A4B MoE** (recommended)<br>**Gemma-4-26B-A4B MoE**(권장) | [`unsloth/gemma-4-26B-A4B-it-GGUF`](https://huggingface.co/unsloth/gemma-4-26B-A4B-it-GGUF)<br>[`unsloth/gemma-4-26B-A4B-it-GGUF`](https://huggingface.co/unsloth/gemma-4-26B-A4B-it-GGUF) | `gemma-4-26B-A4B-it-UD-Q4_K_M.gguf` + `mmproj-F16.gguf`<br>`gemma-4-26B-A4B-it-UD-Q4_K_M.gguf` + `mmproj-F16.gguf` | ~16 GB + ~1.2 GB<br>~16GB + ~1.2GB |

> **Why not a coder model?** A code-focused model tends to fabricate
> **코더 모델이 아닌 이유는 무엇입니까?** 코드 중심 모델은 조작하는 경향이 있습니다.
> answers instead of calling the search tool — see
> 검색 도구를 호출하는 대신 답변 — 참조
> [RAG Database Setup §2.3](README.md#23-components).
> [RAG 데이터베이스 설정 §2.3](README.md#23-구성 요소).

Before leaving the connected machine, also confirm:
연결된 기기에서 내리기 전에 다음 사항도 확인하세요.
- ✅ `transfer/Lemonade/` — versioned Lemonade installer, complete ROCm backend
- ✅ `transfer/Lemonade/` — 버전이 지정된 Lemonade 설치 프로그램, 완전한 ROCm 백엔드
(`llamacpp-rocm/`), and `.gguf` model files
(`llamacpp-rocm/`) 및 `.gguf` 모델 파일
- ✅ The staged `llama-server --version` command succeeds
- ✅ 단계적 `llama-server --version` 명령이 성공합니다.
- ✅ `MANIFEST.sha256` covers every staged runtime and model file.
- ✅ `MANIFEST.sha256`은 모든 단계적 런타임 및 모델 파일을 다룹니다.
- Create it
- 만들기
after all files are copied:
모든 파일을 복사한 후:
  ```bash
  (cd transfer/Lemonade && find llamacpp-rocm models -type f -print0 \
    | sort -z | xargs -0 sha256sum) > MANIFEST.sha256
  ```

---

## 2.
## Installation
## 설치

Skip this whole section if the air-gapped machine already has a working
에어 갭이 있는 기계에 이미 작동 중인 경우 이 전체 섹션을 건너뛰십시오.
OpenAI-compatible LLM.
OpenAI 호환 LLM.

> **Does "OpenAI-compatible" mean this needs Internet access?
> **"OpenAI 호환"이란 인터넷 액세스가 필요하다는 뜻입니까?
> No.** It
> 아니요.**
> refers only to the **API schema** — the same request/response format
> **API 스키마**만 참조합니다 — 동일한 요청/응답 형식
> popularized by OpenAI (`/v1/chat/completions`, `/v1/models`, etc.).
> OpenAI(`/v1/chat/completions`, `/v1/models` 등)에 의해 대중화되었습니다.
> The
> 는
> local `llama-server` (or Lemonade, if selected) implements that same schema
> 로컬 `llama-server`(또는 선택한 경우 Lemonade)는 동일한 스키마를 구현합니다.
> **locally**, on `127.0.0.1` — no OpenAI account, API key, or cloud
> **로컬**, `127.0.0.1` — OpenAI 계정, API 키 또는 클라우드 없음
> connection is involved.
> 연결이 포함됩니다.
> It's the same reason
> 같은 이유야
> `curl http://127.0.0.1:8000/v1/models` works entirely offline.
> `curl http://127.0.0.1:8000/v1/models`는 완전히 오프라인으로 작동합니다.

1. Verify the transfer bundle before installing or running anything:
1. 무엇이든 설치하거나 실행하기 전에 전송 번들을 확인하세요.
   ```bash
   (cd transfer/Lemonade && sha256sum -c MANIFEST.sha256)
   ```
2. Install the delivered Lemonade package only if you intend to use Lemonade's
2. 레모네이드를 사용하려는 경우에만 제공된 레모네이드 패키지를 설치하십시오.
own server or model-management commands.
자체 서버 또는 모델 관리 명령.
It is not needed to run the staged
Staged를 실행할 필요는 없습니다.
runtime directly.
직접 런타임.
Follow the package's platform-specific offline install
패키지의 플랫폼별 오프라인 설치를 따릅니다.
procedure; all OS package dependencies must already be available on the
절차; 모든 OS 패키지 종속성은 이미
target.
목표.
3. Install the staged runtime and models in stable locations for direct,
3. 직접 실행을 위해 안정적인 위치에 준비된 런타임과 모델을 설치합니다.
fully offline serving:
완전 오프라인 제공:
   ```bash
   sudo install -d -m 0755 /opt/local-llm/runtime /var/lib/local-llm/models
   sudo cp -a transfer/Lemonade/llamacpp-rocm/. /opt/local-llm/runtime/
   sudo cp -a transfer/Lemonade/models/. /var/lib/local-llm/models/
   sudo chmod -R a-w /opt/local-llm/runtime
   sudo /opt/local-llm/runtime/llama-server --version
   ```

The service user used in [§7](#7-systemd-persistence) must be able to read
[§7](#7-systemd-persistence)에서 사용하는 서비스 사용자는 다음을 읽을 수 있어야 합니다.
`/var/lib/local-llm/models/` and access the ROCm device nodes.
`/var/lib/local-llm/models/` 및 ROCm 장치 노드에 액세스합니다.
Configure
구성
those permissions according to the target platform's ROCm policy.
대상 플랫폼의 ROCm 정책에 따라 해당 권한을 부여합니다.

4. **Optional: Lemonade server mode.** Restore the complete Lemonade cache
4. **선택 사항: 레모네이드 서버 모드.** 전체 레모네이드 캐시 복원
tree captured from the matching staging installation; do not substitute the
일치하는 스테이징 설치에서 캡처된 트리; 대체하지 마십시오
direct-runtime path above for Lemonade's cache.
레모네이드 캐시에 대한 위의 직접 런타임 경로입니다.
After the Lemonade service
레모네이드 서비스 후
is running, register the local models and prohibit all fetches:
실행 중이면 로컬 모델을 등록하고 모든 가져오기를 금지합니다.
   ```bash
   lemonade config set host=localhost offline=true no_fetch_executables=true \
     extra_models_dir=/var/lib/local-llm/models
   lemonade status
   curl http://127.0.0.1:13305/api/v1/models
   ```

`13305` is Lemonade's default port; use the configured port reported by
'13305'는 레모네이드의 기본 포트입니다. 보고된 구성된 포트를 사용하십시오.
`lemonade status` if it differs.
'레모네이드 상태'가 다른 경우.
`offline=true` prevents model downloads,
`offline=true`는 모델 다운로드를 방지합니다.
while `no_fetch_executables=true` prevents a missing backend from being
`no_fetch_executables=true`는 누락된 백엔드가 실행되는 것을 방지합니다.
fetched.
가져왔습니다.
Keep the default loopback binding unless you explicitly require
명시적으로 요구하지 않는 한 기본 루프백 바인딩을 유지합니다.
remote access and have configured a firewall and API authentication.
원격 액세스를 수행하고 방화벽 및 API 인증을 구성했습니다.

Lemonade registers `.gguf` files in `extra_models_dir` as `extra.<name>`.
레모네이드는 `extra_models_dir`의 `.gguf` 파일을 `extra.<name>`으로 등록합니다.
Expected: a JSON response listing your models. **Note the exact model
예상: 모델을 나열하는 JSON 응답입니다. **정확한 모델을 참고하세요
`id` values** — needed for client configuration.
`id` 값** — 클라이언트 구성에 필요합니다.

   > For AMD ROCm hardware, also see
   > AMD ROCm 하드웨어의 경우도 참조하세요.
   > [§3](#3-deployment) and
   > [§3](#3-배포) 및
   > [§6](#6-vram-planning) for tuned launch flags,
   > [§6](#6-vram-planning) 조정된 발사 플래그의 경우,
   > multi-model VRAM budgeting, and the recommended 2-model configuration
   > 다중 모델 VRAM 예산 책정 및 권장되는 2모델 구성
   > used in this deployment.
   > 이번 배포에 사용되었습니다.
   > In short: `--flash-attn on`,
   > 간단히 말해서: `--flash-attn on`,
   > `--cache-type-k q8_0 --cache-type-v q8_0`, and `--parallel 1` for
   > `--cache-type-k q8_0 --cache-type-v q8_0` 및 `--parallel 1`
   > Gemma models — see those sections for the full picture.
   > Gemma 모델 - 전체 그림을 보려면 해당 섹션을 참조하세요.

> **Which port will my client use?** Lemonade's built-in server defaults to
> **내 클라이언트는 어떤 포트를 사용합니까?** 레모네이드의 내장 서버는 기본적으로
> port `13305` and serves one model at a time — fine for a quick
> '13305' 포트를 사용하고 한 번에 하나의 모델만 제공합니다. — 빠르게는 괜찮습니다.
> single-model start.
> 단일 모델 시작.
> The tuned configuration in
> 조정된 구성
> [§3](#3-deployment) instead runs
> [§3](#3-배포)가 대신 실행됩니다.
> `llama-server` directly on port `8000` (and `8001` for a second model),
> 포트 `8000`(두 번째 모델의 경우 `8001`)에 직접 `llama-server`
> which is exactly what the client configuration in
> 이것이 바로 클라이언트 구성입니다.
> [§4](#4-client-setup) points at.
> [§4](#4-client-setup)이 다음을 가리킵니다.
> Use `13305`
> '13305'를 사용하세요.
> for the simple Lemonade default, or `8000`/`8001` if you follow the tuned
> 간단한 레모네이드 기본값의 경우, 조정된 내용을 따르는 경우 `8000`/`8001`
> two-model launch in §3.
> §3에서 2가지 모델 출시.

Then connect your client — [§4](#4-client-setup).
그런 다음 클라이언트를 연결하세요 — [§4](#4-client-setup).

---

## 3.
## Deployment
## 배포

> **Reference platform.** The launch flags, context sizes, and tuning in
> **참조 플랫폼.** 실행 플래그, 컨텍스트 크기 및 튜닝
> this section were developed and validated on an AMD Ryzen AI Max
> 이 섹션은 AMD Ryzen AI Max에서 개발 및 검증되었습니다.
> "Strix Halo" APU — a unified-memory x86 platform with an integrated AMD
> "Strix Halo" APU — AMD가 통합된 통합 메모리 x86 플랫폼
> GPU, running ROCm.
> ROCm을 실행하는 GPU.
> Treat the values below as a known-good starting point
> 아래 값을 알려진 양호한 시작점으로 취급하십시오.
> and adjust the GFX version, ports, and paths to match your own supported
> 지원되는 버전과 일치하도록 GFX 버전, 포트 및 경로를 조정합니다.
> AMD GPU.
> AMD GPU.

### 3.1 Runtime
### 3.1 런타임

- **Server**: `llama-server` (ROCm build), staged at
- **서버**: `llama-server`(ROCm 빌드), 다음 위치에 스테이징됨
`/opt/local-llm/runtime/llama-server` (see [§1](#1-transfer-bundle)).
`/opt/local-llm/runtime/llama-server`([§1](#1-transfer-bundle) 참조).
- **Runtime environment:** `LD_LIBRARY_PATH` makes the staged ROCm libraries
- **런타임 환경:** `LD_LIBRARY_PATH`는 단계적 ROCm 라이브러리를 만듭니다.
discoverable.
발견 가능.
`HSA_OVERRIDE_GFX_VERSION` is **not** universally required;
`HSA_OVERRIDE_GFX_VERSION`은 보편적으로 필요하지 **않습니다**.
set it only when the ROCm support guidance for the target GPU requires an
대상 GPU에 대한 ROCm 지원 지침에 필요한 경우에만 설정하십시오.
override.
재정의.
The value below is the reference Strix Halo value, not a default
아래 값은 기본값이 아닌 참고 Strix Halo 값입니다.
for other GPUs:
다른 GPU의 경우:
  ```bash
  # Reference platform only; omit unless your target requires this override.
  export HSA_OVERRIDE_GFX_VERSION=11.5.1
  export LD_LIBRARY_PATH=/opt/local-llm/runtime:/opt/local-llm/runtime/hipblaslt:/opt/local-llm/runtime/rocblas
  ```
- **Hardware**: a supported AMD GPU running ROCm.
- **하드웨어**: ROCm을 실행하는 지원되는 AMD GPU.
- On unified-memory APUs,
- 통합 메모리 APU에서는
"VRAM" is shared with system RAM; confirm the available budget via
"VRAM"은 시스템 RAM과 공유됩니다. 를 통해 사용 가능한 예산을 확인하세요.
`rocm-smi --showmeminfo vram`.
`rocm-smi --showmeminfo vram`.

### 3.2 Models
### 3.2 모델

| Model<br>모델 | Notes<br>메모 | Weights + mmproj<br>무게 + mmproj |
|---|---|---|
| **Qwen3.6-35B-A3B** (MoE, ~3B active)<br>**Qwen3.6-35B-A3B**(MoE, ~3B 활성) | Very large context window at low KV-cache cost; reliable tool calling<br>낮은 KV 캐시 비용으로 매우 큰 컨텍스트 창을 제공합니다. 안정적인 도구 호출 | ~22GB + 0.9GB<br>~22GB + 0.9GB |
| **Gemma-4-26B-A4B-it** (MoE, ~3.8B active)<br>**Gemma-4-26B-A4B-it**(MoE, ~3.8B 활성) | Compact footprint; reliable tool calling<br>컴팩트한 설치 공간; 안정적인 도구 호출 | ~16GB + 1.2GB<br>~16GB + 1.2GB |

### 3.3 Launch Flags
### 3.3 출시 플래그

Both models share this baseline, tuned for the AMD ROCm backend:
두 모델 모두 AMD ROCm 백엔드에 맞게 조정된 이 기준을 공유합니다.

```
--n-gpu-layers 99 --reasoning off --batch-size 512 --ubatch-size 256 \
--cache-type-k q8_0 --cache-type-v q8_0 --flash-attn on
```

Gemma models additionally need `--parallel 1` (fixes sliding-window-attention
Gemma 모델에는 추가로 '--parallel 1'이 필요합니다(슬라이딩 창 주의 수정).
checkpoint-cache thrashing observed with the default parallel-slot count).
기본 병렬 슬롯 수를 사용하면 체크포인트 캐시 스래싱이 관찰됩니다.

**Actual launch commands used in this deployment:**
**이 배포에 사용된 실제 시작 명령:**

```bash
# Qwen3.6-35B-A3B — loopback port 8000, ctx 131072
/opt/local-llm/runtime/llama-server \
  --model /var/lib/local-llm/models/Qwen3.6-35B-A3B-GGUF/Qwen3.6-35B-A3B-UD-Q4_K_XL.gguf \
  --mmproj /var/lib/local-llm/models/Qwen3.6-35B-A3B-GGUF/mmproj-F16.gguf \
  --host 127.0.0.1 --port 8000 --ctx-size 131072 --n-gpu-layers 99 --reasoning off \
  --batch-size 512 --ubatch-size 256 --cache-type-k q8_0 --cache-type-v q8_0 --flash-attn on

# Gemma-4-26B-A4B-MoE — loopback port 8001, ctx 32768
/opt/local-llm/runtime/llama-server \
  --model /var/lib/local-llm/models/gemma-4-26B-A4B-it-GGUF/gemma-4-26B-A4B-it-UD-Q4_K_M.gguf \
  --mmproj /var/lib/local-llm/models/gemma-4-26B-A4B-it-GGUF/mmproj-F16.gguf \
  --host 127.0.0.1 --port 8001 --ctx-size 32768 --n-gpu-layers 99 --parallel 1 --reasoning off \
  --batch-size 512 --ubatch-size 256 --cache-type-k q8_0 --cache-type-v q8_0 --flash-attn on
```

Only pass `--mmproj` when the downloaded model release includes a compatible
다운로드한 모델 릴리스에 호환되는 버전이 포함된 경우에만 `--mmproj`를 전달하세요.
multimodal projector and you need vision support.
다중 모달 프로젝터 및 시력 지원이 필요합니다.
Remove that option for a
해당 옵션을 제거하세요.
text-only model.
텍스트 전용 모델.

Verify each server:
각 서버를 확인합니다.

```bash
curl -s http://127.0.0.1:8000/v1/models
curl -s http://127.0.0.1:8001/v1/models
```

---

## 4.
## Client Setup
## 클라이언트 설정

Both clients need the **local LLM** above and the **amd-embedded-doc-search MCP
두 클라이언트 모두 위의 **로컬 LLM**과 **amd-embedded-doc-search MCP가 필요합니다.
server** ([Operate & Troubleshoot](operate.md)).
서버**([운영 및 문제 해결](operate.md)).

```text
        Local LLM (llama-server / Lemonade)
        http://127.0.0.1:8000-8001/v1  (this deployment)
        or http://127.0.0.1:13305/api/v1  (single-model Lemonade default)
                  |
             OpenAI API
                  |
        +---------+---------+
        |                   |
        v                   v
   VS Code (BYOK)       OpenCode
        |                   |
        +---------+---------+
                  |
                  v
        amd-embedded-doc-search MCP server
        http://127.0.0.1:8080/mcp/doc-search
```

### 4.1 VS Code
### 4.1 VS 코드

VS Code's built-in chat supports **Bring Your Own Key (BYOK)** models, so
VS Code의 내장 채팅은 **BYOK(Bring Your Own Key)** 모델을 지원하므로
you can use your local model with **no GitHub account, no Copilot plan,
**GitHub 계정이나 Copilot 계획 없이도 로컬 모델을 사용할 수 있습니다.
and fully offline**.
완전히 오프라인**입니다.

**A.1 — Install VS Code**
**A.1 - VS Code 설치**

- Windows: run `VSCodeSetup.exe` · Ubuntu/Debian: `sudo dpkg -i code.deb` · RHEL/Fedora: `sudo rpm -i code.rpm`
- Windows: `VSCodeSetup.exe` 실행 · Ubuntu/Debian: `sudo dpkg -i code.deb` · RHEL/Fedora: `sudo rpm -i code.rpm`

**A.2 — Add your local model (BYOK, no sign-in)**
**A.2 - 로컬 모델 추가(BYOK, 로그인 없음)**

1. Open the Chat view → model picker → **Manage Language Models** (gear icon), or run **Chat: Manage Language Models** from the Command Palette.
1. 채팅 보기 → 모델 선택기 → **언어 모델 관리**(기어 아이콘)를 열거나 명령 팔레트에서 **채팅: 언어 모델 관리**를 실행합니다.
2. Select **Add Models → Custom Endpoint**.
2. **모델 추가 → 사용자 정의 엔드포인트**를 선택합니다.
3. Name the group (e.g.
3. 그룹 이름을 지정합니다(예:
3. `Lemonade`) and give an API key value of `lemonade` (ignored, but required).
3. `Lemonade`) API 키 값 `lemonade`를 제공합니다(무시되지만 필수).
4. VS Code opens `chatLanguageModels.json` — configure it:
4. VS Code는 `chatLanguageModels.json`을 엽니다. 이를 구성합니다.

```json
[
  {
    "name": "Lemonade",
    "vendor": "customendpoint",
    "apiKey": "lemonade",
    "apiType": "chat-completions",
    "models": [
      {
        "id": "Qwen3.6-35B-A3B-UD-Q4_K_XL.gguf",
        "name": "Qwen3.6 35B A3B (Local)",
        "url": "http://127.0.0.1:8000/v1/chat/completions",
        "toolCalling": true,
        "vision": true,
        "maxInputTokens": 129024,
        "maxOutputTokens": 2048
      },
      {
        "id": "gemma-4-26B-A4B-it-UD-Q4_K_M.gguf",
        "name": "Gemma 4 26B A4B MoE (Local)",
        "url": "http://127.0.0.1:8001/v1/chat/completions",
        "toolCalling": true,
        "vision": true,
        "maxInputTokens": 32000,
        "maxOutputTokens": 2048
      }
    ]
  }
]
```

- Set `id`/`url` to match the exact server and model from `curl .../v1/models` (§3.3) — one entry per running `llama-server` port (8000 = Qwen3.6-35B-A3B, 8001 = Gemma-4-26B-A4B MoE, per §3.2).
- `curl .../v1/models`(§3.3)의 정확한 서버 및 모델과 일치하도록 `id`/`url`을 설정합니다. 실행 중인 `llama-server` 포트당 하나의 항목입니다(8000 = Qwen3.6-35B-A3B, 8001 = Gemma-4-26B-A4B MoE, §3.2에 따라).
- The sample token limits reserve 2048 output tokens from the configured server context; reduce them if you choose a smaller context size.
- 샘플 토큰 제한은 구성된 서버 컨텍스트에서 2048개의 출력 토큰을 예약합니다. 더 작은 컨텍스트 크기를 선택하면 이를 줄이세요.
- **`toolCalling` must be `true`** — the model only appears in Agent mode (required for MCP) if it supports tool calling.
- **`toolCalling`은 `true`여야 합니다** — 모델이 도구 호출을 지원하는 경우 에이전트 모드(MCP에 필요)에만 나타납니다.
- Save, then restart VS Code if the models don't appear right away.
- 모델이 바로 표시되지 않으면 저장한 다음 VS Code를 다시 시작하세요.

**A.3 — Enable utility models (required for offline BYOK)**
**A.3 — 유틸리티 모델 활성화(오프라인 BYOK에 필요)**

Without a GitHub sign-in, VS Code's background helpers (title generation,
GitHub 로그인이 없으면 VS Code의 백그라운드 도우미(제목 생성,
etc.) need a local model too.
등) 로컬 모델도 필요합니다.
In **Settings (JSON)**:
**설정(JSON)**에서:

```json
{
  "chat.utilityModel": "Lemonade",
  "chat.utilitySmallModel": "Lemonade"
}
```

**A.4 — Add the MCP server**
**A.4 - MCP 서버 추가**

Create `.vscode/mcp.json` (or your user `mcp.json`):
`.vscode/mcp.json`(또는 사용자 `mcp.json`)을 만듭니다.

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

Reload VS Code when prompted.
메시지가 표시되면 VS Code를 다시 로드합니다.

**A.5 — Verify**
**A.5 — 확인**

1. Open Chat, switch to **Agent** mode.
1. 채팅을 열고 **에이전트** 모드로 전환하세요.
2. Select your `Lemonade` model.
2. '레모네이드' 모델을 선택하세요.
3. Confirm the `amd-embedded-doc-search` tool is listed in the tools picker.
3. 'amd-embedded-doc-search' 도구가 도구 선택기에 나열되어 있는지 확인하세요.
4. Ask: *"Search the documentation for AXI SmartConnect."* (or any Vivado,
4. 질문: *"AXI SmartConnect에 대한 문서를 검색하세요."* (또는 Vivado,
Vitis, PDM, ChipScope, System Software, example-design, Wiki, or Answer
Vitis, PDM, ChipScope, 시스템 소프트웨어, 예제 디자인, Wiki 또는 Answer
Record topic)
녹음 주제)

Expected: the model calls the `vivado_doc_search` tool and answers using the
예상: 모델은 'vivado_doc_search' 도구를 호출하고 다음을 사용하여 응답합니다.
documentation database.
문서 데이터베이스.

**A.6 — (Optional) The `amd-embedded-doc-search` custom agent — reduce prompt overhead**
**A.6 - (선택 사항) `amd-embedded-doc-search` 사용자 정의 에이전트 - 프롬프트 오버헤드 감소**

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
every turn — the same fixed-overhead problem described for OpenCode's
매 턴 — OpenCode에서 설명한 것과 동일한 고정 오버헤드 문제
default `build` agent in [§4.2.B3](#42-opencode).
[§4.2.B3](#42-opencode)의 기본 `build` 에이전트.

VS Code supports **custom agents** (`.agent.md` files) that restrict which
VS Code는 다음을 제한하는 **사용자 지정 에이전트**(`.agent.md` 파일)를 지원합니다.
tools are available for a given persona — the equivalent of OpenCode's
주어진 페르소나에 대해 도구를 사용할 수 있습니다. 이는 OpenCode와 동일합니다.
`docsearch` agent.
'문서 검색' 에이전트.
Create:
만들다:

```text
~/.copilot/agents/amd-embedded-doc-search.agent.md
```

(A per-VS Code-profile user location;
(VS Code 프로필별 사용자 위치.
`.github/agents/amd-embedded-doc-search.agent.md` in this workspace is the
이 작업공간의 `.github/agents/amd-embedded-doc-search.agent.md`는
team-shared equivalent if you'd rather commit it to a repo.)
리포지토리에 커밋하려는 경우 팀 공유에 상응하는 파일입니다.)

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
agent to **only** the doc-search MCP server's tools, dropping the terminal,
에이전트는 문서 검색 MCP 서버의 도구만 **사용하고, 터미널을 삭제하고,
file-edit, and other built-in tool schemas from the prompt entirely (the
프롬프트에서 파일 편집 및 기타 내장 도구 스키마를 완전히(
same principle as OpenCode's minimal `docsearch` agent in §4.2.B3, just
§4.2.B3에 있는 OpenCode의 최소 `docsearch` 에이전트와 동일한 원칙입니다.
expressed via VS Code's own custom-agent frontmatter instead of OpenCode's
OpenCode 대신 VS Code의 자체 사용자 정의 에이전트 프론트매터를 통해 표현됩니다.
`agent.tools` JSON).
`agent.tools` JSON).

Select it from the agents dropdown in the Chat view
채팅 보기의 상담원 드롭다운에서 선택하세요.
(`@amd-embedded-doc-search`) instead of the default Agent mode when you
(`@amd-embedded-doc-search`)를 사용하는 경우 기본 에이전트 모드 대신
only need documentation lookups.
문서 조회만 필요합니다.

**Why it helps.** Scoping the agent to `tools: [amd-embedded-doc-search/*]` sends only
**도움이 되는 이유.** 에이전트 범위를 `tools: [amd-embedded-doc-search/*]`로 지정하면 전송만 가능
the doc-search tool schema to the model each turn instead of VS Code's full
VS Code의 전체 대신 문서 검색 도구 스키마를 모델에 추가합니다.
built-in tool belt.
내장형 공구 벨트.
That trims the fixed prompt overhead paid before your
그러면 이전에 지불한 고정 프롬프트 오버헤드가 줄어듭니다.
question is processed, which lowers response latency — the same effect as
질문이 처리되어 응답 지연 시간이 단축됩니다.
OpenCode's minimal `docsearch` agent (§4.2.B3).
OpenCode의 최소 `docsearch` 에이전트(§4.2.B3).

### 4.2 OpenCode
### 4.2 오픈코드

Use this if you prefer a terminal-based client.
터미널 기반 클라이언트를 선호하는 경우 이를 사용하십시오.

**B.1 — Install OpenCode**
**B.1 - OpenCode 설치**

```bash
tar xzf opencode.tar.gz
opencode --version
```

**B.2 — Provider and MCP configuration**
**B.2 — 공급자 및 MCP 구성**

`~/.config/opencode/opencode.json`:
`~/.config/opencode/opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "webfetch": "deny",
    "websearch": "deny"
  },
  "provider": {
    "lemonade-qwen36": {
      "name": "Lemonade - Qwen3.6 (local)",
      "npm": "@ai-sdk/openai-compatible",
      "options": { "baseURL": "http://127.0.0.1:8000/v1", "apiKey": "none" },
      "models": {
        "Qwen3.6-35B-A3B-UD-Q4_K_XL.gguf": {
          "name": "Qwen3.6 35B A3B (Lemonade)",
          "contextLength": 131072,
          "maxTokens": 2048
        }
      }
    },
    "lemonade-gemma26moe": {
      "name": "Lemonade - Gemma 4 26B MoE (local)",
      "npm": "@ai-sdk/openai-compatible",
      "options": { "baseURL": "http://127.0.0.1:8001/v1", "apiKey": "none" },
      "models": {
        "gemma-4-26B-A4B-it-UD-Q4_K_M.gguf": {
          "name": "Gemma 4 26B A4B MoE it (Lemonade)",
          "contextLength": 32768,
          "maxTokens": 2048
        }
      }
    }
  },
  "mcp": {
    "amd-embedded-doc-search": {
      "type": "remote",
      "url": "http://127.0.0.1:8080/mcp/doc-search",
      "enabled": true
    }
  }
}
```

> Note each provider points at a **different port** — one llama-server process
> 각 공급자는 **다른 포트**(하나의 라마 서버 프로세스)를 가리킵니다.
> per model, run simultaneously (VRAM budget permitting; see §6).
> 모델별로 동시에 실행됩니다(VRAM 예산 허용, §6 참조).

**B.3 — The `docsearch` agent — minimize prompt overhead**
**B.3 — `docsearch` 에이전트 — 프롬프트 오버헤드 최소화**

OpenCode's default `build` agent exposes its full
OpenCode의 기본 `build` 에이전트는 전체를 노출합니다.
built-in tool belt (bash, edit, read, grep, glob, list, lsp, task, todowrite,
내장 도구 벨트(bash, edit, read, grep, glob, list, lsp, task, todowrite,
question, skill, webfetch, websearch) in every system prompt.
질문, 기술, 웹 가져오기, 웹 검색)을 모든 시스템 프롬프트에 표시합니다.
For this
이를 위해
RAG-only workflow, most of those tools are never used, but their tool-schema
RAG 전용 워크플로, 이러한 도구의 대부분은 사용되지 않지만 도구 스키마
definitions still cost a large, fixed block of overhead per request —
정의에는 여전히 요청당 고정된 대규모 오버헤드가 발생합니다.
paid before the model even sees the user's question.
모델이 사용자의 질문을 보기도 전에 비용이 지불됩니다.

**What did *not* work:**
**잘 작동하지 *않은* 것:**
- `permission: {webfetch: "deny", websearch: "deny"}` — blocks execution
- `permission: {webfetch: "deny", websearch: "deny"}` — 실행 차단
correctly but does **not** shrink the prompt (schemas are still sent).
정확하지만 프롬프트를 축소하지 **않습니다**(스키마는 계속 전송됩니다).
- `agent.build.tools: {webfetch: false, websearch: false}` — same result;
- `agent.build.tools: {webfetch: false, websearch: false}` — 동일한 결과;
webfetch/websearch are a small fraction of the total tool-schema overhead.
webfetch/websearch는 전체 도구 스키마 오버헤드의 작은 부분입니다.

**What worked:** a dedicated, minimal agent with every non-essential tool
**효과적인 점:** 필수적이지 않은 모든 도구를 갖춘 최소한의 전용 에이전트
disabled:
비활성화됨:

```json
{
  "agent": {
    "docsearch": {
      "description": "Minimal agent for fast Vivado doc-search RAG lookups only. All non-essential tools disabled to reduce prompt overhead and latency.",
      "prompt": "You are a documentation search assistant for AMD Embedded (Vivado, Vitis, Power Design Manager, ChipScope, System Software, example designs, Wiki, and Answer Records). Immediately call vivado_doc_search with the user's question; do not ask clarifying questions first. Start every search with limit 5; use limit 10 only when the user explicitly requests a broad/deep answer, the request has three or more distinct subtopics, or the first search lacks enough evidence — prefer a second focused search over a large, loosely relevant result set. Scale the answer to the task, not the model: be concise (about 150 words) for a simple fact or how-to; give the complete supported explanation with headings, steps, tables, and short code or Tcl examples for multi-part, diagnostic, or explicitly detailed requests; never omit a necessary qualification, prerequisite, limitation, or troubleshooting step to meet a word target. Do not fabricate: ground every claim in the search results and cite the source URL with the claim, and do not add a command, constraint type, parameter, implementation pattern, or rationale unless a returned source explicitly supports it. If results are insufficient, ambiguous, or conflicting, run narrower follow-up searches before answering. If there are no relevant results, say so and suggest a rephrased query. Do not assume the active model's identity or capability. End every answer with a short Sources list of every URL used.",
      "tools": {
        "webfetch": false, "websearch": false, "bash": false, "edit": false,
        "read": false, "grep": false, "glob": false, "list": false,
        "lsp": false, "task": false, "todowrite": false, "question": false, "skill": false
      },
      "permission": {
        "webfetch": "deny", "websearch": "deny", "bash": "deny", "edit": "deny"
      }
    }
  }
}
```

Disabling individual tools only removes their own schema cost.
개별 도구를 비활성화하면 해당 도구의 스키마 비용만 제거됩니다.
The dominant fixed cost comes from the *entire* built-in tool set — you must
지배적인 고정 비용은 *전체* 내장 도구 세트에서 발생합니다.
build a dedicated minimal agent (not tweak permissions on the
전용 최소 에이전트를 구축합니다(권한을 조정하지 않음).
general-purpose agent) to see a meaningful reduction.
범용 에이전트) 의미 있는 감소를 확인할 수 있습니다.

Invoke it with:
다음을 사용하여 호출하세요.

```bash
opencode run --agent docsearch --model lemonade-qwen36/Qwen3.6-35B-A3B-UD-Q4_K_XL.gguf "<question>"
```

**B.4 — Verify**
**B.4 — 확인**

```text
Search Vivado docs for AXI SmartConnect.
```

Expected: OpenCode invokes the `vivado_doc_search` MCP tool and returns documentation results.
예상: OpenCode는 `vivado_doc_search` MCP 도구를 호출하고 문서 결과를 반환합니다.

---

## 5.
## Model Comparison
## 모델 비교

Both recommended models — Qwen3.6-35B-A3B and Gemma-4-26B-A4B MoE — handle
두 가지 권장 모델(Qwen3.6-35B-A3B 및 Gemma-4-26B-A4B MoE) 모두 핸들
the amd-embedded-doc-search workflow reliably.
amd-embedded-doc-search 작업 흐름을 안정적으로 수행합니다.
In testing across a range of
다양한 테스트를 통해
FPGA-engineering questions (design architecture, RTL lint, utilization
FPGA 엔지니어링 질문(설계 아키텍처, RTL 린트, 활용도)
analysis, timing closure, and hardware debug), both consistently called the
분석, 타이밍 클로저 및 하드웨어 디버그), 둘 다 일관되게
`vivado_doc_search` tool immediately, answered concisely, and cited real source
`vivado_doc_search` 도구가 즉시 간결하게 답변하고 실제 소스를 인용했습니다.
documents (User Guides, Product Guides, Answer Records) rather than
문서(사용자 가이드, 제품 가이드, 답변 기록)가 아닌
fabricating content.
콘텐츠를 제작하고 있습니다.

Choose based on your constraints:
제약 조건에 따라 선택하세요.

- **Qwen3.6-35B-A3B** — a very large context window at low KV-cache cost,
- **Qwen3.6-35B-A3B** — 낮은 KV 캐시 비용으로 매우 큰 컨텍스트 창,
useful if you expect long conversations or large retrieved contexts.
긴 대화나 검색된 대규모 컨텍스트가 필요한 경우 유용합니다.
- **Gemma-4-26B-A4B MoE** — a more compact footprint, useful when VRAM is
- **Gemma-4-26B-A4B MoE** — 더욱 컴팩트한 설치 공간으로 VRAM이
tighter or you want to run it alongside other models.
더 단단하거나 다른 모델과 함께 실행하고 싶습니다.

If you're unsure, either works well for documentation lookups — try both on
확실하지 않은 경우 문서 검색에 적합합니다. 둘 다 시도해 보세요.
your own hardware and pick the one that best fits your memory and latency
자신의 하드웨어를 선택하고 메모리와 대기 시간에 가장 적합한 하드웨어를 선택하세요.
budget.
예산.

---

## 6.
## VRAM Planning
## VRAM 계획

The main driver of GPU memory use is the **model weights**; the KV cache
GPU 메모리 사용의 주요 동인은 **모델 가중치**입니다. KV 캐시
(which grows with context length) is a comparatively small part of the
(컨텍스트 길이에 따라 증가)는 상대적으로 작은 부분입니다.
total.
총.
In practice that means:
실제로 이는 다음을 의미합니다.

- Context-size tuning saves relatively little memory — the bigger lever is
- 컨텍스트 크기 조정은 상대적으로 적은 양의 메모리를 절약합니다. 레버가 클수록
simply which model(s) you load.
어떤 모델을 로드하는지 간단히 알 수 있습니다.
- Qwen3.6 uses a hybrid architecture with a recurrent-state component, so
- Qwen3.6은 반복 상태 구성 요소가 있는 하이브리드 아키텍처를 사용하므로
its KV cache stays small even at long context lengths — you can keep a
KV 캐시는 긴 컨텍스트 길이에서도 작게 유지됩니다.
large context window without a large memory penalty.
큰 메모리 패널티 없이 큰 컨텍스트 창을 사용합니다.
- Gemma-4-26B-A4B has a more compact footprint overall.
- Gemma-4-26B-A4B는 전체적으로 더 작은 설치 공간을 갖습니다.

### 6.1 Context Sizes
### 6.1 컨텍스트 크기

| Model<br>모델 | Recommended ctx<br>추천 ctx | Rationale<br>이론적 근거 |
|---|---|---|
| Qwen3.6<br>Qwen3.6 | 131072 | KV cost stays low; no reason to shrink<br>KV 비용은 낮게 유지됩니다. 줄어들 이유가 없다 |
| Gemma 26B MoE<br>젬마 26B MoE | 32768 | Comfortably covers typical RAG workloads<br>일반적인 RAG 워크로드를 편안하게 처리합니다. |

### 6.2 VRAM Footprint
### 6.2 VRAM 공간

To help you size hardware, these are the approximate amounts of GPU memory
하드웨어 크기를 조정하는 데 도움이 되도록 대략적인 GPU 메모리 양은 다음과 같습니다.
each model used to load at the recommended context size on the
권장되는 컨텍스트 크기로 로드하는 데 사용되는 각 모델
[reference platform](#3-deployment).
[참조 플랫폼](#3-배포).
Actual usage varies with GPU, driver, quantization, and context length —
실제 사용량은 GPU, 드라이버, 양자화 및 컨텍스트 길이에 따라 다릅니다.
treat these as planning estimates, not exact figures:
이를 정확한 수치가 아닌 계획 추정치로 간주합니다.

| Loaded<br>로드됨 | Approx. VRAM<br>대략. VRAM |
|---|---|
| Qwen3.6-35B-A3B (ctx 131072)<br>Qwen3.6-35B-A3B (ctx 131072) | ~23 GB |
| Gemma-4-26B-A4B MoE (ctx 32768)<br>Gemma-4-26B-A4B MoE(ctx 32768) | ~17 GB |
| Both models loaded at once<br>두 모델을 동시에 로드 | ~40 GB |

Model weights account for most of this; the KV cache is a relatively small
모델 가중치가 이 중 대부분을 차지합니다. KV 캐시는 상대적으로 작습니다.
part, so shrinking the context window frees comparatively little memory.
부분이므로 컨텍스트 창을 축소하면 비교적 적은 메모리가 확보됩니다.

### 6.3 Running Models
### 6.3 모델 실행

Whether you can run a single model or both simultaneously depends on your
단일 모델을 실행할 수 있는지 아니면 둘 다 동시에 실행할 수 있는지 여부는 귀하의 상황에 따라 다릅니다.
GPU's available memory.
GPU의 사용 가능한 메모리.
Load one model first and check usage with
먼저 하나의 모델을 로드하고 다음을 사용하여 사용법을 확인하세요.
`rocm-smi --showmeminfo vram` before adding a second.
두 번째를 추가하기 전에 `rocm-smi --showmeminfo vram`.
On unified-memory
통합 메모리에서
APUs, the VRAM allocation is carved out of system RAM and is often
APU의 경우 VRAM 할당은 시스템 RAM에서 분리되며 종종
adjustable in the BIOS if you need more headroom.
더 많은 여유 공간이 필요한 경우 BIOS에서 조정할 수 있습니다.

---

## 7. systemd Persistence
## 7. 체계화된 지속성

Two unit files enable persistent launch of the models across reboots.
두 개의 유닛 파일을 사용하면 재부팅 후에도 모델을 지속적으로 시작할 수 있습니다.

**`llama-qwen.service`:**
**`llama-qwen.service`:**

```ini
[Unit]
Description=Llama.cpp Server - Qwen3.6-35B-A3B (tuned: q8_0 KV cache + flash-attn)
After=local-fs.target

[Service]
Type=simple
User=local-llm
# Reference Strix Halo only; omit or replace for another supported GPU.
# Environment="HSA_OVERRIDE_GFX_VERSION=11.5.1"
Environment="LD_LIBRARY_PATH=/opt/local-llm/runtime:/opt/local-llm/runtime/hipblaslt:/opt/local-llm/runtime/rocblas"
ExecStart=/opt/local-llm/runtime/llama-server \
  --model /var/lib/local-llm/models/Qwen3.6-35B-A3B-GGUF/Qwen3.6-35B-A3B-UD-Q4_K_XL.gguf \
  --mmproj /var/lib/local-llm/models/Qwen3.6-35B-A3B-GGUF/mmproj-F16.gguf \
  --host 127.0.0.1 --port 8000 --ctx-size 131072 --n-gpu-layers 99 --reasoning off \
    --batch-size 512 --ubatch-size 256 --cache-type-k q8_0 --cache-type-v q8_0 --flash-attn on
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

**`llama-gemma.service`:**
**`llama-gemma.service`:**

```ini
[Unit]
Description=Llama.cpp Server - Gemma-4-26B-A4B MoE (tuned)
After=local-fs.target

[Service]
Type=simple
User=local-llm
# Reference Strix Halo only; omit or replace for another supported GPU.
# Environment="HSA_OVERRIDE_GFX_VERSION=11.5.1"
Environment="LD_LIBRARY_PATH=/opt/local-llm/runtime:/opt/local-llm/runtime/hipblaslt:/opt/local-llm/runtime/rocblas"
ExecStart=/opt/local-llm/runtime/llama-server \
  --model /var/lib/local-llm/models/gemma-4-26B-A4B-it-GGUF/gemma-4-26B-A4B-it-UD-Q4_K_M.gguf \
  --mmproj /var/lib/local-llm/models/gemma-4-26B-A4B-it-GGUF/mmproj-F16.gguf \
  --host 127.0.0.1 --port 8001 --ctx-size 32768 --n-gpu-layers 99 --parallel 1 --reasoning off \
  --batch-size 512 --ubatch-size 256 --cache-type-k q8_0 --cache-type-v q8_0 --flash-attn on
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Before enabling either unit, create a dedicated service account, make it the
두 유닛 중 하나를 활성화하기 전에 전용 서비스 계정을 생성하고 이를
model-data owner, and confirm it has the ROCm device permissions required by
모델 데이터 소유자에게 필요한 ROCm 장치 권한이 있는지 확인합니다.
your distribution:
귀하의 배포판:

```bash
sudo useradd --system --home-dir /var/lib/local-llm --shell /usr/sbin/nologin local-llm
sudo chown -R local-llm:local-llm /var/lib/local-llm/models
```

Install with:
다음을 사용하여 설치:

```bash
# Save the two complete units above as these filenames first.
sudo cp llama-qwen.service llama-gemma.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now llama-qwen.service llama-gemma.service
```

---

## 8.
## Validation
## 검증

**Local LLM:**
**로컬 LLM:**

```bash
curl http://127.0.0.1:8000/v1/models   # or :13305/api/v1/models for a single-model Lemonade default
```
- ✅ Returns your downloaded models.
- ✅ 다운로드한 모델을 반환합니다.

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

**Air-gap confirmation:**
**공극 확인:**

- ✅ An enforced egress policy or no default route prevents outbound traffic;
- ✅ 강제 송신 정책이 있거나 기본 경로가 없으면 아웃바운드 트래픽이 차단됩니다.
a failed `ping` alone is not proof of an air gap because ICMP may simply be
실패한 '핑'만으로는 에어 갭을 증명할 수 없습니다. 왜냐하면 ICMP는 단순히
blocked.
차단되었습니다.
- ✅ For Lemonade server mode, `offline=true` and
- ✅ 레모네이드 서버 모드의 경우 `offline=true` 및
`no_fetch_executables=true` are set before loading any model.
모델을 로드하기 전에 'no_fetch_executables=true'가 설정됩니다.
- ✅ `ss -ltnp` shows the local LLM listening only on `127.0.0.1` unless a
- ✅ `ss -ltnp`는 로컬 LLM이 `127.0.0.1`만 수신하는 것을 보여줍니다.
secured remote-access policy explicitly requires another bind address.
보안 원격 액세스 정책에는 명시적으로 다른 바인드 주소가 필요합니다.
- ✅ `curl http://127.0.0.1:8000/v1/models` succeeds (local LLM works).
- ✅ `curl http://127.0.0.1:8000/v1/models` 성공(로컬 LLM 작동).
- ✅ Documentation search works in your chosen client.
- ✅ 문서 검색은 선택한 클라이언트에서 작동합니다.

The environment is now fully functional without Internet connectivity.
이제 인터넷 연결 없이도 환경이 완벽하게 작동합니다.

---

## 9.
## Troubleshooting
## 문제 해결

| Symptom<br>증상 | Likely cause / fix<br>가능한 원인/수정 사항 |
|---------|--------------------|
| Model not listed by `curl .../v1/models`<br>`curl .../v1/models`에 나열되지 않은 모델 | Confirm the server is running and that the URL and configured port match. A loopback (`127.0.0.1`/`localhost`) bind is expected for a same-machine client. Use `0.0.0.0` only for a deliberately secured remote-access deployment.<br>서버가 실행 중이고 URL과 구성된 포트가 일치하는지 확인하세요. 동일한 머신 클라이언트에 대해 루프백(`127.0.0.1`/`localhost`) 바인드가 예상됩니다. 의도적으로 보안이 설정된 원격 액세스 배포에만 '0.0.0.0'을 사용하세요. |
| Model files not detected<br>모델 파일이 감지되지 않음 | Confirm `extra_models_dir` points to the folder holding the `.gguf` files: `lemonade config set extra_models_dir=/path/to/models`, then restart.<br>`extra_models_dir`이 `.gguf` 파일이 들어 있는 폴더(`lemonade config set extra_models_dir=/path/to/models`)를 가리키는지 확인한 다음 다시 시작하세요. |
| Model missing from VS Code picker<br>VS Code 선택기에서 모델이 누락되었습니다. | `toolCalling` not set to `true`, wrong `id`, or VS Code needs a restart.<br>`toolCalling`이 `true`로 설정되지 않았거나 `id`가 잘못되었거나 VS Code를 다시 시작해야 합니다. |
| VS Code warns about utility models<br>VS Code는 유틸리티 모델에 대해 경고합니다. | Set `chat.utilityModel` and `chat.utilitySmallModel` to your local model (§4.1, A.3).<br>`chat.utilityModel` 및 `chat.utilitySmallModel`을 로컬 모델로 설정합니다(§4.1, A.3). |
| MCP tool not offered<br>MCP 도구는 제공되지 않습니다 | Chat must be in **Agent** mode; confirm `.vscode/mcp.json` is valid and reload VS Code.<br>채팅은 **에이전트** 모드여야 합니다. `.vscode/mcp.json`이 유효한지 확인하고 VS Code를 다시 로드하세요. |
| MCP server not reachable<br>MCP 서버에 연결할 수 없습니다 | Run `amd-embedded-doc-search ps` and `amd-embedded-doc-search doctor` (see [Operate & Troubleshoot](operate.md)); confirm the client URL matches `http://127.0.0.1:<DOCSEARCH_MCP_PORT>/mcp/doc-search`.<br>`amd-embedded-doc-search ps` 및 `amd-embedded-doc-search doctor`를 실행합니다([운영 및 문제 해결](operate.md) 참조). 클라이언트 URL이 `http://127.0.0.1:<DOCSEARCH_MCP_PORT>/mcp/doc-search`와 일치하는지 확인하세요. |
| Search fails with `vector lengths don't match` (e.g. `768 vs 1024`)<br>'벡터 길이가 일치하지 않습니다'(예: '768 대 1024')로 인해 검색이 실패합니다. | You switched embedding-model packages using `amd-embedded-doc-search stop` instead of `amd-embedded-doc-search purge`, so the old package's vectors are still in the data volume. Run `amd-embedded-doc-search purge --yes` then `amd-embedded-doc-search configure --start` to reimport cleanly with the new model. See [RAG Database Setup §1.5](README.md#15-embedding-packages).<br>`amd-embedded-doc-search purge` 대신 `amd-embedded-doc-search stop`을 사용하여 임베딩 모델 패키지를 전환했으므로 이전 패키지의 벡터가 여전히 데이터 볼륨에 있습니다. `amd-embedded-doc-search purge --yes`를 실행한 다음 `amd-embedded-doc-search 구성 --start`를 실행하여 새 모델로 깔끔하게 다시 가져옵니다. [RAG 데이터베이스 설정 §1.5](README.md#15-embedding-packages)를 참조하세요. |
| MCP client gets `Connection reset by peer` right after a fresh import finishes<br>MCP 클라이언트는 새로운 가져오기가 완료된 직후 '피어에 의한 연결 재설정'을 받습니다. | Expected and harmless: the `amd-embedded-doc-search-mcp-1` container restarts itself once the importer finishes populating Weaviate, and needs ~15-30 seconds to come back up. Wait briefly (`docker ps --filter name=amd-embedded-doc-search-mcp-1` until `STATUS` shows it's been up for at least 15-30s) and retry the request.<br>예상 가능하고 무해함: 가져오기 도구가 Weaviate 채우기를 완료하면 'amd-embedded-doc-search-mcp-1' 컨테이너가 자동으로 다시 시작되며, 다시 시작하는 데 ~15~30초가 필요합니다. 잠시 기다렸다가(`STATUS`에 최소 15~30초 동안 작동 중이라고 표시될 때까지 `docker ps --filter name=amd-embedded-doc-search-mcp-1`) 요청을 다시 시도하세요. |
| OpenCode ignores the model<br>OpenCode는 모델을 무시합니다. | Model key must match the exact id from the server's `/v1/models`; restart OpenCode after editing config.<br>모델 키는 서버 `/v1/models`의 정확한 ID와 일치해야 합니다. 구성을 편집한 후 OpenCode를 다시 시작하세요. |
| Model gives fewer retries than expected on a RAG query<br>모델은 RAG 쿼리에서 예상보다 적은 수의 재시도를 제공합니다. | Check the retrieved chunks — RAG corpus contamination can cause a model to give up early on a poor-quality search result; this is a data issue, not necessarily a model issue.<br>검색된 청크 확인 — RAG 코퍼스 오염으로 인해 모델이 품질이 낮은 검색 결과로 인해 조기에 포기될 수 있습니다. 이는 데이터 문제이지 반드시 모델 문제는 아닙니다. |
| VRAM exhausted / OOM when loading multiple models<br>여러 모델을 로드할 때 VRAM이 소진됨/OOM | Check `rocm-smi --showmeminfo vram`; see §6 for VRAM planning guidance and the BIOS VRAM allocation option.<br>`rocm-smi --showmeminfo vram`을 확인하세요. VRAM 계획 지침 및 BIOS VRAM 할당 옵션은 §6을 참조하세요. |

## Next Steps
## 다음 단계

- Need the RAG database steps again? →
- RAG 데이터베이스 단계가 다시 필요합니까? →
[RAG Database Setup](README.md)
[RAG 데이터베이스 설정](README.md)
- Don't have GPU-class hardware after all? →
- 결국 GPU급 하드웨어가 없나요? →
[Frontier Model](frontier-model.md)
[프론티어 모델](frontier-model.md)

---

<p class="sphinxhide" align="center"><sub>Copyright © 2026 Advanced Micro Devices, Inc</sub></p>
<p class="sphinxhide" align="center"><sup><a href="https://www.amd.com/en/corporate/copyright">Terms and Conditions</a></sup></p>
