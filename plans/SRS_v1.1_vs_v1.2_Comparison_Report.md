# SRS v1.1 vs v1.2 변경 사항 비교 리포트

- **비교 대상:** `SRS_v0_1_Opus.md` (v1.1) ↔ `SRS_v1_2.md` (v1.2)
- **작성일:** 2026-05-02
- **목적:** 3대 피벗 전략 적용 전후의 변경점을 기술 스택, MVP 목표, 기타 차이점 관점으로 정리

---

## 1. 기술 스택의 명확성

> v1.1은 상용급 인프라(증권사 REST API 연동, Vercel Cron, 자체 매칭 로직 등)를 전제했으나, v1.2에서는 **바이브 코딩(AI 코드 생성 100% 의존)에 최적화된 기술 스택**으로 전면 재조정되었습니다.

| 항목 | v1.1 (SRS_v0_1_Opus.md) | v1.2 (SRS_v1_2.md) | 변경 근거 |
|---|---|---|---|
| **데이터 수집 방식** | 멀티 증권사 REST API 연동 (한국투자·토스·LS), OAuth/인증 키 기반 자동 동기화 | **CSV 수동 업로드** (한국투자/토스 표준 템플릿), Server Action 파싱 | 증권사 API 연동 복잡도를 제거하여 2주 내 MVP 배포 가능하도록 난이도 하향 |
| **데이터 동기화 메커니즘** | Vercel Cron Job (1~5분 주기) + 클라이언트 폴링 이원화 | **사용자 액션(CSV 업로드) 트리거** 단일 이벤트 기반 | 크론잡 의존성 제거, 서버리스 환경 비용 최적화 |
| **복기 일지 생성 로직** | 자체 규칙 기반 매칭 로직 (LLM은 인프라 구성만, Phase 2 적용) | **Gemini API 프롬프트 기반 추론** (Vercel AI SDK 직접 연동, MVP 적용) | 자체 로직 설계/튜닝 비용 제거, LLM을 MVP 런타임에 즉시 활용 |
| **인프라 플랜** | Vercel Hobby (무료), Supabase Free | **Vercel Pro ($20) + Supabase Pro ($25)**, 월 $45 예산 산정 | Free 플랜의 크론 제한·함수 실행 시간 제약 해소 |
| **Component Diagram 외부 시스템** | 증권사 REST API (3사), 매크로 뉴스 API, Stripe, Vercel Cron | **Stripe만 존재** (증권사 API·뉴스 API·Cron 노드 삭제) | 외부 의존성 대폭 축소 → 아키텍처 단순화 |
| **Server Actions 모듈** | `actions/accounts.ts` (증권사 연동), `Supabase Realtime`, `/api/cron/sync` (주기 동기화) | **`actions/upload.ts`** (CSV 업로드/파싱/적재), Cron 관련 모듈 전체 삭제 | 증권사 연동 흐름 → CSV 업로드 단일 파이프라인으로 교체 |
| **Gemini API 역할** | AI Layer에 존재하나 설명 없이 연결만 (C-TEC-005: MVP 런타임 제외) | **"복기 일지 요약 및 타점 분석 로직 수행"으로 역할 명시**, MVP에서 적극 활용 | LLM 활용 범위를 인프라 테스트→실전 추론으로 승격 |
| **인증 프로토콜** | OAuth 2.0 PKCE (증권사 연동 포함), AES-256 토큰 암호화 | **Supabase Auth** 기반 단순 인증 (OAuth/PKCE/AES-256 토큰 삭제) | CSV 업로드로 전환되어 증권사 인증 불필요 |
| **뉴스 데이터 연동** | FRED/Statista 뉴스 API → MACRO_EVENT 테이블 매칭 | **삭제** (MACRO_EVENT 엔터티 및 뉴스 API 연동 전체 제거) | Gemini 프롬프트에 뉴스 매칭 로직 위임, 별도 API 의존 제거 |
| **Storage 용도** | Supabase Storage (PDF/파일) | Supabase Storage (**CSV/PDF 파일**) | CSV 원본 보관 용도 추가 |

---

