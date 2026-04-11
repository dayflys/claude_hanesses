---
name: ssl-orchestrator
description: "SSL Speaker Verification 연구 프로젝트의 에이전트 팀을 조율하는 오케스트레이터. 새 실험 생성, 실험 수정, 모델 아키텍처 변경, 하이퍼파라미터 튜닝, 학습 디버깅, 실험 비교, 결과 분석, 실험 검증/QA, 실험 큐 관리 요청 시 반드시 이 스킬을 사용. 후속 작업: 실험 결과 수정, 부분 재실행, 업데이트, 보완, 다시 실행, 이전 결과 개선, 실험 코드 리뷰 요청 시에도 사용."
---

# SSL Research Orchestrator

SSL 기반 화자 검증(Speaker Verification) 연구의 실험 관리를 위한 에이전트 팀 오케스트레이터.

## 실행 모드: 서브 에이전트

이 프로젝트의 작업은 대부분 독립적이고, 에이전트 간 실시간 통신보다 결과 전달이 중요하므로 서브 에이전트 모드를 사용한다.

## 에이전트 구성

| 에이전트 | subagent_type | 역할 | 모델 |
|---------|--------------|------|------|
| experiment-builder | experiment-builder | 실험 생성/수정 | opus |
| code-analyzer | code-analyzer | 실험 분석/디버깅 | opus |
| experiment-qa | experiment-qa | 실험 검증/QA | opus |

## 워크플로우

### Phase 0: 컨텍스트 확인

1. 사용자 요청을 분석하여 작업 유형을 결정한다:
   - **실험 생성**: 새 실험 디렉토리 생성 → Phase 1A
   - **실험 수정**: 기존 실험의 특정 컴포넌트 변경 → Phase 1B
   - **실험 분석**: 실험 비교, 결과 해석, 디버깅 → Phase 1C
   - **실험 검증**: QA/리뷰 → Phase 1D
   - **복합 작업**: 여러 유형이 조합된 경우 → 해당 Phase를 순차 수행

2. `_workspace/` 디렉토리 존재 여부를 확인한다:
   - 미존재 → 초기 실행
   - 존재 + 부분 수정 요청 → 해당 에이전트만 재호출
   - 존재 + 새 입력 → `_workspace/`를 타임스탬프 백업 후 새 실행

### Phase 1A: 실험 생성

1. 사용자 요청에서 파악:
   - 연구 트랙 (OSSV / Short / SSRFNet / SpD / NAW-SV)
   - 모델 아키텍처 (Student + Teacher 조합)
   - 손실 함수 조합
   - 변경할 하이퍼파라미터
   - 참조할 기존 실험 (없으면 자동 선택)

2. `_workspace/` 생성

3. **code-analyzer** 호출 (background):
   - 참조 실험의 구조와 설정을 분석
   - 유사 실험들과의 차이점 정리
   ```
   Agent(
     name: "analyzer",
     subagent_type: "code-analyzer",
     model: "opus",
     prompt: "다음 실험을 분석하라: {참조 실험 경로}. arguments.py의 모든 설정을 카테고리별로 정리하고, 같은 트랙의 다른 실험과 비교하여 특이한 값이 있는지 보고하라.",
     run_in_background: true
   )
   ```

4. **experiment-builder** 호출:
   - 참조 실험을 기반으로 새 실험 생성
   - 사용자 요청에 따른 수정 적용
   ```
   Agent(
     name: "builder",
     subagent_type: "experiment-builder",
     model: "opus",
     prompt: "{사용자 요청}에 따라 새 실험을 생성하라. 참조 실험: {경로}. 변경 사항: {상세}. 실험을 {대상 경로}에 생성하라."
   )
   ```

5. **experiment-qa** 호출:
   - 생성된 실험의 일관성 검증
   ```
   Agent(
     name: "qa",
     subagent_type: "experiment-qa",
     model: "opus",
     prompt: "다음 실험의 일관성을 검증하라: {생성된 실험 경로}. 참조 실험: {경로}와 비교하여 의도하지 않은 차이가 없는지 확인하라."
   )
   ```

6. QA 결과에 문제가 있으면 experiment-builder를 재호출하여 수정

7. 결과 요약 보고

### Phase 1B: 실험 수정

1. 수정 대상 실험과 변경 사항 파악

