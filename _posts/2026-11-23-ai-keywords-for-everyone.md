---
layout: post
title: '뉴스에 나오는 AI 용어, 한 번에 정리하기: LLM부터 에이전트, MCP까지'
author : 효준
date: 2026-11-23 10:00
tags: [ai]
image: /files/covers/neuron.jpg
---

## 뉴스에 나오는 AI 용어, 한 번에 정리하기

요즘 기사나 회사 회의에서 "LLM", "RAG", "AI 에이전트", "MCP" 같은 말을 자주 듣습니다. 들을 때마다 대충 고개는 끄덕이지만, 막상 설명하려면 막히는 경우가 많습니다.

이 글은 개발자가 아니어도 읽을 수 있게 썼습니다. 자주 나오는 AI 용어 일곱 개와, 그 용어 뒤에 있는 주요 회사들을 순서대로 정리했습니다. 날짜와 사실은 각 회사의 공식 발표와 원 논문을 기준으로 적었고, 글 끝에 출처를 모아 두었습니다.

<!--more-->

## 1. 한눈에 보는 용어 지도

먼저 전체 그림입니다. 아래로 갈수록 앞의 개념을 바탕으로 만들어진 것입니다.

| 용어 | 한 줄 설명 | 비유 |
|---|---|---|
| 트랜스포머 | 지금 AI 대부분이 쓰는 기본 설계 방식 | 엔진의 설계도 |
| LLM | 글을 아주 많이 읽고 다음 말을 예측하도록 학습한 모델 | 엄청나게 책을 많이 읽은 사람 |
| 생성형 AI | 글·그림·소리 등을 새로 만들어 내는 AI | 쓰고 그리는 사람 |
| 멀티모달 | 글뿐 아니라 사진·음성·영상도 함께 다루는 것 | 눈과 귀가 생긴 것 |
| 환각 | 그럴듯하지만 틀린 내용을 지어내는 현상 | 모르는 걸 아는 척하기 |
| RAG | 답하기 전에 자료를 먼저 찾아보게 하는 방법 | 오픈북 시험 |
| AI 에이전트 | 스스로 도구를 쓰며 여러 단계를 처리하는 AI | 심부름을 알아서 해 오는 비서 |
| MCP | AI와 외부 도구를 연결하는 공통 규격 | AI용 USB-C 포트 |

## 2. 기본이 되는 개념

### 트랜스포머(Transformer)

트랜스포머는 2017년 구글 연구진이 발표한 논문 「Attention Is All You Need」에서 처음 소개된 모델 구조입니다. 문장 안의 단어들이 서로 얼마나 관련이 있는지를 한꺼번에 계산하는 "어텐션(attention)"이라는 방식이 핵심입니다.

논문은 원래 번역 문제를 다뤘습니다. 그런데 이 구조가 다른 분야에서도 잘 통하면서, 지금 우리가 쓰는 대화형 AI 대부분의 바탕이 되었습니다. ChatGPT의 "GPT"에서 T도 Transformer의 약자입니다.

### LLM(대규모 언어 모델)

LLM은 Large Language Model의 줄임말입니다. 아주 많은 양의 글을 읽히면서 "이 다음에 올 단어는 무엇일까"를 맞히도록 학습한 모델입니다.

이렇게만 들으면 단순한 자동완성 같습니다. 하지만 학습한 글의 양과 모델의 크기가 커지자 요약, 번역, 질문 답변, 글쓰기 같은 일을 꽤 잘하게 되었습니다. 우리가 "AI 챗봇"이라고 부르는 서비스 대부분은 그 안에 LLM이 들어 있습니다.

### 생성형 AI와 멀티모달

**생성형 AI**는 무언가를 새로 만들어 내는 AI를 넓게 부르는 말입니다. 글을 쓰는 LLM도 생성형 AI이고, 그림이나 음악을 만드는 AI도 여기에 들어갑니다.

**멀티모달(multimodal)**은 글, 이미지, 음성, 영상처럼 서로 다른 형태의 정보를 함께 다룬다는 뜻입니다. 예를 들어 사진을 보여 주고 "이 그래프를 설명해 줘"라고 물을 수 있다면 멀티모달 기능을 쓰는 것입니다. 구글은 2023년 12월 Gemini를 발표하면서 처음부터 여러 형태의 데이터를 함께 학습시킨 멀티모달 모델이라고 소개했습니다.

