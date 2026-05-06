---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] DB-007: TRADE_REVIEW 테이블 스키마 및 마이그레이션"
labels: 'feature, database, priority:high, sprint:0'
assignees: ''
---

## :dart: Summary
- **기능명:** [DB-007] TRADE_REVIEW 테이블 스키마 및 마이그레이션 (summary_json JSONB)
- **목적:** F3(자동 매매 복기 일지)의 결과물인 Gemini API 기반 복기 일지를 저장하는 TRADE_REVIEW 테이블을 생성한다. Gemini가 반환한 구조화된 인사이트 데이터를 `summary_json` JSONB 컬럼에 저장하여, 복기 일지 목록 조회(QRY-REVIEW-001)와 상세 대시보드(UI-REVIEW-002)의 데이터 소스로 활용한다.

## :link: References (Spec & Context)
> :bulb: AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`/SRS_v1.md#6.2-TRADE_REVIEW`](../05_SRS_v1.md) — TRADE_REVIEW 엔티티 (id, user_id, summary_json JSONB, created_at)
- SRS ERD: [`/SRS_v1.md#6.2.1-ERD`](../05_SRS_v1.md) — `USER ||--o{ TRADE_REVIEW : "생성 1:N"`, `TRADE ||--o| TRADE_REVIEW : "요약 기반 데이터 N:1"`
- SRS 문서: [`/SRS_v1.md#REQ-FUNC-013`](../05_SRS_v1.md) — Gemini 기반 프롬프트 추론으로 구조화된 일지 저장
- SRS 문서: [`/SRS_v1.md#REQ-FUNC-014`](../05_SRS_v1.md) — 3분 내 핵심 인사이트 스캔 가능 대시보드 UI
- SRS 문서: [`/SRS_v1.md#REQ-FUNC-015`](../05_SRS_v1.md) — 주간 복기 요약 PDF 추출
- SRS 시퀀스: [`/SRS_v1.md#3.4.3`](../05_SRS_v1.md) — 프롬프트 기반 자동 매매 복기 생성 흐름
- SRS 문서: [`/SRS_v1.md#REQ-NF-006`](../05_SRS_v1.md) — Gemini 복기 생성 ≤ 30초
- 태스크 분해: [`/TASKS/06_MVP_Task_Breakdown.md#DB-007`](./06_MVP_Task_Breakdown.md)

## :white_check_mark: Task Breakdown (실행 계획)

### Phase A: 마이그레이션 파일 생성
- [ ] `npx supabase migration new create_trade_reviews_table` 실행 → 마이그레이션 SQL 파일 생성
- [ ] 마이그레이션 파일명: `supabase/migrations/<timestamp>_create_trade_reviews_table.sql`

### Phase B: TRADE_REVIEW 테이블 스키마 정의
- [ ] 아래 스키마를 마이그레이션 SQL로 작성:
  ```sql
  -- TRADE_REVIEW 테이블: Gemini 기반 자동 매매 복기 일지
  CREATE TABLE public.trade_reviews (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
    
    -- 복기 대상 기간
    review_date DATE NOT NULL,                     -- 복기 대상 날짜 (당일 매매 기준)
    
    -- Gemini 인사이트 데이터 (JSONB)
    summary_json JSONB NOT NULL,
    
    -- AI 모델 메타데이터
    model_version VARCHAR(50),                     -- 사용된 Gemini 모델 버전
    prompt_tokens INTEGER,                         -- 프롬프트 토큰 수
    completion_tokens INTEGER,                     -- 응답 토큰 수
    processing_time_ms INTEGER,                    -- 처리 소요 시간 (ms)
    
    -- 복기 대상 매매 건수
    trade_count INTEGER NOT NULL DEFAULT 0,        -- 복기에 포함된 매매 건수
    
    -- 상태
    status VARCHAR(15) NOT NULL DEFAULT 'completed'
      CHECK (status IN ('pending', 'processing', 'completed', 'failed')),
    
    -- 메타데이터
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    -- 동일 사용자, 동일 날짜에 중복 복기 방지
    UNIQUE (user_id, review_date)
  );

  -- 인덱스
  CREATE INDEX idx_trade_reviews_user_id ON public.trade_reviews(user_id);
  CREATE INDEX idx_trade_reviews_review_date ON public.trade_reviews(review_date DESC);
  CREATE INDEX idx_trade_reviews_user_date 
    ON public.trade_reviews(user_id, review_date DESC);
  
  -- JSONB 내부 검색용 GIN 인덱스 (향후 인사이트 검색 시 활용)
  CREATE INDEX idx_trade_reviews_summary_gin 
    ON public.trade_reviews USING GIN (summary_json);

  -- updated_at 자동 갱신 트리거
  CREATE TRIGGER trigger_trade_reviews_updated_at
    BEFORE UPDATE ON public.trade_reviews
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
  ```

