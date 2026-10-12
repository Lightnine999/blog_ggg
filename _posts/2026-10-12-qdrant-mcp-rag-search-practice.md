---
title: "Qdrant MCP로 의미 검색 실습"
date: 2026-10-12 16:41:42 +0900
categories: [Claude_code, 실습]
tags: [qdrant, mcp, rag, embedding, vector-db, fastembed]
description: "Claude Code에 Qdrant MCP 서버를 연결해 문장을 저장·검색하고, 임베딩 모델에 따라 검색 결과가 어떻게 달라지는지 확인한 실습 기록입니다."
---

## 한 줄 요약

Claude Code에 **Qdrant MCP 서버**를 연결해서 문장 3개를 저장하고, 질문과 **뜻이 비슷한 문장**을 찾게 했다.
처음 설정된 영어 전용 임베딩 모델은 "택배 언제 와?"에 배송 문장을 찾지 못했고, 다국어 모델로 바꾸자 찾았다.

---

## 1. 이 실습을 왜 했나 (배경 개념)

### 비유로 먼저

도서관 사서를 떠올리면 쉽다.

- **보통 검색(키워드 검색)**: 책 제목에 "환불"이라는 글자가 **똑같이** 들어 있어야 찾아준다.
- **의미 검색(벡터 검색)**: "돈 돌려받는 거 언제까지야?"라고 물어도, 글자는 달라도 **뜻이 가까운** "환불 규정" 책을 찾아준다.

이번 실습은 Claude에게 두 번째 사서를 붙여준 것이다.

### 용어 정리

| 용어 | 쉬운 설명 | 이번 실습에서는 |
|---|---|---|
| **임베딩(Embedding)** | 문장을 숫자 목록(벡터)으로 바꾸는 것. 뜻이 비슷한 문장은 비슷한 숫자가 된다 | `fastembed` 라이브러리의 모델이 담당 |
| **벡터 DB** | 이 숫자 목록을 저장하고, "가장 가까운 것"을 빠르게 찾아주는 데이터베이스 | **Qdrant** |
| **컬렉션(Collection)** | 벡터 DB 안의 폴더 같은 것. 문장들을 묶어 두는 단위 | `policies`, `faq` |
| **유사도(코사인 유사도)** | 두 벡터가 얼마나 같은 방향인지. 1에 가까울수록 비슷하다 | 0.4, 0.8 같은 점수 |
| **RAG** | 질문이 들어오면 관련 문서를 먼저 **찾아서(Retrieval)** 그걸 보고 **답을 만드는(Generation)** 방식 | 이번엔 "찾기" 부분만 실습 |
| **MCP** | Claude 같은 AI가 바깥 도구(DB, 파일, API)를 쓸 수 있게 해주는 표준 연결 규격 | Claude ↔ Qdrant 연결 |

### 전체 흐름

```
[나] "이 문장 저장해 줘"
   ↓
[Claude] qdrant-store 도구 호출
   ↓
[MCP 서버] 문장 → 임베딩 모델 → 벡터 → Qdrant에 저장

[나] "환불 기한이 어떻게 돼?"와 비슷한 문장 찾아 줘
   ↓
[Claude] qdrant-find 도구 호출
   ↓
[MCP 서버] 질문 → 벡터 → Qdrant에서 가까운 순으로 꺼냄 → Claude에게 돌려줌
```

핵심: **Claude는 벡터 계산을 직접 하지 않는다.** Claude는 "어떤 도구를 어떤 인자로 부를지"만 정하고, 실제 저장·검색은 MCP 서버가 한다.
그래서 이번 실습의 관찰 포인트가 "모델이 어떤 도구를 **어떤 인자**로 호출하는가"였다.

---

## 2. 연결 설정

### 설정 파일 (`qdrant_rag_test/.mcp.json`)

```json
{
  "mcpServers": {
    "qdrant": {
      "command": "uvx",
      "args": ["mcp-server-qdrant"],
      "env": {
        "QDRANT_URL": ":memory:",
        "COLLECTION_NAME": "policies",
        "EMBEDDING_PROVIDER": "fastembed",
        "EMBEDDING_MODEL": "BAAI/bge-small-en-v1.5"
      }
    }
  }
}
```

