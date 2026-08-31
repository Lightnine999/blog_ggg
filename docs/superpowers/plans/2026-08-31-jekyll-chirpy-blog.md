# GGG의 학습 기록 (Jekyll + Chirpy 블로그) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `blog_ggg` 저장소에 Chirpy 테마 기반 Jekyll 블로그를 scaffold하고, GitHub Actions로 GitHub Pages에 배포되도록 구성한다.

**Architecture:** 공식 [chirpy-starter](https://github.com/cotes2020/chirpy-starter) 템플릿(테마를 gem으로 참조, fork 아님)의 파일 구성을 그대로 채택하고 `_config.yml`만 사이트 정보로 커스터마이징한다. 배포는 starter에 이미 포함된 `.github/workflows/pages-deploy.yml`(GitHub Actions → Pages)을 그대로 사용한다.

**Tech Stack:** Jekyll, jekyll-theme-chirpy ~> 7.6 (gem), Ruby, GitHub Actions, GitHub Pages

**Spec:** [docs/superpowers/specs/2026-08-31-jekyll-chirpy-blog-design.md](../specs/2026-08-31-jekyll-chirpy-blog-design.md)

## Global Constraints

- title: "GGG의 학습 기록" / author(social.name): "GGG"
- lang: `ko-KR`, timezone: `Asia/Seoul`
- url: `https://lightnine999.github.io`, baseurl: `/blog_ggg`
- github.username: `Lightnine999`
- 배포 브랜치: `main` (starter 워크플로우가 이미 `main`/`master` 둘 다 트리거로 잡음)
- 실제 학습 기록 본문 작성은 범위 밖 — 샘플 포스트 1개만 작성

---

### Task 1: Chirpy starter 파일 스캐폴딩

Chirpy는 GitHub Pages 기본 허용 gem 목록에 없는 플러그인(jekyll-archives 등)을 쓰기 때문에, gem으로 설치했을 때 Jekyll이 읽지 못하는 파일들(`_config.yml`, `_plugins`, `_tabs`, `index.html`, 워크플로우)을 공식 starter 템플릿에서 그대로 가져온다. 이미 `/private/tmp/claude-501/-Users-kwonkwanggoo-aiffel-work/c6f4132e-8249-4684-ad5f-11a1415e1813/scratchpad/chirpy-starter`에 `git clone --depth 1 https://github.com/cotes2020/chirpy-starter.git`로 클론이 되어 있는 상태다 (없다면 같은 명령으로 다시 클론).

**Files:**
- Create: `_config.yml`
- Create: `Gemfile`
- Create: `.gitignore`
- Create: `index.html`
- Create: `LICENSE`
- Create: `_data/contact.yml`
- Create: `_data/share.yml`
- Create: `_plugins/posts-lastmod-hook.rb`
- Create: `_tabs/about.md`
- Create: `_tabs/archives.md`
- Create: `_tabs/categories.md`
- Create: `_tabs/tags.md`
- Create: `.github/workflows/pages-deploy.yml`

**Interfaces:**
- Produces: Jekyll이 빌드에 필요로 하는 사이트 루트 파일 일체. Task 2가 `_config.yml`을 이어서 수정하고, Task 3이 `_posts/`를 새로 추가한다.

- [ ] **Step 1: starter 클론 확인**

```bash
STARTER=/private/tmp/claude-501/-Users-kwonkwanggoo-aiffel-work/c6f4132e-8249-4684-ad5f-11a1415e1813/scratchpad/chirpy-starter
test -f "$STARTER/_config.yml" && echo OK || git clone --depth 1 https://github.com/cotes2020/chirpy-starter.git "$STARTER"
```

Expected: `OK` 출력 (이미 클론되어 있음)

- [ ] **Step 2: 필요한 파일만 blog_ggg로 복사**

```bash
STARTER=/private/tmp/claude-501/-Users-kwonkwanggoo-aiffel-work/c6f4132e-8249-4684-ad5f-11a1415e1813/scratchpad/chirpy-starter
DEST=/Users/kwonkwanggoo/aiffel_work/blog_ggg
cp "$STARTER/_config.yml" "$DEST/_config.yml"
cp "$STARTER/Gemfile" "$DEST/Gemfile"
cp "$STARTER/.gitignore" "$DEST/.gitignore"
cp "$STARTER/index.html" "$DEST/index.html"
cp "$STARTER/LICENSE" "$DEST/LICENSE"
mkdir -p "$DEST/_data" "$DEST/_plugins" "$DEST/_tabs" "$DEST/.github/workflows"
cp "$STARTER/_data/contact.yml" "$DEST/_data/contact.yml"
cp "$STARTER/_data/share.yml" "$DEST/_data/share.yml"
cp "$STARTER/_plugins/posts-lastmod-hook.rb" "$DEST/_plugins/posts-lastmod-hook.rb"
cp "$STARTER/_tabs/about.md" "$DEST/_tabs/about.md"
cp "$STARTER/_tabs/archives.md" "$DEST/_tabs/archives.md"
cp "$STARTER/_tabs/categories.md" "$DEST/_tabs/categories.md"
cp "$STARTER/_tabs/tags.md" "$DEST/_tabs/tags.md"
cp "$STARTER/.github/workflows/pages-deploy.yml" "$DEST/.github/workflows/pages-deploy.yml"
```

Expected: 명령이 에러 없이 종료

- [ ] **Step 3: 복사 결과 확인**

```bash
cd /Users/kwonkwanggoo/aiffel_work/blog_ggg && find . -not -path './.git*' -not -path './docs*' -type f | sort
```

Expected: 위 Files 목록의 13개 파일이 모두 보임

- [ ] **Step 4: README.md를 블로그용으로 새로 작성**

`/Users/kwonkwanggoo/aiffel_work/blog_ggg/README.md` (starter의 원본 README를 그대로 쓰지 않고 이 블로그 전용으로 교체):

```markdown
# GGG의 학습 기록

AIFFEL 학습 기록을 정리하는 개인 블로그입니다.

- 테마: [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy)
- 배포: GitHub Actions → GitHub Pages
- 주소: https://lightnine999.github.io/blog_ggg/

## 새 글 작성

`_posts/YYYY-MM-DD-제목.md` 형식으로 파일을 추가하세요. front matter 예시는
`_posts/`의 기존 글을 참고하세요.
```

- [ ] **Step 5: 커밋**

```bash
cd /Users/kwonkwanggoo/aiffel_work/blog_ggg
git add _config.yml Gemfile .gitignore index.html LICENSE _data _plugins _tabs .github README.md
git commit -m "scaffold: chirpy-starter 기반 Jekyll 블로그 뼈대 추가"
```

---

### Task 2: `_config.yml` 사이트 정보 커스터마이징

**Files:**
- Modify: `_config.yml`

**Interfaces:**
- Consumes: Task 1이 생성한 `_config.yml` (starter 원본, 모든 값이 placeholder)
- Produces: 사이트 메타데이터가 채워진 `_config.yml`. Task 3의 샘플 포스트가 이 설정(baseurl, permalink 규칙)을 전제로 렌더링된다.

- [ ] **Step 1: 값 치환**

`_config.yml`에서 아래 필드를 정확히 이 값으로 바꾼다 (나머지 필드는 starter 기본값 그대로 둔다):

| 필드 | 원본 (starter 기본값) | 변경 후 |
|---|---|---|
| `lang` | `en` | `ko-KR` |
| `timezone` | (빈 값) | `Asia/Seoul` |
| `title` | `Chirpy` | `GGG의 학습 기록` |
| `tagline` | `A text-focused Jekyll theme` | `AIFFEL 학습 기록` |
| `description` | (기본 설명) | `AIFFEL에서 배운 내용을 주제별로 정리하는 학습 블로그입니다.` |
| `url` | `""` | `https://lightnine999.github.io` |
| `github.username` | `github_username` | `Lightnine999` |
| `social.name` | `your_full_name` | `GGG` |
| `social.email` | `example@domain.com` | (사용자가 나중에 직접 채우도록 빈 문자열 `""`로 둠 — 개인 이메일을 코드에 하드코딩하지 않음) |
| `social.links` | twitter/github placeholder 2줄 | `https://github.com/Lightnine999` 한 줄만 남기고 twitter 줄은 삭제 |
| `baseurl` | `""` | `/blog_ggg` |

Edit 예시 (일부):

```yaml
lang: ko-KR

timezone: Asia/Seoul

title: GGG의 학습 기록

tagline: AIFFEL 학습 기록

description: >-
  AIFFEL에서 배운 내용을 주제별로 정리하는 학습 블로그입니다.

url: "https://lightnine999.github.io"

github:
  username: Lightnine999

social:
  name: GGG
  email: ""
  fediverse_handle:
  links:
    - https://github.com/Lightnine999
```

그리고 twitter 섹션은 다음과 같이 비운다:

```yaml
twitter:
  username:
```

파일 하단의 `baseurl: ""`도 `baseurl: "/blog_ggg"`로 바꾼다.

- [ ] **Step 2: YAML 문법 검증**

```bash
cd /Users/kwonkwanggoo/aiffel_work/blog_ggg
ruby -ryaml -e "YAML.load_file('_config.yml'); puts 'YAML OK'"
```

Expected: `YAML OK` 출력 (에러 없음)

- [ ] **Step 3: 커밋**

```bash
git add _config.yml
git commit -m "config: 사이트 제목/언어/URL 등 GGG 블로그 정보로 설정"
```

---

### Task 3: 샘플 학습 기록 포스트 + about 탭 내용 작성

**Files:**
- Create: `_posts/2026-08-31-hello-world.md`
- Modify: `_tabs/about.md`

**Interfaces:**
- Consumes: Task 2에서 설정한 `permalink: /posts/:title/` 규칙 (starter `_config.yml`의 `defaults` 블록, 수정하지 않음)
- Produces: `categories: [주제, 세부주제]` 형식 예시 — 이후 사용자가 직접 추가하는 글들이 참고할 템플릿

- [ ] **Step 1: 샘플 포스트 작성**

`/Users/kwonkwanggoo/aiffel_work/blog_ggg/_posts/2026-08-31-hello-world.md`:

```markdown
---
title: 블로그를 시작하며
date: 2026-08-31 15:30:00 +0900
categories: [Python, 기초]
tags: [aiffel, 시작]
---

AIFFEL 학습 기록을 정리할 블로그를 시작합니다.

## 카테고리 사용법

글 앞부분(front matter)의 `categories`에 `[대주제, 소주제]` 형식으로 최대 2단계까지
분류를 적을 수 있습니다. 예를 들어 이 글은 `Python > 기초`로 분류되어 있습니다.

앞으로 학습 주제에 따라 아래와 같은 카테고리를 사용할 예정입니다.

- `Python`
- `딥러닝`
- `프로젝트`

## 다음 할 일

- [ ] 첫 실제 학습 기록 작성하기
- [ ] 카테고리 체계를 학습 진도에 맞게 다듬기
```

- [ ] **Step 2: about 탭 내용 작성**

`/Users/kwonkwanggoo/aiffel_work/blog_ggg/_tabs/about.md`:

```markdown
---
# the default layout is 'page'
icon: fas fa-info-circle
order: 4
---

AIFFEL에서 학습한 내용을 정리하고 기록하기 위한 블로그입니다.

주로 Python, 딥러닝, 프로젝트 진행 과정을 카테고리별로 정리합니다.
```

- [ ] **Step 3: front matter YAML 검증**

```bash
cd /Users/kwonkwanggoo/aiffel_work/blog_ggg
ruby -ryaml -e "
content = File.read('_posts/2026-08-31-hello-world.md')
fm = content.split('---')[1]
YAML.load(fm)
puts 'front matter OK'
"
```

Expected: `front matter OK` 출력

- [ ] **Step 4: 커밋**

```bash
git add _posts/2026-08-31-hello-world.md _tabs/about.md
git commit -m "content: 샘플 학습 기록 포스트와 about 탭 내용 추가"
```

---

### Task 4: 로컬 빌드 검증 (best-effort) 및 fallback 결정

시스템 Ruby가 2.6.10으로 오래되어 Chirpy 7.x(Jekyll 4.3+ 요구)가 로컬에서 안 돌아갈 수 있다. 될 때까지 억지로 맞추지 않고, 안 되면 CI(GitHub Actions, Ruby 3.4)로 검증을 넘긴다.

**Files:** (없음 — 검증만 수행)

**Interfaces:**
- Consumes: Task 1~3에서 만든 모든 파일
- Produces: 로컬 빌드 성공 여부 판단 결과. 실패 시 Task 5에서 "push 후 Actions 로그로 확인"으로 대체.

- [ ] **Step 1: bundle install 시도**

```bash
cd /Users/kwonkwanggoo/aiffel_work/blog_ggg
bundle config set --local path 'vendor/bundle'
bundle install 2>&1 | tail -30
```

Expected: 성공(`Bundle complete!`) 또는 Ruby 버전 불일치 에러(`your Ruby version is X, required Y`) 둘 중 하나

- [ ] **Step 2a (bundle install 성공 시): 로컬 빌드**

```bash
cd /Users/kwonkwanggoo/aiffel_work/blog_ggg
JEKYLL_ENV=production bundle exec jekyll b -d "_site/blog_ggg" --baseurl "/blog_ggg" 2>&1 | tail -40
```

Expected: `Done in` 로그와 함께 종료 코드 0. `_site/blog_ggg/index.html`, `_site/blog_ggg/posts/블로그를-시작하며/index.html`(또는 slug화된 경로) 생성 확인:

```bash
find /Users/kwonkwanggoo/aiffel_work/blog_ggg/_site -name "index.html" | head -10
```

- [ ] **Step 2b (bundle install 실패 시): fallback 기록**

로컬 빌드를 건너뛰고, Task 5에서 push 후 `gh run watch`로 GitHub Actions 빌드 로그를 확인하는 것으로 검증을 대체한다는 점을 사용자에게 알린다. 별도 코드 변경 없음.

- [ ] **Step 3: `_site`가 생성됐다면 정리**

```bash
rm -rf /Users/kwonkwanggoo/aiffel_work/blog_ggg/_site
```

(`.gitignore`에 `_site`가 이미 포함되어 있어 커밋되지는 않지만, 로컬 디스크 정리 차원)

---

### Task 5: 원격 배포 (사용자 승인 필요)

이 태스크의 두 단계 모두 안전 규칙상 **사용자에게 명시적으로 확인받은 뒤에만 실행**한다: (a) push는 "다른 사람이 볼 수 있는 상태로 만드는 행위", (b) Pages 소스 변경은 "계정/저장소 설정 변경"에 해당한다. 실행 전 각각 채팅으로 확인을 구한다.

**Files:** (없음 — 원격 작업)

**Interfaces:**
- Consumes: Task 1~4에서 로컬에 커밋된 모든 변경
- Produces: 배포된 GitHub Pages 사이트

- [ ] **Step 1 (승인 후): 원격 push**

```bash
cd /Users/kwonkwanggoo/aiffel_work/blog_ggg
git push -u origin main
```

Expected: push 성공, `origin/main`이 로컬 `main`을 가리킴

- [ ] **Step 2 (승인 후): GitHub Pages 소스를 Actions로 변경**

```bash
gh api repos/Lightnine999/blog_ggg/pages -X PUT -f build_type=workflow 2>&1
# 위 명령이 404를 반환하면(Pages가 아직 한 번도 활성화된 적 없음) 아래로 최초 활성화
gh api repos/Lightnine999/blog_ggg/pages -X POST -f build_type=workflow 2>&1
```

Expected: 200/201 응답, 이후 `gh api repos/Lightnine999/blog_ggg/pages --jq .build_type`이 `workflow` 출력

- [ ] **Step 3: Actions 빌드 확인**

```bash
cd /Users/kwonkwanggoo/aiffel_work/blog_ggg
gh run list --branch main --limit 3
gh run watch $(gh run list --branch main --limit 1 --json databaseId --jq '.[0].databaseId')
```

Expected: 워크플로우 `Build and Deploy`가 `completed` / `success`로 종료

- [ ] **Step 4: 배포 확인**

```bash
curl -sI https://lightnine999.github.io/blog_ggg/ | head -5
```

Expected: `HTTP/2 200`

- [ ] **Step 5: 임시 클론 정리**

```bash
rm -rf /private/tmp/claude-501/-Users-kwonkwanggoo-aiffel-work/c6f4132e-8249-4684-ad5f-11a1415e1813/scratchpad/chirpy-starter
```
