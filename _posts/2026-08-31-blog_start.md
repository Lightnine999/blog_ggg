---
title: 블로그를 생성하며
date: 2026-08-31 15:50:00 +0900
categories: [Claude_code, 기초]
tags: [aiffel, 시작]
---

Claude Code와 함께 Jekyll + Chirpy 테마로 블로그를 만든 과정을 정리해봅니다.

## 블로그 제작 과정

1. blog_ggg 저장소 상태 확인 (빈 저장소, origin 이미 연결됨)
2. 브레인스토밍으로 방향 결정 — Jekyll + Chirpy 테마, 현재 저장소 그대로 배포
3. 블로그 제목("GGG의 학습 기록") · 초기 콘텐츠 · 카테고리 구조 확인
4. 설계 문서 작성 및 커밋 (`docs/superpowers/specs/`)
5. 공식 chirpy-starter 템플릿을 로컬에 클론해서 실제 구조 확인
6. 구현 계획 문서 작성 및 커밋 (`docs/superpowers/plans/`)
7. chirpy-starter 파일들(`_config.yml`, `Gemfile`, 워크플로우 등) 복사 + README 작성 → 커밋
8. `_config.yml` 커스터마이징 (제목/언어/타임존/URL/작성자 등) → 커밋
9. 샘플 포스트("블로그를 시작하며") + about 탭 내용 작성 → 커밋
10. 로컬 빌드 검증 시도 (시스템 Ruby가 오래돼서 실패 → CI로 대체 결정)
11. `git push` 시도 → 개인 이메일 노출로 GitHub에 거부됨
12. 커밋 작성자 이메일을 GitHub noreply 주소로 변경(rebase) 후 재push 성공
13. GitHub Pages를 "GitHub Actions" 빌드 소스로 활성화
14. 첫 Actions 실행 실패 확인 (Pages 활성화 전에 push돼서) → 재실행
15. 재실행 성공 확인 (build → deploy 모두 성공)
16. 브라우저로 실제 배포된 블로그(<https://lightnine999.github.io/blog_ggg/>) 접속·확인
17. 임시로 클론했던 chirpy-starter 파일 정리

## 이번 작업에서 다룬 주제

- `Claude_code`
- `Git & Github`
- `blog`

## 다음 할 일

- [ ] 첫 실제 학습 기록 작성하기
- [ ] 카테고리 체계를 학습 진도에 맞게 다듬기
