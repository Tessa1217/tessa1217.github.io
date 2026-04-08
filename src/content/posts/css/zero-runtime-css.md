---
title: ❓ Zero-runtime CSS-in-JS?
published: 2025-09-30
tags: [CSS, Side Project, Benchmark]
category: CSS
series: CSS
draft: false
---
# ❓ Zero-runtime CSS-in-JS?

최근 진행하고 있는 사이드 프로젝트에서 디자인 토큰 시스템을 개발하면서 **Vanilla Extract**를 도입하게 되었다. 이 과정에서 자연스럽게 ***“Zero-runtime CSS-in-JS**란 무엇일까?”*라는 의문이 생겼고, 관련 개념을 공부하다 보니 CSS-in-JS의 등장 배경과 Zero-runtime CSS-in-JS가 어떤 한계를 보완하기 위해 나왔는지 이해할 필요가 있었다.

전통적인 CSS의 한계를 극복하기 위해 등장한 CSS-in-JS, 그리고 그 단점을 보완하기 위해 진화한 Zero-runtime CSS-in-JS까지의 흐름을 정리하고, 두 접근 방식이 실제로 어떤 차이를 보이는지 간단한 벤치마킹을 진행해 보았다.

---

## 전통적인 CSS의 문제점

브라우저의 렌더링 동작 방식(Critical Rendering Path)은 다음과 같다:

1. 서버로부터 HTML을 다운로드 받고 DOM 트리를 생성한다.
2. CSS 파일을 다운로드하고 CSSOM 트리를 생성한다.
3. DOM 트리와 CSSOM 트리를 결합하여 Render Tree를 생성한다.
4. Render Tree를 기반으로 레이아웃과 페인트 작업을 거친다. 

Render Tree는 실제로 화면에 그려질 요소와 스타일 정보를 결합한 구조다.

이때 CSSOM이 완성되지 않으면 렌더링이 지연되며, CSS 파일이 크거나 선택자가 많을수록 CSSOM 생성 시간이 길어져 사용자 경험을 해칠 수 있다.

이를 개선하기 위한 한 가지 방법으로는 CSS를 작은 단위로 분리해 CSSOM을 빠르게 구성하는 방식을 고려할 수 있다. 하지만 이렇게 분할된 파일이 늘어나면 늘어날수록 관리 포인트가 많아지고, 전역 스코프 문제로 인해 클래스/선택자 충돌 위험이 커지는 단점이 존재한다. 컴포넌트 수가 많아질수록 이 문제는 더 심각해질 것이다.

이러한 한계를 보완하기 위해 등장한 것이 바로 **CSS-in-JS다.**

---

## 🎨 CSS-in-JS

CSS-in-JS는 컴포넌트 안에서 자바스크립트와 함께 CSS를 작성할 수 있게 해준다.

이 방식은 직관적이고, 여러 CSS 파일을 따로 관리하는 것보다 유지보수가 용이하다.

```tsx
import styled from "styled-components";

const StyledButton = styled.button`
  padding: 0.75em 1em;
  border-radius: 4px;
  background-color: ${ ({ primary }) => ( primary ? "blue" : "gray" ) };
  color: white;
  &:hover {
    background-color: #111;
  }
`;

export default StyledButton;

// App
import StyledButton from './components/StyledButton'

function App() {
	return (
		<div className="app">
			<StyledButton>Button</StyledButton>
			<StyledButton primary>Primary Button</StyledButton>			
		</div>
	)
}
```

CSS-in-JS의 가장 큰 장점은 **스코프 문제 해결**이다. 스타일이 컴포넌트 로컬 스코프에 선언되므로 다른 컴포넌트와 충돌할 염려가 없다.

또한 props에 따라 동적으로 스타일을 지정할 수 있어 동적 스타일링이 가능한 점도 큰 장점이다.

### CSS-in-JS의 단점

이처럼 CSS-in-JS는 전통적인 CSS의 한계를 극복하기 위해 탄생했지만 모든 문제를 완벽히 해결하지는 못한다.

CSS-in-JS는 런타임 시점에 라이브러리가 스타일을 해석하고 적용하기 때문에 런타임 오버헤드가 발생한다. 또한 전통적인 CSS는 브라우저 캐싱의 이점을 누릴 수 있지만, 런타임에 동적으로 생성되는 CSS-in-JS는 캐싱 효과를 기대하기 어렵다.

