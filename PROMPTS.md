# PROMPTS Log — HEALNet 재현 및 TCGA 오믹 단독 평가

> AI Coding Tool을 활용한 멀티모달 생존 분석 모델(HEALNet) 재현 및 분석 과정의 프롬프트 로그

## 메타 정보

| 항목 | 내용 |
|---|---|
| 과목 | 스마트팩토리캡스톤디자인1 |
| 학기 | 2026학년도 1학기 |
| 작성자 | 차호준 (학번 기입) |
| 대상 논문 | Hemker, Simidjievski, Jamnik, "HEALNet: Multimodal Fusion for Heterogeneous Biomedical Data", NeurIPS 2024 |
| 구현 범위 | (1) 합성 데이터 기반 HEALNet 구조 재현 (tutorial), (2) TCGA 오믹 단독 4개 데이터셋(BLCA/BRCA/KIRP/UCEC) 5-fold 평가 재현 |
| 사용 AI 도구 | Anthropic Claude (대화형 AI assistant) |
| 작업 기간 | 2026년 5월 |
| 결과 산출물 | https://github.com/horaijun/HEALNetTest , https://www.kaggle.com/code/copolymer/healnet-final |

---

## 작업 흐름 요약

본 과제는 총 6단계로 진행되었다.

1. **모델/논문 선정** — 멀티모달 생존 분석 모델 후보 검토 후 HEALNet 선정
2. **튜토리얼 노트북 구현** — 수백 GB WSI 없이 합성 데이터로 HEALNet 전 구조 실행 (healnet-final-tutorial.ipynb)
3. **오믹 실험 노트북 구현** — 실제 TCGA 오믹 데이터로 4개 데이터셋 5-fold 학습 (healnet-final.ipynb)
4. **결과 분석** — 5-fold 평균 c-Index를 논문 기준값과 비교
5. **발표자료·보고서 작성** — PPT, 발표 대본, 구현 보고서 산출
6. **배포 정리** — GitHub repo + Kaggle 노트북 공개 (비로그인/Colab 실행 호환)

각 단계의 주요 프롬프트 의도와 학습 내용은 다음과 같다.

---

## Phase 1 — 모델/논문 선정

### 1.1 선정 프롬프트의 의도

멀티모달 생존 분석(survival analysis) 영역에서, 개인 환경(GPU 1장, Kaggle/Colab 무료 자원)에서 재현 가능하면서도 코드가 공개된 모델 식별을 요청.

### 1.2 검토 결과

| 후보 | 결정 | 사유 |
|---|---|---|
| 단순 concat / late-fusion baseline | 탈락 | 모달리티 고유 구조 보존 불가, 누락 모달리티 대응 불가 |
| Perceiver IO 직접 구현 | 탈락 | 생존 분석 파이프라인·데이터 로더 직접 구현 부담 |
| **HEALNet** (Hemker et al., 2024) | **선정** | 공식 코드 공개, TCGA 평가 재현 가능, 누락 모달리티 강건성·해석성 강점 |

### 1.3 학습 사항

- 모델 선정 기준은 단일 성능이 아닌 (1) 코드 공개 여부, (2) 평가 데이터셋 접근성, (3) 개인 자원 제약, (4) 재현 난이도의 4축으로 평가해야 한다.
- HEALNet의 핵심은 **공유 잠재 배열(Shared Latent Array)** 에 각 모달리티가 전용 교차 어텐션으로 정보를 주입하는 구조이며, 이 구조 덕분에 특정 모달리티가 없어도 해당 단계를 건너뛰는 것만으로 추론이 가능하다.

---

## Phase 2 — 튜토리얼 노트북 구현 (healnet-final-tutorial.ipynb)

### 2.1 목표

WSI 원본 데이터(데이터셋당 수백 GB)를 다운로드하지 않고도, 실제 TCGA-KIRP와 동일한 shape의 합성 데이터로 HEALNet 전 구조가 정상 동작함을 검증하는 노트북 작성.

### 2.2 셀별 구현 프롬프트

#### Step 1 — 설치 및 import

**프롬프트 의도**: konst-int-i/healnet 리포지토리를 클론하고 `pip install -e .`로 editable 설치하는 셀 작성.

