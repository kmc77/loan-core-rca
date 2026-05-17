# Loan Re-aging Fault-Injection & RCA Exercise (Apache Fineract / FINERACT-1971)

실 코어뱅킹 코드베이스(Apache Fineract)의 Loan **Re-aging** 경로에 현실적인 원금 계산 결함을
**의도적으로 주입**하고, 결정적(deterministic) 통합테스트로 **재현**한 뒤, 근본 원인을 분석(RCA)하고
정상 로직으로 **원복**하는 과정을 기록한 학습/훈련 프로젝트입니다.

> 이 저장소는 `apache/fineract`의 fork이며, 본 프로젝트가 추가한 작업은
> 결함 주입 커밋과 원복 커밋, 그리고 재현용 통합테스트뿐입니다.

---

## 1. 이 프로젝트가 무엇이고, 무엇이 아닌가

명확히 합니다. 신뢰가 깨질 여지를 먼저 제거합니다.

- **아닙니다**: 이것은 "Apache Fineract에서 발견된 실제 버그"가 아닙니다.
  `FINERACT-1971`은 버그 티켓이 아니라 Re-aging 기능을 처음 구현한 **기능(feature) 티켓**이며,
  해당 코드는 최초 구현 시점부터 잔여원금 기준으로 정상 동작합니다(아래 §6 근거).
- **맞습니다**: 정상 동작하는 코어뱅킹 코드에 **현실적인 결함을 통제된 형태로 주입**하고,
  이를 자동화된 결정적 테스트로 격리·재현한 뒤 RCA를 거쳐 원복하는 **fault-injection 훈련**입니다.

운영 디버깅에서 가장 비싼 단계는 "고치기"가 아니라 **"상태 의존적 결함을 결정적으로 재현하기"**입니다.
Re-aging은 *실행 → 부분 상환 → 일자 경과 → 재조정* 순서에 따라 결과가 달라지는
상태 의존 로직이라 이 근육을 훈련하기에 적합합니다.

---

## 2. 대상 도메인: Loan Re-aging

Re-aging은 연체/회수 관리에서 **남은 채무를 새 상환 스케줄로 재조정**하는 절차입니다(워크아웃·약정 변경 맥락).
핵심 불변식 하나: **재조정의 기준 금액은 "최초 승인 원금"이 아니라 "재조정 시점의 잔여 원금"**이어야 합니다.
이미 상환된 원금을 다시 스케줄에 태우면 채무자가 갚은 돈이 장부에서 사라지는 셈이 됩니다.

---

## 3. 주입한 결함 (Injected Defect)

수정 지점: `fineract-provider/.../loanaccount/service/reaging/LoanReAgingServiceImpl.java`
의 `createReAgeTransaction(...)`.

정상 코드는 잔여 원금을 참조합니다.

```java
// 정상 (Apache 원본)
Money txPrincipal = loan.getTotalPrincipalOutstandingUntil(transactionDate);
BigDecimal txPrincipalAmount = txPrincipal.getAmount();
```

주입한 결함은 이를 **최초 승인 원금**으로 바꿉니다.

```java
// 주입된 결함
BigDecimal txPrincipalAmount = loan.getApprovedPrincipal();
```

`getApprovedPrincipal()`은 상환 이력과 무관하게 **승인 시점의 원금 총액**을 반환합니다.
따라서 부분 상환 후 Re-age를 실행하면, 재조정 트랜잭션이 이미 상환된 금액까지 포함한
승인 원금 전체로 생성됩니다.

---

## 4. 결정적 재현 (Deterministic Reproduction)

재현 경로: `integration-tests/.../loan/reaging/LoanReAgingIntegrationTest.java`

테스트 `test_LoanReAgeTransaction_Works` 시나리오:

| 단계 | 행위 | 일자 | 금액 |
|---|---|---|---|
| 1 | 대출 실행(Disbursement) | 2023-01-01 | 1,250 |
| 2 | 부분 상환(Repayment) | 2023-01-15 | 250 |
| 3 | Re-age 실행 | 2023-02-02 | (검증 대상) |

상환 후 실제 잔여 원금은 **1,000**입니다.

- **결함 주입 상태**: Re-age 트랜잭션이 `1,250`(승인 원금)으로 생성 → 상환 250이 무시됨. 재현 확인.
- **정상 로직 상태**: Re-age 트랜잭션이 `1,000`(잔여 원금)으로 생성.

테스트는 트랜잭션 목록을 다음과 같이 단언합니다:

```text
Disbursement 1250.0  @ 2023-01-01
Repayment     250.0  @ 2023-01-15
Re-age       1000.0  @ 2023-02-02   // 정상 로직 기준 기대값
```

> 참고: 동일 파일의 `test_LoanUndoReAgeTransaction_Works`는 **상환이 없는 시나리오**라
> 잔여 원금 = 승인 원금 = 1,250 으로 일치하며, 잔여원금 로직 하에서도 기대값 1,250 이 정상입니다.
> 두 테스트는 모순되지 않으며, 상환 유무에 따른 잔여원금 차이를 의도적으로 대비시킨 구성입니다.

---

## 5. 근본 원인 분석 (RCA)

### 5.1 의미론적 오류

