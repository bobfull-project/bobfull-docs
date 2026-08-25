# [모니터링] 운영 관측 체계 구축

## 1. 로그만으로 부족했던 이유

초기에는 CloudWatch Logs와 구조화 로그를 이용해 개별 오류 원인을 추적할 수 있었습니다.

하지만 로그를 직접 검색하는 것만으로는 다음 상태를 계속 확인하기 어려웠습니다.

- 최근 오류가 증가하고 있는지
- API 응답 시간이 느려지는지
- CPU·JVM 메모리가 올라가는지
- DB Connection Pool이 포화되는지
- 예약·결제·환불 같은 비즈니스 이벤트가 정상적으로 처리되는지

그래서 **사건의 상세 원인은 로그, 시간에 따른 변화는 메트릭**으로 역할을 나눴습니다.

## 2. 도구 선택

모니터링 도구는 기능 수보다 프로젝트 규모, Spring Boot 연동, 운영 비용을 함께 비교했습니다.

Velog 기록에서 Datadog, New Relic, ELK Stack, Zabbix, Prometheus + Grafana를 비교했고 최종적으로 다음 조합을 선택했습니다.

| 도구 | 역할 |
|---|---|
| CloudWatch | AWS 리소스 상태 확인 |
| CloudWatch Logs | 애플리케이션 구조화 로그 저장·검색 |
| Prometheus | 애플리케이션·비즈니스 메트릭 수집 |
| Grafana | 메트릭 시각화 및 Alert |
| Slack | 운영 알림 수신 |

Datadog과 New Relic은 기능은 강했지만 프로젝트 규모에서 비용·기능 범위가 과하다고 판단했고, ELK는 이미 사용 중인 CloudWatch Logs와 역할이 겹쳤습니다.

원본 기록:
- [모니터링이 필요한 이유와 모니터링 툴 선정](https://velog.io/@gpekd5/%EC%B5%9C%EC%A2%85-%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8-%EB%AA%A8%EB%8B%88%ED%84%B0%EB%A7%81%EC%9D%B4-%ED%95%84%EC%9A%94%ED%95%9C-%EC%9D%B4%EC%9C%A0%EC%99%80-%EB%AA%A8%EB%8B%88%ED%84%B0%EB%A7%81-%ED%88%B4-%EC%84%A0%EC%A0%95)
- [Prometheus와 Grafana란?](https://velog.io/@gpekd5/%EC%B5%9C%EC%A2%85-%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8-Prometheus%EC%99%80-Grafana%EB%9E%80)

## 3. 최종 수집 흐름

```text
Spring Boot
→ Actuator + Micrometer
→ /actuator/prometheus
→ Prometheus
→ Grafana
→ Grafana Alerting
→ Slack
```

Prometheus와 Grafana는 App EC2와 분리한 Monitoring EC2에서 실행했습니다.

Blue-Green 배포로 Active App이 바뀌면 Prometheus가 새 Active 2대를 바라보도록 모니터링 대상을 갱신합니다.

## 4. 무엇을 봤는가

### 시스템 지표

- API 요청 수
- API 처리 시간
- HTTP 5xx
- CPU 사용률
- JVM 메모리
- HikariCP Active / Pending

### 비즈니스·운영 관측

- 예약·결제·환불 처리 흐름
- 정합성 이상 상태
- 구조화 오류 로그
- 배포 후 Readiness / Health 상태

사용자 ID, 예약 ID처럼 값의 종류가 계속 늘어나는 데이터는 Prometheus Label로 남기지 않고 로그에서 추적하도록 구분했습니다.

## 5. 장애 대응에서의 역할

운영 관측의 목적은 대시보드를 만드는 것 자체가 아니라 다음 흐름을 만드는 것이었습니다.

```text
이상 징후 감지
→ Grafana Alert / Slack
→ Prometheus 지표 확인
→ CloudWatch / Docker 로그 확인
→ 원인 추적
→ 조치
```

실제로 단일 EC2 장애에서는 CPU보다 메모리와 프로세스 상태를 함께 확인하면서 자원 경쟁 문제를 좁힐 수 있었고, 성능 부하 테스트에서는 CPU뿐 아니라 HikariCP Pending까지 함께 보며 Auto Scaling 적용 여부를 판단했습니다.

## 관련 문서

- [[인프라] 단일 EC2 메모리 장애와 자원 분리](./resource-separation.md)
- [[성능] Query·Cache·Hikari 병목과 확장 판단](./performance-and-scaling.md)
- [System Architecture](../architecture/system-architecture.md)