#### Step 2 — 합성 데이터 생성

**프롬프트 의도**: 실제 KIRP와 동일 shape의 랜덤 텐서 생성 셀 작성.

- 오믹 텐서 `(200, 1, 1587)` — N_OMIC=1587은 실제 KIRP 유전자 피처 수
- WSI 패치 피처 `(200, 256, 1024)` — PATCH_DIM=1024는 ResNet50 출력 차원
- 생존 라벨 3종: `censorship`, `event_time`, `y_disc` (N_BINS=4)

#### Step 3 — Dataset / DataLoader

**프롬프트 의도**: 멀티모달 데이터를 묶는 Dataset과 Train/Val/Test 분할 셀 작성.

- 200 샘플 → Train 70% / Val 15% / Test 15% (140 / 30 / 30)

#### Step 4 — 모델 초기화 및 학습

**프롬프트 의도**: 논문 KIRP 하이퍼파라미터로 HealNet을 초기화하고 학습 루프를 작성.

- `l_c=25, l_d=119, depth=5, x_heads=1, l_heads=8, cross_dim_head=27, latent_dim_head=113, attn_dropout=0.32, ff_dropout=0.05, snn=True`
- 총 파라미터 약 5,527,323개 / CrossEntropyLoss + Adam(lr=1e-3), 10 epoch

#### Step 5 — 누락 모달리티 시연

**프롬프트 의도**: 논문 Table 2의 누락 모달리티 강건성을 직접 보여주는 3가지 추론 시나리오 작성.

- `model([omic, wsi])` / `model([omic, None])` / `model([None, wsi])`

### 2.3 발생한 호환성 이슈

#### 이슈 1 — HealNet 파라미터명 불일치

**증상**: `TypeError: HealNet.__init__() got an unexpected keyword argument 'cross_heads'`

**프롬프트 의도**: 네 번째 셀의 TypeError 원인 진단 + 수정 요청.

**해결**: `healnet/models/healnet.py`의 실제 파라미터명은 `cross_heads`가 아닌 `x_heads`, `latent_heads`가 아닌 `l_heads`. 모델 초기화 인자를 교정.

**Lesson**: 논문 본문/그림의 표기와 실제 구현체의 인자명은 다를 수 있으므로, 클래스 시그니처를 직접 확인 후 사용해야 한다.

#### 이슈 2 — 미사용 sksurv import

**증상**: `ModuleNotFoundError: No module named 'sksurv'`

**해결**: 셀 4 상단의 미사용 import `from sksurv.metrics import concordance_index_censored` 삭제. 튜토리얼 목적에 c-Index 계산은 불필요하므로 의존성 자체를 제거.

**Lesson**: 원본 코드에서 복사한 import 중 실제로 호출되지 않는 것은 불필요한 의존성을 유발하므로 제거한다.

#### 이슈 3 — openslide 환경 의존성

**증상**: `ModuleNotFoundError: No module named 'openslide'` (Kaggle에서는 정상, 로컬에서 실패)

**프롬프트 의도**: 동일 코드가 Kaggle에서는 되는데 로컬에서 안 되는 이유 진단.

**해결**:
- `healnet/etl/loaders.py` 최상단의 `from openslide import OpenSlide`를 try/except로 감싸는 패치를 셀 0에 삽입 (설치 직후 파일 직접 수정).
- `from healnet.etl import MMDataset` 대신, 동일 기능의 MMDataset 클래스를 셀 2에 직접 정의하여 `healnet.etl` import 자체를 우회.

**Lesson**: Kaggle은 openslide가 기본 설치되어 있으나 Windows·Colab에는 없다. 환경별로 다르게 동작하는 근본 원인은 "외부 코드의 무조건적 전역 import"이며, 패치 또는 우회로 의존성을 끊어야 한다.

### 2.4 학습 노트

- 튜토리얼의 가치는 단순 실행이 아니라, 데이터 없이도 **모델 구조 전체(교차 어텐션 + 잠재 배열 + 누락 모달리티 처리)** 가 동작함을 검증하는 데 있다.
- 합성 데이터의 shape를 실제 데이터셋과 동일하게 맞추면, 실제 데이터 투입 시 구조 변경 없이 그대로 확장 가능하다.

