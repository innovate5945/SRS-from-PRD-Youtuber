---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] API-002: Trades Upload DTO 정의 (CSV 업로드 Request/Response)"
labels: 'feature, backend, api-contract, priority:high, sprint:0'
assignees: ''
---

## :dart: Summary
- **기능명:** [API-002] Trades Upload DTO 정의 (multipart/form-data Request, 성공/에러 Response)
- **목적:** CSV 파일 업로드 기능(F2)의 프론트엔드/백엔드 간 데이터 통신 계약(Contract)을 TypeScript DTO(Data Transfer Object)로 선제 정의한다. 이 계약이 확립되면 프론트엔드(UI-CSV-001)와 백엔드(CMD-CSV-001~005)가 독립적으로 병렬 개발할 수 있다.

## :link: References (Spec & Context)
> :bulb: AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`/SRS_v1.md#6.1-API-Endpoint-#3`](../SRS_v1.md) — `POST /api/v1/trades/upload` (multipart/form-data, Bearer 인증)
- SRS 문서: [`/SRS_v1.md#3.3-actions/upload.ts`](../SRS_v1.md) — Server Action 모듈 정의
- SRS 문서: [`/SRS_v1.md#3.4.1-CSV-파싱-시퀀스`](../SRS_v1.md) — CSV 업로드 전체 흐름
- SRS 문서: [`/SRS_v1.md#REQ-FUNC-007~009`](../SRS_v1.md) — CSV 업로드 기능 요구사항
- SRS ERD: [`/SRS_v1.md#6.2-TRADE`](../SRS_v1.md) — TRADE 엔티티 스키마 (ticker, side, price, quantity, source_file_hash 등)
- SRS 문서: [`/SRS_v1.md#REQ-NF-004`](../SRS_v1.md) — CSV 파싱 및 적재 ≤ 5초
- 태스크 분해: [`/TASKS/MVP_Task_Breakdown.md#API-002`](./MVP_Task_Breakdown.md)

## :white_check_mark: Task Breakdown (실행 계획)

### Phase A: Request DTO 정의 (CSV Upload Request)
- [ ] `src/types/trades.dto.ts` 파일 생성
- [ ] 클라이언트 → 서버 전송 규격 정의:
  ```typescript
  // === Request ===
  
  /**
   * CSV 업로드 요청 - multipart/form-data
   * Content-Type: multipart/form-data
   * 인증: Bearer Token (Supabase Auth Session)
   */
  export interface TradeUploadRequest {
    /** CSV 파일 (한국투자 또는 토스 표준 형식) */
    file: File;
    /** 증권사 템플릿 타입 */
    broker_type: BrokerType;
  }
  
  /** 지원 증권사 타입 */
  export type BrokerType = 'kis' | 'toss';
  
  /** 클라이언트 단 1차 검증 규칙 */
  export const UPLOAD_VALIDATION = {
    /** 최대 파일 크기: 5MB */
    MAX_FILE_SIZE: 5 * 1024 * 1024,
    /** 허용 확장자 */
    ALLOWED_EXTENSIONS: ['.csv'] as const,
    /** 허용 MIME 타입 */
    ALLOWED_MIME_TYPES: ['text/csv', 'application/vnd.ms-excel'] as const,
    /** 최대 행 수 (헤더 제외) */
    MAX_ROW_COUNT: 10_000,
  } as const;
  ```

### Phase B: Response DTO 정의 (성공/에러)
- [ ] 성공 응답 DTO:
  ```typescript
  // === Success Response ===
  
  export interface TradeUploadSuccessResponse {
    success: true;
    data: {
      /** 적재된 매매 건수 */
      inserted_count: number;
      /** 중복 스킵된 건수 (동일 해시 내 중복 행) */
      skipped_count: number;
      /** 업로드 파일 해시 */
      source_file_hash: string;
      /** 처리 소요 시간 (ms) */
      processing_time_ms: number;
    };
    message: string;
  }
  ```
