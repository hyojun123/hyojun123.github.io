---
layout: post
title: 'LLM temperature·top_p·seed 이해하기: logprobs로 다음 단어 후보의 확률 직접 보기'
author : 효준
date: 2026-11-26 17:00
tags: [ai]
image: /files/covers/rubix-cube.jpg
---

## 같은 질문인데 왜 답이 매번 다를까

LLM에 같은 질문을 두 번 던지면 답이 조금씩 다르게 나옵니다. API 문서를 보면 `temperature`, `top_p`, `seed` 같은 옵션이 있고, "낮추면 일정해지고 높이면 다양해진다"는 설명이 붙어 있습니다. 그런데 이 숫자가 모델 안에서 정확히 무엇을 바꾸는지는 설명만 읽어서는 잘 와닿지 않습니다.

이 글에서는 모델이 다음 단어를 고르기 직전에 가지고 있는 **후보별 확률**을 `logprobs` 옵션으로 직접 꺼내 봅니다. 그다음 `temperature`, `top_p`, `top_k`, `seed`를 하나씩 바꿔 가며 답이 어떻게 달라지는지 숫자로 확인합니다. [Ollama로 로컬 LLM 시작하기](/2026/11/24/ollama-local-llm-start/)에서 설치한 환경을 그대로 쓰므로, 아직 Ollama가 없다면 그 글을 먼저 따라 해 주세요.

<!--more-->

글의 실행 결과는 Apple M4 Pro(메모리 48GB) Mac, Ollama 0.40.16, `gemma3:1b` 모델, Python 3.14로 직접 돌려 본 것입니다. 확률 값과 답변 내용은 컴퓨터 사양, Ollama 버전, 모델, 실행할 때마다 달라질 수 있습니다.

## 1. LLM은 한 번에 토큰 하나를 고른다

LLM은 문장을 통째로 만들어 내지 않습니다. 지금까지의 글을 보고 **다음에 올 토큰 하나**를 고르고, 그 토큰을 붙인 글을 다시 보고 또 하나를 고르는 일을 반복합니다. 토큰은 모델이 글을 다루는 단위로, 한 글자일 수도 있고 단어 하나일 수도 있습니다.

토큰을 고를 때 모델은 "정답 하나"를 내놓는 것이 아니라, 자신이 아는 모든 토큰에 확률을 매깁니다. 그 확률표에서 하나를 뽑는 과정을 **샘플링**이라고 합니다. 이 글에서 다루는 옵션은 모두 이 뽑기 단계에 관여합니다.

| 옵션 | 하는 일 |
|---|---|
| `logprobs`, `top_logprobs` | 뽑기에 쓰인 후보와 확률을 응답에 함께 돌려줍니다 |
| `temperature` | 확률표를 뾰족하게 또는 평평하게 만듭니다 |
| `top_k` | 확률이 높은 순서로 K개만 후보로 남깁니다 |
| `top_p` | 확률을 높은 순서로 더해 P에 닿을 때까지만 후보로 남깁니다 |
| `seed` | 뽑기에 쓰는 난수의 시작값을 고정합니다 |

`top_p` 방식은 nucleus sampling이라고도 부릅니다. 2019년에 나온 논문 "The Curious Case of Neural Text Degeneration"이 제안한 방법으로, 확률이 낮아 믿기 어려운 꼬리 부분을 잘라 내고 남은 후보 안에서만 뽑자는 생각입니다.

## 2. 준비: 공통 함수 만들기

별도 라이브러리 없이 파이썬 표준 라이브러리만으로 Ollama API를 부릅니다. 작업 폴더를 하나 만들고 `common.py`를 저장합니다.

```python
"""Ollama /api/chat 를 표준 라이브러리만으로 부르는 공통 함수."""
import json
import urllib.request

URL = "http://localhost:11434/api/chat"
MODEL = "gemma3:1b"


def chat(prompt, options=None, top_logprobs=None):
    body = {
        "model": MODEL,
        "messages": [{"role": "user", "content": prompt}],
        "stream": False,
        "options": options or {},
    }
    if top_logprobs is not None:
        body["logprobs"] = True
        body["top_logprobs"] = top_logprobs
    req = urllib.request.Request(
        URL,
        data=json.dumps(body).encode("utf-8"),
        headers={"Content-Type": "application/json"},
    )
    with urllib.request.urlopen(req, timeout=120) as res:
        return json.loads(res.read())
```