## 2. MVP 목표 및 가치전달 조정 내용

> v1.1은 **SOM 5,000명 대상 상용 서비스 수준(SLA 99.5%)**을 목표로 했으나, v1.2는 **초기 테스터 50명 대상의 현실적 MVP**로 목표를 재설정했습니다. 핵심 가치(감정 제어, 자동 복기)는 유지하면서 전달 방식만 변경합니다.

### 2.1 기능 요구사항 (Functional Requirements) 변경

| 영역 | 항목 | v1.1 | v1.2 | 가치 영향 |
|---|---|---|---|---|
| **F1 알람 트리거** | REQ-FUNC-003 | 클라이언트 폴링 + 증권사 API 실시간 모니터링 | **CSV 업로드 시점 사후 트리거** | 실시간 → 사후 확인으로 변경되나, 매매 호흡 고려 시 핵심 가치 유지 |
| **F1 쿨타임** | REQ-FUNC-004 | 매수 **버튼 비활성화** (기능적 차단, 우회율 < 1%) | 대시보드에 **시각적 경고** 고정 표시 (추가 매수 억제/유도) | 강제 차단 → 경고 유도로 약화. 실매매가 앱 외부에서 발생하므로 현실 반영 |
| **F1 리포트** | REQ-FUNC-005 | Vercel 일일 Cron 트리거, 정확도 >= 99% | 누적 데이터 기반 집계 스크립트 (Cron 의존 삭제) | 리포트 생성은 유지, 트리거 방식만 변경 |
| **F1 장애대응** | REQ-FUNC-006 | 증권사 API 장애 시 폴백 알람 + PagerDuty 알림 | **삭제** (증권사 API 비사용) | CSV 방식에서는 API 장애 시나리오 자체가 불필요 |
| **F2 데이터 수집** | REQ-FUNC-007~012 | 증권사 OAuth 연동, 5분 동기화, 복수 계좌 합산, AES-256 토큰 암호화 (6건) | **CSV 업로드·헤더 검증·Hash 중복 방지** (3건) | 6건 → 3건으로 요구사항 축소. 핵심 가치(데이터 적재)는 동일 |
| **F3 복기 로직** | REQ-FUNC-013 | 자체 규칙 기반 매칭 (LLM 배제), 일일 Cron 트리거 | **Gemini API 프롬프트 추론**, CSV 업로드 직후 트리거 | 가치 **강화** — LLM 인사이트 품질 향상, 실시간성 확보 |
| **F3 기능 축소** | REQ-FUNC-016, 017 | 매매 0건 안내, 뉴스 API 실패 처리 | **삭제** (뉴스 API 미사용, 0건 로직은 시퀀스에만 존재) | 엣지 케이스 요구사항 경량화 |
| **F5 대시보드** | REQ-FUNC-018 | 로딩 p95 <= 2초, 동시 500명 기준 | 차트 정상 표시 (성능 수치 제거) | MVP 단계에서 엄격한 성능 기준 완화 |
| **F5 기능 축소** | REQ-FUNC-020~022 | 상한 도달 시 쿨다운, 월간 리포트 자동 생성, 신규 유저 안내 (3건) | **삭제** | 대시보드 핵심(통계 시각화 + 횟수 설정)만 남기고 부가 기능 이관 |

### 2.2 비기능 요구사항 (Non-Functional Requirements) 변경

