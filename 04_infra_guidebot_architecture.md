# 오픈소스 LLM 기반 인프라가이드봇 설계

작성일: 2026-05-14  
수정일: 2026-05-14

## 1. 목표

이 문서는 사내 개발자를 위한 **인프라가이드봇** 설계 방향을 정리한다.

초기 범위는 Jira, Wiki, Workflow, 내부 게시판, 과거 신청 사례를 RAG로 검색해 인프라 신청 절차를 안내하는 것이었다. 그러나 실제 사용 시나리오를 고려하면 인프라가이드봇은 단순한 절차 안내 챗봇이 아니라 다음 기능을 포함하는 **업무 포털형 에이전트**로 설계해야 한다.

1. 절차 안내
2. 담당 부서 문의 라우팅
3. 신청서 초안 작성 지원
4. 아키텍처 기반 신청 항목 도출
5. 범위 외 문의 처리
6. 자동 처리 불가 시 Human handoff

다만 실제 신청 생성 자동화는 초기 핵심 기능이 아니다. 신청서 양식은 봇이 볼 수 있으므로 필요한 값을 어느 정도 미리 채운 **신청서 draft**를 만들 수 있지만, 실제 신청은 우선 사용자가 직접 처리하는 형태를 기본으로 한다. 향후 마지막 단계에 가까워졌을 때 API를 통해 draft를 만들고 Slack으로 신청자에게 전달하는 흐름을 추가할 수 있다.

따라서 이 봇의 핵심은 답변 생성 자체보다, 사용자의 요청을 올바른 업무 흐름으로 보내는 **Supervisor / Orchestrator** 설계에 있다.

## 2. 핵심 결론

권장 기본 구조는 다음과 같다.

```text
Supervisor + Hybrid RAG + Procedure Card + Request/Form Extractor + Critic + Human Handoff
```

한 줄로 정리하면 다음과 같다.

> 이 봇은 “답변 생성 모델”보다 “업무 라우팅 시스템”으로 설계해야 한다. Supervisor가 사용자의 의도를 절차 안내, 부서 문의, 신청서 초안 작성, 아키텍처 검토, 범위 외 문의로 먼저 나누고, Qwen3.5-120B는 복합 판단과 최종 답변에 집중시키는 구조가 가장 안정적이다.

권장 모델 역할은 다음과 같다.

| 역할 | 추천 모델 | 설명 |
|---|---|---|
| Supervisor / Router | Qwen 3.6-32B | 의도 분류, 도메인 분류, 범위 판단, 신청/문의/안내 분기 |
| Query Rewriter | Qwen 3.6-32B | RAG 검색 질의 재작성, 검색 범위 축소 |
| Form Extractor | Qwen 3.6-32B | 신청 필드 추출, 누락 필드 확인, 구조화 |
| Main Reasoner | Qwen3.5-120B | 복합 아키텍처 분석, 여러 부서 신청 항목 도출, 최종 답변 작성 |
| Critic / Verifier | gpt-oss-120B 또는 Qwen3.5-120B | 누락 항목, 부서 라우팅 오류, 근거 부족, 위험한 자동 신청 검토 |
| Summarizer / Fallback | Gemma4 | 낮은 중요도 문서 요약, 게시판/위키 글 요약, 간단 분류 보조 |

여러 모델을 병렬로 답변하게 한 뒤 합치는 방식은 추천하지 않는다. 하나의 메인 모델이 최종 답변을 만들고, 다른 모델은 라우팅, 추출, 검증, 요약 같은 제한된 역할에만 사용한다.

## 3. Supervisor, Reflection, Debate의 차이

| 패턴 | 핵심 구조 | 적합한 상황 | 주의점 |
|---|---|---|---|
| Supervisor | 중앙 관리자 Agent가 전문 Agent와 Tool을 호출 | 업무가 여러 도메인으로 나뉘는 경우 | 라우팅이 틀리면 전체 답변이 흔들림 |
| Reflection / Reflexion | 초안 생성 후 Critic이 검토하고 수정 반복 | 절차 안내, 신청서 초안, 아키텍처 검토 | 같은 모델이 자기 오류를 못 잡을 수 있음 |
| Debate | 여러 Agent가 가설을 주장하고 Judge가 판단 | 정책 충돌, 아키텍처 대안 비교, 복수 원인 분석 | 비용과 지연이 크고 Judge가 약하면 효과가 낮음 |

인프라가이드봇의 기본값은 다음이다.

```text
Supervisor + RAG + Form Extractor + Critic
```

복잡한 경우는 다음이다.

```text
Supervisor + Domain Agents + Critic
```

매우 복잡한 아키텍처 비교나 정책 충돌에는 다음을 선택적으로 쓴다.

```text
Supervisor + Domain Debate + Judge + Critic
```

