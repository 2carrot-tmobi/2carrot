---
name: github-pr-create-merge
description: Create a GitHub pull request via GitHub MCP for deployment branches (`ci/dev`, `ci/stg`), ticket branches (`feature/{ticket}` -> `feature-base/{ticket}`), and release-candidate branches (`feature-base/{ticket}` -> `rc/{yyyymmdd}-1`), with duplicate checks, commit-based title/body generation, and automatic `rc/{yyyymmdd}-1 -> main` PR creation for production-release prep.
---

# GitHub PR Create

GitHub MCP로 최소 입력만 받아 PR을 생성하고, 필요 시에만 머지를 수행한다.

## Inputs

다음 입력을 수집한다.

- Repository: `owner/repo` (required)
- 입력 모드 A (배포용):
  - Ticket number (required)
  - Target environment or branch:
    - `dev` or `ci/dev` -> `ci/dev`
    - `stg` or `ci/stg` -> `ci/stg`
  - source는 타겟 기준으로 자동 결정:
    - `ci/dev`: `feature/{ticket}`
    - `ci/stg`: `feature-base/{ticket}`
- 입력 모드 B (코드리뷰용):
  - Ticket number (required)
  - source/target은 아래 규칙으로 자동 결정:
    - source: `feature/{ticket}`
    - target: `feature-base/{ticket}`
- 입력 모드 C (운영 배포용):
  - Ticket number (required)
  - Date (required, `yyyymmdd`)
  - source/target은 아래 규칙으로 자동 결정:
    - source: `feature-base/{ticket}`
    - target: `rc/{yyyymmdd}-1`
  - 추가 후속 작업:
    - 운영 배포 준비 PR을 머지한 뒤 `rc/{yyyymmdd}-1` -> `main` PR도 생성한다.

선택 입력:

- PR body
- Draft flag
- Merge method (`merge`, `squash`, `rebase`; default `merge`, 자동 머지 모드(A/C)에서 사용)

## Workflow

1. 저장소 입력(`owner/repo`)을 파싱한다.
   - `repo` 값에 `.git` suffix가 있으면 제거한다. (예: `2carrot.git` -> `2carrot`)
   - 입력 앞의 `/`는 제거한다. (예: `/org/repo` -> `org/repo`)
2. 입력 모드를 결정한다.
   - Ticket number와 Date(`yyyymmdd`)가 함께 주어지면 운영 배포용 모드(C)로 처리한다.
   - 배포 타겟(`dev|ci/dev|stg|ci/stg`)이 주어지면 배포용 모드(A)로 처리한다.
   - 배포 타겟 없이 Ticket number만 주어지면 코드리뷰용 모드(B)로 처리한다.
   - Date가 입력된 요청은 다른 모드로 폴백하지 않는다(항상 모드 C).
3. 브랜치를 결정한다.
   - 모드 A: 타겟 입력을 정규화한다.
   - `dev` -> `ci/dev`
   - `stg` -> `ci/stg`
   - `ci/dev`, `ci/stg`는 그대로 사용한다.
   - 모드 A: 정규화 타겟 기준으로 source를 자동 결정한다.
   - target=`ci/dev`면 source=`feature/{ticket}`
   - target=`ci/stg`면 source=`feature-base/{ticket}`
   - 모드 A에서는 위 매핑 외 source 패턴을 허용하지 않는다.
   - 모드 B(코드리뷰용): source=`feature/{ticket}`, target=`feature-base/{ticket}`를 사용한다.
   - 모드 C(운영 배포용): source=`feature-base/{ticket}`, target=`rc/{yyyymmdd}-1`를 사용한다.
4. 유효성 검증을 수행한다.
   - 모드 A에서 타겟이 `ci/dev`, `ci/stg` 외 값이면 중단한다.
   - 모드 C(운영 배포용)에서 Date는 `yyyymmdd` 형식(숫자 8자리)이어야 한다.
   - source와 target이 동일하면 중단한다.
5. 브랜치 존재를 확인하고 모드별로 처리한다.
   - 공통: `mcp__github__list_branches`로 source/target 브랜치 존재를 확인한다.
   - source 브랜치가 없으면 모든 모드에서 즉시 실패한다.
   - 모드 A(배포용): target 브랜치가 없으면 실패한다.
   - 모드 B(코드리뷰용), 모드 C(운영 배포용): target 브랜치가 없으면 생성 후 진행한다.
