# AI 서비스 개발 Mini-project

SKALA「AI 서비스 개발 Mini-project」(3일 과정) 조별 프로젝트 저장소입니다.
강의 제공 압축파일(`ai-practice.zip`)을 기반으로 폴더를 정리했습니다.

```
Vue.js 프론트 → Java 인증서버(:8080, JWT 발급) → FastAPI AI 서비스
                                              ├─ RAG 서비스(:8001)  PDF·웹 → FAISS → OpenAI
                                              └─ 음성 서비스(:8008) Whisper STT → GPT → TTS
```

---

## 📌 원본 압축파일에서 바뀐 점

> 압축파일을 받은 팀원은 아래 표를 보고 경로를 맞춰 주세요.

### 1. 폴더 이름 변경

교재 Day 3 `docker-compose.yml` 예시의 `build` 경로(`./auth-server`, `./rag-service`, `./frontend`)에 맞췄습니다.

| 원본 (`share/…`) | 변경 후 | 내용 |
|---|---|---|
| `workplace/` | **`auth-server/`** | Java 인증서버 (회원가입·로그인·JWT) |
| `rag-code-template_ext/` | **`rag-service/`** | External RAG 서비스 템플릿 (조별 직접 구현 대상) |
| `voice-ai_code/` | **`voice-service/`** | 음성 서비스 (STT → GPT → TTS, Day1 데모·확장 트랙) |
| `intro_main.html` 외 2개 (루트) | **`consoles/`** | RAG API 테스트용 콘솔 페이지 |
| — | `frontend/` (예정) | Vue.js 프론트 — 압축파일에 없음, 추후 추가 |

- 코드 안에서 폴더명을 경로로 참조하는 곳은 없어서 **이름 변경으로 동작이 바뀌지 않습니다.**
- Java 패키지명 `workplace`, Gradle 프로젝트명 `java-server`, 서비스 안의 파일명은 원본 그대로입니다.
  (주석에 `workplace % ./gradlew build`라고 나오면 `auth-server`로 읽으면 됩니다.)

### 2. 파일 수정

| 파일 | 변경 내용 | 이유 |
|---|---|---|
| `voice-service/requirements.txt` | UTF-16 → **UTF-8**, CRLF → LF (패키지 내용 동일) | UTF-16이면 `pip install -r` 이 실패할 수 있음 |

### 3. 새로 추가

| 파일 | 내용 |
|---|---|
| `.gitignore` | `.env`, 빌드 산출물(`.gradle/`, `build/`, `bin/`), `__pycache__`, 가상환경, `node_modules`, 음성 파일, `faiss_db/` 등 제외 |
| `.env.example` | 필요한 환경변수 목록 (값은 비워 둠) |
| `README.md` | 이 문서 |

### 4. 제외한 것

- `__pycache__/`, `.DS_Store` 는 복사하지 않았습니다.
- `rag-service/.env`(원본에 API 키 포함)와 `rag-service/faiss_db/`(미리 만든 벡터 인덱스)는 폴더에는 있지만 **Git에서는 무시**됩니다.
  인덱스를 팀원과 공유하려면 `.gitignore`의 `faiss_db/` 줄을 주석 처리하세요.

---

## 📁 폴더 구조

```
AI-mini/
├── auth-server/        Java 17 · Gradle · 순수 HttpServer · MariaDB · BCrypt · JWT
│   └── app/src/main/
│       ├── java/workplace/SimpleAuthServer.java
│       └── resources/web/   login.html · signup.html · app.js · style.css
├── rag-service/        FastAPI · LangChain 1.x · FAISS · OpenAI(gpt-4o)
│   ├── rag_external-web_pdf_closed_llm.py
│   ├── requirements-e.txt
│   └── faiss_db/{pdf,web}/
├── voice-service/      FastAPI · faster-whisper · OpenAI GPT(gpt-4o-mini) · TTS(tts-1, nova)
│   ├── app/{main.py, routers/, services/, static/}
│   ├── Dockerfile · docker-compose.yml
│   └── requirements.txt
├── consoles/           intro_main.html · external_index.html · internal_index.html
├── .env.example
└── .gitignore
```

---

## 🔑 환경변수

서비스마다 `.env`를 읽는 위치가 다릅니다. `.env.example`을 참고해 각 위치에 만들어 주세요.

