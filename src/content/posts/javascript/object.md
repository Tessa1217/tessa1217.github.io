---
title: 🧩 Javascript의 Object
published: 2025-08-20
tags: [Javascript]
category: Javascript
series: Javascript
draft: false
---
# 🧩 Javascript의 Object

최근 **깊은 복사를 위한 유틸리티 함수**를 직접 구현하라는 과제를 받았다.

단순한 객체만이 아니라 Array, Map, Set은 물론 커스텀 객체까지 모두 제대로 복사할 수 있어야 했다. 처음엔 키-값 쌍을 재귀적으로 새 객체에 복사하는 방식으로 해결하려 했지만, 곧 다양한 객체 타입을 정확히 다루려면 자바스크립트의 Object, 속성 구조, 그리고 프로토타입 체계에 대한 이해가 필수적이었다.

이 과정에서 내가 객체 시스템에 대해 얼마나 피상적으로 알고 있었는지 실감했고, 이를 계기로 관련 개념을 깊이 있게 학습하고 정리해보게 되었다.

---

## 1) Object란?

객체는 **데이터와 기능(메서드)**를 **키–값** 쌍으로 저장하는 구조.

```jsx
const person = {
  name: 'MyName',
  age: 25,
  greeting() {
    console.log("Hello, my name is", this.name);
  }
};
```

- `name`, `age`는 **속성(property)**, `greeting`은 **메서드(method)**.

---

## 2) 객체 생성 방법

### ✅ 리터럴

```jsx
const obj = { key: 'value', number: 12345 };
```

### ✅ 생성자 & Object.create

```jsx
const obj = new Object();
obj.key = 'value';
obj.number = 12345;

// 원본의 프로토타입/디스크립터까지 유지 복사
const clone = Object.create(
  Object.getPrototypeOf(obj),
  Object.getOwnPropertyDescriptors(obj)
);
```

- **`Object.create(proto, descriptors)`**: 지정한 프로토타입과 디스크립터로 새 객체 생성.

### ✅ 클래스

```jsx
class Person {
  constructor(name, age) { this.name = name; this.age = age; }
}
const personA = new Person("a", 27);
```

### ✅ Getter/Setter

```jsx
const person = {
  firstName: 'value',
  lastName: '12345',
  get fullName() { return `${this.firstName} ${this.lastName}`; },
  set fullName(name) { [this.firstName, this.lastName] = name.split(" "); }
};
```

---

## 3) 속성 접근

```jsx
const a = new Person("a", 27);
console.log(a.name);      // 점 표기법
console.log(a['name']);   // 대괄호 표기법
```

---

## 4) 속성 추가/수정/삭제

```jsx
const p = new Person("a", 27);
p.job = 'Developer'; // 등록
p.age = 26; // 변경
delete p.job; // 삭제
```

### Object.defineProperty / defineProperties

```jsx
const p2 = new Person("a", 27);
Object.defineProperties(p2, {
  job: { value: 'Developer', writable: true, configurable: true, enumerable: true }
});
Object.defineProperty(p2, "favoriteFood", {
  value: 'apple', writable: true, configurable: true, enumerable: true
});
```

- 디스크립터로 **쓰기/열거/재정의 가능 여부**를 제어.
    - writable
    - configurable
    - enumerable

---

## 5) 변경 제약: freeze / seal (＋preventExtensions)

- `Object.freeze()` 메서드를 통해 객체를 **동결**하게 되면 동결된 객체는 더 이상 새로운 속성을 추가하거나 제거하는 것을 방지한다.
    - 존재하는 속성의 불변성, 설정 가능성, 작성 가능성이 변경되는 것 역시 방지되며, 존재 속성의 값을 수정하는 것도 방지한다.
- `Object.seal()` 메서드를 통해 객체를 **밀봉**하게 되면 새로운 속성의 추가가 불가능하며 현재 존재하는 속성을 설정 불가능 상태로 만들어준다. 다만 **쓰기 가능한 속성**에 대한 속성의 값은 밀봉 후에도 변경할 수 있다는 점에서 `Object.freeze()` 메서드와 차이가 있다.
- 동결/밀봉 여부는 `Object.isFrozen()`, `Object.isSealed()` 메서드를 통해 확인할 수 있다.

