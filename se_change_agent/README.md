# SE Change Agent 아키텍처 설계 문서

검토 반영 버전: 2026-05-15

본 문서는 SE(System Engineering)팀 관점에서 서버 운영 변경 작업을 안전하게 표준화하고, AI Agent와 Jenkins/Git 기반 프로세스를 통해 L2/L3 수준의 변경 작업 지원 및 자동화를 구현하기 위한 아키텍처 설계안이다.

본 설계의 핵심은 AI Agent가 운영 서버에서 임의 명령을 실행하는 구조가 아니다. Agent는 요청 분석, Runbook 매칭, 실행 계획 생성, Git PR 생성, 로그 분석, 수정안 제안, 후속 작업 생성을 담당하는 오케스트레이터이며, 운영 서버 접근과 실제 실행은 Jenkins와 작업자 승인 체계 안에서만 수행한다.

---

## 1. 설계 목표

본 설계는 다음 문제를 해결하는 것을 목표로 한다.

- SE팀 반복 서버 변경 작업의 표준화
- 작업 전 위험도 분석 및 누락 항목 검출
- 승인된 스크립트의 Git 기반 이력 관리
- Jenkins를 통한 스크립트 배포 및 실행 통제
- L2/L3 실행 모드 분리
- 실패 시 로그 수집, 원인 분석, 후속 PR 생성
- Runbook 기반의 점진적 자동화 확대
- 승인, 실행, 결과, 예외 처리의 감사 가능성 확보

---

## 2. 설계 원칙

### 2.1 운영 서버 직접 실행 금지

Agent는 운영 서버에 직접 접속하지 않는다.

```text
Agent   → 요청 분석 / Runbook 매칭 / PR 생성 / 결과 분석
GitHub  → 변경 계획, 스크립트, 정책, 결과 요약의 기준 저장소
Jenkins → 승인된 commit 기준으로 배포, 실행, 결과 수집
작업자  → PR 리뷰, 승인, 수동 실행, 최종 판단
```

### 2.2 Runbook 기반 실행

L3 자동 실행은 승인된 Runbook Registry에 존재하는 작업 유형에 대해서만 허용한다.

```text
Runbook 있음       → L2/L3 가능
Runbook 없음       → L3 금지
Runbook 없음 반복  → 신규 Runbook 후보 등록
```

Runbook이 없는 작업은 Review Only 또는 L2 Flexible로 처리한다.

### 2.3 Git 기반 승인과 감사

모든 스크립트, 실행 계획, 정책, 결과 요약은 Git에서 관리한다.

- 작업별 `change.yaml` 생성
- `precheck.sh`, `execute.sh`, `postcheck.sh`, `rollback.sh`, `run.sh` PR 생성
- PR review/merge 후 Jenkins 실행
- 결과 summary와 script diff는 Git에 기록
- 원본 로그는 Jenkins artifact 또는 Object Storage에 저장

### 2.4 L2와 L3의 단계적 분리

```text
L2 = Jenkins Script Delivery
     PR merge 후 Jenkins가 대상 서버에 승인된 스크립트를 배포한다.
     실제 실행은 작업자가 서버에서 직접 수행한다.

L3 = Jenkins Script Execution
     PR merge 후 Jenkins가 대상 서버에 스크립트를 배포하고,
     precheck → execute → postcheck까지 자동 실행한다.
```

### 2.5 실패를 정상 시나리오로 간주

L2/L3 모두 실패할 수 있다. 따라서 성공 경로뿐 아니라 실패 시 중단, 진단 정보 수집, rollback 판단, follow-up PR 생성, 수동 전환 경로까지 설계에 포함한다.

---

## 3. 적용 범위

### 3.1 포함 범위

1차 범위는 SE팀이 직접 소유하고 운영하는 서버 변경 작업이다.

- NGINX/Apache 설정 변경 및 reload
- Consul 설정 변경
- Consul + NGINX 연계 설정 변경
- systemd 서비스 restart/reload
- 인증서 교체 후 reload
- 로그 수집기 설정 변경
- 특정 OS 패키지 업데이트
- Ubuntu 20.04 OpenSSL 업데이트와 같은 단일 OS/단일 패키지 작업

### 3.2 제외 범위

초기 범위에서 제외하는 작업은 다음과 같다.

- DB 설정 변경 및 DB 재시작
- Network 장비 변경
- 방화벽/라우팅/LB 장비 직접 변경
- Cloud IAM/VPC/Subnet 변경
- Ceph OSD/CRUSH/pool 정책 변경
- fstab/mount 변경
- OS major upgrade
- rollback이 불명확한 작업
- 여러 팀 승인이 필수인 복합 변경

