---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] DB-008: 전 테이블 Row-Level Security (RLS) 정책 작성"
labels: 'feature, database, security, priority:critical, sprint:0'
assignees: ''
---

## :dart: Summary
- **기능명:** [DB-008] 전 테이블 Row-Level Security (RLS) 정책 작성 (user_id 기반 격리)
- **목적:** DB-002~DB-007에서 개별적으로 적용한 RLS 정책을 통합 검증하고, 누락된 정책을 보완하여 전 테이블에 걸쳐 일관된 사용자 간 데이터 격리를 보장한다. 교차 사용자 접근 시나리오를 체계적으로 테스트하여 `REQ-NF-020`(Supabase RLS 기반 데이터 격리)을 100% 충족시킨다.

## :link: References (Spec & Context)
> :bulb: AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`/SRS_v1.md#REQ-NF-020`](../05_SRS_v1.md) — Supabase RLS를 통해 사용자 간 데이터 접근을 엄격히 격리
- SRS 문서: [`/SRS_v1.md#REQ-NF-018`](../05_SRS_v1.md) — TLS 1.3 전 트래픽 암호화
- SRS 문서: [`/SRS_v1.md#REQ-NF-023a`](../05_SRS_v1.md) — 개인정보보호법 준수
- 선행 태스크: [`/TASKS/ISSUE_DB-002.md`](./ISSUE_DB-002.md) ~ [`/TASKS/ISSUE_DB-007.md`](./ISSUE_DB-007.md)
- 태스크 분해: [`/TASKS/06_MVP_Task_Breakdown.md#DB-008`](./06_MVP_Task_Breakdown.md)

## :white_check_mark: Task Breakdown (실행 계획)

### Phase A: 전 테이블 RLS 활성화 상태 감사
- [ ] 아래 쿼리로 전 테이블의 RLS 활성화 상태 확인:
  ```sql
  SELECT schemaname, tablename, rowsecurity
  FROM pg_tables
  WHERE schemaname = 'public'
  ORDER BY tablename;
  ```
- [ ] RLS 비활성화 테이블이 있으면 즉시 활성화
- [ ] 각 테이블별 적용된 정책 목록 확인:
  ```sql
  SELECT schemaname, tablename, policyname, permissive, roles, cmd, qual, with_check
  FROM pg_policies
  WHERE schemaname = 'public'
  ORDER BY tablename, policyname;
  ```

### Phase B: RLS 정책 완전성 검증 매트릭스
- [ ] 아래 매트릭스에 따라 각 테이블별 정책 완전성 검증:

  | 테이블 | SELECT | INSERT | UPDATE | DELETE | 격리 기준 |
  |---|---|---|---|---|---|
  | `users` | ✅ own | — (Auth 트리거) | ✅ own | — (Auth CASCADE) | `auth.uid() = id` |
  | `accounts` | ✅ own | ✅ own | ✅ own | — (CASCADE) | `auth.uid() = user_id` |
  | `trades` | ✅ own account | ✅ own account | — (immutable) | — (CASCADE) | `account_id ∈ own accounts` |
  | `upload_files` | ✅ own account | ✅ own account | — (immutable) | — (CASCADE) | `account_id ∈ own accounts` |
  | `alert_rules` | ✅ own | ✅ own | ✅ own | ✅ own | `auth.uid() = user_id` |
  | `alert_logs` | ✅ own | ✅ own | — (immutable) | — (immutable) | `auth.uid() = user_id` |
  | `trade_reviews` | ✅ own | ✅ own | ✅ own | — (CASCADE) | `auth.uid() = user_id` |

### Phase C: 누락 정책 보완 마이그레이션
- [ ] `npx supabase migration new rls_policies_audit` 실행
- [ ] Phase B 매트릭스에서 누락된 정책이 있으면 보완 SQL 작성
- [ ] 불필요한 과도한 정책이 있으면 제거 (최소 권한 원칙)
- [ ] 비인증 사용자(anon role) 접근 차단 확인:
  ```sql
  -- 비인증 사용자는 모든 public 테이블 접근 불가 확인
  -- RLS가 활성화된 상태에서 정책이 없으면 기본적으로 차단됨
  ```

### Phase D: 교차 사용자 접근 차단 통합 테스트 스크립트
- [ ] 테스트 SQL 스크립트 작성 (`supabase/tests/rls_test.sql`):
  ```sql
  -- 테스트 시나리오: 2명의 사용자 생성 후 교차 접근 시도

  -- 1. User A로 세션 설정
  -- SET LOCAL ROLE authenticated;
  -- SET LOCAL request.jwt.claim.sub = '<user_a_id>';

  -- 2. User A 데이터 조회 → 자신의 데이터만 반환 확인
  -- 3. User B 데이터 직접 조회 시도 → 0건 반환 확인

  -- 4. User B로 세션 전환
  -- SET LOCAL request.jwt.claim.sub = '<user_b_id>';

  -- 5. User A의 데이터 접근 시도 → 0건 반환 확인
  -- 6. User B가 User A의 account_id로 trade INSERT 시도 → RLS 위반 차단
  ```

