---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] DB-002: USER 테이블 스키마 및 마이그레이션 스크립트 작성"
labels: 'feature, database, priority:critical, sprint:0'
assignees: ''
---

## :dart: Summary
- **기능명:** [DB-002] USER 테이블 스키마 및 마이그레이션 스크립트 작성
- **목적:** 시스템의 모든 인증·인가·데이터 격리(RLS)의 기반이 되는 USER 테이블을 설계하고 마이그레이션을 작성한다. Supabase Auth의 `auth.users` 테이블과 연동되는 public.users 프로필 테이블을 구현하여, 후속 모든 도메인 테이블(ACCOUNT, TRADE, ALERT_RULE, TRADE_REVIEW)이 FK로 참조할 수 있는 기준 엔티티를 확립한다.

## :link: References (Spec & Context)
> :bulb: AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`/SRS_v1.md#6.2-ERD-USER`](../SRS_v1.md) — USER 엔티티 정의 (id UUID PK, email VARCHAR UK)
- SRS 문서: [`/SRS_v1.md#REQ-NF-020`](../SRS_v1.md) — RLS 기반 사용자 간 데이터 격리
- SRS 문서: [`/SRS_v1.md#REQ-NF-023a`](../SRS_v1.md) — 개인정보보호법 동의 이력 관리
- SRS ERD: [`/SRS_v1.md#6.2.1-ERD`](../SRS_v1.md) — `USER ||--o{ ACCOUNT`, `USER ||--o{ ALERT_RULE`, `USER ||--o{ TRADE_REVIEW`
- 태스크 분해: [`/TASKS/MVP_Task_Breakdown.md#DB-002`](./MVP_Task_Breakdown.md)

## :white_check_mark: Task Breakdown (실행 계획)

### Phase A: 마이그레이션 파일 생성
- [ ] `npx supabase migration new create_users_table` 실행 → 마이그레이션 SQL 파일 생성
- [ ] 마이그레이션 파일명: `supabase/migrations/<timestamp>_create_users_table.sql`

### Phase B: USER 테이블 스키마 정의
- [ ] 아래 스키마를 마이그레이션 SQL로 작성:
  ```sql
  -- public.users 프로필 테이블 (Supabase Auth와 연동)
  CREATE TABLE public.users (
    id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
    email VARCHAR(255) NOT NULL UNIQUE,
    display_name VARCHAR(100),
    
    -- 개인정보보호법(PIPA) 동의 이력 (REQ-NF-023a)
    privacy_consent_at TIMESTAMPTZ,
    marketing_consent_at TIMESTAMPTZ,
    
    -- 메타데이터
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
  );
  
  -- 인덱스
  CREATE INDEX idx_users_email ON public.users(email);
  
  -- updated_at 자동 갱신 트리거
  CREATE OR REPLACE FUNCTION update_updated_at_column()
  RETURNS TRIGGER AS $$
  BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
  END;
  $$ LANGUAGE plpgsql;
  
  CREATE TRIGGER trigger_users_updated_at
    BEFORE UPDATE ON public.users
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
  ```

### Phase C: Supabase Auth 연동 — 자동 프로필 생성 트리거
- [ ] `auth.users`에 레코드 생성 시 `public.users`에 자동으로 프로필 레코드를 삽입하는 트리거 함수 작성:
  ```sql
  -- Auth 회원가입 시 자동 프로필 생성
  CREATE OR REPLACE FUNCTION public.handle_new_user()
  RETURNS TRIGGER AS $$
  BEGIN
    INSERT INTO public.users (id, email)
    VALUES (NEW.id, NEW.email);
    RETURN NEW;
  END;
  $$ LANGUAGE plpgsql SECURITY DEFINER;
  
  CREATE TRIGGER on_auth_user_created
    AFTER INSERT ON auth.users
    FOR EACH ROW
    EXECUTE FUNCTION public.handle_new_user();
  ```

### Phase D: Row-Level Security 기본 정책 (사전 설정)
- [ ] USER 테이블에 대한 기본 RLS 정책 활성화:
  ```sql
  ALTER TABLE public.users ENABLE ROW LEVEL SECURITY;
  
  -- 사용자는 자신의 프로필만 조회 가능
  CREATE POLICY "Users can view own profile"
    ON public.users FOR SELECT
    USING (auth.uid() = id);
  
  -- 사용자는 자신의 프로필만 수정 가능
  CREATE POLICY "Users can update own profile"
    ON public.users FOR UPDATE
    USING (auth.uid() = id)
    WITH CHECK (auth.uid() = id);
  ```

