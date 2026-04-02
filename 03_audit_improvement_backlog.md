# 감사 개선 백로그

## 1. 목적

이 문서는 감사 및 보안 실사에서 지적사항을 줄이기 위해 개선이 필요한 항목을 백로그 형태로 정리한 문서다.

현재 최우선 과제는 `출입기록 대조 체계 개선`이다.

## 2. 현재 최우선 과제

### 출입기록 대조 체계 개선

개선 방향:

- 출입대장 전산화
- 승인/요청과 실제 체크인 자동 대조
- 예외 Slack 알림
- 외주 인력 확인 요청 자동 생성
- 예외 종료 이력 관리

관련 문서:

- [01_overview_and_rollout.md](/f:/projects/persona/idc_automation/access_log_improvement/01_overview_and_rollout.md)
- [02_form_and_data.md](/f:/projects/persona/idc_automation/access_log_improvement/02_form_and_data.md)
- [03_automation_and_followup.md](/f:/projects/persona/idc_automation/access_log_improvement/03_automation_and_followup.md)
- [04_build_tasks.md](/f:/projects/persona/idc_automation/access_log_improvement/04_build_tasks.md)

## 3. 다음 감사 개선 후보

### 1. 작업 승인 및 결과보고 통제

- 승인 없는 작업 탐지
- 결과보고 누락 탐지
- 고위험 작업 입회 기록 확보

### 2. 외주 인력 통제

- 교육 이수 여부
- NDA/서약서 유효 여부
- 교대 인수인계 기록 확보

### 3. 증적 보관 체계

- 출입/작업/점검/승인 이력 보관
- 월별 증적 누락 탐지
- 감사 제출용 증적 묶음 자동화

### 4. 반입반출/자산 통제

- 반입 승인과 자산대장 일치 여부
- 반출 후 자산 반영 여부

### 5. 반복 지적사항 관리

- 재발 유형 식별
- 반복 위반 업체 또는 인력 관리
- 재발방지 조치 이력화

## 4. 감사 대응 관점 공통 원칙

- 탐지 통제만 만들지 말고 조치 통제까지 연결
- 가능하면 Slack 알림 후 자동 후속 요청 생성
- 원본 기록과 보정 기록을 분리 저장
- 모든 예외는 종료 상태와 종료 사유를 남김

## 5. 결론

감사 개선은 문서만 만드는 것이 아니라 `탐지 -> 조치 -> 종료 -> 재발방지` 흐름을 운영에 붙이는 것이 핵심이다.

현재는 출입기록 개선을 먼저 완료하고, 이후 이 백로그에서 순차적으로 확장한다.
