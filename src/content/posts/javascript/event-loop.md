---
title: 이벤트 루프 (Event Loop)
published: 2025-08-12
tags: [Javascript]
category: Javascript
series: Javascript
draft: false
---
## 🧩 JavaScript의 비동기 처리 개요

---

자바스크립트는 **싱글 스레드** 언어이다.

즉 한 번에 하나의 작업만 수행할 수 있으며, 코드 실행을 담당하는 **호출 스택(Call Stack)**도 하나뿐이다.

싱글 스레드 구조는 복잡한 처리가 필요한 **동시성 문제를 줄이는 장점**이 있지만, 네트워크 요청·파일 읽기처럼 오래 걸리는 작업이 있을 때 전체 실행 흐름을 멈추는 **블로킹(blocking)**이 발생할 수 있다. 

이를 보완하기 위해 **비동기 처리**와 **이벤트 루프(Event Loop)** 개념이 도입되었다.

- ❓ JavaScript는 왜 싱글 스레드로 설계되었을까?
    
    자바스크립트가 처음 등장했을 때는 멀티 스레드에 대한 개념이 대중화된 시기가 아니었다. 또한 자바스크립트의 전신인 LiveScript는 브라우저에서 간단한 UI 조작을 위한 스크립트로 사용되었다.
    
    따라서 복잡한 멀티 스레드 제어보다는 **“Run-to-Completion”** — 하나의 작업이 끝난 뒤 “동기적”으로 다음 작업을 실행하는 순차적 처리 방식 — 이 합리적인 선택이었다.
    

## 🔄 이벤트 루프란?

---

이벤트 루프는 **ECMAScript** **표준이 아닌, 브라우저/Node.js 런타임이 구현한 매커니즘**이다. 

자바스크립트 엔진은 **동기 실행**만 담당하고, 비동기 처리는 **브라우저 Web API** 또는 **Node.js의** c++ 라이브러리인 **libuv**가 담당한다.

