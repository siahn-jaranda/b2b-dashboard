# B2B 포인트 영업 대시보드 — 작업 인수인계

## 프로젝트 개요

B2B 제휴사별 포인트/쿠폰 발급·사용 현황을 파악하고, 혜택을 받고도 서비스를 이용하지 않는 사용자를 식별해 영업 액션을 지원하는 대시보드.

## 현재 상태 (2026-03-27 기준)

### 완료된 것
- `index.html`: 단일 HTML 파일 대시보드 (정적 데이터 내장)
  - **회사별 현황 탭**: 포인트 회사 카드 + 쿠폰 회사 카드 + 종료 예정 카드
  - **전체 사용자 탭**: 검색/필터/정렬 테이블 (포인트·쿠폰 통합) + CSV 다운로드
  - **영업 액션 탭**: Tier1(포인트 미사용 큰 금액) + Tier2(쿠폰 미사용) 액션 리스트
- 프로덕션 DB에서 실제 데이터 추출 완료 (Read Replica 사용)

### 핵심 데이터 발견 (수정됨)
- 포인트 6개사: 발급 대비 서비스 이용률 **약 73%** (블리자드 기준)
- 쿠폰 3개사: 실사용 3명 / 등록 100명
- 서비스 미이용자: 포인트 회사 약 10명, 쿠폰 회사 약 97명

## 제휴 포인트/쿠폰 구조

### 혜택 유형

B2B 제휴사에는 **포인트**와 **쿠폰** 두 가지 혜택 유형이 있다.

| 구분 | 포인트 | 쿠폰 |
|------|--------|------|
| 테이블 | `jrdtbl_jaranda_point_info` | `coupon_instance` |
| 식별 | `b2b_company.point_tag_sid` → `jrdtbl_jaranda_point.point_tag_sid` | `b2b_coupon_policy.coupon_policy_id` → `coupon_policy.id` |
| 발급 방식 | 매월 자동 발급 (DAY_OF_MONTH) | 회원 인증 시 자동 발급 (AUTO_CERTIFICATION) |
| 금액 | 회사별 상이 (15만~50만/월) | 10% 할인 (최대 1만원 할인) |
| 만료 | 발급일 기준 1개월 후 (`expire_datetime_at`) | 쿠폰 정책별 상이 |

### 포인트 발급/사용 추적의 함정

**`jrdtbl_jaranda_point_info` 테이블 구조:**
```
jaranda_point_use_type:
  1 = 발급 (양수) — jaranda_point_sid 있음 → point_tag_sid로 B2B 식별 가능
  2 = 사용 (음수) — jaranda_point_sid가 NULL → B2B 여부 직접 식별 불가
  3 = 소멸/취소 (양수)
  4 = 차감 (음수)
  5 = 기타 차감 (음수)
```

**핵심**: 사용(type=2) 레코드는 `jaranda_point_sid`가 NULL이다. 어떤 포인트 풀에서 차감됐는지 추적할 수 없다. `pre_jaranda_point_info_sid`로 원래 발급 건을 참조하지만, 해당 발급 건도 `jaranda_point_sid`가 NULL인 경우가 많다.

**따라서 대시보드에서는:**
- **발급**: `point_tag_sid`로 B2B 포인트만 정확히 집계
- **사용**: B2B 회원 계정의 `type=2` 전체 합산 (서비스 이용 실적 = 영업 목적에 부합)
- 사용 금액에는 마케팅 포인트(5,000원 프로모션 등) 소액이 포함될 수 있으나 무시 가능한 수준

### 쿠폰 구조

```
b2b_coupon_policy
  ├── b2b_company_id → b2b_company.id
  └── coupon_policy_id → coupon_policy.id
        └── coupon_instance (개별 쿠폰)
              ├── parent_id = account.sid (사용자)
              ├── status: ISSUED(미등록) / REGISTERED(등록) / USED(사용) / EXPIRED(만료)
              └── discount_amount (실 할인 금액)
```

`coupon_policy.status`가 `REGISTERING`인 정책만 활성 상태. `EXPIRED`면 해당 회사의 쿠폰 제휴가 종료된 것.

## DB 스키마 (jaranda DB, Read Replica)

