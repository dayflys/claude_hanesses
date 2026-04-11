---
name: data-pipeline
description: "Audio spoofing detection 데이터 파이프라인 스킬. 오디오 데이터 로딩, 전처리, 증강(augmentation), 새 데이터셋 통합, 프로토콜 파싱, 데이터 분석. ASVspoof2019/2021 데이터, FLAC/WAV 오디오, 데이터 증강 추가, 데이터 로더 수정, 새 데이터셋 추가, 데이터 분석, 프로토콜 파일, bonafide/spoof 비율 조정 등 데이터 관련 요청 시 반드시 이 스킬을 사용."
---

# Data Pipeline -- Anti-Spoofing 데이터 파이프라인 스킬

## 데이터 처리 워크플로우

### 1. 데이터 경로
- ASVspoof2019: `/home/koo/DB2/datas/ASVspoof2019/`
  - LA train: `LA/ASVspoof2019_LA_train/flac/`
  - LA dev: `LA/ASVspoof2019_LA_dev/flac/`
  - LA eval: `LA/ASVspoof2019_LA_eval/flac/`
  - 프로토콜: `LA/ASVspoof2019_LA_cm_protocols/`

### 2. 프로토콜 파일 형식
```
LA_0001 LA_E_1000001 - - bonafide
LA_0001 LA_E_1000002 - A01 spoof
```
컬럼: speaker_id, file_id, null, attack_type, label

### 3. 데이터 로더 패턴

**학습 데이터 (TrainSet):**
- bonafide/spoof를 분리 로더로 구성
- 비율 제어: `ratio_bona=0.1`, `ratio_spoof=0.9`
- 프레임 크롭: `num_train_frames * 160` 샘플 (400프레임 = 64,000 샘플)
- 짧은 발화: wrap 패딩

**평가 데이터 (EnrollmentSet):**
- 고정 프레임: `num_test_frames=400`
- 전체 발화 평가 (크롭 없이 패딩만)

### 4. 증강 기법

| 증강 | 파라미터 | 적용 위치 |
|------|---------|----------|
| ACN (AddColoredNoise) | SNR 10-40dB, freq_decay -2~2 | GPU 배치 |
| HPF (HighPassFilter) | cutoff 20-2400Hz | GPU 배치 |
| LPF (LowPassFilter) | cutoff 150-7500Hz | GPU 배치 |
| Gain | -15~+5dB | GPU 배치 |
| Speed Perturbation | [0.9, 0.95, 1.0, 1.05, 1.1] | GPU 배치 |
| RawBoost | SNR 10-40dB | GPU 배치 |
| Codec (MP3/M4A/OGG) | 선택적 | CPU |

### 5. 새 데이터셋 통합 시
1. 프로토콜 파싱 클래스 구현 (`dataset/` 디렉토리)
2. `Trial`/`Item` dataclass 패턴 따르기
3. `data_loaders.py`에 새 데이터셋용 로더 추가
4. `arguments.py`에 데이터 경로 추가

### 6. 체크포인트
- [ ] 16kHz 모노 오디오 기준
- [ ] DistributedSampler 사용
- [ ] bonafide/spoof 분리 로더 패턴
- [ ] 증강은 GPU 배치 단위
- [ ] 프로토콜 파싱 정확성 (레이블 매핑)