제외 범위에 해당하더라도 Agent는 Review Only 수준의 위험도 분석과 체크리스트 제공은 가능하다.

---

## 4. 전체 아키텍처

```text
[Workflow System]
      ↓
[Change Intake Agent]
      ↓
[Change Type Classifier]
      ↓
[Runbook Registry]
      ↓
[Server Profile Collector]
      ↓
[Risk & Execution Mode Decider]
      ↓
[Plan / Script Generator]
      ↓
[GitHub PR]
      ↓
[GitHub Protection / Required Checks]
      ↓
[Jenkins]
  ├─ L2 Script Delivery
  ├─ L3 Script Execution
  ├─ Lock / Lease Control
  └─ Result Collect
      ↓
[Result Storage]
  ├─ Jenkins Artifact / Object Storage
  └─ GitHub Summary
      ↓
[Agent WebView / Chat]
  ├─ 실시간 로그 확인
  ├─ 실패 원인 분석
  ├─ 다음 조치 추천
  └─ Follow-up PR 생성
```

---

## 5. 주요 구성요소

### 5.1 Workflow System

기존 사내 Workflow 시스템은 승인 source of truth로 사용한다.

역할:

- 작업 요청서 저장
- 승인 상태 확인
- 요청자/승인자/작업 대상/작업 시간 확인
- 기존 1~3년 작업 이력 분석 소스
- change window 충돌 검증의 기준 데이터 제공

Jenkins 운영 실행 조건에는 Workflow 승인 완료 상태가 포함되어야 한다.

```text
Workflow 승인 미완료 → L2/L3 진행 불가
```

### 5.2 GitHub Repository

GitHub는 변경 스크립트, Runbook, 정책, 결과 요약의 기준 저장소이다.

```text
se-change-agent/
├── runbooks/
│   ├── nginx_config_change/
│   ├── consul_config_change/
│   ├── consul_nginx_config_change/
│   ├── cert_replace/
│   └── ubuntu2004_openssl_update/
│
├── changes/
│   └── CHG-20260520-001/
│       ├── change.yaml
│       ├── render-manifest.json
│       ├── precheck.sh
│       ├── execute.sh
│       ├── postcheck.sh
│       ├── rollback.sh
│       └── README.md
│
├── results/
│   └── CHG-20260520-001/
│       ├── summary.md
│       ├── result.json
│       └── script-diff.patch
│
├── policies/
│   ├── execution-level-policy.yaml
│   ├── failure-policy.yaml
│   ├── approval-policy.yaml
│   ├── command-allowlist.yaml
│   ├── lock-policy.yaml
│   ├── log-retention-policy.yaml
│   └── promotion-policy.yaml
│
├── .github/
│   ├── CODEOWNERS
│   └── workflows/
│       ├── validate-change.yaml
│       └── policy-check.yaml
│
└── tools/
    ├── render_runbook.py
    ├── validate_change.py
    ├── collect_result.py
    └── policy_check.py
```

### 5.3 Jenkins

Jenkins는 실행 엔진이다.

역할:

- 서버 profile 수집
- Validation Server/VM 테스트
- L2 스크립트 배포
- L3 precheck/execute/postcheck 실행
- 실패 시 진단 정보 수집
- L2/L3 결과 수집
- 원본 로그 artifact 저장
- GitHub/Jira/Workflow 결과 업데이트
- host/service 단위 execution lock 관리

Jenkins는 반드시 `change.yaml`과 승인된 Git commit을 기준으로 실행해야 하며, 임의 parameter 실행을 제한해야 한다.

### 5.4 Agent

Agent는 다음 역할로 분리한다.

#### Change Intake Agent

Workflow 요청서를 읽고 구조화한다.

```yaml
change_request:
  change_id:
  workflow_id:
  requester:
  approver:
  requested_change:
  target_hosts:
  maintenance_window:
  approval_status:
```

#### Change Type Classifier

요청 작업을 Runbook Registry의 작업 유형과 매칭한다.

```yaml
classification:
  runbook_id: ubuntu2004_openssl_update
  confidence: 0.93
  matched: true
```

confidence가 낮거나 Runbook이 없으면 L3 금지.

#### Planner Agent

Runbook, 대상 서버 profile, 정책을 기반으로 실행 계획을 생성한다.

- L2/L3 추천
- target group 분리
- canary/batch 전략 생성
- lock 대상 산정
- change.yaml 생성
- PR 설명 생성

#### Script Generator Agent

Runbook template 기반으로 스크립트를 렌더링한다.

```text
LLM이 임의 shell script를 자유 생성하지 않는다.
Runbook template + variables 기반으로 생성한다.
```

