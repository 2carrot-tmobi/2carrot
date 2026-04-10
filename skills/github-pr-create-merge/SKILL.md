---
name: github-pr-create-merge
description: Create and optionally merge GitHub pull requests via GitHub MCP for deployment branches (`ci/dev`, `ci/stg`), review branches (`feature/{ticket}` -> `feature-base/{ticket}`), and operation-release branches (`feature-base/{ticket}` -> `rc/{yyyymmdd}-1`), then update the deployment wiki for operation-release flow.
---

# GitHub PR Create

GitHub PR 작업은 GitHub MCP 우선으로 처리한다. GitHub 직접 API 호출(`curl`, `gh api`)은 기본 경로로 사용하지 않는다. 운영 배포 준비 모드(C)의 위키 업데이트는 `mcp__mcp_atlassian__updateConfluencePage`보다 Confluence REST API 직접 호출을 우선 사용한다. 이때 구현 가이드는 별도 런타임 의존을 늘리지 않도록 `bash + curl` 중심으로 본다. 운영 배포 준비 모드(C)는 `rc/{yyyymmdd}-1 -> main` PR 확인과 배포 위키 업데이트까지 포함한다.
모든 브랜치 상태, source/target compare 내역, PR 중복 확인은 반드시 원격 저장소 기준으로 판단한다. 로컬 저장소의 브랜치, fetch 상태, working tree 상태는 판단 근거로 사용하지 않는다.

## Inputs

- Repository: `owner/repo` (required)
- Ticket number: `TICKET-123` 형식 (required)
- 모드 A, 배포용: `target=dev|ci/dev|stg|ci/stg`
- 모드 B, 코드리뷰용: `ticket`만 입력
- 모드 C, 운영 배포 준비용: `ticket + date(yyyymmdd)`
- 사용자가 복사한 브랜치 요약에 `oldsha..newsha` 형식 범위가 섞여 있어도 입력 정규화 단계에서 버린다
- 선택 입력: PR body, draft 여부, merge method(기본 `merge`)

## Workflow

1. 저장소 입력을 정규화한다.
   - 앞 `/` 제거
   - `.git` suffix 제거
   - 복사된 줄에 포함된 `oldsha..newsha` 형식의 해시 범위는 입력 노이즈로 간주하고 버린다
   - 사용자가 함께 준 해시 범위를 PR 대상 산정 근거로 사용하지 않는다
2. 입력 모드를 결정한다.
   - `ticket + date`가 있으면 모드 C
   - `target`이 있으면 모드 A
   - 그 외 `ticket`만 있으면 모드 B
3. source/target 브랜치를 결정한다.
   - 모드 A: `feature/{ticket} -> ci/dev`, `feature-base/{ticket} -> ci/stg`
   - 모드 B: `feature/{ticket} -> feature-base/{ticket}`
   - 모드 C: `feature-base/{ticket} -> rc/{yyyymmdd}-1`
4. 기본 유효성 검증을 수행한다.
   - source와 target이 같으면 중단
   - 모드 A target은 `ci/dev|ci/stg`만 허용
   - 모드 C date는 숫자 8자리만 허용
5. 원격 저장소 기준으로 source/target 브랜치 존재를 확인한다.
   - source가 없으면 즉시 중단
   - 모드 A target은 이미 있어야 한다
   - 모드 B/C target은 `mcp__github__create_branch`를 `from_branch=develop`로 먼저 시도한다
   - `already exists` 성격의 실패면 기존 브랜치로 간주하고 계속 진행한다
6. 동일한 head/base 오픈 PR이 있는지 `mcp__github__search_pull_requests`로 확인한다.
   - 있으면 새 PR을 만들지 않고 기존 URL을 반환한다
