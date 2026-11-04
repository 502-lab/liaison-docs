# 도메인 모델 요약

- **담당:** 개발 총괄. 이름은 [GLOSSARY.md](GLOSSARY.md), 규칙은 [CODE_CONVENTIONS.md](CODE_CONVENTIONS.md)를 따릅니다.
- **상태:** v1.0 스키마 설계(WBS 1.0.2·1.0.3) 전의 요약입니다. 확정되면 ERD와 함께 갱신합니다.

## 1. 핵심 관계

```txt
User(계정: 로그인 수단·기기 토큰만)
 └─< Membership(소속: 학원 안의 사람, 역할 여러 개, 학생 정보, 상태) >─ Organization(학원 | 과외)
       ├─< MembershipConsent(개인정보 동의 이력: 종류·동의서 버전·방식·증빙·철회)
       ├─< TeacherAssignment(담당 배정: 학생 소속 ↔ 선생님 소속, 한 학생에 여러 명)
       ├─< Homework(숙제: 책임 선생님, 교재·페이지, due_at)
       │     ├── SubmissionSession(제출 세션: started_at, 업로드 만료)
       │     ├── Submission ─< SubmissionPhoto(제출 사진: delete_after)
       │     └── PerformanceRecord(수행 기록: 값·상태) ─< 수정 이력
       ├─< TextbookProgress(학생별·교재별 마지막 페이지)
       ├─< ExamScore(시험 점수, 선택 동의한 학생만)
       ├─< TeacherMemo / CounselingNote(작성자 + 원장만 열람)
       └─< Notification(알림함 한 건, read_at)
User ─< TermsConsent(이용약관 동의: 계정 단위)
User ─< DeviceToken(앱·웹 푸시 대상)
Textbook(교재: 모든 학원 공유, edition_group_id) ─< TextbookChapter(단원명·시작 페이지)
Invitation(1회용 초대: token_hash, first_invited_at)
AccessLog(접근 기록: ID만, 학생 삭제 후에도 보관)
```

- 사람을 가리키는 외래 키는 모두 **소속(Membership) id**입니다. 학생 정보(이름·학년·생년월일·학교명)는 계정이 아니라 소속에 있습니다.
- 학원 간 격리: 모든 학원 데이터는 소속 → 학원으로 이어지고, 조회는 학원 범위로만 합니다.

## 2. 상태 값

| 대상 | enum | 값 |
|---|---|---|
| 역할 | `MembershipRole` | `DIRECTOR`, `TEACHER`, `STUDENT`, `GUARDIAN` (소속당 여러 개) |
| 소속 상태 | `MembershipStatus` | `INVITED` → `PENDING_APPROVAL` → `ACTIVE` / `SUSPENDED` / `ENDED` |
| 종료 사유 | `MembershipEndReason` | `CONSENT_WITHDRAWN`, `LEFT`, `ACADEMY_TERMINATED`, `PILOT_ENDED`, `RECONSENT_EXPIRED` |
| 수행 기록 값 | `PerformanceResult` | `COMPLETE`, `PARTIAL`, `INCOMPLETE`, `EXEMPT` |
| 수행 기록 상태 | `RecordState` | `PROPOSED`(시스템이 미리 채움), `CONFIRMED`(선생님 확정). "미기록" = 확정되지 않음 |
| 제출(API용) | `SubmissionStatus` | `NOT_SUBMITTED`, `SUBMITTED` |
| 동의 종류 | `ConsentType` | `REQUIRED`, `GUARDIAN_VIEW`, `EXAM_SCORE`, `GUARDIAN_CONSENT` |
| 동의 방식 / 입력 주체 | `ConsentMethod` / `ConsentActor` | `ELECTRONIC`, `WRITTEN` / `SELF`, `DIRECTOR`, `OPERATOR` |
| 과목 | `Subject` | `KOREAN`, `ENGLISH`, `MATH`, `SCIENCE`, `SOCIAL`, `ETC` |
| 학원 종류 | `OrganizationType` | `ACADEMY`, `TUTOR` |
| 요금제 | `Plan` | `FREE`, `BASIC`, `PREMIUM` |
| 알림 채널 | `NotificationChannel` | `DEFAULT`, `SILENT`(iOS passive, Android 중요도 낮음) |

## 3. 시간 규칙 요약
- 마감 `due_at`은 숙제마다 저장합니다(학원 기본 마감 변경은 앞으로 낼 숙제에만). 마감 허용 범위는 09:00~23:59(한국 시간)입니다.
- 마감 전에 시작한 제출은 마감 + 10분까지 인정합니다(서버 시각 기준, `SubmissionSession.started_at`).
- 미리 채우기와 학생 미제출 통보(무음)는 마감 + 10분에, 리마인드는 min(21:00, 마감 − 1시간)에 보냅니다. 이미 지난 리마인드는 건너뜁니다.
- 삭제 규칙은 [PRIVACY_RULES.md](PRIVACY_RULES.md)를 따릅니다.
