---
title: Concurrent Rendering을 활용할 수 있는 React API
published: 2025-10-26
tags: [React]
category: React
series: React
draft: false
---
## Concurrent Rendering을 활용할 수 있는 React API

React 18부터는 내부 스케줄러(Fiber Scheduler)가 Concurrent Rendering을 지원하면서 개발자가 렌더링 우선순위를 직접 제어하거나 힌트를 줄 수 있는 다양한 API가 새롭게 추가되었다.

이 API들은 모두 “UI 반응성을 유지하면서 무거운 렌더링이나 비동기 처리를 부드럽게 실행”하는 것을 목표로 한다.

---

### useTransition

> `useTransition` is a React Hook that lets you render a part of the UI in the background.
`useTransition`은 UI의 특정 부분을 백그라운드에서 렌더링할 수 있도록 하는 React 훅이다.
> 

```jsx
const [isPending, startTransition] = useTransition();
```

### useTransition 훅의 우선순위 조정

`useTransition`은 상태 업데이트를 **낮은 우선순위(Transition 상태)**로 마킹해준다.

React는 모든 상태 업데이트를 Lane(우선순위 트랙)에 배치하여 스케줄링한다. `startTransition`으로 감싼 업데이트는 **TransitionLane**에 할당되어 (사용자 입력 등과 같이 높은 우선순위 Lane을 부여 받은 작업보다) 낮은 우선순위로 처리된다.

```
사용자 입력(setKeyword) → SyncLane (높은 우선순위)
필터링(setFiltered)     → TransitionLane (낮은 우선순위)
```

즉 “지금 바로 반응해야 하는 작업(입력, 클릭)”과 “조금 늦게 반영되어도 되는 작업(리스트 렌더링, 필터링) 결과 표시”를 분리해준다.

### 📦 사용자 입력 + 검색 필터링 예시

```jsx
import { useState, useTransition } from 'react';

function SearchList({ items }) {
	const [keyword, setKeyword] = useState("");
	const [filtered, setFiltered] = useState(items);
	const [isPending, startTransition] = useTransition();
	
	const handleChange = (e) => {
		const value = e.target.value;
		setKeyword(value); // 즉시 반영
		
		// 렌더링 여유 있을 때 실행하도록 우선순위 조정
		startTransition(() => {
			const result = items.filter((item) => item.toLowerCase().includes(value.toLowerCase()));
			setFiltered(result);
		});		
	}
	
	return (
		<div>
			<input value={keyword} onChange={handleChange} />
			{isPending && <p>검색 중입니다...</p>}
			<ul>
				{filtered.map((item) => (
					<li key={item.itemId}>{item.name}</li>
				))}
			</ul>
		</div>
	)
}
```

**🧠 동작 원리**

- `setKeyword` → **즉시 렌더링 (높은 우선순위)**
    - 사용자의 입력이 즉각 반영되어 입력 지연이 발생하지 않는다
- `startTransition` 내부의 setFiltered → **낮은 우선순위로 스케줄링**
    - React는 브라우저가 idle 상태일 때(즉, 여유가 있을 때) 해당 렌더링을 수행한다.
    - 만약 렌더링 도중 새로운 high-priority 이벤트가 발생하면, transition 렌더링은 즉시 중단(yield)되었다가 다시 재개된다.
- `isPending` → **Transition 상태 동안 true**
    - 백그라운드 렌더링이 진행 중임을 나타내며, 로딩 표시 등에 활용할 수 있다

**⚡장점**

- **✅ 입력 반응성 유지**
    - 무거운 필터링이나 렌더링이 있어도 입력 타이핑 끊김 현상 발생 방지
- **✅ 부드러운 전환 (Transition)**
    - 리스트나 UI 변화가 부드럽게 진행되며, React가 여유 있을 때만 렌더링을 수행
- **✅ 로딩 상태 표시 가능**
    - `isPending`을 통해 Transition 중임을 감지하여, 스켈레톤 UI, 스피너 등을 표시할 수 있음
