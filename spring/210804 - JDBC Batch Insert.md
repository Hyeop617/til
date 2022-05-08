## JDBC BATCH INSERT

다량의 데이터를 (2000개) insert 혹은 update 칠 일이 생겼다.<br>
그냥 for loop으로 JPA save를 날리니.. 데이터 길이가 긴 로우들이였어서 그런지 디비가 터져버렸다.

그럼 saveAll()을 날리면 되지..라고 예전엔 생각했겠으나<br>
saveAll()은 bulk insert를 날려주지 않는다는 것을 알고 있었다.<br>
이름은 뭔가 한번에 insert 쿼리를 모아서 날려줄 것 같지만, 실제로는 하나씩 insert를 넣는다<br>
(하이버네이트에서 show_sql을 해보면 알 수 있다)

```java
List<User> userList;
userRepository.saveAll(list);						// 1

for(User user : userList){
  userRepository.save(user);						// 2
}
```

그렇다면 1과 2는 똑같을까?<br>
쿼리상으로 보면 둘 다 똑같이 insert 쿼리를 한 번씩 userList.size()만큼 날린다.<br>
하지만 SimpleJpaRepository의 소스를 보자.

```java
	@Transactional
	@Override
	public <S extends T> List<S> saveAll(Iterable<S> entities) {

		Assert.notNull(entities, "Entities must not be null!");

		List<S> result = new ArrayList<S>();

		for (S entity : entities) {
			result.add(save(entity));
		}

		return result;
	}
```

~~~java
	@Transactional
	@Override
	public <S extends T> S save(S entity) {

		Assert.notNull(entity, "Entity must not be null.");

		if (entityInformation.isNew(entity)) {
			em.persist(entity);
			return entity;
		} else {
			return em.merge(entity);
		}
	}
~~~

save()와 saveAll()모두 @Transactional 어노테이션이 있다.<br>
그렇다면 2번 처럼 for-loop에서 save()를 하면?<br>
최악의 경우에 트랜잭션을 생성 - 삭제를 for-loop만큼 반복한다.

@Transactional 선언된 메소드 안에서 for-loop가 돌아간다고 가정해보자.<br>
그렇다면 save()는 Transactional인 메소드 안에서 돌아가기 때문에, 트랜잭션 1개를 쭉 유지할 것이다.<br>
하지만 그렇지 않다면? save()를 호출할 때마다 트랜잭션이 생성되고 종료되고를 반복할 것이다.<br>
 따라서 엄청나게 성능상의 이슈가 생길 수 있다..

그러면 @Transactional이 선언된 메소드에서 실행하면 되냐? (한 트랜잭션으로 유지)<br>
하지만 롱 트랜잭션을 가져가기 부담스러울 수 있다. 롱 트랜잭션 사이에 조회가 일어나면..?

그렇다면 방법은 없나? 있다.<br>
대신 JPA(하이버네이트)에서 bulk insert를 하려면 몇 가지 조건이 있다.

먼저 Entity의 ID가 GeneratedValue.IDENTITY면 안된다.<br>
ID의 순서를 GeneratedValue.TABLE, GeneratedValue.SEQUENCE 등 JPA가 예측할 수 있어야 한다!

결국 JPA로 해결할 방법이 없었기에 그냥 JdbcTemplate을 이용해서 하기로 했다.<br>
JPA를 쓰고 싶었는데.. 하지만 속도가 중요한 서비스에서는 이렇게 JdbcTemplate을 쓰기도 하니까 뭐 어쩔 수 없다.

```java
public int[] batchInsertAccountTransaction(List<AccountTransaction> accountTransactionList){
        String query = "insert into account_transaction (account_id, amount, transaction_type, balance, transaction_date, note, created_at, updated_at) values (?, ?, ?, ?, ?, ?, ?, ?)";
        return jdbcTemplate.batchUpdate(query, new BatchPreparedStatementSetter() {
            @Override
            public void setValues(PreparedStatement ps, int i) throws SQLException {
                AccountTransaction accountTransaction = accountTransactionList.get(i);
                ps.setLong(1, accountTransaction.getAccount().getAccountId());
                ps.setLong(2, accountTransaction.getAmount());
                ps.setString(3, accountTransaction.getTransactionType().name());
                ps.setLong(4, accountTransaction.getBalance());
                ps.setTimestamp(5, Timestamp.valueOf(DateUtil.nowDateTime()));
                ps.setString(6, accountTransaction.getNote());
                ps.setTimestamp(7, Timestamp.valueOf(DateUtil.nowDateTime()));
                ps.setTimestamp(8, Timestamp.valueOf(DateUtil.nowDateTime()));
            }

            @Override
            public int getBatchSize() {
                return accountTransactionList.size();
            }
        });
    }
```

옛날 기억이 새록새록나는 JdbcTemplate..<br>
저런 식으로 쿼리를 작성한다음 구멍을 뚫어서 값을 채우면 된다.<br>
소스만 보면 자바 기초니까 자세한 설명은 필요 없을 것 같다.

참고

출처: https://prosaist0131.tistory.com/entry/insert와-bulk-insert-무엇을-써야할까요 [내일은 개발을 잘 할 거야]<br>
출처: https://2dongdong.tistory.com/29 [즐겁게 살고 싶은 개발자]
