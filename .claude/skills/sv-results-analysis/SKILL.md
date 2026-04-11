---
name: sv-results-analysis
description: "Speaker Verification 실험 결과 분석 스킬. EER/minDCF 메트릭 수집, 실험 간 비교표 생성, ablation 결과 분석, 학습 곡선 해석, 논문용 LaTeX 테이블 생성을 수행한다. '결과 분석', '실험 비교', 'EER', 'minDCF', '비교표', 'ablation 분석', '논문 테이블', '성능 비교' 요청 시 사용."
---

# SV Results Analysis

Speaker Verification 실험 결과의 분석 및 비교를 담당하는 스킬.

## 데이터 소스

### 결과 디렉토리
```
/home/koo/code/environment/results/IRLAB/SSLBackend2025/
└── {experiment_name}/
    ├── scripts/          ← 학습 시 백업된 코드
    ├── checkpoints/      ← 모델 체크포인트
    ├── logs/             ← CSV 학습 로그
    └── neptune_run_id    ← Neptune 실험 ID
```

### 메트릭 수집 방법
1. **CSV 로그**: 학습 중 기록된 validation 메트릭 (step별 EER, minDCF)
2. **체크포인트 파일명**: 일부 실험은 파일명에 메트릭 인코딩
3. **Neptune 로그**: 원격 실험 추적 (접근 가능 시)
4. **콘솔 출력 로그**: 학습 과정의 stdout/stderr

## 메트릭 해석

### EER (Equal Error Rate)
- FAR = FRR인 지점의 오류율 (%)
- 낮을수록 좋다
- VoxCeleb1-O 기준 최신 SOTA: ~0.5% 수준
- 0.01%p 차이는 통계적으로 유의미하지 않을 수 있다

### minDCF (minimum Detection Cost Function)
- p_target=0.05, C_miss=1, C_fa=1 기준
- 0~1 범위, 낮을수록 좋다
- 운영 환경의 비용 함수를 반영하는 실용적 메트릭

### 테스트셋별 특성

| 테스트셋 | 특성 | 난이도 |
|---------|------|-------|
| VoxCeleb1-O | Original, 깨끗한 조건 | 쉬움 |
| VoxCeleb1-E | Extended, 더 많은 trial | 중간 |
| VoxCeleb1-H | Hard, 동일 국적 화자 | 어려움 |
| VOiCES | 실제 환경 노이즈/반향 | 어려움 |
| VoxSRC23 | 대회 테스트셋 | 어려움 |
| VCMix | 혼합 조건 | 중간~어려움 |

## 분석 워크플로우

### 실험 비교표 생성
1. 비교 대상 실험 디렉토리에서 최종 메트릭 수집
2. 동일 테스트셋 기준으로 정렬
3. 최고 성능에 **볼드** 표시
4. 파라미터 차이점 열 추가

```markdown
| 실험 | 변경점 | Vox1-O EER | Vox1-E EER | Vox1-H EER |
|------|-------|-----------|-----------|-----------|
| baseline | - | 1.23 | 1.45 | 2.67 |
| exp_A | margin=0.4 | **1.15** | 1.42 | **2.58** |
```

### Ablation 분석
1. 기준 실험(baseline) 대비 각 변형의 메트릭 변화량 계산
2. 변화량의 방향(개선/악화)과 크기를 해석
3. 교호작용 효과가 의심되면 명시한다

### 논문용 LaTeX 테이블
사용자가 요청하면 비교표를 LaTeX booktabs 형식으로 변환한다:

```latex
\begin{table}[t]
\centering
\caption{Comparison of experimental results}
\begin{tabular}{lccc}
\toprule
System & Vox1-O & Vox1-E & Vox1-H \\
\midrule
Baseline & 1.23 & 1.45 & 2.67 \\
Proposed & \textbf{1.15} & \textbf{1.42} & \textbf{2.58} \\
\bottomrule
\end{tabular}
\end{table}
```

## 주의사항
- 서로 다른 테스트셋 간 EER을 직접 비교하지 않는다 (난이도가 다르므로)
- 실험 조건(학습 steps, 데이터)이 다르면 공정한 비교가 아님을 명시한다
- 단일 시드 결과만으로 결론을 내리지 않도록 경고한다
