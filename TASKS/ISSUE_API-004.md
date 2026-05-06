---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] API-004: Alert Rules CRUD DTO 정의"
labels: 'feature, backend, api-contract, priority:high, sprint:0'
assignees: ''
---

## :dart: Summary
- **기능명:** [API-004] Alert Rules CRUD DTO 정의 (GET/POST /alerts/rules — threshold 범위 검증)
- **목적:** F1(뇌동매매 방지) 알람 규칙 CRUD API의 Request/Response 구조를 TypeScript DTO로 정의한다. 손실폭 임계치(0.5%~30%) 범위 검증, 매매횟수 상한, 쿨타임 설정의 DTO와 Zod 스키마를 확립하여 CMD-ALERT-001, QRY-ALERT-001, UI-ALERT-001이 참조할 SSOT를 구축한다.

## :link: References (Spec & Context)
> :bulb: AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`/SRS_v1.md#6.1-API-#5~6`](../05_SRS_v1.md) — `GET/POST /api/v1/alerts/rules`
- SRS 문서: [`/SRS_v1.md#3.3-actions/alerts.ts`](../05_SRS_v1.md) — 알람 규칙 CRUD 및 계산 모듈
- SRS 문서: [`/SRS_v1.md#REQ-FUNC-001`](../05_SRS_v1.md) — 손실폭 슬라이더 0.5%~30%
- SRS 문서: [`/SRS_v1.md#REQ-FUNC-002`](../05_SRS_v1.md) — 범위 외 입력 거부
- SRS 문서: [`/SRS_v1.md#REQ-FUNC-019`](../05_SRS_v1.md) — 매매횟수 상한 설정
- SRS ERD: [`/SRS_v1.md#6.2.1-ERD`](../05_SRS_v1.md) — ALERT_RULE 엔티티
- 태스크 분해: [`/TASKS/06_MVP_Task_Breakdown.md#API-004`](./06_MVP_Task_Breakdown.md)

## :white_check_mark: Task Breakdown (실행 계획)

### Phase A: Alert Rule Request DTO
- [ ] `src/types/alerts.dto.ts` 파일 생성
- [ ] 알람 규칙 생성/수정 Request DTO:
  ```typescript
  /** 알람 규칙 생성 요청 */
  export interface CreateAlertRuleRequest {
    /** 일일 최대 손실폭 임계치 (%) — 0.5 ~ 30.0 */
    threshold: number;
    /** 쿨타임 (분) — 기본 30, 범위 1 ~ 1440 */
    cooldown_minutes?: number;
    /** 일일 매매횟수 상한 (선택, NULL이면 미설정) */
    max_trades_per_day?: number | null;
  }

  /** 알람 규칙 수정 요청 (Partial) */
  export interface UpdateAlertRuleRequest {
    threshold?: number;
    cooldown_minutes?: number;
    max_trades_per_day?: number | null;
    active?: boolean;
  }
  ```

### Phase B: Alert Rule Response DTO
- [ ] 응답 DTO 정의:
  ```typescript
  /** 알람 규칙 단일 항목 */
  export interface AlertRuleItem {
    id: string;
    user_id: string;
    threshold: number;
    cooldown_minutes: number;
    max_trades_per_day: number | null;
    active: boolean;
    created_at: string;
    updated_at: string;
  }

  /** 알람 규칙 목록 조회 성공 응답 */
  export interface AlertRuleListResponse {
    success: true;
    data: { rules: AlertRuleItem[] };
  }

  /** 알람 규칙 생성/수정 성공 응답 */
  export interface AlertRuleMutationResponse {
    success: true;
    data: { rule: AlertRuleItem };
    message: string;
  }

  /** 에러 응답 */
  export interface AlertRuleErrorResponse {
    success: false;
    error: { code: AlertRuleErrorCode; message: string };
  }

  export type AlertRuleErrorCode =
    | 'THRESHOLD_OUT_OF_RANGE'   // 0.5~30 범위 외
    | 'INVALID_COOLDOWN'         // 쿨타임 범위 외
    | 'INVALID_MAX_TRADES'       // 매매횟수 상한 오류
    | 'RULE_NOT_FOUND'           // 규칙 미존재
    | 'UNAUTHORIZED'             // 인증 실패
    | 'SERVER_ERROR';            // 서버 오류
  ```

