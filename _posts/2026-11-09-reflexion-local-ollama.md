---
layout: post
title: "Reflexion을 맥북 로컬 LLM으로 돌려 보기 — 15판 0승, 그런데 마지막 한 줄만 틀렸다"
date: 2026-11-09 18:00:00 +0900
description: 공식 Reflexion 코드의 LLM 호출부만 Ollama + Qwen2.5-7B로 바꿔 ALFWorld를 돌리고, 실패 원인을 통제 실험 네 개로 좁혔습니다.
tags: alfworld agent reflexion ollama qwen llm
categories: notes
toc:
  sidebar: left
---

> **이 글에 대해**
> [앞 글]({% post_url 2026-11-08-react-reflexion-papers %})에서 정리한 Reflexion을 직접 돌려 본 기록입니다.
> API 없이 맥북(M4 Pro)에서 오픈소스 LLM(Qwen2.5-7B)으로 실행했습니다.
> 게임 5개 · 모델 1개로 돌린 결과라 숫자보다는 실패의 모양에 초점을 둡니다.

## 1. 왜 로컬 LLM인가

공식 Reflexion 코드는 OpenAI의 `text-davinci-003`을 부릅니다. 이 모델은 서비스가 끝났고, 다른 API 모델은 유료입니다.
그래서 LLM 호출부만 [Ollama](https://ollama.com)로 바꿨습니다. Ollama는 오픈소스 LLM을 내 컴퓨터에서 서버(`localhost:11434`)로 띄워 주는 프로그램입니다.

| 항목 | 설정 |
| --- | --- |
| 모델 | `qwen2.5:7b` (4.7GB, Apple M4 Pro GPU로 실행) |
| 호출 방식 | `/api/generate`, `raw: true` (채팅 템플릿 없이 글 이어 쓰기 — `text-davinci-003`과 같은 방식) |
| 문맥 길이 | `num_ctx 8192` (프롬프트가 3,600토큰 이상이라 Ollama 기본값 4,096으로는 반성문이 붙으면 잘림) |
| 평가 | ALFWorld valid unseen 앞 5게임 × 최대 3회 시도 |

## 2. 바꾼 코드

나머지 Reflexion 로직(ReAct 루프, 실패 판정, 반성문 생성, 기록 저장)은 원본 그대로 두었습니다.

| 파일 | 바꾼 것 | 이유 |
| --- | --- | --- |
| `utils.py` | OpenAI 호출 → Ollama 호출 | `text-davinci-003` 서비스 종료 |
| `alfworld_trial.py` | `getattr(..., 'AlfredTWEnv')` → `get_environment('AlfredTWEnv')` | alfworld 0.4.x에서 환경을 불러오는 방식이 바뀜 |
| `alfworld_trial.py` | 프롬프트 파일을 환경변수로 선택 | 통제 실험용 |
| `generate_reflections.py` | 반성문을 터미널에 출력 | 실험 중 실시간 확인 |

```python
def _ollama(prompt, model, temperature, max_tokens, stop):
    body = {
        "model": model, "prompt": prompt,
        "raw": True,              # 채팅 템플릿 없이 글 이어 쓰기
        "stream": False,
        "options": {"temperature": temperature, "num_predict": max_tokens,
                    "num_ctx": 8192, "stop": stop or []},
    }
    req = urllib.request.Request("http://localhost:11434/api/generate",
                                 data=json.dumps(body).encode(), headers={"Content-Type": "application/json"})
    with urllib.request.urlopen(req, timeout=600) as resp:
        return json.loads(resp.read())["response"]
```

터미널 두 개로 실행합니다. 하나에서는 `ollama serve`로 LLM 서버를 띄우고, 다른 하나에서 Reflexion 코드를 돌립니다.

{% include figure.liquid loading="eager" path="assets/img/reflexion_qwen_run.png" class="img-fluid rounded z-depth-1" zoomable=true %}
<div class="caption">
실행 화면. 위는 Ollama 서버 로그, 아래는 Qwen이 실패 후 쓴 반성문입니다.
</div>

## 3. 실험 1 — 원본 프롬프트: 15판 0승

| 게임 | 태스크 | Qwen이 한 것 | 막힌 곳 |
| --- | --- | --- | --- |
| 0 | cool tomato → microwave | 찾기·집기·식히기·전자레인지 열기 | `put tomato 1 in microwave 1` |
| 1 | clean spatula → drawer | 찾기·집기·씻기·서랍으로 이동 | `put spatula 1 in/on drawer 1` |
| 2 | clean plate → countertop | 찾기·집기·씻기 | `put plate 2 in/on countertop 1` |
| 3 | watch → safe | 서랍에서 시계 찾아 집기 | `put watch 1 in/on safe 1` |
| 4 | clean cloth → cabinet | towel을 보고 그쪽으로 | cloth를 towel로 착각 |

5게임 × 3회 = 15판에서 한 번도 이기지 못했습니다. 하지만 4게임은 중간 단계(찾기, 집기, 씻기·식히기)를 다 맞게 하고 마지막 놓기 한 줄에서만 막혔습니다.
[첫 글]({% post_url 2026-11-05-alfworld-first-play %})에서 예상한 대로입니다. 프롬프트 예시는 `put X in/on Y`인데 최신 alfworld는 `move X to Y`만 받습니다.

한 번 막히면 실패가 연쇄로 이어졌습니다. 물건을 놓지 못하니 손에 든 채로 다니고, 한 번에 하나만 들 수 있어서 이후 `take`도 전부 실패합니다. Qwen은 이것을 "거기에 물건이 없다"로 해석했습니다.

## 4. 반성문은 원인을 지어냈다

환경은 실패 이유를 말해 주지 않습니다. 문법이 틀려도, 위치가 틀려도 `Nothing happens.` 한 문장뿐입니다. 그래서 반성문에는 그럴듯한 이야기가 적혔습니다.

| 게임 | 반성문 (발췌) | 실제 원인 |
| --- | --- | --- |
| 0 토마토 | "I encountered issues with the microwave not functioning properly…" | 전자레인지 고장을 가정. 실제로는 문법 |
| 1 뒤집개 | "the spatula could not be placed in any surface or container…" | 물건 속성 탓. 실제로는 문법 |
| 2 접시 | "I was unsuccessful in finding a plate." | 사실과 다름. 실제로 찾아서 씻었음 |
| 3 시계 | "I will also check the safe before putting the watch on it." (수십 회 반복) | 같은 문장 반복으로 퇴화 |
| 4 천 | "…cleaning a cloth without water is not possible." | 게임 중 지어낸 생각("싱크대에 물이 없나 보다")이 반성문으로 옮겨감 |

진짜 원인(문법, 위치)을 짚은 반성문은 0개였습니다. 잘못된 반성문은 다음 시도를 다시 잘못된 방향으로 이끌었습니다.

## 5. 통제 실험 1–4 — 한 번에 한 곳씩 put을 지우기

원인이 정말 문법인지 확인하려고, 프롬프트 예시에서 `put`을 한 곳씩 `move`로 바꿔 가며 돌렸습니다.

| 실험 | 예시에서 바꾼 줄 | 남은 put | 첫 시도 | 3회 후 |
| --- | --- | --- | --- | --- |
| 1 | 없음 (원본) | 전부 | 0/5 | 0/5 |
| 2 | 행동 36줄 → `move X to Y` | 생각·관측·태스크 지시문 | 2/5 | 2/5 |
| 3 | + 생각 36줄 → `move it to` | 관측·태스크 지시문 | 2/5 | 2/5 |
| 4 | + 관측 36줄 → `You move the X to the Y.` | 태스크 지시문만 | 1/5 | 2/5 |

- 실험 2: 행동 줄만 바꿔도 바로 풀리는 게임이 생겼습니다(게임 1은 14스텝, 게임 2는 9스텝).
- 실험 3: 생각 줄은 원인이 아니었습니다.
- 실험 4: 이상한 점이 있었습니다. 예시에서 `in/on`이라는 글자를 모두 지웠는데도 Qwen이 `put plate 2 in/on countertop 1`을 그대로 썼습니다.

`in/on`은 옛 ALFWorld 프롬프트 고유의 표기입니다. 그렇다면 이 표기는 어디서 왔을까요? 후보는 두 가지입니다.

1. 아직 남아 있는 **태스크 지시문**("put a clean plate in countertop")
2. 공개된 ReAct/ALFWorld 기록을 사전학습에서 본 **기억**

## 6. 정리와 다음 글

- 원본 프롬프트의 실패는 거의 전부 마지막 놓기 문법 하나 때문이었습니다.
- 반성문은 원인을 맞히지 못했습니다. 실패 피드백이 한 문장뿐이면 반성은 추측이 됩니다.
- 5게임은 작습니다. 게임 하나 차이(20% vs 40%)는 우연일 수 있습니다.

[다음 글]({% post_url 2026-11-10-one-word-success-rate %})에서는 게임을 20개로 늘리고, 태스크 지시문까지 바꾸고, 환경 쪽에서 문법을 번역해 보면서 put이 어디서 오는지 끝까지 따라갑니다.
