---
layout: about
title: about
permalink: /
subtitle: ELION LAB · Konkuk University

selected_papers: false # includes a list of papers marked as "selected={true}"
social: true # includes social icons at the bottom of the page

announcements:
  enabled: false # includes a list of news items

latest_posts:
  enabled: true
  scrollable: false # adds a vertical scroll bar if there are more than 3 new posts items
  limit: 5 # leave blank to include all the blog posts
---

건국대학교 ELION LAB에서 **언어 에이전트**를 공부하고 있습니다.
에이전트가 가진 기술(Skill)이 어떤 형태로 표현되는지, 그리고 그 기술을 고르는 언어가 실제 행동과 어디서 어긋나는지에 관심이 있습니다.

이 블로그는 논문을 읽고, 코드를 직접 돌리고, 실패한 지점을 기록하는 공부 노트입니다.

## 공부 기록

<div class="series-cards">
  <a class="series-card" href="{% post_url 2026-11-05-alfworld-first-play %}">
    <span class="series-card-label">Series 1 · 4편</span>
    <span class="series-card-title">ALFWorld에서 본 언어와 행동</span>
    <span class="series-card-desc">텍스트 게임을 직접 플레이하고, ReAct · Reflexion을 로컬 LLM으로 돌려 실패 원인을 통제 실험으로 좁힌 기록</span>
  </a>
  <a class="series-card" href="{% post_url 2026-11-14-voyager-code-as-skill %}">
    <span class="series-card-label">Series 2 · 4편</span>
    <span class="series-card-title">마인크래프트 에이전트 논문 읽기</span>
    <span class="series-card-desc">Voyager · VPT · STEVE-1을 같은 항목으로 정리하고, Skill이 자연어 · 코드 · 신경망 가중치일 때를 비교한 기록</span>
  </a>
  <a class="series-card" href="{% post_url 2026-11-22-steve1-baseline-reproduction %}">
    <span class="series-card-label">Series 3 · 3편</span>
    <span class="series-card-title">STEVE-1 직접 돌려 보기</span>
    <span class="series-card-desc">공개 가중치로 논문 숫자를 재현하고, 이후에 나온 논문 세 편을 읽은 뒤, 같은 일을 여덟 가지 문장으로 시켜 행동이 어떻게 달라지는지 확인한 기록</span>
  </a>
</div>

## 지금 보고 있는 것

- **Skill의 표현 방식** — 자연어 메모(Reflexion), 실행 코드(Voyager), 신경망 가중치(VPT · STEVE-1)
- **언어–행동 불일치** — 지시문의 단어 하나가 성공률을 바꾸는 현상과 그 원인
- **STEVE-1 지시문 실험** — 같은 뜻의 지시문 사이에서 나무 수집량이 평균 0.4개에서 6.0개까지 갈렸고, "don't chop trees"는 나무를 캤습니다. 지시문마다 5판인 예비 실험입니다
- **다음** — 판 수를 늘리고, 다른 과제와 다른 부정 표현으로 넓혀 원인을 확인하기
