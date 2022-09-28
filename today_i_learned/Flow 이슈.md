## 1. lifecycleScope.launch 와 lifecycleScope.launchWhenXXX의 차이점
## 2. upstream과 downstream의 차이
```kotlin
// emit된 value가 upstream : which flow I will operate
flow {
  emit(0.1f)
  emit(0.9f)
}
// map을 거친 result value가 downstream : which flow I will produce
 .map { floatNumber -> floatNumber.toInt() }
 .filter { number -> number > 0 }
 .collect { /**/ }
 ```
## 3. `map`, `filter`등의 중간 연산을 `flow collector`라 칭한다.

## 4. Channel
Sender와 Receiver 사이를 연결이 목적. 이는 closed될 수 있으며, channel과 연관된 모든 operation은 중단될 수 있다(suspendable).
Flow와 가장 큰 차이는 누가 이를 observe하는지 여부와 관계없이 값을 분출(start to emit)할 수 있다는 것이다.
또한 channel에 접근하기만 하면, channel의 참조값이 있는 어느 곳에서든 값을 emit하거나 receive할 수 있다.
반면 Flow는 flow 블록 안에서만 값을 분출할 수 있다.
이러한 특성으로 미루어 볼 때 channel이 hot stream이다.

  ### 4.1. SendChannel
  ```kotlin
  interface SendChannel<in E> {
    val isClosedForSend: Boolean // Channel이 closed 상태일 경우 더이상 data를 전송할 수 없다는 것을 알려준다. close 상태일 때 값을 전송하려고 할 경우 ClosedSendChannelException을 받는다.

    suspend fun send(element: E) // 비동기 emit values
    fun offer(element: E): Boolean // 동기 emit values

    fun close(cause: Throwable? = null): Boolean // channel의 sender를 닫기 위해 사용한다.
    fun invokeOnClose(handler: (cause: Throwable?) -> Unit) // sender혹은 receiver가 닫힐 경우 자동으로 실행된다(invoke).
  }
  ```
  
  #### 4.1.2. Sending values - `offer()`를 쓸 것인가, 아니면 `send()`를 쓸 것인가?
  버퍼에 데이터를 집어 넣을 때 왜 두 개의 메소드 (`offer()` 혹은 `send()`)가 사용되는가?
  
  - (suspend X) `offer()`은 버퍼에 element를 가능한 즉시 추가할 때 사용한다. 버퍼가 존재하면서 꽉 차지 않았을 때(버퍼의 용적을 초과하지 않는 선에서 가능한 한 즉시). 또한 `offer()`는 boolean을 반환하는데 element가 추가되었으면 true, 아니면 false를 리턴한다. 만약 버퍼가 존재하지 않거나 꽉 찼다면 바로 false를 반환하고, 값은 버퍼에 unreachable된다. 
  - (suspend O) `send()`는 버퍼가 존재하면서 꽉 차지 않았을 때는 `offer()`와 동일하게 작동한다. 그러나 버퍼가 가득 차거나 존재하지 않을 때 `send()`를 사용하면 `send()`호출자가 버퍼가 생성되거나 버퍼 공간에 여유가 생길 때까지 중단된다(send() caller gets suspended). 즉 둘 사이의 가장 큰 차이는 suspendable의 여부이다. 당장 동기화를 시킬 수 없을 경우 중단되는 것이다.
  
  #### 4.1.3. 정리

  - 값을 즉시 전송하고자 할 경우 `offer()`를 사용한다. 단 버퍼에 도착하지 못할 수 있다(바로 false를 리턴).
  - 값이 **특정 대상**에게 전송되는 것을 보장하려면, `send()`를 사용한다. **특정 대상**이 나타날 때까지 기다릴 수 있다.

  ### 4.2. ReceiveChannel
  ```kotlin
  interface ReceiveChannel<out E> {
    
    val isClosedForReceive: Boolean
    
    suspend fun receive(): E
    fun poll(): E?
    
    fun cancel(cause: CancellationException? = null)
  }
  ```  
  
  #### 4.2.1. `receive()`와 `poll()`의 차이 (<=> `send()`와 `offer()`의 차이와 동일하다.)
  
  - (suspend X) `poll()`은 값을 동기적으로 받는다. 만약 버퍼 안에 무엇인가 들어 있다면, 이를 반환한다. 버퍼 안이 비어있을 경우는 null을 반환한다.
  - (suspend O) `receive()`는 값을 비동기적으로 받는다. 버퍼 안에 무엇인가 들어있을 경우 이를 반환한다. 그러나 버퍼가 존재하지 않거나 비어있을 경우 `receive()` 호출자는 새로운 값이 버퍼에 올라오기까지 중단(suspended)된다.
  
  #### 4.2.2. 정리
  
  - 값 반환이 보장되지 않더라도 (null을 반환하더라도) 당장 값을 리턴받고 싶을 경우 `poll()`을 사용한다.
  - 보장된 값을 받고 싶을 경우 `receive()`를 사용한다.

