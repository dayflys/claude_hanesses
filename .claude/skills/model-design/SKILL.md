---
name: model-design
description: "Audio spoofing detection 모델 아키텍처 설계 및 구현 스킬. HM-Conformer, AASIST, RawNet2, LCNN, SEResNet34 등 anti-spoofing 모델의 새 아키텍처 구현, 기존 모델 구조 수정, 모델 컴포넌트(인코더, 풀링, 분류기, 손실함수) 변경, 논문 기반 모델 재현. 모델 설계, 모델 구현, 모델 수정, 새 모델, 아키텍처 변경, 레이어 추가, conformer, attention 등 모델 관련 요청 시 반드시 이 스킬을 사용."
---

# Model Design -- Anti-Spoofing 모델 아키텍처 스킬

## 모델 구현 워크플로우

### 1. 기존 모델 파악
작업 전 반드시 타겟 baseline 디렉토리의 현재 모델 코드를 읽는다:
- `model/get_model.py` — 모델 팩토리, 등록된 모델 목록
- `model/loss.py` — OC-Softmax 손실함수 인터페이스
- `model/_processing.py` — 특징 추출 (LFCC, positional encoding)
- `arguments.py` — 모델 하이퍼파라미터 (`model_args` 섹션)

### 2. 모델 구현 규칙

**forward 시그니처 호환:**
```python
def forward(self, x, label=None):
    # x: (batch, 1, num_samples) — raw waveform
    # 또는 (batch, num_frames, num_features) — LFCC
    # return: score, loss (학습 시) 또는 score (평가 시)
```

**OC-Softmax 연결:**
- 모델의 임베딩 출력은 `ClassifierWithLoss`에 전달된다
- 임베딩 차원은 `output_size` (기본 128)와 일치해야 한다
- 다중 레벨 CLS 사용 시 `loss_weight` 리스트로 가중치를 지정한다

**모델 등록:**
```python
# model/get_model.py에 추가
if args['model'] == 'NewModel':
    from model.new_model import NewModel
    return NewModel(args)
```

### 3. 핵심 컴포넌트 참조

| 컴포넌트 | 파일 | 역할 |
|----------|------|------|
| Conformer 블록 | `model/hm_conformer.py` | RelPositionAttention + ConvModule + FFN |
| LFCC 추출 | `model/_processing.py` | 120-bin 스펙트로그램, TRIMF, delta |
| 풀링 | `model/lfcc_hm_conformer.py` | SelfWeightedPooling, Hierarchical CLS |
| 손실함수 | `model/loss.py` | OC-Softmax (r_real=0.9, r_fake=0.2, alpha=20) |
| 다운샘플링 | `model/hm_conformer.py` | Conv2dSubsampling2 |

### 4. 체크포인트
- [ ] forward 시그니처가 기존 모델과 호환
- [ ] `get_model.py`에 등록
- [ ] `arguments.py`에 하이퍼파라미터 추가
- [ ] DDP 호환 (SyncBatchNorm 등)
- [ ] GPU 텐서 연산 호환
