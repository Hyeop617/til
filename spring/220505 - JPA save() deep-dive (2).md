# 220505 - JPA save() deep-dive (2)

insert일 땐 알아보았다. 그러면 수정일 땐 어떻게 될까?

먼저 @Version 애노테이션이 없을 때를 살펴보자.

![스크린샷 2022-05-05 오후 5.09.06](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xlena8xoj20fx07edg6.jpg)</center>

SimpleJpaRepository의 save()에서 isNew()를 거쳐서,

![스크린샷 2022-05-05 오후 5.18.44](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xlor8ly3j20ux081wfj.jpg)</center>

마찬가지로 JpaMetamodelEntityInformation에서 isNew()를 거친다.
상위 클래스(AbstractEntityInformation)의 isNew()를 호출하는 것을 확인해보자.

![스크린샷 2022-05-05 오후 5.23.35](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xltn62z3j20lk0b4ab0.jpg)</center>

Id는 현재 Long.class고 값이 존재하므로, id == null 은 false가 된다.
따라서 entityInformation.isNew()는 false가 될 것이다.



그렇다면 @Version이 있을 때는 어떨까?

![스크린샷 2022-05-05 오후 5.27.15](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xlxfgf4zj20ps09e75g.jpg)</center>

일단 JpaMetamodelEntityInformation에서 versionAttribute.map(it -> wrapper.getProperty(it.getName())) == null을 체크하는 부분까진 insert와 같다.

그리고 DirectFieldAccessFallbackBeanWrapper의 getPropertyValue()로 들어가게 되는 부분도 같다.

![스크린샷 2022-05-05 오후 5.28.43](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xlyyrchzj20im0cimyg.jpg)</center>



그러면 AbstractNestablePropertyAccessor의 getPropertyValue()로 들어가는 부분도 insert와 같은데...

![스크린샷 2022-05-05 오후 5.29.06](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xlzcwjgoj20pf05pq3v.jpg)</center>

해당 클래스는 아직 지식이 부족해 봐도 잘 모르므로...
일단 Step into를 클릭하다 보면 insert와는 다르게 version의 value를 가져온다.

![스크린샷 2022-05-05 오후 5.32.50](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xm38pexmj20mb06xjsn.jpg)</center>

Object value = ph.getValue();에서 value가 0인 것을 확인할 수 있다.

다시 JpaMetamodelEntityInformation으로 빠져나가자.

![스크린샷 2022-05-05 오후 5.33.46](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xm48b9zyj20nu09gwfn.jpg)</center>

그러면 251번 라인에서 map의 wrapper.getPropertyValue()는 version의 value인 0이다.
0 == null 하면 false니까 false를 리턴해준다.



JPA에서 save()에 대해서 알아보았다.



그냥 @Id만 있을 땐?

- @Id의 값을 EntityInformation을 이용해서 가져온다.

- @Id값이 null인지 확인한다.

@Version 어노테이션이 있을 땐?

- @Version 어노테이션의 값을 EntityInformation을 이용해서 가져온다.

- @Version의 값이 null인지 확인한다.