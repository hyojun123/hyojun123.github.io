---
layout: post
title: '캐시 스탬피드: 캐시가 만료되는 순간 DB가 무너지는 이유와 Redis로 막는 4가지 방법'
author : 효준
date: 2026-11-24 15:00
tags: [backend]
image: /files/covers/traffic.jpg
---

## 캐시를 붙였는데 왜 DB가 죽을까

트래픽이 많은 서비스는 DB 앞에 Redis 같은 캐시를 둡니다. 평소에는 대부분의 요청이 캐시에서 끝나니 DB는 한가합니다. 그런데 **인기 있는 캐시 키 하나가 만료되는 순간**, 그 키를 찾던 요청들이 한꺼번에 캐시 미스를 보고 모두 DB로 몰려갈 수 있습니다.

이 현상을 **캐시 스탬피드(cache stampede)**라고 부릅니다. 독파일링(dog-piling), 캐시 미스 폭풍이라고도 합니다. 이 글은 Node.js와 Redis로 스탬피드를 직접 재현하고, 요청 합치기·Redis 락·확률적 조기 갱신·TTL 지터 네 가지 방법으로 막아 본 결과를 정리합니다.

<!--more-->

글의 숫자는 노트북에서 Docker로 띄운 Redis 7.4와 **200ms 걸리도록 만든 가짜 DB 함수**로 측정한 예제 결과입니다. 앱 서버 4대는 한 프로세스 안에서 Redis 연결 4개로 흉내 냈습니다. 실제 서비스의 수치가 아니며, 환경에 따라 달라질 수 있습니다.

## 1. 스탬피드는 왜 생기나

가장 흔한 캐시 코드는 이렇게 생겼습니다. 캐시에 있으면 돌려주고, 없으면 DB에서 읽어 캐시에 넣습니다(cache-aside).

```js
async function naiveGet(redis) {
  const hit = await redis.get(KEY);
  if (hit) return JSON.parse(hit);
  const value = await slowDbQuery();          // 200ms 걸리는 조회
  await redis.set(KEY, JSON.stringify(value), { EX: 60 });
  return value;
}
```

문제는 캐시가 비어 있는 동안 들어온 요청이 **전부** DB를 부른다는 점입니다. 이 현상을 다룬 논문(Vattani 외, PVLDB 2015)은 예를 하나 듭니다. 초당 10번 읽히는 키를 다시 계산하는 데 3초가 걸리면, 그동안 들어온 30개 요청이 모두 다시 계산한다는 것입니다. 요청이 많을수록, 다시 계산하는 시간이 길수록 피해가 커집니다.

더 나쁜 건 악순환입니다. DB에 요청이 몰리면 조회가 느려지고, 조회가 느려지면 캐시가 비어 있는 시간이 길어지고, 그동안 더 많은 요청이 DB로 갑니다. Wikipedia는 이를 일종의 연쇄 장애(cascading failure)로 설명합니다.

## 2. 재현: 요청 500개가 빈 캐시를 만나면

캐시를 지운 직후 요청 500개를 동시에 보냈습니다. 서버 4대에 고르게 나눠 들어가도록 했습니다.

| 방법 | DB 호출 | 동시 DB 호출 최대 | p99 응답 |
|---|---|---|---|
| 대비 없음 | 500회 | 500 | 212ms |
| 방법 1: 서버 안에서 요청 합치기 | 4회 | 4 | 208ms |
| 방법 2: Redis 락 | 1회 | 1 | 227ms |

대비가 없으면 요청 500개가 모두 DB를 불렀습니다. 같은 값을 500번 읽은 것입니다. 실제 DB라면 커넥션 풀이 바닥나고 다른 쿼리까지 밀렸을 상황입니다. 세 번 반복해도 결과는 같았습니다.

## 3. 방법 1: 서버 안에서 같은 요청 합치기

