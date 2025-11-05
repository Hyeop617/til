## Symbol

ES6에서 추가된 primitive type.<br>
객체에 속성을 추가할 때 고유한 키를 부여하는데 주로 사용.

객체의 Symbol은 접근이 어렵기 때문에, 외부에서 접근할 수 없도록 보호하는 용도로도 사용됨. (캡슐화, 은닉화)

```
const sym1 = Symbol('description');
const sym2 = Symbol('description');
console.log(sym1 === sym2); // false
```

`Symbol.for()`와 `Symbol.keyFor()`를 사용해 애플리케이션 전역에서 공유되는 심볼을 생성 가능.<br>
따라서 위 2개 함수를 사용하면 동일한 키로 생성된 심볼은 동일한 값을 가짐.
```
const sym1 = Symbol.for('hello');
const sym2 = Symbol.for('hello');
console.log(sym1 === sym2); // true
```
또한 해당 심볼은 전역적으로 사용되므로 가비지 컬렉션 대상이 되지 않음.

## Iterable, Iterator

Iterable은 `Symbol.iterator` 메소드를 구현한 객체를 의미.<br>

Iterator는 `next()` 메소드를 구현한 객체를 의미.<br>`next()` 메소드는 `{value: any, done: boolean}` 형태의 객체를 반환함.

Iterable 객체는 for..of 문, 스프레드 연산자, 배열 디스트럭처링 할당 등에서 사용될 수 있음.
그리고 Iterable 객체의 `Symbol.iterator` 메소드는 Iterator 객체를 반환함.

주요 Iterable 객체로는 Array, String, Map, Set 등이 있음.

### 유사 객체 (Array-like)

유사 객체는 인덱스와 length 속성을 가지고 있는 객체.<br>
Iterable 객체가 아니여서 for..of 문을 사용할 수는 없음.

그러나 한 객체가 유사 객체이면서 Iterable 객체인 경우도 있음. (예: Array, String)<br>
Map과 Set은 유사 객체가 아님.

## WeakMap, WeakSet

WeakMap과 WeakSet은 일반 Map과 Set과 유사하지만, 다음과 같은 차이점이 있음.

- Key는 반드시 Garbage-collectable한 타입이여야함.
  - Primitive type은 key로 사용할 수 없음. (Symbol 제외)
- keys(), values(), entries() 메서드가 없음.
  - key와 value가 참조를 잃을 때 GC 대상이 되므로, 정확히 key와 value가 무엇인지 알 수 없음.
- for...of 루프를 사용할 수 없음.
  - Iterable 객체가 아님.

일반적인 Map과 Set은 그 안의 element들은 가비지 컬렉션 대상이 되지 않음.<br>
```js
// map 객체가 사라지지 않는한 (undefined, null) key와 value 중 객체는 GC 대상이 되지 않음.
let phoneByUser = new Map();
let user = { "key" : "hi"}
const phone = { "number" : "010-1234-5678"}
phoneByUser.set(user, phone);

// user를 null로 설정해도 map 객체가 사라지지 않으면 GC 대상이 되지 않음.
user = null;

// map 객체 안에는 여전히 user 객체가 존재함.
console.log(map.keys().toArray()[0])

// map 객체의 참조를 없애면 user는 GC 대상이 됨.
phoneByUser = null;
```

```js
let phoneByUser = new WeakMap();
let user = { "key" : "hi"}
const phone = { "number" : "010-1234-5678"}
phoneByUser.set(user, phone);

// user를 null로 설정하면 이제 user는 GC 대상이 됨. (phone은 GC 대상이 아님)
user = null;

let user2 = { "key" : "hello" }
phoneByUser.set(user2, { "number" : "010-9876-5432" });

// user2를 null로 설정하면 user2와 010-9876-5432 객체 모두 GC 대상이 됨.
user2 = null;
```

WeakMap은 추가 데이터를 저장하거나, 캐싱을 할 때 유용하게 사용될 수 있음.

```js
let visitsCountMap = new WeakMap(); // WeakMap에 사용자의 방문 횟수를 저장함

// 사용자가 방문하면 방문 횟수를 늘려줍니다.
function countUser(user) {
  let count = visitsCountMap.get(user) || 0;
  visitsCountMap.set(user, count + 1);
}
```
user 객체가 더 이상 참조되지 않으면, user의 count 데이터도 GC 대상이 됨.

하지만 count의 type은 primitive type이고, number 이므로 Stack에 저장이 된다고 생각할 수 있음.<br>
Stack은 Garbage Collector가 접근할 수 없음.<br>

하지만 