---
title: 📦 npm Private Registry 병렬 베타 배포 파이프라인 개선기
published: 2026-05-26
tags: [npm, SemVer, CICD, Bitbucket, ComponentLibrary]
category: DevOps
series: DevOps
image: /images/npm.webp
draft: false
---

# 📦 npm Private Registry 병렬 베타 배포 파이프라인 개선기

> SemVer · Prerelease · Feature-Scoped Versioning · Bitbucket Pipelines
>
> 라이브러리 운영이 집중 개발기에서 안정화 후 병렬 기술 과제 수행기로 전환됨에 따라, 버저닝 전략을 함께 진화시킨 기록

---

## 목차

1. [TL;DR](#tldr)
2. [개요](#-개요)
3. [SemVer 기초 정리](#-semver-기초-정리)
4. [버저닝 전략 전환의 배경](#-버저닝-전략-전환의-배경)
5. [업계 벤치마킹: 어떤 전략들이 있나](#-업계-벤치마킹-어떤-전략들이-있나)
6. [우리가 채택한 전략 — 단계적 의사결정](#-우리가-채택한-전략--단계적-의사결정)
7. [Before / After 비교](#-before--after-비교)
8. [최종 파이프라인 구현](#-최종-파이프라인-구현)
9. [운영 흐름 예시](#-운영-흐름-예시)
10. [운영 시 고려할 엣지케이스](#-운영-시-고려할-엣지케이스)
11. [📚 참고 자료](#-참고-자료)

---

## TL;DR

라이브러리 초기 집중 개발기에는 **단일 feature branch에 여러 명이 모여 작업하고 한 번에 버전업하는 직렬 모델**로 충분했고, 기존 `npm version prerelease --preid=beta` 기반 파이프라인은 이 운영 모드에 잘 맞았다. 라이브러리의 큰 그림이 완성되고 컴포넌트 리팩토링·기술 전환·신규 컴포넌트 추가 등 **여러 기술 과제를 병렬로 진행하는 단계**로 넘어오면서, 동시에 진행되는 두 feature가 동일한 `1.5.1-beta.0`을 발급받아 publish가 막히는 상황이 나타났다. 이는 기존 파이프라인의 결함이 아니라 **운영 모델 변화에 따라 버저닝 전략의 진화가 필요한 시점**이라는 신호다. 본 문서는 그 진화의 방향과 결정 과정을 정리한 기록이다. 결과 형식은 `1.5.1-refactoring.beta.0`, `1.6.0-grid.beta.0`처럼 표준 SemVer 2.0.0에 부합하면서 사람이 읽기 좋고, 컨슈머는 정확한 버전을 `package.json`에 pinning하여 재현 가능한 베타 테스트가 가능해진다.

---

## 📌 개요

| 항목 | 내용 |
|---|---|
| 대상 | 컴포넌트 라이브러리 (standalone repo) |
| 배포 채널 | npm Private Registry |
| CI/CD | Bitbucket Pipelines (custom pipelines로 beta / patch / minor 분리 운영) |
| 컨슈머 | 같은 organization 내 별도 웹 프로젝트들 |
| 이전 운영 모드 | 단일 feature branch 집중 개발 → 한 번에 버전업 (직렬) |
| 현재 운영 모드 | 컴포넌트 리팩토링·기술 전환·신규 컴포넌트 추가의 병렬 진행 |
| 전환의 계기 | 병렬 베타 시 동일 prerelease 버전 발급으로 publish 충돌 발생 |
| 진화 방향 | Feature-scoped preid + 자동 build counter + 명시적 base target |
| 영향 범위 | `npm_publish` step 스크립트 + `publish_beta` 파이프라인의 입력 변수 |

---

## 🔢 SemVer 기초 정리

본격적인 전략 논의 전, 팀원 모두가 같은 어휘를 쓸 수 있도록 SemVer 2.0.0 표준의 핵심을 정리한다.

### 기본 형식

```
MAJOR . MINOR . PATCH
```

| 자릿수 | 언제 올리나 | 호환성 | 예시 변경 |
|---|---|---|---|
| **MAJOR** | API에 backward-incompatible 변경 발생 | ❌ 깨짐 | 기존 prop 제거, 컴포넌트 시그니처 변경 |
| **MINOR** | backward-compatible한 신규 기능 추가 | ✅ 유지 | 새 컴포넌트 추가, 새 prop (optional) 추가 |
| **PATCH** | backward-compatible한 버그 수정 | ✅ 유지 | 스타일 오류 수정, 내부 로직 fix |

> 핵심: **호환성을 깨는 변경은 반드시 MAJOR**. "조금 깨는 정도라 minor로 가도 되지 않나"는 SemVer 위반이며 컨슈머에게 silent failure를 안긴다.

### Prerelease Identifier

PATCH 뒤에 하이픈(`-`)을 붙이고 점(`.`)으로 구분된 식별자를 이어붙이면 prerelease 버전이 된다.

```
1.5.0-alpha
1.5.0-alpha.1
1.5.0-beta.2
1.5.0-rc.1
1.5.0-refactoring.beta.0          ← dot-segment로 다층 식별자 가능
```

**표준 규칙 (semver.org spec)**

- 식별자는 ASCII 영숫자와 하이픈만 (`[0-9A-Za-z-]`)
- 식별자는 비어있을 수 없음
- 숫자 식별자는 leading zero 금지 (`1.0.0-01`은 invalid)
- prerelease 버전은 동일 normal 버전보다 **낮은 precedence**를 가짐
  - 즉 `1.5.0-beta.0` < `1.5.0`
- dot-separated 식별자는 좌→우로 순차 비교
  - `1.5.0-alpha < 1.5.0-alpha.1 < 1.5.0-beta < 1.5.0-rc.1 < 1.5.0`

### npm의 prerelease 취급 방식

npm은 SemVer range 매칭에서 **기본적으로 prerelease를 포함하지 않는다.**

```json
{ "dependencies": { "my-lib": "^1.5.0" } }
```

위 range는 `1.5.0`, `1.6.0`, `1.7.3`을 매칭하지만 `1.5.1-beta.0`은 **매칭하지 않는다.** 이 동작 덕분에 베타가 stable 컨슈머에게 실수로 흘러가지 않는다.

베타를 받으려면 컨슈머가 두 방법 중 하나로 명시적으로 받아야 한다:

```bash
# 방법 1: dist-tag로 받기 (최신 베타로 자동 갱신됨)
npm install my-lib@beta

# 방법 2: 정확한 버전 pinning (재현 가능)
npm install my-lib@1.5.1-refactoring.beta.0
```

### dist-tag

npm의 dist-tag는 **버전에 붙는 이름표**다. git tag와 유사하지만 mutable하며, registry에서 "이 이름은 지금 이 버전을 가리킨다"를 표시한다.

```bash
npm publish                    # 기본적으로 latest 태그로 publish
npm publish --tag beta         # beta 태그로 publish
npm dist-tag ls my-lib         # 모든 dist-tag 목록 조회
npm dist-tag rm my-lib beta    # dist-tag 제거
```

`latest`는 `npm install` 시 (별도 지정 없을 때) 기본으로 설치되는 버전을 가리키는 특별한 태그다.

---

## 🔄 버저닝 전략 전환의 배경

### 이전 운영 모드 — 직렬 집중 개발기

라이브러리 초기에는 운영 모델이 명확했다.

- **하나의 feature branch에 여러 명이 모여 작업**
- 작업이 완료되면 **한 번에 버전업** (베타 → 검증 → patch / minor로 promote)
- 같은 시점에 라이브러리 다른 부분이 별도 branch로 동시 진행되는 일이 거의 없음

이 운영 모드에서는 prerelease counter가 단일이어도 충돌이 발생하지 않았다. 베타 → patch promote 사이클이 항상 직렬이었기 때문이다.

### 기존 publish step의 동작

```bash
LOCAL_PACKAGE_VERSION=$(node -p -e "require('./package.json').version")
if [ $UPDATE_VERSION == 'beta' ]; then
  npm version prerelease --preid=beta -m "Upgrade to %s [skip ci]"
else
  npm version $UPDATE_VERSION -m "Upgrade to %s [skip ci]"
fi
# ... publish 후
git push --follow-tags
```

`npm version prerelease --preid=beta`는 `package.json`의 현재 버전을 기반으로 단일 prerelease counter를 1 증가시킨다.

| 현재 버전 | 명령 실행 후 |
|---|---|
| `1.5.0` | `1.5.1-beta.0` |
| `1.5.1-beta.0` | `1.5.1-beta.1` |
| `1.5.1-beta.1` | `1.5.1-beta.2` |

직렬 모델에서는 이 단일 counter가 곧 라이브러리 전체의 "다음 베타 후보 번호"였고, 자연스러운 진행이었다.

### 현재 운영 모드 — 안정화 후 병렬 기술 과제 수행기

라이브러리의 큰 그림이 완성된 후, 운영 모드가 바뀌었다.

- **컴포넌트 리팩토링** — 기존 컴포넌트를 점진적으로 개선
- **기술 전환** — 빌드 도구 / 디자인 토큰 시스템 등 인프라 레벨 업그레이드
- **신규 컴포넌트 추가** — 라이브러리 surface 확장
- 각각이 **별도의 feature branch에서 병렬로 진행**되며, 베타 → 정식 출시 사이클도 각자의 속도로 흘러감

병렬화는 더 이상 예외 상황이 아니라 **상시 운영 모드**가 되었다.

### 단일 counter가 병렬 모드와 만났을 때

이 새로운 운영 모드에서 기존 단일 counter 방식의 한계가 드러난다.

```
[시점 T0]
main 브랜치 package.json: 1.5.0

[시점 T1] feature/refactoring 브랜치 작업자가 publish_beta 실행
  - LOCAL_PACKAGE_VERSION = 1.5.0
  - npm version prerelease → 1.5.1-beta.0
  - registry에 1.5.1-beta.0 publish 성공
  - git push로 feature/refactoring 브랜치에 commit (package.json: 1.5.1-beta.0)

[시점 T2] feature/grid 브랜치 작업자가 publish_beta 실행 (병렬)
  - feature/grid는 1.5.0에서 분기됨
  - LOCAL_PACKAGE_VERSION = 1.5.0 (feature/refactoring의 commit과 격리됨)
  - npm version prerelease → 1.5.1-beta.0
  - registry에 publish 시도 → ❌ 409 Conflict
```

### 기존 전략의 가정과 현재 요구 사이의 간극

기존 파이프라인이 잘못 설계된 것이 아니라, **그 시점의 운영 모델 가정 위에서 설계된 도구**였다. 운영 모델이 바뀐 지금, 그 가정과 새로운 요구 사이에 세 가지 간극이 보인다.

| 기존 전략의 가정 | 현재 운영 모드의 요구 |
|---|---|
| 베타 카운터는 라이브러리 전체에 하나면 충분하다 | feature별로 독립된 카운터가 필요하다 |
| 베타도 git history에 남겨 추적한다 | 병렬 branch 간 `package.json` state 격리가 필요하다 |
| 컨슈머는 dist-tag 없이도 어떤 베타인지 알 수 있다 | feature 단위로 명확한 라벨이 있어야 컨슈머가 구분 가능하다 |

다음 섹션부터는 이 간극을 메우기 위해 업계는 어떤 전략을 사용하는지 살펴보고, 우리 환경에 맞게 어떤 형태로 조합했는지 정리한다.

---

## 🌐 업계 벤치마킹: 어떤 전략들이 있나

병렬 베타 운영은 라이브러리 생태계에서 흔히 마주치는 단계이며, 대형 OSS들은 이미 자신만의 방식으로 이 단계를 거쳐왔다. 각 전략과 그 출처를 정리한다.

### 1. React Release Channels

React는 `Latest`, `Canary`, `Experimental` 세 채널을 운영한다.

- **Latest**: 전통적인 SemVer를 엄수하는 stable 채널
- **Canary**: main 브랜치를 추적하는 prerelease 채널. **컨슈머에게 pin을 요구하며 breaking change가 포함될 수 있다고 명시.**
- **Experimental**: 빌드 내용과 커밋 날짜로 unique한 버전 생성 (예: `0.0.0-experimental-68053d940-20210623`). **충돌이 구조적으로 불가능한 패턴.**

> 시사점: 비-stable 채널은 SemVer를 느슨하게 쓰되, 컨슈머에게 명시적 pinning을 요구하는 것이 표준 운영 방식이다.

출처: [React Versioning Policy](https://react.dev/community/versioning-policy), [React Canaries 공식 블로그](https://react.dev/blog/2023/05/03/react-canaries)

### 2. Next.js Stable / Canary 채널

Next.js는 두 채널만 운영한다.

- **Stable**: `npm install next` 시 받는 기본 채널, semver 엄수
- **Canary**: `npm install next@canary`로 명시적 설치, stable로 갈 모든 변경사항을 미리 담는 채널

> 시사점: 채널을 단순화하더라도 dist-tag로 명확히 분리하면 운영이 깔끔하다.

출처: [Next.js Release Channels 문서](https://github.com/vercel/next.js/blob/canary/contributing/repository/release-channels-publishing.md)

### 3. semantic-release Pre-release Branches

semantic-release는 **branch 단위로 prerelease identifier를 매핑**한다.

```
main 브랜치 → @latest, semver 엄수 (예: 1.0.1)
beta 브랜치 → @beta, 2.0.0-beta.1, 2.0.0-beta.2 ...
alpha 브랜치 → @alpha, 3.0.0-alpha.1 ... (병렬 major 작업)
```

> 시사점: 병렬 prerelease는 **brand별 long-lived branch + 각자의 preid**로 풀 수 있다.

출처: [semantic-release Pre-releases 공식 recipe](https://semantic-release.gitbook.io/semantic-release/recipes/release-workflow/pre-releases)

### 4. Oleksii Popov의 Branch-Scoped Alias 패턴

Branch 이름을 prerelease identifier alias로 박는 실용 패턴.

```
feature/sunfish 브랜치 → v0.0.2-sunfish.0
컨슈머: npm install pkg@sunfish
```

> 시사점: 회사 내부에서 dist-tag와 preid를 branch/feature name과 1:1로 매핑하면 충돌이 원천 차단된다.

출처: [Feature branches approach in CI/CD of NPM libraries — Oleksii Popov](https://oleksiipopov.com/blog/feature-branches-approach-in-ci-cd-of-npm-libraries/)

### 5. Changesets Snapshot 모드

Changesets는 `--snapshot <tag>` 옵션으로 `{tag}-{datetime}` 형태의 일회성 버전을 발급한다. 1.0+ 에서는 `snapshot.prereleaseTemplate` 설정으로 commit SHA 등 커스텀 식별자 조합 가능.

> 시사점: 본 프로젝트는 Changesets를 도입하지 않지만, "registry에 누적되는 임시 버전 + tag-based 식별"이라는 컨셉을 차용했다.

출처: [Changesets Prereleases 문서](https://github.com/changesets/changesets/blob/main/docs/prereleases.md), [snapshot.prereleaseTemplate PR](https://github.com/changesets/changesets/pull/858)

### 벤치마킹 종합

각 전략의 핵심 아이디어를 추리면:

| 아이디어 | 출처 |
|---|---|
| Feature/branch name을 preid에 박아 카운터 분리 | Oleksii Popov, semantic-release |
| Build identifier(SHA, build number)로 unique 버전 보장 | React Experimental, Changesets snapshot |
| dist-tag로 feature별 채널 분리 + 컨슈머는 pinning 권장 | React Canary, Next.js canary |
| Prerelease는 git history에 commit하지 않음 | React, Next.js, semantic-release (자동화) |

이 네 가지 모두를 우리 환경에 맞게 조합한다.

---

## 🪜 우리가 채택한 전략 — 단계적 의사결정

병렬 운영 모드의 요구를 만족시키면서도 표준 SemVer 위에서 동작하고, 컨슈머의 기존 사용 패턴(직접 pinning)을 깨지 않는 방향으로 단계적으로 결정했다. 각 결정의 근거와 함께 순서대로 기록한다.

### Decision 1. Feature-scoped Prerelease Identifier

**결정**: prerelease identifier에 feature name을 dot-segment로 포함시켜 카운터를 feature별로 분리한다.

**형식**:
```
{base}-{feature}.beta.{build}
```

**근거**: SemVer 2.0.0이 dot-separated identifier를 표준으로 허용하며, 두 feature가 서로 다른 식별자 공간을 갖게 되어 충돌이 구조적으로 불가능해진다.

### Decision 2. SemVer-Clean Format 선호 (SHA suffix 배제)

**고려안 두 가지**:

| 옵션 A: SHA suffix | 옵션 B: SemVer-clean |
|---|---|
| `1.5.1-refactoring.a3f9c12` | `1.5.1-refactoring.beta.0` |
| 충돌 불가능, 자동화 단순 | 사람이 읽기 좋음, 표준 SemVer |
| 가독성 떨어짐 | build counter 관리 필요 |

**결정**: **옵션 B (SemVer-clean)**. 내부 라이브러리이고 PRD 동안 개발자끼리 베타 버전을 공유해 읽기 때문에 가독성이 우선.

**근거**: 컨슈머가 베타 버전을 직접 `package.json`에 pinning하여 사용 중이므로, 버전 문자열을 사람이 읽고 의미 파악할 수 있어야 한다.

### Decision 3. Build Counter는 Registry 조회로 자동 계산

**결정**: `npm view <pkg> versions --json`으로 동일 prefix의 기존 버전 중 max build number를 찾고 +1.

**근거**:
- `$BITBUCKET_BUILD_NUMBER`는 파이프라인 전역 공유라 숫자가 띄엄띄엄해져 가독성이 떨어진다.
- Registry 조회 방식은 feature별로 0부터 시작하는 깨끗한 카운터를 만들어준다.
- 같은 branch 동시 publish race는 Bitbucket Deployment concurrency=1로 방지 가능.

### Decision 4. `BETA_TARGET` 변수로 base 버전 명시

**결정**: 베타 publish 시 `BETA_TARGET=patch | minor`를 받아서 base 버전을 분기 계산.

**근거**: 베타 버전 문자열의 base 숫자가 최종 stable 출시 버전과 일치해야 컨슈머가 "이 베타가 어디로 갈지" 예측할 수 있다. 예를 들어 `1.6.0-grid.beta.0`을 본 개발자는 "다음 minor 출시(`1.6.0`)에 들어갈 기능이구나"를 즉시 파악할 수 있다.

| BETA_TARGET | base 계산 | 예 (현재 1.5.0) |
|---|---|---|
| `patch` (기본값) | `c+1` | `1.5.1-{feat}.beta.{n}` |
| `minor` | `b+1.0` | `1.6.0-{feat}.beta.{n}` |

### Decision 5. dist-tag는 자동 발급, 컨슈머는 Pinning이 표준

**결정**: 베타 publish 시 dist-tag를 `{feature}-beta`로 자동 발급. 단, **컨슈머의 표준 사용법은 dist-tag가 아닌 정확한 버전 pinning**으로 명시.

**근거**:
- 컨슈머가 PRD 중 `"private-library": "1.4.12-refactoring.beta.0"` 처럼 직접 pinning하는 패턴이 이미 자리잡혀 있다.
- Pinning은 재현 가능한 빌드 측면에서 오히려 권장되는 패턴이다 (React Canary 정책과 일치).
- dist-tag는 "최신 베타가 뭔지 찾을 때 쓰는 인덱스" 용도로 보조.

### Decision 6. 베타는 Git에 Commit하지 않음

**결정**: 베타 publish 시 `npm version <ver> --no-git-tag-version`을 사용하여 `package.json`만 일시적으로 변경하고 git push하지 않는다. patch / minor는 기존대로 commit + push.

**근거**:
- 베타가 git history에 박히면 다음 patch 계산이 베타 버전 위에서 이루어져 의도와 다른 결과가 나온다.
- main 브랜치 `package.json`이 항상 마지막 stable 상태로 유지되어야 `npm version patch`가 깨끗하게 동작한다.
- Registry가 베타의 single source of truth가 되고, git은 stable 릴리스만 추적한다.

---

## 🔁 Before / After 비교

### 파이프라인 입력 변수

| 항목 | Before | After |
|---|---|---|
| beta 입력 변수 | `GIT_TAG`만 | `GIT_TAG` + `FEATURE_NAME` + `BETA_TARGET` |
| FEATURE_NAME | (없음) | **필수** — feature 식별자 |
| BETA_TARGET | (없음) | `patch` 기본, `minor` 선택 가능 |
| dist-tag | 입력값으로 받으나 거의 미사용 | 자동 발급 (`{feature}-beta`) |

### 베타 버전 형식

| 시나리오 | Before | After |
|---|---|---|
| feature1 첫 베타 | `1.5.1-beta.0` | `1.5.1-refactoring.beta.0` |
| feature2 첫 베타 (병렬) | `1.5.1-beta.0` ❌ 충돌 | `1.5.1-grid.beta.0` ✅ |
| feature2가 minor 타겟일 때 | (구분 불가) | `1.6.0-grid.beta.0` (의도 명시) |

### Git Push 동작

| 파이프라인 | Before | After |
|---|---|---|
| publish_beta | `git push --follow-tags` | **push하지 않음** |
| publish_patch | `git push --follow-tags` | 동일 (변경 없음) |
| publish_minor | `git push --follow-tags` | 동일 (변경 없음) |

### 충돌 가능성

| 시나리오 | Before | After |
|---|---|---|
| 동일 feature 같은 branch 순차 publish | 정상 동작 | 정상 동작 |
| 다른 feature 병렬 publish | ❌ 409 Conflict | ✅ 독립적으로 publish |
| 동일 feature 동시 publish (드문 케이스) | ❌ Conflict 가능 | Deployment concurrency=1로 방지 권장 |

---

## ⚙️ 최종 파이프라인 구현

```yaml
image: node:20  # Node 16 → 20 업그레이드 (EOL 회피)

definitions:
  steps:
    - step: &storybook_Build_Deploy
        # 기존 그대로 유지 (변경 없음)
        name: 'Build & Deploy: storybook'
        size: 4x
        # ... (생략)

    - step: &npm_publish
        name: 'Build & Publish'
        script:
          - printf "//`node -p \"require('url').parse(process.env.NPM_REGISTRY_URL || 'https://registry.npmjs.org').host\"`/:_authToken=${NPM_TOKEN}\nregistry=${NPM_REGISTRY_URL:-https://registry.npmjs.org}\nunsafe-perm=true\n" >> ~/.npmrc
          - npm install
          - npm run build
          - git checkout . && git clean -f .
          - LOCAL_PACKAGE_VERSION=$(node -p -e "require('./package.json').version")
          - >
            if [ "$UPDATE_VERSION" == "beta" ]; then
              # ===== 입력 검증 =====
              if [ -z "$FEATURE_NAME" ]; then
                echo "ERROR: FEATURE_NAME is required for beta publish"
                exit 1
              fi
              if [ "$BETA_TARGET" != "patch" ] && [ "$BETA_TARGET" != "minor" ]; then
                echo "ERROR: BETA_TARGET must be 'patch' or 'minor'"
                exit 1
              fi

              # ===== 1. FEATURE_NAME 안전화 (소문자, 영숫자/하이픈만) =====
              SAFE_FEATURE=$(echo "$FEATURE_NAME" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9-]/-/g')

              # ===== 2. BETA_TARGET에 따라 base 버전 계산 =====
              BASE_VERSION=$(echo "$LOCAL_PACKAGE_VERSION" | sed 's/-.*//')
              if [ "$BETA_TARGET" == "minor" ]; then
                TARGET_VERSION=$(node -e "const [a,b]='${BASE_VERSION}'.split('.').map(Number);console.log(\`\${a}.\${b+1}.0\`)")
              else
                TARGET_VERSION=$(node -e "const [a,b,c]='${BASE_VERSION}'.split('.').map(Number);console.log(\`\${a}.\${b}.\${c+1}\`)")
              fi

              # ===== 3. Registry 조회로 build number 자동 계산 =====
              PKG_NAME=$(node -p -e "require('./package.json').name")
              PREFIX="${TARGET_VERSION}-${SAFE_FEATURE}.beta."
              NPM_VIEW=$(npm view "${PKG_NAME}" versions --json 2>/dev/null || echo "[]")
              NEW_BUILD=$(node -e "
                const data = JSON.parse(process.argv[1]);
                const versions = Array.isArray(data) ? data : [data];
                const prefix = process.argv[2];
                const builds = versions
                  .filter(v => typeof v === 'string' && v.startsWith(prefix))
                  .map(v => parseInt(v.slice(prefix.length), 10))
                  .filter(n => Number.isInteger(n));
                console.log(builds.length ? Math.max(...builds) + 1 : 0);
              " "${NPM_VIEW}" "${PREFIX}")

              NEW_VERSION="${PREFIX}${NEW_BUILD}"
              DIST_TAG="${SAFE_FEATURE}-beta"

              echo "=========================================="
              echo "✓ Publishing beta: ${NEW_VERSION}"
              echo "  dist-tag: ${DIST_TAG}"
              echo "  Pin in package.json:"
              echo "  \"${PKG_NAME}\": \"${NEW_VERSION}\""
              echo "=========================================="

              # ===== 4. Git에 commit하지 않고 publish =====
              npm version "${NEW_VERSION}" --no-git-tag-version
              npm publish --tag "${DIST_TAG}"

              # ===== 5. Working tree 원복 =====
              git checkout .
            else
              # ===== patch / minor: 기존 룰 그대로 =====
              npm version "$UPDATE_VERSION" -m "Upgrade to %s [skip ci]"
              npm publish
            fi
          - >
            if [ "$UPDATE_VERSION" != "beta" ]; then
              if [ "$GIT_TAG" == "false" ]; then
                git push
              else
                git push --follow-tags
              fi
            fi

  custom:
    - variables: &npm_publish_tag
      - name: GIT_TAG
        default: "false"
        allowed-values:
          - "true"
          - "false"
    - variables: &npm_publish_beta_vars
      - <<: *npm_publish_tag
      - name: FEATURE_NAME
        description: "Feature 식별자 (예: refactoring, grid-layout, search-v2)"
      - name: BETA_TARGET
        default: "patch"
        allowed-values:
          - "patch"
          - "minor"
        description: "이 베타가 최종 어느 단계로 출시될지 (patch=버그픽스, minor=새 기능)"

pipelines:
  custom:
    0.storybook:
      - step:
          <<: *storybook_Build_Deploy
          deployment: Development
    1.publish_beta:
      - variables: *npm_publish_beta_vars
      - step:
          <<: *npm_publish
          deployment: Publish_Beta
    2.publish_patch:
      - step:
          <<: *npm_publish
          deployment: Publish_Patch
    3.publish_minor:
      - step:
          <<: *npm_publish
          deployment: Publish_Minor
```

---

## 🎬 운영 흐름 예시

### 시나리오: 안정 버전 `1.5.0`에서 두 feature 병렬 진행

```
[T0] 안정 버전: @yourorg/lib@1.5.0

[T1] refactoring feature (버그픽스 성격) 베타 시작
    publish_beta { FEATURE_NAME=refactoring, BETA_TARGET=patch }
    → Registry 조회: 1.5.1-refactoring.beta.* 없음
    → Publish: 1.5.1-refactoring.beta.0
    → dist-tag: refactoring-beta

[T2] grid feature (새 기능 성격) 베타 시작 — refactoring과 병렬
    publish_beta { FEATURE_NAME=grid, BETA_TARGET=minor }
    → Registry 조회: 1.6.0-grid.beta.* 없음
    → Publish: 1.6.0-grid.beta.0
    → dist-tag: grid-beta
    ✅ 충돌 없음

[T3] refactoring 추가 수정
    publish_beta { FEATURE_NAME=refactoring, BETA_TARGET=patch }
    → Registry 조회: 1.5.1-refactoring.beta.0 발견, max=0
    → Publish: 1.5.1-refactoring.beta.1
    → dist-tag: refactoring-beta (덮어쓰기)

[T4] refactoring 머지 후 정식 출시
    publish_patch
    → package.json은 여전히 1.5.0 (베타가 commit되지 않았으므로)
    → npm version patch → 1.5.1
    → Publish: 1.5.1
    → dist-tag: latest

[T5] grid 머지 후 정식 출시
    publish_minor
    → package.json은 1.5.1 (refactoring patch 반영)
    → npm version minor → 1.6.0
    → Publish: 1.6.0
    → dist-tag: latest

[T6 / 선택] dist-tag 정리
    npm dist-tag rm @yourorg/lib refactoring-beta
    npm dist-tag rm @yourorg/lib grid-beta
```

### 컨슈머 측 사용 예시

```json
// 웹 프로젝트 package.json
{
  "dependencies": {
    "@yourorg/lib": "1.5.1-refactoring.beta.1"  // ← 정확한 버전 pinning
  }
}
```

또는 (최신 베타를 자동으로 받고 싶을 때):

```bash
npm install @yourorg/lib@refactoring-beta
```

---

## ⚠️ 운영 시 고려할 엣지케이스

### 1. 베타 도중 BETA_TARGET이 바뀔 때

`refactoring`를 `patch`로 진행하다가 "이거 사실 minor야"로 바뀌면 `BETA_TARGET=minor`로 다시 publish하면 base가 `1.5.1` → `1.6.0`으로 바뀌면서 build number가 0부터 새로 시작한다.

- 이전 `1.5.1-refactoring.beta.*` 베타는 registry에 그대로 남는다.
- dist-tag `refactoring-beta`는 새 버전을 가리키므로 컨슈머는 자동으로 올바른 쪽을 받는다.
- 옛 베타들은 그대로 두어도 무방하며, 누적이 신경 쓰이면 분기별로 `npm deprecate`로 정리 권장 (`npm unpublish`는 24시간 룰과 lockfile 깨짐 위험이 있어 비권장).

### 2. 같은 feature 동시 publish race

이론적으로 같은 feature branch에서 두 CI가 동시에 돌면 registry 조회 결과가 같아서 둘 다 같은 build number를 계산할 수 있다.

- Bitbucket Pipelines는 보통 같은 branch에서 순차 실행되므로 현실적으로 거의 발생하지 않음.
- 완벽한 안전을 위해 **Bitbucket Deployment 설정 → Concurrency: 1**로 설정 권장.

### 3. dist-tag 누적

머지 후 feature dist-tag를 그대로 두면 시간이 갈수록 dist-tag 리스트가 지저분해진다.

- 옵션 A: 머지 후 수동으로 `npm dist-tag rm` (가장 간단)
- 옵션 B: `publish_patch` / `publish_minor` 스크립트에 "최근 6개월 이상 미사용 dist-tag 자동 제거" 로직 추가
- 옵션 C: 분기별 점검 (`npm dist-tag ls @yourorg/lib`) 루틴화

### 4. Registry 조회 권한

`npm view <pkg> versions`는 .npmrc에 인증 토큰이 있어야 한다. 위 파이프라인은 step 시작에 이미 토큰을 .npmrc에 쓰므로 문제 없음. Private registry라면 토큰에 read 권한이 포함되어 있는지 확인 필요.

### 5. Node 16 → 20 업그레이드

기존 `image: node:16`은 EOL 상태. Node 20+로 올리는 김에 npm 최신 동작(특히 dist-tag, prerelease 처리)도 안정적으로 확보 가능. 컴포넌트 라이브러리 빌드 호환성만 사전 확인하면 큰 부담은 없는 변경.

---

## 📚 참고 자료

### SemVer 표준
- [Semantic Versioning 2.0.0 (semver.org)](https://semver.org/) — 본 문서가 따르는 SemVer 표준 명세
- [npm semver 패키지 문서](https://docs.npmjs.com/cli/v6/using-npm/semver/) — npm CLI의 prerelease range 매칭 동작

### 벤치마킹한 OSS 전략
- [React Versioning Policy](https://react.dev/community/versioning-policy) — 채널 분리 운영 원칙
- [React Canaries 공식 블로그](https://react.dev/blog/2023/05/03/react-canaries) — Canary 채널 도입 배경
- [Next.js Release Channels](https://github.com/vercel/next.js/blob/canary/contributing/repository/release-channels-publishing.md) — stable / canary 이원화
- [semantic-release Pre-releases recipe](https://semantic-release.gitbook.io/semantic-release/recipes/release-workflow/pre-releases) — branch별 preid 매핑
- [Feature branches approach in CI/CD of NPM libraries — Oleksii Popov](https://oleksiipopov.com/blog/feature-branches-approach-in-ci-cd-of-npm-libraries/) — branch-scoped alias 패턴
- [Changesets Prereleases 문서](https://github.com/changesets/changesets/blob/main/docs/prereleases.md) — snapshot 모드 컨셉
- [npm Package Versioning With SemVer — Inedo](https://blog.inedo.com/npm/smarter-npm-versioning-with-semver) — 병렬 prerelease 충돌 회피 사례

### Bitbucket Pipelines
- [Bitbucket Pipelines — Custom Pipelines](https://support.atlassian.com/bitbucket-cloud/docs/run-pipelines-manually/) — custom 파이프라인 입력 변수
- [Bitbucket Deployments — Concurrency](https://support.atlassian.com/bitbucket-cloud/docs/set-up-and-monitor-deployments/) — deployment 동시 실행 제어