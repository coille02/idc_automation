# 오픈소스 LLM 기반 인프라가이드봇 설계

작성일: 2026-05-14  
수정일: 2026-05-14

## 1. 목표

이 문서는 사내 개발자를 위한 **인프라가이드봇** 설계 방향을 정리한다.

인프라가이드봇은 단순한 절차 안내 RAG 챗봇이 아니라, 개발자의 인프라 관련 질문을 적절한 업무 흐름으로 라우팅하는 **업무 포털형 에이전트**로 설계한다.

봇이 다뤄야 하는 주요 범위는 다음과 같다.

1. 신청 절차 안내
2. 신청서 양식 기반 채팅 추천
3. 담당 부서 문의 라우팅
4. 아키텍처 기반 신청 항목 도출
5. 범위 외 문의 처리
6. 자동 판단 불가 시 Human handoff

다만 **초기 구현 범위는 훨씬 단순하게 잡는다.**

처음 구현해야 하는 것은 신청서 양식을 봇이 보고, 사용자의 채팅 입력을 바탕으로 각 신청서 필드에 어떤 값을 쓰면 좋을지 추천해주는 수준이다. 실제 신청 생성, API draft 생성, Slack 전달은 초기 핵심 기능이 아니며 거의 완료 직전 또는 운영 안정화 이후의 낮은 우선순위 기능으로 둔다.

## 2. 핵심 결론

초기 구현의 권장 구조는 다음과 같다.

```text
User Chat
  ↓
Qwen3.6 Supervisor
  ↓
신청서 양식 / Procedure Card 검색
  ↓
Qwen3.6 Form Field Recommender
  ↓
Qwen3.6 Self-Critique
  ↓
채팅으로 신청서 작성값 추천
```

확장 구조는 다음과 같다.

```text
Supervisor + Hybrid RAG + Procedure Card + Form Field Recommender + Self-Critique + Human Handoff
```

한 줄로 정리하면 다음과 같다.

> 이 봇은 실제 신청을 대신 처리하는 자동화 시스템이 아니라, 먼저 신청서 양식에 맞춰 개발자가 어떤 값을 작성하면 되는지 채팅으로 추천하는 도우미부터 시작한다. Supervisor가 절차 안내, 부서 문의, 신청서 양식 추천, 아키텍처 검토, 범위 외 문의를 나누고, 복잡한 판단이 필요한 경우에만 큰 모델이나 Debate를 호출한다.

## 3. 내부 모델 실험 결과 반영

간단한 단문/장문 질문 테스트 결과, 기존의 “Qwen3.5-120B를 항상 메인으로 두고 gpt-oss-120B를 Critic으로 둔다”는 가정은 조정이 필요하다.

관찰 결과는 다음과 같다.

| 실험 항목 | 관찰 결과 | 설계 반영 |
|---|---|---|
| Reflexion / self-critique | Qwen3.6 self-critique가 Qwen3.5보다 점수가 높음 | 초기 검증/자기수정은 Qwen3.6-32B self-critique를 우선 사용 |
| Debate pair 회전 | Qwen3.6 + Gemma debater pair가 미세하게 최고. Qwen3.5 + Qwen3.6 pair와 차이는 약 0.8~0.9% | Debate가 필요할 때는 Qwen3.6 + Gemma pair를 후보로 유지 |
| Debate judge 회전 | Debater를 Qwen3.5 + Qwen3.6으로 고정했을 때 Gemma judge가 압도적으로 높음 | Judge가 필요한 경우 Gemma를 우선 후보로 둔다 |
| gpt-oss critic | 약 93%에서 raw JSON이 누출됨 | 초기 Critic/Verifier로 사용하지 않는다 |

따라서 초기 모델 배치는 다음과 같이 수정한다.

