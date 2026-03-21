# github-pr-create-merge Quick Guide

아래 형식으로 요청하면 된다.

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
