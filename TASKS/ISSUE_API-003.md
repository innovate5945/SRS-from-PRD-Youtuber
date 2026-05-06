---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] API-003: Trades Query DTO 정의 (GET /trades)"
labels: 'feature, backend, api-contract, priority:high, sprint:0'
assignees: ''
---

## :dart: Summary
- **기능명:** [API-003] Trades Query DTO 정의 (GET /trades — 필터, 페이징, Response 구조)
- **목적:** 매매 내역 통합 조회 API의 Request(쿼리 파라미터) 및 Response 구조를 TypeScript DTO로 정의한다. 필터링(기간, 종목, 매수/매도), 페이징(offset/limit), 정렬 파라미터를 표준화하여 QRY-CSV-001과 UI-CSV-001이 참조할 SSOT를 확립한다.

## :link: References (Spec & Context)
> :bulb: AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`/SRS_v1.md#6.1-API-#4`](../05_SRS_v1.md) — `GET /api/v1/trades` (Bearer 인증)
- SRS 문서: [`/SRS_v1.md#3.3-actions/trades.ts`](../05_SRS_v1.md) — 매매 내역 통합 조회 모듈
- SRS 문서: [`/SRS_v1.md#REQ-FUNC-018`](../05_SRS_v1.md) — 전체/기간별 매매 통계 시각화
- SRS ERD: [`/SRS_v1.md#6.2-TRADE`](../05_SRS_v1.md) — TRADE 엔티티 구조
- 태스크 분해: [`/TASKS/06_MVP_Task_Breakdown.md#API-003`](./06_MVP_Task_Breakdown.md)

## :white_check_mark: Task Breakdown (실행 계획)

### Phase A: Query Parameter DTO 정의
- [ ] `src/types/trades-query.dto.ts` 파일 생성
- [ ] 조회 필터 파라미터 정의:
  ```typescript
  /** 매매 내역 조회 필터 */
  export interface TradeQueryParams {
    /** 조회 시작일 (ISO 8601, inclusive) */
    start_date?: string;
    /** 조회 종료일 (ISO 8601, inclusive) */
    end_date?: string;
    /** 종목 코드 필터 */
    ticker?: string;
    /** 매수/매도 필터 */
    side?: 'buy' | 'sell';
    /** 계좌 ID 필터 */
    account_id?: string;
    /** 정렬 기준 */
    sort_by?: TradeSortField;
    /** 정렬 방향 */
    sort_order?: 'asc' | 'desc';
    /** 페이지 번호 (1-indexed) */
    page?: number;
    /** 페이지 당 항목 수 (기본 20, 최대 100) */
    page_size?: number;
  }

  export type TradeSortField = 'executed_at' | 'ticker' | 'price' | 'quantity' | 'created_at';
  ```

### Phase B: Response DTO 정의
- [ ] 단일 매매 내역 항목 DTO:
  ```typescript
  /** 매매 내역 단일 항목 */
  export interface TradeItem {
    id: string;
    account_id: string;
    ticker: string;
    ticker_name?: string;
    side: 'buy' | 'sell';
    price: number;
    quantity: number;
    /** 매매 금액 (price × quantity) */
    amount: number;
    executed_at: string;
    created_at: string;
  }
  ```
- [ ] 페이지네이션 응답 DTO:
  ```typescript
  /** 페이지네이션 메타데이터 */
  export interface PaginationMeta {
    current_page: number;
    page_size: number;
    total_items: number;
    total_pages: number;
    has_next: boolean;
    has_prev: boolean;
  }

  /** 매매 내역 조회 성공 응답 */
  export interface TradeListSuccessResponse {
    success: true;
    data: {
      trades: TradeItem[];
      pagination: PaginationMeta;
    };
  }

  /** 매매 내역 조회 에러 응답 */
  export interface TradeListErrorResponse {
    success: false;
    error: {
      code: TradeQueryErrorCode;
      message: string;
    };
  }

  export type TradeQueryErrorCode =
    | 'INVALID_DATE_RANGE'   // 시작일 > 종료일
    | 'INVALID_PAGE'         // 유효하지 않은 페이지 번호
    | 'UNAUTHORIZED'         // 인증 실패
    | 'QUERY_ERROR';         // DB 쿼리 오류
  ```

