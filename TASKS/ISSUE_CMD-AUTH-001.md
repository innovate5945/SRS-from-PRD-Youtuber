---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] CMD-AUTH-001: Supabase Auth 기반 이메일 회원가입 Server Action 구현"
labels: 'feature, backend, auth, priority:high, sprint:1'
assignees: ''
---

## :dart: Summary
- **기능명:** [CMD-AUTH-001] Supabase Auth 기반 이메일 회원가입 Server Action 구현
- **목적:** 사용자가 서비스에 접근하기 위한 고유 계정을 안전하게 생성한다. Supabase Auth의 이메일/비밀번호 인증을 활용하여 `auth.users` 테이블에 레코드를 생성하고, 트리거를 통해 `public.users` 프로필이 자동 생성되는 전체 회원가입 파이프라인을 Next.js Server Action으로 구현한다.

## :link: References (Spec & Context)
> :bulb: AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`/SRS_v1.md#6.1-API-#1`](../SRS_v1.md) — `POST /api/v1/auth/signup` (인증 없음)
- SRS 문서: [`/SRS_v1.md#3.3-actions/auth.ts`](../SRS_v1.md) — 사용자 인증 및 세션 관리 모듈
- SRS 문서: [`/SRS_v1.md#REQ-NF-023a`](../SRS_v1.md) — 개인정보보호법 동의 이력 관리
- SRS ERD: [`/SRS_v1.md#6.2-USER`](../SRS_v1.md) — USER 엔티티 (id, email)
- 시퀀스 다이어그램: Supabase Auth → `on_auth_user_created` 트리거 → `public.users` INSERT
- DTO 명세: [`/TASKS/ISSUE_API-001.md`](./ISSUE_API-001.md) — Auth 도메인 DTO (미작성 시 본 이슈 내 인라인 정의)
- 태스크 분해: [`/TASKS/MVP_Task_Breakdown.md#CMD-AUTH-001`](./MVP_Task_Breakdown.md)

## :white_check_mark: Task Breakdown (실행 계획)

### Phase A: Auth DTO 및 검증 스키마 정의
- [ ] `src/types/auth.dto.ts` 파일 생성 (또는 API-001에서 작성된 파일 활용):
  ```typescript
  // === Sign Up DTO ===
  export interface SignUpRequest {
    email: string;
    password: string;
    display_name?: string;
    /** 개인정보 처리 동의 (필수) */
    privacy_consent: boolean;
    /** 마케팅 수신 동의 (선택) */
    marketing_consent?: boolean;
  }
  
  export interface SignUpSuccessResponse {
    success: true;
    data: {
      user_id: string;
      email: string;
    };
    message: string;
  }
  
  export interface SignUpErrorResponse {
    success: false;
    error: {
      code: SignUpErrorCode;
      message: string;
    };
  }
  
  export type SignUpErrorCode =
    | 'INVALID_EMAIL'           // 이메일 형식 오류
    | 'WEAK_PASSWORD'           // 비밀번호 보안 정책 미충족
    | 'DUPLICATE_EMAIL'         // 이미 가입된 이메일
    | 'PRIVACY_CONSENT_REQUIRED' // 개인정보 동의 누락
    | 'AUTH_SERVICE_ERROR';     // Supabase Auth 서비스 오류
  ```
- [ ] Zod 검증 스키마 정의 (`src/types/auth.schema.ts`):
  ```typescript
  import { z } from 'zod';
  
  export const signUpSchema = z.object({
    email: z
      .string()
      .email('유효한 이메일 형식을 입력해주세요.')
      .max(255, '이메일은 255자 이내여야 합니다.'),
    password: z
      .string()
      .min(8, '비밀번호는 최소 8자 이상이어야 합니다.')
      .max(72, '비밀번호는 72자 이내여야 합니다.')
      .regex(/[A-Z]/, '영문 대문자를 1자 이상 포함해야 합니다.')
      .regex(/[a-z]/, '영문 소문자를 1자 이상 포함해야 합니다.')
      .regex(/[0-9]/, '숫자를 1자 이상 포함해야 합니다.')
      .regex(/[^A-Za-z0-9]/, '특수문자를 1자 이상 포함해야 합니다.'),
    display_name: z.string().max(100).optional(),
    privacy_consent: z.literal(true, {
      errorMap: () => ({ message: '개인정보 처리 방침에 동의해주세요.' }),
    }),
    marketing_consent: z.boolean().optional().default(false),
  });
  
  export type SignUpInput = z.infer<typeof signUpSchema>;
  ```

