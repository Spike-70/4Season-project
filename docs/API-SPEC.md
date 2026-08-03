# 투자의 사계 — API 명세서

| 항목 | 내용 |
| --- | --- |
| 문서 성격 | 상세 설계 문서 (구현 기준) |
| 상위 문서 | [PRD](./PRD.md) |
| 선행 문서 | [데이터베이스 모델](./DATABASE-MODEL.md) |
| 프로토콜 | HTTPS / JSON / REST |
| 베이스 URL | `https://api.{domain}/v1` |
| 인증 | Bearer JWT (액세스 토큰) + 쿠키 리프레시 토큰 |
| 관련 문서 | [백엔드 아키텍처](./BACKEND-ARCHITECTURE.md), [프론트엔드 아키텍처](./FRONTEND-ARCHITECTURE.md) |

---

## 목차

1. [설계 원칙](#1-설계-원칙)
2. [표준 응답 구조](#2-표준-응답-구조)
3. [오류 체계](#3-오류-체계)
4. [목록 규약 — 페이지네이션·필터·정렬](#4-목록-규약--페이지네이션필터정렬)
5. [공통 규약](#5-공통-규약)
6. [공통 객체 스키마](#6-공통-객체-스키마)
7. [API 총람](#7-api-총람)
8. [인증](#8-인증)
9. [사용자·설정](#9-사용자설정)
10. [온보딩](#10-온보딩)
11. [종목 풀·종목·시장](#11-종목-풀종목시장)
12. [사이클](#12-사이클)
13. [계좌·성과](#13-계좌성과)
14. [규칙 투자](#14-규칙-투자)
15. [매매·가설 기록](#15-매매가설-기록)
16. [조건 도달](#16-조건-도달)
17. [회고](#17-회고)
18. [일지·깨달은 것](#18-일지깨달은-것)
19. [투자 수칙](#19-투자-수칙)
20. [예측·비상 선언](#20-예측비상-선언)
21. [배지](#21-배지)
22. [연습 모드](#22-연습-모드)
23. [결산·리포트](#23-결산리포트)
24. [알림](#24-알림)
25. [내보내기·탈퇴](#25-내보내기탈퇴)
26. [내부 배치 API](#26-내부-배치-api)
27. [비기능 요구](#27-비기능-요구)

---

## 1. 설계 원칙

| # | 원칙 | 적용 |
| --- | --- | --- |
| 1 | **모든 응답이 동일한 봉투를 쓴다** | 성공·실패·목록이 같은 최상위 구조를 갖는다. 프론트엔드는 단 하나의 응답 해석기만 갖는다 |
| 2 | **오류는 코드로 식별하고 문구는 서버가 준다** | `code`는 프로그램 분기용, `message`는 그대로 노출 가능한 한국어. PRD 4.1·17.1의 톤 규칙을 서버에서 일괄 통제한다 |
| 3 | **모든 성과 응답에 기준일을 넣는다** | `as_of_date` 없는 성과 필드는 존재하지 않는다 (PRD 7.9) |
| 4 | **비율은 항상 분자·분모와 함께 반환한다** | `{rate, numerator, denominator, is_displayable}` 구조. PRD 5.4의 "n회 중 m회" 표기를 프론트가 임의로 만들지 않게 한다 |
| 5 | **표시 가능 여부를 서버가 판정한다** | 기준 건수 미달 시 `is_displayable=false`와 `insufficient_reason`을 내려보낸다. 클라이언트마다 다른 임계값을 쓰는 것을 원천 차단한다 (PRD 7.11) |
| 6 | **서비스가 사용자에게 하는 말은 두 형식뿐이다** | 응답의 모든 문구 필드는 `statement`(사실 진술) 또는 `recall`(과거 기록 인용)로 타입이 구분된다. 권유형 문구를 담을 필드가 스키마에 없다 (PRD 17.1) |
| 7 | **쓰기는 멱등성 키를 받는다** | 매매 기입·회고·확인 버튼은 네트워크 재시도로 중복 생성되면 안 된다 |
| 8 | **버전을 경로에 둔다** | `/v1`. 응답 구조 파괴 변경 시 `/v2`를 병렬 운영한다 |

---

## 2. 표준 응답 구조

### 2.1 성공 응답

```json
{
  "success": true,
  "data": { },
  "meta": {
    "request_id": "01J8X...",
    "server_time": "2026-08-04T09:12:33+09:00"
  }
}
```

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| `success` | boolean | Y | 항상 `true` |
| `data` | object \| array \| null | Y | 리소스 본문. 목록 응답은 배열 |
| `meta.request_id` | string | Y | 추적 ID. 오류 문의 시 사용자에게 노출 |
| `meta.server_time` | string(ISO8601, KST) | Y | 서버 기준 시각 |
| `meta.pagination` | object | N | 목록 응답에만 |
| `meta.as_of_date` | string(YYYY-MM-DD) | N | 시세 기반 데이터 포함 시 필수 |
| `meta.notice` | array | N | 규제 고지·안내 문구 (§6.9) |
| `meta.recompute_pending` | boolean | N | 계좌·성과 관련 응답에만. `true`면 재계산 진행 중이며 값이 잠정치임 (§13.6) |
| `meta.recompute_job_id` | number | N | `recompute_pending=true`일 때 진행 상태 조회용 |

### 2.2 목록 응답

```json
{
  "success": true,
  "data": [ ],
  "meta": {
    "request_id": "01J8X...",
    "server_time": "2026-08-04T09:12:33+09:00",
    "pagination": {
      "mode": "cursor",
      "limit": 20,
      "next_cursor": "eyJpZCI6MTIzfQ",
      "has_next": true,
      "total_count": null
    }
  }
}
```

### 2.3 실패 응답

```json
{
  "success": false,
  "error": {
    "code": "TRADE_BACKDATE_NOT_ALLOWED",
    "message": "지난 날짜의 매매는 기입할 수 없습니다.",
    "detail": "가장 최근 개장일인 2026-08-03 이후로만 기입할 수 있습니다.",
    "field_errors": [
      { "field": "trade_date", "code": "OUT_OF_RANGE", "message": "기입 가능한 날짜가 아닙니다." }
    ],
    "docs_url": null,
    "retryable": false
  },
  "meta": {
    "request_id": "01J8X...",
    "server_time": "2026-08-04T09:12:33+09:00"
  }
}
```

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| `error.code` | string | Y | 대문자 SNAKE_CASE. 프론트 분기 키 |
| `error.message` | string | Y | **그대로 사용자에게 노출 가능한 한 문장.** 훈계·비난 표현 금지 |
| `error.detail` | string \| null | Y | 보조 설명. 노출 여부는 클라이언트 판단 |
| `error.field_errors` | array | Y | 폼 검증 실패 시 필드별 오류. 없으면 빈 배열 |
| `error.retryable` | boolean | Y | 동일 요청 재시도로 해결 가능한가 |

> **`message`를 프론트엔드가 만들지 않는다.** 오류 문구는 톤 정책의 일부이며(PRD 4.1), 클라이언트마다 문구가 달라지면 "혼내지 않는다" 원칙이 국지적으로 깨진다. 프론트는 `code`로 UI 형태(토스트/인라인/모달)만 결정한다.

---

## 3. 오류 체계

### 3.1 HTTP 상태 코드 사용 규칙

| 상태 | 사용 상황 |
| --- | --- |
| 200 | 조회·수정 성공 |
| 201 | 생성 성공. `Location` 헤더 포함 |
| 202 | 비동기 작업 접수 (내보내기, 리포트 생성) |
| 204 | 삭제 성공. 본문 없음 |
| 400 | 요청 형식·검증 실패 |
| 401 | 미인증·토큰 만료 |
| 403 | 인가 실패 (타인 리소스 접근) |
| 404 | 리소스 없음 |
| 409 | 상태 충돌 (도메인 규칙 위반) |
| 422 | 형식은 맞으나 도메인 제약 위반 |
| 429 | 요청 한도 초과 |
| 500 | 서버 오류 |
| 503 | 배치 중 일시 잠금 |

> **409와 422의 구분**: 409는 "지금은 안 되는 것"(첫 조정일이 지나 재시작 불가), 422는 "언제도 안 되는 것"(비상 선언 3회차). 프론트엔드의 안내 문구 유형이 달라진다.

### 3.2 공통 오류 코드

| code | HTTP | message (예) |
| --- | --- | --- |
| `UNAUTHENTICATED` | 401 | 로그인이 필요합니다. |
| `TOKEN_EXPIRED` | 401 | 로그인이 만료되었습니다. 다시 로그인해 주세요. |
| `FORBIDDEN` | 403 | 접근 권한이 없습니다. |
| `RESOURCE_NOT_FOUND` | 404 | 요청하신 정보를 찾을 수 없습니다. |
| `VALIDATION_FAILED` | 400 | 입력값을 다시 확인해 주세요. |
| `IDEMPOTENCY_CONFLICT` | 409 | 같은 요청이 이미 처리되었습니다. |
| `RATE_LIMITED` | 429 | 요청이 많습니다. 잠시 후 다시 시도해 주세요. |
| `INTERNAL_ERROR` | 500 | 일시적인 문제가 발생했습니다. |
| `SERVICE_MAINTENANCE` | 503 | 오늘의 시세를 정리하는 중입니다. 잠시 후 다시 열어 주세요. |

### 3.3 도메인 오류 코드

| 도메인 | code | HTTP | 발생 조건 (PRD) |
| --- | --- | --- | --- |
| 사이클 | `CYCLE_NOT_STARTED` | 409 | 가설 기록 미완료로 `PREPARING` 상태 (14.3) |
| | `CYCLE_ALREADY_EXISTS` | 409 | 해당 연도 사이클 중복 생성 (5.2) |
| | `CYCLE_CLOSED` | 409 | 결산 완료 사이클에 쓰기 시도 |
| | `CYCLE_RESTART_WINDOW_CLOSED` | 409 | 첫 정기 조정일 경과 (13.4) |
| | `CYCLE_RESTART_LIMIT_EXCEEDED` | 422 | 사이클당 1회 초과 (13.4) |
| | `STOCK_POOL_NOT_PUBLISHED` | 409 | 다음 해 풀 미공개 (13.1의 공백기) |
| 매매 | `TRADE_BACKDATE_NOT_ALLOWED` | 422 | 소급 입력 (9.1) |
| | `TRADE_MARKET_CLOSED` | 422 | 폐장 전 또는 휴장일 기입 (9.1) |
| | `TRADE_STOCK_NOT_IN_POOL` | 422 | 풀 밖 종목 매수 (9.2) |
| | `TRADE_INSUFFICIENT_CASH` | 422 | 현금 부족 |
| | `TRADE_INSUFFICIENT_POSITION` | 422 | 보유 수량 초과 매도 |
| | `TRADE_STOCK_SUSPENDED` | 422 | 거래정지 종목 (7.8) |
| | `TRADE_ALREADY_SUPERSEDED` | 409 | 이미 정정된 건을 다시 정정 |
| 가설 | `HYPOTHESIS_REQUIRED` | 422 | 매수에 되돌아볼 조건 누락 (9.3) |
| | `HYPOTHESIS_ALREADY_EXISTS` | 409 | 매수 1건에 가설 2개 |
| | `CONDITION_ACTION_REQUIRED` | 422 | `planned_action` 미선택 (9.4) |
| | `CONDITION_ALREADY_TRIGGERED` | 409 | 도달 완료 조건 수정 |
| 조건 | `TRIGGER_ALREADY_RESPONDED` | 409 | 응답 중복 |
| | `SELF_REPORT_NOT_ALLOWED` | 422 | `AUTO` 조건에 자기 보고 시도 (7.5) |
| | `CONDITION_ALREADY_ACTIVE` | 409 | 활성 조건이 있는데 신규 생성 시도 (7.6) |
| 규칙 | `REBALANCE_CONFIRM_DEADLINE_PASSED` | 422 | 다음 조정일 경과 (8.5) |
| | `REBALANCE_ALREADY_CONFIRMED` | 409 | 중복 확인 |
| | `REBALANCE_NOT_AVAILABLE` | 404 | `is_taster` 사이클 (18.6) |
| 일지 | `JOURNAL_BACKFILL_EXPIRED` | 422 | 14일 초과 소급 (10.2) |
| | `JOURNAL_ALREADY_EXISTS` | 409 | 같은 기간 중복 |
| | `JOURNAL_FUTURE_DATE` | 422 | 미래 날짜 |
| 수칙 | `PRINCIPLE_LIMIT_EXCEEDED` | 422 | 5개 초과 (11.4) |
| | `PRINCIPLE_CONTENT_TOO_LONG` | 400 | 120자 초과 |
| | `PRINCIPLE_NOT_EDITABLE` | 409 | 사이클 진행 중 확정 수칙 수정 |
| 비상 | `EMERGENCY_LIMIT_EXCEEDED` | 422 | 연 2회 초과 (8.7) |
| | `EMERGENCY_REASON_REQUIRED` | 400 | 이유 미입력 (4.3) |
| 결산 | `SETTLEMENT_NOT_READY` | 409 | 결산일 미도래 |
| | `REPORT_NOT_GENERATED` | 404 | 생성 배치 미완료 |
| | `RETROSPECTIVE_AFTER_REPORT_VIEWED` | 409 | 리포트 열람 후 회고 최초 작성 (13.1) |
| 설정 | `NOTIFICATION_TYPE_LOCKED` | 422 | 끌 수 없는 알림 비활성화 시도 |
| 연습 | `PRACTICE_PLAN_REQUIRED` | 422 | 사전 선언 없이 재생 시작 (12.2) |
| | `PRACTICE_SESSION_COMPLETED` | 409 | 종료된 세션에 결정 추가 |

---

## 4. 목록 규약 — 페이지네이션·필터·정렬

### 4.1 페이지네이션

**두 모드를 지원하며 엔드포인트마다 하나로 고정한다.**

| 모드 | 사용처 | 요청 파라미터 | `meta.pagination` |
| --- | --- | --- | --- |
| `cursor` | 시간순 무한 스크롤 (일지, 매매, 알림, 깨달은 것) | `limit`, `cursor` | `mode`, `limit`, `next_cursor`, `has_next`, `total_count`(항상 null) |
| `page` | 총 개수가 의미 있는 목록 (종목 풀, 배지, 수칙) | `limit`, `page` | `mode`, `limit`, `page`, `total_pages`, `total_count`, `has_next` |

| 파라미터 | 기본값 | 최대 | 비고 |
| --- | --- | --- | --- |
| `limit` | 20 | 100 | 초과 시 `VALIDATION_FAILED` |
| `cursor` | — | — | 서버 발급 불투명 문자열. 클라이언트가 해석하지 않는다 |
| `page` | 1 | — | 1-base |

> **커서 기본 원칙**: 이 제품의 목록은 대부분 시간순 누적 기록이며(PRD 13.3의 영구 누적), 총 개수 표시가 사용자 가치와 무관하다. `total_count` 계산은 누적 데이터에서 비싸므로 커서 모드에서는 계산하지 않는다.

### 4.2 필터

| 규약 | 내용 |
| --- | --- |
| 형식 | `?<field>=<value>` 단순 등호. 복합 연산자를 쓰지 않는다 |
| 날짜 범위 | `from`, `to` (YYYY-MM-DD, 양끝 포함) |
| 다중 값 | 쉼표 구분 (`tag=SELL_TIMING,MY_PSYCHOLOGY`) |
| 사이클 범위 | `cycle_id=<id>` 또는 `cycle_scope=CURRENT|ALL`. 기본 `CURRENT` |
| 알 수 없는 필터 | **무시하지 않고 400 반환.** 오타로 인한 조용한 전체 조회를 막는다 |

### 4.3 정렬

| 규약 | 내용 |
| --- | --- |
| 형식 | `?sort=<field>` / `?order=asc|desc` |
| 기본값 | 엔드포인트별 명시. 대부분 `created_at desc` |
| 허용 필드 | 엔드포인트별 화이트리스트. 그 외 400 |

### 4.4 부분 응답

| 파라미터 | 설명 |
| --- | --- |
| `include` | 연관 리소스 임베드. 쉼표 구분. 허용값은 엔드포인트별 명시 (예: `include=hypothesis,review`) |

과도한 조합을 막기 위해 `include`는 **1단계 깊이만** 허용한다.

---

## 5. 공통 규약

### 5.1 헤더

| 헤더 | 방향 | 설명 |
| --- | --- | --- |
| `Authorization: Bearer <token>` | 요청 | 인증 필요 엔드포인트 |
| `Idempotency-Key: <uuid>` | 요청 | 모든 POST. 24시간 보관 |
| `X-Client-Version` | 요청 | 앱 버전. 강제 업데이트 판단 |
| `X-Request-Id` | 응답 | `meta.request_id`와 동일 |
| `Retry-After` | 응답 | 429·503 시 |

### 5.2 데이터 표현

| 항목 | 규약 |
| --- | --- |
| 날짜 | `YYYY-MM-DD` |
| 시각 | ISO 8601 + KST 오프셋 (`2026-08-04T09:12:33+09:00`) |
| 금액·포인트 | **문자열** (`"29850000.0000"`). 부동소수점 오차 방지 |
| 비율 | **문자열 소수** (`"0.0312"` = 3.12%) |
| 수량 | 문자열 (`"120.500000"`) |
| null | 값 없음을 명시. 빈 문자열을 쓰지 않는다 |
| 열거값 | 대문자 SNAKE_CASE 문자열 |
| 배열 | 항목 없으면 `[]`. null 금지 |

> **금액과 비율을 문자열로 반환하는 이유**: JSON number는 IEEE 754 배정밀도로 파싱되어 `NUMERIC(20,4)`의 값을 정확히 표현하지 못한다. 이 제품의 핵심 산출물이 계좌 간 차이(뺄셈 결과)이므로 표시 단계까지 정밀도를 유지해야 한다. 프론트엔드는 전용 `Decimal` 래퍼로 다룬다.

### 5.3 인증·인가

| 항목 | 규약 |
| --- | --- |
| 액세스 토큰 | JWT, 만료 30분, `Authorization` 헤더 |
| 리프레시 토큰 | HttpOnly·Secure·SameSite=Lax 쿠키, 만료 30일, 회전 발급 |
| 인가 | 모든 사용자 소유 리소스는 **경로 id + 소유자 일치**를 서비스 계층에서 검증. 불일치 시 404(존재 은닉) |
| 공개 엔드포인트 | 랜딩 통계, 약관, 헬스체크만 |

---

## 6. 공통 객체 스키마

반복 사용되는 객체를 여기서 한 번만 정의한다. 이하 명세는 이 이름으로 참조한다.

### 6.1 `Money`

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `amount` | string | 포인트 금액 |
| `currency` | string | 항상 `"POINT"` |

### 6.2 `RateMetric` — 비율 지표

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `rate` | string \| null | 비율 (0.0312) |
| `numerator` | number | 분자 |
| `denominator` | number | 분모. **0 가능** |
| `label` | string | 표시명 (예: `"내 계획 지킨 비율"`) |
| `display_text` | string | `"7회 중 4회"` — 서버가 생성 |
| `is_displayable` | boolean | 기준 미달·분모 0이면 false |
| `insufficient_reason` | string \| null | `SAMPLE_TOO_SMALL`, `NO_DENOMINATOR`, `CYCLE_TOO_SHORT` |

> `rate` 단독 사용을 방지하기 위해 **`display_text`를 서버가 만들어 내려보낸다.** PRD 5.4의 "분모를 함께 표시" 요구가 클라이언트 구현에 의존하지 않게 한다.

### 6.3 `PerformanceValue` — 성과 값

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `rate` | string | 수익률 |
| `rate_pp` | string | 퍼센트포인트 표시값 (`"7.4"`) |
| `amount` | Money | 금액 병기 (PRD 7.9) |
| `as_of_date` | string | 기준일 |
| `period_months` | string | 사이클 길이 (PRD 7.11-4) |

### 6.4 `AccountSummary`

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `account_type` | string | `RULE`/`FREE`/`PLAN`/`HOLD` |
| `label` | string | `"규칙 투자 계좌"` 등 PRD 5.5 명칭 |
| `initial_capital` | Money | |
| `total_value` | Money | |
| `cash_balance` | Money | |
| `market_value` | Money | |
| `cash_ratio` | string | 현금 비중 (PRD 9.2) |
| `return_rate` | string | |
| `position_count` | number | |
| `as_of_date` | string | |
| `is_identical_to_hold` | boolean | `is_taster` 사이클의 `RULE` 계좌 (PRD 18.6) |

### 6.5 `StockSummary`

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `stock_id` | number | |
| `ticker` | string | |
| `name` | string | |
| `market` | string | |
| `sector_name` | string \| null | |
| `status` | string | `LISTED`/`SUSPENDED_SHORT`/... |
| `open_price` | string | 직전 영업일 시가 |
| `close_price` | string | 직전 영업일 종가 |
| `change_rate` | string | 전일 대비 |
| `as_of_date` | string | |

> **시가를 함께 반환하는 것이 요구사항이다.** PRD 9.1은 "보유 종목의 일별 시가·종가를 항상 표시하여 기입 오류를 방지"할 것을 요구한다. 종가만 내려보내면 사용자가 매매 기입 시 가격을 대조할 기준이 하나뿐이 된다. 다만 **판정과 평가는 여전히 종가만 사용한다** (PRD 7.11-3).

### 6.6 `ConditionObject`

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `condition_id` | number | |
| `kind` | string | `REVIEW`/`TARGET` |
| `condition_key` | string \| null | 카탈로그 키 |
| `display_text` | string | `"목표가 대비 15% 하락"` |
| `params` | object | |
| `evaluation_mode` | string | `AUTO`/`SELF_REPORT` |
| `planned_action` | string | `SELL_ALL`/`SELL_HALF`/`HOLD` |
| `planned_action_text` | string | `"전량 판다"` |
| `status` | string | |
| `threshold_price` | string \| null | |
| `triggered_on` | string \| null | |

### 6.7 `RecallBlock` — 과거 기록 인용

PRD 17.1이 허용하는 두 문구 형식 중 "당신이 예전에 이렇게 적었습니다"를 담는 표준 블록. 매수 화면·조건 도달 알림·회고 화면에서 재사용된다 (PRD 9.5, 10.4).

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `recall_type` | string | `PAST_HYPOTHESIS`, `PAST_INSIGHT`, `PRINCIPLE`, `PAST_REVIEW` |
| `recorded_on` | string | 원문 작성일 |
| `cycle_year` | number | 어느 사이클의 기록인가 |
| `content` | string | 사용자가 쓴 문장 원문 |
| `context` | object \| null | 종목·결과 등 부가 정보 |
| `resource_ref` | object | `{type, id}` — 상세로 이동 |

### 6.8 `StatementBlock` — 사실 진술

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `statement_key` | string | 문구 템플릿 키 |
| `text` | string | 완성된 사실 문장 |
| `values` | object | 치환된 수치 |
| `is_interpretive` | boolean | 해석 문장 여부. 기준 건수 미달 시 서버가 아예 내려보내지 않는다 |

### 6.9 `Notice` — 고지

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `notice_key` | string | `RULE_IS_EXAMPLE_ONLY`, `CLOSE_PRICE_BASIS`, `SELF_REPORT_BIAS`, `POOL_IS_NOT_RECOMMENDATION`, `REAL_TRADE_IS_USER_RESPONSIBILITY` |
| `text` | string | 고지 문구 |
| `severity` | string | `INFO`/`IMPORTANT` |

> 고지는 화면 하드코딩이 아니라 **응답 `meta.notice` 배열**로 내려온다. PRD 17장의 8개 원칙은 법률 검토 결과에 따라 문구가 바뀔 수 있으며, 그때 앱 배포 없이 갱신되어야 한다.

---

## 7. API 총람

| 그룹 | 경로 접두 | 엔드포인트 수 |
| --- | --- | --- |
| 인증 | `/auth` | 4 |
| 사용자·설정 | `/me` | 6 |
| 온보딩 | `/onboarding` | 4 |
| 종목·시장 | `/stock-pools`, `/stocks`, `/market` | 9 |
| 사이클 | `/cycles` | 8 |
| 계좌·성과 | `/cycles/{id}/accounts`, `/performance`, `/recompute-jobs` | 6 |
| 규칙 투자 | `/cycles/{id}/rebalances` | 3 |
| 매매·가설 | `/trades`, `/hypotheses` | 10 |
| 조건 | `/conditions`, `/condition-triggers` | 6 |
| 회고 | `/trade-reviews` | 3 |
| 일지 | `/journals`, `/insights` | 10 |
| 수칙 | `/principles` | 8 |
| 예측·비상 | `/predictions`, `/emergency-declarations` | 4 |
| 배지 | `/badges` | 2 |
| 연습 | `/practice` | 5 |
| 결산 | `/settlements`, `/retrospective`, `/reports` | 7 |
| 알림 | `/notifications` | 5 |
| 내보내기·탈퇴 | `/exports`, `/me` | 4 |
| 내부 배치 | `/internal` | 13 |

---

## 8. 인증

### 8.1 `POST /auth/kakao`

카카오 인가 코드로 로그인·가입.

**요청 본문**

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| `authorization_code` | string | Y | 카카오 인가 코드 |
| `redirect_uri` | string | Y | 발급 시 사용한 URI |

**응답 201/200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `access_token` | string | JWT |
| `token_type` | string | `Bearer` |
| `expires_in` | number | 초 |
| `is_new_user` | boolean | 신규 가입 여부 |
| `user` | UserObject | §9.1 |
| `onboarding` | object | `{is_completed, current_step}` |

리프레시 토큰은 `Set-Cookie`로 전달한다.

**오류**: `KAKAO_AUTH_FAILED`(401), `KAKAO_PROFILE_UNAVAILABLE`(502)

### 8.2 `POST /auth/refresh`

쿠키의 리프레시 토큰으로 액세스 토큰 재발급. 리프레시 토큰은 회전 발급된다.

**응답 200**: `access_token`, `expires_in`

**오류**: `TOKEN_EXPIRED`(401), `TOKEN_REVOKED`(401)

### 8.3 `POST /auth/logout`

현재 리프레시 토큰 폐기. **204**

### 8.4 `GET /auth/session`

토큰 유효성과 사용자 상태 확인. 앱 부팅 시 1회 호출.

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `user` | UserObject | |
| `current_cycle` | CycleSummary \| null | §12.1 |
| `onboarding` | object | |
| `unread_notification_count` | number | |
| `pending_actions` | array | `[{action_key, resource_ref}]` — `CONDITION_TRIGGER_UNANSWERED`, `REBALANCE_UNCONFIRMED`, `SETTLEMENT_PENDING`, `RETROSPECTIVE_PENDING`, `PRINCIPLE_SETUP_PENDING`, `CYCLE_START_AVAILABLE` |

> `pending_actions`가 첫 화면의 진입점을 결정한다. 클라이언트가 여러 엔드포인트를 조합해 판단하면 화면마다 다른 결론이 나온다.

---

## 9. 사용자·설정

### 9.1 `GET /me`

**응답 200 — UserObject**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `user_id` | number | |
| `public_id` | string(uuid) | |
| `nickname` | string | |
| `email` | string \| null | |
| `status` | string | |
| `investor_type` | string \| null | `AGGRESSIVE`/`DEFENSIVE` |
| `joined_at` | string | |
| `onboarded_at` | string \| null | |
| `cycle_count` | number | 완주한 사이클 수 |

### 9.2 `PATCH /me`

**요청**: `nickname`(선택)

### 9.3 `GET /me/settings`

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `is_badge_enabled` | boolean | 배지 기능 자체 (PRD 7.10) |
| `is_badge_public_hidden` | boolean | 배지 숨김 |
| `daily_journal_reminder_hour` | number | |
| `weekly_review_weekday` | number | |
| `theme_preference` | string | `SYSTEM`/`LIGHT`/`DARK` |
| `profit_color_scheme` | string | `KR`(상승 적) / `INTL`(상승 녹) |
| `notification_preferences` | array | `[{notification_type, is_enabled, is_locked}]` |

> 표시 설정을 서버에 두는 이유는 기기 간 동기화다. 손익 색 반전이 기기마다 다르면 같은 수치가 반대 의미로 읽힌다.

`is_locked=true`인 항목(`CONDITION_HIT`, `REBALANCE_DAY`, `SETTLEMENT_DAY`)은 끌 수 없다.

### 9.4 `PATCH /me/settings`

부분 갱신. `is_locked` 항목을 끄려 하면 `NOTIFICATION_TYPE_LOCKED`(422).

### 9.5 `GET /me/survey`

가장 최근 투자 성향 설문 결과와 문항 정의.

**응답 200**: `result_type`, `answered_at`, `answers[]`, `questions[]`

### 9.6 `POST /me/survey`

**요청**: `answers: [{question_key, choice_key}]` (3문항 이내)

**응답 201**: `result_type`, `applied_to` — 결과의 쓰임 명시 (`["REPORT_COMPARISON", "POOL_ORDERING"]`). PRD 14.4의 "성향에 따라 다른 종목 풀을 주지 않는다"를 응답으로 드러낸다.

---

## 10. 온보딩

### 10.1 `GET /onboarding`

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `current_step` | number | 1~8 |
| `completed_steps` | array | |
| `steps` | array | `[{step, key, title, is_completed, is_blocking}]` |
| `cycle_preview` | object | 6단계에 보여줄 일정 정보 (§10.3) |
| `is_completed` | boolean | |

**단계 정의**

| step | key | 대응 PRD |
| --- | --- | --- |
| 1 | `LANDING` | 14.1 |
| 2 | `LOGIN` | |
| 3 | `SURVEY` | |
| 4 | `PRACTICE` | |
| 5 | `DRAFT_PRINCIPLE` | |
| 6 | `SCHEDULE_REVIEW` | |
| 7 | `STOCK_SELECTION` | |
| 8 | `FIRST_HYPOTHESIS` | |

> 7과 8은 `is_blocking`이 같은 값을 가지며 한 덩어리로 처리한다 (PRD 14.3). 7만 완료하고 이탈하면 사이클은 `PREPARING`에 머문다.

### 10.2 `POST /onboarding/steps/{step_key}/complete`

**요청**: 단계별 페이로드 (`payload` 객체)

**응답 200**: 갱신된 온보딩 상태. **6단계 정보는 매 호출 시 재계산**되어 반환된다 — 이탈 중 조정일이 지나갔을 수 있다 (PRD 14.3).

### 10.3 `GET /onboarding/cycle-preview`

종목 선택 전 반드시 보여줘야 하는 일정 (PRD 14.2).

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `year` | number | |
| `today` | string | |
| `settlement_date` | string | |
| `length_months` | string | 예상 사이클 길이 |
| `remaining_rebalance_count` | number | 3/2/1/0 |
| `remaining_rebalance_dates` | array | 남은 정기 조정일 |
| `initial_symbol_count` | number | 10/8/6/4 |
| `capital_per_symbol` | Money | 종목당 배분액 |
| `reduction_path` | array | `[10,8,6,4]` |
| `is_taster` | boolean | 조정 0회 (PRD 18.6) |
| `taster_notice` | StatementBlock \| null | `"이번 사이클은 ○개월입니다. 정기 조정은 없고, 12월에 결산합니다"` |

### 10.4 `POST /onboarding/complete`

8단계 완료 후 호출. 사이클을 `ACTIVE`로 전이시키고 `started_on`을 확정한다.

**응답 200**: `cycle` (CycleSummary)

**오류**: `ONBOARDING_STEP_INCOMPLETE`(409)

---

## 11. 종목 풀·종목·시장

### 11.1 `GET /stock-pools/current`

그 해의 종목 풀 30개.

**쿼리**: `year`(선택, 기본 현재 연도), `order`(`DEFAULT`/`BY_INVESTOR_TYPE`)

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `stock_pool_id` | number | |
| `year` | number | |
| `published_at` | string \| null | null이면 미공개 |
| `stock_count` | number | 30 |
| `exclusion_rules` | array | `[{rule_key, label, threshold_text}]` (PRD 18.4) |
| `disclosure_statement` | StatementBlock | `"선정된 종목에 대한 의견은 제공하지 않습니다."` |
| `stocks` | array | StockSummary 확장 |

각 종목 항목에는 **선정 사유 필드가 없다.** PRD 17.1의 금지 문구를 API 계약 수준에서 차단한다.

`meta.notice`에 `POOL_IS_NOT_RECOMMENDATION`이 항상 포함된다.

**오류**: `STOCK_POOL_NOT_PUBLISHED`(409) — 공백기 (PRD 13.1)

### 11.2 `GET /stocks/{stock_id}`

**응답 200**: StockSummary + `listed_at`, `status_notice`(거래정지·상폐 시 StatementBlock)

### 11.3 `GET /stocks/{stock_id}/prices`

**쿼리**: `from`, `to`, `interval`(`DAILY` 고정)

**응답 200**: `[{trade_date, open, high, low, close, volume}]`

`meta.as_of_date`와 `meta.notice: [CLOSE_PRICE_BASIS]` 포함.

### 11.4 `GET /stocks/{stock_id}/disclosures`

**쿼리**: `from`, `to`, `limit`, `cursor`

**응답 200**: `[{disclosure_id, title, disclosed_at, source_url, submitter}]`

> **본문·요약 필드가 없다** (PRD 15.2). 클라이언트는 `source_url`로 외부 이동만 제공한다.

### 11.5 `GET /stocks/{stock_id}/earnings-schedule`

실적 발표 일정 (PRD 15.2가 명시한 '사실' 3종 중 하나).

**쿼리**: `from`, `to`

**응답 200**: `[{fiscal_period, scheduled_date, is_confirmed, source_url}]`

> **실적 수치·전망·컨센서스 필드가 없다.** 일정만 전달하는 것이 사실 전달이고, 수치를 정리해 주는 순간 해석 제공이 되며 유료 데이터 의존이 생긴다 (PRD 15.3).

`is_confirmed=false`인 항목에는 "잠정 일정입니다" 캡션을 붙일 수 있도록 플래그를 내려보낸다.

### 11.6 `GET /stocks/{stock_id}/my-records`

이 종목에 대해 내가 쓴 모든 기록. **지난 사이클 포함** (PRD 10.4-1).

**쿼리**: `cycle_scope`(기본 `ALL`), `limit`, `cursor`

**응답 200**: `RecallBlock[]` — 가설 기록·회고·깨달은 것·일지 항목이 시간순으로 섞여 반환된다.

### 11.7 `GET /market/summary`

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `as_of_date` | string | |
| `indices` | array | `[{index_code, close_value, change_rate}]` |
| `trading_value` | array | 시장별 거래대금 |
| `investor_flows` | array | `[{market, investor_type, net_buy_amount}]` |
| `basis_statement` | StatementBlock | `"2026년 8월 3일 종가 기준입니다."` |

### 11.8 `GET /market/monthly-context`

월간 일지 작성 시 보여줄 수치 (PRD 10.1).

**쿼리**: `year_month`(YYYY-MM)

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `market` | object | 지수 등락, 거래대금, 투자 주체별 순매수 합계 |
| `my_stocks` | array | 보유 종목 등락 |
| `my_behavior` | object | `trade_count`, `journal_entry_days`, `principle_encounter_count`, `principle_kept_count` |

이 응답은 `journal_entry.market_context`로 스냅샷 저장된다.

---

## 12. 사이클

### 12.1 `GET /cycles/current`

**응답 200 — CycleSummary**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `cycle_id` | number | |
| `year` | number | |
| `status` | string | |
| `started_on` | string \| null | |
| `settlement_date` | string | |
| `days_until_settlement` | number | |
| `length_months` | string | 진행 기준 |
| `is_taster` | boolean | |
| `initial_symbol_count` | number | |
| `remaining_rebalance_count` | number | |
| `next_rebalance_date` | string \| null | |
| `rebalance_schedule` | array | `[{sequence, event_date, is_settlement, is_past, is_confirmed}]` |
| `restart_available` | boolean | 첫 조정일 이전 & 미사용 (PRD 13.4) |
| `stock_pool_id` | number | |

### 12.2 `POST /cycles`

사이클 생성 (온보딩 7단계에서 호출).

**요청**

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| `stock_ids` | array\<number\> | Y | 선택 종목. 길이는 `initial_symbol_count`와 정확히 일치 |
| `selection_reasons` | array | N | `[{stock_id, reason_text}]` (PRD 16-B4) |

**응답 201**: CycleSummary + `accounts`(4개 계좌 초기 상태)

**오류**: `CYCLE_ALREADY_EXISTS`(409), `STOCK_COUNT_MISMATCH`(422), `STOCK_NOT_IN_POOL`(422)

> 이 시점에 4개 계좌가 생성되고 초기 균등 배분 원장이 적재된다. 단 `cycle.status`는 `PREPARING`이며, 첫 가설 기록 완료 전까지 조건 판정·조정 대상이 되지 않는다.

### 12.3 `GET /cycles/start-context`

**두 번째 사이클 이후의 시작 화면**에 필요한 것을 한 번에 반환한다 (PRD 16-B). 온보딩(§10)은 최초 1회 전용이며, 재가입자는 설문·연습·임시 수칙 단계를 다시 겪지 않는다.

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `is_available` | boolean | 새 사이클 시작 가능 여부 |
| `unavailable_reason` | string \| null | `POOL_NOT_PUBLISHED`(공백기), `CYCLE_ALREADY_ACTIVE` |
| `active_principles` | array\<PrincipleObject\> | **지난 결산에서 확정한 수칙** (PRD 16-B1) |
| `schedule` | object | §10.3의 `cycle-preview`와 동일 구조 |
| `stock_pool` | object | §11.1과 동일 구조 |
| `previous_cycle_summary` | object \| null | 지난 사이클 최종 성과 요약 |
| `steps` | array | `[{step, key, is_completed}]` — 수칙 확인 → 일정 확인 → 종목 선택 → 가설 기록 → 예측 |

**시작 단계 정의** (온보딩 8단계와 다르다)

| step | key | 대응 PRD |
| --- | --- | --- |
| 1 | `PRINCIPLE_REVIEW` | 16-B1. 첫 사이클이면 생략 |
| 2 | `SCHEDULE_REVIEW` | 16-B2 |
| 3 | `STOCK_SELECTION` | 16-B3·B4 |
| 4 | `FIRST_HYPOTHESIS` | 16-B5·B6 |
| 5 | `PREDICTION` | 16-B7 |

3~4단계는 온보딩과 마찬가지로 한 덩어리다. 종목만 고르고 가설 기록 없이 이탈하면 사이클은 `PREPARING`에 머문다.

### 12.4 `GET /cycles/{cycle_id}/stock-selections`

사이클 시작 시 고른 종목과 선택 이유 (PRD 16-B4).

**쿼리**: `generation`(선택, 기본 최신)

**응답 200**: `[{stock, display_order, reason_text, selected_at, restart_generation}]`

재시작한 사이클은 이전 세대도 조회 가능하다. 무엇을 골랐다가 무엇으로 바꿨는지가 결산의 회고 재료가 된다.

### 12.5 `GET /cycles`

사용자의 전체 사이클 목록 (다년 이력).

**응답 200**: CycleSummary 배열 + 각 사이클의 최종 성과 요약

### 12.6 `POST /cycles/{cycle_id}/restart`

사이클 재시작 (PRD 13.4).

**요청**: `stock_ids`, `confirm: true`

**응답 200**: 새 CycleSummary

**오류**: `CYCLE_RESTART_WINDOW_CLOSED`(409), `CYCLE_RESTART_LIMIT_EXCEEDED`(422)

**부수 효과**: 계좌·보유·가설 기록은 폐기, **일지와 깨달은 것은 유지** (PRD 13.4). 응답 `preserved` 필드로 무엇이 남았는지 명시한다.

### 12.7 `GET /cycles/{cycle_id}/restart-preview`

재시작 전 확인 화면용. 무엇이 사라지고 무엇이 남는지.

**응답 200**: `{will_be_discarded: [...], will_be_preserved: [...], is_available, unavailable_reason}`

### 12.8 `GET /cycles/{cycle_id}/timeline`

사이클 내 주요 사건 연대기 (조정일, 조건 도달, 비상 선언, 매매).

**쿼리**: `event_types`, `from`, `to`, `cursor`

**응답 200**: `[{occurred_on, event_type, title, resource_ref}]`

---

## 13. 계좌·성과

### 13.1 `GET /cycles/{cycle_id}/accounts`

4개 계좌 현재 상태.

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `accounts` | array\<AccountSummary\> | 4건. `RULE`, `FREE`, `PLAN`, `HOLD` 순 |
| `pairs` | array | `[{primary: "PLAN", counterpart: "FREE", label: "계획과 실제"}]` — 짝 관계 (PRD 7.2) |
| `as_of_date` | string | |

`meta.notice`에 `CLOSE_PRICE_BASIS` 포함. `is_taster` 사이클이면 `RULE` 계좌의 `is_identical_to_hold=true`와 함께 해당 고지가 추가된다.

### 13.2 `GET /cycles/{cycle_id}/accounts/{account_type}/positions`

**응답 200**: `[{stock, quantity, average_cost, market_value, unrealized_pnl, return_rate, weight, is_valuation_frozen}]` + `cash_balance`, `cash_ratio`

### 13.3 `GET /cycles/{cycle_id}/performance`

성과 곡선과 세 개의 차이. **이 제품의 핵심 응답이다.**

**쿼리**: `from`, `to`, `granularity`(`DAILY`/`WEEKLY`)

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `series` | array | `[{account_type, points: [{date, total_value, return_rate}]}]` |
| `events` | array | 차트 마커용 사건 목록 (아래) |
| `gaps` | array | 세 개의 차이 (아래) |
| `adherence` | object | `{plan: RateMetric, rule_confirm: RateMetric}` |
| `as_of_date` | string | |
| `period_months` | string | |

**`events` 항목 구조**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `event_type` | string | `REBALANCE`, `SETTLEMENT`, `EMERGENCY_DECLARATION`, `CONDITION_HIT`, `CYCLE_START` |
| `occurred_on` | string | |
| `label` | string | 마커 툴팁 문구 |
| `resource_ref` | object | `{type, id}` — 상세로 이동 |

> 마커 데이터를 성과 응답에 포함시키는 이유는, 클라이언트가 여러 엔드포인트를 조합해 차트를 그리면 곡선과 마커의 기준일이 어긋날 수 있기 때문이다. 곡선과 사건은 같은 응답에서 나와야 한다.

**`gaps` 항목 구조**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `gap_key` | string | `PLAN_MINUS_FREE` / `FREE_MINUS_HOLD` / `PLAN_MINUS_HOLD` |
| `label` | string | `"계획과 실제의 차이"` 등 PRD 7.9 명칭 |
| `value` | PerformanceValue | 퍼센트포인트 + 금액 병기 |
| `statement` | StatementBlock \| null | 기준 건수 미달 시 null |

> **`adherence.plan`과 `adherence.rule_confirm`은 별개 객체로만 반환된다.** 합산 필드가 스키마에 존재하지 않는다 (PRD 5.4).

### 13.4 `GET /cycles/{cycle_id}/deviations`

항목별 내역 (PRD 7.9).

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `items` | array | 아래 |
| `total_gap` | PerformanceValue | 계획과 실제의 차이 |
| `self_report_ratio` | RateMetric | 측정 편향 지표 (PRD 18.11) |
| `bias_notice` | Notice \| null | 자기 보고 비중이 높을 때 |

**`items` 구조**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `category` | string | `SAID_SELL_BUT_HELD` 등 |
| `label` | string | `"판다고 해놓고 안 팔았다"` |
| `counterpart_category` | string \| null | 대칭 짝. **`SAID_SELL_BUT_HELD` ↔ `SAID_HOLD_BUT_SOLD` 쌍에만 존재하고 나머지는 null** (DB 문서 §13.8) |
| `render_group` | string | `PAIR` 또는 `SINGLE`. 클라이언트 렌더링 분기 기준 |
| `count` | number | 건수 |
| `impact` | PerformanceValue \| null | 사이클 종료 전에는 null |
| `is_displayable` | boolean | 3건 미만이면 false |
| `insufficient_reason` | string \| null | |
| `events` | array | `[{occurred_on, stock, resource_ref}]` |

> `render_group`으로 렌더링 방식을 서버가 지정한다. `PAIR` 항목은 반드시 짝과 나란히 그려야 하며, 한쪽만 그리면 "파는 것이 정상"이라는 메시지가 된다 (PRD 7.9). `SINGLE` 항목(조기 매도, 계획 없는 매수, 가설 없는 매수)은 대응하는 반대 행동이 존재하지 않으므로 단독으로 표시한다. 없는 반대말을 인위적으로 만들면 존재하지 않는 행동 유형을 세게 된다.

### 13.5 `GET /recompute-jobs/{job_id}`

계좌 재계산 진행 상태. 매매 정정 후 클라이언트가 폴링한다.

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `job_id` | number | |
| `status` | string | `QUEUED`/`RUNNING`/`SUCCEEDED`/`FAILED` |
| `trigger_reason` | string | `TRADE_REVISED` 등 |
| `from_date` | string | 재계산 시작일 |
| `queued_at` / `finished_at` | string \| null | |
| `statement` | StatementBlock | `"정정한 내용을 반영하는 중입니다."` |

**폴링 규약**: 3초 간격, 최대 2분. 타임아웃 시 클라이언트는 수동 새로고침을 안내한다. `FAILED`면 `meta.notice`에 안내가 포함되며, 사용자에게 재시도 버튼을 제공한다.

> 재계산 중에도 계좌 조회는 **막지 않는다.** 이전 값을 `meta.recompute_pending=true`와 함께 반환한다. 화면을 잠그면 사용자는 정정을 실수로 인식하게 되며, 정정은 이 제품이 권장하는 자가 처리 행위다 (PRD 3.5).

### 13.6 `GET /cycles/{cycle_id}/cost-policy`

적용된 거래비용 기준 (PRD 7.7의 "사용자가 확인할 수 있어야").

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `commission_rate` | string | 네 계좌 공통 |
| `tax_rate_sell` | string | |
| `slippage_rate_virtual` | string | 가상 계좌 집행 전용 |
| `applies_to` | object | 각 요율이 어느 계좌에 적용되는지 |
| `effective_from` | string | |
| `statement` | StatementBlock | 설명 문구 |

---

## 14. 규칙 투자

### 14.1 `GET /cycles/{cycle_id}/rebalances`

정기 조정 목록.

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `strategy` | object | `{name, description, disclaimer: Notice}` |
| `items` | array | 아래 |
| `confirm_rate` | RateMetric | 규칙 실행 확인율 |

**`items` 구조**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `rebalance_id` | number | |
| `sequence` | number | |
| `event_date` | string | |
| `status` | string | `SCHEDULED`/`EXECUTED` |
| `sold_items` | array | `[{stock, period_return_rate, rank, tie_break_reason}]` |
| `redistribution` | array | `[{stock, added_amount}]` |
| `position_count_before/after` | number | |
| `confirmation` | object \| null | `{confirmed_at, is_available, deadline_date}` |

`meta.notice`에 `RULE_IS_EXAMPLE_ONLY` 항상 포함 (PRD 8.4).

**오류**: `REBALANCE_NOT_AVAILABLE`(404) — `is_taster` 사이클

### 14.2 `GET /rebalances/{rebalance_id}`

단건 상세. 조정일 화면용. 선정 근거 전체를 포함한다 (PRD 17장 4번).

### 14.3 `POST /rebalances/{rebalance_id}/confirm`

"이대로 하겠습니다".

**요청**: 본문 없음. `Idempotency-Key` 필수.

**응답 201**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `confirmed_at` | string | |
| `confirm_rate` | RateMetric | 갱신된 확인율 |
| `account_unaffected_statement` | StatementBlock | `"이 확인은 계좌를 움직이지 않습니다."` (PRD 8.5) |

**오류**: `REBALANCE_CONFIRM_DEADLINE_PASSED`(422), `REBALANCE_ALREADY_CONFIRMED`(409)

> 응답에 계좌 변경 필드가 없다. 확인이 계좌를 움직이지 않는다는 원칙(PRD 7.4)을 API 계약으로 표현한다.

---

## 15. 매매·가설 기록

### 15.1 `GET /trades`

**쿼리**: `cycle_id`, `stock_id`, `side`, `from`, `to`, `status`(기본 `ACTIVE`), `include`(`hypothesis,review`), `limit`, `cursor`

**응답 200 — TradeObject 배열**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `trade_id` | number | |
| `stock` | StockSummary | |
| `side` | string | |
| `quantity` | string | |
| `price` | string | |
| `amount` | Money | |
| `fee_amount` / `tax_amount` | Money | |
| `trade_date` | string | |
| `entered_at` | string | |
| `status` | string | |
| `is_full_exit` | boolean | |
| `revision_number` | number | |
| `hypothesis` | HypothesisObject \| null | `include` 시 |
| `review` | ReviewObject \| null | `include` 시 |

`meta.notice`에 `REAL_TRADE_IS_USER_RESPONSIBILITY` 포함 (PRD 17장 7번).

### 15.2 `GET /trades/entry-context`

**매매 기입 화면 진입 시 필요한 모든 것을 한 번에 반환한다.** 여러 요청을 조합하면 화면마다 다른 정보가 뜨게 되고, PRD 9.5의 "필요한 순간에 저절로 나타나야 한다"가 클라이언트 구현 품질에 좌우된다.

**쿼리**: `stock_id`, `side`

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `is_entry_allowed` | boolean | 폐장 후·개장일 여부 |
| `entry_blocked_reason` | string \| null | `MARKET_NOT_CLOSED`, `HOLIDAY`, `CYCLE_NOT_ACTIVE` |
| `tradable_date` | string | 기입 대상 거래일 |
| `stock` | StockSummary | 시가·종가 표시용 (PRD 9.1) |
| `available_cash` | Money | |
| `current_position` | object \| null | 보유 수량·평균단가 |
| `principles` | array\<RecallBlock\> | `ON_BUY`/`ON_SELL` 수칙 (PRD 11.5) |
| `past_records` | array\<RecallBlock\> | 이 종목 과거 기록 (PRD 9.5) |
| `condition_catalog` | array | 조건 선택지 (PRD 9.4) |
| `mood_options` / `conviction_scale` | array | 선택지 |
| `existing_condition` | ConditionObject \| null | 추가 매수 시 "기존 조건 유지?" 질문용 (PRD 7.6) |

### 15.3 `POST /trades`

매매 기입. 매수인 경우 가설 기록을 **같은 요청에 포함**한다 — 분리하면 조건 없는 매수가 생겨 계산이 성립하지 않는다 (PRD 9.3).

**요청**

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| `stock_id` | number | Y | |
| `side` | string | Y | |
| `quantity` | string | Y | |
| `price` | string | Y | |
| `trade_date` | string | Y | 당일만 허용 |
| `hypothesis` | object | 매수 시 Y | 아래 |
| `keep_existing_condition` | boolean | N | 추가 매수 시 (PRD 7.6) |

**`hypothesis` 객체**

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| `rationale` | string | N | 논리 |
| `review_condition` | object | **Y** | 되돌아볼 조건 + 그때 할 일 |
| `target_condition` | object | N | 목표 + 그때 할 일 |
| `expected_holding_until` | string | N | |
| `sell_on_holding_expiry` | boolean | N | 기본 false (PRD 7.5) |
| `conviction_level` | number | N | 1~5 |
| `mood` | string | N | |

**조건 객체 (`review_condition` / `target_condition`)**

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| `condition_key` | string | 조건부 | 카탈로그 선택 시 |
| `custom_text` | string | 조건부 | 자유 서술 시 |
| `params` | object | N | `{percent: 15}` |
| `planned_action` | string | **Y** | `SELL_ALL`/`SELL_HALF`/`HOLD`. **기본값 없음** |

**응답 201**: TradeObject + `hypothesis` + `account_after`(FREE 계좌 갱신 결과)

**오류**: `HYPOTHESIS_REQUIRED`(422), `CONDITION_ACTION_REQUIRED`(422), `TRADE_BACKDATE_NOT_ALLOWED`(422), `TRADE_MARKET_CLOSED`(422), `TRADE_STOCK_NOT_IN_POOL`(422), `TRADE_INSUFFICIENT_CASH`(422), `TRADE_INSUFFICIENT_POSITION`(422)

> 매도 시 `is_full_exit=true`면 응답에 `next_action: "TRADE_REVIEW"`, 부분 매도면 `next_action: "SET_NEW_CONDITION"`이 포함된다 (PRD 9.7). 클라이언트가 보유 수량을 계산해 분기하지 않는다.

### 15.4 `PATCH /trades/{trade_id}`

매매 정정 (PRD 3.5).

**요청**: `quantity`, `price`, `trade_date`(선택), `reason`(선택)

**응답 200**: 새 TradeObject + `recompute_job_id`, `recompute_status`

**오류**: `TRADE_ALREADY_SUPERSEDED`(409), `CYCLE_CLOSED`(409)

> 정정은 원본을 `SUPERSEDED`로 두고 새 레코드를 만든다. 응답에 재계산 작업 ID가 포함되며, 클라이언트는 `GET /cycles/{id}/accounts`를 다시 호출하기 전에 완료를 기다려야 한다. 재계산 중 계좌 조회는 `meta.recompute_pending=true`를 반환한다.

### 15.5 `DELETE /trades/{trade_id}`

**응답 204** + 재계산 작업 등록

### 15.6 `GET /trades/{trade_id}/revisions`

정정 이력.

**응답 200**: `[{revision_type, before, after, reason, revised_at}]`

### 15.7 `GET /hypotheses/{hypothesis_id}`

**응답 200 — HypothesisObject**: 조건 배열, 확신도, 기분, 예상 보유 기간, `is_active_for_stock`

### 15.8 `PATCH /hypotheses/{hypothesis_id}`

빈칸 나중 채우기 (PRD 4.3). 논리·목표·보유 기간만 수정 가능하며, **이미 도달한 조건은 수정할 수 없다.**

**오류**: `CONDITION_ALREADY_TRIGGERED`(409)

### 15.9 `GET /cycles/{cycle_id}/hypotheses`

사이클 내 전체 가설 기록.

**쿼리**: `stock_id`, `is_active`, `cursor`

---

## 16. 조건 도달

### 16.1 `GET /conditions`

내 활성 조건 목록.

**쿼리**: `cycle_id`, `status`, `evaluation_mode`, `stock_id`

**응답 200**: ConditionObject 배열 + 각 항목의 `hypothesis_ref`, `stock`

### 16.2 `POST /conditions/{condition_id}/self-report`

자기 보고 조건의 도달 표시 (PRD 7.5).

**요청**: `reported_on`(선택, 기본 오늘), `note`(선택)

**응답 201**: ConditionTriggerObject + `plan_execution`(PLAN 계좌 집행 결과)

**오류**: `SELF_REPORT_NOT_ALLOWED`(422) — `AUTO` 조건

### 16.3 `PUT /conditions/{condition_id}/replan`

조건 재설정 (PRD 9.8-3). 기존 조건은 `SUPERSEDED`, 새 조건이 활성화된다.

**요청**: 조건 객체 (§15.3과 동일 구조)

**응답 200**: 새 ConditionObject + `previous_condition_id`

### 16.4 `GET /condition-triggers`

조건 도달 이벤트 목록.

**쿼리**: `cycle_id`, `user_response`, `from`, `to`, `cursor`

**응답 200 — ConditionTriggerObject 배열**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `trigger_id` | number | |
| `condition` | ConditionObject | |
| `stock` | StockSummary | |
| `triggered_on` | string | |
| `trigger_price` | string | |
| `detection_mode` | string | |
| `recall` | RecallBlock | **"3월 12일에 이렇게 정하셨습니다"** 형식 (PRD 9.8) |
| `principles` | array\<RecallBlock\> | `ON_CONDITION_HIT` 수칙 (PRD 11.5) |
| `user_response` | string | |
| `available_responses` | array | `[{key, label}]` — 3가지 모두 동등하게 제시 |
| `plan_execution` | object \| null | PLAN 계좌가 무엇을 했는지 |

> `available_responses`의 3개 항목에 **권장 표시나 순서 강조가 없다.** PRD 9.8의 "셋 다 정당하다"를 API가 보장한다.

### 16.5 `GET /condition-triggers/{trigger_id}`

단건 상세. 알림 딥링크의 착지 화면이 사용한다.

**응답 200**: §16.4의 ConditionTriggerObject + 아래 추가 필드

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `stock_price_context` | object | 판정일 이후 가격 추이 (판단 참고용 사실) |
| `related_records` | array\<RecallBlock\> | 이 종목의 과거 기록 (PRD 10.4-3) |
| `position` | object | 현재 보유 수량·평가손익 |

### 16.6 `POST /condition-triggers/{trigger_id}/respond`

**요청**: `response`(`FOLLOWED`/`DEVIATED`/`REPLANNED`), `note`(선택), `new_condition`(REPLANNED 시)

**응답 200**: 갱신된 트리거 + `adherence`(갱신된 내 계획 지킨 비율)

**응답에 평가 문구가 없다.** PRD 9.8의 "앱은 아무런 부정적 반응을 보이지 않는다"를 지킨다.

---

## 17. 회고

### 17.1 `GET /trade-reviews/context`

전량 매도 회고 화면 진입 시 필요한 것 (PRD 9.7).

**쿼리**: `trade_id`

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `trade` | TradeObject | 매도 건 |
| `original_hypothesis` | HypothesisObject | **그대로 다시 띄운다** |
| `holding_summary` | object | 보유 기간, 실현손익, 조건 도달 이력 |
| `questions` | array | 4문항 정의와 선택지 |
| `past_records` | array\<RecallBlock\> | 관련 과거 기록 (PRD 10.4-3) |

### 17.2 `POST /trade-reviews`

**요청**

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| `trade_id` | number | Y | |
| `q1_logic_correct` | string | N | `CORRECT`/`WRONG`/`UNKNOWN` |
| `q2_followed_plan` | boolean | N | |
| `q2_deviation_reason` | string | N | 선택지 키 |
| `q3_result_attribution` | string | N | `ORIGINAL_LOGIC`/`OTHER_REASON`/`LUCK` |
| `q4_lesson_text` | string | N | 배운 것 한 줄 |

**전 항목이 선택이다** (PRD 4.3의 빈칸 허용). 단 `q4_lesson_text`가 있으면 `insight`가 자동 생성된다 (PRD 9.7).

**응답 201**: ReviewObject + `created_insight_id`

### 17.3 `PATCH /trade-reviews/{review_id}`

나중에 채우기.

### 17.4 부분 매도 후 새 조건 설정

PRD 9.7의 "남은 물량에 새 조건을 세우시겠어요?"는 **새 가설을 만들지 않는다.** 매도 건에는 가설 기록이 붙지 않기 때문이다. 처리 경로는 아래와 같다.

| 상황 | 사용 엔드포인트 |
| --- | --- |
| 기존 활성 조건을 교체 | `PUT /conditions/{condition_id}/replan` (§16.3) |
| 활성 조건이 없는 상태 (조건 없이 매수했던 종목) | `POST /hypotheses/{hypothesis_id}/conditions` |
| 건너뛰기 | 요청 없음. 잔여 물량은 결산일까지 보유 (PRD 7.5) |

`POST /trades` 응답의 `next_action`에 어느 경로를 쓸지 함께 내려보낸다: `SET_NEW_CONDITION_REPLAN` 또는 `SET_NEW_CONDITION_CREATE`. 클라이언트가 활성 조건 존재 여부를 판단하지 않는다.

### 17.5 `POST /hypotheses/{hypothesis_id}/conditions`

활성 조건이 없는 가설에 조건을 추가한다. 조건 없이 매수한 종목에 나중에 조건을 세우는 경로이며, PRD 4.3의 "빈칸을 나중에 채울 수 있어야 한다"에 해당한다.

**요청**: 조건 객체 (§15.3과 동일 구조)

**응답 201**: ConditionObject + `plan_track_notice` — 이 조건이 언제부터 '내 계획대로 계좌'에 반영되는지 안내(설정일 이후 발생분부터)

**오류**: `CONDITION_ALREADY_ACTIVE`(409) — 이미 활성 조건이 있으면 `replan`을 써야 한다

---

## 18. 일지·깨달은 것

### 18.1 `GET /journals`

**쿼리**: `entry_type`, `from`, `to`, `cursor`

**응답 200 — JournalEntryObject 배열**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `journal_id` | number | |
| `entry_type` | string | |
| `period_start_date` / `period_end_date` | string | |
| `items` | array | `[{position, content, related_stock, insight_id}]` |
| `is_backfilled` | boolean | |
| `market_context` | object \| null | 월간만 |
| `completed_at` | string | |

### 18.2 `GET /journals/writable-dates`

작성 가능한 날짜 (소급 14일 이내, PRD 10.2).

**응답 200**: `{daily: [{date, is_written}], weekly: [...], monthly: [...], backfill_limit_date}`

### 18.3 `POST /journals`

**요청**

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| `entry_type` | string | Y | |
| `period_start_date` | string | Y | |
| `items` | array | Y | 최대 3개. **1개만 있어도 됨** (PRD 10.1) |
| `items[].content` | string | Y | |
| `items[].related_stock_id` | number | N | |
| `items[].mark_as_insight` | boolean | N | 별표 (PRD 10.3) |
| `items[].insight_tag` | string | N | 미지정 시 `UNCLASSIFIED` |

**응답 201**: JournalEntryObject + `streak`(갱신된 연속일) + `created_insight_ids`

**오류**: `JOURNAL_BACKFILL_EXPIRED`(422), `JOURNAL_ALREADY_EXISTS`(409)

> 응답의 `streak`에 **끊김에 대한 필드가 없다.** `current_streak_days`만 반환하며 "며칠 만이다" 같은 정보를 내려보내지 않는다 (PRD 10.5).

### 18.4 `PATCH /journals/{journal_id}`

### 18.5 `GET /journals/streak`

**응답 200**: `{current_streak_days, longest_streak_days, total_entry_days, stamps: [{date, has_entry}]}`

**쿼리**: `year_month` — 스탬프 달력용

### 18.6 `GET /insights`

**쿼리**: `tag`, `stock_id`, `cycle_scope`(기본 `ALL`), `source_type`, `cursor`

**응답 200 — InsightObject 배열**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `insight_id` | number | |
| `content` | string | |
| `tag` | string | |
| `tag_label` | string | |
| `source_type` | string | |
| `source_ref` | object | |
| `related_stock` | StockSummary \| null | |
| `cycle_year` | number \| null | |
| `marked_at` | string | |
| `is_used_as_principle` | boolean | |

### 18.7 `POST /insights`

일지 항목·회고·연습 결과를 사후에 '깨달은 것'으로 표시. **소급 제한 없음** (PRD 10.2).

**요청**: `source_type`, `source_id`, `content`(선택, 미지정 시 원문 복사), `tag`, `related_stock_id`

### 18.8 `PATCH /insights/{insight_id}`

태그 변경, 내용 수정.

### 18.9 `DELETE /insights/{insight_id}`

표시 해제. **원본 일지 항목은 남는다.**

### 18.10 `GET /insights/tags`

태그 목록과 각 태그별 개수 (주제별 모아보기 진입점, PRD 10.4-2).

**응답 200**: `[{tag_key, label, count}]`

---

## 19. 투자 수칙

### 19.1 `GET /principles`

**쿼리**: `status`(기본 `ACTIVE`), `cycle_id`, `trigger_point`

**응답 200 — PrincipleObject 배열**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `principle_id` | number | |
| `content` | string | |
| `status` | string | |
| `display_order` | number | |
| `trigger_points` | array | `[{key, label}]` |
| `created_cycle_year` | number \| null | |
| `revised_from` | object \| null | 이전 문장 (PRD 19.2) |
| `source` | object \| null | `{type: "INSIGHT"/"TEMPLATE", ref}` |
| `scorecard` | object \| null | §19.6 |

### 19.2 `GET /principles/candidates`

수칙 후보 (PRD 11.4). **문장 틀 방식** (PRD 18.7).

**쿼리**: `cycle_id`

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `from_insights` | array | `[{insight_id, content, tag, suggested_text}]` |
| `from_patterns` | array | 아래 |
| `symmetry_check` | object | `{is_balanced, missing_counterparts}` |

**`from_patterns` 구조**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `template_key` | string | |
| `suggested_text` | string | 치환 완료된 문장 |
| `evidence` | object | `{metric_key, sample_count, value}` |
| `counterpart_template_key` | string | 대칭 짝 |
| `default_trigger_points` | array | |
| `is_displayable` | boolean | 표본 3건 미만이면 false |

> `symmetry_check.is_balanced=false`인 응답은 서버 내부 검증 실패로 취급하며 로깅한다. 한쪽 방향 후보만 제시되면 그 자체가 특정 매매 행위를 권하는 장치다 (PRD 11.4).

### 19.3 `POST /principles`

**요청**

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| `content` | string | Y | 최대 120자 |
| `trigger_points` | array | Y | 최소 1개 |
| `applies_to_cycle_id` | number | N | 미지정 시 다음 사이클 |
| `source_insight_id` / `source_template_id` | number | N | |
| `revised_from_principle_id` | number | N | 수정 시 |

**응답 201**: PrincipleObject

**오류**: `PRINCIPLE_LIMIT_EXCEEDED`(422) — 5개 초과

**온보딩 임시 수칙의 처리** (PRD 11.4의 "첫 사이클 전에도 하나는 만들 수 있게 한다")

| 항목 | 처리 |
| --- | --- |
| 생성 시점 | 온보딩 5단계. 아직 사이클이 없다 |
| 상태 | `DRAFT`. `created_cycle_id`는 null |
| 적용 | 사이클이 `ACTIVE`가 되는 시점에 `ACTIVE`로 전이하고 `applies_to_cycle_id`가 채워진다 |
| 개수 제한 | 임시 수칙은 1개. 이후 결산에서 5개까지 확정 |
| 노출 | 온보딩 8단계(첫 가설 기록)에서 즉시 나타난다 — "내가 정한 규칙이 실제로 다시 나타난다"는 경험 (PRD 14.2) |

### 19.4 `PATCH /principles/{principle_id}` / `DELETE /principles/{principle_id}`

수정은 `REVISED` 상태 전이 + 새 레코드 생성. 삭제는 `RETIRED` 전이(물리 삭제 아님).

### 19.5 `POST /principles/{principle_id}/encounters/{encounter_id}/mark`

수칙을 마주쳤을 때 지켰는지 탭 표시 (PRD 18.9).

**요청**: `kept`(boolean)

**응답 200**: 갱신된 encounter

### 19.6 `GET /principles/{principle_id}/scorecard`

수칙 성적표 (PRD 11.6).

**쿼리**: `cycle_id`

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `encounter_count` | number | 마주친 횟수 |
| `marked_count` | number | 표시된 횟수 |
| `kept` | RateMetric | 지킨 비율. **분모는 `marked_count`** |
| `result_when_kept` | PerformanceValue \| null | |
| `result_when_not_kept` | PerformanceValue \| null | |
| `is_verdict_recommended` | boolean | 3회 미만이면 false |
| `verdict_notice` | Notice \| null | `"마주친 횟수가 적어 판단을 보류하시길 권합니다"` |
| `unmarked_notice` | Notice \| null | `"표시된 것만으로 계산했습니다"` (PRD 18.9) |

> `result_when_kept`가 `result_when_not_kept`보다 나쁠 수 있으며 **그대로 반환한다** (PRD 11.6의 정직성).

### 19.7 `POST /principles/{principle_id}/verdict`

결산 시 유지·수정·폐기 결정.

**요청**: `verdict`(`KEEP`/`REVISE`/`RETIRE`/`DEFERRED`), `revised_content`(REVISE 시)

### 19.8 `GET /principles/encounters`

수칙 마주침 이력.

**쿼리**: `cycle_id`, `principle_id`, `kept`(`true`/`false`/`null`)

---

## 20. 예측·비상 선언

### 20.1 `GET /cycles/{cycle_id}/predictions`

**응답 200**: `[{prediction_id, prediction_type, sequence, question_text, options, answer, predicted_at, actual_result, is_correct, is_answerable}]`

`is_taster` 사이클은 `CYCLE_WINNER`만 반환된다 (PRD 9.9).

### 20.2 `POST /cycles/{cycle_id}/predictions`

**요청**: `prediction_type`, `sequence`, `answer`

### 20.3 `GET /cycles/{cycle_id}/emergency-declarations`

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `items` | array | 선언 목록 |
| `remaining_count` | number | 남은 횟수 (2 - 사용) |
| `limit_statement` | StatementBlock | `"비상 선언은 한 사이클에 2번까지 할 수 있습니다."` |

**항목 구조**: `declared_at`, `reason_text`, `account_snapshot`(4계좌), `market_snapshot`, `as_of_date`

### 20.4 `POST /cycles/{cycle_id}/emergency-declarations`

**요청**: `reason_text`(**필수**, PRD 4.3)

**응답 201**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `declaration` | object | |
| `account_snapshot` | array\<AccountSummary\> | 저장된 시점 상태 |
| `no_account_change_statement` | StatementBlock | `"비상 선언은 계좌를 움직이지 않습니다."` (PRD 8.7) |

**오류**: `EMERGENCY_LIMIT_EXCEEDED`(422), `EMERGENCY_REASON_REQUIRED`(400)

> 응답에 계좌 변경 필드나 후속 행동 유도 필드가 없다. 이후 대응은 자유 투자 트랙에서 사용자가 알아서 한다.

---

## 21. 배지

### 21.1 `GET /badges`

**쿼리**: `cycle_scope`(기본 `ALL`), `category`

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `is_enabled` | boolean | 사용자가 배지를 껐는가 (PRD 7.10) |
| `items` | array | 아래 |

**항목 구조**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `badge_key` | string | |
| `name` | string | 자조적 이름 |
| `description` | string | **획득 조건 사전 공개** |
| `category` | string | |
| `counterpart_badge_key` | string \| null | 대칭 짝 |
| `count_value` | number | |
| `is_earned` | boolean | |
| `first_earned_at` | string \| null | |
| `icon_key` | string | |

**미획득 배지도 조건과 함께 반환한다** (PRD 7.10의 "숨겨진 조건은 두지 않습니다").

### 21.2 `GET /badges/definitions`

전체 배지 도감. 사용자 데이터 없이 조건만.

---

## 22. 연습 모드

### 22.1 `POST /practice/sessions`

새 판 시작. 시나리오는 서버가 선택한다 (미플레이 우선, `PLAN_LOSES` 포함 보장).

**요청**: `is_onboarding`(선택)

**응답 201**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `session_id` | number | |
| `masked_label` | string | `"A 기업"` |
| `total_days` | number | |
| `initial_chart` | array | **선언 전에 보여줄 구간만** (PRD 12.2-1) |
| `condition_catalog` | array | 조건 선택지 |
| `stage` | string | `AWAITING_PLAN` |

**종목명·날짜 필드가 이 응답에 존재하지 않는다.**

### 22.2 `POST /practice/sessions/{session_id}/plan`

사전 선언 (PRD 12.2-2). **이후 흐름을 보기 전에 반드시 완료해야 한다.**

**요청**: `declared_target`(선택), `declared_condition`(필수: `condition_key`/`custom_text`, `params`, `planned_action`)

**응답 200**: `stage: "PLAYING"`, `current_day_index: 0`

**오류**: `PRACTICE_PLAN_REQUIRED`(422)

### 22.3 `POST /practice/sessions/{session_id}/decisions`

하루치 결정.

**요청**: `day_index`, `decision`(`HOLD`/`SELL`)

**응답 200**: `{next_day: {day_index, ohlc}, is_completed, current_state}`

**오류**: `PRACTICE_SESSION_COMPLETED`(409)

### 22.4 `GET /practice/sessions/{session_id}/result`

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `comparison` | object | `{plan_result_rate, actual_result_rate}` — **이 화면 전용 대조값** (PRD 12.2-4) |
| `adherence_rate` | string | 미리 정한 대로 한 비율 |
| `reveal` | object | 종목명·기간 공개 (PRD 12.2-5) |
| `decisions` | array | 일자별 결정과 선언 일치 여부 |
| `share_payload` | object | 공유 이미지용. **`adherence_rate`만 포함, 수익률 제외** (PRD 12.4) |
| `insight_prompt` | object | 한 줄 적기 유도 (PRD 12.3-6) |

### 22.5 `GET /practice/stats`

**응답 200**: `{play_count, adherence_rate: RateMetric}`

> **수익률 필드가 없다** (PRD 12.4). §22.4의 `comparison`은 결과 화면 1회성 대조이고, 누적 통계에는 수익률이 존재하지 않는다.

---

## 23. 결산·리포트

### 23.1 `GET /cycles/{cycle_id}/settlement`

결산 진행 상태 (PRD 13.1의 6단계).

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `settlement_status` | string | `PENDING`/`LIQUIDATED`/`REPORT_READY`/`COMPLETED` |
| `cycle_status` | string | `CycleStatus` (`SETTLING`/`CLOSED`). **결산 진행 상태와 사이클 상태는 다른 축이다** |
| `settlement_date` | string | |
| `steps` | array | `[{step, key, is_completed}]` — 6단계 (PRD 13.1) |
| `liquidation_summary` | object \| null | 전 종목 매도 결과 |
| `next_cycle_notice` | StatementBlock | 다이어리 은유 안내 (PRD 13.1) |

**6단계 매핑**

| step | key | 엔드포인트 |
| --- | --- | --- |
| 1 | `LIQUIDATION` | 자동 (배치) |
| 2 | `FINAL_PERFORMANCE` | §13.3 |
| 3 | `RETROSPECTIVE` | §23.3 |
| 4 | `REPORT` | §23.4 |
| 5 | `PRINCIPLE_SETUP` | §19.3 |
| 6 | `NEXT_CYCLE_NOTICE` | 안내만 |

### 23.2 `GET /cycles/{cycle_id}/retrospective`

**사이클 회고** (PRD 13.1의 결산 3단계). 표준 포맷, 선택형 위주.

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `questions` | array | `[{question_key, prompt, input_type, options, auto_hint}]` |
| `answers` | array \| null | 기존 응답 |
| `completed_at` | string \| null | |
| `is_report_unlocked` | boolean | 회고 완료 여부와 무관하게 항상 true |

`auto_hint`는 자동 집계로 얻은 사실을 문항 옆에 제시하는 값이다(예: 가장 자주 고른 기분). **답을 제안하지 않고 사실만 보여준다** (PRD 17.1).

### 23.3 `POST /cycles/{cycle_id}/retrospective`

**요청**: `answers: [{question_key, choice_key, free_text}]`, `free_note`

**응답 201**: 저장된 회고

**오류**: `RETROSPECTIVE_AFTER_REPORT_VIEWED`(409) — 리포트를 이미 열람한 뒤의 최초 작성

> **회고는 리포트보다 먼저 써야 한다.** 숫자를 본 뒤에 쓴 회고는 그 숫자에 맞춰 기억이 재구성된 것이며, PRD 1.2가 지목한 문제를 제품이 스스로 만드는 셈이 된다. 다만 **회고를 건너뛰어도 리포트는 볼 수 있다** — 강제하면 결산이 막히고, 결산이 막히면 다음 사이클도 막힌다 (PRD 8.6). 순서는 권장하되 통과 조건으로 두지 않는다.

### 23.4 `GET /cycles/{cycle_id}/report`

올해의 투자 레슨 (PRD 11.3).

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `report_id` | number | |
| `report_mode` | string | `FULL`/`SUMMARY`/`NO_DATA` (PRD 13.2) |
| `mode_notice` | Notice \| null | `"이번 사이클은 ○개월이라 비교가 이릅니다"` |
| `generated_at` | string | |
| `sections` | object | 아래 7개 |
| `user_finalized_at` | string \| null | |

**`sections` 구조**

| 키 | 내용 |
| --- | --- |
| `numbers` | 4계좌 성과, 세 개의 차이, `adherence.plan`·`adherence.rule_confirm`(각각 RateMetric), `period_months`, `self_report_ratio`, 측정 편향 Notice |
| `my_notes` | 태그별 insight 묶음 |
| `patterns` | 기분별·확신도별 패턴. 각 항목에 `is_displayable`·`sample_count` |
| `predictions` | 예측 vs 실제 |
| `emergency` | 선언 시점 기록 + **동기간 RULE·HOLD 계좌 추이** (PRD 11.3-5) |
| `principle_scorecard` | 지난 사이클 수칙 성적표. 첫 사이클이면 null |
| `best_worst_candidates` | 자동 추린 후보. **선택은 사용자** (PRD 11.3-7) |

**오류**: `REPORT_NOT_GENERATED`(404)

### 23.5 `POST /cycles/{cycle_id}/report/decisions`

가장 잘한/아쉬운 판단 선택.

**요청**: `best_decision_ref`, `worst_decision_ref` (`{type, id}`)

### 23.6 `POST /cycles/{cycle_id}/settlement/followup`

"실제 계좌에서도 정리하셨나요?" (PRD 8.6).

**요청**: `answer`(`YES`/`NO`/`NO_REAL_ACCOUNT`)

**응답 200**. **답하지 않아도 결산은 완료된다.**

### 23.7 `GET /reports`

지난 사이클 리포트 목록. 다음 사이클 중에도 언제든 열람 가능 (PRD 11.3).

**응답 200**: `[{cycle_id, year, report_mode, generated_at, summary}]`

---

## 24. 알림

### 24.1 `GET /notifications`

**쿼리**: `is_read`, `notification_type`, `cursor`

**응답 200**: `[{notification_id, notification_type, title, body, payload, resource_ref, sent_at, read_at}]`

`SUPPRESSED` 상태 알림도 인앱 목록에는 나타난다 (PRD 10.6).

### 24.2 `POST /notifications/{id}/read` / `POST /notifications/read-all`

### 24.3 `GET /notifications/unread-count`

**응답 200**: `{total, by_type: {...}}`

### 24.4 `POST /notifications/devices`

푸시 토큰 등록.

**요청**: `push_token`, `platform`

### 24.5 `DELETE /notifications/devices/{device_id}`

---

## 25. 내보내기·탈퇴

### 25.1 `POST /exports`

**요청**: `target`(`CYCLE_REPORT`/`TRADES`/`JOURNALS`/`PRINCIPLES`/`ALL`), `cycle_id`(선택), `format`

**응답 202**: `{export_job_id, status: "QUEUED"}`

### 25.2 `GET /exports/{export_job_id}`

**응답 200**: `{status, download_url, expires_at, file_size}`

### 25.3 `POST /me/withdrawal`

탈퇴 요청.

**요청**: `confirm: true`, `export_offered_response`(`EXPORTED`/`DECLINED`)

**응답 200**: `{withdrawn_at, grace_period_days: 7, permanent_deletion_at}`

> 요청 전 클라이언트는 §25.1의 내보내기를 반드시 한 번 권유해야 하며, 그 응답을 `export_offered_response`로 보낸다 (PRD 13.5).

### 25.4 `DELETE /me/withdrawal`

유예 기간 내 탈퇴 취소.

---

## 26. 내부 배치 API

Lambda 스케줄러가 호출하며 외부에 노출하지 않는다. 인증은 IAM + 내부 시크릿 헤더.

| 엔드포인트 | 실행 시각(KST) | 설명 |
| --- | --- | --- |
| `POST /internal/ingest/daily-price` | 18:00 | 종목 풀 30개 시세 수집 |
| `POST /internal/ingest/disclosure` | 18:10 | 공시 발생 목록·링크 수집 |
| `POST /internal/ingest/market` | 18:10 | 지수·수급 |
| `POST /internal/ingest/earnings-schedule` | 주 1회 월 07:00 | 실적 발표 일정 갱신 |
| `POST /internal/engine/evaluate-conditions` | 18:30 | `AUTO` 조건 종가 판정 → `condition_trigger` 생성 → PLAN 계좌 집행 |
| `POST /internal/engine/rebalance` | 18:40 | 조정일에만. RULE 계좌 실행 |
| `POST /internal/engine/revalue-accounts` | 18:50 | 4계좌 평가·스냅샷·지표 갱신 |
| `POST /internal/engine/settlement` | 결산일 19:00 | 전량 매도·리포트 생성 |
| `POST /internal/notify/dispatch` | 19:30 | 우선순위 적용 후 하루 1건 발송 |
| `POST /internal/notify/periodic-review` | 토 09:00 / 월초 09:00 | 주간·월간 회고 알림 예약 |
| `POST /internal/notify/next-cycle-open` | 종목 풀 공개일 10:00 | 다음 사이클 개시 안내 (PRD 18.6) |
| `POST /internal/jobs/recompute` | 5분 주기 + SQS 이벤트 | 대기 중 재계산 작업 처리 |
| `POST /internal/jobs/housekeeping` | 03:00 | 휴면 판정, 탈퇴 확정, 만료 파일·멱등성 레코드 정리 |

> 재계산은 **SQS 이벤트 구동이 기본**이고 5분 주기 실행은 유실 대비 스윕이다. 두 경로가 같은 작업을 집어도 `recompute_job`의 사이클 단위 락으로 중복 실행이 방지된다.

**공통 요청**: `target_date`(선택, 기본 오늘), `dry_run`(boolean), `cycle_ids`(선택, 부분 재실행)

**공통 응답**: `{processed_count, skipped_count, failed_count, duration_ms, details}`

**멱등성**: 모든 배치는 같은 `target_date`로 재실행해도 결과가 같아야 한다. `ingestion_run`의 유니크 제약이 중복 수집을 막고, 엔진 배치는 대상 구간 파생 데이터를 삭제 후 재생성한다.

**실행 순서 의존**: 시세 수집 → 조건 판정 → 조정 → 평가 → 알림. 앞 단계 실패 시 뒤 단계는 실행하지 않고 `SKIPPED`로 기록한다. 조건 판정이 어제 시세로 실행되면 사용자에게 잘못된 도달 알림이 간다.

---

## 27. 비기능 요구

### 27.1 성능 목표

| 대상 | 목표 |
| --- | --- |
| 조회 API p95 | 400ms |
| 쓰기 API p95 | 700ms |
| 첫 화면 필요 호출 수 | 3회 이하 (`/auth/session`, `/cycles/current`, `/cycles/{id}/accounts`) |
| 배치 완료 | 각 20분 이내 |

### 27.2 요청 한도

| 대상 | 한도 |
| --- | --- |
| 인증된 사용자 | 300 req/min |
| 로그인 시도 | 10 req/min/IP |
| 쓰기 API | 60 req/min |
| 내보내기 | 5 req/hour |

초과 시 429 + `Retry-After`.

### 27.3 캐시

| 대상 | 정책 |
| --- | --- |
| 종목 풀, 배지 정의, 조건 카탈로그, 태그 목록 | `Cache-Control: public, max-age=3600` + ETag |
| 시세·시장 요약 | `max-age=300`. 배치 완료 시점에 무효화 |
| 사용자 데이터 | `no-store` |

### 27.4 관측성

모든 응답에 `request_id`를 부여하고 구조화 로그에 기록한다. 오류 응답은 `code`별로 집계하여 특정 도메인 오류가 급증하면 경보한다 — 1~3인 운영에서는 사용자 문의보다 지표가 먼저 문제를 알려야 한다 (PRD 3.5).

### 27.5 API 변경 정책

| 변경 유형 | 처리 |
| --- | --- |
| 필드 추가 | 즉시 가능 (클라이언트는 미지 필드 무시) |
| 필드 삭제·타입 변경 | `/v2` 신설 |
| 열거값 추가 | 가능. 클라이언트는 미지 값에 대한 기본 처리를 갖춰야 한다 |
| 오류 코드 추가 | 가능. 클라이언트는 미지 코드를 일반 오류로 처리 |
| 기본 정렬·페이지 크기 변경 | 파괴 변경으로 간주 |
