### Spring Boot HTTPS 적용



### 참고자료

- [Spring Boot HTTP SSL 적용하기](https://velog.io/@cho876/Springboot-httpsSSL-%EC%A0%81%EC%9A%A9%ED%95%98%EA%B8%B0)

<br>



### HTTP vs HTTPS

https 는 http 프로토콜에 암호화 기능이 추가된 프로토콜이다. https 는 대칭키 암호화/ 비대칭 키 암호화 방식을 통해 네트워크 상에서 제 3자가 정보를 볼 수 없도록 암호화를 지원한다.<br>

http 는 기본 포트로 80 포트를 사용하고

https 는 기본 포트로 443 포트를 사용한다.

<br>



### 신규프로젝트 생성

#### 의존성

의존성은 아래 1개만 추가해준다.

- sprint web 

<br>



#### keystore 생성

```bash
keytool -genkey -alias spring -storetype PKCS12 -keyalg RSA -keysize 2048 -keystore keystore.p12 -validity 4000
```

<br>



위와 같이 명령을 내렸을 때 `keystore.p12` 라는 파일이 현재 경로에 생성된다.<br>

<br>



#### application.properties 수정

springboot 프로젝트가 현재 만들어진 keystore 를 인식할수 있도록 속성들을 기입해준다.

```properties
server.port: 8084 // 개인적으로 포트를 8084로 바꿔줌 (ssl과 무관)

server.ssl.key-store:keystore.p12
server.ssl.key-store-type=PKCS12
server.ssl.key-store-password=qwerty1!
```

- **server.ssl.key-store**: keystore.p12가 위치한 경로 (default: 현재 프로젝트 root)
- **server.ssl.key-store-type**: keystore의 storetype
  우리는 앞선 명령어에서 `keytool ~ -storetype PKCS12 ~`로 설정했기 때문에 **PKCS12**로 잡아준다.
- **server.ssl.key-store-password**: 앞서 설정한 keystore password

<br>



### 결과확인

`https://localhost:8084` 로 접속해보자.

최초로 접속하는 경우 보안 관련해서 경고문구가 뜬다. 이것은 SpringBoot 가 사용하는 keystore 가 약식으로 만들어진 것이기 때문이다.

<br>

