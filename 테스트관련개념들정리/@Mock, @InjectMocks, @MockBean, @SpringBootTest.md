# @Mock, @InjectMocks, @MockBean, @SpringBootTest

MockMvc 테스트는 서버 애플리케이션의 비즈니스 로직이 복잡해졌을 때, 비즈니스로직을 제외하고 API 에서 에러가 나는지를 검증하기 위한 용도로 쓴다. 즉, 단순 API 콜만 문제인지를 확인하기 위해 사용한다. <Br>

가끔 API에서 에러가 날때 어떤게 문제인지 모든문제들이 중첩되어있기에 찾기가 어려운 경우가 있다. 이런 경우에 API 계층에 한정해서만 테스트 할 때 MockMvc 는 유용하다.<br>

코드가 많아지고, 요구사항이 많아지고, 사업이 발전해나갈수록 기능추가가 이뤄지고 애플리케이션에 Side-Effect 라는 것은 항상 발생하는데, 이런 것들을 검출해내려면 일부 계층에 한정해서 테스트하거나 계층을 적당한 크기나 범위로 한정해놓으면, 에러를 검출하기 좋다.<br>

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



### @MockBean, @SpringBootTest

@SpringBootTest 를 사용한다고 하더라도 컴포넌트 스캔할 대상을 지정해서 특정 범위에 한정해서 테스트를 할 수 있다. 그리고 의존성 객체들은 @MockBean 등을 통해 목객체로 지정해준다. 내 경우는 Service 계층은 가급적이면 순수 자바와 순수 객체로만 테스트 가능하도록 설계/구현 하기에 주로 사용하지는 않는 방식이다. 그래서 예제는 생략.

<br>







