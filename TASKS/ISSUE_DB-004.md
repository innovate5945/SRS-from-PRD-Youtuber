---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] DB-004: TRADE 테이블 스키마 및 마이그레이션"
labels: 'feature, database, priority:critical, sprint:0'
assignees: ''
---

## :dart: Summary
- **기능명:** [DB-004] TRADE 테이블 스키마 및 마이그레이션 (source_file_hash UNIQUE 포함)
- **목적:** CSV 업로드를 통해 적재되는 매매 내역의 핵심 저장소인 TRADE 테이블을 생성한다. 이 테이블은 F1(뇌동매매 방지), F2(CSV 업로드), F3(자동 복기), F5(통계 대시보드) 등 MVP의 거의 모든 기능이 의존하는 중심 데이터 엔티티이다. `source_file_hash` UNIQUE 제약을 통해 동일 CSV 파일의 중복 업로드를 방지한다.

## :link: References (Spec & Context)
> :bulb: AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`/SRS_v1.md#6.2-TRADE`](../05_SRS_v1.md) — TRADE 엔티티 정의 (id, account_id, ticker, side, price, quantity, source_file_hash, executed_at)
- SRS ERD: [`/SRS_v1.md#6.2.1-ERD`](../05_SRS_v1.md) — `ACCOUNT ||--o{ TRADE : "기록 1:N"`, `TRADE ||--o| TRADE_REVIEW : "요약 기반 데이터 N:1"`
- SRS 문서: [`/SRS_v1.md#REQ-FUNC-007~009`](../05_SRS_v1.md) — CSV 업로드 및 중복 방지 요구사항
- SRS 문서: [`/SRS_v1.md#REQ-NF-004`](../05_SRS_v1.md) — CSV 파싱 및 적재 ≤ 5초 (1,000건 기준)
- SRS 문서: [`/SRS_v1.md#REQ-NF-020`](../05_SRS_v1.md) — RLS 기반 사용자 간 데이터 격리
- API DTO: [`/TASKS/ISSUE_API-002.md`](./ISSUE_API-002.md) — NormalizedTrade DTO 구조 참조
- 태스크 분해: [`/TASKS/06_MVP_Task_Breakdown.md#DB-004`](./06_MVP_Task_Breakdown.md)

## :white_check_mark: Task Breakdown (실행 계획)

### Phase A: 마이그레이션 파일 생성
- [ ] `npx supabase migration new create_trades_table` 실행 → 마이그레이션 SQL 파일 생성
- [ ] 마이그레이션 파일명: `supabase/migrations/<timestamp>_create_trades_table.sql`

### Phase B: TRADE 테이블 스키마 정의
- [ ] 아래 스키마를 마이그레이션 SQL로 작성:
  ```sql
  -- TRADE 테이블: CSV 업로드를 통해 적재되는 매매 내역
  CREATE TABLE public.trades (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id UUID NOT NULL REFERENCES public.accounts(id) ON DELETE CASCADE,
    
    -- 매매 정보
    ticker VARCHAR(20) NOT NULL,              -- 종목 코드
    ticker_name VARCHAR(100),                 -- 종목명 (표시용)
    side VARCHAR(4) NOT NULL 
      CHECK (side IN ('buy', 'sell')),        -- 매수/매도 구분
    price DECIMAL(15, 2) NOT NULL 
      CHECK (price > 0),                      -- 체결 가격
    quantity INTEGER NOT NULL 
      CHECK (quantity > 0),                   -- 체결 수량
    
    -- CSV 중복 방지
    source_file_hash VARCHAR(64) NOT NULL,    -- SHA-256 해시 (파일 단위 중복 검증)
    
    -- 체결 일시
    executed_at TIMESTAMPTZ NOT NULL,         -- 체결 일시
    
    -- 메타데이터
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
  );

  -- source_file_hash에 UNIQUE 인덱스 적용 (동일 파일 재업로드 차단)
  -- 주의: 파일 단위 유니크이므로, 동일 파일 내 복수 TRADE는 동일 hash 공유
  -- → 파일 레벨 중복 체크는 별도 테이블(upload_files) 또는 INSERT 전 조회로 처리
  CREATE INDEX idx_trades_source_file_hash ON public.trades(source_file_hash);
  
  -- 성능 인덱스
  CREATE INDEX idx_trades_account_id ON public.trades(account_id);
  CREATE INDEX idx_trades_executed_at ON public.trades(executed_at DESC);
  CREATE INDEX idx_trades_ticker ON public.trades(ticker);
  
  -- 복합 인덱스: 당일 매매 집계 쿼리 최적화 (알람 로직용)
  CREATE INDEX idx_trades_account_executed 
    ON public.trades(account_id, executed_at DESC);
  ```