| 서비스 | 위치 | 변수 |
|---|---|---|
| rag-service | `rag-service/.env` | `OPENAI_API_KEY`, `NAVER_CLIENT_ID`, `NAVER_CLIENT_SECRET`, (`ASYNC_DB_URL`) |
| voice-service | `voice-service/app/.env` | `OPENAI_API_KEY`, (`WHISPER_MODEL`, 기본 `small`) |
| auth-server | 셸 환경변수 (`export`) | `JWT_SECRET`, `DB_PASSWORD` |

> ⚠️ **JWT 비밀키 불일치**: `voice-service/app/main.py`는 `SECRET_KEY = "RANDOM_SECRET_KEY"`로 하드코딩되어 있고,
> Java는 `JWT_SECRET` 환경변수를 읽습니다. 두 값이 같아야 토큰 검증이 통과합니다.

---

## ▶️ 실행 방법

### 0. MariaDB 준비

인증서버는 `jdbc:mariadb://localhost:53301/sql_db` (user `root`)에 접속합니다. `users` 테이블을 미리 만들어 둡니다.

```sql
CREATE TABLE `users` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `username` varchar(100) NOT NULL,
  `password` varchar(255) NOT NULL,
  `created_at` timestamp NOT NULL DEFAULT current_timestamp(),
  `updated_at` timestamp NOT NULL DEFAULT current_timestamp() ON UPDATE current_timestamp(),
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_users_username` (`username`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 1. 인증서버 (:8080)

```bash
cd auth-server
export JWT_SECRET=... DB_PASSWORD=...
./gradlew build
./gradlew :app:run --no-configuration-cache
```

→ http://localhost:8080/login

### 2. RAG 서비스 (:8001)

```bash
cd rag-service
python -m venv .venv && source .venv/bin/activate
pip install -r requirements-e.txt
python -m uvicorn rag_external-web_pdf_closed_llm:app --port 8001 --reload
```

→ http://localhost:8001/docs (Swagger) · 엔드포인트: `/upload-paper`, `/upload-topic`, `/rag-query`

### 3. 음성 서비스 (:8008)

```bash
brew install ffmpeg
cd voice-service
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python -m uvicorn app.main:app --port 8008
```

→ 인증서버에서 로그인한 뒤 http://localhost:8008/index.html, http://localhost:8008/stt-tts.html
(Docker로 실행하려면 `docker compose up -d --build`)

### 4. 콘솔 페이지

`consoles/intro_main.html`을 브라우저로 열고, BASE_URL을 `http://localhost:8001`로 설정해 RAG API를 호출합니다.

---

## ✅ 조별로 해야 할 일 (교재 기준)

현재 `rag-service`는 **템플릿 상태**이고, 교재의 요구사항과 다음이 다릅니다.

| 항목 | 현재 템플릿 | 교재 목표 |
|---|---|---|
| 인증 | 없음 | `Depends(verify_jwt)` — 토큰 없으면 **401** |
| 요청 | `{prompt, source, top_k}` | `{question, source}` |
| 응답 | `{prompt, response}` | `{answer, sources, needs_review, handoff, latency_ms}` |
| 근거 없음 | LLM 호출 | 검색 0건이면 LLM 호출 안 함 → `"찾을 수 없습니다"`, `handoff: true` |
| 임베딩 | `OpenAIEmbeddings` | 교재 예시는 `BAAI/bge-m3` (선택) |

그 외:

- [ ] `frontend/` — Vue.js 화면 (로딩·오류·출처·handoff·401 처리)
- [ ] 루트 `docker-compose.yml` — `db`, `auth`, `rag`, `web` 일괄 기동
- [ ] 평가셋 20쌍 이상 (예외 질문 3개 이상 포함) + 개선 전후 비교
- [ ] 지연·토큰 로그 미들웨어
- [ ] (확장 +5) 음성 연동 / Internal RAG / Fine-tuning·Agent 중 1개

### 체크포인트

| # | 시점 | 내용 |
|---|---|---|
| ① | Day1 | 1장 기획서 (사용자 가치·데이터·아키텍처/API·역할 분담) |
| ② | Day1 | 로그인 → JWT → `/rag-query` 호출 1회 성공 |
| ③ | Day2 | Swagger로 API 시연 (401/200, 근거 답변, 예외 질문) |
| ④ | Day2 | End-to-End 데모 |
| ⑤ | Day3 | 최종 제출물 (데모·아키텍처·발표·README·평가셋) |
