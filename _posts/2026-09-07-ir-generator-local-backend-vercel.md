---
title: 무료 Codex CLI를 지키면서 Vercel에 배포하기
date: 2026-09-07 21:00:00 +0900
categories: [TypeScript, 프로젝트]
tags: [aiffel, nextjs, supabase, vercel, rls, cloudflare-tunnel, codex-cli]
---

IR 자료 생성 서비스(Next.js + Supabase)를 만들다가, "로컬 컴퓨터에서만 돌아가는 LLM 호출부"를
그대로 둔 채 프론트엔드만 인터넷에 배포하는 문제를 풀었다. 중간에 "그냥 하나로 합쳐서 배포해줘"라는
요청이 왔는데, 그게 왜 안 되는지를 설명하는 것부터가 오늘의 절반이었다.

결과부터 적으면 이렇다.

| 항목 | 결과 |
| --- | --- |
| 오늘 커밋 | 9개 |
| 변경 파일 | 56개 (+3,202 / −501) |
| 배포 | Vercel 프로덕션, GitHub 연동 자동 배포까지 확인 |
| 백엔드 | 로컬 컴퓨터 + Cloudflare Tunnel (무료 임시 URL) |

## 1. "합치자"는 요청과 무료 Codex CLI 사이의 갈림길

이 서비스는 슬라이드 내용을 로컬에 설치된 Codex CLI(ChatGPT 구독)로 만든다. API 과금이 없는
대신 조건이 하나 붙는다 — **이 코드를 실행하는 컴퓨터에 codex가 설치·로그인돼 있어야 한다.**

프론트엔드를 Vercel에 올리고 나니 "백엔드도 같이 합쳐서 배포해달라"는 요청이 왔다. 그런데 Vercel의
서버는 요청마다 새로 뜨는 임시 환경이라 `codex` 실행 파일 자체가 없다. 합치는 순간 예전에 이미 겪었던
실패("Vercel엔 codex 바이너리가 없어 생성이 항상 실패한다")가 그대로 재현된다.

선택지는 사실 두 가지뿐이었다.

| | 분리 유지 | 진짜로 합치기 |
| --- | --- | --- |
| 비용 | 무료 (ChatGPT 구독) | 유료 (OpenAI/Anthropic API) |
| 배포 | 프론트만 Vercel, 백엔드는 로컬 + 터널 | 전부 Vercel 한 곳 |
| 로컬 컴퓨터 의존 | 있음 (꺼지면 생성 중단) | 없음 |

지금 단계는 무료가 중요해서 **분리 유지**로 정하고, 나중에 서비스가 커지면 유료 API로 전환하며
합치기로 했다. 기술 선택을 코드로 밀어붙이지 않고 트레이드오프 표로 먼저 보여준 게 여기서
제일 잘한 일이었다.

## 2. 프록시 패턴 — 함수 하나만 바꿔서 백엔드를 통째로 교체하기

분리하기로 했으면, 프론트엔드 코드는 최대한 안 건드리고 싶었다. 원래 구조는 이랬다.

```
[브라우저] → /api/generate (Next.js) → codex 서브프로세스 직접 실행
```

바꾼 구조는 이렇다.

```
[브라우저] → /api/generate (Next.js, 그대로) → HTTP 요청 → backend/server.ts → codex 서브프로세스
```

`generateSlidePlan()`이라는 함수 하나의 **구현만** 바꿨다. 로컬 spawn 대신 `fetch`로 옆 포트의
백엔드를 부르는 걸로.

```ts
// src/lib/deck/generate-client.ts
export async function generateSlidePlan(companyName: string, sourceText: string) {
  const res = await fetch(`${process.env.BACKEND_URL}/generate`, {
    method: "POST",
    headers: { "Content-Type": "application/json", "x-api-key": process.env.BACKEND_SECRET! },
    body: JSON.stringify({ companyName, sourceText }),
  });
  // ...
}
```

`/api/generate/route.ts`는 import 경로 한 줄만 바뀌었다. 인증(Supabase 세션 확인)도, PPTX 렌더도,
Storage 업로드도 전부 그대로 살아남았다. **함수 시그니처를 그대로 두고 안쪽만 바꾸는 게** 이번에도
제일 적은 변경으로 아키텍처를 바꾸는 길이었다.

## 3. 미리보기, PPTX를 이미지로 굽는 대신 데이터를 다시 그렸다

생성 결과 페이지에 "미리보기는 아직 준비 안 됨"이라는 자리표시자가 있었다. PPTX 파일을 이미지로
바꿔서 보여주려면 방법이 세 가지였다.

| | LibreOffice 변환 | 클라우드 API | 슬라이드 데이터로 재구성 |
| --- | --- | --- | --- |
| 새로 설치할 것 | LibreOffice(~1GB) | 없음(API 키) | 없음 |
| 비용 | 없음 | 건당 과금 | 없음 |
| PPTX와 픽셀 단위로 동일한가 | 예 | 예 | 아니오 |

셋 다 "PPTX를 이미지로 바꾼다"는 전제였는데, 생각해보니 이미 슬라이드를 만드는 재료
(`SlidePlan` — 제목·결론·카드/타임라인/흐름/지표 JSON)가 손에 있었다. 그걸 웹 컴포넌트 4종으로
다시 그리면 이미지 변환 없이도 같은 내용을 보여줄 수 있었다.

문제는 그 JSON을 애초에 **저장하지 않고 버리고 있었다**는 것. PPTX만 만들고 나면 원본 데이터는
사라졌다. `decks` 테이블에 `plan_json jsonb` 컬럼 하나 추가하고, 생성할 때 같이 저장하는 걸로
해결했다. 새로 만드는 자료부터는 미리보기가 되고, 이전 자료는 여전히 자리표시자로 남는다 —
없는 데이터를 있는 척하지 않는 쪽을 골랐다.

