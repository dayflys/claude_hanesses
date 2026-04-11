---
name: sv-code-impl
description: "Speaker Verification 코드 구현 스킬. SVMixer/ReDimNet/ECAPA 모델 수정, 손실함수 구현, 데이터 augmentation 추가, PyTorch Lightning 학습 로직 변경, 버그 수정을 수행한다. '모델 수정', '아키텍처 변경', '손실함수', 'augmentation', '버그 수정', '코드 구현', '학습 로직' 요청 시 사용."
---

# SV Code Implementation

Speaker Verification 코드베이스의 구현 및 수정을 담당하는 스킬.

## 코드베이스 구조

```
code/SSLbackend/{experiment_variant}/
├── arguments.py      ← 실험 설정 (딕셔너리)
├── main.py           ← 진입점: Neptune 로깅, DataModule/ModelModule 생성, Trainer 실행
├── data_module.py    ← LightningDataModule: VoxCeleb2 로딩, augmentation, 배치 구성
└── model_module.py   ← LightningModule: 학습/평가 step, 메트릭 계산

공유 모듈 (여러 실험에서 import):
├── models/
│   ├── svmixer.py    ← Speaker Mixer 백엔드
│   ├── redimnet.py   ← ReDimNet 차원축소 네트워크
│   └── ecapa.py      ← ECAPA-TDNN 백엔드
├── loss/
│   ├── aam_softmax.py        ← Additive Angular Margin Softmax
│   └── distillhubert_loss.py ← 지식증류 손실
├── dataset/
│   ├── voxceleb2.py  ← VoxCeleb2 데이터셋
│   ├── voices.py     ← VOiCES 데이터셋
│   └── vcmix.py      ← VCMix 데이터셋
├── data_augmentation/ ← MUSAN noise, RIR reverb, clipping 등
└── utils/
    └── metrics.py    ← EER, minDCF 계산
```

## 구현 패턴

### 모델 수정 시
1. 기존 모델 코드를 먼저 읽는다
2. forward() 함수의 입출력 텐서 shape을 파악한다
3. WavLM 프론트엔드 출력: `(batch, seq_len, 1024)` — 24개 레이어의 hidden states
4. 백엔드 입력은 프론트엔드 출력의 가중합 또는 특정 레이어 선택
5. 최종 출력: `(batch, embedding_size)` — speaker embedding

### 손실함수 수정 시
- AAMSoftmax 파라미터: margin, scale, num_classes
- forward 입력: `(embedding, label)` → 출력: `(loss, accuracy)`
- 새 손실함수 추가 시 loss/ 디렉토리에 파일 생성, model_module.py에서 import

### 데이터 파이프라인 수정 시
- augmentation은 data_augmentation/ 모듈에서 관리
- 적용 순서: raw audio → augmentation → feature extraction
- crop_size 변경은 메모리에 직접 영향. batch_size와 함께 조정 필요

### 학습 로직 수정 시
- Lightning Trainer 설정은 main.py에서 관리
- 콜백, 체크포인트 전략은 main.py의 Trainer 초기화 부분
- 평가 로직(EER/minDCF 계산)은 model_module.py의 on_validation_epoch_end()

## 주의사항

- **DDP 호환성**: all_gather, sync_dist 사용 확인. 단일 GPU에서 동작하더라도 DDP로 확장 가능해야 한다
- **체크포인트 호환성**: state_dict 키 변경 시 기존 체크포인트 로딩이 깨질 수 있다. 새 파라미터 추가 시 strict=False 옵션 고려
- **메모리 관리**: gradient accumulation 설정(grad_accum)과 batch_size의 관계. effective batch = batch_size * grad_accum
- **스크립트 백업**: main.py 라인 31-38에서 학습 시작 시 코드를 결과 디렉토리에 자동 백업한다. 이 로직을 깨뜨리지 않도록 한다