### Phase C: 파일 중복 체크 보조 테이블 (선택)
- [ ] CSV 파일 단위 중복 검증을 위한 보조 구조 검토:
  ```sql
  -- 업로드 파일 이력 관리 (source_file_hash UNIQUE 적용)
  CREATE TABLE public.upload_files (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id UUID NOT NULL REFERENCES public.accounts(id) ON DELETE CASCADE,
    source_file_hash VARCHAR(64) NOT NULL,
    original_filename VARCHAR(255),
    row_count INTEGER NOT NULL DEFAULT 0,
    uploaded_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    -- 동일 계좌 내 동일 파일 중복 방지
    UNIQUE (account_id, source_file_hash)
  );

  CREATE INDEX idx_upload_files_hash ON public.upload_files(source_file_hash);
  ```

### Phase D: Row-Level Security 정책
- [ ] TRADE 테이블에 대한 RLS 정책 활성화:
  ```sql
  ALTER TABLE public.trades ENABLE ROW LEVEL SECURITY;

  -- 사용자는 자신의 계좌에 속한 매매 내역만 조회 가능
  CREATE POLICY "Users can view own trades"
    ON public.trades FOR SELECT
    USING (
      account_id IN (
        SELECT id FROM public.accounts WHERE user_id = auth.uid()
      )
    );

  -- 사용자는 자신의 계좌에만 매매 내역 삽입 가능
  CREATE POLICY "Users can insert own trades"
    ON public.trades FOR INSERT
    WITH CHECK (
      account_id IN (
        SELECT id FROM public.accounts WHERE user_id = auth.uid()
      )
    );

  -- upload_files 테이블도 동일 RLS 적용
  ALTER TABLE public.upload_files ENABLE ROW LEVEL SECURITY;

  CREATE POLICY "Users can view own upload files"
    ON public.upload_files FOR SELECT
    USING (
      account_id IN (
        SELECT id FROM public.accounts WHERE user_id = auth.uid()
      )
    );

  CREATE POLICY "Users can insert own upload files"
    ON public.upload_files FOR INSERT
    WITH CHECK (
      account_id IN (
        SELECT id FROM public.accounts WHERE user_id = auth.uid()
      )
    );
  ```

### Phase E: 마이그레이션 적용 및 검증
- [ ] 로컬: `npx supabase db reset` 으로 마이그레이션 적용 확인
- [ ] 원격: `npx supabase db push` 로 배포 환경 적용
- [ ] Supabase Studio에서 `public.trades` 및 `public.upload_files` 테이블 구조 확인
- [ ] TypeScript 타입 재생성: `npm run gen:types`
- [ ] 1,000건 벌크 INSERT 성능 테스트 (5초 이내 목표)

## :test_tube: Acceptance Criteria (BDD/GWT)

**Scenario 1: 마이그레이션 적용 성공**
- **Given:** DB-003까지의 마이그레이션이 적용되어 `public.accounts` 테이블이 존재함
- **When:** `npx supabase db reset`을 실행함
- **Then:** `public.trades` 테이블이 모든 컬럼과 제약 조건과 함께 생성되어야 한다.

**Scenario 2: 유효한 매매 내역 INSERT 성공**
- **Given:** 유효한 account_id와 매매 데이터가 주어짐 (ticker: '005930', side: 'buy', price: 72000, quantity: 10)
- **When:** `public.trades`에 INSERT함
- **Then:** 레코드가 정상 생성되고, 자동 생성된 UUID `id`를 가져야 한다.

