---
title: 자바스크립트의 데이터 타입
published: 2025-07-17
tags: [Javascript]
category: Javascript
series: Javascript
draft: false
---
## 자바스크립트의 데이터 타입

자바스크립트의 데이터 타입은 크게 두 가지로 나눌 수 있다. **기본형**과 **참조형**이다.

![자바스크립트 데이터 타입의 종류](@/assets/posts/javascript-datatype.png)

자바스크립트 데이터 타입의 종류

기본형과 참조형의 차이점은 기본형은 할당이나 연산 시 값이 담긴 주솟값을 바로 복제하는 반면 참조형은 값이 담긴 주솟값들로 이루어진 묶음을 가르키는 주솟값을 복제한다는 점이다. 이러한 점에서 기본형은 불변성을 띈다. 

### 메모리 구조와 불변성

```jsx
// 변수 a를 선언하고 'abc'라는 문자열을 할당
var a = 'abc';
// 기본형 값의 변경
a = a + 'def';
```

위 코드가 실행되었을 때 자바스크립트의 메모리는 다음과 같은 방식으로 작동한다.

먼저 변수 `a`를 선언하고 문자열 `‘abc’`를 할당하면, 자바스크립트 엔진은 `‘abc’`라는 문자열 값을 저장하기 위한 별도의 메모리 공간을 확보한 뒤, 이 공간의 주소값을 변수 `a`가 가르키도록 만든다. 즉, 변수는 문자열 자체를 저장하는 것이 아닌, 해당 문자열이 존재하는 메모리 공간을 참조(포인트)하게 된다.

이처럼 변수는 **값을 직접 담기보다는 값을 저장한 메모리 위치를 가르키는 식별자**로 동작한다.

그렇다면 이후 `a = a + 'def'`와 같이 문자열을 변경하려고 하면 어떻게 될까?

표현상으로는 기존 문자열 데이터 `'abc'`에 `'def'`가 “덧붙여져서” 변경된 것처럼 보이지만, 자바스크립트에서 문자열은 **불변(immutable)** 하기 때문에 기존 문자열을 수정하는 것이 아니라, **새로운 문자열** `'abcdef'`를 **생성**하고 이를 위한 메모리 공간을 다시 확보한다. 그리고 변수 `a`는 이제 새롭게 생성된 문자열을 가르키게 된다. 기존의 `'abc'`가 다른 곳에서 참조되지 않는다면 (참조 카운트가 0인 메모리 주소라면) 가비지 컬렉터의 수거 대상이 될 수 있다.

즉, 변경처럼 보이는 이 동작은 실제로는 **새로운 값을 생성해서 변수에 재할당하는 방식**으로 이루어진다. 이런 특성 때문에 문자열(String)을 포함한 자바스크립트의 **기본형 타입은 불변값**이라고 할 수 있다.

💡 **문자열은 문자 단위 변경도 불가능!**

```jsx
let s = 'abc';
console.log(s[0]) // a
s[0] = 'z';
console.log(s); // 'abc' => 문자열 내부 요소에 대한 직접 변경을 허용하지 않음
```

### 참조형 데이터의 가변성

참조형 데이터가 **가변(mutable)**하다고 하는 이유는 **값 자체가 아니라 값을 가르키는 “참조”를 통해 조작**되기 때문이다. 

```jsx
var obj1 = {
	a : 1,
	b : 'bbb'
}

var obj2 = obj1;

obj2.a = 2;

console.log(obj1.a); // 2
console.log(obj2.a); // 2
```

위 예제에서 `obj1`과 `obj2`는 **동일한 객체를 참조**하고 있다. 따라서 `obj2.a = 2`와 같이 `obj2`를 통해 객체의 프로퍼티를 변경하면, `obj1`을 통해 접근해도 변경된 값이 그대로 나타난다. 이는 두 변수가 **같은객체를공유하고 있기 때문에, 하나의 변경이 곧 전체 객체에 영향을 주는 것**이다. 

즉 참조형 데이터는 객체의 구조나 내용을 **자유롭게 수정할 수 있으므로** **가변성을 가진다**라고 볼 수 있다.

```jsx
var obj1 = {c : 10, d : 'ddd'}
var obj2 = obj1;

obj2 = {c : 20, d : 'ddd'}

console.log(obj1.c);
console.log(obj2.c);
```

