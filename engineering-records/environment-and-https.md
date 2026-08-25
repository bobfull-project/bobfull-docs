# [배포] 운영 환경 분리와 HTTPS 진입 구조

## 1. 로컬과 운영 설정을 분리한 이유

같은 Spring Boot 애플리케이션이라도 로컬과 AWS 운영 환경에서는 DB 주소, 외부 API Key, CORS 허용 주소, 실행 방식이 달랐습니다.

그래서 Spring Profile을 기준으로 설정을 분리했습니다.

```text
application.yml
application-local.yml
application-prod.yml
```

실제 비밀값은 설정 파일에 직접 넣지 않고 환경변수로 주입했습니다.

```text
로컬
.env
→ application-local.yml

운영
AWS Systems Manager Parameter Store
→ Docker 환경변수
→ application-prod.yml
```

`.env`, 실제 로컬 설정 파일은 Git과 Docker Image에 포함되지 않도록 관리하고, 팀원에게 필요한 변수 목록은 `.env.example`과 예시 설정 파일로 공유했습니다.

원본 기록:
- [배포 #2 - 로컬·운영 환경 설정 분리](https://velog.io/@gpekd5/%EC%B5%9C%EC%A2%85-%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8-%EB%B0%B0%ED%8F%AC-1-%EB%A1%9C%EC%BB%AC%EC%9A%B4%EC%98%81-%ED%99%98%EA%B2%BD-%EC%84%A4%EC%A0%95-%EB%B6%84%EB%A6%AC)

## 2. 초기 외부 진입 구조의 문제

V2 초반에는 브라우저가 App EC2의 8080 포트로 직접 접근했습니다.

```text
Browser
→ Internet
→ App EC2 :8080
```

서비스는 동작했지만 다음 문제가 있었습니다.

- App EC2가 인터넷에 직접 노출됨
- HTTPS 미적용
- 이후 App EC2를 여러 대로 늘릴 경우 동일 진입 구조를 다시 설계해야 함

## 3. ALB + HTTPS로 변경

최종적으로 백엔드 외부 진입은 다음 구조로 변경했습니다.

```text
Client
→ Route 53
→ ALB :443
→ Target Group
→ App EC2 :8080
```

- ACM 인증서로 TLS 종료
- HTTP 80 요청은 HTTPS 443으로 전환
- App EC2 8080은 인터넷 전체가 아니라 ALB Security Group에서 오는 요청만 허용
- Target Group Health Check로 App 상태 확인

이를 통해 보안 문제를 줄이는 동시에 이후 App EC2 이중화와 Blue-Green 배포에서 같은 ALB를 그대로 활용할 수 있었습니다.

원본 기록:
- [V2 인프라 구성 정리 및 후속 개선 계획](https://velog.io/@gpekd5/%EC%B5%9C%EC%A2%85-%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8-V2-%EC%9D%B8%ED%94%84%EB%9D%BC-%EA%B5%AC%EC%84%B1-%EC%A0%95%EB%A6%AC-%EB%B0%8F-%ED%9B%84%EC%86%8D-%EA%B0%9C%EC%84%A0-%EA%B3%84%ED%9A%8D)
- [배포 #10 - 백엔드 진입 구조 및 HTTPS 개선 검토](https://velog.io/@gpekd5/%EC%B5%9C%EC%A2%85-%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8-%EB%B0%B0%ED%8F%AC-10-%EB%B0%B1%EC%97%94%EB%93%9C-%EC%A7%84%EC%9E%85-%EA%B5%AC%EC%A1%B0-%EB%B0%8F-HTTPS-%EA%B0%9C%EC%84%A0-%EA%B2%80%ED%86%A0)

## 4. 배포 제어는 SSH 대신 SSM 사용

GitHub Actions에서 EC2에 직접 SSH로 접속하는 대신 AWS Systems Manager Run Command를 사용했습니다.

```text
GitHub Actions
→ ECR Push
→ SSM Run Command
→ App EC2
→ ECR Pull
→ Docker 실행
```

운영 Secret은 Parameter Store에서 가져오고, 배포 명령은 SSM을 통해 전달해 SSH 배포용 포트를 별도로 열지 않는 방향을 유지했습니다.

## 관련 문서

- [System Architecture](../architecture/system-architecture.md)
- [[배포] Blue-Green 무중단 배포와 롤백](./blue-green-deployment.md)
