# 코드·API 규칙

- **담당:** 개발 총괄. 이름 자체(표준어·코드 이름)는 [GLOSSARY.md](GLOSSARY.md)가 기준입니다.

## 1. 이름 규칙

| 대상 | 규칙 | 예 |
|---|---|---|
| Java 클래스·엔티티 | PascalCase 단수 | `Membership`, `TeacherAssignment` |
| DB 테이블 | snake_case **복수** | `memberships`, `teacher_assignments` |
| DB 컬럼 | snake_case | `delete_scheduled_at` |
| API JSON 필드 | camelCase | `deleteScheduledAt` |
| enum 값 | UPPER_SNAKE, DB에는 문자열로 저장 | `PENDING_APPROVAL` |
| 시각 | `*_at`, DB 타입 `TIMESTAMPTZ`(UTC 저장) | `due_at`, `started_at` |
| 기본 키 | UUID | |
| 사람을 가리키는 외래 키 | 계정이면 `user_id`, 학원 안의 사람이면 **소속(Membership) id**이고 역할을 이름에 붙임 | `student_id`, `teacher_id`, `responsible_teacher_id` → `memberships.id` |
| Dart(앱)·TypeScript(웹) | 각 언어 관례(lowerCamelCase 변수, PascalCase 타입). API 필드 이름은 JSON과 같게 | `dueAt` |

- Java에서 예약어나 기본 클래스와 겹치는 이름은 쓰지 않습니다(예: 반은 `Class`가 아니라 `ClassGroup`).

## 2. API 규칙
- **경로:** `/api/v1/...`, 복수 명사(`/api/v1/homeworks/{homeworkId}`).
- **성공 응답:** `ApiResponse` 래퍼 `{ "code": 200, "status": "success", "message": "OK", "data": ... }`. 204는 본문이 없습니다.
- **실패 응답:** RFC 9457 Problem Details(`Content-Type: application/problem+json`)에 `code`(ErrorCode 이름)를 더합니다. 앱·웹은 문구가 아니라 `code`로 분기합니다.
  ```json
  { "status": 404, "title": "Not Found", "detail": "요청한 정보를 찾을 수 없어요.", "code": "NOT_FOUND" }
  ```
  - 입력 검증 실패는 `errors: [{ "field": "...", "reason": "..." }]`를 더합니다. **입력값 자체는 싣지 않습니다.**
  - 예외 메시지·스택은 응답에 싣지 않습니다(서버 로그에만, 개인정보 없이).
- **시간:** API의 시각은 ISO 8601 + 오프셋(예: `2026-11-04T14:59:00Z`). 날짜만 필요한 값(생년월일)은 `YYYY-MM-DD`.
- **ID:** UUID 문자열. 순번을 노출하지 않습니다.
- **권한:** 모든 조회는 학원(Organization) 범위를 서버에서 겁니다. 다른 학원 ID로 호출하면 404(존재 여부를 숨김)입니다.
- **페이지:** 목록 API는 커서 방식 `?cursor=...&size=...`을 기본으로 합니다(세부는 첫 목록 API에서 확정).

## 3. 시간 규칙
- 서버·DB·JVM 기본 시간대는 UTC입니다.
- 업무 시각(마감, 리마인드 21:00, 미제출 통보, 보호자 07:00, 만 나이)은 `Asia/Seoul`을 **명시해서** 계산합니다.
- 현재 시각은 주입받은 `Clock`으로 얻습니다(테스트에서 고정 시각으로 바꿈).
- 경계값 테스트: 마감 직전·직후, 생일 전날·당일, 2월 29일생(평년은 3월 1일 기준).

## 4. 로그
- 레벨: 운영 INFO, 로컬 DEBUG.
- **로그에 남기지 않는 것:** 이름, 생년월일, 입력값, 토큰, 사진 주소, SQL 바인딩 값.
- 남기는 것: ID, 동작, 결과, 건수, 소요 시간.

## 5. 테스트
- 백엔드: JUnit 5 + Testcontainers(PostgreSQL 17, 운영과 같은 메이저 버전).
- 테스트 이름은 한글로 행동을 적어도 됩니다(예: `마감_10분_뒤_업로드는_거절한다`).
- 학원 격리·권한은 "다른 학원 ID로 호출하면 실패" 테스트를 반드시 둡니다.