7. PR 생성 전 source/target compare 내역이 있는지 확인한다.
   - 반드시 GitHub 원격 저장소 기준으로 판단한다
   - compare 의미는 GitHub compare view와 동일하게 `target...source` 기준으로 본다
   - 로컬 `git log`, `git rev-parse`, `git merge-base` 결과로 원격 상태를 대체하지 않는다
   - source/target 최근 커밋 목록을 따로 나열해 PR 생성 여부를 수동 판단하지 않는다
   - GitHub MCP에 compare 전용 도구가 없으면 direct API를 기본 사용하지 않는다
   - 이 경우 `mcp__github__search_pull_requests`로 중복 PR을 먼저 확인한 뒤, `mcp__github__create_pull_request`의 성공/실패 응답으로 diff 존재 여부를 판단한다
   - `No commits between`, `There isn’t anything to compare`, `A pull request already exists` 성격의 응답을 그대로 사용한다
8. PR 제목을 만든다.
   - 단, 모드 C에서 후속으로 만드는 `rc/{yyyymmdd}-1 -> main` PR 제목은 항상 `<source> -> <target>` 형식을 사용한다
   - 사용자가 body/title을 주지 않았다면 source에는 있고 target에는 없는 신규 non-merge commit들 중 가장 오래된 commit 제목을 사용한다
   - compare 전용 데이터가 있으면 `target...source` compare 범위에서 가장 오래된 non-merge commit 제목을 사용한다
   - compare 전용 데이터가 없으면 source/target 커밋 목록에서 source-only 신규 non-merge commit 집합을 추려 가장 오래된 제목을 사용한다
   - 형식: `<대표 제목> -> <target>`
   - 적합한 제목이 없으면 `chore: <source> -> <target>`
9. PR 본문을 만든다.
   - 사용자가 body를 주면 그대로 사용
   - 없으면 source에는 있고 target에는 없는 신규 non-merge commit 집합 전체를 기준으로 요약한다
   - compare 전용 데이터가 있으면 `target...source` compare 범위를 그대로 사용한다
   - compare 전용 데이터가 없으면 source/target 커밋 목록에서 source-only 신규 non-merge commit 집합을 추린다
   - 커밋 타이틀과 커밋 본문을 바탕으로 사용자 관점 변경사항만 2~5개 불릿으로 압축한다
   - `Summary`, `Notes` 같은 고정 템플릿이나 source/target, sha range, 내부 판단 메모는 쓰지 않는다
   - commit 제목 한 개만 복붙하지 말고 commit 본문 설명까지 반영한다
10. `mcp__github__create_pull_request`로 PR을 생성한다.
11. 머지 여부를 결정한다.
   - 모드 A/C는 draft가 아니면 즉시 머지
   - 모드 B는 PR 생성만 수행
12. 모드 A/C에서 draft가 아니면 `mcp__github__merge_pull_request`로 머지한다.
   - 머지 실패가 `conflict`, `dirty`, `not mergeable`, `405 Pull Request is not mergeable` 성격이면 충돌 상태로 간주한다
   - 충돌 상태면 추가 해결 작업을 시도하지 않고, 해당 PR 링크와 충돌 상태만 결과로 반환한다
   - 충돌 상태에서 로컬 저장소 탐색, 추가 compare, conflict-resolve 브랜치 생성, 수동 머지 시도는 하지 않는다
13. 모드 C에서는 후속 운영 PR을 확인한다.
   - `rc/{yyyymmdd}-1 -> main` 오픈 PR 검색
   - 없으면 생성
   - 제목은 항상 `rc/{yyyymmdd}-1 -> main` 형식을 사용한다
   - 본문 규칙은 앞선 일반 PR 생성 규칙과 동일하게 적용한다
   - 이 PR은 자동 머지하지 않는다