### Phase C: summary_json JSONB 스키마 문서화
- [ ] Gemini 응답의 기대 JSON 구조를 주석으로 문서화:
  ```sql
  -- summary_json 기대 구조 (Gemini 응답 파싱 결과):
  -- {
  --   "overall_assessment": "string",           -- 전체 매매 평가 요약
  --   "emotional_score": number,                -- 감정적 매매 점수 (0~100)
  --   "insights": [
  --     {
  --       "type": "strength" | "weakness" | "suggestion",
  --       "title": "string",
  --       "description": "string",
  --       "related_trades": ["trade_id_1", "trade_id_2"]
  --     }
  --   ],
  --   "trade_analysis": [
  --     {
  --       "trade_id": "string",
  --       "ticker": "string",
  --       "timing_score": number,               -- 타점 평가 (0~100)
  --       "comment": "string"
  --     }
  --   ],
  --   "daily_stats": {
  --     "total_trades": number,
  --     "win_rate": number,
  --     "total_pnl": number,
  --     "max_drawdown": number
  --   }
  -- }
  ```

### Phase D: Row-Level Security 정책
- [ ] TRADE_REVIEW 테이블에 대한 RLS 정책 활성화:
  ```sql
  ALTER TABLE public.trade_reviews ENABLE ROW LEVEL SECURITY;

  -- 사용자는 자신의 복기 일지만 조회 가능
  CREATE POLICY "Users can view own reviews"
    ON public.trade_reviews FOR SELECT
    USING (auth.uid() = user_id);

  -- 사용자는 자신의 복기 일지만 생성 가능 (Server Action 경유)
  CREATE POLICY "Users can insert own reviews"
    ON public.trade_reviews FOR INSERT
    WITH CHECK (auth.uid() = user_id);

  -- 사용자는 자신의 복기 일지만 수정 가능 (재생성 시)
  CREATE POLICY "Users can update own reviews"
    ON public.trade_reviews FOR UPDATE
    USING (auth.uid() = user_id)
    WITH CHECK (auth.uid() = user_id);
  ```

### Phase E: 마이그레이션 적용 및 검증
- [ ] 로컬: `npx supabase db reset` 으로 마이그레이션 적용 확인
- [ ] 원격: `npx supabase db push` 로 배포 환경 적용
- [ ] Supabase Studio에서 `public.trade_reviews` 테이블 구조 확인
- [ ] TypeScript 타입 재생성: `npm run gen:types`
- [ ] JSONB 데이터 INSERT/SELECT 테스트

## :test_tube: Acceptance Criteria (BDD/GWT)

**Scenario 1: 마이그레이션 적용 성공**
- **Given:** DB-002까지의 마이그레이션이 적용되어 `public.users` 테이블이 존재함
- **When:** `npx supabase db reset`을 실행함
- **Then:** `public.trade_reviews` 테이블이 모든 컬럼과 제약 조건과 함께 생성되어야 한다.

