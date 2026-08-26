# PortOne SDK 동기 대기와 외부 API 예외 변환 미검증

> 원본 Notion 페이지는 제목만 있고 본문이 비어 있어, Backend의 기존 `docs/troubleshooting/결제_트러블슈팅.md` 기록을 기준으로 복원했다. · 작성자: 김현승

## 문제 정의

PortOne 결제 단건 조회 Adapter는 SDK의 비동기 결과를 `join()`으로 기다리는 동기 방식이었다. 정상 응답만 보면 단순하지만 외부 PG가 느려지거나 Timeout·네트워크 오류·SDK 예외가 발생할 때 다음 문제가 있었다.

- 요청 스레드가 외부 응답을 기다린다.
- SDK 예외가 우리 API의 어떤 ErrorCode/HTTP Status로 변환되는지 계약이 불명확하다.
- 외부 연동 실패와 결제 검증 실패를 같은 예외로 처리하면 운영 원인을 구분하기 어렵다.
- Mock 중심 테스트만으로 실제 SDK Timeout과 예외 계층을 충분히 검증하기 어렵다.

## 확인한 원인

핵심은 “동기 호출 자체가 항상 잘못”인 것이 아니라 **외부 I/O를 기다리는 구간의 최대 시간과 실패 변환 정책이 명확하지 않은 상태**였다.

```text
HTTP 요청
→ PortOne SDK async call
→ join()
→ 외부 응답 대기
→ 성공/예외 변환
```

외부 API에 대한 대기 시간이 길어지면 App Thread가 오래 점유될 수 있고, 예외 변환이 불명확하면 동일 장애가 4xx·5xx 중 어느 쪽으로 노출돼야 하는지 흔들릴 수 있다.

## 대응 방향

- PortOne 연동 Adapter에서 외부 예외를 애플리케이션 예외 계약으로 명시적으로 변환한다.
- 네트워크/외부 서비스 오류와 내부 결제 검증 실패를 분리한다.
- Timeout은 “업무 실패”와 “결과 불명확”을 구분해 다룬다.
- 실제 SDK/테스트 채널에서 Timeout·5xx·비정상 응답을 확인하기 전에는 완전히 검증됐다고 주장하지 않는다.

이 문제는 이후 환불 설계에서 더 구체화됐다. 특히 금전 처리 Timeout은 외부에서 이미 성공했을 수 있으므로 단순 Retry보다 **현재 PortOne 상태를 재조회하는 Reconciliation**이 필요했다.

## 현재 상태와 한계

Backend 원본 기록 자체도 이 항목을 “실제 연동 환경에서 충분히 검증되지 않은 부분”으로 남겼다. 따라서 이 문서 역시 해결 완료 수치를 만들지 않고 **외부 API 실패 계약과 후속 검증 필요성을 보존**한다.

## 관련 문서

- [환불 Timeout Reconciliation](./refund-timeout-reconciliation.md)
- [Email 외부 I/O 스레드 격리](../event-processing/email-outbox-request-latency.md)
- [Backend 결제 트러블슈팅 원본](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/troubleshooting/%EA%B2%B0%EC%A0%9C_%ED%8A%B8%EB%9F%AC%EB%B8%94%EC%8A%88%ED%8C%85.md)
