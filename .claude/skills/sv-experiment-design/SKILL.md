---
name: sv-experiment-design
description: "Speaker Verification 실험 설계 스킬. 새 실험 변형의 arguments.py 생성, 하이퍼파라미터 탐색 전략, ablation study 설계를 수행한다. '새 실험', '실험 설계', 'arguments 생성', '하이퍼파라미터 변경', 'ablation', '변형 추가' 요청 시 사용."
---

# SV Experiment Design

Speaker Verification 실험의 설계와 구성을 담당하는 스킬.

## 워크플로우

### 1. 기존 실험 조사

새 실험 설계 전 반드시 기존 변형을 파악한다:

```
/home/koo/code/code/SSLbackend/   ← 100+ 실험 변형 디렉토리
```

각 변형 디렉토리는 디렉토리명으로 변경 내용을 인코딩한다:
- `overlap4_insert4` → overlap/insert 파라미터 변경
- `W1d_1dk5` → 가중치/차원 변경
- `seed_1024` → 시드 변경
- `SVMixer`, `ReDim` → 아키텍처 변경

기존 변형의 arguments.py를 읽어 어떤 파라미터 조합이 이미 시도되었는지 확인한다.

### 2. 실험 구성 생성

arguments.py 구조 (딕셔너리 기반):

```python
args = {
    # 프로젝트 설정
    "project_name": "IRLAB/SSLBackend2025",
    "experiment_name": "{의미론적_이름}",
    
    # 모델 설정
    "hidden_size": 1024,        # WavLM hidden dim (변경 드묾)
    "embedding_size": 192,      # speaker embedding dim
    "merge_layer_num": 4,       # SSL 레이어 병합 수
    "insert_feature_num": 4,    # 특징 삽입 수
    
    # 학습 설정
    "batch_size": 1536,
    "grad_accum": 32,
    "max_steps": 18000,
    "eval_interval": 300,
    "lr_max": 3e-3,
    "lr_min": 3e-4,
    
    # 손실함수
    "margin": 0.3,
    "scale": 30.0,
    
    # 데이터
    "crop_size": 48000,         # 3초 @ 16kHz
    "num_seg": 10,              # 평가 시 세그먼트 수
}
```

### 3. 설계 원칙

**Ablation 기본 규칙:** 한 번에 하나의 변수만 변경한다. 복합 변경이 필요하면 중간 실험을 먼저 설계하여 각 변수의 독립적 효과를 측정한다.

**네이밍 규칙:** 디렉토리명에 변경 내용을 인코딩한다. 기존 변형과 비교 가능하도록 base 실험을 명시한다.

**핵심 하이퍼파라미터 탐색 범위:**

| 파라미터 | 일반적 범위 | 주의사항 |
|---------|-----------|---------|
| embedding_size | 128, 192, 256 | 크면 표현력 증가, 메모리 증가 |
| merge_layer_num | 2, 4, 6, 8 | SSL 레이어 수(24)를 넘지 않도록 |
| margin (AAMSoftmax) | 0.1 ~ 0.5 | 너무 크면 학습 불안정 |
| scale (AAMSoftmax) | 15, 30, 64 | margin과 함께 조정 |
| lr_max | 1e-4 ~ 1e-2 | SSL 프론트엔드 unfreeze 시 더 작게 |
| crop_size | 32000, 48000, 64000 | 길면 정확도 향상, 메모리 증가 |

### 4. 디렉토리 생성

새 실험 디렉토리에 필수 파일 4개를 생성한다:
1. `arguments.py` — 실험 구성
2. `main.py` — 학습 진입점 (기존 변형에서 복제)
3. `data_module.py` — 데이터 모듈 (기존 변형에서 복제)
4. `model_module.py` — 모델 모듈 (변경 필요 시 수정, 아니면 복제)

기존 변형의 파일을 복제한 뒤 arguments.py만 수정하는 것이 가장 안전하다. 아키텍처 변경이 필요하면 code-engineer에게 위임한다.