| NFR 영역 | 항목 | v1.1 | v1.2 | 판단 |
|---|---|---|---|---|
| **성능** | REQ-NF-001 | API p95 ≤ 500ms (동시 500명, k6 ramp-up) | API p95 ≤ 500ms (측정 조건 삭제) | 목표값 유지, 측정 조건 완화 |
| **성능** | REQ-NF-003~005, 007~008 | 대시보드 로딩 2초, 증권사 동기화 5분, 계좌 연동 60초, PDF 10초, 폴백 10초 (5건) | **전체 삭제** | 증권사 연동·크론 관련 성능 지표 불필요 |
| **성능** | REQ-NF-004 | 증권사 동기화 지연 ≤ 5분 | **CSV 파싱·적재 ≤ 5초** (신규) | 성능 측정 대상 교체 (동기화 → 파싱) |
| **성능** | REQ-NF-006 | 복기 일지 생성 ≤ 30분 (Cron 후 매칭) | 복기 일지 생성 ≤ **30초** (Gemini API 응답) | 30분 → 30초로 **600배 개선** (LLM 즉시 추론) |
| **확장성** | REQ-NF-009 | 동시 2,000명, p95 ≤ 1초 | **초기 테스터 50명, 동시접속 10명** | 200배 축소. MVP 현실 반영 |
| **확장성** | REQ-NF-010 | 동시 2,000 클라이언트 폴링 지원 | **삭제** (폴링 미사용) | — |
| **가용성** | REQ-NF-011 | 월 가용성 ≥ **99.5%** (다운타임 ≤ 3.6시간/월) | **Best Effort** (Vercel/Supabase SLA 상속) | 엔터프라이즈 SLA → 플랫폼 SLA 위임 |
| **가용성** | REQ-NF-012 | 동기화 오류율 ≤ 0.1% | 데이터 파싱 유실률 **0%** | 파싱 정합성으로 재정의, 목표 강화 |
| **가용성** | REQ-NF-013~016b | 알람 실패율, 리포트 정확도, 매칭 정확도, 대시보드 정합성, RTO/RPO (6건) | **삭제** (REQ-NF-015만 Gemini JSON 실패율 < 2%로 대체) | 증권사·뉴스 관련 신뢰성 지표 전면 제거 |
| **보안** | REQ-NF-017, 019 | AES-256 토큰 암호화, OAuth 2.0 PKCE | **삭제** (증권사 토큰 비사용) | — |
| **보안** | REQ-NF-021~023 | 침투 테스트(분기), ISMS 갱신(연간), RBAC | **삭제** | MVP 단계에서 과도한 보안 인증 비용 배제 |
| **보안** | REQ-NF-023a | 개인정보보호법(PIPA) 및 신용정보법 준수 | **유지** | 법적 필수 사항 — 양 버전 모두 존재 |
| **비용** | REQ-NF-024 | Vercel Hobby **(무료)** 플랜 한도 내 구현 | **월 $45** (Vercel Pro $20 + Supabase Pro $25) | 무료 → 유료 전환. 비용 발생하나 기능 제약 해소 |
| **비용** | REQ-NF-025 | 인프라 마진 ≥ 90% | **삭제** | 초기 MVP에서 마진 목표 불필요 |
| **모니터링** | REQ-NF-026~030 (5건) | API latency, 5xx 에러율, 동기화 지연, 알람 실패율, 사용자 비용 모니터링 | **전체 삭제** | MVP 단계에서 PagerDuty/Slack 연동 과도 |
| **유지보수** | REQ-NF-031~032 | 모듈화, 증권사 어댑터 패턴 확장 | **삭제** | 증권사 어댑터 패턴 불필요 (CSV 단일 채널) |
| **비즈니스 KPI** | REQ-NF-033~037 (5건) | WAU 뇌동매매 0회 비율, D7 리텐션, 무료→유료 전환, 복기 시간, NPS | **전체 삭제** | KPI 측정은 Mixpanel/Delighted 등 외부 도구 영역으로 분리 |

---

## 3. 기타 차이점

> 문서 구조, 서술 스타일, 데이터 모델, 다이어그램, 추적성 등 1·2에 해당하지 않는 변경 사항입니다.

### 3.1 문서 구조 및 서술 변경

| 항목 | v1.1 | v1.2 | 비고 |
|---|---|---|---|
| **문서 분량** | 961줄 (45KB) | 529줄 (26KB) | **45% 축소** — MVP 다운사이징 |
| **서술체** | 평어체 (~한다, ~된다) | **경어체** (~합니다, ~됩니다) | 이해관계자 대상 문서 톤 통일 |
| **FR 테이블 컬럼** | ID, 요구사항, Source, Priority, AC (5열) | ID, 요구사항, Priority, AC (**4열**, Source 제거) | 문서 간결화 — Story 추적은 Traceability에 위임 |
| **NFR 표현 방식** | 테이블 형식 (ID, 요구사항, 측정 조건, Source) | **불릿 포인트 서술형** | 가독성 중심 재구성 |
| **Traceability Matrix** | Story 단위 22행 (세부 AC별 매핑) | Feature 단위 **8행** (범위 기반 그룹 매핑) | 추적 단위를 Story→Feature로 추상화 |