### Phase B: Server Action 구현
- [ ] `src/actions/auth.ts` 파일 생성:
  ```typescript
  'use server';
  
  import { createServerActionClient } from '@/lib/supabase/actions';
  import { signUpSchema } from '@/types/auth.schema';
  import type { SignUpSuccessResponse, SignUpErrorResponse } from '@/types/auth.dto';
  
  export async function signUp(
    formData: FormData
  ): Promise<SignUpSuccessResponse | SignUpErrorResponse> {
    // 1. 입력 값 추출 및 Zod 검증
    const rawInput = {
      email: formData.get('email') as string,
      password: formData.get('password') as string,
      display_name: formData.get('display_name') as string | undefined,
      privacy_consent: formData.get('privacy_consent') === 'true',
      marketing_consent: formData.get('marketing_consent') === 'true',
    };
    
    const validation = signUpSchema.safeParse(rawInput);
    if (!validation.success) {
      // 첫 번째 에러 반환
      return { success: false, error: { code: 'INVALID_EMAIL', message: validation.error.issues[0].message } };
    }
    
    const { email, password, display_name, privacy_consent, marketing_consent } = validation.data;
    
    // 2. Supabase Auth 회원가입 호출
    const supabase = await createServerActionClient();
    const { data, error } = await supabase.auth.signUp({
      email,
      password,
      options: {
        data: {
          display_name,
        },
      },
    });
    
    // 3. 에러 처리
    if (error) {
      if (error.message.includes('already registered')) {
        return { success: false, error: { code: 'DUPLICATE_EMAIL', message: '이미 가입된 이메일입니다.' } };
      }
      return { success: false, error: { code: 'AUTH_SERVICE_ERROR', message: '회원가입 처리 중 오류가 발생했습니다.' } };
    }
    
    // 4. 개인정보 동의 이력 저장 (public.users 업데이트)
    if (data.user) {
      await supabase
        .from('users')
        .update({
          display_name,
          privacy_consent_at: new Date().toISOString(),
          marketing_consent_at: marketing_consent ? new Date().toISOString() : null,
        })
        .eq('id', data.user.id);
    }
    
    // 5. 성공 응답
    return {
      success: true,
      data: { user_id: data.user!.id, email },
      message: '회원가입이 완료되었습니다.',
    };
  }
  ```

### Phase C: 비밀번호 보안 정책 검증
- [ ] 비밀번호 정책 확인:
  - 최소 8자, 최대 72자 (Bcrypt 제한)
  - 영문 대문자 1자 이상 포함
  - 영문 소문자 1자 이상 포함
  - 숫자 1자 이상 포함
  - 특수문자 1자 이상 포함
- [ ] Supabase Dashboard → Auth → Settings에서 비밀번호 정책 동기화 확인
- [ ] 비밀번호 평문 로깅 절대 금지 확인 (Server Action 내 `console.log` 등에 password 노출 방지)

### Phase D: 에러 처리 및 보안 고려
- [ ] Supabase Auth 에러 코드를 앱 에러 코드로 매핑하는 핸들러 구현
- [ ] Rate Limiting 고려: Supabase Auth의 기본 Rate Limit 확인 (기본 60req/분)
- [ ] 이메일 확인(Email Confirmation) 전략 결정:
  - MVP: 이메일 확인 비활성화 (즉시 로그인 가능) — 빠른 온보딩 우선
  - 또는: 이메일 확인 활성화 (보안 강화) — Supabase Dashboard 설정
- [ ] 에러 응답에 비밀번호 관련 상세 정보 노출 금지 (타이밍 공격 방지)

### Phase E: Server Action 호출 인터페이스 검증
- [ ] `useFormState` (React 19) 또는 `useActionState`를 통한 Server Action 호출 패턴 확인
- [ ] FormData 기반 호출과 직접 호출(객체 전달) 양쪽 지원 검증
- [ ] 에러 상태의 프론트엔드 전달 구조 확인 (revalidation 불필요, 직접 반환)

## :test_tube: Acceptance Criteria (BDD/GWT)

**Scenario 1: 정상적인 회원가입**
- **Given:** 유효한 형태의 이메일(`newuser@example.com`)과 보안 정책을 충족하는 비밀번호(`Test1234!`)가 주어짐
- **When:** `signUp` Server Action을 호출함
- **Then:**
  - `auth.users`에 유저가 생성되고,
  - `public.users`에 트리거를 통해 프로필이 생성되며,
  - `privacy_consent_at`이 기록되고,
  - `success: true`와 함께 `user_id`, `email`을 반환한다.