| 역할 | 우선 모델 | 비고 |
|---|---|---|
| Supervisor / Router | Qwen3.6-32B | Intent, Domain, Action 분류 |
| 신청서 양식 추천 | Qwen3.6-32B | 신청서 필드와 사용자 입력을 매핑하고 작성값을 추천 |
| Self-Critique | Qwen3.6-32B | 단문/장문 테스트에서 Qwen3.5보다 우수했던 결과 반영 |
| 복합 아키텍처 분석 | Qwen3.5-120B | 복잡한 구성요소 도출, 여러 부서 신청 항목 종합 시 사용 |
| Debate Debater | Qwen3.6-32B + Gemma | 정책 충돌이나 복잡한 판단에만 선택적으로 사용 |
| Debate Judge | Gemma | 내부 실험에서 judge 성능이 가장 높았던 결과 반영 |
| gpt-oss-120B | 초기 제외 | raw JSON 누출 문제가 해결될 때까지 Critic으로 사용하지 않음 |

## 4. Supervisor, Reflection, Debate의 차이

| 패턴 | 핵심 구조 | 적합한 상황 | 이번 설계에서의 위치 |
|---|---|---|---|
| Supervisor | 중앙 관리자 Agent가 전문 Agent와 Tool을 호출 | 업무가 여러 도메인으로 나뉘는 경우 | 필수 |
| Reflection / Reflexion | 초안 생성 후 Critic이 검토하고 수정 반복 | 신청서 필드 추천, 절차 안내, 누락 확인 | Qwen3.6 self-critique로 우선 적용 |
| Debate | 여러 Agent가 가설을 주장하고 Judge가 판단 | 정책 충돌, 아키텍처 대안 비교, 복수 원인 분석 | 기본 기능 아님. 필요 시 Qwen3.6 + Gemma, Judge는 Gemma |

초기 기본값은 다음이다.

```text
Supervisor + RAG + Form Field Recommender + Qwen3.6 Self-Critique
```

복잡한 경우는 다음이다.

```text
Supervisor + Qwen3.5 Architecture Reasoner + Qwen3.6 Self-Critique
```

매우 복잡한 아키텍처 비교나 정책 충돌에는 다음을 선택적으로 쓴다.

```text
Supervisor + Debate(Qwen3.6 + Gemma) + Judge(Gemma)
```

## 5. 사용자 요청은 3축으로 분류한다

Supervisor는 사용자 요청을 최소한 다음 3가지 축으로 분류해야 한다.

1. Intent: 사용자가 무엇을 하려는가?
2. Domain: 어느 부서/영역과 관련 있는가?
3. Action: 답변만 필요한가, 신청서 필드 추천이나 문의 초안까지 필요한가?

예시는 다음과 같다.

| 사용자 질문 | Intent | Domain | Action |
|---|---|---|---|
| Redis 신청은 어떻게 해요? | 절차 문의 | DBA | 안내 |
| Redis 신청서에 뭐라고 적어야 해요? | 신청서 작성 지원 | DBA | 필드값 추천 |
| 운영 서버 방화벽 열어야 해요 | 절차/신청서 작성 | NE, Security, SE | 필드값 추천 |
| 이 아키텍처면 뭐 신청해야 해요? | 아키텍처 분석 | NE, SE, DBA, Cloud, Security | 신청 항목 도출 |
| Kafka topic 권한이 안 돼요 | 문의/트러블슈팅 | Platform/Middleware | 담당부서 문의 |
| 노트북 고장났어요 | 범위 외 또는 IT 지원 | IT Helpdesk | 라우팅/범위 외 안내 |
| 휴가 신청 어디서 해요? | 범위 외 | HR/총무 | 범위 외 안내 |

Supervisor는 다음 질문에 답해야 한다.

- 절차 안내인가?
- 단순 문의인가?
- 신청서 양식 작성 추천인가?
- 아키텍처 검토인가?
- 담당 부서에 문의해야 하는가?
- 인프라 범위가 아닌가?
- 정보가 부족해서 추가 질문해야 하는가?
- 실제 draft/API/Slack 연동이 필요한 단계인가, 아니면 채팅 추천만 하면 되는가?

## 6. 전체 Agent 구조

초기 구조는 다음과 같다.

