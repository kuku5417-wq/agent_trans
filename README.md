# agent_trans — 사외망에서 만들고 사내망으로 넘기기

아직 사내망에 올리지 못한 앱(첫 대상: meeting)을 **사외망에서 새로 만들거나, 이미 만든 것을 검토·수정해서** 사내망으로 넘길 때 쓰는 묶음이다.

| 이럴 때 | 이 묶음 대신 |
|---|---|
| 사내망에서 이미 돌고 있는 앱을 다시 짓는다 | shi_dev의 [docs/사내망_적용.md](https://github.com/kuku5417-wq/kuku5417-wq-shi_dev/blob/main/docs/사내망_적용.md) |
| 사내망만 쓰는 동료에게 나눠 준다 | [agent_new](https://github.com/kuku5417-wq/agent_new) |

## 저장소 지도 (네 저장소가 하는 일)

| 저장소 | 이 PC 폴더 | 하는 일 |
|---|---|---|
| [kuku5417-wq-shi_dev](https://github.com/kuku5417-wq/kuku5417-wq-shi_dev) | `JSH\agent 가이드` | **엔진.** 공통 플러그인 shi(에이전트 8·스킬 10·가드 훅·틀)를 만들고 판(0.4.0 …)을 낸다. 규칙을 바꾸는 곳은 여기 하나 |
| [agent_trans](https://github.com/kuku5417-wq/agent_trans) | `JSH\agent 가이드_trans` | **사외망 → 사내망 넘기기.** 사외망에서 새로 만들거나 이미 만든 앱을 shi 규칙으로 검토·수정해 사내망으로 옮기는 가이드·점검표·사례. 첫 사례 meeting(결정 ①~⑧) |
| [agent_new](https://github.com/kuku5417-wq/agent_new) | `JSH\agent 가이드_new` | **사내망 동료 배포.** GitHub 없는 사내망 개발 동료에게 shi를 zip(NAS·아티팩트)으로 나눠 주는 문서와 `make_bundle.py` |
| [meeting](https://github.com/kuku5417-wq/meeting) | (본인 PC `F:\code\_원본\meeting`) | **원본 보관.** 커미셔닝팀 회의자료 앱(FastAPI) 원본. 고치지 않는다 — 전환 작업은 별도 브랜치·작업본에서 |

흐름: shi_dev가 판을 내면 → agent_trans가 그 판으로 앱을 넘기고, agent_new가 그 판을 동료에게 나눠 준다.

## 들어 있는 것

| 파일 | 내용 |
|---|---|
| [전환_가이드.md](전환_가이드.md) | 순서 전체: 출발(새로 / 이미 만든 것) → 사외망 개발 → 넘기기 → 사내망 마무리 → 배포 |
| [검토_점검표.md](검토_점검표.md) | 이미 만든 앱을 shi 규칙과 대조할 때 사람이 같이 보는 표 (`/shi:review` 결과를 읽는 법) |
| [반입_점검표.md](반입_점검표.md) | 사내망으로 가져갈 것 / 가져가지 않을 것, 사내망에서 가져올 것 |
| [사례/meeting.md](사례/meeting.md) | 원본 meeting(FastAPI) 검토 결과와 정해야 할 것 |
| workinglog.md | 진행 기록 |

## 필요한 것

- shi 플러그인 **0.4.0 이상** — 설치는 shi_dev README (`/plugin` 이 안 되면 설치_안내.md)
- 사외망 PC: 상위 폴더 `F:\code`, git, uv, Node.js, Python, Claude 데스크톱 앱
- 사내망 PC: 같은 shi 판, 그 앱이 읽을 데이터의 계약(`/shi:datamap` 결과)

에이전트·스킬·훅은 이 묶음에 복사하지 않는다. 모두 shi 플러그인 하나에서 온다.
