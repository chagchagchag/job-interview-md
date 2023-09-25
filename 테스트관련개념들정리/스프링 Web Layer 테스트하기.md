# 스프링 Web Layer 테스트하기

문서를 어딘가에 정리는 해두고 싶고, Spring 관련 내용만 어디에 모아두기엔 또 주제가 협소하고 그래서 매번 로컬 PC 어딘가에 문서를 작성해두고 매번 까먹었던 문서다. 이 참에 그냥 Spring Security 저장소에 저장해두려 한다.



### 참고

- Testing the Web Layer
  - [https://spring.io/guides/gs/testing-web/](https://spring.io/guides/gs/testing-web/)
- [[docs.spring.io] MockMvcBuilders](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/test/web/servlet/setup/MockMvcBuilders.html)
- [[docs.spring.io] AbstractMockMvcBuilder](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/test/web/servlet/setup/AbstractMockMvcBuilder.html#build--)
- [[docs.spring.io] StandaloneMockMvcBuilder](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/test/web/servlet/setup/StandaloneMockMvcBuilder.html)
- [[docs.spring.io] MockMvc](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/test/web/servlet/MockMvc.html)
- [[docs.spring.io] ResultActions](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/test/web/servlet/ResultActions.html)

<br>



### 테스트 시 웹 레이어를 구동시키는 3가지 방법들

- `@SpringBootTest` 
- `@AutoConfigureMockMvc` 
- `@WebMvcTest` 

<Br>



### @SpringBootTest

스프링 애플리케이션 컨텍스트를 구동해서 테스트를 하는 방식이다.<br>

@SpringBootTest 어노테이션을 사용하면 메인 Configuration 클래스(예를 들면 @SpringBootApplication 과 함게 선언된)를 참조하고, 이 설정 파일을 이용해서 스프링 애플리케이션 컨텍스트를 구동시킨다.<br>

아래는 @SpringBootTest 어노테이션을 사용해 테스트를 구동하는 예제다. 스프링 부트 애플리케이션을 구동시켜서 테스트한다. 즉, 임베디드 톰캣을 직접 구동시켜서 하므로 어느정도는 무거운 테스트 방식이다.<br>

```java
package com.example.testingweb;

import org.junit.jupiter.api.Test;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.SpringBootTest.WebEnvironment;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.boot.web.server.LocalServerPort;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
public class HttpRequestTest {

	@LocalServerPort
	private int port;

	@Autowired
	private TestRestTemplate restTemplate;

	@Test
	public void greetingShouldReturnDefaultMessage() throws Exception {
		assertThat(this.restTemplate.getForObject("http://localhost:" + port + "/",
				String.class)).contains("Hello, World");
	}
}
```



- `webEnvironment = WebEnvironment.RANDOM_PORT`

- - 테스트를 위한 서버를 랜덤한 포트로 구동시킨다. (테스트 환경이 충돌날수도 있는 것을 피할 수있는 좋은 방법)

- `@LocalServerPort`

- - 할당받은 Server 포트를 멤버필드에 저장해두기 위해 사용한 어노테이션이다.

<br>



### @AutoConfigureMockMvc

`@AutoConfigureMockMvc` 어노테이션을 사용하면 서버를 구동시키지 않으면서, 서버가 동작하는 아래의 계층(레이어)만 테스트하는 방식이다.<br>

`@AutoConfigureMockMvc` 어노테이션을 사용하면 실제 HTTP 요청을 처리할 때와 동일한 방식으로 코드가 호출되지만 서버 시작 비용은 들지 않는다. 그리고 거의 모든 스택이 사용된다.<br>

`@AutoConfigureMockMvc` 어노테이션은 Spring 의 MockMvc를 사용하고, 테스트 케이스에서 `@AutoConfigureMockMvc` 를 사용한다.<br>

서버를 구동시는 것은 아니지만, 스프링 애플리케이션의 거의 모든 컨텍스트가 시작된다.<br>

<br>

```java
package com.example.testingweb;

import static org.hamcrest.Matchers.containsString;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultHandlers.print;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

import org.junit.jupiter.api.Test;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.servlet.MockMvc;

@SpringBootTest
@AutoConfigureMockMvc
public class TestingWebApplicationTest {

	@Autowired
	private MockMvc mockMvc;

	@Test
	public void shouldReturnDefaultMessage() throws Exception {
		this.mockMvc.perform(get("/")).andDo(print()).andExpect(status().isOk())
				.andExpect(content().string(containsString("Hello, World")));
	}
}
```

<br>



### @WebMvcTest

@AutoConfigureMockMvc 는 서버를 구동시는 것은 아니지만, 스프링 애플리케이션의 거의 모든 컨텍스트가 시작된다. 하지만, web layer 만에 한정해서 테스트의 범위를 줄일 수 있는 경우 역시 있다.<br><br>

@WebMvcTest 어노테이션을 사용하면 테스트 범위를 웹 레이어에 한정하도록 축소해서 테스트할 수 있다.<br>

또는 특정 컨트롤러에 한정해서만 @WebMvcTest를 사용할 수 있다. (ex. @WebMvcTest(HomeController.class) )<br>



```java
// 특정 컨트롤러(HomeController)에 한정하는 테스트는 아래와 같이 특정 컨트롤러를 명시해준다. 
// 특정 컨트롤러를 한정하지 않고 전체 테스트를 하려면 인자값을 주지 않고 @WebMvcTest 만 붙이면 된다.
@WebMvcTest(HomeController.class)

public class WebLayerTest {

	@Autowired
	private MockMvc mockMvc;

	@Test
	public void shouldReturnDefaultMessage() throws Exception {
		this.mockMvc.perform(get("/")).andDo(print()).andExpect(status().isOk())
				.andExpect(content().string(containsString("Hello, World")));
	}
}
```

<br>



### MockMvc 테스트란?

MVC 계층에서 사용되는 객체를 Mocking해서 테스트하는 방식이다. `MockMvc` 객체는 스프링 MVC 계층을 Mocking 할 수 있도록 스프링에서 제공해주는 클래스이다. 그리고 이 `MockMvc` 객체는 `perform()` , `andDo()` , `andExpect()` 의 동작을 수행할수 있도록 메서드를 제공해주고 있다. 그리고 이 메서드 각각이 `ResultHandlers` , `ResultMatcher` 인스턴스 각각의 동작을 받아 서로 다른 동작을 할 수 있도록 하는 역할을 수행한다.<br>

MockMvc 객체를 테스트할 때 일반적으로 자주 거치는 순서/단계는 아래와 같다.

- MockMvc 객체 생성하기
  - MockMvcBuilders 를 이용해 생성한다.
- MockMvc 객체로 perform 
- MockMvc 객체로 perform 에 대해 expect 선언문 작성하기
  - perform 에 대해 기대되는 결괏값은 여러 가지가 있을 수 있는데, 이것에 대해 정의하는 과정이다.
- perform 과 expect 이전에 해야 하는 동작을 andDo 로 선언하기

<br>



### MockMvc 객체 생성하기

@Autowired 로 MockMvc 객체를 주입받지 못할 때에 사용한다. 이럴 경우 보통 MockMvcBuilder 를 이용해 MockMvc 객체를 세팅한다. 주로 모놀리딕 기반 레거시 시스템에서 자주 발생하는 문제다.

```java
@WebMvcTest(EmployeeController.class) // SpringBootTest 역시 가능하다.
public class EmployeeControllerTest {
  @Autowired
  private EmployeeController controller;
  
  private MockMvc mockMvc;
  
  @BeforeEach
  void setup(){
    mockMvc = MockMvcBuilders
        // MockMvc 인스턴스는 보통 MockMvcBuilders 클래스의 standaloneSetup(Controller) 메서드를 사용한다.
        // 인자값으로 사용되는 controller 는 Spring 컨테이너 내에 존재하는 Controller 인스턴스이다.
        .standaloneSetup(controller)
        // MockMvc 객체를 최종적으로 반환해주는 것은 build() 메서드이다.
        .build();
  }
}
```

<br>





