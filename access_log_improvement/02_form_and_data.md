# 체크인 폼 및 데이터 구조 정의서

## 1. 목적

이 문서는 Google Form 문항, Google Sheets 저장 구조, n8n 정규화 필드를 한 번에 정리한 실행 문서다.

## 2. Google Form 기본 정보

폼 제목:

- `IDC 외부 방문 체크인`

설명문:

- `외부 방문 인력 또는 현장 확인 대상 작업의 체크인을 위한 폼입니다. 파트너 아지트 승인번호 또는 요청번호와 기본 방문 정보를 입력해 주세요.`

운영 권장:

- `checkin_at`은 사용자가 직접 입력하지 않고 Google Form의 `Timestamp`를 공식 체크인 시각으로 사용한다
- 현장에서는 `케이지 출입 직전` 또는 `현장 확인 직후 즉시 제출`을 원칙으로 운영한다

## 3. 필수 문항

### 문항 1. 파트너 아지트 승인번호 또는 요청번호

- 필드명: `partner_request_id`
- 형식: 단답형
- 필수: 예

### 문항 2. 방문 센터

- 필드명: `site_code`
- 형식: 드롭다운
- 선택지:
  - `가산(LG) IDC`
  - `판교(SK) IDC`
- 필수: 예

### 문항 3. 방문자 이름

- 필드명: `visitor_name`
- 형식: 단답형
- 필수: 예

### 문항 4. 방문자 소속 업체명

- 필드명: `vendor_company`
- 형식: 단답형
- 필수: 예

### 문항 5. 방문 목적

- 필드명: `visit_purpose`
- 형식: 드롭다운
- 선택지:
  - `점검`
  - `설치`
  - `장비교체`
  - `장애조치`
  - `케이블 작업`
  - `기타`
- 필수: 예

### 문항 6. 확인한 상주 외주 인력 이름

- 필드명: `onsite_operator_name`
- 형식: 드롭다운
- 필수: 예

### 문항 7. 케이지 또는 작업 위치

- 필드명: `cage_or_zone`
- 형식: 드롭다운
- 필수: 예

## 4. 선택 문항

- `party_size`
- `expected_checkout_at`
- `notes`
- `photo_attachment`

## 5. Google Sheets 시트 구성

### `form_responses_raw`

- Google Form 원본 응답 저장
- Google Form `Timestamp`를 포함한다

### `checkins_normalized`

- n8n 정규화 데이터
- Jenkins 입력 기준 시트

### `checkins_invalid`

- 필수값 누락 또는 형식 오류 데이터

## 6. 정규화 필드

- `submission_id`
- `checkin_at`
- `submitted_at`
- `input_channel`
- `partner_request_id`
- `site_code`
- `visitor_name`
- `vendor_company`
- `party_size`
- `visit_purpose`
- `visit_purpose_other`
- `onsite_operator_name`
- `cage_or_zone`
- `expected_checkout_at`
- `notes`
- `qr_location_code`
- `validation_status`
- `validation_reason`
- `normalized_at`

## 7. 코드값 정규화

### 센터 코드

- `가산(LG) IDC` -> `GASAN_LG`
- `판교(SK) IDC` -> `PANGYO_SK`

### 방문 목적 코드

- `점검` -> `INSPECTION`
- `설치` -> `INSTALLATION`
- `장비교체` -> `HW_REPLACEMENT`
- `장애조치` -> `INCIDENT_RESPONSE`
- `케이블 작업` -> `CABLE_WORK`
- `기타` -> `OTHER`

## 8. 유효성 검증

아래 값이 비면 `INVALID` 처리한다.

- `partner_request_id`
- `site_code`
- `visitor_name`
- `vendor_company`
- `visit_purpose`
- `onsite_operator_name`
- `cage_or_zone`

`checkin_at` 운영 원칙:

- 초기 버전에서는 Google Form의 `Timestamp` 값을 `checkin_at`으로 사용한다
- `submitted_at`은 n8n이 정규화 저장한 시각으로 구분한다

`MISSING_REQUEST_ID` 운영 원칙:

- Google Form 경로에서는 `partner_request_id`를 필수로 받기 때문에 원칙적으로 발생하지 않는다
- 이 코드는 향후 API 연계, 수동 적재, 비정상 입력 같은 예외 경로를 위한 보호 규칙으로 유지한다

## 9. submission_id 규칙

권장 포맷:

- `CHK-YYYYMMDD-####`

예:

- `CHK-20260403-0012`

## 10. 운영 원칙

- Jenkins는 Google Form 원본이 아니라 `checkins_normalized`만 읽는다
- `partner_request_id`는 방문 승인번호 또는 파트너 아지트 요청번호를 뜻하는 공통 식별자로 사용한다
- 사람 친화적 문항명과 시스템 필드명은 분리한다
- `checkin_at`은 Google Form `Timestamp`, `submitted_at`은 n8n 저장 시각으로 구분한다

## 11. 결론

이번 도입에서 가장 중요한 데이터 원칙은 아래 세 가지다.

- 승인/요청 식별자를 반드시 받는다
- 센터와 방문자 정보를 구조화한다
- Jenkins 입력 스키마를 고정한다