```text
[User]
  ↓
[Qwen3.6 Supervisor / Router]
  ↓
[Intent + Domain + Action Classification]
  ↓
[Hybrid RAG + 신청서 양식 + Procedure Card]
  ↓
[Qwen3.6 Form Field Recommender]
  ↓
[Qwen3.6 Self-Critique]
  ↓
[Final Chat Recommendation]
```

확장 구조는 다음과 같다.

```text
[User]
  ↓
[Supervisor / Router]
  ↓
[Hybrid RAG + Structured Catalog]
  ↓
업무 유형별 Agent
  ├─ Procedure Agent
  ├─ Department Q&A Agent
  ├─ Request Form Agent
  ├─ Architecture Analysis Agent
  ├─ Ticket/Jira Draft Agent
  ├─ Slack Draft Delivery Agent(P3)
  ├─ Out-of-Scope Handler
  └─ Human Handoff Agent
  ↓
[Verifier / Self-Critique]
  ↓
[Final Response]
```

각 Agent의 역할은 다음과 같다.

| Agent | 역할 | 초기 여부 |
|---|---|---|
| Procedure Agent | 신청 절차, 필요 정보, 승인 라인, 진행 순서 안내 | P0 |
| Request Form Agent | 신청서 양식 필드 추출, 필드별 추천값 생성, 누락 질문 | P0 |
| Department Q&A Agent | NE/SE/DBA/Cloud/Security 부서별 문의 응답 | P1 |
| Architecture Analysis Agent | 아키텍처 구성요소를 신청 항목으로 변환 | P1 |
| Ticket/Jira Draft Agent | 담당부서 문의 티켓 초안 작성 | P2 |
| Slack Draft Delivery Agent | API로 신청서 draft를 만들고 Slack으로 신청자에게 전달 | P3 |
| Out-of-Scope Handler | 인프라 범위와 관련 없는 문의 처리 | P1 |
| Human Handoff Agent | 자동 답변 불가 시 담당부서 연결 | P2 |

## 7. 신청 지원은 채팅 기반 필드 추천부터 시작한다

현재 우선순위에서 봇은 실제 신청을 대신 완료하는 시스템이 아니다. 또한 초기에는 신청서 draft를 시스템에 저장하는 것도 목표가 아니다.

초기 목표는 다음이다.

1. 신청서 양식을 읽는다.
2. 양식의 필수/선택 필드를 파악한다.
3. 사용자 채팅에서 이미 제공된 값을 필드에 매핑한다.
4. 부족한 필드를 질문한다.
5. 각 필드에 어떤 값을 쓰면 좋을지 채팅으로 추천한다.
6. Qwen3.6 self-critique로 누락, 추측, 잘못된 매핑을 점검한다.

예를 들어 사용자가 다음처럼 말할 수 있다.

> Redis 신청하려고 하는데 서비스는 pay-api고 운영 환경이야. 세션 저장용이고 HA는 필요해.

봇은 다음처럼 답하는 것이 초기 목표다.

```markdown
Redis 신청서 기준으로 현재 채울 수 있는 항목은 다음입니다.

## 확인된 값
- 서비스명: pay-api
- 환경: 운영
- 사용 목적: 세션 저장
- HA 필요 여부: 필요

## 추가로 필요한 값
1. 예상 메모리 용량
2. 예상 TPS 또는 동시 접속 규모
3. 접근 서버 IP 또는 대역
4. 개인정보 저장 여부
5. 희망 완료일

## 작성 추천
- Redis 용도: 운영 서비스 pay-api의 세션 저장소
- HA 필요 여부: 운영 환경이므로 HA 필요로 작성 권장
- 개인정보 저장 여부: 세션에 개인정보가 포함되는지 확인 필요
```

금지해야 할 응답은 다음이다.

```text
신청 완료했습니다.
```

봇은 신청서를 작성하도록 돕지만, 최종 신청은 신청자가 직접 처리한다.

## 8. API/Slack draft 전달은 낮은 우선순위다

향후 낮은 우선순위 기능으로 다음을 추가할 수 있다.

