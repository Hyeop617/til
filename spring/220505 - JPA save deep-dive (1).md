# 220505 - JPA save() deep-dive (1)

JPA를 사용할 때 그런가보다 했던 부분을 더 깊게 파보자.

먼저 save()에 대해서 보자.

![스크린샷 2022-05-03 오후 8.08.44](https://tva1.sinaimg.cn/large/e6c9d24egy1h1vfctgm3aj20g107ddg4.jpg)</center>



SimpleJpaRepository의 save() 메소드다.

저번에 정리를 해서 entityInformation.isNew()로 판별하는 것은 알고 있었다.
그런데 EntityInformation 이 친구는 뭐하는 친구일까?

Save()에 breakpoint를 걸어보고 실행시켜보았다.

![스크린샷 2022-05-03 오후 6.46.43](https://tva1.sinaimg.cn/large/e6c9d24egy1h1vczoctzsj20t80jt766.jpg)</center>



entityInformation에 JpaMetamodelEntityInformation 이란 객체가 들어왔다. <br>
이 친구는 또 EntityInformation과 어떤 관계일까?



![스크린샷 2022-05-03 오후 6.48.26](https://tva1.sinaimg.cn/large/e6c9d24egy1h1vd1bck8kj20k60btwfm.jpg)</center>

복잡한 인간관계를 가진 친구였다...

하나하나 살펴보자.



### EntityMetadata

EntityMetadata는 Entity의 정보를 갖고 있는 인터페이스다.

![스크린샷 2022-05-03 오후 7.32.19](https://tva1.sinaimg.cn/large/e6c9d24egy1h1veayd8raj208w05zmx7.jpg)</center>



getJavaType() 메소드가 있고 도메인의 클래스 타입을 리턴해준다고 한다.



### JpaEntityMetadata

JpaEntityMetadata는 EntityMetadata를 상속받는 인터페이스다.

![스크린샷 2022-05-03 오후 8.15.37](https://tva1.sinaimg.cn/large/e6c9d24egy1h1vfjzeh8rj20ev05e0sv.jpg)</center>



getEntityName()은 엔티티의 이름을 리턴한다고 한다.



### EntityInformation

![스크린샷 2022-05-03 오후 8.20.15](https://tva1.sinaimg.cn/large/e6c9d24egy1h1vfoudpljj20pk0mcgng.jpg)</center>

엔티티가 새로운 엔티티인지 (isNew), 엔티티의 Id를 가져오고(getId), 엔티티의 Id의 Class를 가져오는(getIdType) 메소드가 있다.
save()에 쓰이는 isNew가 여기에 있는 것을 알 수 있다.

### AbstractEntityInformation

![스크린샷 2022-05-03 오후 8.44.09](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xi5p1azhj20mh0ho75o.jpg)</center>

isNew()가 구현되어 있다.
먼저 EntityInformation의 getId(entity)를 호출한다.

그리고 IdType이 Wrapper Class 일땐 getId(entity)한 값과 null 체크를 한다.
Primitive일 땐 0인지 체크한다.



### JpaMetamodelEntityInformation

![스크린샷 2022-05-03 오후 8.59.25](https://tva1.sinaimg.cn/large/e6c9d24egy1h1vgtl3t0rj20n306zt9g.jpg)</center>

일단 versionAttribute는 Optional로 감싸져있는 것을 확인했다.

versionAttribute가 존재하지 않거나 versionAttribute가 Primitive 타입이라면 부모 클래스의 isNew()를 호출하는 것 같다.
디버그 해보니 역시 AbstractEntityInformation()의 isNew()를 호출하였다.



그렇다면 versionAttribute 얘는 뭘까??
이름에서 알 수 있듯이 @Version 과 관계된 속성인 것 같다.

![스크린샷 2022-05-05 오후 3.10.51](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xhzlnq5qj20fm0fc3zl.jpg)</center>



보다시피 현재 Order 엔티티에는 @Version 어노테이션이 없다.
@Version이 있는 엔티티로 시도해보자.

![스크린샷 2022-05-05 오후 3.11.56](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xi0mm5lwj20br08ymxi.jpg)</center>



![스크린샷 2022-05-05 오후 3.12.40](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xi1gfv9dj20pu0jx40l.jpg)</center>



SingularAttributeImpl 이라는 친구가 들어왔다.
이 친구도 한 번 살펴보자.

![SingularAttributeImpl](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xidgkps5j20oy10p78j.jpg)</center>

이 친구도 인간관계가 복잡하지만, 천천히 보다보면 엔티티안에서 해당하는 데이터의 속성을 나타내주는 클래스 인 것 같다.


![스크린샷 2022-05-05 오후 3.26.04](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xife7fshj20o10723z2.jpg)</center>

일단 해당 데이터가 PK인지, Version 속성인지, Optional인지, 컬럼의 이름은 무엇인지, attributeNature는 무엇인지 등을 갖고 있다.

attributeNature는 뭘까 그럼?



![스크린샷 2022-05-05 오후 3.28.10](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xihku15xj20bn0gzgm3.jpg)</center>

해당 데이터의 타입을 나타내는 이넘 클래스다.



대충 이해가 된 것 같으니 다시 JpaMetamodelEntityInformation의 isNew로 가보자.



![스크린샷 2022-05-05 오후 3.32.37](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xim6l67uj20o00emmyz.jpg)</center>

@Version 어노테이션을 Long.class로 지정했기 때문에 primitive 타입도 아니다.
따라서 entity로 BeanWrapper라는 인스턴스를 생성하였다.
그리고 wrapper.getPropertyValue()에 versionAttribute.getName()한 값이 null인지를 반환하는 것으로 마무리 짓는 코드다.
일단, versionAttribute.getName()은 versionAttribute의 name을 리턴할테니 해당 값은 여기서 "version"이 된다.
그리고 디버그를 해보면 BeanWrapper의 구현체는 DirectFieldAccessFallbackBeanWrapper인 것을 알 수 있다.

그리고 AbstractNestablePropertyAcccessor에서 해당 property의 Value를 가져오는데 그 값을 null로 가져온다.
따라서 isNew()는 true가 된다.