여기서 눈여겨볼 점은 옵션이 들어가는 자리입니다. `temperature`, `top_p`, `top_k`, `seed`, `num_predict`(최대 생성 토큰 수)는 `options` 안에 넣고, `logprobs`와 `top_logprobs`는 `options` 바깥인 요청 본문 최상위에 넣습니다.

## 3. logprobs로 후보 확률 보기

Ollama 공식 문서에 따르면 `logprobs`를 `true`로 주면 생성한 토큰의 로그 확률을 돌려주고, `top_logprobs`에 숫자를 주면 각 자리에서 확률이 높았던 후보를 그만큼 함께 돌려줍니다. 먼저 `curl`로 응답 모양을 확인합니다.

```bash
curl -s http://localhost:11434/api/chat -d '{
  "model": "gemma3:1b",
  "messages": [{"role": "user", "content": "대한민국의 수도는 어디야? 도시 이름만 답해."}],
  "stream": false,
  "logprobs": true,
  "top_logprobs": 5,
  "options": {"temperature": 0, "num_predict": 8}
}'
```

응답에는 `logprobs` 배열이 추가됩니다. 토큰 하나마다 `token`, `logprob`, `bytes`, `top_logprobs`가 들어 있습니다. 아래는 제 환경의 응답에서 `bytes`를 빼고 줄인 것입니다.

```json
{
  "message": {"role": "assistant", "content": "서울"},
  "logprobs": [
    {
      "token": "서울",
      "logprob": -0.5564380288124084,
      "top_logprobs": [
        {"token": "서울", "logprob": -0.5564380288124084},
        {"token": "서울특별시", "logprob": -0.8532252907752991},
        {"token": " 서울", "logprob": -7.945809841156006},
        {"token": "Seoul", "logprob": -8.654275894165039},
        {"token": "성", "logprob": -10.103656768798828}
      ]
    }
  ]
}
```

`logprob`은 확률에 자연로그를 씌운 값입니다. 그래서 `math.exp()`에 넣으면 확률로 돌아옵니다. `exp(-0.556)`은 약 0.573이므로 "서울"의 확률은 57.3%입니다. 값이 0에 가까울수록 확률이 높고, 음수로 커질수록 낮습니다.

매번 손으로 계산하기 번거로우니 스크립트로 만듭니다. `01_logprobs.py`입니다.

```python
"""답의 토큰마다 후보 5개와 확률을 출력한다."""
import math
from common import chat

PROMPTS = [
    "대한민국의 수도는 어디야? 도시 이름만 답해.",
    "새로 여는 동네 카페 이름을 하나만 지어 줘. 이름만 답해.",
]

for prompt in PROMPTS:
    r = chat(prompt, {"temperature": 0, "num_predict": 12}, top_logprobs=5)
    print(f"\n질문: {prompt}")
    print(f"답: {r['message']['content']!r}")
    for i, pos in enumerate(r["logprobs"], 1):
        cands = ", ".join(
            f"{c['token']!r} {math.exp(c['logprob']) * 100:.1f}%"
            for c in pos["top_logprobs"]
        )
        print(f"  {i}번째 토큰 {pos['token']!r}: {cands}")
```

```bash
python3 01_logprobs.py
```

```text
질문: 대한민국의 수도는 어디야? 도시 이름만 답해.
답: '서울'
  1번째 토큰 '서울': '서울' 57.3%, '서울특별시' 42.6%, ' 서울' 0.0%, 'Seoul' 0.0%, '성' 0.0%

질문: 새로 여는 동네 카페 이름을 하나만 지어 줘. 이름만 답해.
답: '별빛 카페'
  1번째 토큰 '별': '별' 76.5%, '햇' 9.5%, '카페' 4.0%, '따' 2.7%, '달' 1.9%
  2번째 토큰 '빛': '빛' 94.5%, ' 헤' 1.2%, '하' 1.0%, '반' 0.7%, '담' 0.5%
  3번째 토큰 ' 카페': ' 카페' 89.6%, ' 정' 4.5%, ' ' 2.9%, '카페' 1.7%, '당' 0.4%
```