Debate는 기본 기능으로 넣지 않는다. 부서 간 판단이 충돌하거나, 보안/비용/성능/안정성 사이의 trade-off가 큰 경우에만 사용한다.

## 4. 사용자 요청은 3축으로 분류한다

사용자 질문을 단순히 “NE 질문인가, DBA 질문인가”로만 분류하면 부족하다. Supervisor는 최소한 다음 3가지를 동시에 판단해야 한다.

1. Intent: 사용자가 무엇을 하려는가?
2. Domain: 어느 부서/영역과 관련 있는가?
3. Action: 답변만 필요한가, 문의/신청서 초안까지 필요한가?

예시는 다음과 같다.

| 사용자 질문 | Intent | Domain | Action |
|---|---|---|---|
| Redis 신청은 어떻게 해요? | 절차 문의 | DBA | 안내 |
| Redis 하나 신청해줘 | 신청서 작성 요청 | DBA | 신청서 draft 작성 |
| 운영 서버 방화벽 열어야 해요 | 절차/신청서 작성 | NE, Security, SE | 신청 지원 |
| 이 아키텍처면 뭐 신청해야 해요? | 아키텍처 분석 | NE, SE, DBA, Cloud, Security | 신청 항목 도출 |
| Kafka topic 권한이 안 돼요 | 문의/트러블슈팅 | Platform/Middleware | 담당부서 문의 |
| 노트북 고장났어요 | 범위 외 또는 IT 지원 | IT Helpdesk | 라우팅/범위 외 안내 |
| 휴가 신청 어디서 해요? | 범위 외 | HR/총무 | 범위 외 안내 |

Supervisor는 최종적으로 다음 질문에 답해야 한다.

- 절차 안내인가?
- 단순 문의인가?
- 신청서 초안 작성 요청인가?
- 아키텍처 검토인가?
- 담당 부서에 문의해야 하는가?
- 인프라 범위가 아닌가?
- 정보가 부족해서 추가 질문해야 하는가?
- Slack으로 신청서 draft를 전달할 수 있는 단계인가?

## 5. 전체 Agent 구조

권장 구조는 다음과 같다.

```text
[User]
  ↓
[Supervisor / Router]
  ↓
[Intent + Domain + Action Classification]
  ↓
[Hybrid RAG + Structured Catalog]
  ↓
업무 유형별 Agent
  ├─ Procedure Agent
  ├─ Department Q&A Agent
  ├─ Request Form Agent
  ├─ Architecture Analysis Agent
  ├─ Ticket/Jira Draft Agent
  ├─ Slack Draft Delivery Agent
  ├─ Out-of-Scope Handler
  └─ Human Handoff Agent
  ↓
[Critic / Verifier]
  ↓
[Final Response]
```

각 Agent의 역할은 다음과 같다.

| Agent | 역할 |
|---|---|
| Procedure Agent | 신청 절차, 필요 정보, 승인 라인, 진행 순서 안내 |
| Department Q&A Agent | NE/SE/DBA/Cloud/Security 부서별 문의 응답 |
| Request Form Agent | 신청서에 필요한 필드 추출, 누락 질문, 신청서 초안 생성 |
| Architecture Analysis Agent | 아키텍처 구성요소를 신청 항목으로 변환 |
| Ticket/Jira Draft Agent | 담당부서 문의 티켓 초안 작성 |
| Slack Draft Delivery Agent | API로 신청서 draft를 만들고 Slack으로 신청자에게 전달. 낮은 우선순위 |
| Out-of-Scope Handler | 인프라 범위와 관련 없는 문의 처리 |
| Human Handoff Agent | 자동 답변 불가 시 담당부서 연결 |
| Critic / Verifier | 누락, 충돌, 위험한 자동 처리, 근거 부족 검토 |

## 6. 도메인 분류는 부서 기준과 업무 기준을 같이 사용한다

부서만 기준으로 나누면 애매한 케이스가 많다. 예를 들어 “방화벽 열어주세요”는 NE만의 문제가 아닐 수 있다.

방화벽 오픈에는 다음 관점이 함께 걸릴 수 있다.

- NE: 네트워크 정책, ACL, 방화벽 rule
- Security: 보안 검토, 외부망 여부, 개인정보 처리 여부
- SE: 서버 출발지/목적지 확인, 운영 환경 확인
- 개발팀: 서비스 목적, 포트 사용 근거, 오픈 기간

따라서 도메인 taxonomy는 다음처럼 부서와 업무 항목을 함께 가진다.

