# 예약 운영 정책과 인증 무효화 설계

> 원본 제목: `예약 참여자 노쇼 처리, 관리자 전체 노쇼 현황 조회, 사장님(OWNER) 예약 전체 취소, 모집 마감 기한 후 자동 취소 스케줄러, Access Token Blacklist 및 로그아웃 즉시 무효화`  
> 원본 분류: 기술적 의사결정 · 작성자: 정용태 · 관련 Issue: #48, #134, #46, #47, #186

한 페이지에 정리돼 있던 5개의 의사결정을 원본 단위 그대로 보존한다.

## 1. 예약 참여자 노쇼 처리 — Issue #48

OWNER가 식사 종료 후 노쇼 후보를 조회하고 참여자별 노쇼 처리·해제를 할 수 있도록 했다.

동시에 같은 참여자를 처리하면 둘 다 사전 검증을 통과해 이력이 중복될 수 있어 `PESSIMISTIC_WRITE`와 엔티티 상태 전이 가드를 함께 사용했다. 관리자성 저빈도 기능이라 처리량보다 정합성과 기존 코드베이스의 락 전략 일관성을 우선했다.

실제 MySQL 동시성 통합 테스트에서 두 요청 중 **1건 성공·1건 409·이력 1건**만 남는 것을 검증했다.

## 2. 관리자 전체 노쇼 현황 조회 — Issue #134

노쇼 처리→해제→재처리가 반복되면 `marked=true` 이력만 필터링해서는 같은 참여자가 여러 번 보일 수 있다. 필요한 것은 “현재 marked=true”가 아니라 **참여자별 최신 이력이 marked=true인지**였다.

SQL Window Function, JPQL 서브쿼리, 평면 조회 후 Java grouping을 비교했고, 당시 JPQL 제약과 기존 #48 구현 패턴을 고려해 **평면 조회 + Java grouping + 수동 페이지네이션**을 재사용했다.

DB 페이지네이션을 직접 쓰지 못해 대상 전체를 메모리에 올리는 단점은 현재 데이터 규모에서 감수하고, 규모가 커지면 QueryDSL/Native Query/조회 모델을 재검토한다.

## 3. OWNER 예약 전체 취소 — Issue #46

Issue 작성 당시에는 “Reservation 즉시 CANCELLED, 실제 환불은 후속 작업”이라는 계약이 있었지만 작업을 재개할 때는 이미 공통 취소·환불 아키텍처가 merge돼 있었다.

오래된 Issue 계약을 그대로 구현하면 `Refund=REQUESTED`가 남는 등 최신 상태 모델과 충돌하므로 **기존 공통 파이프라인(접수 → Transaction 밖 외부 환불 → 완료 확정)**을 재사용했다.

이 경험 이후 보류된 Issue를 다시 시작할 때는 “작성 당시 계약이 현재 아키텍처와 여전히 맞는가?”를 다시 검증하는 기준을 두었다.

## 4. 모집 마감 후 자동 취소 Scheduler — Issue #47

식사 시작 2시간 전 확정 인원이 기준 미달이면 자동으로 취소·전액 환불을 접수한다.

분산 락 도입도 검토했지만 당시 실제 배포는 단일 App 인스턴스였고 기존 Scheduler들도 분산 락을 사용하지 않았다. 따라서 새 인프라 의존성을 추가하지 않고 **항목별 비관적 락 + 처리 직전 상태 재확인**으로 중복 처리를 막았다.

다중 App에서 같은 Scheduler가 동시에 실행되는 운영 조건이 생기면 ShedLock 등 분산 실행 제어를 다시 검토한다.

## 5. Access Token Blacklist와 로그아웃 즉시 무효화 — Issue #186

로그아웃 시 Refresh Token 삭제만으로는 이미 발급된 Access Token이 만료까지 계속 유효하다. 이를 즉시 무효화하기 위해 Access Token에 `jti`를 부여하고 Redis Blacklist를 사용하도록 확장했다.

이 결정은 이후 **기존 jti 없는 토큰의 하위 호환 문제**를 만들었다. 배포 전 발급 토큰을 곧바로 거절하지 않도록 `jti`를 한시적으로 nullable로 처리하고, jti가 있는 경우에만 Blacklist 조회·등록하도록 보완했다.

세부 배포 호환성 문제는 별도 트러블슈팅으로 분리한다.

## 관련 문서

- [Access Token jti 하위 호환 버그](../troubleshooting/auth/access-token-jti-compatibility.md)
- [비관적 락 이후에도 Reservation이 CANCELLING에 멈춘 문제](../troubleshooting/reservation/pessimistic-lock-cancelling-state.md)