14. 모드 C에서는 위키를 업데이트한다.
   - 운영 배포 준비는 위키 업데이트까지 완료되어야 종료로 본다
   - 위키 대상 페이지는 `https://tmobi.atlassian.net/wiki/spaces/TMAPDRIVER` 스페이스 안에서 요청한 배포일자로 검색해 찾는다
   - 검색 시작점은 상위 페이지 `https://tmobi.atlassian.net/wiki/spaces/TMAPDRIVER/pages/253723323`의 하위 페이지들이다
   - 하위 페이지 제목은 우선적으로 `{yyyy}년 배포 내역 ({mm}~{mm}월)` 패턴을 찾는다
   - 요청한 배포 요청 날짜가 제목의 월 범위 안에 포함되는 페이지를 1차 후보로 선택한다
   - 같은 연도 안에서 월 범위를 해석할 때는 제목의 시작 월과 종료 월을 포함 범위로 본다
   - 제목 기준으로 월 범위가 맞는 페이지가 1개면 그 페이지를 사용한다
   - 제목 기준 후보가 여러 개면 하위 페이지 본문을 읽어 실제 table row에 해당 날짜가 있는 페이지를 우선 선택한다
   - 제목 기준 후보가 0개면 그때만 요청한 `yyyymmdd`를 기반으로 만든 날짜 표현으로 하위 페이지 title/body 검색을 수행한다
   - 검색은 요청한 `yyyymmdd`를 기반으로 만든 날짜 표현으로 수행한다. 최소 `YYYY-MM-DD`, `YYYY/M/D`, `M/D`, `MM/DD`, `M월 D일` 변형을 함께 고려한다
   - 검색 결과 후보 페이지는 title/body 중 하나 이상에서 요청한 배포일자 표현이 확인되어야 한다
   - 후보가 여러 개면 페이지 본문을 읽어 실제 table row에 해당 날짜가 있는 페이지를 우선 선택한다
   - 그래도 여러 개면 제목이 `{yyyy}년 배포 내역 ({mm}~{mm}월)` 패턴에 맞는 페이지를 우선하고, 끝내 단일 페이지로 좁혀지지 않으면 위키 업데이트를 중단하고 ambiguity를 결과에 명시한다
   - 검색 결과가 없으면 고정 페이지로 폴백하지 않고 위키 업데이트를 중단하고 not found를 결과에 명시한다
   - 선택된 페이지에 대해 Confluence REST API로 현재 페이지를 읽는다: `GET /wiki/api/v2/pages/{pageId}?body-format=atlas_doc_format`
   - `.body.atlas_doc_format.value`를 JSON으로 파싱해 현재 ADF를 기준 문서로 사용한다
   - 선택된 페이지에서 해당 날짜 행을 찾고 배포 애플리케이션 항목을 반영한다
   - 페이지 전체를 새로 구성하지 말고, 현재 ADF의 table node에서 대상 날짜 행만 추가 또는 수정한다
   - 같은 날짜 행이 있으면 기존 앱 항목은 유지하고, 새로 확인된 앱 항목만 추가 또는 수정한다
   - 같은 날짜 행이 없으면 현재 페이지의 첫 data row 형태를 기준으로 새 행 1개만 복제해 최신 날짜가 위로 오는 위치에 삽입한다
   - `rc -> main` PR과 ticket별 운영 배포 준비 PR 중, 실제 배포 브랜치 컬럼에는 앱별 `rc -> main` PR 링크를 우선 반영한다
   - 릴리즈 링크는 비워두지 말고 먼저 채우려고 시도한다
   - 사용자가 릴리즈 링크를 주면 그대로 사용한다
   - 없으면 `TMAPDRIVER` Jira version 중 요청한 운영 배포일에 해당하는 버전을 찾아 `https://tmobi.atlassian.net/projects/TMAPDRIVER/versions/{versionId}/tab/release-report-all-issues` 형식으로 채운다
   - Jira version을 끝내 찾지 못한 경우에만 릴리즈 링크를 빈 칸으로 두고, 결과에 unresolved 상태를 명시한다
   - 비고는 사용자 입력이나 기존 행 컨텍스트가 없으면 빈 칸으로 둔다
   - 위키 반영은 REST API로 수행한다: 현재 `version.number + 1`로 PUT 요청을 만들고 `body.representation=atlas_doc_format`를 사용한다
15. 결과를 반환한다.
   - 생성 또는 재사용한 운영 배포 준비 PR URL
   - `rc -> main` PR URL
   - 위키 업데이트 여부

## Wiki Rules

