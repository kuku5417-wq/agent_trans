# workinglog — agent 가이드_trans (사외망에서 만들고 사내망으로 전환)

## 2026-11-13 계획
정한 것: 엔진은 shi 플러그인 하나(shi_dev), 이 폴더는 전환용 가이드·점검표·사례만 / GitHub 새 저장소로 관리 / 첫 대상 meeting — 원본(FastAPI, kuku5417-wq/meeting) 기준으로 검토·수정, meeting_v(교육장 Streamlit)는 참고

### 만들 것
- [ ] 0. GitHub 새 저장소 연결 (사용자가 만들고 Claude 앱 권한에 추가) → git init, 첫 커밋
- [x] 1. README.md — 이 묶음이 무엇이고 언제 쓰는지, shi 설치(플러그인 또는 설치_안내)
- [x] 2. 전환_가이드.md — 두 출발점
      A. 새로 만들기: 사외망 `/shi:init` → 명세 → 슬라이스
      B. 이미 만든 것 검토·수정: `/shi:review`(새 스킬) 차이표 → 고칠 순서 → 슬라이스
      공통 뒤: `/shi:migrate-net` → GitHub → 사내망 PC `/shi:init internal` → `사내망_이전.md` A·B → 실데이터 계약 대조 → `/shi:package` → 배포
- [x] 3. 검토_점검표.md — shi 규칙 대비 항목(전환 지점 4곳, config/paths.py, pytest, 127.0.0.1, 데이터 계약, 비밀, 원본 보호, 디자인 토큰)
- [x] 4. 반입_점검표.md — 사내망으로 가져갈 것/가져가지 않을 것, 사내망에서 가져올 것(데이터 계약 — 값 없이)
- [ ] 5. 사례/meeting.md — 원본 meeting 검토 결과와 결정 필요 항목
      - 서버 `0.0.0.0` → `127.0.0.1` 실행 파일 (FastAPI 유지)
      - code_N 공통지침(path_config.py, app_config.py, _smoke.py) → shi 규칙(config/paths.py, pytest)으로
      - 데이터 계약: 읽는 pjtlist·milestone (data_manager) — 사내망 `/shi:datamap` 계약 필요
      - **여러 PC가 같은 SQLite에 쓰게 되는 문제** — 저장 방식 결정 (참고: 교육장 meeting-저장구조.md)
      - LLM 체인(SOLA→Upstage→OpenAI→Mock) → 전환 지점 llm_client 하나로

### shi 플러그인에 더할 것 (agent 가이드 = shi_dev 에서, 0.4.0)
- [ ] `/shi:review` — 기존 프로젝트를 shi 규칙과 대조해 차이표·고칠 순서
- [ ] `/shi:sample` — 데이터 계약으로 사외망 개발용 가짜 데이터 만들기

### 다음 할 일
- 저장소는 나중에 (사용자). 그때까지 로컬 git

## 동시 진행 계획 (2026-11-13)
세 폴더를 각자 작업 줄(레인)로 동시에 진행한다. 저장소는 나중에 만들고, 그때까지 폴더마다 로컬 git으로 커밋한다.

| 레인 | 폴더 | 하는 일 | 다른 레인에 주는 것 |
|---|---|---|---|
| 1 | agent 가이드 (shi_dev) | shi 0.4.0: `/shi:review`, `/shi:sample` | 스킬 이름·입력·출력 (먼저 약속), 0.4.0 판 |
| 2 | agent 가이드_trans | 전환 가이드, 점검표 2종, meeting 사례 | — |
| 3 | agent 가이드_new | make_bundle.py, 동료용 문서 4종, 배포 절차, 안내 페이지 | — |

맞춰 보는 때 (이때만 레인끼리 기다린다)
- **M0 시작 전**: `/shi:review`, `/shi:sample`의 이름·입력·출력을 agent 가이드 `docs/약속_0.4.md`에 먼저 적는다 → 레인 2·3은 이 약속대로 쓴다
- **M1 레인 1 끝**: 0.4.0 커밋 → 레인 2는 가이드의 명령 이름·출력 파일 이름을 대조, 레인 3은 0.4.0으로 zip을 만들어 본다
- **M2 마무리**: 세 폴더 용어·명령 이름·파일 경로 교차 점검, 각 workinglog 갱신

레인끼리 겹치지 않게
- 플러그인 파일은 레인 1만 고친다. 레인 2·3이 플러그인에 바랄 것이 생기면 자기 workinglog "레인 1에 요청"에 적는다
- 레인 2·3은 플러그인 문서를 복사하지 않고 링크(저장소 경로)로 가리킨다