```text
Domain
  ├─ NE
  │   ├─ 방화벽
  │   ├─ L4/L7
  │   ├─ DNS
  │   ├─ VIP
  │   ├─ GSLB
  │   └─ 네트워크 대역
  │
  ├─ SE
  │   ├─ 서버 발급
  │   ├─ OS 설정
  │   ├─ 계정/권한
  │   ├─ 모니터링
  │   ├─ 로그 수집
  │   └─ 백업
  │
  ├─ DBA
  │   ├─ DB 계정
  │   ├─ Schema
  │   ├─ Oracle/MySQL/PostgreSQL
  │   ├─ Redis
  │   └─ 백업/복구
  │
  ├─ Cloud
  │   ├─ VM
  │   ├─ Kubernetes
  │   ├─ Object Storage
  │   ├─ IAM
  │   ├─ LB
  │   └─ Managed Service
  │
  ├─ Security
  │   ├─ 개인정보
  │   ├─ 외부망 연동
  │   ├─ 접근통제
  │   ├─ 인증서
  │   └─ 보안성 검토
  │
  └─ Platform / Middleware
      ├─ Kafka
      ├─ MQ
      ├─ API Gateway
      ├─ CI/CD
      └─ Container Registry
```

이 구조가 있어야 Supervisor가 “어느 부서에 물어봐야 하는지”와 “무슨 신청이 필요한지”를 동시에 판단할 수 있다.

## 7. 신청 지원은 신청서 draft 중심으로 처리한다

“알려줘”와 “신청해줘”는 완전히 다르다. 신청 요청은 바로 생성하면 안 되고, 필수 입력값 확인과 사용자 확인 단계를 반드시 거쳐야 한다.

현재 우선순위에서 봇은 실제 신청을 대신 완료하는 시스템이 아니다. 신청서 양식은 봇이 볼 수 있으므로, 사용자의 입력과 양식 필드를 매핑해 **신청서 draft**를 미리 작성해주는 것이 목표다. 실제 신청 버튼을 누르거나 Workflow/Jira에서 최종 제출하는 일은 신청자가 직접 처리한다.

향후 낮은 우선순위 기능으로 다음을 추가할 수 있다.

```text
신청서 draft 생성
  ↓
API를 통해 draft 저장 또는 메시지 payload 생성
  ↓
Slack으로 신청자에게 신청서 draft 전달
  ↓
신청자가 직접 확인 및 제출
```

이 기능은 완료 직전 또는 운영 안정화 이후에 진행한다. 초기 버전에서는 신청 절차 안내, 필수 입력값 확인, 신청서 초안 작성, 담당 부서 문의 초안 작성이 우선이다.

예를 들어 사용자가 다음처럼 말할 수 있다.

> Redis 신청해줘.

이때 바로 신청하면 안 된다. 먼저 필요한 정보를 확인해야 한다.

Redis 신청에 필요한 정보 예시는 다음과 같다.

- 서비스명
- 담당자
- 운영/개발 환경
- Redis 용도
- 예상 TPS
- 메모리 용량
- HA 필요 여부
- 접근 서버 IP 또는 대역
- 개인정보 저장 여부
- 희망 완료일

신청 Agent는 다음 상태머신처럼 동작한다.

```text
신청 의도 감지
  ↓
신청 유형 판별
  ↓
신청서 양식 조회
  ↓
필수 입력값 조회
  ↓
사용자 입력값 추출
  ↓
누락 필드 확인
  ↓
신청서 draft 생성
  ↓
사용자 확인
  ↓
신청자가 직접 처리하도록 draft 제공
```

나중에 API/Slack 연동을 붙이면 마지막 단계만 다음처럼 확장한다.

```text
사용자 확인
  ↓
API로 신청서 draft 생성
  ↓
Slack으로 신청자에게 draft 전달
  ↓
신청자가 최종 확인 및 제출
```

다음과 같은 처리는 금지한다.

```text
사용자: Redis 신청해줘.
봇: 신청 완료했습니다.
```

## 8. 담당 부서 문의 기능

개발자는 절차만 묻는 것이 아니라 실제 부서에 문의하고 싶을 수 있다.

예시는 다음과 같다.

- 이 방화벽 신청이 가능한지 NE에 물어봐줘.
- Oracle DB 용량 산정이 맞는지 DBA에 문의하고 싶어.
- 이 구조에서 L4가 필요한지 확인해줘.
- Kafka Topic ACL 구성이 맞는지 플랫폼팀에 문의해줘.

이 경우는 신청이 아니라 문의 티켓이다. 별도 흐름이 필요하다.

```text
문의 의도 감지
  ↓
담당 부서 추정
  ↓
문의 초안 생성
  ↓
필요 정보 누락 확인
  ↓
사용자 확인
  ↓
사용자가 직접 문의하거나, 향후 Slack/Jira draft 전달
```

문의 초안 형식 예시는 다음과 같다.

