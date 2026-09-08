# LaPoem 챗봇 서버 - 책 이야기 상대 "스텔라"

도서 서비스 **LaPoem**에서 책 한 권마다 붙는 대화형 챗봇의 백엔드입니다.
사용자가 읽고 있는 책에 대해 물으면 줄거리·주제·인물을 설명하고, 되물어서 대화를 이어 갑니다.
FastAPI 위에 WebSocket으로 실시간 대화를 받고, LangChain + OpenAI로 답을 만들고, PostgreSQL에 대화를 남깁니다.

> 프론트엔드 [lapoem_front](https://github.com/9seebird/lapoem_front) · 메인 백엔드 [lapoem_back](https://github.com/9seebird/lapoem_back) 과 함께 동작합니다. 이 저장소는 챗봇 서버만 담고 있습니다.

## 무엇을 하나

- **책별 채팅방** - 회원(member_num)과 책(book_id) 조합마다 채팅방이 하나 생기고, 대화가 DB에 저장되어 다시 들어와도 이어집니다.
- **일회성 채팅** - book_id 없이 접속하면 저장하지 않는 자유 대화 모드로 동작합니다.
- **"이 책 설명해줘" 감지** - 책 소개를 요청하는 표현을 정규식으로 잡아, DB에서 책 제목을 가져와 그 책에 대한 설명을 먼저 만듭니다.
- **대화 맥락 유지** - 이전 대화를 프롬프트에 함께 넣어 앞뒤가 이어지는 답을 냅니다.
- **한국어 전용 페르소나** - 시스템 프롬프트로 "친절한 책 전문가"를 정의하고, 주제가 벗어나면 책 이야기로 되돌립니다.

## 구조

```
브라우저 (lapoem_front)
   │  WebSocket  /ws/chat?member_num=…&book_id=…
   ▼
FastAPI (main.py)
   ├─ endpoints.py          라우터 · 메시지 분기(책 설명 요청 / 일반 대화)
   ├─ connection_manager.py 연결 관리 · 채팅방 찾기/만들기 · 이전 대화 로드
   ├─ chat_model.py         LangChain ChatOpenAI(gpt-4o-mini) · 프롬프트 템플릿
   └─ database.py           PostgreSQL 비동기 연결 (databases + asyncpg)
```

| 파일 | 역할 |
|---|---|
| `main.py` | 앱 생성, CORS, DB 연결 수명 관리 |
| `endpoints.py` | REST 2개 + WebSocket 1개. 메시지를 받아 책 설명 요청인지 일반 대화인지 나눠 처리하고 DB에 저장 |
| `connection_manager.py` | 접속 목록 관리, `chatbot` 테이블에서 채팅방 조회·생성, 재접속 시 이전 대화 전송 |
| `chat_model.py` | 모델 초기화와 프롬프트. 페르소나 규칙과 대화 기록 자리가 여기 있음 |
| `database.py` | `DATABASE_URL`로 DB 객체 하나 생성 |

## API

| 방식 | 경로 | 설명 |
|---|---|---|
| WS | `/ws/chat?member_num={회원번호}&book_id={책ID}` | 대화 채널. `{"message": "…"}` 를 보내면 `{"sender_id": "stella", "message": "…"}` 가 돌아옴. `book_id=0` 이면 저장 없는 일회성 대화 |
| GET | `/chat-list/{member_num}` | 그 회원이 대화 중인 책 목록 |
| GET | `/chat/{book_id}/{member_num}` | 특정 책 채팅방의 전체 대화 내역 |
| GET | `/` | 서버 동작 확인 |

WebSocket 메시지 흐름:

```
1. 접속  → chatbot 테이블에서 (member_num, book_id) 채팅방을 찾거나 만들고, 이전 대화를 순서대로 보내 줌
2. 수신  → "책 설명해줘" 류면 책 제목으로 소개 생성 / 아니면 대화 기록 + 질문을 프롬프트에 넣어 생성
3. 저장  → chating_content 에 사용자 메시지와 스텔라 답을 각각 INSERT (일회성 채팅은 저장 안 함)
4. 응답  → 같은 소켓으로 답 전송
```

## 실행

### 환경 변수 (`.env`)

```
OPENAI_API_KEY=sk-…
DATABASE_URL=postgresql://user:password@host:5432/dbname
```

### 로컬

```bash
pip install -r requirements.txt
uvicorn main:app --host 0.0.0.0 --port 9002 --reload
```

### Docker

```bash
docker build -t lapoem-chatbot .
docker run --env-file .env -p 9002:9002 lapoem-chatbot
```

### DB 테이블

이 서버가 읽고 쓰는 테이블입니다. 스키마 원본은 lapoem_back에 있고, 여기서는 참고용으로 필요한 컬럼만 적었습니다.

```sql
book            (book_id, book_title, …)
chatbot         (chat_id, book_id, member_num)              -- 회원·책당 채팅방 1개
chating_content (chat_id, chat_content, sender_id, timestamp)  -- sender_id: 'user' | 'stella'
```

## 기술

Python 3.12 · FastAPI · WebSocket · LangChain · OpenAI (gpt-4o-mini) · PostgreSQL (databases + asyncpg) · Docker

## 만들면서 정한 것

- **대화 기록은 메모리와 DB 둘 다** - 응답 생성에는 메모리의 기록을 쓰고, 재접속 대비로 DB에도 남깁니다. 서버가 재시작되면 DB에서 다시 불러옵니다.
- **책 소개 요청은 따로 분기** - 일반 대화 프롬프트로도 답은 나오지만, "이 책 뭐야?" 같은 첫 질문에는 대화 기록보다 책 제목이 더 중요해서 별도 경로로 뺐습니다.
- **일회성 모드를 남겨 둔 이유** - 로그인 전이나 책을 고르기 전에도 챗봇을 써 볼 수 있게 하려고 `book_id=0` 을 저장 없는 모드로 두었습니다.

## 한계와 다음에 한다면

- 책 내용을 **모델의 기억에만 의존**합니다. 책 본문이나 서평을 벡터 DB에 넣어 RAG로 근거를 붙이는 게 다음 단계입니다.
- 의도 감지가 정규식이라 표현이 조금만 달라도 일반 대화로 빠집니다. 분류를 모델에 맡기거나 Tool Calling으로 바꾸면 단순해집니다.
- 메모리의 대화 기록이 서버 수명 동안 계속 쌓입니다. 최근 N개만 유지하거나 요약해서 넣는 처리가 필요합니다.
- CORS가 전체 허용이고 `--reload` 로 뜹니다. 실제 배포에서는 허용 도메인을 좁히고 reload를 빼야 합니다.

## 라이선스

MIT
