### SecurityFilterChain 세부 구현

0번 문서에서 SecurityFilterChain 방식으로 전환되면서 Filter 들을 어떻게 추가해서 정의하는지 간단한 예제를 정리했었다.

이번 문서에서는 이번 달 진행중인 MSA + DDD 예제 프로젝트에서 사용하는 SecurityConfig 의 세부적인 구현을 예제로 남겨보려 한다.

설명은 조금 나중에 해야 할 것 같다. MSA + DDD 예제 구현과 개념정리 작업을 같이 병행하다보니 진도가 안나고 있다. 코딩 테스트 역시 준비하는 중이라 설명문서는 조금씩 양을 줄여나가고 있다. 11월 쯤에 다시 개념정리를 시작할 예정!!

<br>



### kotlin 코드

```kotlin
package net.spring.cloud.prototype.userservice.domain.config
// ...

@EnableWebSecurity
@Configuration
class SecurityConfig (
    val customUserDetailsService: CustomUserDetailsService
){
    @Bean
    fun passwordEncoder() : BCryptPasswordEncoder {
        return BCryptPasswordEncoder()
    }

    @Bean
    fun filterChain(httpSecurity: HttpSecurity,
                    authenticationConfiguration: AuthenticationConfiguration)
    : SecurityFilterChain {
        try{
            val authenticationManager = authenticationConfiguration.authenticationManager
            val jwtAuthenticationFilter = JwtAuthenticationFilter(authenticationManager)
            jwtAuthenticationFilter.setFilterProcessesUrl("/users/login")

            return httpSecurity
                .csrf { it.disable() }
                .formLogin { it.disable() }
                .httpBasic { it.disable() }
                .addFilter(jwtAuthenticationFilter)
                .sessionManagement { it.sessionCreationPolicy(SessionCreationPolicy.STATELESS) }
                .headers { customizer -> customizer.frameOptions { it.disable() } }
                .authorizeHttpRequests {}
                .authorizeHttpRequests {
                    it.requestMatchers(
                        AntPathRequestMatcher("/"),
                        AntPathRequestMatcher("/img/**"),
                        AntPathRequestMatcher("/css/**"),
                        AntPathRequestMatcher("/users/signup"),
                    )
                    .permitAll()
                    .requestMatchers(
                        AntPathRequestMatcher("/users/logout"),
                        AntPathRequestMatcher("/users/detail"),
                        AntPathRequestMatcher("/users/**"),
                    )
                    .hasAnyAuthority("ROLE_USER", "ROLE_MANAGER", "ROLE_ADMIN")
                }
                .userDetailsService(customUserDetailsService)
                .build()
        }
        catch (e: Exception){
            throw RuntimeException(e)
        }
    }
}
```

<br>



### java

코틀린 코드는 java 로 구현하면 아래와 같은 코드로 정의할수 있다. (1년 전에 만들어봤던 예제..)

```java
package io.gosgjung.samples.jwt_security_2710.config.security;

// ...

@EnableWebSecurity
@Configuration
public class SecurityConfig {
  	// ...
  
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity httpSecurity,
                                           AuthenticationConfiguration authenticationConfiguration,
                                           @Qualifier("customCorsFilter") CorsFilter corsFilter,
                                           UserDetailsService customDetailsService,
                                           UsersRepository usersRepository){
        try {
            AuthenticationManager authenticationManager = authenticationConfiguration.getAuthenticationManager();
          	// /users/login 에서 Filter 가 동작하도록 설정
            JwtAuthenticationFilter authenticationFilter = new JwtAuthenticationFilter(authenticationManager);
            authenticationFilter.setFilterProcessesUrl("/users/login");

            JwtAuthorizationFilter authorizationFilter = new JwtAuthorizationFilter(authenticationManager, usersRepository);

            httpSecurity
                    .csrf().disable()
                    .formLogin().disable()
                    .httpBasic().disable()
                    .addFilter(corsFilter)              // 직접 작성한 corsFilter
                    .addFilter(authenticationFilter)    // 직접 작성한 authenticationFilter ("/users/login") 에서만 발동
                    .addFilter(authorizationFilter)     // 직접 작성한 authorizationFilter
                    .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                    .and()
                    .headers(customizer -> customizer.frameOptions().disable())
                    .authorizeHttpRequests()
                    .requestMatchers(
                            new AntPathRequestMatcher("/"),
                            new AntPathRequestMatcher("/img/**"),
                            new AntPathRequestMatcher("/css/**"),
                            new AntPathRequestMatcher("/users/signup")
                    )
                    .permitAll()
                    .requestMatchers(
                            new AntPathRequestMatcher("/users/logout"),
                            new AntPathRequestMatcher("/users/detail"),
                            new AntPathRequestMatcher("/users/**")
                    )
                    .hasAnyAuthority("ROLE_USER", "ROLE_MANAGER", "ROLE_ADMIN")
                    .and()
                    .userDetailsService(customDetailsService);

            return httpSecurity.build();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```

