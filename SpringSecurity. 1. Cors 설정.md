### Cors 설정

Cors 설정 역시 필터로 생성해서 추가해주면 된다.

아래는 예제 코드다.

```kotlin
package net.spring.cloud.prototype.userservice.domain.config.security.filter

// ...

@Configuration
class CorsFilterConfig {
    val apiUrl: String = "/api/**"

    @Bean
    fun corsFilter(): CorsFilter {
        val source = UrlBasedCorsConfigurationSource()
        val config = CorsConfiguration()

        config.allowCredentials = true 			// (1)
        config.addAllowedOriginPattern("*") // (2)
        config.addAllowedHeader("*") 				// (3)
        config.addAllowedMethod("*") 				// (4)

        source.registerCorsConfiguration(apiUrl, config)
        return CorsFilter(source)
    }
}
```

<br>



- (1) 
  - 쿠키 요청을 허용한다. 디폴트 값은 false 로 선언된다. 쿠키 요청은 보안상 취약하기에 디폴트 설정은 false 로 적용되어 있다.
  - OAuth 처럼 다른 도메인에서 인증하는 경우 (e.g. auth.google.com ↔ user.abc.com) 에만 허용(true)한다.
  - 구글 로그인 기능도 추가할 것이기에 true 로 지정했다.

- (2)
  - 어떤 출처(Origin)에서의 요청을 허용할 지를 지정
  - 위의 예제에서는 `*` 으로 지정해서 전체 요청을 허용하게 해줬다.
  - Spring Boot 2.4.0 부터는 allowCredentials 가 true 일 때 모든 출처 허용하는 구문 작성시 allowedOrigins("\*") 처럼 사용하지 못하고 allowedOriginPattern("\*") 으로 지정해줘야 한다.
    - [Not possible to use allowedOrigins "*" in StompEndpointRegistry after upgrade to Spring Boot 2.4.0](https://github.com/spring-projects/spring-framework/issues/26111)
    - [Updates to CORS patterns contribution](https://github.com/spring-projects/spring-framework/commit/0e4e25d227dedd1a3ecddc4e40c263f190ca1c2b)
- (3)
  - 요청 내의 어떤 헤더를 허용할 것인지 지정
- (4)
  - GET,POST,... etc

<br>



### 참고자료

- [CORS 를 해결하는 3가지 방법](https://wonit.tistory.com/572)
- [allowCredentials true 설정시 Access-Control-Allow-Origins 에...](https://gareen.tistory.com/66)
- [CORS 설정시 allowed Origins 에러](https://cotak.tistory.com/248)
- [Not possible to use allowedOrigins "*" in StompEndpointRegistry after upgrade to Spring Boot 2.4.0](https://github.com/spring-projects/spring-framework/issues/26111)
- [Updates to CORS patterns contribution](https://github.com/spring-projects/spring-framework/commit/0e4e25d227dedd1a3ecddc4e40c263f190ca1c2b)

<br>