---

## Phase 3 — 오믹 실험 노트북 구현 (healnet-final.ipynb)

### 3.1 목표

실제 TCGA 오믹 데이터로 BLCA·BRCA·KIRP·UCEC 4개 암종에 대해 5-fold 교차검증을 수행하고, 논문 Appendix B Table 3(오믹 단독)과 c-Index를 비교.

### 3.2 셀별 구현 프롬프트

#### Step 1 — 환경 설치

**프롬프트 의도**: HEALNet 클론 + `scikit-survival, matplotlib, wandb` 설치 셀 작성.

#### Step 2 — gdc-client 설치 (선택)

**프롬프트 의도**: TCGA WSI 공식 다운로드 도구(gdc-client v1.6.1 Ubuntu x64) 설치 셀 작성. 오믹 단독 실험에서는 미사용이나 향후 멀티모달 확장을 위해 준비.

#### Step 3 — 오믹 데이터 수신

**프롬프트 의도**: `git lfs pull`로 4개 데이터셋 오믹 CSV 수신 셀 작성.

#### Step 4 — 설정 수정 + loaders.py 패치

**프롬프트 의도**: 잘못된 하이퍼파라미터 교정과 오믹 단독 실행 호환 패치를 한 번에 적용하는 셀 작성.

- `best_hyperparams.yml`의 BRCA/KIRP/UCEC에 잘못 입력된 Perceiver 기본값 → 논문 HEALNet 값(`num_latents=25, latent_dim=119`)으로 교체
- `loaders.py` 6개 항목 패치: (0) openslide try/except, (1) slide_ids FileNotFoundError 방어, (2) sample_slide IndexError 방어, (3) filter_overlap 전체 필터링 버그 수정, (4) get_resize_dims None 폴백, (5) get_info None 폴백

#### Step 5 — 학습 실행

**프롬프트 의도**: 4개 데이터셋 순차 학습 셀 작성.

- `sources=[omic]`, `n_folds=5`, `epochs=50`, `early_stopping=True(patience=5)`, `survival.loss=nll`

#### Step 6 — 결과 요약

**프롬프트 의도**: 로그에서 c-Index를 파싱해 5-fold 평균을 계산하고 논문 기준값과 비교하는 셀 작성.

### 3.3 발생한 호환성 이슈

#### 이슈 4 — Kaggle 비로그인 환경 git clone 실패

**증상**: 첫 번째 셀 실행 후 `/kaggle/working/healnet` 경로 없음.

**프롬프트 의도**: 비로그인 Kaggle에서 첫 셀이 실패하는 원인 진단. 이어서 "그냥 폴더를 먼저 만들면 되지 않냐"는 해결 방향 제시.

**해결**: 비로그인 Kaggle은 인터넷이 차단되어 `!git clone`이 실패하지만 Python 예외는 발생하지 않아, 이후 `os.chdir`이 없는 경로에서 실패. `os.makedirs(HEALNET_DIR, exist_ok=True)`로 디렉토리를 먼저 만들고 `os.listdir`이 비어 있을 때만 clone하도록 변경.

**Lesson**: Jupyter의 `!` 셸 명령은 실패해도 Python 예외를 던지지 않는다. 후속 Python 코드가 셸 명령 성공을 전제하면 안 된다.

#### 이슈 5 — openslide 오류 재발

**증상**: `ModuleNotFoundError: No module named 'openslide'` (healnet-final에서 다시 발생)

**해결**: 튜토리얼의 패치는 별도 파일이라 적용되지 않았음. 셀 4(패치 셀)의 loaders.py 패치 맨 앞에 openslide try/except 패치(Patch 0)를 추가하여 기존 5개 패치보다 먼저 실행.

**Lesson**: 동일 원인의 버그라도 노트북마다 독립적으로 패치를 적용해야 한다.

#### 이슈 6 — Windows에서 bash 전용 문법 실행 불가

**증상**: 셀 2에서 `'.'은(는) 인식이 안 되는...` 출력 후, 이후 학습이 전혀 시작되지 않음. (Colab은 정상)

