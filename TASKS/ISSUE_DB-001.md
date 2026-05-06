---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] DB-001: Supabase 프로젝트 초기화 및 Next.js 연동 설정"
labels: 'feature, infra, database, priority:critical, sprint:0'
assignees: ''
---

## :dart: Summary
- **기능명:** [DB-001] Supabase 프로젝트 초기화 및 Next.js 연동 설정 (env, client)
- **목적:** MVP의 전체 데이터 계층(PostgreSQL, Auth, Storage, Realtime)을 담당하는 Supabase 프로젝트를 생성하고, Next.js App Router에서 서버/클라이언트 양측에서 안전하게 Supabase에 접근할 수 있는 클라이언트 설정을 확립한다. 이 태스크가 모든 DB 스키마 마이그레이션과 Auth 구현의 선행 조건이다.

## :link: References (Spec & Context)
> :bulb: AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`/SRS_v1.md#3.1-External-Systems`](../SRS_v1.md) — Supabase Pro 외부 시스템 정의
- SRS 문서: [`/SRS_v1.md#C-TEC-003`](../SRS_v1.md) — DB는 로컬 SQLite 개발, 배포 시 Supabase(PostgreSQL)
- SRS 문서: [`/SRS_v1.md#C-TEC-006`](../SRS_v1.md) — Supabase Pro 월 $25 예산 제약
- SRS 문서: [`/SRS_v1.md#3.0-Component-Diagram`](../SRS_v1.md) — Supabase 아키텍처 위치
- 구현 계획: [`/plans-prompt/Nextjs_MVP_Implementation_Plan.md#Phase-1`](../plans-prompt/Nextjs_MVP_Implementation_Plan.md)
- 태스크 분해: [`/TASKS/MVP_Task_Breakdown.md#DB-001`](./MVP_Task_Breakdown.md)

## :white_check_mark: Task Breakdown (실행 계획)