---

## 🧩 Zero-runtime CSS-in-JS의 등장

**Zero-runtime CSS-in-JS**는 CSS-in-JS의 단점을 보완하기 위해 등장했다.

대표적으로 **Vanilla Extract**, **Panda CSS** 등이 있다.

Vanilla Extract는 빌드 시점에 className을 확정하고 CSS 파일을 정적으로 출력한다. 따라서 런타임에는 단순히 문자열을 바인딩하는 것만으로 빠르게 스타일을 적용할 수 있다.

즉, CSS-in-JS가 제공하는 생산성과 확장성은 유지하면서도, 런타임 오버헤드를 제거하여 성능 문제를 보완하는 방식이다.

---

# 🆚 CSS-in-JS vs. Zero-runtime CSS-in-JS 벤치마크

그렇다면 실제로 CSS-in-JS와 Zero-runtime CSS-in-JS는 얼마나 차이가 있을까? 
이를 확인하기 위해 **React + Vite 환경에서 3,000개의 Box 컴포넌트**를 렌더링하고, **Styled-components**와 **Vanilla Extract**를 비교하는 간단한 벤치마킹을 진행했다.

---

## 📃 벤치마크 시나리오

- React + Vite 환경에서 3,000개의 Box 컴포넌트 렌더링
- Styled-components vs. Vanilla Extract 비교

![CSS-in-JS 벤치마크](@/assets/posts/css-in-js-benchmark.png)

CSS-in-JS 벤치마크

### 코드 예시

**Vanilla Extract를 활용한 Box Style**

```tsx
// zero-runtime.css.ts
import { style } from "@vanilla-extract/css";

export const box = style({
  width: "24px",
  height: "24px",
  borderRadius: "4px",
});

export const variants = [
  style({ background: "#1e3a8a" }),
  style({ background: "#2563eb" }),
  style({ background: "#16a34a" }),
  style({ background: "#d97706" }),
  style({ background: "#dc2626" }),
];

export const bordered = style({
  outline: "1px solid rgba(0,0,0,.08)",
});

export const shadowed = style({
  boxShadow: "0 1px 2px rgba(0,0,0,.12)",
});

// ZeroRuntimeBoxes.tsx
import * as styles from "../zero-runtime.css";

type BoxProps = {
  variant: number;
  bordered: boolean;
  shadowed: boolean;
};

function Box({ variant = 0, bordered = false, shadowed = false }: BoxProps) {
  const boxStyles = [
    styles.box,
    styles.variants[variant],
    bordered ? styles.bordered : "",
    shadowed ? styles.shadowed : "",
  ].join(" ");
  return <div className={boxStyles}></div>;
}

type ZeroRuntimeBoxesProps = {
  N: number;
  variantSeed: number;
};

export default function ZeroRuntimeBoxes({
  N,
  variantSeed,
}: ZeroRuntimeBoxesProps) {
  const items = new Array(N).fill(0).map((_, i) => (i + variantSeed) % 5);
  const bordered = variantSeed % 2 === 0;
  const shadowed = variantSeed % 3 === 0;
  return items.map((v, i) => (
    <Box key={i} variant={v} bordered={bordered} shadowed={shadowed} />
  ));
}

```

**Styled Component를 활용한 Box Style**

```tsx

import styled from "styled-components";

type BoxProps = {
  variant: number;
  bordered?: boolean;
  shadowed?: boolean;
};

const Box = styled.div<BoxProps>`
  width: 24px;
  height: 24px;
  border-radius: 4px;
  background: ${(p) =>
      ["#1e3a8a", "#2563eb", "#16a34a", "#d97706", "#dc2626"][p.variant]}
    ${(p) => (p.bordered ? "outline: 1px solid rgba(0,0,0,.08);" : "")}
    ${(p) => (p.shadowed ? "box-shadow: 0 1px 2px rgba(0,0,0,.12);" : "")};