이번 예제에서는 `obj2`에 **새로운 객체를 할당**함으로써 참조 대상 자체를 변경했다. 그러면 `obj2`는 더 이상 `obj1`과 같은 객체를 참조하지 않고, **완전히 새로운 객체를 가르키게 된다.** 따라서 `obj2.c`의 값을 변경해도 `obj1`에는 아무런 영향을 주지 않는다. 

이처럼, 참조형 데이터가 가변적이라는 말은 **참조된 객체의 내부 상태를 수정할 수 있다**는 의미이지 **참조 자체를 바꾸면 원본도 바뀐다**는 의미는 아니다. 즉 가변성이란 **참조형 데이터의 내부 프로퍼티를 변경할 수 있다는 특성**을 의미한다. 

## 불변 객체 (Immutable Object)

불변 객체는 최근의 React, Vue.js, Angular 등의 라이브러리나 프레임워크, 함수형 프로그래밍, 디자인 패턴 등에서 매우 중요한 기초가 되는 개념이다. 객체의 불변성이 필요한 이유는 주로 **예측 가능성, 디버깅 용이성, 성능 최적화, 사이드 이펙트 방지** 등과 관련이 있다.

### 상태 변경 추적 및 예측 가능성 향상

객체가 불변이라면 객체가 변경되었을 때 **이전 상태와 이후 상태가 명확하게 구분**된다. 

```jsx
const oldObject = {a : 1, b : 2}
const newObject = {...oldObject, b : 3}
```

예제에서는 `oldObject`를 직접 수정하지 않고, 변경된 값을 반영한 새로운 객체를 `newObject`에 생성했다. 이런 방식은 **디버깅, 타임 트래블 디버깅, Undo/Redo 기능** **구현**에 매우 유리하다.

### 사이드 이펙트(side effects) 방지

같은 객체를 여러 곳에서 참조하고 있을 때, 어느 한 곳에서 누군가가 그 객체를 변경하면 **예상치 못한 버그**가 발생할 수 있다.

```jsx
const user = { name : 'user 1', age : 20 }

function updateName(user) {
	user.name = 'Changed user name'; // 원본 user까지 바뀜
}

function updateNameImmutably(user) {
	return {...user, name : 'Changed user name'} // 원본은 그대로 유지한 채 새로운 객체 반환
}
```

불변 객체는 **원본을 건드리지 않고 새로운 객체를 생성**하므로써 사이드 이펙트 없이 안정적으로 동작한다.

### React, Redux 등 상태 관리 라이브러리에서는 필수 개념

React는 `state`가 변경되어야만 컴포넌트를 리렌더링한다. 이때 객체를 **불변으로 다뤄야** **이전 상태와의 얕은 비교(shallow compare)**가 가능하다. 

Redux 또한 `store`의 상태를 직접 변경하지 않고, 항상 **새 상태를 반환**해야 하므로 불변성 유지가 핵심이다.

### 성능 최적화

불변 객체는 참조 값만 비교해도 같고 다름을 판단할 수 있다. 이를 통해 **깊은 비교(deep compare)** 대신 **얕은 비교(shallow compare)**만으로도 성능 최적화를 이룰 수 있다.

```jsx
if (prevState == nextState) {
	// 상태가 변경되지 않았으므로 리렌더링 생략 가능
}
```

## 얕은 복사와 깊은 복사

불변성을 유지하기 위해 기존 객체의 값을 복사하는 방법에는 **얕은 복사(shallow copy)**와 **깊은 복사(deep copy)**가 있다.

**얕은 복사**는 바로 아래 단계의 값만 복사하는 방법이고, **깊은 복사**는 내부의 모든 값들을 하나하나 재귀적으로 순회하며 전부 복사하는 방법이다. 

### 얕은 복사 (Shallow Copy)

```jsx
// 얕은 복사
const shallowObj1 = {
	a : 1,
	b : 2,
	c : 3
}

const shallowObj2 = {
	name : "user1",
	urls : {
		portfolio: "http://github.com/user1",
		facebook: "http://facebook.com/user1"
	}
}

const copyObj1 = {...shallowObj1}
const copyObj2 = {...shallowObj2}

copyObj1.a = 4

console.log(shallowObj1.a === copyObj1.a) // false

copyObj2.name = "changed user"

console.log(shallowObj2.name === copyObj2.name) // false

copyObj2.urls.portfolio = "https://notion.com/user1"

console.log(shallowObj2.urls.portfolio === copyObj2.urls.portfolio) // true
```