- [ ] 에러 응답 DTO:
  ```typescript
  // === Error Response ===
  
  export interface TradeUploadErrorResponse {
    success: false;
    error: {
      /** 에러 코드 */
      code: TradeUploadErrorCode;
      /** 사용자 노출 에러 메시지 (국문) */
      message: string;
      /** 상세 에러 정보 (dev 환경 전용) */
      details?: ValidationErrorDetail[];
    };
  }
  
  export type TradeUploadErrorCode =
    | 'INVALID_FILE_FORMAT'      // 파일 확장자/MIME 오류
    | 'FILE_TOO_LARGE'           // 파일 크기 초과
    | 'INVALID_CSV_HEADER'       // CSV 헤더 형식 불일치
    | 'INVALID_DATA_TYPE'        // 데이터 타입 정합성 오류
    | 'DUPLICATE_FILE'           // source_file_hash 중복
    | 'ROW_LIMIT_EXCEEDED'       // 행 수 초과
    | 'PARSE_ERROR'              // CSV 파싱 실패
    | 'DB_INSERT_ERROR'          // DB 적재 실패
    | 'UNAUTHORIZED';            // 인증 실패
  
  export interface ValidationErrorDetail {
    /** 오류 발생 행 번호 (1-indexed, 헤더 제외) */
    row?: number;
    /** 오류 발생 컬럼명 */
    column?: string;
    /** 오류 설명 */
    message: string;
  }
  ```

### Phase C: 정규화된 매매 내역 DTO (파싱 결과 표준 구조)
- [ ] 증권사별 CSV 파싱 후 정규화된 공통 구조 정의:
  ```typescript
  // === Normalized Trade DTO ===
  
  /** DB 적재 전 정규화된 매매 내역 (증권사 무관 통합 구조) */
  export interface NormalizedTrade {
    /** 종목 코드 */
    ticker: string;
    /** 종목명 (표시용) */
    ticker_name?: string;
    /** 매수/매도 */
    side: 'buy' | 'sell';
    /** 체결 가격 */
    price: number;
    /** 체결 수량 */
    quantity: number;
    /** 체결 일시 (ISO 8601) */
    executed_at: string;
  }
  
  /** 파싱 결과 전체 (DB 적재 전 중간 객체) */
  export interface ParsedCsvResult {
    /** 정규화된 매매 내역 배열 */
    trades: NormalizedTrade[];
    /** 원본 파일 SHA-256 해시 */
    source_file_hash: string;
    /** 파싱 경고 (무시 가능한 행 등) */
    warnings: string[];
  }
  ```

### Phase D: 증권사별 CSV 헤더 매핑 상수 정의
- [ ] `src/constants/csv-templates.ts` 파일 생성:
  ```typescript
  // === CSV 템플릿 헤더 매핑 ===
  
  /** 한국투자증권 CSV 필수 헤더 */
  export const KIS_REQUIRED_HEADERS = [
    '주문일', '종목코드', '종목명', '매매구분', '체결가', '체결수량'
  ] as const;
  
  /** 토스증권 CSV 필수 헤더 */
  export const TOSS_REQUIRED_HEADERS = [
    '거래일시', '종목코드', '종목명', '거래유형', '거래가격', '거래수량'
  ] as const;
  
  /** 헤더 → NormalizedTrade 필드 매핑 */
  export const HEADER_FIELD_MAP: Record<BrokerType, Record<string, keyof NormalizedTrade>> = {
    kis: {
      '주문일': 'executed_at',
      '종목코드': 'ticker',
      '종목명': 'ticker_name',
      '매매구분': 'side',
      '체결가': 'price',
      '체결수량': 'quantity',
    },
    toss: {
      '거래일시': 'executed_at',
      '종목코드': 'ticker',
      '종목명': 'ticker_name',
      '거래유형': 'side',
      '거래가격': 'price',
      '거래수량': 'quantity',
    },
  };
  ```

### Phase E: 통합 타입 및 유효성 검증 스키마
- [ ] Zod 기반 런타임 검증 스키마 정의 (`src/types/trades.schema.ts`):
  ```typescript
  import { z } from 'zod';
  
  export const tradeUploadSchema = z.object({
    file: z.instanceof(File)
      .refine(f => f.size <= UPLOAD_VALIDATION.MAX_FILE_SIZE, '파일 크기가 5MB를 초과합니다.')
      .refine(f => f.name.endsWith('.csv'), 'CSV 파일만 업로드 가능합니다.'),
    broker_type: z.enum(['kis', 'toss']),
  });
  
  export const normalizedTradeSchema = z.object({
    ticker: z.string().min(1).max(20),
    ticker_name: z.string().optional(),
    side: z.enum(['buy', 'sell']),
    price: z.number().positive(),
    quantity: z.number().int().positive(),
    executed_at: z.string().datetime(),
  });
  ```
- [ ] Zod 패키지 설치: `npm install zod`

