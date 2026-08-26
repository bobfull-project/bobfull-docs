# TS-01 — CI/CD Pipeline 및 Docker Layer 구조 개선

> 원본: `[배포 최적화] CI/CD Pipeline 및 Docker Layer 구조 개선` · 작성자: 김홍기

## 문제 정의

GitHub Actions 전체 Workflow가 약 **16m 51s** 걸렸고 동일 Commit을 배포하는 과정에서 Gradle/Docker Build가 반복됐다.

```text
CI Gradle Build/Test
→ CI Docker Build
→ Dockerfile 내부 bootJar
→ CD Docker Build
→ Dockerfile 내부 bootJar
→ ECR Push
→ SSM Deploy
```

Docker Image도 약 190MB Fat JAR가 하나의 Layer여서 실제 Application Class가 약 1.4MiB인데 코드 일부 변경만으로 약 **199MB Layer 전체**가 바뀌었다.

## 가설

- CI 결과물을 CD에서 재사용하면 중복 Build를 제거할 수 있다.
- 고정 sleep 대신 실제 상태 polling을 사용하면 불필요한 대기를 줄일 수 있다.
- Dependency와 Application Layer를 분리하면 코드 변경 범위를 줄일 수 있다.
- Runtime Image를 경량화할 수 있다.

## 1차 개선 — Pipeline

```text
CI
Gradle Test + bootJar
→ Artifact Upload

CD
Artifact Download
→ Docker Build 1회
→ ECR Push
→ SSM Deploy
```

- CI Docker Build 제거
- Dockerfile 내부 Gradle Build 제거
- SSM polling `10초 → 3초`
- 컨테이너 실행 후 고정 sleep 제거
- Parameter Store 중복 조회 제거
- 테스트의 고정 `Thread.sleep`을 조건 기반 대기로 변경

## 2차 개선 — Layered JAR + Distroless

Spring Boot Layered JAR를 사용해 다음으로 분리했다.

| Layer | 크기 |
|---|---:|
| dependencies | 약 198MB |
| spring-boot-loader | 약 696KB |
| snapshot-dependencies | 약 4.1KB |
| application | 약 **2.83MB** |

Runtime 후보는 Temurin JRE, Alpine, Distroless Java17을 비교했다. Alpine이 가장 작았지만 `musl libc` 기반 Native Library 호환 위험을 고려해 **glibc 기반 Distroless Java 17**을 선택했다.

## 결과

| 항목 | Before | After | 변화 |
|---|---:|---:|---:|
| 전체 Workflow | 16m 51s | 14m 37s | 약 13.3% 감소 |
| CD Job | 5m 06s | 3m 29s | 약 31.7% 감소 |
| Docker Build + ECR Push | 1m 46s | 32s | 약 69.8% 감소 |
| Docker Desktop Size | 약 838MB | 약 697MB | 약 16.8% 감소 |
| Docker Inspect Size | 약 299MB | 약 262.6MB | 약 12.1% 감소 |
| 코드 변경 주요 Layer | 약 199MB | 약 2.83MB | 변경 범위 축소 |
| 단일 EC2 관측 다운타임 | 48.65s | 약 40.25s | 약 17.3% 감소 |

Application Resource를 변경해 재빌드했을 때 Dependency Layer는 Cache가 재사용되고 Application Layer만 변경되는 것도 확인했다.

## 한계

이 최적화는 배포 작업량을 줄였지만 **단일 EC2에서 컨테이너를 교체하는 구조 자체를 바꾸지 않았기 때문에 서비스 중단을 제거하지 못했다.** 이후 Multi-AZ Blue-Green에서 배포 가용성 문제를 별도로 해결했다.

또 Gradle Build/Test는 여전히 약 10분 이상으로 가장 큰 병목이며 Kafka 통합 테스트/Testcontainers가 주요 후보였지만 테스트 신뢰성을 해칠 수 있어 이 작업 범위에서는 줄이지 않았다.

## 관련 문서

- [ER-05 — CI/CD 기술 기록](../../engineering-records/cicd-evolution.md)
- [CS-01 — SPOF → Blue-Green](../../case-studies/cs-01-spof-to-multi-az-blue-green.md)