**프롬프트 의도**: Colab에서는 되는데 VS Code(Windows)에서는 안 되는 이유 진단 (파일 수정 없이).

**해결(진단)**: bash 전용 문법 3가지가 Windows에서 미지원:
- `./gdc-client` — 상대 경로 바이너리 실행 미지원
- `WANDB_MODE=disabled python ...` — 인라인 환경변수 문법
- `| tee train_log.txt` — tee 명령어 부재

결론: healnet-final.ipynb는 Colab/Linux 전용으로 안내. healnet-final-tutorial.ipynb는 bash 문법을 쓰지 않으므로 Windows 로컬에서도 동작.

**Lesson**: 노트북이 셸 명령을 사용하면 OS 종속성이 생긴다. 크로스 플랫폼이 필요하면 셸 대신 Python subprocess/os 모듈로 작성해야 한다.

### 3.4 학습 노트

- `best_hyperparams.yml`의 오입력 교정이 재현성 향상의 핵심이었다. 공개 리포지토리의 설정 파일이라도 논문 값과 대조 검증이 필요하다.
- 오믹 단독(uni-modal)이라도 HEALNet의 교차 어텐션·잠재 배열 구조를 그대로 사용하므로, WSI 추가만으로 멀티모달 확장이 가능하다.

---

## Phase 4 — 결과 분석

5-fold 교차검증 평균 c-Index를 논문 Appendix B Table 3(오믹 단독)과 비교.

| 데이터셋 | 실험 결과 | 논문 기준 | 차이 |
|---|---|---|---|
| BLCA | 0.6423 | 0.606 | +0.036 |
| BRCA | 0.5467 | 0.556 | −0.009 |
| KIRP | 0.7387 | 0.771 | −0.032 |
| UCEC | 0.5350 | 0.509 | +0.026 |

→ 4개 데이터셋 모두 논문 기준값과 ±0.04 이내로 수렴. fold 구성의 무작위성으로 실행마다 결과가 소폭 달라질 수 있으며, 논문도 5회 반복의 평균·표준편차로 보고한다.

---

## Phase 5 — 발표자료·보고서 산출물

| 산출물 | 내용 |
|---|---|
| HEALNet_차호준_오픈소스.pptx | 논문 요약 + 구조 재현(튜토리얼) + 오믹 적용 (17장) |
| HEALNet_차호준_오픈소스_대본.pptx | 슬라이드 노트에 20~30분 분량 발표 대본 삽입 |
| HEALNet_구현보고서.docx | 순차적 구현 매뉴얼 + 프롬프트 + 분석 (약 3페이지) |
| PROMPTS.md | 본 파일 |

---

## 학습 정리

### 멀티모달 생존 분석 영역

- HEALNet의 공유 잠재 배열 + 모달리티별 교차 어텐션 구조는, 모달리티 결측 시 해당 단계를 건너뛰는 것만으로 강건성을 확보한다(논문 Table 2를 코드로 직접 확인).
- 생존 분석 평가지표 c-Index는 fold 분할 무작위성에 민감하므로, 단일 실행이 아닌 다회 반복 평균으로 보고해야 한다.

### AI Coding Tool 사용 경험

**장점**:
- 환경 의존성 이슈 진단(openslide, Kaggle 비로그인, Windows bash 문법)에서 시간 절약 효과가 컸다. 단독 디버깅 시 오래 걸렸을 원인이 단일 진단으로 좁혀졌다.
- "같은 코드가 환경 A에서는 되고 B에서는 안 되는" 유형의 문제에서, 환경 간 차이(기본 설치 패키지, 셸 종류)를 빠르게 짚어냈다.

**한계**:
- 외부 라이브러리의 실제 인자명·최신 버전 동작은 사용자가 직접 검증해야 한다(예: x_heads vs cross_heads).
- AI가 제시한 셸 명령이 OS에 따라 실패할 수 있어, traceback 해석에 사용자 역량이 필요하다.

**효율적 활용 방식**:
- 큰 작업을 셀(step) 단위로 분해하여 각 결과를 검증하고 다음으로 진행.
- 오류 메시지를 그대로 전달하면 원인 진단이 빨라지며, "파일을 수정하지 마"처럼 행동 범위를 명시하면 분석만 받을 수 있다.