**Scenario 2: 유효한 복기 일지 INSERT 성공**
- **Given:** 유효한 user_id, review_date = '2026-05-01', summary_json = '{"overall_assessment": "..."}'
- **When:** INSERT함
- **Then:** 레코드가 정상 생성되어야 한다.

**Scenario 3: 동일 사용자/날짜 중복 복기 차단**
- **Given:** User A의 2026-05-01 복기 일지가 이미 존재함
- **When:** 동일 user_id + review_date로 INSERT 시도함
- **Then:** UNIQUE 제약 위반 에러가 발생해야 한다.

**Scenario 4: summary_json JSONB 쿼리**
- **Given:** summary_json에 `{"emotional_score": 75}` 데이터가 저장됨
- **When:** `WHERE summary_json->>'emotional_score'` 쿼리를 실행함
- **Then:** 해당 값이 정상적으로 추출되어야 한다.

**Scenario 5: RLS — 타 사용자 복기 일지 접근 차단**
- **Given:** User A와 User B가 각각 복기 일지를 보유함
- **When:** User A가 인증된 상태에서 `SELECT * FROM public.trade_reviews`를 실행함
- **Then:** User A의 복기 일지만 반환되어야 한다.

**Scenario 6: 날짜별 조회 정렬**
- **Given:** User A가 5일치 복기 일지를 보유함
- **When:** `ORDER BY review_date DESC` 쿼리를 실행함
- **Then:** 최신 날짜 순으로 정렬된 결과가 반환되어야 한다.

**Scenario 7: status CHECK 제약 검증**
- **Given:** status = 'invalid_status'인 데이터가 주어짐
- **When:** INSERT를 시도함
- **Then:** CHECK 제약 위반 에러가 발생해야 한다.

**Scenario 8: CASCADE 삭제 — 사용자 삭제 시 복기 일지 자동 삭제**
- **Given:** User A가 10건의 복기 일지를 보유함
- **When:** `public.users`에서 User A를 삭제함
- **Then:** 관련 trade_reviews 10건이 CASCADE로 자동 삭제되어야 한다.

## :gear: Technical & Non-Functional Constraints
- **보안:** RLS 필수 활성화 — `REQ-NF-020` 준수
- **JSONB 유연성:** summary_json은 Gemini 응답 구조 변경에 유연하게 대응 가능
- **중복 방지:** (user_id, review_date) UNIQUE 제약으로 동일 날짜 중복 복기 방지
- **성능:** JSONB GIN 인덱스로 인사이트 내부 검색 최적화
- **상태 관리:** status 필드로 비동기 복기 생성 과정 추적 (pending → processing → completed/failed)
- **타입 안전:** 마이그레이션 적용 후 반드시 `gen:types` 실행하여 TypeScript 타입 동기화

## :checkered_flag: Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 마이그레이션 SQL 파일이 `supabase/migrations/` 에 커밋되었는가?
- [ ] RLS 정책이 활성화되고 테스트되었는가?
- [ ] JSONB INSERT/SELECT가 정상 동작하는가?
- [ ] (user_id, review_date) UNIQUE 제약이 동작하는가?
- [ ] GIN 인덱스가 생성되었는가?
- [ ] `database.types.ts`가 갱신되어 `TradeReviews` 타입이 포함되었는가?

## :construction: Dependencies & Blockers
- **Depends on:**
  - `DB-002` (USER 테이블 — `user_id FK → users.id` 참조)
- **Blocks:**
  - `DB-008` (전 테이블 RLS 정책 — trade_reviews RLS 패턴 참조)
  - `CMD-REVIEW-003` (Gemini 응답 파싱 → TRADE_REVIEW 저장)
  - `QRY-REVIEW-001` (복기 일지 목록 조회)
  - `QRY-REVIEW-002` (복기 일지 상세 조회 — summary_json 파싱)
  - `UI-REVIEW-001` (복기 일지 목록 화면)
  - `UI-REVIEW-002` (복기 일지 상세 대시보드)
