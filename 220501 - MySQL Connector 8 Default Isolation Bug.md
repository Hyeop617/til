## 210501 - MySQL Connector 8 Default Isolation Level Bug Fix!!!

어이가 없던? 이슈에 대해 말해보고자 한다.
많은 RDB들이 트랜잭션 고립 수준을 기본으로 Read Commited로 설정한다고 알고 있다.
그렇지만 MySQL만 특이하게 Repeatable Read로 설정한다고 한다.
(과거에 binlog 저장방식때문에 그렇다고 했던 것 같다.)

트랜잭션에 관해서 정리하다가 문득 궁금해졌다.

> 그러면 Spring은 MySQL일 때에 Repeatable Read로 설정한다는 얘긴데..
> 어디서 어떻게 설정하는 것일까??

타고타고 들어가다보니 MySQL Connector (com.mysql.cj.jdbc.DatabaseMetaData) 에서 설정되어 있는 것을 알 수 있었다.
그런데...
![스크린샷 2022-05-01 오전 2.54.36](https://tva1.sinaimg.cn/large/e6c9d24egy1h1sa8836lnj20wm0dfmz5.jpg)

분명 MySQL인데 기본 Isolation level이 READ_COMMITED 로 되어 있었다.

지금까지 잘못 알고 있었나 싶어서 찾아보니 그것도 아니고...
나와 같은 고민을 역시 다른 사람도 했었다.
https://stackoverflow.com/questions/58472144/why-mysql-jdbc-driver-returns-transaction-read-committed-as-default-isolation-le

답변에 따르자면 JDBC Driver는 Read Commited로 작동하지만 MySQL 에서는 기본으로 Repeatable Read로 작동이 된다는 것 같았다.
그리고 왜 그렇게 설정되었는지 알 수 없다는 말과 함께..

찝찝한 기운을 지울 순 없었다..
MySQL 도큐멘테이션에는 뭔가 자세한 이유가 있으리라라는 생각으로 구글링을 하다가 다음을 발견했다.
https://dev.mysql.com/doc/relnotes/connector-j/8.0/en/news-8-0-29.html

![스크린샷 2022-05-01 오전 2.58.59](https://tva1.sinaimg.cn/large/e6c9d24egy1h1sacsfp7gj20wp0lgwi9.jpg)

어이 없게도 버그였다고 한다.
그래서 글을 쓰는 현재 시점 기준으로 5일만 버그 패치가 늦었어도 영영 모른채 지나갈 뻔 했다.

위의 스택오버플로우 글을 보면 2년 전부터 그랬던 것 같은데.. 설마 MySQL 8 초기버전부터 그랬나 싶은 생각도 든다.
뭐 스택오버플로우 말대로라면 MySQL 엔진 내에서는 Repeatable Read로 작동을 할테니 심각하지 않은 버그인가 싶기도 하고.
서로 Isolation Level이 다른데 어떻게 동작하는지도 궁금하지만.. 너무 deep-dive 싶기도 하니 여기까지만 알아봐야겠다.