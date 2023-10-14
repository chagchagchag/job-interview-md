# @Mock, @InjectMocks, @MockBean, @SpringBootTest

테스트를 아무리 자주 사용하더라도, 안하다가 다시 보면 다시 까먹는 경우가 많다. 그래서 그럴때마다 이 글을 다시보면 좋겠다 싶어서 정리. 기억하지말고 문서를 찾아서 읽으면 되니까.

<br>



### 참고자료

- [https://giron.tistory.com/115](https://giron.tistory.com/115)

<br>



### 요약

- Controller 테스트 시 @SpringBootTest 방식 사용
  - 특정 컴포넌트만 지정해서 컴포넌트 스캔하도록 Filter 하지 않으면, 모든 컴포넌트를 로딩한다.
  - 테스트 용도의 Profile 을 따로 마련해야 한다.
  - @MockBean 을 사용한다.
  - @MockBean 객체들을 의존성 주입 하는 Target 은 @SpringBootTest 다.
- Controller 테스트 시 @InjectMocks 방식 사용
  - 모든 컴포넌트를 로딩하지 않는다.
  - @Mock 을 사용한다.
  - @Mock 객체들 의존성 주입 하는 Target 은 @InjectMocks 다.

<br>



### 컨트롤러에 의존성이 없을 때의 MockMvc 테스트 

컨트롤러 내에 의존성이 많지 않다면 단순하게 아래의 방식으로 테스트 가능하다. 설명은 생략.

SampleController.java

```java
@RestController
public class SampleController {
    @GetMapping("/sample/page1")
    public String getSamplePage(@RequestParam int size, @RequestParam Long offset){
        return "page1";
    }

    @GetMapping("/sample2/{pageName}")
    public String getSamplePage2(@PathVariable String pageName){
        return pageName;
    }
}

```

<br>



테스트코드 - SampleControllerMockMvcTest

```java
@WebMvcTest(SampleController.class)
public class SampleControllerMockMvcTest {

    private MockMvc mockMvc;

    @Autowired
    private SampleController sampleController;

    @BeforeEach
    public void init(){
        mockMvc = MockMvcBuilders
                .standaloneSetup(sampleController)
                .build();
    }

    @Test
    public void TEST_USING_MULTIVALUEMAP() throws Exception{
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

}

```

<br>



### @Mock, @InjectMocks 를 사용하는 방식

@SpringBootTest 처럼 모든 Bean 들을 로딩해서 테스트하는 방식이 아니다.

컨트롤러 외의 모든 계층을 Mock 으로 두어서 API 레벨의 코드 자체에 에러가 있는지 검증을 위해 API 계층외의 Bean 들을 가짜 객체로 @Mock 으로 선언하고, @InjectMocks 를 이용해 바인딩해서 사용한다.

<br>



**예제**<br>

Controller - UsersController.java

```java
@Controller
public class UsersController {
    private final UserServiceImpl userService;

    public UsersController(UserServiceImpl userService){
        this.userService = userService;
    }

    @PostMapping("/user/signup")
    public @ResponseBody String signup(@RequestBody SignupRequestDto signupRequestDto){
        userService.signupUser(signupRequestDto);
        return "SUCCESS"; // 임시적으로 문자를 리턴하도록 지정
    }

    @PostMapping("/api/me")
    public @ResponseBody LoginResponseDto apiMe(@AuthenticationPrincipal CustomUserDetails userDetails){
        return new LoginResponseDto(userDetails);
    }
}
```

<br>



Service - UserServiceImpl.java

의존성 객체들

- UsersRepository
- PasswordEncoder 

```java
@Service
public class UserServiceImpl {

    private final UsersRepository usersRepository;
    private final PasswordEncoder passwordEncoder;

    public UserServiceImpl(UsersRepository usersRepository,
                           @Qualifier("bCryptPasswordEncoder") PasswordEncoder bCryptPasswordEncoder){
        this.usersRepository = usersRepository;
        this.passwordEncoder = bCryptPasswordEncoder;
    }

    @Transactional
    public void signupUser(SignupRequestDto signupRequestDto){
        Users user = Users.of(
                signupRequestDto.getUsername(),
                signupRequestDto.getPassword(),
                "ROLE_USER",
                passwordEncoder
        );

        usersRepository.save(user);
    }

}
```

<br>



**테스트코드**<br>

테스트 코드의 내용을 요약해보면 이렇다.

- MockMvc 객체를 직접 생성하고, 의존성 역시 직접 주입한다.
- 주입하는 의존성들은 모두 @Mock 어노테이션을 통해 Mock 객체로 지정해준다.
- Mock 객체를 모두 중첩해서 포함하는 객체를 @InjectMock 어노테이션을 통해 Mock 객체로 지정해준다.



```java
@ExtendWith(MockitoExtension.class)
public class UsersControllerMockMvcTest {

    private MockMvc mockMvc;

    @Mock
    private PasswordEncoder passwordEncoder;

    @Mock
    private UsersRepository usersRepository;

    @InjectMocks
    private UserServiceImpl userService;


    @BeforeEach
    public void init(){
        passwordEncoder = new BCryptPasswordEncoder();

        mockMvc = MockMvcBuilders.standaloneSetup(new UsersController(userService))
                .build();
    }

    @Test
    public void USER_SIGNUP_API_TEST() throws Exception {
        SignupRequestDto signupRequest = new SignupRequestDto("asdf", "1234");
        String param = new ObjectMapper().writeValueAsString(signupRequest);

        this.mockMvc
                .perform(
                        post("/user/signup")
                                .content(param)
                                .contentType(MediaType.APPLICATION_JSON)
                )
                .andDo(print())
                .andExpect(status().isOk())
                .andExpect(content().string(containsString("SUCCESS")));
    }

    @Test
    public void API_ME() throws Exception {

    }
}
```

<br>



### @MockBean

예를 들어 인터넷에 돌아다니는 MockMvc 개념을 설명하는 테스트 코드들을 보면 실제 Repository 를 주입받아서 DB까지 테스트를 하는 예제가 많다. 이렇게 되면 테스트가 무엇을 테스트하려는지 모호해진다. "MVC 계층에서 DB까지 테스트할 것인지?" 라는 물음에 도달하게 된다.<Br>

이런 경우에 현재 테스트에서 사용하려는 Controller 에 연관된 Bean 들을 @MockBean 으로 지정한다. 그리고 예외가 나는 경우에 대해 Mockito 를 이용해 상황을 가정해준다.<br>

Controller 에서는 DB 쿼리 내용까지 집착하고 기억할 필요 없이 단순하게 웹 계층에만 테스트 범위를 국한한 테스트만 수행하면 된다. 즉 예를 들면 아래와 같은 테스트만 수행하면 된다.

- Service 계층의 비지니스 로직에서 어떤 결과를 낼 경우에 Return 시에 어떤 상태코드를 내는지
- 파라미터 바인딩이 올바른지, 요청이 이럴 때를 가정했을 때 이런 결과가 나와야 제품이 원하는 기능이다.

<br>

나 조차도 많이 알고 있는 사람은 아니기에 Controller 에서 DB까지 통으로 테스트할 때 MockMvc 를 가져다 쓰는 경우에 대해서 무슨 말을 하고 싶지는 않다. 다만, 언젠가는 코드를 수정할 때 어떤 테스트 코드가 꼬이는 등의 현상이 일어나거나, 프로그램이 잘 동작하는지 확신할 수 없는 상황이 생겨서 눈으로 다시 일일이 처음부터 검수를 하는 경우가 생길 수 도 있다는 점은 기억해야 할 것 같다. <br>

매우 기본적인 내용이라 이런 개념들은 이미 정확하게 알고 있는 분들이 많지 않을 까 싶다.

<br>



### @SpringBootTest

통합테스트나 DB 테스트를 수행할 때 쓰는데, 가급적 DB 테스트는 또 DB만 국한해서 테스트하는 경우가 많다. @SpringBootTest 시에는 주의해야 할 점은 Profile을 잘못 지정해서 운영/QA Profile 에서 테스트 할 수 있는 경우를 조심해야 할 것 같다.<br>



