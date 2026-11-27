---
layout: post
title: '레이트 리밋 설계: 고정 윈도·슬라이딩 윈도·토큰 버킷을 Redis로 구현해 비교하기'
author : 효준
date: 2026-11-27 01:00
tags: [backend]
image: /files/covers/openapi.png
---

## "1분에 100번까지"는 생각보다 정하기 어렵다

API를 공개하면 언젠가 한 클라이언트가 요청을 쏟아붓는 날이 옵니다. 버그가 난 배치일 수도 있고, 재시도 루프일 수도 있고, 크롤러일 수도 있습니다. 이때 한 사용자가 쓸 수 있는 양에 상한을 두는 장치가 **레이트 리밋(rate limit)**입니다.

"2초에 100건까지"라는 규칙은 단순해 보입니다. 그런데 그 2초를 **어디서부터 세느냐**에 따라 실제로 통과하는 요청 수가 두 배까지 달라지고, 카운터를 **어떻게 올리느냐**에 따라 한도가 아예 지켜지지 않기도 합니다. 이 글은 Redis로 레이트 리미터 네 가지를 직접 만들고, 같은 요청을 흘려 보내 무엇이 다른지 비교합니다.

<!--more-->

글의 숫자는 노트북에서 Docker로 띄운 Redis 7.4.12(`redis:7` 이미지)와 Node.js 24, `redis` npm 패키지 4.7로 측정한 예제 결과입니다. 앱 서버 4대는 한 프로세스 안의 Redis 연결 4개로 흉내 냈고, 한도는 모두 **사용자당 2초에 100건(초당 50건)**으로 맞췄습니다. 실제 서비스의 수치가 아니며, 환경에 따라 달라질 수 있습니다.

## 1. 레이트 리밋에서 틀리기 쉬운 두 가지

HTTP에서 한도를 넘은 요청에는 보통 `429 Too Many Requests`로 답합니다. 이 상태 코드를 정의한 RFC 6585는 429를 사용자가 정해진 시간 동안 너무 많은 요청을 보냈다는 뜻이라고 설명합니다. 다만 같은 문서는 서버가 사용자를 어떻게 식별하고 요청을 어떻게 세는지는 **정의하지 않는다**고 분명히 적어 두었습니다. 세는 방법은 전적으로 구현하는 쪽의 몫입니다.

직접 구현할 때 문제가 되는 지점은 크게 둘입니다.

- **동시성:** 서버 여러 대가 같은 카운터를 읽고 올리는 사이에 다른 요청이 끼어듭니다.
- **윈도 경계:** "2초"를 시계 기준으로 딱딱 끊으면, 경계 양쪽에 요청을 몰아 보내는 클라이언트는 한도의 두 배를 통과시킬 수 있습니다.

둘을 차례로 재현해 보겠습니다.

## 2. 재현 1: 읽고 나서 올리면 한도가 사라진다

가장 먼저 떠오르는 코드는 이렇습니다. 현재 값을 읽고, 한도보다 작으면 올립니다.

```js
async function naive(redis, user, { limit, windowMs }) {
  const key = `rl:naive:${user}:${Math.floor(Date.now() / windowMs)}`;
  const n = Number(await redis.get(key)) || 0;   // 1) 읽고
  if (n >= limit) return false;
  await redis.incr(key);                          // 2) 올린다
  await redis.pExpire(key, windowMs * 2);
  return true;
}
```

한도가 100건인 사용자 키 하나에 요청 500건을 동시에 보냈습니다.

| 구현 | 보낸 요청 | 통과한 요청 |
|---|---|---|
| 읽고 나서 올리기(GET 후 INCR) | 500건 | 500건 |
| INCR와 만료 설정을 Lua 스크립트 하나로 | 500건 | 100건 |

세 번 반복해도 결과는 같았습니다. 읽고 나서 올리는 구현은 **500건을 전부 통과**시켰습니다. 요청들이 값을 읽는 시점에는 아직 아무도 카운터를 올리지 않았기 때문에, 모두가 "0건이니 통과"라고 판단한 것입니다. 한도가 필요한 순간은 요청이 몰릴 때인데, 바로 그때 한도가 작동하지 않습니다.

Redis의 `INCR` 명령 문서에도 레이트 리미터 패턴이 실려 있습니다. 문서는 `GET`으로 확인한 뒤 `INCR`와 `EXPIRE`를 따로 부르는 방식에 경쟁 상태(race condition)가 있다고 경고합니다. 클라이언트가 `INCR`만 하고 `EXPIRE`를 못 하면 키가 만료 없이 남는다는 것입니다. 문서가 제시하는 해결책이 Lua 스크립트입니다.

