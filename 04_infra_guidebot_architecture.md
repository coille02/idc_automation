# 오픈소스 LLM 기반 인프라가이드봇 설계

작성일: 2026-05-14

## 1. 목표

이 문서는 사내 개발자를 위한 **인프라가이드봇** 설계 방향을 정리한다.

인프라가이드봇의 목적은 다음과 같다.

- 개발자가 NE, SE, DBA, Cloud 인프라 신청 절차를 쉽게 찾도록 돕는다.
- Jira, Wiki, Workflow, 내부 게시판, 과거 신청 사례를 RAG로 검색해 근거 기반 답변을 제공한다.
- 사용자가 아키텍처를 설명하면 필요한 신청 항목, 담당 조직, 승인 절차, 사전 확인 사항을 정리한다.
- Claude Sonnet/Opus 수준의 답변 품질에 최대한 근접하도록 오픈소스 LLM 조합과 Agent 구조를 설계한다.

## 2. 핵심 결론

인프라가이드봇에는 단순히 여러 모델을 섞어서 답변하게 하는 방식보다, **하나의 주 모델 + 역할별 보조 모델 + 검증 단계**가 적합하다.

권장 기본 구조는 다음과 같다.

```text
Supervisor + RAG + Procedure Card + Critic
```

권장 모델 역할은 다음과 같다.

| 역할 | 추천 모델 | 설명 |
|---|---|---|
| 메인 답변/추론 | Qwen3.5-120B | 최종 답변, 복잡한 아키텍처 해석, 신청 항목 도출 |
| 라우팅/질의 재작성/요약 | Qwen 3.6-32B | 질문 분류, 검색 질의 생성, 짧은 구조화 작업 |
| 검증/누락 확인 | gpt-oss-120B 또는 Qwen3.5-120B | 최종 답변의 근거 부족, 신청 항목 누락, 충돌 확인 |
| 보조 요약/fallback | Gemma4 | 낮은 중요도 문서 요약, 간단 질의 보조 |

중요한 원칙은 다음과 같다.

> 여러 모델이 각자 답변을 만들고 그것을 섞는 방식은 피한다. 하나의 메인 모델이 최종 답변을 만들고, 다른 모델은 라우팅, 검색 보조, 검증, 누락 확인에만 사용한다.

## 3. Supervisor, Reflection, Debate의 차이

| 패턴 | 핵심 구조 | 적합한 상황 | 주의점 |
|---|---|---|---|
| Supervisor | 중앙 관리자 Agent가 전문 Agent와 Tool을 호출 | 업무가 여러 도메인으로 나뉘는 경우 | 라우팅이 틀리면 전체 답변이 흔들림 |
| Reflection / Reflexion | 초안 생성 후 Critic이 검토하고 수정 반복 | 절차 안내, 코드/YAML 생성, 장애 분석 보고서 | 같은 모델이 자기 오류를 못 잡을 수 있음 |
| Debate | 여러 Agent가 가설을 주장하고 Judge가 판단 | 원인 후보가 여러 개이거나 trade-off가 큰 판단 | 비용과 지연이 크고 Judge가 약하면 효과가 낮음 |

인프라가이드봇의 기본값은 **Supervisor + RAG + Reflection**이 좋다.

Debate는 기본 구조로 넣기보다 다음 경우에만 선택적으로 사용한다.

- 아키텍처 대안 비교
- NE, SE, DBA, Cloud 관점이 충돌하는 경우
- 보안 정책과 개발 편의성이 충돌하는 경우
- 비용, 성능, 안정성 사이의 trade-off 판단이 필요한 경우
- 장애 원인 후보가 여러 개인 경우

## 4. 단순 모델 혼합이 위험한 이유

Qwen, Gemma, gpt-oss 같은 모델을 동시에 호출해 답변을 섞으면 품질이 좋아질 것처럼 보일 수 있지만, 사내 절차 안내 업무에서는 오히려 위험할 수 있다.

주요 이유는 다음과 같다.

