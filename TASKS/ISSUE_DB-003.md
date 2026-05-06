---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] DB-003: ACCOUNT 테이블 스키마 및 마이그레이션"
labels: 'feature, database, priority:high, sprint:0'
assignees: ''
---

## :dart: Summary
- **기능명:** [DB-003] ACCOUNT 테이블 스키마 및 마이그레이션 (id, user_id, status, created_at)
- **목적:** 사용자의 증권 계좌 정보를 관리하는 ACCOUNT 테이블을 생성한다. 하나의 사용자가 복수의 계좌를 보유할 수 있으며(1:N), 각 계좌에 소속된 TRADE 내역이 연결되는 중간 엔티티 역할을 한다. 계좌 활성/비활성 상태를 관리하여 비활성 계좌의 매매 데이터를 필터링할 수 있는 기반을 마련한다.

## :link: References (Spec & Context)
> :bulb: AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`/SRS_v1.md#6.2-ACCOUNT`](../05_SRS_v1.md) — ACCOUNT 엔티티 정의 (id UUID PK, user_id FK, status ENUM, created_at)
- SRS ERD: [`/SRS_v1.md#6.2.1-ERD`](../05_SRS_v1.md) — `USER ||--o{ ACCOUNT : "보유 1:N"`, `ACCOUNT ||--o{ TRADE : "기록 1:N"`
- SRS 문서: [`/SRS_v1.md#REQ-NF-020`](../05_SRS_v1.md) — RLS 기반 사용자 간 데이터 격리
- 태스크 분해: [`/TASKS/06_MVP_Task_Breakdown.md#DB-003`](./06_MVP_Task_Breakdown.md)
- 선행 태스크: [`/TASKS/ISSUE_DB-002.md`](./ISSUE_DB-002.md) — USER 테이블 (FK 참조 대상)

## :white_check_mark: Task Breakdown (실행 계획)

### Phase A: 마이그레이션 파일 생성
- [ ] `npx supabase migration new create_accounts_table` 실행 → 마이그레이션 SQL 파일 생성
- [ ] 마이그레이션 파일명: `supabase/migrations/<timestamp>_create_accounts_table.sql`

### Phase B: ACCOUNT 테이블 스키마 정의
- [ ] 아래 스키마를 마이그레이션 SQL로 작성:
  ```sql
  -- ACCOUNT 테이블: 사용자의 증권 계좌 관리
  CREATE TABLE public.accounts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
    
    -- 계좌 메타데이터
    broker_name VARCHAR(50),               -- 증권사명 (예: '한국투자', '토스')
    account_alias VARCHAR(100),            -- 사용자 지정 계좌 별칭
    status VARCHAR(10) NOT NULL DEFAULT 'active'
      CHECK (status IN ('active', 'inactive')),
    
    -- 타임스탬프
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
  );

  -- 인덱스
  CREATE INDEX idx_accounts_user_id ON public.accounts(user_id);
  CREATE INDEX idx_accounts_status ON public.accounts(status);

  -- updated_at 자동 갱신 트리거 (DB-002에서 생성한 함수 재사용)
  CREATE TRIGGER trigger_accounts_updated_at
    BEFORE UPDATE ON public.accounts
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
  ```

### Phase C: Row-Level Security 정책
- [ ] ACCOUNT 테이블에 대한 RLS 정책 활성화:
  ```sql
  ALTER TABLE public.accounts ENABLE ROW LEVEL SECURITY;

  -- 사용자는 자신의 계좌만 조회 가능
  CREATE POLICY "Users can view own accounts"
    ON public.accounts FOR SELECT
    USING (auth.uid() = user_id);

  -- 사용자는 자신의 계좌만 생성 가능
  CREATE POLICY "Users can create own accounts"
    ON public.accounts FOR INSERT
    WITH CHECK (auth.uid() = user_id);

  -- 사용자는 자신의 계좌만 수정 가능
  CREATE POLICY "Users can update own accounts"
    ON public.accounts FOR UPDATE
    USING (auth.uid() = user_id)
    WITH CHECK (auth.uid() = user_id);
  ```