### 왜 Lua 스크립트인가

Redis 문서는 스크립트 실행이 원자적이라고 보장합니다. 스크립트가 도는 동안에는 서버의 다른 활동이 모두 멈추기 때문에, 읽기와 쓰기 사이에 다른 요청이 끼어들 수 없습니다. 아래에서 만드는 리미터는 모두 "읽고 → 판단하고 → 쓰는" 과정을 스크립트 하나에 넣습니다.

같은 이유로 스크립트는 짧아야 합니다. 스크립트가 오래 걸리면 그동안 Redis 전체가 기다립니다.

## 3. 방법 1: 고정 윈도 카운터

시간을 2초 단위로 자르고, 구간마다 카운터 키를 하나씩 둡니다. 가장 단순하고 메모리도 가장 적게 씁니다.

```js
const FIXED = `
local n = redis.call('INCR', KEYS[1])
if n == 1 then redis.call('PEXPIRE', KEYS[1], ARGV[2]) end
if n > tonumber(ARGV[1]) then return 0 end
return 1`;

async function fixedWindow(redis, user, { limit, windowMs }) {
  const key = `rl:fixed:${user}:${Math.floor(Date.now() / windowMs)}`;
  return (await redis.eval(FIXED, {
    keys: [key], arguments: [String(limit), String(windowMs * 2)],
  })) === 1;
}
```

`INCR`는 그 자체로 원자적이라, 올린 **뒤의** 값으로 판단하면 경쟁이 생기지 않습니다. 위 표의 두 번째 줄이 이 구현의 결과입니다.

문제는 경계입니다. 윈도가 끝나기 직전에 100건, 새 윈도가 시작한 직후에 100건을 보내면 두 묶음은 서로 다른 키에 기록됩니다. 다음 절에서 실제로 보내 봅니다.

## 4. 재현 2: 윈도 경계에 몰아 보내기

윈도 경계 **200ms 전에 100건**, 경계 **50ms 후에 100건**을 보냈습니다. 두 묶음의 간격은 약 250ms입니다. 뒤에서 설명할 다른 방식들도 같은 조건으로 함께 측정했습니다.

| 방식 | 경계 전 100건 중 통과 | 경계 후 100건 중 통과 | 약 250ms 동안 통과 합계 |
|---|---|---|---|
| 고정 윈도 | 100 | 100 | 200 |
| 슬라이딩 윈도 로그 | 100 | 0 | 100 |
| 슬라이딩 윈도 카운터 | 100 | 2 | 102 |
| 토큰 버킷(용량 100, 초당 50개) | 100 | 12 | 112 |
| 토큰 버킷(용량 10, 초당 50개) | 10 | 10 | 20 |

세 번 모두 같은 값이 나왔습니다. 고정 윈도는 "2초에 100건"이라는 규칙을 두고도 **0.25초 사이에 200건**을 통과시켰습니다. Redis의 레이트 리미터 튜토리얼도 고정 윈도의 단점을 경계에서 2배 버스트를 허용하는 것이라고 정리합니다.

이게 문제인지는 무엇을 지키려는지에 달렸습니다. 요금제의 일일 사용량처럼 긴 구간의 총량을 세는 용도라면 경계의 버스트는 큰 문제가 아닙니다. 반대로 뒤쪽 DB나 외부 API가 순간 부하에 약해서 리밋을 거는 것이라면, 한도의 두 배가 한꺼번에 들어오는 상황은 리밋을 둔 목적 자체를 무너뜨립니다.

## 5. 방법 2: 슬라이딩 윈도 로그

경계 문제를 정확히 없애려면 "지금으로부터 과거 2초"를 매번 다시 세면 됩니다. 허용한 요청의 시각을 정렬 집합(sorted set)에 모두 기록해 두고, 요청이 올 때마다 2초보다 오래된 기록을 지운 뒤 남은 개수를 셉니다.

```js
const LOG = `
local limit = tonumber(ARGV[1])
local windowMs = tonumber(ARGV[2])
local t = redis.call('TIME')
local now = t[1] * 1000 + math.floor(t[2] / 1000)
redis.call('ZREMRANGEBYSCORE', KEYS[1], 0, now - windowMs)
if redis.call('ZCARD', KEYS[1]) >= limit then return 0 end
redis.call('ZADD', KEYS[1], now, ARGV[3])
redis.call('PEXPIRE', KEYS[1], windowMs)
return 1`;
```