## 3. 알고 쓰면 좋은 개념

### 환각(Hallucination)

환각은 AI가 사실이 아닌 내용을 그럴듯하게 지어내는 현상입니다. 없는 논문 제목을 대거나, 틀린 날짜를 자신 있게 말하는 경우가 대표적입니다.

OpenAI는 2025년 9월 공개한 연구에서 그 원인 중 하나를 이렇게 설명했습니다. 지금의 학습·평가 방식이 "모르겠다"고 답하는 것보다 일단 찍는 쪽에 더 높은 점수를 주기 쉽다는 것입니다. 객관식 시험에서 빈칸으로 두면 0점이니 일단 찍는 것과 비슷합니다. 같은 글에서 OpenAI는 환각이 줄어들고는 있지만 아직 해결된 문제는 아니라고 밝혔습니다.

그래서 AI가 알려 준 숫자, 날짜, 인용문은 원래 자료에서 한 번 더 확인하는 습관이 필요합니다.

### RAG(검색 증강 생성)

RAG는 Retrieval-Augmented Generation의 줄임말입니다. 2020년 Patrick Lewis 등 연구진이 발표한 논문에서 이 이름을 처음 썼습니다.

방식은 간단합니다. AI가 답을 만들기 전에 관련 문서를 먼저 찾아오고, 그 문서를 참고해서 답하게 합니다. 외운 것만으로 답하는 "닫힌 책 시험"이 아니라, 자료를 펴 놓고 푸는 "오픈북 시험"에 가깝습니다.

회사 안의 규정집이나 매뉴얼을 바탕으로 답하는 사내 챗봇이 대표적인 RAG 활용 사례입니다. 모델을 새로 학습시키지 않아도 최신 자료나 회사만의 자료를 반영할 수 있다는 점이 장점입니다.

## 4. 요즘 가장 많이 들리는 개념

### AI 에이전트(Agent)

지금까지의 챗봇이 "질문하면 답하는" 방식이었다면, 에이전트는 목표를 주면 필요한 단계를 스스로 정하고 도구를 써 가며 일을 처리하는 방식입니다.

Anthropic은 2024년 12월 글 「Building effective agents」에서 둘을 이렇게 나눴습니다.

- **워크플로(workflow):** 사람이 미리 정해 둔 순서대로 AI와 도구가 움직이는 방식
- **에이전트(agent):** AI가 상황을 보고 다음에 무엇을 할지, 어떤 도구를 쓸지 스스로 정하는 방식

같은 글에서 Anthropic은 가능한 한 가장 단순한 방법을 먼저 찾고, 꼭 필요할 때만 복잡도를 높이라고 권합니다. 예측 가능해야 하는 일은 워크플로가, 유연하게 판단해야 하는 일은 에이전트가 더 잘 맞는다는 것입니다.

### MCP(Model Context Protocol)

에이전트가 일을 하려면 메일, 캘린더, 파일 저장소, 사내 시스템 같은 외부 도구에 접근해야 합니다. 문제는 도구마다 연결 방법이 제각각이라는 점이었습니다.

MCP는 이 연결 방법을 하나로 맞추려는 공개 규격입니다. Anthropic이 2024년 11월 처음 발표했고, 발표 당시 Google Drive, Slack, GitHub 같은 서비스와 연결하는 예제 서버도 함께 공개했습니다. 휴대폰 충전기가 USB-C로 통일되면 하나의 케이블로 여러 기기를 충전할 수 있듯, 도구 쪽에서 MCP를 한 번 지원하면 여러 AI 서비스에서 그 도구를 쓸 수 있게 됩니다.

2025년 12월에는 리눅스 재단 아래에 Agentic AI Foundation이 만들어지면서 MCP도 이곳으로 옮겨졌습니다. 발표에 따르면 Anthropic, Block, OpenAI가 함께 세웠고, 특정 회사가 아닌 중립적인 곳에서 규격을 관리하게 되었습니다.