```jsx
const f = Object.freeze(new Person("a", 25));
f.age = 27; // 무시
console.log(Object.isFrozen(f), Object.isSealed(f)); // true, true

const s = Object.seal(new Person("a", 26));
s.age = 27; // 가능(속성 writable이면)
console.log(Object.isFrozen(s), Object.isSealed(s)); // false, true
```

| 메서드 | 새 속성 추가 | 기존 속성 삭제 | 값 변경(쓰기) | 디스크립터 변경 |
| --- | --- | --- | --- | --- |
| `Object.freeze` | ❌ | ❌ | ❌ | ❌ |
| `Object.seal` | ❌ | ❌ | ✅ (writable일 때) | 일부 ❌ |
| `Object.preventExtensions` | ❌ | ✅ | ✅ | ✅ |

---

## 6) 속성 보유 확인: `hasOwnProperty` vs `Object.hasOwn`

객체가 특정 속성을 자신의 속성으로 가지고 있는지 확인할 때 `hasOwnproperty` 메서드나 ES2022에 새로 추가된 정적 메서드인 `Object.hasOwn()`을 사용할 수 있다.

```jsx
class Person {
  constructor(name, age) { this.name = name; this.age = age; }
  greeting() { console.log("Hi", this.name); }
}
const p = new Person("person", 25);
p.sayHello = function() { console.log("Hello", this.name); };

console.log(p.hasOwnProperty("name"));      // true
console.log(Object.hasOwn(p, "name"));      // true
console.log(Object.hasOwn(p, "sayHello"));  // true
console.log(Object.hasOwn(p, "greeting"));  // false (프로토타입의 메서드)
```

- `Object.hasOwn(obj, key)`는 **안전한 정적 메서드** (인스턴스가 `hasOwnProperty`를 오버라이드해도 안전).

### 🧬 프로토타입 체인

**🔗 프로토타입**

- 자바스크립트의 모든 객체는 내부 `[[Prototype]]` 링크를 가진다.
- 이러한 구조를 통해 객체는 클래스 기반 언어의 상속처럼 공통 속성과 메서드를 상속받거나 참조할 수 있으며, 이렇게 형성된 상속 구조를 **프로토타입 체인(Prototype Chain)**이라고 한다.
    - **없으면 자기 자신 → 프로토타입 → … → Object.prototype → null**로 탐색.

```jsx
const arr = [1, 2, 3];
console.log(arr.toString()); // "1,2,3"  (Array.prototype → Object.prototype)
```

**🛠️ 생성자 함수와 프로토타입**

- new 연산자로 객체를 생성하면 해당 객체는 생성자 함수의 prototype 속성을 자신의 [[Prototype]]으로 연결한다.
    
    ```jsx
    function Person(name) { 
    	this.name = name; 
    }
    Person.prototype.sayHello = function () { console.log(`Hi, I'm ${this.name}`); };
    const p1 = new Person("Alice");
    console.log(p1.sayHello === Person.prototype.sayHello); // true
    ```
    

---

## 7) 속성 설명자 조회

```jsx
const p = new Person("a", 1);
Object.defineProperty(p, "job", {
  value: 'Developer', writable: true, configurable: false, enumerable: true
});
console.log(Object.getOwnPropertyDescriptors(p));
```

- `getOwnPropertyDescriptors`로 **모든 own 속성의 디스크립터**를 조회할 수 있다.

---

## 8) 객체 순회

**for…in**

```jsx
// for...in (자신 + 상속된 enumerable 포함)
for (const k in p) console.log(k, p[k]);
```

**Object.keys / values / entries**

```tsx
// own enumerable만
Object.keys(p);    // 키 배열
Object.values(p);  // 값 배열
Object.entries(p); // [키,값] 배열  (정수 키 → 문자열 키 삽입 순서)
```

### 열거 불가능/심볼 키

```jsx
const obj = { a: 1, b: 2 };
Object.defineProperty(obj, "c", { value: 3, enumerable: false });

