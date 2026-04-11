---
name: evaluation
description: "Audio spoofing detection 평가/분석 스킬. EER 계산, t-DCF 메트릭, DET 곡선, 공격 유형별 성능 분석, 모델 비교, 오류 분석, 결과 시각화. 모델 평가, 성능 분석, EER 확인, 결과 비교, 공격별 분석, 벤치마크, 시각화, 그래프 등 평가 관련 요청 시 반드시 이 스킬을 사용."
---

# Evaluation -- Anti-Spoofing 평가/분석 스킬

## 평가 워크플로우

### 1. 평가 데이터셋

| 데이터셋 | 용도 | 특성 |
|---------|------|------|
| ASVspoof2019 LA (19LA) | 기본 평가 | 알려진 TTS/VC 공격 |
| ASVspoof2021 LA (21LA) | 채널 강건성 | 전화 채널 조건 |
| ASVspoof2021 DF (21DF) | **주 평가 기준** | deepfake 조건 |

### 2. EER 계산

```python
# scipy 기반 EER 계산
from scipy.optimize import brentq
from scipy.interpolate import interp1d
from sklearn.metrics import roc_curve

fpr, tpr, thresholds = roc_curve(labels, scores, pos_label=1)
eer = brentq(lambda x: 1. - x - interp1d(fpr, tpr)(x), 0., 1.)
threshold = interp1d(fpr, thresholds)(eer)
```

### 3. 평가 모드

**학습 중 평가 (`train.py`):**
- 5에폭 간격으로 자동 실행
- `EnrollmentSet`으로 고정 프레임 평가
- 최저 EER 모델 체크포인트 갱신

**독립 평가:**
- `main.py --eval --initial_model /results/model.pth`
- 모든 데이터셋에 대해 순차 평가

### 4. 공격 유형별 분석

프로토콜 파일의 attack_type 컬럼을 활용:
- ASVspoof2019 LA: A01~A06 (TTS), A07~A12 (VC)
  - A01~A06: 학습에 사용된 알려진 공격
  - A07~A19: 평가 전용 미지의 공격

공격별 EER을 개별 계산하여 모델의 약점을 파악한다.

### 5. 결과 보고 형식

```markdown
## 평가 결과

| 데이터셋 | EER (%) | 임계값 |
|---------|---------|-------|
| 19LA    | X.XX    | 0.XXX |
| 21LA    | X.XX    | 0.XXX |
| 21DF    | X.XX    | 0.XXX |

## 공격 유형별 분석 (선택)
| 공격 | 유형 | EER (%) |
|------|------|---------|
| A07  | VC   | X.XX    |
| ...  | ...  | ...     |
```

### 6. 시각화

- **DET 곡선**: FPR vs FNR (로그 스케일)
- **Score 분포**: bonafide vs spoof 점수 히스토그램
- **공격별 박스플롯**: 공격 유형별 점수 분포
- `matplotlib` 사용, 저장 경로: `environment/results/`

### 7. 체크포인트
- [ ] EER 계산 코드 정확성
- [ ] 평가 데이터셋 경로 확인
- [ ] 프로토콜 파싱 정확성
- [ ] 점수 파일 형식 일관성
- [ ] 시각화 저장 경로 확인
