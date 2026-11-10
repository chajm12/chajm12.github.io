---
layout: post
title: "단어 하나가 바꾼 성공률 — Reflexion 통제 실험 5–8"
date: 2026-11-10 09:00:00 +0900
description: 같은 모델, 같은 20게임에서 태스크 지시문의 동사 하나를 바꾸자 성공률이 45%에서 80%로, 환경 쪽에서 문법을 번역하자 100%가 됐습니다.
tags: alfworld agent reflexion ollama qwen llm prompt
categories: notes
toc:
  sidebar: left
---

> **이 글에 대해**
> [앞 글]({% post_url 2026-11-09-reflexion-local-ollama %})의 질문을 이어서 풉니다. 프롬프트 예시에서 put을 다 지웠는데도 Qwen은 왜 `put … in/on`을 썼을까요?
> 환경 파일과 프롬프트 구조를 직접 열어 보고, 같은 20게임에서 한 번에 한 가지만 바꾸는 실험을 네 개 더 돌렸습니다.
> 모델은 qwen2.5:7b 하나, 실행은 한 번씩입니다.

**용어.** 이 글에서 **태스크**는 ALFWorld 게임 한 판에서 풀어야 할 일입니다. **태스크 지시문**은 화면에 나오는 `Your task is to: …` 뒤의 문장이고, 코드에서는 `task_desc`라고 부릅니다.

## 1. 환경은 어디에 저장되고 어떻게 불러오나

ALFWorld 데이터(약 2.2GB)는 `~/.cache/alfworld`에 있습니다.

```
~/.cache/alfworld/
├── logic/
│   ├── alfred.pddl     ← 행동 규칙 (cool은 냉장고 앞에서만 등 전제조건)
│   └── alfred.twl2     ← 문장 틀 ("move {o} to {r}", "You move the {o} to the {r}.")
└── json_2.1.1/{train, valid_seen, valid_unseen}/
    └── pick_cool_then_place_in_recep-Bread-.../trial_.../
        ├── game.tw-pddl    ← 게임 한 판 (규칙 · 문장 틀 · 초기 상태와 목표 · 정답 풀이)
        └── traj_data.json  ← 원래 ALFRED 데이터
```

불러오는 순서는 이렇습니다.

1. 설정 파일(`base_config.yaml`)에서 데이터 경로를 읽습니다.
2. `get_environment("AlfredTWEnv")`가 `valid_unseen` 폴더의 게임 파일들을 모읍니다.
3. `env.reset()`이 다음 `game.tw-pddl`을 열어 첫 화면 문장을 돌려줍니다. 이 문장 끝에 `Your task is to: …`가 붙어 있습니다.
4. 에이전트 코드가 그 문장을 프롬프트 끝에 붙입니다.
5. 매 행동(`env.step`)마다 환경이 처리합니다.
   - 문장 틀로 명령을 해석합니다.
   - PDDL 규칙으로 전제조건을 확인합니다.
   - 결과 문장을 돌려줍니다. 해석이 안 되거나 조건이 안 맞으면 `Nothing happens.`입니다.

**태스크 지시문은 게임 파일에 이미 박혀 있습니다.** alfworld 패키지의 `gen/goal_library.py`에 있는 틀(`'put a cool {obj} in {recep}'`, `'cool some {obj} and put it in {recep}'`) 중 하나를 골라 게임 파일을 만들 때 넣어 둡니다.
반면 **성공 판정은 이 문장이 아니라 PDDL 목표(논리식)**로 합니다. 그래서 지시문을 바꿔도 채점 기준은 그대로입니다.

환경 안에서부터 말이 어긋나 있는 셈입니다. 지시문 틀은 아직 `put`인데, 명령 틀은 `move`로 바뀌었습니다.

## 2. LLM이 받는 프롬프트에는 태스크 지시문이 세 번 나온다

```
Interact with a household to solve a task. Here are two examples.

[예시 1] You are in the middle of a room ...
         Your task is to: put some spraybottle on toilet.            ① 예시 파일
         > go to cabinet 1 ...   (정답 풀이 기록)
[예시 2] ...
         Your task is to: find some apple and put it in sidetable.   ② 예시 파일

Your memory for the task below:
Trial 0: ... (반성문, 최근 3개까지)

Here is the task:
         You are in the middle of a room ...
         Your task is to: put a cool bread in countertop.            ③ env.reset()
>
```

①②는 예시 파일(ReAct 저자가 미리 적어 둔 다른 게임의 풀이)에서, ③은 지금 풀 게임 파일에서 옵니다. 앞 글의 실험 4까지는 ①②③에 put이 그대로 남아 있었습니다.

무엇을 바꾸느냐에 따라 실험의 의미도 달라집니다.

