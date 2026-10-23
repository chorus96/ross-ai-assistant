<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# Support
# 지원

## Getting Help
## 도움 받기

Use the official AMD Adaptive Support community.
공식 AMD 적응형 지원 커뮤니티를 이용하세요.
Post questions, issues, and workflows there so other users and AMD can help.
다른 사용자와 AMD가 도움을 줄 수 있도록 질문, 문제 및 워크플로를 게시하세요.

**[AMD Adaptive Support](https://adaptivesupport.amd.com/s/?language=en_US)**
**[AMD 적응형 지원](https://adaptivesupport.amd.com/s/?언어=en_US)**

- **Post questions and issues** — search existing articles and discussions first, then open a new thread if needed
- **질문 및 문제 게시** - 기존 기사와 토론을 먼저 검색한 다음 필요한 경우 새 스레드를 엽니다.
- **Share tips and workflows** — useful prompts and setups help other users
- **팁 및 작업 흐름 공유** - 유용한 메시지와 설정이 다른 사용자에게 도움이 됩니다.
- **Stay informed** — product updates and known issues are published on Adaptive Support
- **최신 정보를 받아보세요** — 제품 업데이트 및 알려진 문제는 Adaptive Support에 게시됩니다.

### Reporting Issues
### 보고 문제

When reporting an issue on the forum, please include:
포럼에 문제를 보고할 때 다음 사항을 포함해 주세요.

1. **Tool and version** — IDE/CLI tool name and version, MCP server version, extension version
1. **도구 및 버전** — IDE/CLI 도구 이름 및 버전, MCP 서버 버전, 확장 버전
2. **Vivado/HLS version** — Output of `version` in the Vivado Tcl console, or `vitis-run --version`
2. **Vivado/HLS 버전** — Vivado Tcl 콘솔의 `version` 출력 또는 `vitis-run --version`
3. **Steps to reproduce** — What you asked the agent and what happened
3. **재현 단계** — 상담원에게 문의한 내용과 발생한 일
4. **Logs** — Relevant logs from:
4. **로그** — 관련 로그:
    - Vivado Tcl console output or HLS build log
    - Vivado Tcl 콘솔 출력 또는 HLS 빌드 로그
    - MCP server output (check your IDE's Output panel > Vivado MCP)
    - MCP 서버 출력(IDE의 출력 패널 > Vivado MCP 확인)
    - Agent conversation transcript
    - 상담원 대화 내용
5. **Design context** — Target device, design size, which skill was used
5. **디자인 컨텍스트** — 대상 장치, 디자인 크기, 사용된 스킬

## Known Limitations
## 알려진 제한 사항

| Limitation<br>한계 | Workaround<br>해결 방법 |
|------------|------------|
| LSF tool (`vivado_lsf`) only works inside the AMD network<br>LSF 도구(`vivado_lsf`)는 AMD 네트워크 내부에서만 작동합니다. | Use `vivado_ssh` for remote deployment outside AMD. LSF support for external customers is planned for a future release<br>AMD 외부의 원격 배포에는 'vivado_ssh'를 사용하세요. 외부 고객을 위한 LSF 지원은 향후 릴리스에서 계획되어 있습니다. |
| GUI mode may open on a virtual/VNC display instead of your native display<br>기본 디스플레이 대신 가상/VNC 디스플레이에서 GUI 모드가 열릴 수 있습니다. | Tell the agent your display value, e.g. *"use display :0"* or *"use my $DISPLAY"*. Run `echo $DISPLAY` in your terminal to find it<br>상담원에게 표시 값을 알려주세요. *"디스플레이 사용:0"* 또는 *"내 $DISPLAY 사용"*. 찾으려면 터미널에서 `echo $DISPLAY`를 실행하세요. |
| Long-running Vivado commands may cause the chat to stop waiting for a response<br>오랫동안 실행되는 Vivado 명령으로 인해 채팅이 응답 대기를 중지할 수 있습니다. | Ask the agent to *"check status"* or *"continue"*. This is an AI infrastructure limitation, not specific to the MCP server<br>상담원에게 *"상태 확인"* 또는 *"계속"*을 요청하세요. 이는 MCP 서버에만 국한된 것이 아닌 AI 인프라 제한 사항입니다. |
| The agent may interpret "visualize the design" as taking GUI screenshots via `vivado_computer_use`<br>에이전트는 'vivado_computer_use'를 통해 GUI 스크린샷을 찍는 것으로 '디자인 시각화'를 해석할 수 있습니다. | Disable the `vivado_computer_use` tool if you do not need automated GUI interaction<br>자동화된 GUI 상호작용이 필요하지 않은 경우 'vivado_computer_use' 도구를 비활성화하세요. |
| Some LLMs may not automatically call `vivado_doc_search` for knowledge queries<br>일부 LLM은 지식 쿼리에 대해 'vivado_doc_search'를 자동으로 호출하지 않을 수 있습니다. | Add a system prompt instructing the agent to *"always use vivado_doc_search for AMD/Xilinx documentation questions"*, or explicitly state in your query: *"use vivado_doc_search to look up..."*. More capable models generally handle tool selection reliably; smaller or older models may need this guidance<br>*"AMD/Xilinx 문서 관련 질문에는 항상 vivado_doc_search를 사용하세요"*라고 에이전트에 지시하는 시스템 프롬프트를 추가하거나 쿼리에 명시적으로 명시하세요. *"조회하려면 vivado_doc_search를 사용하세요..."*. 보다 유능한 모델은 일반적으로 도구 선택을 안정적으로 처리합니다. 더 작거나 오래된 모델에는 이 지침이 필요할 수 있습니다. |

## Providing Feedback
## 피드백 제공

Your feedback directly shapes the product.
귀하의 피드백은 제품을 직접적으로 형성합니다.
We're especially interested in:
우리는 특히 다음에 관심이 있습니다:

- **New skill ideas** — What workflows would you like automated?
- **새로운 기술 아이디어** — 어떤 워크플로를 자동화하고 싶습니까?
- **Accuracy** — Did the agent give correct recommendations?
- **정확성** — 상담원이 올바른 권장사항을 제공했습니까?
- **Usability** — Was the setup process smooth?
- **사용성** — 설정 과정이 원활했나요?
- What was confusing?
- 무엇이 혼란스러웠나요?
- **Missing features** — What capabilities are you missing?
- **누락된 기능** — 어떤 기능이 누락되어 있나요?

You can also submit feedback directly from the chat terminal using the **`vivado_feedback`** MCP tool.
**`vivado_feedback`** MCP 도구를 사용하여 채팅 터미널에서 직접 피드백을 제출할 수도 있습니다.
Simply type in the chat:
채팅에 간단히 입력하세요:

> *#vivado_feedback the timing closure workflow worked great, rated good*
> *#vivado_feedback 타이밍 클로저 워크플로가 훌륭하게 작동했으며 좋은 평가를 받았습니다*

The agent will call the `vivado_feedback` tool with your rating (`good`, `neutral`, or `poor`) and comments, which are recorded and sent to the AMD engineering team.
에이전트는 귀하의 평가('좋음', '중립' 또는 '나쁨')와 의견을 포함하여 'vivado_feedback' 도구를 호출합니다. 이 내용은 기록되어 AMD 엔지니어링 팀에 전송됩니다.
No forms to fill out — just tell the agent what you thought.
작성할 양식이 없습니다. 상담원에게 귀하의 생각을 알려주십시오.

Post feedback on [AMD Adaptive Support](https://adaptivesupport.amd.com/s/?language=en_US).
[AMD 적응형 지원](https://adaptivesupport.amd.com/s/?언어=en_US)에 피드백을 게시하세요.

---