console.log(Object.keys(obj));               // ['a','b']
console.log(Object.getOwnPropertyNames(obj));// ['a','b','c']

const sym = Symbol('id');
const o2 = { [sym]: 123, a: 1 };
console.log(Object.getOwnPropertySymbols(o2)); // [Symbol(id)]
```

---

## 9) Map ↔ Object 변환

- `Object.entries()`는 for...in과 같은 순서로 주어진 객체 자체의 **enumerable** 속성을 키-값 쌍 형태의 배열로 반환한다. 이를 활용하여 Map으로 변환할 수 있다.
- 반대로 `Object.fromEntries`는 키-값 쌍 형태의 배열을 객체로 변환해주는 메서드이다. 이를 통해 Map을 Object로 변환할 수 있다.

```jsx
const object = { a: 1, b: 2, c: 3 };
const map = new Map(Object.entries(object));   // Object → Map

const fromMap = new Map([["a",1],["b",2]]);
const toObj = Object.fromEntries(fromMap);     // Map → Object
```

---

## 10) 객체 복사

### ✅ 얕은 복사

```jsx
const copy1 = Object.assign({}, person);
const copy2 = { ...person };

// 프로토타입/비열거 속성까지 살리는 얕은 복사
const copyAll = Object.create(
  Object.getPrototypeOf(person),
  Object.getOwnPropertyDescriptors(person)
);
```

- **얕은 복사 시에는 중첩 객체는 참조 공유** → 내부 변경이 원본에 전파한다.
- `Object.assign()`을 통한 복사는 목표 객체로 열거 가능한 속성과 객체의 속성들만 복사할 수 있지만 `Object.getPrototypeOf`, `Object.getOwnPropertyDescriptors`를 활용하면 열거 불가능한 속성들도 복사할 수 있다.

### ✅ 병합

```jsx
const m = Object.assign({}, obj1, obj2); // 뒤에 온 키가 덮어씀
const m2 = { ...obj1, ...obj2 };
```

- Spread 연산자, Object.assign 메서드를 활용해 다중 객체를 병합할 수 있다. 다만 후자의 키가 덮어씌워지기 때문에 병합 순서에 유의해야 한다.

### ✅ 깊은 복사 (주의: JSON 기반 한계)

```jsx
const deepCopied = JSON.parse(JSON.stringify(person));
```

- JSON.parse, JSON.stringify를 활용하여 간단하게 깊은 복사를 할 수 있다. 다만 JSON.parse, JSON.stringify를 사용한 깊은 복사에는 여러가지 제약사항 및 한계가 존재한다.
    - **무시/불가**: `undefined`, `Function`, `Symbol`, `BigInt(에러)`, `Map/Set`, `Date/RegExp(손실)`, 순환 참조(에러).
    - 대안: **직접 구현** 또는 `lodash.cloneDeep`.

---

## 11) 객체 비교

객체는 참조형 타입으로 내부 속성의 값이 동일하더라도 참조값이 다를 경우 서로 다른 객체로 간주된다.

```jsx
const person = new Person("a", 26);
const copy = { ...person };
console.log(person === copy); // false (참조 비교)
```

### “값 동등” 검사(간단 용도)

```jsx
JSON.stringify(person) === JSON.stringify(copy);
```

- 객체 내부 속성 값이 실제로 동일한지 비교할 때, JSON.stringify를 활용할 수 있다. 단, JSON.stringify는 **함수, undefined, Symbol** 등은 무시하고, 속성 순서가 다를 경우 다른 경우가 나올 수 있다.
    - 함수/undefined/Symbol 무시, 키 순서 영향 가능.
    - **정교한 비교**는 `lodash.isEqual` 등 사용 권장.

---

### 📚 참고 자료

- [MDN Object](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object)
- [W3School Objects](https://www.w3schools.com/js/js_objects.asp)
- [Javascript.Info Object](https://ko.javascript.info/object)