---
name: spoofing-orchestrator
description: "Audio spoofing detection 프로젝트의 에이전트를 조율하는 오케스트레이터. 모델 설계/구현, 데이터 파이프라인, 학습 설정, 평가 분석, 코드 품질 검증 등 anti-spoofing 관련 복합 작업을 전문 에이전트에게 위임하여 수행. ASVspoof, 음성 위변조 탐지, deepfake detection, HM-Conformer, AASIST, 모델 학습, 데이터 증강, EER 평가 등 관련 작업 요청 시 반드시 이 스킬을 사용. 후속 작업: 결과 수정, 부분 재실행, 업데이트, 보완, 다시 실행, 이전 결과 개선, 코드 리뷰 요청 시에도 사용."
---

# Spoofing Detection Orchestrator

Audio spoofing detection 프로젝트의 전문 에이전트를 조율하여 복합 작업을 수행하는 통합 스킬.

## 실행 모드: 서브 에이전트

ML 연구 프로젝트의 특성상 사용자가 단계별로 주도하며, 전문가 간 실시간 통신보다 각 전문가의 독립적 작업 결과가 중요하므로 서브 에이전트 모드를 사용한다.

## 에이전트 구성

| 에이전트 | subagent_type | 역할 | 스킬 | 출력 |
|---------|--------------|------|------|------|
| architect | architect | 모델 아키텍처 설계/구현 | model-design | 모델 코드 |
| data-engineer | data-engineer | 데이터 파이프라인 | data-pipeline | 데이터 로더/증강 코드 |
| trainer | trainer | 학습/실험 관리 | training | 학습 설정/스크립트 |
| evaluator | evaluator | 평가/분석 | evaluation | 평가 보고서 |
| qa | qa | 품질 검증 | - | 검증 보고서 |

## 워크플로우

### Phase 0: 컨텍스트 확인

1. `_workspace/` 디렉토리 존재 여부 확인
2. 실행 모드 결정:
   - **`_workspace/` 미존재** -> 초기 실행. Phase 1로 진행
   - **`_workspace/` 존재 + 부분 수정 요청** -> 해당 에이전트만 재호출. 이전 산출물 경로를 프롬프트에 포함
   - **`_workspace/` 존재 + 새 입력** -> 기존 `_workspace/`를 `_workspace_{timestamp}/`로 이동 후 Phase 1 진행

### Phase 1: 요청 분석 및 라우팅

사용자 요청을 분석하여 필요한 에이전트를 결정한다:

| 요청 유형 | 주 에이전트 | 보조 에이전트 |
|----------|-----------|-------------|
| 새 모델 구현/수정 | architect | qa |
| 데이터 전처리/증강 | data-engineer | qa |
| 학습 설정/실행 | trainer | - |
| 성능 평가/분석 | evaluator | - |
| 코드 리뷰/검증 | qa | - |
| 모델 구현 + 학습 설정 | architect -> trainer | qa |
| 데이터 + 모델 + 학습 | data-engineer -> architect -> trainer | qa |
| 전체 파이프라인 | data-engineer -> architect -> trainer -> evaluator | qa |

### Phase 2: 에이전트 실행

**단일 에이전트 작업:**
```
Agent(
  name: "{agent-name}",
  subagent_type: "{agent-name}",
  model: "opus",
  prompt: "{역할 설명} + {사용자 요청} + {프로젝트 컨텍스트}"
)
```

**복합 작업 (독립적인 에이전트 병렬):**
독립적인 작업은 `run_in_background: true`로 병렬 실행한다.

**순차 의존 작업:**
파이프라인 순서로 순차 실행한다. 이전 에이전트의 산출물을 다음 에이전트의 프롬프트에 포함한다.

### Phase 3: QA 검증 (선택적)

코드 변경이 있는 작업에서 qa 에이전트로 검증한다:
```
Agent(
  name: "qa",
  subagent_type: "qa",
  model: "opus",
  prompt: "다음 변경사항의 정합성을 검증하라: {변경 파일 목록} + {변경 내역 요약}"
)
```

검증 범위:
- 모델 변경: forward shape 호환, loss 연결, get_model 등록
- 데이터 변경: 프로토콜 파싱, 레이블 매핑, 로더 인터페이스
- 학습 변경: arguments.py와 실제 사용처 일치

### Phase 4: 결과 보고

1. 각 에이전트의 작업 결과를 수집
2. QA 검증 결과가 있으면 포함
3. 사용자에게 요약 보고:
   - 변경된 파일 목록
   - 핵심 변경 내용
   - QA 검증 결과 (통과/문제)
   - 다음 단계 제안

## 데이터 흐름

```
[오케스트레이터]
    ├── 요청 분석 → 에이전트 선택
    ├── Agent(architect)  → 모델 코드
    ├── Agent(data-engineer) → 데이터 코드
    ├── Agent(trainer)    → 학습 설정
    ├── Agent(evaluator)  → 평가 보고서
    └── Agent(qa)         → 검증 보고서
         ↓
    결과 종합 → 사용자 보고
```

## 에러 핸들링

| 상황 | 전략 |
|------|------|
| 에이전트 실패 | 1회 재시도. 재실패 시 에러 내용을 사용자에게 보고하고 수동 개입 요청 |
| QA 검증 실패 | 문제점을 해당 에이전트에게 전달하여 수정 요청 (최대 2회) |
| 파일 충돌 | 기존 파일을 백업 후 덮어쓰기. 백업 경로를 사용자에게 알림 |
| 의존 에이전트 실패 | 해당 파이프라인 중단. 완료된 부분까지의 결과 보고 |

## 프로젝트 컨텍스트

에이전트 프롬프트에 항상 포함할 정보:
- 작업 대상 baseline 디렉토리: `code/baseline/baseline_HMConformer_2/` (기본)
- 데이터 경로: `/home/koo/DB2/datas/ASVspoof2019/`
- 결과 경로: `environment/results/`
- 현재 주 모델: HM-Conformer (LFCC + Conformer + Hierarchical Pooling + OC-Softmax)

## 테스트 시나리오

### 정상 흐름
1. 사용자: "HM-Conformer에 SE(Squeeze-and-Excitation) 모듈을 추가해줘"
2. Phase 1: architect 에이전트 선택 + qa 보조
3. Phase 2: architect가 모델 코드 수정
4. Phase 3: qa가 shape 호환성, get_model 등록, forward 시그니처 검증
5. Phase 4: 변경 파일, 핵심 변경, QA 결과 보고

### 에러 흐름
1. 사용자: "AASIST 모델을 새 baseline으로 구현해줘"
2. Phase 2: architect 실행 중 에러 (기존 AASIST 코드와 충돌)
3. 에러 내용 분석 → 1회 재시도 (충돌 해결 지시 추가)
4. 재시도 성공 → Phase 3 QA 검증 진행
5. QA에서 loss 연결 문제 발견 → architect에게 수정 요청
6. 수정 완료 → 최종 보고
