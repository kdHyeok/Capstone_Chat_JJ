# Chat JJ 전체 시스템 구조

이 문서는 Chat JJ 팀 프로젝트의 프론트엔드, 백엔드, AI 서버, 데이터 수집 파이프라인, 배포 자료가 어떻게 분리되고 연결되는지 정리합니다.

## 1. 전체 아키텍처

![Chat JJ 전체 시스템 구조](assets/cj-system-overview.svg)

전체 서비스는 두 흐름으로 나뉩니다.

1. 실시간 질문 처리 경로
   - 사용자가 Flutter 앱에서 질문 입력
   - Spring Boot 백엔드가 요청 수신
   - 백엔드가 AI 서버로 질문 중계
   - AI 서버가 RAG, 룰 기반 처리, Ollama LLM 호출을 조합해 답변 생성
   - 백엔드가 답변과 채팅 이력을 저장한 뒤 앱에 반환

2. 지식 데이터 공급 경로
   - 크롤링 노트북이 전주대학교 공지 데이터를 수집
   - 본문, 이미지, OCR, 마감일, 카테고리 정보를 정리
   - 최종 CSV가 AI 서버의 검색/RAG 원천 데이터로 사용
   - ChromaDB와 카테고리 분류 로직이 질문과 관련된 공지를 좁히는 데 사용

## 2. 레포별 역할

| 레포 | 담당 영역 | 핵심 역할 |
|---|---|---|
| `CJ_Front` | Flutter 앱 | 로그인, 회원가입, 챗봇 화면, 문의 화면, 다국어 UI |
| `CJ_Back` | Spring Boot API | 인증, JWT, MySQL/Redis 저장, 채팅 API, AI 서버 중계 |
| `CJ_AI` | Quart AI 서버 | RAG, SBERT 유사도, ChromaDB 검색, Ollama 호출, 다국어 응답 |
| `CJ_Scraping` | 데이터 수집 파이프라인 | 전주대 공지 크롤링, 본문/OCR/마감일 추출, 카테고리 분류 CSV 생성 |
| `CJ_Server` | 서버/배포 자료 | AWS 서버 접속, 배포, 운영 환경 관련 자료 |

## 3. 질문 처리 시퀀스

![Chat JJ 질문 처리 시퀀스](assets/cj-runtime-sequence.svg)

질문 처리 흐름은 다음과 같습니다.

```text
사용자
→ CJ_Front
→ CJ_Back /api/chat 또는 /api/loginChat
→ CJ_AI /chat 또는 /loginChat
→ ChromaDB/CSV 검색
→ Ollama LLM 호출
→ CJ_AI response 반환
→ CJ_Back DB 저장 및 응답 반환
→ CJ_Front 화면 출력
```

비로그인 사용자는 질문만 전달됩니다.

```json
{
  "question": "장학금 공지 알려줘"
}
```

로그인 사용자는 개인화에 필요한 정보가 함께 전달됩니다.

```json
{
  "username": "user01",
  "question": "내 학과랑 관련된 공지 알려줘",
  "major": "컴퓨터공학과",
  "grade": 3,
  "language": "한국어"
}
```

이 정보는 AI 서버의 프롬프트에 반영되어 학년, 학과, 언어 기반 답변을 만드는 데 사용됩니다.

## 4. 데이터 파이프라인

![Chat JJ 공지 데이터 파이프라인](assets/cj-data-pipeline.svg)

크롤링 파이프라인은 챗봇이 검색할 수 있는 공지 데이터를 만드는 역할을 합니다.

| 단계 | 처리 내용 |
|---|---|
| 공지 목록 크롤링 | 통합공지/학사공지 목록에서 제목, 부서, 등록일, 상세 링크 수집 |
| 상세 본문 수집 | 상세 페이지 본문, 첨부 링크, 메타데이터 정리 |
| 이미지/OCR 보강 | 이미지 중심 공지에서 이미지 링크 추출 후 OCR로 텍스트 보완 |
| 일정·마감일 추출 | 정규표현식 기반 날짜 후보 추출, 모집 상태 정리 |
| 카테고리 분류 | 제목, 본문, 부서 기준으로 분류 축 점수화 |

최종 산출물은 다음 두 파일입니다.

| 파일 | 역할 |
|---|---|
| `04_통합공지_카테고리분류_결과.csv` | 통합공지 기반 RAG/검색 원천 데이터 |
| `05_학사공지_카테고리분류_결과.csv` | 학사공지 기반 RAG/검색 원천 데이터 |

## 5. 백엔드와 AI 서버 연결

백엔드는 Flutter 앱에서 받은 요청을 AI 서버로 중계합니다.

```text
CJ_Front → CJ_Back
POST /api/chat
POST /api/loginChat
```

```text
CJ_Back → CJ_AI
POST http://localhost:5001/chat
POST http://localhost:5001/loginChat
```

AI 서버는 내부에서 Ollama를 호출합니다.

```text
CJ_AI → Ollama
POST http://localhost:11434/api/chat
model = gemma3:4b
```

AI 서버는 CSV와 ChromaDB에서 관련 공지를 찾고, 검색 결과를 LLM 프롬프트의 context로 넣어 답변을 생성합니다.

## 6. 데이터 파이프라인의 역할

크롤링 파트는 단순 수집이 아니라 AI 답변 품질을 좌우하는 지식 기반입니다.

- 최신 공지를 검색 대상으로 제공
- 본문과 이미지 기반 정보를 함께 보강
- 마감일과 모집 상태를 추출해 질문 응답에 활용
- 카테고리 점수화로 질문 의도와 관련된 공지를 우선 검색
- RAG와 SBERT 기반 검색이 사용할 수 있는 구조화 데이터 제공
