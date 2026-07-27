---
title: 이미지 Lazy Load (canceled) 대량 발생 — 세 원인 분해와 캐시 프로브 자기취소 수정
published: 2026-07-22
tags: [Image, LazyLoading, Network, Javascript, Performance, Troubleshooting]
category: Troubleshooting
series: Troubleshooting & Refactoring Notes
draft: false
---

# 이미지 Lazy Load (canceled) 대량 발생 — 세 원인 분해와 캐시 프로브 자기취소 수정

> **TL;DR** — 이미지 그리드 지면에서 lazy 로드 시 devtools Network에 `(canceled)` 요청이 대량으로 찍히는 현상을 조사해, 겹쳐 보이던 증상을 **성격이 다른 세 원인**으로 분리했다. ① lazy loader의 "캐시 프로브 자기취소"(로드마다 요청을 만들었다 즉시 취소하고 재요청), ② CDN 캐시 헤더로 인한 브라우저 캐시 미사용, ③ scroll-top 시 언마운트로 인한 정상적인 로드 중단. 이번 글에서는 그중 ①을 **단일 `Image`로 캐시 프로브와 프리로드를 겸용**하도록 수정해 이미지당 요청 1개·스퓨리어스 취소 0으로 해소한 과정을 기록한다. 원인 ①의 유입 지점은 [2편](/posts/troubleshooting-and-refactoring/image-component-refactoring/)에서 "캐시 히트 동기 감지"로 소개했던 바로 그 최적화였다. 원인 ②는 코드 밖(인프라) 문제라 별도로 풀었고, [4편](/posts/troubleshooting-and-refactoring/image-cdn-cache-control/)에서 다룬다.

---

## 배경 — devtools를 뒤덮은 (canceled)

카테고리 같은 이미지 그리드 지면에서 이미지를 lazy 로드할 때, devtools Network 탭에 `(canceled)` 상태의 이미지 요청이 대량으로 잡혔다. 특히 **빠른 역스크롤**이나 **scroll-top 버튼**을 눌렀을 때 두드러졌고, "브라우저 캐시가 안 되고 있는 것 아니냐"는 의심으로 이어졌다.

재현 조건은 이랬다.

- **환경**: 운영·QA 모두 동일 재현. 특정 지면의 앱 코드나 클라이언트 리사이즈 적용 여부와 무관
- **트리거**: 빠른 스크롤 / 빠른 연속 역스크롤 / scroll-top 버튼
- **영향 범위**: 디자인 시스템 이미지 컴포넌트의 lazy 로드를 쓰는 모든 지면

`(canceled)`가 많다는 것 자체는 버그의 증거가 아니다. 뷰포트를 떠난 이미지의 로드를 중단하는 것은 lazy loading의 올바른 동작이기 때문이다. 문제는 **어디까지가 정상 취소이고 어디부터가 낭비인지**가 겹쳐 보여서 구분되지 않는다는 점이었다. 그래서 조사의 목표를 "취소를 없애자"가 아니라 "**취소를 성격별로 분해하자**"로 잡았다.

---

## 조사 — 추측을 수치로 바꾸기

세 가지 도구를 겹쳐 썼다.

**① 코드 리딩** — 사용처 컴포넌트에서 시작해 디자인 시스템의 코어 이미지 컴포넌트 → lazy load 디렉티브 → `ImageLazyLoader` 유틸까지 이미지 로드 경로를 따라 내려갔다.

**② git log/blame** — `ImageLazyLoader`에서 요청을 만들었다 취소하는 코드가 언제 들어왔는지 커밋 단위로 추적했다. 유입 시점은 [2편](/posts/troubleshooting-and-refactoring/image-component-refactoring/)에서 다뤘던 로딩 파이프라인 개선 커밋 — "캐시 히트 동기 감지"를 도입한 바로 그 변경이었다.

