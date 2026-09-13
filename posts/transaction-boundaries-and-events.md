---
title: "트랜잭션을 나눴더니 이벤트는 데이터를 잃었다."
description: "REQUIRES_NEW를 적용한 뒤 발생한 문제를 통해, 트랜잭션 경계가 이벤트의 실행 시점, 데이터 가시성까지 결정하는 과정을 살펴본다."
publishedAt: 2026-08-23
draft: false
tags:
  - Backend
  - Spring
  - Transaction
  - Event-Driven Architecture
---

![분리된 두 트랜잭션 경계 사이에서 주문 데이터와 이벤트가 서로 다른 영역에 놓인 모습](https://raw.githubusercontent.com/yw7148/blog.youngwon.me-content/main/images/transaction-boundaries-hero.png)

`Spring Transaction` `REQUIRES_NEW` `TransactionalEventListener` `Data Visibility` `Domain Event`

메서드 하나의 트랜잭션을 분리했을 뿐인데, 왜 잘 동작하던 이벤트 리스너가 데이터를 찾지 못했을까?

트랜잭션은 저장 작업의 성공과 실패 범위만 정하지 않는다. 이벤트가 어느 커밋에 결합되는지, 후속 작업이 어떤 데이터를 볼 수 있는지까지 결정한다. `REQUIRES_NEW`를 적용한 뒤 마주친 문제를 따라가며 트랜잭션 경계가 실제로 바꾼 것들을 살펴보자.

## 하나의 트랜잭션에서는 문제가 없었다

주문을 처리해 구독을 만드는 흐름이 있었다. `handleSubscriptionPurchased`는 주문에 구독을 연결하는 전체 과정을 조율하고, `createSubscription`은 실제 구독을 생성한다.

아래 코드는 구조를 설명하기 위해 단순화한 의사 코드다.

```kotlin
@Transactional
fun handleSubscriptionPurchased(orderId: Long) {
    val order = orderRepository.getById(orderId)

    val subscription = subscriptionService.createSubscription(order.customerId)

    order.linkSubscription(subscription.id)
}

fun createSubscription(customerId: Long): Subscription {
    val subscription = subscriptionRepository.save(
        Subscription.create(customerId)
    )

    eventPublisher.publishEvent(
        SubscriptionCreatedEvent(subscription.id)
    )

    return subscription
}
```

이벤트 리스너는 구독이 생성된 뒤 주문 정보를 조회해 후속 작업을 수행했다.

```kotlin
@TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
fun handle(event: SubscriptionCreatedEvent) {
    val order = orderRepository.findBySubscriptionId(event.subscriptionId)
        ?: return

    // 주문과 구독 정보를 이용한 후속 작업
}
```

이 구조에서 두 메서드는 하나의 물리 트랜잭션에 참여한다. `createSubscription`에서 이벤트를 발행해도 `SubscriptionCreatedEvent`는 바깥의 `handleSubscriptionPurchased`가 시작한 트랜잭션에 결합된다.

흐름을 단순화하면 다음과 같다.

```text
[Transaction A 시작]
Order 조회
  -> Subscription 생성
  -> SubscriptionCreatedEvent 발행
  -> Order에 Subscription 연결
  -> BEFORE_COMMIT 리스너 실행
[Transaction A 커밋]
```

리스너는 구독 생성과 주문 연결이 모두 끝난 Transaction A의 커밋 직전에 실행된다. 같은 트랜잭션 안에서는 자신이 변경한 데이터를 볼 수 있으므로, 리스너는 연결된 주문을 정상적으로 조회할 수 있었다.

## 예외의 영향을 분리하려고 REQUIRES_NEW를 적용했다

이후 구독 생성 작업을 바깥 흐름과 독립적으로 커밋할 필요가 생겼다. `createSubscription`에 `REQUIRES_NEW`를 적용했다.

```kotlin
@Transactional(propagation = Propagation.REQUIRES_NEW)
fun createSubscription(customerId: Long): Subscription {
  // create subscription
}
```

> **참고:** 예제에서는 트랜잭션 프록시를 거치도록 두 메서드가 서로 다른 Spring Bean에 있다고 가정한다. 같은 객체 내부에서 직접 호출하면 프록시를 거치지 않아 `REQUIRES_NEW`가 적용되지 않을 수 있다.

Spring에서 `REQUIRES_NEW`는 기존 트랜잭션에 참여하지 않고 항상 독립된 물리 트랜잭션을 사용한다. 안쪽 트랜잭션은 바깥 트랜잭션과 별도로 커밋하거나 롤백할 수 있다.

의도한 변화는 이것이었다.

```text
Subscription 생성 실패
  -> Transaction B만 롤백
  -> Transaction A의 처리 여부를 별도로 결정
```

하지만 실제로 달라진 것은 롤백 범위만이 아니었다. 변경 후 이벤트 리스너는 구독과 연결된 주문을 조회하지 못했다.

## 이벤트는 사라지지 않고 다른 트랜잭션에 결합됐다

처음에는 이벤트가 누락됐다고 생각하기 쉽다. 실제로는 이벤트가 사라진 것이 아니었다. 이벤트가 결합되는 트랜잭션이 바뀌었다.

`@TransactionalEventListener`는 이벤트를 발행한 코드의 상위 메서드 이름을 기준으로 동작하지 않는다. **이벤트가 발행되는 순간 활성화된 트랜잭션**에 결합된다. 기본 phase는 `AFTER_COMMIT`이며, `BEFORE_COMMIT`, `AFTER_ROLLBACK`, `AFTER_COMPLETION` 등으로 바꿀 수 있다.

`createSubscription`이 `REQUIRES_NEW`로 실행되면 바깥의 Transaction A는 잠시 중단되고, 독립된 Transaction B가 시작된다. 이때 발행한 이벤트는 Transaction B에 결합된다.

```text
[Transaction A 시작]
Order 조회
  |
  | Transaction A 중단
  v
  [Transaction B 시작]
  Subscription 생성
  SubscriptionCreatedEvent 발행 ──> Transaction B에 결합
  BEFORE_COMMIT 리스너 실행
  [Transaction B 커밋]
  |
  | Transaction A 재개
  v
Order에 Subscription 연결
[Transaction A 커밋]
```

Transaction B의 `BEFORE_COMMIT` 리스너는 Transaction A가 재개되어 주문과 구독을 연결하기 전에 실행된다. 설령 Transaction A에서 이미 주문을 변경한 뒤 Transaction B를 시작한 구조라도 문제는 남는다. Transaction A의 변경은 아직 커밋되지 않았기 때문에 독립된 Transaction B와 그 후속 작업에서는 보이지 않는다.

PostgreSQL의 기본 격리 수준인 `READ COMMITTED`에서 조회는 다른 트랜잭션이 커밋하지 않은 변경을 읽지 않는다. 하나의 트랜잭션 안에서는 자신의 미커밋 변경을 볼 수 있지만, `REQUIRES_NEW`로 시작한 트랜잭션은 더 이상 같은 트랜잭션이 아니다.

결국 Transaction B가 보장하는 것은 구독 생성 완료뿐이었다. 리스너는 이 이벤트를 받으면서 Transaction A가 확정할 주문 연결까지 기대하고 있었다.

## 이벤트 phase를 바꿔도 해결되지 않았다

`BEFORE_COMMIT`에서 문제가 생기면 Transaction B가 커밋된 뒤 실행되는 `AFTER_COMMIT`으로 바꾸면 되지 않을까?

```kotlin
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
fun handle(event: SubscriptionCreatedEvent) {
    // ...
}
```

하지만 phase는 **결합된 트랜잭션 안에서 언제 실행할지**를 정할 뿐, 이벤트를 다른 트랜잭션으로 옮겨주지 않는다.

- `BEFORE_COMMIT`이면 Transaction B가 커밋되기 전에 실행된다.
- `AFTER_COMMIT`이면 Transaction B가 성공적으로 커밋된 뒤 실행된다.
- 어느 쪽이든 Transaction A의 미커밋 변경을 볼 수는 없다.

문제의 핵심은 실행 시점의 앞뒤가 아니라 이벤트가 Transaction B에 결합됐다는 사실이다. phase를 바꾸는 것은 리스너가 필요로 하는 데이터의 소유권과 완료 상태가 맞을 때 의미가 있다.

## 이벤트 발행 위치를 바깥으로 옮긴 임시 조치

당장 문제를 해결하기 위해 이벤트 발행을 `handleSubscriptionPurchased`로 옮길 수 있었다.

```kotlin
@Transactional
fun handleSubscriptionPurchased(orderId: Long) {
    val order = orderRepository.getById(orderId)
    val subscription = subscriptionService.createSubscription(order.customerId)

    order.linkSubscription(subscription.id)

    eventPublisher.publishEvent(
        SubscriptionCreatedEvent(subscription.id)
    )
}
```

이제 이벤트는 Transaction A에 결합된다. `BEFORE_COMMIT` 리스너도 주문과 구독의 연결이 끝난 같은 트랜잭션 안에서 실행되므로 필요한 데이터를 조회할 수 있다.

현상은 해결됐지만 의미는 어색해졌다. 실제로 구독을 생성한 `createSubscription`이 아니라 전체 흐름을 조율하는 메서드가 `SubscriptionCreatedEvent`를 발행한다. 더 중요한 문제도 있다.

```text
Transaction B: Subscription 생성 후 커밋 성공
Transaction A: Order 연결 중 실패하여 롤백
결과: Subscription은 존재하지만 SubscriptionCreatedEvent는 발행되지 않음
```

`REQUIRES_NEW`를 선택한 순간 부분적으로 완료된 상태를 허용한 셈이다. 이벤트 위치만 옮기면 이 상태를 없앨 수 없다. 오히려 구독은 생성됐는데 구독 생성 이벤트는 없는 새로운 불일치가 생긴다.

## 이벤트 이름이 보장하는 완료 상태를 다시 정의했다

근본적인 문제는 이벤트 발행 코드의 위치보다 이벤트의 의미에 있었다.

`SubscriptionCreatedEvent`라는 이름은 구독이 생성됐다는 사실을 표현한다. 그렇다면 이 이벤트가 보장해야 하는 상태는 Transaction B가 커밋한 `Subscription`이다. 아직 Transaction A가 확정하지 않은 주문 연결까지 리스너가 기대해서는 안 된다.

주문과 구독의 연결 완료가 필요한 후속 작업에는 그 상태를 명시적으로 표현하는 이벤트가 더 자연스럽다.

```kotlin
order.linkSubscription(subscription.id)

eventPublisher.publishEvent(
    OrderSubscriptionLinkedEvent(order.id, subscription.id)
)
```

두 이벤트가 의미하는 완료 상태는 다르다.

| 이벤트 | 보장하는 상태 | 결합할 트랜잭션 |
| --- | --- | --- |
| `SubscriptionCreatedEvent` | 구독 생성 완료 | 구독을 생성하는 Transaction B |
| `OrderSubscriptionLinkedEvent` | 주문과 구독 연결 완료 | 연결을 확정하는 Transaction A |

이벤트 소비자도 자신에게 실제로 필요한 완료 상태를 선택할 수 있다. 구독 자체만 필요하면 `SubscriptionCreatedEvent`를, 주문과 구독의 연결이 필요하면 `OrderSubscriptionLinkedEvent`를 처리한다.

## REQUIRES_NEW가 정말 필요했을까

돌이켜보면 두 작업이 함께 성공하거나 실패해야 한다면 하나의 트랜잭션에 두는 편이 자연스럽다. 예외에 의한 rollback-only가 문제라면 트랜잭션부터 나누기보다 PostgreSQL의 `INSERT ... ON CONFLICT`처럼 같은 트랜잭션 안에서 충돌을 처리할 방법을 먼저 검토할 수 있다.

분리가 반드시 필요하다면 안쪽 트랜잭션만 성공한 부분 완료 상태를 도메인이 허용하는지, 남은 작업을 어떻게 이어갈지까지 정해야 한다. `REQUIRES_NEW`는 단순한 예외 회피 장치가 아니라 새로운 커밋 경계를 만드는 선택이다.

## 정리

`createSubscription`에 `REQUIRES_NEW`를 적용했을 때 분리된 것은 메서드 하나가 아니었다. 커밋과 롤백 범위뿐 아니라 이벤트가 결합되는 트랜잭션과 후속 작업이 볼 수 있는 데이터도 함께 달라졌다.

기준은 이벤트를 발행한 코드의 위치가 아니라 완료된 비즈니스 상태였다. 트랜잭션 경계를 바꿀 때는 그 안의 쿼리만 보지 말고, 그 경계에 의존하던 이벤트와 후속 작업까지 함께 추적해야 한다. 트랜잭션은 생각보다 훨씬 많은 것을 결정한다.

## 참고 자료

- [Spring Framework - Transaction Propagation](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html)
- [Spring Framework - Transaction-bound Events](https://docs.spring.io/spring-framework/reference/data-access/transaction/event.html)
- [PostgreSQL - Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [PostgreSQL - INSERT](https://www.postgresql.org/docs/current/sql-insert.html)