결과에서 세 가지를 읽을 수 있습니다. 첫째, 수도 질문은 후보가 사실상 "서울"과 "서울특별시" 둘뿐이고 어느 쪽이든 맞는 답입니다. 둘째, 카페 이름처럼 정답이 없는 질문도 첫 토큰의 76.5%가 "별"에 몰려 있습니다. 셋째, 첫 토큰이 정해지면 그다음 토큰은 훨씬 확신 있게 이어집니다("빛" 94.5%).

이 모델에서 "서울"과 "서울특별시"는 각각 토큰 하나였지만 "별빛"은 "별"과 "빛" 두 토큰으로 나뉘었습니다. 토큰을 나누는 방식은 모델마다 다르므로, 다른 모델에서는 같은 답도 다르게 쪼개집니다.

## 4. temperature: 확률표를 뾰족하게, 평평하게

`temperature`는 확률표의 모양을 바꿉니다. 1이면 모델이 계산한 확률 그대로 뽑고, 1보다 작으면 원래 높던 후보가 더 높아지며, 1보다 크면 후보 사이의 차이가 줄어듭니다. 0은 뽑기를 하지 않고 항상 확률이 가장 높은 토큰을 고르는 설정입니다.

같은 카페 이름 질문을 `temperature`별로 10번씩 던져 서로 다른 답이 몇 가지 나오는지 셉니다. `02_temperature.py`입니다.

```python
"""같은 질문을 temperature 별로 10번씩 던져 답이 몇 가지 나오는지 센다."""
from collections import Counter
from common import chat

PROMPT = "새로 여는 동네 카페 이름을 하나만 지어 줘. 이름만 답해."
N = 10

for temp in [0, 0.5, 1.0, 1.5, 2.0]:
    answers = Counter(
        chat(PROMPT, {"temperature": temp, "num_predict": 20})["message"]["content"].strip()
        for _ in range(N)
    )
    print(f"\ntemperature={temp}: 서로 다른 답 {len(answers)}가지")
    for text, count in answers.most_common():
        print(f"  {count}회  {text!r}")
```

제 환경에서 나온 결과를 표로 정리하면 다음과 같습니다.

| temperature | 서로 다른 답 | "별빛 카페" 횟수 | 그 밖에 나온 답 예 |
|---|---|---|---|
| 0 | 1가지 | 10회 | 없음 |
| 0.5 | 1가지 | 10회 | 없음 |
| 1.0 | 5가지 | 6회 | 햇살Cafe, 따스한 햇살, 달콤한 뜰 |
| 1.5 | 8가지 | 3회 | 별빛 뜰, 달콤한 볕, 카페 르네상스 |
| 2.0 | 5가지 | 6회 | 카페골, 햇살 카페, 달콤한 정원 |

0과 0.5에서는 열 번 모두 같은 답이 나왔고, 1.0부터 다른 이름이 섞이기 시작했습니다. 다만 2.0이 1.5보다 덜 다양하게 나온 점이 눈에 띕니다. 10번은 경향을 말하기에 너무 적은 횟수이고, 뒤에서 볼 `top_k`와 `top_p`가 함께 걸려 있기 때문에 `temperature`만의 효과라고 보기도 어렵습니다.

### 첫 토큰만 200번 뽑아 보기

그래서 조건을 더 엄격하게 잡았습니다. `top_k`를 0, `top_p`를 1.0으로 두어 후보를 자르지 않게 하고, 첫 토큰 하나만 200번씩 뽑아 비율을 셉니다. `05_first_token_frequency.py`입니다.

```python
"""첫 토큰만 200번씩 뽑아, logprobs 가 말한 확률과 실제로 뽑힌 비율을 비교한다."""
import math
from collections import Counter
from common import chat

PROMPT = "새로 여는 동네 카페 이름을 하나만 지어 줘. 이름만 답해."
N = 200

r = chat(PROMPT, {"temperature": 0, "num_predict": 1}, top_logprobs=5)
print("logprobs 가 알려 준 확률:")
for c in r["logprobs"][0]["top_logprobs"]:
    print(f"  {c['token']!r} {math.exp(c['logprob']) * 100:.1f}%")

for temp in [0.5, 1.0, 2.0]:
    # top_k, top_p 로 후보를 자르지 않게 해서 temperature 의 영향만 본다.
    options = {"temperature": temp, "top_k": 0, "top_p": 1.0, "num_predict": 1}
    reported = set()
    tokens = Counter()
    for _ in range(N):
        res = chat(PROMPT, options, top_logprobs=5)
        pos = res["logprobs"][0]
        tokens[pos["token"]] += 1
        reported.add(tuple(round(math.exp(c["logprob"]) * 100, 1) for c in pos["top_logprobs"]))
    top = ", ".join(f"{t!r} {n / N * 100:.1f}%" for t, n in tokens.most_common(5))
    print(f"\ntemperature={temp}: 첫 토큰 {len(tokens)}종류 | {top}")
    print(f"  응답에 담긴 후보 확률 조합 {len(reported)}가지: {sorted(reported)[:2]}")
```

