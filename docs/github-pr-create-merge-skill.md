# github-pr-create-merge Skill

`github-pr-create-merge`는 GitHub MCP를 사용해 PR 생성과 즉시 머지를 자동으로 수행하는 워크플로우입니다.

## 목적

- 최소 입력으로 PR 생성/머지 자동화
- 제한된 타깃 브랜치 규칙 강제
- 중복 PR 방지 및 안전 가드레일 적용

## 입력

필수 입력:

- Repository: `owner/repo`
- Source branch: 머지할 소스 브랜치
- Target environment or branch:
  - `dev` 또는 `ci/dev` -> `ci/dev`
  - `stg` 또는 `ci/stg` -> `ci/stg`

선택 입력:

- PR body
- Draft 여부
- Merge method (`merge`, `squash`, `rebase`; 기본값 `merge`)

## 동작 순서

1. 저장소(`owner/repo`) 파싱
2. 타깃 브랜치 정규화 (`dev` -> `ci/dev`, `stg` -> `ci/stg`)
3. 허용 타깃(`ci/dev`, `ci/stg`) 검증
4. source/target 동일 여부 검증
5. source/target 브랜치 존재 확인
6. 동일 head/base 오픈 PR 중복 확인
7. PR 제목 산정
   - source 브랜치 커밋 조회
   - `Merge pull request ...` 형태 merge 커밋 제외
   - 첫 유효 커밋 제목 기반으로 `<first-commit-title> -> <target-branch>` 생성
   - 대체 제목: `chore: <source> -> <target>`
8. PR 본문 결정
9. PR 생성
10. draft가 아니면 즉시 머지
11. PR URL/번호/제목/타깃/머지 결과 반환

## 중복 PR 확인 쿼리

- `repo:<owner>/<repo> is:pr is:open head:<source> base:<target>`

포크 워크플로우:

- `repo:<owner>/<repo> is:pr is:open head:<owner>:<source> base:<target>`

## 가드레일

- source와 target이 같으면 PR 생성 금지
- 최종 target은 `ci/dev` 또는 `ci/stg`만 허용
- 타깃 정규화를 모든 검증 전에 수행
- 제목 산정 시 `Merge pull request ...` 커밋 제외
- 동일 조건 오픈 PR이 있으면 기존 PR 재사용
- 브랜치 미존재 시 명확한 원인과 함께 중단
- 즉시 머지 실패 시 실패 사유를 반환하고 PR은 열린 상태 유지

## 예시

- 입력: `source=feature/login`, `target=dev`
- 정규화: `target=ci/dev`
- 결과: `feature/login` -> `ci/dev` PR 생성 후 즉시 머지