```text
신청서 필드 추천
  ↓
사용자 확인
  ↓
API를 통해 draft 저장 또는 메시지 payload 생성
  ↓
Slack으로 신청자에게 신청서 draft 전달
  ↓
신청자가 직접 확인 및 제출
```

이 기능은 거의 완료 직전 또는 운영 안정화 이후에 진행한다.

초기 버전에서는 다음에 집중한다.

- 신청 절차 안내
- 신청서 양식 기반 필드 추천
- 필수 입력값 확인
- Qwen3.6 self-critique
- 범위 외 문의 처리

## 9. 도메인 분류는 부서 기준과 업무 기준을 같이 사용한다

부서만 기준으로 나누면 애매한 케이스가 많다. 예를 들어 “방화벽 열어주세요”는 NE만의 문제가 아닐 수 있다.

방화벽 오픈에는 다음 관점이 함께 걸릴 수 있다.

- NE: 네트워크 정책, ACL, 방화벽 rule
- Security: 보안 검토, 외부망 여부, 개인정보 처리 여부
- SE: 서버 출발지/목적지 확인, 운영 환경 확인
- 개발팀: 서비스 목적, 포트 사용 근거, 오픈 기간

권장 taxonomy는 다음과 같다.

```text
Domain
  ├─ NE
  │   ├─ 방화벽
  │   ├─ L4/L7
  │   ├─ DNS
  │   ├─ VIP
  │   ├─ GSLB
  │   └─ 네트워크 대역
  ├─ SE
  │   ├─ 서버 발급
  │   ├─ OS 설정
  │   ├─ 계정/권한
  │   ├─ 모니터링
  │   ├─ 로그 수집
  │   └─ 백업
  ├─ DBA
  │   ├─ DB 계정
  │   ├─ Schema
  │   ├─ Oracle/MySQL/PostgreSQL
  │   ├─ Redis
  │   └─ 백업/복구
  ├─ Cloud
  │   ├─ VM
  │   ├─ Kubernetes
  │   ├─ Object Storage
  │   ├─ IAM
  │   ├─ LB
  │   └─ Managed Service
  ├─ Security
  │   ├─ 개인정보
  │   ├─ 외부망 연동
  │   ├─ 접근통제
  │   ├─ 인증서
  │   └─ 보안성 검토
  └─ Platform / Middleware
      ├─ Kafka
      ├─ MQ
      ├─ API Gateway
      ├─ CI/CD
      └─ Container Registry
```

## 10. 담당 부서 문의 기능

개발자는 절차만 묻는 것이 아니라 실제 부서에 문의하고 싶을 수 있다.

예시는 다음과 같다.

- 이 방화벽 신청이 가능한지 NE에 물어봐줘.
- Oracle DB 용량 산정이 맞는지 DBA에 문의하고 싶어.
- 이 구조에서 L4가 필요한지 확인해줘.
- Kafka Topic ACL 구성이 맞는지 플랫폼팀에 문의해줘.

이 경우는 신청이 아니라 문의 티켓이다. 초기에는 실제 티켓 생성이 아니라 문의 초안을 채팅으로 추천하는 수준이면 충분하다.

```text
문의 의도 감지
  ↓
담당 부서 추정
  ↓
문의 초안 생성
  ↓
필요 정보 누락 확인
  ↓
사용자가 직접 문의
```

문의 초안 예시는 다음과 같다.

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

## 11. 범위 외 문의 처리

인프라와 관련 없는 문의가 들어올 수 있다. 이를 억지로 답하면 봇 신뢰도가 떨어진다.

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

## 12. 아키텍처 기반 신청 항목 도출

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

복합 아키텍처 분석은 Qwen3.5-120B를 사용한다. 단, 결과 검증은 우선 Qwen3.6 self-critique를 사용한다.

## 13. 핵심 Structured Data

RAG chunk만으로는 한계가 있다. 최소한 다음 structured data가 필요하다.

### 13.1 부서 카탈로그

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

