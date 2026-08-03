# 투자의 사계 — 프론트엔드 아키텍처 설계서

| 항목 | 내용 |
| --- | --- |
| 문서 성격 | 상세 설계 문서 (구현 기준) |
| 상위 문서 | [PRD](./PRD.md) |
| 선행 문서 | [API 명세](./API-SPEC.md), [디자인 시스템](./DESIGN-SYSTEM.md) |
| 기술 스택 | React 19 / Vite / TypeScript / TailwindCSS v3 |
| 범위 | 레이어 구조, 디렉토리, 상태 관리, 화면 구성, 품질 게이트 |

---

## 목차

1. [아키텍처 원칙](#1-아키텍처-원칙)
2. [기술 선택과 근거](#2-기술-선택과-근거)
3. [레이어 구조와 의존 규칙](#3-레이어-구조와-의존-규칙)
4. [디렉토리 구조](#4-디렉토리-구조)
5. [TypeScript 제약과 코딩 규약](#5-typescript-제약과-코딩-규약)
6. [라우팅](#6-라우팅)
7. [API 클라이언트 계층](#7-api-클라이언트-계층)
8. [서버 상태 관리](#8-서버-상태-관리)
9. [클라이언트 상태 관리](#9-클라이언트-상태-관리)
10. [엔티티 계층](#10-엔티티-계층)
11. [피처 계층](#11-피처-계층)
12. [위젯과 페이지](#12-위젯과-페이지)
13. [화면 인벤토리](#13-화면-인벤토리)
14. [폼 처리](#14-폼-처리)
15. [수치 표현 계층](#15-수치-표현-계층)
16. [오류·로딩 처리](#16-오류로딩-처리)
17. [디자인 시스템 연결](#17-디자인-시스템-연결)
18. [PWA와 알림](#18-pwa와-알림)
19. [성능](#19-성능)
20. [접근성 구현](#20-접근성-구현)
21. [테스트 전략](#21-테스트-전략)
22. [품질 게이트](#22-품질-게이트)
23. [빌드와 배포](#23-빌드와-배포)
24. [확장 대비](#24-확장-대비)

---

## 1. 아키텍처 원칙

| # | 원칙 | 귀결 |
| --- | --- | --- |
| 1 | **화면은 서버가 준 것을 그대로 보여준다** | 수치 재계산·문구 생성·표시 임계값 판정을 클라이언트가 하지 않는다. PRD 17장의 표현 규칙과 7.11의 정직성 요구를 서버가 통제하기 때문이다 |
| 2 | **도메인 개념이 폴더 구조에 드러난다** | `components/`, `hooks/`, `utils/` 같은 기술 축 분할을 쓰지 않는다. '가설 기록', '되돌아볼 조건', '깨달은 것'이 디렉토리 이름으로 존재한다 |
| 3 | **의존 방향은 한쪽이다** | 상위 레이어가 하위를 쓰고 그 반대는 없다. 순환 의존이 생기면 화면 하나를 고칠 때 전체가 흔들린다 |
| 4 | **서버 상태와 클라이언트 상태를 섞지 않는다** | 서버에서 온 것은 캐시이지 앱 상태가 아니다. 전역 스토어에 API 응답을 복사하지 않는다 |
| 5 | **입력 마찰이 코드 구조에 반영된다** | 자동 저장·부분 저장·시트 상태 보존이 폼 계층의 기본 동작이다 (PRD 4.3) |
| 6 | **모바일이 기본 경로다** | 반응형 분기에서 모바일이 기본값이고 데스크톱이 예외다 |
| 7 | **금지된 것은 타입과 린트로 막는다** | 대칭 짝 없는 항목 렌더링, 기준일 없는 수치 표시, 금지 용어 사용을 컴파일·CI 단계에서 차단한다 |

---

## 2. 기술 선택과 근거

| 영역 | 선택 | 근거 | 대안 검토 |
| --- | --- | --- | --- |
| 빌드 | Vite | 기존 프로젝트 구성. HMR 속도 | — |
| 라우팅 | React Router v7 (declarative mode) | 중첩 레이아웃, 코드 스플리팅. SPA 전용이므로 프레임워크 불필요 | TanStack Router(타입 안전성은 우수하나 팀 학습 비용) |
| 서버 상태 | TanStack Query v5 | 캐시·무효화·재시도·낙관적 갱신 표준. 이 앱의 화면 대부분이 서버 데이터 조회 | SWR(무효화 제어 부족), 직접 구현(재발명) |
| 클라이언트 상태 | Zustand | 전역 상태가 매우 적다(테마, 토스트, 시트). Context 남발보다 명시적 | Redux(과잉), Context(리렌더 문제) |
| 폼 | React Hook Form + Zod | 비제어 기반 성능, 스키마 검증 재사용 | Formik(유지보수 정체) |
| 스타일 | TailwindCSS v3 + CVA | 디자인 시스템 문서의 토큰 체계와 직결 | CSS Modules(토큰 강제 어려움) |
| 차트 | Recharts | 디자인 시스템 §14 결정 | — |
| 날짜 | date-fns + `ko` 로케일 | 트리 셰이킹. Temporal 미성숙 | dayjs(플러그인 의존) |
| 수치 | decimal.js-light | 금액·비율을 문자열로 받아 정확히 다룬다 | 네이티브 number(정밀도 손실) |
| 테스트 | Vitest + Testing Library + MSW | Vite 통합. 현재 테스트 러너 없음 → 신규 도입 | Jest(Vite와 이중 설정) |
| E2E | Playwright | 주요 여정 검증 | Cypress |
| PWA | vite-plugin-pwa | 푸시 알림·설치 필요 | 수동 SW 관리 |

---

## 3. 레이어 구조와 의존 규칙

**Feature-Sliced Design**을 채택한다. 이 제품은 도메인 개념이 많고(4개 계좌, 가설 기록, 조건, 수칙, 레슨) 화면 간 재사용이 복잡해 기술 축 분할로는 관리되지 않는다.

```
app/         앱 초기화, 프로바이더, 라우터, 전역 스타일
  ↓
pages/       라우트 단위 화면 조립
  ↓
widgets/     여러 엔티티·피처를 결합한 독립 UI 블록
  ↓
features/    사용자 행동 단위 (매매 기입, 조건 응답, 수칙 확정)
  ↓
entities/    도메인 객체와 그 표현 (계좌, 종목, 가설, 일지, 수칙)
  ↓
shared/      도메인 무관 공통 (UI 프리미티브, API 클라이언트, 유틸)
```

### 3.1 의존 규칙

| 레이어 | import 허용 | 금지 |
| --- | --- | --- |
| `app` | 전부 | — |
| `pages` | widgets, features, entities, shared | 다른 page |
| `widgets` | features, entities, shared | pages, 다른 widget |
| `features` | entities, shared | pages, widgets, 다른 feature |
| `entities` | shared, 다른 entity의 `@x` 공개 API | features, widgets, pages |
| `shared` | shared 내부만 | 상위 전부 |

**강제 수단**: `eslint-plugin-boundaries` 또는 `eslint-plugin-import` 규칙으로 CI에서 검증한다.

### 3.2 슬라이스 내부 구조

각 슬라이스는 세그먼트로 나뉜다.

| 세그먼트 | 내용 |
| --- | --- |
| `ui/` | 컴포넌트 |
| `model/` | 타입, 훅, 상태, 셀렉터 |
| `api/` | 쿼리·뮤테이션 정의 |
| `lib/` | 슬라이스 전용 유틸 |
| `config/` | 상수, 매핑 |
| `index.ts` | **공개 API.** 외부는 이것만 import |

**공개 API 규칙**: 슬라이스 내부 파일을 외부에서 직접 import하지 않는다. 이 규칙이 없으면 리팩터링 시 어디가 깨질지 알 수 없다.

### 3.3 엔티티 간 참조

엔티티끼리는 원칙적으로 참조하지 않지만, 도메인상 불가피한 경우(가설 → 종목, 조건 → 가설)가 있다. 이때는 `entities/<slice>/@x/<other-slice>.ts`로 **교차 참조 전용 공개 API**를 만든다. 무분별한 상호 참조를 막으면서 필요한 연결은 허용하는 FSD 표준 방식이다.

---

## 4. 디렉토리 구조

```
frontend/
├── index.html
├── vite.config.ts
├── tailwind.config.ts
├── tsconfig.json / tsconfig.app.json / tsconfig.node.json
├── public/
│   ├── icons.svg                        # SVG 스프라이트
│   ├── fonts/
│   ├── manifest.webmanifest
│   └── og/
└── src/
    ├── main.tsx
    ├── app/
    │   ├── App.tsx
    │   ├── providers/
    │   │   ├── QueryProvider.tsx
    │   │   ├── RouterProvider.tsx
    │   │   ├── ThemeProvider.tsx
    │   │   ├── ToastProvider.tsx
    │   │   ├── AuthProvider.tsx
    │   │   └── index.tsx                # 프로바이더 합성
    │   ├── router/
    │   │   ├── routes.tsx               # 라우트 정의
    │   │   ├── guards.tsx               # 인증·온보딩 가드
    │   │   └── paths.ts                 # 경로 상수
    │   ├── layouts/
    │   │   ├── AppLayout.tsx            # 헤더 + 탭바
    │   │   ├── OnboardingLayout.tsx
    │   │   ├── FullscreenLayout.tsx     # 연습 모드, 결산
    │   │   └── AuthLayout.tsx
    │   └── styles/
    │       └── index.css
    │
    ├── pages/
    │   ├── landing/
    │   ├── auth-callback/
    │   ├── onboarding/
    │   │   ├── survey/
    │   │   ├── practice-intro/
    │   │   ├── draft-principle/
    │   │   ├── schedule-review/
    │   │   ├── stock-selection/
    │   │   └── first-hypothesis/
    │   ├── cycle-start/                  # 두 번째 사이클 이후
    │   │   ├── principle-review/
    │   │   ├── schedule-review/
    │   │   ├── stock-selection/
    │   │   ├── first-hypothesis/
    │   │   └── prediction/
    │   ├── home/
    │   ├── stocks/
    │   │   ├── pool/
    │   │   └── detail/
    │   ├── trades/
    │   │   ├── list/
    │   │   └── entry/
    │   ├── conditions/
    │   ├── journal/
    │   │   ├── write/
    │   │   └── calendar/
    │   ├── records/
    │   │   ├── insights/
    │   │   ├── principles/
    │   │   └── past-reports/
    │   ├── performance/
    │   ├── rebalance/
    │   ├── practice/
    │   ├── settlement/
    │   │   ├── overview/
    │   │   ├── retrospective/
    │   │   ├── report/
    │   │   └── principle-setup/
    │   ├── badges/
    │   ├── notifications/
    │   ├── settings/
    │   └── errors/
    │
    ├── widgets/
    │   ├── app-header/
    │   ├── tab-bar/
    │   ├── account-overview/            # 4계좌 2×2 요약
    │   ├── performance-chart-panel/     # 곡선 + 범례 + 기준일
    │   ├── gap-summary/                 # 세 개의 차이
    │   ├── deviation-breakdown/         # 항목별 내역(대칭 렌더링)
    │   ├── cycle-schedule/              # 남은 조정일·결산일
    │   ├── market-summary/
    │   ├── holdings-list/
    │   ├── stock-record-timeline/       # 종목별 내 기록 모아보기
    │   ├── journal-composer/            # 일지 작성 통합
    │   ├── stamp-calendar/
    │   ├── insight-collection/          # 주제별 모아보기
    │   ├── principle-board/
    │   ├── recall-panel/                # 과거 기록 자동 노출
    │   ├── condition-alert-panel/
    │   ├── rebalance-detail/
    │   ├── report-viewer/               # 올해의 투자 레슨 렌더러
    │   ├── practice-player/
    │   └── notice-stack/                # meta.notice 렌더링
    │
    ├── features/
    │   ├── auth-kakao/
    │   ├── onboarding-step/
    │   ├── survey-submit/
    │   ├── stock-select/
    │   ├── trade-entry/                 # 매매 기입 + 가설 기록
    │   ├── trade-correct/
    │   ├── hypothesis-fill/             # 빈칸 나중 채우기
    │   ├── condition-replan/
    │   ├── condition-self-report/
    │   ├── condition-respond/
    │   ├── trade-review-submit/
    │   ├── journal-write/
    │   ├── insight-mark/                # 별표 + 태그
    │   ├── principle-create/
    │   ├── principle-mark-kept/
    │   ├── principle-verdict/
    │   ├── rebalance-confirm/
    │   ├── prediction-submit/
    │   ├── emergency-declare/
    │   ├── practice-play/
    │   ├── retrospective-submit/        # 사이클 회고 (결산 3단계)
    │   ├── condition-create/            # 조건 없던 종목에 조건 추가
    │   ├── report-decision/             # 잘한/아쉬운 판단 선택
    │   ├── cycle-restart/
    │   ├── export-data/
    │   ├── withdraw-account/
    │   └── notification-settings/
    │
    ├── entities/
    │   ├── user/
    │   ├── cycle/
    │   ├── account/
    │   ├── stock/
    │   ├── trade/
    │   ├── hypothesis/
    │   ├── condition/
    │   ├── deviation/
    │   ├── journal/
    │   ├── insight/
    │   ├── principle/
    │   ├── badge/
    │   ├── prediction/
    │   ├── emergency/
    │   ├── practice/
    │   ├── retrospective/
    │   ├── report/
    │   └── notification/
    │
    └── shared/
        ├── api/
        │   ├── client.ts                # fetch 래퍼
        │   ├── envelope.ts              # 봉투 언랩
        │   ├── errors.ts                # ApiError 정규화
        │   ├── query-keys.ts            # 쿼리 키 팩토리
        │   ├── query-client.ts
        │   ├── pagination.ts
        │   ├── idempotency.ts
        │   └── types.ts                 # 공통 응답 타입
        ├── ui/                          # 디자인 시스템 프리미티브
        │   ├── button/
        │   ├── chip/
        │   ├── input/
        │   ├── card/
        │   ├── sheet/
        │   ├── modal/
        │   ├── toast/
        │   ├── tabs/
        │   ├── list-item/
        │   ├── empty-state/
        │   ├── error-state/
        │   ├── skeleton/
        │   ├── notice-banner/
        │   ├── step-indicator/
        │   ├── infinite-list/
        │   └── index.ts
        ├── metric/                      # 수치 표현 (§15)
        │   ├── MetricValue.tsx
        │   ├── RateMetricValue.tsx
        │   ├── GapValue.tsx
        │   ├── decimal.ts
        │   └── format.ts
        ├── recall/
        │   └── RecallCard.tsx
        ├── chart/
        │   ├── LineChartBase.tsx
        │   ├── BarChartBase.tsx
        │   └── ChartDataTable.tsx       # 접근성 대체 표
        ├── lib/
        │   ├── cn.ts                    # clsx + tailwind-merge
        │   ├── date.ts
        │   ├── storage.ts
        │   ├── device.ts
        │   └── assert.ts
        ├── hooks/
        │   ├── useMediaQuery.ts
        │   ├── useDisclosure.ts
        │   ├── useDebouncedValue.ts
        │   ├── useIntersection.ts
        │   └── useDraftPersist.ts       # 시트 입력 보존
        ├── config/
        │   ├── env.ts
        │   ├── constants.ts
        │   └── labels.ts                # 열거값 → 한국어 라벨
        └── types/
            └── global.d.ts
```

---

## 5. TypeScript 제약과 코딩 규약

기존 `tsconfig.app.json`의 설정이 코드 작성 방식을 직접 규정한다. 아래는 그 설정이 강제하는 규약이다.

| 설정 | 강제되는 작성 방식 |
| --- | --- |
| `verbatimModuleSyntax` | 타입만 가져올 때 `import type { X } from '...'`. 혼합 import 금지 |
| `erasableSyntaxOnly` | **`enum` 사용 불가.** `as const` 객체 + 유니온 타입으로 대체. 네임스페이스·생성자 파라미터 프로퍼티도 금지 |
| `allowImportingTsExtensions` | import 경로에 확장자 명시 (`./App.tsx`, `./format.ts`) |
| `noUnusedLocals` / `noUnusedParameters` | 미사용 바인딩은 빌드 실패. 의도적 미사용은 `_` 접두 |
| `strict` | `any` 금지. 불가피하면 `unknown` + 좁히기 |

### 5.1 열거값 표현

`enum`을 쓸 수 없으므로 API의 열거값은 아래 형태로 정의한다.

| 요소 | 위치 |
| --- | --- |
| 값 객체 | `entities/<slice>/model/constants.ts`의 `as const` 객체 |
| 타입 | 값 객체로부터 파생한 유니온 |
| 라벨 매핑 | `shared/config/labels.ts` 또는 슬라이스 `config/` |

**라벨을 값과 분리하는 이유**: PRD 5.5의 용어 규칙이 바뀌면 라벨 파일 하나만 고치면 되고, 값(서버 계약)은 건드리지 않는다.

### 5.2 명명 규약

| 대상 | 규약 | 예 |
| --- | --- | --- |
| 컴포넌트 파일 | PascalCase.tsx | `AccountCard.tsx` |
| 훅 파일 | camelCase.ts, `use` 접두 | `useCurrentCycle.ts` |
| 타입 파일 | `types.ts` 또는 `model/types.ts` | |
| 쿼리 훅 | `use<Entity><Action>Query` | `useAccountsQuery` |
| 뮤테이션 훅 | `use<Action><Entity>Mutation` | `useCreateTradeMutation` |
| 타입 이름 | 도메인 개념 그대로 | `HypothesisCondition` |
| 불리언 | `is`/`has`/`can` 접두 | `isDisplayable` |

### 5.3 절대 경로

`@/` 별칭으로 `src/`를 가리킨다. 상대 경로는 같은 슬라이스 내부에서만 사용한다.

---

## 6. 라우팅

### 6.1 라우트 구조

| 경로 | 레이아웃 | 가드 | 페이지 |
| --- | --- | --- | --- |
| `/` | Auth | 비로그인 | 랜딩 |
| `/auth/callback` | Auth | — | 카카오 콜백 |
| `/onboarding/*` | Onboarding | 로그인, 온보딩 미완료 | 최초 8단계 |
| `/cycle-start/*` | Onboarding | 온보딩 완료, 활성 사이클 없음 | **두 번째 사이클 이후 시작 5단계** |
| `/home` | App | 로그인, 온보딩 완료 | 홈 |
| `/stocks` | App | 동일 | 종목 풀 |
| `/stocks/:stockId` | App | 동일 | 종목 상세 |
| `/trades` | App | 동일 | 매매 내역 |
| `/trades/new` | App(시트) | 동일 | 매매 기입 |
| `/conditions` | App | 동일 | 조건 현황 |
| `/conditions/triggers/:triggerId` | App | 동일 | 조건 도달 응답 |
| `/journal` | App | 동일 | 일지 달력 |
| `/journal/write/:type/:date` | App | 동일 | 일지 작성 |
| `/records/insights` | App | 동일 | 주제별 모아보기 |
| `/records/principles` | App | 동일 | 수칙 현황 |
| `/records/reports` | App | 동일 | 지난 레슨 |
| `/performance` | App | 동일 | 성과 상세 |
| `/rebalances/:rebalanceId` | App | 동일 | 조정일 상세 |
| `/practice` | Fullscreen | 로그인 | 연습 모드 |
| `/settlement` | Fullscreen | 결산 기간 | 결산 개요·최종 성과 |
| `/settlement/retrospective` | Fullscreen | 결산 기간 | **사이클 회고 (리포트보다 먼저)** |
| `/settlement/report` | Fullscreen | 리포트 생성 완료 | 올해의 투자 레슨 |
| `/settlement/principles` | Fullscreen | 리포트 열람 후 | 수칙 확정 |
| `/badges` | App | 동일 | 배지 |
| `/notifications` | App | 동일 | 알림 |
| `/settings/*` | App | 동일 | 설정·내보내기·탈퇴 |
| `*` | — | — | 404 |

### 6.2 가드

| 가드 | 판정 기준 | 리다이렉트 |
| --- | --- | --- |
| `RequireAuth` | 액세스 토큰 유효 | `/` |
| `RequireOnboarded` | `onboarding.is_completed` | `/onboarding/{current_step}` |
| `RequireActiveCycle` | `cycle.status === 'ACTIVE'` | 온보딩 미완료면 `/onboarding/...`, 완료했으나 사이클이 없으면 `/cycle-start/...`, 공백기면 안내 화면 |
| `RequireSettlementWindow` | 결산 상태 | `/home` |

> **온보딩 완료 여부와 활성 사이클 존재 여부는 별개 축이다.** 2년차 사용자는 온보딩을 마쳤지만 1월에는 활성 사이클이 없다. 두 조건을 하나로 합치면 재방문자가 온보딩으로 되돌려 보내진다.

가드는 `GET /auth/session` 응답 하나로 판정한다. 여러 엔드포인트를 조합하면 화면 간 판단이 어긋난다 (API 명세 §8.4).

### 6.3 진입점 결정

앱 부팅 시 `pending_actions`를 확인해 미응답 조건 도달·미확인 조정일·결산 대기가 있으면 홈에 배너로 노출한다. **강제 리다이렉트하지 않는다** — 사용자가 무엇을 볼지 서비스가 정하면 그것도 일종의 지시가 된다.

### 6.4 코드 스플리팅

라우트 단위 `lazy()` 분할. 특히 아래는 별도 청크로 강제한다.

| 청크 | 이유 |
| --- | --- |
| 결산·리포트 | 연 1회만 사용. 평소 로드 불필요 |
| 연습 모드 | 차트 재생 로직 포함 |
| 차트 라이브러리 | Recharts를 성과·종목 상세에서만 로드 |
| 온보딩 | 최초 1회 |

---

## 7. API 클라이언트 계층

### 7.1 계층 구성

```
shared/api/client.ts        fetch 래퍼 — 헤더, 타임아웃, 재시도
    ↓
shared/api/envelope.ts      봉투 언랩 — data 추출, meta 보존
    ↓
shared/api/errors.ts        오류 정규화 — ApiError 인스턴스
    ↓
entities/<slice>/api/*.ts   엔드포인트별 함수 + 쿼리 훅
```

### 7.2 클라이언트 책임

| 항목 | 처리 |
| --- | --- |
| 베이스 URL | `shared/config/env.ts` |
| 인증 헤더 | 메모리의 액세스 토큰 자동 부착 |
| 토큰 갱신 | 401 수신 시 `/auth/refresh` 1회 시도 → 실패 시 로그아웃. **동시 401은 단일 갱신으로 합류** |
| 멱등성 키 | POST 요청에 UUID 자동 생성·부착 |
| 타임아웃 | 15초 |
| 재시도 | GET만, 네트워크 오류·5xx에 2회 (지수 백오프) |
| 요청 ID 로깅 | 응답 `meta.request_id`를 오류 보고에 포함 |

**액세스 토큰을 localStorage에 저장하지 않는다.** 메모리 보관 + HttpOnly 쿠키 리프레시 조합으로 XSS 노출면을 줄인다. 새로고침 시 `/auth/refresh`로 복원한다.

### 7.3 봉투 언랩

모든 응답이 동일 구조이므로(API 명세 §2) 언랩은 한 곳에서 처리한다.

| 반환 | 내용 |
| --- | --- |
| `data` | 리소스 본문 |
| `meta` | `as_of_date`, `notice`, `pagination`, `request_id` |

**`meta`를 버리지 않는 것이 중요하다.** `as_of_date`와 `notice`는 화면 요구사항이며(PRD 7.9, 17장), 언랩 단계에서 사라지면 각 화면이 다시 조회해야 한다. 쿼리 훅은 `{ data, meta }` 형태를 그대로 반환한다.

### 7.4 오류 정규화

| 클래스 | 내용 |
| --- | --- |
| `ApiError` | `code`, `message`, `detail`, `fieldErrors`, `status`, `retryable`, `requestId` |
| `NetworkError` | 연결 실패·타임아웃 |
| `AuthError` | 401 계열 |

**`message`는 서버 값을 그대로 보존한다.** 클라이언트가 문구를 만들지 않는다는 원칙(PRD 4.1)의 구현 지점이다.

### 7.5 타입 생성

서버 스키마와의 정합성은 **수동 타입 정의 + 런타임 검증**으로 유지한다.

| 항목 | 방침 |
| --- | --- |
| 응답 타입 | `entities/<slice>/model/types.ts`에 수동 정의 |
| 런타임 검증 | 개발 모드에서만 Zod 파싱. 운영에서는 생략(성능) |
| 불일치 감지 | 개발 중 콘솔 경고 + 테스트 실패 |

OpenAPI 자동 생성을 쓰지 않는 이유는 Chalice가 스키마를 자동 노출하지 않고, 수동 유지 비용이 자동화 도입 비용보다 낮은 규모이기 때문이다. 단 §22의 계약 테스트로 불일치를 잡는다.

---

## 8. 서버 상태 관리

### 8.1 쿼리 키 규약

`shared/api/query-keys.ts`에서 **팩토리 함수로만** 생성한다. 문자열 배열을 각 파일에서 직접 만들면 무효화가 어긋난다.

| 계층 | 형태 |
| --- | --- |
| 도메인 루트 | `['cycle']` |
| 리스트 | `['cycle', 'list', filters]` |
| 단건 | `['cycle', 'detail', cycleId]` |
| 하위 리소스 | `['cycle', 'detail', cycleId, 'accounts']` |

### 8.2 캐시 정책

| 데이터 | staleTime | gcTime | 근거 |
| --- | --- | --- | --- |
| 세션·사용자 | 5분 | 30분 | |
| 종목 풀 | 1시간 | 24시간 | 연 1회 변경 |
| 조건 카탈로그·태그·배지 정의 | 1시간 | 24시간 | 사실상 정적 |
| 시세·시장 요약 | 5분 | 30분 | 일 1회 갱신 |
| 계좌·성과 | 1분 | 10분 | 배치 후 변경 |
| 매매·일지 목록 | 30초 | 10분 | |
| 조건 도달 | 0 (항상 최신) | 5분 | 시의성 |
| 알림 | 0 | 5분 | |

### 8.3 무효화 매핑

뮤테이션 성공 시 무효화할 키를 **엔티티 API 파일에 함께 정의**한다. 호출부가 기억하는 방식은 반드시 누락된다.

| 뮤테이션 | 무효화 대상 |
| --- | --- |
| 매매 기입 | 계좌, 성과, 보유, 매매 목록, 조건 목록, 사이클 |
| 매매 정정·삭제 | 위 전부 + 항목별 내역 |
| 조건 재설정 | 조건 목록, 가설, 계좌(PLAN) |
| 조건 응답 | 트리거 목록, 준수율, 항목별 내역 |
| 회고 제출 | 매매 상세, 깨달은 것 |
| 일지 작성 | 일지 목록, 연속일, 깨달은 것 |
| 별표 표시 | 깨달은 것, 태그 개수 |
| 수칙 확정 | 수칙 목록, 사이클 |
| 조정 확인 | 조정 목록, 확인율, 성과 |
| 비상 선언 | 선언 목록, 사이클 |
| 회고 제출 | 회고, 결산 상태 |
| 사이클 생성 | 사이클, 계좌, 성과, 종목 선택, 세션 |

### 8.4 재계산 대기 처리

매매 정정 시 서버가 재계산 작업을 등록하고 `recompute_job_id`를 반환한다 (API 명세 §15.4). 클라이언트는 아래 흐름을 따른다.

| 단계 | 처리 |
| --- | --- |
| 1 | 정정 성공 → 계좌·성과 쿼리 무효화 |
| 2 | 계좌 조회 응답의 `meta.recompute_pending`이 true면 인라인 배너 표시 |
| 3 | 3초 간격 폴링 (최대 2분) |
| 4 | 폴링 대상은 `GET /recompute-jobs/{job_id}` (API 명세 §13.5). 계좌 조회를 반복하지 않는다 |
| 5 | 완료 시 폴링 중단, 배너 제거, 계좌·성과 재조회 |
| 6 | 타임아웃(2분) 시 수동 새로고침 안내. `FAILED`면 재시도 버튼 |

**낙관적 갱신을 하지 않는다.** 계좌 수치는 이 제품의 핵심 산출물이며, 잠깐이라도 틀린 값을 보여주면 안 된다.

### 8.5 낙관적 갱신 허용 범위

| 대상 | 허용 |
| --- | --- |
| 별표 표시/해제 | 허용 |
| 수칙 지킴 표시 | 허용 |
| 알림 읽음 | 허용 |
| 일지 항목 자동 저장 | 허용 |
| **매매·조건·계좌 관련 전부** | **금지** |
| **결산·수칙 확정** | **금지** |

### 8.6 무한 스크롤

`useInfiniteQuery` + 서버 커서. `shared/ui/infinite-list`가 관찰자·재시도·종료 표시를 공통 처리한다.

---

## 9. 클라이언트 상태 관리

전역 클라이언트 상태는 아래로 제한한다. 그 외는 전부 서버 상태이거나 지역 상태다.

| 스토어 | 내용 |
| --- | --- |
| `authStore` | 액세스 토큰(메모리), 인증 상태 |
| `themeStore` | 라이트/다크, 손익 색상 반전. **서버 설정(`GET /me/settings`)이 원본이고 스토어는 즉시 반영용 캐시다.** 변경 시 `PATCH /me/settings`로 동기화하며, 응답 실패 시 이전 값으로 되돌린다. 기기마다 손익 색이 다르면 같은 수치가 반대 의미로 읽힌다 |
| `toastStore` | 토스트 큐 |
| `sheetStore` | 열린 시트 스택 (중첩 시트 관리) |
| `draftStore` | 폼 임시 저장 (§14.3) |

**API 응답을 스토어에 복사하지 않는다.** 복사본이 생기면 무효화 시점에 두 값이 어긋나고, 어느 쪽이 진실인지 알 수 없게 된다.

---

## 10. 엔티티 계층

각 엔티티는 **타입 + API + 표현 컴포넌트**를 갖는다. 행동(뮤테이션 트리거, 폼)은 features에 둔다.

### 10.1 엔티티 목록과 책임

| 엔티티 | 타입 | 주요 컴포넌트 |
| --- | --- | --- |
| `user` | User, Settings, Survey | ProfileSummary |
| `cycle` | Cycle, Schedule, PlanPreview | CycleScheduleList, CycleStatusBadge |
| `account` | Account, Position, PerformanceSeries, Gap | AccountCard, AccountLegend, PositionRow, PerformanceChart |
| `stock` | Stock, Price, Disclosure | StockCard, StockPriceChart, DisclosureItem |
| `trade` | Trade, TradeRevision | TradeRow, TradeDetailCard |
| `hypothesis` | Hypothesis, Condition | HypothesisCard, ConditionSentence, ConditionActionPicker |
| `condition` | ConditionTrigger | TriggerCard, TriggerResponseOptions |
| `deviation` | DeviationItem | DeviationPairRow |
| `journal` | JournalEntry, JournalItem, Streak | JournalCard, StampCalendar |
| `insight` | Insight, InsightTag | InsightItem, TagFilter |
| `principle` | Principle, Encounter, Scorecard | PrincipleCard, PrincipleScorecard |
| `badge` | Badge | BadgeTile |
| `prediction` | Prediction | PredictionCard |
| `emergency` | Declaration | DeclarationCard |
| `practice` | Scenario, Session, Result | PracticeChart, PracticeResultPanel |
| `retrospective` | RetrospectiveQuestion, Answer | RetrospectiveQuestionItem |
| `report` | CycleReport, ReportSection | ReportSection 렌더러들 |
| `notification` | Notification | NotificationItem |

### 10.2 엔티티 컴포넌트 규칙

| 규칙 | 내용 |
| --- | --- |
| 순수 표현 | props로 데이터를 받고, 자체 쿼리를 하지 않는다 |
| 행동 없음 | 뮤테이션을 호출하지 않는다. 콜백 prop으로 위임 |
| 예외 | 목록 스크롤 등 표현 관련 지역 상태는 허용 |

### 10.3 계좌 엔티티 상세

4개 계좌를 다루는 방식이 이 앱의 중심이다.

| 항목 | 설계 |
| --- | --- |
| 계좌 타입 상수 | `ACCOUNT_TYPES = ['RULE','FREE','PLAN','HOLD'] as const` |
| 표시 순서 | 상수 배열 순서 고정. 성과순 정렬 금지 |
| 라벨·색 매핑 | `entities/account/config/account-meta.ts` 단일 소스 |
| 짝 관계 | API `pairs` 응답 사용. 클라이언트가 하드코딩하지 않는다 |
| `is_identical_to_hold` | 카드에 캡션 표시 (PRD 18.6) |

**성과순 정렬을 금지하는 이유**: 정렬하면 1등이 생기고, 그 순간 서비스가 "이 계좌가 낫다"고 말하게 된다. 계좌는 비교 대상이지 경쟁 대상이 아니다.

---

## 11. 피처 계층

각 피처는 **하나의 사용자 행동**을 담당하며, 폼·뮤테이션·성공 처리를 포함한다.

### 11.1 주요 피처 상세

#### `trade-entry` — 매매 기입 + 가설 기록

가장 복잡한 피처이며 PRD 9.3~9.6이 집중된다.

| 구성 | 내용 |
| --- | --- |
| 진입 | `GET /trades/entry-context` 1회 호출로 필요한 전부 확보 |
| 화면 구성 | ① 수칙·과거 기록 노출 ② 매매 정보 입력 ③ 가설 기록 ④ 확신도·기분 |
| 필수 | 되돌아볼 조건 + 그때 할 일만 |
| 제출 | 매매와 가설을 단일 요청으로 전송 |
| 성공 후 | 매도 응답의 `next_action`에 따라 회고 또는 새 조건 시트로 연결 |
| 임시 저장 | 시트를 닫아도 입력 보존 |

**과거 기록 패널을 접힘 상태로 두지 않는다.** PRD 9.5는 이 노출이 "이 제품이 사용자의 행동을 실제로 바꾸는 유일한 지점"이라고 규정한다. 접혀 있으면 아무도 열지 않는다.

#### `condition-respond` — 조건 도달 응답

| 구성 | 내용 |
| --- | --- |
| 표시 | RecallCard(그때 정한 것) + 현재 가격 + 관련 수칙 |
| 선택지 | 3가지를 동일 시각 무게로 (PRD 9.8) |
| 제출 후 | **평가 문구 없음.** "기록했습니다" 토스트만 |
| 3번 선택 시 | 새 조건 입력 폼으로 연결 |

#### `journal-write` — 일지 작성

| 구성 | 내용 |
| --- | --- |
| 유형 | 일간·주간·월간 세그먼트 |
| 항목 | 최대 3개. **1개만 써도 저장 가능** |
| 자동 저장 | 항목 단위 1초 디바운스 |
| 별표 | 항목별 토글 + 태그 시트 (한 번의 탭) |
| 월간 | `GET /market/monthly-context` 수치를 상단에 표시 |
| 소급 | `writable-dates`로 선택 가능 날짜 제한 |

#### `rebalance-confirm` — 이대로 하겠습니다

| 구성 | 내용 |
| --- | --- |
| 표시 | 정리 종목 2개 + 선정 근거 + 재분배 + 예시 규칙 고지 |
| 버튼 | "이대로 하겠습니다" 단일 |
| 성공 후 | 확인율 갱신 + "계좌를 움직이지 않습니다" 안내 |
| 미확인 | 부정적 표시 없음. 기한 안내만 |

#### `principle-create` — 수칙 확정

| 구성 | 내용 |
| --- | --- |
| 후보 | 깨달은 것 목록 + 문장 틀 후보 (대칭 쌍으로 표시) |
| 편집 | 후보 선택 후 문장 수정 가능 |
| 개수 | 최대 5개. 초과 시 비활성 + 안내 |
| 노출 시점 | 수칙별 선택형 지정 |
| 직접 작성 | 항상 가능 |

#### `practice-play` — 연습 모드

| 구성 | 내용 |
| --- | --- |
| 1단계 | 가려진 차트 표시 |
| 2단계 | 목표·조건·대응 선언 (**미완료 시 진행 불가**) |
| 3단계 | 하루씩 재생. '그대로 둔다'/'판다' 2택 |
| 4단계 | 선언대로 vs 실제 대조 |
| 5단계 | 종목·시기 공개 + 깨달은 것 한 줄 |
| 공유 | 준수율만 담긴 이미지 (수익률 제외) |

### 11.2 피처 공통 규칙

| 규칙 | 내용 |
| --- | --- |
| 뮤테이션 소유 | 피처가 뮤테이션 훅을 소유하고, 무효화를 담당 |
| 성공 피드백 | 토스트 1개. 축하 연출 없음 |
| 실패 처리 | `field_errors`는 폼에, 그 외는 토스트 |
| 로딩 | 제출 버튼 비활성 + 스피너 |
| 중복 제출 | 멱등성 키 + 버튼 비활성 |

---

## 12. 위젯과 페이지

### 12.1 위젯의 역할

위젯은 **자기 데이터를 스스로 가져오는 독립 블록**이다. 페이지는 위젯을 배치할 뿐 데이터를 내려주지 않는다.

| 이유 | 설명 |
| --- | --- |
| 페이지 단순화 | 홈 화면이 6개 쿼리를 조합하는 코드가 되지 않는다 |
| 독립 로딩 | 위젯별 스켈레톤·오류 경계로 한 블록 실패가 화면 전체를 막지 않는다 |
| 재사용 | 같은 위젯이 홈과 성과 페이지에 동시에 놓인다 |

### 12.2 주요 위젯

| 위젯 | 데이터 | 사용 페이지 |
| --- | --- | --- |
| `account-overview` | 4계좌 요약 | 홈, 성과 |
| `performance-chart-panel` | 성과 곡선 + 범례 + 기준일 | 홈, 성과 |
| `gap-summary` | 세 개의 차이 | 성과, 결산 |
| `deviation-breakdown` | 항목별 내역 | 성과, 결산 |
| `cycle-schedule` | 남은 일정 | 홈, 온보딩 6단계 |
| `market-summary` | 시장 수치 | 홈, 월간 일지 |
| `recall-panel` | 과거 기록 | 매매 기입, 조건 응답, 회고 |
| `stock-record-timeline` | 종목별 내 기록 | 종목 상세 |
| `report-viewer` | 레슨 7개 섹션 | 결산, 지난 레슨 |
| `notice-stack` | `meta.notice` | 전역 (레이아웃 내) |

### 12.3 오류 경계

위젯 단위로 오류 경계를 둔다. 시장 요약이 실패해도 계좌 요약은 보여야 한다.

### 12.4 페이지의 역할

| 하는 것 | 하지 않는 것 |
| --- | --- |
| 위젯 배치와 레이아웃 | 데이터 조회 |
| 라우트 파라미터 → 위젯 prop | 비즈니스 로직 |
| 페이지 제목·메타 | 폼 처리 |

---

## 13. 화면 인벤토리

PRD 16장의 사용자 여정을 화면으로 매핑한다.

### 13.1 온보딩 (PRD 14)

| 단계 | 화면 | 핵심 |
| --- | --- | --- |
| 1 | 랜딩 | 한 문장 가치 제안 |
| 2 | 카카오 로그인 | |
| 3 | 성향 설문 | 3문항, 칩 선택 |
| 4 | 연습 모드 1판 | 전체 화면. 3분 |
| 5 | 임시 수칙 1개 | 연습 결과 기반 후보 |
| 6 | 일정 확인 | **○개월, 조정 ○회 명시** |
| 7 | 종목 선택 | 30개 중 N개. 성향별 정렬 |
| 8 | 첫 가설 기록 | 5단계에서 만든 수칙 노출 |

7~8단계는 한 흐름으로 처리하고, 이탈 시 "종목 선택까지 저장되었습니다" 안내를 보여준다.

### 13.1a 사이클 시작 — 두 번째 사이클 이후 (PRD 16-B)

| 단계 | 화면 | 핵심 |
| --- | --- | --- |
| 1 | 수칙 확인 | 지난 결산에서 확정한 3~5개 |
| 2 | 일정 확인 | ○개월, 조정 ○회 |
| 3 | 종목 선택 | 30개 중 N개 + **선택 이유(선택 입력)** |
| 4 | 첫 가설 기록 | 확정 수칙과 이 종목의 과거 기록 노출 |
| 5 | 이번 판 예측 | 규칙 vs 자유, 계획 준수 예측 |

**온보딩의 설문·연습·임시 수칙 단계를 반복하지 않는다.** 데이터는 `GET /cycles/start-context` 한 번으로 확보하며, 3~4단계는 온보딩과 마찬가지로 한 덩어리다.

### 13.2 홈

| 블록 | 내용 |
| --- | --- |
| 1 | 시장 수치 + 기준일 |
| 2 | 4계좌 2×2 카드 |
| 3 | 성과 곡선 (요약) |
| 4 | 남은 기간·다음 조정일 |
| 5 | 대기 중 액션 배너 (조건 도달, 조정일, 결산) |
| 6 | 오늘 일지 쓰기 (한 번의 탭) |
| 7 | 연습 모드 한 판 |
| 8 | 보유 종목 요약 |

### 13.3 종목 상세

| 블록 | 내용 |
| --- | --- |
| 1 | 시세 차트 + **시가·종가** + 기준일 |
| 2 | 공시 목록 (제목 + 원문 링크) |
| 3 | 실적 발표 일정 (날짜만) |
| 4 | **이 종목에 대해 내가 쓴 기록** (지난 사이클 포함) |
| 5 | 현재 보유·조건 상태 |
| 6 | 매매 기입 진입 |

### 13.4 성과

| 블록 | 내용 |
| --- | --- |
| 1 | 4계좌 곡선 (기간 선택) |
| 2 | 세 개의 차이 |
| 3 | 항목별 내역 (대칭 배치) |
| 4 | 두 준수율 (각각 분모와 함께) |
| 5 | 측정 편향 안내 |
| 6 | 거래비용 기준 보기 |

### 13.5 결산

| 순서 | 화면 | 라우트 |
| --- | --- | --- |
| 1 | 결산 완료 안내 (자동 청산 결과) | `/settlement` |
| 2 | 4계좌 최종 성과 + 세 개의 차이 | `/settlement` |
| 3 | **사이클 회고** (선택형 위주) | `/settlement/retrospective` |
| 4 | 올해의 투자 레슨 (7개 섹션) | `/settlement/report` |
| 5 | 잘한/아쉬운 판단 선택 | `/settlement/report` |
| 6 | 투자 수칙 확정 | `/settlement/principles` |
| 7 | 다음 사이클 안내 + 내보내기 권유 | `/settlement/principles` |

**3번을 4번보다 먼저 두는 순서를 UI에서 지킨다.** 리포트의 숫자를 본 뒤에 쓴 회고는 그 숫자에 맞춰 기억이 재구성된 것이다 (PRD 1.2). 다만 **회고를 건너뛰어도 리포트로 진행할 수 있다** — 강제하면 결산이 막히고, 결산이 막히면 다음 사이클도 막힌다 (PRD 8.6). 회고 화면에는 "나중에" 버튼을 두되, 리포트 화면에서 되돌아오는 링크는 제공하지 않는다.

7번에 내보내기를 배치하는 것은 PRD 18.12의 "결산 직후가 내보내기를 권할 최적의 시점"에 따른 것이다.

---

## 14. 폼 처리

### 14.1 구성

| 항목 | 방침 |
| --- | --- |
| 라이브러리 | React Hook Form + `@hookform/resolvers/zod` |
| 스키마 위치 | 피처 슬라이스의 `model/schema.ts` |
| 서버 오류 매핑 | `field_errors[].field` → `setError` |
| 제출 | `handleSubmit` + 뮤테이션 |

### 14.2 검증 시점

| 시점 | 규칙 |
| --- | --- |
| 입력 중 | 검증하지 않음 (`mode: 'onBlur'`) |
| 이탈 시 | 형식 검증 |
| 제출 시 | 전체 검증 + 첫 오류로 스크롤·포커스 |

### 14.3 임시 저장

`useDraftPersist` 훅이 폼 상태를 세션 스토리지에 보관한다.

| 대상 | 보존 |
| --- | --- |
| 매매 기입 + 가설 | 세션 |
| 일지 | 서버 자동 저장 (임시 저장 불필요) |
| 회고 | 서버 자동 저장 |
| 수칙 확정 | 세션 |
| 연습 모드 선언 | 서버 세션 |

**시트를 닫아도 값이 남는다.** PRD 4.3의 "빈칸 허용, 나중에 채울 수 있어야"는 앱 이탈 후 재진입까지 포함하는 요구다.

### 14.4 조건 입력 폼

가장 주의가 필요한 폼이다.

| 규칙 | 내용 |
| --- | --- |
| 기본값 | **없다.** `defaultValues`에 `planned_action`을 넣지 않는다 |
| 선택지 순서 | 고정 |
| 필수 표시 | `(필수)` 텍스트 |
| 미선택 제출 | 필드 오류 + 포커스 |
| 조건 유형 | 카탈로그 칩 우선, "직접 쓰기"는 마지막 |
| 자기 보고 안내 | `evaluation_mode=SELF_REPORT` 선택 시 "도달했을 때 직접 표시해야 반영됩니다" 캡션 |

마지막 항목이 PRD 18.11의 측정 편향을 입력 시점에 완화하는 장치다.

---

## 15. 수치 표현 계층

`shared/metric/`이 디자인 시스템 §9의 표기 규칙을 코드로 강제한다.

### 15.1 Decimal 처리

| 항목 | 처리 |
| --- | --- |
| 수신 | 서버가 문자열로 전송 |
| 파싱 | `decimal.js-light`로 변환 |
| 연산 | **하지 않는다.** 표시 형식 변환만 |
| 예외 | 퍼센트 변환(×100), 부호 판정 |

**클라이언트에서 계좌 값을 계산하지 않는다.** 차이·수익률·비율은 전부 서버 계산 결과다. 클라이언트가 계산하면 서버와 값이 달라지고, 어느 쪽이 맞는지 사용자가 알 수 없게 된다.

### 15.2 컴포넌트 계약

| 컴포넌트 | 필수 prop | 강제 효과 |
| --- | --- | --- |
| `MetricValue` | `asOfDate` | 기준일 없는 수치를 렌더링할 수 없다 |
| `MetricValue` (사이클 성과) | `periodMonths` | 사이클 길이 없는 성과를 표시할 수 없다 |
| `RateMetricValue` | `metric` 객체 전체 | 분모 없는 비율을 만들 수 없다 |
| `GapValue` | `gap` 객체 전체 | 라벨·차이 종류가 항상 함께 |
| `DeviationPairRow` | `item`, `counterpart` | 짝이 정의된 항목을 한쪽만 그릴 수 없다 |
| `DeviationRow` | `item` | 단독 항목 전용. 짝을 요구하지 않는다 |

`deviation-breakdown` 위젯은 서버가 준 `render_group`에 따라 두 컴포넌트를 분기한다. **클라이언트가 어느 항목에 짝이 있는지 판단하지 않는다** — 대칭 관계는 서버의 시드 데이터에 정의되어 있다.

**타입 시스템으로 PRD 규칙을 강제하는 것이 이 계층의 목적이다.** 문서로만 존재하는 규칙은 화면이 늘어나면 반드시 새어 나간다.

### 15.3 표시 불가 처리

`is_displayable === false`인 지표는 수치를 회색으로 표시하고 `insufficient_reason`에 대응하는 문구를 붙인다. **해석 문장은 서버가 아예 내려보내지 않으므로** 클라이언트가 숨길 것이 없다.

| reason | 문구 |
| --- | --- |
| `SAMPLE_TOO_SMALL` | "기록이 n건 미만이라 아직 판단하기 이릅니다" |
| `NO_DENOMINATOR` | "이번 사이클에는 정기 조정이 없었습니다" |
| `CYCLE_TOO_SHORT` | "이번 사이클은 ○개월이라 비교가 이릅니다" |

---

## 16. 오류·로딩 처리

### 16.1 오류 계층

| 계층 | 처리 |
| --- | --- |
| 전역 오류 경계 | 렌더 오류 → 전체 화면 ErrorState + 새로고침 |
| 위젯 오류 경계 | 블록 단위 ErrorState + 재시도 |
| 쿼리 오류 | `error` 상태 → 위젯 내 처리 |
| 뮤테이션 오류 | 폼 필드 또는 토스트 |
| 인증 오류 | 인터셉터에서 갱신·로그아웃 |

### 16.2 오류 코드별 UI 매핑

`shared/api/error-ui.ts`가 코드 → UI 유형 매핑을 관리한다.

| 유형 | 코드 예 |
| --- | --- |
| 인라인 필드 | `VALIDATION_FAILED`, `TRADE_INSUFFICIENT_CASH` |
| 토스트 | `TRADE_MARKET_CLOSED`, `JOURNAL_BACKFILL_EXPIRED` |
| 모달 안내 | `CYCLE_RESTART_WINDOW_CLOSED`, `EMERGENCY_LIMIT_EXCEEDED` |
| 전체 화면 | `RESOURCE_NOT_FOUND`, `FORBIDDEN`, `INTERNAL_ERROR` |
| 배너 | `SERVICE_MAINTENANCE` |

**문구는 매핑하지 않는다.** 서버 `message`를 그대로 표시하고, 클라이언트는 표시 형태만 정한다.

### 16.3 로딩

| 상황 | 처리 |
| --- | --- |
| 라우트 전환 | 상단 진행 바 |
| 위젯 최초 | 스켈레톤 (실제 높이) |
| 백그라운드 갱신 | 표시하지 않음 |
| 뮤테이션 | 버튼 스피너 |
| 재계산 대기 | 인라인 배너 + 값 흐리게 |

---

## 17. 디자인 시스템 연결

### 17.1 토큰 사용

| 규칙 | 내용 |
| --- | --- |
| CSS 변수 | `app/styles/tokens.css`에 정의 |
| Tailwind 노출 | `tailwind.config.ts`에서 시맨틱 토큰만 색상으로 등록 |
| 기본 팔레트 | **비운다.** `text-gray-500` 같은 클래스가 불가능해진다 |
| 임의 값 | `[...]` 금지. 필요 시 토큰 추가 |

### 17.2 컴포넌트 변형

CVA로 variant를 정의하고, 컴포넌트는 `cn()`으로 클래스를 병합한다.

### 17.3 테마 전환

`ThemeProvider`가 `data-theme` 속성을 루트에 설정한다. 손익 색상 반전도 같은 방식으로 CSS 변수만 교체한다.

### 17.4 아이콘

`shared/ui/icon/Icon.tsx`가 스프라이트 `<use>`를 감싼다. `name` prop은 스프라이트에 존재하는 ID의 유니온 타입이며, 목록은 빌드 시 스프라이트에서 생성한다. 오타가 컴파일 오류가 된다.

---

## 18. PWA와 알림

### 18.1 PWA 구성

| 항목 | 설정 |
| --- | --- |
| 매니페스트 | 이름, 아이콘, `display: standalone`, 테마 색 |
| 서비스 워커 | `vite-plugin-pwa` (generateSW) |
| 캐시 | 앱 셸·폰트·아이콘 precache. API 응답은 캐시하지 않는다 |
| 업데이트 | 새 버전 감지 시 "새로고침" 배너 |
| 오프라인 | 오프라인 안내 화면만. 오프라인 쓰기는 지원하지 않는다 |

**API 응답을 서비스 워커에 캐시하지 않는다.** 오래된 계좌 수치가 기준일 표기 없이 보이면 PRD 7.9를 위반한다.

### 18.2 푸시 알림

| 단계 | 처리 |
| --- | --- |
| 권한 요청 | 온보딩 완료 직후. **앱 첫 진입 시 요청하지 않는다** |
| 구독 | Web Push 구독 → `POST /notifications/devices` |
| 수신 | 서비스 워커에서 표시 |
| 클릭 | 딥링크로 해당 화면 이동 |
| 거부 | 인앱 알림만 사용. 반복 요청하지 않는다 |

권한 요청 시점을 온보딩 이후로 미루는 것은, 사용자가 앱의 가치를 이해하기 전에 요청하면 거부율이 높고 재요청이 불가능하기 때문이다.

### 18.3 딥링크

| 알림 유형 | 이동 |
| --- | --- |
| 조건 도달 | `/conditions/triggers/:id` |
| 정기 조정일 | `/rebalances/:id` |
| 결산일 | `/settlement` |
| 주간·월간 회고 | `/journal/write/:type/:date` |
| 일지 리마인더 | `/journal/write/daily/:date` |

---

## 19. 성능

### 19.1 목표

| 지표 | 목표 |
| --- | --- |
| LCP (4G, 중급 모바일) | 2.5초 |
| INP | 200ms |
| CLS | 0.1 |
| 초기 JS (gzip) | 180KB |
| 라우트 청크 | 각 60KB |

### 19.2 번들 전략

| 항목 | 처리 |
| --- | --- |
| 라우트 분할 | `lazy()` |
| 차트 | 동적 import. 성과·종목 상세에서만 |
| 날짜 | date-fns 개별 함수 import |
| 폰트 | 서브셋 woff2 + preload |
| 아이콘 | 단일 스프라이트 |
| 분석 | `rollup-plugin-visualizer`를 CI에서 실행, 예산 초과 시 실패 |

### 19.3 렌더 최적화

| 대상 | 처리 |
| --- | --- |
| 긴 목록 | 100개 초과 시 가상 스크롤 |
| 차트 | 데이터 변경 시에만 리렌더 (`useMemo`) |
| 폼 | 비제어 기반으로 타이핑 리렌더 없음 |
| 컨텍스트 | 값 객체 메모이제이션. 스토어는 셀렉터 구독 |

### 19.4 이미지

빈 상태 일러스트는 SVG. 공유 이미지는 런타임 Canvas 생성.

---

## 20. 접근성 구현

| 항목 | 구현 |
| --- | --- |
| 시맨틱 | 랜드마크 요소 사용. `div` 버튼 금지 |
| 포커스 | `:focus-visible` 스타일 전역 정의 |
| 시트·모달 | 포커스 트랩, Esc 닫기, 열기 전 요소로 복귀 |
| 칩 그룹 | `role="radiogroup"` + 화살표 키 |
| 라이브 영역 | 토스트 `polite`, 치명 오류 `assertive` |
| 차트 | `ChartDataTable`을 `sr-only`로 병기 |
| 수치 | `aria-label`에 단위·기준일 포함 |
| 검사 | `eslint-plugin-jsx-a11y` + Playwright axe 스캔 |

---

## 21. 테스트 전략

### 21.1 계층

| 계층 | 대상 | 도구 | 비율 |
| --- | --- | --- | --- |
| 단위 | 포맷터, 유틸, 스키마 | Vitest | 25% |
| 컴포넌트 | 엔티티·프리미티브 | Vitest + Testing Library | 35% |
| 통합 | 피처 (폼 + 뮤테이션) | + MSW | 30% |
| E2E | 주요 여정 | Playwright | 10% |

### 21.2 필수 테스트

PRD 규칙이 UI에서 지켜지는지 검증하는 테스트를 **회귀 방지 자산**으로 관리한다.

| 테스트 | 검증 |
| --- | --- |
| 조건 대응 선택기에 기본값이 없다 | PRD 9.4 |
| 세 선택지의 시각 무게가 동일하다 (클래스 비교) | PRD 9.4 |
| `render_group="PAIR"` 항목이 짝과 함께 렌더링된다 | PRD 7.9 |
| 단독 항목에 인위적 반대말이 생성되지 않는다 | PRD 7.9 |
| 회고 화면이 리포트보다 먼저 노출되고, 건너뛸 수 있다 | PRD 13.1 |
| 온보딩 완료 사용자가 사이클 없을 때 `/cycle-start`로 간다 | PRD 16-B |
| 기준일 없이 `MetricValue`를 만들 수 없다 (타입 테스트) | PRD 7.9 |
| 비율이 분모와 함께 표시된다 | PRD 5.4 |
| 두 준수율이 별개로 표시된다 | PRD 5.4 |
| 계좌 목록이 성과순으로 정렬되지 않는다 | 원칙 |
| 조건 응답 후 평가 문구가 없다 | PRD 9.8 |
| 연속일이 끊겨도 부정 문구가 없다 | PRD 10.5 |
| 연습 모드 통계에 수익률이 없다 | PRD 12.4 |
| 종목 풀 화면에 선정 사유가 없다 | PRD 17.1 |
| 매매 완료 시 축하 연출이 없다 | PRD 4.2 |

### 21.3 E2E 시나리오

| 시나리오 | 범위 |
| --- | --- |
| 온보딩 완주 | 로그인 → 설문 → 연습 → 임시 수칙 → 일정 → 종목 선택 → 첫 가설 |
| 매매 기입 → 조건 도달 → 응답 | 핵심 루프 |
| 일지 작성 → 별표 → 주제별 모아보기 | 레슨 축적 |
| 결산 → 회고 → 레슨 → 수칙 확정 | 사이클 종료 |
| 두 번째 사이클 시작 (수칙 확인 → 종목 선택 → 가설 → 예측) | 재방문자 경로 |
| 매매 정정 → 재계산 대기 → 반영 | 파생 데이터 |

### 21.4 MSW 핸들러

API 명세를 기준으로 핸들러를 작성하고, **성공·오류·빈 상태·기준 미달**의 4가지 응답을 각 엔드포인트에 준비한다. 기준 미달 응답을 준비하지 않으면 `is_displayable=false` 경로가 한 번도 테스트되지 않는다.

---

## 22. 품질 게이트

CI에서 실행하며 하나라도 실패하면 병합을 막는다.

| # | 게이트 | 도구 |
| --- | --- | --- |
| 1 | 타입 검사 | `tsc -b` |
| 2 | 린트 | oxlint + ESLint(경계·a11y 규칙) |
| 3 | 포맷 | Prettier + tailwindcss 플러그인 |
| 4 | 레이어 의존 검증 | `eslint-plugin-boundaries` |
| 5 | 공개 API 위반 검사 | 슬라이스 내부 직접 import 금지 규칙 |
| 6 | **금지 용어 스캔** | 커스텀 스크립트 |
| 7 | 단위·컴포넌트 테스트 | Vitest |
| 8 | 번들 예산 | visualizer + 임계값 |
| 9 | 접근성 스캔 | Playwright + axe |
| 10 | 계약 테스트 | 응답 타입 vs MSW 픽스처 |

### 22.1 금지 용어 스캔

디자인 시스템 §17.3의 용어 사전을 스크립트로 검사한다. JSX 텍스트 노드와 라벨 상수 파일이 대상이다.

| 검출 대상 | 대체 |
| --- | --- |
| 유니버스, 추천 종목, 유망 | 종목 풀, 관찰 대상 종목 |
| 시즌 | 사이클 |
| 리밸런싱 | 정기 조정 |
| 포트폴리오 | 계좌, 보유 종목 |
| 손절, 스톱로스 | 되돌아볼 조건 |
| 지킨 비율(단독) | 내 계획 지킨 비율 / 규칙 실행 확인율 |
| 저널, 다이어리 | 투자 일지 |
| 인사이트, 리캡 | 깨달은 것, 올해의 투자 레슨 |

추가로 **권유형 어미 패턴**(`~하세요`, `~하시는 게`, `추천`, `유망`)을 경고로 검출한다.

### 22.2 예외 처리

불가피한 경우 `// allow-term: <사유>` 주석으로 예외 처리하되, 예외 목록은 리뷰 대상이다.

---

## 23. 빌드와 배포

### 23.1 명령

| 명령 | 내용 |
| --- | --- |
| `npm run dev` | Vite 개발 서버 |
| `npm run build` | `tsc -b` → `vite build` |
| `npm run preview` | 빌드 결과 확인 |
| `npm run lint` | oxlint |
| `npm run test` | Vitest (**신규 추가 필요**) |
| `npm run test:e2e` | Playwright (**신규 추가 필요**) |

> 현재 테스트 러너가 설치되어 있지 않다. Vitest·Testing Library·MSW·Playwright를 devDependency로 추가하고 위 스크립트를 등록해야 §21이 성립한다.

### 23.2 환경 변수

| 변수 | 용도 |
| --- | --- |
| `VITE_API_BASE_URL` | API 베이스 |
| `VITE_KAKAO_CLIENT_ID` | 카카오 JS 키 |
| `VITE_KAKAO_REDIRECT_URI` | |
| `VITE_VAPID_PUBLIC_KEY` | 웹 푸시 |
| `VITE_APP_VERSION` | 빌드 시 주입 |
| `VITE_SENTRY_DSN` | 오류 수집 (선택) |

`VITE_` 접두 변수는 번들에 포함되므로 **비밀 값을 넣지 않는다.**

### 23.3 개발 시 CORS

Vite 프록시로 `/api`를 로컬 Chalice(`127.0.0.1:8000`)에 전달한다. 운영은 CloudFront 동일 도메인 경로 라우팅 또는 API Gateway CORS 설정.

프록시를 쓰는 이유는 개발 환경에서 쿠키 기반 리프레시 토큰이 SameSite 제약 없이 동작하기 때문이다.

### 23.4 배포

| 항목 | 내용 |
| --- | --- |
| 산출물 | `dist/` 정적 파일 |
| 호스팅 | S3 + CloudFront |
| SPA 폴백 | 404 → `index.html` |
| 캐시 | 해시 자산 1년, `index.html` 무캐시 |
| 배포 후 | 캐시 무효화 |

### 23.5 오류 모니터링

| 항목 | 처리 |
| --- | --- |
| 수집 | 렌더 오류, 미처리 Promise |
| 컨텍스트 | 라우트, `request_id`, 앱 버전 |
| 제외 | **사용자가 입력한 텍스트는 전송하지 않는다.** 일지·회고·수칙은 개인의 판단과 감정 기록이다 |

---

## 24. 확장 대비

| PRD | 확장 | 준비 |
| --- | --- | --- |
| 19.1 | 복수 규칙 세트 | 계좌 타입 상수와 색 매핑이 단일 파일. 배열에 항목 추가로 확장. 차트·범례·카드가 자동 대응 |
| 19.2 | 수칙 확장 | 수칙 엔티티에 이력 필드 존재. 성적표 컴포넌트가 사이클별 데이터를 받도록 설계 |
| 19.3 | 실투자 전환 | 매매 목록이 `entry_source`를 이미 표시 가능 |
| 19.6 | 구독·결제 | 새 피처 슬라이스 추가. 기존 레이어 영향 없음 |
| 19.7 | 다년 비교 리포트 | 리포트 뷰어가 섹션 렌더러 레지스트리 구조. 섹션 추가만으로 확장 |
| 다국어 | i18n | 라벨이 `shared/config/labels.ts`와 슬라이스 `config/`에 집중되어 있어 추출 가능. **단 서버 문구(`message`, `StatementBlock`)는 서버 다국어화가 선행되어야 한다** |

**마지막 항목이 구조적 제약이다.** 이 앱은 문구 생성 책임을 서버에 두었으므로(원칙 1), 클라이언트만 다국어화할 수 없다. 이는 의도된 트레이드오프이며, PRD 17장의 표현 규칙을 한 곳에서 통제하기 위해 감수한 비용이다.
