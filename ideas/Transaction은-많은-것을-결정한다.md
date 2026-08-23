# 트랜잭션을 나눴더니 이벤트가 데이터를 잃어버렸다.

## 핵심 아이디어

기존에는 주문에 구독을 연결하는 전체 흐름을 담당하는 `handleSubscriptionPurchased`와 실제 구독을 생성하는 `createSubscription`이 하나의 트랜잭션에서 동작했다. 따라서 `createSubscription`이 발행한 `SubscriptionCreatedEvent` 역시 `handleSubscriptionPurchased`의 트랜잭션에 결합되어 있었다. 이후 핵심 작업인 `createSubscription`을 `REQUIRES_NEW`로 분리하자, 잘 동작하던 이벤트 리스너가 `handleSubscriptionPurchased`에서 변경한 Order 데이터를 더 이상 조회하지 못했다.

분리한 것은 하나의 메서드라고 생각했지만, 실제로는 그 안에서 발행한 이벤트가 참여하는 트랜잭션과 데이터 가시성까지 함께 분리된 것이다. `REQUIRES_NEW`를 적용할 때는 커밋과 롤백의 범위뿐 아니라, 해당 트랜잭션에 결합된 이벤트와 후속 작업이 바라보는 데이터의 경계도 확인해야 한다.

## 다룰 내용

- 하나의 트랜잭션에서 정상적으로 동작하던 기존 코드
- `createSubscription`을 `REQUIRES_NEW`로 분리한 뒤 이벤트 리스너가 Order를 찾지 못한 현상
- 이벤트가 사라진 것이 아니라, `handleSubscriptionPurchased`가 아닌 `createSubscription`의 트랜잭션에 결합된 과정
- `handleSubscriptionPurchased`에서 이미 변경한 데이터도 커밋 전에는 `createSubscription`의 독립된 트랜잭션에서 볼 수 없는 이유
- `BEFORE_COMMIT`과 `AFTER_COMMIT`등 이벤트 발행 시점보다 이벤트가 어느 트랜잭션에 결합됐는지가 중요했던 이유:
  - 이벤트가 발행된 트랜잭션이 커밋되기 전에는, 해당 트랜잭션에서 변경한 데이터는 다른 트랜잭션에서 볼 수 없다.
  - 따라서 이벤트 리스너가 조회하는 데이터는 이벤트가 발행된 트랜잭션의 커밋 시점까지 반영되지 않는다.
- 이벤트 발행을 `handleSubscriptionPurchased`로 옮긴 임시 조치와 이벤트가 의미하는 완료 상태에 대한 고민
  - 임시로 `SubscriptionCreatedEvent`의 발행 위치를 `createSubscription`에서 `handleSubscriptionPurchased`로 옮겼다. 이벤트가 바깥 트랜잭션에 결합되면서 Order와 Subscription의 연결 작업까지 마친 뒤 리스너가 실행되어 기존 문제는 해결할 수 있었다.
  - 하지만 실제로 Subscription을 생성한 `createSubscription`이 아니라 전체 흐름을 조율하는 메서드가 생성 이벤트를 직접 발행하게 되었다. 또한 `createSubscription`은 커밋됐지만 `handleSubscriptionPurchased`가 롤백된다면, Subscription은 존재하면서도 생성 이벤트는 발행되지 않는다는 한계가 남았다.
  - 근본적으로는 `SubscriptionCreatedEvent`가 보장하는 상태와 리스너가 기대하는 상태를 구분해야 했다. Subscription의 생성 사실은 `SubscriptionCreatedEvent`로 표현하고, Order와 Subscription의 연결까지 필요한 리스너에는 `OrderSubscriptionLinkedEvent`와 같이 전체 비즈니스 작업의 완료를 나타내는 별도 이벤트를 전달하는 것이 최종적인 설계 방향이다.
  - 이벤트를 발행한 코드의 위치보다 중요한 것은 이벤트가 어떤 상태의 완료를 의미하며, 그 상태를 확정하는 트랜잭션이 무엇인지였다.
  - 함께 성공하거나 실패해야 하는 작업은 가능한 한 하나의 트랜잭션에 둔다. 단순히 예외의 영향을 피하기 위해 트랜잭션을 먼저 분리하기보다 PostgreSQL의 `ON CONFLICT` 등 하나의 트랜잭션 안에서 해결할 방법을 우선 검토한다.
  - 트랜잭션 분리가 반드시 필요하다면 부분적으로 완료된 상태가 발생할 수 있음을 인정하고 이벤트를 단계별로 구분한다. 신뢰성 있는 후속 처리가 필요할 경우에는 Outbox, 멱등한 재시도 또는 Saga와 같은 방법도 함께 고려한다.

## 회고

트랜잭션을 분리하면서 데이터 저장의 성공과 실패 범위만 달라진다고 생각했다. 하지만 트랜잭션은 이벤트의 실행 경계와 데이터 가시성까지 결정하고 있었다. 앞으로 트랜잭션 경계를 변경할 때는 그 안에서 실행되는 코드뿐 아니라, 해당 경계에 암묵적으로 의존하던 이벤트와 후속 작업까지 함께 추적해야 한다.