```markdown
제목:
[DBA 문의] 신규 서비스 Redis 구성 가능 여부 확인 요청

본문:
안녕하세요. 아래 신규 서비스 구성과 관련하여 Redis 구성 가능 여부를 문의드립니다.

1. 서비스명:
2. 환경: 운영/개발
3. 사용 목적:
4. 예상 트래픽:
5. 필요 용량:
6. 접근 서버:
7. 희망 일정:

확인 요청 사항:
- Redis 사용 가능 여부
- 권장 구성
- HA 필요 여부
- 추가 신청 필요 항목
```

이 기능은 개발자가 담당 부서를 몰라도 적절한 문의 초안을 만들 수 있게 해준다.

## 9. 범위 외 문의 처리

부서와 관련 없는 문의가 들어올 수 있다. 이를 억지로 답하면 봇 신뢰도가 떨어진다.

예시는 다음과 같다.

- 휴가 신청 어디서 해요?
- 법인카드 한도 올려주세요.
- 노트북 고장났어요.
- 회의실 예약 어떻게 해요?
- 급여명세서 어디서 봐요?
- 사내 식당 메뉴 알려줘.

Out-of-Scope Handler는 다음 순서로 처리한다.

1. 인프라 영역인지 판단
2. 관련 채널이 알려져 있으면 안내
3. 모르면 모른다고 말하고 사내 포털/Helpdesk 검색을 유도

답변 예시는 다음과 같다.

```text
이 문의는 NE/SE/DBA/Cloud 인프라 신청 범위와는 직접 관련이 낮아 보입니다.

다만 사내 IT 장비 문제라면 IT Helpdesk 또는 자산관리/총무 쪽 문의일 가능성이 있습니다. 정확한 신청 경로는 사내 포털에서 “노트북 수리” 또는 “IT Helpdesk”로 검색해보는 것을 권장합니다.
```

핵심은 모르는 것을 억지로 답하지 않는 것이다.

## 10. 아키텍처 기반 신청 항목 도출

아키텍처 질문은 단순 RAG Q&A가 아니다. 사용자가 설명한 구성요소를 분해하고, 필요한 신청 항목과 담당 부서로 변환해야 한다.

예시 입력은 다음과 같다.

```text
외부 사용자가 접속하는 웹서비스를 만들려고 합니다.
WAS 4대, Redis, Kafka, Oracle DB가 필요하고 외부 API와도 통신합니다.
```

내부 구조화 결과 예시는 다음과 같다.

```json
{
  "components": [
    "external user access",
    "WAS 4",
    "Redis",
    "Kafka",
    "Oracle DB",
    "external API"
  ],
  "required_requests": [
    "서버/VM/컨테이너 자원 신청",
    "DNS 신청",
    "L4/L7 신청",
    "방화벽 신청",
    "Redis 신청",
    "Kafka topic/account 신청",
    "Oracle DB schema/account 신청",
    "외부망 연동 보안 검토",
    "모니터링 등록",
    "로그 수집 등록",
    "백업 정책 확인"
  ],
  "related_departments": [
    "SE",
    "NE",
    "DBA",
    "Security",
    "Platform"
  ],
  "missing_info": [
    "운영/개발 환경",
    "개인정보 처리 여부",
    "예상 트래픽",
    "도메인명",
    "접근 주체 IP",
    "희망 오픈일"
  ]
}
```

최종 답변은 다음처럼 구성한다.

```markdown
말씀하신 구성은 여러 부서 신청이 필요한 복합 인프라 구성입니다.

필요 신청 항목은 다음과 같습니다.

1. SE: WAS 서버 또는 컨테이너 자원 신청
2. NE: DNS, L4/L7, 방화벽 신청
3. DBA: Oracle DB 계정/스키마 신청
4. DBA 또는 Platform: Redis 신청
5. Platform: Kafka Topic/ACL 신청
6. Security: 외부 API 연동 보안 검토
7. SE/관제: 모니터링 및 로그 수집 등록

추가로 아래 정보가 있어야 신청서 초안을 만들 수 있습니다.
- 운영/개발 환경
- 서비스명
- 개인정보 처리 여부
- 예상 TPS
- 도메인명
- 출발지/목적지 IP
- 오픈 희망일
```

## 11. RAG 문서는 Procedure Card로 구조화한다

Claude Sonnet/Opus와 오픈소스 LLM의 답변 품질 차이를 줄이려면, RAG 문서를 단순 chunk로만 저장하지 말고 신청 절차 단위의 **Procedure Card**로 구조화해야 한다.

예시는 다음과 같다.

