## Spring Boot 2.x로 OAuth 2.0 구현하기 w/ SPA(Vue.js, React)  - (1)

이미 스프링 부트와 OAuth 2.0을 여~러번 구현한 적이 있는데도 불구하고 다시 구현하려고 하니 헷갈리는 점이 많았다.

이번이 마지막이지 다음엔 헤매지 않기로 결심하고 이번 기회에 정리를 처음부터 끝까지 하기로 했다.



이미 기존에 SNS 로그인 연동을 스프링 부트로 구현한 코드와 블로그 예제는 정말 많다.

나 또한 이동욱(jojoldu)님의 [스프링 부트와 AWS로 혼자 구현하는 웹 서비스]을 보면서 간단한 SNS 로그인을 구현한 적이 있었기에

*생각보다 별거 아닌데??*

라고 생각하고 있었다. (심지어 이 땐 OAuth Flow도 잘 몰랐음)



하지만 SPA 입장에서는 달랐다..

구글링을 잘 못한 (구글링도 실력이다..) 나의 잘못도 있지만 관련 코드는 왜 이렇게 부족한지, 이 Bean은 왜 구현했는지 정보가 확실히 적었다.

그래서 다른 분들은 이런 시행착오를 덜으셨으면 (다 해결해주지는 못해요 ㅠ) 하는 마음에 코드를 (내 입맛대로) 정리해본다.



## application.yml

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          naver:
            clientName: "Naver"
            clientId: > 네이버 클라이언트 ID <
            clientSecret: > 네이버 클라이언트 시크릿 <
            scope:
              - profile
            redirectUri: "{baseUrl}/api/oauth2/callback/{registrationId}"
            clientAuthenticationMethod: "POST"
            authorizationGrantType: "authorization_code"
          kakao:
            clientName: "Kakao"
            clientId: > 카카오 클라이언트 ID < 
            clientSecret: > 카카오 클라이언트 시크릿 <
            scope:
	            - profile
            redirectUri: "{baseUrl}/api/oauth2/callback/{registrationId}"
            clientAuthenticationMethod: "POST"
            authorizationGrantType: "authorization_code"
        provider:
          naver:
            authorizationUri: "https://nid.naver.com/oauth2.0/authorize"
            tokenUri: "https://nid.naver.com/oauth2.0/token"
            userInfoUri: "https://openapi.naver.com/v1/nid/me"
            userNameAttribute: "response"
          kakao:
            authorizationUri: "https://kauth.kakao.com/oauth/authorize"
            tokenUri: "https://kauth.kakao.com/oauth/token"
            userInfoUri: "https://kapi.kakao.com/v2/user/me"
            userNameAttribute: "id"
```



전엔 CommonOAuth2Provider(스프링에서 기본적으로 구글, 페이스북, 깃허브, 옥타의 설정값을 제공해주는데, 그 설정값이 들어있는 클래스)와 비슷하게

CustomOAuth2Provider 라는 Enum 클래스를 만들어서 활용하였다.



**![스크린샷 2021-05-02 오전 6.35.43](/Users/hyeonhyeop/Library/Application Support/typora-user-images/스크린샷 2021-05-02 오전 6.35.43.png)**

위와 같은 Enum 클래스를 만들어서, 빌더를 만든 후



**![스크린샷 2021-05-02 오전 6.37.06](/Users/hyeonhyeop/Desktop/스크린샷 2021-05-02 오전 6.37.06.png)**

ClientRegistration List에 위에서 만든 빌더를 이용해 provider의 설정값들을 넣어주었다.

하지만 yml과 클래스 두 군데에 설정값을 저장해서 사용하는 것 과,

CommonOAuth2Provider을 위해 클래스를 따로 선언해야 하는 것 (자동설정의 이점을 살리지 못하는 것) 때문에 이번에는 yml에 OAuth 관련 설정값들을 다 넣어보았다.

참고로 여기서는 네이버, 카카오만 설정했지만, 기존에 스프링에서 제공하는 구글, 깃허브, 페북의 경우에도 구현을 하려면 yml에 설정값을 넣어야한다.



## public class SecurityConfig extends WebSecurityConfigurerAdapter

```java
 http
                .cors()					
                    .disable()			
                .csrf()
                    .disable()	
                .sessionManagement()
                    .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                    .and()
                .authorizeRequests()
                    .anyRequest()
                        .permitAll()
                        .and()
                .oauth2Login()
                    .authorizationEndpoint()
                        .baseUri("/api/oauth2/authorization")
                        .authorizationRequestRepository(cookieOAuth2AuthorizationRequestRepository())
                        .and()
                    .redirectionEndpoint()
                        .baseUri("api/oauth2/callback/*")
                        .and()
                    .userInfoEndpoint()
                        .userService(customOAuth2UserService)