#### Result Analysis Agent

Jenkins/L2 결과 로그를 분석한다.

- 실패 단계 식별
- 실패 원인 요약
- rollback 필요 여부 판단 보조
- 수동 확인 명령어 제안
- follow-up PR 생성
- Runbook 개선 후보 제안

### 5.5 Agent WebView / Chat

작업자는 WebView에서 변경 작업 상태와 로그를 확인한다.

- Change ID별 상태 확인
- Runbook / execution level 표시
- 서버별 진행 상태 표시
- Jenkins stage 상태 표시
- 실시간 로그 tail
- Agent 실패 원인 요약
- rollback / retry / abort / follow-up PR 버튼

---

## 6. Runbook Registry 설계

### 6.1 Runbook as Code

Runbook은 위키 문서가 아니라 Git에서 코드처럼 관리한다.

각 Runbook은 다음 3개를 포함한다.

```text
1. Human Runbook
   - 사람이 이해하는 절차 문서

2. Machine Runbook
   - Agent/Jenkins가 해석 가능한 runbook.yaml

3. Script Templates
   - precheck/execute/postcheck/rollback/run.sh 템플릿
```

### 6.2 Runbook 디렉터리 구조

```text
runbooks/ubuntu2004_openssl_update/
├── runbook.yaml
├── README.md
├── templates/
│   ├── collect_profile.sh.j2
│   ├── precheck.sh.j2
│   ├── execute.sh.j2
│   ├── postcheck.sh.j2
│   ├── rollback.sh.j2
│   └── run.sh.j2
├── policies/
│   ├── risk.yaml
│   ├── validation.yaml
│   ├── execution.yaml
│   ├── allowed-commands.yaml
│   └── lock.yaml
└── examples/
    └── sample-change.yaml
```

### 6.3 runbook.yaml 필수 항목

```yaml
id: ubuntu2004_openssl_update
name: Ubuntu 20.04 OpenSSL Package Update
owner_team: se
maturity: L3 Canary

supported_os_profiles:
  - ubuntu2004

supported_execution_levels:
  - L2
  - L3

default_execution_level: L3
change_category: os_package_update

required_inputs:
  - change_id
  - workflow_id
  - target_hosts
  - package_names
  - maintenance_window
  - approver

precheck:
  required: true
  checks:
    - os_version_check
    - apt_lock_check
    - package_current_version
    - package_candidate_version
    - disk_space_check

execute:
  strategy: staged
  commands:
    - apt_update
    - apt_install_only_upgrade

postcheck:
  required: true
  checks:
    - package_version_after
    - openssl_version_check
    - failed_systemd_units_check
    - reboot_required_check
    - deleted_library_process_check

rollback:
  supported: conditional
  auto_rollback: false
  requires:
    - previous_package_version_available
    - explicit_human_approval

execution_policy:
  l3:
    require_workflow_approval: true
    require_pr_approval: true
    require_canary: true
    require_human_approval_after_canary: true
    stop_on_first_failure: true
    auto_continue: false
    auto_rollback: false

  l2:
    script_delivery_only: true
    result_collection_required: true
    local_modification_allowed: true
    local_modification_must_be_collected: true
```

### 6.4 Runbook Maturity

```text
Draft
  ↓
L2 Flexible
  ↓
L2 Strict
  ↓
L3 Canary
  ↓
L3 Staged
  ↓
L3 Standard
```

초기 Runbook은 L3로 바로 승격하지 않는다.

---

## 7. L2/L3 실행 모델

### 7.1 L2 Script Delivery

```text
Workflow 승인
  ↓
Agent 분석 / Runbook 매칭
  ↓
PR 생성
  ↓
PR review / merge
  ↓
Jenkins Script Delivery
  ↓
작업자 run.sh 수동 실행
  ↓
result.json / logs 생성
  ↓
Jenkins collect-result
  ↓
Agent 결과 분석
  ↓
GitHub/Jira/Workflow 결과 기록
```

L2 서버 배포 구조:

```text
/opt/se-change/CHG-001/
├── change.yaml
├── render-manifest.json
├── run.sh
├── precheck.sh
├── execute.sh
├── postcheck.sh
├── rollback.sh
├── CHECKSUMS
└── result/
    ├── result.json
    ├── precheck.log
    ├── execute.log
    ├── postcheck.log
    ├── rollback.log
    └── script-diff.patch
```

작업자는 개별 스크립트를 직접 실행하지 않고 wrapper를 사용한다.

```bash
sudo ./run.sh precheck
sudo ./run.sh execute
sudo ./run.sh postcheck
sudo ./run.sh rollback
```

