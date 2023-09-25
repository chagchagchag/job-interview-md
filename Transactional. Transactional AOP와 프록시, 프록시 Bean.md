### Transactional. Transactional AOP와 프록시, 프록시 Bean

스프링에서 Bean 등록 시 `@Transactional` 이 붙은 클래스에 대해 프록시 객체가 생성/등록되는 원리에 대해 정리해봤다.<br>

<br>



### AOP 기반으로 동작하는 @Transactional 프록시

Spring Data 는 Spring AOP 를 통해 @Transactional 어노테이션이 붙은 타겟 메서드/클래스에 대해 트랜잭션 자원의 전처리/후처리를 수행한다.<br>

Spring Data 는 @Transactional 이 하나라도 포함된 클래스를 직접 접근해서 AOP 처리하지 않고 실제 객체를 상속(확장)한 프록시 객체를 통해 AOP 처리를 한다.<br>

<br>



### 프록시 객체란?

- `@Transactional` 이 적용된 타겟은 프록시 객체라고도 부른다.
- spring data 내에서 `@Transactional` 이 붙은 타겟을 접근할 때 직접 접근하지 않고 <u>기존 로직의 기능을 확장(상속)해서</u> 트랜잭션 자원 처리 기능을 추가한 프록시 객체를 새로 생성해서 사용한다.
- 프록시 객체는 쉽게 설명하면 가짜 객체라고 생각하면 된다.
- 이렇게 생성된 프록시 객체 내에는 트랜잭션의 시작/커밋/롤백 처리를 후처리할 수 있는 내부 로직들이 정의되어 있다.



<br>



### @Transactional 프록시의 동작, 어려워지는 점

Spring Data 는 @Transactional 이 하나라도 포함된 클래스를 직접 접근해서 AOP 처리하지 않고 실제 객체를 상속(확장)한 프록시 객체를 통해 AOP 처리를 한다.<br>

이렇게 실제 객체를 상속(확장)한 프록시 객체로 전처리/후처리를 훅 처럼 처리한다.<br>

이때 `@Transactional` 이 있는 클래스를 **프록시 객체로 메서드를 수행하는지** 또는 **부모클래스의 객체로 메서드를 실행하게 되는지**를 판단하는 것이 어렵게 된다. 이런 점이 많은 사람들이 @Transactional 을 어렵게 느끼는 이유가 된다.<br>

왜냐하면 메서드를 트랜잭셔널 프록시 객체로 실행하지 않고 일반 객체의 메서드로 실행할 경우는 트랜잭션 자원 처리에 관련된 콜백 메서드들이 적용되지 않기 때문이다.<br>



<br>





### 프록시 Bean : 프록시 객체를 빈(Bean) 등록

- 스프링 컨테이너 로딩 시 스프링 컨테이너는 @Transactional 적용 대상의 객체를 프록시 라이브러리를 이용해서 <u>실제 객체를 확장한 가짜 객체</u>를 프록시 방식으로 생성해둔다.
- 스프링 컨테이너는 이렇게 생성된 <u>프록시 객체</u>를 빈(Bean) 으로 등록한다. 
- 실제 객체를 빈(Bean) 으로 등록하는 것이 아니라 트랜잭션 기능이 확장된 프록시 객체(가짜객체)를 빈(Bean)으로 등록한다.



<br>



### @PostConstruct vs ApplicationReadyEvent

애플리케이션 로딩 완료 후 Transactional 한 초기화 작업이 필요한 경우가 있다. 이 경우 애플리케이션의 초기화 완료 이벤트로 사용되는 ApplicationReadyEvent, @PostConstruct 를 떠올릴 수 있다. 이 중 ApplicationReadyEvent 를 사용하는 것이 권장되는 편이다.

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

