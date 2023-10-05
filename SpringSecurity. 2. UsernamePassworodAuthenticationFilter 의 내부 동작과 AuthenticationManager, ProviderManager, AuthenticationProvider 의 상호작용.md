### UsernamePassworodAuthenticationFilter 의 내부 동작과 AuthenticationManager, ProviderManager, AuthenticationProvider 의 상호작용

OAuth2 기능도 같이 추가해서 정리할까 했는데, 10월 한달 동안은 기본 인증(id/pw 방식)만 다루어 정리하기로 했다. 너무 잡다하게 이것 저것 붙이면 핵심내용이 안보일것 같아서다. 아마도 지금 작성중인 MSA+DDD 예제 프로젝트의 중반부에 기능을 추가해서 덧붙이게 될 듯 싶다.<br>

OAuth2 Spring Security 인증관련 자료는 [여기](https://junuuu.tistory.com/415) 를 참조하면 되고, 시중에 나온 책들 중에서도 OAuth2 인증에 대해서 다룬 책들 역시 많다.<br>



### 들어가기 전에...

내용을 정리하기에 앞서서 생각의 지도를 만들어보자.

이번 MSA + DDD 예제에서 10월달 구현 버전에서 만들어서 사용할 필터는 아래와 같다.

아래의 두 필터를 SecurityFilterChain 에서 순서대로 거치도록 SecurityConfig 를 구현해둔다.

- JwtAuthenticationFilter : UsernamePasswordAuthenticationFilter
  - id,pw 로 최초 로그인시에 거치는 필터다.
  - 앞서 JwtAuthenticationFilter 라고 하는 Filter를 만들었다. UsernamePasswordAuthenticationFilter 를 구현한 Filter 다. 
  - 존재하는 사용자인지를 판단하고 이것을 기반으로 Authentication 객체를 만들어내기 위한 목적의 필터다.
  
- GrantSecurityContextFilter: BasicAuthenticationFilter
  - JWT 토큰을 통해 인증을 할때 거치는 필터다.
  - 인터넷에서는 보통 AuthorizationFilter, JwtAuthorizationFilter 라는 이름으로 많이 불린다. 그런데 나는 영어권 사람이 아니기에 Authorization 이라는 영어 단어의 의미가 맞는지 자신이 없다. 그래서 일단은 GrantSecurityContextFilter 라는 이름을 정해뒀다.
  - JWT 를 분해해서 username, password가 올바른지 확인한 뒤에 SecurityContext 에 Authentication 객체를 저장한다.
  - BasicAuthenticationFilter 를 활용해 GrantSecurityContextFilter 는 다음 문서에서 정리한다.


<br>

이 중 이번 문서에서 정리할 Filter 는 `UsernamePasswordAuthenticationFilter` 다. BasicAuthenticationFilter 는 다음 문서에서 정리하기로 했다.<br>

<br>



### 참고자료

- [javadevjournal.com | Spring Security Authentication](https://www.javadevjournal.com/spring-security/spring-security-authentication/)
- [Baeldung | Spring Security Authentication Provider](https://www.baeldung.com/spring-security-authentication-provider)
- docs
  - [AuthenticationProvider](https://docs.spring.io/spring-security/site/docs/current/api/org/springframework/security/authentication/AuthenticationProvider.html)
  - [DaoAuthenticationProvider](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/dao-authentication-provider.html)
  - [UserDetailsService](https://docs.spring.io/spring-security/site/docs/current/api/org/springframework/security/core/userdetails/UserDetailsService.html#loadUserByUsername(java.lang.String))




<br>

자료는 'Spring Security Authentication' 이라는 검색어로 구글에서 검색해서 찾았다. [그림](https://prod-acb5.kxcdn.com/wp-content/uploads/2019/09/Spring-Security-Architecture-.png.webp) 과 유사한 그림이 한국 블로그에 많이 돌아다니는데, 원본 출처가 폐쇄된 상태라 새로운 자료를 찾던 중에 찾은 자료. 그림이 어째 더 깔끔하네? ㅎㅎ

<br>





### UsernamePasswordAuthenticatonFilter

#### attemptAuthentication(), successfulAutentication()

UsernamePasswordAuthenticationFilter 를 상속해서 필터를 등록할 때는 attemptAuthentication(req, resp), successfulAuthentication(...) 메서드를 오버라이딩 해서 사용한다.

- attemptAuthentication(req, resp)
  - AbstractAuthenticationProcessingFilter 의 추상메서드인 attemptAuthentication (req, resp) 메서드다. 즉, 반드시 오버라이딩 해야한다.
- successfulAuthentication(req, resp, filterChain, authentication)
  - 오버라이딩 하지 않으면 AbstractAuthenticationProcessingFilter 내에 기본으로 정의된 successfullAuthentication(req, resp, filterChain, authentication) 이 사용된다.
  - 이번 예제에서는 Response의 Header 에 Bearer 토큰을 추가하는 기능을 작성한다.



이번 문서에서는 attemptAuthentication(req, resp) 메서드를 구현할 때 AuthenticationManager 객체를 이용해 사용자가 존재하는 지 등을 조회할때 AuthenticationManager 객체가 사용하는 ProviderManager,  AuthenticationProvider, AuthenticationProvider 객체 각각의 역할을 정리한다.

<br>



#### attempAuthentication() 내부에 정의할 주요 흐름

UsernamePasswordAuthenticationFilter 를 구현한 JwtAuthenticationFilter 내의 인증시도 구문인 attemptAuthentication() 에서는  AuthenticationManager 를 주입받은 후  AuthenticationManager → ProviderManager → AuthenticationProvider → UserDetailsService 를 이용해 DB에 사용자가 존재하는지를 검색한다.

```kotlin
package net.spring.cloud.prototype.userservice.domain.config.security.filter

// 중략 ...

class JwtAuthenticationFilter (
    private val authenticationManager: AuthenticationManager
): UsernamePasswordAuthenticationFilter(){

  	// ...
  
    override fun attemptAuthentication(request: HttpServletRequest?, response: HttpServletResponse?): Authentication {
      	// 중략 ...
      	val authenticationToken = loginRequest.generateAuthenticationToken() // (1)
      	val authentication = authenticationManager.authenticate(authenticationToken) // (2)
      	// 중략 ...
      	return authentication
    }

  	// ...
}
```



- 기본적으로는 AuthenticationManager 는 ProviderManager 인스턴스를 통해 여러 종류의 AuthenticationProvider 를 통해 인증을 수행한다. 코드 내에서 AuthenticationProvider 를 implement 한 Bean 이 없다면 기본 설정으로 지정된 DaoAuthenticationProvider 가 UserDetailsService 를 이용해 인증을 수행하는데 아래의 로직을 수행한다.
  - 사용자가 존재하면, username, password 기반으로 UserDetails 객체를 생성해서 return 하고
  - 사용자가 존재하지 않으면 UsernameNotFoundException 을 throw 한다.
- 호출 구조나 객체바인딩 구조는 이번 문서에 정리해두었다.

<br>



> 참고) Custom AuthenticationProvider를 사용하는 경우
>
> - DaoAuthenticationProvider 는 `read-only` DAO 인 `UserDetailsService` 을 사용하고, username 만을 이용해 접근하기에 조금 더 복잡한 비즈니스 로직을 구현하려면 Custom 하게 AuthenticationProvider 를 구현해서 사용한다.
> - 여기에 대해서는 [Spring Security Authentication Provider](https://www.baeldung.com/spring-security-authentication-provider) 를 참고하자.

<br>



### AuthenticationManager → ProviderManager → AuthenticationProvider → UserDetailsService

> 그림의 출처는 https://www.javadevjournal.com/spring-security/spring-security-authentication/ 이다.

내가 구현한 JwtAuthenticationFilter 는 UsernamePasswordAuthenticationFilter 를 확장한 Filter다. UsernamePasswordAuthenticationFilter 혼자서는 아무 인증을 수행할 수 없기에 AuthenticationManager 인스턴스를 의존성 주입 받는다. 이렇게해서 AuthenticationManager 에서부터 인증을 위한 콜 스택이 시작된다. AuthenticationManager 에서부터 시작되는 객체의 상호작용은 AuthenticationManager → ProviderManager → AuthenticationProvider → UserDetailsService 의 순서로 객체가 상호작용을 하게 된다.

AuthenticationProvider 는 여러 종류가 등록되어 있는데, 직접 커스터마이징한 AuthenticationProvider 빈이 스프링 컨테이너 내에 존재하지 않는다면 스프링은 [DaoAuthenticationProvider](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/dao-authentication-provider.html) 를 기본으로 사용한다. DaoAuthenticationProvider 는 read only User DAO 인 username 기반의 UserDetailsService 를 사용한다.<br>

LDAP 인증요청,  중앙집중형 Authentication 서버가 있다든지, 상위 그룹 인증서버도 한번 거친다거나 이런 상황처럼 복잡한 상황 또는 복잡한 구조의 인증을 사용해야 한다면, UserDetailsService를 사용하는 것을 멈추고 AuthenticationProvider 를 직접 구현해서 사용해야 한다. 물론 이번 예제에서는 단순한 개념을 정리하는 것이 목적이기에 UserDetailsService 기반의 예제를 다룬다. <br>

> AuthenticationProvider 를 implements 해서 커스터마이징하는 것에 대해서는 [Spring Security Authentication Provider](https://www.baeldung.com/spring-security-authentication-provider) 를 참고하자.

<img src="https://prod-acb5.kxcdn.com/wp-content/uploads/2019/09/Spring-Security-Architecture-.png.webp" width="60%" height="60%"/>

<br>



### AuthenticationManager

UsernamePasswordAuthenticationFilter 를 확장해서 오버라이딩한 JwtAuthenticationFilter 는 아래와 같은 모습니다.

어려운 코드는 모두 제거하고 단순한 코드만 남겨놓았다.

```kotlin
package net.spring.cloud.prototype.userservice.domain.config.security.filter

// 중략 
class JwtAuthenticationFilter (
    private val authenticationManager: AuthenticationManager
): UsernamePasswordAuthenticationFilter(){

  	// ...
  
    override fun attemptAuthentication(request: HttpServletRequest?, response: HttpServletResponse?): Authentication {
      	// 중략 ...
      	val authenticationToken = loginRequest.generateAuthenticationToken() // (1)
      	val authentication = authenticationManager.authenticate(authenticationToken) // (2)
      	// 중략 ...
      	return authentication
    }

  	// ...
}
```



- (1) LoginRequest 객체 내의 generateAuthenticationToken() 메서드
  - 중략된 코드에서는 HttpServletRequest 로부터 id, pw 를 추출하고 이것는 LoginRequest 객체로 변환한다. 
  - LoginRequest 객체 내의 generateAuthenticationToken() 메서드는 id, pw 를 UsernamePasswordAuthenticationToken 객체로 감싸서 리턴한다.
- (2) AuthenticationManager 의 authenticate (AuthenticationToken) 메서드
  - AuthenticationManager 를 이용해 인증을 수행한다.
  - 시스템에 존재하는 사용자의 정상적인 요청이라면 Authentication 객체를 정상적으로 리턴하게 된다.
  - 이때 <u>AuthenticationManager → ProviderManager → AuthenticationProvider → UserDetailsService</u> 의 순서로 객체가 상호작용을 하게 된다.

<br>



### ProviderManager

이름을 보고 이해하면 쉽게 기억할 수 있다. 

- <u>Provider</u>Manager 객체는 Authentication<u>Provider</u> 를 관리하는 <u>Manager</u> 역할의 객체다. 

<br>

참고로 위에서 정리했듯 AuthenticationManager 로부터 시작되는 객체의 상호작용은 <u>AuthenticationManager → ProviderManager → AuthenticationProvider → UserDetailsService</u> 의 순서로 객체가 상호작용을 한다.<br>

<br>



스프링에서 제공되는 ProviderManager 클래스 내의 authenticate(auth) 메서드의 일부분을 발췌한 내용은 아래와 같다.

이 코드의 주요 흐름을 한문장으료 요약하면 이렇다.

- Provider 중 authenticate()를 통해 인증을 거친 result가 null 이 아닌 결과가 아닐 경우 부가적인 후처리를 수행 후 result 를 return 한다.

```java
public class ProviderManager implements AuthenticationManager, MessageSourceAware, InitializingBean {

  // ...
  
	@Override
	public Authentication authenticate(Authentication authentication) throws AuthenticationException {
    // 중략 ...
    for (AuthenticationProvider provider : getProviders()) {
      if (!provider.supports(toTest)) {
				continue;
			}
      // ...
      try {
				result = provider.authenticate(authentication);
				if (result != null) {
					copyDetails(authentication, result);
					break;
				}
			}
			catch (AccountStatusException | InternalAuthenticationServiceException ex) {
				// ...
			}
      // ...
    }
    
    if (result != null) {
			if (this.eraseCredentialsAfterAuthentication && (result instanceof CredentialsContainer)) {
				((CredentialsContainer) result).eraseCredentials();
			}
			if (parentResult == null) {
				this.eventPublisher.publishAuthenticationSuccess(result);
			}

			return result;
		}
    
  }
  
  // ...
}
```



### AuthenticationProvider

AuthenticationProvider 는 종류가 엄청나게 많다. 위에서 살펴본 ProviderManager 객체는 이 중에서 우선순위가 높은 AuthentiationProvider 의 인증의 결과가 정상이면 결과값인 Authentication 객체를 AuthenticationManager 에 리턴한다.

<img src="https://prod-acb5.kxcdn.com/wp-content/uploads/2020/06/AuthenticationProvider.png.webp" width="60%" height="60%"/>

> 그림 출처 : [Spring Security Authentication](https://www.javadevjournal.com/spring-security/spring-security-authentication/)

<br>



AuthenticaitonProvider 들에는 아래와 같은 것들이 있다.

1. DaoAuthenticationProvider.
2. JAAS Authentication
3. OpenID Authentication
4. X509 Authentication
5.  SAML 2.0
6. OAuth 2.0
7. RememberMeAuthenticationProvider
8. LdapAuthenticationProvider

<br>

AuthenticationProvider 를 직접 구현해서 Bean 으로 등록하지 않는다면, 스프링시큐리티는 DaoAuthenticationProvider 를 디폴트로 설정하고 있기에 DaoAuthenticationProvider 를 이용해 UserDetailsService 를 접근한다.

<img src="https://docs.spring.io/spring-security/reference/_images/servlet/authentication/unpwd/daoauthenticationprovider.png" width="60%" height="60%"/>

> 그림 출처 : [https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/dao-authentication-provider.html](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/dao-authentication-provider.html)

<br>



### UserDetailsService

AuthenticationProvider 를 직접 구현해서 Bean 으로 등록하지 않는다면, 스프링시큐리티는 DaoAuthenticationProvider 를 디폴트로 설정한다. DaoAuthenticationProvider 는 `read-only` DAO 인 `UserDetailsService` 을 사용하고, UserDetailsService 는 username 만을 이용해 접근하기에 단순한 인증로직에만 적합하고, 복잡한 인증 로직에는 적합하지 않다.<br>

 조금 더 복잡한 비즈니스 로직을 구현하려면 Custom 하게 AuthenticationProvider 를 구현해서 사용한다.

- 여기에 대해서는 [Spring Security Authentication Provider](https://www.baeldung.com/spring-security-authentication-provider) 를 참고하자.

<br>



```JAVA
@Service
public class CustomUserDetailsService implements UserDetailsService {

    private final UsersRepository usersRepository;

    public CustomUserDetailsService(UsersRepository usersRepository){
        this.usersRepository = usersRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        Users user = usersRepository.findByUsername(username);
      	if(user == null) throw new UsernameNotFoundException("사용자가 존재하지 않습니다")
        return new CustomUserDetails(user);
    }
}

```

<BR>