## 5. 용어 뒤에 있는 주요 회사들

모델 이름과 버전은 몇 달마다 바뀌기 때문에, 여기서는 각 회사가 처음 대표 서비스를 내놓은 시점을 기준으로 정리했습니다.

| 회사 | 대표 서비스·모델 | 처음 공개 | 특징 |
|---|---|---|---|
| OpenAI | ChatGPT | 2022년 11월 30일 | 대화형 AI를 대중에게 널리 알린 서비스. 처음엔 무료 연구용 미리보기로 공개 |
| Anthropic | Claude | 2023년 3월 14일 | MCP를 처음 만든 회사. 이후 MCP를 리눅스 재단 산하 재단에 기증 |
| 구글(Google DeepMind) | Gemini | 2023년 12월 | 트랜스포머 논문을 낸 곳. 처음부터 멀티모달로 설계했다고 소개 |
| 메타(Meta) | Llama | 2023년 2월 24일 | 연구자에게 모델을 내려받아 쓸 수 있게 공개하는 방식을 택함 |
| 네이버 | 하이퍼클로바X | 2023년 8월 24일 | 한국어에 맞춰 만든 초대규모 AI. 2021년 하이퍼클로바의 후속 모델 |

한 가지 덧붙이면, 이 표는 회사의 우열이나 가치를 비교하려는 것이 아닙니다. 어떤 회사가 어떤 흐름을 만들었는지 큰 줄기를 보려는 정리입니다.

## 6. 마무리

정리하면 이렇습니다.

- **트랜스포머**라는 설계 위에 **LLM**이 만들어졌고,
- LLM이 글 말고 사진·음성도 다루게 된 것이 **멀티모달**,
- 틀린 말을 지어내는 **환각**을 줄이려고 자료를 먼저 찾게 하는 것이 **RAG**,
- 여러 단계를 스스로 처리하는 것이 **에이전트**,
- 에이전트가 외부 도구에 연결되는 공통 규격이 **MCP**입니다.

다음에 AI 관련 기사를 볼 때 이 순서를 떠올리면, 새로운 서비스가 나와도 "아, 이건 이 단계 이야기구나" 하고 위치를 잡기 쉬워집니다. 2025년에 쓴 [AI 생태계 정리 글](/2025/06/27/ai-ecosystem-overview/)과 함께 읽으셔도 좋습니다.

## 참고 자료

- Vaswani 외, [Attention Is All You Need](https://arxiv.org/abs/1706.03762), 2017
- Lewis 외, [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401), 2020
- OpenAI, [Introducing ChatGPT](https://openai.com/index/chatgpt/), 2022-11-30
- OpenAI, [Why language models hallucinate](https://openai.com/index/why-language-models-hallucinate/), 2025-09
- Anthropic, [Introducing Claude](https://www.anthropic.com/news/introducing-claude), 2023-03-14
- Anthropic, [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents), 2024-12
- Anthropic, [Introducing the Model Context Protocol](https://www.anthropic.com/news/model-context-protocol), 2024-11-25
- Linux Foundation, [Linux Foundation Announces the Formation of the Agentic AI Foundation](https://www.linuxfoundation.org/press/linux-foundation-announces-the-formation-of-the-agentic-ai-foundation), 2025-12-09
- MCP Blog, [MCP joins the Agentic AI Foundation](https://blog.modelcontextprotocol.io/posts/2025-12-09-mcp-joins-agentic-ai-foundation/), 2025-12-09
- Google, [Introducing Gemini: our largest and most capable AI model](https://blog.google/technology/ai/google-gemini-ai/), 2023-12-06
- Meta, [Introducing LLaMA: A foundational, 65-billion-parameter large language model](https://ai.meta.com/blog/large-language-model-llama-meta-ai/), 2023-02-24
- 네이버, [보도자료: 하이퍼클로바X 기반 서비스 라인업 공개(8월 공개 예고)](https://www.navercorp.com/media/pressReleasesDetail?seq=31233), 2023-07-21
- TechCrunch, [Korea's internet giant Naver unveils generative AI services](https://techcrunch.com/2023/08/24/koreas-internet-giant-naver-unveils-generative-ai-services/), 2023-08-24
