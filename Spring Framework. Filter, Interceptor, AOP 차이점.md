### Filter, Interceptor, AOP 차이점

### 참고자료

[https://giron.tistory.com/70](https://giron.tistory.com/70)

<br>



### Filter, Interceptor, AOP

Filter, Interceptor, AOP 의 각각의 스프링에서의 동작흐름은 아래와 같은 순서로 진행된다.

- Filter → Interceptor → AOP 

<br>



Filter 는 서블릿에 요청이 도착하기 직전에 Filter가 동작한다.

Filter의 위치를 흐름으로 표현하면 HTTP → WAS → **Filter** → Servlet → Controller 이다.<br>

<br>



Interceptor 는 컨트롤러에 요청이 도착하기 직전에 Interceptor가 동작한다.

Interceptor 의 위치를 흐름으로 표현하면 HTTP → WAS → Filter → Servlet → **Interceptor** → Controller 다.<br>

<br>



AOP 는 애플리케이션 레벨에서 정의해둔 메서드의 호출 시작/끝 지점에 수행할 동작을 의미한다. @Around, @Before, @After 라고 불리는 어드바이스와 execution(...), annotation(...), bean(...) 과 같은 포인트 컷으로 어떻게 수행할지를 정의한다.

AOP의 호출 위치는 Filter, Interceptor 와 비교하면 Filter → Interceptor → AOP 로 수행된다.



 