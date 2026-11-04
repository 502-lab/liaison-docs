# liaison-docs

Liaison의 **공통 문서 저장소**입니다. `liaison-app`, `liaison-back`, `liaison-web`이 `docs/common/`(git submodule)으로 이 저장소를 연결하고, 각 저장소의 `CLAUDE.md`·`AGENTS.md`가 아래 문서를 자동으로 불러옵니다.

> ⚠ **공개 저장소입니다.** 실제 학생 데이터, 비밀키, 법무 원문(PIA·처리방침 초안·위탁 문서·내부 관리계획·유출 대응 절차), 탐지 기준, 운영자 접근 경로, 서버 주소·계정·업체 계약 정보는 **올리지 않습니다.**

## 문서

| 문서 | 내용 | 담당 |
|---|---|---|
| [AI_RULES.md](AI_RULES.md) | AI가 작업 전에 반드시 지킬 규칙(가장 먼저 읽기) | 개발 총괄 |
| [GLOSSARY.md](GLOSSARY.md) | 용어집 — 표준어·코드 이름·뜻 | 기획(PM) |
| [CODE_CONVENTIONS.md](CODE_CONVENTIONS.md) | 코드 이름 규칙, API 규칙(응답·오류·시간·ID) | 개발 총괄 |
| [GIT_CONVENTIONS.md](GIT_CONVENTIONS.md) | 커밋·브랜치·PR 규칙 | 개발 총괄 |
| [DOMAIN.md](DOMAIN.md) | 도메인 모델 요약(엔티티 관계, 상태 값) | 개발 총괄 |
| [PRIVACY_RULES.md](PRIVACY_RULES.md) | 개발자가 지킬 개인정보 규칙 요약 | 법무 |

## 고치는 방법
1. 이 저장소에 PR을 올립니다. 문서 담당 세션(위 표)의 확인을 받습니다.
2. 병합되면 각 저장소에 Dependabot이 "docs/common 버전 갱신" PR을 자동으로 만듭니다. 그 PR을 병합하면 반영됩니다.
3. 급하면 각 저장소에서 `git submodule update --remote docs/common` 후 커밋해도 됩니다.

## 각 저장소에서 처음 받을 때
```bash
git clone --recursive https://github.com/502-lab/liaison-back.git
# 이미 받았다면
git submodule update --init
```