`;

type StyledBoxesProps = {
  N: number;
  variantSeed: number;
};

export default function StyledBoxes({ N, variantSeed }: StyledBoxesProps) {
  const items = new Array(N).fill(0).map((_, i) => (i + variantSeed) % 5);
  const bordered = variantSeed % 2 === 0;
  const shadowed = variantSeed % 3 === 0;
  return items.map((v, i) => (
    <Box key={i} variant={v} bordered={bordered} shadowed={shadowed} />
  ));
}

```

- React DevTools Profiler로 렌더링 시간 측정

![Styled Components](@/assets/posts/benchmark-profiler1.png)

Styled Components

![Vanilla Extract](@/assets/posts/benchmark-profiler2.png)

Vanilla Extract

![Styled-components Mount, Update 시간 출력](@/assets/posts/mount-time1.png)

Styled-components Mount, Update 시간 출력

![Vanilla Extract Mount, Update 시간 출력](@/assets/posts/mount-time2.png)

Vanilla Extract Mount, Update 시간 출력

<aside>
💡<strong>벤치마크 결과</strong>

**(렌더링 시간 비교)**

- **Styled-components** 렌더링 시간: 339.3ms
- **Vanilla Extract** 렌더링 시간: 284.5ms
- 약 **16% 차이**

**(Console 측정 기준)**

- **Mount**: Styled-components는 0.50ms, Vanilla Extract는 0.20ms → 초기 렌더링에서 약 2.5배 차이
- **Update**: Styled-components는 0.72ms, Vanilla Extract는 0.08ms → 업데이트에서 약 9배 차이

👉 벤치마크 결과 측정 결과 상 Zero-runtime 방식이 런타임 오버헤드를 줄인다는 점을 확인할 수 있다. 

</aside>

---

# ✏️ 벤치마킹을 진행하면서…

사실 CSS-in-JS를 활용한 경험이 전무했던 터라, 두 접근 방식이 어떤 차이가 있는지, 그리고 DX(Developer Experience) 측면에서 각각의 장단점이 무엇인지 뚜렷하게 알지 못한 상태에서 이번 벤치마킹을 시작했다.

여전히 거대한 `global.css` 빌드 결과물의 크기를 줄이기 위해 고군분투하고, 컴포넌트 단위 CSS 모듈화를 두고 치열하게 고민하는 입장에서 이번 미니 벤치마킹은 굉장히 신선한 경험이었다.

특히 컴포넌트 내부에서 스타일을 정의하고 조건부 처리나 variant 기반의 디자인을 적용할 수 있다는 점에서 **DX적인 장점**을 크게 체감할 수 있었다.

벤치마킹을 진행하며 느낀 두 방식의 차이점은,

- **Styled-components**는 이미 많은 회사에서 활용되는 만큼 생태계와 플러그인이 풍부하고, 직관적인 API를 통해 스타일을 작성할 수 있는 점이 매력적으로 다가왔다. 하지만 런타임 오버헤드라는 태생적인 성능 문제는 피하기 어렵고, 컴포넌트 단위가 커질수록 스타일 코드가 함께 섞이면서 직관성이 떨어질 수 있겠다는 우려도 생겼다.
- **Vanilla Extract**는 CSS-in-JS의 런타임 오버헤드 문제를 최소화하면서도, 현재 설계 중인 **디자인 토큰 시스템과 연동**하여 테마 관리와 확장을 손쉽게 할 수 있다는 장점이 있었다. 이런 이유로 이번 프로젝트에서는 Vanilla Extract를 도입하기로 결정했다.

전통적인 CSS 방식을 고수하던 관성에서 벗어나, 다양한 스타일링 접근 방식을 직접 경험해 볼 수 있었다는 점에서 이번 벤치마킹은 매우 유익한 시간이었다.

---

## 📚 참고자료

- [https://junghan92.medium.com/번역-우리가-css-in-js와-헤어지는-이유-a2e726d6ace6](https://junghan92.medium.com/%EB%B2%88%EC%97%AD-%EC%9A%B0%EB%A6%AC%EA%B0%80-css-in-js%EC%99%80-%ED%97%A4%EC%96%B4%EC%A7%80%EB%8A%94-%EC%9D%B4%EC%9C%A0-a2e726d6ace6)
- https://blog.logrocket.com/css-vs-css-in-js/#render-blocking-css
- https://vanilla-extract.style/