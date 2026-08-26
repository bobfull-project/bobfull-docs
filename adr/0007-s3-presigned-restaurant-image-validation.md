# ADR-0007 — S3 Presigned URL + Lambda 이미지 검증

> 포트폴리오 요약본  
> Source of Truth: [Backend ADR-0007](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/adr/0007-s3-presigned-restaurant-image-validation.md)

## 배경

식당 이미지가 Spring Boot 서버를 거쳐 S3로 올라가면 파일 전송량에 따라 App EC2의 네트워크·메모리·응답시간 부담이 증가한다. Presigned URL을 사용하면 Browser가 S3로 직접 업로드할 수 있지만, 파일명·확장자·Content-Type·요청 크기는 클라이언트가 제공한 값이라 실제 파일을 보장하지 않는다.

따라서 다음 두 조건을 동시에 만족해야 했다.

1. 이미지 바이트는 App EC2를 거치지 않는다.
2. S3에 실제 저장된 객체를 기준으로 검증한다.

## 결정

`Presigned PUT → S3 temp → ObjectCreated Event → Java Lambda 검증 → final` 구조를 선택했다.

```text
Browser
→ Presigned PUT URL
→ S3 temp/restaurants/{ownerId}/{uuid}.{ext}
→ S3 ObjectCreated
→ Java Lambda
→ 실제 size / MIME / file signature 검증
→ restaurants/{ownerId}/{uuid}.{ext}
```

- 허용 형식: JPEG, PNG
- 최대 크기: 5MB
- Presigned PUT 유효시간: 5분
- 업로드마다 UUID Key 발급
- 검증 성공 객체만 식당과 연결
- DB에는 만료되는 URL이 아니라 최종 Object Key 저장
- 조회 시 Presigned GET URL 생성
- 다른 OWNER의 Key 사용 방지
- 실패·미사용 temp 객체 정리

## 대안 비교

| 방식 | 장점 | 단점 |
|---|---|---|
| Backend Multipart | 검증 흐름 단순 | 이미지가 App EC2를 거침 |
| Presigned URL only | 가장 단순, 서버 부하 낮음 | 실제 객체 검증 부족 |
| **Presigned + Lambda** | 서버 부하 분리 + 실제 파일 검증 | S3 Event/Lambda 운영과 비동기 지연 추가 |

V1에서는 일반적인 식당 대표 이미지를 충분히 지원하면서 검증 범위를 제한하기 위해 JPEG/PNG, 5MB를 선택했다. WebP 등 추가 형식은 후속 확장 범위다.

## Trade-off

App 자원과 파일 검증 책임을 분리하는 대신 Lambda 배포·CloudWatch 로그·S3 temp/final lifecycle이라는 운영 요소가 추가된다.

## 관련 문서

- [ER-02 — 이미지 업로드 파이프라인 기술 기록](../engineering-records/image-upload-pipeline.md)
