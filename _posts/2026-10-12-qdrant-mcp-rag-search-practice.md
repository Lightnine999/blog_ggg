---
title: "Qdrant MCP로 보는 도구 호출 구조"
date: 2026-10-12 16:41:42 +0900
categories: [Claude_code, 실습]
tags: [mcp, harness, tool-calling, qdrant, rag, embedding]
description: "남이 만든 MCP 서버(Qdrant RAG)를 Claude Code에 붙여, 모델·하네스·도구 정의서가 실제로 어떻게 움직이는지 관찰한 실습 기록입니다."
---

## 이 글의 위치

노드 주제: **LLM이 하네스를 통해 도구를 쓰는 방법, 그리고 그 통로인 MCP를 어떻게 정의하고 다루는가.**

이 글은 노드 5장 실습 2의 마지막 단계 **"남이 만든 MCP 서버(Qdrant RAG) 붙이기"**를 Claude Code에서 해본 기록이다.
앞 실습(`get_inventory`)에서는 도구를 **직접 만들었다**. 여기서는 **남이 만든 도구 서버**를 가져다 붙이고, 노드의 개념이 그대로 보이는지 확인한다.

| 노드 장 | 내용 | 이 글과의 관계 |
|---|---|---|
| 1장 | 실패의 세 순간, 도구 호출 4단계 순환 | **2절**에서 Qdrant 호출에 대입 |
| 2장 | 도구 설명은 모델이 읽는 지시다 | **3절** `collection_name` 실험 |
| 3장 | 도구 정의서와 실패 사례 | **4절** Qdrant 도구 정의서, **6절** 실패 사례 |
| 5장 | 도구 실행, 트레이스, RAG를 MCP로 붙이기 | **이 실습 자체** |
| 6장 | 권한과 신뢰 경계 | **7절** 읽기/쓰기 분리 |
| 8장 | 전후 판정과 회귀 | **5절** 임베딩 모델 교체 |

---

## 1. 핵심 개념 한 장 요약

### 비유

모델은 **주문서를 쓰는 손님**이고, 하네스는 **주문서를 확인하고 실제로 요리를 내오는 직원**이다.
손님은 주방에 들어갈 수 없다. "김치찌개 하나, 덜 맵게"라고 **적을** 뿐이다. 직원이 메뉴에 있는지, 옵션이 맞는지 확인하고 주방에 전달한 뒤 음식을 가져다준다.

### 네 낱말

| 낱말 | 하는 일 | 이 실습에서 |
|---|---|---|
| **모델** | 텍스트를 읽고 텍스트를 쓴다. 도구를 **직접 실행하지 못한다** | Claude |
| **하네스** | 허용 목록 확인 → 인자 검사 → 실제 실행 → 결과를 모델에 다시 넣기 | Claude Code |
| **도구(MCP 서버)** | 실제 기능을 가진 프로그램. MCP는 하네스와 도구가 대화하는 규격 | `mcp-server-qdrant` |
| **루프** | 요청 → 실행 → 결과 재입력을 반복하는 한 바퀴 | 저장 3번 + 검색 1번 |

**에이전트**는 이 전체(모델 + 도구 계약 + 하네스 + 루프)를 묶어 "행동하는 것"으로 만든 것이다.

### 도구 호출 4단계 (노드 1장)

```
① 하네스: 도구 정의(이름·설명·인자 규격)를 모델 입력에 넣는다
② 모델:   도구를 실행하지 않고 "호출 요청 JSON"만 쓴다
③ 하네스: 허용된 도구인지, 인자가 규격에 맞는지 검사하고 실제로 실행한다
④ 하네스: 실행 결과를 다시 모델 입력에 넣고, 모델이 그걸 근거로 답한다
```

### 실패의 세 순간 (노드 1장)

| 실패 | 어디의 문제인가 | 빠진 것 |
|---|---|---|
| 도구를 **잘못 고름** | 도구 계약(정의서) | 언제 쓰고 언제 안 쓰는지 |
| 결과를 **잘못 읽음** | 모델의 해석 | 반환 근거를 남기는 관찰·판정 |
| 같은 실수 **반복** | 하네스·루프 | 중단 기준, 사람에게 넘기는 조건 |

