# github-pr-create-merge Quick Guide


## GitHub MCP 추가

GitHub MCP가 없으면 먼저 MCP 설정에 `github` 서버를 추가하면 된다.

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "ghp_xxx"
      }
    }
  }
}
```

- MCP 설정 파일의 `mcpServers`에 위 예시를 추가
- `GITHUB_TOKEN`은 GitHub Personal Access Token 사용
- 권한은 최소 `repo`, `pull_request` 정도는 있어야 한다
- 설정 반영 후 Codex 또는 IDE를 재시작

## 사용법

### dev 배포

```text
/{owner}/{repo}.git {TICKET} dev 배포해줘
```

### stg 배포

```text
/{owner}/{repo}.git {TICKET} stg 배포해줘
```

### 코드리뷰 PR

```text
/{owner}/{repo}.git {TICKET} 코드리뷰 PR 만들어줘
```

### 운영 배포

```text
/{owner}/{repo}.git {TICKET} {yyyymmdd} 운영 배포해줘
```

## 예시

```text
/tmobi-internal/tmap-drive-user-api.git TMAPDRIVER-11531 dev 배포해줘
/tmobi-internal/tmap-drive-user-api.git TMAPDRIVER-11531 stg 배포해줘
/tmobi-internal/tmap-drive-user-api.git TMAPDRIVER-11531 코드리뷰 PR 만들어줘
/tmobi-internal/tmap-drive-user-api.git TMAPDRIVER-11531 20260316 운영 배포해줘
```

## 메모

- 저장소 입력은 앞 `/` 제거, 뒤 `.git` 제거 후 사용
- 운영 배포는 `ticket + yyyymmdd`가 같이 있어야 한다
