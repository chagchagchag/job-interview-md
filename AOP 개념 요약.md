

### Aspect

어떤 클래스나 메서드로 여러가지 공통 기능들이 모여 있을 때 이것을 관점 단위로 모으는 것

<br>



### Target

Aspect 를 어떤 클래스, 어떤 메서드에 적용할 지를 정의하는 것을 의미

<br>



### @Advice (해야할 일)

Target 으로 지정한 클래스 또는 메서드를 기반으로 어떤 동작을 할 지를 의미하는 구현체

아래 코드에서는 timeCheck 가 Advice 이다.

Advice 에 @Around 애노테이션을 통해 PointCut, JoinPoint 를 명시할 수 있다.

@Around : 이 Advice 를 어떻게 적용할 것인가? 에 대한 정의. 그 메서드를 감싸는 형태로 정의된다.

```kotlin
@Component
@Aspect
class TimeCheckExecutionAspect(
){
  
  // ...
  
  @Around("execution(* io.study.aop_study..*.ExtractingCoffeeService1.*(..))")
  fun timeCheck(pjp : ProceedingJoinPoint): Any? {
    val begin: Long = System.currentTimeMillis()

    // ProceedingJoinPoint 를 통해 Aspect 가 감싸고 있는 구현체의 로직을 대신 실행한다.
    val result = pjp.proceed()

    val timeCost = System.currentTimeMillis() - begin
    println("time cost >>> $timeCost ms.")

    return result
  }
}
```

<br>



### Advice (e.g. @Around, @Before, @After)

스프링 AOP 에서는 어드바이스의 종류가 여러가지다. 그 중 대표적인 것들만 추려보면 아래와 같다.

- @Around
- @Before
- @After

<br>



```kotlin
@Around("execution(* io.study.aop_study..*.ExtractingCoffeeService1.*(..))")
fun timeCheck(pjp : ProceedingJoinPoint) : Any? {
  // ...
}
```

"Around 어드바이스를 사용하겠다." 는 의미.

execution 이라는 것은 포인트 컷을 의미. 어디에 적용하겠다를 명시함

<br>

위의 코드는 `io.study.aop_study` 패키지 밑에 있는 모든 클래스 중에서 ExtractingCoffeeService 안에 있는 모든 메서드에 <br>

아래 정의한 timeCheck 내부의 동작을 적용하라는 의미.<br>

> 즉, Pointcut 도 여기에 바로 적용한 것

<br>



### PointCut

스프링 AOP 에서는 포인트 컷의 종류가 여러가지다. 그 중 대표적인 것들만 추려보면 아래와 같다.

- execution
  - e.g. `@Around("execution(* io.study.aop_study..*.ExtractingCoffeeService1.*(..))")`
- annotation
  - e.g. `@Around("@annotation(CoffeeTimeCheck)")`
- bean
  - e.g. `@Around("bean(mokapotExtractingCoffeeService3)")`