```

CORS와 CSRF는 이 글에서는 설명하기엔 길어질까봐 disable() 처리.

먼저 로그인 과정에서 세션을 사용 안할 것이기 때문에 SessionCreationPolicy.STATELESS로 설정해주자. (참고로 아예 세션을 생성하지 않는 것은 아니다.)

authorizeRequests()도 똑같이 글이 길어질까봐 permitAll

oauth2Login()이 핵심 부분이다.



#### authorizationEndpoint()에서 인가와 관련된 설정을 할 수 있다.



**baseUri**

baseUri는 프론트(뷰 단)에서 백엔드로 로그인 요청할때의 URL이다.

예를 들어 domain이 http://spring.backend.com 이면,

현재 프론트에서 네이버 로그인의 URL은 http://spring.backend.com/api/oauth2/authorization/naver 이어야 한다.





**authorizationRequestRepository**



이 코드를 이해하기 위해선 먼저 도큐먼트를 읽는 것이 빠를 것 같다.

[스프링 시큐리티 도큐먼트](https://docs.spring.io/spring-security/site/docs/5.3.2.RELEASE/reference/html5/#oauth2Client-auth-code-grant)를 보면 다음과 같다.

***

The `AuthorizationRequestRepository` is responsible for the persistence of the `OAuth2AuthorizationRequest` from the time the Authorization Request is initiated to the time the Authorization Response is received (the callback).

> The `OAuth2AuthorizationRequest` is used to correlate and validate the Authorization Response.

The default implementation of `AuthorizationRequestRepository` is `HttpSessionOAuth2AuthorizationRequestRepository`, which stores the `OAuth2AuthorizationRequest` in the `HttpSession`.

***

 [토리맘님의 번역 문서](https://godekdls.github.io/Spring%20Security/oauth2/)에는 다음과 같다.

`AuthorizationRequestRepository`는 인가 요청을 시작한 시점부터 인가 요청을 받는 시점까지 (콜백) `OAuth2AuthorizationRequest`를 유지해준다.

> `OAuth2AuthorizationRequest`는 인가 응답을 연계하고 검증할 때 사용한다.

`AuthorizationRequestRepository`의 디폴트 구현체는 `HttpSession`에 `OAuth2AuthorizationRequest`를 저장하는 `HttpSessionOAuth2AuthorizationRequestRepository`다.

***



OAuth 로그인을 요청할때 state가 CSRF 토큰의 역할을 한다. [OAuth 2.0 대표 취약점과 보안 고려 사항 알아보기](https://meetup.toast.com/posts/105)



이 state는 OAuth2AuthorizationRequestResolver에서 자동으로 생성해주며,

우리가 따로 구현하지 않으면 DefaultOAuth2AuthorizationRequestResolver를 스프링에서 사용한다.

**![스크린샷 2021-05-02 오전 7.38.42](/Users/hyeonhyeop/Library/Application Support/typora-user-images/스크린샷 2021-05-02 오전 7.38.42.png)**

(DefaultOAuth2AuthorizationRequestResolver가 StateGenerator 라는 것을 이용해 state를 알아서 넣어주고 있다!)



따라서 Request에서 보낸 state를 Response의 state와 검증을 해야하는데,

그 요청을 스프링에서는 OAuth2AuthorizationRequest가 들고 있고,

이 요청을 저장해 주는 곳이 AuthorizationRequestRepository인 것이다.



하지만 우리는 Session 기반 인증이 아니므로 AuthorizationRequestRepository를 구현해야 하며

그래서 만든 구현체가 CookieOAuth2AuthorizationRequestRepository 라고 이해하면 된다.



**CookieOAuth2AuthorizationRequestRepository**

```java
@Slf4j
@Component
public class HttpCookieOAuth2AuthorizationRequestRepository implements AuthorizationRequestRepository {

    public static final String OAUTH2_AUTHORIZATION_REQUEST_COOKIE_NAME = "oauth2_auth_request";
    public static final int cookieExpireSeconds = 180;

    @Override
    public OAuth2AuthorizationRequest loadAuthorizationRequest(HttpServletRequest request) {
        log.info("=============== loadAuthorizationRequest ==============================");
        OAuth2AuthorizationRequest oAuth2AuthorizationRequest = CookieUtil.getCookie(request, OAUTH2_AUTHORIZATION_REQUEST_COOKIE_NAME)
                .map(c -> CookieUtil.deserialize(c, OAuth2AuthorizationRequest.class))
                .orElse(null);
        return oAuth2AuthorizationRequest;
    }

