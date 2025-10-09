



---
layout: post
title: "CodeAgents + Structure: A Better Way to Execute Actions"
author: jimin
categories: [Agent]
image: assets/images/blog/posts/2025-10-13-structured-codeagent/thumbnail.png
---

* TOC
{:toc}
<!--toc-->

_이 글은 Hugging Face 블로그의 [CodeAgents + Structure: A Better Way to Execute Actions](https://huggingface.co/blog/structured-codeagent)를 한국어로 번역한 글입니다._

---
# CodeAgents + Structure: 액션 실행을 위한 더 나은 방법

오늘 우리는 AI 에이전트 설계에서 두 가지 강력한 패러다임을 연결하는 연구를 공유합니다. 바로 코드 기반 액션의 표현력과 구조화된 생성의 신뢰성입니다. 우리의 연구 결과, **CodeAgents**에게 생각(thought)과 코드를 모두 구조화된 JSON 형식으로 생성하도록 강제하면, 여러 벤치마크에서 기존 접근 방식보다 성능이 크게 향상됨을 보여줍니다.

![](https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/blog/structured-codeagent/accuracy.png)
그림 1: 세 가지 접근 방식의 정확도 비교 — Structured CodeAgent(파란색), CodeAgent(주황색), ToolCallingAgent(회색) — <ins>[**SmolBench (GAIA, MATH, SimpleQA, and Frames**](https://huggingface.co/datasets/akseljoonas/smolbench)</ins>)에서. 오류 막대는 95% 신뢰 구간을 나타냅니다.

## 🤔 에이전트 액션의 진화

AI 에이전트는 API 호출, 데이터 처리, 복잡한 문제 해결 등 세상에서 실제로 액션을 취할 수 있어야 합니다. 에이전트가 이러한 액션을 표현하는 방식은 여러 패러다임을 거쳐 발전해왔습니다.

**전통적인 JSON Agent**: 에이전트는 구조화된 JSON을 생성해 도구를 호출합니다.

```json
{"tool": "get_weather", "arguments": {"city": "Paris"}}
```

이러한 에이전트는 미리 정의된 도구 목록에서 선택하고 JSON 형식의 호출을 생성하여 동작합니다. 이 방식은 OpenAI의 <ins>[**function calling API**](https://openai.com/index/function-calling-and-other-api-updates/)</ins>로 널리 알려졌으며, 그 이후 가장 일반적인 도구 호출 방식이 되었습니다.

이 방식은 신뢰할 수 있지만 다음과 같은 한계가 있습니다:

* **제한된 액션 범위**: 에이전트가 취할 수 있는 액션은 미리 정의된 도구에만 의존하므로 기능이 제한됩니다.
* **합성력 부족**: 여러 출처에서 정보를 조합해야 하는 작업에서 JSON 에이전트는 중간 상태를 유지할 수 없어 어려움을 겪습니다. 일부 모델은 병렬 도구 호출을 지원하지만, 한 도구의 출력이 다음 액션을 결정하거나 결과를 함께 비교하고 처리해야 하는 시나리오는 다루기 어렵습니다.
* **경직된 구조**: 도구가 필요한 작업에 정확히 대응하지 못하는 경우 유연하게 대처하기 어렵습니다.

**Code Agents**: 에이전트는 고유한 코딩 능력을 활용해 직접 실행 가능한 Python 코드를 작성합니다.

```python
# 한 번의 모델 호출로 3개 도시의 평균 기온을 구할 수 있습니다.
temperature_sum = 0
for city in ["Paris", "Tokyo", "New York"]:
    temp = get_weather(city)
    temperature_sum += temp
    
print(f"Average temperature: {temperature_sum / 3:.1f}°C")
```

이러한 전환은 <ins>[“**Executable Code Actions Elicit Better LLM Agents**”](https://arxiv.org/abs/2402.01030)</ins> 논문에서 CodeAct라는 이름으로 처음 제시되었으며, 에이전트가 도구 호출뿐 아니라 임의의 실행 가능한 Python 코드를 작성할 수 있는 유연성을 제공했습니다.

핵심 통찰은 **도구가 코드 내부에서 직접 호출된다는 점**입니다. 이를 통해 변수 및 상태 관리가 훨씬 안정적이 됩니다. 에이전트는 루프, 함수, 조건문 안에서 도구를 호출할 수 있으며, 이는 각 액션에서 동적으로 실행 그래프를 생성하는 것과 같습니다.

<ins>[**CodeAgent**](https://github.com/huggingface/smolagents/blob/6a12ebdf210207eec22d5940157f522463fc1c59/src/smolagents/agents.py#L1344)</ins> 사용의 장점:

* **스마트한 도구 사용**: 상황에 따라 어떤 도구를 사용할지 스스로 결정
* **무제한 유연성**: 목표 달성을 위해 Python 기능을 자유롭게 활용 가능
* **사고 테스트 가능**: 가설을 세우고 테스트할 수 있어 액션에 더 큰 유연성 확보

하지만 마크다운에서 코드를 파싱하는 과정은 오류가 발생하기 쉬우며, 이를 해결하기 위한 제안이 있습니다: 구조화된 생성을 사용해 코드 액션을 생성하는 것입니다.

## ➡️ Code Agent에 구조화된 출력 추가하기

구조화된 출력을 사용하면 LLM이 명시적으로 생각과 코드를 JSON blob 형태로 생성하도록 강제할 수 있습니다:

```json
// "code" 블록은 실행 가능한 Python으로 파싱됩니다.
{
  "thoughts": "I want to find the average temperature across 3 cities.",
  "code": "temperature_sum = 0\nfor city in [\"Paris\", \"Tokyo\", \"New York\"]:\n    temp = get_weather(city)\n    temperature_sum += temp\n\nprint(f\"Average temperature: {temperature_sum / 3:.1f}°C\")"
}
```

여기서 핵심 차이는 생성이 강제된다는 점입니다. 단순히 생각과 코드를 프롬프트로 유도하는 것이 아니라, <ins>[**structured outputs**](https://huggingface.co/docs/text-generation-inference/en/conceptual/guidance)</ins>를 사용해 구조를 반드시 따르도록 강제합니다.

이 접근 방식은 코드 실행의 유연성에 구조화된 생성의 신뢰성을 더해 두 가지 장점을 모두 제공합니다.

* **명시적 추론**: thoughts 필드가 에이전트가 액션을 취하기 전 반드시 사고하도록 강제
* **신뢰할 수 있는 파싱**: JSON 구조로 마크다운 파싱 오류 제거
* **전체 코드 표현력 유지**: code 필드로 CodeAgent의 유연성 유지
* **명확한 분리**: 계획과 실행을 명확히 분리

## 🧪 벤치마크 결과

우리는 GAIA, MATH, SimpleQA, Frames를 포함한 여러 벤치마크에서 이 세 가지 패러다임을 비교했습니다. 결과는 명확합니다. **코드 액션 + 구조화된 생성이 강력한 모델에서 성능을 지속적으로 개선합니다.**

대부분의 강력한 모델에서 구조화된 접근은 기존 CodeAgent보다 평균 2~7%포인트 높은 성능을 보였습니다.

* **OpenAI 모델**: 구조화된 방식에서 특히 추론이 중요한 작업에서 큰 향상
* **Claude 모델**: 구조화로 성능 향상, 특히 Claude 3.7 Sonnet에서 강한 결과
* **Qwen 모델**: 구조화로 전반적 향상, 하지만 작은 모델에서는 “구조화 세금(structure tax)” 발생

## 💡 구조화가 (일반적으로) 도움이 되는 이유

### 파싱 문제는 실제로 존재한다

<ins>[**smolagents의 CodeAgent 구현**](https://github.com/huggingface/smolagents/blob/6a12ebdf210207eec22d5940157f522463fc1c59/src/smolagents/agents.py#L1344)</ins>은 LLM 출력에서 Python 코드를 추출하는데, 다음과 같은 상황에서 실패할 수 있습니다:

* 마크다운의 코드 블록이 불완전하거나 잘못 포맷된 경우
* 하나의 응답에 여러 코드 블록이 포함된 경우

구조화된 생성은 안정적인 JSON 파싱으로 이러한 문제를 제거합니다.

우리는 벤치마크 전반에서 15,724개의 에이전트 트레이스를 분석했습니다. 결과는 다음과 같습니다:

* **2.4%**의 트레이스에서 첫 호출 시 파싱 오류 발생
* 첫 호출에 **파싱 오류가 있는** 트레이스 성공률: **42.3%**
* 첫 호출에 **파싱 오류가 없는** 트레이스 성공률: **51.3%**

**파싱 오류가 없는 에이전트 트레이스는 있는 경우보다 21.3% 더 자주 성공했습니다.**

이는 단순한 편의성의 문제가 아닙니다. 파싱 오류는 실패의 연쇄를 일으켜 전체 성능에 큰 영향을 줍니다. 첫 액션 실행에 실패하면 이후 문제 해결 과정이 무너질 가능성이 커집니다.

![](https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/blog/structured-codeagent/parsing_error.png)
그림 2: 첫 단계에서 파싱 오류가 발생하면 성공률이 21.3% 감소하고, 평균 스텝 수가 3.18에서 4.63으로 증가합니다.

#### 추가적으로: 강제된 추론 과정

구조화된 생성과 명시적 `thoughts` 사용은 단순한 프롬프트가 아니라, 에이전트가 액션 전 반드시 추론을 서술하도록 강제합니다. 이는 다음과 같은 효과를 가져옵니다:

* **더 나은 계획**: 문제 해결을 더 체계적으로 생각
* **향상된 신뢰성**: 명시적 추론으로 논리적 오류를 조기에 포착

### 구조화 세금 (Structure Tax)

결과는 또한 명확한 능력 임계값을 보여줍니다. 모델은 구조화된 생성의 이점을 얻기 위해 충분한 지시 따르기 능력과 JSON 학습 경험이 필요합니다. 즉, 구조화된 접근은 다음과 같은 경우에 가장 효과적입니다:

* 대형, 고성능 모델
* 지시 따르기 능력이 뛰어난 모델
* 구조화된 생성에 파인튜닝된 모델

#### 구조화가 실패하는 경우: 실제 예시

작은 모델(`mistralai/Mistral-7B-Instruct-v0.3`)이 구조화된 코드를 생성하려 할 때, 인지 부하가 과도해집니다:

```json
{
  "thought": "I need to find the height...",
  "code": "web_search(query=\"Eiffel Tower height\")\", "
}
```

모델이 생성한 Python 코드는 `web_search(query="Eiffel Tower height")",`와 같이 문법적으로 잘못되어 즉시 SyntaxError와 실행 실패로 이어집니다.

이것이 “구조화 세금”의 예입니다. 작은 모델은 JSON 형식, Python 문법, 실제 문제 해결 로직을 동시에 처리하지 못해 단순한 마크다운 기반 코드 생성보다 성능이 떨어질 수 있습니다.

## 🚀 Structured CodeAgents 사용 시기

#### ✅ Structured CodeAgents를 사용해야 하는 경우:

* 강력한 모델(32B+ 파라미터 또는 프런티어 모델)을 사용할 때
* 복잡한 추론 및 코드 실행이 필요한 작업일 때
* 에이전트 출력 파싱의 신뢰성이 필요할 때

#### ⚠️ 다음과 같은 경우에는 대안을 고려:

* 구조화된 생성에 어려움을 겪는 작은 모델을 사용할 때
* 단순하고 사전 정의된 워크플로우로 충분할 때

### smolagents에서 사용 방법:

매우 간단합니다! `use_structured_outputs_internally:` 옵션만 활성화하면 됩니다.

```python
from smolagents import CodeAgent, InferenceClientModel, GoogleSearchTool

# 구조화된 생성 활성화
agent = CodeAgent(
    tools=[GoogleSearchTool(provider="serper")],
    model=InferenceClientModel("Qwen/Qwen3-235B-A22B", provider='nebius'),
    use_structured_outputs_internally=True # 구조화된 출력 활성화
)

result = agent.run("Calculate the time for a cheetah to run across the Golden Gate Bridge")
```

LLM은 다음과 같이 생성합니다:

```json
{
  "thoughts": "I need to find the length of the Golden Gate Bridge and the top speed of a cheetah, then calculate the time.",
  "code": "bridge_info = web_search('Golden Gate Bridge length meters')\ncheetah_speed = web_search('Cheetah top speed') ..."
}
```

그 후 "code" 부분은 기존 CodeAgent와 동일하게 실행됩니다. 단, 이제 파싱 신뢰도는 100%입니다!

### 구현 팁

1. **명확한 프롬프트**: 예상되는 JSON 구조를 프롬프트에 명확히 명시하세요.
2. **모델 선택**: 구조화된 생성 능력이 강한 모델을 선택하세요.
3. **적절한 제공자 선택**: OpenAI나 Anthropic 같은 일부 API 제공자는 구조화된 생성을 기본적으로 지원합니다. Hugging Face Inference 제공자를 사용하는 경우, 구조화된 생성 지원 여부는 제공자마다 다릅니다. 구조화된 생성을 지원하는 제공자 목록은 다음을 참고하세요: <ins>[**Structured generation support for Models in smolagents‣**](https://www.notion.so/huggingface2/Structured-generation-support-for-Models-in-smolagents-1f51384ebcac8074a051e6dd03d1fe1d)</ins>

### 더 큰 그림 - 다음 단계는?

이 연구는 우리가 에이전트 아키텍처를 더 정교하게 이해하는 방향으로 나아가고 있음을 보여줍니다. 단순히 “에이전트가 무엇을 할 수 있는가?”가 아니라 “에이전트가 그것을 어떻게 생각하고 수행해야 하는가?”에 대한 문제입니다.

추론 과정을 더 명시적으로 만드는 것이 모델이 경로를 유지하는 데 도움이 될 수도 있고, 단순히 파싱이 쉬워서일 수도 있습니다. 어느 쪽이든 이점입니다.

하지만 이것은 시작에 불과합니다. 앞으로 탐구할 질문은 많습니다:

* 어떤 다른 구조적 개선이 도움이 될까?
* 특히 작은 모델에서 이를 더 잘 작동하게 만들 방법은 무엇일까?
* 이것이 AI 추론의 본질에 대해 무엇을 시사할까?

지금 smolagents를 사용 중이거나 자체 CodeAgent 시스템을 만들고 있다면, 구조화된 출력을 시도해보는 것을 추천합니다. 파싱 오류가 사라지고, 성능이 눈에 띄게 개선될 수도 있습니다!