```text
logprobs 가 알려 준 확률:
  '별' 76.4%
  '햇' 9.6%
  '카페' 4.0%
  '따' 2.7%
  '달' 1.9%

temperature=0.5: 첫 토큰 3종류 | '별' 98.5%, '햇' 1.0%, '달' 0.5%
  응답에 담긴 후보 확률 조합 1가지: [(76.4, 9.6, 4.0, 2.7, 1.9)]

temperature=1.0: 첫 토큰 12종류 | '별' 69.5%, '햇' 10.0%, '카페' 6.0%, '따' 4.5%, '달' 2.5%
  응답에 담긴 후보 확률 조합 1가지: [(76.4, 9.6, 4.0, 2.7, 1.9)]

temperature=2.0: 첫 토큰 81종류 | '별' 24.0%, '햇' 6.5%, '카페' 6.0%, '따' 6.0%, '향' 5.0%
  응답에 담긴 후보 확률 조합 1가지: [(76.4, 9.6, 4.0, 2.7, 1.9)]
```

이번에는 경향이 뚜렷합니다. `temperature`가 0.5일 때 "별"은 200번 중 98.5%를 차지했고, 1.0에서는 69.5%로 `logprobs`가 알려 준 76.4%와 비슷한 수준이었으며, 2.0에서는 24.0%까지 내려가고 첫 토큰 종류가 81가지로 늘었습니다.

0.5의 결과는 계산으로도 맞춰 볼 수 있습니다. `temperature`를 T라고 하면 각 후보의 확률을 1/T 제곱한 뒤 합이 1이 되도록 다시 나누는 것과 같습니다. T가 0.5이면 제곱이므로 "별"은 0.764² ≈ 0.584, "햇"은 0.096² ≈ 0.009가 되고, 위 다섯 후보만으로 다시 나누면 "별"이 약 98%입니다. 실제로 뽑힌 98.5%와 거의 같습니다.

### logprobs 값은 temperature를 바꿔도 그대로였다

출력의 마지막 줄도 중요합니다. `temperature`를 0.5, 1.0, 2.0으로 바꿔도 응답에 담긴 후보 확률은 (76.4, 9.6, 4.0, 2.7, 1.9)로 한 가지뿐이었습니다. 즉 제 환경에서 Ollama가 돌려준 `logprobs`는 `temperature`를 적용하기 전의 확률이었습니다.

이 동작은 공식 문서에 적혀 있지 않아 제가 실행해서 본 것만 적습니다. `logprobs`를 "실제로 그 토큰이 뽑힐 확률"로 읽으면 `temperature`가 1이 아닐 때 틀리게 됩니다. 다른 버전이나 다른 서비스에서는 직접 확인해 보시기 바랍니다.

## 5. top_p와 top_k: 후보를 잘라 내기

`temperature`가 확률표의 모양을 바꾼다면, `top_p`와 `top_k`는 후보 자체를 줄입니다. 앞에서 본 첫 토큰의 확률로 예를 들어 보겠습니다. "별"이 혼자 76.4%이므로 `top_p`가 0.5이면 "별" 하나만으로 이미 0.5를 넘습니다. 0.95라면 확률을 높은 순서로 더해 0.95에 닿을 때까지의 후보가 남고, 그 아래의 꼬리는 버려집니다.

`temperature`는 1.0으로 고정하고 `top_p`와 `top_k`만 바꿔 10번씩 실행합니다. `03_top_p_top_k.py`입니다.

