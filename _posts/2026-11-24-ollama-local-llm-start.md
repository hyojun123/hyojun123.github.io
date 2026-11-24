---
layout: post
title: 'Ollama로 로컬 LLM 시작하기: 가벼운 모델 설치부터 대화, API·파이썬 호출까지'
author : 효준
date: 2026-11-24 18:00
tags: [ai]
image: /files/covers/solid-state-logic.jpg
---

## 내 컴퓨터에서 AI 모델 돌려 보기

ChatGPT나 Claude 같은 서비스는 인터넷 너머의 서버에서 모델이 돌아갑니다. 반면 **로컬 LLM**은 모델 파일을 내 컴퓨터에 내려받아 직접 실행합니다. 인터넷이 없어도 쓸 수 있고, 입력한 내용이 내 컴퓨터 밖으로 나가지 않으며, 사용량 요금도 없습니다.

이 글은 로컬 LLM을 처음 써 보는 분을 위해 **Ollama**로 가벼운 모델을 설치하고, 터미널 대화 → API 호출 → 파이썬 코드 → 나만의 모델 설정까지 따라 해 보는 과정을 정리했습니다. 초보자용 로컬 LLM 시리즈의 1편이고, 다음 편에서는 내 문서로 답하는 RAG를 다룹니다.

<!--more-->

글의 실행 결과는 Apple M4 Pro(메모리 48GB) Mac, Ollama 0.40.16, `gemma3:1b` 모델로 직접 돌려 본 것입니다. 속도와 답변 내용은 컴퓨터 사양, 버전, 실행할 때마다 달라질 수 있습니다.

## 1. Ollama가 뭔가요

Ollama는 공개된 LLM을 내려받아 내 컴퓨터에서 실행하게 해 주는 도구입니다. 명령 한 줄로 모델을 받아 대화할 수 있고, 실행하는 동안 `http://localhost:11434`에서 API 서버가 함께 돌아 다른 프로그램에서도 모델을 부를 수 있습니다.

한 가지 알아 둘 점이 있습니다. Ollama 라이브러리에는 이름에 `cloud`가 붙은 모델도 있는데, 이런 모델은 내 컴퓨터가 아니라 Ollama의 클라우드에서 실행됩니다. 이 글에서는 내 컴퓨터에서 도는 모델만 다룹니다.

## 2. 설치하기

Ollama 공식 문서 기준으로 macOS는 **Sonoma(14) 이상**이 필요합니다. Apple M 시리즈 칩은 CPU와 GPU를 모두 쓰고, 인텔(x86) Mac은 CPU만 씁니다.

