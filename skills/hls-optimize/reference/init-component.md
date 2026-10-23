<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# init-component
# 초기화 구성 요소

Initialize the HLS component (ensure `vitis-comp.json` exists) and the git repository (if needed), then commit source/config files as baseline.
HLS 구성 요소(`vitis-comp.json`이 있는지 확인)와 git 저장소(필요한 경우)를 초기화한 다음 소스/구성 파일을 기준으로 커밋합니다.

## Check the vitis-comp.json file exists
## vitis-comp.json 파일이 있는지 확인하세요.

Check the `vitis-comp.json` file in the component directory.
컴포넌트 디렉토리에서 `vitis-comp.json` 파일을 확인하세요.
If it does not exist, you must create one before running HLS flow.
존재하지 않는 경우 HLS 흐름을 실행하기 전에 생성해야 합니다.
You must use the following format to create the component folder and the `vitis-comp.json` file:
구성 요소 폴더와 `vitis-comp.json` 파일을 생성하려면 다음 형식을 사용해야 합니다.

```
## folder structure:
hls_component/
├── vitis-comp.json

{
  "name": "hls_component",
  "type": "HLS",
  "configuration": {
    "componentType": "HLS",
    "configFiles": [
      "<hls_config.cfg relative path>"
    ]
  }
}
```

## Init-git: How to run
## Init-git: 실행 방법

Run the following commands in the root directory of your HLS project, don`t change the command, just copy and paste it to terminal and execute.
HLS 프로젝트의 루트 디렉터리에서 다음 명령을 실행하세요. 명령을 변경하지 말고 복사하여 터미널에 붙여넣고 실행하세요.
```bash
# Step 1: Initialize git if needed
if [ ! -d ".git" ]; then
    git init 2>/dev/null
    git config user.email "your-email@example.com"
    git config user.name "your-name"
    echo "Git repository initialized."
fi

# Step 2: Create or update .gitignore
HLS_PATTERNS="_ide/
**/.cache/
**/hls/
**/reports/
**/logs/
*summary"

if [ ! -f ".gitignore" ]; then
    echo "$HLS_PATTERNS" > .gitignore
    echo ".gitignore created."
else
    # Append missing patterns
    while IFS= read -r pattern; do
        if ! grep -qxF "$pattern" .gitignore; then
            echo "$pattern" >> .gitignore
        fi
    done <<< "$HLS_PATTERNS"
    echo ".gitignore updated with HLS patterns."
fi

# Step 3: Add files and commit if no commits yet
if ! git rev-parse HEAD >/dev/null 2>&1; then
    # Find and add C/C++ source files (excluding hls directories)
    find . -type d -name "hls" -prune -o \( -name "*.cpp" -o -name "*.hpp" -o -name "*.h" -o -name "*.c" \) -print | xargs -r git add
    # Find and add config files (excluding hls directories)
    find . -type d -name "hls" -prune -o \( -name "*.cfg" -o -name "*.tcl" \) -print | xargs -r git add
    # Add .gitignore
    git add .gitignore 2>/dev/null
    git commit -m "Initial commit: baseline HLS kernel"
    echo "Initial commit created."
else
    echo "Git repository already has commits."
    git log --oneline -3
fi
```

## Verify
## 확인

```bash
git status && git log --oneline -3
```
