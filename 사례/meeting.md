# 사례 — meeting (원본 FastAPI) 검토

> 대상: `kuku5417-wq/meeting` main `b625e53` (원본 zip 그대로). 교육장 `meeting_v`(Streamlit)는 참고만.
> 이 문서는 사람이 읽고 정하기 위한 **사전 검토**다. 실제 `review.md`는 사외망 작업본에서 `/shi:review`로 다시 만든다.
> 2026-11-13 작성 · 코드를 직접 연 곳만 적었다.

## 1. 지금 모습

| 항목 | 원본 |
|---|---|
| 화면 | FastAPI + Jinja2 + 바닐라 JS, 사진 편집 Konva.js. 앱 코드 약 8,900줄 |
| 실행 | `run.bat` — 운영 PC 한 대가 `uvicorn --host 0.0.0.0 --port 8600`, 7개 부서가 브라우저로 접속 |
| 데이터 | `<NAS_BASE_PATH 또는 DATA_PATH>\meeting\` — `meeting.db`(SQLite, 표 40여 개), `photos\`, `exports\` |
| 다른 앱 데이터 | 중앙 parquet **pjtlist·milestone을 읽기만** (`app/services/central.py`, 생산자는 data_manager_F) |
| LLM | `app/llm/client.py` 폴백 체인 `sola → upstage → openai → mock` |
| 메일·DRM | 서버 PC의 클래식 Outlook·Excel(COM)로 읽음 (`outlook_client.py`, `drm_reader.py`), 없으면 쉼 |
| 경로·비밀 | `path_config.py` 한 곳, `secret/.env` 위로 올라가며 탐색 |
| 망 | `app_config.py`(전 repo 동일본)가 망을 자동 감지 |
| 테스트 | `_smoke.py` 검사 약 420개 (pytest 아님) |
| 사내망 | **아직 올리지 않음** — 사내망에 쌓인 meeting 데이터가 없다 |

## 2. shi 규칙과 차이 (예비 차이표)

| 규칙 | 지금 | shi | 위치 | 크기 |
|---|---|---|---|---|
| R1 경로 | `path_config.py` 한 곳 (이미 창구) | `config/paths.py` | `path_config.py` | S (옮기고 이름만) |
| R2 비밀 | `path_config.read_secret`, `app_config` | `config/settings.py` | 같은 곳 | S |
| R3 LLM | `app/llm/client.py` 한 곳, 4단 체인 | `modules/llm_client.py`, 사외망 OpenAI·사내망 SOLA | `app/llm/client.py` | M (체인 → 망별 하나) |
| R4 NAS | `_nas_base()`가 경로만 확인, 접속 처리 없음 | `modules/nas.py` (도메인 → 실패 시 입력창) | `path_config.py:74` | M |
| R5 망 감지 | `app_config.py` 자동 감지 | `.claude/shi.json`의 `net` | `app_config.py`, `_netcheck.py` | M |
| R6 서버 | **`0.0.0.0`** (정책 위반) | `127.0.0.1` 실행 파일 | `run.bat:20`, `setup_env.bat:121` | S |
| R7 테스트 | `_smoke.py` 약 420개 | pytest | `_smoke.py` | M (옮기기 — 지금 동작을 고정하는 귀한 자산) |
| R8 계약 | pjtlist·milestone 읽기 | 그대로, 계약으로 고정 | `app/services/central.py` | S (사내망 계약 필요) |
| **R9 동시 쓰기** | 서버 한 대가 SQLite 하나를 씀 | 각자 PC 실행 → **7개 부서 PC가 NAS의 SQLite 하나에 쓰게 됨** | `app/db.py`, `path_config.db_path` | **L** |
| R10 원본 보호 | 메모 새 판·사진 좌표 JSON — 이미 지킴 | 같음 | CLAUDE.md 원칙 1 | — |
| R11 비밀 | `.gitignore`에 `.env`·`*.db` | 같음 | | — |
| R12 디자인 | `design/tokens.css` 있음 | 토큰 유지 + 빠진 것 채움 | | S |
| R13 에이전트 | `.claude/commands/` (`/next`, `/check-mockup`) | code-gen, spec-guard, `requirements.md` | | M |
| R14 의존성 | `pyproject.toml` + `uv.lock` | 같음 | | — |

**메일·DRM**: 서버 PC에서 읽던 것이 각자 PC에서 읽게 된다. 각자 PC의 Outlook·Excel이면 동작은 하지만 "누구의 메일함을 읽나"가 바뀐다 → 정해야 할 것 3.

## 3. 고쳐 쓸까, 다시 지을까

검토 점검표 3절 기준: L이 1개(R9), 테스트가 많다 → **고쳐 쓰기**를 권한다. 화면(FastAPI·Konva 편집기)은 그대로 쓴다.

## 4. 정해야 할 것

### ① 저장 방식 (R9) — 가장 큰 결정
meeting은 **사내망에 쌓인 데이터가 아직 없다.** 그래서 저장 방식을 지금 바꿔도 옮길 데이터가 없다. 사내망에 올리기 전에 정하는 게 가장 싸다.

| 안 | 내용 | 장점 | 대가 |
|---|---|---|---|
| A 잠금 + 한 번에 한 PC | NAS의 SQLite는 그대로, 쓰기 전에 잠금 파일을 잡은 PC만 쓰고 나머지는 읽기만 | 코드 변경이 가장 적다 | 마감 때 7개 부서가 몰리면 기다림. 잠금이 남는 경우 처리. NAS 위 SQLite 읽기도 완전히 안전하지는 않음 |
| B 부서별 DB + 모으기 | 부서마다 자기 SQLite에 쓰고, 기획 주관자 PC가 회의자료를 모음 | 쓰기 충돌이 없다 | 부서를 넘는 일(이관, 회의 진행·종결)을 다시 설계 |
| C 파일 단위 저장 | 이슈 하나 = 폴더 하나, 덧붙이기, 각 PC SQLite는 검색 캐시 (교육장 `meeting-저장구조.md`) | NAS에 가장 안전, 이력 보존 | 데이터 층을 다시 짬 (표 40여 개). 일이 가장 크다 |

**결정 (2026-11-13): A안을 보강해서 쓴다.** NAS에 SQLite 하나, 쓰기는 NAS 잠금 파일로 한 번에 한 PC.

B·C를 고르지 않은 이유 — 원본 코드가 "DB 하나, 쓰기는 한 줄로"를 전제로 한다.
- 24개 표가 정수 번호를 자동으로 매긴다(`app/models/*`)
- `ISS-2026-0142` 같은 번호·순서 번호를 "가장 큰 값 + 1"로 만든다(`services/writing.py`·`metrics.py`·`tables.py`·`studio.py`)
- PC·부서마다 따로 쓰고 합치면 번호가 겹친다 → 번호 체계를 표 40여 개에 걸쳐 다시 짜야 한다

보강 6가지 (고칠 곳은 `app/db.py` 주변)

| # | 내용 | 이유 |
|---|---|---|
| 1 | 쓰기 전에 NAS `meeting.lock`(PC 이름·시각)을 단독 생성, 저장 뒤 지움. 2분 넘은 잠금은 주인 없는 것으로 정리 | SMB 위 SQLite 파일 잠금은 믿을 수 없다 → 앱이 막는다 |
| 2 | 잠금은 저장하는 순간에만. AI 정리·PPT·사진 처리는 잠금 밖 | 대기 1초 안쪽, 마감 때도 거의 안 기다림 |
| 3 | `journal_mode=DELETE`, `busy_timeout`. WAL 금지 | WAL은 네트워크 드라이브에서 안 됨 |
| 4 | 블록·이슈에 판 번호. 저장 때 판이 달라졌으면 "다른 사람이 먼저 고쳤어요. 새로 불러오세요" | 동시에 고쳐도 조용히 덮이지 않음 |
| 5 | 매일 첫 실행에 백업 사본 + `integrity_check`. 깨지면 알리고 읽기 전용 | 손실을 하루 안으로 |
| 6 | 사진·내보내기 파일은 겹치지 않는 이름, 덮어쓰지 않음 (지금 방식 유지) | 이미 안전 |

남는 위험: 저장 순간에 다른 PC가 읽으면 드물게 "DB 읽기 실패" → 다시 시도로 넘김.
자주 생기면 **2단계**: 읽기는 내 PC 사본, 저장만 NAS(번호 체계는 그대로). 마감 동시 사용이 10~20명을 넘으면 2단계를 처음부터 넣는다.
옮길 사내망 데이터는 없다.

### ② 테스트 옮기기
`_smoke.py` 420개를 pytest로 옮기는 것이 고칠 순서 1번이다. 옮기는 동안 원본 동작이 바뀌지 않았는지가 이것으로 확인된다. 그대로 두고 pytest가 `_smoke.py`를 부르게 하는 방법도 있다(작게 시작).

**결정 (2026-11-13): shi 규칙대로 pytest로 옮긴다.** `_smoke.py`는 옮기기가 끝나면 지운다(원본 저장소에는 그대로 남음).

옮기는 방법 — 원본 구조(2,276줄, 검사 422개, `t_*` 함수 27개)
- `tests/conftest.py`: `_smoke.py` 맨 위 환경 주입(임시 데이터 폴더, 오늘 날짜 2026-10-23 고정, Outlook 끔, 키 비움)을 **app import 전에** 그대로 옮기고, `TestClient` 하나를 세션 범위 fixture로 둔다
- `t_*`는 앞 검사가 만든 데이터를 이어 쓴다 → 순서를 지킨다. 영역별로 번호 붙인 파일(`test_01_기본.py`, `test_02_작성.py` …)로 나누고, 파일 안 순서도 원본 순서 그대로
- `check(이름, 조건)`는 한 함수 안에서 여러 개 → fixture `check`가 실패를 모았다가 함수 끝에서 한 번에 assert. 검사 이름 422개가 실패 메시지에 그대로 남는다
- 화면 문구에 의존하는 검사는 **계약**이라는 원본 주석을 유지
- 완료 기준: pytest 통과 개수 = 원본 `_smoke.py` PASS 422개. 그 뒤에 다른 슬라이스(경로·저장 방식 등)를 시작한다

### ③ 메일·DRM을 누구 PC에서
- 각자 자기 Outlook을 읽게 한다 / 기획 주관자 PC에서만 가져오기를 연다 / 1차에서 끈다.

**결정 (2026-11-13): 1차 판에서는 끈다.**
- 끄는 것: 메일 가져오기(`services/outlook_client.py`)와 DRM 엑셀 열기(`services/drm_reader.py`의 Excel COM). 둘 다 "서버 PC 한 대"를 전제로 했다
- 남기는 것: PPT·HTML·PDF·일반 엑셀 가져오기(파일을 직접 올리는 방식)
- 방법: 설정 하나(`imports.com = off`, 기본 off)로 두 기능을 휴면. 메일 탭은 "1차 판에서는 메일 가져오기를 쓰지 않습니다"로 안내. DRM 엑셀은 "DRM을 풀고 저장한 뒤 올려 주세요"로 안내
- 코드는 지우지 않는다 → 2차에서 각자 PC Outlook 또는 기획 주관자 PC만 켜기 중 다시 정함. 실행 파일에 pywin32를 넣지 않아도 되어 크기·검사도 줄어든다
- 테스트: 원본도 `MEETING_NO_OUTLOOK`로 휴면을 시험한다 → 같은 검사가 "꺼진 상태 안내"를 확인하도록 유지

### ④ LLM 체인
사외망 판은 OpenAI 하나, 사내망 판은 SOLA 하나로 단순화할지(전환 지점 한 곳), 지금처럼 체인을 둘지. 원칙 9(보고 내용은 사내 LLM으로만)와 함께 정한다.

**결정 (2026-11-13): SOLA와 GPT 두 개만 둔다.** 체인 `sola,openai` (Upstage 뺌)
- 창구는 전환 지점 `modules/llm_client.py` 한 곳. 순서는 설정 `MEETING_LLM_CHAIN`
- Mock은 사용자 체인에서 빼고 **테스트에서만** 쓴다(pytest가 키 없이 돌아가야 함)
- 둘 다 실패하면: 화면에 공급자별 실패 사유를 보여 주고, 블록은 사람이 직접 쓰게 한다(메모 원문은 그대로 저장)
- ⚠ 원칙 9: GPT는 외부 LLM이다. 원본의 "테스트 단계 예외"(실제 회사 메모는 넣지 않음)를 그대로 이어 간다. 실제 메모를 넣기 전에 사내망 판 체인을 `sola` 하나로 줄일지 다시 정한다

### ⑤ code_N 공통지침 동일본
`app_config.py`·`_netcheck.py`는 "전 repo 동일본, 수정 금지"다. meeting만 shi 규칙으로 바꾸면 동일본에서 벗어난다. 다른 사내망 앱도 shi로 옮길 계획이므로 벗어나도 되는지 확인한다.

**결정 (2026-11-13, 추천안으로 진행 — 바꿀 수 있음): 벗어난다.**
- meeting 작업본에서 `app_config.py`·`_netcheck.py`를 빼고 shi 전환 지점으로 대신한다: `.env` 읽기·프록시는 `config/settings.py`, 망은 `.claude/shi.json`의 `net` 하나(자동 감지 없음)
- 지금 쓰는 곳: `app/settings.py`(import 부수효과로 .env 로드), `check_env.py`, `setup_env.bat`(설치 때 망 감지)
- 원본 저장소와 다른 앱의 동일본은 건드리지 않는다. 다른 앱도 shi로 옮길 때 같은 방식으로 뺀다 → 결국 동일본이 shi 전환 지점으로 바뀐다

### ⑥ 화면 색 (사이드바·주 색)
**결정 (2026-11-13 오전, 교육장 meeting_v에서 정한 것을 원본에도 적용): 흰 사이드바, 주 색 `#0963c4`.**
- 원본 상태: 시안 `v2m-*`와 `tokens.css`(design/·app/static/css/ 두 벌)는 **네이비 사이드바** `--side-bg #1c2f4f`, 큰 제목 남색. 주 색은 tokens.css가 이미 `#0963c4`(시안 `v2m.css`만 `#0a6cd6`)
- 바꿀 값 (라이트): `--side-bg #ffffff`, `--side-fg #111114`, `--side-muted #48484d`, 오른쪽 경계선 `--sep`
  - 지금 메뉴: 바탕 `--accent-soft`, 글자 `--accent-ink`, 왼쪽 막대 `--accent`
  - 큰 제목(h1): `--fg`
  - 다크 테마 사이드바 값은 작업본에서 대비(4.5:1 이상)를 재고 정한다
- 화면 배치·문구는 v2m 그대로. 시안 파일은 고치지 않고, 작업본 `docs/디자인_규칙.md`에 "사이드바 색은 정한 것을 따른다"로 적는다
- 언제: 슬라이스 중 화면 첫 슬라이스(또는 `/shi:design`)에서 tokens.css 두 벌을 함께 고치고 ui-reviewer로 확인

## 5. 진행 순서 (전환 가이드 기준, B 출발)

| # | 할 일 | 비고 |
|---|---|---|
| 0 | 사내망: `/shi:datamap` → pjtlist·milestone 계약 가져오기 | 값 없이 |
| 1 | 사외망: `_원본\meeting` + 작업본 `meeting`(브랜치 `trans`), `/shi:init` (stack fastapi) | |
| 2 | `/shi:inventory` → `/shi:review` → 이 문서와 대조 | |
| 3 | ①~⑤ 결정 → `requirements.md` 결정 기록 | 결정 끝(2026-11-13, 이 문서 4절 ①~⑥). 작업본 만들 때 옮겨 적기 |
| 4 | 슬라이스: 테스트 옮기기 → 경로·비밀·LLM 창구 → 망 감지 제거 → NAS 접속 → 저장 방식 → 서버 주소·실행 파일 | `/shi:sample`로 pjtlist·milestone 가짜 데이터 |
| 5 | `/shi:migrate-net` → GitHub → 사내망 `meeting_F` | |
| 6 | 사내망: 실데이터 계약 대조, **두 PC 이상 동시 사용 시험** | store-reviewer |
| 7 | `/shi:package` → 부서별 시범 | |