- 모델마다 근거 문서를 선택하는 기준이 다르다.
- 문서가 충돌할 때 어떤 문서를 우선해야 하는지 불명확해진다.
- 최종 합성 단계에서 그럴듯하지만 실제 절차와 다른 답변이 만들어질 수 있다.
- 신청 절차, 승인 라인, 보안 검토처럼 정확성이 중요한 영역에서는 "좋아 보이는 답변"보다 "근거 있는 답변"이 중요하다.

따라서 여러 모델을 사용하더라도 역할을 명확히 분리해야 한다.

## 5. 권장 모델 조합

### 5.1 안정성 우선 구성

```text
Supervisor / Router       : Qwen 3.6-32B
Query Rewriter            : Qwen 3.6-32B
RAG Answer                : Qwen3.5-120B
Architecture Reasoning    : Qwen3.5-120B
Critic / Verifier         : gpt-oss-120B 또는 Qwen3.5-120B
Final Writer              : Qwen3.5-120B
```

이 구성은 가장 무난하다.

Qwen 32B급 모델은 짧고 구조적인 작업에 쓰고, Qwen3.5-120B는 최종 추론과 답변 생성에 집중시킨다. gpt-oss-120B는 메인 답변 모델보다 검증자 역할로 제한하는 편이 안전하다.

### 5.2 비용/속도 우선 구성

```text
Supervisor / Router       : Qwen 3.6-32B
Simple Procedure Q&A      : Qwen 3.6-32B
Complex Architecture Q&A  : Qwen3.5-120B
Critic / Verifier         : Qwen 3.6-32B 또는 gpt-oss-120B
Final Writer              : Qwen3.5-120B
```

예를 들어 "Redis 신청은 어디서 해요?" 같은 질문은 Qwen 32B와 정확한 RAG만으로도 충분하다.

반대로 다음과 같은 질문은 120B급 모델이 필요하다.

> 신규 서비스 아키텍처가 WAS 4대, Redis, Kafka, Oracle DB, 외부망 연동이 필요한데 신청해야 할 항목을 정리해줘.

## 6. gpt-oss-120B 사용 전략

gpt-oss-120B가 RAG와 섞였을 때 답변 품질이 불안정했다면, 메인 답변 생성에는 사용하지 않는 것이 좋다.

적합한 용도는 다음과 같다.

- 최종 답변 검토
- 누락된 신청 항목 찾기
- 답변이 근거 문서에 기반하는지 확인
- 아키텍처 리스크 찾기
- 여러 신청 절차 간 충돌 확인

피해야 할 용도는 다음과 같다.

- 최종 답변 메인 생성
- 긴 RAG 문서 여러 개를 한 번에 넣고 답변 생성
- 한국어 사내 절차를 자연스럽게 설명하는 역할
- 여러 모델 답변을 합치는 Judge 역할

## 7. RAG 문서는 Procedure Card로 구조화한다

Claude Sonnet/Opus와 오픈소스 LLM의 답변 품질 차이를 줄이려면, RAG 문서를 단순 chunk로만 저장하지 말고 신청 절차 단위의 **Procedure Card**로 구조화해야 한다.

예시는 다음과 같다.

```json
{
  "service_name": "Redis 신청",
  "category": "DBA",
  "request_type": "신규 발급",
  "target_user": "개발자",
  "when_to_use": [
    "서비스에서 캐시가 필요한 경우",
    "세션 저장소가 필요한 경우"
  ],
  "required_inputs": [
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
  "approval_line": [
    "개발팀 리더",
    "DBA 담당자",
    "보안 검토"
  ],
  "related_teams": [
    "DBA",
    "SE",
    "보안"
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
  "last_verified_at": "2026-05-14"
}
```

이렇게 구조화하면 모델은 자연어 문서에서 절차를 추측하지 않고, 정리된 필드를 바탕으로 안정적으로 답변할 수 있다.

## 8. RAG 검색 구조

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
| domain | NE, SE, DBA, Cloud, Security |
| environment | dev, stg, prod |
| request_type | 신규, 변경, 삭제, 권한, 장애, 예외 |
| source | wiki, jira, workflow, board, incident |
| last_verified_at | 2026-05-14 |
| owner_team | DBA, Network, Cloud Platform |
| status | active, deprecated, draft |

검색 결과는 최신성, 문서 출처, 업무 도메인, 신청 유형을 기준으로 재정렬해야 한다.