## :test_tube: Acceptance Criteria (BDD/GWT)

**Scenario 1: DTO 타입 Import 정상 동작**
- **Given:** `src/types/trades.dto.ts`가 작성됨
- **When:** 다른 파일에서 `import { TradeUploadRequest, TradeUploadSuccessResponse, TradeUploadErrorCode } from '@/types/trades.dto'`를 수행함
- **Then:** TypeScript 컴파일 에러 없이 타입이 정상 참조되어야 한다.

**Scenario 2: Zod 스키마 검증 — 유효한 파일**
- **Given:** 5MB 이하의 `.csv` 확장자 파일과 `broker_type: 'kis'`가 주어짐
- **When:** `tradeUploadSchema.safeParse()`로 검증함
- **Then:** `success: true`를 반환해야 한다.

**Scenario 3: Zod 스키마 검증 — 파일 크기 초과**
- **Given:** 6MB 크기의 CSV 파일이 주어짐
- **When:** `tradeUploadSchema.safeParse()`로 검증함
- **Then:** `success: false`와 함께 "파일 크기가 5MB를 초과합니다." 에러 메시지를 반환해야 한다.

**Scenario 4: NormalizedTrade 정규화 스키마 검증**
- **Given:** 유효한 매매 데이터 (`ticker: '005930', side: 'buy', price: 72000, quantity: 10`)가 주어짐
- **When:** `normalizedTradeSchema.safeParse()`로 검증함
- **Then:** `success: true`를 반환해야 한다.

**Scenario 5: 에러 코드 열거형 완전성**
- **Given:** `TradeUploadErrorCode` 타입이 정의됨
- **When:** SRS §3.4.1의 모든 에러 분기(형식 오류, 중복, 크기 초과)를 확인함
- **Then:** 각 분기에 대응하는 에러 코드가 누락 없이 정의되어 있어야 한다.

**Scenario 6: 증권사 헤더 매핑 완전성**
- **Given:** `KIS_REQUIRED_HEADERS`와 `TOSS_REQUIRED_HEADERS`가 정의됨
- **When:** 각 헤더가 `HEADER_FIELD_MAP`에서 `NormalizedTrade` 필드로 매핑됨
- **Then:** 모든 필수 헤더가 누락 없이 NormalizedTrade의 필드에 1:1 매핑되어야 한다.

## :gear: Technical & Non-Functional Constraints
- **타입 안전:** 모든 DTO는 TypeScript strict 모드에서 에러 없어야 함
- **런타임 검증:** 서버 사이드에서 Zod 스키마 기반 검증 필수 (프론트엔드 검증만으로 불충분)
- **파일 크기:** 최대 5MB 제한 — 1,000건 기준 CSV 예상 크기 ~200KB, 10배 여유 확보
- **성능:** 전체 업로드→파싱→적재 파이프라인이 5초 이내 완료 — `REQ-NF-004`
- **국제화:** 에러 메시지는 한국어로 작성 (MVP 국내 타겟)
- **확장성:** `BrokerType`에 향후 LS증권 등 추가 시 최소 변경으로 대응 가능한 구조

## :checkered_flag: Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] `src/types/trades.dto.ts` 파일이 생성되고 모든 DTO 타입이 export 되었는가?
- [ ] `src/types/trades.schema.ts` 파일이 생성되고 Zod 스키마가 동작하는가?
- [ ] `src/constants/csv-templates.ts` 파일이 생성되고 헤더 매핑이 완료되었는가?
- [ ] `npm run build` 시 타입 에러가 없는가?
- [ ] Zod 패키지가 설치되고 `package.json`에 의존성이 등록되었는가?

## :construction: Dependencies & Blockers
- **Depends on:**
  - `INFRA-001` (프로젝트 초기화 — `src/types/` 디렉토리 필요)
  - 없음 (API DTO는 독립적으로 선행 정의 가능)
- **Blocks:**
  - `MOCK-002` (CSV 업로드 Mock Response — DTO 타입 참조)
  - `CMD-CSV-001` (클라이언트 단 파일 검증 — UPLOAD_VALIDATION 상수 참조)
  - `CMD-CSV-002` (서버 단 CSV 검증 — Zod 스키마 및 헤더 매핑 참조)
  - `CMD-CSV-003` (CSV 파싱 및 정규화 — NormalizedTrade DTO 참조)
  - `UI-CSV-001` (CSV 업로드 화면 — Request/Error DTO 참조)