### Phase A: Supabase 프로젝트 생성
- [ ] [supabase.com](https://supabase.com) 에서 신규 프로젝트 생성 (리전: Northeast Asia — `ap-northeast-1` 또는 `ap-northeast-2`)
- [ ] 프로젝트 URL, Anon Key, Service Role Key 확보
- [ ] Supabase Dashboard에서 PostgreSQL 데이터베이스 접속 확인
- [ ] Pro 플랜 활성화 확인 (월 $25 예산 — `REQ-NF-024`)

### Phase B: Supabase CLI 설정 (로컬 개발 환경)
- [ ] `npx supabase init` 실행 → `supabase/` 디렉토리 생성
- [ ] `supabase/config.toml` 설정 (프로젝트 ID, API 포트 등)
- [ ] 로컬 Supabase 개발 환경 기동 테스트: `npx supabase start`
- [ ] 로컬 PostgreSQL 접속 확인 (psql 또는 Supabase Studio `localhost:54323`)

### Phase C: Next.js 연동 — Supabase Client 설정
- [ ] `@supabase/supabase-js` 및 `@supabase/ssr` 패키지 설치
- [ ] 환경변수 설정:
  ```env
  # .env.local
  NEXT_PUBLIC_SUPABASE_URL=https://<project-ref>.supabase.co
  NEXT_PUBLIC_SUPABASE_ANON_KEY=<anon-key>
  SUPABASE_SERVICE_ROLE_KEY=<service-role-key>  # 서버 전용, NEXT_PUBLIC 아님
  ```
- [ ] 서버 컴포넌트용 Supabase 클라이언트 생성 (`src/lib/supabase/server.ts`):
  ```typescript
  // createServerClient 패턴 (cookies 기반 세션 관리)
  import { createServerClient } from '@supabase/ssr'
  import { cookies } from 'next/headers'
  ```
- [ ] 클라이언트 컴포넌트용 Supabase 클라이언트 생성 (`src/lib/supabase/client.ts`):
  ```typescript
  // createBrowserClient 패턴
  import { createBrowserClient } from '@supabase/ssr'
  ```
- [ ] Server Actions용 클라이언트 팩토리 함수 (`src/lib/supabase/actions.ts`)
- [ ] 미들웨어에서 세션 갱신 로직 구현 (`src/middleware.ts`):
  ```typescript
  // Supabase Auth 세션 리프레시를 위한 미들웨어
  import { updateSession } from '@/lib/supabase/middleware'
  ```

### Phase D: 연동 검증
- [ ] 서버 컴포넌트에서 Supabase 접속 테스트 (간단한 SELECT 쿼리)
- [ ] 클라이언트 컴포넌트에서 Supabase 접속 테스트 (Anon Key 기반)
- [ ] Service Role Key가 클라이언트 번들에 노출되지 않음을 확인 (빌드 후 `.next/` 검증)
- [ ] `.env.local`이 `.gitignore`에 포함되어 있는지 확인

### Phase E: 타입 생성 파이프라인 설정
- [ ] `npx supabase gen types typescript --local > src/types/database.types.ts` 스크립트 추가
- [ ] `package.json`에 `"gen:types"` npm script 등록
- [ ] 생성된 타입이 Supabase 클라이언트에서 제네릭으로 적용되는지 확인

## :test_tube: Acceptance Criteria (BDD/GWT)

**Scenario 1: Supabase 서버 클라이언트 정상 연결**
- **Given:** `.env.local`에 유효한 Supabase URL과 Anon Key가 설정됨
- **When:** Server Component에서 `createServerClient`로 생성한 클라이언트가 `supabase.from('test').select()` 을 호출함
- **Then:** 에러 없이 응답을 반환해야 한다 (테이블 미존재 에러는 허용, 연결 에러는 불허).

**Scenario 2: 클라이언트 브라우저 클라이언트 정상 연결**
- **Given:** `NEXT_PUBLIC_*` 환경변수가 설정됨
- **When:** Client Component에서 `createBrowserClient`로 생성한 클라이언트가 Supabase에 요청함
- **Then:** 에러 없이 응답을 반환해야 한다.

**Scenario 3: Service Role Key 클라이언트 노출 차단**
- **Given:** `SUPABASE_SERVICE_ROLE_KEY`가 `.env.local`에 `NEXT_PUBLIC_` 접두어 없이 설정됨
- **When:** `npm run build` 후 `.next/` 빌드 산출물 전체를 문자열 검색함
- **Then:** Service Role Key 값이 클라이언트 번들 어디에도 포함되어 있지 않아야 한다.

**Scenario 4: 로컬 개발 환경 정상 기동**
- **Given:** `npx supabase init`이 완료됨
- **When:** `npx supabase start`를 실행함
- **Then:** 로컬 Supabase Studio (localhost:54323)에 접근 가능하고, 로컬 PostgreSQL이 가동 중이어야 한다.

**Scenario 5: TypeScript 타입 자동 생성**
- **Given:** Supabase 프로젝트에 최소 1개 이상의 테이블이 존재함
- **When:** `npm run gen:types`를 실행함
- **Then:** `src/types/database.types.ts`에 해당 테이블의 Row/Insert/Update 타입이 생성되어야 한다.

## :gear: Technical & Non-Functional Constraints
- **보안:** `SUPABASE_SERVICE_ROLE_KEY`는 절대 클라이언트 번들에 노출 금지 (`NEXT_PUBLIC_` 접두어 사용 불가)
- **세션 관리:** Supabase Auth의 cookie 기반 세션을 사용, JWT를 localStorage에 저장하지 않음
- **비용:** Supabase Pro 플랜 월 $25 이내 — `REQ-NF-024` 준수
- **리전:** 한국 사용자 대상이므로 아시아 리전 (`ap-northeast-1` 또는 `ap-northeast-2`) 선택
- **로컬 개발:** Docker Desktop이 필요할 수 있음 (Supabase CLI 로컬 환경)
- **타입 안전성:** Supabase 클라이언트는 반드시 `Database` 타입 제네릭을 사용할 것

## :checkered_flag: Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] Supabase 프로젝트가 생성되고 Pro 플랜이 활성화되었는가?
- [ ] 서버/클라이언트/Server Action 용 Supabase 클라이언트 3종이 구현되었는가?
- [ ] 미들웨어 세션 갱신 로직이 구현되었는가?
- [ ] TypeScript 타입 자동 생성 스크립트가 동작하는가?
- [ ] Service Role Key가 클라이언트에 노출되지 않는 것이 검증되었는가?
- [ ] `.env.local.example`에 필수 환경변수가 문서화되었는가?

## :construction: Dependencies & Blockers
- **Depends on:**
  - `INFRA-001` (Next.js 프로젝트 초기화 — `src/lib/supabase/` 디렉토리 필요)
  - `INFRA-003` (Supabase Pro 환경 구성 — 프로젝트 생성 동시 진행 가능)
- **Blocks:**
  - `DB-002` (USER 테이블 스키마 마이그레이션)
  - `DB-003` ~ `DB-007` (모든 테이블 마이그레이션)
  - `CMD-AUTH-001` (Supabase Auth 기반 회원가입)
  - `CMD-REVIEW-001` (Vercel AI SDK 초기화 — Supabase 클라이언트 참조)
