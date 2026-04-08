---
title: 🧩 동시성(Concurrency)
published: 2025-10-26
tags: [React]
category: React
series: React
draft: false
---
## 🧩 동시성(Concurrency)

---

**동시성(Concurrency)**이란 여러 개의 작업을 **동시에 처리하는 것처럼** 보이게 하는 것이다. 

단일 CPU 환경에서 **빠른 전환(Context Switching)**을 통해 여러 작업이 동시에 진행되는 것처럼 보이게 하는 기법으로, CPU가 한순간 하나의 작업만 처리하지만, 빠른 전환 속도로 마치 여러 작업이 겹쳐 실행되는 듯한 효과를 낼 수 있다. 

즉 동시성은 실제로 여러 작업이 동시에 실행되는 것이 아니라, **작업의 “진행 순서를 조율”**함으로써 **동시에 처리되는 듯한 효과**를 만드는 것이다. 따라서 **동시성은 ‘일정한 시간 단위 내에서 여러 일을 조율하는 능력**이라고 볼 수 있다.

### 🆚 병렬성(Parallelism)과의 차이점?

**병렬성**은 **여러 작업을 실제로 동시에 처리**하는 것이다. 

여러 CPU 또는 코어를 사용하여 *(물리적으로)* 여러 작업을 병렬로 처리한다. 각각의 작업은 별도의 프로세스나 스레드에서 실행되며, 이러한 작업들은 각각이 독립적으로 실행되기 때문에 서로 영향을 주지 않는다.

| 구분 | 설명 | 예시 |
| --- | --- | --- |
| 동시성(Concurrency) | 여러 작업을 *동시에 겹쳐서 처리*하되, 실제 CPU는 하나씩 번갈아가며 처리 | 한 사람이 전화, 메일, 채팅 업무를 번갈아 가면서 처리 |
| 병렬성(Parallelism) | 여러 작업을 *동시에 여러 CPU 코어*에서 수행 | 3명이 각각 전화, 메일, 채팅 업무를 받아서 동시에 처리 |

### 🚦 프론트엔드에서 동시성

브라우저는 사용자 상호작용과 다양한 비동기 이벤트를 동시에 다뤄야 한다.

- 사용자의 입력 이벤트 (Click, Typing input values 등)
- 네트워크 통신 (API 요청)
- 애니메이션 렌더링 (CSS, requestAnimationFrame)
- 타이머 (setTimeout, setInterval)
- React의 상태 업데이트

이러한 작업을 동시에 진행하면서 **서로 영향을 주지 않고 매끄럽게 동작**하여 사용자의 UX에 부정적인 영향이 가지 않도록 하게 하는 것이 중요하다. 즉, 단일 스레드 환경인 브라우저에서 비동기 이벤트를 효율적으로 스케줄링하여 “사용자 상호작용(User Interaction)과 렌더링이 경쟁하지 않도록” 해야 한다.

### 📝 동시성이 중요한 몇 가지 사례 (Cases)

### 💬 **검색 자동완성 / 필터링 UI**

- **Situation**:
    - 사용자가 입력할 때마다 사용자의 입력값을 기반으로 API 요청
    - 사용자가 빠르게 타이핑 시 이전 요청에 대한 응답이 돌아오기 전에 새로운 요청 발생
    - 나중에 처리된 이전 요청의 응답이 **새로운 검색 결과를 덮어쓰는 상황 발생**
- **Problem**:
    - UI가 최신 상태를 반영하지 못함
    - 뒤늦은 응답이 최신 입력을 덮어버리는 **Race Condition** 발생
- **How to solve**?
    - `AbortController`로 이전 요청에 대한 중단을 통해 Race Condition 방지
    - Concurrent Rendering: `useTransition`을 사용해 입력(우선순위 High) vs. 렌더링(우선순위 Low) 구분지어 우선순위 기준으로 변경사항 처리

### 📑 **탭 전환 + API 요청**

- **Situation**:
    - 사용자가 탭을 전환하는 순간, 이전 탭의 데이터 요청이 아직 완료되지 않았을 경우
    - 전환 후 이전 탭 전환 시 호출했던 요청에 대한 응답 도착 → 새로운 탭의 UI를 이전 요청에 대한 응답이 덮어씀
- **Problem**:
    - **상태 꼬임 (State inconsistency)**
    - 사용자 경험 저하 (잘못된 데이터 표시 또는 UI 깜빡거림 발생)
- **How to solve?**
    - `AbortController` 또는 React Query의 `cancelQueries()`로 이전 요청 취소
    - React의 Concurrent Rendering 활용
        - `startTransition`을 이용해 전환 렌더링을 낮은 우선순위로 처리