### Phase C: 페이징 상수 및 Zod 스키마
- [ ] `src/types/trades-query.schema.ts` 파일 생성:
  ```typescript
  import { z } from 'zod';

  export const PAGINATION_DEFAULTS = {
    DEFAULT_PAGE: 1,
    DEFAULT_PAGE_SIZE: 20,
    MAX_PAGE_SIZE: 100,
  } as const;

  export const tradeQuerySchema = z.object({
    start_date: z.string().datetime().optional(),
    end_date: z.string().datetime().optional(),
    ticker: z.string().max(20).optional(),
    side: z.enum(['buy', 'sell']).optional(),
    account_id: z.string().uuid().optional(),
    sort_by: z.enum(['executed_at', 'ticker', 'price', 'quantity', 'created_at']).default('executed_at'),
    sort_order: z.enum(['asc', 'desc']).default('desc'),
    page: z.coerce.number().int().min(1).default(PAGINATION_DEFAULTS.DEFAULT_PAGE),
    page_size: z.coerce.number().int().min(1).max(PAGINATION_DEFAULTS.MAX_PAGE_SIZE)
      .default(PAGINATION_DEFAULTS.DEFAULT_PAGE_SIZE),
  }).refine(
    (data) => {
      if (data.start_date && data.end_date) {
        return new Date(data.start_date) <= new Date(data.end_date);
      }
      return true;
    },
    { message: '시작일은 종료일 이전이어야 합니다.', path: ['start_date'] }
  );
  ```

## :test_tube: Acceptance Criteria (BDD/GWT)

**Scenario 1: DTO 타입 Import 정상 동작**
- **Given:** `src/types/trades-query.dto.ts`가 작성됨
- **When:** 다른 파일에서 `import { TradeQueryParams, TradeListSuccessResponse }` 수행
- **Then:** TypeScript 컴파일 에러 없이 참조되어야 한다.

**Scenario 2: tradeQuerySchema — 유효한 파라미터 검증**
- **Given:** start_date: '2026-01-01T00:00:00Z', page: 1, page_size: 20
- **When:** `tradeQuerySchema.safeParse()` 실행
- **Then:** `success: true` 반환.

**Scenario 3: tradeQuerySchema — 잘못된 날짜 범위 거부**
- **Given:** start_date: '2026-12-31', end_date: '2026-01-01' (시작 > 종료)
- **When:** `tradeQuerySchema.safeParse()` 실행
- **Then:** `success: false`, '시작일은 종료일 이전이어야 합니다.' 메시지.

**Scenario 4: tradeQuerySchema — page_size 초과 거부**
- **Given:** page_size: 500 (최대 100 초과)
- **When:** `tradeQuerySchema.safeParse()` 실행
- **Then:** `success: false`.

**Scenario 5: 기본값 적용 확인**
- **Given:** 파라미터 없이 빈 객체가 주어짐
- **When:** `tradeQuerySchema.safeParse({})` 실행
- **Then:** sort_by: 'executed_at', sort_order: 'desc', page: 1, page_size: 20으로 기본값 적용.

## :gear: Technical & Non-Functional Constraints
- **타입 안전:** TypeScript strict 모드 에러 없음
- **런타임 검증:** Zod 스키마로 쿼리 파라미터 검증
- **성능:** 대시보드 API p95 ≤ 500ms — `REQ-NF-001`
- **페이징:** offset 기반 페이징 (MVP), 향후 cursor 기반 전환 가능
- **국제화:** 에러 메시지 한국어

## :checkered_flag: Definition of Done (DoD)
- [ ] `src/types/trades-query.dto.ts` 생성 및 모든 DTO export
- [ ] `src/types/trades-query.schema.ts` Zod 스키마 동작
- [ ] PAGINATION_DEFAULTS 상수 정의
- [ ] `npm run build` 타입 에러 없음

## :construction: Dependencies & Blockers
- **Depends on:**
  - `INFRA-001` (프로젝트 초기화)
- **Blocks:**
  - `MOCK-003` (매매 내역 Mock 데이터)
  - `QRY-CSV-001` (매매 내역 통합 조회 Server Action)
  - `UI-CSV-001` (매매 내역 표시 UI)