위 예제에서 사용된 **Spread 연산자**는 ES6부터 도입된 문법으로, 배열이나 객체의 전체 또는 일부를 다른 배열이나 객체로 빠르게 복사할 수 있게 해준다. 

하지만 이때 주의할 점은 Spread 연산자를 활용한 복사는 **앝은 복사**라는 점이다.

위 예제에서 `shallowObj1`과 `shallowObj2`를 각각 `copyObj1`과 `copyObj2`라는 객체에 복사했다. 

- `copyObj1.a`와 `copyObj2.name`은 기본형 데이터를 가진 프로퍼티이므로, 복사 후 값을 변경해도 원복 객체에 영향을 주지 않는다.
- 그러나 `copyObj2.urls` **참조형 데이터**를 ****가진 프로퍼티이다. 얕은 복사 시 이 객체의 참조만 복사되므로, 복사한 객체에서 내부 값을 변경하면 **원본 객체에도 영향을 준다.**

따라서 참조형 데이터를 포함한 객체의 불변성을 유지하기 위해서는 반드시 **깊은 복사**를 통해 하위의 중첩 객체들에 대해서도 재귀적인 복사를 진행하여 원본과 완전히 분리해야만 불변성을 유지할 수 있다.

**💡 대표적인 얕은 복사 방법**

**Spread 연산자** 외에도 자바스크립트에서 얕은 복사를 수행할 수 있는 방법은 여러 가지가 있다. 대표적으로 아래 두 가지 방법이 자주 사용된다:

1. **Array.prototype.slice()**
    1. 배열을 얕은 복사하려면  `slice` 메서드를 활용할 수 있다. 이 메서드는 start부터 end까지의 요소를 기존 배열에서 추출하여 새로운 배열을 반환한다.
    2. Array.prototype.slice() 활용 예시
    
    ```jsx
    const users = [
    	{name : 'user1', age : 20},
    	{name : 'user2', age : 21},
    	{name : 'user3', age : 22},
    	{name : 'user4', age : 23},
    	{name : 'user5', age : 24},
    ]
    const copyUsers = users.slice()
    
    copyUsers[0].name = 'changed user1'
    
    console.log(JSON.stringify(users) === JSON.stringify(copyUsers)); // true
    console.log(users[0].name === copyUsers[0].name) // true
    
    // 내부 객체가 공유되므로, 원본 배열 요소도 변경됨
    ```
    
- ⚠️ 배열 자체는 새로 복사되지만, 내부 객체는 같은 주소를 참조하므로 변경 시 원본도 영향을 받는다.
1. **Object.assign(target, source)**
    1.  `Object.assign()`은 하나 이상의 소스 객체로부터 속성을 복사하여 target 객체에 할당하는 메서드이다. 이때도 내부 객체까지는 복사되지 않기 때문에 **얕은 복사**만 수행된다.
    2. Object.assign() 활용 예시
    
    ```jsx
    const object = {
    	a : 1,
    	b : [1, 2, 3]
    }
    
    const copyObj = Object.assign({}, object);
    
    console.log(object.b === copyObj.b); // true
    
    copyObj.b[0] = 0
    
    console.log(object.b === copyObj.b); // true
    console.log(object.b[0] === copyObj.b[0]); // true
    
    // 이때도 마찬가지로 object 내부에 있던 참조형 데이터를 가진 b를 변경했을 때
    // 원본 객체 역시 함께 변경되고 있음을 알 수 있다.
    ```
    

### 깊은 복사 (Deep Copy)

```jsx
const copyObjectDeep = function (target) {
	let result = {}
	if (typeof target === 'object' && target !== null) {
		for (let prop in target) {
			result[prop] = copyObjectDeep(target[prop]); // 재귀적으로 복사를 수행
		}
	} else {
		result = target;
	}
	return result;
}
```

위 예제는 범용적으로 객체의 깊은 복사를 수행할 수 있는 재귀 함수이다. 

만약 `target`이 객체인 경우에는 내부 프로퍼티들을 순회하며 `copyObjectDeep()` 함수를 재귀적으로 호출하고, 객체가 아닌 경우에는 원시 값을 그대로 반환한다. 

이를 통해 원본과 복사본이 **서로 다른 참조를 가지게 되어,** 어느 한 쪽의 (참조형 데이터를 가진) 프로퍼티를 변경해도 **다른 쪽에 전혀 영향을 주지 않게 된다.**

**💡 깊은 복사를 수행하는 주요 방법**