### 📃 **대량 데이터 렌더링 + 사용자 입력**

- **Situation**:
    - 몇 만 개의 데이터를 렌더링 중에 사용자의 입력 발생
    - 렌더링이 끝나기 전까지 사용자의 입력이 UI에 반영되지 않음 (입력이 “먹통”처럼 보임)
- **Problem**:
    - 메인 스레드가 렌더링으로 점유되어 입력 이벤트 처리 지연
- **How to solve?**
    - Virtualization (`react-window`, `react-virtualized`)
    - React Concurrent Mode로 렌더링을 나누어 “프레임 단위”로 분할 처리

### ✏️ **자동 저장(Auto Save) + 사용자 입력**

- **Situation**:
    - 사용자가 입력 중일 때 일정 주기로 자동 저장 API 호출
    - 네트워크가 느릴 경우 이전 입력 값으로 덮어써버림
- **Problem**:
    - 서버와 클라이언트의 상태 불일치 (Stale data)
    - 입력 중단이나 깜빡임 발생
- **How to solve?**
    - 클라이언트에 버전 키(version key) 관리
    - 요청 완료 시점 기준이 아닌 입력 타임스탬프 기준으로 반영
    - `useDeferredValue`로 입력값을 늦게 반영해 충돌 방지

### 🎨 **애니메이션 처리 + API 호출**

- **Situation**:
    - 스크롤 애니메이션, 모션 효과가 있는 상태에서 fetch 요청이 동시에 일어남
    - 렌더링 성능 떨어지면서 프레임 드랍(Frame drop) 발생
- **Problem**:
    - 동시성 부하로 인해 애니메이션이 끊김
- **How to solve?**
    - `requestIdleCallback`으로 idle 타임에 비동기 호출
    - Concurrent Rendering으로 렌더링과 애니메이션 스케줄링 분리

---

## 🧩 React의 Concurrent Rendering 이전에 동시성 제어

React 18 이후부터는 React에서 **React 내부 스케줄러(Fiber Scheduler)**가 렌더링 우선순위를 자동으로 조정하여 동시성 제어를 지원한다. 

그렇다면 React가 이러한 제어를 하기 이전에는 프론트엔드의 사용자 입력 지연 문제, API 경쟁 상태(Race Condition) 등을 어떻게 처리했을까?

### 🧩 수동적인 제어 방식

React 18 이전에는 모든 상태 업데이트와 렌더링이 **동기적(Synchronous)**으로 진행되었다.

즉 입력 이벤트, API 호출, 렌더링이 모두 같은 우선순위로 실행되었기 때문에 사용자 입력 중에도 UI가 버벅이거나, 이전 요청이 나중에 도착하면서 최신 상태를 덮어쓰는 등의 문제가 발생했다. 이러한 문제 해결을 위해서 `debounce`, `throttle`, `request cancellation` 등 **수동적인 제어 방식**을 활용했다.

### ⚙️ Throttle and Debounce

`debounce` 함수 구현 예시

→ 사용자가 입력을 멈추고 일정 시간(delay) 동안 아무 입력이 없을 때만 요청을 보냄

```jsx
function debounce(fn, delay) {
  let timer;
  return (...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  };
}

// 검색 API
const handleSearch = debounce((value) => {
  fetch(`/api/search?q=${value}`);
}, 500);

<input onChange={(e) => handleSearch(e.target.value)} />
```

`throttle` 함수 구현 예시

→ 200ms 단위로 스크롤 이벤트가 실행하도록 제한

```jsx
function throttle(fn, limit) {
  let inThrottle;
  return (...args) => {
    if (!inThrottle) {
      fn(...args);
      inThrottle = true;
      setTimeout(() => (inThrottle = false), limit);
    }
  };
}

// 스크롤 이벤트
window.addEventListener(
  'scroll',
  throttle(() => console.log(window.scrollY), 200)
);
```

| 함수 | 작동 방식 | 대표 사례 | 함수의 실행 목적 |
| --- | --- | --- | --- |
| debounce | 마지막 이벤트 이후 일정 시간 동안 추가 이벤트 진행이 없을 때 실행 | 검색 자동완성 | “마지막 입력만 반영” → 불필요한 API 호출 방지 |
| throttle | 일정 시간 간격마다 한 번만 실행 | 스크롤/resize 이벤트 | “일정 시간을 기준으로 실행” → CPU 부하 완화 |

### ⚙️ AbortController (Request Cancellation - 요청 취소)