```python
"""temperature 는 1 로 두고 top_p, top_k 만 바꿔 답이 몇 가지 나오는지 센다."""
from collections import Counter
from common import chat

PROMPT = "새로 여는 동네 카페 이름을 하나만 지어 줘. 이름만 답해."
N = 10
CASES = [
    ("top_p=0.1", {"top_p": 0.1}),
    ("top_p=0.5", {"top_p": 0.5}),
    ("top_p=0.95", {"top_p": 0.95}),
    ("top_p=1.0", {"top_p": 1.0}),
    ("top_k=1", {"top_k": 1}),
]

for name, extra in CASES:
    options = {"temperature": 1.0, "num_predict": 20, **extra}
    answers = Counter(
        chat(PROMPT, options)["message"]["content"].strip() for _ in range(N)
    )
    print(f"\n{name}: 서로 다른 답 {len(answers)}가지")
    for text, count in answers.most_common():
        print(f"  {count}회  {text!r}")
```

| 설정 (temperature 1.0) | 서로 다른 답 | "별빛 카페" 횟수 |
|---|---|---|
| `top_p` 0.1 | 1가지 | 10회 |
| `top_p` 0.5 | 1가지 | 10회 |
| `top_p` 0.95 | 3가지 | 8회 |
| `top_p` 1.0 | 8가지 | 3회 |
| `top_k` 1 | 1가지 | 10회 |

`top_p`가 0.5 이하일 때는 `temperature`가 1.0인데도 열 번 모두 같은 답이 나왔습니다. 토큰마다 1등 후보의 확률이 0.5를 넘어서 후보가 하나만 남았기 때문으로 보입니다. `top_p`를 1.0으로 풀자 "카페  bloom", "별빛당"처럼 확률이 낮던 답까지 나왔습니다. `top_k`를 1로 두면 1등 후보만 남으므로 `temperature` 0과 같은 결과가 됩니다.

여기서 4장의 의문도 풀립니다. `ollama show gemma3:1b`를 실행해 보면 이 모델에는 `temperature 1`, `top_k 64`, `top_p 0.95`가 기본값으로 들어 있습니다. 4장의 10회 실험은 `temperature`만 바꿨기 때문에 `top_k` 64와 `top_p` 0.95가 계속 걸려 있었고, 그래서 `temperature`를 2.0까지 올려도 답이 퍼지는 폭이 제한된 것으로 보입니다.

```bash
ollama show gemma3:1b
```

```text
  Parameters
    stop           "<end_of_turn>"
    temperature    1
    top_k          64
    top_p          0.95
```

기본값은 한 가지가 아니라는 점도 기억해 두세요. Ollama Modelfile 문서에 적힌 기본값은 `temperature` 0.8, `top_k` 40, `top_p` 0.9이지만, 위에서 보듯 모델이 자기 값을 따로 가지고 있으면 그 값이 쓰입니다. 실험할 때는 기본값에 기대지 말고 바꾸려는 옵션을 모두 직접 적는 편이 안전합니다.

## 6. seed: 뽑기를 다시 재현하기

샘플링은 난수로 뽑기를 하므로 같은 설정이어도 답이 달라집니다. `seed`는 그 난수의 시작값을 고정합니다. Ollama Modelfile 문서는 `seed`를 특정 숫자로 정하면 같은 프롬프트에 같은 글을 생성한다고 설명합니다. `temperature`를 1.5로 높게 둔 채 확인해 봅니다. `04_seed.py`입니다.

```python
"""seed 를 고정하면 temperature 가 높아도 같은 답이 나오는지 본다."""
from collections import Counter
from common import chat

PROMPT = "새로 여는 동네 카페 이름을 하나만 지어 줘. 이름만 답해."
N = 5
CASES = [
    ("seed 없음", {}),
    ("seed=42", {"seed": 42}),
    ("seed=7", {"seed": 7}),
    ("seed=42 다시", {"seed": 42}),
]

for name, extra in CASES:
    options = {"temperature": 1.5, "num_predict": 20, **extra}
    answers = Counter(
        chat(PROMPT, options)["message"]["content"].strip() for _ in range(N)
    )
    print(f"\n{name}: 서로 다른 답 {len(answers)}가지")
    for text, count in answers.most_common():
        print(f"  {count}회  {text!r}")
```

```text
seed 없음: 서로 다른 답 3가지
  2회  '달콤한 오후'
  2회  '별빛 카페'
  1회  '달콤한 정원'

seed=42: 서로 다른 답 1가지
  5회  '카페 뜰'

seed=7: 서로 다른 답 1가지
  5회  '별빛 쉼표'

seed=42 다시: 서로 다른 답 1가지
  5회  '카페 뜰'
```

