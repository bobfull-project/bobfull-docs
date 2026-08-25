# [인프라] Presigned URL과 Lambda 이미지 검증 파이프라인

## 1. 이미지 파일을 App EC2가 직접 받지 않도록 분리

식당과 메뉴 이미지를 App EC2의 로컬 디스크에 저장하면 서버 교체나 다중 App 구성에서 파일 상태를 맞추기 어렵습니다. DB에 바이너리 데이터를 직접 저장하는 방식도 데이터베이스 용량과 백업 부담이 커질 수 있습니다.

그래서 서비스 이미지는 S3에 저장하고, 업로드 데이터 자체는 백엔드 서버를 거치지 않도록 Presigned URL 방식을 사용했습니다.

```text
Browser
→ Backend에 업로드 URL 요청
→ Backend가 Presigned PUT URL 발급
→ Browser가 S3로 직접 업로드
```

백엔드는 업로드 권한과 저장 Key를 결정하지만 실제 이미지 바이트를 중계하지 않습니다.

## 2. 업로드만 성공했다고 신뢰하지 않음

Presigned URL로 S3 직접 업로드를 허용하면 사용자가 전달한 파일이 실제 이미지인지 별도로 검증해야 했습니다.

따라서 바로 최종 경로에 저장하지 않고 임시 경로를 거치도록 구성했습니다.

```text
1. Backend
   Presigned PUT URL 발급
   prefix = temp/restaurants/

2. Browser
   → S3 임시 경로 직접 업로드

3. S3 ObjectCreated Event
   → Lambda Image Validator 실행

4. Lambda
   ├─ 검증 성공 → restaurants/ 최종 경로로 복사
   └─ 검증 실패 → 임시 객체 삭제

5. Lambda 실행 로그
   → CloudWatch Logs
```

## 3. 구성 요소별 책임

| 구성 요소 | 책임 |
|---|---|
| Spring Boot | 업로드 요청 검증, Presigned URL 발급, 저장 Key 관리 |
| Browser | 발급받은 URL을 이용해 S3에 직접 PUT |
| S3 | 임시·최종 이미지 저장, ObjectCreated Event 발생 |
| Lambda | 업로드된 파일 검증, 성공 시 최종 경로 이동, 실패 시 정리 |
| CloudWatch Logs | Lambda 검증 결과와 오류 확인 |

## 4. 이 구조를 선택한 이유

### App 서버의 파일 전송 부담 분리

```text
직접 업로드
Browser → App EC2 → S3

Presigned URL
Browser ─────────→ S3
         App은 URL 발급만 수행
```

이미지 파일 자체가 App EC2 네트워크와 메모리를 거쳐가지 않으므로 애플리케이션 서버는 API 처리에 집중할 수 있습니다.

### App 서버가 늘어나도 파일 상태를 공유할 필요가 없음

이미지가 EC2 로컬에 남지 않기 때문에 App EC2 #1과 #2 사이에 파일을 복제할 필요가 없습니다.

### 검증 책임을 비동기로 분리

업로드 API에서 파일 전체를 검사하는 대신 S3 Event 이후 Lambda가 검증하도록 분리했습니다. 업로드와 검증의 실행 자원을 독립시킬 수 있고 검증 실패 파일은 최종 경로로 승격하지 않습니다.

## 5. 실제 AWS 구조에서의 위치

최종 시스템에서는 프론트엔드 정적 파일용 S3와 서비스 이미지용 S3의 목적을 분리하고, 이미지 업로드 경로에 Lambda Image Validator를 연결했습니다.

<img width="1642" height="952" alt="BobFull System Architecture" src="https://github.com/user-attachments/assets/5a1371a7-7486-4fca-8a8a-43f8f1c44995" />

## 6. 운영 시 확인할 지점

- Presigned URL 만료 시간
- 허용 Content-Type과 업로드 경로
- S3 Event가 Lambda를 정상 호출하는지
- 검증 성공 객체가 최종 경로에 존재하는지
- 검증 실패 객체가 임시 경로에 남지 않는지
- Lambda 오류는 CloudWatch Logs에서 추적 가능한지

## 관련 기록

- Velog: `[최종 프로젝트] 배포 #6 - S3 Presigned URL과 Lambda를 활용한 식당 이미지 업로드 구현`
- Velog: `[최종 프로젝트] 배포 #7 - 단일 EC2 Docker 환경에 Redis 연결하기`의 앞선 이미지 검증 결과
- [System Architecture](../architecture/system-architecture.md)