| 항목 | 의미 |
|---|---|
| `command: uvx` + `args` | 파이썬 패키지 `mcp-server-qdrant`를 설치 없이 바로 실행 (uv 필요) |
| `QDRANT_URL: ":memory:"` | Qdrant 서버를 따로 띄우지 않고 **메모리에만** 저장. MCP 서버가 다시 켜지면 데이터가 사라진다 |
| `COLLECTION_NAME` | 기본 컬렉션. **이게 있으면 도구에서 `collection_name` 인자가 사라진다** (아래 4장) |
| `EMBEDDING_MODEL` | 문장을 벡터로 바꾸는 모델 |

`.mcp.json`은 프로젝트 폴더에 두는 **프로젝트 범위** 설정이다. Claude Code는 이 파일의 서버를 처음 쓸 때 한 번 승인을 받는다.

### 연결 확인

1. `/mcp` 입력 → `qdrant` 선택 → 연결 안 됐으면 **Reconnect**
2. `qdrant` 선택 → **View tools** → `qdrant-store`, `qdrant-find` 2개 확인

`/mcp` 메뉴는 사용자 화면에서만 열린다. Claude가 대신 누를 수 없어서 reconnect는 내가 직접 했다.

> **설정을 바꾸면 반드시 Reconnect.** 이미 켜져 있는 MCP 서버는 옛 설정으로 계속 돈다. `.mcp.json`을 고쳐도 reconnect 전까지는 반영되지 않는다.

---

## 3. 도구 정의서

