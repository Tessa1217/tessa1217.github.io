---
title: WebView Bridge 통신 Part 1
published: 2026-04-12
tags: [Browser, Javascript, Webview]
category: Browser
series: WebView
draft: false
---
# WebView Bridge 통신 — Part 1: 개요 및 이론

> 웹뷰에서 네이티브 앱과 어떻게 대화하는가 — 프로토콜, 패턴, Bridge Pattern까지

---

## 목차

1. [WebView란 무엇인가](#1-webview란-무엇인가)
2. [왜 웹과 앱 간 통신이 필요한가](#2-왜-웹과-앱-간-통신이-필요한가)
3. [WebView 통신 프로토콜](#3-webview-통신-프로토콜)
4. [자주 쓰이는 Bridge 통신 패턴](#4-자주-쓰이는-bridge-통신-패턴)
5. [Bridge Pattern (디자인 패턴)](#5-bridge-pattern-디자인-패턴)

---

## 1. WebView란 무엇인가

**WebView**는 네이티브 앱(iOS/Android) 안에 웹 콘텐츠를 렌더링하는 내장 브라우저 컴포넌트다. 사용자 입장에서는 앱처럼 보이지만, 실제로는 HTML/CSS/JavaScript로 만들어진 웹 페이지가 네이티브 껍데기 안에서 동작하고 있다.

```
┌─────────────────────────────────┐
│        네이티브 앱 (iOS/Android) │
│  ┌───────────────────────────┐  │
│  │         WebView           │  │
│  │  ┌─────────────────────┐  │  │
│  │  │   HTML/CSS/JS (웹)  │  │  │
│  │  └─────────────────────┘  │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

### 플랫폼별 WebView 구현체

| 플랫폼 | 컴포넌트 | 내부 엔진 |
|--------|----------|-----------|
| iOS | `WKWebView` | WebKit (Safari 엔진) |
| Android | `WebView` | Chromium (Blink 엔진) |
| React Native | `<WebView>` 컴포넌트 | 플랫폼 네이티브 WebView 래핑 |

### 하이브리드 앱 아키텍처

WebView를 활용한 하이브리드 앱은 크게 두 가지 방식으로 나뉜다.

- **Full WebView**: 앱 전체가 웹으로 구성. 네이티브는 껍데기 역할만 함
- **Partial WebView**: 일부 화면(주문, 결제, 이벤트 등)만 웹으로 구성하고 나머지는 네이티브

후자가 실무에서 훨씬 흔하며, 이때 **네이티브와 웹 간 통신**이 반드시 필요해진다.

---

## 2. 왜 웹과 앱 간 통신이 필요한가

웹(WebView)과 네이티브 앱은 서로 다른 실행 환경에서 동작한다. 웹은 브라우저 샌드박스 안에 갇혀 있고, 네이티브 기능(카메라, 햅틱, 푸시 알림, 키체인 등)에 직접 접근할 수 없다.

### 통신이 필요한 대표적인 시나리오

#### 웹 → 네이티브 방향 (명령)

| 시나리오 | 설명 |
|----------|------|
| **네비게이션** | 웹 페이지에서 네이티브 뒤로가기 스택을 pop하거나 탭을 전환 |
| **햅틱 피드백** | 버튼 탭, 찜하기 등 사용자 인터랙션에 촉각 피드백 요청 |
| **카메라/갤러리** | 웹에서 사진 촬영 또는 앨범 접근 요청 |
| **생체 인증** | Face ID / 지문 인증 요청 |
| **푸시 알림 권한** | 알림 권한 요청 다이얼로그 트리거 |
| **소셜 공유** | 네이티브 share sheet 호출 |
| **결제** | 네이티브 결제 SDK 호출 (인앱결제, 카카오페이 등) |

#### 네이티브 → 웹 방향 (데이터 전달)

| 시나리오 | 설명 |
|----------|------|
| **인증 토큰** | 네이티브 키체인/키스토어에 저장된 JWT를 웹으로 전달 |
| **유저 정보** | 네이티브에서 관리하는 사용자 프로필 데이터 전달 |
| **딥링크** | 외부에서 들어온 딥링크 정보를 웹에 전달 |
| **기기 정보** | OS 버전, 앱 버전, 디바이스 ID 등 전달 |
| **네트워크 상태** | 오프라인 전환 이벤트 등 전달 |

이처럼 웹이 단독으로 처리할 수 없는 영역이 명확히 존재하기 때문에, 웹뷰 환경에서 **양방향 통신 채널** — 즉 Bridge — 이 필수적이다.

---

## 3. WebView 통신 프로토콜

플랫폼마다 웹과 네이티브가 통신하는 방식이 다르다.

### 3.1 iOS: WKWebView

iOS에서는 `WKWebView`와 `WKScriptMessageHandler`를 사용해 JavaScript ↔ Swift/Objective-C 통신을 구현한다.

#### 웹 → 네이티브 (JavaScript → Swift)

```javascript
// JavaScript (웹)
window.webkit.messageHandlers.bridge.postMessage({
  type: 'HAPTIC_SELECTION'
})
```

```swift
// Swift (네이티브)
class BridgeHandler: NSObject, WKScriptMessageHandler {
  func userContentController(
    _ userContentController: WKUserContentController,
    didReceive message: WKScriptMessage
  ) {
    guard let body = message.body as? [String: Any],
          let type = body["type"] as? String else { return }

    switch type {
    case "HAPTIC_SELECTION":
      let generator = UISelectionFeedbackGenerator()
      generator.selectionChanged()
    default: break
    }
  }
}

// WKWebView 설정
let config = WKWebViewConfiguration()
config.userContentController.add(bridgeHandler, name: "bridge")
let webView = WKWebView(frame: .zero, configuration: config)
```

#### 네이티브 → 웹 (Swift → JavaScript)

```swift
// Swift: 웹의 전역 함수를 직접 호출
webView.evaluateJavaScript("window.bridgeCallback('\(jsonString)')")
```

```javascript
// JavaScript (웹): 전역 함수로 수신
window.bridgeCallback = function(json) {
  const message = JSON.parse(json)
  // 메시지 처리
}
```

iOS의 특징: `postMessage()`에 JavaScript 객체를 **그대로** 전달할 수 있다. 내부적으로 직렬화가 처리된다.

### 3.2 Android: JavascriptInterface

Android에서는 `@JavascriptInterface` 어노테이션으로 Kotlin/Java 메서드를 JavaScript에 노출한다.

#### 웹 → 네이티브 (JavaScript → Kotlin)

```javascript
// JavaScript (웹) — Android는 문자열만 허용하므로 JSON 직렬화 필요
window.Android.postMessage(JSON.stringify({
  type: 'NAVIGATE_BACK'
}))
```

```kotlin
// Kotlin (네이티브)
class AndroidBridge(private val activity: Activity) {
  @JavascriptInterface
  fun postMessage(json: String) {
    val message = JSONObject(json)
    when (message.getString("type")) {
      "NAVIGATE_BACK" -> activity.onBackPressed()
    }
  }
}

// WebView 설정
webView.settings.javaScriptEnabled = true
webView.addJavascriptInterface(AndroidBridge(this), "Android")
```

#### 네이티브 → 웹 (Kotlin → JavaScript)

```kotlin
webView.evaluateJavascript("window.bridgeCallback('$jsonString')", null)
```

Android의 특징: `@JavascriptInterface` 메서드는 **문자열 인자만** 받을 수 있다. 객체를 주고받으려면 반드시 JSON 직렬화/역직렬화가 필요하다.

### 3.3 React Native: WebView

React Native는 `react-native-webview` 라이브러리를 통해 웹과 통신한다.

#### 웹 → 네이티브

```javascript
// JavaScript (웹뷰 내부) — 문자열만 허용
window.ReactNativeWebView.postMessage(JSON.stringify({
  type: 'HAPTIC_IMPACT',
  payload: { style: 'medium' }
}))
```

```jsx
// React Native
<WebView
  onMessage={(event) => {
    const message = JSON.parse(event.nativeEvent.data)
    if (message.type === 'HAPTIC_IMPACT') {
      Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium)
    }
  }}
/>
```

#### 네이티브 → 웹

```jsx
// injectJavaScript로 JS 실행 (true 반환값 필요)
webViewRef.current.injectJavaScript(`
  window.bridgeCallback('${jsonString}'); true;
`)
```

### 플랫폼별 통신 방식 비교

| 구분 | iOS (WKWebView) | Android (WebView) | React Native |
|------|----------------|-------------------|--------------|
| **웹 → 네이티브** | `webkit.messageHandlers.{name}.postMessage(obj)` | `Android.{method}(str)` | `ReactNativeWebView.postMessage(str)` |
| **네이티브 → 웹** | `evaluateJavaScript(js)` | `evaluateJavascript(js, cb)` | `injectJavaScript(js)` |
| **데이터 타입 (웹→네이티브)** | 객체 직접 전달 가능 | 문자열만 | 문자열만 |
| **전역 수신 창구** | `window.bridgeCallback` (커스텀) | `window.bridgeCallback` (커스텀) | `window.bridgeCallback` (커스텀) |

---

## 4. 자주 쓰이는 Bridge 통신 패턴

### 4.1 단방향 Fire-and-Forget

가장 단순한 패턴. 웹이 명령만 보내고 응답을 기다리지 않는다.

```
웹 ──[NAVIGATE_BACK]──→ 네이티브
```

```javascript
window.webkit.messageHandlers.bridge.postMessage({ type: 'NAVIGATE_BACK' })
```

**사용 사례**: 네비게이션, 햅틱 피드백, 소셜 공유 트리거

---

### 4.2 Request-Response (요청-응답)

웹이 요청을 보내고, 네이티브가 처리 후 콜백으로 응답한다.

```
웹 ──[AUTH_TOKEN_REQUEST]──→ 네이티브
웹 ←──[AUTH_TOKEN_RECEIVE]── 네이티브
```

```javascript
// 1. 수신 핸들러 등록
window.bridgeCallback = function(json) {
  const message = JSON.parse(json)
  if (message.type === 'AUTH_TOKEN_RECEIVE') {
    useToken(message.payload.token)
  }
}

// 2. 요청 전송
window.ReactNativeWebView.postMessage(JSON.stringify({
  type: 'AUTH_TOKEN_REQUEST'
}))
```

**사용 사례**: 인증 토큰 조회, 기기 정보 조회, 권한 상태 확인

---

### 4.3 Event-Driven (이벤트 기반)

네이티브가 웹의 요청 없이 이벤트를 push한다. 수신 핸들러만 미리 등록해두면 된다.

```
웹 ←──[AUTH_USER_RECEIVE]── 네이티브 (앱 초기화 시 자동 push)
웹 ←──[DEEP_LINK_RECEIVED]── 네이티브 (외부 딥링크 수신 시)
```

**사용 사례**: 딥링크 처리, 네트워크 상태 변화, 푸시 알림 수신

---

### 4.4 Promise-based Bridge

콜백 중첩을 피하기 위해 Bridge 호출을 Promise로 래핑한다.

```javascript
function callBridge(requestType, responseType, timeout = 5000) {
  return new Promise((resolve, reject) => {
    const timer = setTimeout(() => reject(new Error('Bridge timeout')), timeout)

    window.bridgeCallback = (json) => {
      const message = JSON.parse(json)
      if (message.type === responseType) {
        clearTimeout(timer)
        resolve(message.payload)
      }
    }

    window.ReactNativeWebView.postMessage(JSON.stringify({ type: requestType }))
  })
}

// 사용
const { token } = await callBridge('AUTH_TOKEN_REQUEST', 'AUTH_TOKEN_RECEIVE')
```

**장점**: async/await 문법으로 가독성 향상  
**주의**: 동시에 여러 요청이 발생하면 `bridgeCallback` 충돌 가능 → `requestId` 패턴으로 대응

---

### 4.5 Message Queue 패턴

WebView 로드 전에 도착하는 네이티브 메시지를 잃지 않기 위한 큐 패턴.

```javascript
const messageQueue = []
let isReady = false

// 네이티브 메시지 수신 창구 (앱 초기화 즉시 설치)
window.bridgeCallback = function(json) {
  if (!isReady) {
    messageQueue.push(json)  // 준비 전 메시지는 큐에 보관
    return
  }
  processMessage(JSON.parse(json))
}

// 웹앱 초기화 완료 후 호출
function onAppReady() {
  isReady = true
  messageQueue.forEach(json => processMessage(JSON.parse(json)))  // 밀린 메시지 처리
  messageQueue.length = 0
}
```

**사용 사례**: 딥링크 처리, 앱 시작 시 유저 정보 push 등 타이밍 민감한 시나리오

---

## 5. Bridge Pattern (디자인 패턴)

GoF(Gang of Four)의 **Bridge Pattern**은 구조 패턴(Structural Pattern) 중 하나로, **Abstraction(추상화)** 과 **Implementation(구현)** 을 분리하여 둘 다 독립적으로 확장할 수 있게 한다.

> "Decouple an abstraction from its implementation so that the two can vary independently."
> — GoF Design Patterns

### 5.1 핵심 구조

```
Abstraction
  └─ impl: Implementor  ← 합성(composition)으로 주입
        ├─ ConcreteImplementorA
        └─ ConcreteImplementorB

RefinedAbstraction extends Abstraction
  └─ 도메인 특화 메서드 추가
```

- **Abstraction**: "무엇을 할지"를 정의. 실제 동작은 `impl`에 위임
- **Implementor**: 구현체의 인터페이스(계약)
- **ConcreteImplementor**: 실제 플랫폼/환경별 구현
- **RefinedAbstraction**: Abstraction을 상속해 도메인 특화 메서드 추가

### 5.2 WebView Bridge에 Bridge Pattern이 적합한 이유

WebView 통신 구현의 핵심 문제는 **환경 분기**다.

```
"햅틱 피드백을 보내라"는 명령이 있을 때...
  → 실제 앱(RN):   window.ReactNativeWebView.postMessage(JSON)
  → 실제 앱(iOS):  window.webkit.messageHandlers.bridge.postMessage(obj)
  → 실제 앱(AOS):  window.Android.postMessage(JSON)
  → 브라우저(개발): console.log로 시뮬레이션
```

Bridge Pattern 없이 구현하면 환경 분기 코드가 **기능 메서드마다 반복**된다.

```typescript
// ❌ Bridge Pattern 없이 — 조건 분기가 기능마다 중복
function goBack() {
  if (window.ReactNativeWebView) {
    window.ReactNativeWebView.postMessage(JSON.stringify({ type: 'NAVIGATE_BACK' }))
  } else if (window.webkit?.messageHandlers?.bridge) {
    window.webkit.messageHandlers.bridge.postMessage({ type: 'NAVIGATE_BACK' })
  } else if (window.Android) {
    window.Android.postMessage(JSON.stringify({ type: 'NAVIGATE_BACK' }))
  } else {
    console.log('[Dev] NAVIGATE_BACK')
  }
}

function hapticSelection() {
  // 위와 동일한 분기 코드가 또 반복...
}
```

Bridge Pattern을 적용하면 **환경 분기 로직을 구현체 한 곳에 격리**하고, 기능 메서드는 깔끔하게 유지된다.

```typescript
// ✅ Bridge Pattern 적용 후
class HapticBridge extends BridgeAbstraction {
  selection(): void {
    this.send({ type: 'HAPTIC_SELECTION' })  // 플랫폼을 전혀 모름
  }
}

// 플랫폼 분기는 NativeBridge.send() 한 곳에만 존재
class NativeBridge implements IBridgeImpl {
  send(message): void {
    if (window.ReactNativeWebView) { ... }
    else if (window.webkit?.messageHandlers?.bridge) { ... }
    else if (window.Android) { ... }
  }
}
```

### 5.3 Bridge Pattern vs 유사 패턴 비교

| 패턴 | 목적 | 차이 |
|------|------|------|
| **Bridge** | Abstraction과 Implementation을 독립적으로 확장 | 둘 다 계층 구조를 가질 수 있음 |
| **Adapter** | 호환되지 않는 인터페이스를 연결 | 기존 코드를 변경 없이 사용하기 위한 사후 적용 |
| **Strategy** | 알고리즘을 런타임에 교체 | 단일 레벨의 알고리즘 교체, Abstraction 계층 없음 |
| **Facade** | 복잡한 서브시스템에 단순한 인터페이스 제공 | 계층 분리가 목적이 아님 |

---

**Part 2**: [실전 구현 — Nuxt 3 프로젝트의 Bridge 아키텍처](/posts/browser/web-view-2/)

---

## 참고 자료

### iOS / Apple 공식 문서 
- **WKScriptMessageHandler** — JavaScript 메시지를 수신하는 핸들러 프로토콜. `userContentController(_:didReceive:)` 메서드 명세  
  https://developer.apple.com/documentation/webkit/wkscriptmessagehandler

- **WKUserContentController.add(_:name:)** — `window.webkit.messageHandlers.{name}.postMessage()` JavaScript 함수를 앱에 등록하는 메서드  
  https://developer.apple.com/documentation/webkit/wkusercontentcontroller/add(_:name:)

- **WKWebView.evaluateJavaScript(_:completionHandler:)** — Swift에서 JavaScript 문자열을 실행하는 메서드. 네이티브 → 웹 방향 통신에 사용  
  https://developer.apple.com/documentation/webkit/wkwebview/evaluatejavascript(_:completionhandler:)

### Android 공식 문서
- **WebView — Android Developer Guide** — `addJavascriptInterface`, `evaluateJavascript`, `settings.javaScriptEnabled` 등 Android WebView 전반 가이드  
  https://developer.android.com/develop/ui/views/layout/webapps/webview

- **@JavascriptInterface annotation** — JavaScript에 노출할 Kotlin/Java 메서드를 지정하는 어노테이션 명세  
  https://developer.android.com/reference/android/webkit/JavascriptInterface

### React Native WebView

- **React Native WebView 통신 가이드** — `window.ReactNativeWebView.postMessage`, `onMessage` prop, `injectedJavaScript`, `injectJavaScript` 양방향 통신 상세 설명  
  https://github.com/react-native-webview/react-native-webview/blob/master/docs/Guide.md

### Bridge Pattern (GoF)
- **Bridge Pattern 개요** — Abstraction과 Implementation 분리 원칙, 적용 맥락, 유사 패턴(Adapter, Strategy, Facade)과의 비교  
  https://refactoring.guru/design-patterns/bridge

- **Bridge Pattern TypeScript 예제** — `Abstraction`, `ExtendedAbstraction`, `Implementation`, `ConcreteImplementationA/B` 구조의 전체 코드  
  https://refactoring.guru/design-patterns/bridge/typescript/example
