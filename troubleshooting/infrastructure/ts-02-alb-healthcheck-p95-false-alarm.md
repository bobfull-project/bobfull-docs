# TS-02 — ALB Health Check 요청으로 발생한 p95 응답시간 알람 오탐 개선

> 작성자: 김홍기

## 문제 정의

ALB 적용 후 Grafana에서 p95 응답시간 알람이 반복됐지만 실제 사용자 API를 확인하면 성능 저하가 재현되지 않았다.

## 가설

ALB가 `/actuator/health`를 주기적으로 호출하고 있었고 이 요청이 일반 API와 같은 Prometheus HTTP latency 지표에 포함돼 p95를 왜곡한다고 판단했다.

## 해결

Health Check는 서비스 라우팅에 필요하므로 제거하지 않고 **사용자 API 성능 지표에서만 제외**했다.

Grafana Alert Rule의 PromQL에 다음 조건을 추가했다.

```promql
uri!="/actuator/health"
```

기존 `/actuator/prometheus` 제외와 함께 내부 관측 요청이 사용자 API latency에 섞이지 않도록 분리했다. Provisioning 설정을 Monitoring EC2에 반영하고 Grafana 컨테이너 재시작 후 실제 Alert Query에 적용된 것을 확인했다.

## 결과

ALB Health Check가 사용자 API p95 계산에서 제외돼 **인프라 상태 확인 트래픽과 사용자 요청 성능을 다른 관점으로 관찰**할 수 있게 됐다.

## 배운 점

모든 HTTP 요청을 하나의 latency 집계에 넣으면 Health Check·Metrics Scrape 같은 내부 트래픽이 사용자 경험 지표를 왜곡할 수 있다. Metric은 수집하는 것만큼 **무엇을 포함·제외하는가**가 중요하다.

## 관련 문서

- [ER-07 — 운영 관측 체계 구축](../../engineering-records/monitoring-observability.md)