**③ Playwright 계측** — `img.src` 세터를 패치해 src가 빈 문자열로 교체되는(= 로드 중단) 횟수를 세는 스크립트를 만들어, 스크롤 시나리오별로 취소를 수치화했다. 그리드 지면에서 빠른 역스크롤 시나리오로 abort 96~239건이 재현됐다. 같은 계측에서 이미지 로드 소요시간은 p50 3ms / p90 158ms로, "이미지가 느려서 취소가 쌓인다"는 가설은 기각됐다.

**④ HAR 분석** — 동일 URL이 세 번 요청되는 케이스(200 풀 다운로드 2회 + status 0 canceled 1회)를 응답 헤더·타이밍·initiator로 분해했다. 여기서 취소와 무관한 별개의 문제(원인 ②)가 드러났다.

---

## 원인 분해 — 성격이 다른 세 가지

겹쳐 보이던 `(canceled)`는 세 개의 독립된 원인으로 갈라졌다.

| 원인                                   | 성격                    | 처리                                                                                 |
| -------------------------------------- | ----------------------- | ------------------------------------------------------------------------------------ |
| ① 캐시 프로브 자기취소                 | lazy loader 버그 (회귀) | **이번 수정으로 해소**                                                               |
| ② CDN 캐시 헤더 → 브라우저 캐시 미사용 | 인프라 설정 문제        | 스코프 분리 → [4편](/posts/troubleshooting-and-refactoring/image-cdn-cache-control/) |
| ③ 언마운트 시 로드 중단                | 정상 동작               | 유지 (수정하지 않음)                                                                 |

### 원인 ① — 캐시 프로브 자기취소 (이번 수정 대상)

`loadImage()`는 브라우저 캐시 히트를 동기적으로 판별하기 위해 프로브용 `Image`(`testImg`)에 `src`를 세팅한다. 이 시점에 **네트워크 요청이 시작된다**. 동기 `complete` 체크가 미스하면 `preloadImage()`로 넘어가는데, 그 첫 줄의 정리 로직이 `testImg.src = ''`로 방금 시작한 요청을 취소하고, **새 `Image`를 만들어 같은 URL을 다시 요청**한다.

```text
loadImage()
├── new Image() testImg → src 세팅          ← 요청 A 시작
├── (동기 캐시 미스) preloadImage()
│     └── cleanupImage() → testImg.src=''   ← 요청 A "(canceled)"
└── new Image() → src 세팅                  ← 요청 B (실제 로드)
```

결과적으로 **동기 캐시 히트가 아닌 거의 모든 이미지가 "취소된 프로브 1개 + 실제 요청 1개"** 를 만든다. 그리드 지면처럼 이미지가 수백 장인 곳에서는 이것만으로 Network 탭이 `(canceled)`로 뒤덮인다.

이 코드는 2편에서 "캐시 히트 동기 감지 — 캐시된 이미지는 placeholder를 스킵한다"로 소개했던 최적화와 함께 들어왔다. 캐시 히트 경로만 보면 훌륭하게 동작하지만, **캐시 미스 경로에서 프로브가 자기 요청을 취소하고 재요청하는 비용**은 당시 검증에서 놓쳤다. 개선 작업이 회귀를 함께 실어 온 셈이라, 시리즈의 기록 차원에서도 정직하게 남겨둔다.

### 원인 ② — CDN 캐시 헤더로 인한 브라우저 캐시 미사용 (스코프 밖)

HAR에서 같은 이미지(약 152KB)가 30초 안에 **두 번 모두 풀 다운로드**되는 것을 확인했다. 응답은 `cache-control: max-age=2592000`(30일)인데 `age` 헤더가 약 47.6일 — **브라우저에 도착하는 순간 이미 만료(stale)** 된 응답이었다. 게다가 `ETag`/`Last-Modified`가 없어 재검증(304)도 불가능해, 재요청은 매번 200 풀 다운로드가 된다.

