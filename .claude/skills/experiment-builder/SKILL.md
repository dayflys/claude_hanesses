---
name: experiment-builder
description: "SSL Speaker Verification 실험 코드를 생성하고 수정하는 스킬. 새 실험 디렉토리 생성, arguments.py 수정, 모델 아키텍처 변경 (SVMixer, ReDimNet, ECAPA, ResNet34, WavLM), 손실 함수 변경 (AAMSoftmax, DistilHubertKD, SphereFace2), 학습 스케줄 조정, 데이터 파이프라인 수정. '실험 만들어줘', '모델 바꿔줘', '하이퍼파라미터 변경', '새 실험', 'AAM margin 변경' 등 실험 코드 작업 시 반드시 사용."
---

# Experiment Builder Skill

SSL 화자 검증 연구의 실험 코드를 생성하고 수정한다.

## 실험 생성 워크플로우

### Step 1: 참조 실험 선택

사용자가 참조 실험을 지정하지 않으면, 연구 트랙과 요청 내용을 기반으로 가장 적합한 기존 실험을 선택한다.

**연구 트랙 판별 기준:**
- WavLM/Wav2Vec2 teacher + distillation → KD 트랙 (Short, SSRFNet)
- SVMixer + ECAPA → Short/baseline
- SVMixer + ReDimNet → SSRFNet
- ResNet34 / ECAPA standalone → OSSV
- Spoof/anti-spoofing → SpD

### Step 2: 실험 복사

참조 실험의 전체 파일을 새 디렉토리로 복사한다. 이후 요청된 변경 사항만 수정한다.

### Step 3: arguments.py 수정

arguments.py는 `args = { ... }` 딕셔너리 형태다. 변경이 필요한 키만 수정한다.

**주요 설정 카테고리:**

| 카테고리 | 주요 키 | 설명 |
|---------|--------|------|
| 메타데이터 | `name`, `project`, `entity`, `seed` | 실험 이름/프로젝트/시드 |
| 모델 | `hidden_size`, `num_hidden_layers`, `seq_len`, `embed_dim` | 모델 아키텍처 파라미터 |
| 분류기 | `embedding_size`, `C`, `F` (ReDimNet) | 분류기 파라미터 |
| 학습 | `warmup_steps`, `rise_steps`, `total_train_steps`, `num_epochs` | 학습 스케줄 |
| 학습률 | `warmup_lr_max`, `lr_max`, `lr_min`, `weight_decay` | 학습률 설정 |
| 손실 | `aam_margin`, `aam_scale`, `cos_lambda`, `topk_penalty` | 손실 함수 파라미터 |
| 데이터 | `batch_size`, `crop_size`, `num_seg`, `augment_probability` | 데이터 설정 |
| 인프라 | `result_path`, `neptune_token`, `wandb` 관련 | 로깅/결과 저장 |

### Step 4: main.py 수정 (필요 시)

main.py 수정이 필요한 경우:
- **모델 아키텍처 변경** — import 문과 모델 초기화 코드 수정
- **손실 함수 변경** — import 문과 criterion 초기화 수정
- **로거 변경** — Neptune ↔ WandB 전환
- **콜백 추가/삭제** — ModelCheckpoint, OptimizerSwitch 등

main.py 수정 시 기존 패턴을 따른다:
```python
# KD 트랙 패턴
teacher_model = WavLMModel.from_pretrained(args['path_huggingface'])
teacher_model = FrozenWrapper(teacher_model)
student_model = SVMixer(args)  # 또는 다른 student
classifier = ECAPA_TDNN(args) # 또는 ReDimNet
```

### Step 5: model_module.py 수정 (필요 시)

model_module.py 수정이 필요한 경우:
- **optimizer 변경** — Adam, SGD, AdamW 전환
- **학습률 스케줄 변경** — warmup/rise 파라미터, cosine annealing 등
- **마진 스케줄 변경** — ExponentialIncrease/Decrease 전환
- **평가 로직 변경** — 새 메트릭 추가, TTA 설정 변경

### Step 6: 기타 파일 수정

- **models/**: 새 모델 아키텍처 추가 시
- **loss/**: 새 손실 함수 추가 시
- **dataset/**: 새 데이터셋 추가 시
- **data_augmentation/**: 새 증강 기법 추가 시

## 실험 수정 워크플로우

기존 실험의 특정 컴포넌트만 변경할 때:

1. 대상 실험의 관련 파일을 읽는다
2. 변경 사항을 적용한다
3. 변경이 다른 파일에 영향을 주는지 확인한다:
   - arguments.py의 모델 파라미터 변경 → main.py의 모델 초기화 확인
   - 모델 output_dim 변경 → 분류기 input_dim, 손실 함수 확인
   - batch_size 변경 → accumulate_grad_batches 조정 필요 여부 확인

## 주의사항

- 데이터 경로(`path_*`)는 서버 환경에 따라 다를 수 있으므로, 참조 실험의 경로를 그대로 유지한다. 사용자가 명시적으로 변경을 요청한 경우에만 수정한다.
- Neptune/WandB 토큰은 arguments.py에 하드코딩되어 있다. 복사 시 그대로 유지한다.
- `result_path`에 실험 이름이 포함된다. 새 실험 생성 시 이름에 맞게 업데이트한다.
- seed는 기본 4221을 사용한다. 사용자가 다른 seed를 요청하면 변경한다.
