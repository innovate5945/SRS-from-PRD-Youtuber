---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] INFRA-001: Next.js 프로젝트 초기화 및 디자인 시스템 구축"
labels: 'feature, infra, priority:critical, sprint:0'
assignees: ''
---

## :dart: Summary
- **기능명:** [INFRA-001] Next.js 15+ (App Router) 프로젝트 초기화 + Tailwind CSS + shadcn/ui 적용
- **목적:** 전체 MVP의 프론트엔드/백엔드 단일 코드베이스를 확립하고, 모든 후속 태스크가 참조하는 프로젝트 골격(Skeleton)과 디자인 시스템을 구축한다. 이 태스크가 완료되어야 DB/API/UI 관련 모든 작업이 병렬로 시작 가능하다.

## :link: References (Spec & Context)
> :bulb: AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`/SRS_v1.md#C-TEC-001`](../SRS_v1.md) — 단일 풀스택 프레임워크 제약
- SRS 문서: [`/SRS_v1.md#C-TEC-004`](../SRS_v1.md) — Tailwind CSS + shadcn/ui 일관성 강제
- SRS 문서: [`/SRS_v1.md#C-TEC-007`](../SRS_v1.md) — Vercel Git Push 자동 배포
- 구현 계획: [`/plans-prompt/Nextjs_MVP_Implementation_Plan.md#Phase-1`](../plans-prompt/Nextjs_MVP_Implementation_Plan.md)
- 태스크 분해: [`/TASKS/MVP_Task_Breakdown.md#INFRA-001`](./MVP_Task_Breakdown.md)

## :white_check_mark: Task Breakdown (실행 계획)

### Phase A: Next.js 프로젝트 초기화
- [ ] `npx -y create-next-app@latest ./` 실행 (App Router, TypeScript, ESLint, src/ 디렉토리 사용)
- [ ] Node.js 버전 고정 (`.nvmrc` 또는 `engines` 필드에 `>=20.0.0` 명시)
- [ ] `tsconfig.json` strict 모드 활성화 및 path alias (`@/*`) 설정 확인

### Phase B: 디자인 시스템 적용
- [ ] Tailwind CSS 4.x 설정 확인 (create-next-app 기본 포함 여부 검증)
- [ ] `npx shadcn@latest init` 실행 — 기본 테마(New York 스타일), CSS Variables 사용
- [ ] shadcn/ui 필수 공통 컴포넌트 설치: `Button`, `Input`, `Card`, `Dialog`, `Toast`, `Form`, `Label`, `Slider`, `Table`
- [ ] 글로벌 CSS (`globals.css`)에 커스텀 디자인 토큰 정의 (색상 팔레트: primary/destructive/warning)

### Phase C: 프로젝트 디렉토리 구조 확립
- [ ] 아래 디렉토리 구조를 생성하고 빈 `index.ts` (barrel export) 또는 `.gitkeep` 배치:
  ```
  src/
  ├── app/                    # App Router 페이지
  │   ├── (auth)/             # 인증 그룹 라우트
  │   ├── (dashboard)/        # 대시보드 그룹 라우트
  │   └── api/                # Route Handlers
  ├── components/             # UI 컴포넌트
  │   ├── ui/                 # shadcn/ui 자동 생성
  │   └── shared/             # 커스텀 공용 컴포넌트
  ├── lib/                    # 유틸리티, 클라이언트 설정
  │   ├── supabase/           # Supabase client (서버/브라우저)
  │   └── utils.ts            # cn() 등 공용 헬퍼
  ├── actions/                # Server Actions
  ├── types/                  # 공용 TypeScript 타입/DTO
  └── constants/              # 상수 정의
  ```

### Phase D: 개발 환경 품질 도구 설정
- [ ] ESLint 설정 확인 (Next.js 기본 + `@typescript-eslint` 추가)
- [ ] Prettier 설치 및 `.prettierrc` 설정 (singleQuote, semi, tabWidth: 2)
- [ ] `.env.local.example` 템플릿 생성 (NEXT_PUBLIC_SUPABASE_URL, SUPABASE_ANON_KEY, GEMINI_API_KEY 등)
- [ ] `.gitignore` 검증 (.env.local, node_modules, .next 등 포함 확인)

### Phase E: 빌드 및 개발 서버 검증
- [ ] `npm run dev` 실행 → localhost:3000 정상 접속 확인
- [ ] `npm run build` 실행 → 빌드 에러 0건 확인
- [ ] `npm run lint` 실행 → 경고/에러 0건 확인

## :test_tube: Acceptance Criteria (BDD/GWT)

**Scenario 1: 프로젝트 초기화 후 정상 기동**
- **Given:** Next.js 15+ App Router 프로젝트가 초기화됨
- **When:** `npm run dev`를 실행함
- **Then:** localhost:3000에서 기본 페이지가 정상 렌더링되고, 콘솔에 에러가 없어야 한다.

**Scenario 2: shadcn/ui 컴포넌트 렌더링 확인**
- **Given:** shadcn/ui 가 정상 설치됨
- **When:** 임의 페이지에서 `<Button>`, `<Card>`, `<Input>` 컴포넌트를 import하여 렌더링함
- **Then:** 스타일이 정상 적용된 상태로 화면에 표시되어야 한다.

**Scenario 3: TypeScript strict 모드 위반 시 빌드 실패**
- **Given:** `tsconfig.json`에 strict 모드가 활성화됨
- **When:** 타입 에러가 포함된 코드로 `npm run build` 실행함
- **Then:** 빌드가 실패하고 명확한 타입 에러 메시지가 출력되어야 한다.

**Scenario 4: 프로덕션 빌드 성공**
- **Given:** 모든 설정 완료 후
- **When:** `npm run build`를 실행함
- **Then:** 빌드가 에러 없이 성공하고, `.next/` 디렉토리에 산출물이 생성되어야 한다.

## :gear: Technical & Non-Functional Constraints
- **프레임워크:** Next.js 15+ (App Router) — `C-TEC-001` 준수
- **UI 라이브러리:** Tailwind CSS + shadcn/ui — `C-TEC-004` 준수, 커스텀 CSS 최소화
- **배포 환경:** Vercel Pro 호환 보장 — `C-TEC-007` 준수
- **빌드 시간:** 초기 cold build ≤ 60초 (CI 기준)
- **번들 크기:** 초기 페이지 JS 번들 ≤ 200KB (gzip)
- **패키지 매니저:** npm (lockfile 커밋 필수)

## :checkered_flag: Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] `npm run dev`, `npm run build`, `npm run lint` 모두 에러 없이 통과하는가?
- [ ] 디렉토리 구조가 명세대로 생성되었는가?
- [ ] shadcn/ui 필수 컴포넌트 9종이 설치 및 렌더링 가능한가?
- [ ] `.env.local.example` 템플릿이 생성되어 필수 환경변수 키가 문서화되었는가?
- [ ] README.md에 프로젝트 기동 방법이 기술되었는가?

## :construction: Dependencies & Blockers
- **Depends on:** 없음 (최우선 시작 태스크)
- **Blocks:**
  - `INFRA-002` (Vercel 배포 설정)
  - `INFRA-003` (Supabase 환경 구성)
  - `DB-001` (Supabase 프로젝트 초기화 및 Next.js 연동)
  - 모든 `UI-*` 태스크 (디자인 시스템 의존)
  - 모든 `CMD-*`, `QRY-*` 태스크 (Server Actions 디렉토리 의존)
