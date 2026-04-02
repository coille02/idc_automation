# 자동 대조 및 후속조치 설계서

## 1. 목적

이 문서는 Jenkins 대조 로직, n8n 연계, Slack 알림, 파트너 아지트 자동 확인요청 생성을 한 문서로 묶은 실행 설계서다.

## 2. 대조 대상

### 승인/요청 데이터

- 출처: 파트너 아지트
- 필수 필드:
  - `partner_request_id`
  - `request_status`
  - `site_code`
  - `approved_visit_date`
  - `vendor_company`
  - `visitor_name`

### 체크인 데이터

- 출처: Google Form -> n8n -> `checkins_normalized`
- 필수 필드:
  - `partner_request_id`
  - `visitor_name`
  - `vendor_company`
  - `site_code`
  - `checkin_at`
  - `onsite_operator_name`
  - `submission_id`

운영 기준:

- `checkin_at`은 사용자가 직접 적는 시간이 아니라 Google Form `Timestamp`를 사용한다
- 현장 체크인은 `출입 직전 또는 확인 직후 즉시 제출`을 원칙으로 한다

## 3. Jenkins 대조 기준

### 기본 키

- `partner_request_id`

### 보조 검증

- `site_code`
- `visitor_name`
- `vendor_company`
- 방문일 허용 범위

### 상태 판정

- `정상`
- `누락 의심`
- `절차 위반`
- `정보 불일치`

### 대표 reason_code

- `NO_CHECKIN`
- `MISSING_REQUEST_ID`
- `INVALID_REQUEST_ID`
- `SITE_MISMATCH`
- `NAME_MISMATCH`
- `COMPANY_MISMATCH`
- `DATE_OUT_OF_RANGE`

보충 설명:

- `MISSING_REQUEST_ID`는 Google Form 경로에서는 원칙적으로 발생하지 않는다
- 이 코드는 API 연계, 수동 적재, 비정상 입력 등 예외 채널에 대한 보호 규칙으로 유지한다

## 4. 배치 주기

권장 주기:

- 매일 오전 8시
- 매일 오후 4시

운영 원칙:

- 오전 배치는 전일 누락 점검
- 오후 배치는 당일 중간 점검

## 5. n8n 역할

- Google 응답 수집
- 데이터 정규화
- Jenkins 입력 데이터 준비
- Jenkins 결과 수신
- Slack 알림 발송
- 자동 후속조치 대상이면 파트너 아지트 API 또는 봇을 통해 확인요청서 생성

## 6. Slack 알림 기준

### 즉시 알림

- `MISSING_REQUEST_ID`
- `INVALID_REQUEST_ID`
- `SITE_MISMATCH`
- 배치 실패
- 승인 데이터 수집 실패

### 익일 아침 알림

- `NO_CHECKIN`
- `NAME_MISMATCH`
- `COMPANY_MISMATCH`

## 7. 파트너 아지트 자동 후속조치

### 자동 생성 대상

- `NO_CHECKIN`
- `MISSING_REQUEST_ID`
- `INVALID_REQUEST_ID`
- `SITE_MISMATCH`

### 확인요청서에 들어갈 값

- 요청 유형
- 예외 유형
- `partner_request_id`
- 방문자명
- 업체명
- 센터
- 승인 방문일
- 체크인 시각 또는 체크인 누락 여부
- `reason_code`
- 확인 요청 내용
- 회신 기한

구현 방식:

- 파트너 아지트는 API 연계를 기준으로 구현한다
- 확인요청 생성은 n8n에서 API 호출 또는 봇 호출 방식으로 처리한다

### 운영 의미

- Slack은 IDC 담당자 인지용
- 파트너 아지트 요청서는 외주 인력 확인 지시용

## 8. 예외 조치 흐름

1. Jenkins 예외 탐지
2. Slack 알림
3. 자동 후속조치 대상이면 파트너 아지트 확인요청서 생성
4. 외주 인력 사실확인
5. 필요 시 보정 기록 작성
6. IDC 담당자 종료 승인

## 9. 감사 대응 포인트

- 예외를 자동 탐지한다
- IDC 담당자에게 즉시 통보한다
- 외주 인력 확인 업무를 자동 요청서로 생성한다
- 보정 및 종료 이력을 남긴다

## 10. 결론

이번 자동화의 핵심은 `탐지`만이 아니라 `후속 확인 지시까지 자동으로 붙이는 것`이다.

즉 최종 구조는 아래다.

- Jenkins가 대조
- n8n이 연동
- Slack이 알림
- 파트너 아지트가 외주 확인 지시와 회신 창구
