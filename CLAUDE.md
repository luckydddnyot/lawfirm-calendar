# CLAUDE.md

법무법인 일정공유 캘린더 프로젝트입니다. 변호사별 일정을 공유하고, 업무량 통계·상담가능시간 찾기 기능을 제공합니다.

## 아키텍처

- **단일 파일 구조**: 전체 앱이 `index.html` 하나(약 1,570줄)에 들어 있습니다. HTML + `<style>` + `<script>`가 모두 한 파일에 포함됩니다. 빌드 도구·번들러·패키지 매니저 없음.
- **백엔드: Supabase** (PostgreSQL + RLS + Auth + Realtime)
  - 클라이언트: `@supabase/supabase-js@2` (jsDelivr CDN로 로드)
  - 인증: Google OAuth (`signInWithOAuth`)
  - 실시간: `postgres_changes` 채널 구독 + 60초 폴링 + 포커스 시 재로드
  - 접속 정보(`SUPABASE_URL`, `SUPABASE_KEY`)는 `index.html` 상단 `<script>`에 하드코딩 (publishable anon key)
- **외부 API**: 공휴일은 Nager.Date 무료 API(`date.nager.at`)에서 가져옴

## 배포 (중요)

- **GitHub Pages**로 호스팅. `main` 브랜치에 push하면 **자동 배포**됩니다.
- 배포 반영까지 **약 1~2분** 소요됩니다. push 직후 바로 확인되지 않을 수 있음.

## 작업 규칙 (중요)

1. **DB 스키마 변경이 필요한 작업**은, 코드를 수정하기 전에 **실행할 SQL을 먼저 사용자에게 알려줄 것.** Supabase 대시보드에서 사용자가 직접 SQL을 실행해야 하며, 새 컬럼/테이블/RLS 정책 등이 적용된 뒤에 프론트엔드 코드를 반영합니다. (CLI로 마이그레이션을 적용하지 말 것 — 마이그레이션 파일이나 Supabase CLI 설정이 이 저장소에 없음)
2. **코드 수정 후에는** 수정 내용을 요약한 커밋 메시지를 만들고, `git commit` 과 `git push` 까지 진행할 것. (push하면 위 "배포" 규칙대로 1~2분 뒤 자동 배포됨)

## DB 스키마 (코드에서 사용 중인 테이블/컬럼)

스키마 정의 파일은 저장소에 없으므로, 아래는 `index.html`의 쿼리에서 역으로 파악한 구조입니다. 스키마를 바꿀 때는 이 목록과의 정합성을 확인하세요.

- **`members`** — 가입/권한 관리
  - `id`, `email`, `name`, `role`('변호사'|'송무직원'|'상담직원'|'기타'), `lawyer_id`(→lawyer, 본인/담당 변호사 연결), `created_at`
  - `status`: `pending`(승인 대기) | `viewer`(읽기전용) | `approved`(편집가능) | `admin`(관리자)
  - 신규 로그인 시 자동으로 `pending`으로 insert됨. 관리자가 승인.
- **`lawyer`** — 변호사
  - `id`, `name`, `color`, `staff`(송무직원), `dept`(소속), `accepts_consult`(상담가능시간 찾기 포함 여부, 기본 true), `created_at`
- **`event_type`** — 일정 타입
  - `id`, `name`, `symbol`(기호), `weight`(통계 점수 가중치, 기본 1), `created_at`
- **`event`** — 일정
  - `id`, `title`, `lawyer_id`(→lawyer), `type_id`(→event_type), `date`, `start_time`, `end_time`, `loc`, `memo`
  - `is_remote_court`(★ 서울 외 지방 출석), `is_video`(영상 진행)
  - `created_by`, `updated_by`, `updated_at`, `created_at`
- **`feedback`** — 개발자 문의 (관리자만 열람)
  - `id`, `email`, `name`, `content`, `created_at`

## 권한 모델

- `canEdit()` = status가 `approved` 또는 `admin` 인 경우만 일정 등록/수정/삭제 및 [관리] 사용 가능
- `viewer`는 읽기전용, `pending`은 승인 대기 화면만 표시
- 멤버 승인/권한 변경·문의 열람은 `admin`만 가능
- RLS 정책이 이 권한 모델을 DB에서도 강제하고 있을 가능성이 높으므로, 권한 관련 동작을 바꿀 때는 프론트와 RLS를 함께 점검할 것

## 주요 기능 영역 (index.html 내 위치는 주석으로 구분됨)

- 인증/멤버 해석: `initAuth`, `resolveMember`
- 데이터 로드/실시간: `loadAll`, `subscribeRealtime`
- 월간 뷰: `renderMonth` / 주간 리소스 뷰(변호사 × 날짜): `renderWeek`
- 일정 모달: `openEventModal`, 저장/삭제 핸들러
- 관리 모달(변호사·타입·멤버): `renderManage`, `renderMembers`
- 업무량 통계: `lawyerMonthStats`, `renderStats` (점수 = Σ타입 가중치 + 지방건수 × 지방가중치)
- 상담가능시간 찾기: `freeSlots`, `renderFind`
- 문의: `renderFbList`, `updateFbBadge`

## 로컬 개발

- 빌드/설치 단계 없음. `index.html`을 직접 열거나 정적 서버로 서빙하면 됩니다.
- 단, Google OAuth 리디렉트(`redirectTo: location.origin + location.pathname`)와 Supabase 허용 URL 설정 때문에, 로그인 흐름은 배포된 GitHub Pages URL에서 확인하는 것이 정확합니다.