이건 lazy loader가 아니라 CDN/이미지 서버 설정의 문제다. 클라이언트 코드로는 개입할 수 없는 영역이라 **별도 트랙으로 분리**했고, 분석·협조 요청·적용·검증의 전체 과정은 [4편 — 브라우저 disk cache가 동작하지 않던 이유](/posts/troubleshooting-and-refactoring/image-cdn-cache-control/)에 정리했다.

### 원인 ③ — scroll-top 언마운트 취소 (정상 동작)

scroll-top 버튼은 Virtual Scroll이 적용된 페이지 상단으로 순간 이동하면서 중간 행들을 언마운트시킨다. 이때 로드 중이던 이미지 요청이 abort되어 `net::ERR_ABORTED`로 찍힌다. HAR의 initiator 체인은 `handleIntersect → loadImage → new Image()`였고, 상당수는 큐(blocked) 상태에서 전송도 되기 전에 취소됐다.

이것은 **고쳐야 할 문제가 아니다**. 화면을 떠난 행의 이미지를 끝까지 받는 것이야말로 대역폭 낭비이고, lazy loading을 쓰는 이유와 정반대다. 언마운트 경로는 내부적으로 destroy 가드가 있어 `onError` 콜백이나 콘솔 경고도 내지 않는다 — 남는 것은 devtools의 `(canceled)` 표기뿐이며, 이는 무해하다.

---

## 수정 — 단일 Image로 프로브와 프리로드를 겸용한다

원인 ①의 수정 방향은 단순하다. **프로브용 `Image`를 따로 만들어 취소·재요청하지 말고, 프로브에 쓴 `Image`를 그대로 프리로드에 재사용한다.** 요청은 이미지당 한 번만 시작되고, 취소는 실제로 로드 도중 파괴(destroy-mid-load)될 때만 발생한다.

```typescript
// After — loadImage(): 단일 Image로 캐시 프로브 + 프리로드 겸용
const probeImage = new Image();
this.currentImage = probeImage;
if (this.imageSrcset) probeImage.srcset = this.imageSrcset;
probeImage.src = this.imageSrc; // 네트워크 요청은 이 한 번뿐

// 동기 캐시 히트(srcset 미사용 시에만 신뢰) → 즉시 표시. 기존 최적화 유지
if (!this.imageSrcset && probeImage.complete && probeImage.naturalWidth > 0) {
  this.currentImage = null;
  showImage();
  return;
}
// 캐시 미스 → 같은 Image를 preloadImage()가 그대로 기다린다
```

```typescript
// After — preloadImage(): 새 Image를 만들지 않는다
private preloadImage(): Promise<void> {
  return new Promise((resolve, reject) => {
    const img = this.currentImage;
    if (!img) {
      reject(new Error("no current image"));
      return;
    }

    // 핸들러 부착 전에 이미 로드가 끝난 경우 (프로브 직후 동기 완료 등)
    if (img.complete) {
      img.naturalWidth > 0
        ? resolve()
        : reject(new Error(`Failed to load image: ${img.src}`));
      return;
    }

    img.onload = () => resolve();   // 최신 요청 여부·destroy 가드는 유지
    img.onerror = () => reject(new Error(`Failed to load image: ${img.src}`));
  });
}
```

구현하면서 챙긴 디테일 세 가지.

- **동기 캐시 히트 판정은 `srcset` 미사용일 때만 신뢰한다.** `srcset`이 걸려 있으면 브라우저의 후보 선택 과정이 개입하므로, `src` 세팅 직후의 동기 `complete` 판정을 캐시 히트의 근거로 쓸 수 없다.
- **핸들러 부착 전 완료 케이스를 별도 처리한다.** 프로브가 이미 로드를 끝낸 상태로 `preloadImage()`에 도달할 수 있으므로, `complete`면 핸들러를 붙이지 않고 즉시 resolve/reject한다. 이때도 `naturalWidth > 0` 확인이 필요하다 — 에러 응답도 `complete === true`가 되기 때문이다 (2편에서 다뤘던 함정 그대로).
- **destroy 시 정리 경로는 그대로 유지한다.** 진행 중 요청의 취소와 pending Promise reject는 원인 ③의 정상 동작을 담당하는 부분이라 건드리지 않았다.

