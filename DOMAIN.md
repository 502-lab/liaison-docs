# 도메인 모델 요약

- **담당:** 개발 총괄. 이름은 [GLOSSARY.md](GLOSSARY.md), 규칙은 [CODE_CONVENTIONS.md](CODE_CONVENTIONS.md)를 따릅니다.
- **상태:** v1.0 스키마 설계(WBS 1.0.2·1.0.3) 전의 요약입니다. 확정되면 ERD와 함께 갱신합니다. (2026-11-04 숙제 세부 규칙·추가 인증 반영)

## 1. 핵심 관계

```txt
User(계정: 로그인 수단·기기 토큰·추가 인증 수단만)
 ├─< TermsConsent(이용약관 동의: 계정 단위)
 ├─< DeviceToken(앱·웹 푸시 대상)
 ├─< PasskeyCredential / TotpSecret(원장·선생님 추가 인증, 계정 단위)
 ├─< AuthApprovalRequest(새 기기·분실 복구·잠금 해제 승인 요청, expires_at = 24시간)
 ├─< Notification(알림함 한 건: 계정 단위, read_at) ─< NotificationItem(소속 id, 숙제 id)
 └─< Membership(소속: 학원 안의 사람, 역할 여러 개, 학생 정보, 상태) >─ Organization(학원 | 과외)
       ├─< MembershipConsent(개인정보 동의 이력: 종류·동의서 버전·방식·증빙·철회)
       ├─< TeacherAssignment(담당 배정: 학생 소속 ↔ 선생님 소속, 한 학생에 여러 명)
       ├─< Homework(숙제: kind, 책임 선생님, due_at, photo_required, canceled_at·canceled_by)
       │     ├── SubmissionSession(제출 세션: started_at, 업로드 만료)
       │     ├── Submission ─< SubmissionPhoto(제출 사진 1~10장: delete_after)
       │     ├── WorksheetFile(학습지 파일, v1.3: 마감 + 30일 삭제)
       │     └── PerformanceRecord(수행 기록: 값·상태) ─< 수정 이력
       ├─< TextbookProgress(학생별·교재별 시작점 base_page)
       ├─< ExamScore(시험 점수, 선택 동의한 학생만)
       └─< TeacherMemo / CounselingNote(작성자 + 원장만 열람)
Textbook(교재: 모든 학원 공유, edition_group_id, merged_into_id) ─< TextbookChapter(단원명·시작 페이지)
TextbookMerge(중복 교재 병합 기록, v1.3)
Invitation(1회용 초대: token_hash, first_invited_at)
AccessLog(접근 기록: ID만, 1년 이상) / PermissionChangeLog(권한 변경·승인 기록: ID만, 3년)
```

- 사람을 가리키는 외래 키는 모두 **소속(Membership) id**입니다. 학생 정보(이름·학년·생년월일·학교명)는 계정이 아니라 소속에 있습니다.
- 학원 간 격리: 모든 학원 데이터는 소속 → 학원으로 이어지고, 조회는 학원 범위로만 합니다.
- **예외: 알림함(`Notification`)은 계정 단위입니다.** 학원이 달라도 같은 시각의 알림을 1건으로 묶기 때문입니다. 학원별 연결은 `NotificationItem`(소속 id, 숙제 id)에 둡니다. 학생 소속이 삭제되면 그 학원의 항목도 지우고, 항목이 남지 않은 알림은 함께 지웁니다. 원장·선생님은 학생의 알림함을 볼 수 없습니다.

## 2. 숙제

| 항목 | 규칙 |
|---|---|
| 형태 | `HomeworkKind`: `TEXTBOOK`(교재 + 페이지 범위 page_start~page_end) / `FREE`(내용 텍스트 content_text). 학습지는 자유형 숙제의 내용일 뿐이라 따로 구분하지 않습니다 |
| 날짜 | 숙제의 날짜는 **마감일**입니다(`due_at`, 한국 시간 기준 날짜). 학원 기본 마감 변경은 앞으로 낼 숙제에만 적용합니다 |
| 사진 | `photo_required`(기본 켬, 선생님이 끌 수 있음). 끄면 그 숙제는 리마인드·미제출 통보·미제출 횟수에서 빠지고, 수행 기록을 미리 채우지 않습니다(선생님이 직접 기록, 미기록 알림·기록률에는 포함). 제출은 숙제 1건에 사진 1~10장이고, 사진 한 장을 여러 숙제에 묶지 않습니다. 다시 내면 **사진 묶음 전체를 교체**하고 이전 사진은 즉시 지웁니다. 부여 뒤 마감 전이면 미리 낼 수 있습니다 |
| 사진 메타데이터 | 앱에서 촬영 위치·시각·기기 정보(EXIF)를 모두 지운 뒤 올리고, 서버는 위치 정보가 남은 이미지를 거절합니다(L-04). 학습지 PDF는 문서 정보를 지웁니다 |
| 부여 | 초안은 서버에 저장하지 않습니다. 진도 자동 생성은 저장하지 않는 계산 API가 하고, 웹은 결과를 화면 메모리에만 둡니다. [부여]하면 학생마다 숙제가 한 트랜잭션으로 만들어지고 학생마다 알림 1건이 갑니다 |
| 수정 | 책임 선생님만 **마감 전까지** 고칩니다(페이지·내용·마감 시각·사진 제출 필요). 새 마감은 지금보다 뒤, 09:00~23:59 |
| 취소 | 언제든(기록 확정 뒤에도) 책임 선생님이 합니다. 지우지 않고 `canceled_at`·`canceled_by`(소속 id)를 남깁니다. 상태 enum은 두지 않습니다. 이행률·기록률·미제출 횟수·주의 판정·미기록 알림에서 모두 빼고(`canceled_at IS NULL`), 제출 사진은 즉시 지웁니다. 되돌릴 수 없습니다 |

