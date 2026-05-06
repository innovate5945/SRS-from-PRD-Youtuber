---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] API-005: Reviews DTO 정의 (GET /reviews, POST /reviews/generate)"
labels: 'feature, backend, api-contract, priority:high, sprint:0'
assignees: ''
---

## :dart: Summary
- **기능명:** [API-005] Reviews DTO 정의 (GET /reviews, POST /reviews/generate — Gemini JSON 스키마)
- **목적:** F3(자동 매매 복기 일지) API의 Request/Response 구조를 TypeScript DTO로 정의한다. Gemini API가 반환하는 구조화된 인사이트 JSON의 타입 스키마를 확립하고, 복기 일지 목록 조회/상세 조회/생성 트리거의 DTO와 Zod 검증 스키마를 제공하여 CMD-REVIEW, QRY-REVIEW, UI-REVIEW 태스크의 SSOT를 구축한다.

## :link: References (Spec & Context)
> :bulb: AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`/SRS_v1.md#6.1-API-#7~8`](../05_SRS_v1.md) — `GET /reviews`, `POST /reviews/generate`
- SRS 문서: [`/SRS_v1.md#3.3-actions/reviews.ts`](../05_SRS_v1.md) — Gemini API 기반 복기 일지 생성 모듈
- SRS 문서: [`/SRS_v1.md#REQ-FUNC-013`](../05_SRS_v1.md) — Gemini 프롬프트 추론 → 구조화된 일지 저장
- SRS 문서: [`/SRS_v1.md#REQ-FUNC-014`](../05_SRS_v1.md) — 3분 내 핵심 인사이트 스캔
- SRS 문서: [`/SRS_v1.md#REQ-NF-006`](../05_SRS_v1.md) — 복기 생성 ≤ 30초
- SRS 문서: [`/SRS_v1.md#REQ-NF-015`](../05_SRS_v1.md) — Gemini JSON 파싱 실패율 < 2%
- SRS ERD: [`/SRS_v1.md#6.2-TRADE_REVIEW`](../05_SRS_v1.md) — summary_json JSONB 구조
- 시퀀스: [`/SRS_v1.md#3.4.3`](../05_SRS_v1.md) — 자동 매매 복기 생성 흐름
- 태스크 분해: [`/TASKS/06_MVP_Task_Breakdown.md#API-005`](./06_MVP_Task_Breakdown.md)

## :white_check_mark: Task Breakdown (실행 계획)

### Phase A: Gemini 인사이트 JSON 타입 정의 (핵심)
- [ ] `src/types/reviews.dto.ts` 파일 생성
- [ ] Gemini 응답의 구조화된 JSON 타입 정의:
  ```typescript
  /** Gemini가 반환하는 인사이트 타입 */
  export type InsightType = 'strength' | 'weakness' | 'suggestion';

  /** 개별 인사이트 */
  export interface ReviewInsight {
    type: InsightType;
    title: string;
    description: string;
    related_trades?: string[];
  }

  /** 개별 매매 분석 */
  export interface TradeAnalysis {
    trade_id: string;
    ticker: string;
    ticker_name?: string;
    side: 'buy' | 'sell';
    timing_score: number;    // 타점 평가 (0~100)
    comment: string;
  }

  /** 일일 통계 스냅샷 */
  export interface DailyStats {
    total_trades: number;
    win_rate: number;        // 0~100 (%)
    total_pnl: number;       // 총 손익 (원)
    max_drawdown: number;    // 최대 낙폭 (%)
  }

  /** summary_json의 전체 구조 (JSONB 저장 형식) */
  export interface ReviewSummaryJson {
    overall_assessment: string;
    emotional_score: number;   // 감정적 매매 점수 (0~100, 낮을수록 이성적)
    insights: ReviewInsight[];
    trade_analysis: TradeAnalysis[];
    daily_stats: DailyStats;
  }
  ```