### 7.2 L2 Flexible vs L2 Strict

```text
L2 Flexible
- 현장 수정 허용
- 수정 diff/result 회수 필수
- 초기 도입/비정형 작업용
- 승인된 실행이 아니라 현장 예외 실행으로 표시
- 반복 발생 시 반드시 follow-up PR로 Runbook/template에 반영

L2 Strict
- 승인된 스크립트만 실행 가능
- checksum 불일치 시 실행 차단
- 반복/정형 작업용
```

### 7.3 L3 Script Execution

```text
Workflow 승인
  ↓
Agent 분석 / Runbook 매칭
  ↓
서버 profile 수집
  ↓
Validation 필요 시 수행
  ↓
PR 생성
  ↓
PR review / merge
  ↓
Jenkins lock 획득
  ↓
Jenkins precheck
  ↓
Jenkins execute
  ↓
Jenkins postcheck
  ↓
결과 수집
  ↓
Agent 결과 분석
  ↓
Workflow/Jira/Git 결과 기록
  ↓
Jenkins lock 해제
```

L3 필수 조건:

- 승인된 Runbook 존재
- Workflow 승인 완료
- PR review/merge 완료
- branch protection 통과
- required status check 통과
- rollback 정책 존재
- postcheck 존재
- 대상 서버 profile 일치
- forbidden command 없음
- host/service lock 획득
- precheck 성공
- 실패 시 stop-on-first-failure 가능

---

## 8. GitHub Governance

### 8.1 Branch Protection

`main` 브랜치는 보호되어야 한다.

필수 정책:

- PR 없이 direct push 금지
- required approving review 설정
- required status checks 설정
- stale review dismissal 활성화
- force push 금지
- delete branch 금지
- admin bypass 최소화

필수 status check 예:

```text
validate-change
policy-check
render-integrity-check
command-allowlist-check
shellcheck
CODEOWNERS-review
```

### 8.2 CODEOWNERS

중요 경로는 책임 리뷰어를 지정한다.

```text
/runbooks/                 @se-runbook-owners
/policies/                 @se-platform-owners @security-owners
/tools/                    @se-platform-owners
/.github/workflows/        @se-platform-owners @security-owners
/changes/**/execute.sh     @se-runbook-owners
/changes/**/rollback.sh    @se-runbook-owners
```

운영 정책과 실행 템플릿은 일반 문서보다 높은 리뷰 기준을 둔다.

### 8.3 PR Review Rule

- Runbook 변경: owner team 승인 필수
- 정책 변경: platform owner + security owner 승인 필수
- L3 change PR: runbook owner 승인 필수
- high risk change: 2인 이상 승인 권장
- emergency change: 사후 review와 post-implementation review 필수

---

## 9. Jenkins Security & RBAC

Jenkins는 실제 실행 권한을 가진 구성요소이므로 권한 모델을 별도로 둔다.

### 9.1 권한 분리

```text
Agent 권한
- PR 생성
- Jenkins build 요청
- Jenkins 결과 조회
- 운영 credential 직접 접근 금지

Jenkins 권한
- 승인된 commit checkout
- 대상 서버 배포/실행
- 결과 artifact 저장
- Workflow/Jira 상태 업데이트

작업자 권한
- PR review/approval
- L2 수동 실행
- L3 canary 이후 진행 승인
- rollback/retry/abort 판단
```

### 9.2 Jenkins Credential 원칙

- credential은 runbook/job 단위로 최소 권한 부여
- 공용 root credential 사용 금지
- credential scope를 folder/job 단위로 제한
- secret은 console log에 출력되지 않도록 masking
- credential file은 workspace에 장기 보관 금지
- Jenkins agent에 다중 executor가 있는 경우 secret file 노출 가능성을 고려해 격리 실행 권장

### 9.3 sudo 정책

L3에서 Jenkins가 사용할 sudo 권한은 command allowlist 기반으로 제한한다.

```text
허용:
- runbook별 wrapper 실행
- 사전 정의된 service reload
- 사전 정의된 package update 명령

금지:
- 임의 shell
- interactive shell
- wildcard 기반 위험 명령
- 승인되지 않은 systemctl restart
- reboot/shutdown 직접 실행
```

### 9.4 Jenkins Parameter 제한

Jenkins job은 임의 parameter로 운영 명령을 구성하지 않는다.

허용 parameter:

```text
change_id
approved_commit
execution_level
batch_id
manual_approval_token
```

금지 parameter:

```text
raw_command
script_body
target_host_override
sudo_command
package_name_override
```

---

## 10. Execution Lock / Change Conflict Policy

