### Transactional. TransactionSynchronizationManager 초기화 완료 시점에 대한 작업 구현

애플리케이션 로딩 완료 후 Transactional 한 초기화 작업이 필요한 경우가 있다. 트랜잭션 프록시가 세팅이 되어있어야 작업이 가능하기 때문에 트랜잭션 매니저가 초기화가 완료될 때까지 기다려야 한다. 이 경우 애플리케이션의 초기화 완료 이벤트로 사용되는 ApplicationReadyEvent, @PostConstruct 를 떠올릴 수 있다. <br>

이 중 ApplicationReadyEvent 를 사용하는 것이 권장되는 편이다.<br>

<br>



#### @PostConstruct : 의존성 주입이 완료된 시점에 호출

- 스프링 컨테이너의 로딩이 완료된 시점이 아니라, 의존성 주입이 완료된 후에 호출된다.

<br>



#### ApplicationReadyEvent : 스프링 컨테이너 로딩이 완료된 시점에 호출

- 스프링 컨테이너가 로딩된 시점에 호출된다.
- 애플리케이션이 리퀘스트를 받을 준비가 되었을 때 `ApplicationReadyEvent` 가 발생한다.

<br>



### 테스트 코드

#### @PostConstruct 테스트 코드

```java
@SpringBootTest
public class SpringInitTest1{

    @PostConstruct
    @Transactional
    public void initTest(){
        boolean isActive = TransactionSynchronizationManager.isActualTransactionActive();
        log.info("Init Complete, txActive ?? {}", isActive);
    }
}
```

<br>



@PostConstruct 시점에 TransactionSynchronizationManager 를 통해 트랜잭션이 활성화되었는지를 로그로 출력해보면 아래와 같이 `isActualTransactionActive()` 함수를 통해 얻은 결과가 `false` 다.

```plain
Init Complete, txActive ?? false
```

<br>



#### ApplicationReadyEvent

ApplicationReadyEvent 가 발생하는 시점은 스프링 컨테이너가 로딩이 완료된 시점이다. 애플리케이션이 리퀘스트를 받을 준비가 되었을 때를 의미한다.

```java
@SpringBootTest
public class SpringInitTest2{

    @EventListener(value = ApplicationReadyEvent.class)
    @Transactional
    public void initTest(){
        boolean isActive = TransactionSynchronizationManager.isActualTransactionActive();
        log.info("Init Complete, txActive ?? {}", isActive);
    }
}
```

<br>



ApplicationReadyEvent 이벤트가 발동되는 시점에 TransactionSynchronizationManager 를 통해 트랜잭션이 활성화되었는지를 로그로 출력해보면 아래와 같이 `isActualTransactionActive()` 함수를 통해 얻은 결과가 `true` 다.

```plain
Init Complete, txActive ?? true
```

<br>

