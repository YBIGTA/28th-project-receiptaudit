# 🧾 Transparent-Audit

**조직 회계 투명성을 위한 AI 기반 전자동 영수증 감사 시스템**
_(YBIGTA 28기 신기풀 프로젝트)_

---

## 🌟 프로젝트 개요 (Overview)

현재 학생 사회나 자치 조직의 회계 감사는 100% 수동으로 이루어지고 있습니다. 규정에 맞지 않는 지출 내역이나 총액 오차를 사람이 일일이 대조하며 찾아내야 하는 **'시맨틱 인식의 부재'**는 시스템적 병목으로 작용하여 고노동과 높은 운영 비용을 초래합니다.

**Transparent-Audit**은 이러한 기존의 '수동 입력 및 수동 검토' 방식을 **'자동 추출과 시맨틱 감사'**로 패러다임 시프트(Paradigm Shift)하기 위해 기획되었습니다. 영수증 이미지를 분석해 데이터를 자동 추출하고, RAG(Retrieval-Augmented Generation) 기반의 AI 에이전트가 복잡한 조직 규정을 바탕으로 지능적인 감사를 수행하여 위반 항목의 누락률을 획기적으로 개선합니다.

---

## 🏗 시스템 아키텍처 (Architecture)

클라우드 환경에서 즉시 배포 및 확장이 가능한 풀스택(E2E) 아키텍처로 구축되었습니다.

- **Frontend**: React + TypeScript (초기 Streamlit 프토토타입에서 안정성 강화를 위해 전환)
- **Backend**: FastAPI, PostgreSQL
- **AI/ML**: Upstage Solar LLM & Embedding, PaddleOCR, ChromaDB
- **Infra**: AWS EC2, S3, Docker

---

## ⚙️ 핵심 작동 방식 (Core Modules)

시스템은 크게 세 가지 파이프라인을 거쳐 동작합니다:

### 1. 시각 해석 엔진 (OCR Engine)

영수증 이미지에서 정밀하게 텍스트를 추출하기 위한 전처리 및 OCR 레이어입니다.

- **전처리 프로세스**: 이미지 업로드 시 OpenCV를 활용해 디코딩 및 그레이스케일 변환을 수행합니다. 특징 추출을 방해할 수 있는 과도한 노이즈 제거(이진화 등)는 과감히 제외하고, **Deskewing(기울기 보정, Affine Transformation)**만을 적용하여 시스템의 Robustness(강건성)를 확보했습니다.
- **텍스트 예측 및 Fallback**: PaddleOCR을 기반으로 텍스트를 인식합니다. 이때 신뢰도(Confidence)가 0.6 미만일 경우 이미지가 뒤집힌 것으로 판단, 180도 회전 후 재시도하는 자체 Fallback 로직을 구현했습니다.
- **성능 측정**: 악조건(30~45도 회전)의 이미지에서도 전처리 적용 후 성능이 약 두 배(정확도 최대 58%)로 상승했습니다. 모델 성능 지표 또한 단순 일치(Exact Match) 대신 유사도 기반 메트릭으로 변경하여 실제 서비스 환경에서의 명칭 오차를 합리적으로 반영했습니다.

### 2. 구조화 계층 (NLP & Rule Layer)

분산된 OCR 인식 텍스트 라인들을 구조화된 정형 데이터로 변환합니다.

- **속성 병렬 추출**: 텍스트 정규화 및 불용어 필터링을 거친 후, 정규식과 키워드 매칭을 통해 5가지 주요 속성(총액, 가맹점, 주소, 일시, 품목 명세)을 병렬적으로 추출해 JSON 포맷으로 통합합니다.
- _(엔지니어링 결정)_: 텍스트 구조화 교정 단계에 LLM을 테스트한 결과, 교정이 반복될수록 과교정 및 환각(Hallucination) 현상이 발생해 최종 정확도가 저하(66.6% → 55.3%)됨을 확인했습니다. 따라서 파이프라인에서 불필요한 LLM 교정을 배제하여 효율과 정확도를 모두 잡았습니다.

### 3. AI 감사 브레인 (Agent & RAG)

단순 키워드 매칭을 넘어서, 추출된 JSON 데이터의 속성을 논리적, 의미론적으로 판단해 감사를 수행합니다.

- **지식 파이프라인 (RAG)**: 조직의 재무 규정(PDF/TXT)을 Upstage Solar Embedding을 통해 ChromaDB에 실시간 벡터화하여 지식 베이스를 구축합니다.
- **에이전트 판단**:
  1. 네이버 Local API를 연동해 가맹점 업종의 진위를 일차적으로 검증합니다.
  2. 벡터 유사도 검색 및 LLM Re-ranker로 영수증 항목에 대한 최적의 규정을 검색합니다.
  3. 판단 모델(Solar LLM)에 프롬프트를 전송, "참이슬"을 "주류"로 시맨틱하게 추론하는 등의 논리적 위반 판단을 수행합니다.
- **Human-in-the-loop**: 위반 사유와 근거가 담긴 결과 JSON을 바탕으로, 가변형 테이블 UI를 통해 최종적으로 인간 창구가 승인 및 확정을 내리는 구조로 안전성을 보완합니다.

---

## 🛠 주요 문제 해결 (Troubleshooting)

### C++ 라이브러리 기반 메모리 누수 극복 (OOM 최적화)

- **이슈**: C++ 기반인 PaddleOCR 특성상, 반복된 영수증 예측 작업에서 지속적인 Memory-leaking이 발생하여 AWS EC2 인스턴스에서 OOM(Out of Memory)으로 처리 프로세스가 멈추는 병목 현상이 발생했습니다.
- **해결**: 집중적인 RAM 사용량 프로파일링 및 최적화 작업을 수행하고, 컨테이너 환경의 메모리 관리 기법을 통해 현상황에서는 OOM 없이 EC2 환경에서도 매우 안정적으로 실행되도록 엔지니어링 퍼포먼스를 높였습니다.

---

## 🚀 시작하기 (Getting Started)

### 1. 환경 설정

- Python 3.9+ 및 Node.js 필요

```bash
# 가상환경 생성 및 활성화
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate

# 백엔드 의존성 설치
pip install -r requirements.txt
```

### 2. 환경 변수 설정

`.env.example` 파일을 복사해 `.env`를 생성하고, Upstage API Key, Naver API Key 등 필수 정보를 입력합니다.

### 3. RAG 데이터 준비

규정집 문서를 `data/raw/` 경로에 배치한 후 백터 DB를 구축합니다.

```bash
python -m core.rag_engine.ingest
```

### 4. 서비스 실행

**Docker Compose로 전체 실행 (권장)**:

```bash
docker-compose up --build
```

**로컬 개별 실행**:

- **windows**:
  ```bash
  sh dev.sh
  ```
- **macOS**:
  ```bash
  zsh dev.sh
  ```
