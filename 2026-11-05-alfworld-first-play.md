---
layout: post
title: "ALFWorld를 직접 플레이하며 찾은 실패 다섯 가지"
date: 2026-11-05 18:00:00 +0900
description: LLM 에이전트를 돌리기 전에, 사람이 먼저 텍스트 게임을 풀어 보며 환경이 어디서 막히는지 기록했습니다.
tags: alfworld agent react reflexion
categories: notes
toc:
  sidebar: left
---

> **이 글에 대해**
> 교수님 과제로 Agentic AI의 SKILL Action 연구를 시작하면서, 첫 번째 환경인 ALFWorld를 맥북에 설치하고 게임 한 판을 직접 플레이했습니다.
> ReAct·Reflexion 같은 LLM 에이전트를 돌리기 전에, 사람이 먼저 같은 게임을 풀어 보면 에이전트가 어디서 막힐지 미리 알 수 있다고 봤습니다.
> 게임 한 판(약 35스텝)의 관찰이라 일반화하기에는 표본이 작습니다.

## 1. ALFWorld가 무엇인가

ALFWorld는 집안일 과제를 텍스트 게임으로 푸는 환경입니다([Shridhar et al., ICLR 2021](https://arxiv.org/abs/2010.03768)).
원래는 ALFRED라는 3D 가상 집에서 로봇이 카메라를 보며 수행하는 과제인데, 같은 방과 같은 과제를 텍스트로도 만들어 둔 것입니다.
둘을 묶는 건 PDDL입니다. 방의 상태("pot 1이 countertop 3 위에 있다")와 행동 규칙("cool을 하려면 냉장고 앞에 있어야 한다")을 PDDL로 적어 두면, 같은 상태를 3D 장면으로도 문장으로도 보여줄 수 있습니다.

- **관측**: 방에 무엇이 있는지, 행동 결과가 무엇인지를 문장으로 줍니다.
- **행동**: 정해진 문법의 텍스트 명령입니다. `go to`, `open`, `take ... from`, `cool ... with` 등.
- **과제 6유형**: Pick, Clean, Heat, Cool, Look(Examine in Light), Pick Two.
- **평가**: 처음 보는 방으로 구성된 134개 게임(valid unseen)에서의 성공률. 에피소드당 최대 50스텝.

ReAct와 Reflexion이 이 134개 게임으로 성공률을 보고하기 때문에, LLM 에이전트 연구에서는 사실상 표준 시험장으로 쓰입니다.

## 2. 설치

맥북(Apple Silicon)에서 GPU 없이 설치했습니다. 세 번 막혔습니다.

| #   | 증상                                                                          | 원인                                                                                                                                   | 해결                                 |
| --- | ----------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------ |
| 1   | `bad CPU type in executable: pip`                                             | 문서 권장대로 `CONDA_SUBDIR=osx-64`(인텔용) 환경을 만들었는데 Rosetta 2가 없었음                                                       | `softwareupdate --install-rosetta`   |
| 2   | `alfworld[full]` 설치 중 visdom 빌드 실패 (`No module named 'pkg_resources'`) | setuptools 80부터 `pkg_resources`가 빠졌는데 visdom이 아직 씀. `[full]`은 3D 시각 환경과 학습용 패키지 묶음이라 텍스트 실험에는 불필요 | `pip install alfworld` (텍스트 전용) |
| 3   | spacy 설치 실패 (`No matching distribution found for thinc`)                  | 최신 spacy가 요구하는 thinc가 Python 3.10 이상 전용. 문서 예시는 3.9                                                                   | Python 3.10으로 환경 재생성          |

최종 명령은 다음과 같습니다.

```bash
CONDA_SUBDIR=osx-64 conda create -n alfworld python=3.10 -y
conda activate alfworld
pip install alfworld
alfworld-download
alfworld-play-tw
```

## 3. 게임 한 판

`alfworld-play-tw`가 고른 게임은 학습용(train) 분할의 Cool & Place 과제였습니다.

```
Your task is to: cool some pot and put it in countertop.
```

방에는 cabinet 9개, drawer 9개, countertop 3개, stoveburner 4개, fridge, microwave 등이 있었습니다.
pot은 countertop 3에 있었고, 거기까지 가는 데 약 25스텝을 썼습니다. 전체로는 약 35스텝 만에 성공했습니다.

{% include figure.liquid loading="eager" path="assets/img/alfworld_first_win.png" class="img-fluid rounded z-depth-1" %}
<div class="caption">
게임 후반부. 냉장고 앞에 가서야 cool이 성공하고, put이 계속 실패하다가 help로 확인한 move 문법으로 끝납니다.
</div>

## 4. 실패 다섯 가지

### 4-1. 오타, 불가능, 없는 대상이 모두 같은 응답

`go to stovenurner 2`(오타), `go to stoveburner 2`(존재하지만 이동 불가), `go to stoveburner 5`(존재하지 않음)가 전부 `Nothing happens.`였습니다.
환경 내부는 이유를 알지만 문장으로 구분해 주지 않습니다. 저도 오타를 고친 뒤 같은 명령을 두 번 더 쳤습니다.
ALFWorld 논문에도 실패하거나 변화가 없는 행동은 모두 이 문장으로 처리한다고 적혀 있어, 설계상의 단순화입니다.

### 4-2. 같은 이름, 다른 조작

cabinet 1~3, 5, 6은 `open` 해야 안이 보였는데 cabinet 4는 가자마자 "On the cabinet 4"였습니다.
이름만으로는 열어야 하는 물건인지 알 수 없습니다. "cabinet은 열어야 한다"고 일반화하면 cabinet 4에서는 틀립니다.

### 4-3. 말해 주지 않는 전제조건

pot을 든 채 카운터 앞에서 `cool pot 1 with fridge 1`을 치면 `Nothing happens.`, 냉장고 앞으로 가서 치면 성공했습니다.
PDDL의 cool 행동에 "냉장고 앞에 있어야 한다"는 전제조건이 있기 때문입니다. 환경은 이 조건을 알고 있지만 알려주지 않습니다.

### 4-4. 탐색에 쓴 스텝

pot 위치를 몰라서 stoveburner, cabinet, shelf, countertop을 차례로 돌았습니다. 35스텝 중 약 25스텝이 탐색입니다.
"부엌에서 pot은 보통 어디 있다"는 상식이나, "지난번엔 countertop 3에 있었다"는 기억이 있었다면 줄었을 스텝입니다.
앞의 것이 ReAct의 생각(Thought)이 하는 일이고, 뒤의 것이 Reflexion의 반성 메모가 하는 일입니다.

### 4-5. 문법이 바뀌어 있었다

마지막에 `put pot 1 in/on countertop 1`이 계속 `Nothing happens.`였습니다. `help`를 쳐 보니 이 버전(alfworld 0.4.2)은 `move (object) to (receptacle)`을 받습니다.

```
> move pot 1 to countertop 2
You won!
```

ALFWorld 논문의 행동 목록과 ReAct 프롬프트 예시는 `put ... in/on` 문법입니다. 원본 프롬프트를 최신 패키지에 그대로 쓰면, 에이전트가 과제를 다 풀고 마지막 단계에서 실패할 수 있습니다.
재현 수치가 논문보다 낮게 나오면 이것부터 확인해야 합니다. 아직 실제 에이전트로 확인하지는 않았습니다.

## 5. 정리

| #   | 관찰                           | 유형              | 관련 방법                           |
| --- | ------------------------------ | ----------------- | ----------------------------------- |
| 1   | 모든 실패가 `Nothing happens.` | 피드백 부재       | 실패 원인을 언어로 추론해야 함      |
| 2   | 같은 이름, 다른 조작           | affordance 불투명 | 과도한 일반화 위험                  |
| 3   | 알려주지 않는 전제조건         | 숨은 규칙         | ReAct few-shot 예시                 |
| 4   | 탐색에 25스텝                  | 탐색 비효율       | ReAct의 상식 추론, Reflexion의 기억 |
| 5   | `put` → `move`                 | 버전 불일치       | 재현 전 확인 필요                   |

다음에는 ReAct를 같은 환경에서 돌려 보고, 위 다섯 가지 중 무엇을 LLM이 실제로 겪는지 확인할 계획입니다.