동일 서버 또는 동일 서비스에 대해 변경 작업이 동시에 실행되면 사고 가능성이 높아진다. 따라서 Jenkins 실행 전 lock을 획득해야 한다.

### 10.1 Lock 단위

```text
host lock      : 동일 서버 중복 변경 방지
service lock   : 동일 서비스 reload/restart 충돌 방지
runbook lock   : 특정 위험 Runbook의 동시 실행 제한
window lock    : maintenance window 충돌 방지
```

### 10.2 Lock 정책 예

```yaml
lock_policy:
  default:
    require_host_lock: true
    require_service_lock: false
    fail_if_locked: true

  nginx_config_change:
    require_host_lock: true
    require_service_lock: true
    service_lock_name: nginx

  ubuntu2004_openssl_update:
    require_host_lock: true
    require_service_lock: false
    max_parallel_hosts: 10
```

### 10.3 Jenkins 구현 방향

- pipeline 단위로 `disableConcurrentBuilds()` 적용
- host/service 단위는 Lockable Resources plugin 또는 별도 lock service 사용
- lock 획득 실패 시 실행하지 않고 대기 또는 실패 처리
- lock owner, acquired_at, expires_at, change_id를 기록
- Jenkins 장애 시 stale lock 해제 절차를 정의

---

## 11. Render Provenance & Script Integrity

L3에서는 PR에 올라간 스크립트가 Runbook template에서 나온 것인지 검증해야 한다.

단순 비교뿐 아니라 렌더링 provenance를 남긴다.

### 11.1 render-manifest.json

```json
{
  "change_id": "CHG-20260520-001",
  "runbook_id": "ubuntu2004_openssl_update",
  "runbook_version": "v1.3.0",
  "template_commit": "abc123def",
  "renderer_version": "0.4.2",
  "policy_version": "2026-05-15",
  "input_hash": "sha256:...",
  "rendered_files": {
    "precheck.sh": "sha256:...",
    "execute.sh": "sha256:...",
    "postcheck.sh": "sha256:...",
    "rollback.sh": "sha256:...",
    "run.sh": "sha256:..."
  }
}
```

### 11.2 Integrity Gate

L3 실행 전 Jenkins는 다음을 검증한다.

- 현재 checkout commit이 PR 승인 commit과 일치
- `render-manifest.json` 존재
- script checksum이 manifest와 일치
- 현재 template으로 재렌더링한 결과와 script가 일치
- command allowlist 위반 없음
- forbidden command 없음

불일치 시 L3를 차단한다.

---

## 12. Server Profile 및 OS Profile 설계

### 12.1 운영 OS 현황

```text
- CentOS 7.9
- Rocky 8.10
- Rocky 9.x
- Ubuntu 20.04
- Ubuntu 22.04
- Ubuntu 24.04
```

### 12.2 OS Profile

OS는 minor 버전별로 모두 복제하지 않고, OS family/major 기준으로 profile을 정의한다.

```text
validation-profiles/
├── centos79
├── rocky8
├── rocky9
├── ubuntu2004
├── ubuntu2204
└── ubuntu2404
```

### 12.3 Component Profile

```text
Component Profile:
- nginx
- consul
- vector
- filebeat
- fluent-bit
- java
- tomcat
- apache
```

```text
OS Profile + Component Profile = Validation Environment
```

### 12.4 Target Group 분리 정책

대상 서버 OS profile이 다르면 작업을 분리한다.

```text
CentOS 7.9 + Ubuntu 22.04 혼합 대상
→ 단일 L3 작업 금지
→ OS group별 작업 분리
```

```yaml
target_groups:
  - name: rocky8_group
    os_profile: rocky8
    targets:
      - gw-r8-001
      - gw-r8-002

  - name: ubuntu22_group
    os_profile: ubuntu2204
    targets:
      - gw-u22-001
```

---

## 13. Validation Server/VM 전략

### 13.1 기본 입장

Validation Server/VM은 운영 서버 복제본이 아니다.

역할:

```text
- 스크립트 문법/흐름 검증
- 권한/패키지/서비스 구조 검증
- Runbook template 검증
- rollback/postcheck 흐름 검증
```

운영 서버별 실제 적합성은 Production Precheck에서 최종 확인한다.

### 13.2 Validation Path

```text
Fast Path
- static check
- policy check
- 운영 precheck
- 재설치 없음

Standard Path
- 기존 Validation Server에서 빠른 테스트
- 재설치 없음 또는 주기적 재설치

Full Path
- 운영 profile 수집
- Validation Server 초기화/재설치
- 필요한 component 설치
- 실제 테스트/rollback/reboot 검증
```

### 13.3 재설치 정책

