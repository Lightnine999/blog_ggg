# GGG의 학습 기록 — Jekyll + Chirpy 블로그 설계

- 날짜: 2026-08-31
- 저장소: https://github.com/Lightnine999/blog_ggg
- 상태: APPROVED

## 목적

AIFFEL 학습 기록을 주제별로 정리해 게시하는 개인 학습 블로그. GitHub Pages로 무료 호스팅.

## 스택

- **정적 사이트 생성기**: Jekyll
- **테마**: [jekyll-theme-chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) — gem 기반 참조 (테마 저장소를 fork하지 않고 `chirpy-starter` 구조로 구성해 테마 업데이트를 gem 버전 업으로 처리)
- **배포**: GitHub Actions 빌드 → `gh-pages` 브랜치 배포 (Chirpy가 GitHub Pages 기본 허용 gem 목록에 없는 플러그인을 사용하므로 네이티브 Jekyll 빌드 대신 Actions 필요)

## 배포 경로

- 저장소명이 `blog_ggg`이므로 프로젝트 페이지로 배포: `https://lightnine999.github.io/blog_ggg/`
- `_config.yml`: `url: "https://lightnine999.github.io"`, `baseurl: "/blog_ggg"`
- 저장소 Settings → Pages → Source를 "GitHub Actions"로 변경 필요 (계정 설정 변경이라 사용자 승인 후 실행)

## 디렉토리 구조

```
blog_ggg/
├── _config.yml
├── Gemfile
├── Gemfile.lock (bundle install 후 생성)
├── _posts/
│   └── 2026-08-31-hello-world.md   # 샘플 포스트
├── _tabs/
│   └── about.md
├── assets/
├── .github/workflows/pages-deploy.yml
├── .gitignore
└── README.md
```

## 설정값 (_config.yml)

- title: "GGG의 학습 기록"
- author: "GGG"
- lang: `ko-KR`
- timezone: `Asia/Seoul`
- url / baseurl: 위 배포 경로 참조

## 콘텐츠 구조

- `_posts/`의 각 글은 front matter `categories: [주제, 세부주제]`로 분류
- 초기 카테고리 예시 체계: `Python`, `딥러닝`, `프로젝트` 등 — 실제 학습 진행에 따라 확장
- 샘플 포스트 1개(`hello-world`)로 작성법(카테고리, 태그, front matter) 시연

## 검증

- 로컬 Ruby/Bundler 존재 시: `bundle exec jekyll serve`로 로컬 렌더링 확인
- 부재 시: push 후 GitHub Actions 워크플로우 빌드 로그로 확인

## 범위 밖 (Out of scope)

- 커스텀 도메인 연결
- 실제 학습 기록 본문 작성 (사용자가 이후 직접 추가)
- 댓글 시스템, 검색 등 Chirpy 고급 기능 커스터마이징
