---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] DB-005: ALERT_RULE 테이블 스키마 및 마이그레이션"
labels: 'feature, database, priority:high, sprint:0'
assignees: ''
---

## :dart: Summary
- **기능명:** [DB-005] ALERT_RULE 테이블 스키마 및 마이그레이션 (threshold, cooldown_minutes, active)
- **목적:** F1(뇌동매매 방지 강제 제어)의 핵심 데이터인 알람 규칙을 저장하는 ALERT_RULE 테이블을 생성한다. 사용자가 설정하는 일일 최대 손실폭 임계치(0.5%~30%), 쿨타임 시간(기본 30분), 활성화 상태를 관리하며, CSV 업로드 트리거 시 누적 손실과 비교하는 기준값을 제공한다.

## :link: References (Spec & Context)
> :bulb: AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS ERD: [`/SRS_v1.md#6.2.1-ERD`](../05_SRS_v1.md) — `USER ||--o{ ALERT_RULE : "설정 1:N"`, `ALERT_RULE ||--o{ ALERT_LOG : "발동 기록 1:N"`
- SRS 문서: [`/SRS_v1.md#REQ-FUNC-001`](../05_SRS_v1.md) — 일일 최대 손실폭 슬라이더(0.5%~30%) 설정
- SRS 문서: [`/SRS_v1.md#REQ-FUNC-002`](../05_SRS_v1.md) — 범위 외 값 입력 시 인라인 에러
- SRS 문서: [`/SRS_v1.md#REQ-FUNC-004`](../05_SRS_v1.md) — 알람 발동 후 30분 쿨타임
- SRS 문서: [`/SRS_v1.md#REQ-FUNC-019`](../05_SRS_v1.md) — 일일 매매 횟수 상한 설정
- SRS 문서: [`/SRS_v1.md#REQ-NF-020`](../05_SRS_v1.md) — RLS 기반 사용자 간 데이터 격리
- 태스크 분해: [`/TASKS/06_MVP_Task_Breakdown.md#DB-005`](./06_MVP_Task_Breakdown.md)

## :white_check_mark: Task Breakdown (실행 계획)

### Phase A: 마이그레이션 파일 생성
- [ ] `npx supabase migration new create_alert_rules_table` 실행 → 마이그레이션 SQL 파일 생성
- [ ] 마이그레이션 파일명: `supabase/migrations/<timestamp>_create_alert_rules_table.sql`

### Phase B: ALERT_RULE 테이블 스키마 정의
- [ ] 아래 스키마를 마이그레이션 SQL로 작성:
  ```sql
  -- ALERT_RULE 테이블: 뇌동매매 방지 알람 규칙
  CREATE TABLE public.alert_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
    
    -- 손실폭 임계치 (0.5% ~ 30%)
    threshold DECIMAL(5, 2) NOT NULL
      CHECK (threshold >= 0.5 AND threshold <= 30.0),
    
    -- 쿨타임 설정 (분 단위, 기본 30분)
    cooldown_minutes INTEGER NOT NULL DEFAULT 30
      CHECK (cooldown_minutes >= 1 AND cooldown_minutes <= 1440),
    
    -- 매매횟수 상한 (일일 기준, NULL이면 미설정)
    max_trades_per_day INTEGER
      CHECK (max_trades_per_day IS NULL OR max_trades_per_day > 0),
    
    -- 활성화 상태
    active BOOLEAN NOT NULL DEFAULT true,
    
    -- 메타데이터
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
  );

  -- 인덱스
  CREATE INDEX idx_alert_rules_user_id ON public.alert_rules(user_id);
  CREATE INDEX idx_alert_rules_active ON public.alert_rules(user_id, active)
    WHERE active = true;

  -- updated_at 자동 갱신 트리거
  CREATE TRIGGER trigger_alert_rules_updated_at
    BEFORE UPDATE ON public.alert_rules
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
  ```

### Phase C: Row-Level Security 정책
- [ ] ALERT_RULE 테이블에 대한 RLS 정책 활성화:
  ```sql
  ALTER TABLE public.alert_rules ENABLE ROW LEVEL SECURITY;

  -- 사용자는 자신의 알람 규칙만 조회 가능
  CREATE POLICY "Users can view own alert rules"
    ON public.alert_rules FOR SELECT
    USING (auth.uid() = user_id);

  -- 사용자는 자신의 알람 규칙만 생성 가능
  CREATE POLICY "Users can create own alert rules"
    ON public.alert_rules FOR INSERT
    WITH CHECK (auth.uid() = user_id);

  -- 사용자는 자신의 알람 규칙만 수정 가능
  CREATE POLICY "Users can update own alert rules"
    ON public.alert_rules FOR UPDATE
    USING (auth.uid() = user_id)
    WITH CHECK (auth.uid() = user_id);

  -- 사용자는 자신의 알람 규칙만 삭제 가능
  CREATE POLICY "Users can delete own alert rules"
    ON public.alert_rules FOR DELETE
    USING (auth.uid() = user_id);
  ```