- **✅ Race condition 방지**
    - `startTransition`으로 감싼 비동기 처리(fetch 등)는 렌더링 반영 시점을 조절하기 때문에,
        - 사용자의 빠른 입력 변경과 서버 응답 도착 간의 **UI 불일치 현상(stale UI)**을 줄이는 데 도움이 된다.
        - (단, 네트워크 요청 자체를 취소하려면 `AbortController`를 병행해야 한다.)

---

### useDeferredValue

> `useDeferredValue` is a React Hook that lets you defer updating a part of the UI.
`useDeferredValue`는 UI의 특정 부분의 업데이트를 지연시킬 수 있도록 하는 React 훅이다.
> 

```jsx
const deferredValue = useDeferredValue(value)
```

공식 문서에 따르면 useDeferredValue는 새로운 콘텐츠가 로딩되는 동안 stale(신선하지 않은 상태)한 콘텐츠를 보여줄 때 사용할 수 있다. (**Showing stale content while fresh content is loading**)

따라서, useDeferredValue는 **값(value)**을 기반으로 렌더링되는 UI가 있을 때, 그 값을 “조금 늦게 반영”하도록 만들어 UI 반응성을 유지하는 훅이다. 

업데이트가 진행될 때, 최초 리렌더링 시에는 이전 값(deferred value)을 그대로 사용하고, 백그라운드에서 새롭게 전달 받은 값을 다음 렌더링 시에 반영함으로서 **UI의 상태 불일치를** 최소화할 수 있도록 한다.

### useDeferredValue 훅의 우선순위 조정

`useTransition`과 마찬가지로 `useDeferredValue`로 생성된 값의 업데이트는 **TransitionLane**에 매핑된다.

```
사용자 입력(setKeyword) → SyncLane (높은 우선순위)
필터링(setFiltered)     → TransitionLane (낮은 우선순위)
```

결과적으로 사용자 입력은 즉시 반영하지만 리스트는 조금 늦게 업데이트되는 자연스러운 UX 경험을 제공할 수 있다.

### 📦 사용자 입력 + 검색 필터링 예시

```jsx
import { useState, useDeferredValue } from 'react';

function SearchList({ items }) {
	const [keyword, setKeyword] = useState('');
	const deferredKeyword = useDeferredvalue(keyword); // 입력값 늦게 반영
	
	const filteredItems = items.filter((item) => item.toLowerCase().includes(deferredQuery.toLowerCase());
	
	return (
		<div>
			<input value={keyword} onChange={(e) => setKeyword(e.target.value)} />
			<ul>
				{filteredItems.map((item) => (
					<li key={item.itemId}>{item.name}</li>
				))}			
			</ul>
		</div>
	)
}
```

**🧠 동작 원리**

- `setKeyword` → **즉시 렌더링 (높은 우선순위)**
    - 사용자의 입력이 즉각 반영되어 입력 지연이 발생하지 않는다
- `deferredQuery` → keyword의 **지연된 버전**
    - React가 브라우저의 렌더링 우선순위를 고려해 **TransitionLane(낮은 우선순위)**로 스케줄링
    - 렌더링이 무거운 경우, React는 **사용자 입력이 먼저 처리된 뒤** 여유가 있을 때 `deferredQuery` 기반의 렌더링을 수행

**⚡장점**

- **✅ 입력 반응성 유지**
    - 무거운 필터링이나 렌더링이 있어도 입력 타이핑 끊김 현상 발생 방지
- **✅ 렌더링 부하 완화**
    - 무거운 필터링, 정렬, 데이터 렌더링 연산을 브라우저가 idle한 시점에 수행 가능
- **✅ 다른 훅에서 활용 가능**
    - `useDeferredValue`는 React Query, 상태 관리 라이브러리 등에 상태 값에도 적용 가능하여 전역 상태 기반 UI 렌더링 반응성 개선이 가능하다

---

### 📚 참고자료

- https://react.dev/reference/react/useTransition
- https://react.dev/reference/react/useDeferredValue
- https://goidle.github.io/react/in-depth-react18-lane/