### AuthenticationManager, ProviderManager, AuthenticationProvider 의 상호작용

이번 문서에서는 AuthenticationManager 객체의 authentication() 메서드를 이용해 사용자가 존재하는 지 등을 조회할때 AuthenticationManager 객체가 사용하는 ProviderManager,  AuthenticationProvider, AuthenticationProvider 객체 각각의 역할을 정리한다.

<br>



### 참고자료

- [javadevjournal.com | Spring Security Authentication](https://www.javadevjournal.com/spring-security/spring-security-authentication/)
- [Baeldung | Spring Security Authentication Provider](https://www.baeldung.com/spring-security-authentication-provider)
- docs
  - [AuthenticationProvider](https://docs.spring.io/spring-security/site/docs/current/api/org/springframework/security/authentication/AuthenticationProvider.html)
  - [DaoAuthenticationProvider](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/dao-authentication-provider.html)
  - [UserDetailsService](https://docs.spring.io/spring-security/site/docs/current/api/org/springframework/security/core/userdetails/UserDetailsService.html#loadUserByUsername(java.lang.String))

<br>



### AuthenticationManager → ProviderManager → AuthenticationProvider → UserDetailsService

AuthenticationManager 객체로 authenticate() 메서드를 호출되면, 이때부터 스프링 시큐리티 내부의 인증을 위한 콜 스택이 시작된다.<br>

AuthenticationManager 에서부터 시작되는 객체의 상호작용은 AuthenticationManager → ProviderManager → AuthenticationProvider → UserDetailsService 의 순서로 객체가 상호작용을 하게 된다.<br>

AuthenticationProvider 는 여러 종류가 등록되어 있는데, 직접 커스터마이징한 AuthenticationProvider 빈이 스프링 컨테이너 내에 존재하지 않는다면 스프링은 [DaoAuthenticationProvider](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/dao-authentication-provider.html) 를 기본으로 사용한다. DaoAuthenticationProvider 는 read only User DAO 인 username 기반의 UserDetailsService 를 사용한다.<br>



만약 LDAP 인증요청,  중앙집중형 Authentication 서버가 있다든지, 상위 그룹 인증서버도 한번 거친다거나 이런 상황처럼 복잡한 상황 또는 복잡한 구조의 인증을 사용해야 한다면, UserDetailsService를 사용하는 것을 멈추고 AuthenticationProvider 를 직접 구현해서 사용해야 한다. 물론 이번 예제에서는 단순한 개념을 정리하는 것이 목적이기에 UserDetailsService 기반의 예제를 다룬다. <br>

> AuthenticationProvider 를 implements 해서 커스터마이징하는 것에 대해서는 [Spring Security Authentication Provider](https://www.baeldung.com/spring-security-authentication-provider) 를 참고하자.

<img src="https://prod-acb5.kxcdn.com/wp-content/uploads/2019/09/Spring-Security-Architecture-.png.webp" width="60%" height="60%"/>

> 그림의 출처는 https://www.javadevjournal.com/spring-security/spring-security-authentication/ 이다.

<br>



### AuthenticationManager

UsernamePasswordAuthenticationFilter 를 확장해서 오버라이딩한 JwtAuthenticationFilter 는 아래와 같은 모습니다.

어려운 코드는 모두 제거하고 단순한 코드만 남겨놓았다.

```kotlin
// 중략 
class JwtAuthenticationFilter (
    private val authenticationManager: AuthenticationManager,
) : OncePerRequestFilter (){

  	// ...
  
    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain
    ) {
        val authorizationHeader : String? = request.getHeader(HttpHeaders.AUTHORIZATION)

        authorizationHeader?.let{ headerString ->
          // 중략 ... 
					
          // Header Validation 작업 + Jwt String 으로부터 email, password 추출 
          // (parsedJwt 객체는 email, password, name, expiration 을 필드로 가지고 있다.)
              
          val auth = UsernamePasswordAuthenticationToken(parsedJwt.email, parsedJwt.password)
          
          // (1) AuthenticationManager 의 authenticate() 메서드로 인증 수행
          authenticationManager.authenticate(auth)
          SecurityContextHolder.getContext().authentication = auth
        }

        filterChain.doFilter(request, response)
    }

  	// ...
}
```



- (1) AuthenticationManager 의 authenticate (AuthenticationToken) 메서드
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

- Provider 중 authenticate()를 통해 인증을 거친 result가 null 이 아닐 경우에만 부가적인 후처리를 수행 후 result 를 return 한다.

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

<br>



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









