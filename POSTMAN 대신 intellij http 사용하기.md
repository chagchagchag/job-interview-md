# POSTMAN 대신 intellij http 사용하기

설명은 추후 정리 예정..



참고자료

검색어 : intellij http response header, intellij http jwt header

- [Handling responses in the HTTP Client](https://blog.jetbrains.com/phpstorm/2018/04/handling-reponses-in-the-http-client/)
- [Intellij http Client 응답값 변수로 저장하고 사용하기](https://jojoldu.tistory.com/366)
  - JWT 토큰을 저장해서 다음 단계의 테스트를 수행하기 위해 참고
- [아직도 postman 쓰세요? Intellij http 를 통해 api를 테스트해보자!](https://sihyung92.oopy.io/etc/intellij/2)





내가 작성한 코드

```BASH
### 회원가입
POST http://localhost:8080/user/signup
Content-Type: application/json

{
  "username": "asdf",
  "password": 1234
}

> {%
    client.test("회원가입 테스트 ", function(){
        client.log("Response Code >> " + response.status);
        client.assert(response.status == 200 || response.status == 302, "응답 실패");
    });
%}

### 로그인
POST http://localhost:8080/login
Content-Type: application/json

{
"username": "asdf",
"password": "1234"
}

> {%
    client.test("로그인 테스트 ", function(){
        client.log("Response Code >> " + response.status);
        client.assert(response.status == 200 || response.status == 302, "응답 실패");
    });

    client.log(" >>> " + response.headers);
    client.global.set("access_token", response.headers.valueOf("Authorization"));
    client.log("토큰 : " + client.global.get("access_token"));
 %}



### /api/me
POST http://localhost:8080/api/me
Content-Type: application/json
Authorization: {{access_token}}

{
}

> {%
    client.test("/user/me 조회 테스트 ", function(){
        client.log("Response Code >> " + response.status);
        client.assert(response.status == 200 || response.status == 302, "응답 실패");
    });
%}
```