### 주요 테이블 관계
```
b2b_company (제휴사)
  ├── id, name, email_domain, point_tag_sid, account_type
  ├── point_tag_sid → jrdtbl_jaranda_point_tag.sid (포인트 태그)
  │
  ├── b2b_user (제휴 회원)
  │     ├── account_sid → account.sid (자란다 계정 연결)
  │     ├── company_id → b2b_company.id
  │     ├── b2b_email, register_status (REGISTERED/UNREGISTERED)
  │     └── deleted_at (soft delete)
  │
  ├── b2b_benefit_group (혜택 그룹)
  │     ├── b2b_company_id, name, addition_type (ADMIN/CERTIFICATION_AUTO)
  │     ├── b2b_point_issue_policy (포인트 발급 정책)
  │     └── b2b_user_benefit_group (사용자-혜택 매핑)
  │
  ├── b2b_coupon_policy (쿠폰 정책)
  │     ├── b2b_company_id → b2b_company.id
  │     └── coupon_policy_id → coupon_policy.id
  │
  ├── jrdtbl_jaranda_point (포인트 마스터)
  │     └── jrdtbl_jaranda_point_info (발급/사용 내역)
  │
  └── coupon_instance (쿠폰 개별 인스턴스)
```

### 핵심 쿼리 — 포인트 사용자별 현황 (수정됨)
```sql
-- 발급: B2B point_tag_sid로 정확히 집계
-- 사용: 계정의 전체 서비스 이용 금액 (type=2)
SELECT
  c.name AS company_name,
  bu.b2b_email,
  a.name AS user_name,
  COALESCE(issued.total, 0) AS total_issued,
  COALESCE(ABS(used.total), 0) AS total_used,
  a.last_signed_in
FROM b2b_user bu
JOIN b2b_company c ON bu.company_id = c.id
LEFT JOIN account a ON bu.account_sid = a.sid
LEFT JOIN (
  SELECT pi.account_sid, SUM(pi.jaranda_point) AS total
  FROM jrdtbl_jaranda_point_info pi
  JOIN jrdtbl_jaranda_point jp ON pi.jaranda_point_sid = jp.sid
  JOIN b2b_user bu2 ON pi.account_sid = bu2.account_sid
  JOIN b2b_company c2 ON bu2.company_id = c2.id AND jp.point_tag_sid = c2.point_tag_sid
  WHERE pi.jaranda_point_use_type = 1
    AND bu2.deleted_at IS NULL AND bu2.register_status = 'REGISTERED'
  GROUP BY pi.account_sid
) issued ON bu.account_sid = issued.account_sid
LEFT JOIN (
  SELECT pi.account_sid, SUM(pi.jaranda_point) AS total
  FROM jrdtbl_jaranda_point_info pi
  WHERE pi.jaranda_point_use_type = 2 AND pi.jaranda_point < 0
  GROUP BY pi.account_sid
) used ON bu.account_sid = used.account_sid
WHERE bu.deleted_at IS NULL
  AND bu.register_status = 'REGISTERED'
  AND c.name IN ('블리자드','HMM','코니바이에린','HMM오션서비스','HMM 해상','블리자드 스튜디오')
ORDER BY c.name, total_issued DESC;
```

### 핵심 쿼리 — 쿠폰 사용자별 현황
```sql
SELECT
  c.name AS company_name,
  bu.b2b_email,
  a.name AS user_name,
  cp.name AS coupon_name,
  COUNT(ci.id) AS coupon_issued,
  SUM(ci.status = 'USED') AS coupon_used,
  SUM(ci.status = 'REGISTERED') AS coupon_registered,
  a.last_signed_in
FROM b2b_user bu
JOIN b2b_company c ON bu.company_id = c.id
JOIN b2b_coupon_policy bcp ON bcp.b2b_company_id = c.id
JOIN coupon_policy cp ON bcp.coupon_policy_id = cp.id AND cp.status = 'REGISTERING'
LEFT JOIN account a ON bu.account_sid = a.sid
LEFT JOIN coupon_instance ci ON ci.coupon_policy_id = cp.id AND ci.parent_id = a.sid
WHERE bu.deleted_at IS NULL
  AND bu.register_status = 'REGISTERED'
GROUP BY c.name, bu.b2b_email, a.name, cp.name, a.last_signed_in
ORDER BY c.name, coupon_issued DESC;
```

## 제휴사 목록

