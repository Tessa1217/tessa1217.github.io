---
title: 🐛 Nuxt SSR 환경에서 `useState`에 함수 저장 시 발생한 Serialization 오류
published: 2026-05-21
tags: [Nuxt, Vue, SSR, Troubleshooting]
category: Troubleshooting
series: Troubleshooting & Refactoring Notes
image: /images/nuxt.webp
draft: false
---

# 🐛 Nuxt SSR 환경에서 `useState`에 함수 저장 시 발생한 Serialization 오류

> Nuxt 3 · SSR · `useState` · Payload Serialization
>
> 직접 진입과 SPA 네비게이션에서 동작이 갈리는 이유를 파헤친 기록

---

## 목차

1. [TL;DR](#tldr)
2. [개요](#-개요)
3. [증상](#-증상)
4. [원인 분석](#-원인-분석)
5. [문제 발생의 배경](#-문제-발생의-배경)
6. [회고 및 대안 고찰](#-회고-및-대안-고찰)
7. [배운 점](#-배운-점)
8. [📚 참고 자료](#-참고-자료)

---

## TL;DR

Nuxt 3의 `useState`는 SSR → CSR 전달 과정에서 payload를 직렬화(serialize)하는데, 이때 함수(function) 타입은 직렬화가 불가능해 500 에러가 발생한다. `import.meta.server` 가드로 서버 측 등록을 우회해 해결했지만, 근본적으로는 **"함수를 전역 상태에 담는" 설계 자체**를 재고할 필요가 있었다. 대안으로는 Pinia setup store 내부에 핸들러를 보관하는 방식이 가장 자연스러워 보인다.

---

## 📌 개요

| 항목      | 내용                                                           |
| --------- | -------------------------------------------------------------- |
| 환경      | Nuxt 3, Vue 3 (Composition API), SSR                           |
| 영향 범위 | 새로고침 플로팅 버튼이 동작하는 모든 라우트                    |
| 발생 조건 | 페이지 직접 진입(URL 입력) 또는 브라우저 새로고침 (= SSR 경로) |
| 해결 방식 | `import.meta.server` 가드로 서버 측 registry 등록 차단         |
| 소요 시간 | (작성 시 채우기)                                               |

---

## 🔥 증상

- ❌ 페이지를 **직접 진입(브라우저 주소창에 URL 입력)** 또는 **브라우저 새로고침** 시 → **500 에러** 발생
- ✅ **탭/링크 클릭으로 SPA 네비게이션** 시에는 정상 동작
- 즉, **SSR이 동작하는 경로에서만 발생**, CSR 전환 경로에서는 미발생

> 이 패턴은 거의 항상 "SSR에서만 깨지는 무언가" — 윈도우 객체 접근, hydration mismatch, 또는 **payload 직렬화 실패** — 셋 중 하나를 의심하게 된다.

---

## 🔍 원인 분석

### 🧪 1차 시도 — 인터페이스/옵션 확인

- 우선 호출되는 컴포저블의 타입과 인터페이스를 한 차례 점검
- `useAsyncData`의 `server: false` 옵션을 적용해 **SSR 시점에 API 호출을 우회**하면 정상 동작하는지 확인
- **결과:** 여전히 동일한 500 에러 발생
- **의미:** API fetch 자체의 문제가 아님. SSR 렌더링 파이프라인의 더 앞단에서 깨지고 있다.

### 🧪 2차 시도 — API 동작 순차 배제

- 페이지에서 호출되는 API들을 하나씩 비활성화하며 어디서 터지는지 좁혀보려 함
- **결과:** 모든 API가 SSR 시점에서 동일하게 실패함
- **의미:** 특정 API의 문제가 아니라 **SSR 렌더링 자체**가 망가져 있다. 페이로드 직렬화 단계를 의심해야 함.

### 🧪 3차 시도 — SSR 코드 / `useState` 분석 → 🎯 원인 발견

- 최근 도입한 **refresh registry 패턴**의 `useState` 저장값을 점검
- registry에 각 컴포저블의 **refresh 함수(function)** 가 그대로 등록되고 있었음
- Nuxt 3는 SSR 렌더링 후 클라이언트로 페이로드를 전달할 때 `devalue`를 사용해 상태를 직렬화한다. **함수는 직렬화 대상이 아니다.**
- 따라서 SSR 렌더 도중 registry에 함수가 등록되는 순간, 응답 직전 payload serialization에서 깨지며 500을 반환한 것

```ts
// ❌ Before — 서버에서도 함수가 useState에 등록됨
const registry = useState<Record<string, () => Promise<void>>>(
  "refresh-registry",
  () => ({}),
);

export function useRegisterRefresh(key: string, fn: () => Promise<void>) {
  registry.value[key] = fn; // ← SSR 페이로드 직렬화 시 폭발
}
```

```ts
// ✅ After — 서버에서는 등록 자체를 스킵
export function useRegisterRefresh(key: string, fn: () => Promise<void>) {
  if (import.meta.server) return; // 🛡️ SSR 가드
  registry.value[key] = fn;
}
```

> registry는 **클라이언트에서만 의미 있는** 상태(사용자 인터랙션으로 트리거되는 새로고침 함수들의 모음)이므로, 서버에서 굳이 채울 필요가 없다. 가드를 두는 것이 자연스러운 결론.

---

## 🌱 문제 발생의 배경

처음부터 registry 패턴이었던 것은 아니다. 요구사항이 한 단계씩 늘어나면서 점진적으로 구조가 커진 케이스다.

1. **1차 구현** — 스크롤 시 노출되는 "새로고침 플로팅 버튼" 작업
   - 단순히 `refreshTrigger`를 자식 컴포넌트에 prop/provide로 내려주고, 각 컴포넌트에서 자체적으로 refresh를 수행
2. **요구사항 변경** — UI/UX에서 새로고침 시 **전체 화면 loading indicator** 추가 요청
   - 개별 컴포넌트들의 상태를 모아 "전체가 새로고침 중인지"를 알 필요가 생김
   - → `isRefreshing` 전역 상태 도입
3. **구조 변경** — 일원화된 흐름을 위해 **registry 패턴** 도입
   - 각 컴포저블이 마운트 시점에 자신의 refresh 함수를 registry에 등록
   - 플로팅 버튼 클릭 → `refreshAll()` → registry의 모든 함수를 `Promise.all`로 실행 → `isRefreshing` 토글
4. **테스트 통과** — SPA 네비게이션 환경에서는 모든 flow가 정상 동작
5. **🔥 사건 발생** — 직접 진입/새로고침(=SSR 경로)에서 500

> **회고 포인트:** SPA 내부 동선만 테스트했기 때문에, registry에 함수가 등록되어 payload에 포함되는 시점이 **클라이언트뿐**이었다. 직접 진입 케이스에서는 첫 페이지가 SSR로 그려지므로 등록 시점이 서버로 옮겨가 폭발했다. 향후 SSR 환경에서는 **"직접 진입/새로고침 경로"를 반드시 회귀 테스트에 포함**해야 한다.

---

## 💭 회고 및 대안 고찰

이번 이슈는 단순히 가드 한 줄로 해결됐지만, 그보다 더 본질적인 질문은 **"애초에 함수를 전역 상태에 넣는 것이 옳은가?"** 였다. 가능한 대안들을 정리해본다.

### A. `useState` 대신 `ref`(모듈 스코프)로 관리

```ts
// composables/useRefreshRegistry.ts
const registry = ref<Record<string, () => Promise<void>>>({}); // 모듈 스코프

export function useRefreshRegistry() {
  return { registry, register, refreshAll };
}
```

- ✅ **모듈 스코프 ref는 SSR 페이로드 직렬화 대상이 아님** → 직렬화 오류는 발생하지 않는다.
- ❌ 그러나 Nuxt SSR에서 **모듈 스코프 변수는 요청 간 공유**되어 다른 사용자의 상태가 섞일 수 있다. (Nuxt 공식 문서가 `useState`를 권장하는 이유이기도 함)
- ❌ 실용적으로는 "registry는 클라이언트에서만 의미 있으니 SSR 메모리 공유는 문제 안 됨"이라고 합리화할 수 있지만, **"왜 SSR-safe한 `useState`를 안 쓰지?"** 라는 질문에 매번 답해야 한다는 점에서 코드 컨벤션상 깔끔하지 않다.
- → 채택할 거라면 **명시적인 주석/네이밍**(`_clientOnlyRegistry` 등)이 필수.

### B. Event Bus (mitt 등) 도입

```ts
// utils/refreshBus.ts
import mitt from "mitt";
export const refreshBus = mitt<{ refresh: void; refreshDone: string }>();
```

```ts
// 각 컴포저블에서
onMounted(() => {
  refreshBus.on("refresh", handleRefresh);
});
onBeforeUnmount(() => refreshBus.off("refresh", handleRefresh));
```

- ✅ **함수를 상태에 담을 필요가 없어진다.** 각 컴포넌트는 자기 리스너만 등록하면 됨 → 직렬화 이슈 원천 차단.
- ✅ 컴포넌트 마운트/언마운트 라이프사이클과 자연스럽게 묶여 cleanup이 명확하다.
- ⚠️ `isRefreshing` 같은 "전체 상태"는 별도 관리 필요. (e.g., `refreshDone` 이벤트를 모아 카운트하는 작은 store)
- ⚠️ Event bus는 **흐름이 코드 상에서 추적하기 어려워지는** 단점이 있다. 누가 발행하고 누가 구독하는지 IDE 레퍼런스만으로 보이지 않는다.
- → 작은 규모에선 매우 깔끔하지만, 화면 수가 늘어나면 디버깅 난이도가 올라간다.

### C. Pinia Store + Action 직접 호출

```ts
export const useRefreshStore = defineStore("refresh", () => {
  const handlers = new Map<string, () => Promise<void>>(); // store 인스턴스 내부 Map
  const isRefreshing = ref(false);

  function register(key: string, fn: () => Promise<void>) {
    if (import.meta.server) return;
    handlers.set(key, fn);
  }

  async function refreshAll() {
    isRefreshing.value = true;
    try {
      await Promise.all([...handlers.values()].map((fn) => fn()));
    } finally {
      isRefreshing.value = false;
    }
  }
  return { register, refreshAll, isRefreshing };
});
```

- ✅ Pinia의 `state`만 SSR 직렬화 대상이고, **`setup store` 내부의 일반 변수(`Map`, 클로저 등)는 직렬화 대상이 아니다.** 함수도 store 내부에 안전하게 보관 가능.
- ✅ `isRefreshing`만 reactive state로 두면 자연스럽게 SSR-safe.
- ✅ 의존성과 흐름이 명시적 → 테스트/추적이 쉽다.
- ⚠️ Pinia 의존을 새로 추가해야 한다면 비용. (이미 쓰고 있다면 사실상 베스트)
- → 현재 코드베이스에 Pinia가 이미 있다면 **이 방법이 가장 자연스럽고 안전**해 보인다.

### D. `provide` / `inject` (컴포넌트 트리 의존성 주입)

- 부모(layout 또는 페이지)에서 registry를 만들고 자식에서 `inject` 해서 등록
- ✅ Vue 트리 안에 갇히므로 SSR 페이로드와 무관
- ❌ 트리 외부(plugin, util 등)에서 호출 불가 → 활용 범위가 좁다.

### 📊 비교 정리

| 방식                          | SSR 직렬화 안전        | 흐름 추적 용이성 | 도입 비용          | 추천도          |
| ----------------------------- | ---------------------- | ---------------- | ------------------ | --------------- |
| `useState` + 함수 등록 (현재) | ❌ (가드 필요)         | 보통             | 낮음               | △ (가드로 봉합) |
| 모듈 스코프 `ref`             | ⚠️ (요청 간 공유 주의) | 보통             | 낮음               | △               |
| Event Bus                     | ✅                     | ❌ 약함          | 낮음               | ○ (소규모)      |
| **Pinia Setup Store**         | ✅                     | ✅ 강함          | 낮음(이미 사용 중) | ◎               |
| `provide` / `inject`          | ✅                     | 보통             | 낮음               | △ (범위 제약)   |

> **잠정 결론:** 현재는 가드로 봉합했지만, 후속 리팩토링 시 **Pinia setup store 내부 `Map`으로 핸들러를 보관하고, `isRefreshing`만 reactive state로 노출**하는 형태로 옮기는 것이 가장 깔끔할 듯하다.

---

## 🧠 배운 점

- **SSR과 CSR의 가장 큰 차이는 "상태가 직렬화되어 네트워크를 건너간다"는 점**이다. CSR 기반으로만 작업해온 패턴 — 특히 함수/클래스 인스턴스/Symbol 등을 전역 상태에 거리낌 없이 담는 패턴 — 은 SSR에서는 곧장 폭탄이 된다.
- **`useState`의 정체성**을 다시 확인했다. 단순한 "전역 ref"가 아니라, **요청 단위로 격리되고 SSR → CSR로 페이로드를 통해 전달되는 상태**다. 그래서 직렬화 가능한 값(POJO, 원시값, 배열, 일반 객체)만 담아야 한다.
- **테스트 동선**은 사용자가 페이지에 도달할 수 있는 모든 경로를 포함해야 한다. SPA 내부 네비게이션과 **직접 진입/새로고침**은 SSR 관점에서 완전히 다른 코드 경로다.
- 패턴 도입(여기서는 registry 패턴) 자체가 잘못된 것은 아니지만, **"이 자료구조가 어디서 살아야 하는가 — 서버? 클라이언트? 둘 다?"** 를 먼저 묻고 들어갔으면 더 빨리 도달했을 결론이었다.

---

## 📚 참고 자료

- [Nuxt 3 `useState` 공식 문서](https://nuxt.com/docs/api/composables/use-state) — payload serialization 관련 설명
- [devalue](https://github.com/Rich-Harris/devalue) — Nuxt가 payload 직렬화에 사용하는 라이브러리, 지원/미지원 타입 확인
- [Nuxt `import.meta` 문서](https://nuxt.com/docs/api/advanced/hooks#nuxt-app-hooks-runtime) — `import.meta.server` / `import.meta.client` 가드 패턴
- [Pinia Setup Store](https://pinia.vuejs.org/core-concepts/#setup-stores) — store 내부 비-reactive 변수 활용
