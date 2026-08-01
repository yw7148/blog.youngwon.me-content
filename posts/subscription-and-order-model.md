---
title: "구독과 주문을 하나의 모델로 관리하면 생기는 문제"
description: "반복 결제 도메인에서 구독 계약과 개별 주문을 분리해야 하는 이유를 정리한다."
publishedAt: 2026-08-01
draft: true
tags:
  - Domain Design
  - Subscription
  - Operations
series: "첫 글 후보"
---

## 한 줄 요약

문제와 결론을 2~3문장으로 먼저 설명한다.

## 상황

어떤 코드 또는 시스템에서 문제가 발생했는지 설명한다.

## 처음 생각했던 해결책

처음에는 왜 이 접근이 맞다고 생각했는지 적는다.

## 문제가 된 부분

로그, 코드, 데이터 흐름을 이용해 원인을 설명한다.

## 검토한 선택지

### 선택지 A

장점과 단점

### 선택지 B

장점과 단점

## 최종 결정

선택한 방법과 선택 이유를 적는다.

## 적용 코드

```kotlin
data class SubscriptionContract(
    val id: String,
    val customerId: String,
    val status: String,
)

data class RecurringOrder(
    val id: String,
    val subscriptionContractId: String,
    val sequence: Int,
)
```

## 남은 한계

완전히 해결되지 않은 부분과 적용 조건을 적는다.

## 정리

- 핵심 정리 1
- 핵심 정리 2
- 핵심 정리 3

## 참고 자료

- 공식 문서 또는 공개 자료를 연결한다.
