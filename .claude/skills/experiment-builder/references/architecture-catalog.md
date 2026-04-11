# Architecture Catalog — SSL Speaker Verification

실험 구성 시 참조하는 모델/손실/데이터 카탈로그.

## 목차
1. [Student Models](#1-student-models)
2. [Teacher Models](#2-teacher-models)
3. [Classifiers](#3-classifiers)
4. [Loss Functions](#4-loss-functions)
5. [Training Schedules](#5-training-schedules)
6. [Research Track Templates](#6-research-track-templates)

---

## 1. Student Models

### SVMixer
- **파일**: `models/svmixer.py`
- **역할**: Audio feature encoder (MLP-Mixer 기반)
- **주요 파라미터**:
  - `num_hidden_layers`: 레이어 수 (2~12)
  - `seq_len`: 시퀀스 길이 (49~149)
  - `hidden_size`: 히든 차원 (1024)
- **출력**: 각 레이어의 hidden states (KD loss 대상)
- **입력**: Raw waveform (16kHz)
- **용도**: Short/baseline, SSRFNet 트랙

### ResNet34
- **파일**: `models/resnet34.py` 또는 Wespeaker 구현
- **역할**: Supervised speaker encoder
- **주요 파라미터**:
  - `embed_dim`: 임베딩 차원 (256)
  - `pooling_func`: TSTP (Time-Aware SAP)
- **출력**: Speaker embedding vector
- **입력**: FBANK features (80 mel bins)
- **용도**: OSSV 트랙

### gMLP
- **파일**: `model/gmlp.py`
- **역할**: Spoof detection encoder
- **용도**: SpD 트랙
- **SSL backbone**: Wav2Vec2-XLS-R-300M (frozen)

## 2. Teacher Models

### WavLM-Large
- **HuggingFace**: `microsoft/wavlm-large`
- **Hidden dim**: 1024
- **Layers**: 24
- **KD target layers**: 보통 `[8, 16, 24]`
- **사용법**: `FrozenWrapper(WavLMModel.from_pretrained(...))`
- **용도**: Short/baseline, SSRFNet

### Wav2Vec2-XLS-R-300M
- **HuggingFace**: `facebook/wav2vec2-xls-r-300m`
- **용도**: SpD 트랙

## 3. Classifiers

### ECAPA-TDNN
- **파일**: `models/ecapa.py`
- **주요 파라미터**:
  - `C`: 채널 수 (512~1024)
  - `embedding_size`: 출력 임베딩 (192)
- **구조**: Bottle2neck + ASP (Attentive Statistics Pooling)
- **용도**: Short/baseline

### ReDimNet
- **파일**: `models/redimnet.py`
- **주요 파라미터**:
  - `F`: Feature dimension (64)
  - `C`: Channel (16)
  - `embedding_size`: 출력 임베딩 (192)
  - `insert_feature_num`: 삽입 피처 수 (4)
  - `merge_layer_num`: 병합 레이어 수 (4)
- **구조**: ConvNext-like blocks + SE channels
- **용도**: SSRFNet

## 4. Loss Functions

### AAMSoftmax (Additive Angular Margin)
- **파일**: `loss/aam_softmax.py`
- **파라미터**:
  - `margin`: 각도 마진 (0.2)
  - `scale`: 스케일링 (30)
  - `topk_penalty`: Hard negative mining (선택)
- **필요**: `num_classes` (화자 수)
- **마진 스케줄**: 0.0 → margin (Exponential Increase during rise phase)

### DistilHubertKDLoss
- **파일**: `loss/distillhubert_loss.py`
- **파라미터**:
  - `target_layers`: Teacher 타겟 레이어 `[8, 16, 24]`
  - `cos_lambda`: Cosine loss 가중치 (1.0)
- **계산**: L1 loss - cos_lambda * cosine_similarity
- **입력**: Student hidden states + Teacher hidden states

### SphereFace2
- **파일**: `loss/sphere2.py`
- **용도**: OSSV 트랙 대안

### CrossEntropyLoss
- **PyTorch 내장**
- **용도**: OSSV baseline

## 5. Training Schedules

### Epoch-based (OSSV)
```
Optimizer: SGD (momentum=0.9, weight_decay=1e-4)
LR: ExponentialDecrease 0.1 → 5e-5
Warmup: 6 epochs
Margin: ExponentialIncrease 0.0 → 0.2
Total: 150 epochs
```

### Step-based (KD tracks: Short, SSRFNet)
```
Phase 1 - Warmup: Adam (lr 1e-5 → 1e-3), warmup_steps
Phase 2 - Rise: Adam (lr 1e-3), margin 0.0 → 0.2, rise_steps
Phase 3 - Main: SGD (lr 1e-5 → 1e-1, cosine), total_train_steps
Eval: Every eval_interval_steps (2000)
Gradient clip: 10.0
```

## 6. Research Track Templates

### OSSV Template
```
Student: ResNet34 (embed_dim=256, TSTP)
Teacher: None
Loss: CrossEntropy + ArcMargin (scale=32, margin=0.2)
Data: FBANK(80 bins) + speed_perturb + reverb/noise
Logger: WandB
Training: Epoch-based, SGD, 150 epochs
```

### Short/baseline Template
```
Student: SVMixer (layers=2, seq_len=49, hidden=1024)
Classifier: ECAPA-TDNN (C=512, embed=192)
Teacher: WavLM-Large (frozen)
Loss: AAMSoftmax (margin=0.2, scale=30) + DistilHubertKD
Data: Raw waveform + augmentation
Logger: WandB
Training: Step-based, Adam→SGD, ~300k steps
```

### SSRFNet Template
```
Student: SVMixer (layers=12, seq_len=149, hidden=1024)
Classifier: ReDimNet (F=64, C=16, embed=192)
Teacher: WavLM-Large (frozen)
Loss: AAMSoftmax (margin=0.2, scale=30) + DistilHubertKD
Data: Raw waveform + augmentation
Logger: Neptune
Training: Step-based, Adam→SGD, ~400k steps
```

### SpD Template
```
Model: gMLP + Wav2Vec2-XLS-R-300M
Loss: P2G Loss (CCE based)
Data: ASVspoof2019/2021, PartialSpoof
Training: Separate pipeline (non-Lightning or Lightning variant)
```
