### 트랜잭션 프록시 적용 규칙

@Transactional 메서드 여러개를 조합해서 중첩 Call 이 발생하거나, Non Transactional 메서드에서 @Transactional 메서드를 호출할 때 트랜잭셔널 하게 동작하는지에 대한 규칙에 대한 확신이 없으면 @Transactional 적용 메서드에 여러 비즈니스로직을 길게 늘여쓰게 되어 코드를 유지보수하기 쉽지 않아질 수 있다.<br>

이런 상황을 예방하려면 보통은 다른 계층에 독립적으로 서비스 계층에 대해서만 테스트 코드를 작성해둔다. 개인적인 경험으로 실무에서는 대부분 테스트에 대한 인식 자체가 부정적인 케이스를 자주 봤기에 일반적으로 대부분 MockMvc 계층의 테스트만 작성하고, @Transactional 관련 로직들에 대한 테스트 코드는 전무한 경우가 꽤 많다. 이런 이유로 @Transactional 관련 로직은 몇백줄 이상되었던 경우도 봤었다.<br>

이번 문서는 이런 @Transactional 프록시가 여러 개가 중첩되어 있을때 결론 적으로 어떻게 적용되는지에 대해 일반적인 규칙들에 대해 정리하는 것이 목적이다. 일반적인 규칙들 외에도 테스트 코드로 어떻게 검증하는지 예제문서를 따로 정리해두기로 생각중인데 마음먹기 꽤 쉽지 않은 상황이다.<br>

<br>



### 요약

- 클래스레벨에 @Transactional

  - 클래스레벨에 @Transactional 이 적용되면 모든 public 메서드에 @Transactional 자동 적용

- @Transactional 덮어쓰기

  - 최 하위레벨에서 덮어쓴 선언이 최종 @Transactional 선언이 된다.

- 인터페이스에 @Transactional

  - 인터페이스에도 @Transactional 이 적용될 수 있다.

- @Transactional 적용 우선순위

  - 1\) 클래스의 메서드 (가장 하위 레벨. 최종적으로 덮어쓰기 됨)
  - 2\) 클래스 레벨 (클래스 Type 레벨)
  - 3\) 인터페이스의 메서드
  - 4\) 인터페이스 레벨 (인터페이스 Type 레벨)

- 예외케이스) 같은 클래스 내 non-tx 메서드 → tx 메서드 호출하는 경우

  - @Transactional 이 적용되지 않은 일반 메서드에서 같은 클래스 내의 @Transactional 메서드를 내부 호출하는 경우, 트랜잭션이 적용되지 않는다.

  - 이렇게 @Transactional 이 적용되지 않는 이유는 아래와 같은 순으로 실행되기 때문이다.(프록시 객체가 실제 객체의 메서드를 호출하게 되기 때문)

    - 가짜 객체의 일반 메서드 A 호출<br>

      ↓

    - 가짜 객체는 실제 객체의 일반 메서드 A 호출<br>

      ↓

    - 실제 객체의 일반 메서드 A에서 @Transactional 메서드 AA 호출<br>

      ↓

    - @Transactional 적용 안됨

<br>



### 클래스 레벨에 @Transactional : public 메서드 들에 @Transactionl 일괄 적용

클래스 레벨에 트랜잭셔널을 적용하면 그 클래스 내의 모든 public 메서드에 트랜잭셔널이 일괄적으로 적용된다. 즉, 이런 경우 트랜잭션이 의도하지 않은 곳 까지도 과도하게 적용된다는 단점이 있다.<br>

만약 public 이 적용되지 않은 곳에 `@Transactional` 이 적용되어 있으면 예외가 발생하지는 않고 트랜잭션 적용만 무시된다.<br>

<br>



### @Transactional 덮어쓰기 : 최하위 레벨에서 덮어쓴 선언이 최종 @Transactional 선언이 된다.

@Transactional 이 적용되는 것은 우선순위가 있다. 

가장 작은 단위에 적용된 @Tranactional 이 그 윗 레벨에 적용된 @Transactional 을 모두 덮어쓰게 된다. 

예를 들면, readOnly = true , readOnly = false 같은 옵션이 적용된 @Transactional 이 호출 스택 곳곳에 산재해있는 아래의 경우를 예로 들 수 있다.

<br>



e.g.

```java
@Slf4j
@Transactional(readOnly = true)
public class BookService{
    // ...
    @Transactional(readOnly = false)
    public void txSaveBook(){
        printTxInfo();
    }

    // ...
    public void printTxInfo(){
        boolean txActive = TransactionSynchronizationManager.isActualTransactionActive();
        log.info("tx active = {}", txActive);
        boolean readOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
        log.info("tx readOnly = {}", readOnly);
    }
}
```

<br>



출력결과

```plain
tx active = true
tx readOnly = true
```

<br>



### 인터페이스에 @Transactional : 스프링에서는 권장하지 않는 방식

간혹 스프링 버전에 따라서 인터페이스에 @Transactional 이 적용하는 방식이 지원되지 않는 경우도 있다. (스프링 5.0 부터는 가능하지만, 가급적 사용하지 않는 것을 추천)<br>

인터페이스에도 @Transactional 이 적용되어 있다면, 이 경우 @Transactional 이 적용되는 우선순위를 높은 순서로 나열해보면 아래와 같다.<br>

<br>



- 1\) 클래스의 메서드 (가장 하위 레벨. 최종적으로 덮어쓰기 됨)
- 2\) 클래스 레벨 (클래스 Type 레벨)
- 3\) 인터페이스의 메서드
- 4\) 인터페이스 레벨 (인터페이스 Type 레벨)



<br>



### 예외케이스 ) @Transactional 이 적용되지 않는 경우

#### @Transactional 이 적용되지 않은 메서드에서 같은 클래스 내의 @Transactional 이 적용된 메서드를 내부 호출하는 경우

Transactional 이 적용되지 않은 메서드에서 같은 클래스 내의 `@Transactional` 이 적용된 메서드를 내부 호출하는 경우 @Transactional 이 적용되지 않는다. (프록시 객체가 실제 객체의 메서드를 호출하게 되기 때문)

이렇게 @Transactional 이 적용되지 않는 이유는 아래와 같은 순으로 실행되기 때문이다.

- 가짜 객체의 일반 메서드 A 호출<br>

  ↓

- 가짜 객체는 실제 객체의 일반 메서드 A 호출<br>

  ↓

- 실제 객체의 일반 메서드 A에서 @Transactional 메서드 AA 호출<br>

  ↓

- @Transactional 적용 안됨