### Phase E: 데이터 파기 정책 지원 (REQ-NF-023a)
- [ ] 사용자 탈퇴 시 데이터 파기를 위한 CASCADE 설정 검증 (`auth.users` → `public.users`)
- [ ] 향후 데이터 파기 요청 처리를 위한 soft delete 또는 anonymization 전략 주석 문서화

### Phase F: 마이그레이션 적용 및 검증
- [ ] 로컬: `npx supabase db reset` 으로 마이그레이션 적용 확인
- [ ] 원격: `npx supabase db push` 로 배포 환경 적용
- [ ] Supabase Studio에서 `public.users` 테이블 구조 확인
- [ ] TypeScript 타입 재생성: `npm run gen:types`

## :test_tube: Acceptance Criteria (BDD/GWT)

**Scenario 1: 마이그레이션 적용 성공**
- **Given:** Supabase 프로젝트가 초기화되어 있음
- **When:** `npx supabase db reset` 또는 `npx supabase db push`를 실행함
- **Then:** `public.users` 테이블이 모든 컬럼(id, email, display_name, privacy_consent_at, marketing_consent_at, created_at, updated_at)과 함께 생성되어야 한다.

**Scenario 2: Supabase Auth 회원가입 시 자동 프로필 생성**
- **Given:** `public.users`가 생성되고 `on_auth_user_created` 트리거가 활성화됨
- **When:** Supabase Auth를 통해 `test@example.com`으로 회원가입함
- **Then:** `public.users`에 해당 user의 id와 email이 자동 삽입되어야 한다.

**Scenario 3: RLS — 타 사용자 프로필 접근 차단**
- **Given:** User A(alice@test.com)와 User B(bob@test.com)가 각각 가입됨
- **When:** User A가 인증된 상태에서 `SELECT * FROM public.users` 쿼리를 실행함
- **Then:** User A 자신의 레코드만 반환되고, User B의 데이터는 조회되지 않아야 한다.

**Scenario 4: email UNIQUE 제약 위반**
- **Given:** `exist@example.com`으로 이미 가입된 사용자가 존재함
- **When:** 동일한 이메일로 `public.users`에 INSERT를 시도함
- **Then:** UNIQUE 제약 위반 에러가 발생하고 레코드가 삽입되지 않아야 한다.

**Scenario 5: updated_at 자동 갱신**
- **Given:** User A의 프로필 레코드가 존재함
- **When:** display_name을 수정함
- **Then:** `updated_at` 컬럼이 현재 시각으로 자동 갱신되어야 한다.

**Scenario 6: 회원 탈퇴 CASCADE 삭제**
- **Given:** User A가 가입되어 `public.users`에 레코드가 존재함
- **When:** `auth.users`에서 User A를 삭제함 (관리자 API 또는 Dashboard)
- **Then:** `public.users`의 해당 레코드도 CASCADE로 자동 삭제되어야 한다.

## :gear: Technical & Non-Functional Constraints
- **보안:** RLS 필수 활성화 — `REQ-NF-020` 준수. 인증되지 않은 요청은 모든 행에 대해 접근 불가
- **컴플라이언스:** 개인정보보호법 동의 이력 컬럼 포함 — `REQ-NF-023a` 준수
- **데이터 무결성:** `auth.users(id)` FK + CASCADE 삭제로 참조 정합성 보장
- **성능:** email 인덱스를 통한 조회 최적화
- **확장성:** 향후 `display_name`, `avatar_url`, `plan_tier` 등 프로필 확장 가능 구조
- **타입 안전:** 마이그레이션 적용 후 반드시 `gen:types` 실행하여 TypeScript 타입 동기화

## :checkered_flag: Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 마이그레이션 SQL 파일이 `supabase/migrations/` 에 커밋되었는가?
- [ ] RLS 정책이 활성화되고 테스트되었는가?
- [ ] Auth 연동 트리거가 동작하여 회원가입 시 프로필이 자동 생성되는가?
- [ ] `database.types.ts`가 갱신되어 `Users` 타입이 포함되었는가?
- [ ] CASCADE 삭제가 정상 동작하는가?

## :construction: Dependencies & Blockers
- **Depends on:**
  - `DB-001` (Supabase 프로젝트 초기화 및 Next.js 연동 — 클라이언트 및 CLI 필요)
- **Blocks:**
  - `DB-003` (ACCOUNT 테이블 — `user_id FK → users.id`)
  - `DB-005` (ALERT_RULE 테이블 — `user_id FK → users.id`)
  - `DB-007` (TRADE_REVIEW 테이블 — `user_id FK → users.id`)
  - `DB-008` (전 테이블 RLS 정책 — users RLS 패턴 참조)
  - `CMD-AUTH-001` (회원가입 Server Action — users 테이블 필요)
  - `SEC-003` (개인정보보호법 동의 이력 관리 — privacy_consent_at 컬럼 활용)