### 3.2 데이터 모델 변경

| 엔터티 | v1.1 | v1.2 | 비고 |
|---|---|---|---|
| **ACCOUNT** | 7개 필드 (broker_name, encrypted_token, last_synced_at 포함) | **4개 필드** (id, user_id, status, created_at만 유지) | 증권사 연동 필드 전면 삭제 |
| **TRADE** | 9개 필드 (is_planned, synced_at 포함) | **8개 필드** (`source_file_hash` 추가, is_planned·synced_at 삭제) | CSV 중복 방지 해시 추가, 동기화·계획 매매 태깅 삭제 |
| **TRADE_REVIEW** | 7개 필드 (review_date, trade_count, news_matched, pdf_url 포함) | **4개 필드** (id, user_id, summary_json, created_at) | Gemini JSON 응답 중심으로 단순화 |
| **MACRO_EVENT** | 존재 (5개 필드, TRADE와 M:N 관계) | **삭제** | 뉴스 매칭을 Gemini 프롬프트에 위임 |
| **ERD 관계** | TRADE ↔ MACRO_EVENT M:N 매칭 | **삭제** | — |

### 3.3 시퀀스 다이어그램 변경

| 다이어그램 | v1.1 | v1.2 | 비고 |
|---|---|---|---|
| **3.4.1** | 뇌동매매 방지 알람 흐름 (증권사 API 폴링 기반) | **CSV 파일 파싱 및 DB 적재 흐름** (프론트 검증→서버 파싱→적재) | 완전 재작성 |
| **3.4.2** | 자동 매매 복기 일지 생성 (Vercel Cron + 규칙 기반 매칭) | **데이터 업로드 즉시 누적 손실 계산 및 알람 발동** | 완전 재작성 — 알람 흐름을 업로드 트리거로 재구성 |
| **3.4.3** | 멀티 증권사 통합 연동 흐름 (OAuth + 동기화) | **프롬프트 기반 자동 매매 복기 생성 흐름** (Gemini 연동) | 완전 재작성 — 증권사 → Gemini 복기 흐름으로 교체 |
| **6.3.1** | 통계 대시보드 + 매매횟수 브레이크 상세 흐름 | **삭제** | — |
| **6.3.2** | 온보딩 → 증권사 연동 → 첫 복기 일지 E2E 흐름 | **삭제** (6.3 Detailed Interaction Models를 텍스트 서술로 대체) | 3개 상세 시퀀스 → 3줄 워크플로우 요약 |
| **6.3.3** | 뇌동매매 방지 → 주간 리포트 전체 사이클 | **삭제** | — |

### 3.4 API Endpoint 변경

| 항목 | v1.1 | v1.2 | 비고 |
|---|---|---|---|
| **총 엔드포인트 수** | 22개 | **9개** | **59% 축소** |
| **삭제된 API** | auth/refresh, accounts (CRUD 4건), accounts/{id}/sync, trades/{id}, alerts (PUT/DELETE/logs/email 4건), reviews/{id}·tag, reports (weekly/monthly/pdf 3건) — 13건 | — | 증권사 연동·세부 CRUD·리포트 관련 API 전면 삭제 |
| **신규 API** | — | **POST `/api/v1/trades/upload`** (CSV 멀티파트 업로드), **POST `/api/v1/reviews/generate`** (Gemini 수동 트리거) | CSV 업로드 + AI 분석 전용 엔드포인트 신설 |
| **Rate Limit** | 모든 엔드포인트에 Rate Limit 명시 | **Rate Limit 컬럼 삭제** | MVP 단계 간소화 |

### 3.5 Class Diagram 변경

