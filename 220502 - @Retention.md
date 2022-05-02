## 220502 - @Retention

[정아마추어님 블로그](https://jeong-pro.tistory.com/234) 를 참고하였습니다.



@Retention은 **어노테이션의 라이프사이클**, 즉 **언제까지 어노테이션이 살아있는지를 정하는 것** 이라고한다.

RetentionPolicy에는 3가지가 있다. (기본값은 RetentionPolicy.CLASS)

- RetentionPolicy.SOURCE
- RetentionPolicy.CLASS
- RetentionPolicy.RUNTIME

RetentionPolicy.SOURCE는 소스 코드(.java)까지 남아있는 것을 뜻한다.
다음 예시를 보자. Lombok의 RequiredArgsConstructor다.

![스크린샷 2022-05-02 오전 11.54.46](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tvgk6msjj209m02aglm.jpg)

@RequiredArgsConstructor는 final 필드의 생성자를 만들어주는 어노테이션이다.
위 어노테이션은 생성자를 만든 뒤에 사라질 것이다. 즉, 해당 어노테이션은 build 한 뒤에는 찾을 수 없을 것이다.
바이트 코드를 만드는 어노테이션이므로, 바이트 코드에는 남을 필요가 없다. 그래서 RetentionPolicy.SOURCE로 설정한 것이다.



다음은 RetentionPolicy.RUNTIME 이다.

![스크린샷 2022-05-02 오후 12.00.43](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tvmqhukyj20qc02z3yt.jpg)

스프링의 Autowired 어노테이션이다.
스프링이 처음 실행될 때, @Autowired가 붙은 Bean들을 스프링 빈으로 등록해야 한다.
따라서 해당 어노테이션은 Runtime 시점까지 남아 있어야, Bean으로 등록이 가능할 것이다.



다음은 RetentionPolicy.CLASS다.
소스코드에는 남아 있지 않고, 빌드 후 클래스파일까지 유지하는 것이다.

![스크린샷 2022-05-02 오후 12.05.29](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tvroxsggj20po03bdg7.jpg)

Lombok의 NonNull 어노테이션이다.
해당 값은 null이 될 수 없다고 하는 어노테이션인데... 그러면 RetentionPolicy.SOURCE로도 가능한 것이 아닌가?? 싶은 의문이 든다.
정아마추어님 블로그를 보니 다음 댓글이 있었다.
![스크린샷 2022-05-02 오후 12.07.28](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tvtqw00ij20ne079jsp.jpg)

라이브러리에는 소스파일이 없다. 당연히 class 파일밖에 없을 것이다.
이럴 때 RetentionPolicy.SOURCE라면 해당 라이브러리에는 NonNull 관련이 바이트코드로는 남아 있겠지만 어노테이션은 사라져있을 것이다.
그렇게 된다면 IDE 등의 도움을 받을 수 없을 것이다. 왜? 해당 어노테이션 정보가 없으니까.

확실히 해당 어노테이션을 라이브러리안에 포함시켜 재사용하려면 RetentionPolicy.CLASS가 더 개발에 유용할 것 같긴하다.