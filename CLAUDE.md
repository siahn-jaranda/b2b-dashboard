# Claude Code 작업 지침

## 컨텍스트

이 프로젝트는 자란다(jaranda) B2B 제휴사 포인트 영업 대시보드입니다.
Fynn이 초기 데이터 분석과 대시보드 프로토타입을 만들었고, 팀원이 이어서 개발합니다.

## 현재 파일 구조

```
b2b-dashboard/
├── index.html      # 대시보드 (정적 데이터 내장, 단일 파일)
├── HANDOFF.md      # DB 스키마, 데이터 발견, 이어할 작업 상세
└── CLAUDE.md       # 이 파일 (Claude Code 지침)
```

## 작업 시 반드시 지켜야 할 것

1. **DB는 Read Replica만 사용** — SELECT 쿼리만 허용. 절대 INSERT/UPDATE/DELETE 하지 않음
2. **개인정보 주의** — 핸드폰번호는 대시보드에 노출하지 않음. 이메일까지만 표시
3. **GCP 리소스 생성 시** — `fynn-` prefix 사용 (예: `fynn-b2b-dashboard`)
4. **한국어 UI** — 대시보드는 한국어로 유지
5. **다크 테마** — 기존 CSS 변수 체계 유지 (--bg, --surface, --accent 등)

## 데이터 갱신 방법

현재 `index.html` 안의 `const users = [...]` 배열이 정적 데이터입니다.
DB에서 최신 데이터를 가져오려면 `HANDOFF.md`의 "핵심 쿼리" 섹션 참고.

MCP 도구가 있다면:
```
서버: user-jaranda_prod_db
도구: mysql_query
인자: { "sql": "SELECT ..." }
```

## 핵심 비즈니스 맥락

- **목적**: 포인트를 발급받고도 안 쓰는 사람들에게 영업해서 사용하게 만드는 것
- **현실**: 사용률이 0.39%로, "누가 안 쓰는지" 이전에 "왜 안 쓰는지"가 더 근본적 질문
- **2가지 문제**: (1) 포인트 있는데 안 쓰는 사람 97명 (2) 포인트 발급 자체가 안 되는 회사 6개
- **최대 적체**: 블리자드 권병우님 4,200만원 (매월 50만원씩 쌓이는 중)

## 코드 수정 가이드

### 데이터 추가/수정
`index.html` 내 `const users` 배열에 객체 추가:
```js
{
  company_name: "회사명",
  b2b_email: "email@company.com",
  user_name: "이름",
  total_issued: 1000000,   // 누적 발급액 (원)
  total_used: 0,            // 누적 사용액 (원)
  last_signed_in: "2026-03-27"  // 마지막 로그인
}
```

### 새 회사 카드 추가 (포인트 미발급)
`const noPointCompanies` 배열에 추가:
```js
{name:"회사명", users:10, policy:"정책 설명"}
```

### CSS 커스터마이즈
`:root` 변수 수정으로 전체 테마 변경 가능:
- `--accent`: 강조색 (현재 #ff3b5c)
- `--bg`: 배경색
- `--surface`: 카드 배경색

## 다음 스텝 제안 (우선순위순)

1. **포인트 만료 임박 데이터 추가** — `expire_datetime_at` 기반 30일 내 만료 예정 표시
2. **CSV 다운로드 버튼** — 영업팀이 엑셀로 활용할 수 있게
3. **데이터 자동 갱신** — 정적 → API 또는 Google Sheets 연동
4. **Slack 알림** — 신규 미사용자, 만료 임박 자동 알림
