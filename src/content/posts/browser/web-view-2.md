---
title: WebView Bridge 통신 Part 2
published: 2026-04-12
tags: [Browser, Javascript, Nuxt, Webview]
category: Browser
series: WebView
draft: false
---
# WebView Bridge 통신 - Part 2: 실전 구현

> webview-study 프로젝트의 Bridge 아키텍처 — 설계 결정부터 컴포넌트 연동까지
**Part 1**: [개요 및 이론](/posts/browser/web-view-1/)

---
## 목차
1. [전체 아키텍처 개요](#1-전체-아키텍처-개요)
2. [메시지 타입 설계](#2-메시지-타입-설계)
3. [Core 레이어](#3-core-레이어)
4. [Implementation 레이어](#4-implementation-레이어)
5. [Feature Bridge 레이어](#5-feature-bridge-레이어)
6. [환경 감지와 싱글턴](#6-환경-감지와-싱글턴)
7. [Nuxt 통합: Plugin + Composable](#7-nuxt-통합-plugin--composable)
8. [컴포넌트 연동 예시](#8-컴포넌트-연동-예시)
9. [SSR 환경 격리 전략](#9-ssr-환경-격리-전략)
10. [전체 흐름 다이어그램](#10-전체-흐름-다이어그램)

---

## 1. 전체 아키텍처 개요

### 레이어 구조

```
Vue 컴포넌트 / 미들웨어
  └─ composable (useBridge)
       └─ Feature Bridge — Refined Abstraction
            ├─ NavigationBridge  goBack() / switchTab()
            ├─ HapticBridge      impact() / selection() / notification()
            └─ AuthBridge        requestToken() / onTokenReceived()
                  │
                  └─ BridgeAbstraction — Abstraction 베이스
                        └─ impl: IBridgeImpl
                              ├─ NativeBridge  (RN / iOS / Android)
                              └─ MockBridge    (브라우저 개발 환경)
```

### 디렉토리 구조

```
bridge/
  core/
    IBridgeImpl.ts          # Implementation 인터페이스 (계약)
    BridgeAbstraction.ts    # Abstraction 베이스 클래스
  features/
    NavigationBridge.ts     # Refined Abstraction: 네비게이션
    HapticBridge.ts         # Refined Abstraction: 햅틱
    AuthBridge.ts           # Refined Abstraction: 인증
  impl/
    NativeBridge.ts         # Concrete Implementation: 실제 네이티브
    MockBridge.ts           # Concrete Implementation: 브라우저 시뮬레이터
  index.ts                  # 환경 감지 + 싱글턴 export

types/
  bridge.ts                 # 메시지 타입 중앙 관리

composables/
  useBridge.ts              # Vue 컴포넌트용 composable

plugins/
  bridge.client.ts          # Nuxt client-only 플러그인 (SSR 격리)
```

### 설계 원칙

| 원칙 | 적용 방식 |
|------|----------|
| **환경 분기 격리** | 플랫폼별 `postMessage` 로직은 `NativeBridge.send()` 한 곳에만 존재 |
| **메시지 타입 중앙 관리** | `types/bridge.ts`가 웹·네이티브 양쪽의 프로토콜 계약 |
| **개발 환경 독립** | `MockBridge`로 네이티브 앱 없이 전체 기능 개발·테스트 가능 |
| **SSR 안전** | `bridge.client.ts` 플러그인으로 `window` 접근을 클라이언트 전용으로 격리 |
| **싱글턴 공유** | 모듈 레벨 싱글턴으로 앱 전체에서 동일 인스턴스 + 핸들러 상태 유지 |

---

## 2. 메시지 타입 설계

`types/bridge.ts`는 웹과 네이티브가 주고받는 모든 메시지의 형식을 정의한다. 이 파일 하나가 양쪽의 **통신 프로토콜 계약**이 된다.

### 2.1 메시지 식별자

```typescript
// enum 대신 const object + as const 사용
// 이유: enum은 런타임에 실제 객체를 생성하지만, const object는 트리셰이킹에 유리
export const BridgeMessageType = {
  NAVIGATE_BACK: 'NAVIGATE_BACK',
  NAVIGATE_TAB: 'NAVIGATE_TAB',
  HAPTIC_IMPACT: 'HAPTIC_IMPACT',
  HAPTIC_SELECTION: 'HAPTIC_SELECTION',
  HAPTIC_NOTIFICATION: 'HAPTIC_NOTIFICATION',
  AUTH_TOKEN_REQUEST: 'AUTH_TOKEN_REQUEST',
  AUTH_TOKEN_RECEIVE: 'AUTH_TOKEN_RECEIVE',
  AUTH_USER_RECEIVE: 'AUTH_USER_RECEIVE',
} as const

// `typeof X[keyof typeof X]` 패턴으로 value union type 추출
export type BridgeMessageTypeValue = (typeof BridgeMessageType)[keyof typeof BridgeMessageType]
// → 'NAVIGATE_BACK' | 'NAVIGATE_TAB' | 'HAPTIC_IMPACT' | ...
```

### 2.2 메시지 방향 분리

```typescript
// 웹 → 네이티브 (Outbound)
export type OutboundBridgeMessage =
  | NavigateBackMessage
  | NavigateTabMessage
  | HapticImpactMessage
  | HapticSelectionMessage
  | HapticNotificationMessage
  | AuthTokenRequestMessage

// 네이티브 → 웹 (Inbound)
export type InboundBridgeMessage =
  | AuthTokenReceiveMessage
  | AuthUserReceiveMessage

// 전체 (방향 무관)
export type BridgeMessage = OutboundBridgeMessage | InboundBridgeMessage
```

방향을 타입으로 분리하면 **잘못된 방향으로 메시지를 보내는 실수를 컴파일 타임에 방지**할 수 있다.

### 2.3 개별 메시지 타입

```typescript
// 네비게이션
export type TabName = 'home' | 'products' | 'wishlist' | 'mypage'

export interface NavigateTabMessage {
  type: typeof BridgeMessageType.NAVIGATE_TAB
  payload: { tab: TabName }
}

// 햅틱 — iOS UIFeedbackGenerator 스펙 기준
export type HapticStyle = 'light' | 'medium' | 'heavy'
export type HapticNotificationType = 'success' | 'warning' | 'error'

export interface HapticImpactMessage {
  type: typeof BridgeMessageType.HAPTIC_IMPACT
  payload: { style: HapticStyle }
}

export interface HapticNotificationMessage {
  type: typeof BridgeMessageType.HAPTIC_NOTIFICATION
  payload: { notificationType: HapticNotificationType }
}

// 인증
export interface AuthTokenReceiveMessage {
  type: typeof BridgeMessageType.AUTH_TOKEN_RECEIVE
  payload: {
    token: string
    expiresAt: number  // Unix timestamp (ms)
  }
}

export interface AuthUser {
  id: string
  email: string
  name: string
  avatarUrl?: string
}
```

### 2.4 공통 응답 타입 + 핸들러 타입

```typescript
// 네이티브가 처리 결과를 돌려줄 때의 공통 형식
export interface BridgeResponse<T = unknown> {
  success: boolean
  data?: T
  error?: string
}

// receive()에 등록하는 콜백 타입
export type BridgeMessageHandler<T extends BridgeMessage = BridgeMessage> = (
  message: T,
) => void
```

---

## 3. Core 레이어

### 3.1 IBridgeImpl — Implementation 인터페이스

모든 구현체가 반드시 따라야 하는 계약. `BridgeAbstraction`은 이 인터페이스에만 의존하기 때문에, 구현체가 바뀌어도 Abstraction 코드는 전혀 수정할 필요가 없다.

```typescript
// bridge/core/IBridgeImpl.ts
export interface IBridgeImpl {
  /** 웹 → 네이티브 방향 메시지 전송 */
  send(message: BridgeMessage): void

  /** 네이티브 → 웹 방향 메시지 수신 핸들러 등록 */
  receive(type: BridgeMessageTypeValue, handler: BridgeMessageHandler): void

  /** 현재 환경에서 이 구현체를 사용할 수 있는지 여부 */
  isAvailable(): boolean
}
```

세 메서드만으로 Bridge의 전체 통신 계약이 완성된다.

### 3.2 BridgeAbstraction — Abstraction 베이스

Feature Bridge들의 공통 베이스 클래스. "무엇을 할지"만 정의하고 "어떻게 할지"는 `impl`에 위임한다.

```typescript
// bridge/core/BridgeAbstraction.ts
export abstract class BridgeAbstraction {
  // protected: 자식 클래스(Feature Bridge)에서만 접근 가능, 외부에서 직접 사용 불가
  protected readonly impl: IBridgeImpl

  constructor(impl: IBridgeImpl) {
    this.impl = impl
  }

  protected send(message: BridgeMessage): void {
    this.impl.send(message)
  }

  protected receive(type: BridgeMessageTypeValue, handler: BridgeMessageHandler): void {
    this.impl.receive(type, handler)
  }

  isAvailable(): boolean {
    return this.impl.isAvailable()
  }
}
```

`send()`와 `receive()`가 `protected`인 이유: Feature Bridge의 도메인 메서드(`goBack()`, `selection()` 등)를 통해서만 통신하도록 강제하기 위함이다. 외부에서 raw 메시지를 직접 보내는 것을 막는다.

---

## 4. Implementation 레이어

### 4.1 NativeBridge — 실제 네이티브 구현체

플랫폼별 분기 로직이 여기에 집중된다. Feature Bridge는 이 내부를 전혀 알 필요가 없다.

```typescript
// bridge/impl/NativeBridge.ts
export class NativeBridge implements IBridgeImpl {
  private readonly handlers = new Map<BridgeMessageTypeValue, BridgeMessageHandler[]>()

  isAvailable(): boolean {
    return (
      !!window.ReactNativeWebView ||
      !!window.webkit?.messageHandlers?.bridge ||
      !!window.Android
    )
  }

  send(message: BridgeMessage): void {
    // 우선순위 체인: 처음 감지된 채널만 사용
    if (window.ReactNativeWebView) {
      // React Native: 문자열만 허용 → JSON 직렬화
      window.ReactNativeWebView.postMessage(JSON.stringify(message))
      return
    }
    if (window.webkit?.messageHandlers?.bridge) {
      // iOS WKWebView: 객체를 그대로 전달 가능
      window.webkit.messageHandlers.bridge.postMessage(message)
      return
    }
    if (window.Android) {
      // Android JavascriptInterface: 문자열만 허용 → JSON 직렬화
      window.Android.postMessage(JSON.stringify(message))
    }
  }

  receive(type: BridgeMessageTypeValue, handler: BridgeMessageHandler): void {
    const existing = this.handlers.get(type) ?? []
    this.handlers.set(type, [...existing, handler])

    // window.bridgeCallback을 전역 수신 창구로 설치 (중복 설치 방지)
    // 네이티브 앱은 `window.bridgeCallback(JSON.stringify(message))` 형태로 호출
    if (!window.bridgeCallback) {
      window.bridgeCallback = (json: string) => this.dispatch(json)
    }
  }

  private dispatch(json: string): void {
    let message: BridgeMessage
    try {
      message = JSON.parse(json) as BridgeMessage
    } catch {
      console.error('[NativeBridge] 메시지 파싱 실패:', json)
      return
    }
    const handlers = this.handlers.get(message.type) ?? []
    handlers.forEach((handler) => handler(message))
  }
}
```

**포인트**: `send()`의 우선순위 체인 덕분에 하나의 구현체가 RN / iOS / Android 세 환경을 모두 처리한다.

### 4.2 MockBridge — 개발/테스트용 시뮬레이터

네이티브 앱 없이 브라우저에서도 Bridge 동작을 확인할 수 있다. `send()`는 `console.log`로 출력하고, `emit()`으로 네이티브 응답을 시뮬레이션한다.

```typescript
// bridge/impl/MockBridge.ts
export class MockBridge implements IBridgeImpl {
  private readonly handlers = new Map<BridgeMessageTypeValue, BridgeMessageHandler[]>()

  isAvailable(): boolean {
    return true  // 개발 fallback이므로 항상 사용 가능
  }

  send(message: BridgeMessage): void {
    // DevTools 콘솔에서 [MockBridge] 접두어로 확인
    console.log(`[MockBridge] → send:`, message.type, message)
  }

  receive(type: BridgeMessageTypeValue, handler: BridgeMessageHandler): void {
    const existing = this.handlers.get(type) ?? []
    this.handlers.set(type, [...existing, handler])
  }

  // 개발/테스트 전용: 네이티브 응답을 시뮬레이션
  // NativeBridge에서는 window.bridgeCallback이 이 역할을 담당
  emit(message: BridgeMessage): void {
    const handlers = this.handlers.get(message.type) ?? []
    handlers.forEach((handler) => handler(message))
  }
}
```

브라우저 DevTools 콘솔에서 직접 `emit()`을 호출해 토큰 수신 등을 테스트할 수 있다:

```javascript
// DevTools 콘솔에서
import { authBridge } from '/bridge/index.ts'
authBridge.impl.emit({
  type: 'AUTH_TOKEN_RECEIVE',
  payload: { token: 'test-token', expiresAt: Date.now() + 3600000 }
})
```

---

## 5. Feature Bridge 레이어

### 5.1 NavigationBridge

```typescript
// bridge/features/NavigationBridge.ts
export class NavigationBridge extends BridgeAbstraction {
  /** 네이티브 앱의 navigation stack을 pop한다 (웹의 history.back()과 다름) */
  goBack(): void {
    this.send({ type: BridgeMessageType.NAVIGATE_BACK })
  }

  /** 네이티브 BottomTabBar의 탭을 전환한다 */
  switchTab(tab: TabName): void {
    this.send({ type: BridgeMessageType.NAVIGATE_TAB, payload: { tab } })
  }
}
```

### 5.2 HapticBridge

iOS `UIFeedbackGenerator`의 세 가지 피드백 타입에 대응한다.

```typescript
// bridge/features/HapticBridge.ts
export class HapticBridge extends BridgeAbstraction {
  /** 충격 피드백 — 버튼 탭, 스와이프 등 물리적 인터랙션 */
  impact(style: HapticStyle): void {
    this.send({ type: BridgeMessageType.HAPTIC_IMPACT, payload: { style } })
  }

  /** 선택 피드백 — 찜하기, 체크박스 토글 등 항목 선택 */
  selection(): void {
    this.send({ type: BridgeMessageType.HAPTIC_SELECTION })
  }

  /** 알림 피드백 — 작업 성공/실패/경고 결과를 촉각으로 전달 */
  notification(notificationType: HapticNotificationType): void {
    this.send({ type: BridgeMessageType.HAPTIC_NOTIFICATION, payload: { notificationType } })
  }
}
```

### 5.3 AuthBridge

웹뷰는 네이티브 키체인/키스토어에 직접 접근할 수 없다. Bridge를 통해 토큰을 요청하고 응답을 핸들러로 받는다.

```typescript
// bridge/features/AuthBridge.ts
export class AuthBridge extends BridgeAbstraction {
  /** 네이티브 앱에 JWT 토큰을 요청한다 */
  requestToken(): void {
    this.send({ type: BridgeMessageType.AUTH_TOKEN_REQUEST })
  }

  /** 토큰 수신 핸들러 등록 */
  onTokenReceived(handler: (token: string) => void): void {
    // payload 추출 래핑 패턴:
    // 메시지 전체 객체가 아닌 token만 추출하여 호출자에게 전달
    // → 호출자가 Bridge 내부 메시지 구조를 알 필요가 없음
    this.receive(BridgeMessageType.AUTH_TOKEN_RECEIVE, (message) => {
      const { payload } = message as AuthTokenReceiveMessage
      handler(payload.token)
    })
  }

  /** 유저 정보 수신 핸들러 등록 */
  onUserReceived(handler: (user: AuthUser) => void): void {
    this.receive(BridgeMessageType.AUTH_USER_RECEIVE, (message) => {
      const { payload } = message as AuthUserReceiveMessage
      handler(payload.user)
    })
  }
}
```

**payload 추출 래핑 패턴**: `receive()`에 등록하는 내부 콜백에서 필요한 데이터만 추출해서 외부 핸들러를 호출한다. 덕분에 컴포넌트는 `BridgeMessage` 구조를 몰라도 된다.

---

## 6. 환경 감지와 싱글턴

### 6.1 환경 감지

```typescript
// bridge/index.ts
export type BridgeEnvironment = 'react-native' | 'ios' | 'android' | 'mock'

export function detectEnvironment(): BridgeEnvironment {
  if (typeof window === 'undefined') return 'mock'  // SSR 안전장치
  if (window.ReactNativeWebView) return 'react-native'
  if (window.webkit?.messageHandlers?.bridge) return 'ios'
  if (window.Android) return 'android'
  return 'mock'  // 브라우저/개발 환경 fallback
}
```

우선순위 체인:
```
1. window.ReactNativeWebView  →  'react-native'
2. window.webkit.messageHandlers.bridge  →  'ios'
3. window.Android  →  'android'
4. (없으면)  →  'mock'
```

### 6.2 팩토리 함수 + 싱글턴

```typescript
// 팩토리: 환경에 따라 구현체 결정
export function createBridgeImpl(env?: BridgeEnvironment): IBridgeImpl {
  const resolvedEnv = env ?? detectEnvironment()
  // RN / iOS / Android 세 환경은 모두 NativeBridge로 통합
  // 각 채널의 차이는 NativeBridge.send() 내부에서 처리
  if (resolvedEnv === 'mock') return new MockBridge()
  return new NativeBridge()
}

// 모듈 레벨 싱글턴 — 앱 전체에서 동일 인스턴스 + 핸들러 상태 공유
const impl = createBridgeImpl()

export const navigationBridge = new NavigationBridge(impl)
export const hapticBridge = new HapticBridge(impl)
export const authBridge = new AuthBridge(impl)
```

싱글턴으로 관리하는 이유:
- 핸들러 등록 상태를 앱 전체에서 일관되게 유지
- 불필요한 객체 생성 방지
- `onTokenReceived()` 등 이벤트 기반 핸들러가 의도치 않게 중복 등록되는 것을 막음

---

## 7. Nuxt 통합: Plugin + Composable

### 7.1 Client-Only Plugin

```typescript
// plugins/bridge.client.ts

// .client.ts 접미사 → Nuxt가 서버에서 이 파일을 실행하지 않음
// window, ReactNativeWebView 등에 안전하게 접근 가능
import { navigationBridge, hapticBridge, authBridge } from '@/bridge'

export default defineNuxtPlugin((nuxtApp) => {
  // provide('bridge', value) → 앱 전체에서 nuxtApp.$bridge로 접근 가능
  nuxtApp.provide('bridge', {
    navigation: navigationBridge,
    haptic: hapticBridge,
    auth: authBridge,
  })
})
```

### 7.2 Composable

Plugin의 `$bridge`를 사용할 수도 있지만, 더 직관적인 composable을 제공한다.

```typescript
// composables/useBridge.ts
import { navigationBridge, hapticBridge, authBridge } from '@/bridge'
import type { NavigationBridge } from '@/bridge/features/NavigationBridge'
import type { HapticBridge } from '@/bridge/features/HapticBridge'
import type { AuthBridge } from '@/bridge/features/AuthBridge'

export function useNavigationBridge(): NavigationBridge {
  return navigationBridge
}

export function useHapticBridge(): HapticBridge {
  return hapticBridge
}

export function useAuthBridge(): AuthBridge {
  return authBridge
}
```

composable이 싱글턴을 그대로 반환하는 이유: Bridge는 Vue 반응형 상태가 아니라 통신 채널이기 때문에 `ref`로 감쌀 필요가 없다.

---

## 8. 컴포넌트 연동 예시

### 8.1 HapticBridge — 찜하기 버튼

```vue
<!-- components/ProductCard.vue -->
<script setup lang="ts">
import { useHapticBridge } from '@/composables/useBridge'

const hapticBridge = useHapticBridge()
const isWishlisted = ref(false)

function handleWishlist() {
  isWishlisted.value = !isWishlisted.value
  hapticBridge.selection()  // 찜하기 토글 시 선택 피드백
}
</script>

<template>
  <button
    :aria-pressed="String(isWishlisted)"
    @click.prevent="handleWishlist"
  >
    {{ isWishlisted ? '찜 해제' : '찜하기' }}
  </button>
</template>
```

### 8.2 NavigationBridge — 뒤로가기 (폴백 포함)

```vue
<!-- components/AppHeader.vue -->
<script setup lang="ts">
import { useNavigationBridge } from '@/composables/useBridge'

const router = useRouter()
const navigationBridge = useNavigationBridge()

function handleBack() {
  if (navigationBridge.isAvailable()) {
    // 네이티브 환경: 네이티브 navigation stack pop
    navigationBridge.goBack()
  } else {
    // 브라우저 환경: 웹 히스토리 back 폴백
    router.back()
  }
}
</script>
```

`isAvailable()` 체크 후 폴백을 두는 패턴은 **웹뷰와 브라우저 모두에서 동일한 컴포넌트를 사용**할 수 있게 해준다.

### 8.3 AuthBridge — 토큰 요청 + 수신

```typescript
// middleware/auth.ts (개념 코드)
import { authBridge } from '@/bridge'

export default defineNuxtRouteMiddleware(() => {
  // 1. 수신 핸들러 먼저 등록
  authBridge.onTokenReceived((token) => {
    // 토큰을 받으면 Pinia store나 cookie에 저장
    useAuthStore().setToken(token)
  })

  // 2. 토큰 요청 전송
  authBridge.requestToken()
})
```

`onTokenReceived()` → `requestToken()` 순서가 중요하다. 핸들러를 먼저 등록하지 않으면 토큰이 도착했을 때 받을 수단이 없다.

---

## 9. SSR 환경 격리 전략

Nuxt 3은 기본적으로 SSR을 사용한다. 서버에는 `window`가 없으므로 Bridge 코드를 서버에서 실행하면 오류가 발생한다.

### 위험 상황

```typescript
// ❌ 서버에서 import하면 즉시 오류
import { navigationBridge } from '@/bridge'  // window 접근 발생
```

### 대응 방법

**1. `.client.ts` 플러그인으로 격리**

```
plugins/
  bridge.client.ts  ← 서버에서 절대 실행되지 않음
```

Bridge 싱글턴은 이 플러그인이 import할 때 초기화된다. 서버는 이 파일을 실행하지 않으므로 `window` 접근이 발생하지 않는다.

**2. `detectEnvironment()`의 안전장치**

```typescript
export function detectEnvironment(): BridgeEnvironment {
  if (typeof window === 'undefined') return 'mock'  // SSR → 즉시 mock 반환
  // ...
}
```

**3. 컴포넌트에서 직접 import할 때**

```typescript
// ✅ onMounted 또는 import.meta.client 가드 사용
onMounted(() => {
  const { authBridge } = await import('@/bridge')
  authBridge.requestToken()
})
```

### 위험 수준별 정리

| 상황 | 위험 수준 | 대응 |
|------|----------|------|
| SSR에서 `window` 접근 | HIGH | `.client.ts` 플러그인으로 격리 |
| `window` 타입 미정의 | LOW | `types/global.d.ts`에 Window 인터페이스 확장 |
| AuthBridge 핸들러 타이밍 | MEDIUM | 핸들러 먼저 등록, 요청은 나중에 |
| 네이티브 앱 없는 테스트 | MEDIUM | MockBridge + `emit()`으로 시뮬레이션 |

---

## 10. 전체 흐름 다이어그램

### 웹 → 네이티브 (찜하기 햅틱 피드백)

```
[사용자 찜하기 클릭]
        │
        ▼
ProductCard.vue — handleWishlist()
  hapticBridge.selection()
        │
        ▼
HapticBridge.selection()         Refined Abstraction
  this.send({ type: 'HAPTIC_SELECTION' })
        │
        ▼
BridgeAbstraction.send()         Abstraction
  this.impl.send(message)
        │
        ├── [네이티브 환경] ──▶ NativeBridge.send()
        │                         ├─ RN:  ReactNativeWebView.postMessage(JSON)
        │                         ├─ iOS: webkit.messageHandlers.bridge.postMessage(obj)
        │                         └─ AOS: Android.postMessage(JSON)
        │
        └── [브라우저 환경] ──▶ MockBridge.send()
                                  console.log('[MockBridge] → send: HAPTIC_SELECTION')
```

### 네이티브 → 웹 (인증 토큰 수신)

```
[네이티브 앱이 토큰 전송]
        │
        ▼
window.bridgeCallback('{"type":"AUTH_TOKEN_RECEIVE","payload":{"token":"..."}}')
        │
        ▼
NativeBridge.dispatch(json)
  JSON.parse(json) → message
  handlers.get('AUTH_TOKEN_RECEIVE') → [wrappedHandler]
  wrappedHandler(message)
        │
        ▼
AuthBridge.onTokenReceived 내부 래퍼
  const { payload } = message as AuthTokenReceiveMessage
  handler(payload.token)           payload에서 token만 추출
        │
        ▼
컴포넌트/미들웨어의 콜백
  (token: string) => { ... }       깔끔한 인터페이스로 수신
```

### 레이어별 책임 요약

```
┌────────────────────────────────────────────────────────────┐
│               Vue 컴포넌트 / Nuxt 미들웨어                  │
│  hapticBridge.selection()  |  authBridge.requestToken()    │
└──────────────────────────┬─────────────────────────────────┘
                           │ composable (useBridge)
┌──────────────────────────▼─────────────────────────────────┐
│              Feature Bridge  (Refined Abstraction)          │
│    NavigationBridge | HapticBridge | AuthBridge            │
│    도메인 특화 메서드, send()/receive() 호출                 │
└──────────────────────────┬─────────────────────────────────┘
                           │ extends
┌──────────────────────────▼─────────────────────────────────┐
│              BridgeAbstraction  (Abstraction)               │
│    impl.send() / impl.receive() 위임                        │
└──────────────────────────┬─────────────────────────────────┘
                           │ IBridgeImpl (의존성 주입)
┌─────────────────┬────────▼────────────────────────────────┐
│   NativeBridge  │              MockBridge                  │
│  (실제 네이티브) │           (브라우저/테스트)               │
│  RN/iOS/Android │        console.log + emit()             │
│  postMessage    │                                         │
└─────────────────┴─────────────────────────────────────────┘
```

---

## 마치며

이 프로젝트에서 WebView Bridge를 구현하며 배운 핵심:

1. **타입이 프로토콜이다** — `types/bridge.ts`의 메시지 타입이 웹-네이티브 계약이 된다. 타입이 맞지 않으면 컴파일 단계에서 잡힌다.

2. **환경 분기는 한 곳에** — `NativeBridge.send()` 밖에서는 플랫폼 분기가 없다. Feature Bridge는 iOS인지 Android인지 전혀 모른다.

3. **개발 환경을 먼저 설계하라** — `MockBridge`와 `emit()`이 없었다면 네이티브 앱 없이 Bridge 관련 기능을 개발하거나 테스트하는 것이 불가능했을 것이다.

4. **SSR은 항상 고려대상으로 두어야 한다** — `window`에 접근하는 코드는 반드시 `.client.ts`로 격리하거나 `typeof window === 'undefined'` 가드를 달아야 한다.

5. **Bridge Pattern의 진짜 가치** — Feature Bridge가 플랫폼을 몰라도 된다는 것은 단순한 코드 정리가 아니다. 나중에 새 플랫폼(예: 데스크톱 WebView)이 추가되거나, MockBridge를 테스트용 Spy로 교체할 때 **Feature Bridge 코드는 손댈 필요가 없다**.

---

*Part 1로 돌아가기*: [개요 및 이론 — WebView란, 프로토콜, 통신 패턴, Bridge Pattern](/posts/browser/web-view-1/)

---

## 참고 자료

### 프로젝트 소스 파일
> 이 문서의 모든 코드 예시는 아래 실제 구현 파일에서 발췌했다.

| 파일 | 역할 |
|------|------|
| `bridge_plan.md` | 전체 구현 계획, MVP 체크리스트, Phase별 설계 결정 사항 |
| `types/bridge.ts` | `BridgeMessageType`, `BridgeMessage` union, `BridgeResponse`, `BridgeMessageHandler` 타입 정의 |
| `bridge/core/IBridgeImpl.ts` | `send()` / `receive()` / `isAvailable()` 3개 메서드로 구성된 Implementation 인터페이스 |
| `bridge/core/BridgeAbstraction.ts` | `protected send()` / `receive()` 위임 메서드를 갖는 abstract Abstraction 베이스 클래스 |
| `bridge/impl/NativeBridge.ts` | ReactNativeWebView → webkit → Android 우선순위 체인, `window.bridgeCallback` 전역 수신 창구 |
| `bridge/impl/MockBridge.ts` | `Map` 기반 이벤트 에미터, `emit()` 네이티브 시뮬레이션 메서드 |
| `bridge/features/NavigationBridge.ts` | `goBack()`, `switchTab()` Refined Abstraction |
| `bridge/features/HapticBridge.ts` | `impact()`, `selection()`, `notification()` Refined Abstraction |
| `bridge/features/AuthBridge.ts` | `requestToken()`, `onTokenReceived()` payload 추출 래핑 패턴 |
| `bridge/index.ts` | `detectEnvironment()`, `createBridgeImpl()` 팩토리, 모듈 레벨 싱글턴 export |
| `plugins/bridge.client.ts` | Nuxt client-only 플러그인 — `defineNuxtPlugin` + `nuxtApp.provide()` |
| `composables/useBridge.ts` | `useNavigationBridge()`, `useHapticBridge()`, `useAuthBridge()` composable |

### Nuxt 3 공식 문서
- **Nuxt 3 Plugins** — `defineNuxtPlugin`, `.client.ts` 네이밍 컨벤션, `nuxtApp.provide()` 사용법. `bridge.client.ts` 구현의 직접 근거  
  https://nuxt.com/docs/guide/directory-structure/plugins

- **Nuxt 3 Composables** — `useNuxtApp()`을 통한 plugin injection 접근 패턴. `useBridge.ts` composable 설계 근거  
  https://nuxt.com/docs/guide/directory-structure/composables

- **Nuxt App 인스턴스** — runtime nuxt app instance 접근 방법, `$injected` 프로퍼티 네이밍 규칙  
  https://nuxt.com/docs/guide/going-further/nuxt-app

### iOS Haptics — Apple 공식 문서

> `HapticStyle` (`light` / `medium` / `heavy`) 과 `HapticNotificationType` (`success` / `warning` / `error`) 타입이 아래 iOS 스펙을 기준으로 설계됐다.

- **UIImpactFeedbackGenerator** — `light` / `medium` / `heavy` 강도 충격 피드백. `HapticBridge.impact(style:)` 의 근거  
  https://developer.apple.com/documentation/uikit/uiimpactfeedbackgenerator

- **UISelectionFeedbackGenerator** — 항목 선택 시 피드백. `HapticBridge.selection()` 의 근거  
  https://developer.apple.com/documentation/uikit/uiselectionfeedbackgenerator

- **UINotificationFeedbackGenerator** — `success` / `warning` / `error` 알림 피드백. `HapticBridge.notification(type:)` 의 근거  
  https://developer.apple.com/documentation/uikit/uinotificationfeedbackgenerator
