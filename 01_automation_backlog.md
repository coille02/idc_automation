# IDC 자동화 백로그

## 1. 목적

이 문서는 IDC 담당자 관점에서 앞으로 자동화하거나 agentic AI를 적용할 수 있는 과제를 백로그 형태로 정리한 문서다.

현재 최우선 과제는 `센터 출입기록 개선`이며, 그 이후 순차적으로 검토할 자동화 후보를 정리한다.

상세 설계와 실행은 [access_log_improvement](/f:/projects/persona/idc_automation/access_log_improvement)에서 진행한다.

## 2. 현재 최우선 과제

### 센터 출입기록 개선

목표:

- 출입대장 전산화
- 승인/요청과 현장 체크인 자동 대조
- Slack 알림
- 파트너 아지트 자동 확인요청 생성
- 감사 대응 가능한 이력 확보

관련 문서:

- [01_overview_and_rollout.md](/f:/projects/persona/idc_automation/access_log_improvement/01_overview_and_rollout.md)
- [02_form_and_data.md](/f:/projects/persona/idc_automation/access_log_improvement/02_form_and_data.md)
- [03_automation_and_followup.md](/f:/projects/persona/idc_automation/access_log_improvement/03_automation_and_followup.md)
- [04_build_tasks.md](/f:/projects/persona/idc_automation/access_log_improvement/04_build_tasks.md)

## 3. 다음 자동화 후보

### 1. 작업요청-입회-결과보고 관리

자동화 후보:

- 작업요청서 접수
- 필수 항목 누락 탐지
- 고위험 작업 분류
- 작업 종료 후 결과보고 누락 탐지
- Slack 알림 및 후속 확인 요청

### 2. 외주 상주/교대/인수인계 관리

자동화 후보:

- 교대 인수인계 입력 전산화
- 인수인계 미작성 탐지
- 상주 일정 및 실제 근무 대조
- 교육/NDA 만료 알림

### 3. 점검표 및 종이 체크리스트 전산화

자동화 후보:

- 정기 점검표 모바일 입력
- 월 1회 스캔 업로드 제거
- 점검 누락 자동 탐지
- 이상 항목 Slack 통보

### 4. 증적 자동 수집

자동화 후보:

- 출입, 작업, 점검, 승인 이력 자동 저장
- 감사 요청 시 증적 패키지 자동 정리
- 월별 증적 누락 탐지

### 5. 반입반출 및 자산 정합성 확인

자동화 후보:

- 반입반출 승인과 자산대장 대조
- 미반영 자산 탐지
- 임시 반입 장비 회수 예정 추적

### 6. 지적사항 및 재발방지 관리

자동화 후보:

- 지적사항 등록
- 조치 상태 추적
- 반복 위반 탐지
- 재발방지 조치 이력 관리

### 7. 일일 운영 브리핑 자동화

자동화 후보:

- 전일 출입 예외 요약
- 작업 예정/미종결 요약
- 점검 누락 요약
- IDC 담당자용 Slack 브리핑 생성

## 4. 공통 적용 기술

현재 기준 공통 조합:

- 입력: Google Forms 또는 내부 폼
- 연동 허브: sandbox n8n
- 비교/점검: sandbox Jenkins
- 알림: Slack
- 후속 확인: 파트너 아지트 요청서
- 보고: Confluence

## 5. 운영 우선순위

1. 출입기록 개선
2. 작업요청-입회-결과보고 관리
3. 외주 상주/교대/인수인계 관리
4. 점검표 전산화
5. 증적 자동 수집
6. 반입반출/자산 정합성
7. 지적사항 관리

## 6. 결론

상위 백로그는 크게 잡고, 실제 실행은 과제별 하위 폴더에서 상세화한다.

현재는 `센터 출입기록 개선`이 가장 우선이며, 나머지는 이 문서에서 순차적으로 관리한다.