    @Override
    public void saveAuthorizationRequest(OAuth2AuthorizationRequest authorizationRequest, HttpServletRequest request, HttpServletResponse response) {
        log.info("=============== saveAuthorizationRequest ==============================");
        CookieUtil.addCookie(response, OAUTH2_AUTHORIZATION_REQUEST_COOKIE_NAME, CookieUtil.serialize(authorizationRequest), cookieExpireSeconds);
    }

    @Override
    public OAuth2AuthorizationRequest removeAuthorizationRequest(HttpServletRequest request) {
        log.info("removeAuthorizationRequest");
        return this.loadAuthorizationRequest(request);
    }
```



처음에 SNS 로그인을 시도하면 saveAuthorizationRequest에서 인증 요청 객체를 쿠키에 저장하고,

후에 정상적으로 우리 서버로 callback 요청이 들어오면, 그때 저장했던 인증 요청 객체 쿠키를 불러온다. (CustomOAuth2SuccessHandler 에서)



**CustomOAuth2SuccessHandler**

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class CustomOAuth2SuccessHandler extends SimpleUrlAuthenticationSuccessHandler {

    private final TokenProvider tokenProvider;
    private final HttpCookieOAuth2AuthorizationRequestRepository httpCookieOAuth2AuthorizationRequestRepository;

    @Value("${front.url.login-callback}")
    private String frontUrl;

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException {
        String targetUrl = determineTargetUrl(request, response, authentication);

        clearAuthenticationAttributes(request, response);
        getRedirectStrategy().sendRedirect(request, response, targetUrl);
    }

    private String determineTargetUrl(HttpServletRequest request, HttpServletResponse response, Authentication authentication) {
        String token = tokenProvider.createUserToken((UserPrincipal) authentication.getPrincipal());
        String redirectUri = UriComponentsBuilder.fromUriString(frontUrl).queryParam("token", token).build().toUriString();
        return redirectUri;
    }

    private void clearAuthenticationAttributes(HttpServletRequest request, HttpServletResponse response) {
        super.clearAuthenticationAttributes(request);
        httpCookieOAuth2AuthorizationRequestRepository.removeAuthorizationRequestCookies(request, response);

    }

    @Override
    protected String determineTargetUrl(HttpServletRequest request, HttpServletResponse response) {
        return super.determineTargetUrl(request, response);
    }
}
```

우리 서버 URL로 콜백된 유저는 결국 프론트 URL로 가야하기 때문에, 프론트 콜백 URL을 프로퍼티에 저장해서 불러오게 하였다.

정상적으로 우리 서버로 콜백 된 유저는 onAuthenticationSuccess로 들어가서 프론트 콜백 URI를 받는다.

TokenProvider에서 사용자의 로그인 토큰(JWT)을 만들어, 그 값을 쿼리 파라미터로 넘겨서 프론트 콜백 URI를 만든다.

그리고 인증 처리가 완료되었으므로, 사용자 인증 객체를 저장한 쿠키는 삭제를 한다. (clearAuthenticationAttributes)

그리고 프론트 URI로 사용자를 리다이렉트 시켜서 로그인 처리를 완료한다.





**UserPrincipal**

```java
@AllArgsConstructor
@NoArgsConstructor
@Getter
@Builder
@ToString
public class UserPrincipal implements OAuth2User {
    private Long id;
    private String username;
    private String password;
    private Collection<? extends GrantedAuthority> authorities;
    private Map<String, Object> attributes;


    public static UserPrincipal create(User user, Map<String, Object> attributes){
        List<GrantedAuthority> authorities = Collections.singletonList(new SimpleGrantedAuthority("ROLE_USER"));
        UserPrincipal userPrincipal = UserPrincipal.builder()
                .id(user.getId())
                .username(user.getName())
                .password(null)
                .authorities(authorities)
                .build();
        userPrincipal.attributes = attributes;
        return userPrincipal;
    }
}
```

**KakaoOAuth2User**

```java

@Getter
@Setter
public class KakaoOAuth2User implements OAuth2User {
    private List<GrantedAuthority> authorityList = AuthorityUtils.createAuthorityList("ROLE_USER");
    private Map<String, Object> attributeList;
    @JsonSetter("id")
    private String name;

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return this.authorityList;
    }

    @Override
    public Map<String, Object> getAttributes() {
        return this.attributeList;
    }

    @JsonProperty("properties")
    public void unpack(Map<String, Object> data){
        this.attributeList = new HashMap<>();
        this.attributeList.put("username", data.get("nickname"));
    }
}
```

**NaverOAuth2User**

```java

@Getter
@Setter
public class NaverOAuth2User implements OAuth2User {

    private List<GrantedAuthority> authorityList = AuthorityUtils.createAuthorityList("ROLE_USER");
    private Map<String, String> attributeList;
    private String name;

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return this.authorityList;
    }

    @Override
    public Map<String, Object> getAttributes() {
        return (Map) this.attributeList;
    }


    @JsonProperty("response")
    public void unpack(Map<String, String> data){
        this.name = data.get("id");
        this.attributeList = new HashMap<>();
        attributeList.put("username", data.get("name"));
    }
}

```

네이버와 카카오의 각 프로필 정보 응답에 맞게 OAuth2User을 구현한 객체를 만들어주었다.

여기서는 name이 각 유저 객체의 고유 ID값이 된다.

(참고로 얼마 전부터 네이버의 고유 ID가 숫자에서 "Ab9onJ-Ko-SdQWIZK7SBmQf0To3-z5d2Bbqfxb4Rl2s" 같은 문자열로 바뀌었으니 남들과 다르다고 놀라지 않아도 된다.. 절대 내 얘기 아님.)

그리고 이 객체들을 UserPrincipal라는 객체로 받아서 쓰도록 하였다.





**TokenProvider**

```java

@Component
@Slf4j
public class TokenProvider {
  
public String createUserToken(UserPrincipal userPrincipal) {
        UserPrincipal userPrincipal = (UserPrincipal) authentication.getPrincipal();
        log.info("principal is {} " , userPrincipal.toString());
        List<? extends GrantedAuthority> authorities = (List<? extends GrantedAuthority>) userPrincipal.getAuthorities();
        log.info("principal is {} " , authorities.get(0));
        Date now = new Date();

        Map<String, Object> claims = new HashMap<>();
        claims.put("userId", userPrincipal.getId());
        claims.put("role", authorities.get(0));
        claims.put("name", userPrincipal.getName());
  			String secret = userPrincipal.getId() + userPrincipal.getName();

        return Jwts.builder()
                .setClaims(claims)
                .setSubject(Long.toString(userPrincipal.getId()))
                .setIssuedAt(now)
                .setExpiration(new Date(now.getTime() + TOKEN_VALID_TIME))
                .signWith(SignatureAlgorithm.HS256, Base64.getEncoder().encodeToString(secret.getBytes(StandardCharsets.UTF_8)))
                .compact();
    }
  
  private Claims parseWithoutSecret(String token) {
        try {
            String[] splitToken = token.split("\\.");
            String unsignedToken = splitToken[0] + "." + splitToken[1] + ".";

            Jwt<Header, Claims> claimsJws = Jwts.parser().parseClaimsJwt(unsignedToken);
            return claimsJws.getBody();
        } catch (SignatureException | MalformedJwtException | UnsupportedJwtException e) {                              // Signature 다를 시 (우리쪽에서 발급한 토큰이 아님) or 구조 이상할
            throw new ErrorException(ErrorStatus.USER_UNAUTHORIZED);
        } catch (ExpiredJwtException e) {
            throw new ErrorException(ErrorStatus.USER_AUTHORIZATION_EXPIRED);
        }
    }
  
  public Long getValueFromLoginToken(String authorization, String key) {
        String token = authorization.substring(AUTH_TYPE.length());
        Claims claimsWithoutSecret = parseWithoutSecret(token);

        String userId = Optional.ofNullable(claimsWithoutSecret.get("userId").toString()).orElse("");
        String name = Optional.ofNullable(claimsWithoutSecret.get("name").toString()).orElse("");
        
        Claims claimsWithSecret = parseWithSecret(token, Base64.getEncoder().encodeToString((userId + name ).getBytes(StandardCharsets.UTF_8)));

        Object object = Optional.ofNullable(claimsWithSecret.get(key)).orElseThrow(() -> new ErrorException(ErrorStatus.USER_UNAUTHORIZED));
        return Long.valueOf(object.toString());
    }
}
```



secret은 각 유저의 고유아이디 + 유저이름으로 정하였다.

원래는 fixed된 스트링의 base64 값을 사용했는데, 보안이 중요한 서비스에서는 저런식으로 동적으로 키를 설정한다는 말을 듣고 바꿔보았다.

그리고 토큰을 검증해야할 땐, parseWithoutSecret로 먼저 시크릿 없이 파싱을 한 뒤 (페이로드와 클레임만 남기고, 시그니처를 날려버린 토큰)

클레임의 값을 조합해서 시크릿을 만들고, 그 시크릿으로 Jws를 받아서 검증하는 식으로 구현하였다.

이 코드도 물론 누군가가 class 파일을 뜯는다면 소용 없겠지만, 그래도 정적인 시크릿 키 보다는 나을 것 같다.