`ARGV[3]`에는 요청마다 다른 값(예제에서는 UUID)을 넣습니다. 정렬 집합의 멤버가 같으면 덮어써져 한 건으로 세어지기 때문입니다.

시각은 앱 서버가 아니라 Redis의 `TIME` 명령으로 얻었습니다. 앱 서버마다 시계가 조금씩 다르면 판단도 흔들리는데, Redis 한 곳의 시계를 쓰면 그 문제가 없습니다. Redis 문서에 따르면 스크립트 안에서 `TIME` 같은 비결정적 명령을 자유롭게 쓸 수 있는 것은 효과 복제(effects replication) 덕분이고, 이 방식은 Redis 5.0부터 기본값입니다. 그보다 오래된 버전을 쓴다면 시각을 인자로 넘겨야 합니다.

경계 실험에서 이 방식은 정확히 100건만 통과시켰습니다. 대가는 메모리입니다. 허용한 요청 하나하나를 저장하므로 한도가 클수록 키가 커집니다. 이 차이는 8절에서 숫자로 확인합니다.

## 6. 방법 3: 슬라이딩 윈도 카운터

로그를 다 저장하지 않고 **카운터 두 개**로 슬라이딩 윈도를 근사하는 방법입니다. Cloudflare가 자사 레이트 리미팅을 설명한 글에서 소개한 방식으로, 지난 윈도의 카운터에 "지금 윈도와 겹치는 비율"을 곱해 현재 윈도의 카운터와 더합니다.

그 글의 예시는 이렇습니다. 한도가 분당 50건이고, 지난 1분 동안 42건, 15초 전에 시작한 이번 1분 동안 18건을 받았다면 다음처럼 계산합니다.

```text
rate = 42 × ((60 − 15) / 60) + 18 = 42 × 0.75 + 18 = 49.5건
```

구현은 다음과 같습니다.

```js
const COUNTER = `
local limit = tonumber(ARGV[1])
local windowMs = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local cur = tonumber(redis.call('GET', KEYS[1]) or '0')
local prev = tonumber(redis.call('GET', KEYS[2]) or '0')
local elapsed = (now % windowMs) / windowMs
local estimate = prev * (1 - elapsed) + cur
if estimate + 1 > limit then return 0 end
redis.call('INCR', KEYS[1])
redis.call('PEXPIRE', KEYS[1], windowMs * 2)
return 1`;

async function slidingCounter(redis, user, { limit, windowMs }) {
  const now = Date.now();
  const w = Math.floor(now / windowMs);
  return (await redis.eval(COUNTER, {
    keys: [`rl:cnt:${user}:${w}`, `rl:cnt:${user}:${w - 1}`],
    arguments: [String(limit), String(windowMs), String(now)],
  })) === 1;
}
```

경계 실험에서는 경계 후 100건 중 2건만 통과했습니다. 경계에서 50ms 지난 시점의 가중치는 `1 − 50/2000 = 0.975`이므로 지난 윈도의 100건이 97.5건으로 계산되고, 2건이 더 들어갈 자리가 생긴 것입니다.

이 방식은 지난 윈도의 요청이 **고르게 분포했다고 가정**합니다. Cloudflare 글도 이 점을 밝히고, 그래서 결과가 근삿값이라고 설명합니다. 같은 글은 27만 개 출처에서 온 요청 4억 건을 분석한 결과, 잘못 허용하거나 잘못 차단한 요청이 0.003%였고 실제 비율과 근사 비율의 차이는 평균 6%였다고 보고합니다. 정확도를 조금 내주고 메모리를 크게 아끼는 선택입니다.

## 7. 방법 4: 토큰 버킷

토큰 버킷은 관점이 다릅니다. 사용자마다 토큰이 담긴 양동이가 있고, 요청 하나가 토큰 하나를 꺼내 갑니다. 토큰은 일정한 속도로 다시 채워지고, 양동이가 비어 있으면 요청을 거절합니다. Stripe는 자사 API의 레이트 리미터를 이 알고리즘과 Redis로 구현했다고 밝혔고, AWS API Gateway 문서도 토큰 버킷으로 요청을 조절한다고 설명합니다.

설정값은 두 개입니다. API Gateway 문서의 표현을 빌리면 **rate**는 토큰이 채워지는 초당 속도이고, **burst**는 양동이의 용량입니다.

