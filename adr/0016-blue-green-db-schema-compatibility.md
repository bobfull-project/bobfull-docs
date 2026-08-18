# ADR 0016: Blue-Green 공존 기간의 DB Schema 호환 전략

> BobFull Backend의 전체 ADR 19개 중 포트폴리오에서 보여줄 대표 의사결정을 요약한 문서입니다. 상세 원본은 [Backend ADR](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/adr/0016-blue-green-db-schema-compatibility.md)을 기준으로 합니다.

## 문제

Blue와 Green 애플리케이션은 같은 RDS MySQL을 공유합니다. App Traffic은 ALB로 되돌릴 수 있지만, Green 배포 과정에서 기존 컬럼을 drop/rename하면 Traffic을 Blue로 롤백해도 구버전이 현재 Schema와 호환되지 않을 수 있습니다.

## 결정

Production에서는 `ddl-auto=validate`를 사용하고, Blue-Green Rollback 가능 기간에는 **additive schema change를 우선**합니다.

```text
Additive Schema 변경
→ 기존 Blue 호환 확인
→ Green 배포 / validate / 대표 API 검증
→ Blue·Green 동일 RDS 공존 확인
→ Traffic Switch
→ Rollback Window 유지
→ 안정화 이후 후속 정리
```

Rollback Window에서는 기존 컬럼 즉시 `DROP/RENAME`, 구버전이 사용하는 table/column 제거, 호환성을 깨는 type/constraint 변경을 한 번에 수행하지 않습니다.

## 왜 이 선택을 했나

운영 App 기동 시 자동 Schema 변경을 막으면서도, 프로젝트 최종 단계에서 Flyway/Liquibase를 새로 도입하는 운영 비용은 피하고 실제 Blue/Green 공존과 Rollback 호환성을 먼저 검증하기 위해서입니다.

## 검증

- Production `ddl-auto=update → validate` 전환 후 Green readiness/API 정상
- 기존 DB에 nullable column/index 추가 후 데이터 보존
- 같은 변경 Schema에서 Green 신버전과 Blue 구버전 모두 `validate` 기동
- Blue/Green이 동일 RDS에서 대표 API를 처리하고 동일 데이터를 읽는지 확인

## 트레이드오프

Flyway/Liquibase 같은 versioned migration history가 없어 SQL 적용 순서와 이력을 직접 관리해야 합니다. 검증 범위도 nullable additive change 중심이므로 destructive migration의 안전성을 주장하지 않습니다.