- 위키 대상은 항상 `https://tmobi.atlassian.net/wiki/spaces/TMAPDRIVER` 스페이스 안에서 요청한 배포일자로 검색해 결정한다.
- 검색 시작점은 상위 페이지 `253723323`의 직계 또는 하위 descendant 페이지다.
- 페이지 선택은 하위 페이지 제목 `{yyyy}년 배포 내역 ({mm}~{mm}월)` 패턴과 요청 배포일자의 포함 관계를 우선 사용한다.
- 요청 배포일자의 연도와 제목의 연도가 다르면 후보에서 제외한다.
- 요청 배포일자의 월이 제목의 시작 월과 종료 월 사이에 포함되는 페이지를 우선 후보로 본다.
- 제목 기준으로 단일 후보가 잡히면 날짜 문자열 전체 검색보다 우선 채택한다.
- 고정 `pageId` 전제는 두지 않는다.
- 검색 결과가 0건이면 업데이트를 중단하고 not found로 처리한다.
- 검색 결과가 2건 이상이면 본문 table row의 날짜 일치 여부까지 확인해 1건으로 좁힌다.
- 날짜 일치 확인 후에도 여러 페이지가 남으면 임의 선택하지 말고 ambiguity로 중단한다.
- 검색은 배포일자 문자열의 단일 포맷에만 의존하지 않는다. `YYYY-MM-DD`, `YYYY/M/D`, `M/D`, `MM/DD`, `M월 D일` 같은 변형을 함께 고려한다.
- 위키 자동화는 별도 `python`/`node` 스크립트 전제를 두지 않고, 기본적으로 `bash + curl` 방식으로 수행한다고 가정한다.
- 위키 업데이트는 Confluence REST API의 현재 ADF를 기준으로 수정하고, 임의 재생성한 Markdown/HTML을 기준으로 덮어쓰지 않는다.
- 위키 수정 전 현재 행 구조와 기존 앱 항목을 먼저 확인한다.
- 페이지 상단 smart link, 헤더, 다른 날짜 행, 구분선, 표 외부 텍스트는 변경하지 않는다.
- 가능하면 페이지 전체 본문 재작성 대신 대상 날짜 행만 추가/수정하는 방식으로 반영한다.
- 새 행을 넣을 때는 기존 첫 data row의 셀 구조, `localId`, `colwidth`, paragraph/bullet 배치를 기준으로 필요한 부분만 복제한다.
- 같은 날짜 행 수정 시 해당 행의 필요한 셀만 바꾸고, 다른 셀의 기존 문구/링크/공백 표현은 유지한다.
- 새 날짜 행 추가 시 인접 행의 마크업 형태를 그대로 복제해서 사용한다.
- Markdown 표 구분선 `| --- | --- | --- | --- |` 는 헤더 바로 아래 1회만 유지한다. 각 데이터 행 사이에 새 구분선을 추가하지 않는다.
- 각 애플리케이션 셀의 줄바꿈, 공백, smart link 표현, 빈 줄 유무는 기존 행 스타일을 그대로 유지한다.
- 표의 cell width, layout, 정렬 관련 표현은 원본 페이지 기준을 유지하고, 모든 셀을 동일 폭처럼 보이게 재정렬하지 않는다.
- 위키 표는 최신 날짜가 위로 오므로, 새 날짜 행은 기존 날짜 내림차순 흐름을 깨지 않도록 상단 쪽에 삽입한다.
- 릴리즈 링크 셀은 가능하면 Jira version release report 링크로 채운다. 예전 백업 문서의 `릴리즈 링크/비고는 ... 빈 칸` 규칙은 릴리즈 링크에는 더 이상 적용하지 않는다.
- 기존 Confluence 행의 다른 애플리케이션 항목을 임의로 삭제하지 않는다.
- 삭제 지시가 명시적으로 있을 때만 기존 앱 항목을 제거한다.
- 배포 브랜치 셀은 첫 줄에 분류명(예: `BE`)을 두고, 다음 줄부터 불릿 목록을 사용한다.
- 앱 항목 형식은 `- 애플리케이션명 : GitHub PR 링크`
- 비고는 불릿 없는 plain text
- 페이지 수정 시 기존 table `layout`, `width` 같은 표시 속성은 원본 값을 그대로 유지한다.
- 특정 width 값으로 고정해서 덮어쓰지 않는다.

## Duplicate Check Query