토큰을 채우려고 타이머를 돌릴 필요는 없습니다. 남은 토큰 수와 마지막 계산 시각만 저장해 두고, 요청이 왔을 때 지난 시간만큼 한꺼번에 채우면 됩니다.

```js
const BUCKET = `
local capacity = tonumber(ARGV[1])
local ratePerSec = tonumber(ARGV[2])
local t = redis.call('TIME')
local now = t[1] * 1000 + math.floor(t[2] / 1000)
local d = redis.call('HMGET', KEYS[1], 'tokens', 'ts')
local tokens = tonumber(d[1]) or capacity
local ts = tonumber(d[2]) or now
tokens = math.min(capacity, tokens + (now - ts) * ratePerSec / 1000)
local allowed = 0
local retryMs = 0
if tokens >= 1 then
  tokens = tokens - 1
  allowed = 1
else
  retryMs = math.ceil((1 - tokens) / ratePerSec * 1000)
end
redis.call('HSET', KEYS[1], 'tokens', tostring(tokens), 'ts', now)
redis.call('PEXPIRE', KEYS[1], math.ceil(capacity / ratePerSec * 1000))
return { allowed, retryMs }`;
```

키의 만료 시간은 빈 양동이가 가득 차는 데 걸리는 시간으로 잡았습니다. 그만큼 요청이 없었다면 어차피 가득 찬 상태이므로, 키가 사라져도 결과가 같습니다.

거절할 때는 토큰 하나가 채워질 때까지 남은 시간(`retryMs`)을 함께 돌려줍니다. RFC 6585는 429 응답에 얼마나 기다려야 하는지 알려 주는 `Retry-After` 헤더를 넣을 수 있다고 하고, 예시로 `Retry-After: 3600`을 듭니다. 예제에서 용량 10, 초당 2개인 버킷을 비운 직후 요청하니 `retryMs`가 499로 나왔습니다. 이 값을 초 단위로 올림해 헤더에 넣으면 됩니다.

### 용량이 곧 버스트 크기다

경계 실험 표를 다시 보면, 용량 100인 버킷은 첫 묶음 100건을 모두 통과시킨 뒤 250ms 후에 12건을 더 통과시켰습니다. 그 사이 `0.25초 × 50개 = 12.5개`가 채워졌기 때문입니다. 용량을 10으로 줄이면 첫 묶음에서 10건만 통과합니다.

토큰 버킷이 보장하는 것은 "2초에 100건"이 아니라 **"어느 구간에서든 용량 + 속도 × 시간 이하"**입니다. 용량을 얼마로 잡느냐가 순간 버스트를 얼마나 봐줄지를 정합니다. AWS Builders' Library의 멀티테넌트 공정성 글은 버스트 용량이 너무 크면 레이트 리밋의 보호 효과가 사라진다고 경고합니다.

## 8. 계속 들어오는 과부하와 비용 비교

### 초당 200건을 10초 동안

한도(초당 50건)의 4배인 초당 200건을 10초 동안, 모두 2,000건 보냈습니다. 초당 50건씩 10초면 500건이 기준입니다. 세 번 돌린 범위입니다.

| 방식 | 통과한 요청 | 어느 2초 구간에서든 최대 통과 |
|---|---|---|
| 고정 윈도 | 502~526건 | 103~127건 |
| 슬라이딩 윈도 로그 | 500건 | 100~101건 |
| 슬라이딩 윈도 카운터 | 497~511건 | 100~119건 |
| 토큰 버킷(용량 100) | 599건 | 199건 |
| 토큰 버킷(용량 10) | 509건 | 109건 |

기대와 다른 결과가 둘 있었습니다.

첫째, 슬라이딩 윈도 카운터가 한 번은 2초 구간에 **119건**을 통과시켰습니다. 실험이 윈도 중간에 시작해 첫 윈도의 100건이 끝부분에 몰렸는데, 알고리즘은 그 100건이 윈도 전체에 고르게 퍼져 있었다고 가정하기 때문입니다. 요청이 몰려서 들어오는 패턴에서는 근사 오차가 평균보다 커질 수 있습니다. 반대로 497건처럼 기준보다 덜 통과시킨 경우도 있었습니다.

둘째, 용량 100인 토큰 버킷은 2초 구간에 199건을 통과시켰습니다. 버그가 아니라 정의대로입니다. 처음에 가득 찬 100개를 쓰고, 2초 동안 채워진 100개를 또 썼습니다. "2초에 100건"과 "용량 100, 초당 50개"는 **서로 다른 정책**입니다. 윈도 방식과 비슷한 총량을 원하면 용량을 작게 잡아야 하고, 용량 10일 때는 509건으로 기준에 가까웠습니다.

