---
title: 브라우저 disk cache가 동작하지 않던 이유 — stale-on-arrival과 CDN Cache-Control 적용
published: 2026-07-22
tags: [Browser, Cache, CDN, CloudFront, HTTP, Performance, Troubleshooting]
category: Troubleshooting
series: Troubleshooting & Refactoring Notes
draft: false
---

# 브라우저 disk cache가 동작하지 않던 이유 — stale-on-arrival과 CDN Cache-Control 적용

> **TL;DR** — [이미지 Lazy Load 취소 관련](/posts/troubleshooting-and-refactoring/lazy-load-canceled-requests/) 원인을 분석하면서 이미지 CDN 응답이 CloudFront 캐시는 잘 타는데 **브라우저 disk cache로는 전혀 재사용되지 않았음을 발견했다.** 원인은 ① CDN이 오리진의 `max-age`(30일)보다 오래 엣지에 객체를 보관하면서 `Cache-Control`은 그대로 전달해, 브라우저가 **도착 즉시 만료(stale)된 응답**을 받는 것, ② `ETag`/`Last-Modified`가 없어 재검증(304)조차 불가능한 것이었다. 클라이언트 코드로는 개입할 수 없는 문제라 인프라 담당 조직에 협조를 요청했고, 뷰어 `Cache-Control`이 `public, max-age=31536000, immutable`로 적용됐다. 적용 후 HAR 실측에서 이미지 요청 1,633건 중 **1,375건(84.2%)이 disk cache 히트**로 전환된 것을 확인했다.

---

## 배경 — CDN 히트인데 왜 매번 다시 받는가

이미지 Lazy Load 처리에 대한 대량 `(canceled)` 요청을 분해하다가 별개의 문제 하나가 드러났다. 같은 이미지가 30초 안에 두 번 모두 풀 다운로드되고 있었던 것이다. 응답 헤더에는 `x-cache: Hit from cloudfront`가 찍혀 있었다 — CDN 캐시는 동작한다. 그런데 **브라우저의 disk cache는 왜 이 이미지를 재사용하지 않는가?**

CDN 히트라 응답 자체는 빠르지만, 네트워크 왕복과 이미지 바이트 전송이 매번 발생한다. 사용자가 페이지를 재방문하거나 이동할 때마다 (특히 Virtual Scroll이 적용된 경우 리스트 아이템이 reverse scroll로 다시 마운트 되는 시점) 같은 이미지를 CDN에서 다시 내려받는다면, 브라우저 캐시가 정상 동작할 때 대비 체감 성능과 데이터 사용량 모두 손해가 발생한다. 이미지가 수십 장씩 노출되는 홈/목록 지면에서는 특히 그렇다.

문제의 도메인은 서비스 전체가 공유하는 이미지 CDN(CloudFront + istio-envoy 오리진)이었으므로, 원인이 확인되면 **설정 변경 한 번으로 모든 클라이언트가 이득**을 보는 구조이기도 했다.

---

## 분석 1 — 도착하는 순간 이미 만료된(stale-on-arrival) 응답

프로덕션 홈 지면에 노출되는 이미지 20건의 응답 헤더를 실측했다 (2026-07-14 기준).

| 구분       | x-cache              | Age         | Cache-Control          | ETag / Last-Modified |
| ---------- | -------------------- | ----------- | ---------------------- | -------------------- |
| 이미지 A   | Hit from cloudfront  | **75.09일** | max-age=2592000 (30일) | 없음 / 없음          |
| 이미지 B   | Hit from cloudfront  | **33.12일** | max-age=2592000 (30일) | 없음 / 없음          |
| 이미지 C   | Hit from cloudfront  | 1.87일      | max-age=2592000 (30일) | 없음 / 없음          |
| 그 외 16건 | Miss from cloudfront | 0일         | max-age=2592000 (30일) | 없음 / 없음          |

캐시 히트한 4건 중 2건이 **`Age`가 이미 `max-age`를 초과한 상태**로 도착했고, 20건 전부 검증자(`ETag`/`Last-Modified`)가 없었다.

이게 왜 disk cache를 무력화하는지는 HTTP 캐시 표준(RFC 9111)의 신선도 계산을 따라가면 명확해진다.
브라우저는 응답의 나이를 계산할 때 **`Age` 헤더를 초기값으로 삼는다** (`corrected_initial_age = max(apparent_age, age_value)`).
관측된 실제 헤더로 계산하면:

```text
freshness_lifetime = max-age            = 2,592,000s (30.00일)
current_age        = Age + 체류시간      = 2,685,457s (31.08일)
→ current_age > freshness_lifetime      = STALE (남은 신선도 -93,457s)
```

즉 이 응답은 **다운로드가 끝나는 그 순간 이미 수명을 넘긴 상태**다. 브라우저는 디스크에 저장은 하지만 "재사용 전 반드시 재검증"으로 분류하므로, disk cache 히트가 원천적으로 발생할 수 없다.

Age가 max-age를 넘는 이유는 CloudFront 쪽에 있다. CloudFront는 캐시 동작(cache behavior)의 TTL 설정에 따라 **오리진이 지정한 30일보다 오래 엣지에 객체를 보관할 수 있는데, 이때도 오리진의 `Cache-Control`은 수정 없이 그대로 뷰어에게 전달**한다. 엣지 보존 기간은 CloudFront TTL이 이기고, 브라우저가 받는 헤더는 여전히 `max-age=2592000`이다. 엣지 수명 후반부에 그 객체를 처음 받는 사용자는 항상 만료된 응답을 받는다. 실측 Age 최대값이 75일이라는 점에서 CloudFront TTL이 30일을 크게 상회한다고 추론했다 (정확한 Min/Max TTL 값은 배포 설정 권한 밖이라 확인하지 못했다).

## 분석 2 — 검증자가 없어 304조차 불가능하다

stale이 됐더라도 검증자가 있으면 이야기가 다르다. 브라우저가 `If-None-Match`/`If-Modified-Since`로 조건부 요청을 보내고, 서버가 304 Not Modified로 답하면 본문 전송 없이 캐시를 되살릴 수 있다.

그런데 이 응답들에는 `ETag`도 `Last-Modified`도 없다. 조건부 요청을 보낼 근거 자체가 없으므로, stale 캐시의 재검증은 **매번 200 + 이미지 전체 바이트 재전송**이 된다. 실제로 `If-Modified-Since`를 강제로 붙여 요청해도 304가 아닌 200 + 전체 본문(84,273 bytes)이 반환되는 것을 확인했다.

두 원인이 결합된 최종 흐름은 이렇다.

```text
사용자 재방문
└── 브라우저: 디스크에 이미지 있음
    └── 신선도 판정 → Age(31~75일) > max-age(30일) → STALE   ← 원인 1
        └── 재검증 시도 → ETag/Last-Modified 없음 → 조건부 요청 불가   ← 원인 2
            └── 이미지 전체 재다운로드 (x-cache: Hit from cloudfront)
```

샘플 이미지는 건당 5~85KB 수준이고 지면 하나에 수십 장이 노출되므로, 재방문마다 수 MB가 불필요하게 재전송되는 셈이다.

---

## 해결 방향 — 뷰어 max-age ≥ 엣지 TTL

핵심 원칙은 부등식 하나로 요약된다.

> **뷰어에게 내려주는 `max-age` ≥ CDN이 엣지에 객체를 보관하는 TTL**

이 부등식이 깨지면 엣지 수명 후반부에 전달된 객체는 항상 만료된 채 브라우저에 도착한다. 이미지 URL이 `{uuid}_{size}.ext` 형태의 **콘텐츠 불변(immutable) 경로**라는 점을 감안하면, 짧은 `max-age`를 유지할 이유가 없다. 그래서 제안한 값은:

```text
Cache-Control: public, max-age=31536000, immutable
```

1년으로 설정하면 Age가 75일이어도 브라우저에서 여전히 신선하고, `immutable` 지시자 덕분에 새로고침 시의 불필요한 재검증도 건너뛴다. 오리진(리사이즈 서버) 수정이 어렵다면 CloudFront **Response Headers Policy**로 뷰어 응답의 `Cache-Control`을 덮어쓰는 방법도 같은 효과를 낸다.

여기에 두 가지를 함께 요청했다. ② 오리진에 `ETag` 또는 `Last-Modified` 추가 — 캐시가 만료되거나 강제 새로고침될 때 304로 끝낼 수 있게. ③ CloudFront TTL과 오리진 `max-age`의 정렬 확인 — 향후 `max-age`를 조정할 때 같은 문제가 재발하지 않게.