### Phase B: 복기 일지 조회 DTO
- [ ] 목록/상세 조회 Response DTO:
  ```typescript
  /** 복기 일지 목록 항목 (요약) */
  export interface ReviewListItem {
    id: string;
    review_date: string;         // ISO date
    emotional_score: number;
    trade_count: number;
    status: ReviewStatus;
    daily_stats: DailyStats;
    created_at: string;
  }

  export type ReviewStatus = 'pending' | 'processing' | 'completed' | 'failed';

  /** 복기 일지 상세 항목 */
  export interface ReviewDetailItem extends ReviewListItem {
    summary_json: ReviewSummaryJson;
    model_version?: string;
    processing_time_ms?: number;
  }

  /** 목록 조회 Query Params */
  export interface ReviewQueryParams {
    start_date?: string;
    end_date?: string;
    status?: ReviewStatus;
    page?: number;
    page_size?: number;
  }

  /** 목록 조회 성공 응답 */
  export interface ReviewListResponse {
    success: true;
    data: {
      reviews: ReviewListItem[];
      pagination: { current_page: number; total_pages: number; total_items: number };
    };
  }

  /** 상세 조회 성공 응답 */
  export interface ReviewDetailResponse {
    success: true;
    data: { review: ReviewDetailItem };
  }
  ```

### Phase C: 복기 생성 트리거 DTO
- [ ] 수동 생성 트리거 Request/Response:
  ```typescript
  /** 복기 생성 트리거 요청 */
  export interface GenerateReviewRequest {
    /** 복기 대상 날짜 (기본: 오늘) */
    review_date?: string;
  }

  /** 복기 생성 성공 응답 */
  export interface GenerateReviewSuccessResponse {
    success: true;
    data: {
      review_id: string;
      review_date: string;
      status: ReviewStatus;
      processing_time_ms: number;
    };
    message: string;
  }

  export type ReviewErrorCode =
    | 'NO_TRADES_FOUND'         // 해당 날짜 매매 내역 없음
    | 'REVIEW_ALREADY_EXISTS'   // 이미 생성된 복기
    | 'GEMINI_API_ERROR'        // Gemini API 호출 실패
    | 'GEMINI_PARSE_ERROR'      // Gemini 응답 JSON 파싱 실패
    | 'TIMEOUT'                 // 30초 초과
    | 'UNAUTHORIZED'
    | 'SERVER_ERROR';

  export interface ReviewErrorResponse {
    success: false;
    error: { code: ReviewErrorCode; message: string };
  }
  ```

### Phase D: Zod 검증 스키마
- [ ] `src/types/reviews.schema.ts` 파일 생성:
  ```typescript
  import { z } from 'zod';

  export const reviewSummaryJsonSchema = z.object({
    overall_assessment: z.string().min(1),
    emotional_score: z.number().min(0).max(100),
    insights: z.array(z.object({
      type: z.enum(['strength', 'weakness', 'suggestion']),
      title: z.string().min(1),
      description: z.string().min(1),
      related_trades: z.array(z.string()).optional(),
    })).min(1),
    trade_analysis: z.array(z.object({
      trade_id: z.string(),
      ticker: z.string(),
      ticker_name: z.string().optional(),
      side: z.enum(['buy', 'sell']),
      timing_score: z.number().min(0).max(100),
      comment: z.string(),
    })),
    daily_stats: z.object({
      total_trades: z.number().int().min(0),
      win_rate: z.number().min(0).max(100),
      total_pnl: z.number(),
      max_drawdown: z.number(),
    }),
  });

  export const generateReviewSchema = z.object({
    review_date: z.string().date().optional(),
  });

  export const reviewQuerySchema = z.object({
    start_date: z.string().date().optional(),
    end_date: z.string().date().optional(),
    status: z.enum(['pending', 'processing', 'completed', 'failed']).optional(),
    page: z.coerce.number().int().min(1).default(1),
    page_size: z.coerce.number().int().min(1).max(50).default(20),
  });
  ```

