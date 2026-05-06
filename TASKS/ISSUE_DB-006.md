---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] DB-006: ALERT_LOG 테이블 스키마 및 마이그레이션"
labels: 'feature, database, priority:high, sprint:0'
assignees: ''
---

## :dart: Summary
- **기능명:** [DB-006] ALERT_LOG 테이블 스키마 및 마이그레이션 (rule_id FK, triggered_at)
- **목적:** 알람 규칙(ALERT_RULE)이 발동된 이력을 기록하는 ALERT_LOG 테이블을 생성한다. 알람 발동 시점, 당시 누적 손실률, 쿨타임 만료 시점을 저장하여, 쿨타임 잔여 시간 조회(QRY-ALERT-002) 및 주간 뇌동매매 리포트(CMD-ALERT-006)의 데이터 소스 역할을 한다.

## :link: References (Spec & Context)
> :bulb: AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS ERD: [`/SRS_v1.md#6.2.1-ERD`](../05_SRS_v1.md) — `ALERT_RULE ||--o{ ALERT_LOG : "발동 기록 1:N"`
- SRS 문서: [`/SRS_v1.md#REQ-FUNC-003`](../05_SRS_v1.md) — 임계치 도달 시 즉각 알람 발동
- SRS 문서: [`/SRS_v1.md#REQ-FUNC-004`](../05_SRS_v1.md) — 30분 쿨타임 경고 고정 표시
- SRS 문서: [`/SRS_v1.md#REQ-FUNC-005`](../05_SRS_v1.md) — 주간 뇌동매매 리포트 데이터 소스
- SRS 시퀀스: [`/SRS_v1.md#3.4.2`](../05_SRS_v1.md) — 데이터 업로드 즉시 알람 발동 흐름
- 태스크 분해: [`/TASKS/06_MVP_Task_Breakdown.md#DB-006`](./06_MVP_Task_Breakdown.md)

## :white_check_mark: Task Breakdown (실행 계획)

### Phase A: 마이그레이션 파일 생성
- [ ] `npx supabase migration new create_alert_logs_table` 실행 → 마이그레이션 SQL 파일 생성
- [ ] 마이그레이션 파일명: `supabase/migrations/<timestamp>_create_alert_logs_table.sql`

### Phase B: ALERT_LOG 테이블 스키마 정의
- [ ] 아래 스키마를 마이그레이션 SQL로 작성:
  ```sql
  -- ALERT_LOG 테이블: 알람 발동 이력
  CREATE TABLE public.alert_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rule_id UUID NOT NULL REFERENCES public.alert_rules(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
    
    -- 발동 시점 데이터
    triggered_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),      -- 알람 발동 시점
    cooldown_expires_at TIMESTAMPTZ NOT NULL,              -- 쿨타임 만료 시점
    
    -- 발동 당시 스냅샷
    actual_loss_pct DECIMAL(5, 2) NOT NULL,               -- 발동 당시 실제 누적 손실률 (%)
    threshold_pct DECIMAL(5, 2) NOT NULL,                  -- 발동 당시 설정된 임계치 (%)
    
    -- 알림 채널 상태
    email_sent BOOLEAN NOT NULL DEFAULT false,             -- 이메일 발송 여부
    email_sent_at TIMESTAMPTZ,                             -- 이메일 발송 시점
    
    -- 발동 유형
    alert_type VARCHAR(20) NOT NULL DEFAULT 'loss_limit'
      CHECK (alert_type IN ('loss_limit', 'trade_count')), -- 손실폭 / 매매횟수 구분
    
    -- 메타데이터
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
  );

  -- 인덱스
  CREATE INDEX idx_alert_logs_rule_id ON public.alert_logs(rule_id);
  CREATE INDEX idx_alert_logs_user_id ON public.alert_logs(user_id);
  CREATE INDEX idx_alert_logs_triggered_at ON public.alert_logs(triggered_at DESC);
  
  -- 쿨타임 잔여 시간 조회 최적화 (대시보드 배너용)
  CREATE INDEX idx_alert_logs_cooldown_active
    ON public.alert_logs(user_id, cooldown_expires_at DESC)
    WHERE cooldown_expires_at > NOW();
  
  -- 주간 리포트 집계용 인덱스
  CREATE INDEX idx_alert_logs_weekly
    ON public.alert_logs(user_id, triggered_at);
  ```

### Phase C: Row-Level Security 정책
- [ ] ALERT_LOG 테이블에 대한 RLS 정책 활성화:
  ```sql
  ALTER TABLE public.alert_logs ENABLE ROW LEVEL SECURITY;

  -- 사용자는 자신의 알람 로그만 조회 가능
  CREATE POLICY "Users can view own alert logs"
    ON public.alert_logs FOR SELECT
    USING (auth.uid() = user_id);

  -- 알람 로그 INSERT는 시스템(Server Action)에서만 수행
  -- Service Role로 INSERT하거나, user_id 기반 RLS 적용
  CREATE POLICY "Users can insert own alert logs"
    ON public.alert_logs FOR INSERT
    WITH CHECK (auth.uid() = user_id);
  ```