### 13.2 신청 유형 카탈로그

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
  "workflow_form": "방화벽 신청서",
  "draft_supported": false,
  "field_recommendation_supported": true,
  "final_submission_owner": "requester",
  "automation_priority": "low"
}
```

### 13.3 신청서 양식 카드

```json
{
  "form_name": "Redis 신규 신청서",
  "domain": "DBA",
  "request_type": "Redis 신규 신청",
  "fields": [
    {
      "name": "서비스명",
      "required": true,
      "recommendation_rule": "사용자가 말한 서비스명 또는 시스템명을 사용"
    },
    {
      "name": "환경",
      "required": true,
      "allowed_values": ["개발", "검증", "운영"]
    },
    {
      "name": "용도",
      "required": true,
      "recommendation_rule": "캐시, 세션, rate limit 등 사용 목적을 요약"
    },
    {
      "name": "메모리 용량",
      "required": true,
      "ask_if_missing": true
    }
  ],
  "final_submission_owner": "requester"
}
```

### 13.4 범위 외 라우팅 규칙

```json
{
  "topic": "노트북/PC 장애",
  "infra_bot_scope": false,
  "recommended_channel": "IT Helpdesk",
  "response_policy": "직접 처리하지 않고 담당 채널 안내"
}
```

## 14. RAG 검색 구조

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
| action_type | guide, recommend_form_fields, ask_department, draft_ticket |
| source | wiki, jira, workflow, board, incident, form |
| owner_team | DBA, Network, Cloud Platform |
| status | active, deprecated, draft |
| automation_priority | high, medium, low |

## 15. Supervisor 판단 JSON

신청서 양식 추천 예시는 다음과 같다.

```json
{
  "intent": "form_field_recommendation",
  "domain": ["DBA"],
  "request_type": "Redis 신규 신청",
  "is_in_scope": true,
  "can_recommend_fields": true,
  "can_create_draft": false,
  "can_submit_request": false,
  "final_submission_owner": "requester",
  "required_fields": [
    "서비스명",
    "환경",
    "용도",
    "용량",
    "접근 서버",
    "HA 여부"
  ],
  "filled_fields": {
    "서비스명": "pay-api",
    "환경": "운영",
    "용도": "세션 저장"
  },
  "missing_fields": [
    "용량",
    "접근 서버",
    "HA 여부"
  ],
  "next_action": "recommend_known_fields_and_ask_missing_fields"
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

## 16. 답변 포맷

### 16.1 신청서 양식 추천형

```markdown
## 신청서 기준 추천
- 신청 유형: Redis 신규 신청

## 현재 채울 수 있는 항목
- 서비스명: pay-api
- 환경: 운영
- 용도: 세션 저장

## 추가로 필요한 정보
1. 예상 메모리 용량
2. 접근 서버 IP 또는 대역
3. HA 필요 여부
4. 개인정보 저장 여부
5. 희망 완료일

## 작성 팁
- 운영 환경이면 HA 필요 여부를 반드시 확인하세요.
- 세션에 개인정보가 포함되면 보안 검토가 필요할 수 있습니다.
```

### 16.2 절차 안내형

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
```

### 16.3 부서 문의형

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
```

### 16.4 범위 외 문의형

```markdown
이 문의는 NE/SE/DBA/Cloud 인프라 신청 범위와는 직접 관련이 낮아 보입니다.

관련 채널이 확인되는 경우:
- 권장 문의 채널: IT Helpdesk / HR Helpdesk / 사내 포털

관련 채널이 확인되지 않는 경우:
- 사내 포털에서 관련 키워드로 검색하거나 Helpdesk에 문의하는 것을 권장합니다.
```

## 17. Self-Critique 체크리스트

초기 검증자는 gpt-oss-120B가 아니라 Qwen3.6 self-critique를 우선 사용한다.

체크리스트는 다음과 같다.

1. 신청서 양식의 필수 필드를 누락하지 않았는가?
2. 사용자 채팅에서 확인된 값과 추측한 값을 분리했는가?
3. 추천값이 양식 필드 의미와 맞는가?
4. 모르는 값을 임의로 채우지 않았는가?
5. 추가 질문이 필요한 필드를 명확히 제시했는가?
6. 실제 신청이 완료된 것처럼 표현하지 않았는가?
7. 최종 제출은 신청자가 직접 한다는 점을 유지했는가?
8. 담당 부서 라우팅이 필요한 경우 이를 안내했는가?
9. 범위 외 문의를 억지로 답하지 않았는가?
10. raw JSON이 사용자에게 노출되지 않았는가?

## 18. 운영 파라미터

운영/절차/보안 안내는 창의성이 필요하지 않다.

권장 생성 파라미터는 다음과 같다.

```yaml
temperature: 0.0-0.2
top_p: 0.8-0.95
```

출력은 가능한 한 구조화하되, 사용자에게 raw JSON을 그대로 노출하지 않는다.

내부 구조화 예시는 다음과 같다.

```json
{
  "intent": "form_field_recommendation|procedure_guide|department_question|architecture_analysis|out_of_scope",
  "is_in_scope": true,
  "domain": [],
  "form_name": "",
  "filled_fields": {},
  "missing_fields": [],
  "recommended_values": {},
  "needs_human_handoff": false,
  "evidence": [],
  "final_answer": ""
}
```

## 19. 우선순위

초기 구현 우선순위는 다음과 같다.

| 우선순위 | 기능 | 설명 |
|---|---|---|
| P0 | 신청서 양식 기반 채팅 추천 | 신청서 양식을 보고 각 필드에 들어갈 값을 채팅으로 추천 |
| P0 | Intent/Domain/Action 분류 | Qwen3.6 Supervisor 핵심 분기 |
| P0 | Procedure/Form RAG | Workflow 양식, Wiki, Jira, 게시판 기반 검색 |
| P0 | Qwen3.6 Self-Critique | 신청서 추천값의 누락, 부정확 항목, 근거 부족 검토 |
| P1 | 절차 안내 RAG 고도화 | 신청 위치, 승인 라인, 관련 부서 안내 |
| P1 | 범위 외 문의 처리 | 인프라 외 문의를 적절히 라우팅 |
| P1 | 아키텍처 기반 신청 항목 도출 | Qwen3.5-120B를 사용해 구성요소를 신청 항목과 부서로 변환 |
| P2 | 부서 문의 초안 작성 | 담당부서 문의 제목/본문 생성 |
| P2 | Debate/Judge 경로 | Qwen3.6 + Gemma debater, Gemma judge. 복잡한 판단에만 사용 |
| P3 | API 기반 draft 생성 | 신청서 draft를 API로 저장하거나 payload 생성 |
| P3 | Slack draft 전달 | 신청자에게 Slack으로 draft 전달 후 사용자가 직접 처리 |

API/Slack 기반 신청서 draft 전달은 매우 낮은 우선순위다.

## 20. 평가셋 구성

최소 120개 샘플을 만든다.

| 유형 | 개수 |
|---|---:|
| 신청서 양식 기반 필드 추천 | 30 |
| 단순 신청 절차 질문 | 20 |
| 복합 신청 절차 질문 | 15 |
| 부서 문의 요청 | 15 |
| 아키텍처 기반 신청 항목 도출 | 15 |
| 담당 부서/승인 라인 질문 | 10 |
| 문서 충돌/구버전 문서 판단 | 5 |
| 범위 외 문의 | 5 |
| 애매한 질문에서 되물어야 하는 케이스 | 5 |

평가 기준은 다음과 같다.

- 신청서 필드와 사용자 입력을 정확히 매핑했는가?
- 필수 필드를 누락하지 않았는가?
- 모르는 값을 임의로 채우지 않았는가?
- 추가 질문이 적절한가?
- Intent, Domain, Action을 정확히 분류했는가?
- 담당 부서 라우팅이 정확한가?
- 실제 신청 완료로 오해하게 만들지 않았는가?
- raw JSON이 노출되지 않았는가?
- 범위 외 문의를 억지로 답하지 않았는가?
- 개발자가 바로 신청서 작성에 사용할 수 있는 답변인가?

## 21. 최종 권장 아키텍처

초기 권장 아키텍처는 다음과 같다.

```text
[User Query]
   ↓
[Supervisor / Router - Qwen3.6]
   ↓
[Intent + Domain + Action Classification]
   ├─ 신청서 양식 추천
   ├─ 절차 안내
   ├─ 부서 문의
   ├─ 아키텍처 기반 신청 항목 도출
   ├─ 범위 외 문의
   └─ Human handoff 필요 여부
   ↓
[Hybrid RAG + Structured Catalog]
   ├─ Workflow 신청 양식
   ├─ Wiki
   ├─ Jira
   ├─ 내부 게시판
   ├─ 과거 문의/신청 사례
   ├─ 부서 카탈로그
   ├─ 신청 유형 카탈로그
   ├─ 신청서 양식 카드
   └─ 범위 외 라우팅 규칙
   ↓
[Form Field Recommender - Qwen3.6]
   ↓
[Self-Critique - Qwen3.6]
   ↓
[Final Chat Recommendation - Qwen3.6]
```

확장 경로는 다음과 같다.

```text
[Complex Architecture Reasoning - Qwen3.5-120B]
   ↓
[Optional Debate - Qwen3.6 + Gemma]
   ↓
[Optional Judge - Gemma]
```

## 22. 최종 추천

지금 만들려는 인프라가이드봇의 첫 구현은 다음에 집중한다.

1. Qwen3.6 Supervisor
   - 의도 분류
   - 도메인 분류
   - 범위 외 판단
   - 신청서 추천/절차 안내/부서 문의 분기

2. Qwen3.6 Form Field Recommender
   - 신청서 양식 필드와 사용자 채팅 입력 매핑
   - 각 필드에 들어갈 추천값 생성
   - 부족한 입력값 질문

3. Qwen3.6 Self-Critique
   - 내부 테스트에서 Qwen3.5보다 높은 점수를 보인 self-critique 경로
   - 추천값 누락 확인
   - 근거 부족 확인
   - 실제 신청 완료로 오해되는 표현 검토

4. Hybrid RAG + Reranker
   - Workflow 신청 양식
   - Wiki
   - Jira
   - 내부 게시판
   - 과거 문의/신청 사례
   - 부서/신청 유형/신청서 양식 카탈로그

5. Qwen3.5-120B
   - 초기 기본 경로가 아님
   - 복합 아키텍처 분석이나 여러 부서 신청 항목 도출에만 확장 사용

6. Gemma
   - Debate judge 후보
   - Qwen3.6 + Gemma debater pair가 필요한 복잡한 판단에만 선택 사용

7. gpt-oss-120B
   - raw JSON 누출률이 높으므로 초기 Critic에서 제외
   - 출력 안정화가 확인된 뒤 제한적으로 재평가

8. API/Slack Draft Delivery
   - 낮은 우선순위
   - 신청서 draft를 API로 만들고 Slack으로 신청자에게 전달
   - 최종 제출은 신청자가 직접 수행
   - 거의 완료 직전 또는 운영 안정화 이후 진행

최종 결론은 다음과 같다.

> 인프라가이드봇은 처음부터 신청 자동화나 복잡한 멀티에이전트 Debate를 구현할 필요가 없다. 첫 구현은 신청서 양식을 읽고, 채팅으로 들어온 사용자 정보를 각 필드에 맞게 추천해주는 기능이면 충분하다. 내부 테스트 결과를 반영하면 Qwen3.6-32B를 Supervisor, Form Field Recommender, Self-Critique의 기본 모델로 두고, Qwen3.5-120B는 복합 아키텍처 분석에만 확장 사용하며, Gemma는 Debate Judge 후보로 유지하는 것이 안정적이다. gpt-oss-120B Critic은 raw JSON 누출 문제가 해결되기 전까지 초기 경로에서 제외한다.
