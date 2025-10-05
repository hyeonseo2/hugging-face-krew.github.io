---
layout: post
title: "Hugging Face MCP 서버 구축하기"
author: hyeonseo
categories: [MCP]
image: assets/images/blog/posts/2025-09-29-building-hf-mcp/thumbnail.png
---
* TOC
{:toc}
<!--toc-->
_이 글은 Hugging Face 블로그의 [Building the Hugging Face MCP Server](https://huggingface.co/blog/building-hf-mcp)를 한국어로 번역한 글입니다._

---

# Hugging Face MCP 서버 구축하기

> [!TIP]
> **요약:** Hugging Face 공식 MCP 서버는 Hub에 접근하는 AI 어시스턴트를 위한 독특한 커스터마이즈 옵션을 제공하며, 하나의 간단한 URL로 수천 개의 AI 애플리케이션을 이용할 수 있습니다. 배포를 위해 MCP의 "Streamable HTTP" 전송 방식을 사용했으며, 서버 개발자가 직면하는 여러 고려사항(trade-offs)을 자세히 살펴봅니다. 
>
> 지난 한 달 동안 유용한 MCP 서버 구축에 관한 많은 것을 배웠습니다. 여기에서 그 과정을 설명하겠습니다.

## 소개

모델 컨텍스트 프로토콜(MCP)은 AI 어시스턴트를 외부 세계와 연결하는 표준으로 자리잡아가고 있습니다.

Hugging Face에서는 MCP를 통해 Hub에 접근할 수 있도록 하는 것이 당연했고, 그 [`hf.co/mcp`](https://hf.co/mcp) MCP 서버 개발 경험을 이 글을 통해 공유합니다.

## 설계 선택

커뮤니티는 연구, 개발, 콘텐츠 제작 등을 위해 Hub를 활용합니다. 우리는 사람들이 자신의 필요에 맞게 서버를 커스터마이즈하고, Space에서 제공되는 수천 개의 AI 애플리케이션에 쉽게 접근할 수 있게 했습니다. 이는 사용자의 도구를 상황에 맞게 조정하여 MCP 서버를 동적으로 만드는 것을 의미합니다.

<figure class="image text-center">
  <img src="https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/hf-mcp-remote/hf-mcp-settings.png" alt="The Hugging Face MCP Settings Page">
  <figcaption><a href="https://huggingface.co/settings/mcp">Hugging Face MCP 설정 페이지</a> 사용자가 도구를 구성할 수 있는 곳입니다.</figcaption>
</figure>

복잡한 다운로드나 설정을 피함으로써 접근을 간편하게 만들고자 했기에, 간단한 URL을 통해 원격으로 접근할 수 있도록 하는 것이 필수적이었습니다.

## 원격 서버

원격 MCP 서버를 구축할 때 가장 먼저 결정해야 할 사항은 클라이언트가 서버에 접속하는 방법입니다. MCP는 다양한 전송 옵션을 제공하며, 각각 장단점이 있습니다. **요약:** 오픈소스 코드는 모든 방식을 지원하지만, 실제 운영 환경에서는 가장 최신 방식을 선택했습니다. 이 섹션에서는 각 옵션을 상세히 살펴봅니다. 

2024년 11월 출시 이후, MCP는 9개월 동안 3차례의 프로토콜 개정을 거치며 급속한 발전을 이루었습니다. 이 과정에서 SSE 전송 방식이 Streamable HTTP로 교체되었으며, 인증 방식도 새롭게 도입되고 개편되었습니다.

이러한 급격한 변화로 인해 클라이언트 애플리케이션별로 MCP 기능과 개정판 지원이 달라지고, 이는 설계 선택에 추가적인 도전 과제가 되었습니다.

다음은 모델 컨텍스트 프로토콜(MCP) 및 관련 SDK에서 제공하는 전송 옵션에 대한 간략한 요약입니다: 

| 전송 방식         | 설명                                                                                                                                        |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `STDIO`           | 일반적으로 MCP 서버가 클라이언트와 동일한 컴퓨터에서 실행될 때 사용됩니다. 필요한 경우 파일과 같은 로컬 리소스에 접근할 수 있습니다. |
| `HTTP with SSE`   | HTTP를 통한 원격 연결에 사용됩니다. MCP의 2025-03-26 버전에서 사용 중단되었으나 여전히 사용 중입니다.                                     |
| `Streamable HTTP` | 기존 SSE 버전보다 배포 옵션이 더 많은 유연한 원격 HTTP 전송 방식입니다.                                |

`STDIO`와 `HTTP with SSE`는 기본적으로 완전한 양방향 통신을 지원합니다. 즉, 클라이언트와 서버는 연결을 유지한 상태로 언제든지 서로에게 메시지를 전송할 수 있습니다. 

> [!TIP]
> SSE는 "서버 전송 이벤트(Server Sent Events)"를 의미합니다. 이는 HTTP 서버가 열린 연결을 유지하고 요청에 대한 응답으로 이벤트를 전송하는 방식입니다.

#### Streamable HTTP 이해하기

MCP 서버 개발자는 Streamable HTTP 전송을 설정할 때 많은 선택지에 직면하게 됩니다.

선택 가능한 주요 통신 방식은 세 가지입니다:

- **직접 응답(Direct Response)** - 단순한 요청/응답 방식(표준 REST API와 유사). 단순 검색처럼 직관적이고 상태를 유지하지 않는(Stateless) 작업에 적합합니다.
- **요청 범위 스트림(Request Scoped Streams)** - 특정 요청에 연결된 임시 SSE 스트림. 비디오 생성처럼 도구 호출에 시간이 오래 걸릴 때 [진행 상황 업데이트](https://modelcontextprotocol.io/specification/2025-06-18/basic/utilities/progress)를 전송하는 데 유용합니다. 또한 서버가 사용자에게 [추가 정보 요청(Elicitation)](https://modelcontextprotocol.io/specification/2025-06-18/client/elicitation)을 하거나 샘플링 요청(Sampling)을 수행해야 할 때도 활용됩니다.
- **서버 푸시 스트림(Server Push Streams)** - 서버에서 시작하는 메시지를 지원하는 장기(Long-lived) SSE 연결. 이를 통해 자원, 도구, 프롬프트 목록 변경 알림이나 수시 샘플링 및 추가 정보 요청을 처리할 수 있습니다. 이러한 연결은 재연결 시 [연결 유지(keep-alive)](https://modelcontextprotocol.io/specification/2025-06-18/basic/utilities/ping) 관리와 재개(resumption) 메커니즘 같은 추가적인 관리가 필요합니다. 

> [!TIP]
> 공식 SDK에서 요청 범위 스트림을 사용할 때는 `RequestHandlerExtra` 매개변수(TypeScript)에 제공된 `sendNotification()` 및 `sendRequest()` 함수를 사용하거나 `related_request_id`(Python)를 설정하여 메시지를 올바른 스트림으로 전송해야 합니다.

추가적으로 고려해야 할 요소는 MCP 서버 자체가 각 연결에 대한 상태를 유지해야 하는지 여부입니다. 이는 클라이언트가 초기화(Initialize) 요청을 보낼 때 서버가 결정합니다:

| | 무상태(Stateless) | 상태 유지(Stateful) |
|---|-----------|----------|
| **세션 ID(Session IDs)** | 필요 없음 | 서버가 `mcp-session-id`로 응답 |
| **의미(What it means)** | 각 요청은 독립적으로 처리됨 | 서버가 클라이언트 컨텍스트를 유지 |
| **확장성(Scaling)** | 단순 수평 확장: 어떤 인스턴스도 모든 요청을 처리 가능 | 세션 어피니티(session affinity) 또는 공유 상태 메커니즘 필요 |
| **재개(Resumption)** | 필요 없음 | 끊어진 연결에 대해 메시지 재전송 가능 |

아래 표는 MCP 기능과 지원되는 통신 방식을 요약한 것입니다:

| MCP 기능(MCP Feature)             | 서버 푸시(Server Push)                  | 요청 범위(Request Scoped)                         | 직접 응답(Direct Response) |
| -------------------------- | ---------------------------- | -------------------------------------- | -------- |
| 도구, 프롬프트, 자원(Tools, Prompts, Resources) | Y                            | Y                                      | Y |
| 샘플링/추가 정보 요청(Sampling/Elicitation)      | 서버에서 언제든 시작 가능 | 클라이언트가 시작한 요청과 연관 | N        |
| 자원 구독(Resource Subscriptions)     | Y                            | N                                      | N        |
| 도구/프롬프트 목록 변경(Tool/Prompt List Changes)   | Y                            | N                                      | N        |
| 도구 진행 상황 알림(Tool Progress Notification) | -                            | Y                                      | N        |

요청 범위 스트림의 경우, 샘플링 및 추가 정보 요청이 응답을 올바르게 연결시키기 위해 `mcp-session-id`를 사용할 수 있도록 상태 유지(Stateful) 연결이 필요합니다.

> [!TIP]
> [Hugging Face MCP 서버](https://github.com/evalstate/hf-mcp-server)는 오픈 소스이며, STDIO, SSE 및 Streamable HTTP 배포를 직접 응답(Direct Response) 및 서버 푸시(Server Push) 모드에서 모두 지원합니다. 서버 푸시 스트림 사용 시 연결 유지(keep-alive) 및 마지막 활동 타임아웃을 설정할 수 있습니다. 또한 내장된 관측(observability) 대시보드를 통해 서로 다른 클라이언트가 연결을 관리하는 방식을 파악하고 도구 목록 변경 알림을 처리할 수 있습니다.

다음 그림은 "서버 푸시" Streamable HTTP 모드에서 실행 중인 MCP 서버 연결 대시보드입니다:

<figure class="image text-center">
  <img src="https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/hf-mcp-remote/hf-mcp-connections.png" alt="The Hugging Face MCP Server Connection Dashboard">
  <figcaption>The Hugging Face MCP 서버 연결 대시보드.</figcaption>
</figure>


### 프로덕션 배포(Production Deployment)

프로덕션 환경에서는 다음과 같은 이유로 무상태(Stateless), 직접 응답(Direct Response) 구성을 갖춘 Streamable HTTP 방식으로 MCP 서버를 실행하기로 결정했습니다:

**무상태(Stateless)** 익명 사용자의 경우, Hub 사용을 위한 기본 도구 세트와 이미지 생성기를 제공합니다. 인증된 사용자의 경우, 상태(state)는 [선택한 도구](https://huggingface.co/settings/mcp)와 사용자가 고른 Gradio 애플리케이션으로 구성됩니다. 또한 각 계정의 ZeroGPU 할당량이 정확히 적용되도록 보장합니다. 이는 요청 시 조회되는 `HF_TOKEN`이나 OAuth 자격 증명을 통해 관리됩니다. 기존 도구들 중에는 요청 간 상태를 유지해야 할 필요가 없습니다.

> [!TIP]
> MCP 서버 URL에 `?login`을 추가하여 OAuth 로그인을 사용할 수 있습니다. 예시: `https://huggingface.co/mcp?login`. `claude.ai` 원격 통합이 최신 OAuth 사양을 지원하게 되면 이를 기본값으로 설정할 수 있습니다.

**직접 응답(Direct Response)**  가장 낮은 배포 자원 오버헤드를 제공하며, 현재 사용 중인 도구들은 실행 중에 샘플링(Sampling)이나 추가 정보 요청(Elicitation)를 필요로 하지 않습니다.

**향후 지원(Future Support)** 출시 당시 많은 MCP 클라이언트에서 "HTTP with SSE" 전송 방식이 여전히 원격 기본값으로 설정되어 있었습니다. 그러나 곧 사용 중단될 예정이었기 때문에, 이를 관리하는 데 많은 노력을 기울이고 싶지 않았습니다. 다행히도 인기 있는 클라이언트(VSCode 및 Cursor)는 이미 전환을 시작했고, 출시 일주일 이내에 `claude.ai`도 지원을 추가했습니다. SSE 연결이 필요하다면, [FreeCPU Hugging Face Space](https://huggingface.co/new-space)에 서버의 복사본을 배포하여 사용할 수 있습니다.

#### 도구 목록 변경 알림

향후 사용자가 Hub에서 설정을 업데이트할 때 실시간 도구 목록 변경 알림을 지원하고자 합니다. 그러나 이와 관련해 몇 가지 현실적인 문제가 있습니다.

첫째, 사용자들은 클라이언트에서 선호하는 MCP 서버를 설정하고 활성화 상태로 유지하는 경향이 있습니다. 이는 애플리케이션이 열려 있는 동안 클라이언트가 계속 연결된 상태를 유지한다는 뜻입니다. 알림을 전송하려면 사용자가 도구 구성을 변경할 경우를 대비해, 실제 사용 여부와 관계없이 현재 활성 클라이언트 수만큼 열린 연결을 유지해야 합니다. 

둘째, 대부분의 MCP 서버와 클라이언트는 일정 시간 동안 비활성 상태일 경우 연결을 끊고, 필요할 때 재개합니다. 이 때문에 즉각적인 푸시 알림은 채널이 이미 닫혀 있기 때문에 놓칠 수밖에 없습니다. 실제로 클라이언트가 필요에 따라 연결과 도구 목록을 새로고침하는 것이 훨씬 간단합니다.

클라이언트/서버 쌍을 충분히 제어할 수 있는 상황이 아니라면, 도구 목록 새로 고침을 위한 저자원 솔루션이 존재할 때 **서버 푸시 스트림** 사용은 공개 배포에 상당한 복잡성을 가중시킵니다.

#### URL 사용자 경험

출시 직전, [`@julien-c`](https://huggingface.co/julien-c)가 `hf.co/mcp`를 방문하는 사용자를 위한 친절한 안내문을 포함하도록 PR을 제출했습니다. 이 덕분에 사용자 경험이 크게 개선되었으며, 그렇지 않았다면 기본 응답이 다소 불친절한 JSON 응답으로 표시되었을 것입니다.

초기에는 이로 인해 엄청난 양의 트래픽이 발생한다는 것을 발견했습니다. 조사 결과, HTTP 405 오류 대신 웹 페이지를 반환할 경우 VSCode가 해당 엔드포인트를 초당 여러 번 반복 요청(polling)한다는 사실을 알아냈습니다!

[`@coyotte508`](https://huggingface.co/coyotte508)가 제안한 해결책은 브라우저를 정확히 감지하여 해당 상황에서만 페이지를 반환하는 것이었습니다. 또한 이를 신속하게 [수정](https://github.com/microsoft/vscode/pull/251288/files)해준 VSCode 팀에도 감사드립니다. 

명시적으로 규정되지는 않았지만, 이러한 방식으로 페이지를 반환하는 것은 MCP 사양(MCP Specification) 내에서 _실제로_ 허용되는 것으로 보입니다. 

#### MCP 클라이언트 동작

MCP 프로토콜은 초기화 과정에서 여러 요청을 전송합니다. 일반적인 연결 순서는 다음과 같습니다: `Initialize`, `Notifications/Initialize`, `tools/list`, 그리고 `prompts/list`. 

MCP 클라이언트는 열려 있는 동안 연결과 재연결을 수행하며, 사용자가 주기적으로 호출을 한다는 점을 고려할 때, 각 도구 호출당 약 100개의 MCP 제어 메시지가 발생하는 비율을 확인했습니다.

일부 클라이언트는 무상태(Stateless) 직접 응답(Direct Response) 구성에 부합하지 않는 요청도 전송합니다. 예를 들어 핑, 취소 요청 또는 자원 목록 조회 시도(현재 지원하지 않는 기능) 등이 있습니다.

2025년 7월 첫 주에는 무려 164개의 서로 다른 클라이언트가 서버에 접속했습니다. 흥미롭게도 가장 인기 있는 도구 중 하나는 [`mcp-remote`](https://github.com/geelen/mcp-remote)입니다. 전체 클라이언트의 약 절반이 이 도구를 원격 서버 연결을 위한 중개 수단(bridge)으로 사용합니다. 

## 결론

MCP는 빠르게 발전하고 있으며, 지난 몇 달간 채팅 애플리케이션, IDE, 에이전트 및 MCP 서버 전반에서 매우 고무적인 성과를 이뤘습니다.

Hugging Face Hub의 통합이 얼마나 강력한지 확연히 알 수 있었고, Gradio Spaces 지원으로 이제 대규모 언어 모델을 최신 [머신 러닝 애플리케이션](https://huggingface.co/blog/gradio-mcp-servers)으로 쉽게 확장할 수 있게 되었습니다.

지금까지 MCP 서버로 구현된 훌륭한 사례 몇 가지를 소개합니다:
- [영상 제작 오케스트레이션](https://x.com/victormustar/status/1937095435316822244)
- [이미지 편집](https://x.com/reach_vb/status/1942247029515735263)
- [문서 검색](https://x.com/NielsRogge/status/1940472422790242561)
- [AI 애플리케이션 개발](https://x.com/llmindsetuk/status/1940358288220336514)
- [기존 모델에 추론 기능 추가](https://www.linkedin.com/posts/ben-burtenshaw_im-a-big-fan-of-local-models-in-lmstudio-activity-7344001099533590528-zNyw)

이 글이 원격 MCP 서버 구축 시 필요한 결정 사항에 대한 통찰을 제공했기를 바라며, 여러분이 선호하는 MCP 클라이언트에서 예제들을 직접 시도해 보시길 권장합니다.

[오픈 소스 MCP 서버](https://github.com/evalstate/hf-mcp-server)를 둘러보고 클라이언트에서 다양한 전송 옵션을 시험해보세요. 개선 사항 또는 새로운 기능 제안을 위한 이슈나 풀 리퀘스트를 열어 주셔도 좋습니다.

이 [토론 스레드](https://huggingface.co/spaces/huggingface/README/discussions/26)에서 여러분의 생각, 피드백 및 질문을 공유해 주시길 바랍니다.