### 채택하지 않은 대안

| 대안                                         | 장점                                    | 미채택 이유                                                     |
| -------------------------------------------- | --------------------------------------- | --------------------------------------------------------------- |
| 프로브(testImg) 자체를 제거하고 바로 preload | 구현 단순                               | 동기 캐시 히트 즉시 표시 최적화 상실 — 2편 개선을 되돌리는 방향 |
| destroy 시 abort하지 않고 로드를 끝까지 완료 | devtools에서 `(canceled)` 표기가 사라짐 | 떠난 행의 이미지까지 풀 다운로드 → 대역폭 낭비. 목표와 정반대   |

두 번째 대안은 특히 경계할 만하다. "Network 탭을 깔끔하게"가 목표가 되는 순간, 사용자에게 이로운 취소(원인 ③)까지 없애는 퇴보를 최적화로 착각하게 된다.

---

## Before / After

| 구분                       | Before                             | After             |
| -------------------------- | ---------------------------------- | ----------------- |
| 동기 캐시 히트가 아닌 로드 | 요청 2개 (프로브 취소 + 실제 요청) | 요청 1개 · 취소 0 |
| 로드마다의 프로브 취소     | 발생                               | 없음              |
| 동기 캐시 히트 즉시 표시   | 동작                               | 유지              |
| 언마운트(scroll-top) 취소  | 발생 (정상)                        | 유지 (정상)       |

수정 후 같은 그리드 지면에서 Network 탭의 `(canceled)`는 실제 언마운트 시점의 것만 남는다. 남은 취소는 버그가 아니라 lazy loading이 일하고 있다는 흔적이다.

---

## 마무리, 그리고 남은 과제

이번 조사에서 얻은 것은 수정 자체보다 **분해의 순서**였다. "취소가 많다"는 하나의 증상 안에 버그(①), 인프라 설정(②), 정상 동작(③)이 겹쳐 있었고, 세 가지를 같은 방법으로 다뤘다면 어느 것도 제대로 풀지 못했을 것이다. 계측으로 "느려서가 아님"을 먼저 기각한 것도 방향을 줄이는 데 컸다.

- **원인 ② (CDN 캐시 헤더)** — 클라이언트 코드 밖의 문제라 인프라 담당 조직과의 협조로 별도 진행했다. 분석부터 적용 후 실측까지 [4편](/posts/troubleshooting-and-refactoring/image-cdn-cache-control/)에 이어진다.
- **취소 개수 자체를 더 줄이는 일** — 남은 취소는 리스트 렌더링 레버(overscan 확대, `<KeepAlive>` 등)로 줄일 수는 있으나, 이는 로더가 아니라 지면 구조의 문제이고 devtools 표기를 깔끔하게 만드는 것 이상의 실익이 없어 진행하지 않기로 했다.
- **회귀를 놓치지 않는 검증** — 캐시 히트 경로의 개선이 캐시 미스 경로의 회귀를 실어 왔다. 로딩 파이프라인을 고칠 때 히트/미스 양 경로의 네트워크 요청 수를 함께 확인하는 것을 이후 작업의 체크 항목으로 남긴다.

---

## 관련 자료

- [2편 — Image Component Refactoring: flex 의존성 제거와 CSS 크로스페이드](/posts/troubleshooting-and-refactoring/image-component-refactoring/)
- [4편 — 브라우저 disk cache가 동작하지 않던 이유와 CDN Cache-Control 적용](/posts/troubleshooting-and-refactoring/image-cdn-cache-control/)
- [MDN — HTMLImageElement.complete](https://developer.mozilla.org/en-US/docs/Web/API/HTMLImageElement/complete)
- [MDN — Lazy loading](https://developer.mozilla.org/en-US/docs/Web/Performance/Guides/Lazy_loading)
- [Chrome DevTools — Network features reference](https://developer.chrome.com/docs/devtools/network/reference)
