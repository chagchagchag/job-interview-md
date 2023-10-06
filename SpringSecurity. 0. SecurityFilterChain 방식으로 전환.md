### SecurityFilterChain 방식으로 전환

> 혼자 프로젝트를 해보고 있다가 예전에 혼자 스터디하고 로컬 디스크에 남아있는 걸 발견하고 주섬주섬 정리를 시작해봤다.

<br>



### WebSecurityConfigurerAdapter DEPRECATED 이슈

Spring Security 5.7.0-M2 부터 WebSecurityConfigurerAdapter 가 deprecated 되었다. 

- 관련 링크 : [Spring Security without the WebSecurityConfigurerAdapter](https://spring.io/blog/2022/02/21/spring-security-without-the-websecurityconfigureradapter)

<br>

내가 보기에는 Filter 를 이용한 방식이 조금 더 상식적이고 합리적인 방식이라는 생각...이 든다... (헛.. 누군가 째려보는 것 같다) 

<br>



### SecurityFilterChain 객체 Bean 등록

새로 업데이트되는 버전에서는 `HttpSecurity` 객체를 이용해 생성하는  SecurityFilterChain 객체를 Bean 으로 등록할 때 아래의 두 가지의 방식으로 등록가능하다.

- HttpSecurity.addFilter() 를 이용하는 방식
- HttpSecurity.apply() 를 이용하는 방식
  - AbstractConfiguredSecurityBuilder 클래스 내의 apply 메서드를 오버라이딩하는 방식이다.

<br>



### e.g. addFilter() 를 사용하는 예제

#### SecurityFilterChain 인스턴스를 Bean 으로 등록

SecurityConfig 라는 클래스를 만들고 아래와 같이 FilterChain 을 Bean 으로 등록하는 코드를 작성한다.

클래스 명은 SecurityConfig가 아니어도 괜찮다. 여기저기 Bean을 늘여놓기보다 SecurityConfig 내에 Bean 들을 모아두고자 이렇게 이름을 지어두었다.

```java
@EnableWebSecurity
@Configuration
public class SecurityConfig {
  
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity httpSecurity, AuthenticationConfiguration authenticationConfiguration){ // (1)
        
        try {
            AuthenticationManager authenticationManager = authenticationConfiguration.getAuthenticationManager();
          	
          	// (2)
          	// JwtAuthenticationFilter 클래스는 내가 직접 만든 클래스다.
            JwtAuthenticationFilter authenticationFilter = new JwtAuthenticationFilter(authenticationManager);
          	// 필터가 로그인 동작으로 판단해 동작하도록 인식할 URL 을 지정한다.
          	authenticationFilter.setFilterProcessUrl("/users/login");

            httpSecurity
                    .csrf(c -> c.disable())
              			// (3)
                    .addFilter(authenticationFilter)
                    .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                	// ...
                
            
            return httpSecurity.build();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
        
    }
}
```



- (1)
  - SecurityFilterChain 인스턴스를 생성하기 위해서 HttpSecurity 객체, AuthenticationConfiguration 객체를 스프링 컨테이너로부터 의존성 주입받았다.
- (2)
  - JwtAuthenticationFilter 클래스는 내가 직접 만든 클래스다. 
  - UsernamePasswordAuthenticationFilter 클래스를 상속받아 attemptAuthentication() 메서드를 JWT 방식에 맞게 커스터마이징한 코드다.
- (3)
  - HttpSecurity 객체 내에 jwtAuthenticationFilter 를 filter로 등록해줬다.

<br>



#### JwtAuthenticationFilter

JwtAuthenticationFilter 는 아래와 같이 작성해줬다. 

> - Bean 으로 등록해서 사용해도 되긴 하지만, 아래 코드는 Bean 으로 등록했던 코드는 아니다. 굳이 Bean으로 등록해야할까? 하고 생각하면서 일반 클래스로 정의했었다.
>
> - 어쩌면 SecurityConfig 클래스 내에 inner 클래스로 선언해두는게 더 나았을까 싶기도 하다.



```java
public class JwtAuthenticationFilter extends UsernamePasswordAuthenticationFilter {
    private final AuthenticationManager authenticationManager;

  	// 생성자 주입 코드 (중략) ...


    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {
        return LoginRequestDto.of(request)  // 1.1) HttpServletRequest 에서 userId, password 추출
                .map(loginRequestDto -> {
                    // 1.2) Authentication 생성
                    Authentication authenticationToken =
                            loginRequestDto.generateAuthenticationToken();

                    // 1.3) AuthenticationManager 를 통해 부수적인 인증 메서드들 내부 호출
                    Authentication authentication = authenticationManager.authenticate(authenticationToken);

                    // 로그를 찍어보기 위해 추가했다.
                    // CustomUserDetails userDetails = (CustomUserDetails) authentication.getPrincipal();
                    // logger.info("JwtAuthenticationFilter, user = {}", userDetails.getUsername());

                    return authentication;
                })
                .orElseThrow(
                        () -> new IllegalArgumentException("사용자 정보가 부정확합니다.")
                );
    }

    @Override
    protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain chain, Authentication authResult) throws IOException, ServletException {
        // 2.1) authentication 에서 UserDetails 를 추출
        CustomUserDetails userDetails = (CustomUserDetails) authResult.getPrincipal();

        // 2.2) userDetails 의 user, username, password 기반으로 token 을 생성
        String jwtToken = JwtTokenProvider.generateToken(SecurityProperties.key, userDetails);

        // 2.3) "Authorization Bearer " 에 jwtToken 을 추가
        response.addHeader("Authorization", "Bearer " + jwtToken);
    }
}

```

- JwtTokenProvider, SecurityProperties 는 내가 직접 만든 클래스다. 별내용 없고 상수이거나 JWT 를 생성하는 로직인데, 이번 문서의 요점과 맞지 않아 생략했다.



<br>



### e.g. apply() 를 사용하는 예제

[Spring Security exposing AuthenticationManager without WebSecurityConfigurerAdapter](https://stackoverflow.com/questions/71281032/spring-security-exposing-authenticationmanager-without-websecurityconfigureradap) 에서는 Local AuthenticationManager 라고 이야기해주고 있다.

아직 사용해본 적이 있는 코드는 아니다.

<br>



**JWTHttpConfigurer.java**

먼저 JWTHttpConfigurer 라는 이름의 **AbstractHttpConfigurer** 타입의 Bean 을 선언한다.

```java
@Component
public class JWTHttpConfigurer extends AbstractHttpConfigurer<JWTHttpConfigurer, HttpSecurity> {

    private final JWTTokenUtils jwtTokenUtils;

    public JWTHttpConfigurer(JWTTokenUtils jwtTokenUtils) {
        this.jwtTokenUtils = jwtTokenUtils;
    }

    @Override
    public void configure(HttpSecurity http) {
        final AuthenticationManager authenticationManager = http.getSharedObject(AuthenticationManager.class);
        http.antMatcher("/graphql").addFilter(new JWTAuthorizationFilter(authenticationManager, jwtTokenUtils));
    }

}
```

<br>





**SecurityConfig.java**

```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig {

    @Autowired
    private JWTTokenUtils jwtTokenUtils;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
                // disable CSRF as we do not serve browser clients
                .csrf().disable()
                // allow access restriction using request matcher
                .authorizeRequests()
                // authenticate requests to GraphQL endpoint
                .antMatchers("/graphql").authenticated()
                // allow all other requests
                .anyRequest().permitAll().and()
            	// (1)
                // JWT authorization filter
                .apply(new JWTHttpConfigurer(jwtTokenUtils)).and()
                // make sure we use stateless session, session will not be used to store user's state
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS);
        return http.build();
    }

}
```

(1) 에서 JWTHttpConfgurer 를 세팅해주고 있다.

JwtTokenUtils 는 아마도 JWT 토큰을 뜯어서 이게 토큰이 맞는지 검증하는 등의 부분들이 있는 모듈인 것으로 보인다.