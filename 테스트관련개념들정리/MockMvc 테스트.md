# MockMvc 테스트



### 참고

- [MockMvc GET, POST](https://shinsunyoung.tistory.com/52)
- [Spring Boot MockMvc](https://velog.io/@geesuee/Spring-Spring-Boot-MockMvc)

<br>



### HTTP 메서드 별 테스트

예를 들면 아주 간단하게 사용하는 예는 아래와 같다.

GET

```java
mock.perform(get("/hello"))
    .andExpect(status().isOk())
    .andDo(print());
```

<br>



POST

```java
mock.perform(post("/hello"))
    .andExpect(status().isOk())
    .andDo(print());
```

<br>



PUT

```java
mock.perform(put("/hello"))
    .andExpect(status().isOk())
    .andDo(print());
```

<br>



DELETE

```java
mock.perform(delete("/hello"))
    .andExpect(status().isOk())
    .andDo(print());
```

<br>



### POST 요청을 파라미터와 함께 

/user/signup 라는 API 에 아래의 파라미터로 요청을 아래와 같이 수행한다고 해보자.

```json
{
    "email": "abc@gmail.com",
    "password": 1234
}
```

<br>



MockMvc로 위와 같은 상황은 파라미터 가정을 해서 테스트를 할 수 있다.

<br>



**1\) 파라미터를 String으로 직렬화해서 요청하는 테스트**<br>

예를 들어 Json 요청의 RequestBody 를 단순 문자열로 변환한 파라미터로 POST 요청을 보내는 것을 가정하는 코드는 아래와 같다.

```java
mock.perform(post("/user/signup"))
    // 파라미터
    .content("{\"email\" : \"abc@gmail.com\", \"password : 1234\"}")
    .contentType(MediaType.APPLICATION_JSON)
    .accept(MediaType.APPLICATION_JSON)
    .andExpect(status().isOk())
    .andDo(print());
```

<br>



위와 같은 코드를 일일이 손으로 적을 수는 없다. 테스트시에 시간이 많이 걸리고, 변수에 대한 값을 바꿔서 테스트하기에 유연하지 않다.

```java
@Autowired
private ObjectMapper objectMapper;

@Autowired
private MockMvc mockMvc;

public void TEST1(){
    User user = new User("abc@gmail.com", 1234);
    String serializedParam = objectMapper.writeValueAsString(user);
    
    mockMvc.perform(post("/user/signup"))
        .content(serializedParam)
        .contentType(MediaType.APPLICATION_JSON)
        .accept(MediaType.APPLICATION_JSON)
        .andExpect(status().isOk())
        .andDo(print());
}
```

<br>



**2\) MultivalueMap 으로 파라미터를 세팅해서 POST 요청하는 테스트**

```java
MultivalueMap<String, String> param = new LinkedMultiValueMap<>();
param.add("email", "abc@gmail.com");
param.add("password", "1234");

mock.perform(post("/user/signup"))
    // 파라미터
    .params(param)
    .andExpect(status().isOk())
    .andDo(print());
```

<br>



### GET 요청을 파라미터와 함께

/api/posts?offset=1&size=10 

<br>



**1\) QueryParam 테스트**<br>

예제로 사용할 Controller 는 아래와 같은 모양이다.

```java
@RestController
public class SampleController {
    @GetMapping("/sample/page1")
    public String getSamplePage(@RequestParam int size, @RequestParam Long offset){
        return "page1";
    }
    
    // ...
}
```



<br>



MultiValueMap 이용방식

```java
@Test
public void TEST1() throws Exception{
    MultiValueMap<String, String> param = new LinkedMultiValueMap<>();
    param.add("size", String.valueOf(10));
    param.add("offset", String.valueOf(1));

    this.mockMvc
            .perform(
                    get("/sample/page1")
                            .params(param)
            )
            .andDo(print())
            .andExpect(status().isOk())
            .andExpect(content().string(containsString("page1")));
}
```

<br>



MockHttpServletRequestBuilder 의 queryParam(String, String) 이용방식

```java
@Test
public void TEST_USING_PARAM() throws Exception{
    this.mockMvc
            .perform(
                    get("/sample/page1")
                            .queryParam("size", String.valueOf(10))
                            .queryParam("offset", String.valueOf(1))
            )
            .andDo(print())
            .andExpect(status().isOk())
            .andExpect(content().string(containsString("page1")));

}
```

<br>



2\) PathVariable 테스트

Controller 는 아래와 같이 작성되어 있는 상태

```java
@RestController
public class SampleController {
    // ..
    
    @GetMapping("/sample2/{pageName}")
    public String getSamplePage2(@PathVariable String pageName){
        return pageName;
    }
}
```

<br>



위의 코드를 테스트하기 위한 테스트코드는 아래와 같이 작성

```java
@Test
public void TEST_USING_PATHVARIABLE() throws Exception{
    this.mockMvc
            .perform(
                    get("/sample2/{pageName}", "page1")
            )
            .andDo(print())
            .andExpect(status().isOk())
            .andExpect(content().string(containsString("page1")));
}
```

<br>







