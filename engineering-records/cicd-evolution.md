# [배포] GitHub Actions CI/CD 구축과 배포 과정 최적화

## 1. 수동 배포에서 시작

초기에는 Docker 이미지를 직접 ECR에 Push하고, EC2의 Session Manager에 접속해 최신 이미지를 내려받은 뒤 컨테이너를 교체했습니다.

기능 검증에는 충분했지만 배포할 때마다 같은 작업을 반복해야 했고, 사람마다 순서가 달라질 가능성도 있었습니다.

그래서 다음 흐름을 GitHub Actions로 자동화했습니다.

```text
코드 변경
→ GitHub Actions
→ Gradle Build / Test
→ Docker Image Build
→ ECR Push
→ SSM Run Command
→ EC2 Image Pull
→ 컨테이너 교체
```

EC2 배포 명령은 SSH 대신 AWS Systems Manager Run Command를 사용했고, GitHub Actions는 OIDC로 AWS 임시 권한을 받아 ECR·SSM에 접근하도록 구성했습니다.

## 2. 자동화 뒤에도 배포가 오래 걸렸다

CI/CD를 만들었다고 끝내지 않고 Workflow 시간을 단계별로 측정했습니다.

초기 측정 결과:

| 항목 | Before |
|---|---:|
| 전체 Workflow | `16m 51s` |
| CI Job | `11m 39s` |
| Gradle Build / Test | `10m 15s` |
| CI Docker Build | `1m 14s` |
| CD Job | `5m 06s` |
| Docker Build + ECR Push | `1m 46s` |
| SSM Deploy | `2m 54s` |

하나의 Commit을 배포하면서 Gradle Build와 Docker Build가 여러 번 반복되고 있었습니다.

```text
기존
CI Gradle Build
→ CI Docker Build
→ Dockerfile 내부 bootJar
→ CD Docker Build
→ Dockerfile 내부 bootJar
→ ECR Push
→ SSM Deploy
```

## 3. 1차 개선 — 중복 Build와 고정 대기 제거

### CI Docker Build 제거

CI에서는 테스트와 `bootJar` 생성까지만 수행하고 JAR를 Artifact로 전달했습니다.

```text
CI
Gradle Test + bootJar
→ JAR Artifact Upload

CD
Artifact Download
→ Docker Build + ECR Push 1회
```

### Dockerfile 내부 Gradle Build 제거

GitHub Actions에서 이미 생성한 JAR를 Runtime Image에 복사하도록 Dockerfile을 단순화했습니다.

```dockerfile
FROM eclipse-temurin:17-jre
WORKDIR /app
COPY build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

### SSM 고정 대기 제거

- SSM polling 간격 `10초 → 3초`
- 컨테이너 실행 후 고정 `sleep 10초` 제거
- 고정 대기 대신 Readiness를 즉시 확인
- Parameter Store Required Parameter 중복 조회 제거

시간을 임의로 기다리는 대신 실제 애플리케이션 준비 상태를 확인하는 방향으로 바꿨습니다.

## 4. 2차 개선 — Docker Image와 변경 Layer 축소

배포 시 매번 큰 Layer를 다시 전송하는 문제도 줄였습니다.

| 항목 | Before | After |
|---|---:|---:|
| Docker Image | 약 `838MB` | 약 `697MB` |
| Inspect Size | 약 `299MB` | 약 `262.6MB` |
| 변경 Layer | 약 `199MB` | 약 `2.83MB` |

## 5. 단계별 결과

```text
Before
전체 Workflow 16m 51s
단일 EC2 배포 중단 48.65s
Docker Image 약 838MB
변경 Layer 약 199MB

↓

1차 - CI/CD Pipeline 최적화
전체 Workflow 14m 37s
CD Job 5m 06s → 3m 29s
단일 EC2 배포 중단 46.42s

↓

2차 - Docker Image / Layer 최적화
Docker Image 약 838MB → 697MB
변경 Layer 약 199MB → 약 2.83MB
단일 EC2 배포 중단 41.36s
```

> 위 `48.65s → 41.36s`는 CI/CD·Docker 최적화 실험에서 측정한 단일 EC2 배포 구간입니다. 최종 인프라 회고에서 별도로 기록한 단일 배포 약 `40.25초`와 같은 측정값으로 합치지 않습니다.

## 6. CI/CD 최적화만으로는 해결되지 않은 것

배포 Workflow는 줄었지만 단일 EC2에서 컨테이너를 교체하는 한 서비스 요청을 처리하지 못하는 구간 자체는 남았습니다.

즉 두 문제를 분리해야 했습니다.

```text
CI/CD가 느림
→ 중복 Build / Docker Layer / 고정 대기 최적화

배포 중 서비스가 멈춤
→ 서버를 교체하는 구조 자체를 변경
→ ALB + App 이중화 + Blue-Green
```

그래서 다음 단계에서는 배포 시간을 더 줄이는 것보다 **기존 Active 서버가 요청을 처리하는 동안 새 버전을 별도 환경에 배포하는 Blue-Green 구조**로 넘어갔습니다.

## 관련 문서

- [[배포] Blue-Green 무중단 배포와 롤백](./blue-green-deployment.md)
- [[배포] 운영 환경 분리와 HTTPS 진입 구조](./environment-and-https.md)
- [System Architecture](../architecture/system-architecture.md)