자바스크립트에서는 위의 `copyObjectDeep`처럼 별도로 깊은 복사를 구현하지 않고도 깊은 복사를 할 수 있는 방법을 제공하고 있다:

1. **JSON.stringify() + JSON.parse()**
    1. 가장 간단한 방법은 객체를 JSON 문자열로 변환한 뒤 다시 객체로 파싱하여 깊은 복사를 수행하는 방식이다.
    2. JSON.stringify → JSON.parse 예시
    
    ```jsx
    const object = {
    	a : 1,
    	b : [1, 2, 3]
    }
    
    const copyObject = JSON.parse(JSON.stringify(object))
    
    copyObject.b[0] = 0
    
    console.log(object.b === copyObject.b) // false
    console.log(object.b[0] === copyObject.b[0]) // false
    
    // 복사한 copyObject에서 참조형 데이터를 변경했음에로
    // 원본 객체에는 영향이 가지 않는 것을 확인할 수 있다
    ```
    
- ⚠️ 다만 JSON.stringify를 활용한 깊은 복사 방식에는 몇 가지 한계가 존재한다.
    - JSON 문자열로 변환이 불가능한 메서드(함수)나 숨겨진 프로퍼티인 __proto__, getter/seter, Date, RegExp, Map, Set과 같이 변환할 수 없는 객체 등의 속성은 **모두 무시하거나 제거**된 채 변환하는 제약사항이 있다.
    - 순환 참조가 있는 객체는 **오류가 발생**할 수 있다.
    - 이러한 깊은 복사 방식은 httpRequest로 받은 데이터를 저장한 객체를 복사하는 등 순수한 정보만 다룰 때 활용하기 좋은 방법이다.
1. **structuredClone()**
    1. `structuredClone` 메서드는 자바스크립트에서 structured clone 알고리즘을 사용하여 주어진 값의 깊은 복사를 생성하는 전역 함수이다. ES2021 이후부터 사용할 수 있는 **공식적인 깊은 복사 전역 함수이다.** 이 메서드는 새로운 객체로 복사하는 대신 원래 값에서 전송 가능한 객체를 전송할 수도 있다. 
    2. structuredClone 예시
    
    ```jsx
    const object = {
    	a : 1,
    	b : [1, 2, 3]	
    }
    
    const copyObject = structuredClone(object)
    
    copyObject.b[0] = 0
    
    console.log(object.b === copyObject.b) // false
    console.log(object.b[0] === copyObject.b[0]) // false
    ```
    
- ⚠️ JSON.stringify와 마찬가지로 몇 가지 제약사항이 존재한다.
    - 함수 속성은 복사되지 않고 undefined로 처리된다.
    - 객체의 프로토타입 체인을 유지하지 않는다.
    - 다만 JSON을 활용한 깊은 복사 방법과는 달리 Date, RegExp 등 다양한 타입을 지원한다는 차이점이 있으며 성능적으로 JSON 문자열을 활용하는 방법보다는 빠르다는 장점이 있다. 또한 순환 참조를 지원한다.
    - 최신 브라우저 또는 Node.js 환경에서 안전하고 빠른 깊은 복사를 할 때 활용하기 좋은 방법이다.
1. **Lodash 라이브러리의 cloneDeep()**
    1. Lodash 라이브러리를 활용한다면 `cloneDeep()` 메서드를 통해 깊은 복사를 진행할 수 있다. Lodash 외에도 **Immer, immutable.js** 등 다양한 라이브러리들에서 깊은 복사를 위한 방법을 제공하고 있다.
    2. Lodash의 cloneDeep()을 활용한 예제
    
    ```jsx
    import cloneDeep from 'lodash/cloneDeep'
    
    const object = {
    	a : 1,
    	b : [1, 2, 3],
    	f : { g : { h : 100 } } 
    }
    
    const copyObject = cloneDeep(object);
    
    copyObject.f.g.h = 999;
    
    console.log(object.f.g.h) // 100
    ```
    
    - 이러한 라이브러리들은 앞서 언급한 방식에서 제한적이었던 프로토타입 체인 유지, 순환 참조와 다양한 타입 지원 등 폭넓은 깊은 복사 방식을 지원한다.
    - 다만 외부 라이브러리에 의존해야 한다는 단점이 존재한다. 또한 번들 크기에 영향을 줄 수 있다.
    - 복잡한 구조의 데이터에 대한 깊은 복사가 필요한 경우 안정적으로 사용할 수 있다.

### 📚 참고 자료

---

- [코드 자바스크립트](https://ko.javascript.info/js)