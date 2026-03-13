---
name: github-pr-create-merge
description: Create and optionally merge GitHub pull requests via GitHub MCP for deployment branches (`ci/dev`, `ci/stg`), review branches (`feature/{ticket}` -> `feature-base/{ticket}`), and operation-release branches (`feature-base/{ticket}` -> `rc/{yyyymmdd}-1`), then update the deployment wiki for operation-release flow.
---

# GitHub PR Create

GitHub MCP로 최소 입력만 받아 PR을 생성한다. 운영 배포 준비 모드(C)는 PR 생성/머지에서 끝나지 않고, `rc/{yyyymmdd}-1 -> main` PR 확인과 배포 위키 업데이트까지 포함한다.

## Inputs

- Repository: `owner/repo` (required)
- Ticket number: `TICKET-123` 형식 (required)
- 입력 모드 A, 배포용:
  - target: `dev|ci/dev|stg|ci/stg`
  - source는 자동 결정:
    - `ci/dev` <- `feature/{ticket}`
    - `ci/stg` <- `feature-base/{ticket}`
- 입력 모드 B, 코드리뷰용:
  - source=`feature/{ticket}`
  - target=`feature-base/{ticket}`
- 입력 모드 C, 운영 배포 준비용:
  - date: `yyyymmdd` (required)
  - source=`feature-base/{ticket}`
  - target=`rc/{yyyymmdd}-1`

선택 입력:

- PR body
- Draft flag
- Merge method: `merge|squash|rebase` (default `merge`)
- Wiki page
  - 운영 배포 준비 결과를 반영할 Confluence 페이지
  - 주어지지 않으면 기존 페이지를 임의로 추측하지 않는다

## Workflow

1. 저장소 입력을 정규화한다.
   - 앞 `/` 제거
   - `.git` suffix 제거
2. 입력 모드를 결정한다.
   - `ticket + date`가 있으면 모드 C
   - `target`이 있으면 모드 A
   - 그 외 `ticket`만 있으면 모드 B
3. source/target 브랜치를 결정한다.
   - 모드 A:
     - `dev -> ci/dev`
     - `stg -> ci/stg`
     - `ci/dev <- feature/{ticket}`
     - `ci/stg <- feature-base/{ticket}`
   - 모드 B:
     - `feature/{ticket} -> feature-base/{ticket}`
   - 모드 C:
     - `feature-base/{ticket} -> rc/{yyyymmdd}-1`
4. 기본 유효성 검증을 수행한다.
   - source와 target이 같으면 중단
   - 모드 A target은 `ci/dev|ci/stg`만 허용
   - 모드 C date는 숫자 8자리만 허용
5. source 브랜치 존재를 확인한다.
   - source가 없으면 어떤 브랜치가 없는지 명시하고 즉시 중단
6. target 브랜치를 처리한다.
   - 모드 A:
     - target은 이미 있어야 한다
     - 없으면 즉시 중단
   - 모드 B/C:
     - `mcp__github__create_branch`를 먼저 시도한다
     - 성공하면 새 target 브랜치 생성으로 처리한다
     - `already exists` 성격의 실패면 기존 브랜치가 있다고 보고 계속 진행한다
     - 그 외 실패는 중단한다
     - `from_branch`는 `develop`
7. 동일한 head/base 오픈 PR이 있는지 `mcp__github__search_pull_requests`로 확인한다.
   - 있으면 새 PR을 만들지 않고 기존 URL을 반환한다
8. PR 대상 커밋을 준비한다.
   - 반드시 `head=<source>`, `base=<target>` diff 범위만 사용한다
   - `Merge pull request ...`로 시작하는 merge commit은 제외한다
   - 남은 커밋을 오래된 순으로 정렬한다
   - 대상 커밋이 0개면 중단한다
9. PR 제목을 만든다.
   - 첫 번째 non-merge commit 제목 사용
   - 형식: `<first-commit-title> -> <target>`
   - 적합한 제목이 없으면 `chore: <source> -> <target>`
10. PR 본문을 만든다.
   - 사용자가 body를 주면 그대로 사용
   - 없으면 diff 범위 전체 non-merge commit 본문을 기준으로 요약
   - commit 제목 한 개만으로 본문을 만들지 않는다
11. PR을 생성한다.
   - `mcp__github__create_pull_request`
12. 머지 여부를 결정한다.
   - 모드 A/C는 draft가 아니면 즉시 머지
   - 모드 B는 PR 생성만 수행
13. 모드 A/C에서 draft가 아니면 `mcp__github__merge_pull_request`로 머지한다.
14. 모드 C에서는 후속 운영 PR을 확인한다.
   - `rc/{yyyymmdd}-1 -> main` 오픈 PR 검색
   - 없으면 생성
   - 제목은 `rc/{yyyymmdd}-1 -> main`
   - 이 PR은 자동 머지하지 않는다
15. 모드 C에서는 위키를 업데이트한다.
   - 운영 배포 준비는 위키 업데이트까지 완료되어야 종료로 본다
   - 위키 페이지가 주어졌으면 해당 날짜 행을 찾고 배포 애플리케이션 항목을 반영한다
   - 같은 날짜 행이 있으면 기존 앱 항목은 유지하고, 새로 확인된 앱 항목만 추가 또는 수정한다
   - 같은 날짜 행이 없으면 현재 페이지 양식을 따라 새 행을 추가한다
   - `rc -> main` PR과 ticket별 운영 배포 준비 PR 중, 실제 배포 브랜치 컬럼에는 앱별 `rc -> main` PR 링크를 우선 반영한다
   - 릴리즈 링크/비고는 사용자 입력이나 기존 행 컨텍스트가 없으면 빈 칸으로 둔다
16. 결과를 반환한다.
   - 생성 또는 재사용한 운영 배포 준비 PR URL
   - `rc -> main` PR URL
   - 위키 업데이트 여부

## Wiki Rules

- 위키 수정 전 현재 행 구조와 기존 앱 항목을 먼저 확인한다.
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

- 허용 브랜치 패턴 외 다른 source/target 조합은 사용하지 않는다.
- 모드 C에서 date가 있으면 절대 모드 B로 폴백하지 않는다.
- PR 제목/본문 산정은 반드시 head/base diff 범위만 사용한다.
- merge commit은 제목/본문 계산에서 제외한다.
- 모드 B는 자동 머지하지 않는다.
- 모드 C는 운영 배포 준비 PR 머지 후 `rc -> main` PR 확인/생성과 위키 업데이트까지 수행해야 완료다.
- 머지 실패 시 실패 사유를 그대로 반환하고 PR은 열린 상태로 둔다.
- 출력은 간결하게 작성하고 PR 링크를 반드시 포함한다.
