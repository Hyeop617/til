0719

Generic (Super Type)을 이용한 RestTemplate 응답 받기.



지금까지 RestTemplate을 사용할 때 각 엔드포인트마다 Response Dto를 만들어서 사용중이었다.
대신 restTemplate에서 헤더 설정은 공통이기에 마지막에 호출하는 메소드는 따로 빼서 사용하였다.

![스크린샷 2021-07-21 12.29.54](https://tva1.sinaimg.cn/large/008i3skNgy1gsoevbqphmj31140aa0v1.jpg)
코드로 설명하자면, ocrIdcard와 ocrBusiness는 분명히 다른 성격의 메소드다.
각각의 응답형식도 다르고, 보내는 URI도 다르고 데이터도 다를 것이다.
하지만 같은 회사의 API이기에 헤더 형식은 같기에 restTemplate에서 공통세팅과 전송하는 메소드를 따로 분리하였다.
보면은 ocrIdcard와 ocrBusiness에서 postFile(String uri, String token, T t, Class clazz)를 호출하는 것을 알 수 있다.
간략하게 설명하면 uri는 API의 엔드포인트, token은 API 액세스 토큰, Generic T는 파일 데이터, clazz는 Repsonse로 받을 클래스다.

이렇게 잘 사용하고 있었으나, 문득 생각해보니 
응답에는 공통으로 겹치는 헤더, 코드값, 메세지 등이 있어서, Generic을 이용해서 받으면 어떨까라는 생각이 들었다.

![스크린샷 2021-07-21 12.19.11](https://tva1.sinaimg.cn/large/008i3skNgy1gsoek847n1j308d0axdg7.jpg)
(이런식으로 Generic을 이용)



그래서 일단 코드를 다음과 같이 바꾸었다.![스크린샷 2021-07-21 17.28.09](https://user-images.githubusercontent.com/44764810/126593109-aea044c6-c42a-416e-b84b-c98c040dba13.png)
테스트 결과 오류가 난다.. 정상적으로 동작하지 않는다.

보다시피 Response를 받는게 아니라 LinkedHashMap으로 받고 있었다. 아니 왜...?

![image-20210721180442848](https://user-images.githubusercontent.com/44764810/126593443-2ea4dd97-854d-40be-95e7-ca5b5369fc6f.png)
이런저런 삽질을 하다가 Generic은 컴파일 시에 타입이 사라진다는 것을 알았다.
왜 그럴까 싶어서 구글링을 하다가.. [Type Token](https://sabarada.tistory.com/125) 이라는 것을 알게 되었다.

쉽게 말해서 자바는 런타임시에 Generic의 타입을 지워버린다. (Type Erasure)
Generic이 JDK 1.5부터 있기에 하위호환을 위해서 그렇다고 한다.
그래서 Super Type Token이라는 것을 사용해야하는데, Super Type Token이란 Neal Gafter가 만든 기법으로 추상 클래스에 대해서 이를 구현하는 익명 sub 클래스를 만드는 것 라고 한다.

내가 이해한 바에 따르면, Class<T>로 적용을하면 런타임시 T 타입을 알 수가 없어, Class<Object>가 되므로,
Jackson에서 응답 본문이 JSON이니까 Map으로 바꿨을 것이다.

그리고 [umbum님 블로그](https://umbum.dev/925)를 보고 RestTemplate를 사용할 땐 TypeReference 대신 ParameterizedTypeReference를 사용해야 하는 것을 알았다.
그래서 다음과 같이 코드를 변경하였다.

![스크린샷 2021-07-21 18.59.53](https://user-images.githubusercontent.com/44764810/126593211-36199ff3-4873-434e-af1f-fa59e1bc34b8.png)


*평소에 이렇게 Generic에 대해서 생각해보지 않았는데... Generic에 대해서는 다음에 다시 정리해야겠다.*
