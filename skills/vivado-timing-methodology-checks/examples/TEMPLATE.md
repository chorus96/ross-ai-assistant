<!-- ko-translation: enabled -->
<!--
Copyright (C) 2026, Advanced Micro Devices, Inc.
저작권 (C) 2026, Advanced Micro Devices, Inc.
All rights reserved.
모든 권리 보유.
SPDX-License-Identifier: MIT
SPDX-라이센스-식별자: MIT
-->

# [CHECK-ID]: [Brief title]
# [CHECK-ID]: [간단한 제목]

## Metadata
## 메타데이터

- **Check ID**: [e.g., TIMING-1]
- **ID 확인**: [예: TIMING-1]
- **Severity**: [Critical Warning | Warning | Advisory]
- **심각도**: [중요한 경고 | 경고 | 주의보]
- **Hierarchy Name**: [e.g., Timing.Bad Practice]
- **계층 구조 이름**: [예: Timing.Bad Practice]
- **First Release**: [Vivado version, e.g., 2013.3]
- **첫 번째 릴리스**: [Vivado 버전, 예: 2013.3]

## Description
## 설명

[Brief description of the methodology check]
[방법론 점검에 대한 간략한 설명]

### Full Message Description property.
### 전체 메시지 설명 속성입니다.

```
[Full message template with <PLACEHOLDERS>]
```
**Example:**
**예:**
```
[Real violation message from actual design]
```
[Explanation of placeholders if needed]
[필요한 경우 자리표시자 설명]

### Explanation
### 설명
[Detailed explanation of why this violation occurs and what it indicates]
[이 위반이 발생하는 이유와 그것이 의미하는 바에 대한 자세한 설명]

**DO**
✅ [Required action or best practice]
✅ [필수 조치 또는 모범 사례]
✅ [Another required action]
✅ [또 다른 필수 조치]
✅ [Another required action]
✅ [또 다른 필수 조치]

**DO NOT**
**하지 마세요**
❌ [Common mistake to avoid]
❌ [피해야 할 일반적인 실수]
❌ [Another mistake to avoid]
❌ [피해야 할 또 다른 실수]


## Flow
## 흐름
1. [First step with clear action]
1. [명확한 액션의 첫걸음]
2. [Second step - include TCL commands if relevant]
2. [두 번째 단계 - 해당되는 경우 TCL 명령 포함]
```tcl
[Example TCL command]
```

**DO NOT**
**하지 마세요**
❌ [Specific caution for this step]
❌ [이 단계에서 특별한 주의사항]
❌ [Another caution]
❌ [또 하나의 주의사항]

3. [Next step]
3. [다음 단계]

**DO**
✅ [Best practice for this step]
✅ [이 단계의 모범 사례]
✅ [Reference to helper documents if applicable]
✅ [해당되는 경우 도우미 문서 참고]

**DO NOT**
**하지 마세요**
❌ [Caution]
❌ [주의사항]
❌ [Caution]
❌ [주의사항]

3. Determine the best solution to offer the user.
3. 사용자에게 제공할 최상의 솔루션을 결정합니다.

**CASE 1:**
**사례 1:**
* **Type**: [Automated Resolution | User Resolution]
* **유형**: [자동 해결 | 사용자 해상도]
* **Confidence**: [High | Medium | Low]
* **신뢰도**: [높음 | 중간 | 낮음]
* **Usefulness**: [High | Medium | Low]
* **유용성**: [높음 | 중간 | 낮음]
* **When**: [Conditions when this case applies]
* **시기**: [이 사례가 적용되는 조건]
1. [Resolution step 1]
1. [해결 1단계]
2. [Resolution step 2]
2. [해결 2단계]

**DO**
✅ [When to apply this case]
✅ [이 사건을 적용하는 경우]

**DO NOT**
**하지 마세요**
❌ [When NOT to apply this case]
❌ [이 사건을 적용하지 말아야 할 경우]

**CASE 2:**
**사례 2:**
* **Type**: [Automated Resolution | User Resolution]
* **유형**: [자동 해결 | 사용자 해상도]
* **Confidence**: [High | Medium | Low]
* **신뢰도**: [높음 | 중간 | 낮음]
* **Usefulness**: [High | Medium | Low]
* **유용성**: [높음 | 중간 | 낮음]
* **When**: [Conditions when this case applies]
* **시기**: [이 사례가 적용되는 조건]
1. [Resolution step 1]
1. [해결 1단계]
2. [Resolution step 2]
2. [해결 2단계]

**CASE 3:**
**사례 3:**
* **Type**: User Resolution
* **유형**: 사용자 해상도
* **Confidence**: [High | Medium | Low]
* **신뢰도**: [높음 | 중간 | 낮음]
* **Usefulness**: [Low | Medium | High]
* **유용성**: [낮음 | 중간 | 높음]
* **Prompt**:
* **프롬프트**:
```
[Prompt text to present to user with options a), b), c), etc.]
```
* **Explanation**: [Provide explanation format]
* **설명**: [설명 형식 제공]
* **Examples**:
* **예**:
```
[Example waiver or explanation format]
```

## Verification
## 검증
* [ ] Methodology violation is no longer present
* [ ] 방법론 위반이 더 이상 존재하지 않습니다.
* [ ] No new unresolved methodology warnings are created
* [ ] 해결되지 않은 새로운 방법론 경고가 생성되지 않습니다.
* [ ] [Check-specific verification item]
* [ ] [체크별 검증항목]


## Final Report
## 최종 보고서
[Description of what to include in the final report]
[최종 보고서에 포함할 내용 설명]

[If applicable, include a table showing before/after state]
[해당하는 경우 이전/이후 상태를 보여주는 표를 포함하세요.]

| Aspect<br>측면 | [Column 1]<br>[열 1] | [Column 2]<br>[칼럼 2] | [Column 3]<br>[3열] | [Column 4]<br>[4열] |
|--------|-----------|-------------|----------|------------------------|
| **Before**<br>**이전** | [Value]<br>[값] | [Value]<br>[값] | [Value]<br>[값] | [Value]<br>[값] |
| **After**<br>**이후** | [Value]<br>[값] | [Value]<br>[값] | [Value]<br>[값] | [Value]<br>[값] |


## References
## 참고자료

- [UltraFast Design Methodology Guide (UG949)](references)
- [UltraFast 설계 방법론 가이드(UG949)](참조)
- [Vivado Design Suite User Guide: Design Analysis and Closure Techniques (UG906)](references)
- [Vivado Design Suite 사용자 가이드: 설계 분석 및 폐쇄 기술(UG906)](참조)
- [Using Constraints (UG903)](references)
- [제약조건 사용(UG903)](참조)
- [Related methodology documents if applicable](../references/RELATED_DOC.md)
- [해당되는 경우 관련 방법론 문서](../references/RELATED_DOC.md)