```json
{
  "procedure_name": "Redis 신규 신청",
  "domain": "DBA",
  "request_type": "신규 발급",
  "target_user": "개발자",
  "when_to_use": [
    "서비스에서 캐시가 필요한 경우",
    "세션 저장소가 필요한 경우"
  ],
  "required_fields": [
    "서비스명",
    "담당 조직",
    "운영/개발 구분",
    "예상 TPS",
    "메모리 용량",
    "HA 필요 여부",
    "접근 서버 대역",
    "개인정보 저장 여부"
  ],
  "request_system": "Workflow",
  "draft_supported": true,
  "final_submission_owner": "requester",
  "approval_line": [
    "개발팀 리더",
    "DBA 담당자",
    "보안 검토"
  ],
  "related_teams": [
    "DBA",
    "SE",
    "Security"
  ],
  "related_requests": [
    "방화벽 신청",
    "모니터링 등록"
  ],
  "sla": "영업일 기준 3일",
  "pre_check": [
    "개인정보 저장 여부 확인",
    "운영망 접근 정책 확인",
    "방화벽 오픈 필요 여부 확인"
  ],
  "post_steps": [
    "접속 정보 수령",
    "방화벽 신청",
    "모니터링 등록",
    "장애 연락망 등록"
  ],
  "source_documents": [
    "wiki_url",
    "workflow_form_url",
    "jira_example_url"
  ],
  "last_verified_at": "2026-05-14",
  "status": "active"
}
```

이렇게 구조화하면 모델은 자연어 문서에서 절차를 추측하지 않고, 정리된 필드를 바탕으로 안정적으로 답변할 수 있다.

## 12. 핵심 Structured Data

이 봇은 RAG chunk만으로 만들면 한계가 있다. 최소한 다음 4가지 structured data가 필요하다.

### 12.1 부서 카탈로그

```json
{
  "department": "NE",
  "responsibilities": [
    "방화벽",
    "DNS",
    "L4/L7",
    "VIP",
    "GSLB",
    "네트워크 대역"
  ],
  "contact_channel": "Jira NE 문의",
  "request_system": "Workflow",
  "handoff_rules": [
    "외부망 연동은 보안 검토 포함",
    "운영망 방화벽은 서비스 오너 승인 필요"
  ]
}
```

### 12.2 신청 유형 카탈로그

```json
{
  "request_type": "방화벽 오픈",
  "domain": ["NE", "Security"],
  "required_fields": [
    "출발지 IP",
    "목적지 IP",
    "목적지 Port",
    "Protocol",
    "사용 목적",
    "서비스명",
    "오픈 기간",
    "운영/개발 구분"
  ],
  "approval": [
    "서비스 오너",
    "Security",
    "NE"
  ],
  "workflow_form": "방화벽 신청서",
  "draft_supported": true,
  "final_submission_owner": "requester",
  "automation_priority": "low"
}
```

### 12.3 절차 카드

```json
{
  "procedure_name": "Redis 신규 신청",
  "domain": "DBA",
  "steps": [
    "Workflow에서 Redis 신규 신청 선택",
    "필수 정보 입력",
    "DBA 검토",
    "구성 완료 후 접속 정보 전달"
  ],
  "required_fields": [
    "서비스명",
    "환경",
    "용량",
    "HA 여부",
    "접근 서버"
  ],
  "related_requests": [
    "방화벽 신청",
    "모니터링 등록"
  ],
  "draft_policy": "봇은 신청서 초안을 만들 수 있지만 최종 신청은 신청자가 처리한다."
}
```

### 12.4 범위 외 라우팅 규칙

```json
{
  "topic": "노트북/PC 장애",
  "infra_bot_scope": false,
  "recommended_channel": "IT Helpdesk",
  "response_policy": "직접 처리하지 않고 담당 채널 안내"
}
```

## 13. RAG 검색 구조

단순 벡터 검색만으로는 부족하다.

권장 검색 구조는 다음과 같다.

```text
Hybrid Search
= BM25 keyword search
+ Vector search
+ Metadata filter
+ Reranker
```

중요한 metadata 예시는 다음과 같다.

| 필드 | 예시 |
|---|---|
| service | redis, kafka, oracle, firewall, dns |
| domain | NE, SE, DBA, Cloud, Security, Platform |
| environment | dev, stg, prod |
| request_type | 신규, 변경, 삭제, 권한, 장애, 예외, 문의 |
| action_type | guide, ask_department, create_draft, draft_ticket |
| source | wiki, jira, workflow, board, incident |
| last_verified_at | 2026-05-14 |
| owner_team | DBA, Network, Cloud Platform |
| status | active, deprecated, draft |
| automation_priority | high, medium, low |

검색 결과는 최신성, 문서 출처, 업무 도메인, 신청 유형, 사용자의 action 의도를 기준으로 재정렬해야 한다.

## 14. Supervisor 판단 JSON

Supervisor는 매번 내부적으로 구조화된 판단 결과를 만들어야 한다.

신청서 draft 요청 예시는 다음과 같다.

