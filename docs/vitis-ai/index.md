<!-- ko-translation: enabled -->
# Vitis AI AI-Assisted Workflows Reference
# Vitis AI AI 지원 워크플로 참조

This section describes each AI-assisted workflow that ships with Vitis AI.
이 섹션에서는 Vitis AI와 함께 제공되는 각 AI 지원 워크플로우에 대해 설명합니다.

The workflows are optional.
워크플로는 선택 사항입니다.
Each task is fully documented for manual execution,
각 작업은 수동 실행을 위해 완전히 문서화되어 있습니다.
and the manual procedure is linked from every workflow page.
수동 절차는 모든 작업 흐름 페이지에서 연결됩니다.
If you do not use
사용하지 않는 경우
an AI coding assistant, follow those links instead.
AI 코딩 도우미인 경우 대신 해당 링크를 따르세요.
To set the workflows up, see
워크플로를 설정하려면 다음을 참조하세요.
[the skill installation guide](../../README.md#quick-start).
[스킬 설치 가이드](../../README.md#quick-start).

## Choosing a workflow
## 워크플로우 선택

| If you want to<br>당신이 원한다면 | Workflow<br>작업 흐름 | Manual procedure<br>수동 절차 |
| --- | --- | --- |
| Implement a custom operator that can extend what the NPU can run and/or optimize how it runs.<br>NPU가 실행할 수 있는 범위를 확장하거나 실행 방법을 최적화할 수 있는 사용자 지정 연산자를 구현합니다. | [Custom operator workflow](vai-custom-op.md)<br>[커스텀 오퍼레이터 워크플로](vai-custom-op.md) | [Custom operators](https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/additional_information/custom_ops/custom_ops_introduction.html)<br>[맞춤형 연산자](https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/additional_information/custom_ops/custom_ops_introduction.html) |
| Quantize a model, including per-layer mixed precision<br>레이어별 혼합 정밀도를 포함한 모델 양자화 | [Model quantization workflow](vai-quantization-guide.md)<br>[모델 양자화 워크플로우](vai-Quantization-guide.md) | [Model Quantization](https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/model_quantization/model_quantization.html) , [Mixed Precision Compilation](https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/model_compilation/mixed_precision_compilation.html)<br>[모델 양자화](https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/model_Quantization/model_Quantization.html) , [혼합 정밀도 컴파일](https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/model_compilation/mixed_precision_compilation.html) |
| Increase NPU offload, reduce inference time, or get a model to compile<br>NPU 오프로드를 늘리고, 추론 시간을 줄이거나, 컴파일할 모델을 얻습니다. | [Compiler options workflow](vai-flag-configuration.md)<br>[컴파일러 옵션 작업 흐름](vai-flag-configuration.md) | [Vitis AI EP Options](https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/model_compilation/ep-options.html)<br>[Vitis AI EP 옵션](https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/model_compilation/ep-options.html) |

### Board access
### 보드 액세스

The workflows steer themselves using measurements taken on hardware.
워크플로는 하드웨어에서 수행된 측정을 사용하여 스스로 조정됩니다.
Without
없이
board access they fall back on simulation and host-side compile results, which
보드 액세스는 시뮬레이션 및 호스트 측 컴파일 결과에 의존합니다.
are not a one-to-one match for the hardware.
하드웨어와 일대일로 일치하지 않습니다.
They still run, but expect numeric
여전히 실행 중이지만 숫자가 예상됩니다.
mismatches, untuned performance, and weaker convergence.
불일치, 조정되지 않은 성능 및 약한 수렴.
Each workflow page
각 워크플로 페이지
describes what it can still do without a board.
보드 없이도 무엇을 할 수 있는지 설명합니다.

The workflows reach the board over SSH, using either a password or a key.
워크플로는 비밀번호나 키를 사용하여 SSH를 통해 보드에 도달합니다.
Set
세트
the following before launching your assistant:
어시스턴트를 시작하기 전에 다음을 수행하십시오.

```shell
export BOARDHOST=172.16.0.1        # your board's IP address
export BOARD_USER=amd-edf

# Authenticate with a password
export BOARD_PASSWORD=<password>

# Or with an SSH key, instead of a password
# export BOARD_KEY=/mnt/user/ssh_key_file
```

> **Caution**
> **주의**
>
> `BOARD_PASSWORD` is a credential.
> `BOARD_PASSWORD`는 자격 증명입니다.
> Keep it out of scripts and version
> 스크립트와 버전에 포함되지 않도록 하세요.
> control, and prefer a permission-restricted file over a plain shell export.
> 일반 쉘 내보내기보다 권한이 제한된 파일을 제어하고 선호합니다.
> See the guidance in [the Vitis AI documentation](https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/ai_workflows/llm-setup.html#configure-llm-access).
> [Vitis AI 문서](https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/ai_workflows/llm-setup.html#configure-llm-access)의 지침을 참조하세요.

To persist these across container restarts, append them to `$HOME/.bashrc`.
컨테이너를 다시 시작해도 이를 유지하려면 `$HOME/.bashrc`에 추가하세요.
If you followed the Vitis AI documentation example
Vitis AI 문서 예시를 따랐다면
[Claude Code in the Vitis AI Docker container](https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/ai_workflows/llm-setup.html#claude-code-docker),
[Vitis AI Docker 컨테이너의 Claude 코드](https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/ai_workflows/llm-setup.html#claude-code-docker),
this file is mapped to `$WORKDIR/homedir/.bashrc` on the host.
이 파일은 호스트의 `$WORKDIR/homedir/.bashrc`에 매핑됩니다.

If you prefer not to set them, supply the board details in the prompt instead:
설정하지 않으려면 대신 프롬프트에 보드 세부 정보를 제공하세요.

```text
Run the model ./model.onnx on my board using configuration
./vitisai_config.json. The board's IP address is 172.16.0.1, the user is
amd-edf, the password is mypassword.
```

Boot the board and confirm it is reachable over SSH before you start a session.
세션을 시작하기 전에 보드를 부팅하고 SSH를 통해 연결할 수 있는지 확인하세요.
See the OSPI and SD card boot flow documentation
OSPI 및 SD 카드 부팅 흐름 문서를 참조하세요.
[OSPI and SD Card Boot Flow](https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/setup_and_installation/board_setup_ospi_sd_eou.html)
[OSPI 및 SD 카드 부팅 흐름](https://vitisai.docs.amd.com/projects/gen2/en/latest/docs/setup_and_installation/board_setup_ospi_sd_eou.html)
for board preparation and bootup.
보드 준비 및 부팅을 위해.

- [Custom operator workflow](vai-custom-op.md)
- [커스텀 오퍼레이터 워크플로](vai-custom-op.md)
- [Model quantization workflow](vai-quantization-guide.md)
- [모델 양자화 워크플로우](vai-Quantization-guide.md)
- [Compiler options workflow](vai-flag-configuration.md)
- [컴파일러 옵션 작업 흐름](vai-flag-configuration.md)
