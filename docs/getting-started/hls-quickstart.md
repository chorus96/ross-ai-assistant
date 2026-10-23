<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Vitis HLS AI Assistant
# Vitis HLS AI 어시스턴트

Vitis HLS skills work directly with your source code — no MCP server or live Vivado session needed.
Vitis HLS 기술은 소스 코드와 직접 작동합니다. MCP 서버나 라이브 Vivado 세션이 필요하지 않습니다.
The agent reads your code and HLS project files to provide architecture guidance, code generation, and optimization recommendations.
에이전트는 코드와 HLS 프로젝트 파일을 읽어 아키텍처 지침, 코드 생성 및 최적화 권장 사항을 제공합니다.

## Prerequisites
## 전제조건

- **Vitis HLS** installed and on your `PATH` (for running the HLS compilation flow).
- **Vitis HLS**가 `PATH`에 설치되어 있습니다(HLS 컴파일 흐름 실행용).
- 2026.1 is the tested baseline; the HLS skills and examples also run on 2025.1+ — see each example's README for its minimum.
- 2026.1은 테스트된 기준선입니다. HLS 기술과 예제는 2025.1+에서도 실행됩니다. 최소값은 각 예제의 README를 참조하세요.
- HLS agent skills installed in your workspace — see [Install Agent Skills](../../README.md#quick-start).
- 작업 영역에 설치된 HLS 에이전트 기술 - [에이전트 기술 설치](../../README.md#quick-start)를 참조하세요.
- Use `--skill hls-optimize` for a single skill, or `--all` for everything.
- 단일 기술에는 `--skill hls-optimize`를 사용하고, 모든 기술에는 `--all`을 사용하세요.

For skill descriptions, example workflows, and the full skills reference, see the **Vitis HLS AI Assistant — Skills Reference** guide.
기술 설명, 예시 워크플로 및 전체 기술 참조는 **Vitis HLS AI Assistant - 기술 참조** 가이드를 참조하세요.

---