```json
{
  "intent": "request_draft_creation",
  "domain": ["DBA", "NE"],
  "request_type": "Redis 신규 신청",
  "is_in_scope": true,
  "needs_human_handoff": false,
  "needs_user_confirmation": true,
  "can_create_draft": true,
  "can_submit_request": false,
  "final_submission_owner": "requester",
  "automation_priority": "low",
  "required_fields": [
    "서비스명",
    "환경",
    "용량",
    "접근 서버",
    "HA 여부"
  ],
  "missing_fields": [
    "용량",
    "접근 서버",
    "HA 여부"
  ],
  "next_action": "ask_missing_fields"
}
```

범위 외 문의 예시는 다음과 같다.

```json
{
  "intent": "general_question",
  "domain": [],
  "request_type": null,
  "is_in_scope": false,
  "out_of_scope_reason": "HR/총무성 문의로 판단됨",
  "recommended_channel": "사내 포털 또는 HR Helpdesk",
  "next_action": "out_of_scope_response"
}
```

부서 문의 예시는 다음과 같다.

```json
{
  "intent": "department_question",
  "domain": ["NE", "Security"],
  "request_type": "방화벽 오픈 가능 여부 문의",
  "is_in_scope": true,
  "needs_human_handoff": true,
  "needs_user_confirmation": true,
  "required_fields": [
    "서비스명",
    "출발지 IP",
    "목적지 IP",
    "목적지 Port",
    "사용 목적"
  ],
  "missing_fields": [
    "목적지 IP",
    "목적지 Port"
  ],
  "next_action": "draft_department_ticket"
}
```

이 JSON을 먼저 만든 뒤 답변해야 답변 품질이 안정된다.

## 15. 답변 포맷

### 15.1 절차 안내형

```markdown
## 요약
- 무엇을 신청해야 하는지

## 신청 위치
- Workflow / Wiki / Jira 등

## 신청 전에 필요한 정보
- 서비스명
- 환경
- 담당자
- 용량
- 네트워크 정보

## 진행 절차
1. 신청서 작성
2. 승인
3. 담당팀 처리
4. 완료 확인

## 관련 부서
- NE / SE / DBA / Cloud / Security

## 주의사항
- 방화벽 필요 여부
- 운영 환경 승인 여부
- 개인정보/보안 검토 여부

## 근거
- 참조 문서
```

### 15.2 신청서 draft 지원형

```markdown
## 신청 유형
- Redis 신규 신청

## 현재 확인된 정보
- 서비스명:
- 환경:
- 담당자:

## 추가로 필요한 정보
1. 예상 메모리 용량
2. 접근 서버 IP 또는 대역
3. HA 필요 여부
4. 개인정보 저장 여부
5. 희망 완료일

## 처리 방식
- 봇은 신청서 초안을 작성합니다.
- 최종 신청은 신청자가 Workflow/Jira에서 직접 확인 후 제출합니다.
- 향후 API/Slack 연동이 추가되면 신청서 draft를 Slack으로 전달할 수 있습니다.

위 정보를 알려주시면 신청서 초안을 만들어드리겠습니다.
```

### 15.3 부서 문의형

```markdown
## 문의 대상 부서
- NE / Security

## 문의 초안
제목:
[NE 문의] 방화벽 오픈 가능 여부 확인 요청

본문:
1. 서비스명:
2. 출발지 IP:
3. 목적지 IP:
4. 목적지 Port:
5. 사용 목적:
6. 희망 일정:

## 추가로 필요한 정보
- 목적지 IP
- 목적지 Port
```

### 15.4 아키텍처 검토형

```markdown
## 아키텍처 해석
- 사용자가 말한 구성요소 정리

## 필요 신청 항목
1. 서버/컨테이너 자원
2. DB
3. Redis
4. Kafka
5. DNS
6. L4/L7
7. 방화벽
8. 모니터링
9. 로그 수집
10. 보안 검토

## 관련 부서
- SE
- NE
- DBA
- Platform
- Security

## 누락 확인 질문
- 운영/개발 환경인가?
- 외부망 연동이 있는가?
- 개인정보가 있는가?
- HA가 필요한가?
- 예상 트래픽은 얼마인가?

## 권장 진행 순서
1. 아키텍처 검토
2. 보안 검토
3. 자원 신청
4. 네트워크 신청
5. DB/미들웨어 신청
6. 모니터링/로그 등록

## 근거
- 참조 문서
```

### 15.5 범위 외 문의형

```markdown
이 문의는 NE/SE/DBA/Cloud 인프라 신청 범위와는 직접 관련이 낮아 보입니다.

관련 채널이 확인되는 경우:
- 권장 문의 채널: IT Helpdesk / HR Helpdesk / 사내 포털

관련 채널이 확인되지 않는 경우:
- 사내 포털에서 관련 키워드로 검색하거나 Helpdesk에 문의하는 것을 권장합니다.
```