`getApprovedPrincipal()`과 잔여 원금은 **부분 상환이 1건이라도 존재하는 순간 의미가 갈립니다**.
Re-aging은 "앞으로 갚을 채무"를 재배치하는 절차이므로 기준값은 정의상 *현재 잔여 원금*이어야 합니다.
승인 원금을 쓰면 상환 완료분이 재조정 스케줄에 부활하여 장부상 채무가 부풀려집니다.
이는 단순 표시 오류가 아니라 **원장 정합성 결함**입니다.

### 5.2 정상 메서드의 동작

`Loan.getTotalPrincipalOutstandingUntil(LocalDate)` (Loan.java):

```java
public Money getTotalPrincipalOutstandingUntil(LocalDate date) {
    return getRepaymentScheduleInstallments().stream()
        .filter(i -> i.getDueDate().isBefore(date) || i.getDueDate().isEqual(date))
        .map(i -> i.getPrincipalOutstanding(loanCurrency()))
        .reduce(Money.zero(loanCurrency()), Money::add);
}
```

상환 스케줄의 회차별 `principalOutstanding`을 기준일까지 합산합니다.
즉 상환이 반영된 **회차 단위 실잔액**의 합이며, 이것이 Re-age가 사용해야 할 값입니다.

### 5.3 교차 근거 (canonical 패턴)

같은 패키지의 형제 로직 `LoanReAmortizationServiceImpl.createReAmortizeTransaction(...)`
(이 프로젝트가 손대지 않은 원본)도 **동일한 메서드를 동일한 의도로** 사용합니다.

```java
// in case of a reamortize transaction, only the outstanding principal amount
// until the business date is considered
Money txPrincipal = loan.getTotalPrincipalOutstandingUntil(transactionDate);
```

Re-amortize와 Re-age는 같은 "잔여 원금 기준 재조정" 계열이므로,
잔여원금 참조가 이 코드베이스의 정식 패턴임을 교차 확인할 수 있습니다.

---

## 6. 원복과 검증

원복은 §3의 정상 코드로 되돌리는 것입니다. 핵심은, **원복 결과가 Apache 원본과 동일**하다는 점입니다.

이 저장소 기준 base 커밋 `4d4502fbe` (*FINERACT-1971: Loan re-aging foundational implementation*,
Apache 원본)의 `createReAgeTransaction`은 최초 구현부터 이미
`loan.getTotalPrincipalOutstandingUntil(transactionDate)`를 사용합니다.
→ `getApprovedPrincipal()` 결함은 **원본에 존재한 적이 없으며**, 본 프로젝트가 주입했다가 원복한 것입니다.

검증: `LoanReAgingIntegrationTest`가 정상 로직 기준 기대값(Re-age = 1,000)을 인코딩합니다.
원복 상태에서 아래 절차로 그린을 직접 재현하십시오(빌드 결과는 환경에 따라 로컬에서 확인 필요).

---

## 7. 재현 절차

```bash
# 1) 결함 주입 상태 재현: createReAgeTransaction 을 getApprovedPrincipal() 로 교체 후
./gradlew integrationTest --tests "*LoanReAgingIntegrationTest*"   # 실패(재현) 확인

# 2) 정상 로직으로 원복 후
./gradlew integrationTest --tests "*LoanReAgingIntegrationTest*"   # 그린 확인

# 빌드
./gradlew clean bootJar
```

> 통합테스트는 DB 등 실행 환경을 요구합니다. 본 README는 기대 동작을 기술하며,
> 그린 여부는 각자 환경에서 재현하는 것을 전제로 합니다.

---

## 8. 커밋 이력 (투명 공개)

방법론을 숨기지 않습니다. 결함 주입과 원복을 분리된 커밋으로 남겨 RCA 과정을 추적 가능하게 했습니다.

```text
cc37c055b  fix: FINERACT-1971 outstanding principal 참조 로직 정상화   (원복)        ← 본 프로젝트
b968deb10  test & bug: FINERACT-1971 재현 (상환액 무시 로직 주입 및 테스트 추가)      ← 본 프로젝트
5f9845f86  FINERACT-1971: Triggering a loan level event ...                          ← Apache 원본
4d4502fbe  FINERACT-1971: Loan re-aging foundational implementation                  ← Apache 원본 (base)
```

`b968deb10`이 결함 주입, `cc37c055b`가 원복입니다.
두 커밋의 순 변경은 base 대비 기능적으로 0이며, 이는 의도된 것입니다(주입→재현→RCA→원복).

---

## 9. 범위와 한계

- 주입된 결함은 **합성 시나리오**이며 Apache Fineract의 실제 결함이 아닙니다(§1, §6).
- 본 프로젝트의 가치는 산출물 자체가 아니라 **상태 의존 코어뱅킹 결함의 결정적 재현 + RCA 절차**의
  훈련에 있습니다.
- 단일 시나리오 연습입니다. 동일 기법을 다른 로직으로 반복 적용하는 것이 다음 단계입니다.

---

## About Apache Fineract

본 저장소는 [Apache Fineract](https://github.com/apache/fineract)의 fork입니다.
Fineract는 금융 소외 계층을 위한 오픈소스 코어 뱅킹 플랫폼이며, Apache-2.0 라이선스를 따릅니다.
원본 소스 및 라이선스, NOTICE는 상위 프로젝트를 따릅니다.

- Build: `./gradlew clean bootJar`
- Test: `./gradlew test`