슬라이딩 윈도 로그의 101건은 측정 방식 때문입니다. 통과 시각은 앱에서 보낸 시점으로 기록했고 리미터는 Redis 서버 시각으로 판단했기 때문에, 구간 끝에서 1건이 어긋날 수 있습니다.

### 키 하나의 메모리와 호출 지연

한도를 꽉 채운 상태에서 `MEMORY USAGE` 명령으로 사용자 한 명의 키 크기를 쟀습니다.

| 방식 | 한도 100건일 때 | 한도 10,000건일 때 |
|---|---|---|
| 고정 윈도 | 80바이트 | 88바이트 |
| 슬라이딩 윈도 로그 | 5,192바이트 | 약 1.24~1.27MB |
| 슬라이딩 윈도 카운터 | 80바이트 × 키 2개 | 80바이트 × 키 2개 |
| 토큰 버킷 | 120바이트 | 120바이트 |

슬라이딩 윈도 로그만 한도에 비례해 커집니다. 한도 10,000건이면 사용자 한 명이 1MB를 넘게 쓰므로, 사용자가 10만 명이면 계산이 맞지 않습니다. 로그의 멤버로 UUID를 썼기 때문에 더 짧은 값을 쓰면 줄어들지만, 한도에 비례한다는 성질은 그대로입니다.

호출 한 번의 지연은 순차 호출 3,000회 기준으로 네 방식 모두 p50 약 0.12ms, p99 0.18~0.32ms였습니다. 같은 노트북 안의 Redis라 네트워크 지연이 거의 없는 조건이고, 이 정도 규모에서는 방식에 따른 차이가 보이지 않았습니다.

## 9. 주의할 점과 한계

- **무엇을 기준으로 셀지 먼저 정합니다.** RFC 6585가 말하듯 식별 방법은 정해져 있지 않습니다. 사용자 ID, API 키, IP 중 무엇을 키로 쓰느냐에 따라 공용 IP 뒤의 사용자들이 한꺼번에 막히거나, 한 사용자가 키를 여러 개 만들어 한도를 피할 수 있습니다.
- **시계를 한 곳으로 맞춥니다.** 예제의 고정 윈도와 슬라이딩 윈도 카운터는 키 이름을 만들려고 앱 서버의 시계를 썼습니다. 서버 시계가 서로 어긋나면 같은 순간의 요청이 다른 윈도로 들어갑니다. Redis 문서는 스크립트가 인자로 받은 키만 접근하고, 스크립트 안에서 키 이름을 만들어 내지 말라고 안내하므로, 키 이름은 앱에서 만들되 서버 시계를 동기화해 두어야 합니다.
- **리미터가 죽었을 때의 동작을 정합니다.** Stripe 글은 리미터 코드에서 예외가 나면 요청을 그대로 통과시키도록(fail open) 만들어, Redis가 내려가도 API는 계속 동작하게 했다고 설명합니다. 반대로 한도가 곧 비용이나 보안인 API라면 막는 쪽이 맞을 수 있습니다.
- **바로 차단하지 말고 먼저 지켜봅니다.** 같은 글은 리미터를 실제로 적용하기 전에 차단됐을 트래픽만 기록해 보는 방식(dark launch)과, 문제가 생기면 끌 수 있는 장치를 권합니다.
- **429는 공격을 막는 수단이 아닙니다.** RFC 6585의 보안 고려 사항은 공격을 받는 중에 요청마다 429로 답하는 것 자체가 자원을 쓴다고 지적하고, 연결을 끊는 편이 더 적절할 수 있다고 합니다.
- **응답 헤더는 아직 표준이 아닙니다.** 남은 한도를 알려 주는 `RateLimit`, `RateLimit-Policy` 헤더는 IETF에서 인터넷 초안(draft-ietf-httpapi-ratelimit-headers) 단계입니다. 지금 확실히 쓸 수 있는 표준 신호는 429와 `Retry-After`입니다.
- **예제의 한계:** Redis 한 대, 사용자 키 하나로 실험했습니다. Redis를 여러 대로 나눈 구성이나 복제 지연이 있는 상황은 다루지 않았습니다.

