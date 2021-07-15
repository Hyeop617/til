## Spring Cloud Feign - Decoder

인코더에 관한 내용을 다루었으니 이번엔 디코더에 관해 정리하겠다.

Feign의 기본 설정을 담당하는 FeignClientsConfiguration을 보면 기본적으로 Decoder는 다음과 같다.

![스크린샷 2021-07-16 오전 1.47.13](https://tva1.sinaimg.cn/large/008i3skNgy1gsi47z5ivuj30kd03awem.jpg)

OptionalDecoder는 Optional로 디코딩을 지원하고, ResponseEntityDecoder는 ResponseEntity로 디코딩을 지원한다.

테스트는 해보지 않았으나 위의 코드대로면 Optional이나 ResponseEntity로도 응답을 받을 수 있을 것이다.

하지만 핵심은 역시 SpringDecoder 이 클래스를 살펴봐야 한다.



SpringDecoder도 SpringEncoder와 유사하게 HttpMessageConverter를 통해서 응답을 디코딩한다.

전에 인코더에서는 살펴보지 않았지만, 스프링 부트에서 기본적으로 지원하는 HttpMessageConverter는 다음과 같다.

![스크린샷 2021-07-16 오전 1.58.11](https://tva1.sinaimg.cn/large/008i3skNgy1gsi4ih39dwj30nh0hgtad.jpg)



Byte, String, JSON, XML등 다양한 포맷을 지원하는 것을 알 수 있다.

(예전에 백기선님의 강의를 보면서 보고 그냥 넘어갔었는데, 이렇게 중요한 것일지 몰랐다..)



여기서 JSON 파싱을 담당하는 MappingJackson2HttpMessageConverter의 코드를 살펴보자.

![스크린샷 2021-07-16 오전 2.04.59](https://tva1.sinaimg.cn/large/008i3skNgy1gsi4pitoxmj30ph05g3zb.jpg)

MappingJackson2HttpMessageConverter은 AbstractJackson2HttpMessageConverter을 상속받고 있다.

그리고 ObjectMapper를 인자로 받는 생성자가, AbstractJackson2HttpMessageConverter의 생성자를 호출하는 것을 알 수 있다.

(ObjectMapper와 MediaType[] 배열을 넘긴다.)



AbstractJackson2HttpMessageConverter의 생성자는 또 다음과 같다.

![스크린샷 2021-07-16 오전 2.06.35](https://tva1.sinaimg.cn/large/008i3skNgy1gsi4r6nbuqj30m202e74j.jpg)

인자로 받은 MediaType 여러개를 setSupportedMediaTypes라는 메소드에 넘기는 것을 알 수 있다.

자세히 소스를 살펴보지는 않았으나.. 메소드명이나 MediaType이라는 클래스명 부터 유추할 수 있었다.

즉, Content-Type이 application/json인 것만 MappingJackson2HttpMessageConverter은 지원한다!



사실 너무나도 당연한 소리다.

당연히 JSON형식인 것만 JSON으로 파싱이 가능할테니까

그런데 내 경우에는 외부 API가 분명히 JSON 형식으로 응답을 주고 있지만, Content-Type이 text/html인 상황이였으므로,

위의 소스를 뜯어서 확인할 필요가 있었다..

따라서 Decoder도 다음과 같이 설정해주었다.



![스크린샷 2021-07-16 오전 2.09.48](https://tva1.sinaimg.cn/large/008i3skNgy1gsi4uj4q8nj30mp04p74w.jpg)

(위의 CustomDecoder는 일단 무시하자)

MappingJackson2HttpMessageConverter의 인스턴스를 새로 생성하는데,

대신 새로 생성한 인스턴스는 모든 MediaType을 지원하도록 설정하였다.

따라서 text/html도 문제없이 디코딩할 수 있을것이고, 테스트 해본결과 문제 없이 실행되는 것을 확인할 수 있었다.



그리고 앞서 말했던 예외 처리를 위해 설정을 하려고 찾아보았는데..

여러모로 알아보았지만 CustomDecoder를 설정해 Decoder내에서 해결하는 방법밖에 찾지 못했다.

이 방법은 나중에 다시 정리해야겠다.