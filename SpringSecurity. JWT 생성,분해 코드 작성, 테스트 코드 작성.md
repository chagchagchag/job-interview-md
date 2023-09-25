# JWT 생성,분해 코드 작성, 테스트 코드 작성

> 다른 스터디도 해야할 상황이라 정리하는 시간이 부족해질 수도 있기에 일단은 정리 안된 문서라도 급하게라도 정리한 내용들을 계속 해서 추가해냐갈 예정



### 의존성 추가

build.gradle

```groovy
// ...

dependencies {
    // ...
    
	implementation 'io.jsonwebtoken:jjwt-api:0.11.2'
	implementation 'io.jsonwebtoken:jjwt-impl:0.11.2'
	implementation 'io.jsonwebtoken:jjwt-jackson:0.11.2'
    
    // ...
}

// ...
```

<br>

### SecurityProperties.java

- 그냥 임의로 만든 파일이다.
- SECRET 은 어느 정도 길이가 되어야만 에러가 나지 않는다.

```java
package io.sgjung.samples.security_sample_2710.config.security;

import io.jsonwebtoken.security.Keys;

import java.nio.charset.StandardCharsets;
import java.security.Key;

public class SecurityProperties {

    public static final String SECRET = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMLOPQRTTTTTTTTT";

    public static final Key key = Keys.hmacShaKeyFor(SECRET.getBytes(StandardCharsets.UTF_8));
}

```

<br>



### 테스트 코드

JWT 토큰 생성 후 생성된 토큰을 분해 후 역으로 검증하는 테스트 코드

```java
package io.sgjung.samples.security_sample_2710.config.security.filter;

import io.jsonwebtoken.*;
import io.sgjung.samples.security_sample_2710.config.security.SecurityProperties;
import org.assertj.core.api.Assertions;
import org.junit.jupiter.api.Test;

import java.security.Key;
import java.util.Date;

import static org.assertj.core.api.Assertions.assertThat;

public class JwtTokenCreateAndParseTest {

    @Test
    public void JWT_토큰_생성_후_분해하는_작업_역시_정상적으로_되는지_검증(){
        Key key = SecurityProperties.key;
        String username = "Bob Dillon";
        String password = "Moon in the river";
        Long userId = 1L;

        String jwtToken = Jwts.builder()
                .setSubject(username)
                .setExpiration(new Date(System.currentTimeMillis() + 864000000))
                .claim("id", userId)
                .claim("username", username)
                .claim("password", password)
                .signWith(key, SignatureAlgorithm.HS256)
                .compact();

        JwtParser parser = Jwts.parserBuilder()
                .setSigningKey(SecurityProperties.key)
                .build();

        Jws<Claims> claimsJws = parser.parseClaimsJws(jwtToken);

        assertThat(claimsJws.getBody().get("username", String.class)).isEqualTo(username);
        assertThat(claimsJws.getBody().get("password", String.class)).isEqualTo(password);
        assertThat(claimsJws.getBody().get("id", Long.class)).isEqualTo(userId);

    }
}
```