## 5. LiveData -> SharedFlow | StateFlow로 갈아타야 하는가에 관한 이슈
flow는 항상 collection을 통해 materialize되기 때문에, 만약에 다수의 collector가 존재할 경우, 새로운 flow가 각각의 collector로 인해 materialize되는 문제가 생긴다. 각 flow는 서로 다른 flow로부터 완전히 독립적이다. 만약 DB나 네트워크 호출이라면 이는 매우 비효율적. 한 개의 listener가 각 collector에 add된다면, 이는 CPU 사이클과 메모리 관점에서 hugely ineffective.
- `asLiveData()`를 쓴다면? 만약 ViewModel에서 Repository에 정의된 Flow를 LiveData로 변환한다면 LiveData는 Flow의 데이터를 수집하는 오로지 한 개의 collector가 된다. 표현 계층에 얼마나 많은 observer가 있는가와 관계없이 단 한 개의 flow만 collect된다. 그러나 이 아키텍쳐가 잘 동작하려면 다른 컴포넌트들 또한 ViewModel의 LiveData를 통해 데이터를 받도록 설계되어야 한다. 절대 Repository로부터 직접적으로 flow를 collect해서는 안된다. 그만큼 앱 로직이 각 컴포넌트별로 decoupling이 잘 되어 있어야 한다. (Repository를 필요로 하는 모든 컴포넌트가, 이젠 ViewModel에 접근하기 위해 Activity instance에 의존성을 갖게 된다. 이로써 컴포넌트의 범위(라이프사이클)가 제한될 수 있다.)
- 이 방식 말고는 없나? Listener 한 개로 무수히 많은 collector가 Flow의 data를 수집하게 하려면? “SharedFlow”를 이용한다. SharedFlow는 다수의 collectors가 하나의 Flow를 수집할 수 있도록 한다. 하나의 flow가 all of the simultaneous collectors에 의해 materialize되기에 효율적이다. StateFlow또한 같은 결과를 얻기 위해 사용된다. 그러나 SharedFlow에 `.value`(현재 상태) 기능을 덧붙인 것.
Flow를 SharedFlow로 바꾸려면?

```kotlin
fun <T> Flow<T>.shareIn(
	scope: CoroutineScope,
	started: SharingStarted,
	replay: Int = 0
): SharedFlow<T> (source)
```

- 여기서 `scope`는 Flow가 collector에 의해 materialize되는 일련의 연산이 수행되는 곳이다.
만약 Data Source가 @Singleton의 범위를 갖고 있다면, 이는 application의 scope를 따르는 것이다. LifecycleCoroutineScope가 프로세스 생성 시 생성되며, 프로세스 파괴 시 파괴된다.
- `started`에는 `SharingStarted.WhileSubscribed()`를 넘겨줄 수 있는데, 이는 Flow가 materialize되기 시작할 때 구독자를 0 -> 1로 올리고, stop되었을 때 1 -> 0로 변경한다. LiveData의 `onActive()`, `onInActive()`와 유사한 방식으로 작동한다. 이 외에도 설정 방식이 여러가지가 있는데, eagerly start(즉시 materialized, 절대 dematerialize 안되는 방식), lazy(처음 collect될 때 materialized, 이후 never dematerialized)가 그것이다.

> 막간을 이용한 용어 정리<br>
Observer: LiveData and collector for cold flows<br>
Subscriber: SharedFlow

- `replay`의 용도는 무엇? 1을 넘길 경우 새로운 subscriber가 가장 최근에 emit된 value를 즉각적으로 받는 것을 의미함. 
SharedFlow는 flow collector 그 자체로 이해하면 쉽다. (Flow collector는 cold flow upstream을 hot flow로 materialize하며, 수집된 값을 다수의 collector에 downstream으로 공유하는 역할을 함)
SharedFlow를 사용해 구독하면 Activity에 추가 로직을 반영하지 않아도 된다고 생각할 수도 있는데, 이는 착각!! coroutine을 `launchWhenStarted`로 실행하여 flow를 구독하면 `onStop`에 pause되고, `onStart`에 resume된다. 그러나 일시중지가 되는 것일 뿐 여전히 flow를 구독하고 있다. 즉 pause됐을 때 `MutableSharedFlow<T>.subscriptionCount`는 변하지 않는다(일시중지지 취소된 것은 아니라는 뜻). `SharingStarted.WhileSubscribed()`를 잘 이용하려면, `onStop()`에 구독을 취소하고, `onStart()`에 재구독 로직을 추가해야 한다. collection을 취소하고 다시 coroutine을 생성하여 collect하라는 의미다.

- **주의사항** : `==`으로 `oldState`과 `newState`를 비교하기 때문에 리스트 item의 일부 속성이 바뀌지 않으면 `.mutableList()`나 `ArrayList(list)` 따위로 새로운 리스트를 만들어 줘도 기존값을 유지한다(새로운 값으로 업데이트 하지 않는다).

### 참고
- https://proandroiddev.com/going-deep-on-flows-channels-part-3-channels-df150458693b
- https://proandroiddev.com/should-we-choose-kotlins-stateflow-or-sharedflow-to-substitute-for-android-s-livedata-2d69f2bd6fa5
