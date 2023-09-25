# JWT란 (짧게 요약)



### JWT 란?

> 출처 : [http://jwt.io/introduction](https://jwt.io/introduction)



- 당사자 간에 정보를 JSON 개체로 안전하게 전송하기 위한 간결하고 독립적인 방법을 정의하는 개방형 표준 (RFC7519)
- 서명(Sign) 용도로 사용 
- JWT 는 HMAC 또는 RSA 를 사용한다. 
  - HMAC 을 쓸 때와 RSA 를 쓸 때의 사용방식이 다르다.
- JWT 의 핵심은 '서명 된 토큰'에 중심을 둔다는 점이다.
  - 당사자간에 Secret 을 제공할 수 도 있지만, '서명된 토큰'에 중점을 둔다.
- 서명된 토큰은 그 안에 포함된 어떤 정보(=클레임, 요구사항)의 무결성을 확인할 수 있게 해준다.

<br>



### JWT 의 구조 

`.` 으로 구분된 아래와 같은 형식으로 구성된다.

- {header}.{Payload}.{Signature}

- `xxxxx.yyyyy.zzzzz`

즉, **JWT 는 헤더, 페이로드, 서명(Signature)** 로 구성된다.

<br>



#### header (헤더) - "어떤 알고리즘을 사용해서 서명을 했는지"

header 는 아래와 같은 구조의 문자열로 구성되는데, 이 문자열을 Base64 URL 로 인코딩해서 JWT 의 첫번째 부분에 지정한다.

```json
{
    "alg": "HS256",
    "typ": "JWT"
}
```

<br>



#### payload (페이로드) - "JWT에 실어보낼 페이로드"

> 페이로드는 클레임들의 집합이다.



e.g.

```json
{
  "sub": "1234567890", // 등록된 클레임
  "name": "John Doe", // 개인클레임 
  "admin": true // 개인클레임
}
```

<br>



페이로드는 클레임을 포함하고 있는다.

클레임은 **엔티티 및 부가정보에 대한 선언문**이다.

클레임의 유형은 3가지가 있다.

- Registered Claim
  - 필수는 아니다. 하지만 권장된다.
  - Claim 들의 집합인데, 권장되는 Claim 들의 집합이다. 필수는 아니다.
  - 안넣어도 되고, 엄청 중요한 것은 아니다.
  - e.g. iss(issuer), exp(expiration time), sub(subject), aud(audience), [그 외 기타](https://www.rfc-editor.org/rfc/rfc7519#section-4.1)
- Public Claim
- Private Claim
  - 개인이 필요할 때 만들어서 추가해서 넣을 수 있는 클레임.
  - 사용자임을 식별할 수 있는 어떠한 primary key 같은 것들을 Primary Claim 에 넣는다.
  - 개인적으로 추가되는 정보임을 나타내는 정보
  - e.g. "user_id" : 1 



핵심은 페이로드에 무엇을 넣는지가 중요하다. 그 중 가장 중요한 것은 Private Claim (개인 클레임) 이다. 이 개인 클레임(Private Claim)에 user_id 나 primary_key 같은 식별 가능한 키를 암호화해서 넣기도 한다.

<br>



#### Signature (서명)

header, payload 와 secret(개인키, 일반적으로 서버에서 따로 보안된 위치에 따로 보관)를 HMAC 으로 암호화 한다. HS256 이라는 것은 HMAC SHA 256 이라는 알고리즘을 의미한다.

HMACSHA256 (...) 이라는 것은 프로그램의 어떤 함수라고 이해해도 된다. 쉽게 말해 HMACSHA256() 은 암호화를 수행하는 함수다. (HMAC 이라는 방식으로 SHA256 암호화를 수행.)<br>

`secret` 이라는 것은 사용자가 지정한 임의의 문자열이다. 예를 들면 'platform-team-api-gatway-svc' 같은 문자열을 팀에서 키를 조합할때 사용할 비밀키로 지정했을 때 이 'platform-team-api-gatway-svc' 라는 문자열과 `header`, `payload` 값을 HMACSHA256 함수에 인자로 해서 생성하면 서명된 문자열이 도출된다. 이것을 서명이라고 이야기한다.<br>

<br>



JWT 의 헤더는 아래와 같이 구성되는데 `"alg" : "HS256"` 이라는 것은 알고리즘으로 `HMAC SHA 256` 이라는 알고리즘을 사용하겠다는 것을 의미한다.

```json
{
    "alg": "HS256",
    "typ": "JWT"
}
```

<br>