6. 모드 B/C(코드리뷰용/운영 배포용)에서 target 브랜치가 없을 때 `mcp__github__create_branch`로 생성한다.
   - `branch`: 최종 target 브랜치
   - `from_branch`: `develop`
7. `mcp__github__search_pull_requests`로 동일 head/base 오픈 PR 존재 여부를 확인한다.
8. 커밋 데이터를 준비한다.
   - PR의 `head=<source>`, `base=<target>` 범위(두 브랜치 diff)에 포함된 커밋만 대상으로 사용한다.
   - source 브랜치 전체 히스토리를 그대로 사용하지 않는다.
   - 커밋 메시지 첫 줄이 `Merge pull request`로 시작하는 merge 커밋은 제외한다.
   - 제외 후 남은 커밋을 오래된 순으로 정렬한다.
   - 대상 커밋이 0개면 PR 생성을 중단한다. (`No commits between ...`)
9. PR 제목을 결정한다.
   - 8단계 결과 중 첫 번째(가장 오래된) 커밋 제목(first commit title)을 사용한다.
   - 제목을 `<first-commit-title> -> <target-branch>` 형태로 만든다.
   - 적합한 커밋을 찾지 못하면 `chore: <source> -> <target>`를 사용한다.
10. PR 본문을 결정한다.
   - 사용자가 본문을 주면 그대로 사용한다.
   - 없으면 모든 모드에서 동일 규칙으로 자동 생성한다.
   - PR 제목에 사용된 커밋 1개만 기준으로 본문을 만들지 않는다.
   - 항상 8단계의 `head/base diff` 범위 전체 non-merge 커밋을 기준으로 본문을 만든다.
   - non-merge 커밋이 1개면 해당 커밋의 제목(첫 줄)은 제외하고 본문만 사용한다.
   - non-merge 커밋이 여러 개면 모든 non-merge 커밋의 제목(첫 줄)은 제외하고 본문을 추출한다.
   - 추출한 본문들을 중복 제거 후 간략 요약해 하나의 본문으로 작성한다.
   - 이 경우 PR 본문 첫 줄에 `커밋 요약 내용` 헤더를 반드시 포함한다.
11. `mcp__github__create_pull_request`로 PR을 생성한다.
   - `owner`: 저장소 owner
   - `repo`: 저장소 이름
   - `head`: source 브랜치
   - `base`: 최종 target 브랜치
   - `title`: 9단계에서 결정한 제목
   - `body`: 선택
   - `draft`: 선택
12. 머지 실행 여부를 모드별로 결정한다.
   - 모드 A(배포용): draft가 아니면 즉시 머지한다.
   - 모드 B(코드리뷰용): 머지하지 않고 PR 생성 결과만 반환한다.
   - 모드 C(운영 배포용): draft가 아니면 즉시 머지한다.
13. 모드 A(배포용) 또는 모드 C(운영 배포용)에서 draft가 아니면 `mcp__github__merge_pull_request`로 즉시 머지한다.
   - `owner`: 저장소 owner
   - `repo`: 저장소 이름
   - `pullNumber`: 생성된 PR 번호
   - `merge_method`: 사용자 지정 값 또는 기본값 `merge`
14. 모드 C(운영 배포용)에서는 운영 배포 준비 PR 머지 후 `rc/{yyyymmdd}-1` -> `main` PR 상태를 확인한다.
   - `mcp__github__search_pull_requests`로 동일 head/base 오픈 PR 존재 여부를 확인한다.
   - 없으면 `mcp__github__create_pull_request`로 `head=rc/{yyyymmdd}-1`, `base=main` PR을 생성한다.
   - 제목은 `<rc-branch> -> main` 형식을 사용한다. 예: `rc/20260316-1 -> main`
   - 이 PR은 자동 머지하지 않는다.
15. PR URL, PR 번호, 최종 제목, source/target을 반환한다.
   - 모든 모드에서 PR 링크(PR URL)는 항상 포함한다.
   - 모드 C(운영 배포용)는 ticket별 운영 배포 준비 PR 링크와 함께 `rc -> main` PR 링크도 포함한다.