## 16. Critic 검증 체크리스트

Critic Agent는 추상적으로 “검토해줘”가 아니라 명확한 체크리스트를 받아야 한다.

권장 체크리스트는 다음과 같다.

1. 답변이 제공된 문서, Workflow, Jira, Wiki 근거에 기반하는가?
2. 확인된 사실과 추측을 분리했는가?
3. 필요한 신청 항목을 누락하지 않았는가?
4. 불필요하거나 잘못된 신청 항목을 추가하지 않았는가?
5. 담당 부서와 승인 라인이 정확한가?
6. 최신 문서를 우선했는가?
7. deprecated 절차를 사용하지 않았는가?
8. 운영/개발 환경, 보안, 개인정보, 외부망 연동 조건을 확인했는가?
9. 실제 신청을 완료했다고 말하지 않았는가?
10. 봇의 역할이 신청서 draft 작성이며 최종 제출은 신청자 역할임을 분명히 했는가?
11. 부서 문의와 신청서 draft를 혼동하지 않았는가?
12. 범위 외 문의를 억지로 답하지 않았는가?
13. 자동 판단이 어려운 경우 Human handoff를 제안했는가?
14. Slack/API draft 전달 기능을 낮은 우선순위로 분리했는가?
15. 개발자가 바로 행동할 수 있는 수준으로 답변했는가?

## 17. 운영 파라미터

운영/절차/보안 안내는 창의성이 필요하지 않다.

권장 생성 파라미터는 다음과 같다.

```yaml
temperature: 0.0-0.2
top_p: 0.8-0.95
```

출력은 가능한 한 구조화한다.

```json
{
  "intent": "procedure_guide|department_question|request_draft_creation|architecture_analysis|out_of_scope",
  "is_in_scope": true,
  "domain": [],
  "required_requests": [],
  "conditional_requests": [],
  "missing_fields": [],
  "related_teams": [],
  "can_create_draft": false,
  "can_submit_request": false,
  "final_submission_owner": "requester",
  "needs_user_confirmation": false,
  "needs_human_handoff": false,
  "evidence": [],
  "risk_level": "low|medium|high|critical",
  "final_answer": ""
}
```

## 18. 우선순위

초기 구현 우선순위는 다음과 같다.

| 우선순위 | 기능 | 설명 |
|---|---|---|
| P0 | 절차 안내 RAG | Wiki/Jira/Workflow/게시판 기반 절차 안내 |
| P0 | Intent/Domain/Action 분류 | Supervisor 핵심 분기 |
| P0 | 아키텍처 기반 신청 항목 도출 | 구성요소를 신청 항목과 부서로 변환 |
| P1 | 신청서 초안 작성 | 신청서 양식을 기반으로 필요한 필드와 draft 작성 |
| P1 | 부서 문의 초안 작성 | 담당부서 문의 제목/본문 생성 |
| P1 | 범위 외 문의 처리 | 인프라 외 문의를 적절히 라우팅 |
| P2 | Critic 검증 | 누락, 충돌, 근거 부족, 위험 검토 |
| P3 | API 기반 draft 생성 | 신청서 draft를 API로 저장하거나 payload 생성 |
| P3 | Slack draft 전달 | 신청자에게 Slack으로 draft 전달 후 사용자가 직접 처리 |

API/Slack 기반 신청서 draft 전달은 매우 낮은 우선순위다. 거의 완료 직전 또는 운영 안정화 이후에 진행한다.

## 19. 평가셋 구성

Claude와 비교하려면 감각적인 평가보다 내부 평가셋이 필요하다.

최소 120개 샘플을 만든다.

| 유형 | 개수 |
|---|---:|
| 단순 신청 절차 질문 | 25 |
| 복합 신청 절차 질문 | 20 |
| 신청서 draft 요청 | 15 |
| 부서 문의 요청 | 15 |
| 아키텍처 기반 신청 항목 도출 | 20 |
| 담당 부서/승인 라인 질문 | 10 |
| 문서 충돌/구버전 문서 판단 | 5 |
| 범위 외 문의 | 5 |
| 애매한 질문에서 되물어야 하는 케이스 | 5 |

평가 기준은 다음과 같다.

- Intent, Domain, Action을 정확히 분류했는가?
- 필요한 신청 항목을 누락하지 않았는가?
- 잘못된 신청 절차를 안내하지 않았는가?
- 담당 부서 라우팅이 정확한가?
- 신청과 문의를 구분했는가?
- 실제 신청 완료로 오해하게 만들지 않았는가?
- 신청서 draft와 최종 제출 주체를 명확히 분리했는가?
- 범위 외 문의를 억지로 답하지 않았는가?
- 근거 문서를 제대로 사용했는가?
- 최신 문서를 우선했는가?
- 모르는 부분을 단정하지 않았는가?
- 추가 확인 질문이 적절한가?
- 개발자가 바로 행동할 수 있는 답변인가?