---

## 2. Qdrant 호출에 4단계 대입하기

입력:

> 다음 세 문장을 저장해 줘: "환불은 구매 후 7일 이내에 가능하다." "배송은 영업일 기준 2일이 걸린다." "회원 등급은 매월 1일에 갱신된다." 그리고 "환불 기한이 어떻게 돼?"와 비슷한 문장을 찾아 줘.

| 단계 | 실제로 일어난 일 |
|---|---|
| ① 도구 정의 주입 | Claude Code가 MCP 서버에서 받은 `qdrant-store`, `qdrant-find`의 이름·설명·인자 규격을 Claude 입력에 넣음 |
| ② 호출 요청 | Claude가 `qdrant-store(information: "환불은 ...")` 같은 요청을 씀 (3번) + `qdrant-find(query: "환불 기한이 어떻게 돼?")` |
| ③ 검사·실행 | Claude Code가 MCP 서버로 전달 → 서버가 문장을 임베딩해 Qdrant에 저장·검색 |
| ④ 결과 재입력 | `Remembered: ... in collection policies`, 검색 결과 목록이 Claude에게 돌아오고, Claude가 이를 읽고 답함 |

**Claude는 벡터 계산도, DB 저장도 하지 않았다.** 쓴 것은 도구 이름과 인자뿐이다. 노드 결과 고찰의 "각 호출의 인자가 무엇인지 관찰해보자"가 ② 단계를 보라는 뜻이다.

### 트레이스 (노드 5장 형식)

```
== 실행 조건 ==
날짜: 2026-10-12
하네스: Claude Code / MCP 서버: mcp-server-qdrant (uvx)
설정: QDRANT_URL=:memory:, COLLECTION_NAME=policies, EMBEDDING_MODEL=BAAI/bge-small-en-v1.5

== [1] 모델의 요청 ==
qdrant-store {information: "환불은 구매 후 7일 이내에 가능하다."}
qdrant-store {information: "배송은 영업일 기준 2일이 걸린다."}
qdrant-store {information: "회원 등급은 매월 1일에 갱신된다."}
qdrant-find  {query: "환불 기한이 어떻게 돼?"}

== [2] 하네스의 실행 결과 ==
Remembered: ... in collection policies (x3)
Results: 1) 환불은 ... 2) 배송은 ... 3) 회원 등급은 ...

== [3] 결과를 읽은 최종 답 ==
"환불은 구매 후 7일 이내에 가능하다."가 가장 비슷한 문장
```

Claude Code 화면에서 도구 호출 줄이 접혀 안 보이면 **Ctrl+O**로 펼친다.

---

## 3. 도구 정의가 바뀌면 모델이 쓸 수 있는 인자가 바뀐다 (노드 2장)

강의 화면에서는 `qdrant-store(collection_name: "faq", information: ...)`처럼 **컬렉션 이름을 모델이 직접 넣었다.** 내 실행에는 `collection_name`이 없었다.

원인은 서버 설정이다. `mcp_server.py`를 보면 `COLLECTION_NAME`이 설정돼 있을 때 그 인자를 **미리 채우고 도구 정의에서 지운다.**

```python
if self.qdrant_settings.collection_name:
    find_foo = make_partial_function(
        find_foo, {"collection_name": self.qdrant_settings.collection_name}
    )
    store_foo = make_partial_function(
        store_foo, {"collection_name": self.qdrant_settings.collection_name}
    )
```

| 설정 | ① 모델에게 주입되는 인자 | ② 모델이 쓴 요청 |
|---|---|---|
| `COLLECTION_NAME=policies` | `information`만 | `qdrant-store(information: ...)` |
| `COLLECTION_NAME` 삭제 후 Reconnect | `information`, `collection_name` | `qdrant-store(collection_name: "faq", information: ...)` |

노드 2장의 "도구 설명을 쓰는 일은 프롬프트를 쓰는 일이다"가 그대로 보인다. **모델은 정의서에 적힌 인자만 쓸 수 있다.** 정의서에서 인자를 지우면 모델은 그 선택지를 아예 모른다.