![https://dev.to/gautam_kumar_d3daad738680/understanding-call-stack-callback-queue-event-loop-and-microtask-queue-in-javascript-2c7n](@/assets/posts/event-loop-gautam-kumar.png)

https://dev.to/gautam_kumar_d3daad738680/understanding-call-stack-callback-queue-event-loop-and-microtask-queue-in-javascript-2c7n

✅ **이벤트 루프의 핵심 역할**

1. **호출 스택**을 감시
2. 스택이 비어 있으면 **큐(Event Queue, Callback Queue)**에 있는 작업을 꺼내 실행

이처럼 이벤트 루프는 메인 스레드와 백그라운드 사이에서 **신호등**처럼 타이밍을 조율하는 역할을 한다. 

### 📦 호출 스택과 이벤트 루프

호출 스택(Call Stack)은 자바스크립트에서 **현재 실행 중이거나 대기 중인 함수**를 LIFO(Last In First Out) 형태로 순차적으로 쌓아두는 스택이다.

다음과 같이 실행되는 코드가 있다고 가정했을 때 이 코드들은 어떤 순서로 호출 스택에 쌓이게 되는 걸까?

```tsx
function a() {
	console.log("a");
}

function b() {
	console.log("b");
}

function ab() {
	console.log("ab");
	a();
	b();
}

ab();
```

[call stack1.mp4](@/assets/posts/call-stack1.mp4)

- **실행 흐름 요약**
    1. ab()가 호출 스택에 먼저 들어간다.
    2. ab() 내부에 console.log가 존재하므로 호출 스택에 들어가 실행이 완료된 후 제거된다. (아직 ab()는 존재)
    3. a()가 호출 스택에 들어간다. 
    4. a() 내부에 console.log가 존재하므로 호출 스택에 들어가 실행이 완료된 후 제거된다. (아직 ab(), a()는 존재)
    5. a()에 더 이상 남은 것이 없으므로 호출 스택에서 제거된다. (아직 ab()는 존재)
    6. b()가 호출 스택에 들어간다.
    7. b() 내부에 console.log가 존재하므로 호출 스택에 들어가 실행이 완료된 후 제거된다. (아직 ab(), b()는 존재)
    8. b()에 더 이상 남은 것이 없으므로 호출 스택에서 제거된다. (아직 ab()는 존재)
    9. ab()에 더 이상 남은 것이 없으므로 호출 스택에서 제거된다.
    10. 이제 호출 스택이 완전히 비워졌다.

이러한 일련의 과정을 거치며 처리해야 할 작업들이 순차적으로 쌓이고 제거되는 과정을 확인할 수 있다. 

그렇다면 비동기 작업의 실행은 어떤 식으로 처리될까?

### ⏰ 비동기 작업과 이벤트 루프

```tsx
function a() {
	console.log("a");
}

function b() {
	console.log("b");
}

function ab() {
	console.log("ab");
	setTimeout(a, 0); // setTimeout을 추가
	b();
}

ab();
```

비동기 작업인 `setTimeout`을 추가했을 때, 호출 스택 내부에서는 다음과 같은 일이 발생한다.

[call stack 2.mp4](@/assets/posts/call-stack2.mp4)

- **실행 흐름 요약**
    1. ab()가 호출 스택에 먼저 들어간다.
    2. ab() 내부에 console.log가 존재하므로 호출 스택에 들어가 실행이 완료된 후 제거된다. (아직 ab()는 존재)
    3. setTimeout(a, 0)이 호출 스택에 들어간다.
    4. 런타임으로 타이머 요청이 전달되고, setTimeout은 스택에서 제거된다.
        1. 메인 스레드(자바스크립트 엔진)는 **이 콜백을 delay후에 실행해줘**라는 요청을 런타임 환경에 전달한다.
        2. 런타임 환경은 백그라운드 스레드에서 타이머를 관리한다.
        3. 지정한 시간이 지나면, 해당 콜백(`a`)을 **태스크 큐**에 등록한다.
        4. 이 과정은 **메인 스레드와 병렬로 진행**된다.
    5. b()가 호출 스택에 들어간다.
    6. b() 내부에 console.log가 존재하므로 호출 스택에 들어가 실행이 완료된 후 제거된다. (아직 ab(), b()는 존재)
    7. b()에 더 이상 남은 것이 없으므로 호출 스택에서 제거된다. (아직 ab()는 존재)
    8. ab()에 더 이상 남은 것이 없으므로 호출 스택에서 제거된다.
    9. 이벤트 루프는 호출 스택이 비워져 있는 것을 확인하고, 태스크 큐에서 a()를 꺼내 호출 스택에 넣는다.
    10. a() 내부에 console.log가 존재하므로 호출 스택에 들어가 실행이 완료된 후 제거된다.
    11. a()에 더 이상 남은 것이 없으므로 호출 스택에서 제거된다.
- 💡 **setTimeout의 실행 흐름**
    
    현재 실행 흐름을 보면 `setTimeout(() => {}, delay)`이 delay만큼 설정된 지연 시간 이후 바로 실행됨을 보장하지 않는다는 것을 알 수 있다.
    
    - `setTimeout`과 같은 비동기 함수는 자바스크립트 엔진이 아닌 **런타임(브라우저의 Web API/Node.js의 libuv)**에서 처리된다.
    - 런타임의 백그라운드 스레드가 타이머를 관리하고, 완료 시 해당 콜백을 **태스크 큐(Task Queue/Callback Queue)**에 등록한다.
    - 이벤트 루프는 호출 스택이 완전히 비워진 뒤, **다음 이벤트 루프 사이클**에서 큐에 있는 작업을 꺼내 실행한다.
        
        → 따라서 최소 지연 시간이 0ms여도, 메인 스레드가 바쁘거나 다른 작업이 대기 중이면 실행이 지연될 수 있다.
        

🧩 **태스크 큐(Task/Callback Queue)**

새롭게 등장한 개념인 태스크 큐는 **실행해야 할 태스크**를 보관하고 있는 공간이다. 이때 실행해야 할 태스크란 런타임(Web API, libuv)이 처리하는 비동기 함수의 **콜백 함수**나 **이벤트 핸들러** 등을 의미한다. 

이벤트 루프는 호출 스택이 비어있음을 확인하면 가장 오래된 태스크부터 꺼내 호출 스택으로 순차적으로 보내어 실행한다.

✅ **주요 태스크**

- `setTimeout`, `setInterval`
- 브라우저 **DOM 이벤트** 콜백
- `postMessage`, `MessageChannel`
- Node.js: `setImmediate` (루프의 check phase)

이 과정에서 이벤트 루프는 호출 스택이 비고, 콜백이 실행 가능한 시점이 되면 해당 콜백을 호출 스택으로 전달하는 역할을 한다. 

### 🧩 태스크 큐와 마이크로 태스크 큐

```tsx
function a() {
	console.log("a");
}

function b() {
	console.log("b");
}

function c() {
	console.log("c");
}

function foo() {
	console.log("foo");
}

foo(); 

setTimeout(a, 0); // setTimeout - 태스크 큐

Promise.resolve().then(b).then(c); // 마이크로태스크 큐
```

다음과 같은 코드가 있다면 실행 순서는 어떻게 될까?

![image.png](@/assets/posts/call-stack1.png)

`foo -> a -> b -> c` 순으로 실행될 거라 예상했지만 Promise와 관련된 콜백이 먼저 실행됨을 알 수 있다. 

![event-loop-webapi.gif](@/assets/posts/event-loop-webapi.gif)

이러한 순서로 실행된 이유는 Promise에 대한 콜백은 태크스 큐(콜백 큐)가 아닌 **마이크로 태스크 큐(Microtask queue)**에 등록되기 때문이다.

- **실행 흐름 요약**
    1. foo() 호출 스택에 먼저 들어간다.
    2. foo() 내부에 console.log가 존재하므로 호출 스택에 들어가 실행이 완료된 후 제거된다. (아직 foo()는 존재)
    3. foo()에 더 이상 남은 것이 없으므로 호출 스택에서 제거된다.
    4. setTimeout(a, 0)이 호출 스택에 들어간다.
    5. setTimeout은 Web API가 타이머 관리를 하므로 호출 스택에서 제거된다.
        1. Web API가 타이머 관리 → 만료 후에 태스크 큐에 a가 등록된다.
    6. Promise.resolve()가 호출 스택에 들어간다
    7. Promise는 Web API가 관리하므로 호출 스택에서 제거된다.
        1. Promise.resolve().then(b) 평가 → 마이크로태스크 큐에 b를 등록한다.
    8. 호출 스택이 비었음을 이벤트 루프가 확인 후 **마이크로태스크 큐부터 먼저 비우기 시작한다.**
        1. b()를 실행한다 → console.log가 존재하므로 호출 스택에 들어가 b 출력 후 제거된다. → then(c)가 마이크로 테스크로 등록된다.
        2. c()를 실행한다. → console.log가 존재하므로 c를 출력 후 제거된다.
    9. 호출 스택이 비었음을 이벤트 루프가 확인 후 태스크 큐에서 a를 가져와 실행한다. console.log가 존재하므로 호출 스택에 넣고 a 출력 후 제거된다.
    10. a()에 더 이상 남은 것이 없으므로 호출 스택에서 제거된다.

💡**마이크로 태스크 큐(Microtask Queue)**

**마이크로 태스크 큐(Microtask Queue)**는 콜백 큐보다 **더 높은 우선순위**를 가지고 있는 큐다. 콜백 큐에 있는 어떤 태스크들보다도 우선적으로 실행된다. 즉 마이크로 태스크 큐가 빌 때까지 그 어떤 콜백 큐 내의 태스크들도 실행될 수 없다. 

마이크로 태스크 큐에 등록되는 태스크들은 대표적으로 **Promise 콜백**과 **Mutation Observer 콜백**이 있다.

**태스크 큐와 마이크로 태스크 큐 비교**

| **구분** | **Task Queue** | **Microtask Queue** |
| --- | --- | --- |
| **콜백** | `setTimeout`, `setInterval`
브라우저 **DOM 이벤트** 콜백
`postMessage`, `MessageChannel`
Node.js: `setImmediate`  | `Promise callbacks` (then, catch, finally)
`MutationObserver callbacks`
`queueMicrotask`
Node.js: `process.nextTick`
(* nextTick은 Node에서 마이크로태스크보다 더 먼저 실행되는 별도 큐이다.) |
| **우선순위** | 낮음 | 높음 |
| **처리 시점** | 호출 스택, 마이크로 태스크를 모두 비운 뒤 처리 |  |

### 🎨 마이크로 태스크 큐와 렌더링

- 브라우저는 보통 마이크로태스크 처리 이후 렌더링(페인트)을 시도한다.
- 마이크로태스크를 과도하게 연쇄 등록하면 렌더링이 지연(Starvation) 될 수 있다.
- 프레임 단위 작업이 필요한 경우에는 `requestAnimationFrame`을 고려할 수 있다.

---

### 📚 참고자료

- 모던 자바스크립트 Deep Dive
- [Inpa Dev: 자바스크립트 이벤트 루프 동작 원리](https://inpa.tistory.com/entry/%F0%9F%94%84-%EC%9E%90%EB%B0%94%EC%8A%A4%ED%81%AC%EB%A6%BD%ED%8A%B8-%EC%9D%B4%EB%B2%A4%ED%8A%B8-%EB%A3%A8%ED%94%84-%EA%B5%AC%EC%A1%B0-%EB%8F%99%EC%9E%91-%EC%9B%90%EB%A6%AC#:~:text=%EC%9D%B4%EB%B2%A4%ED%8A%B8%20%EB%A3%A8%ED%94%84%EB%8A%94%20%EB%B8%8C%EB%9D%BC%EC%9A%B0%EC%A0%80%20%EB%8F%99%EC%9E%91%EC%9D%84%20%EC%A0%9C%EC%96%B4%ED%95%98%EB%8A%94%20%EA%B4%80%EB%A6%AC%EC%9E%90,-%EC%8B%B1%EA%B8%80%20%EC%8A%A4%EB%A0%88%EB%93%9C%EC%9D%B8&text=%EC%9D%B4%EB%B2%A4%ED%8A%B8%20%EB%A3%A8%ED%94%84%EB%8A%94%20%EB%B8%8C%EB%9D%BC%EC%9A%B0%EC%A0%80%20%EB%82%B4%EB%B6%80,%EC%9D%84%20%EC%A0%9C%EC%96%B4%ED%95%98%EB%8A%94%20%EB%85%80%EC%84%9D%EC%9D%B4%EB%8B%A4)
- [Understanding Call Stack, Callback Queue, Event Loop, and Microtask Queue in JavaScript](https://dev.to/gautam_kumar_d3daad738680/understanding-call-stack-callback-queue-event-loop-and-microtask-queue-in-javascript-2c7n)
- [loupe - 콜 스택, 이벤트 루프 시각화](http://latentflip.com/loupe/?code=JC5vbignYnV0dG9uJywgJ2NsaWNrJywgZnVuY3Rpb24gb25DbGljaygpIHsKICAgIHNldFRpbWVvdXQoZnVuY3Rpb24gdGltZXIoKSB7CiAgICAgICAgY29uc29sZS5sb2coJ1lvdSBjbGlja2VkIHRoZSBidXR0b24hJyk7ICAgIAogICAgfSwgMjAwMCk7Cn0pOwoKY29uc29sZS5sb2coIkhpISIpOwoKc2V0VGltZW91dChmdW5jdGlvbiB0aW1lb3V0KCkgewogICAgY29uc29sZS5sb2coIkNsaWNrIHRoZSBidXR0b24hIik7Cn0sIDUwMDApOwoKY29uc29sZS5sb2coIldlbGNvbWUgdG8gbG91cGUuIik7!!!PGJ1dHRvbj5DbGljayBtZSE8L2J1dHRvbj4%3D)