### 활성 — 포인트 (6개사)

| 회사 | 사용자수 | 월 발급액/인 | 서비스 이용률 | 비고 |
|------|---------|------------|-------------|------|
| 블리자드 | 21 | 50만 | ~73% | 19명 활발 사용, 2명 미사용 |
| HMM | 21 | 15~30만 | ~70% | 대부분 사용, 2명 미사용 |
| 코니바이에린 | 17 | 15~30만 | ~40% | 12명 사용, 5명 미사용 |
| HMM오션서비스 | 4 | 15만 | 낮음 | 1명 소액 사용, 3명 미사용 |
| HMM 해상 | 3 | 15만 | 낮음 | 1명 사용 |
| 블리자드 스튜디오 | 1 | 50만 | 0% | 1명 완전 미사용 |

### 활성 — 쿠폰 (4개사)

| 회사 | 등록 사용자 | 쿠폰 유형 | 사용 | 비고 |
|------|-----------|----------|------|------|
| 신용보증기금 | 77 | 긴급돌봄 10%할인 | 3명 사용 | 가장 활발 |
| CJ 제휴사 | 19 | 임직원 10%할인 | 1명 사용 | |
| SK가스 | 4 | 긴급돌봄 10%할인 | 0명 | 전원 미사용 |
| 두산 | 5 | 임직원 전용쿠폰 (2시간) | 2명 사용 (7건) | 기존 "완전 제외"에서 복원 |

### 종료 예정 (2개사)

신규 발급은 중지되었지만, 사용 기한이 남아있는 포인트가 있음.

| 회사 | 사용자수 | 누적 발급 | 마지막 발급 | 상태 |
|------|---------|----------|-----------|------|
| 한국백혈소아암협회 | 13 | 2,792만 | 2025-07-01 | 포인트 발급 중단 8개월, 쿠폰 없음 |
| 비대면바우처 | 10 | 2,177만 | 2024-05-13 | 포인트 발급 중단 1년 10개월, 쿠폰 없음 |

### 완전 제외 (3개사) — 대시보드에서 제거됨

| 회사 | 사유 |
|------|------|
| 이노션 | 포인트 발급 이력 없음 + 쿠폰 정책 만료 |
| 로지스올 | 포인트 발급 이력 없음 + 쿠폰 정책 만료 |
| 판다 미디어 | 포인트 발급 중단 + 쿠폰 정책 만료 |

## 이어서 해야 할 작업

### 우선순위 1: 데이터 라이브 연동
현재는 정적 JSON이 HTML에 하드코딩되어 있음. 선택지:
- **A) Google Sheets 연동**: Apps Script로 DB 쿼리 → 시트 업데이트 → fetch로 읽기
- **B) Cloud Run API**: 경량 API 서버 → DB 직접 쿼리 → JSON 반환
- **C) 주기적 export**: Scheduler로 쿼리 실행 → GCS에 JSON 저장 → fetch

### 우선순위 2: 포인트 만료 데이터 추가
- `jrdtbl_jaranda_point_info.expire_datetime_at` 필드로 만료 임박 포인트 계산
- "30일 내 만료 예정 포인트" 경고 카드 추가
- 만료 임박 사용자 우선 영업 대상으로 분류

### 우선순위 3: 추가 기능
- 회사 카드 클릭 시 상세 사용자 목록 확장
- 시간 추이 차트 (월별 발급/사용 트렌드)
- Slack 알림 연동 (신규 미사용자, 만료 임박 등)

## 기술 스택 / 제약

- 순수 HTML + Vanilla JS (의존성 없음)
- 폰트: Google Fonts (Noto Sans KR, JetBrains Mono)
- 다크 테마 기반 UI
- DB 접근: MySQL Read Replica만 사용 (SELECT only)
- GCP 프로젝트: 리소스 생성 시 `fynn-` prefix 사용

## 안전 규칙

- DB는 **반드시 Read Replica**만 사용 (INSERT/UPDATE/DELETE 금지)
- Cloud Run 배포 시 PROJECT_ID가 `platform-jaranda-kr`(본 프로덕션)이 아닌지 확인
- 개인정보(핸드폰번호 등)는 대시보드에 노출하지 않음 — 이메일까지만 표시
- `.env`, API 키 등은 코드에 하드코딩 금지