클라이언트 쪽 대응도 짝으로 필요합니다. 429를 받은 클라이언트가 곧바로 다시 시도하면 부하는 줄지 않습니다. 이 부분은 [재시도와 지수 백오프·지터를 다룬 글](/2026/11/25/retry-backoff-jitter-budget/)에서 따로 정리했습니다. 한도 초과와 별개로 캐시 만료 때문에 요청이 몰리는 문제는 [캐시 스탬피드 글](/2026/11/24/cache-stampede-redis/)을 참고하세요.

## 10. 어떤 방법을 고를까

| 상황 | 고를 만한 방식 | 이유 |
|---|---|---|
| 일·월 단위 사용량처럼 긴 구간의 총량 | 고정 윈도 | 가장 단순하고 키 하나로 끝남. 경계 버스트가 문제 되지 않음 |
| 일반적인 API의 사용자별 한도 | 슬라이딩 윈도 카운터 | 카운터 두 개로 경계 버스트를 거의 없앰. 약간의 오차 감수 |
| 한도가 작고 정확해야 함(로그인 시도, 문자 발송 등) | 슬라이딩 윈도 로그 | 정확함. 한도가 작으면 메모리 부담도 작음 |
| 짧은 버스트는 봐주되 평균 속도를 지키고 싶음 | 토큰 버킷 | 용량으로 버스트 크기를, 속도로 평균을 따로 조절 |
| 순간 부하에 약한 뒷단을 보호 | 토큰 버킷(작은 용량) | 어느 구간에서든 상한이 용량 + 속도 × 시간으로 분명함 |

Redis의 튜토리얼은 정답인 알고리즘은 없다고 하면서도, 대부분의 API에는 슬라이딩 윈도 카운터를 기본으로 권합니다. 이번 실험 결과도 그 권고와 어긋나지 않았습니다. 다만 요청이 몰려 들어오는 패턴에서는 오차가 커질 수 있다는 점을 알고 써야 합니다.

## 마무리

- 카운터를 **읽고 나서 올리면** 동시 요청에서 한도가 지켜지지 않습니다. 예제에서는 500건이 모두 통과했습니다. 읽기·판단·쓰기를 Lua 스크립트 하나에 넣습니다.
- **고정 윈도**는 가장 싸지만 경계 양쪽으로 한도의 두 배가 통과합니다. 예제에서는 0.25초 사이에 200건이 지나갔습니다.
- **슬라이딩 윈도 로그**는 정확하지만 메모리가 한도에 비례합니다. **슬라이딩 윈도 카운터**는 카운터 두 개로 그에 가까운 결과를 냅니다.
- **토큰 버킷**의 용량은 곧 허용하는 버스트 크기입니다. "2초에 100건"과 "용량 100, 초당 50개"는 같은 정책이 아닙니다.
- 거절할 때는 **429와 `Retry-After`**를 돌려주고, 리미터 자체가 장애일 때 통과시킬지 막을지를 미리 정해 둡니다.

예제 전체 코드는 위 코드 조각에 측정 코드를 더한 것입니다. Docker로 `redis:7`을 띄우고 `redis` npm 패키지 4.7을 설치하면 같은 실험을 해 볼 수 있습니다.

## 참고 자료

- IETF (M. Nottingham, R. Fielding), [RFC 6585: Additional HTTP Status Codes](https://www.rfc-editor.org/rfc/rfc6585), 2012-04
- Redis, [INCR 명령 문서 (Pattern: rate limiter)](https://redis.io/docs/latest/commands/incr/)
- Redis, [Scripting with Lua](https://redis.io/docs/latest/develop/programmability/eval-intro/)
- Redis, [TIME 명령 문서](https://redis.io/docs/latest/commands/time/)
- Redis (William Johnston), [Build 5 Rate Limiters with Redis: Fixed Window, Sliding Window, Token Bucket, and Leaky Bucket](https://redis.io/learn/howtos/ratelimiting), 2026-02-26
- Cloudflare Blog, [How we built rate limiting capable of scaling to millions of domains](https://blog.cloudflare.com/counting-things-a-lot-of-different-things/), 2017-06-07
- Stripe Blog (Paul Tarjan), [Scaling your API with rate limiters](https://stripe.com/blog/rate-limiters), 2017-03-30
- AWS, [Throttle requests to your REST APIs for better throughput in API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-request-throttling.html)
- AWS Builders' Library, [Fairness in multi-tenant systems](https://aws.amazon.com/builders-library/fairness-in-multi-tenant-systems/), 2020-11-13
- IETF HTTPAPI WG, [RateLimit header fields for HTTP (draft-ietf-httpapi-ratelimit-headers)](https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/), 인터넷 초안
