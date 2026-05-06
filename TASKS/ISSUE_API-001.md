---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] API-001: Auth 도메인 Request/Response DTO 정의"
labels: 'feature, backend, api-contract, priority:high, sprint:0'
assignees: ''
---

## :dart: Summary
- **기능명:** [API-001] Auth 도메인 Request/Response DTO 정의 (signup, login) 및 에러 코드 (401, 409)
- **목적:** 인증(Auth) 도메인의 프론트엔드/백엔드 간 데이터 통신 계약을 TypeScript DTO로 선제 정의한다. 회원가입/로그인의 Request/Response 구조, Zod 검증 스키마, 에러 코드 체계를 확립하여 병렬 개발의 SSOT를 구축한다.

## :link: References (Spec & Context)
> :bulb: AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`/SRS_v1.md#6.1-API-#1~2`](../05_SRS_v1.md) — signup/login 엔드포인트 정의
- SRS 문서: [`/SRS_v1.md#3.3-actions/auth.ts`](../05_SRS_v1.md) — 사용자 인증 및 세션 관리 모듈
- SRS 문서: [`/SRS_v1.md#REQ-NF-023a`](../05_SRS_v1.md) — 개인정보보호법 동의 이력 관리
- SRS ERD: [`/SRS_v1.md#6.2-USER`](../05_SRS_v1.md) — USER 엔티티
- 태스크 분해: [`/TASKS/06_MVP_Task_Breakdown.md#API-001`](./06_MVP_Task_Breakdown.md)

## :white_check_mark: Task Breakdown (실행 계획)

### Phase A: Auth DTO 파일 생성 — SignUp
- [ ] `src/types/auth.dto.ts` 파일 생성
- [ ] SignUpRequest: email, password, display_name?, privacy_consent, marketing_consent?
- [ ] SignUpSuccessResponse: success, data(user_id, email), message
- [ ] SignUpErrorResponse: success, error(code, message)

### Phase B: Auth DTO — Login
- [ ] LoginRequest: email, password
- [ ] LoginSuccessResponse: success, data(user_id, email, display_name?, expires_at), message
- [ ] LoginErrorResponse: success, error(code, message)

### Phase C: 에러 코드 체계 정의
- [ ] AuthErrorCode 유니온 타입 정의:
  - 공통: INVALID_EMAIL(400), AUTH_SERVICE_ERROR(500)
  - 회원가입: WEAK_PASSWORD(400), DUPLICATE_EMAIL(409), PRIVACY_CONSENT_REQUIRED(400)
  - 로그인: INVALID_CREDENTIALS(401), ACCOUNT_DISABLED(403), TOO_MANY_REQUESTS(429)
- [ ] AUTH_ERROR_STATUS_MAP: 에러 코드 → HTTP 상태 코드 매핑
- [ ] AUTH_ERROR_MESSAGES: 에러 코드 → 한국어 사용자 메시지 매핑

### Phase D: Zod 런타임 검증 스키마
- [ ] `src/types/auth.schema.ts` 파일 생성
- [ ] signUpSchema: email(max255), password(8~72자, 대소문자+숫자+특수문자), privacy_consent(literal true)
- [ ] loginSchema: email(이메일 형식), password(min 1자)
- [ ] PASSWORD_POLICY 상수 정의 (MIN_LENGTH:8, MAX_LENGTH:72)
- [ ] SignUpInput, LoginInput 추론 타입 export

### Phase E: 통합 타입 유틸리티
- [ ] SignUpResponse = SignUpSuccessResponse | SignUpErrorResponse
- [ ] LoginResponse = LoginSuccessResponse | LoginErrorResponse
- [ ] AuthResponse union 타입

## :test_tube: Acceptance Criteria (BDD/GWT)

**Scenario 1: DTO 타입 Import 정상 동작**
- **Given:** `src/types/auth.dto.ts`가 작성됨
- **When:** 다른 파일에서 `import { SignUpRequest, AuthErrorCode }` 수행
- **Then:** TypeScript 컴파일 에러 없이 참조되어야 한다.

**Scenario 2: signUpSchema — 유효한 입력 검증 성공**
- **Given:** email: 'test@example.com', password: 'Test1234!', privacy_consent: true
- **When:** `signUpSchema.safeParse()` 실행
- **Then:** `success: true` 반환.

**Scenario 3: signUpSchema — 비밀번호 정책 위반**
- **Given:** password: 'abc' (8자 미만)
- **When:** `signUpSchema.safeParse()` 실행
- **Then:** `success: false`, 관련 에러 메시지 포함.

**Scenario 4: signUpSchema — 개인정보 동의 미체크**
- **Given:** privacy_consent: false
- **When:** `signUpSchema.safeParse()` 실행
- **Then:** `success: false`, '개인정보 처리 방침에 동의해주세요.' 메시지.

**Scenario 5: 에러 코드 완전성**
- **Given:** AuthErrorCode, AUTH_ERROR_MESSAGES, AUTH_ERROR_STATUS_MAP이 정의됨
- **When:** SRS §6.1 #1~2 에러 분기 확인
- **Then:** 각 분기에 대응하는 코드·메시지·HTTP 상태가 누락 없어야 한다.

## :gear: Technical & Non-Functional Constraints
- **타입 안전:** TypeScript strict 모드 에러 없음
- **런타임 검증:** 서버 사이드 Zod 스키마 필수
- **보안:** 에러 응답에서 계정 존재 여부 노출 최소화
- **국제화:** 에러 메시지 한국어 (MVP 국내 타겟)
- **비밀번호:** Bcrypt 제한 최대 72자

## :checkered_flag: Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria 충족
- [ ] `src/types/auth.dto.ts` 생성 및 DTO export
- [ ] `src/types/auth.schema.ts` 생성 및 Zod 스키마 동작
- [ ] AUTH_ERROR_MESSAGES/AUTH_ERROR_STATUS_MAP 완전성
- [ ] `npm run build` 타입 에러 없음

## :construction: Dependencies & Blockers
- **Depends on:**
  - `INFRA-001` (프로젝트 초기화 — `src/types/` 디렉토리)
- **Blocks:**
  - `MOCK-001` (Auth Mock 엔드포인트)
  - `CMD-AUTH-001` (회원가입 Server Action)
  - `CMD-AUTH-002` (로그인 Server Action)
  - `UI-AUTH-001` (회원가입/로그인 페이지 UI)