### Phase E: 에러 메시지 매핑
- [ ] 한국어 에러 메시지:
  ```typescript
  export const REVIEW_ERROR_MESSAGES: Record<ReviewErrorCode, string> = {
    NO_TRADES_FOUND: '해당 날짜의 매매 내역이 없습니다.',
    REVIEW_ALREADY_EXISTS: '이미 생성된 복기 일지가 있습니다.',
    GEMINI_API_ERROR: 'AI 분석 서비스에 일시적인 오류가 발생했습니다.',
    GEMINI_PARSE_ERROR: 'AI 분석 결과 처리 중 오류가 발생했습니다.',
    TIMEOUT: 'AI 분석이 시간 초과되었습니다. 다시 시도해주세요.',
    UNAUTHORIZED: '인증이 필요합니다.',
    SERVER_ERROR: '서버 오류가 발생했습니다.',
  };
  ```

## :test_tube: Acceptance Criteria (BDD/GWT)

**Scenario 1: DTO 타입 Import 정상 동작**
- **Given:** `src/types/reviews.dto.ts`가 작성됨
- **When:** `import { ReviewSummaryJson, ReviewListItem }` 수행
- **Then:** TypeScript 컴파일 에러 없이 참조.

**Scenario 2: reviewSummaryJsonSchema — 유효한 Gemini 응답 검증**
- **Given:** 완전한 인사이트 JSON 데이터가 주어짐
- **When:** `reviewSummaryJsonSchema.safeParse()` 실행
- **Then:** `success: true`.

**Scenario 3: reviewSummaryJsonSchema — insights 빈 배열 거부**
- **Given:** insights: [] (최소 1개 필요)
- **When:** `reviewSummaryJsonSchema.safeParse()` 실행
- **Then:** `success: false`.

**Scenario 4: reviewSummaryJsonSchema — emotional_score 범위 검증**
- **Given:** emotional_score: 150 (0~100 범위 초과)
- **When:** `reviewSummaryJsonSchema.safeParse()` 실행
- **Then:** `success: false`.

**Scenario 5: 에러 코드 완전성**
- **Given:** ReviewErrorCode와 REVIEW_ERROR_MESSAGES가 정의됨
- **When:** SRS §6.1 #7~8의 모든 에러 분기 확인
- **Then:** 각 분기에 대응하는 코드·메시지가 누락 없어야 한다.

## :gear: Technical & Non-Functional Constraints
- **타입 안전:** TypeScript strict 모드 에러 없음
- **JSON 검증:** Gemini 응답의 구조적 무결성을 reviewSummaryJsonSchema로 런타임 검증 — `REQ-NF-015`
- **성능:** 복기 생성 전체 ≤ 30초 — `REQ-NF-006`
- **실패 대응:** GEMINI_PARSE_ERROR 시 최대 2회 재시도 — `REQ-NF-015` (실패율 < 2%)
- **국제화:** 에러 메시지 한국어

## :checkered_flag: Definition of Done (DoD)
- [ ] `src/types/reviews.dto.ts` 생성 및 모든 DTO/타입 export
- [ ] `src/types/reviews.schema.ts` Zod 스키마 동작
- [ ] ReviewSummaryJson 타입이 DB-007의 summary_json 구조와 일치
- [ ] REVIEW_ERROR_MESSAGES 완전성
- [ ] `npm run build` 타입 에러 없음

## :construction: Dependencies & Blockers
- **Depends on:**
  - `INFRA-001` (프로젝트 초기화)
- **Blocks:**
  - `MOCK-004` (Gemini 복기 일지 Mock JSON)
  - `CMD-REVIEW-002` (프롬프트 조립 — ReviewSummaryJson 스키마 참조)
  - `CMD-REVIEW-003` (Gemini 응답 파싱 — reviewSummaryJsonSchema 검증)
  - `CMD-REVIEW-004` (재시도 로직 — ReviewErrorCode 참조)
  - `QRY-REVIEW-001` (복기 목록 조회 — ReviewListItem DTO 참조)
  - `QRY-REVIEW-002` (복기 상세 조회 — ReviewDetailItem DTO 참조)
  - `UI-REVIEW-001` (복기 목록 화면)
  - `UI-REVIEW-002` (복기 상세 대시보드)