### 채택하지 않은 대안

| 대안                                | 장점                         | 미채택 이유                                                                                                                |
| ----------------------------------- | ---------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `s-maxage`로 CDN만 장기 캐싱        | 오리진 부하 감소             | 브라우저는 `s-maxage`를 무시하지만 `Age`는 그대로 읽는다 — 신선도 계산 오염이 동일하게 발생, **문제가 전혀 해결되지 않음** |
| CloudFront TTL을 30일 이하로 낮추기 | Age가 max-age를 넘지 않게 됨 | Age 29일 객체를 받은 사용자의 잔여 신선도는 1일뿐. 근본 해결이 아니고 오리진 재요청 비용만 증가                            |
| 클라이언트에서 우회                 | 타 조직 협조 불필요          | HTTP 캐시 신선도 판정은 브라우저 고유 동작 — 클라이언트 코드로 개입할 방법이 **구조적으로 없음**                           |

단, `max-age` 1년에는 전제가 하나 붙는다. **이미 배포된 URL의 이미지 내용을 나중에 교체할 수 없게 된다.** 동일 경로에 다른 이미지를 덮어쓰는 운영 케이스가 없는지 — URL 스킴이 정말 콘텐츠 불변인지 — 를 담당 조직에 확인 항목으로 함께 전달했다.

---

## 협조 요청과 적용 — 코드 밖의 문제를 옮기는 일

이미지 서버/CDN 설정은 devops 팀에서 주로 관리한다. 위 분석을 측정 데이터·재현 커맨드와 함께 문서로 정리해 인프라 담당 조직에 협조 요청했고, 나흘 뒤 **뷰어 `Cache-Control`이 `public, max-age=31536000, immutable`(365일)로 변경 적용**됐다 (2026-07-16 요청 → 07-20 적용). 이미지 리사이즈 결과물의 URL이 콘텐츠 불변이라는 전제도 담당 조직 확인을 거쳤다. 다만 **오리진 `ETag` 추가(요청 ②)는 즉시 반영이 어려워 Backlog** 처리됐다.

클라이언트 코드가 한 줄도 바뀌지 않는 개선이지만, 문제를 분석하고 제안하는 쪽에서 해야 할 일은 분명했다. 실제 CDN에서 저장되는 이미지 전송 및 캐시 처리에 대한 **표준 근거(RFC 9111)와 실측 수치, 제안 값, 리스크(불변 URL 전제)까지 한 문서에 담는 것** — 문제 분석 및 제안사항을 포함한 문서를 전달한 덕분에 원활하게 협조 요청이 이루어졌다.

---

## 검증 — 적용 후 실측

### ① 응답 헤더

적용 후 프로덕션 이미지 응답 헤더에서 제안했던 Cache Control 설정이 전달되는 걸 확인했다.

```text
cache-control: public, max-age=31536000, immutable   ← 제안 값 그대로 적용
age:           2938469                                (34.0일)
x-cache:       Hit from cloudfront
// etag, last-modified 없음                            ← 요청 ② 미적용 (Backlog 확인)
```

같은 `age: 34.0일`짜리 응답이 이전 정책이었다면 도착 즉시 stale이었겠지만, 이제 잔여 신선도가 331일이다.

### ② HAR 실측 — disk cache 히트 84.2%

프로덕션 홈에서 역스크롤을 포함한 세션의 HAR(2026-07-22)로 전수 확인했다.

| 항목                       | 측정값                                                                     |
| -------------------------- | -------------------------------------------------------------------------- |
| 이미지 CDN 도메인 요청     | 1,633건                                                                    |
| **disk cache 히트**        | **1,375건 (84.2%)** — 역스크롤 구간에서도 disk cache 재사용 확인           |
| 네트워크 요청              | 258건 — 전건 `public, max-age=31536000, immutable` + `Hit from cloudfront` |
| Age 분포                   | 최대 89.9일 — 구 정책(30일)이었다면 stale이었을 객체들이 전부 fresh        |
| ETag / Last-Modified       | 258건 중 0건 — 요청 ②는 여전히 유효한 후속 과제                            |
| disk cache가 흡수한 콘텐츠 | 약 73.1MB (실제 네트워크 전송은 약 14.6MB)                                 |

