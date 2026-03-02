# 🏦 Loan Core RCA Project (FINERACT-1971)

이 저장소는 **Apache Fineract** 대출 모듈의 Re-aging 로직에서 발견된 원금 계산 버그를 분석(RCA)하고 해결한 프로젝트입니다.

---

## 🚀 RCA Report: 상환액 무시 버그 수정
### 🔍 문제 정의 (Issue)
대출 실행 후 일부 금액을 상환(Repayment)한 상태에서 **Re-aging**을 진행할 때, 시스템이 남은 원금 잔액이 아닌 **최초 승인 금액(Approved Principal)**을 기준으로 트랜잭션을 생성하는 결함이 발견되었습니다.

### 🧪 버그 재현 (Reproduction)
- **재현 경로**: `integration-tests/src/test/java/org/apache/fineract/integrationtests/loan/reaging/LoanReAgingIntegrationTest.java`
- **시나리오**:
  1. **1,250원** 대출 실행 (1월 1일)
  2. **250원** 중도 상환 완료 (1월 15일) -> **실제 잔액: 1,000원**
  3. **Re-aging** 실행 (2월 2일)
  4. **결과**: 트랜잭션이 1,000원이 아닌 **1,250원**으로 생성됨 (버그 확인 ❌)

### 🛠 해결 방법 (Fix)
- **수정 부위**: `LoanReAgingServiceImpl.java`
- **변경 사항**:
  - 기존: `loan.getApprovedPrincipal()` (최초 승인액 참조)
  - 수정: `loan.getTotalPrincipalOutstandingUntil(transactionDate)` (특정 시점의 실제 원금 잔액 참조)
- **결과**: 테스트 케이스 통과 및 정상 잔액(1,000원) 반영 확인 ✅

---

## 📖 About Apache Fineract
본 프로젝트는 [Apache Fineract](https://github.com/apache/fineract)를 기반으로 합니다. Fineract는 전 세계 금융 소외 계층을 위한 오픈소스 코어 뱅킹 플랫폼입니다.

- **Build**: `./gradlew clean bootJar`
- **Test**: `./gradlew test`