### Phase D: 마이그레이션 적용 및 검증
- [ ] 로컬: `npx supabase db reset` 으로 마이그레이션 적용 확인
- [ ] 원격: `npx supabase db push` 로 배포 환경 적용
- [ ] Supabase Studio에서 `public.alert_logs` 테이블 구조 확인
- [ ] TypeScript 타입 재생성: `npm run gen:types`

## :test_tube: Acceptance Criteria (BDD/GWT)

**Scenario 1: 마이그레이션 적용 성공**
- **Given:** DB-005까지의 마이그레이션이 적용되어 `public.alert_rules` 테이블이 존재함
- **When:** `npx supabase db reset`을 실행함
- **Then:** `public.alert_logs` 테이블이 모든 컬럼과 함께 생성되어야 한다.

**Scenario 2: 알람 발동 이력 INSERT 성공**
- **Given:** 유효한 rule_id, user_id, actual_loss_pct = 7.5, threshold_pct = 5.0이 주어짐
- **When:** `triggered_at = NOW()`, `cooldown_expires_at = NOW() + INTERVAL '30 minutes'`로 INSERT함
- **Then:** 레코드가 정상 생성되어야 한다.

**Scenario 3: 쿨타임 잔여 시간 조회**
- **Given:** 10분 전에 알람이 발동되어 cooldown_expires_at이 현재 시각 + 20분인 로그가 존재함
- **When:** `SELECT * FROM public.alert_logs WHERE user_id = ? AND cooldown_expires_at > NOW()` 실행
- **Then:** 해당 로그가 반환되어야 한다 (쿨타임 진행 중).

**Scenario 4: 쿨타임 만료 후 미반환**
- **Given:** 40분 전에 알람이 발동되어 cooldown_expires_at이 현재 시각 - 10분인 로그가 존재함
- **When:** `WHERE cooldown_expires_at > NOW()` 쿼리를 실행함
- **Then:** 해당 로그가 반환되지 않아야 한다 (쿨타임 만료).

**Scenario 5: RLS — 타 사용자 알람 로그 접근 차단**
- **Given:** User A와 User B가 각각 알람 로그를 보유함
- **When:** User A가 인증된 상태에서 `SELECT * FROM public.alert_logs`를 실행함
- **Then:** User A의 로그만 반환되어야 한다.

**Scenario 6: CASCADE 삭제 — 알람 규칙 삭제 시 로그 자동 삭제**
- **Given:** 특정 alert_rule에 연결된 알람 로그 5건이 존재함
- **When:** 해당 alert_rule을 삭제함
- **Then:** 연결된 5건의 alert_log가 CASCADE로 자동 삭제되어야 한다.

**Scenario 7: alert_type CHECK 제약 검증**
- **Given:** alert_type = 'invalid_type'인 데이터가 주어짐
- **When:** INSERT를 시도함
- **Then:** CHECK 제약 위반 에러가 발생해야 한다.

## :gear: Technical & Non-Functional Constraints
- **보안:** RLS 필수 활성화 — `REQ-NF-020` 준수
- **성능:** 쿨타임 잔여 시간 조회는 대시보드 진입 시 매번 실행되므로, 부분 인덱스로 최적화
- **데이터 보존:** 알람 로그는 주간 리포트 생성에 사용되므로, 최소 30일간 보관 권장
- **확장성:** alert_type 필드로 손실폭/매매횟수 등 다양한 알람 유형 구분
- **이메일 추적:** email_sent / email_sent_at 필드로 이메일 발송 상태 추적

## :checkered_flag: Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 마이그레이션 SQL 파일이 `supabase/migrations/` 에 커밋되었는가?
- [ ] RLS 정책이 활성화되고 테스트되었는가?
- [ ] rule_id FK → alert_rules.id, user_id FK → users.id 참조가 정상 동작하는가?
- [ ] cooldown_expires_at 기반 쿨타임 조회가 인덱스를 활용하는가?
- [ ] `database.types.ts`가 갱신되어 `AlertLogs` 타입이 포함되었는가?

## :construction: Dependencies & Blockers
- **Depends on:**
  - `DB-005` (ALERT_RULE 테이블 — `rule_id FK → alert_rules.id` 참조)
  - `DB-002` (USER 테이블 — `user_id FK → users.id` 참조)
- **Blocks:**
  - `CMD-ALERT-003` (알람 발동 시 ALERT_LOG 기록 + 쿨타임 시작)
  - `CMD-ALERT-004` (이메일 알람 발송 — email_sent 상태 업데이트)
  - `QRY-ALERT-002` (쿨타임 잔여 시간 조회)
  - `QRY-DASH-002` (주간/월간 리포트 데이터 집계)
