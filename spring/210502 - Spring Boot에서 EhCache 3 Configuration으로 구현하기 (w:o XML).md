## Spring Boot에서 EhCache 3 Configuration으로 구현하기 (w/o XML)



https://hyhp.tistory.com/8

저번 글에서는 EhCache 3을 XML 대신 Java Class로 설정하였다.

JCache(자바 표준 캐시 API)를 구현해서 EhCache 3 설정을 했지만 HeapSize를 설정할 수 없는 큰 단점이 있었다.

그래서 이번엔 EhCache 3의 설정 파일로 구현하는 방법을 소개하겠다.



### CacheConfig

```java
@Configuration
public class CacheConfig {

    @Bean
    public JCacheCacheManager jCacheCacheManger(){
        return new JCacheCacheManager(cacheManager());
    }

    @Bean("cacheManager")
    public CacheManager cacheManager() {
        CachingProvider provider = Caching.getCachingProvider();
        CacheManager cacheManager = provider.getCacheManager();
        List<String> cacheNameList = Arrays.asList("accessTokenAAAA", "accessTokenBBBB");
        cacheNameList.forEach(c -> cacheManager.createCache(c, config()));
        return cacheManager;
    }


    private javax.cache.configuration.Configuration<Object, Object> config() {
        DefaultCacheEventListenerConfiguration listenerConfiguration = CacheEventListenerConfigurationBuilder.newEventListenerConfiguration(new CustomEhCacheEventListener(), EventType.CREATED, EventType.UPDATED).build();    // 1️⃣

        ResourcePoolsBuilder heap = ResourcePoolsBuilder.newResourcePoolsBuilder().heap(1000, MemoryUnit.KB); // 2️⃣
        CacheConfiguration<Object, Object> cacheConfiguration = CacheConfigurationBuilder.newCacheConfigurationBuilder(Object.class, Object.class, heap)
                    .withSizeOfMaxObjectSize(1000, MemoryUnit.B)		// 3️⃣
                    .withService(listenerConfiguration)					// 4️⃣
                    .withExpiry(ExpiryPolicyBuilder.timeToLiveExpiration(java.time.Duration.parse("P7D")))            // 7일동안 유효
                    .build();

        javax.cache.configuration.Configuration<Object, Object> simpleKeyStringConfiguration =   Eh107Configuration.fromEhcacheCacheConfiguration(cacheConfiguration);					// 5️⃣
        return simpleKeyStringConfiguration;
```





config()를 제외한 부분은 저번 글과 같다.

이번엔 config()에서 EhCache의 Config 클래스를 만들어서, JCache Config 형태로 변환할 것이다.



1️⃣ 에서 이벤트 리스너 설정 클래스를 listenerConfiguration로 선언한다.

listenerConfiguration에는 뒤에서 구현할 커스텀 이벤트 리스너와, 어떤 이벤트가 일어날때 리스너가 작동할지를 받는다.

여기선 캐시가 생성될 때(EventType.CREATED)와, 갱신될 때(EventType.UPDATED)로 설정하였다.

사라진 캐시 (새로운 캐시가 들어와 가장 오래되어서 삭제당한 캐시)를 보고 싶다면 EventType.EVICTED를 추가하면 된다.

2️⃣ 에서는 캐시의 최대 사이즈를 지정해주었다. heap과 off-heap 그리고 disk 총 세 공간에 캐시를 저장할 수 있다. (속도는 heap > off-heap > disk)

간단하게 설명하자면 off-heap은 JVM에 의해 관리되지 않지만, EhCache에 의해 관리되고 있다. heap과 마찬가지로 메모리 안에 있으므로 disk 보다 빠르게 접근 가능하다.

heap은 캐시의 갯수(entry)와 캐시 공간(size) 두 종류로 설정 할 수 있다.

```java
ResourcePoolsBuilder.newResourcePoolsBuilder()
  .heap(100, MemoryUnit.KB)
```

or

```java
ResourcePoolsBuilder.newResourcePoolsBuilder()
  .heap(100, EntryUnit.ENTRIES)				// EntryUnit.ENTRIES는 생략 가능
```

off-heap을 설정하려면 자바 옵션에 -XX:MaxDirectMemorySize=<size>를 붙여야한다.

따라서 다음과 같이 사용 가능하다.

```java
ResourcePoolsBuilder.newResourcePoolsBuilder()
  .heap(100, EntryUnit.ENTRIES)	
  .offheap(500, MemoryUnit.KB)
  .disk(1000, MemoryUnit.MB)
```

3️⃣ 에서는 캐시에 저장된 각 키의 최대 값을 지정하였다.

주의할 점은 heap이 size로 설정되어 있을 때에만 동작한다. (Entry의 개수일때는 무시된다)

설정한 사이즈보다 큰 값을 저장하면 다음과 같은 경고를 띄워주면서 캐시에 저장이 안된다.

````java
2021-04-18 07:34:10.201  WARN 29347 --- [  restartedMain] o.e.i.internal.store.heap.OnHeapStore    : Max Object Size reached for the object : eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQTMwMzY0OSwiaXNzIjoiaHR0cHM6XC9cLzF3b24udXNlYi5jby5rciIsImRhdGEiOnsiaWQiOiIyNSIsImZpcnN0bmFtZSI6IkRvbmdqb28i
````

4️⃣ 에서는 이벤트 리스너를 설정파일에 등록 하였다.

5️⃣ 에서는 org.ehcache.config의 cacheConfiguration을 JSR-107(JCache) 설정파일로 변환해주었다.



### CustomEhCacheEventListener

```java
@Slf4j
public class CustomEhCacheEventListener implements CacheEventListener {

    @Override
    public void onEvent(CacheEvent event) {
        log.info("============================================================");
        log.info("EHCACHE :: EVENT TYPE -> {} ", event.getType());
        log.info("EHCACHE :: KEY -> {} ", event.getKey());
        log.info("EHCACHE :: VALUE -> {} ", event.getNewValue());
        log.info("============================================================");
    }
}
```

캐시 이벤트를 받아서 로그를 찍게 해주었다.

개인적으로는 JCache보다 더 편하게 로그를 찍을 수 있는 것 같다.



다행히 설정과 이벤트 리스너만 교체하면 쉽게 캐시를 사용할 수 있었다.