### Phase D: 마이그레이션 적용 및 검증
- [ ] 로컬: `npx supabase db reset` 으로 마이그레이션 적용 확인
- [ ] 원격: `npx supabase db push` 로 배포 환경 적용
- [ ] Supabase Studio에서 `public.alert_rules` 테이블 구조 확인
- [ ] TypeScript 타입 재생성: `npm run gen:types`

## :test_tube: Acceptance Criteria (BDD/GWT)

**Scenario 1: 마이그레이션 적용 성공**
- **Given:** DB-002까지의 마이그레이션이 적용되어 `public.users` 테이블이 존재함
- **When:** `npx supabase db reset`을 실행함
- **Then:** `public.alert_rules` 테이블이 모든 컬럼(id, user_id, threshold, cooldown_minutes, max_trades_per_day, active, created_at, updated_at)과 함께 생성되어야 한다.

**Scenario 2: 유효한 임계치 범위 INSERT 성공**
- **Given:** threshold = 5.0 (5%), cooldown_minutes = 30인 데이터가 주어짐
- **When:** `public.alert_rules`에 INSERT함
- **Then:** 레코드가 정상 생성되어야 한다.

**Scenario 3: threshold 하한 미만 CHECK 제약 위반**
- **Given:** threshold = 0.3 (0.3%, 하한 0.5% 미만)인 데이터가 주어짐
- **When:** INSERT를 시도함
- **Then:** CHECK 제약 위반 에러가 발생해야 한다.

**Scenario 4: threshold 상한 초과 CHECK 제약 위반**
- **Given:** threshold = 35.0 (35%, 상한 30% 초과)인 데이터가 주어짐
- **When:** INSERT를 시도함
- **Then:** CHECK 제약 위반 에러가 발생해야 한다.

**Scenario 5: max_trades_per_day NULL 허용 (미설정)**
- **Given:** max_trades_per_day를 설정하지 않은 데이터가 주어짐
- **When:** INSERT함
- **Then:** max_trades_per_day = NULL로 레코드가 생성되어야 한다.

**Scenario 6: RLS — 타 사용자 알람 규칙 접근 차단**
- **Given:** User A와 User B가 각각 알람 규칙을 보유함
- **When:** User A가 인증된 상태에서 `SELECT * FROM public.alert_rules`를 실행함
- **Then:** User A의 규칙만 반환되어야 한다.

**Scenario 7: active 필터 인덱스 효율성**
- **Given:** User A가 3개의 규칙을 보유 (2개 active, 1개 inactive)
- **When:** `WHERE user_id = ? AND active = true` 쿼리를 실행함
- **Then:** active인 2개의 규칙만 반환되어야 한다.

**Scenario 8: CASCADE 삭제 — 사용자 삭제 시 규칙 자동 삭제**
- **Given:** User A가 알람 규칙 3건을 보유함
- **When:** `public.users`에서 User A를 삭제함
- **Then:** 관련 alert_rules 3건이 CASCADE로 자동 삭제되어야 한다.

## :gear: Technical & Non-Functional Constraints
- **보안:** RLS 필수 활성화 — `REQ-NF-020` 준수
- **데이터 무결성:** threshold ∈ [0.5, 30.0], cooldown_minutes ∈ [1, 1440] CHECK 제약
- **비즈니스 규칙:** 0.5% 미만 또는 30% 초과 입력 거부 — `REQ-FUNC-002` 준수
- **기본값:** cooldown_minutes 기본 30분, active 기본 true
- **확장성:** max_trades_per_day NULL 허용으로 선택적 기능 활성화
- **타입 안전:** 마이그레이션 적용 후 반드시 `gen:types` 실행하여 TypeScript 타입 동기화

## :checkered_flag: Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 마이그레이션 SQL 파일이 `supabase/migrations/` 에 커밋되었는가?
- [ ] RLS 정책이 활성화되고 테스트되었는가?
- [ ] CHECK 제약 (threshold 범위, cooldown 범위)이 정상 동작하는가?
- [ ] CASCADE 삭제가 정상 동작하는가?
- [ ] `database.types.ts`가 갱신되어 `AlertRules` 타입이 포함되었는가?

## :construction: Dependencies & Blockers
- **Depends on:**
  - `DB-002` (USER 테이블 — `user_id FK → users.id` 참조)
- **Blocks:**
  - `DB-006` (ALERT_LOG 테이블 — `rule_id FK → alert_rules.id`)
  - `CMD-ALERT-001` (알람 규칙 생성 Server Action)
  - `CMD-ALERT-002` (임계치 비교 로직 — threshold 값 로드)
  - `CMD-ALERT-005` (매매횟수 브레이크 검증 — max_trades_per_day 참조)
  - `QRY-ALERT-001` (알람 규칙 목록 조회)
