## RestTemplate

Spring 에서 Client처럼 Http 통신을 하는 법엔 크게 3가지가 있다.

1. RestTempalte
2. WebClient
3. Feign

주로 Feign을 써왔는데 RestTemplate도 알면 좋겠다 싶어서 정리했다.

![스크린샷 2021-04-12 14.33.45](/Users/hyeop/Library/Application Support/typora-user-images/스크린샷 2021-04-12 14.33.45.png)

(하지만 스프링 5.0부터 RestTemplate이 Maintenance 모드로 전환되었다..)



먼저 RestTemplate를 Bean으로 등록해주자.

```java
		@Bean
    public RestTemplate innerTemplate(RestTemplateBuilder restTemplateBuilder){
        CloseableHttpClient httpClient = HttpClientBuilder.create()
                .setMaxConnTotal(120)
                .setMaxConnPerRoute(30)
                .setConnectionTimeToLive(5, TimeUnit.SECONDS)
                .build();
        HttpComponentsClientHttpRequestFactory factory = new HttpComponentsClientHttpRequestFactory();
        factory.setHttpClient(httpClient);

        return restTemplateBuilder
								.requestFactory(() -> new BufferingClientHttpRequestFactory(factory))
                .setConnectTimeout(Duration.ofMillis(5000))
                .setReadTimeout(Duration.ofMillis(5000))
                .errorHandler(new usebIdentityErrorHandler())
                .additionalMessageConverters(new StringHttpMessageConverter(StandardCharsets.UTF_8))
                .build();
    }
```

여러개의 RestTemplate를 Bean으로 등록할 수 있으며, 이럴 때 @Bean(name = )으로 이름을 설정해도 되지만,

메소드의 이름(innerTemplate)로 설정할 수도 있다.

최대 연결 갯수는 120개, 각 루트당 연결 가능한 갯수는 30개로 설정하였다.

접속 한 뒤 5초까지 연결을 유지시키게 설정하였다. (연결이 안돼서 시도하는 것과는 다르다.)

https://hoonmaro.tistory.com/49

Spring Web RestTemplate을 이용한 HTTP 통신시 Interceptor를 활용하여 요청과 응답 로그를 남긴다.

주의할 점은, ResponseEntity의 Body는 Stream 이므로 로깅 인터셉터에서 Body Stream을 읽어 소비가되면 실제 비즈니스 로직에서는 Body가 없어진다.

이를 해결하기 위해 RestTemplate Bean 설정에서 requestFactory에 BufferingClientHttpRequestFactory를 세팅해줘야 한다.



출처: https://hoonmaro.tistory.com/49 [훈마로의 보물창고]





https://renuevo.github.io/spring/resttemplate-thread-safe/

SimpleClientHttpRequestFactory의 Connection 방식은 매번 새로운 Connection을 만들어서 통신합니다
이 방식은 계속해서 통신을 해야하는 서비스 구조해서는 효율적이지 못한 방식입니다
그래서 `Connection pool`을 적용하기 위해서 `HttpComponentsClientHttpRequestFactor`을 `SimpleClientHttpRequestFactory`대신해서 사용합니다





https://multifrontgarden.tistory.com/249





https://nesoy.github.io/articles/2020-05/RestTemplate