## 3. 진도
- 진도는 저장하지 않고 **계산**합니다: 진도 = max(`base_page`, 취소되지 않은 그 학생·그 교재 숙제의 마지막 페이지). `TextbookProgress`에는 `last_page` 칸을 두지 않습니다(캐시도 없음). (학생, 교재) 인덱스로 바로 계산되므로 필요해지면 그때 캐시를 검토합니다.
- `base_page`는 첫 숙제를 낼 때 적은 "시작 페이지 − 1"이며 `TextbookProgress`(학생 소속 × 교재)에 저장합니다.
- 그래서 숙제를 취소하거나 마감 전에 페이지를 고치면 진도가 저절로 맞습니다. "이어서 N일치"는 진도 다음 페이지부터 만듭니다.
- 총 페이지 수는 본문 마지막 쪽 번호입니다. 넘으면 막지 않고 경고하며, 마지막 페이지까지만 만듭니다.

## 4. 상태 값

| 대상 | enum | 값 |
|---|---|---|
| 역할 | `MembershipRole` | `DIRECTOR`, `TEACHER`, `STUDENT`, `GUARDIAN` (소속당 여러 개) |
| 소속 상태 | `MembershipStatus` | `INVITED` → `PENDING_APPROVAL` → `ACTIVE` / `SUSPENDED` / `ENDED` |
| 종료 사유 | `MembershipEndReason` | `CONSENT_WITHDRAWN`, `LEFT`, `ACADEMY_TERMINATED`, `PILOT_ENDED`, `RECONSENT_EXPIRED` |
| 숙제 형태 | `HomeworkKind` | `TEXTBOOK`, `FREE` |
| 수행 기록 값 | `PerformanceResult` | `COMPLETE`, `PARTIAL`, `INCOMPLETE`, `EXEMPT` |
| 수행 기록 상태 | `RecordState` | `PROPOSED`(시스템이 미리 채움), `CONFIRMED`(선생님 확정). "미기록" = 확정되지 않음 |
| 제출(API용) | `SubmissionStatus` | `NOT_SUBMITTED`, `SUBMITTED` |
| 동의 종류 | `ConsentType` | `REQUIRED`, `GUARDIAN_VIEW`, `EXAM_SCORE`, `GUARDIAN_CONSENT` |
| 동의 방식 / 입력 주체 | `ConsentMethod` / `ConsentActor` | `ELECTRONIC`, `WRITTEN` / `SELF`, `DIRECTOR`, `OPERATOR` |
| 승인 요청 종류 / 상태 | `AuthApprovalReason` / `AuthApprovalStatus` | `NEW_DEVICE`, `RECOVERY`, `UNLOCK` / `PENDING`, `APPROVED`, `REJECTED`, `EXPIRED` |
| 본인 확인 방법 | `VerificationMethod` | `RECORD_CHECK`, `PHONE`, `IN_PERSON` (기기·복구·해제는 `PHONE`·`IN_PERSON`만) |
| 학생과의 관계 | `GuardianRelation` | `FATHER`, `MOTHER`, `GRANDPARENT`, `LEGAL_GUARDIAN`, `OTHER` |
| 과목 | `Subject` | `KOREAN`, `ENGLISH`, `MATH`, `SCIENCE`, `SOCIAL`, `ETC` |
| 학원 종류 | `OrganizationType` | `ACADEMY`, `TUTOR` |
| 요금제 | `Plan` | `FREE`, `BASIC`, `PREMIUM` |
| 알림 채널 | `NotificationChannel` | `DEFAULT`, `SILENT`(iOS passive, Android 중요도 낮음) |

## 5. 시간·알림 규칙 요약
- 마감 허용 범위는 09:00~23:59(한국 시간)입니다.
- 마감 전에 시작한 제출은 마감 + 10분까지 인정합니다(서버 시각 기준, `SubmissionSession.started_at`).
- 미리 채우기와 학생 미제출 통보(무음)는 마감 + 10분에, 리마인드는 min(21:00, 마감 − 1시간)에 보냅니다. 이미 지난 리마인드는 건너뜁니다. **이미 리마인드를 보낸 숙제는 마감을 바꿔도 다시 보내지 않습니다.**
- 알림은 매분 현재 숙제 상태로 다시 계산해서 보냅니다. 그래서 취소된 숙제의 알림은 따로 지우지 않아도 나가지 않습니다.
- **알림 묶기:** 같은 학생(계정)에게 같은 분에 보낼 알림은 학원이 달라도 1건입니다. 부여·수정·취소는 선생님이 한 번에 한 일당 학생마다 1건입니다. 보호자 07:00 통보는 보호자 계정마다 1건입니다. 본문에는 학원명·과목·이름을 넣지 않습니다.
- 삭제 규칙은 [PRIVACY_RULES.md](PRIVACY_RULES.md)를 따릅니다.