2. **code-analyzer** 호출 (해당 실험 현황 분석)

3. **experiment-builder** 호출 (수정 적용)

4. **experiment-qa** 호출 (수정 후 검증)

5. 결과 보고

### Phase 1C: 실험 분석

1. 분석 유형 파악 (비교 / 디버깅 / 결과 해석)

2. **code-analyzer** 호출:
   ```
   Agent(
     name: "analyzer",
     subagent_type: "code-analyzer",
     model: "opus",
     prompt: "{분석 요청 상세}"
   )
   ```

3. 분석 결과를 사용자에게 보고

### Phase 1D: 실험 검증

1. **experiment-qa** 호출:
   ```
   Agent(
     name: "qa",
     subagent_type: "experiment-qa",
     model: "opus",
     prompt: "다음 실험을 검증하라: {실험 경로}. 전체 체크리스트에 따라 검증 결과를 보고하라."
   )
   ```

2. 검증 결과를 사용자에게 보고

## 데이터 흐름

```
[사용자 요청]
     │
     ▼
[오케스트레이터: Phase 0 분류]
     │
     ├── 실험 생성 ──────────────────────────────────┐
     │   ├→ code-analyzer (background: 참조 분석)     │
     │   ├→ experiment-builder (생성)                 │
     │   └→ experiment-qa (검증) ←────────────────────┘
     │
     ├── 실험 수정 ──→ analyzer → builder → qa
     │
     ├── 실험 분석 ──→ code-analyzer
     │
     └── 실험 검증 ──→ experiment-qa
```

## 산출물 저장

| 에이전트 | 출력 경로 |
|---------|----------|
| code-analyzer | `_workspace/analysis_{timestamp}.md` |
| experiment-builder | 실험 디렉토리 내 파일들 (직접 생성/수정) |
| experiment-qa | `_workspace/qa_report_{timestamp}.md` |

## 에러 핸들링

| 상황 | 전략 |
|------|------|
| 참조 실험을 찾을 수 없음 | 사용자에게 연구 트랙과 유사 실험 목록을 보여주고 선택 요청 |
| experiment-builder 실패 | 에러 내용을 분석하여 1회 재시도. 재실패 시 사용자에게 보고 |
| QA에서 차원 불일치 발견 | experiment-builder를 재호출하여 자동 수정 시도 |
| QA에서 경고 수준 이슈 | 사용자에게 경고를 보고하고 진행 여부 확인 |
| 데이터 경로 존재하지 않음 | 경고 표시. 서버 환경에서는 경로가 다를 수 있으므로 차단하지 않음 |

## 연구 트랙별 참조 실험

| 트랙 | 대표 실험 경로 | 특징 |
|------|--------------|------|
| OSSV | `code/OSSV/fire/Final_2_Wespeaker_resnet34/` | Supervised, Epoch 기반 |
| Short | `code/Short/baseline/KW6. baseline2_TTA_3_1S_SVMixer_ECAPA_Origin/` | KD, Step 기반, TTA |
| SSRFNet | `code/SSRFNet/0.SSRFNet_specPurb_stepup_lr_v2/` | KD, SVMixer+ReDimNet |
| SpD | `code/SpD/last_reproduce_SpD4_maxpool_CCE_mul_bi_s2024_baseline/` | Spoof Detection |

## 테스트 시나리오

### 정상 흐름: 새 실험 생성
1. 사용자: "SSRFNet 트랙에서 SVMixer 12레이어 + ReDimNet으로 AAM margin 0.3인 실험을 만들어줘"
2. Phase 0: 실험 생성으로 분류
3. Phase 1A-3: code-analyzer가 기존 SSRFNet 실험 분석 (background)
4. Phase 1A-4: experiment-builder가 0.SSRFNet_specPurb_stepup_lr_v2를 참조하여 새 실험 생성
5. Phase 1A-5: experiment-qa가 검증 — 차원 일관성, margin 값 확인
6. 결과: 새 실험 디렉토리 + QA 보고서

### 에러 흐름: QA 실패
1. Phase 1A-5에서 QA가 차원 불일치 발견 (SVMixer output_dim ≠ ReDimNet input_dim)
2. 오케스트레이터가 experiment-builder를 재호출하여 차원 수정
3. experiment-qa 재검증
4. 통과 시 결과 보고, 재실패 시 사용자에게 상세 보고
