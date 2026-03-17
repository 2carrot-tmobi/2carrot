# github-pr-create-merge Quick Guide

아래 형식으로 요청하면 된다.

## 사용법

### dev 배포

```text
/{owner}/{repo} {TICKET} dev 배포해줘
```

### stg 배포

```text
/{owner}/{repo} {TICKET} stg 배포해줘
```

### 코드리뷰 PR

```text
/{owner}/{repo} {TICKET} 코드리뷰 PR 만들어줘
```

### 운영 배포

```text
/{owner}/{repo} {TICKET} {yyyymmdd} 운영 배포해줘
```

## 예시

```text
/tmobi-internal/tmap-drive-user-api TMAPDRIVER-11531 dev 배포해줘
/tmobi-internal/tmap-drive-user-api TMAPDRIVER-11531 stg 배포해줘
/tmobi-internal/tmap-drive-user-api TMAPDRIVER-11531 코드리뷰 PR 만들어줘
/tmobi-internal/tmap-drive-user-api TMAPDRIVER-11531 20260316 운영 준비 배포해줘
```

## 메모

- 사용 초반에 스킬을 사용과 명시된 대로 작업하라는 학습기간이 필요하다
- 운영 배포는 `ticket + yyyymmdd`가 같이 있어야 한다
- 여러 작업을 병렬로 처리할 수 있다
```text
/tmobi-internal/tmap-drive-user-api TMAPDRIVER-11531 dev 배포랑 코드리뷰 PR 만들어줘
/tmobi-internal/tmap-drive-user-api TMAPDRIVER-11531 stg 배포랑 20260316 운영 준비 배포해줘
```