주의할 점 두 가지:
- `.mcp.json`을 고쳐도 이미 켜진 서버는 옛 설정 그대로다. `/mcp` → **Reconnect**가 필요하다. 이 메뉴는 사용자 화면에서만 열려서 Claude가 대신 누를 수 없다.
- `QDRANT_URL: ":memory:"`라서 Reconnect하면 저장한 문장이 사라진다. `faq`에 처음부터 다시 저장했다.

---

## 4. Qdrant 도구 정의서 (노드 3장 형식으로 읽기)

출처: [qdrant/mcp-server-qdrant](https://github.com/qdrant/mcp-server-qdrant) README와 `settings.py`, `mcp_server.py` (확인 시점 최신 v0.8.1)

| 항목 | `qdrant-store` | `qdrant-find` |
|---|---|---|
| 하는 일 (동작 하나) | 문장을 임베딩해 저장 (**쓰기**) | 질문과 가까운 문장 검색 (**읽기**) |
| 모델이 읽는 설명 | *"Keep the memory for later use, when you are asked to remember something."* | *"Look up memories in Qdrant. Use this tool when you need to: Find memories by their content / Access memories for further analysis / Get some personal information about the user"* |
| 인자 | `information`(필수), `metadata`(선택), `collection_name`(기본 컬렉션 없을 때만) | `query`(필수), `collection_name`(같은 규칙) |
| 반환 | `Remembered: <문장> in collection <이름>` | `Results for the query '...'` + `<entry><content>...</content><metadata>...</metadata></entry>` 목록 |
| 쓰지 말아야 할 때 | 정의서에 없음 | 정의서에 없음 |
| 실패 뒤 행동 | 정의서에 없음 | 정의서에 없음 |

노드 3장 기준으로 보면 비어 있는 칸이 보인다.

- **"쓰는 때"만 있고 "안 쓰는 때"와 "실패 뒤 행동"이 없다.** 노드가 말한 "잘못 고름"의 원인 자리다.
- **`qdrant-find`는 유사도 점수를 돌려주지 않고, 점수 하한도 없다.** 관련 없는 문장도 상위 N개 안에 들면 똑같은 형식으로 돌아온다(기본 최대 10개). 모델 입장에서는 정답과 잡음이 구분되지 않는다. 노드의 "잘못 읽음"이 생기기 쉬운 구조다. 같은 문제가 공식 저장소 [이슈 #188](https://github.com/qdrant/mcp-server-qdrant/issues/188)로 올라와 있다.
- 설명 문구는 환경변수 `TOOL_STORE_DESCRIPTION`, `TOOL_FIND_DESCRIPTION`으로 바꿀 수 있다. 정의서를 고칠 수 있는 손잡이다.

### 주요 환경변수

| 변수 | 기본값 | 설명 |
|---|---|---|
| `QDRANT_URL` | 없음 | Qdrant 주소. `:memory:`면 메모리 모드 |
| `QDRANT_LOCAL_PATH` | 없음 | 로컬 파일 저장 (`QDRANT_URL`과 동시 사용 불가) |
| `COLLECTION_NAME` | 없음 | 기본 컬렉션. 있으면 `collection_name` 인자가 사라짐 |
| `EMBEDDING_PROVIDER` | `fastembed` | 현재 fastembed만 지원 |
| `EMBEDDING_MODEL` | `sentence-transformers/all-MiniLM-L6-v2` | 문장을 벡터로 바꾸는 모델 |
| `QDRANT_SEARCH_LIMIT` | `10` | 검색 결과 최대 개수 |
| `QDRANT_READ_ONLY` | `false` | `true`면 `qdrant-store`(쓰기 도구)가 사라짐 |

---

## 5. 전후 판정: 임베딩 모델 교체 (노드 8장)

### 왜 했나

모델이 **도구를 잘 골라도**, 도구가 돌려준 **결과 자체가 틀리면** 답도 틀린다. 질문을 바꿔서 검색 결과를 확인했다.

| 질문 | 1위 | 판정 |
|---|---|---|
| 환불 기한이 어떻게 돼? | 환불 | ✅ ("환불" 단어가 겹침) |
| 언제 받을 수 있어? | 배송 | ✅ |
| 택배 언제 와? | 환불 | ❌ |

설정된 `BAAI/bge-small-en-v1.5`는 영어로 학습된 모델이라 "택배"와 "배송"을 연결하지 못했다.

### 모델 비교

`qdrant-find`가 점수를 안 주므로, 서버와 같은 fastembed로 직접 유사도를 계산했다. fastembed가 지원하는 다국어 모델 4개와 기존 모델을 같은 질문 3개로 비교했다.

| 모델 | 크기 | 환불 기한이 어떻게 돼? | 언제 받을 수 있어? | 택배 언제 와? | 맞힘 |
|---|---|---|---|---|---|
| bge-small-en-v1.5 (기존) | 작음 | 환불 ✅ | 배송 ✅ | 환불 ❌ | 2/3 |
| paraphrase-multilingual-MiniLM-L12-v2 | 0.22GB | 환불 ✅ | 회원등급 ❌ | 환불 ❌ | 1/3 |
| **paraphrase-multilingual-mpnet-base-v2** | 1.0GB | 환불 ✅ | 회원등급 ❌ | **배송 ✅** | 2/3 |
| potion-multilingual-128M | 0.5GB | 환불 ✅ | 환불 ❌ | 배송·환불 동점 | 1/3 |
| multilingual-e5-large | 2.24GB | 환불 ✅ | 환불 ❌ | 배송 ✅ | 2/3 |

mpnet을 골랐다. 환불 질문에서 정답 0.668, 나머지 0.42-0.43으로 차이가 분명했다. 기존 모델은 모든 문장에 0.7-0.8을 줘서 구분이 약했다.

### 교체 후 실제 MCP로 재검증

| 질문 | 교체 전 1위 | 교체 후 1위 |
|---|---|---|
| 환불 기한이 어떻게 돼? | 환불 ✅ | 환불 ✅ |
| 언제 받을 수 있어? | 배송 ✅ | 회원 등급 ❌ |
| 택배 언제 와? | 환불 ❌ | **배송 ✅** |

교체 후 순서가 미리 계산한 결과와 똑같아서 새 모델이 실제 적용된 것을 확인했다.

**하나를 고쳤더니 다른 하나가 틀어졌다. 노드 8장이 말하는 회귀다.** 맞힌 개수는 2/3로 같다. 문장 3개, 질문 3개로는 어느 모델이 낫다고 확정할 수 없다. "언제 받을 수 있어?"는 환불금을 언제 받냐로도 읽히는 애매한 질문이기도 하다.

---

## 6. 실패 사례를 세 순간으로 나누기

### 이번 실습에서 겪은 것

| 증상 | 세 순간 중 | 원인 | 대응 |
|---|---|---|---|
| 강의와 달리 `collection_name` 인자가 없음 | (정의서 차이) | `COLLECTION_NAME`이 인자를 지움 | 줄 삭제 → Reconnect |
| 설정을 고쳤는데 반영 안 됨 | 반복 | 켜진 서버는 옛 설정 사용 | `/mcp` → Reconnect |
| Reconnect 후 데이터 사라짐 | - | `:memory:` 모드 | 유지하려면 `QDRANT_LOCAL_PATH` |
| "택배"로 배송 문장을 못 찾음 | 잘못 읽음 (근거 자체가 틀림) | 영어 전용 임베딩 모델 | 다국어 모델로 교체 후 전후 비교 |
| 관련 없는 문장이 2·3위로 나옴 | 잘못 읽음 위험 | 점수 하한 없음, 점수 미반환 | 결과 수 제한, 점수가 필요하면 직접 계산 |
| 모델 교체 후 다른 질문이 틀어짐 | 회귀 | 모델마다 강점이 다름 | 평가 질문을 늘려 다시 판정 |

**도구를 잘못 고른 사례는 없었다.** 저장 요청엔 `qdrant-store`, 검색 요청엔 `qdrant-find`를 정확히 불렀다. 도구가 2개뿐이고 이름이 분명해서다.

### 공식 저장소 이슈에서 찾은 것 (연결 단계 실패)

| 이슈 | 증상 | 원인 |
|---|---|---|
| [#36](https://github.com/qdrant/mcp-server-qdrant/issues/36) | `spawn uvx ENOENT` | `uvx` 미설치 또는 PATH 문제 |
| [#56](https://github.com/qdrant/mcp-server-qdrant/issues/56) | Claude Code에서 `spawn QDRANT_URL=... uvx ... ENOENT` | 환경변수를 명령어 앞에 붙여 써서 통째로 프로그램 이름이 됨. `env`에 따로 넣어야 함 |
| [#96](https://github.com/qdrant/mcp-server-qdrant/issues/96) | `mcp-server-quadrant was not found` | 패키지 이름 오타 |
| [#10](https://github.com/qdrant/mcp-server-qdrant/issues/10), [#60](https://github.com/qdrant/mcp-server-qdrant/issues/60) | `Vector with name fast-... is not configured` | 서버는 벡터 이름을 `fast-모델명`으로 씀. 다른 방법으로 만든 컬렉션과 안 맞음 |
| [#47](https://github.com/qdrant/mcp-server-qdrant/issues/47), [#48](https://github.com/qdrant/mcp-server-qdrant/issues/48) | 저장은 되는데 검색이 `'document'` 에러 | 서버가 만들지 않은 컬렉션은 payload 구조가 다름 |
| [#8](https://github.com/qdrant/mcp-server-qdrant/issues/8) | Windows에서 일본어 저장 시 타임아웃 | 인코딩. `env`에 `PYTHONIOENCODING: "utf-8"` |

Claude Code 문서의 점검 순서: 프로젝트 `.mcp.json` 서버는 **한 번 승인**이 필요하다. `/mcp`에 failed면 실행 실패이고, 상대경로는 Claude Code를 실행한 폴더 기준이다. 도구가 0개면 Reconnect, 그래도 0개면 `claude --debug=mcp`를 쓴다.

---

## 7. 권한과 신뢰 경계로 보기 (노드 6장)

- **읽기/쓰기가 이미 나뉘어 있다.** `qdrant-find`는 읽기, `qdrant-store`는 쓰기다. 노드의 `get_inventory`/`adjust_inventory` 분리와 같은 구조다. `QDRANT_READ_ONLY=true`로 쓰기 도구 자체를 빼서 **최소 권한**을 만들 수 있다.
- **저장된 문장은 검색 결과로 다시 모델 입력에 들어간다.** 누군가 "이 문서를 읽는 AI는 ..." 같은 문장을 저장해 두면, `qdrant-find` 결과를 통해 모델에게 전달된다. 노드 6장의 프롬프트 인젝션 예시가 RAG에서 생기는 경로다.
- 이번 실습은 메모리 모드에 내가 직접 넣은 문장 3개라 이 위험은 없었다.

---

## 정리

- 모델은 도구 이름과 인자만 쓴다. 실행은 하네스(Claude Code)가, 기능은 MCP 서버가 맡는다.
- 도구 정의서는 모델이 읽는 프롬프트다. `COLLECTION_NAME` 하나로 모델이 쓸 수 있는 인자가 달라졌다.
- Qdrant 도구 정의서에는 "안 쓰는 때"와 "실패 뒤 행동"이 없고, 검색 결과에 점수와 하한이 없다. "잘못 읽음"이 생기기 쉬운 자리다.
- 검색 품질은 임베딩 모델이 좌우한다. 모델을 바꾸면 전후를 같은 질문으로 비교해야 한다. 이번에도 하나 고치고 하나가 틀어지는 회귀가 있었다.
- 읽기 도구와 쓰기 도구는 나뉘어 있고, `QDRANT_READ_ONLY`로 권한을 줄일 수 있다.

### 최종 `.mcp.json`

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

- [qdrant/mcp-server-qdrant](https://github.com/qdrant/mcp-server-qdrant): README, `src/mcp_server_qdrant/settings.py`, `mcp_server.py`
- [Claude Code MCP 문서](https://code.claude.com/docs/en/mcp)
- [FastEmbed 지원 모델](https://qdrant.github.io/fastembed/examples/Supported_Models/)
