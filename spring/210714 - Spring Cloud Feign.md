## Spring Cloud Feign



Feign은 Netflix OSS에서 시작했다.<br>
현재는 OSS에서 나와 오픈소스로 지원되고 있다. (OpenFeign)



[스프링 클라우드 문서](https://cloud.spring.io/spring-cloud-netflix/multi/multi_spring-cloud-feign.html)에 따르면 선언적 방식이라고 한다.<br>
흔히들 RestTemplate나 WebClient를 많이 사용할 텐데, Feign을 사용하면 더 편하게 이용할 수 있다고 한다.



이번에 외부 API를 사용하면서 Feign을 쓰기로 하였는데,<br>
API의 Request Content-Type이 application/x-www-form-urlencoded,<br>
Response의 Content-Type은 application/json 이였다.<br>
문제는 Response의 형식이 application/json으로 주는 것이 아니라.. text/html로 주고 있었다.<br>
![스크린샷 2021-07-05 오전 3.30.50](https://tva1.sinaimg.cn/large/008i3skNgy1gs5vgekad2j30qo073q3b.jpg)

*응답 형식이 이상하다...*



FeignClientsConfiguration를 보면 다음과 같다.<br>
![스크린샷 2021-07-05 오전 3.26.31](https://tva1.sinaimg.cn/large/008i3skNgy1gs5vghoek2j30mo0e40uu.jpg)<br>
Feign은 살펴보니 기본적으로 Encoder에 SpringEncoder, Decoder에 SpringDecoder를 사용하고 있었다. (ConditionalOnMissingBean)

또 그렇다면 SpringEncoder를 살펴보자..<br>
![스크린샷 2021-07-15 오전 2.05.07](https://tva1.sinaimg.cn/large/008i3skNgy1gsgz4hkwxbj30gz0lstce.jpg)



먼저 기본적으로 multipart/form-data를 지원하는 것을 알 수 있다.<br>
그리고 debug를 해보니 messageConverter에 json을 지원하는 messageConverter만 주입받는 것 같았다.<br>
즉, application/json과 multipart/form-data를 지원한다.<br>
하지만 요청에 application/x-www-form-urlencoded이 필요하므로... 기본 설정만으로는 해결이 안된다!

결론부터 말하자면 SpringFormEncoder를 사용해야 한다.

Encoder를 구현한 객체는 SpringEncoder, SpringFormEncoder, FormEncoder가 있다.<br>
(그리고 SpringFormEncoder는 FormEncoder를 상속받는다.)<br>
FormEncoder의 소스 중 일부는 다음과 같다.<br>
![스크린샷 2021-07-15 오전 2.12.35](https://tva1.sinaimg.cn/large/008i3skNgy1gsgzb3x7ptj30i008et9s.jpg)<br>
(multipart와 urlencoded에 관련된 ContentProcessor를 받고 있다.)

따라서 SpringFormEncoder의 경우에는 multipart/form-data와 application/x-www-form-urlencoded을 지원한다!



따라서 아래와 같이 SpringFormEncoder로 SpringEncoder를 한 번 더 감싸서,<br>
Request에 urlencoded를 파싱가능하게 해주었다.<br>
![스크린샷 2021-07-15 오전 2.17.29](https://tva1.sinaimg.cn/large/008i3skNgy1gsgzg98ujlj30h202xwev.jpg)



인코더는 이렇게 나름(?) 간단하게 끝났지만 디코더는 생각보다 더 할게 많았다.<br>
RestTemplate을 사용할때엔 ErrorHandler를 직접 만들어서 사용했는데, 아쉽게도 Feign에는 ErrorHandler를 Custom하게 사용하기 힘들었다.<br>
(기본적으로 400,500번대만 내부적으로 에러로 처리)

디코더에 관한 삽질은 내일 마저 정리해야겠다.