| 층 | 바꾸는 것 | 이 글의 실험 | 의미 |
| --- | --- | --- | --- |
| 환경 | 게임이 알아듣는 명령 규칙 | 실험 7 코드 번역 | 게임이 put을 받던 논문 당시 조건을 재현 → 기준점 |
| 태스크 (지시문) | 에이전트에게 주는 목표 문장 | 실험 5 지시문까지 move | 시험 문제를 고쳐 쓴 것 → 벤치마크 점수로는 못 씀, 원인 분석용 |
| 에이전트 | 프롬프트 예시·지시 | 실험 2–4, 실험 8 프롬프트 힌트 | 참고서를 바꾼 것 → 프롬프트 엔지니어링 |

## 3. 실험 5–8

| 실험 | 예시(few-shot) | 태스크 지시문 | put 처리 |
| --- | --- | --- | --- |
| 4 (앞 글) | move (지시문 줄만 put) | put | 없음 |
| 5 | 전부 move | **move** | LLM이 보는 글자에 put 0개 |
| 7 코드 번역 | put (원본) | put (원본) | Qwen이 쓴 `put X in/on Y`를 게임에 보내기 직전 코드가 `move X to Y`로 번역 |
| 8 프롬프트 힌트 | put (원본) | put (원본) | 프롬프트 맨 앞에 힌트 한 줄 추가 |

실험 7의 번역은 LLM이 아니라 정규식 한 줄로 했습니다. 바꿀 형태가 정해져 있어서 규칙이면 충분하고, 빠르고 틀리지 않습니다. 기록에는 Qwen이 쓴 put을 그대로 남기므로, Qwen은 자기가 쓴 put이 통했다고만 알게 됩니다.

```python
if TRANSLATE and action.startswith('put '):
    action_env = re.sub(r'^put (.+?) (?:in/on|into|in|on) (.+)$', r'move \1 to \2', action)
env.step([action_env])     # put tomato 1 in/on microwave 1 → move tomato 1 to microwave 1
```

실험 8에서 프롬프트 맨 앞에 넣은 힌트 문장은 이것입니다.

```
Note: this environment does not accept 'put (object) in/on (receptacle)'.
To place an object, use 'move (object) to (receptacle)' instead.
```

## 4. 결과 — 같은 20게임

{% include figure.liquid loading="eager" path="assets/img/reflexion_success_rates.png" class="img-fluid rounded z-depth-1" zoomable=true %}
<div class="caption">
ALFWorld valid unseen 앞 20게임, qwen2.5:7b, 게임당 최대 3회 시도.
</div>

| 실험 | 첫 시도 | 2회 후 | 3회 후 | put 명령 | move 명령 |
| --- | --- | --- | --- | --- | --- |
| 4 예시만 move | 4/20 | 6/20 | 9/20 | 121 | 7 |
| 8 프롬프트 힌트 | 6/20 | 9/20 | 10/20 | 67 | 82 |
| 5 지시문까지 move | 13/20 | 15/20 | 16/20 | 18 | 61 |
| **7 코드 번역** | **18/20** | **19/20** | **20/20** | 18 (전부 번역되어 성공) | 0 |

**1) 태스크 지시문의 동사 하나로 45%가 80%가 됐습니다.** 첫 시도만 보면 20%에서 65%입니다. Reflexion으로 3번 시도해서 오른 폭(+25%p)보다 동사 하나를 바꾼 효과(+35%p)가 더 컸습니다. 앞 글에서 남은 질문의 답은 "put의 상당 부분은 태스크 지시문에서 왔다"입니다.

**2) 원본 그대로 두고 번역만 하면 20/20이었습니다.** Qwen은 게임마다 put을 딱 한 번, 맞는 자리에서만 썼습니다. 원본 프롬프트의 실패는 거의 전부 모델 능력이 아니라 환경 버전의 문법 차이 때문이었습니다.
흥미롭게도 프롬프트를 move로 고친 것(80%)보다 원본을 그대로 두고 번역한 것(100%)이 더 잘 됐습니다. 예시를 고치면 상태 추적 실수가 늘었습니다. Qwen이 원본 ReAct 프롬프트 형식에 가장 익숙하다는 뜻일 수 있습니다.

**3) 힌트 한 줄은 예시 36줄을 거의 이기지 못했습니다(50%).** Qwen은 put과 move를 섞어 썼습니다. 실패한 판 33개 중 28개에서 put을 먼저 썼고, put이 실패한 뒤에야 move를 꺼냈습니다.

**4) 태스크 유형별로 보면 Clean이 약점입니다.** 지시문까지 move로 바꾼 실험 5에서 끝까지 실패한 4게임 중 3게임이 Clean 태스크였습니다.

| 태스크 유형 | 실험 4 | 실험 5 |
| --- | --- | --- |
| Pick & Place | 3/6 | 6/6 |
| Heat | 1/3 | 3/3 |
| Cool | 1/5 | 4/5 |
| Clean | 2/4 | 1/4 |
| Look (desklamp) | 2/2 | 2/2 |

## 5. 실패 분석 — 문법이 풀리면 다음은 상태 추적