세션 중 브라우저가 그린 이미지 콘텐츠의 대부분을 disk cache가 흡수했고, 네트워크로 나간 것은 처음 보는 이미지들뿐이었다. 이전 발생한 문제 상황인 "같은 이미지를 30초 안에 두 번 풀 다운로드"하던 상태와 비교하면 정확히 의도한 방향이다.

### ③ Lighthouse 스냅샷

콜드 로드 기준 Lighthouse 측정값이다. 신상품 목록 지면은 작업 전(2026-06-08) 측정 기록이 남아 있어 before/after를 비교할 수 있다.

| 지면        | 시점           | Performance | FCP  | LCP      | TBT | CLS   | SI   |
| ----------- | -------------- | ----------- | ---- | -------- | --- | ----- | ---- |
| 메인 홈     | Before (06-18)  | 76          | 1.8s | 1.9s     | 0ms | 0.12  | 2.8s |
| 메인 홈     | After (07-22)  | 86          | 1.2s | 1.4s     | 0ms | 0.09  | 2.2s |
| 신상품 목록 | Before (06-08) | 88          | 1.1s | 1.9s     | 0ms | 0.048 | 1.5s |
| 신상품 목록 | After (07-22)  | **92**      | 1.2s | **1.4s** | 0ms | **0** | 1.7s |

신상품 목록은 88 → 92로 올랐고, 특히 **LCP 1.9s → 1.4s(-0.5s)**, **CLS 0.048 → 0**이 개선을 이끌었다. 이미지가 지배하는 그리드 지면에서 LCP와 CLS가 함께 좋아진 것은 이 시리즈에서 다뤄온 이미지 파이프라인 개선(컴포넌트 비율 확정, 로더 정리, 캐시 정책)의 방향과 일치한다.

다만 Lighthouse는 캐시가 빈 콜드 로드를 측정하므로 **점수 변화에는 같은 시기에 진행한 클라이언트 리사이징 등 병행 작업의 효과가 함께 반영**되어 있다. Cache-Control 변경의 직접 효과(재방문 disk cache)의 근거는 위 HAR 실측이다.

---

## 마무리, 그리고 남은 과제

3편의 원인 분해에서 "인프라 영역"으로 밀어뒀던 조각이 이렇게 닫혔다. 1편의 Safari 렌더링 버그에서 시작해 컴포넌트 구조(2편), 로더의 요청 라이프사이클(3편), 그리고 HTTP 캐시 정책(4편)까지 — 이미지 하나가 화면에 뜨기까지의 경로를 계층별로 한 번씩 손본 셈이 됐다.

이번 건에서 남기고 싶은 것은 두 가지다. 하나는 **`x-cache: Hit`가 "캐시가 잘 된다"를 의미하지 않는다**는 것 — CDN 캐시와 브라우저 캐시는 별개의 계층이고, 각자의 신선도 계산을 따른다. 다른 하나는 클라이언트에서 고칠 수 없는 문제는 **빠르고 명확한 원인 분석 및 해결 방안 제안이 곧 해결 속도**라는 것이다.

남은 과제:

- **오리진 검증자(`ETag`/`Last-Modified`) 추가** — 후속 Backlog 상태 (HAR 258건 전부 검증자 없음 확인). 캐시 만료·강제 새로고침 시의 풀 재다운로드는 이 건이 남아 있다.
- **CloudFront TTL과 오리진 `max-age` 정렬 확인** — 미확인. 현재는 뷰어 `max-age`가 충분히 길어 부등식이 성립하지만, 설정 자체를 정렬해야 향후 조정 시 재발하지 않는다.
- **404 응답의 캐시 정책 분리** — 에러 응답에도 성공과 같은 `cache-control`이 붙는 구성이라, 에러/성공 캐시 정책 분리를 확인 요청 항목으로 남겨뒀다.

---

## 관련 자료

- [3편 — 이미지 Lazy Load (canceled) 대량 발생: 세 원인 분해와 캐시 프로브 자기취소 수정](/posts/troubleshooting-and-refactoring/lazy-load-canceled-requests/)
- [RFC 9111 — HTTP Caching: Freshness](https://www.rfc-editor.org/rfc/rfc9111#name-freshness)
- [AWS — Managing how long content stays in the CloudFront cache (TTL)](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Expiration.html)
- [AWS — Response headers policies](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/understanding-response-headers-policies.html)
- [MDN — Cache-Control: immutable](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control#immutable)
- [MDN — HTTP caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Caching)