### Phase E: 서비스 롤 키 사용 정책 문서화
- [ ] Service Role Key는 RLS를 우회하므로, 사용 가능 범위를 문서화:
  - ✅ 허용: 마이그레이션, 관리자 스크립트, 데이터 파기
  - ❌ 금지: 일반 Server Action, 클라이언트 번들
- [ ] Server Action에서는 반드시 `createServerClient` (Anon Key + 사용자 세션) 사용

### Phase F: 마이그레이션 적용 및 최종 검증
- [ ] 로컬: `npx supabase db reset` 으로 전체 마이그레이션 재적용
- [ ] 원격: `npx supabase db push` 로 배포 환경 적용
- [ ] Phase D 테스트 스크립트 실행 → 전 시나리오 통과 확인

## :test_tube: Acceptance Criteria (BDD/GWT)

**Scenario 1: 전 테이블 RLS 활성화 확인**
- **Given:** DB-002~DB-007 마이그레이션이 모두 적용됨
- **When:** `pg_tables`에서 public 스키마 테이블의 `rowsecurity` 값을 확인함
- **Then:** 모든 public 테이블의 `rowsecurity`가 `true`여야 한다.

**Scenario 2: User A가 User B의 trades 조회 시도**
- **Given:** User A(alice)와 User B(bob)가 각각 매매 내역을 보유함
- **When:** User A가 인증된 상태에서 `SELECT * FROM public.trades` 실행
- **Then:** User A의 계좌에 속한 trades만 반환되고, User B의 trades는 0건이어야 한다.

**Scenario 3: User A가 User B의 alert_rules 수정 시도**
- **Given:** User B의 alert_rule (id = xxx)이 존재함
- **When:** User A가 인증된 상태에서 `UPDATE public.alert_rules SET threshold = 1.0 WHERE id = 'xxx'` 실행
- **Then:** 0건이 수정되어야 한다 (RLS가 접근을 차단).

**Scenario 4: User A가 User B의 account_id로 trade INSERT 시도**
- **Given:** User B의 account (id = yyy)이 존재함
- **When:** User A가 인증된 상태에서 `INSERT INTO public.trades (account_id, ...) VALUES ('yyy', ...)` 시도
- **Then:** RLS 정책 위반으로 INSERT가 차단되어야 한다.

**Scenario 5: 비인증 사용자(anon) 접근 차단**
- **Given:** anon 역할(비인증 상태)로 세션이 설정됨
- **When:** `SELECT * FROM public.users` (또는 다른 테이블) 실행
- **Then:** 0건이 반환되어야 한다 (RLS가 접근을 차단).

**Scenario 6: User A가 User B의 trade_reviews 조회 시도**
- **Given:** User B가 복기 일지를 보유함
- **When:** User A가 인증된 상태에서 `SELECT * FROM public.trade_reviews` 실행
- **Then:** User A의 복기 일지만 반환되고, User B의 복기 일지는 조회 불가해야 한다.

**Scenario 7: Service Role Key의 RLS 우회 동작**
- **Given:** Service Role Key로 Supabase 클라이언트가 생성됨
- **When:** `SELECT * FROM public.users` 실행
- **Then:** 전체 사용자 데이터가 반환되어야 한다 (관리자 접근).

## :gear: Technical & Non-Functional Constraints
- **보안:** 전 테이블 RLS 100% 활성화 필수 — `REQ-NF-020` 핵심 요구사항
- **최소 권한:** 각 테이블에 필요한 최소한의 정책만 적용 (SELECT, INSERT만 필요하면 UPDATE/DELETE 정책 생략)
- **성능:** RLS 서브쿼리 (trades의 `account_id IN (SELECT ...)`)는 인덱스를 활용해야 함
- **컴플라이언스:** 개인정보보호법(PIPA) 준수 — 사용자 간 데이터 완전 격리 보장
- **운영:** Service Role Key 사용은 관리자 작업으로 제한, 일반 Server Action은 사용자 세션 기반

## :checkered_flag: Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 전 테이블의 RLS가 활성화(`rowsecurity = true`)되었는가?
- [ ] 교차 사용자 접근 테스트 전 시나리오가 통과하는가?
- [ ] 비인증 사용자(anon) 접근이 차단되는가?
- [ ] 불필요한 과도한 정책이 제거되었는가?
- [ ] Service Role Key 사용 정책이 문서화되었는가?
- [ ] 테스트 스크립트가 `supabase/tests/` 에 커밋되었는가?

## :construction: Dependencies & Blockers
- **Depends on:**
  - `DB-002` (USER 테이블 + RLS 기본 정책)
  - `DB-003` (ACCOUNT 테이블 + RLS 정책)
  - `DB-004` (TRADE 테이블 + RLS 정책)
  - `DB-005` (ALERT_RULE 테이블 + RLS 정책)
  - `DB-006` (ALERT_LOG 테이블 + RLS 정책)
  - `DB-007` (TRADE_REVIEW 테이블 + RLS 정책)
- **Blocks:**
  - `SEC-002` (RLS 교차 사용자 접근 차단 검증 테스트 — 본 태스크의 통합 테스트 확장)
  - 모든 `CMD-*`, `QRY-*` 태스크 (RLS가 활성화된 환경에서 Server Action 동작 보장)