즉시 재설치 필요:

- OS package install/remove 테스트
- reboot 테스트
- systemd unit 변경 테스트
- sudoers/user/group 변경 테스트
- 실패 후 상태 불명확

재설치 불필요:

- shellcheck
- config validate
- read-only 테스트
- dummy config 교체

```yaml
cleanup_policy:
  static_check:
    reinstall: false

  config_validation:
    reinstall: false
    cleanup_workspace: true

  package_install_test:
    reinstall: true

  reboot_test:
    reinstall: true

  failed_unknown_state:
    reinstall: true
```

---

## 14. Result Collection 및 Log 저장 전략

### 14.1 원칙

원본 로그 전체를 GitHub에 저장하지 않는다.

이유:

- 민감정보 포함 가능성
- repo 비대화
- 삭제/보존 정책 어려움
- 내부 URL/IP/token 노출 가능성

권장:

```text
원본 로그 → Jenkins Artifact 또는 Object Storage
GitHub → summary/result.json/script-diff.patch/artifact link
```

### 14.2 Log Retention / Redaction

로그 저장 정책은 별도로 관리한다.

```yaml
log_policy:
  raw_log_storage: object_storage
  github_log_storage: summary_only
  retention_days:
    low_risk: 90
    medium_risk: 180
    high_risk: 365
  redact_patterns:
    - token
    - password
    - secret
    - authorization_header
    - private_key
    - internal_url_query_string
  access_control:
    raw_logs: se_ops + security_audit
    summaries: engineering_read
```

Jenkins artifact에는 보존 기간을 설정하고, Object Storage 사용 시 lifecycle policy를 적용한다.

### 14.3 result.json 예시

```json
{
  "change_id": "CHG-20260520-001",
  "execution_level": "L2",
  "execution_mode": "script_delivery",
  "approved_commit": "abc123def",
  "host": "gw-001",
  "operator": "user01",
  "started_at": "2026-05-20T22:00:00+09:00",
  "ended_at": "2026-05-20T22:07:00+09:00",
  "artifact_url": "https://jenkins.example/artifacts/CHG-20260520-001/gw-001/",
  "steps": {
    "precheck": {
      "status": "success",
      "exit_code": 0,
      "log": "artifact://precheck.log"
    },
    "execute": {
      "status": "success",
      "exit_code": 0,
      "log": "artifact://execute.log"
    },
    "postcheck": {
      "status": "success",
      "exit_code": 0,
      "log": "artifact://postcheck.log"
    }
  },
  "script_integrity": {
    "approved_checksum_match": true,
    "modified_on_server": false,
    "diff_file": null
  },
  "final_status": "success",
  "rollback_executed": false
}
```

---

## 15. Failure Handling 설계

### 15.1 L3 실패 상태

```text
PRECHECK_FAILED
EXECUTE_FAILED
POSTCHECK_FAILED
PARTIAL_SUCCESS
ROLLBACK_RECOMMENDED
ROLLBACK_WAITING_APPROVAL
ROLLBACK_RUNNING
ROLLBACK_COMPLETED
ROLLBACK_FAILED
MANUAL_INTERVENTION_REQUIRED
CLOSED_WITH_EXCEPTION
```

### 15.2 실패 단계별 처리

Precheck 실패:

- execute 실행 금지
- 전체 작업 중단
- 실패 서버/사유 요약
- 수정 PR 또는 대상 제외 후 재승인

Execute 실패:

- 즉시 중단
- 다음 서버 진행 금지
- 현재 서버 상태 수집
- rollback 가능 여부 판단
- 사람 승인 후 rollback 또는 수동 조치

Postcheck 실패:

- 명령은 성공했지만 서비스 상태가 나쁜 상태
- 다음 서버 진행 중단
- rollback/유지/수동 확인 중 선택

Partial Success:

- 성공 서버, 실패 서버, 미진행 서버를 분리 기록
- 전체 rollback 또는 실패 서버만 조치 여부 판단

### 15.3 기본 실패 정책

```yaml
l3_failure_policy:
  stop_on_first_failure: true
  continue_on_failure: false
  auto_rollback_default: false
  rollback_requires_approval: true
  collect_diagnostics_on_failure: true
  manual_intervention_supported: true
  failed_change_blocks_remaining_targets: true
```

---

## 16. Runbook 없는 작업 처리

```text
Runbook 없음 → L3 금지
Runbook 없음 + 단순/비정형 → L2 Flexible 가능
Runbook 없음 + 위험 작업 → Review Only
Runbook 없음 + 반복 가능성 높음 → 신규 Runbook 후보 등록
```

