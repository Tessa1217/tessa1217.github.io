---
title: 🌳 Virtual DOM
published: 2026-08-04
tags: [React, Javascript, Study]
category: React
series: Study
image: /images/frontend-study.webp
draft: false
---
# 🌳 Virtual DOM

> **TL;DR** — 가상 DOM은 실제 DOM 조작 비용을 줄이기 위해 리액트가 도입한 추상화다. 매번 트리 전체를 비교하는 대신 **"타입이 다르면 새로 마운트", "key로 자식을 식별"**이라는 두 가지 휴리스틱으로 `O(n)` 복잡도의 재조정(reconciliation)을 수행하며, 이 작업은 중단·재개가 가능한 **Fiber** 아키텍처 위에서 실행된다. 여기에 **React Compiler**를 더하면 `memo`/`useMemo`/`useCallback` 없이도 불필요한 리렌더링을 컴파일 타임에 자동으로 제거할 수 있다.
>
> 테자스 쿠마르(Tejas Kumar), 『전문가를 위한 리액트(Fluent React)』(O'Reilly) 3장 "가상 DOM"을 읽고 정리한 스터디 노트다.

---

## 목차

1. [Virtual DOM이란](#1-virtual-dom이란)
2. [실제 DOM이란](#2-실제-dom이란)
3. [문서 조각(Document Fragment)과 가상 DOM](#3-문서-조각-document-fragment과-가상-dom)
4. [Virtual DOM 작동 방식](#4-virtual-dom-작동-방식)
5. [React Compiler를 통한 자동 최적화](#5-react-compiler를-통한-자동-최적화)
6. [정리](#6-정리)
7. [참고자료](#-참고자료)

---

## 1. Virtual DOM이란
DOM이란 문서 객체 모델(Document Object Model)로 브라우저 런타임이 HTML 문서를 표현하는 모델을 의미한다. 실제 DOM은 노드 객체로 구성되는데, 가상 DOM은 그 노드 트리를 설명하기 위한 **평범한 자바스크립트 객체**로 구성된다는 점에서 차이가 있다.

리액트는 이러한 Virtual DOM을 활용해, `setState` 등으로 UI 변경사항을 반영해야 할 때 실제 DOM을 바로 조작하지 않는다. 대신 가상 DOM 트리를 먼저 새로 만들고, 실제 DOM에는 그 변경사항 중 필요한 부분만 골라 반영하는 과정을 거치는데 이를 **재조정(Reconciliation)**이라고 한다. 재조정이 정확히 왜 필요한지 이해하려면 먼저 실제 DOM을 직접 조작할 때 어떤 비용이 발생하는지 살펴볼 필요가 있다.

## 2. 실제 DOM이란
웹 브라우저는 HTML 페이지를 읽어들여 구문 분석을 통해 노드와 객체의 트리를 만드는데, 이렇게 변환된 객체 모델의 결과물을 DOM이라고 한다. DOM은 웹 페이지의 현재 상태를 표현한 거대한 자바스크립트 객체로, 사용자가 페이지와 상호작용할 때마다 계속 업데이트된다. 다만 실제 DOM에 직접 업데이트를 할 경우 여러 가지 문제점이 발생하는데, **성능**, **브라우저 간 호환성**, **보안 취약성** 등이 대표적인 문제로 대두된다.

### ⚡ 2.1 실제 DOM 조작과 성능
실제 DOM에 엘리먼트 추가, 제거, 속성 업데이트 등을 하게 되면 브라우저는 레이아웃을 다시 계산(**리플로우**, reflow)하고 페이지의 영향을 받는 부분을 다시 그리는(**리페인트**, repaint) 과정을 거치게 된다. 특히 크고 복잡한 웹 페이지라면 이 과정 자체만으로 많은 리소스를 사용하게 된다.

예를 들어 `offsetWidth` 속성을 읽는 것은 간단한 작업처럼 보이지만, 실제로는 브라우저가 레이아웃을 강제로 다시 계산하게 만드는 비용을 야기한다. 이렇게 레이아웃 속성을 읽고 쓰는 작업이 번갈아가며 반복적으로 발생해 불필요한 레이아웃 재계산이 누적되는 현상을 **레이아웃 스레싱(layout thrashing)**이라고 한다.

```html
<!DOCTYPE html>
<html>
    <head>
        <title>성능 예시</title>
    </head>
    <body>
        <ul id="list">
            <li>목록 항목 1</li>
            <li>목록 항목 2</li>
            <li>목록 항목 3</li>
        </ul>
    </body>
</html>
```

자바스크립트를 통해 목록에 새 항목을 추가한다고 가정한 스크립트를 작성하면 다음과 같을 것이다.

```javascript
const list = document.getElementById('list');
const newItem = document.createElement('li');
newItem.textContent = '목록 항목 4';
list.appendChild(newItem);
```

해당 스크립트를 실행하면 브라우저는 레이아웃을 다시 계산하고, 페이지에서 영향을 받는 부분을 다시 페인트해서 새 항목을 표시한다. 만약 목록의 크기가 크다면 이 작업은 더욱 많은 리소스를 사용하며 성능 저하를 야기할 것이다.

읽기와 쓰기 작업을 종류별로 나눠 일괄 처리하면, 브라우저가 레이아웃을 강제로 다시 계산하는 횟수를 최소화하고 한 번의 작업으로 필요한 정보를 가져오는 처리를 통해 이러한 성능 저하를 최소화할 수 있다. 가상 DOM은 성능 최적화를 위해 **변경이 필요한 부분만 일괄로 업데이트**할 수 있는 구조를 제공함으로써, 실제 DOM을 직접 업데이트하는 것보다 훨씬 가벼운 UI 표현을 가능하게 하고 쾌적한 사용자 경험을 제공할 수 있다.

### 🌍 2.2 브라우저 간 호환성
실제 DOM을 조작할 경우 브라우저별로 특정 DOM 엘리먼트와 속성을 지원하지 않는 경우가 있을 수 있다. 개발자는 브라우저 간 호환성을 위해 실제 비즈니스 로직이 아닌, 브라우저 간의 차이점을 분석하고 이를 흡수하기 위한 추가적인 시간과 노력을 들여야 한다.

리액트는 **합성 이벤트 시스템(Synthetic event system)**을 통해 이러한 브라우저 호환성 문제를 해결하고자 했다. `SyntheticEvent`는 브라우저의 기본 이벤트를 둘러싼 래퍼 객체로, 여러 브라우저에서 일관성을 보장하기 위해 설계되었다. `SyntheticEvent`라는 추상화된 통합 인터페이스를 제공함으로써, 개발자가 특정 브라우저에 맞춘 개별 코드를 작성하지 않고도 이벤트와 상호작용하는 일관된 방법을 제공하는 것이다.

`onChange` 이벤트를 예시로 들자면:
- `<input type="text">`의 경우 일부 브라우저에서 `onChange` 이벤트는 값이 변경되는 즉시 발생하지 않고, 입력이 포커스를 잃은 경우에만 발생한다.
- `<select>`의 경우에는 브라우저에 따라 현재 선택된 옵션을 다시 선택할 때에도 이벤트가 발생한다.
- 구형 브라우저는 특정 폼 엘리먼트에서 `onChange` 이벤트 자체가 발생하지 않는 경우도 있다.

이처럼 동일한 이벤트라도 브라우저마다 처리 방식이 달라지는 문제를 해결하기 위해, 리액트의 `SyntheticEvent`는 **이벤트의 동작을 정규화**하여 제공한다. 이를 통해 네이티브 브라우저 이벤트의 결점과 비일관성을 보완하고, 개발자가 UI 개발 그 자체에 집중할 수 있는 환경을 제공한다.

이처럼 실제 DOM을 직접 다룰 때 발생하는 성능 비용과 호환성 문제는, 브라우저 자체에 오래전부터 존재해온 문서 조각(Document Fragment)이라는 개념으로도 부분적으로 완화할 수 있다. 리액트의 가상 DOM은 이 아이디어를 한 단계 더 발전시킨 결과물이라고 볼 수 있다.

## 3. 문서 조각 (Document Fragment)과 가상 DOM
**문서 조각**은 DOM 노드를 담아둘 수 있는 가벼운 컨테이너다. 문서 조각 자체는 활성화된 문서 트리에 속하지 않기 때문에, 그 안에서 노드를 추가하거나 옮기는 작업은 실제 화면에 아무런 영향을 주지 않는다. 예를 들어 1,000개의 목록 항목을 하나씩 실제 DOM에 추가하면 리플로우가 최대 1,000번 발생할 수 있지만, 문서 조각에 모아뒀다가 한 번에 붙이면 **리플로우는 단 한 번만 발생**한다.

```javascript
const fragment = document.createDocumentFragment();

for (let i = 1; i <= 1000; i++) {
    const li = document.createElement('li');
    li.textContent = `목록 항목 ${i}`;
    fragment.appendChild(li); // 아직 실제 DOM에는 반영되지 않는다
}

// 여기서 단 한 번의 append로 1,000개의 노드가 일괄 반영된다
document.getElementById('list').appendChild(fragment);
```

문서 조각의 업데이트 방식은 가상 DOM과 다음 특징 면에서 매우 유사하다.
- **일괄 업데이트**: 문서의 실제 DOM을 여러 번 개별적으로 업데이트하는 것이 아니라, 문서 조각 내의 모든 변경 사항을 일괄적으로 처리할 수 있다.
- **메모리 효율성**: 문서 조각에 추가된 노드는 실제 문서 DOM에서 분리되어 있으므로, 큰 영역을 재정렬할 때 메모리 사용량을 최적화할 수 있다.
- **중복 렌더링 방지**: 문서 조각은 활성화된 문서 DOM 트리에 속하지 않기 때문에, 문서 조각을 변경하는 것은 실제 문서에 영향을 주지 않는다. 이를 통해 최종적으로 실제 DOM에 반영되기 전까지 스타일 재계산과 스크립트 실행의 중복 수행을 방지할 수 있다.

리액트의 가상 DOM은 이러한 문서 조각의 개념을 보다 나은 방식으로 구현한 것이라고도 볼 수 있다.
- **일괄 업데이트**: 가상 DOM은 문서 조각과 유사하게 여러 변경 사항을 한꺼번에 일괄 처리한다.
- **효율적인 비교 알고리즘**: 가상 DOM과 실제 DOM의 차이점을 확인할 때, 문서 조각에는 없는 효율적인 diff 알고리즘을 활용해 변경 사항을 확인한다.
- **단일 렌더링**: 차이점 식별 후 리액트는 한 번의 일괄 처리를 통해 실제 DOM을 업데이트한다. 이를 통해 비용이 많이 드는 리플로우와 리페인팅을 최소화한다.

즉 문서 조각이 "변경 사항을 모아뒀다가 한 번에 반영"하는 수준이라면, 가상 DOM은 여기에 **"무엇이 바뀌었는지 스스로 계산"하는 비교 알고리즘**까지 더한 개념이라고 볼 수 있다. 이제 이 비교 알고리즘을 포함해 가상 DOM이 실제로 어떻게 동작하는지 살펴본다.

## 4. Virtual DOM 작동 방식

### 4-1. 리액트 엘리먼트
리액트는 컴포넌트 또는 HTML 엘리먼트의 가벼운 형태인 **리액트 엘리먼트**로 트리를 표현한다.

```javascript
const element = React.createElement(
    "div",
    { className: "my-class" },
    "Hello, world!"
)
```

실무에서는 `React.createElement`를 직접 호출하는 대신 JSX 문법을 사용하는 경우가 대부분이지만, JSX는 빌드 타임에 결국 `React.createElement` 호출(또는 React 17 이후 자동 런타임에서는 `jsx`/`jsxs` 호출)로 변환된다. 위 코드를 JSX로 작성하면 다음과 같다.

```jsx
const element = <div className="my-class">Hello, world!</div>;
```

두 코드는 완전히 동일한 리액트 엘리먼트 객체를 생성한다. `React.createElement` 함수를 사용해 생성된 엘리먼트는 다음과 같은 필드를 갖는다.

- `$$typeof`
- `type`
- `ref`
- `props`
- `_owner`
- `_store`

```javascript
{
    $$typeof: Symbol(react.element),
    type: "div",
    key: null,
    ref: null,
    props: {
        className: "my-class",
        children: "Hello, world!"
    },
    _owner: null,
    _store: {}
}
```

### 4-2. Virtual DOM과 실제 DOM 비교
리액트는 컴포넌트가 렌더링되면 새 가상 DOM 트리를 생성하고, 이전 가상 DOM 트리와 비교한 다음, 이전 트리를 새 트리와 일치하도록 업데이트하는 데 필요한 **최소한의 변경 사항**을 계산한다. 이러한 과정을 **재조정 프로세스(Reconciliation process)**라고 한다.

가상 DOM 트리를 간단하게 도식화한다면 다음과 같을 것이다.

```jsx
function App() {
    const [count, setCount] = React.useState(0);

    return React.createElement(
        "div",
        null,
        React.createElement("h1", null, "카운트: ", count),
        React.createElement(
            "button",
            { onClick: () => setCount(count + 1) },
            "증가"
        )
    )
}
```

```text
div
  h1
    "카운트: 0"
  button
    '증가'
```

버튼을 클릭하면 리액트는 다음과 같이 새로운 가상 DOM 트리를 생성한다.

```text
div
  h1
    "카운트: 1"
  button
    '증가'
```

리액트는 두 트리를 비교해 `<h1>` 엘리먼트의 텍스트 내용만 업데이트하면 된다고 계산하고, 실제 DOM에는 해당 부분만 반영한다. `<div>`와 `<button>`은 이전 렌더링에서 사용했던 실제 DOM 노드를 그대로 재사용한다. 그렇다면 리액트는 **"h1의 텍스트만 바뀌었다"는 결론을 어떤 방식으로, 얼마나 빠르게 계산해내는 것일까?**

### 🧩 4-3. 재조정 알고리즘과 두 가지 휴리스틱
리액트에서 새 트리와 이전 트리를 노드별로 비교해 트리의 어느 부분이 변경되었는지 알아내는 작업을 **디핑(diffing)**이라고 한다. 일반적으로 두 트리 사이의 최소 편집 거리(edit distance)를 정확하게 구하는 트리 편집 알고리즘은 노드 개수 n에 대해 `O(n³)`의 시간 복잡도를 가진다. UI 트리는 노드 수가 수백~수천 개에 달할 수 있으므로, 매 렌더링마다 이 비용을 그대로 감수하는 것은 현실적이지 않다.

이 문제를 해결하기 위해 리액트는 실용적인 **두 가지 가정(휴리스틱)**을 전제로 한 `O(n)` 복잡도의 비교 알고리즘을 사용한다.

**가정 1. 서로 다른 타입의 두 엘리먼트는 서로 다른 트리를 만들어낸다.**

엘리먼트의 타입이 바뀌면 리액트는 그 내부를 재귀적으로 비교하지 않는다. 대신 이전 트리(그 자손 전체를 포함해서)를 곧바로 버리고, 새로운 트리를 처음부터 다시 마운트한다.

```jsx
// 이전 렌더링
<div>
  <Counter />
</div>

// 이후 렌더링 - div가 span으로 바뀌었다
<span>
  <Counter />
</span>
```

이렇게 최상위 엘리먼트의 타입이 `div`에서 `span`으로 바뀌면, 그 안에 있는 `Counter`가 실제로는 그대로 유지되더라도 리액트는 `Counter`를 포함한 서브트리 전체를 **언마운트했다가 다시 마운트**한다. 즉 `Counter` 내부에 `useState`로 저장해 둔 상태는 타입이 바뀌는 순간 **초기화되어 사라진다**. 타입이 다른 엘리먼트가 비슷한 트리를 만들어낼 가능성은 거의 없다는 전제 아래, 굳이 트리 내부까지 재귀적으로 비교하는 대신 즉시 `unmount → mount`로 처리해 비교 비용 자체를 없애는 전략이다.

**가정 2. key prop으로 자식 엘리먼트가 렌더링 사이에도 안정적으로 유지되는지 힌트를 줄 수 있다.**

리스트를 렌더링할 때 `key`를 생략하거나 배열의 `index`를 그대로 `key`로 사용하면, 항목이 추가·삭제·재정렬될 때 문제가 발생할 수 있다.

```jsx
function TodoList({ todos }) {
  return (
    <ul>
      {todos.map((todo, index) => (
        <li key={index}>
          <input type="checkbox" defaultChecked={todo.done} />
          {todo.text}
        </li>
      ))}
    </ul>
  );
}
```

`todos` 배열의 맨 앞에 새 항목을 추가(`unshift`)하면, 기존 각 `<li>`의 `index`가 모두 하나씩 밀린다. 리액트는 `key`만으로 이전 트리의 어떤 자식과 대응되는지 판단하기 때문에, `index`를 `key`로 사용하면 실제로는 그대로인 항목도 "다른 항목"으로 오인하게 된다. 그 결과 체크박스처럼 리액트가 직접 관리하지 않는 **비제어(uncontrolled) 엘리먼트**의 로컬 상태(`defaultChecked` 등)가 엉뚱한 항목에 남아버리는 문제가 발생한다.

```jsx
function TodoList({ todos }) {
  return (
    <ul>
      {todos.map((todo) => (
        <li key={todo.id}>
          <input type="checkbox" defaultChecked={todo.done} />
          {todo.text}
        </li>
      ))}
    </ul>
  );
}
```

`todo.id`처럼 항목이 추가·삭제·정렬되어도 변하지 않는 값을 `key`로 지정하면, 리액트는 각 `<li>`를 정확히 동일한 항목으로 추적할 수 있다. 이 경우 리액트는 불필요한 unmount/remount 없이 순서만 옮겨 처리하므로, **각 항목이 가진 상태와 DOM 노드가 그대로 보존**된다.

이 두 가지 가정 덕분에 리액트의 재조정 알고리즘은 일반적인 트리 편집 거리 문제의 `O(n³)` 복잡도 대신 **`O(n)` 복잡도로 동작**할 수 있다. 다만 이 비교 작업이 "언제, 얼마만큼씩" 실행되는지는 또 다른 문제다. 리액트 16부터는 이 비교와 실제 DOM 반영 작업을 잘게 쪼개고, 필요하면 중간에 멈췄다가 이어갈 수 있는 새로운 아키텍처인 **Fiber** 위에서 수행한다.

### 🧵 4-4. Fiber: 재조정을 실행하는 아키텍처
Fiber는 컴포넌트 트리의 각 엘리먼트에 대응하는 **작업 단위(unit of work)**를 표현하는 자바스크립트 객체다. 리액트 16 이전의 **스택 기반 재조정기(Stack Reconciler)**는 한번 트리 비교를 시작하면 완료될 때까지 메인 스레드를 점유해서, 트리가 크면 사용자 입력 처리가 지연되는 문제가 있었다. Fiber 기반 재조정기는 트리 순회를 재귀 호출이 아니라 연결 리스트를 순회하는 형태로 구현해서, 작업을 작은 단위로 나눠 처리하다가 우선순위가 높은 작업(사용자 입력 등)이 들어오면 잠시 **양보(yield)**했다가 나중에 이어서 처리할 수 있다.

```text
FiberNode {
  type,            // 엘리먼트 타입 ("div", Counter 등)
  key,
  child,           // 첫 번째 자식 Fiber
  sibling,         // 다음 형제 Fiber
  return,          // 부모 Fiber
  pendingProps,    // 이번 렌더링에 사용할 props
  memoizedProps,   // 이전 렌더링에 사용했던 props
  memoizedState,   // 이전 렌더링에서의 상태 (hook 정보 포함)
  alternate,       // 짝이 되는 다른 트리의 Fiber
  ...
}
```

리액트는 항상 두 개의 Fiber 트리를 유지한다. 현재 화면에 반영되어 있는 `current` 트리와, 다음 렌더링 결과를 계산하는 `work-in-progress` 트리다. `work-in-progress` 트리의 작업이 모두 끝나면, 두 트리를 가리키는 포인터를 교체(`commit`)하는 방식으로 화면을 갱신한다. 이러한 **이중 버퍼링(double buffering)** 구조 덕분에 작업을 중간에 멈추더라도 화면에는 항상 완성된 이전 트리가 유지되고, 중단했던 지점부터 이어서 계산을 재개할 수 있다. 앞서 다룬 동시성(Concurrency) 스터디 노트에서 살펴본 Concurrent Rendering, `useTransition`, `useDeferredValue` 같은 기능들은 모두 **이 Fiber 아키텍처가 있기에 가능한 것**이다.

### 🔁 4-5. 불필요한 리렌더링과 수동 메모이제이션
리액트는 부모 컴포넌트가 리렌더링되면 **기본적으로 모든 자손 컴포넌트도 함께 리렌더링**한다. 이는 리액트가 애초에 설계된 대로 작동하는 방식이다. 다만 이 방식은 크고 복잡한 사용자 인터페이스를 처리할 때 성능에 상당한 부담을 줄 수 있다. `ParentComponent`에서 전달하는 prop이 바뀌지 않더라도, `ParentComponent`의 상태가 바뀌면 `ChildComponent`도 함께 리렌더링되기 때문이다.

```jsx
const ChildComponent = React.memo(function ChildComponent({ label }) {
  console.log('ChildComponent 렌더링');
  return <p>{label}</p>;
});

function ParentComponent() {
  const [count, setCount] = useState(0);
  const [text] = useState('변하지 않는 값');

  return (
    <div>
      <button onClick={() => setCount(count + 1)}>{count}</button>
      {/* memo가 없다면 count가 바뀔 때마다 ChildComponent도 함께 리렌더링된다 */}
      <ChildComponent label={text} />
    </div>
  );
}
```

`memo`는 이전 props와 새 props를 **얕은 비교(shallow comparison)**해서 동일하면 리렌더링 자체를 건너뛴다. 다만 매 렌더링마다 새로 생성되는 객체나 함수를 props로 전달하면 얕은 비교가 항상 실패하므로, `useMemo`와 `useCallback`으로 참조를 안정화해야 `memo`가 실제로 효과를 발휘한다.

```jsx
function ParentComponent() {
  const [count, setCount] = useState(0);
  const [items, setItems] = useState([1, 2, 3]);

  // items가 바뀌지 않는 한 동일한 배열 참조를 재사용한다
  const sortedItems = useMemo(() => [...items].sort((a, b) => a - b), [items]);
  // count가 바뀌어도 handleSelect의 참조는 유지된다
  const handleSelect = useCallback((id) => console.log(id), []);

  return (
    <div>
      <button onClick={() => setCount(count + 1)}>{count}</button>
      <ItemList items={sortedItems} onSelect={handleSelect} />
    </div>
  );
}
```

리렌더링이 광범위하게 발생하지 않도록 관리하려면, 컴포넌트 구조화에 신경 쓰는 동시에 `memo`, `useMemo`, `useCallback`을 상황에 맞게 조합해서 사용해야 했다. 문제는 **이 세 API를 어디에, 어떤 의존성 배열과 함께 적용해야 하는지 사람이 직접 판단**해야 한다는 점이다. 의존성 배열 하나만 빠뜨려도 최적화가 무의미해지거나, 심하면 오래된 값을 참조하는 버그로 이어지기도 한다. 이 문제를 해결하기 위해 등장한 것이 **React Compiler**다.

## 5. React Compiler를 통한 자동 최적화

### ✨ 5-1. React Compiler란
React Compiler는 컴포넌트와 훅의 소스 코드를 **정적으로 분석**해서, 개발자가 `memo`·`useMemo`·`useCallback`을 직접 작성하지 않아도 빌드 타임에 자동으로 메모이제이션 코드를 삽입해주는 컴파일러다. 리액트 팀 내부에서 "React Forget"이라는 이름으로 실험되다가, 이후 정식으로 공개되어 **2025년 10월 1.0 버전이 stable로 릴리스**되었다.

동작 원리를 요약하면, 컴포넌트 함수 내부에서 각 값과 JSX가 어떤 의존성으로부터 파생되는지 분석해서, 의존성이 바뀌지 않으면 이전 렌더링 결과(값 혹은 JSX 서브트리)를 그대로 재사용하도록 코드를 변환하는 것이다. 즉 4-5에서 손으로 작성했던 `memo`/`useMemo`/`useCallback` 패턴을, **컴파일러가 코드를 분석해서 자동으로 대신 만들어 넣어주는 것**과 같다.

### 5-2. 적용 방법
```bash
pnpm add -D babel-plugin-react-compiler@latest eslint-plugin-react-hooks@latest
```

Babel을 직접 사용하는 프로젝트라면 다음과 같이 설정한다. React Compiler 플러그인은 **반드시 다른 Babel 플러그인보다 먼저 실행**되어야 한다.

```js
// babel.config.js
module.exports = {
  plugins: [
    'babel-plugin-react-compiler', // 반드시 다른 플러그인보다 먼저 실행되어야 한다
  ],
};
```

Vite(v6 이상) 프로젝트에서는 `@rolldown/plugin-babel`을 통해 등록한다.

```js
// vite.config.js
import { defineConfig } from 'vite';
import react, { reactCompilerPreset } from '@vitejs/plugin-react';
import babel from '@rolldown/plugin-babel';

export default defineConfig({
  plugins: [
    react(),
    babel({ presets: [reactCompilerPreset()] }),
  ],
});
```

예전에는 컴파일러 진단을 위해 `eslint-plugin-react-compiler`를 별도로 설치해야 했지만, 현재는 그 기능이 `eslint-plugin-react-hooks`에 통합되어 있다. 이미 `eslint-plugin-react-compiler`가 설치되어 있다면 제거하고 `eslint-plugin-react-hooks@latest`로 옮기면 된다. 린트 단계에서 이 플러그인을 활성화해두면, **컴파일러가 최적화를 포기할 수밖에 없는 코드 패턴을 빌드 이전에 미리 잡아낼 수 있다**. React Compiler는 React 19에서 가장 잘 동작하도록 만들어졌지만, React 17·18 프로젝트에서도 사용할 수 있다.

### 5-3. 적용 전후 비교
4-5에서 다룬 리스트 컴포넌트를 React Compiler 없이 수동으로 최적화하면 다음과 같다.

```jsx
// React Compiler 이전: 수동 메모이제이션이 필요했다
const ExpensiveList = React.memo(function ExpensiveList({ items, onSelect }) {
  const sortedItems = useMemo(() => [...items].sort(), [items]);
  const handleSelect = useCallback((id) => onSelect(id), [onSelect]);

  return (
    <ul>
      {sortedItems.map((item) => (
        <li key={item.id} onClick={() => handleSelect(item.id)}>
          {item.name}
        </li>
      ))}
    </ul>
  );
});
```

React Compiler를 적용하면, 개발자는 `memo`·`useMemo`·`useCallback` 없이 **로직 그 자체에만 집중한 코드**를 작성하면 된다. 컴파일러가 정적 분석을 통해 위와 동등한 메모이제이션 코드를 빌드 결과물에 자동으로 삽입해준다.

```jsx
// React Compiler 적용 후: 로직만 작성하면 메모이제이션은 컴파일러가 처리한다
function ExpensiveList({ items, onSelect }) {
  const sortedItems = [...items].sort();

  return (
    <ul>
      {sortedItems.map((item) => (
        <li key={item.id} onClick={() => onSelect(item.id)}>
          {item.name}
        </li>
      ))}
    </ul>
  );
}
```

### 5-4. 주의할 점
- React Compiler는 [리액트의 규칙(Rules of React)](https://react.dev/reference/rules)을 지키는 코드에서만 안전하게 동작한다. 렌더링 도중에는 **부수 효과(side effect)**를 발생시키지 않아야 하고, props와 state를 직접 `mutate`해서는 안 되며, 훅은 항상 컴포넌트 최상단에서 조건 없이 호출되어야 한다.
- 컴포넌트가 이 규칙을 위반하면 컴파일러는 해당 컴포넌트에 대한 최적화를 건너뛴다(**bail out**). 이 경우 런타임 에러가 발생하지는 않지만, 기대했던 만큼의 성능 개선 효과는 얻지 못한다. 앞서 설정한 ESLint 플러그인이 바로 이런 패턴을 미리 잡아내는 역할을 한다.
- React Compiler는 **재조정(reconciliation) 알고리즘 자체를 대체하지 않는다**. 애초에 리렌더링이 필요 없는 컴포넌트가 재조정 단계에 진입하는 것을 막아주는 역할을 할 뿐이며, 4장에서 다룬 가상 DOM 트리 비교와 Fiber 기반 스케줄링은 컴파일러 적용 여부와 무관하게 동일하게 동작한다.
- 기존에 수동으로 작성해둔 `useMemo`/`useCallback`/`memo` 코드를 컴파일러 도입과 동시에 지울 필요는 없다. 컴파일러는 이미 메모이제이션된 코드 위에서도 안전하게 동작하도록 설계되어 있으므로, **점진적으로 마이그레이션**하면 된다.

## 6. 정리
가상 DOM은 실제 DOM을 직접 조작할 때 발생하는 **성능 비용**과 **브라우저 호환성 문제**를 줄이기 위해 리액트가 채택한 추상화다. 재조정 과정에서는 "타입이 다르면 트리도 다르다", "key로 자식의 정체성을 힌트로 준다"는 **두 가지 휴리스틱**을 통해, 일반적인 트리 비교보다 훨씬 빠른 `O(n)` 복잡도로 변경 사항을 계산한다. 이 재조정 작업은 **Fiber**라는 중단·재개 가능한 작업 단위 위에서 수행되며, 이는 Concurrent Rendering을 비롯한 최신 리액트 기능의 토대가 된다. 그리고 **React Compiler**는 이 파이프라인의 앞단에서, 애초에 불필요한 컴포넌트가 재조정 단계에 진입하지 않도록 자동으로 걸러주는 최신 최적화 도구다.

## 📚 참고자료
- Tejas Kumar, 『전문가를 위한 리액트(Fluent React)』, O'Reilly, 3장 "가상 DOM"
- [Preserving and Resetting State – React](https://react.dev/learn/preserving-and-resetting-state)
- [Rendering Lists – React](https://react.dev/learn/rendering-lists)
- [React Compiler v1.0 – React Blog](https://react.dev/blog/2025/10/07/react-compiler-1)
- [React Compiler Installation – React](https://react.dev/learn/react-compiler/installation)
- [Rules of React – React](https://react.dev/reference/rules)
