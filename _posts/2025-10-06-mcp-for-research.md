---
layout: post
title: "MCP for Research: AI와 연구 도구 연결하기"
author: hyojung
categories: [MCP, Research]
image: assets/images/blog/posts/2025-10-06-mcp-for-research/thumbnail.png
---
* TOC
{:toc}
<!--toc-->

_이 글은 Hugging Face 블로그의 [MCP for Research: How to Connect AI to Research Tools](https://huggingface.co/blog/mcp-for-research)를 한국어로 번역한 글입니다._

---


# MCP for Research: 연구 도구와 AI 연결하기

학술 연구에서는 논문, 코드, 관련 모델과 데이터셋을 찾는 **연구 탐색(research discovery)**이 빈번히 발생합니다. 이는 보통 [arXiv](https://arxiv.org/), [GitHub](https://github.com/), and [Hugging Face](https://huggingface.co/)와 같은 여러 플랫폼을 오가며 수동으로 연결 관계를 파악해야 한다는 뜻이기도 합니다.

[Model Context Protocol (MCP)](https://huggingface.co/learn/mcp-course/unit0/introduction)은 에이전틱 모델(agentic models)이 외부 도구 및 데이터 소스와 통신할 수 있도록 하는 표준입니다. 연구 탐색에 있어 이는 AI가 자연어 요청을 통해 연구 도구를 사용할 수 있게 하여, 플랫폼 간 전환과 교차 참조를 자동화한다는 의미입니다.

![Research Tracker MCP in action](./assets/mcp-for-research/demo.gif)

## 연구 탐색: 세 가지 추상화 계층

소프트웨어 개발과 마찬가지로, 연구 탐색도 추상화 계층으로 설명할 수 있습니다.

### 1. 수동 연구

가장 낮은 추상화 계층에서는 연구자가 직접 검색하고 수작업으로 교차 참조합니다.

```bash
# 전형적인 워크플로우:
1. arXiv에서 논문 찾기
2. GitHub에서 구현체 검색
3. Hugging Face에서 모델/데이터셋 확인
4. 저자와 인용 관계 교차 검토
5. 수동으로 결과 정리
```

이러한 수동 접근 방식은 여러 연구 주제를 추적하거나 체계적인 문헌 검토를 수행할 때 비효율적입니다. 반복적으로 여러 플랫폼을 검색하고 메타데이터를 추출하며 정보를 교차 검토하는 과정은 스크립트를 통한 자동화로 이어지는 것이 자연스럽습니다.

### 2. 스크립트 기반 도구

Python 스크립트를 활용하면 웹 요청 처리, 응답 파싱, 결과 정리 등을 자동화할 수 있습니다.

```python
# research_tracker.py
def gather_research_info(paper_url):
    paper_data = scrape_arxiv(paper_url)
    github_repos = search_github(paper_data['title'])
    hf_models = search_huggingface(paper_data['authors'])
    return consolidate_results(paper_data, github_repos, hf_models)

# 분석할 논문마다 실행
results = gather_research_info("https://arxiv.org/abs/2103.00020")
```

[research tracker](https://huggingface.co/spaces/dylanebert/research-tracker)는 이러한 스크립트를 기반으로 체계적인 연구 탐색을 시연합니다.

스크립트는 수동 연구보다 빠르지만, API 변경, 요청 제한, 파싱 오류 때문에 자동 데이터 수집이 실패하는 경우가 있습니다. 사람의 개입이 없으면 관련 결과를 놓치거나 불완전한 정보를 반환할 수 있습니다.

### 3. MCP 통합

MCP를 사용하면 동일한 Python 도구를 AI 시스템이 자연어로 접근할 수 있습니다.

```markdown
# 예시 연구 지시문
최근 6개월 내 발표된 트랜스포머 아키텍처 논문을 찾아라:
- 구현 코드가 있는 논문만
- 사전학습 모델 제공 논문에 집중
- 가능하면 성능 벤치마크 포함
```

AI는 여러 도구를 오케스트레이션하고, 누락된 정보를 보완하며, 결과를 분석합니다.

```python
# AI 워크플로우:
# 1. 연구 추적 도구 활용
# 2. 누락된 정보 검색
# 3. 다른 MCP 서버와 교차 참조
# 4. 연구 목표와의 관련성 평가

user: "이 논문에 관한 모든 관련 정보(코드, 모델 등)를 찾아줘: https://huggingface.co/papers/2010.11929"
ai: # 여러 도구를 결합해 완전한 정보 수집
```

이는 스크립트 위에 존재하는 또 다른 추상화 계층으로, 여기서 “프로그래밍 언어”는 자연어가 됩니다. 이는 [Software 3.0 비유](https://youtu.be/LCEmiRjPEtQ?si=J7elM86eW9XCkMFj)를 따르며, 자연어 연구 지시가 곧 소프트웨어 구현인 셈입니다.

물론 스크립트와 동일한 주의사항이 있습니다:

- 수동 연구보다 빠르지만, 사람의 가이드 없이는 오류 발생 가능
- 품질은 구현에 따라 좌우됨
- 하위 계층(수동, 스크립트)을 이해해야 더 나은 구현 가능

## 설정 및 사용법

### 빠른 설정

Research Tracker MCP를 추가하는 가장 쉬운 방법은 [Hugging Face MCP 설정](https://huggingface.co/settings/mcp)을 이용하는 것입니다:

1. [huggingface.co/settings/mcp](https://huggingface.co/settings/mcp) 방문
2. 사용 가능한 도구 목록에서 "research-tracker-mcp" 검색
3. 클릭하여 도구에 추가
4. 사용하는 클라이언트(Claude Desktop, Cursor, Claude Code, VS Code 등)에 맞는 설치 지침 따르기

이 워크플로우는 Hugging Face MCP 서버를 활용하며, Hugging Face Spaces를 MCP 도구로 사용하는 표준 방법입니다. 설정 페이지에서는 클라이언트별 설정이 자동으로 생성되고 항상 최신 상태를 유지합니다.

<script
	type="module"
	src="https://gradio.s3-us-west-2.amazonaws.com/4.36.1/gradio.js"
></script>

<gradio-app theme_mode="light" space="dylanebert/research-tracker-mcp"></gradio-app>

## 더 알아보기

**시작하기:**
- [Hugging Face MCP Course](https://huggingface.co/learn/mcp-course/en/unit1/introduction) - 기초부터 도구 제작까지 완전 가이드
- [MCP 공식 문서](https://modelcontextprotocol.io) - 프로토콜 사양 및 아키텍처

**직접 구축하기:**
- [Gradio MCP 가이드](https://www.gradio.app/guides/building-mcp-server-with-gradio) - Python 함수를 MCP 도구로 변환하기
- [Hugging Face MCP 서버 구축](https://huggingface.co/blog/building-hf-mcp) - 실제 프로덕션 구현 사례

**커뮤니티:**
- [Hugging Face Discord](https://hf.co/join/discord) - MCP 개발 논의

연구 탐색을 자동화할 준비가 되었나요? [Research Tracker MCP](https://huggingface.co/settings/mcp)를 사용해 보거나 위의 리소스를 활용해 직접 연구 도구를 구축해 보세요.