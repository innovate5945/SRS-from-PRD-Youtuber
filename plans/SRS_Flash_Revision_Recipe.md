맞습니다. 기획자 본인이 직접 문서를 수정하는 것보다, **명확한 지침(Prompt)을 구조화하여 AI에게 기존 SRS의 개정 작업을 위임하는 것**이 '바이브 코딩'의 본질이자 오케스트레이션 관점의 올바른 접근입니다.

AI(Cursor, Claude, GPT-4 등)가 기존 `SRS_v0_1_Opus.md`를 입력받아, 논리적 충돌 없이 완벽한 `SRS_v1_2.md`로 재작성하도록 지시하는 **AI 작업용 프롬프트 계획서**를 작성해 드립니다. 이 문서의 내용을 복사하여 AI에게 전달하시면 됩니다.

---

# AI 작업용 SRS 개정 지침서 (Prompt Plan for SRS Refactoring)

**[System Role]**
너는 15년 차 수석 소프트웨어 아키텍트이자 테크니컬 라이터다. 내가 제공하는 기존 `SRS v1.1` 문서를 기반으로, 아래의 **[3대 피벗(Pivot) 전략]**과 **[섹션별 세부 수정 지침]**을 완벽하게 반영하여 `SRS v1.2` 문서를 처음부터 끝까지 마크다운(.md) 포맷으로 재작성하라. 

**[Objective]**
'3개월 차 개발 지식을 보유한 기획자'가 '완전 바이브 코딩(AI 코드 생성 100% 의존)'으로 2주 내에 배포할 수 있는 현실적인 MVP 스펙으로 문서를 다운사이징하고, 비즈니스 및 인프라 제약 간의 기술적 모순을 해소한다.

---

## 1. 3대 피벗 전략 (Core Pivot Strategies)

기존 문서의 논리 구조를 유지하되, 다음 세 가지 핵심 변경 사항을 문서 전반에 일관되게 적용하라.

1. **데이터 수집 방식의 전환 (난이도 하향):**
   * **AS-IS:** 멀티 증권사(한국투자, 토스, LS) REST API / OAuth 연동 (F2).
   * **TO-BE:** **CSV 수동 업로드 (표준 템플릿 기반)**. 증권사에서 다운로드한 CSV 파일을 업로드하여 파싱하는 방식으로 전면 교체.
2. **인프라 제약 및 비즈니스 목표의 현실화 (모순 해소):**
   * **AS-IS:** Vercel Hobby(무료) + 상용 서비스 수준(SLA 99.5%, 동시접속 2,000명).
   * **TO-BE:** **유료 인프라 도입(월 $45 예산 산정)** 및 **초기 테스터(50명) 한정 배포**. Vercel Pro 및 Supabase Pro 플랜을 기본으로 산정하고, SLA는 'Best Effort'로 하향 조정.
3. **복기 일지 로직의 위임 (바이브 코딩 최적화):**
   * **AS-IS:** 자체 규칙 기반 타점-뉴스 매칭 (LLM은 인프라만 연결).
   * **TO-BE:** **Gemini API 활용 프롬프트 기반 자동 복기**. CSV에서 파싱된 JSON 데이터를 프롬프트와 함께 Gemini API로 전송하여 핵심 인사이트를 반환받는 구조로 변경.

---

## 2. 섹션별 세부 수정 지침 (Section-by-Section Directives)

### [1. Introduction]
* **1.2 Scope (In-Scope):** 'F2. 멀티 증권사 API 통합 연동'을 'F2. 표준 CSV 매매 내역 업로드 및 파싱'으로 변경할 것.
* **1.2 Constraints:** * `DEP-01` (증권사 API 의존성) 삭제.
  * `C-TEC-006` 수정: Vercel Pro 및 Supabase Pro 도입을 명시(월 예산 $45). 무거운 크론잡 대신 사용자 액션(CSV 업로드) 트리거 기반으로 로직 변경.
* **1.3 Definitions:** PKCE, OAuth 등 API 인증 관련 용어 삭제.

### [2. Stakeholders & Use Case Diagram]
* **Use Case Diagram (Mermaid):**
  * `UC-04: 증권사 계좌 연동`을 `UC-04: CSV 매매 내역 업로드`로 교체.
  * `UC-05: 매매 데이터 자동 동기화`를 `UC-05: 업로드 데이터 정합성 검증`으로 교체.

### [3. System Context and Interfaces]
* **3.0 Component Diagram (Mermaid):**
  * External Systems에서 '증권사 REST API'를 삭제할 것.
  * AI Layer의 'Gemini API' 역할을 '복기 일지 요약 및 타점 분석 로직 수행(MVP 적용)'으로 승격시켜 도식화할 것.
* **3.4 Interaction Sequences:**
  * `3.4.1` 장중 실시간 감시를 제거하고, "CSV 업로드 즉시 당일 누적 손실 계산 및 사후 알람 발동" 흐름으로 시퀀스 다이어그램 재작성.
  * `3.4.3 멀티 증권사 통합 연동 흐름`을 `CSV 파일 파싱 및 DB 적재 흐름`으로 완전히 재작성할 것. (파일 선택 -> 프론트엔드 검증 -> Server Action 전송 -> 정규화 및 DB 적재).

### [4. Specific Requirements]
* **4.1.1 (F1):** 실시간 클라이언트 폴링(REQ-FUNC-003)을 '데이터 업로드 시점 트리거'로 변경.
* **4.1.2 (F2):** 기존 증권사 API 요구사항(REQ-FUNC-007~012) 전면 삭제. 
  * 신규 요구사항 추가: "사용자는 템플릿(한국투자/토스 형식)에 맞는 CSV 파일을 업로드할 수 있어야 한다."
  * 신규 요구사항 추가: "시스템은 업로드된 CSV의 헤더와 데이터 타입을 검증하고 오류 시 인라인 메시지를 표시해야 한다."
* **4.1.3 (F3):** REQ-FUNC-013 로직을 'Gemini API 기반 프롬프트 추론'으로 수정.
* **4.2 Non-Functional:** * `REQ-NF-004, 005` (API 동기화 지연 등) 삭제. 대신 'CSV 파싱 및 적재 처리 시간 <= 5초' 추가.
  * 확장성(REQ-NF-009)을 '초기 테스터 50명, 동시접속 10명 이내 보장'으로 축소.
  * 가용성(REQ-NF-011) 목표를 99.5%에서 Best Effort로 낮추고, 비용(REQ-NF-024)을 '월 $45 이내의 Vercel/Supabase Pro 환경 구축'으로 수정.

### [5 & 6. Traceability & Appendix]
* **6.1 API Endpoint:** 증권사 연동/동기화 API 전체 삭제. `/api/v1/trades/upload` (POST, 멀티파트 폼 데이터) 추가.
* **6.2 ERD 및 Entity:**
  * `ACCOUNT` 테이블 구조 대폭 축소 (encrypted_token, broker_name 삭제).
  * `TRADE` 테이블에 원본 CSV 파일의 Hash 값을 저장하여 중복 업로드 방지 컬럼(`source_file_hash`) 추가.
* **6.3 Detailed Interaction Models:** 내용 전체를 CSV 중심 및 비실시간(Asynchronous) 이벤트 구조에 맞게 논리적 모순 없이 업데이트할 것.

**[Output Format]**
* 제목: `# Software Requirements Specification (SRS) - v1.2`
* 모든 설명과 요구사항은 명료한 경어체로 작성할 것.
* 지시사항을 확인했다는 등의 서론/결론은 일절 생략하고, 즉시 완성된 마크다운 문서 내용만 코드 블록 없이 출력하라.