같은 키를 이미 누가 DB에서 읽고 있다면, 뒤에 온 요청은 새로 읽지 말고 **진행 중인 결과를 함께 기다리면** 됩니다. 이를 요청 합치기(request coalescing)라고 합니다. Go 언어의 `singleflight` 패키지가 대표적인 구현으로, 공식 문서는 이를 중복 함수 호출을 억제하는 장치라고 설명합니다.

```js
function makeCoalescedGet(redis) {
  const inFlight = new Map();                 // 키 → 진행 중인 Promise
  return async function coalescedGet() {
    const hit = await redis.get(KEY);
    if (hit) return JSON.parse(hit);
    if (!inFlight.has(KEY)) {
      const p = (async () => {
        const value = await slowDbQuery();
        await redis.set(KEY, JSON.stringify(value), { EX: 60 });
        return value;
      })().finally(() => inFlight.delete(KEY));
      inFlight.set(KEY, p);
    }
    return inFlight.get(KEY);                 // 다 같이 같은 결과를 기다린다
  };
}
```

결과는 DB 호출 4회였습니다. 서버마다 한 번씩 읽었기 때문입니다. 구현이 간단하고 Redis에 부담도 없지만, **서버가 100대면 DB 호출도 100회**입니다. 서버 수가 적거나 DB 조회가 아주 무겁지 않다면 이것만으로도 충분할 수 있습니다.

## 4. 방법 2: Redis 락으로 전체에서 한 번만

서버가 많다면 Redis에 락을 잡아 **전체에서 한 요청만** DB를 읽게 할 수 있습니다. Redis 공식 문서가 안내하는 단일 인스턴스 락 방식은 `SET 키 임의값 NX PX 밀리초`입니다. `NX`는 키가 없을 때만 저장하고, `PX`는 만료 시간을 밀리초로 정합니다.

```js
const UNLOCK = `if redis.call("get", KEYS[1]) == ARGV[1] then
  return redis.call("del", KEYS[1]) else return 0 end`;

async function lockedGet(redis) {
  for (let i = 0; i < 50; i++) {
    const hit = await redis.get(KEY);
    if (hit) return JSON.parse(hit);
    const token = randomUUID();
    const ok = await redis.set(`lock:${KEY}`, token, { NX: true, PX: 2000 });
    if (ok) {
      try {
        const again = await redis.get(KEY);   // 락 잡는 사이 누가 채웠을 수 있다
        if (again) return JSON.parse(again);
        const value = await slowDbQuery();
        await redis.set(KEY, JSON.stringify(value), { EX: 60 });
        return value;
      } finally {
        await redis.eval(UNLOCK, { keys: [`lock:${KEY}`], arguments: [token] });
      }
    }
    await sleep(20);                          // 잠깐 기다렸다가 캐시를 다시 본다
  }
  return slowDbQuery();                       // 너무 오래 걸리면 최후 수단
}
```

DB 호출은 1회로 줄었습니다. 대신 나머지 요청은 20ms 간격으로 캐시를 다시 확인하며 기다리므로, p99가 227ms로 조금 늘었습니다.

구현할 때 꼭 지킬 점이 있습니다.

- **락 값은 요청마다 고유하게:** Redis 문서는 락 값이 모든 클라이언트와 요청에서 겹치지 않아야 한다고 합니다.
- **풀 때는 내 락인지 확인:** 그냥 `DEL`하면 내 락이 만료된 뒤 다른 요청이 잡은 락을 지울 수 있습니다. 그래서 값이 같을 때만 지우는 Lua 스크립트를 씁니다. Redis 8.4부터는 `DELEX` 명령으로도 할 수 있다고 문서에 나와 있습니다.
- **락에는 반드시 만료 시간:** 락을 잡은 서버가 죽어도 2초 뒤에는 풀리게 합니다.
- **한계를 알고 쓰기:** Redis 문서는 단일 인스턴스 락이 가끔 경쟁 상태가 생겨도 괜찮은 경우에 쓸 만하다고 설명합니다. 장애 조치(failover) 때 두 요청이 동시에 락을 가질 수 있기 때문입니다. 캐시 갱신은 두 번 실행돼도 큰일이 아니니 괜찮지만, 결제처럼 정확히 한 번이어야 하는 일에는 이 방식만으로는 부족합니다.

