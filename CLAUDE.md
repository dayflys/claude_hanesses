# Spoofing Detection Project

Audio deepfake/spoofing detection 프로젝트. ASVspoof 2019/2021 데이터셋 기반, HM-Conformer 중심의 anti-spoofing 모델 연구.

## 프로젝트 구조
- `code/baseline/` — 모델 baseline 구현 (HMConformer 1~4, AASIST 1~2, LCNN, SEResNet34)
- `environment/` — Docker, 실행 스크립트, 실험 결과, WandB 로그
- 데이터: `/home/koo/DB2/datas/ASVspoof2019/`

## 하네스: Spoofing Detection

**목표:** Anti-spoofing 모델 연구의 모든 단계(모델 설계, 데이터 처리, 학습, 평가)를 전문 에이전트로 지원

**트리거:** spoofing detection 관련 복합 작업 요청 시 `spoofing-orchestrator` 스킬을 사용하라. 모델 구현, 데이터 파이프라인, 학습 설정, 성능 평가, 코드 검증 등. 단순 질문은 직접 응답 가능.

**에이전트:** architect(모델), data-engineer(데이터), trainer(학습), evaluator(평가), qa(검증)

**변경 이력:**
| 날짜 | 변경 내용 | 대상 | 사유 |
|------|----------|------|------|
| 2026-04-11 | 초기 구성 | 전체 | 하네스 신규 구축 |