### Phase D: 기본 계좌 자동 생성 트리거 (선택)
- [ ] 회원가입 시 기본 계좌를 자동 생성하는 트리거 검토:
  ```sql
  -- 선택 사항: 사용자 프로필 생성 시 기본 계좌 자동 생성
  CREATE OR REPLACE FUNCTION public.handle_new_user_account()
  RETURNS TRIGGER AS $$
  BEGIN
    INSERT INTO public.accounts (user_id, account_alias, status)
    VALUES (NEW.id, '기본 계좌', 'active');
    RETURN NEW;
  END;
  $$ LANGUAGE plpgsql SECURITY DEFINER;

  CREATE TRIGGER on_user_profile_created
    AFTER INSERT ON public.users
    FOR EACH ROW
    EXECUTE FUNCTION public.handle_new_user_account();
  ```

### Phase E: 마이그레이션 적용 및 검증
- [ ] 로컬: `npx supabase db reset` 으로 마이그레이션 적용 확인
- [ ] 원격: `npx supabase db push` 로 배포 환경 적용
- [ ] Supabase Studio에서 `public.accounts` 테이블 구조 확인
- [ ] TypeScript 타입 재생성: `npm run gen:types`

## :test_tube: Acceptance Criteria (BDD/GWT)

**Scenario 1: 마이그레이션 적용 성공**
- **Given:** DB-002까지의 마이그레이션이 적용되어 `public.users` 테이블이 존재함
- **When:** `npx supabase db reset` 또는 `npx supabase db push`를 실행함
- **Then:** `public.accounts` 테이블이 모든 컬럼(id, user_id, broker_name, account_alias, status, created_at, updated_at)과 함께 생성되어야 한다.

**Scenario 2: 사용자와 계좌 1:N 관계**
- **Given:** User A가 가입되어 `public.users`에 존재함
- **When:** User A의 `user_id`로 2개의 계좌를 INSERT함
- **Then:** 2개의 계좌 레코드가 생성되고, 각각 고유한 UUID `id`를 가져야 한다.

**Scenario 3: RLS — 타 사용자 계좌 접근 차단**
- **Given:** User A와 User B가 각각 계좌를 보유함
- **When:** User A가 인증된 상태에서 `SELECT * FROM public.accounts`를 실행함
- **Then:** User A의 계좌만 반환되고, User B의 계좌는 조회되지 않아야 한다.

**Scenario 4: status CHECK 제약 검증**
- **Given:** `public.accounts` 테이블이 존재함
- **When:** `status = 'deleted'`인 값으로 INSERT를 시도함
- **Then:** CHECK 제약 위반 에러가 발생하고 레코드가 삽입되지 않아야 한다.

**Scenario 5: CASCADE 삭제 — 사용자 삭제 시 계좌 자동 삭제**
- **Given:** User A가 2개의 계좌를 보유함
- **When:** `public.users`에서 User A를 삭제함
- **Then:** `public.accounts`의 User A 소유 계좌 2건이 CASCADE로 자동 삭제되어야 한다.

**Scenario 6: updated_at 자동 갱신**
- **Given:** 계좌 레코드가 존재함
- **When:** `status`를 'inactive'로 수정함
- **Then:** `updated_at` 컬럼이 현재 시각으로 자동 갱신되어야 한다.

## :gear: Technical & Non-Functional Constraints
- **보안:** RLS 필수 활성화 — `REQ-NF-020` 준수. 인증되지 않은 요청은 모든 행에 대해 접근 불가
- **데이터 무결성:** `users(id)` FK + CASCADE 삭제로 참조 정합성 보장
- **상태 관리:** status ENUM은 `active`, `inactive` 두 값만 허용 (CHECK 제약)
- **확장성:** broker_name 필드로 향후 증권사별 필터링 지원 가능
- **타입 안전:** 마이그레이션 적용 후 반드시 `gen:types` 실행하여 TypeScript 타입 동기화

## :checkered_flag: Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 마이그레이션 SQL 파일이 `supabase/migrations/` 에 커밋되었는가?
- [ ] RLS 정책이 활성화되고 테스트되었는가?
- [ ] user_id FK → users.id 참조가 정상 동작하는가?
- [ ] CASCADE 삭제가 정상 동작하는가?
- [ ] `database.types.ts`가 갱신되어 `Accounts` 타입이 포함되었는가?

## :construction: Dependencies & Blockers
- **Depends on:**
  - `DB-002` (USER 테이블 — `user_id FK → users.id` 참조)
  - `DB-001` (Supabase 프로젝트 초기화 — CLI 및 마이그레이션 인프라)
- **Blocks:**
  - `DB-004` (TRADE 테이블 — `account_id FK → accounts.id`)
  - `CMD-CSV-005` (TRADE Bulk Insert — 계좌 참조 필요)
