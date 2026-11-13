# 「지식 그래프 적용 — meeting (사내 Claude 인계 문서)」 코드 대조 검토 (2026-11-13)

대상: 인계 문서(2026-11-13 사외망 작성, D1~D8·§2~§9)
대조: meeting `trans` 브랜치 `e332c08` — `app/models/*.py`, `app/services/knowledge.py`·`events.py`·`closure.py`, 결정 ①·⑦

## 결론 한 줄
방향(그래프 DB 없음·메모리 계산·LLM 제안은 2차·사람 노드 없음·그물 그림 없음)은 맞다. 다만 **이슈 사이 사람 연결은 이미 `block_link` 표에 있어서**, 문서대로 `kg_edge` 에 manual 엣지를 따로 두면 같은 연결이 두 곳에 생긴다. **1차는 새 표 없이 원본 표에서 메모리로 그래프를 만드는 쪽**을 권한다(아래 1).

## §9 점검 항목 — 실제 이름
| 문서 표현 | 실제 | 비고 |
|---|---|---|
| 이슈 표 / 이슈 ID | `block`(block_type `issue`·`problem`), ID `ISS-2026-0142` 문자열 | 공유정보·대시보드 블록은 노드에서 뺄지 정해야 함 |
| 파생 연결 원본 | `block_link(from_block_id, to_block_id, relation)` — 허용 값 `재발·동일원인·관련·참고·파생·합쳐짐`(`events.LINK_RELATIONS`) | **아직 이 표에 쓰는 코드가 없다.** 「새 이슈로 떼어내기」(M15 결정 ⑦)가 들어가야 파생 데이터가 생긴다 → 그래프 G2는 M15 뒤 |
| 해당호선·계통 | `block_tag`(kind `hull`·`system`·`installed_equipment`·`symptom`·`cause` …, `target_id`, `confirmed`·`dismissed`) | dismissed 는 빼고, confirmed 만 쓸지 정해야 함 |
| 담당부서 | `block.owner_department_id` | 이관하면 바뀜 — 과거 시점 질의엔 `block_history` 필요 |
| 선종·자매선 | `hull.ship_type_id`·`hull.sister_group_id` (호선 정본은 중앙 pjtlist) | |
| 회의 회차 | `meeting`(kind `dept`·`share`·`exec`·`hull`·`review`) + 발행 판 `meeting_version` | `dept` 는 부서 자료라 「회의」 노드로 볼지 정해야 함 |
| 다뤄짐 | `report_item_block(item_id, block_id)` + `report_item.block_id` → `report_item.meeting_id` | |
| 종결됨 | `closure(block_id, decision, decided_week_id, decided_at)` — **meeting_id 없음** | 기획 주관자는 회의 밖에서 승인 → 회의 노드로 못 잇는다 (§8 둘째 질문의 답: 주차로 잇거나, 엣지 대신 이슈 상태로) |
| 잠금·트랜잭션 | `app/services/nas_lock.py` — 첫 flush 때 잡고 트랜잭션 끝에 풀림 | D2 그대로 가능 |
| 경로 `modules/…`·`config/settings.py` | 지금은 `app/services/…`·`app/llm/client.py`·`app/settings.py` | 단계 2(전환 지점) 뒤 이름에 맞춰 문서를 고친다 |

## 고칠 것
| # | 문서 위치 | 문제 | 제안 |
|---|---|---|---|
| 1 | D3·§2-2 `관련`(manual)·§3-2 | 사람이 건 이슈 연결은 이미 `block_link`(관련·재발·동일원인·참고)에 있다. `kg_edge` manual 을 따로 두면 두 곳이 어긋난다 | `block_link` 를 원본으로 보고 **그래프는 전부 derived**. 그러면 1차는 `kg_` 표도 재구성 함수도 필요 없다 — 질의 때 원본 표를 읽어 메모리에서 만든다(D4와 같음). 표는 캐시가 느려질 때 더한다 |
| 2 | §2-3 `valid_from`·`valid_to` | derived 엣지는 지우고 다시 쓰므로 이력이 남지 않는다 — 「그때는 어땠나」는 manual 에만 된다. 그런데 manual 원본(`block_link`)엔 날짜 칸이 없다 | 끊기·잇기 기록은 `block_event`(결정 ⑦, 이미 있음)에 kind 를 더해 남긴다. 과거 시점 질의는 2차로 미룬다 |
| 3 | §2-1 노드 종류 | **탑재 장비·장비 모델·증상 코드·원인 코드·자매선이 없다.** CLAUDE.md 원칙 6(이슈 → 탑재 장비 → 모델), 지식창고·장비 허브가 이미 이 축으로 연결한다 | 노드에 `eq:`(탑재 장비)·`model:`·`sym:`·`cause:`·`sister:` 추가. 연관 찾기에서 「같은 모델·같은 원인」이 가장 쓸모 있는 이유 문구다 |
| 4 | §4-2 연관 찾기 | `knowledge.similar_cases()` 가 이미 점수 + 이유(같은 장비 4·같은 모델 3·같은 증상 3·자매선 1·제목 낱말)를 낸다. PPR 은 이를 대신하거나 겹친다 | G4 는 「PPR 로 바꿀지」를 시험으로 정한다 — 같은 픽스처에서 두 방식 상위 5개 비교. 이유 문구는 지금 문구(같은 장비·같은 모델·자매선)를 따른다 |
| 5 | §2-2 `종결됨` | 종결은 회의에 묶이지 않는다(위 표) | 엣지 대신 이슈 노드의 상태(색)로. 회의 화면에서 승인한 건은 `block_event.meeting_id` 로 알 수 있으니 필요하면 거기서 |
| 6 | §5 순서 | 「파생 연결」 데이터가 아직 없다 | G0~G1 은 지금 해도 되지만, G2(파생 나무)는 M15 「새 이슈로 떼어내기」 뒤. ROADMAP 에 의존 표시 |
| 7 | D8 networkx | 맞다(louvain 은 numpy 없음). 다만 1차 G5 를 「계통 × 상태 집계표」로 하면 의존성 0 | §8 질문 그대로 사용자에게 |

## 맞는 것 (그대로)
D1 그래프 DB·별도 파일 금지 · D2 같은 트랜잭션(쓰기를 한다면) · D4 짧게 읽고 메모리 계산 · D5 LLM 제안은 2차·승인제 · D6 사람 노드 없음 · D7 그물 그림 없음(결정 ⑧ 3색과 맞음) · 숫자를 설정으로 · 재귀 CTE 깊이 제한·순환 시험

## 사용자에게 물을 것 (문서 §8 + 위에서 생긴 것)
1. 1차를 **새 표 없이**(원본 표에서 메모리로) 할까, 문서대로 `kg_` 표를 둘까
2. 공유정보·대시보드 블록도 노드로 넣을까 (이슈·문제점만?)
3. AI 가 단 태그는 사람이 확정한 것(`confirmed`)만 쓸까
4. 「회의」 노드에 부서 자료(`dept`)도 넣을까
5. 문서 §8 다섯 가지(노드 가감·종결 기준·보여 줄 대상·networkx·2차 LLM)