공식 저장소: [qdrant/mcp-server-qdrant](https://github.com/qdrant/mcp-server-qdrant) (확인 시점 최신 릴리스 v0.8.1)

### `qdrant-store` — 저장

| 인자 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `information` | string | ✅ | 저장할 문장 |
| `metadata` | JSON | ❌ | 문장과 함께 저장할 부가 정보 (예: `{"category": "환불"}`) |
| `collection_name` | string | 조건부 | `COLLECTION_NAME`이 **없을 때만** 나타나고, 그때는 필수 |

- 반환: `Remembered: <문장> in collection <컬렉션>`
- 컬렉션이 없으면 자동으로 만든다.
- 도구 설명(모델이 읽는 문구): *"Keep the memory for later use, when you are asked to remember something."*

### `qdrant-find` — 검색

| 인자 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `query` | string | ✅ | 검색할 질문 |
| `collection_name` | string | 조건부 | `qdrant-store`와 같은 규칙 |

- 반환: `Results for the query '...'` + `<entry><content>문장</content><metadata>...</metadata></entry>` 목록
- **유사도 점수는 반환하지 않는다.** 순서만 알 수 있다.
- 최대 개수는 `QDRANT_SEARCH_LIMIT`(기본 10). **점수 하한이 없어서** 관련 없는 문장도 같이 나온다.

### 주요 환경변수

| 변수 | 기본값 | 설명 |
|---|---|---|
| `QDRANT_URL` | 없음 | Qdrant 서버 주소. `:memory:`면 메모리 모드 |
| `QDRANT_LOCAL_PATH` | 없음 | 로컬 파일로 저장 (`QDRANT_URL`과 동시 사용 불가) |
| `QDRANT_API_KEY` | 없음 | Qdrant Cloud 등 API 키 |
| `COLLECTION_NAME` | 없음 | 기본 컬렉션 |
| `EMBEDDING_PROVIDER` | `fastembed` | 현재 fastembed만 지원 |
| `EMBEDDING_MODEL` | `sentence-transformers/all-MiniLM-L6-v2` | 임베딩 모델 |
| `QDRANT_SEARCH_LIMIT` | `10` | 검색 결과 최대 개수 |
| `QDRANT_READ_ONLY` | `false` | `true`면 `qdrant-store` 도구가 사라짐 |
| `TOOL_STORE_DESCRIPTION` / `TOOL_FIND_DESCRIPTION` | 기본 문구 | 모델이 읽는 도구 설명을 바꿀 수 있음 |

---

## 4. 실습 과정

### 4-1. 첫 시도: 저장하고 찾기

입력:

> 다음 세 문장을 저장해 줘: "환불은 구매 후 7일 이내에 가능하다." "배송은 영업일 기준 2일이 걸린다." "회원 등급은 매월 1일에 갱신된다." 그리고 "환불 기한이 어떻게 돼?"와 비슷한 문장을 찾아 줘.

Claude가 실제로 보낸 호출:

| 순서 | 도구 | 인자 |
|---|---|---|
| 1-3 | `qdrant-store` | `information: "<문장>"` (3번) |
| 4 | `qdrant-find` | `query: "환불 기한이 어떻게 돼?"` |

결과: 1위 환불 / 2위 배송 / 3위 회원 등급. 저장 위치는 `policies` 컬렉션.

### 4-2. 왜 강의 화면과 다르게 `collection_name`이 안 보였나

강의 캡처에는 `qdrant-store(collection_name: "faq", information: ...)`처럼 컬렉션 이름이 인자로 들어가 있었다. 내 화면에는 없었다.

원인은 **설정 차이**였다. 서버 코드(`mcp_server.py`)를 보면, `COLLECTION_NAME`이 설정돼 있을 때 `collection_name` 인자를 그 값으로 **미리 고정하고 도구 목록에서 지운다.**

```python
if self.qdrant_settings.collection_name:
    find_foo = make_partial_function(
        find_foo, {"collection_name": self.qdrant_settings.collection_name}
    )
    store_foo = make_partial_function(
        store_foo, {"collection_name": self.qdrant_settings.collection_name}
    )
```

| 설정 | 모델이 보는 인자 | 저장 위치 |
|---|---|---|
| `COLLECTION_NAME` 있음 | `information`만 | 항상 기본 컬렉션 |
| `COLLECTION_NAME` 없음 | `information` + `collection_name` | 모델이 직접 정함 |

그리고 Claude Code 화면은 도구 호출 줄이 접혀 보일 수 있다. **Ctrl+O**로 펼치면 인자까지 보인다.

### 4-3. 강의와 똑같이: `COLLECTION_NAME` 지우고 `faq`로

`.mcp.json`에서 `COLLECTION_NAME` 줄을 지우고 `/mcp`에서 reconnect했다. 그러자 서버가 `collection_name` 인자를 받기 시작했다.

| 도구 | 인자 | 결과 |
|---|---|---|
| `qdrant-store` | `collection_name: "faq"`, `information: "<문장>"` (3번) | `Remembered: ... in collection faq` |
| `qdrant-find` | `collection_name: "faq"`, `query: "환불 기한이 어떻게 돼?"` | 1위 환불 |

메모리 모드라 reconnect할 때 `policies`의 데이터는 사라졌고, `faq`에 처음부터 다시 저장했다.

### 4-4. 질문을 바꿔 보니 틀리기 시작했다

질문에 **같은 단어가 없어도** 찾는지 보려고 질문을 바꿨다.

| 질문 | 1위 | 정답? |
|---|---|---|
| 환불 기한이 어떻게 돼? | 환불 | ✅ ("환불" 단어가 겹침) |
| 언제 받을 수 있어? | 배송 | ✅ |
| 택배 언제 와? | 환불 | ❌ (배송이어야 함) |

"택배"와 "배송"은 같은 뜻인데 연결하지 못했다. 설정된 모델 `BAAI/bge-small-en-v1.5`는 **영어로 학습된 모델**이라 한국어 의미를 잘 담지 못한다.

### 4-5. 임베딩 모델 비교

`qdrant-find`는 점수를 안 보여주므로, MCP 서버와 같은 fastembed로 직접 유사도를 계산해 모델 5개를 비교했다. fastembed가 지원하는 다국어 모델 4개와 기존 모델이다.

| 모델 | 크기 | 환불 기한이 어떻게 돼? | 언제 받을 수 있어? | 택배 언제 와? | 맞힘 |
|---|---|---|---|---|---|
| bge-small-en-v1.5 (기존) | 작음 | 환불 ✅ | 배송 ✅ | 환불 ❌ | 2/3 |
| paraphrase-multilingual-MiniLM-L12-v2 | 0.22GB | 환불 ✅ | 회원등급 ❌ | 환불 ❌ | 1/3 |
| **paraphrase-multilingual-mpnet-base-v2** | 1.0GB | 환불 ✅ | 회원등급 ❌ | **배송 ✅** | 2/3 |
| potion-multilingual-128M | 0.5GB | 환불 ✅ | 환불 ❌ | 배송·환불 동점 | 1/3 |
| multilingual-e5-large | 2.24GB | 환불 ✅ | 환불 ❌ | 배송 ✅ | 2/3 |

세 질문을 다 맞힌 모델은 없었다. `paraphrase-multilingual-mpnet-base-v2`를 골랐다.

- 점수 차이가 분명하다. 환불 질문에서 환불 문장 0.668, 나머지 0.42-0.43이다. 기존 모델은 모든 문장을 0.7-0.8로 비슷하게 매겨서 구분이 약했다.
- e5-large도 2/3를 맞혔지만 2.24GB로 무겁고, 점수가 0.80-0.88에 몰려 있었다.

### 4-6. 모델 교체 후 실제 MCP로 재검증

`EMBEDDING_MODEL`을 바꾸고 reconnect한 뒤, 세 문장을 `faq`에 다시 저장하고 검색했다.

| 질문 | 1위 | 2위 | 3위 |
|---|---|---|---|
| 환불 기한이 어떻게 돼? | **환불** ✅ | 회원 등급 | 배송 |
| 언제 받을 수 있어? | 회원 등급 ❌ | 배송 | 환불 |
| 택배 언제 와? | **배송** ✅ | 환불 | 회원 등급 |

미리 계산한 순서와 똑같이 나와서, 새 모델이 실제로 적용된 걸 확인했다.

"언제 받을 수 있어?"는 "환불금을 언제 받냐"로도 읽힐 수 있는 애매한 질문이다. 기존 모델이 이 질문을 맞힌 것도 문장이 3개뿐이라 우연이었을 수 있다.

---

## 5. 실패 사례 모음

### 5-1. 이번 실습에서 겪은 것

| 증상 | 원인 | 해결 |
|---|---|---|
| 도구에 `collection_name` 인자가 없음 | `COLLECTION_NAME` 환경변수가 인자를 고정함 | 줄을 지우고 reconnect |
| 설정을 고쳤는데 그대로임 | 켜져 있는 MCP 서버는 옛 설정 사용 | `/mcp` → Reconnect |
| reconnect 후 저장한 문장이 사라짐 | `QDRANT_URL: ":memory:"` | 유지가 필요하면 `QDRANT_LOCAL_PATH`나 실제 Qdrant 서버 사용 |
| "택배"로 배송 문장을 못 찾음 | 영어 전용 임베딩 모델 | 다국어 모델로 교체 |
| 관련 없는 문장도 2·3위로 나옴 | 점수 하한 없이 상위 N개를 다 돌려줌 | 결과 수를 줄이거나 1위만 사용 |
| 도구 호출이 화면에 안 보임 | Claude Code가 호출 줄을 접어서 표시 | Ctrl+O |
| reconnect 후 Claude 쪽 도구 설명은 그대로 | 세션 시작 때 받은 도구 설명이 남아 있었음 | 실제 호출 결과(`in collection faq`)로 적용 여부 확인 |

### 5-2. 공식 저장소 GitHub 이슈에서 찾은 것

| 이슈 | 증상 | 원인·해결 |
|---|---|---|
| [#36](https://github.com/qdrant/mcp-server-qdrant/issues/36) | `spawn uvx ENOENT` 연결 실패 | `uvx`가 설치 안 됐거나 PATH에 없음. uv 설치 필요 |
| [#56](https://github.com/qdrant/mcp-server-qdrant/issues/56) | Claude Code에서 `spawn QDRANT_URL=... uvx mcp-server-qdrant ENOENT` | 환경변수를 명령어 앞에 붙여 써서 그 전체를 프로그램 이름으로 인식함. 환경변수는 `env`에 따로 넣어야 함 |
| [#96](https://github.com/qdrant/mcp-server-qdrant/issues/96) | `mcp-server-quadrant was not found` | 패키지 이름 오타. 올바른 이름은 `mcp-server-qdrant` |
| [#10](https://github.com/qdrant/mcp-server-qdrant/issues/10), [#60](https://github.com/qdrant/mcp-server-qdrant/issues/60) | `Vector with name fast-... is not configured in this collection` | 서버는 벡터 이름을 `fast-모델명` 규칙으로 쓴다. 다른 방법으로 만든 기존 컬렉션과 이름이 안 맞음 |
| [#47](https://github.com/qdrant/mcp-server-qdrant/issues/47), [#48](https://github.com/qdrant/mcp-server-qdrant/issues/48) | 저장은 되는데 `qdrant-find`가 `'document'` 에러 / 기존 컬렉션 조회 불가 | 서버가 만들지 않은 컬렉션은 payload 구조(`document`, `metadata`)가 달라서 읽지 못함 |
| [#8](https://github.com/qdrant/mcp-server-qdrant/issues/8) | Windows에서 일본어 저장 시 타임아웃 | 문자 인코딩 문제. `env`에 `PYTHONIOENCODING: "utf-8"` 추가로 해결 |
| [#188](https://github.com/qdrant/mcp-server-qdrant/issues/188) | 관련 없는 결과도 정답처럼 반환됨 | `score_threshold`가 없어서 상위 N개를 항상 돌려줌. 이번 실습 2·3위 현상과 같음 (2026-09 기준 열린 이슈) |
| [#151](https://github.com/qdrant/mcp-server-qdrant/issues/151) | Claude Desktop에서 두 도구 모두 빈 에러 | 2026-06 등록, 열린 이슈 |

Claude Code 공식 문서의 MCP 점검 순서:

- 프로젝트 `.mcp.json` 서버는 **한 번 승인**이 필요하다. 승인 창을 닫았으면 `/mcp`에서 승인해야 켜진다.
- `/mcp`에 **failed**로 보이면 서버 실행 실패다. `command`나 `args`의 상대경로는 `.mcp.json` 위치가 아니라 Claude Code를 실행한 폴더 기준이라 자주 틀린다.
- connected인데 **도구가 0개**면 Reconnect. 그래도 0개면 `claude --debug=mcp`로 실행해 로그를 본다.

---

## 6. 정리

- 이번 실습은 RAG의 **찾기(Retrieval)** 부분을 Claude + Qdrant MCP로 직접 해본 것이다.
- Claude는 도구와 인자만 정하고, 벡터 계산과 저장·검색은 MCP 서버가 한다.
- `COLLECTION_NAME`을 설정하면 `collection_name` 인자가 사라진다. 강의 화면과 다르게 보인 이유다.
- 설정을 바꾸면 `/mcp` → Reconnect. `:memory:` 모드는 reconnect하면 데이터가 사라진다.
- 검색 품질은 **임베딩 모델**이 좌우한다. 영어 전용 모델은 "택배"와 "배송"을 연결하지 못했다.
- `qdrant-find`는 점수를 보여주지 않고 관련 없는 결과도 돌려준다. 점수가 필요하면 fastembed로 직접 계산해야 한다.
- 문장 3개, 질문 3개로는 모델 우열을 확정하기 어렵다.

### 최종 설정

```json
{
  "mcpServers": {
    "qdrant": {
      "command": "uvx",
      "args": ["mcp-server-qdrant"],
      "env": {
        "QDRANT_URL": ":memory:",
        "EMBEDDING_PROVIDER": "fastembed",
        "EMBEDDING_MODEL": "sentence-transformers/paraphrase-multilingual-mpnet-base-v2"
      }
    }
  }
}
```

### 참고 자료

- [qdrant/mcp-server-qdrant README](https://github.com/qdrant/mcp-server-qdrant)
- 도구 정의 코드: `src/mcp_server_qdrant/mcp_server.py`, `src/mcp_server_qdrant/settings.py`
- [Claude Code MCP 문서](https://code.claude.com/docs/en/mcp)
- [FastEmbed 지원 모델](https://qdrant.github.io/fastembed/examples/Supported_Models/)