## 9. Agent 설계

권장 Agent 구성은 다음과 같다.

| Agent | 역할 | 추천 모델 |
|---|---|---|
| Intent Classifier | 질문 유형 분류 | Qwen 3.6-32B |
| Query Rewriter | 검색 질의 재작성 | Qwen 3.6-32B |
| Procedure Retriever | 관련 신청 절차 검색 | 검색엔진 + Reranker |
| Architecture Parser | 아키텍처 구성요소 추출 | Qwen3.5-120B |
| Requirement Extractor | 필요한 신청 항목 도출 | Qwen3.5-120B |
| Domain Agent | NE/SE/DBA/Cloud/Security 관점 검토 | Qwen3.5-120B |
| Checklist Builder | 신청 체크리스트 구성 | Qwen3.5-120B |
| Critic / Verifier | 누락, 충돌, 근거 부족 검토 | gpt-oss-120B 또는 Qwen3.5-120B |
| Final Answer Generator | 최종 답변 생성 | Qwen3.5-120B |

전체 흐름은 다음과 같다.

```text
[User]
  ↓
[Qwen 32B - Intent Classifier]
  ↓
[Qwen 32B - Query Rewriter]
  ↓
[Hybrid RAG]
  ├─ Wiki
  ├─ Jira
  ├─ Workflow 신청 양식
  ├─ 내부 게시판
  ├─ 과거 신청 사례
  └─ 표준 아키텍처 문서
  ↓
[Reranker]
  ↓
[Qwen3.5-120B - Reasoning / Answer Draft]
  ↓
[gpt-oss-120B or Qwen3.5-120B - Critic]
  ↓
[Qwen3.5-120B - Final Answer]
```

## 10. 질문 유형별 파이프라인

### 10.1 단순 절차 질문

예시:

> Redis 신청은 어디서 해요?

흐름:

```text
Intent Classifier
  ↓
Procedure Retriever
  ↓
Procedure Card
  ↓
Qwen 32B 또는 Qwen3.5-120B 답변
  ↓
근거 문서 표시
```

### 10.2 신청서 작성 질문

예시:

> Kafka topic 신청서에 뭘 적어야 해요?

흐름:

```text
Intent Classifier
  ↓
Workflow Form Retriever
  ↓
Required Fields Extractor
  ↓
Example Builder
  ↓
Critic
  ↓
Final Answer
```

### 10.3 아키텍처 기반 신청 항목 도출

예시:

> API 서버 3대, Redis, Kafka, Oracle DB, 외부 API 연동이 있는 서비스를 만들려고 하는데 인프라 신청 뭐 해야 해?

먼저 사용자 입력을 구조화한다.

```json
{
  "components": [
    "API server",
    "Redis",
    "Kafka",
    "Oracle DB",
    "External API integration"
  ],
  "environment": "unknown",
  "network_zone": "unknown",
  "data_sensitivity": "unknown",
  "ha_required": "unknown",
  "expected_traffic": "unknown",
  "required_requests": [
    "server/vm/container resource request",
    "Redis request",
    "Kafka topic/account request",
    "DB schema/account request",
    "firewall request",
    "DNS request",
    "L4/L7 request",
    "monitoring registration",
    "logging registration",
    "backup request",
    "security review"
  ],
  "missing_questions": [
    "운영/개발 환경인가?",
    "개인정보를 저장하는가?",
    "외부망 연동이 필요한가?",
    "예상 트래픽은 얼마인가?",
    "HA 구성이 필요한가?"
  ]
}
```

그 다음 각 구성요소별로 RAG를 호출한다.

```text
Redis 신청 절차 검색
Kafka topic 신청 절차 검색
DB 계정/스키마 신청 절차 검색
방화벽 신청 절차 검색
DNS 신청 절차 검색
L4/L7 또는 Ingress 신청 절차 검색
모니터링 등록 절차 검색
로그 수집 신청 절차 검색
보안 검토 절차 검색
```

최종 답변은 필요한 신청 항목과 조건부 항목을 분리해서 제공한다.

## 11. 답변 포맷

### 11.1 절차 안내형

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
- NE / SE / DBA / Cloud / 보안