```text
Workflow 요청서
  ↓
Agent 작업 유형 분류
  ↓
Runbook Registry 검색
  ↓
매칭 실패
  ↓
Unsupported Change Type
  ↓
Review Only / L2 Flexible / 신규 Runbook 후보
```

L2 Flexible 산출물:

```text
changes/CHG-001/
├── change.yaml
├── README.md
├── draft-precheck.sh
├── draft-execute.sh
├── draft-postcheck.sh
├── draft-rollback.sh
├── run.sh
└── unsupported-change-note.md
```

---

## 17. Command Allowlist

Runbook별 허용 명령과 금지 명령을 관리한다.

예: OpenSSL 업데이트 Runbook 허용 명령

```yaml
allowed_commands:
  - apt-get update
  - apt-get install --only-upgrade
  - dpkg-query
  - apt-cache policy
  - openssl version
  - systemctl --failed
  - lsof
  - df
```

금지 명령:

```yaml
forbidden_commands:
  - apt-get dist-upgrade
  - apt-get autoremove
  - apt-get purge
  - reboot
  - shutdown
  - systemctl restart
  - rm -rf
  - sed -i /etc/apt/sources.list
```

---

## 18. 예시 시나리오: Ubuntu 20.04 OpenSSL 100대 업데이트

### 18.1 작업 성격

- 대상: Ubuntu 20.04 서버 약 100대
- 작업: openssl/libssl 패키지 업데이트
- 위험도: HIGH
- 추천 실행 모드: L3 Staged Jenkins Execution

### 18.2 처리 흐름

```text
1. Workflow 승인 확인
2. 대상 서버 profile 수집
3. Ubuntu 20.04 대상만 분리
4. openssl/libssl 현재 버전 수집
5. 업데이트 후보 버전 확인
6. Validation VM 테스트
7. Git PR 생성
8. PR review/merge
9. Jenkins canary 1대 실행
10. 사람 확인
11. early batch 5대 실행
12. 사람 확인
13. normal batch 10대 단위 실행
14. 실패 시 즉시 중단
15. 결과 수집
16. 서비스 restart/reboot 필요 대상 후속 작업 생성
```

### 18.3 작업 분리 원칙

패키지 업데이트와 재부팅/서비스 restart를 한 작업에 묶지 않는다.

```text
CHG-001: OpenSSL package update
CHG-002: 필요한 서비스 rolling restart
CHG-003: 필요한 서버 reboot
```

---

## 19. 웹뷰 / 채팅 UX

### 19.1 Change Detail View

```text
상단:
- Change ID
- Workflow ID
- Runbook
- Execution Level
- Risk Level
- 현재 상태
- Lock 상태

중앙:
- Jenkins stage 상태
- 대상 서버별 상태
- 실시간 로그
- artifact link

오른쪽:
- Agent 분석
- 실패 원인 요약
- 다음 조치 추천
- follow-up PR 버튼

하단:
- rollback / retry / abort / manual intervention 버튼
```

### 19.2 Agent Chat 역할

- 현재 실패 단계 설명
- 로그 핵심 에러 추출
- 수동 확인 명령어 제안
- script patch 제안
- follow-up PR 생성
- Runbook 개선 후보 제안

중요 정책:

```text
L2 Flexible에서는 현장 수정 가능, 단 diff 회수 필수.
L3에서는 실행 중 live patch 금지, follow-up PR 후 재실행.
```

---

## 20. 도입 계획

### Phase 0: Workflow 이력 분석

- 최근 1년 SE 작업 이력 분석
- 최근 3년 참고 분석
- 반복 작업 유형 추출
- 자동화 후보 선정

산출물:

```text
SE Change Automation Candidate Report
```

### Phase 1: L2 Flexible MVP

- Runbook 1~2개
- Workflow 승인 확인
- Runbook 매칭
- GitHub PR 생성
- Jenkins Script Delivery
- run.sh 기반 수동 실행
- result.json/log/diff 생성
- Jenkins collect-result
- Agent 결과 요약

### Phase 2: L2 Strict 및 Runbook 개선

- 반복 작업의 현장 수정 패턴 분석
- Runbook template 개선
- checksum 기반 L2 Strict 도입
- GitHub required check 강화

### Phase 3: L3 Canary

- Jenkins precheck/execute/postcheck
- stop-on-first-failure
- canary 1대 실행
- lock 정책 적용
- 실패 시 Agent 분석
- follow-up PR 생성

### Phase 4: L3 Staged Execution

- batch execution
- human approval after canary
- Object Storage 로그 보관
- 웹뷰 실시간 로그
- Runbook maturity 관리
- DORA/운영 지표 대시보드

---