### 환경 호환성 일반 lesson

1. 외부 코드의 무조건적 전역 import(openslide)는 환경별 실행 가능성을 좌우한다 → try/except 또는 우회.
2. Jupyter `!` 셸 명령은 실패해도 Python 예외를 던지지 않는다 → 후속 코드가 성공을 전제하면 안 된다.
3. 노트북이 셸 명령(bash 문법)을 쓰면 OS 종속성이 생긴다 → 크로스 플랫폼은 Python 모듈로.
4. 공개 리포지토리의 설정 파일이라도 논문 값과 대조 검증이 필요하다.

---

## 향후 작업

1. **멀티모달 확장** — 현재 오믹 단독 결과에 WSI를 추가하여 멀티모달 c-Index 비교 (구조는 그대로 사용 가능).
2. **healnet-final.ipynb 크로스 플랫폼화** — 셀 4의 bash 문법을 Python subprocess로 재작성하여 Windows 로컬 실행 지원.
3. **추가 암종 확장** — 논문의 나머지 TCGA 데이터셋으로 평가 범위 확대.

---

## 부록 — 재현 매뉴얼 (단계별)

### A. 튜토리얼 (healnet-final-tutorial.ipynb) — Windows 로컬 / Kaggle 비로그인 가능

```python
# 1. 클론 + 설치 + openslide 패치
import os, sys
HEALNET_DIR = "/kaggle/working/healnet"   # 로컬은 적절한 경로로 변경
if not os.path.exists(HEALNET_DIR):
    os.system(f"git clone https://github.com/konst-int-i/healnet.git {HEALNET_DIR}")
os.chdir(HEALNET_DIR)
os.system(f"{sys.executable} -m pip install -e . -q")

loaders_path = os.path.join(HEALNET_DIR, "healnet/etl/loaders.py")
with open(loaders_path) as f: src = f.read()
src = src.replace(
    "from openslide import OpenSlide",
    "try:\n    from openslide import OpenSlide\nexcept ImportError:\n    OpenSlide = None")
with open(loaders_path, "w") as f: f.write(src)

# 2~5. 합성 데이터 생성 → Dataset 구성 → 모델 학습 → 누락 모달리티 시연
#      (노트북 셀 1~4 순차 실행)
```

### B. 오믹 실험 (healnet-final.ipynb) — Google Colab / Linux 권장

```python
# 1. 디렉토리 선생성 후 클론 (Kaggle 비로그인 대비)
import os
HEALNET_DIR = "/kaggle/working/healnet"
os.makedirs(HEALNET_DIR, exist_ok=True)
if not os.listdir(HEALNET_DIR):
    os.system(f"git clone https://github.com/konst-int-i/healnet.git {HEALNET_DIR}")
os.chdir(HEALNET_DIR)
```

```bash
# 2. 패키지 설치
pip install -e . -q
pip install scikit-survival matplotlib wandb

# 3. gdc-client (선택)
wget -q https://gdc.cancer.gov/files/public/file/gdc-client_v1.6.1_Ubuntu_x64.zip
unzip -o gdc-client_v1.6.1_Ubuntu_x64.zip && chmod +x gdc-client

# 4. 오믹 데이터 수신
git lfs install && git lfs pull
```

```python
# 5. best_hyperparams.yml 교정 + loaders.py 6개 패치 (노트북 셀 4)
#    - num_latents=25, latent_dim=119
#    - openslide / slide_ids / sample_slide / filter_overlap /
#      get_resize_dims / get_info
```

```bash
# 6. 4개 데이터셋 순차 학습 (Colab/Linux)
for ds in blca brca kirp ucec; do
    WANDB_MODE=disabled python healnet/main.py \
        --config config/main_gpu.yml | tee train_log_${ds}.txt
done
```

> Kaggle에서 실행 시 Internet 사용 설정 + 로그인 필요.
> Windows 로컬은 위 bash 문법(인라인 환경변수, tee, ./바이너리) 미지원 → Colab 사용.

---

마지막 갱신: 2026년 5월