`seed`를 주지 않으면 다섯 번에 세 가지 답이 나왔고, 42로 고정하면 다섯 번 모두 "카페 뜰"이었습니다. `seed`를 7로 바꾸면 다른 답("별빛 쉼표")으로 고정되고, 다시 42로 돌아오면 "카페 뜰"이 그대로 나옵니다. `temperature` 0과 다른 점은, 1등 후보만 고르는 것이 아니라 **다양하게 뽑힌 결과 하나를 다시 불러낸다**는 것입니다.

다만 제가 확인한 것은 같은 컴퓨터, 같은 Ollama 버전, 같은 모델에서의 재현입니다. 환경이 달라져도 같은 답이 나온다고 볼 근거는 없습니다. 참고로 OpenAI의 API 문서도 `seed`에 대해 결정적으로 뽑도록 최선을 다하지만 결정성은 보장되지 않는다고 적고 있습니다.

## 7. 어떤 값을 쓰면 될까

실험 결과를 바탕으로 정리하면 이렇습니다. 정해진 답은 없고, 일의 성격에 따라 출발점을 잡은 뒤 직접 돌려 보며 조정하는 것이 좋습니다.

| 하려는 일 | 출발점 |
|---|---|
| 분류, 정보 추출, 형식 변환처럼 답이 일정해야 하는 일 | `temperature` 0 |
| 이름 짓기, 문구 초안처럼 여러 안이 필요한 일 | `temperature`를 올리고 여러 번 호출 |
| 다양한 답 중 하나를 테스트에서 다시 재현 | `seed` 고정 |
| 엉뚱한 토큰이 섞이는 것을 줄이기 | `top_p`를 1보다 낮게 |

`temperature`와 `top_p`를 동시에 크게 바꾸면 어느 쪽 때문에 결과가 달라졌는지 알기 어렵습니다. OpenAI의 API 문서도 둘 중 하나만 바꾸기를 권합니다. 한 번에 한 옵션만 바꾸고, 5장에서 본 것처럼 나머지 옵션이 기본값으로 걸려 있다는 점을 함께 확인하세요.

## 8. 한계와 주의할 점

### 확률이 높다고 사실은 아니다

`logprobs`의 확률은 "이 토큰이 다음에 올 법한 정도"이지 "이 내용이 사실일 정도"가 아닙니다. 1편에서 `gemma3:1b`가 틀렸던 질문을 다시 던져 확률을 봤습니다(`06_confident_but_wrong.py`, 구조는 `01_logprobs.py`와 같습니다).

```text
질문: 2024년 노벨 문학상 수상자는 누구야? 이름만 답해.
답: '로알드 프랭스'
  1번째 토큰 '로': '로' 49.0%, '오' 32.6%, '벨' 2.4%
  2번째 토큰 '알': '알' 73.4%, '버' 10.5%, '랑' 8.9%
  3번째 토큰 '드': '드' 99.3%, '্ড' 0.3%, '디' 0.2%
  4번째 토큰 ' 프': ' 프' 98.8%, ' 푸' 1.0%, ' 포' 0.1%
  5번째 토큰 '랭': '랭' 44.0%, '루' 25.8%, '라이' 19.9%
  6번째 토큰 '스': '스' 43.0%, '크' 29.0%, '클' 12.9%
```

노벨상 공식 사이트에 따르면 2024년 수상자는 한강 작가입니다. 모델의 답은 틀렸는데도 3번째, 4번째 토큰의 확률은 99% 안팎입니다. 일단 이름을 시작하면 그럴듯하게 이어 가는 데에는 확신이 높다는 뜻일 뿐입니다. 첫 토큰이 49.0%와 32.6%로 갈린 것은 모델이 흔들린다는 신호로 볼 수도 있지만, 이것만으로 틀린 답을 가려낼 수는 없습니다. 사실 확인이 필요한 일에는 [RAG처럼 근거 문서를 함께 주는 방법](/2026/11/24/ollama-rag-saju-terms/)과 사람의 검증이 필요합니다.

### temperature 0도 "정답 모드"가 아니다

`temperature` 0은 가장 확률이 높은 토큰을 고를 뿐입니다. 위 노벨상 질문도 `temperature` 0으로 실행한 결과입니다. 답이 일정해지는 것과 답이 맞는 것은 다른 문제입니다.