## 21. 초기 추천 Runbook 후보

```text
1. nginx_config_change
2. cert_replace
3. log_agent_config_change
4. consul_nginx_config_change
5. ubuntu2004_openssl_update
```

초기 L3 후보는 다음처럼 좁힌다.

```text
- nginx config test/reload
- 인증서 교체 후 reload
- 단일 OS/단일 패키지 업데이트
```

---

## 22. 운영 지표

자동화가 안정적으로 좋아지고 있는지 판단하기 위해 다음 지표를 수집한다.

```text
Change Throughput
- 월별 change 수
- Runbook 기반 change 비율
- L2/L3 실행 비율

Quality
- precheck 실패율
- execute 실패율
- postcheck 실패율
- rollback 비율
- manual intervention 비율
- change failure rate

Recovery
- failed deployment recovery time
- rollback decision time
- follow-up PR 생성까지 걸린 시간

Maturity
- L2 Flexible → L2 Strict 승격 수
- L2 Strict → L3 Canary 승격 수
- L3 Canary → L3 Staged 승격 수
- Runbook별 반복 성공 횟수
```

지표는 개인 평가가 아니라 Runbook 품질과 프로세스 병목을 찾기 위한 목적으로 사용한다.

---

## 23. 주요 정책 요약

```text
1. Agent는 운영 서버에 직접 접속하지 않는다.
2. L3는 승인된 Runbook이 있는 작업만 가능하다.
3. Runbook 없는 작업은 Review Only 또는 L2 Flexible이다.
4. L2는 script delivery + 수동 실행 + 결과 회수까지 포함한다.
5. L3는 Jenkins 자동 실행이지만 실패 시 즉시 중단한다.
6. L3 실행 중 script live patch는 금지한다.
7. L3 수정은 follow-up PR 후 재실행한다.
8. 원본 로그는 GitHub가 아니라 artifact/object storage에 저장한다.
9. GitHub에는 result summary와 diff만 저장한다.
10. 반복되는 L2 작업은 Runbook화해서 L3로 승격한다.
11. Jenkins는 승인된 commit과 change.yaml 기준으로만 실행한다.
12. 운영 실행 전 host/service lock을 획득한다.
13. Runbook template provenance와 script checksum을 검증한다.
14. branch protection, required checks, CODEOWNERS를 적용한다.
15. credential과 sudo 권한은 runbook/job 단위 최소 권한으로 제한한다.
```

---

## 24. 참고 기준

본 설계는 다음 공개 자료의 원칙을 참고했다.

- Google SRE Book: Release Engineering
  - https://sre.google/sre-book/release-engineering/
- Google SRE Workbook: Canarying Releases
  - https://sre.google/workbook/canarying-releases/
- Google SRE Workbook: Postmortem Culture
  - https://sre.google/workbook/postmortem-culture/
- GitHub Docs: Protected Branches
  - https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches
- GitHub Docs: CODEOWNERS
  - https://docs.github.com/articles/about-code-owners
- Jenkins Pipeline Syntax
  - https://www.jenkins.io/doc/book/pipeline/syntax/
- Jenkins Credentials Binding
  - https://www.jenkins.io/doc/pipeline/steps/credentials-binding/
- Jenkins Lockable Resources Plugin
  - https://plugins.jenkins.io/lockable-resources
- NIST SP 800-128: Security-Focused Configuration Management
  - https://csrc.nist.gov/publications/detail/sp/800-128/final
- NIST SP 800-92: Guide to Computer Security Log Management
  - https://csrc.nist.gov/pubs/sp/800/92/final
- DORA Metrics
  - https://dora.dev/guides/dora-metrics-four-keys/

---

## 25. 결론

본 설계의 핵심은 “AI Agent가 운영 작업을 마음대로 자동화하는 것”이 아니다.

핵심은 다음이다.

```text
Runbook as Code
+ Git PR 기반 승인
+ GitHub 보호 정책
+ Jenkins 기반 배포/실행
+ execution lock
+ render provenance
+ L2/L3 단계적 실행
+ 결과 회수와 실패 분석
+ Runbook 개선 루프
```

처음에는 L2 Flexible을 중심으로 시작하고, 충분히 정규화되고 반복 검증된 Runbook만 L3로 승격시키는 방식이 가장 안전하다.

최종 목표는 다음과 같다.

```text
SE팀의 반복 변경 작업을 Runbook Registry로 정규화하고,
작업자는 PR과 웹뷰에서 통제권을 유지하며,
Jenkins는 승인된 절차만 실행하고,
Agent는 분석/계획/로그 해석/후속 PR 생성을 담당하는 구조를 만든다.
```