## Query Pattern For Duplicate Check

다음 패턴을 사용한다.

- `repo:<owner>/<repo> is:pr is:open head:<source> base:<target>`

포크 워크플로가 필요하면 아래 패턴을 사용한다.

- `repo:<owner>/<repo> is:pr is:open head:<owner>:<source> base:<target>`

## Guardrails

- source와 target이 같으면 PR을 만들지 않는다.
- 모드 A(배포용)에서 최종 target은 `ci/dev` 또는 `ci/stg`만 허용한다.
- 모드 A(배포용)에서 source는 수동 입력을 받지 않고 target+ticket으로 자동 계산한다.
- 모드 A(배포용)에서 `ci/dev`는 source=`feature/{ticket}`일 때만 진행한다.
- 모드 A(배포용)에서 `ci/stg`는 source=`feature-base/{ticket}`일 때만 진행한다.
- 모드 B(코드리뷰용)에서 브랜치 규칙은 `feature/{ticket}` -> `feature-base/{ticket}`만 허용한다.
- 모드 C(운영 배포용)에서 브랜치 규칙은 `feature-base/{ticket}` -> `rc/{yyyymmdd}-1`만 허용한다.
- 모드 A/B/C 모두 위 규칙 외 다른 source/target 패턴은 허용하지 않는다.
- Date가 주어진 요청에서 `feature/{ticket} -> feature-base/{ticket}`로 처리하면 잘못된 실행으로 간주하고 즉시 중단한다.
- 모드 C(운영 배포용)에서 Date는 `yyyymmdd` 형식(숫자 8자리)만 허용한다.
- 모드 C(운영 배포용)에서는 ticket PR 머지 후 `rc/{yyyymmdd}-1 -> main` PR도 확인하거나 생성해야 한다.
- 모드 C(운영 배포용)의 `rc -> main` PR 제목은 `<rc-branch> -> main` 형식을 사용한다.
- 모드 C(운영 배포용)의 `rc -> main` PR은 자동 머지하지 않는다.
- 모드 A(배포용)는 모든 검증 전에 `dev -> ci/dev`, `stg -> ci/stg` 정규화를 수행한다.
- PR 제목/본문 산정은 반드시 `head/base diff` 범위 커밋만 사용한다.
- PR 제목 산정 시 `Merge pull request ...` 형태의 merge 커밋은 제외한다.
- PR 본문 자동 생성 시에도 `Merge pull request ...` 커밋은 제외한다.
- 모든 모드에서 non-merge 커밋이 1개면 커밋 제목(첫 줄)은 제외하고 본문만 사용한다.
- 모든 모드에서 non-merge 커밋이 여러 개면 모든 non-merge 커밋의 제목(첫 줄)을 제외한 본문을 합쳐 요약한다.
- non-merge 커밋이 여러 개인 경우 PR 본문은 `커밋 요약 내용` 헤더로 시작해야 한다.
- 동일 조건의 오픈 PR이 이미 있으면 새 PR을 만들지 않고 기존 URL을 반환한다.
- source 브랜치가 없으면 어떤 source가 없는지 명시하고 즉시 중단한다.
- 모드 A(배포용)에서 target 브랜치가 없으면 어떤 target이 없는지 명시하고 중단한다.
- 모드 B(코드리뷰용), 모드 C(운영 배포용)에서 target 브랜치가 없으면 `develop`을 기준으로 target 브랜치를 생성한 뒤 진행한다.
- 모드 A(배포용)는 draft가 아니면 즉시 머지를 시도한다.
- 모드 C(운영 배포용)는 draft가 아니면 즉시 머지를 시도한다.
- 모드 B(코드리뷰용)는 절대 자동 머지하지 않는다.
- 동일한 `rc -> main` 오픈 PR이 이미 있으면 모드 C(운영 배포용)에서는 새로 만들지 않고 기존 URL을 반환한다.
- 머지 실패 시 실패 사유를 그대로 반환하고 PR은 열린 상태로 둔다.
- 출력은 간결하게 작성하고 PR 링크를 반드시 포함한다.
- merge commit SHA와 merge 결과 텍스트는 최종 반환 항목에서 제외한다.