실패한 판의 행동을 따라가며, 들고 있는 물건과 현재 위치를 추적하는 스크립트를 짰습니다. 그리고 `Nothing happens.`가 나온 행동을 규칙으로 분류했습니다. 규칙 기반이라 근사치입니다.

{% include figure.liquid loading="lazy" path="assets/img/reflexion_failure_types.png" class="img-fluid rounded z-depth-1" zoomable=true %}

문법 실패가 줄어들수록 **내가 지금 무엇을 들고 있는지**를 놓치는 실패가 중심이 됩니다. 이미 들고 있는데 또 집으려 하거나, 집지 않은 채 move나 clean을 합니다.
환경은 이것도 `Nothing happens.` 한 문장으로만 알려 줍니다. [첫 글]({% post_url 2026-11-05-alfworld-first-play %})에서 직접 플레이하며 겪은 실패와 같은 종류입니다.

로그를 직접 읽으며 본 장면들입니다.

**같은 단어, 다른 뜻 범위 (실험 8)**

```
> use desklamp 1
You turn on the desklamp 1.
> move bowl 2 to desk 1
Nothing happens.
```

일상 영어의 move는 "집어서 옮겨 놓기"까지 뜻하지만, 환경의 move는 "들고 있는 것을 놓기"만 뜻합니다. 집지 않은 채 move를 쓴 횟수가 57회였습니다.

**생각과 명령이 따로 논다 (실험 8)**

```
> think: I will put the clean spatula 1 on the shelf 3.
> move spatula 1 to shelf 3
Nothing happens.
```

생각에는 put이라고 쓰고, 명령은 move로 합니다. 힌트는 명령어에만 반영되고 계획을 세우는 말은 그대로였습니다.

**프롬프트에 없는 put (실험 5)**

```
> cool tomato 1 with microwave 1
Nothing happens.
> think: ... I will place the tomato on the countertop instead.
> put tomato 1 on countertop 2
```

예시에도 태스크 지시문에도 put이 없는데, 계획이 틀어지자 일상 영어로 돌아갔습니다. 이 게임에서 17번 나왔습니다.

**반성문 복사 (실험 5·6)**

```
Trial 0: ...I was unsuccessful in finding a clean plate...
Trial 1: (글자까지 동일)
Trial 2: (글자까지 동일)
```

반성문은 이번에도 진짜 원인(안 집음, 위치)을 짚지 못했습니다. 실험 5의 반성문 16개 중 6개가 바로 앞 시도의 반성문을 그대로 복사한 것이었습니다.

## 6. 정리 — 언어와 행동 사이의 네 가지 어긋남

| 어긋남 | 예 | 근거 |
| --- | --- | --- |
| 1. 지시문 단어 ↔ 실행 명령어 | 태스크의 put, 게임의 move | 45% → 80% (지시문), 100% (번역) |
| 2. 같은 단어의 뜻 범위 | 일상의 move(집어서 옮기기) vs 환경의 move(놓기만) | 집지 않고 move 57회 |
| 3. 지시문의 물건 ↔ 실제 물건 | cloth를 찾으라는데 towel을 집음 | cloth 게임 |
| 4. 실패 원인 ↔ 설명하는 언어 | `Nothing happens.` 한 문장 → 원인을 지어내고 반성문을 복사 | 실험 1–8 반성문 전체 |

같은 모델, 같은 게임에서 단어 하나만 바꿔도 성공률이 45%에서 100%까지 움직였습니다. 에이전트의 성능 숫자를 볼 때는 지시문과 명령어의 언어가 맞는지부터 확인해야 한다는 것이 이번 실험의 가장 큰 교훈입니다.

## 7. 한계와 다음

- 게임 20개 · 모델 1개 · 실행 1번의 결과입니다. temperature 0이어도 게임 단위 결과가 조금씩 흔들렸습니다.
- 코드 번역 100%는 논문 수치(ReAct PaLM-540B 71%, 134게임)보다 높습니다. 앞 20게임이 쉬웠을 수도 있고, ReAct/ALFWorld 기록이 Qwen의 학습 데이터에 들어갔을 수도(데이터 오염) 있습니다. 예시를 조금 바꾸면 성능이 떨어지는 것도 이 가설과 맞지만, 아직 확인하지는 못했습니다.
- 실험 5는 태스크 지시문을 바꾼 분석용 실험이라 논문과 직접 비교할 수 없습니다.

다음에 해 볼 것들입니다.

1. 코드 번역 실험을 134게임 전체로 돌려 논문과 같은 조건에서 비교하기
2. 다른 계열 모델(Llama, Gemma)과 더 큰 모델로 돌려 오염 효과와 크기 효과를 분리하기
3. 환경이 실패 이유를 말해 주면 반성문이 원인을 짚는지 확인하기
4. 마인크래프트 에이전트(STEVE-1, Voyager)에서도 같은 네 가지 틀로 관찰하기