### Phase C: 상수 및 Zod 검증 스키마
- [ ] `src/types/alerts.schema.ts` 파일 생성:
  ```typescript
  import { z } from 'zod';

  export const ALERT_RULE_LIMITS = {
    THRESHOLD_MIN: 0.5,
    THRESHOLD_MAX: 30.0,
    THRESHOLD_STEP: 0.5,
    COOLDOWN_MIN: 1,
    COOLDOWN_MAX: 1440,
    COOLDOWN_DEFAULT: 30,
  } as const;

  export const createAlertRuleSchema = z.object({
    threshold: z.number()
      .min(ALERT_RULE_LIMITS.THRESHOLD_MIN, `손실폭은 최소 ${ALERT_RULE_LIMITS.THRESHOLD_MIN}%입니다.`)
      .max(ALERT_RULE_LIMITS.THRESHOLD_MAX, `손실폭은 최대 ${ALERT_RULE_LIMITS.THRESHOLD_MAX}%입니다.`),
    cooldown_minutes: z.number().int()
      .min(ALERT_RULE_LIMITS.COOLDOWN_MIN).max(ALERT_RULE_LIMITS.COOLDOWN_MAX)
      .default(ALERT_RULE_LIMITS.COOLDOWN_DEFAULT),
    max_trades_per_day: z.number().int().positive().nullable().optional(),
  });

  export const updateAlertRuleSchema = createAlertRuleSchema.partial().extend({
    active: z.boolean().optional(),
  });
  ```

### Phase D: 에러 메시지 매핑
- [ ] 한국어 에러 메시지 매핑:
  ```typescript
  export const ALERT_RULE_ERROR_MESSAGES: Record<AlertRuleErrorCode, string> = {
    THRESHOLD_OUT_OF_RANGE: '손실폭은 0.5%에서 30% 사이로 설정해주세요.',
    INVALID_COOLDOWN: '쿨타임은 1분에서 1440분 사이로 설정해주세요.',
    INVALID_MAX_TRADES: '매매횟수 상한은 1 이상의 정수여야 합니다.',
    RULE_NOT_FOUND: '알람 규칙을 찾을 수 없습니다.',
    UNAUTHORIZED: '인증이 필요합니다.',
    SERVER_ERROR: '서버 오류가 발생했습니다.',
  };
  ```

## :test_tube: Acceptance Criteria (BDD/GWT)

**Scenario 1: DTO 타입 Import 정상 동작**
- **Given:** `src/types/alerts.dto.ts`가 작성됨
- **When:** `import { CreateAlertRuleRequest, AlertRuleItem }` 수행
- **Then:** TypeScript 컴파일 에러 없이 참조.

**Scenario 2: createAlertRuleSchema — 유효한 입력**
- **Given:** threshold: 5.0, cooldown_minutes: 30
- **When:** `createAlertRuleSchema.safeParse()` 실행
- **Then:** `success: true`.

**Scenario 3: createAlertRuleSchema — threshold 하한 미만 거부**
- **Given:** threshold: 0.3
- **When:** `createAlertRuleSchema.safeParse()` 실행
- **Then:** `success: false`, '손실폭은 최소 0.5%입니다.' 메시지.

**Scenario 4: createAlertRuleSchema — threshold 상한 초과 거부**
- **Given:** threshold: 35.0
- **When:** `createAlertRuleSchema.safeParse()` 실행
- **Then:** `success: false`, '손실폭은 최대 30%입니다.' 메시지.

**Scenario 5: cooldown 기본값 적용**
- **Given:** cooldown_minutes 미설정
- **When:** `createAlertRuleSchema.safeParse({ threshold: 5 })` 실행
- **Then:** cooldown_minutes가 30으로 기본값 적용.

**Scenario 6: updateAlertRuleSchema — 부분 업데이트**
- **Given:** active: false만 주어짐
- **When:** `updateAlertRuleSchema.safeParse({ active: false })` 실행
- **Then:** `success: true` (partial 허용).

## :gear: Technical & Non-Functional Constraints
- **타입 안전:** TypeScript strict 모드 에러 없음
- **비즈니스 규칙:** threshold ∈ [0.5, 30.0] 엄격 검증 — `REQ-FUNC-001~002`
- **슬라이더 UX:** THRESHOLD_STEP: 0.5 단위로 슬라이더 이동
- **국제화:** 에러 메시지 한국어

## :checkered_flag: Definition of Done (DoD)
- [ ] `src/types/alerts.dto.ts` 생성 및 모든 DTO export
- [ ] `src/types/alerts.schema.ts` Zod 스키마 동작
- [ ] ALERT_RULE_LIMITS 상수 정의
- [ ] `npm run build` 타입 에러 없음

## :construction: Dependencies & Blockers
- **Depends on:**
  - `INFRA-001` (프로젝트 초기화)
- **Blocks:**
  - `MOCK-005` (대시보드 통계 Mock — 알람 관련 데이터 참조)
  - `CMD-ALERT-001` (알람 규칙 생성 Server Action)
  - `QRY-ALERT-001` (알람 규칙 목록 조회)
  - `UI-ALERT-001` (알람 설정 화면 UI)