## 주의사항
- 방화벽 필요 여부
- 운영 환경 승인 여부
- 개인정보/보안 검토 여부

## 근거
- 참조 문서
```

### 11.2 아키텍처 검토형

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

## 12. Critic 검증 체크리스트

Critic Agent는 추상적으로 "검토해줘"가 아니라 명확한 체크리스트를 받아야 한다.

권장 체크리스트는 다음과 같다.

1. 답변이 제공된 문서, Workflow, Jira, Wiki 근거에 기반하는가?
2. 확인된 사실과 추측을 분리했는가?
3. 필요한 신청 항목을 누락하지 않았는가?
4. 불필요하거나 잘못된 신청 항목을 추가하지 않았는가?
5. 최신 문서를 우선했는가?
6. 구버전 문서나 deprecated 절차를 사용하지 않았는가?
7. 담당 부서와 승인 라인이 정확한가?
8. 운영/개발 환경, 보안, 개인정보, 외부망 연동 조건을 확인했는가?
9. 사용자에게 추가 확인 질문이 필요한 경우 단정하지 않았는가?
10. 개발자가 바로 행동할 수 있는 수준으로 답변했는가?

## 13. 운영 파라미터

운영/절차/보안 안내는 창의성이 필요하지 않다.

권장 생성 파라미터는 다음과 같다.

```yaml
temperature: 0.0-0.2
top_p: 0.8-0.95
```

출력은 가능한 한 구조화한다.

```json
{
  "confirmed_facts": [],
  "required_requests": [],
  "conditional_requests": [],
  "missing_questions": [],
  "related_teams": [],
  "evidence": [],
  "risk_level": "low|medium|high|critical",
  "final_answer": ""
}
```

## 14. 평가셋 구성

Claude와 비교하려면 감각적인 평가보다 내부 평가셋이 필요하다.

최소 100개 샘플을 만든다.

| 유형 | 개수 |
|---|---:|
| 단순 신청 절차 질문 | 30 |
| 복합 신청 절차 질문 | 20 |
| 아키텍처 기반 신청 항목 도출 | 20 |
| 담당 부서/승인 라인 질문 | 10 |
| 문서 충돌/구버전 문서 판단 | 10 |
| 애매한 질문에서 되물어야 하는 케이스 | 10 |

평가 기준은 다음과 같다.

- 필요한 신청 항목을 누락하지 않았는가?
- 잘못된 신청 절차를 안내하지 않았는가?
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
| C안 | Qwen3.5-120B + gpt-oss Critic |
| D안 | Qwen 32B 단순 Q&A + Qwen3.5-120B 복잡 Q&A |
| E안 | gpt-oss-120B 단독 |

예상 우선순위는 다음과 같다.

1. Qwen3.5-120B + Qwen 32B Router + Critic
2. Qwen3.5-120B 단독 + 좋은 RAG
3. Qwen 32B + 좋은 RAG
4. 여러 모델 답변 단순 혼합

## 15. 최종 권장안

최종 권장 아키텍처는 다음과 같다.

```text
[User Query]
   ↓
[Supervisor / Router - Qwen 32B]
   ↓
[Intent Classification]
   ├─ 단순 절차 문의
   ├─ 신청서 작성 문의
   ├─ 아키텍처 기반 신청 항목 도출
   ├─ 정책/보안 기준 문의
   └─ 담당 부서/승인 라인 문의
   ↓
[Hybrid RAG + Reranker]
   ├─ Wiki
   ├─ Jira
   ├─ Workflow 신청 양식
   ├─ 내부 게시판
   ├─ 과거 신청 사례
   └─ 표준 아키텍처 문서
   ↓
[Reasoning / Answer Draft - Qwen3.5-120B]
   ↓
[Critic / Verifier - gpt-oss-120B or Qwen3.5-120B]
   ↓
[Final Answer - Qwen3.5-120B]
```

한 줄로 정리하면 다음과 같다.

> Claude를 오픈소스 모델로 따라가려면 더 많은 모델을 섞는 것보다, Qwen3.5-120B를 메인으로 두고 RAG 문서를 Procedure Card로 구조화한 뒤, Qwen 32B Router와 Critic 검증 단계를 붙이는 쪽이 훨씬 안정적이다.