**Scenario 3: price/quantity CHECK 제약 검증**
- **Given:** price = -100 또는 quantity = 0인 데이터가 주어짐
- **When:** INSERT를 시도함
- **Then:** CHECK 제약 위반 에러가 발생해야 한다.

**Scenario 4: side CHECK 제약 검증**
- **Given:** side = 'hold'인 데이터가 주어짐
- **When:** INSERT를 시도함
- **Then:** CHECK 제약 위반 에러가 발생해야 한다.

**Scenario 5: 동일 파일 해시 중복 업로드 차단**
- **Given:** `source_file_hash = 'abc123...'`으로 upload_files에 이미 레코드가 존재함
- **When:** 동일 account_id + 동일 source_file_hash로 upload_files에 INSERT 시도함
- **Then:** UNIQUE 제약 위반 에러가 발생해야 한다.

**Scenario 6: RLS — 타 사용자 매매 내역 접근 차단**
- **Given:** User A와 User B가 각각 매매 내역을 보유함
- **When:** User A가 인증된 상태에서 `SELECT * FROM public.trades`를 실행함
- **Then:** User A의 계좌에 속한 매매 내역만 반환되어야 한다.

**Scenario 7: CASCADE 삭제 — 계좌 삭제 시 매매 내역 자동 삭제**
- **Given:** 계좌에 50건의 매매 내역이 연결되어 있음
- **When:** 해당 계좌를 `public.accounts`에서 삭제함
- **Then:** 연결된 50건의 매매 내역이 CASCADE로 자동 삭제되어야 한다.

**Scenario 8: 벌크 INSERT 성능 (1,000건)**
- **Given:** 정규화된 매매 데이터 1,000건이 준비됨
- **When:** Bulk INSERT를 실행함
- **Then:** 5초 이내에 모든 레코드가 적재 완료되어야 한다 (`REQ-NF-004`).

## :gear: Technical & Non-Functional Constraints
- **보안:** RLS 필수 활성화 — `REQ-NF-020` 준수. 계좌 소유자만 매매 내역 접근 가능
- **중복 방지:** source_file_hash 기반 파일 단위 중복 체크 — `REQ-FUNC-009` 준수
- **데이터 무결성:** price > 0, quantity > 0, side ∈ {buy, sell} CHECK 제약
- **성능:** 1,000건 기준 벌크 INSERT ≤ 5초 — `REQ-NF-004` 준수
- **인덱스:** executed_at DESC, account_id 복합 인덱스로 당일 매매 집계 쿼리 최적화
- **타입 안전:** 마이그레이션 적용 후 반드시 `gen:types` 실행하여 TypeScript 타입 동기화

## :checkered_flag: Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 마이그레이션 SQL 파일이 `supabase/migrations/` 에 커밋되었는가?
- [ ] RLS 정책이 활성화되고 테스트되었는가?
- [ ] CHECK 제약 (price > 0, quantity > 0, side ∈ {buy, sell})이 정상 동작하는가?
- [ ] source_file_hash 기반 중복 차단이 동작하는가?
- [ ] 1,000건 벌크 INSERT 성능이 5초 이내인가?
- [ ] `database.types.ts`가 갱신되어 `Trades` 및 `UploadFiles` 타입이 포함되었는가?

## :construction: Dependencies & Blockers
- **Depends on:**
  - `DB-003` (ACCOUNT 테이블 — `account_id FK → accounts.id` 참조)
  - `DB-001` (Supabase 프로젝트 초기화 — CLI 및 마이그레이션 인프라)
- **Blocks:**
  - `CMD-CSV-002` (CSV 헤더/데이터 검증 — trades 스키마 참조)
  - `CMD-CSV-004` (source_file_hash 중복 확인 — upload_files 테이블 참조)
  - `CMD-CSV-005` (TRADE Bulk Insert — trades 테이블 필요)
  - `CMD-ALERT-002` (당일 누적 손실 계산 — trades 데이터 조회)
  - `QRY-CSV-001` (매매 내역 통합 조회)
  - `QRY-DASH-001` (대시보드 통계 집계 — trades 데이터 기반)
  - `CMD-REVIEW-002` (복기 일지 데이터 조립 — trades JSON 화)
