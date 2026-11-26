# Jungmin's research notes

언어 에이전트와 Skill을 공부하며 남기는 기록입니다. 논문을 읽고, 코드를 직접 돌리고, 실패한 지점을 적습니다.

**사이트:** <https://chajm12.github.io>

## 글 목록

### Series 1 · ALFWorld에서 본 언어와 행동

텍스트 게임을 직접 플레이하고, ReAct · Reflexion을 로컬 LLM으로 돌려 실패 원인을 통제 실험으로 좁힌 기록입니다.

| #   | 글                                                                                                         | 내용                                                               |
| --- | ---------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| 1   | [ALFWorld를 직접 플레이하며 찾은 실패 다섯 가지](https://chajm12.github.io/blog/2026/alfworld-first-play/) | 에이전트를 돌리기 전에 사람이 먼저 풀어 보며 막히는 곳을 기록      |
| 2   | [ReAct와 Reflexion 논문 정리](https://chajm12.github.io/blog/2026/react-reflexion-papers/)                 | 두 논문을 같은 항목(데이터, 입력과 출력, Skill, 평가)으로 정리     |
| 3   | [Reflexion을 맥북 로컬 LLM으로 돌려 보기](https://chajm12.github.io/blog/2026/reflexion-local-ollama/)     | Ollama + Qwen2.5-7B로 실행, 15판 0승의 원인을 통제 실험 1–4로 추적 |
| 4   | [단어 하나가 바꾼 성공률](https://chajm12.github.io/blog/2026/one-word-success-rate/)                      | 지시문의 동사 하나로 45% → 80%, 환경 쪽 변환으로 100% (실험 5–8)   |

### Series 2 · 마인크래프트 에이전트 논문 읽기

Skill이 자연어가 아니라 실행 코드나 신경망 가중치로 표현될 때 무엇이 달라지는지 정리한 기록입니다.

| #   | 글                                                                                                     | 내용                                                            |
| --- | ------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------- |
| 1   | [Voyager 논문 정리](https://chajm12.github.io/blog/2026/voyager-code-as-skill/)                        | 성공한 코드를 skill library에 쌓는 에이전트                     |
| 2   | [VPT 논문 정리](https://chajm12.github.io/blog/2026/vpt-video-pretraining/)                            | 라벨 없는 영상 7만 시간으로 키보드와 마우스를 배우는 모델       |
| 3   | [STEVE-1 논문 정리](https://chajm12.github.io/blog/2026/steve1-text-to-behavior/)                      | VPT에 지시문을 받는 입력을 붙인 모델, 지시문 표현에 대한 민감도 |
| 4   | [Skill이 글 · 코드 · 가중치일 때](https://chajm12.github.io/blog/2026/skill-representations-compared/) | 여섯 편 비교와 STEVE-1 지시문 실험 설계                         |

### Series 3 · STEVE-1 직접 돌려 보기

공개된 STEVE-1 가중치를 실행해 논문 숫자를 재현하고, 지시문 표현을 바꿔 가며 행동이 어떻게 달라지는지 확인한 기록입니다.

| #   | 글                                                                                                          | 내용                                                           |
| --- | ----------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| 1   | [STEVE-1을 직접 돌려 논문 숫자 재현하기](https://chajm12.github.io/blog/2026/steve1-baseline-reproduction/) | 논문 부록의 평가 설정으로 20판, 흙은 재현되고 나무는 낮게 나옴 |
| 2   | [STEVE-1 이후 논문 세 편](https://chajm12.github.io/blog/2026/after-steve1-three-papers/)                   | GROOT-2 · ROCKET-1 · CrafText가 본 언어 지시의 한계            |
| 3   | [같은 뜻, 다른 문장](https://chajm12.github.io/blog/2026/steve1-instruction-pilot/)                         | 지시문 8개 × 5판. 같은 뜻인데 6.0개 vs 0.4개, 부정문은 무시됨  |

## 저장소 구조

글과 디자인을 고칠 때 손대는 파일은 아래가 전부입니다.

| 경로                      | 역할                                                                   |
| ------------------------- | ---------------------------------------------------------------------- |
| `_posts/`                 | 블로그 글 (Markdown). 파일 이름은 `YYYY-MM-DD-제목.md`                 |
| `_pages/about.md`         | 첫 화면                                                                |
| `_data/series.yml`        | 시리즈 이름과 순서 정의                                                |
| `_includes/series.liquid` | 글 맨 위에 들어가는 시리즈 목차 상자                                   |
| `_sass/_custom.scss`      | 색 · 글꼴 · 표 · 코드 상자 · 시리즈 상자 스타일. 색을 바꾸려면 이 파일 |
| `assets/css/main.scss`    | 테마 기본 스타일에 `_custom.scss`를 연결하는 파일                      |
| `assets/img/`             | 글에 쓰는 이미지                                                       |
| `_config.yml`             | 사이트 제목, 메뉴, 빌드에서 제외할 파일                                |

al-folio 템플릿의 데모 페이지와 데모 데이터(`_projects/`, `_news/`, `_pages/publications.md` 등)는 지우지 않고 `_config.yml`의 `exclude`에 넣어 빌드에서만 뺐습니다. 다시 쓰려면 해당 줄을 지우면 됩니다.

## 새 글 쓰기

`_posts/`에 파일을 만들고 맨 위에 아래 내용을 넣습니다.

```yaml
---
layout: post
title: "글 제목"
date: 2026-11-20 18:00:00 +0900
description: 글 목록과 링크 미리보기에 보이는 한두 문장
tags: minecraft agent paper
categories: notes
series: minecraft # _data/series.yml 에 정의한 키. 시리즈가 아니면 이 세 줄을 뺀다
series_order: 5
series_title: "목차 상자에 보일 짧은 제목"
toc:
  sidebar: left
---

{% include series.liquid %}
```

- `series`를 넣은 글은 `{% include series.liquid %}` 한 줄로 시리즈 목차 상자가 생깁니다.
- 새 시리즈를 만들려면 `_data/series.yml`에 키와 제목을 추가합니다.
- `date`가 미래이면 그 시각이 지나 다시 빌드될 때까지 글이 보이지 않습니다.

## 로컬에서 미리 보기

```bash
bundle install
bundle exec jekyll serve
# http://localhost:4000 에서 확인
```

Ruby 환경이 없다면 Docker로도 볼 수 있습니다.

```bash
docker compose up
# http://localhost:8080 에서 확인
```

## 배포

`main` 브랜치에 push하면 GitHub Actions(`.github/workflows/deploy.yml`)가 사이트를 빌드해 GitHub Pages에 올립니다.

```bash
git add .
git commit -m "Add post: ..."
git push
```

push 전에 서식을 맞추려면 `npx prettier . --write`를 실행합니다. 저장소의 Prettier 검사가 같은 규칙으로 돌아갑니다.

## 사용한 것

- [al-folio](https://github.com/alshedivat/al-folio) v1 (Jekyll 스타터). 원본의 MIT 라이선스는 [LICENSE](LICENSE)에 그대로 있습니다.
- 글꼴: [Pretendard](https://github.com/orioncactus/pretendard)