## 5. 방법 3: 만료되기 전에 미리, 확률적으로 갱신 (XFetch)

방법 1·2는 캐시가 **이미 비어 버린 뒤**에 몰림을 줄입니다. 아예 비지 않게 하려면 만료 직전에 누군가 미리 갱신하면 됩니다. 다만 모든 요청이 "만료 5초 전이면 갱신"처럼 같은 규칙을 쓰면, 그 시점에 또 한꺼번에 몰립니다.

앞의 논문은 각 요청이 **확률적으로** 조기 갱신 여부를 정하는 방법을 제안하고, 이를 XFetch라고 부릅니다. 핵심은 한 줄입니다.

```text
현재 시각 − Δ · β · ln(rand()) ≥ 만료 시각  이면 지금 다시 계산한다
```

- **Δ(델타):** 지난번 다시 계산에 걸린 시간입니다. 오래 걸리는 값일수록 더 일찍 갱신을 시작합니다.
- **β(베타):** 얼마나 일찍 갱신할지 정하는 값입니다. 논문은 기본값 1로도 효과적이고, 더 일찍 갱신해도 괜찮다면 키워도 된다고 설명합니다.
- **rand():** 0과 1 사이 난수입니다. ln(rand())는 음수라 왼쪽은 현재 시각보다 조금 뒤가 되고, 만료가 가까울수록 조건을 만족할 확률이 높아집니다.

논문은 이 방식의 갱신 시점 분포(지수 분포)가 이론적으로 최적임을 보였습니다. 구현은 캐시 값에 Δ와 만료 시각을 함께 저장하면 됩니다.

```js
async function xfetchGet(redis, ttlMs, beta = 1) {
  const raw = await redis.get(KEY);
  const cached = raw && JSON.parse(raw);
  const now = Date.now();
  if (!cached || now - cached.delta * beta * Math.log(Math.random()) >= cached.expiry) {
    const start = Date.now();
    const value = await slowDbQuery();
    const delta = Date.now() - start;
    const expiry = Date.now() + ttlMs;
    await redis.set(KEY, JSON.stringify({ value, delta, expiry }), { PX: ttlMs });
    return value;
  }
  return cached.value;
}
```

### 계속 들어오는 부하에서 비교

스탬피드는 한 번이 아니라 **TTL이 끝날 때마다** 반복됩니다. 그래서 초당 200건을 10초 동안 보내면서 TTL을 2초로 짧게 두어 여러 번 만료되게 했습니다. 세 번 돌린 결과입니다.

| 방법 | DB 호출 | 동시 DB 호출 최대 | DB를 기다린 요청 |
|---|---|---|---|
| 대비 없음 | 147~150회 | 38 | 147~150 / 2,000 |
| XFetch | 16~23회 | 4~7 | 16~23 / 2,000 |

대비가 없으면 만료될 때마다 30~40개 요청이 한꺼번에 DB로 갔습니다. XFetch는 DB 호출이 6~9분의 1로 줄었고, 무엇보다 **캐시가 비는 순간이 없어** 대부분의 요청이 캐시에서 바로 답을 받았습니다.

다만 동시 DB 호출이 1까지 줄지는 않았습니다. 만료가 가까워지면 여러 요청이 거의 동시에 조기 갱신을 결정할 수 있기 때문입니다. 정확히 한 번만 갱신하고 싶다면, XFetch로 갱신을 결정한 요청이 **방법 2의 락을 잡은 경우에만** 갱신하고 나머지는 기존 값을 돌려주도록 조합할 수 있습니다.

## 6. 방법 4: 많은 키가 동시에 만료되지 않게 TTL에 지터

지금까지는 키 하나의 문제였습니다. 그런데 배포 직후나 매일 정해진 시각에 **수많은 키를 한꺼번에 채우고 TTL을 똑같이** 주면, 그 키들이 같은 순간에 함께 만료됩니다. 이때는 키마다 TTL에 작은 무작위 값(지터, jitter)을 더해 만료 시점을 흩습니다.