| 항목 | v1.1 | v1.2 | 비고 |
|---|---|---|---|
| **클래스 수** | 10개 (User, Account, Trade, TradeReview, AlertRule, AlertLog, MacroEvent, AlertCronHandler, SyncCronHandler, ReviewServerAction) | **5개** (User, Trade, TradeReview, CsvUploadAction, GeminiReviewService) | **50% 축소** |
| **삭제된 클래스** | Account (syncTrades, validateToken), MacroEvent (matchTrades), AlertCronHandler, SyncCronHandler | — | 증권사 동기화·크론·뉴스 매칭 클래스 전면 제거 |
| **신규 클래스** | — | **CsvUploadAction** (validateHeaders, parseCsvData, checkDuplicateHash, persistTrades), **GeminiReviewService** (generateInsight, saveReview) | CSV 파싱 + Gemini 추론 전용 서비스 |

### 3.6 삭제된 섹션

| 섹션 | v1.1 | v1.2 | 비고 |
|---|---|---|---|
| §4.2.6 운영/모니터링 (Observability) | REQ-NF-026~030, PagerDuty/Slack 연동 | **전체 삭제** | MVP에서 상용 모니터링 스택 비용 과도 |
| §4.2.7 유지보수성 (Maintainability) | REQ-NF-031~032 | **전체 삭제** | 어댑터 패턴 등 불필요 |
| §4.2.8 비즈니스 KPI | REQ-NF-033~037 (북극성, 리텐션, 전환율, NPS 등) | **전체 삭제** | KPI 측정은 별도 문서로 분리 가능 |
| §6.4 Validation Plan | A/B 테스트, 코호트 분석, 가격 조사 | **전체 삭제** | MVP 이후 실험 설계로 이관 |

### 3.7 Constraints / Assumptions 변경

| 항목 | v1.1 | v1.2 | 비고 |
|---|---|---|---|
| ASMP-01 Kill Criteria | Closed Beta **n=300** | Closed Beta **n=50** | MVP 대상 규모 축소 반영 |
| ASMP-02 | 3사가 타겟 80% 커버 여부 | CSV 포맷 추출 어려움 없음 | 증권사 커버리지 → CSV UX 가정으로 교체 |
| DEP-01 (증권사 API 의존성) | 존재 (Kill: API 가용률 < 95%) | **삭제** | CSV로 전환되어 외부 의존성 없음 |
| DEP-02 (뉴스 API 의존성) | 존재 (Kill: 데이터 지연 > 1시간) | **삭제** | Gemini에 위임 |
| C-TEC-005 (LLM 제외) | LLM 기능 MVP 런타임 제외 | **삭제** (Gemini를 MVP에서 적극 사용) | 전략적 전환 |
| C-TEC-006 | Vercel Hobby (무료), 일일 Cron 최대 1회 | Vercel Pro + Supabase Pro ($45), CSV 트리거 기반 | 유료 전환 + 이벤트 기반 아키텍처 |
| RISK-01 (증권사 API 변경) | 존재 (확률: 중, 영향: 치명) | **삭제** | CSV로 전환되어 리스크 해소 |
| RISK-03 (계좌 정보 유출) | 존재 (확률: 하, 영향: 치명) | **삭제** | 증권사 계좌/토큰 미보관으로 리스크 해소 |

---

## 총평

| 관점 | 요약 |
|---|---|
| **기술 스택** | 증권사 REST API·OAuth·Cron 중심 → **CSV 업로드 + Gemini 프롬프트 추론** 중심으로 전환. 외부 시스템 의존성 7개 → 4개로 축소. |
| **MVP 목표** | 상용급(SOM 5,000, SLA 99.5%, 동시 2,000) → **초기 테스터 50명, Best Effort, 월 $45** 현실 목표로 재설정. 핵심 가치(감정 제어·자동 복기)는 유지하되, **Gemini 프롬프트 기반 복기**로 가치 전달 방식 강화. |
| **기타** | 문서 961줄 → 529줄 (45% 축소), API 22건 → 9건, 클래스 10개 → 5개, 데이터 엔터티 7개 → 6개로 전반적 다운사이징. 모니터링·KPI·실험 검증 등은 MVP 이후 단계로 이관. |