1. [ollama.com/download](https://ollama.com/download)에서 macOS용 파일을 받습니다.
2. 받은 `ollama.dmg`를 열고 Ollama 앱을 응용 프로그램 폴더로 옮깁니다.
3. 앱을 처음 실행하면 터미널에서 `ollama` 명령을 쓸 수 있게 연결할지 묻습니다. 허용하면 됩니다.
4. 터미널에서 설치를 확인합니다.

```bash
ollama --version
```

모델과 설정은 홈 폴더의 `~/.ollama`에 저장됩니다. 모델 하나가 수백 MB에서 수십 GB까지 하므로 디스크 여유 공간을 먼저 확인해 두세요. Windows와 Linux도 같은 다운로드 페이지에서 받을 수 있습니다.

## 3. 가벼운 모델 고르기

처음에는 **작은 모델**로 시작하는 게 좋습니다. 내려받는 시간이 짧고 메모리도 적게 씁니다. Ollama 라이브러리에 올라온 가벼운 모델 몇 가지는 다음과 같습니다(글 작성 시점의 다운로드 크기).

| 모델 | 크기 | 컨텍스트 | 참고 |
|---|---|---|---|
| `gemma3:270m` | 292MB | 32K | 가장 작음, 텍스트만 |
| `gemma3:1b` | 806MB | 32K | 이 글에서 사용 |
| `gemma3:4b` | 3.4GB | 128K | 이미지 입력도 지원 |
| `qwen3:0.6b` | 523MB | 40K | |
| `qwen3:1.7b` | 1.4GB | 40K | |
| `llama3.2:1b` | 1.3GB | 128K | |
| `llama3.2:3b` | 2.0GB | 128K | |

언어 지원은 모델마다 다릅니다. Gemma 3는 140개가 넘는 언어, Qwen3는 100개가 넘는 언어와 방언을 지원한다고 소개합니다. Llama 3.2 페이지가 공식 지원 언어로 꼽은 8개 언어에는 한국어가 없습니다. 한국어로 쓸 계획이라면 이 점을 참고해 고르세요.

이 글에서는 크기가 작고 다국어를 지원하는 `gemma3:1b`를 씁니다. 숫자 `1b`는 매개변수(파라미터)가 약 10억 개라는 뜻으로, 클수록 답이 좋아지는 대신 무겁습니다.

## 4. 모델 받고 대화하기

모델은 `pull`로 받고, `run`으로 실행합니다. `run`은 모델이 없으면 알아서 먼저 받습니다.

```bash
ollama pull gemma3:1b
ollama run gemma3:1b
```

`>>>` 표시가 나오면 질문을 입력하면 됩니다. 대화를 마치려면 `/bye`를 입력합니다. 여러 줄을 입력하고 싶으면 따옴표 세 개(`"""`)로 감쌉니다.

```text
>>> """아래 문장을 더 정중하게 바꿔 줘.
... 내일까지 자료 보내 주세요.
... """
```

대화창에 들어가지 않고 질문 하나만 던질 수도 있습니다.

```bash
ollama run gemma3:1b "회의를 짧게 끝내는 팁 3가지를 한국어로 알려 줘."
```

제 Mac에서는 이 명령이 약 3초 만에 끝났고, 팁 3가지를 각각 설명과 함께 한국어로 답했습니다.

## 5. 모델 관리 명령

자주 쓰는 명령은 많지 않습니다.

| 명령 | 하는 일 |
|---|---|
| `ollama ls` | 내려받은 모델 목록 |
| `ollama ps` | 지금 메모리에 올라가 있는 모델 |
| `ollama stop gemma3:1b` | 메모리에서 내리기 |
| `ollama rm gemma3:1b` | 모델 파일 삭제 |
| `ollama show gemma3:1b` | 모델 정보(크기, 컨텍스트, 라이선스 등) |

`ollama ps`로 확인해 보니 `gemma3:1b`는 메모리를 974MB 쓰고 GPU에서 100% 돌고 있었습니다. 모델은 쓰지 않고 몇 분이 지나면 메모리에서 자동으로 내려갑니다. `ollama show`를 실행하면 매개변수 약 10억 개, 4비트 양자화(Q4_K_M), 라이선스 Gemma Terms of Use 같은 정보를 볼 수 있습니다.

## 6. API로 부르기

Ollama가 실행 중이면 API 서버도 함께 돌고 있습니다. 다른 프로그램에서는 이 주소로 모델을 부를 수 있습니다.

```bash
curl http://localhost:11434/api/chat -d '{
  "model": "gemma3:1b",
  "messages": [
    { "role": "user", "content": "Docker를 한 문장으로 설명해 줘." }
  ],
  "stream": false
}'
```

`stream`을 생략하면 답이 한 글자씩 나눠 오는 스트리밍 방식이 기본입니다. `false`로 두면 완성된 답을 한 번에 받습니다. 응답 JSON의 `message.content`에 답이 들어 있고, 다음 값으로 성능도 볼 수 있습니다.

- `prompt_eval_count`: 입력 토큰 수
- `eval_count`: 생성한 토큰 수
- `eval_duration`, `total_duration`: 걸린 시간(나노초)

제 환경에서는 입력 20토큰, 출력 27토큰이었습니다. 전체 0.3초가 걸렸고, 계산해 보면 초당 약 189토큰을 생성했습니다. 답은 "Docker는 컨테이너를 사용하여 애플리케이션과 그 종속성을 패키징하고 실행하기 위한 플랫폼입니다."였습니다.

## 7. 파이썬에서 쓰기

### 방법 1: Ollama 공식 파이썬 라이브러리

```bash
pip install ollama
```

```python
from ollama import chat

response = chat(
    model="gemma3:1b",
    messages=[{"role": "user", "content": "파이썬 리스트와 튜플의 차이를 두 줄로 알려 줘."}],
)
print(response.message.content)
```

### 방법 2: OpenAI 라이브러리 그대로 쓰기

Ollama는 OpenAI API와 호환되는 주소(`/v1/`)도 제공합니다. 이미 OpenAI 라이브러리로 만든 코드가 있다면 주소만 바꿔 로컬 모델로 시험해 볼 수 있습니다. API 키 값은 꼭 넣어야 하지만 Ollama는 이 값을 무시합니다.

```python
from openai import OpenAI

client = OpenAI(base_url="http://localhost:11434/v1/", api_key="ollama")
r = client.chat.completions.create(
    model="gemma3:1b",
    messages=[{"role": "user", "content": "HTTP 404가 뭔지 한 문장으로 알려 줘."}],
)
print(r.choices[0].message.content)
```

두 코드 모두 `ollama` 0.6.3, `openai` 3.58.0 라이브러리로 실행해 답을 받는 것까지 확인했습니다.

## 8. Modelfile로 나만의 모델 만들기

매번 "한국어로 짧게 답해 줘"라고 붙이기 번거롭다면, **Modelfile**에 기본 지시(시스템 프롬프트)와 설정을 적어 새 이름의 모델로 저장할 수 있습니다.

```text
FROM gemma3:1b
PARAMETER temperature 0.3
SYSTEM """너는 신입 개발자를 돕는 친절한 선배야. 항상 한국어로, 세 문장 이내로 답해."""
```

- `FROM`: 바탕이 될 모델
- `PARAMETER temperature`: 낮을수록 답이 일정하고, 높을수록 다양해집니다
- `SYSTEM`: 모든 대화에 앞서 적용할 지시

```bash
ollama create dev-helper -f ./Modelfile
ollama run dev-helper
```

제 환경에서는 몇 초 만에 만들어졌고, 위 설정대로 친절한 말투의 한국어로 답했습니다. 프롬프트를 잘 쓰는 요령은 [AI 챗봇에게 잘 묻는 법](/2026/11/23/how-to-ask-ai-well/)에 정리해 두었습니다.

## 9. 작은 모델의 한계: 직접 확인한 것

가벼운 모델은 빠르고 편하지만, 답을 그대로 믿으면 안 됩니다. 실험하면서 이런 일이 있었습니다.

- **없는 사실을 지어냄:** "2024년 노벨 문학상 수상자는 누구야?"라고 물었더니, 존재하는지 확인되지 않는 덴마크 작가의 이름을 댔습니다. 노벨상 공식 사이트에 따르면 2024년 수상자는 한강 작가입니다.
- **지시를 다 지키지 못함:** "두 줄로"라고 했는데 여러 줄로 답했고, `dev-helper`에게 "세 문장 이내"를 지시했는데도 더 길게 답했습니다.
- **설명이 부정확함:** `dev-helper`에게 `git rebase`를 물었을 때 실제 동작과 다른 설명이 섞여 있었습니다.

반면 "한국의 수도는?"처럼 쉬운 질문에는 "서울"이라고 정확히 답했습니다. 정리하면 1B급 모델은 **문장 다듬기, 간단한 분류, 형식 바꾸기**처럼 정답을 내가 바로 확인할 수 있는 일에 쓰고, 사실 확인이 필요한 질문에는 쓰지 않는 편이 안전합니다. 품질이 더 필요하면 `gemma3:4b`처럼 큰 모델로 바꿔 보세요. 메모리와 속도의 차이를 직접 비교해 보는 것도 좋습니다.

## 10. 쓰기 전에 확인할 것

- **라이선스:** 모델마다 라이선스가 다릅니다. `gemma3`는 Gemma Terms of Use를 따릅니다. 회사 업무나 서비스에 쓰려면 `ollama show`와 모델 페이지에서 라이선스를 먼저 확인하세요.
- **사내 규정:** 로컬 모델이라도 회사 PC에 프로그램을 설치하거나 업무 자료를 넣는 것이 사내 규정에 맞는지 확인하세요.
- **클라우드 모델 구분:** `cloud`가 붙은 모델은 내 컴퓨터 밖에서 실행되므로, 데이터를 밖으로 보내지 않으려면 쓰지 마세요.
- **디스크 정리:** 다 쓴 모델은 `ollama rm`으로 지워 공간을 확보합니다.

## 마무리

- Ollama를 설치하면 `ollama run 모델이름` 한 줄로 로컬 LLM과 대화할 수 있습니다.
- 처음에는 `gemma3:1b`처럼 1GB 안쪽의 가벼운 모델로 시작하면 부담이 적습니다.
- `localhost:11434`의 API, 공식 파이썬 라이브러리, OpenAI 호환 주소로 내 프로그램에서 부를 수 있습니다.
- Modelfile로 기본 지시와 설정을 넣은 나만의 모델을 만들 수 있습니다.
- 작은 모델은 사실을 지어낼 수 있으니, 결과를 바로 확인할 수 있는 일부터 맡기세요.

다음 편에서는 이 모델에 **내 문서를 근거로 답하게 하는 RAG**를 가장 간단한 예제로 만들어 보겠습니다.

## 참고 자료

- Ollama, [Quickstart](https://docs.ollama.com/quickstart)
- Ollama, [macOS](https://docs.ollama.com/macos)
- Ollama, [CLI Reference](https://docs.ollama.com/cli)
- Ollama, [Chat API](https://docs.ollama.com/api/chat)
- Ollama, [OpenAI compatibility](https://docs.ollama.com/api/openai-compatibility)
- Ollama, [Modelfile Reference](https://docs.ollama.com/modelfile)
- Ollama, [gemma3](https://ollama.com/library/gemma3), [qwen3](https://ollama.com/library/qwen3), [llama3.2](https://ollama.com/library/llama3.2) 모델 페이지
- Ollama, [ollama-python](https://github.com/ollama/ollama-python)
- Google, [Gemma Terms of Use](https://ai.google.dev/gemma/terms)
- The Nobel Prize, [The Nobel Prize in Literature 2024](https://www.nobelprize.org/prizes/literature/2024/summary/)