## 4. Vercel 배포에서 만난 함정 둘

### "services" 모드로 잘못 인식됨

`vercel link`를 실행했더니 이 저장소를 "frontend + backend 두 개의 서비스"로 자동 인식해버렸다.
`backend/` 폴더가 Node 프로젝트처럼 보였던 모양이다. 그런데 `backend/server.ts`는 로컬 전용이라
Vercel에 배포되면 안 되는 코드다.

`vercel.json`을 지워도 안 됐다. 프로젝트 자체의 **framework 설정**이 이미 "services"로
저장돼 있었기 때문이다. CLI에 정확히 이 상황을 위한 명령이 있었다.

```bash
vercel project update ir-generator --framework nextjs --yes
```

> 로컬 파일(`vercel.json`)만 고치고 프로젝트 설정 자체는 안 바뀌어 있을 수 있다는 걸
> 배포 두 번 실패하고서야 알았다.
{: .prompt-tip }

### 빈 값으로 저장된 환경변수

`NEXT_PUBLIC_SUPABASE_URL`을 CLI로 등록하는데 이런 흐름이 나왔다.

```
? Value?
! Value is empty
? Value? Leave as is
✓ Added NEXT_PUBLIC_SUPABASE_URL ...
```

"Leave as is"를 "다시 입력할게"로 오해하고 그냥 지나갔더니, 실제로는 **빈 값을 그대로
저장하겠다는 확인**이었다. 배포 후 런타임에 이런 에러가 났다.

```
Error: Your project's URL and Key are required to create a Supabase client!
```

`vercel env pull`로 값을 직접 pull해서 확인해보니 정말 빈 문자열이었다. 지우고, 이번엔 대화형
프롬프트를 아예 안 거치게 `.env.local`에서 바로 파이프로 흘려 넣었다.

```bash
printf '%s' "$NEXT_PUBLIC_SUPABASE_URL" | npx vercel env add NEXT_PUBLIC_SUPABASE_URL production
```

> CLI 프롬프트 UX가 헷갈리면 사람이 실수하는 게 아니라 **프롬프트가 실수를 유도한 것**이다.
> 재발 방지는 "조심하기"가 아니라 대화형 입력 자체를 없애는 것.
{: .prompt-warning }

## 5. RLS를 두 번 고쳤다 — 처음엔 role을 안 정했다

만든 예시 자료 두 개를 "로그인한 다른 사람도 볼 수 있게" 공개하기로 했다. 정책은 이렇게 짰다.

```sql
create policy "decks_select_examples" on public.decks
  for select
  using (is_example = true);
```

돌려보니 동작은 했다. 그런데 다시 보니 `to authenticated`를 안 붙였다. `is_example = true`라는
조건 자체가 `auth.uid()`를 안 쓰기 때문에, **로그인 안 한 방문자(anon 역할)한테도 그대로
열려 있었다.** "로그인시 볼 수 있게"라는 요구사항과 실제 정책 범위가 어긋난 것이다.

```sql
create policy "decks_select_examples" on public.decks
  for select to authenticated        -- 이 한 줄이 없으면 anon도 통과한다
  using (is_example = true);
```

기존에 있던 `decks_select_own` 정책은 `to authenticated`가 없어도 안전했다 — 조건 자체가
`auth.uid() = user_id`라서, anon은 `auth.uid()`가 null이라 어차피 걸러진다. 이번 정책은
`auth.uid()`를 아예 안 써서 그 안전망이 없었다. **정책 조건에 `auth.uid()`가 없으면 role을
반드시 명시해야 한다**는 걸 이번에 몸으로 배웠다.

Storage 쪽도 똑같이 열려야 파일이 실제로 다운로드된다는 걸 잊지 않았다.

```sql
create policy "decks_storage_select_examples" on storage.objects
  for select to authenticated
  using (
    bucket_id = 'decks' and exists (
      select 1 from public.decks d
      where d.storage_path = storage.objects.name and d.is_example = true
    )
  );
```

DB 행이 보여도 Storage 파일 정책이 따로 안 열리면 다운로드 링크가 만들어지지 않는다 —
"보인다"와 "받을 수 있다"는 이 서비스에서 서로 다른 정책이 지키는 서로 다른 층이었다.

## 남은 숙제

- Cloudflare Tunnel이 지금은 계정 없는 **임시 URL**이라, 재시작하면 주소가 바뀐다. 도메인
  연결한 고정 터널로 옮겨야 진짜로 안정적이다.
- 로컬 컴퓨터가 꺼지거나 잠들면 생성 기능만 멈춘다. 로그인·화면은 계속 떠 있지만, 이 의존성은
  서비스가 커지면 유료 API로 넘어갈 이유가 된다.
- 미리보기는 PPTX와 픽셀 단위로 같지 않다. 데이터는 같지만 렌더링은 다르다는 걸 화면에도
  명시했다.

## 정리하며

오늘 제일 크게 배운 건 "합쳐달라"는 요청을 그대로 코드로 옮기지 않고, 왜 안 되는지부터
설명한 것이었다. 기술적으로 가능한지 안 되는지는 트레이드오프 표 하나면 충분했다.

그리고 RLS는 이번에도 "동작한다"와 "의도한 대로 동작한다"가 다르다는 걸 확인시켜줬다.
정책이 원하는 사람만 통과시키는지는 role까지 같이 봐야 안다 — `using` 절만 읽어서는
누가 걸러지는지 반쪽만 아는 셈이다.

---

코드는 [GitHub](https://github.com/Lightnine999/Web_Development)에 있고, 실제 배포는
[ir-generator-five.vercel.app](https://ir-generator-five.vercel.app)에서 볼 수 있다.