**Scenario 2: 중복된 이메일 가입 시도**
- **Given:** 이미 DB에 존재하는 이메일(`exist@example.com`)이 주어짐
- **When:** 해당 이메일로 `signUp` Server Action을 호출함
- **Then:** `success: false`, `code: 'DUPLICATE_EMAIL'`, 메시지 "이미 가입된 이메일입니다."를 반환한다.

**Scenario 3: 비밀번호 보안 정책 미충족**
- **Given:** 8자 미만의 비밀번호(`abc`)가 주어짐
- **When:** `signUp` Server Action을 호출함
- **Then:** `success: false`, `code: 'WEAK_PASSWORD'`에 해당하는 에러와 함께 정확한 부족 사항을 안내한다.

**Scenario 4: 이메일 형식 오류**
- **Given:** 유효하지 않은 이메일 형식(`not-an-email`)이 주어짐
- **When:** `signUp` Server Action을 호출함
- **Then:** `success: false`, `code: 'INVALID_EMAIL'`, 메시지 "유효한 이메일 형식을 입력해주세요."를 반환한다.

**Scenario 5: 개인정보 동의 미체크**
- **Given:** `privacy_consent: false`인 상태로 회원가입 요청이 주어짐
- **When:** `signUp` Server Action을 호출함
- **Then:** `success: false`, `code: 'PRIVACY_CONSENT_REQUIRED'`, 메시지 "개인정보 처리 방침에 동의해주세요."를 반환한다.

**Scenario 6: 비밀번호 평문 로깅 방지**
- **Given:** 회원가입 Server Action이 실행됨
- **When:** 서버 로그를 확인함
- **Then:** 비밀번호 원문이 어떠한 로그에도 기록되어 있지 않아야 한다.

## :gear: Technical & Non-Functional Constraints
- **성능:** 회원가입 API 응답 시간 p95 ≤ 500ms (Supabase Auth 호출 포함)
- **보안:**
  - 비밀번호 평문 저장 절대 금지 (Supabase Auth 내부 Bcrypt 처리)
  - 비밀번호 평문 로깅 절대 금지
  - Service Role Key 미사용 (회원가입은 Anon Key + Auth API 사용)
  - 에러 응답에서 "이메일 존재 여부" 노출 최소화 (향후 보안 강화 시 고려)
- **컴플라이언스:** 개인정보보호법(PIPA) — `REQ-NF-023a`
  - 개인정보 처리 동의 필수 (privacy_consent = true)
  - 동의 일시 기록 (privacy_consent_at TIMESTAMPTZ)
  - 마케팅 동의는 선택 (marketing_consent 옵셔널)
- **프레임워크:** Next.js Server Actions (`'use server'` 디렉티브) — `C-TEC-002`
- **안정성:** Supabase Auth 서비스 장애 시 적절한 fallback 에러 메시지 반환

## :checkered_flag: Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] `src/actions/auth.ts`에 `signUp` Server Action이 구현되었는가?
- [ ] `src/types/auth.dto.ts` DTO와 `src/types/auth.schema.ts` Zod 스키마가 작성되었는가?
- [ ] 단위 테스트: Zod 스키마 검증 테스트 (유효/무효 입력 6가지 이상)가 통과하는가?
- [ ] 통합 테스트: 실제 Supabase Auth 호출을 포함한 E2E 흐름이 동작하는가?
- [ ] 비밀번호 평문이 로그 어디에도 노출되지 않는 것이 확인되었는가?
- [ ] `npm run build` 시 타입 에러가 없는가?

## :construction: Dependencies & Blockers
- **Depends on:**
  - `DB-002` (USER 테이블 + `on_auth_user_created` 트리거 — 프로필 자동 생성)
  - `DB-001` (Supabase 클라이언트 설정 — `createServerActionClient`)
  - `API-001` (Auth DTO 정의 — 본 이슈 Phase A에서 인라인 구현 가능)
- **Blocks:**
  - `CMD-AUTH-002` (로그인 및 세션 관리 — 가입된 사용자 필요)
  - `UI-AUTH-001` (회원가입/로그인 페이지 UI — Server Action 호출 인터페이스)
  - `MOCK-001` (Auth Mock API — 실제 구현 후 Mock 전환 불필요할 수 있음)
