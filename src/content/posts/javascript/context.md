---
title: 실행 컨텍스트 (Execution Context)
published: 2025-06-14
tags: [Javascript]
category: Javascript
series: Javascript
draft: false
---
## 실행 컨텍스트 (Execution Context)

---

- 실행 컨텍스트란 자바스크립트 코드가 실행될 때 **식별자(변수, 함수 등)**를 어떻게 찾아서 동작할지를 결정하는 **환경(Context)**
- 자바스크립트가 코드를 어떻게 실행할지 결정하는 공간이며 그 안에 어떤 변수가 어디에 있는지, 어떤 함수를 실행해야 하는지를 담고 있는 **실행 정보 저장소**

### 실행 컨텍스트의 3가지 구성 요소

| 구성 요소 | 설명 |
| --- | --- |
| Variable Environment (변수 환경) | 변수 선언 정보 저장 (var, 함수 선언 등) |
| Lexical Environment (렉시컬 환경) | let, const 정보 및 스코프 체인 연결 |
| This Binding | 현재 컨텍스트의 this가 무엇인지 가리킴 |

## 실행 컨텍스트의 생성과 실행 단계

- 자바스크립트 엔진은 코드를 실행할 때 두 단계로 나눠서 처리
1. **생성 단계 (Creation Phase)**
    1. 실행 컨텍스트가 **생성**
    2. **호이스팅** (변수, 함수 선언이 메모리에 등록)
    3. **스코프 체인 및 this 결정**
2. **실행 단계 (Execution Phase)**
    1. **코드가 한 줄 씩 실행**
    2. **변수 할당, 함수 호출 등의 로직이 실제 작동**

### 실행 컨텍스트 스택 (콜 스택)

- 자바크스립트는 **싱글 스레드** 기반 언어
- 한 번에 하나의 컨텍스트만 실행
- 이 컨텍스트들은 **Stack(스택)** 구조로 관리하며 이를 **콜 스텍(Call Stack)**이라고 함

```jsx
function first() {
	second();
	console.log("first");
}

function second() {
	console.log("second");
}

first();
```

- 실행 컨텍스트의 흐름
    - `global()` → 가장 먼저 실행
    - `first()` 실행 → 컨텍스트 push
    - `second()` 실행 → 컨텍스트 push
    - `second()`  끝 → 컨텍스트 pop
    - `first()` 끝 → 컨텍스트 pop
    - `global()` 종료

## 호이스팅(Hoisiting)이란?

---

- 호이스팅이란 자바스크립트의 **변수와 함수 선언이 해당 범위의** **최상단으로 끌어올려지는 것처럼 동작**하는 현상
- 변수와 함수가 코드에 작성된 위치와 관계없이 **먼저 메모리에 등록되어 참조 가능하게 되는 것**을 의미

### 변수 선언의 호이스팅

```jsx
console.log(x); // undefined
var x = 10;
```

해당 코드는 호이스팅을 통해 다음과 같이 해석

```jsx
var x; // 선언만 호이스팅 됨
console.log(x); // undefined
x = 10; // 초기화는 호이스팅되지 않음
```

### 함수 선언의 호이스팅

```jsx
sayHi(); // TypeError: sayHi is not a function

var sayHi = function() {
	console.log("Say, Hi!");
}
```

해당 코드는 호이스팅을 통해 다음과 같이 해석

```jsx
var sayHi; // 선언 호이스팅
sayHi(); // sayHi는 아직 함수가 아님 -> 에러 발생
sayHi = function() {
	console.log("Say, Hi!");
}
```

## var vs. let/const의 호이스팅 차이

| 구분 | var | let/const |
| --- | --- | --- |
| 선언 | 호이스팅됨 | 호이스팅됨 |
| 초기화 시점 | undefined로 초기화 | 초기화 전까지 접근 불가 (TDZ 발생) |
| 사용 가능 시점 | 선언 이전에도 가능하지만 undefined | 선언 이전에 Reference Error |

### ❓ TDZ (Temporal Dead Zone)란?

- `let`이나 `const`로 선언된 변수는 호이스팅되지만
- 선언 이전에는 접근이 불가능한 일시적 사각지대(Temporal Dead Zone)에 존재함

```jsx
console.log(x); // Reference Error
let x = 10;
```

## 함수 선언 vs. 표현식 vs. 화살표 함수

| 종류 | 호이스팅 여부 | 정의 전 호출 가능 여부 |
| --- | --- | --- |
| 함수 선언식 (`function`) | 전체 호이스팅 | 가능 |
| 함수 표현식 (`var f = function`) | 선언만 호이스팅 | 불가능 |
| 화살표 함수 (`const f = () => {}`) | 선언만 호이스팅 | 불가능 |

## 스코프 체인 (Scope Chain)

---

- 스코프 체인은 자바스크립트에서 식별자를 찾을 때 현재 스코프뿐만 아니라 **외부(상위) 스코프까지 순차적으로 탐색하는 구조**를 의미
- 즉 어떤 변수를 사용할 때, 현재 스코프에서 찾고 → 없으면 상위 스코프로 올라가서 → 전역까지 찾는 **연결된 검색 과정**

### 스코프의 종류

| 스코프 종류 | 설명 |
| --- | --- |
| 전역 스코프 | 코드 어디서든 접근 가능 |
| 함수 스코프 | 함수 내부에서만 접근 가능 |
| 블록 스코프 | {} 내부 (ES6 이후 let, const만) |

```jsx
const a = 1;

function outer() {
	const b = 2;
	function inner() {
		const c = 3;
		console.log(a, b, c);
	}
	
	inner();
}

outer(); // 1, 2, 3
```

- a는 전역에서, b는 outer에서, c는 inner에서 찾는 이 참조 경로가 스코프 체인

### 실행 컨텍스트와의 관계

- 실행 컨텍스트는 Lexical Environment 정보를 갖고 있고 이 내부에는,
    - **Environment Record**: 현재 스코프의 변수/함수 정보
    - **Outer Environment Reference:** 상위 스코프에 대한 참조
- 를 가지고 있으며 이 Outer Reference를 따라 올라가는 구조가 **스코프 체인**

### 스코프 체인 탐색 시 주의 사항

- 하위 → 상위 방향으로만 탐색
    - 상위 스코프에서 하위 스코프는 알 수 없음
- 변수가 shadowing되면 가장 가까운 것을 사용

```jsx
const a = 1;

function test() {
	const a = 2;
	console.log(a); // 2가 가까우므로 2
}
```

- 클로저에서 활용됨
    - 함수가 종료되어도 상위 스코프 참조가 유지 → 클로저 생성