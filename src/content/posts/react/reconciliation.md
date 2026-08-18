---
title: 🌳 재조정과 React의 Fiber 구조
published: 2026-08-19
tags: [React, Javascript, Study]
category: React
series: Study
image: /images/frontend-study.webp
draft: false
---
# 🌳 재조정과 React의 Fiber 구조

> **TL;DR** — 가상 DOM 트리를 이전 트리와 비교해 실제 DOM에 반영하는 과정을 재조정(Reconciliation)이라고 한다. 기존 **스택 재조정자**는 한 번 시작한 렌더링을 중단할 수 없어 사용자 입력이 밀리는 문제가 있었고, 이를 해결하기 위해 **파이버(Fiber) 재조정자**가 등장했다. 파이버는 리액트 엘리먼트마다 생성되는 상태 저장형 연결 리스트 노드로, 작업 루프가 이 트리를 순회하며 **렌더 단계**(중단 가능)와 **커밋 단계**(중단 불가능)로 나눠 작업을 처리한다. 이때 각 업데이트의 긴급도는 **lane**이라는 비트마스크로 표현되고, **React Scheduler**가 이 우선순위에 따라 메인 스레드에 실행을 양보할지 판단한다. `useTransition`은 이 lane 시스템을 개발자가 직접 다룰 수 있게 해주는 훅이다.
>
> 테자스 쿠마르(Tejas Kumar), 『전문가를 위한 리액트(Fluent React)』(O'Reilly)를 읽고 정리한 스터디 노트다. 가상 DOM 스터디 노트에 이어지는 내용이다.

---

## 목차

1. [리액트 엘리먼트와 재조정](#1-리액트-엘리먼트와-재조정)
2. [리액트의 기존 기술: 스택 재조정자](#2-리액트의-기존-기술-스택-재조정자)
3. [파이버 재조정자](#3-파이버-재조정자)
4. [파이버 재조정의 두 단계](#4-파이버-재조정의-두-단계)
5. [우선순위 스케줄링: Lane과 React Scheduler](#5-우선순위-스케줄링-lane과-react-scheduler)
6. [FiberRootNode와 트리 교체](#6-fiberrootnode와-트리-교체)
7. [정리](#7-정리)
8. [참고자료](#-참고자료)

---

## 1. 리액트 엘리먼트와 재조정

```javascript
import { useState } from 'react'

const App = () => {
    const [count, setCount] = useState(0);

    return (
        <main>
            <div>
                <h1>Hello, I'm counter</h1>
                <span>Count: {count}</span>
                <button onClick={() => setCount(count + 1)}>Count Button</button>
            </div>
        </main>
    )
}
```

위 코드에 있는 엘리먼트 트리는 원하는 UI 상태의 **선언적 설명**이다. `App` 컴포넌트는 함수로 호출되면 자손 엘리먼트 4개를 포함한 리액트 엘리먼트를 반환하는데, 이 불변(immutable) 객체인 리액트 엘리먼트는 그리고자 하는 UI의 상태를 표현한다.

리액트는 해당 함수를 실행하여 가상 DOM을 생성하고, 이후 이 트리를 활용해 **최소한의 DOM API만 호출**해 브라우저에 반영한다.

이때 리액트는 어떻게 최소한의 DOM API를 호출하여 효율적으로 DOM을 업데이트할까?

## 2. 리액트의 기존 기술: 스택 재조정자
**스택(Stack)**은 마지막에 들어온 항목이 가장 먼저 나가는 **LIFO(Last In, First Out)** 원칙을 따르는 선형 자료구조다. 기존 리액트는 스택 재조정자를 사용해 새 가상 트리를 이전 가상 트리와 비교하고 그에 따라 DOM을 업데이트했다.

스택 재조정자는 비교적 단순한 애플리케이션에서는 문제 없이 동작했지만, 애플리케이션 규모가 커지고 복잡해지면서 여러 가지 문제가 발생했다. 다음과 같은 동작들이 한 애플리케이션에서 발생한다고 가정해보자:

1. 필수적이지 않고 계산 비용이 비싼 컴포넌트를 렌더링
2. 사용자가 `input` 엘리먼트에 입력
3. 입력이 유효할 경우 버튼을 활성화
4. `Form` 컴포넌트의 상태 변경 시 컴포넌트 재렌더링

스택 재조정자는 작업을 일시 중지하거나 연기하지 않고 순차적으로 변경 사항을 렌더링한다. 사용자 입력처럼 우선순위가 높은 렌더링 작업이 끼어들 때는 현재 진행 중인 렌더링 작업을 멈출 수 있어야 하는데, **스택 재조정자는 업데이트의 우선순위를 설정하지 않고 업데이트를 중단하거나 취소할 수 없다.** 이로 인해 사용자 경험에 중요한 영향을 끼치는 `input` 엘리먼트 입력에 끊김 현상이 발생하거나 사용자 인터페이스의 응답 속도가 느려지는 문제가 발생할 수 있다.

## 3. 파이버 재조정자
리액트는 스택 재조정자의 문제점을 해결하기 위해 **파이버 재조정자(Fiber reconciler)**를 도입했다.

### 🧬 3-1. 파이버란 무엇인가
**파이버(Fiber)**는 재조정자를 위한 작업 단위(unit of work)를 나타내는 데이터 구조다. 파이버는 리액트 엘리먼트로부터 생성되지만 성격은 정반대에 가깝다. 리액트 엘리먼트가 매 렌더링마다 새로 만들어지는 **임시적이고 상태가 없는** 불변 객체라면, 파이버는 컴포넌트가 마운트되어 있는 동안 계속 유지되며 **상태를 저장하는 수명이 긴** 가변(mutable) 객체다.

### 3-2. 파이버의 데이터 구조
```javascript
{
  tag: 3,               // 컴포넌트 유형 (예: 3 = ClassComponent)
  key: null,             // 형제 사이에서 이 파이버를 식별하는 값
  type: App,              // 파이버가 나타내는 함수/클래스 컴포넌트 또는 호스트 태그("div" 등)
  stateNode: null,         // 이 파이버에 대응하는 실제 인스턴스
                            // (호스트 컴포넌트라면 DOM 노드, 클래스 컴포넌트라면 인스턴스, 루트라면 FiberRootNode)

  // 트리 구조를 나타내는 포인터
  return: FiberParent,    // 부모 파이버
  child: FiberChild,      // 첫 번째 자식 파이버
  sibling: FiberSibling,  // 다음 형제 파이버
  index: 0,               // 형제 목록에서의 위치

  // props / state
  pendingProps: {},       // 이번 렌더링에 적용할 props
  memoizedProps: {},      // 마지막 렌더링에 사용했던 props
  memoizedState: null,    // 마지막 렌더링 결과 상태 (훅 연결 리스트 포함)
  updateQueue: null,      // 처리 대기 중인 상태 업데이트 큐

  flags: 0,               // 커밋 단계에서 적용할 부작용을 나타내는 비트마스크
                           // (Placement, Update, Deletion 등)

  lanes: 0,                // 이 파이버에 남아있는 작업의 우선순위
  childLanes: 0,           // 자손 파이버에 남아있는 작업의 우선순위

  alternate: null,         // 다른 트리(current ↔ work-in-progress)에서 짝이 되는 파이버
}
```

파이버 재조정자는 가상 DOM의 각 리액트 엘리먼트에 대해 `createFiberFromTypeAndProps` 함수를 실행하여 파이버 노드를 생성한다. 주요 필드는 다음과 같다.

- **`tag`**: 컴포넌트 유형(클래스, 함수 컴포넌트, 서스펜스, 오류 경계, 조각 등)을 나타내는 고유한 숫자 ID
- **`type`**: 파이버가 나타내는 함수 또는 클래스 컴포넌트
- **`props`**: 컴포넌트에 대한 입력 props 또는 함수에 대한 입력 인수
- **`stateNode`**: 파이버에 대응하는 실제 인스턴스. 호스트 컴포넌트(`div` 등)라면 실제 DOM 노드, 클래스 컴포넌트라면 그 인스턴스, 루트 파이버라면 [FiberRootNode](#6-fiberrootnode와-트리-교체)를 가리킨다
- **`return`/`child`/`sibling`/`index`**: 각각 부모, 첫 자식, 다음 형제, 형제 목록에서의 위치를 의미하며, 파이버 재조정자는 이를 사용해 트리를 순회한다
- **`flags`**: 이 파이버를 커밋할 때 어떤 부작용(effect)을 적용해야 하는지 나타내는 비트마스크로, [4-2. 커밋 단계](#4-파이버-재조정의-두-단계)에서 다룰 배치·업데이트·삭제 효과가 여기 담긴다
- **`lanes`/`childLanes`**: 이 파이버(와 그 자손)에 남아있는 작업의 우선순위를 나타내며, [5. Lane](#5-우선순위-스케줄링-lane과-react-scheduler)에서 자세히 다룬다
- **`alternate`**: 아래에서 설명할 이중 버퍼링에서, 짝이 되는 다른 트리의 동일한 파이버를 가리키는 참조

파이버 노드 생성 후에는 **작업 루프(work loop)**를 사용해 사용자 인터페이스를 업데이트한다. 작업 루프는 루트 파이버 노드에서 시작해 컴포넌트 트리를 따라 내려가며 업데이트가 필요한 각 파이버 노드를 '더티'로 표시한 후(`beginWork`), 끝에 도달하면 다시 반대로 순회하며 브라우저의 DOM 트리와 분리된 새 DOM 트리를 메모리에 생성하는 과정을 거친다(`completeWork`). `beginWork`와 `completeWork`가 실제로 어떤 일을 하는지는 [4. 파이버 재조정의 두 단계](#4-파이버-재조정의-두-단계)에서 자세히 다룬다.

### 🔁 3-3. 작업 루프와 이중 버퍼링
이러한 오프스크린 렌더링 프로세스(off-screen rendering process)는 사용자가 볼 수 없으므로 언제든지 중단하고 버린 후 새로 실행할 수 있다. 이렇게 다음 화면을 화면 밖에서 준비한 다음 현재 화면으로 내보내는 개념을 **더블 버퍼링(double buffering)**이라고 한다.

더블 버퍼링은 원래 컴퓨터 그래픽이나 비디오 처리 시 깜빡임을 줄이고 체감 성능을 개선하기 위해 게임 업계에서 활용하던 기술이다. 이미지 프레임을 저장하기 위한 두 개의 버퍼를 생성하고 일정한 간격으로 두 버퍼를 전환함으로써 최종 이미지나 동영상이 중단 및 지연 없이 표시되도록 할 수 있다.

파이버 재조정은 이 개념에서 착안해, 업데이트가 발생하면 현재 파이버 트리가 포크(fork)되어 주어진 사용자 인터페이스의 새로운 상태를 반영하도록 업데이트한다. 이를 **렌더링**이라고 한다. 그 후 현재 트리를 대체할 트리가 준비되고 사용자가 기대하는 상태를 정확히 반영했을 때, 현재 파이버 트리를 교체하는 작업을 수행하는데 이를 **커밋 단계(commit phase)** 또는 **커밋(commit)**이라고 한다.

**파이버 재조정자의 더블 버퍼링이 주는 이점:**
- 실제 DOM에 대한 불필요한 업데이트를 최소화하여 성능을 개선하고 깜빡임을 줄임
- 화면 밖에서 UI의 새 상태를 계산하고, **우선순위가 더 높은 새로운 업데이트가 있을 경우 언제든지 버리고 새로 시작 가능**
- 사용자가 현재 보고 있는 내용을 망치지 않고 일시 중지했다가 다시 시작할 수 있음

## 4. 파이버 재조정의 두 단계
파이버 재조정은 위에서 언급한 대로 **렌더 단계(Render Phase)**와 **커밋 단계(Commit Phase)**로 나뉜다. 리액트는 렌더링 작업을 수행하고 이를 DOM에 커밋해서 사용자에게 보여주기 전까지는 언제든 작업을 폐기할 수 있다. 이러한 렌더링 중단이 가능한 이유는 리액트 **스케줄러**가 일정 시간마다 실행을 메인 스레드로 돌려주기 때문인데, 구체적인 동작 방식은 [5-2. React Scheduler와 타임 슬라이싱](#-5-2-react-scheduler와-타임-슬라이싱)에서 다룬다.

### 4-1. 렌더 단계 (Render Phase)
렌더 단계는 현재 트리에서 상태 변경 이벤트가 발생하면 시작한다. 리액트는 각 파이버를 재귀적, 단계적으로 순회하며 업데이트가 보류 중이라는 신호를 플래그로 설정해 대체 트리에 오프스크린 변경 작업을 수행한다.

**`beginWork` (작업 시작)**

`beginWork`는 작업용 트리에 있는 파이버 노드의 업데이트를 처리한다. 함수 컴포넌트라면 컴포넌트 함수를 실제로 호출해 새로운 자식 엘리먼트를 얻고, 그 결과를 이전 자식 파이버와 비교해 자식 파이버를 새로 만들거나 재사용하는 **자식 조정(child reconciliation)**을 수행한다. 이 과정에서 바로 가상 DOM 스터디 노트에서 다룬 재조정의 두 가지 휴리스틱 — **타입이 다르면 새로 마운트**, **key로 자식의 정체성을 식별** — 이 적용되어, 각 자식 파이버에 유지·교체·삭제 여부가 `flags`로 기록된다.

`beginWork`의 시그니처는 다음과 같다.

```typescript
function beginWork(
    current: Fiber | null,
    workInProgress: Fiber,
    renderLanes: Lanes,
) : Fiber | null;
```

- **`current`**: 업데이트 중인 작업용 노드에 대응하는, 현재 트리 쪽 파이버 노드에 대한 참조
- **`workInProgress`**: 작업용 트리에서 업데이트 중인 파이버 노드. `beginWork` 함수에 의해 업데이트되어 '더티'로 표시된 채 반환되는 노드
- **`renderLanes`**: 이번 렌더링에서 처리할 우선순위를 나타내는 비트마스크. 파이버의 `lanes`가 `renderLanes`에 포함되지 않으면 해당 파이버(와 그 서브트리)는 이번 렌더링에서 건너뛴다. 자세한 내용은 [5-1. Lane이란](#-5-1-lane이란) 참고

**`completeWork` (작업 완료)**

`completeWork`는 트리를 다시 거슬러 올라가며(bottom-up), 호스트 컴포넌트라면 실제 DOM 노드 인스턴스를 생성하거나 이전 props와 새 props를 비교해 업데이트할 내용을 계산하고, 자식들이 이미 만들어둔 DOM 노드를 자신의 인스턴스에 붙여 나간다. 즉 `beginWork`가 "무엇이 바뀌었는지" 하향식으로 표시한다면, `completeWork`는 그 결과를 상향식으로 모아 **커밋할 새 DOM 트리를 완성**하는 역할을 한다.

`completeWork`의 시그니처는 `beginWork`와 동일하다.

```typescript
function completeWork(
    current: Fiber | null,
    workInProgress: Fiber,
    renderLanes: Lanes
) : Fiber | null
```

### 4-2. 커밋 단계 (Commit Phase)
렌더 단계가 끝나면, 이때 생성된 가상 DOM에 적용된 변경 사항을 실제 DOM에 반영하는 커밋 단계를 수행한다. 렌더 단계와 달리 **커밋 단계는 중단할 수 없다** — 한 번 시작하면 동기적으로 끝까지 실행되어야 사용자에게 일관성 없는 화면이 노출되지 않는다. 커밋 단계는 다시 변형 단계와 레이아웃 단계로 나뉜다.

**변형 단계 (Mutation phase)**

가상 DOM에 적용된 변경 사항을 실제 DOM에 반영하는 과정이다. `commitMutationEffects`라는 특수 함수를 호출하여 실제 업데이트 사항을 DOM에 반영한다.

**레이아웃 단계 (Layout phase)**

DOM에서 업데이트된 노드의 새 레이아웃을 계산하는 단계다. `commitLayoutEffects`라는 특수 함수를 호출하여 새 레이아웃을 계산한다. (참고로 실제 구현에는 클래스 컴포넌트의 `getSnapshotBeforeUpdate`처럼 DOM이 바뀌기 직전 상태를 읽어야 하는 생명주기를 위한 더 작은 "before mutation" 단계도 존재하지만, 개념적으로는 변형·레이아웃 두 단계로 이해해도 충분하다.)

리액트 재조정 과정의 커밋 단계에서는 여러 부작용이 특정 순서로 실행되며 다음과 같은 효과를 낼 수 있다.

- **배치 효과(Placement)**: 새 컴포넌트가 DOM에 추가될 때 발생해 해당 요소를 DOM에 추가
- **업데이트 효과(Update)**: 새 props나 state로 업데이트될 때 발생
- **삭제 효과(Deletion)**: 컴포넌트가 DOM에서 제거될 때 발생
- **레이아웃 효과(Layout effect)**: 브라우저의 페인트 가능 시점 이전에 발생해 페이지 레이아웃을 업데이트하는 데 사용

이와 달리 **패시브 효과(passive effect)**는 브라우저의 페인트 가능 시점 이후에 실행되도록 예약된 사용자 정의 효과이며, `useEffect` 훅을 사용해 관리된다.

## 5. 우선순위 스케줄링: Lane과 React Scheduler
지금까지는 파이버 트리를 "어떻게" 순회하고 갱신하는지를 다뤘다. 이번 장에서는 리액트가 여러 업데이트 중 "무엇을 먼저" 처리할지 결정하는 방식을 살펴본다.

### 🧭 5-1. Lane이란
**Lane**은 각 업데이트의 우선순위를 나타내는 31비트 비트마스크(bitmask)다. 자바스크립트의 비트 연산이 32비트 정수를 기준으로 동작하기 때문에 최대 31개의 lane을 표현할 수 있으며, 비슷한 우선순위의 업데이트는 같은 lane으로 묶인다. 대표적인 lane은 다음과 같다(우선순위가 높은 순).

| Lane | 용도 |
| --- | --- |
| `SyncLane` | 클릭 등 즉시 반영되어야 하는 동기 업데이트 |
| `InputContinuousLane` | 드래그, 스크롤처럼 연속적으로 발생하는 입력 |
| `DefaultLane` | 일반적인 `setState`, 데이터 페칭 결과 반영 등 |
| `TransitionLanes` (16개) | `useTransition`/`startTransition`으로 표시된, 지연되어도 무방한 업데이트 |
| `IdleLane` | 우선순위가 가장 낮은, 화면 밖(offscreen) 작업 등 |

`TransitionLanes`가 16개나 되는 이유는, 여러 개의 트랜지션이 동시에 진행 중이더라도 각각을 서로 다른 lane에 배정해 **독립적으로 추적**하기 위해서다.

리액트는 비트 연산으로 lane을 다룬다. 여러 업데이트가 동시에 발생하면 OR(`|`) 연산으로 하나의 `pendingLanes` 비트마스크에 병합하고, 특정 우선순위의 작업이 포함되어 있는지는 AND(`&`) 연산으로 확인한다.

```javascript
// 여러 업데이트를 하나의 비트마스크로 병합
const pendingLanes = SyncLane | TransitionLane1;

// 특정 lane의 작업이 남아있는지 확인
const hasSyncWork = (pendingLanes & SyncLane) !== NoLanes;
```

3-2에서 살펴본 파이버의 `lanes` 필드는 그 파이버 자신에게 남아있는 작업의 우선순위를, `childLanes`는 자손 파이버 중 어딘가에 남아있는 작업의 우선순위를 나타낸다. `beginWork`가 트리를 내려가다가 어떤 파이버의 `lanes`와 `childLanes`를 모두 `renderLanes`와 비교해 겹치는 비트가 하나도 없다면, 그 파이버는 물론 서브트리 전체를 처리할 필요가 없다고 판단해 통째로 건너뛸 수 있다. 이 덕분에 리액트는 트리 전체를 매번 순회하지 않고도 "어디에 처리할 작업이 있는지"를 빠르게 좁혀나갈 수 있다.

### ⏳ 5-2. React Scheduler와 타임 슬라이싱
**React Scheduler**는 재조정자(reconciler)와는 별개의 패키지로, 다음 역할을 담당한다.

- 대기 중인 작업들을 우선순위 순으로 관리하는 큐를 유지
- `shouldYield()` 함수로 지금 실행을 메인 스레드에 돌려줘야 하는지 판단
- 실행을 양보해야 한다면 `MessageChannel`(폴리필 환경에서는 `setTimeout`)을 이용해 다음 매크로태스크로 작업을 이어감

이렇게 렌더 단계의 작업을 잘게 쪼개 실행하는 기법을 **타임 슬라이싱(time slicing)**이라고 한다. `shouldYield()`는 현재 작업 조각을 시작한 지 약 **5밀리초**가 지났는지를 기준으로 판단하는데, 이는 `requestAnimationFrame`으로 다음 프레임을 기다리는 대신 한 프레임 안에서도 여러 번 양보할 수 있게 해, 사용자 입력이나 브라우저 렌더링이 오래 걸리는 작업에 밀리지 않도록 한다.

작업 루프는 이 양보 가능 여부에 따라 두 가지 모드로 나뉜다.

- **`workLoopSync`**: `shouldYield()`를 확인하지 않고 끝까지 동기적으로 실행. `SyncLane`처럼 가장 급한 업데이트에 사용되며, 렌더 단계라도 중단되지 않는다
- **`workLoopConcurrent`**: 매 단위 작업(unit of work)마다 `shouldYield()`를 확인해, 필요하면 언제든 양보. `TransitionLanes` 등 낮은 우선순위 lane에 사용된다

즉 "렌더 단계는 중단 가능하다"는 설명은 정확히는 **동시성(concurrent) 모드로 처리되는 lane에 한해서만** 성립한다.

### ✨ 5-3. useTransition으로 우선순위 제어하기
`useTransition`은 지금까지 살펴본 lane 시스템을 컴포넌트 코드에서 직접 활용할 수 있게 해주는 훅이다.

```jsx
function SearchResults() {
  const [isPending, startTransition] = useTransition();
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  const handleChange = (e) => {
    setQuery(e.target.value); // 즉시 반영되어야 하는 입력값 (높은 우선순위 lane)
    startTransition(() => {
      setResults(filterResults(e.target.value)); // 지연돼도 무방한 업데이트 (TransitionLane)
    });
  };

  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending && <span>검색 중...</span>}
      <ResultList items={results} />
    </>
  );
}
```

`setQuery`는 사용자가 입력한 값을 화면에 즉시 반영해야 하므로 높은 우선순위 lane에서 처리된다. 반면 `setResults`는 `startTransition`으로 감싸져 있어 `TransitionLane`에 배정되고, `isPending`은 그 작업이 아직 커밋되지 않은 동안 `true`를 유지하는 값이다.

이후 사용자가 계속 타이핑해서 새로운 입력이 들어오면, 스케줄러는 진행 중이던 `TransitionLane` 작업(아직 화면에 반영되지 않은 work-in-progress 트리)을 그대로 버리고 최신 입력을 우선 처리할 수 있다. 이것이 바로 3-3에서 다룬 **더블 버퍼링**, 즉 "화면 밖에서 준비 중인 작업은 언제든 버리고 새로 시작할 수 있다"는 성질이 실제로 활용되는 지점이다.

## 6. FiberRootNode와 트리 교체
**`FiberRootNode`**는 파이버 트리 그 자체가 아니라, `current` 트리와 work-in-progress 트리 중 **어느 쪽이 현재 화면에 반영된 트리인지를 가리키는 최상위 컨테이너**다. `createRoot`를 호출할 때 한 번 생성되며, 그 아래에 실제 컴포넌트 트리의 시작점인 호스트 루트 파이버(HostRootFiber)가 매달린다. 재조정 과정의 커밋 단계를 관리하는 핵심 데이터 구조이기도 하다.

렌더링 프로세스가 완료되면 리액트는 `commitRoot` 함수를 호출해 작업용 트리에 적용된 변경 사항을 실제 DOM에 커밋한다. 이 함수는 `FiberRootNode`의 `current` 포인터를 **지금까지의 현재 트리에서 방금 완성된 work-in-progress 트리로 전환**하며, 이로써 work-in-progress 트리가 새로운 현재 트리가 된다. 다음 업데이트가 발생하면 이번에는 방금 밀려난 이전 트리가 새로운 work-in-progress 트리로 재활용된다.

## 7. 정리
재조정은 가상 DOM 트리의 변경 사항을 실제 DOM에 최소한으로 반영하는 과정이며, 이를 실행하는 엔진이 파이버 재조정자다. **파이버**는 리액트 엘리먼트마다 만들어지는 상태 저장형 연결 리스트 노드로, `return`/`child`/`sibling`으로 트리를 이루고 `alternate`로 짝이 되는 다른 트리와 연결된다. 작업 루프는 이 트리를 **렌더 단계**(하향식 `beginWork`, 상향식 `completeWork`, 중단 가능)와 **커밋 단계**(변형·레이아웃, 중단 불가능)로 나눠 처리하며, 이 두 트리를 오가는 구조가 **더블 버퍼링**이다. 각 업데이트의 긴급도는 **lane**이라는 비트마스크로 표현되고, **React Scheduler**가 약 5ms 단위의 타임 슬라이싱으로 낮은 우선순위 lane의 작업에 양보 시점을 제공한다. `useTransition`은 바로 이 lane 시스템을 개발자가 직접 다룰 수 있게 해주는 API다.

## 📚 참고자료
- Tejas Kumar, 『전문가를 위한 리액트(Fluent React)』, O'Reilly
- [A description of React's new core algorithm, React Fiber – acdlite](https://github.com/acdlite/react-fiber-architecture)
- [useTransition – React](https://react.dev/reference/react/useTransition)
- [startTransition – React](https://react.dev/reference/react/startTransition)