- `repo:<owner>/<repo> is:pr is:open head:<source> base:<target>`
- 포크 워크플로가 필요하면 `head:<owner>:<source>` 사용

## Guardrails

- GitHub MCP 우선으로 처리한다. 직접 API를 새로 조합하는 방식은 MCP로 해결할 수 없을 때만 사용한다.
- GitHub 직접 API 호출은 기본값이 아니다. MCP로 처리 가능한데도 추가 권한이 필요한 `curl`, `gh api`를 먼저 시도하지 않는다.
- 위키 업데이트는 `mcp__mcp_atlassian__updateConfluencePage`보다 Confluence REST API 직접 호출을 우선 사용한다.
- 원격 저장소 기준으로만 판단한다. 로컬 저장소 상태, stale remote-tracking branch, 미실행 fetch를 근거로 PR 생성/중단 여부를 결정하지 않는다.
- 사용자가 붙여 넣은 `oldsha..newsha` 범위는 보조 메모로도 사용하지 않는다. PR 대상 산정은 항상 원격의 source/target compare 결과만 사용한다.
- compare 전용 MCP 도구가 없으면 원격 diff 존재 여부는 GitHub MCP의 중복 PR 조회 결과와 PR 생성 응답으로 판단한다.
- PR 제목은 기본적으로 source에는 있고 target에는 없는 신규 non-merge commit들 중 가장 오래된 commit 제목을 기준으로 만든다.
- 단, 모드 C의 `rc/{yyyymmdd}-1 -> main` PR 제목은 예외 없이 항상 `<source> -> <target>` 형식으로 고정한다. 예: `rc/20260103-1 -> main`
- PR 본문에는 내부 작업 메모를 남기지 않는다. 예: source/target 브랜치명 나열, `remote branch 기준`, `sha range는 판단 근거로 사용하지 않음` 같은 문구 금지.
- PR 본문은 커밋 타이틀과 커밋 본문을 기반으로 변경사항 요약 중심으로 작성한다.
- 위키 페이지 검색은 반드시 `TMAPDRIVER` 스페이스와 요청한 배포일자를 기준으로 수행한다.
- 위키 페이지 검색은 상위 페이지 `253723323` 하위 범위로 제한한다.
- 제목 `{yyyy}년 배포 내역 ({mm}~{mm}월)` 패턴으로 요청일 포함 여부를 먼저 판단하고, 날짜 문자열 자유 검색은 후보가 없을 때만 보조적으로 사용한다.
- 검색 없이 고정 pageId를 바로 사용하는 것은 금지한다.
- 검색 결과가 없거나 여러 건으로 남으면 다른 페이지로 추측 폴백하지 않는다.
- 위키 업데이트 때문에 대상 날짜 행 외 다른 행이나 표 바깥 내용이 바뀌면 안 된다.
- 현재 페이지 ADF를 읽지 않고 새 body를 추측해서 만들지 않는다.
- 허용 브랜치 패턴 외 다른 source/target 조합은 사용하지 않는다.
- 모드 C에서 date가 있으면 절대 모드 B로 폴백하지 않는다.
- source 브랜치와 target 브랜치의 최근 커밋 목록을 각각 조회해 PR 생성 여부를 수동 판단하지 않는다. 다만 compare 전용 데이터가 없을 때 제목/본문 산정용 source-only 신규 commit 추출에는 사용할 수 있다.
- merge commit은 신규 commit 추출과 제목/본문 계산에서 제외한다.
- 모드 B는 자동 머지하지 않는다.
- 모드 C는 운영 배포 준비 PR 머지 후 `rc -> main` PR 확인/생성과 위키 업데이트까지 수행해야 완료다.
- 머지 실패가 충돌(`dirty`, `conflict`, `not mergeable`)이면 해결을 시도하지 말고 PR 링크만 공유한다.
- 충돌이 아닌 일반 머지 실패는 실패 사유를 그대로 반환하고 PR은 열린 상태로 둔다.
- 충돌 상태에서 로컬 clone 확인, fetch, 수동 conflict resolve, 별도 conflict-resolve PR 생성은 하지 않는다.
- 출력은 간결하게 작성하고 PR 링크를 반드시 포함한다.