`fetch` 요청을 보낼 때 이전 요청이 완료되기 전에 새로운 요청이 발생하면, 이전 요청의 응답이 나중에 도착하면서 UI가 이전 상태로 되돌아가는 문제가 발생했다.

이런 문제를 해결하기 위해 수동으로 요청 제어를 할 수 있는 `AbortController`를 사용해 이전 요청을 취소하는 방식이 널리 사용되었다.

```jsx
function useSearch() {
  const controllerRef = useRef();

  const fetchSearchResults = async (query) => {
	  // 진행되는 요청 있으면 취소 (abort)
    if (controllerRef.current) controllerRef.current.abort();
    const controller = new AbortController();
    controllerRef.current = controller;

    try {
      const res = await fetch(`/api/search?q=${query}`, {
        signal: controller.signal, // abort controller를 통한 호출 관리를 위한 signal
      });
      const data = await res.json();
      console.log('결과:', data);
    } catch (error) {
      if (error.name === 'AbortError') {
        console.log('요청이 취소되었습니다.');
      }
    }
  };

  return { fetchSearchResults };
}
```

### 🔥 수동적인 제어 방식이 갖는 한계점

- UX 불안정성
    - 사용자의 입력 타이밍에 따라 반응이 지연되거나, 반영이 느려 보일 수 있음
    - 사용자의 입력 타이밍을 추측하여 그에 맞는 실행 시간에 대한 별도 제어가 필요함
- 렌더링과의 분리 불가
    - 이벤트 발생 빈도 제어는 가능하지만, 렌더링 자체의 우선순위 조절은 불가능
- React 내부 상태 업데이트와 별개
    - React 렌더링 사이클과는 무관하게 동작하여 상태 불일치(stale state) 가능
- 코드 중복 / 복잡도 증가
    - debounce, throttle, abort controller 등을 매 함수마다 따로 관리 필요

---

## 🧩 React 18 이후의 변화 - Concurrent Rendering

### ❓ Concurrent Rendering이란?

**Concurrent Rendering**은 React 18 버전에서 처음 도입되어, React 19에서도 지속적으로 개선되고 있는 **동시적 렌더링(Concurrent Rendering)** 기능이다. 

기존의 동기적(Synchronous)으로 진행되었던 렌더링 작업을 작은 단위로 분할하여 **우선순위 기반으로 스케줄링**할 수 있도록 하며, **필요 시 중단 및 재개**가 가능하도록 하여 **UI의 반응성(Responsiveness)**을 유지하는 **동시적 렌더링 방식**이다.

**Concurrent Rendering의 특징**

- **변경 우선순위에 따른 렌더링 (Prioritized Updates)**: React는 **Fiber 아키텍처**를 기반으로 **변경 우선순위**에 따라 Fiber 노드를 스케줄링한다. 예를 들어, 사용자 입력과 같은 ‘긴급한 업데이트(high priority)’는 즉시 처리하고, 데이터 fetch나 렌더링과 같은 ‘비긴급 업데이트(low priority)’는 나중에 처리하여 빠르게 반영되어야 하는 변경사항을 우선적으로 처리할 수 있다.
- **논 블로킹 (Non-Blocking)**: 렌더링이 메인 스레드를 완전히 점유하지 않고, **일정 단위마다 중단(yield)**하여 이벤트나 사용자 입력을 처리할 수 있는 시간을 확보한다. 따라서 작업량이 많고 복잡한 업데이트를 진행하더라도 애플리케이션은 여전히 끊김 없이 사용자가 상호작용 할 수 있도록 해 UX를 저해하지 않을 수 있다.
- **세밀한 제어 (Fine-Grained Control)**: React는 Fiber 단위로 각 업데이트의 **우선순위를 계산해서 자동 조정**한다. 또한 개발자는 `useTransition`, `useDeferredValue`, `Suspense` 등을 통해 간접적으로 **우선순위 힌트**를 제공할 수 있다. 이를 통해 개발자는 렌더링 흐름을 직접 제어하지 않고도 UI의 반응성과 렌더링 효율을 균형 있게 관리할 수 있다.

✅ Concurrent Rendering은 React가 렌더링 과정을 **동기적(Synchronous)**에서 **스케줄링 가능한 비동기적(Asynchronous) 과정**으로 전환함으로써, 복잡한 UI에서도 **입력 반응성(Responsiveness)**과 **렌더링 유연성(Flexibility)**을 확보하여 끊김 없는 사용자 경험(Seamless UX)를 제공할 수 있는 핵심 기술이다.

---

### 📚 참고자료

- https://yozm.wishket.com/magazine/detail/2996/
- https://www.linkedin.com/pulse/react-19-concurrent-rendering-boosting-performance-modern-machado-m707f/