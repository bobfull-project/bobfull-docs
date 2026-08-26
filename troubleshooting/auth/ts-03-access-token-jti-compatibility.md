# TS-03 — Access Token jti 하위 호환 버그

> 원본 버전: V2 · Issue #186 · 작성자: 정용태

## 문제 정의

Access Token Blacklist 구현을 위해 새 토큰에 `jti` Claim을 추가하고 파싱 시 필수값으로 검증했다. PR 리뷰에서 **배포 이전에 이미 발급된 Access Token에는 jti가 없다는 점**이 발견됐다.

당시 배포는 단일 EC2 컨테이너 교체 방식이고 프론트에는 401 자동 reissue 흐름이 없었다. 그대로 배포하면 기존 활성 토큰이 모두 “필수 Claim 없음”으로 거절돼 사용자가 재로그인해야 할 수 있었다.

## 해결

`jti`를 하위 호환 기간 동안 nullable로 취급했다.

1. `parseAccessTokenClaims`의 필수 Claim에서 jti 제외
2. `jti != null`일 때만 Blacklist 조회
3. 로그아웃도 jti가 있을 때만 Access Token Blacklist 등록
4. Refresh Token 삭제는 jti 여부와 무관하게 항상 수행

```text
기존 토큰(jti 없음)
→ 인증 허용
→ Blacklist 조회/등록 skip

신규 토큰(jti 있음)
→ Blacklist 정책 정상 적용
```

## 검증

- jti 없는 토큰 파싱 성공
- jti 없는 토큰 인증 성공 + Blacklist Store 호출 없음
- jti 없는 토큰 로그아웃 시 Blacklist만 skip, Refresh Token은 삭제
- 전체 테스트 **716개** 재실행, 회귀 없음

이 완화는 영구 호환 정책이 아니라 배포 이전 토큰의 최대 기존 수명인 약 **1시간** 동안만 의미가 있다. 이후 활성 토큰은 모두 jti를 가진다.

## 배운 점

JWT Claim이나 DB Column처럼 **이미 존재하는 데이터에 새 필수값을 추가할 때는 코드 정합성뿐 아니라 배포 순간의 기존 데이터 호환성**을 봐야 한다.

이 사례는 Auth 문제이면서 동시에 Deployment Migration 문제다. 문서는 한 위치에만 두고 Troubleshooting 인덱스에서 두 관점으로 연결한다.

## 관련 문서

- [TD-02 — 예약 운영 정책과 인증 무효화 설계](../../decisions/td-02-reservation-auth-operational-decisions.md)
