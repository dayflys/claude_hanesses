---
name: training
description: "Audio spoofing detection 학습/실험 관리 스킬. 하이퍼파라미터 설정, 학습 루프 구성, DDP 분산학습, 스케줄러 조정, 실험 로깅(WandB/Neptune), 조기 종료, Docker 실행 환경. 모델 학습, 학습 설정, 하이퍼파라미터 튜닝, 실험 실행, 분산학습 설정, 학습률 조정, 배치 크기, 에폭 설정 등 학습 관련 요청 시 반드시 이 스킬을 사용."
---

# Training -- Anti-Spoofing 학습/실험 관리 스킬

## 학습 설정 워크플로우

### 1. 기본 설정 (`arguments.py`)

```python
# 실험 설정
'epoch': 100
'batch_size': 240
'lr': 5e-5
'lr_min': 1e-6
'weight_decay': 0.0001
'T_0': 100  # CosineAnnealingWarmRestarts 주기
'rand_seed': 1

# 데이터 비율
'ratio_bona': 0.1
'ratio_spoof': 0.9

# OC-Softmax
'r_real': 0.9
'r_fake': 0.2
'alpha': 20.0
'loss_weight': [4, 3, 2, 1, 1]  # 다중 레벨 가중치
```

### 2. 학습 루프 구조 (`train.py`)

```
에폭 루프:
  1. bonafide 로더 + spoof 로더 교차 배치
  2. GPU 증강 적용
  3. Forward: model(x, label) → score, loss
  4. Backward: loss.backward() + optimizer.step()
  5. LR 스케줄링: CosineAnnealingWarmRestarts (에폭 단위)
  
  매 5에폭:
    - ASVspoof2021 DF 평가 → EER 계산
    - 최저 EER 모델 체크포인트 저장
    - 6회 연속 미개선 시 조기 종료
```

### 3. DDP 분산학습 (`main.py`)

- `torch.multiprocessing.spawn`으로 GPU 수만큼 프로세스 생성
- 최소 2 GPU 필요 (`CUDA_VISIBLE_DEVICES` 설정)
- 각 프로세스: DDP 초기화 → 모델 래핑 → 학습 → 정리

### 4. Docker 실행 환경

```bash
# 빌드
docker build -t env202305 ./environment/docker/

# 실행
docker run --gpus all -it --rm \
  -v /home/koo/DB2/datas:/data \
  -v /code/baseline/baseline_HMConformer_2:/code \
  -v /results:/results \
  env202305:latest

# 학습 시작
export CUDA_VISIBLE_DEVICES=0,1,2,3
python /code/main.py
```

### 5. 실험 관리

**WandB 로깅:**
- 프로젝트명: `arguments.py`의 `project` 필드
- 실험명: `name` 필드
- 메트릭: 에폭별 loss, EER (19LA, 21LA, 21DF)

**체크포인트:**
- 저장 경로: `/results/{name}.pth`
- 최저 EER 기준 갱신

### 6. 하이퍼파라미터 튜닝 가이드

| 파라미터 | 범위 | 영향 |
|---------|------|------|
| lr | 1e-5 ~ 1e-4 | 수렴 속도, 안정성 |
| batch_size | 120 ~ 480 | GPU 메모리, 일반화 |
| dropout_rate | 0.1 ~ 0.5 | 과적합 방지 |
| T_0 | 50 ~ 200 | LR 재시작 주기 |
| loss_weight | [4,3,2,1,1] 변형 | 다중 레벨 기여도 |
| r_real/r_fake | 마진 조정 | 결정 경계 |

### 7. 체크포인트
- [ ] DDP 초기화 정상 (NCCL backend)
- [ ] 배치 교차 로딩 (bonafide + spoof)
- [ ] GPU 증강 적용
- [ ] 스케줄러 에폭 단위 스텝
- [ ] 조기 종료 조건 정상
- [ ] WandB/Neptune 로깅 연결