```js
const baseTtl = 600;                                  // 10분
const jitter = Math.floor(Math.random() * 60);        // 0~59초
await redis.set(key, value, { EX: baseTtl + jitter });
```

간단하지만 키가 많은 서비스에서는 효과가 큽니다. 같은 이유로, 정해진 시각에 캐시를 일괄 갱신하는 배치가 있다면 실행 시각도 흩어 두는 편이 안전합니다.

## 7. 함께 쓰면 좋은 방법: 오래된 값이라도 돌려주기

AWS Builders' Library의 캐싱 글은 소프트 TTL과 하드 TTL을 나누는 방식을 소개합니다. 소프트 TTL이 지나면 갱신을 시도하되, 갱신이 실패하거나 DB가 응답하지 않으면 **하드 TTL까지는 기존 값을 계속 쓰는** 방식입니다. 같은 글은 요청 합치기도 썬더링 허드(thundering herd)를 막는 방법으로 설명합니다.

조금 오래된 값을 보여 줘도 되는 데이터(상품 목록, 랭킹, 설정 값 등)라면, 스탬피드뿐 아니라 DB 장애 때도 서비스를 버티게 해 줍니다. 반대로 재고나 잔액처럼 늘 최신이어야 하는 값에는 맞지 않습니다.

## 8. 어떤 방법을 고를까

| 상황 | 추천 조합 |
|---|---|
| 서버 몇 대, 조회가 가벼움 | 방법 1 요청 합치기 |
| 서버가 많고 조회가 무거움 | 방법 1 + 방법 2 Redis 락 |
| 아주 인기 있는 키, 응답 지연에 민감 | 방법 3 XFetch (+ 락) |
| 많은 키를 한꺼번에 채움 | 방법 4 TTL 지터 |
| 오래된 값을 보여 줘도 됨 | 소프트/하드 TTL로 기존 값 돌려주기 |

대부분의 서비스는 **요청 합치기와 TTL 지터**만으로 시작해도 충분합니다. 그다음 모니터링에서 특정 키의 만료 시점에 DB 부하가 튀는 게 보이면 락이나 XFetch를 더하면 됩니다.

## 마무리

- 캐시 스탬피드는 인기 키가 만료되는 순간 요청이 한꺼번에 DB로 몰리는 현상입니다.
- 요청 합치기는 **서버 안**의 중복을, Redis 락은 **서버 전체**의 중복을 막습니다.
- XFetch는 만료 전에 **확률적으로 미리 갱신**해 캐시가 비는 순간 자체를 줄입니다.
- 많은 키가 함께 만료되지 않도록 **TTL에 지터**를 더합니다.
- 오래된 값을 보여 줘도 되는 데이터라면 **기존 값을 돌려주며** 갱신하는 방식이 장애에도 강합니다.

예제 전체 코드는 위 코드 조각에 가짜 DB 함수와 측정 코드를 더한 것입니다. Docker로 `redis:7`을 띄우고 `redis` npm 패키지 4.7을 설치하면 같은 실험을 해 볼 수 있습니다.

## 참고 자료

- Andrea Vattani, Flavio Chierichetti, Keegan Lowenstein (PVLDB Vol. 8 No. 8), [Optimal Probabilistic Cache Stampede Prevention](https://www.vldb.org/pvldb/vol8/p886-vattani.pdf), 2015
- Redis, [SET 명령 문서](https://redis.io/docs/latest/commands/set/)
- Redis, [Distributed Locks with Redis](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)
- Go, [singleflight 패키지 문서](https://pkg.go.dev/golang.org/x/sync/singleflight)
- AWS Builders' Library (Jas Chhabra), [Caching challenges and strategies](https://aws.amazon.com/builders-library/caching-challenges-and-strategies/), 2019-12-05
- Wikipedia, [Cache stampede](https://en.wikipedia.org/wiki/Cache_stampede)
