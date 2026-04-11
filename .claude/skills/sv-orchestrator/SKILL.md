---
name: sv-orchestrator
description: "Speaker Verification 연구 오케스트레이터. 실험 설계, 코드 구현, 결과 분석을 조율한다. SV/화자인증 관련 모든 작업 요청 시 사용. 새 실험 설계, 모델 수정, 결과 비교, ablation, 아키텍처 변경, 하이퍼파라미터 탐색, EER/minDCF 분석. 후속: 결과 수정, 부분 재실행, 업데이트, 보완, 다시 실행, 이전 결과 개선, 실험 재설계."
---

# SV Research Orchestrator

Speaker Verification 연구의 에이전트를 조율하여 실험 설계-구현-분석 파이프라인을 실행하는 통합 스킬.

## 실행 모드: 서브 에이전트

작업 유형에 따라 적합한 전문가 에이전트를 호출한다. 대부분의 요청은 단일 에이전트로 처리 가능하며, 복합 요청 시 순차적으로 체이닝한다.

## 에이전트 구성

| 에이전트 | subagent_type | 역할 | 스킬 | 출력 |
|---------|--------------|------|------|------|
| experiment-architect | experiment-architect | 실험 설계, arguments.py 생성 | sv-experiment-design | 새 실험 디렉토리 |
| code-engineer | code-engineer | 모델/학습 코드 구현 | sv-code-impl | 수정된 코드 파일 |
| results-analyst | results-analyst | 결과 분석, 비교표 생성 | sv-results-analysis | 분석 보고서 |

## 워크플로우

### Phase 0: 컨텍스트 확인

1. `_workspace/` 디렉토리 존재 여부 확인
2. 실행 모드 결정:
   - **`_workspace/` 미존재** → 초기 실행. Phase 1로 진행
   - **`_workspace/` 존재 + 부분 수정 요청** → 해당 에이전트만 재호출, 기존 산출물 경로를 프롬프트에 포함
   - **`_workspace/` 존재 + 새 입력** → `_workspace/`를 `_workspace_{YYYYMMDD_HHMMSS}/`로 이동 후 Phase 1

### Phase 1: 요청 분류

사용자 요청을 분석하여 작업 유형을 결정한다:

| 요청 유형 | 호출 에이전트 | 예시 |
|----------|------------|------|
| 실험 설계 | experiment-architect | "margin 0.4로 새 실험 만들어줘" |
| 코드 구현 | code-engineer | "SVMixer에 SE block 추가해줘" |
| 결과 분석 | results-analyst | "최근 5개 실험 EER 비교해줘" |
| 설계 + 구현 | architect → engineer (순차) | "crop_size 64000으로 새 실험 만들고 구현해줘" |
| 전체 파이프라인 | architect → engineer → analyst | "embedding 256으로 ablation 하고 결과 분석" |
| 버그 수정 | code-engineer | "학습 시 loss가 nan이야" |

**복합 요청 처리:** 여러 에이전트가 필요하면 순차 호출한다. 이전 에이전트의 산출물 경로를 다음 에이전트의 프롬프트에 포함한다.

### Phase 2: 에이전트 호출

단일 에이전트 호출 패턴:

```
Agent(
  description: "{작업 요약}",
  subagent_type: "{agent-type}",
  model: "opus",
  prompt: "
    당신은 {역할}입니다.
    Read('.claude/agents/{agent-type}.md')로 에이전트 정의를 확인하라.
    Read('.claude/skills/{skill-name}/SKILL.md')로 스킬 가이드를 확인하라.
    
    작업: {구체적 작업 내용}
    컨텍스트: {이전 에이전트 산출물 경로 또는 사용자 입력}
  "
)
```

병렬 호출 패턴 (독립 작업):

```
# 동시에 여러 Agent 도구 호출
Agent(agent-1, run_in_background: true, ...)
Agent(agent-2, run_in_background: true, ...)
```

### Phase 3: 결과 통합

1. 에이전트 반환값 수집
2. 파일 기반 산출물은 Read로 확인
3. 사용자에게 결과 요약 보고:
   - 무엇이 생성/수정되었는지
   - 핵심 결정 사항과 근거
   - 후속 추천 작업 (해당 시)

### Phase 4: 정리

1. 중간 산출물은 `_workspace/`에 보존
2. 최종 산출물 경로를 사용자에게 안내

## 데이터 흐름

```
[오케스트레이터]
    │
    ├─ 실험 설계 요청 ──→ [experiment-architect] ──→ arguments.py + 실험 디렉토리
    │                                                      │
    ├─ 코드 구현 요청 ──→ [code-engineer] ────────→ 수정된 코드 파일
    │                          ↑ (architect 산출물 참조)
    │
    └─ 결과 분석 요청 ──→ [results-analyst] ──────→ 비교표/보고서
```

## 에러 핸들링

| 상황 | 전략 |
|------|------|
| 에이전트 1개 실패 | 에러 내용 분석 → 1회 재시도 (프롬프트 보완). 재실패 시 사용자에게 보고 |
| 기존 실험과 설정 충돌 | architect가 감지 → 사용자에게 차이점 제안 |
| 결과 파일 미존재 | analyst가 가용한 데이터로 부분 분석, 누락 명시 |
| 코드 호환성 문제 | engineer가 shape/import 에러 분석 → 수정안 제시 |

## 테스트 시나리오

### 정상 흐름: 새 실험 설계 + 구현
1. 사용자: "embedding_size 256으로 새 ablation 실험 만들어줘"
2. Phase 1: "설계 + 구현" 유형 판별
3. Phase 2: experiment-architect 호출 → arguments.py 생성
4. Phase 2: code-engineer 호출 → 실험 디렉토리 완성
5. Phase 3: 생성된 파일 목록과 변경 내용 요약
6. 예상 결과: `/home/koo/code/code/SSLbackend/{new_variant}/` 생성

### 에러 흐름: 결과 파일 부재
1. 사용자: "exp_A와 exp_B 결과 비교해줘"
2. Phase 1: "결과 분석" 유형 판별
3. Phase 2: results-analyst 호출
4. analyst가 exp_B 결과 디렉토리 미존재 감지
5. Phase 3: "exp_B 결과 미존재, exp_A 단독 분석 결과" + 경고 보고