### 이 실험의 범위

- 모델 하나(`gemma3:1b`), 프롬프트 하나, 10회 또는 200회 실행한 결과입니다. 횟수가 적어 4장의 2.0처럼 경향과 어긋나는 값이 섞여 있습니다.
- `logprobs`가 `temperature` 적용 전 값이라는 점, `seed`로 재현된다는 점은 Ollama 0.40.16에서 본 동작입니다.
- `top_logprobs`의 허용 범위는 Ollama 문서에 적혀 있지 않습니다. 제 환경에서는 20까지 받았고 21을 넣자 `top_logprobs must be between 0 and 20` 오류가 났습니다.

### 문서와 실제 동작이 다른 부분

Ollama의 OpenAI 호환 문서는 `/v1/chat/completions`의 지원 기능 목록에서 Logprobs를 지원하지 않는 항목으로 표시하고 있습니다. 반면 Ollama v0.12.11 릴리스 노트는 Ollama API와 OpenAI 호환 API가 로그 확률을 지원한다고 안내합니다. 제 환경(0.40.16)에서는 `/v1/chat/completions`에 `logprobs`와 `top_logprobs`를 넣자 `choices[0].logprobs.content`에 후보 확률이 담겨 왔습니다. 버전에 따라 다를 수 있으니 쓰기 전에 직접 호출해 확인하세요.

### 서비스마다 옵션 지원이 다르다

이 글의 옵션 이름과 동작은 Ollama 기준입니다. 서비스에 따라 범위와 지원 여부가 다릅니다. 예를 들어 Google의 Gemini 프롬프트 설계 문서에는 일부 최신 Gemini 모델이 `temperature`, `top_p`, `top_k`에 직접 지정한 값을 무시한다는 안내가 있습니다. 다른 API로 옮길 때는 그 서비스의 문서를 먼저 확인해야 합니다.

### 라이선스와 사내 규정

`gemma3`는 Gemma Terms of Use를 따릅니다. 회사 업무나 서비스에 쓰기 전에 `ollama show`와 모델 페이지에서 라이선스를 확인하세요. 로컬 모델은 입력이 컴퓨터 밖으로 나가지 않지만, 회사 PC에 설치하고 업무 자료를 넣어도 되는지는 사내 규정을 따라야 합니다.

## 마무리

- LLM은 토큰마다 후보별 확률을 매기고 그중 하나를 뽑습니다. `logprobs`와 `top_logprobs`로 그 확률을 볼 수 있고, `math.exp()`로 퍼센트로 바꿉니다.
- `temperature`는 확률표의 모양을 바꿉니다. 첫 토큰 200회 실험에서 1등 후보의 비율이 0.5일 때 98.5%, 1.0일 때 69.5%, 2.0일 때 24.0%였습니다.
- `top_p`와 `top_k`는 후보를 잘라 냅니다. 모델에 기본값이 들어 있을 수 있으니 `ollama show`로 확인하세요.
- `seed`를 고정하면 같은 환경에서 다양한 답 중 하나를 다시 재현할 수 있습니다.
- 토큰 확률이 높다고 내용이 사실인 것은 아닙니다. 틀린 답도 99% 확률의 토큰으로 이어질 수 있습니다.

다음 글에서는 LLM의 답을 프로그램이 바로 쓸 수 있도록 JSON으로 받는 방법을 다룰 예정입니다.

## 참고 자료

- Ollama, [Generate a chat message (Chat API)](https://docs.ollama.com/api/chat)
- Ollama, [Modelfile Reference](https://docs.ollama.com/modelfile)
- Ollama, [OpenAI compatibility](https://docs.ollama.com/api/openai-compatibility)
- Ollama, [Releases(v0.12.11 Logprobs 안내)](https://github.com/ollama/ollama/releases?q=logprobs)
- Holtzman 외, [The Curious Case of Neural Text Degeneration](https://arxiv.org/abs/1904.09751), 2019-04-22
- OpenAI, [Chat Completions API Reference](https://developers.openai.com/api/reference/resources/chat)
- Google, [Prompt design strategies](https://aistudio.google.com/docs/prompting-strategies)
- Google, [Gemma Terms of Use](https://ai.google.dev/gemma/terms)
- The Nobel Prize, [The Nobel Prize in Literature 2024](https://www.nobelprize.org/prizes/literature/2024/summary/)