비교 실험안은 다음과 같다.

| 실험안 | 구성 |
|---|---|
| A안 | Qwen3.5-120B 단독 |
| B안 | Qwen3.5-120B + Qwen 32B Router |
| C안 | Qwen3.5-120B + Qwen 32B Router + gpt-oss Critic |
| D안 | Qwen 32B 단순 Q&A + Qwen3.5-120B 복잡 Q&A |
| E안 | gpt-oss-120B 단독 |
| F안 | Supervisor + RAG + Form Extractor + Critic + Handoff |

예상 우선순위는 다음과 같다.

1. Supervisor + RAG + Form Extractor + Critic + Handoff
2. Qwen3.5-120B + Qwen 32B Router + Critic
3. Qwen3.5-120B 단독 + 좋은 RAG
4. Qwen 32B + 좋은 RAG
5. 여러 모델 답변 단순 혼합

## 20. 최종 권장 아키텍처

최종 권장 아키텍처는 다음과 같다.

```text
[User Query]
   ↓
[Supervisor / Router - Qwen 32B]
   ↓
[Intent + Domain + Action Classification]
   ├─ 절차 안내
   ├─ 부서 문의
   ├─ 신청서 draft 작성
   ├─ 아키텍처 기반 신청 항목 도출
   ├─ 정책/보안 기준 문의
   ├─ 범위 외 문의
   └─ Human handoff 필요 여부
   ↓
[Hybrid RAG + Structured Catalog]
   ├─ Wiki
   ├─ Jira
   ├─ Workflow 신청 양식
   ├─ 내부 게시판
   ├─ 과거 문의/신청 사례
   ├─ 부서 카탈로그
   ├─ 신청 유형 카탈로그
   ├─ 절차 카드
   └─ 범위 외 라우팅 규칙
   ↓
[Task Agent]
   ├─ Procedure Agent
   ├─ Department Q&A Agent
   ├─ Request Form Agent
   ├─ Architecture Analysis Agent
   ├─ Ticket/Jira Draft Agent
   ├─ Slack Draft Delivery Agent (P3)
   ├─ Out-of-Scope Handler
   └─ Human Handoff Agent
   ↓
[Reasoning / Answer Draft - Qwen3.5-120B]
   ↓
[Critic / Verifier - gpt-oss-120B or Qwen3.5-120B]
   ↓
[Final Answer - Qwen3.5-120B]
```

## 21. 최종 추천

지금 만들려는 인프라가이드봇에는 다음 구성이 가장 적합하다.

1. Qwen 32B Supervisor
   - 의도 분류
   - 도메인 분류
   - 범위 외 판단
   - 신청/문의/안내 분기

2. Qwen 32B Extractor
   - 사용자 입력에서 신청 필드 추출
   - 누락 필드 확인
   - 검색 질의 재작성

3. Hybrid RAG + Reranker
   - Wiki
   - Jira
   - Workflow 신청 양식
   - 내부 게시판
   - 과거 문의/신청 사례
   - 부서/신청 유형/절차 카탈로그

4. Qwen3.5-120B Main Agent
   - 절차 종합
   - 아키텍처 분석
   - 여러 부서 신청 항목 도출
   - 최종 답변 작성

5. gpt-oss-120B Critic
   - 누락 신청 항목 확인
   - 부서 라우팅 오류 확인
   - 근거 부족 확인
   - 실제 신청 완료로 오해되는 표현 검토

6. Human Handoff
   - 자동 판단 불가
   - 부서 확인 필요
   - 사내 정책 충돌
   - 권한/승인이 필요한 실제 신청

7. API/Slack Draft Delivery
   - 낮은 우선순위
   - 신청서 draft를 API로 만들고 Slack으로 신청자에게 전달
   - 최종 제출은 신청자가 직접 수행
   - 거의 완료 직전 또는 운영 안정화 이후 진행

최종 결론은 다음과 같다.

> 인프라가이드봇은 단순 절차 안내 RAG가 아니라 문의 분류, 담당부서 라우팅, 신청서 draft 작성, 범위 외 문의 처리까지 포함하는 업무 포털형 에이전트로 설계해야 한다. 실제 신청 자동화는 초기 목표가 아니며, 봇은 신청서 양식을 바탕으로 초안을 만들어 신청자가 처리할 수 있게 돕는 역할부터 시작한다. Qwen 32B는 분류와 추출에, Qwen3.5-120B는 복합 판단과 최종 답변에, gpt-oss-120B는 검증과 위험 통제에 배치하는 것이 가장 안정적이다.
