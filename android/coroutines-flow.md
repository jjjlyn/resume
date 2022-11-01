# Coroutines & Flow

## CoroutineScope
모든 코루틴 작업은 CoroutineScope가 관리한다. GlobalScope는 CoroutineScope의 전역 인스턴스다.

**CoroutineScope는 더 이상 필요하지 않을 때 코루틴을 취소하거나 정리하는 작업을 한다.** GlobalScope는 프로세스의 생명주기를 따라가기에 가장 수명(생명주기)이 길다. 문서 등에서 예시를 들 경우에는 GlobalScope를 사용하는 것이 적합하지만, 실제 개발 시에는 더 범위가 작은 Scope가 필요하다. 가령 UI를 업데이트하기 위해 I/O를 할 때 다른 화면으로 전환하면 더 이상 이 코루틴 작업이 필요없게 된다. Android에서 Jetpack은 이를 위해 다양한 범위에 걸친 CoroutineScope 구현체를 제공한다.

CoroutineScope는 개발자가 생성한 코루틴을 추적하여, 실행된 코루틴을 모두 취소할 수 있다. 

코루틴 스코프는 다수의 코루틴의 구조화된 동시성을 지원한다. 예를 들어 코루틴 스코프 내부의 특정 코루틴이 crash될 때 나머지 코루틴은 모두 취소된다. 

코루틴 스코프는 어떻게 생성되나?

- GlobalScope
- 코루틴 빌더를 실행할 때
- 프레임워크가 지원하는 스코프를 사용할 때
- withContext()를 호출할 때 - 현재 dispatcher를 변경하여 다른 스레드 풀에서 코루틴을 실행하고자 할 때
- coroutineScope()를 호출할 때 - 현재 dispatcher를 유지한 채 새로운 코루틴 스코프를 생성하고자 하는 경우
- supervisorScope()를 호출할 때 - 코루틴 스코프 내부에서 돌아가는 코루틴 하나가 exception이 나도 나머지 코루틴을 취소하지 않고 실행하고자 할 때
- 테스트 스코프(Test Scopes)
- 커스텀 스코프(Own Custom Scopes)

## 코루틴 빌더
GlobalScope를 호출할 때 사용하는 `launch()`는 코루틴 빌더이다. 코루틴 빌더는 람다 표현식을 사용한다. 코루틴이 수행할 실제 작업을 람다 내부에 작성하면 된다.

`launch()`, `async()`, `runBlocking()` 등의 종류가 있다.

**launch()**

*fire and forget*

한번 실행하고 나면 결과를 반환하지는 않는다. `launch()`는 Job 객체를 반환한다. 이는 현재 실행 중인 작업을 관리(e.g. cancel)하는데 사용한다.

**async()**

Job 객체의 subclass인 Deffered가 반환된다. `launch()`가 람다의 결과를 무시(ignore)하는 것과 달리, async는 **Deffered 객체 안에 람다의 결과를 담아 반환**한다. 직접적인 반환값을 필요로 할 때 사용한다. await()으로 async 람다의 결과를 기다린 후 Deffered 객체를 받는다. 

참고로 `await()` 자체도 suspend 함수이기 때문에 코루틴 빌더 혹은 다른 suspend 함수 내에서 호출해야 한다.

**runBlocking()**

suspend 함수를 코루틴 밖에서 호출하고 싶을 경우 사용한다. 나만의 코루틴을 설정할 수 없는 경우(그런 권한이 없는 곳에서 e.g. 안드로이드 프레임워크 메서드)에 `runBlocking()`을 호출하면 코루틴 launcher를 block한다(모든 코루틴의 실행을 block 상태로 만드는 것). `runBlocking()`도 람다를 제공하여 코루틴을 그 내부에서 실행할 수 있다. 그러나 코루틴 작업이 완료되고나서야 비로소 반환된다(block이 풀린다.) 즉 suspend를 만났다고 다른 코루틴으로 갈아타지 않고 계속 대기 상태로 있는 것.

## Dispatcher
코루틴 빌더에 설정을 주는 역할을 한다. 어떤 스레드 풀이 코루틴 내부 코드를 실행하는지를 지정한다.
- Dispatchers.Default은 일반적인 백그라운드 작업을 할 때 유용하다.
- Dispatchers.Main은 main 스레드가 필요한 작업을 할 때 사용한다.

```kt
fun main() {
    GlobalScope.launch(Dispatchers.Main){
        println("This is executed before the delay")
        stallForTime()
        println("This is executed after the delay")
    }

    println("This is executed immediately")
}
```

`main()`은 `launch()`를 사용하여 코루틴 빌더 내부 코드 블록이 main 스레드에서 실행되도록 한다. 이는 `stallForTime()`과 `println()`을 main 스레드에서 실행한다는 것을 의미한다. 

## suspend Function
코루틴 빌더는 특정 스레드 풀에서 실행되는 코드 블럭을 만든다.

자바 개발자 입장에서는 Runnable 위에 Executor가 올라가 실행되는 것과 유사하다고 생각하면 된다. Dispatchers.Main이라면 안드로이드 개발자 입장에서 View 안에서 Runnable을 `post()`로 감싸서 Runnable이 main 스레드에서 실행되게 하는 것과 비슷하다.

그렇다고 완전 동일한 것은 아니다.

Runnable 시나리오에서는 지정된 스레드(혹은 스레드 풀)의 작업 단위가 Runnable 그 자체이다. 스레드가 Runnable의 코드를 실행하면, 이 스레드가 작업 중으로 바뀌기 때문에 또다른 Runnable 객체를 넘겨줘도 첫번째 Runnable 작업이 종료될 때까지 기다려야 한다.

그러나 코틀린은 suspend 키워드를 제공한다. 코루틴 시스템(현재 스레드)에서는 *현재 실행 중인 코드 블럭을 중지하고* 다른 코루틴 빌더의 코드로 넘어갈 수 있다.(넘어가려는 코루틴 빌더에 실행 가능한 코드가 있다는 전제하에)

`stallForTime()`에 suspend 키워드가 있다. 코틀린이 launch 내부 코드를 실행하면 `stallForTime()`이 호출되는 시점에 코루틴 코드 블럭 실행이 중단되기 때문에, 코틀린이 Dispathcers.Main에서 실행하려는 다른 코루틴들 중 하나를 선택해서 실행할 수 있다. 그러나 코틀린이 `stallForTime()` 뒤에 있는 `println()`을 실행할 수는 없다. `println()`은 `stallForTime()`가 반환되고 나서야 비로소 실행된다.

suspend 키워드가 표시된 함수는 코루틴 빌더 혹은 또다른 suspend 함수 내부에서만 실행할 수 있다. `stallForTime()`은 `launch()` 코루틴 빌더가 실행하는 코드에서 호출한 것이기 때문에 가능하다. `delay()` 역시 suspend 함수이다. 이또한 `stallForTime()`이 suspend 함수이기 때문에 이 내부에서 실행할 수 있다.

## Context
코루틴 빌더에 제공되는 dispatcher는 Coroutinecontext의 일부분인데, 이는 이름 그대로 coroutine을 실행하는 context를 말한다. Dispatcher도 context의 일부분이지만, job이라는 것도 있다.

`withContext()`(최상위 함수 - 클래스 밖에 위치한 전역함수)는 suspend 함수이다. 즉 다른 suspend 함수 내부나 코루틴 빌더 내부에서만 실행될 수 있다. `withContext()`는 기존 코드 블록과 다른 블록으로 갈아타 변경된 CoroutineContext에서 실행된다. 

이 책의 예시에서는 `withContext()`를 사용하여 다른 dispatcher로 갈아탄다. main 스레드에서 코루틴이 처음 시작하고 `stallForTime()` 내부에서 `delay()`를 Dispatchers.Default 스레드에서 실행한다(`withContext(Dispatchers.Default)`). `stallForTime()의 withContext()` 내부 코드 실행이 완료될 때까지 Dispatchers.Main의 현재 코드 블록의 실행이 중지될 것이고(suspend 함수이기 때문에), 이 코드가 Dispatchers.Default 스레드 풀의 특정 스레드에서 실행하고 결과값을 반환하기까지 코틀린은 Dispatchers.Main에서 실행할 수 있는 코루틴들 중 하나를 선택해서 실행할 수 있다.

## 스레드보다 비용이 저렴하다!
스레드 중심적인 프로그래밍에서는 많은 스레드를 생성하는 것에 대한 부담이 있다. 각 스레드는 heap 공간을 꽤 많이 소비한다. 게다가 스레드 간의 컨텍스트 스위칭이 발생하면 이에 따른 CPU 자원 낭비(?) 오버헤드가 생긴다. 그래서 스레드 풀의 사이즈를 조절하는 알고리즘이 있는 것이다.

그러나 코루틴은 이에 비해 비교적 저렴하다.

## 코루틴의 역사
### 선점 방식과 협동 방식의 멀티태스킹
프로그래밍에서 멀티태스킹은 프로그램들이 어떻게 한번에 여러가지 작업을 수행하는 것처럼 보이게 하는가를 의미한다. 주요 관점으로는 협동 방식과 선점 방식이 있다.

대부분의 최신(?) 프로그래머들은 선점 방식의 멀티태스킹을 생각한다. 이 방식은 어떠한 멀티태스킹 방식으로 돌아가기 잊기 쉽게 구현되어 있다. 태스크가 실행되거나 실행되지 않도록 준비하는 작업은 OS가 처리하기 때문이다. OS가 프로세스와 스레드 스케쥴링을 통해 CPU core에 올리고, 다른 프로세스나 스레드로 context switching하는 것을 보장하기 때문에 멀티태스킹의 한 방식이라는 것 조차 잊게 되는 것이다.(개인적으로 프로그래머가 할 일이 없으니...?)

그러나 협동 기반으로 작동하는 멀티태스킹도 있다. 어떤 작업이 CPU 시간을 필요로 할 떄 이 작업으로 context-switching을 하는 것이다. 이는 코드 상에서 명시적으로 yield(양도)를 하겠다고 명시해야 한다. 특정 프레임워크는 yield point에서 태스크를 전환하는 것을 지원한다. 16-bit 윈도우 프로그램이나 원조 Mac OS는 이 방법을 사용했다.

최근에 협동 멀티태스킹은 프로세스 내부 프레임워크에서나 사용된다. OS는 선점 방식을 채택하여, 프로세스의 잘못된 행동을 관리해준다.

**내가 이해한 바로는, 선점 방식은 Round Robin 알고리즘을 이용하여 일정 시간동안 CPU 자원을 사용할 수 있도록 프로세스에 분배하는 것이다. 짧은 시간동안 빈번하게 배분하는 방식이다. 이는 *OS가 주관하기 때문에, 또한 time scheduling을 채택하기 떄문에* 스레드 A에서 스레드 B에 CPU 자원을 양보(yield)할 것이라는 것을 코드 단에서 명시할 필요가 없다.**

**협동 방식은 특정 태스크가 CPU 자원이 필요한 경우(READY 상태) 현재 실행 중인 태스크에서 CPU 자원을 양보해 주는 것이다. 이는 코드 상에서 태스크 A에서 태스크 B로 자원을 양도한다는 것을 명시해야 할 필요가 있다. 코틀린은 언어상 suspend를 통해 타 태스크로 CPU 자원을 양도할 시간이라는 것을 명시한다.**

### Green and Red
가끔 프레임워크가 선점형 멀티태스킹을 제공하는 것과 같은 착각(효과)을 주지만 실제로는 협동 멀티태스킹 모델을 채택하는 경우도 있다.

초기 자바의 경우, 스레드는 JVM이 관리한다. JVM이 스레드 간의 코드 실행을 스위칭하는 방식이다. 코드에서 각 스레드는 타 스레드에 코드 실행을 양도한다는 것을 명시할 필요가 없다. 왜냐하면 JVM이 코드 실행의 책임을 갖고 있어, 자기 자신(JVM)의 '스레드들'을 스위칭하는 것이기 때문이다 -> 여기서 의문: 그렇다면 JVM은 time scheduling을 사용하지 않는가?. *OS의 관점에서 이는 협동 멀티태스킹이라고 볼 수 있다. 그러나 자바 프로그래머 관점에서는 선점 멀티태스킹인 것이다.*

이런 방식을 소위 "green threads"라고 하는데, 지금은 Thread가 OS Thread로 매핑되는 "red threads"(-> native thread라는 건가? 1:1 유저-커널 스레드 매핑을 말하는건가?)로 대체되었다. browser applets(플러그인 같은 작은 응용 프로그램)이나 스윙 데스크탑 앱(이게 뭐지?) 등에서 웹 앱으로 자바 사용처가 변경되면서 CPU 파워의 사용을 최대한 극대화(멀티코어 프로세서를 최대한 사용)하는 병렬 프로세스가 중요해졌다. (뭔가 좀... 무슨 소리인지 모르겠네;;)

> 여기서 Green Threads가 무엇인가?
> 커널 위 유저 공간에 JVM이 올라가면 JVM이 여러 개의 스레드를 생성한다. 이를 Green Threads라 한다. Green Threads는 커널 스레드 한 개와 연결된다. 하나의 스레드만 커널에 접근할 수 있기 때문에 (N:1 모델이기 때문에 여러 개 스레드 중 단 한 개의 유저 영역 스레드만 일정 시간동안 커널 스레드와 매핑된다.) 인터럽트가 발생하면 blocking 돼서 나머지 스레드가 일을 할 수 없게 된다. 멀티코어 아키텍처에서는 비효율적이다. 

Our discussion so far has treated threads in a generic sense. However, support for threads may be provided either at the user level, for user threads, or by the kernel, for kernel threads. User threads are supported above the kernel and are managed without kernel support, whereas kernel threads are supported and managed directly by the operating system. Virtually all contemporary operating systems—including Windows, Linux, and macOS— support kernel threads.
Ultimately, a relationship must exist between user threads and kernel threads, as illustrated in Figure 4.6. In this section, we look at three common ways of establishing such a relationship: the many-to-one model, the one-to- one model, and the many-to-many model.

4.3.1 Many-to-One Model
The many-to-one model (Figure 4.7) maps many user-level threads to one kernel thread. Thread management is done by the thread library in user space, so it is efficient (we discuss thread libraries in Section 4.4). However, the entire process will block if a thread makes a blocking system call. Also, because only

one thread can access the kernel at a time, multiple threads are unable to run in parallel on multicore systems. Green threads—a thread library available for Solaris systems and adopted in early versions of Java—used the many-to- one model. However, very few systems continue to use the model because of its inability to take advantage of multiple processing cores, which have now become standard on most computer systems.

4.3.2 One-to-One Model
The one-to-one model (Figure 4.8) maps each user thread to a kernel thread. It provides more concurrency than the many-to-one model by allowing another thread to run when a thread makes a blocking system call. It also allows mul- tiple threads to run in parallel on multiprocessors. The only drawback to this model is that creating a user thread requires creating the corresponding kernel thread, and a large number of kernel threads may burden the performance of a system. Linux, along with the family of Windows operating systems, imple- ment the one-to-one model.

4.3.3 Many-to-Many Model
The many-to-many model (Figure 4.9) multiplexes many user-level threads to a smaller or equal number of kernel threads. The number of kernel threads may be specific to either a particular application or a particular machine (an application may be allocated more kernel threads on a system with eight processing cores than a system with four cores).

Let’s consider the effect of this design on concurrency. Whereas the many- to-one model allows the developer to create as many user threads as she wishes, it does not result in parallelism, because the kernel can schedule only one kernel thread at a time. The one-to-one model allows greater concurrency, but the developer has to be careful not to create too many threads within an application. (In fact, on some systems, she may be limited in the number of threads she can create.) The many-to-many model suffers from neither of these shortcomings: developers can create as many user threads as necessary, and the corresponding kernel threads can run in parallel on a multiprocessor. Also, when a thread performs a blocking system call, the kernel can schedule another thread for execution.
One variation on the many-to-many model still multiplexes many user- level threads to a smaller or equal number of kernel threads but also allows a user-level thread to be bound to a kernel thread. This variation is sometimes referred to as the two-level model (Figure 4.10).
Although the many-to-many model appears to be the most flexible of the models discussed, in practice it is difficult to implement. In addition, with an increasing number of processing cores appearing on most systems, limiting the number of kernel threads has become less important. As a result, most operating systems now use the one-to-one model. However, as we shall see in Section 4.5, some contemporary concurrency libraries have developers identify tasks that are then mapped to threads using the many-to-many model.

### 코루틴의 컨셉
코루틴은 협동 방식 컨셉이다. 코루틴은 다른 코루틴에 실행을 양도할 수 있는 특징을 가지고 있어, 언제든 바로 실행 가능한 상태(Ready)의 코루틴이 대기하고 있으면 이를 즉시 실행할 수 있다. 또한 코루틴은 다른 코루틴이 실행되는 것을 기다릴 수 있다.

전통적인(?) 코루틴에서는 yield 키워드를 그대로 사용한다. 코루틴에서는 suspend 키워드를 사용하고, 이를 만나는 시점에서 양도(다른 코루틴에 코드 실행을 넘기는 행위)를 자동적으로 수행한다.즉 프로그래밍 관점에서 우리는 yield control에 대해 직접 생각할 필요가 없다. 

코루틴은 여느 협동 방식을 채택하는 멀티태스킹과 같이, 협동(cooperation)을 필요로 한다. 예를 들어 코루틴이 다른 코루틴에 실행을 양도하지 않을 시 다른 코루틴들은 일정 시간동안 실행할 수 없게 된다. 그래서 코루틴의 협동 방식은 앱(프로세스) 간의 관계에서는 좋은 선택이 아닐 수 있다. 왜냐하면 앱 개발자들은 본인의 앱이 타 앱에 비해 중요도가 높다고 생각하기 떄문이다. 그러나 앱 내부에서는 코루틴 간의 협동이 중요하다. 코루틴 하나가 잘못 동작하는데 다른 코루틴들이 이를 돕지 않으면 앱 크래시 혹은 버그가 발생할 수 있기 때문이다. 즉 코틀린 코루틴은 in-app에서 유용한 동시성 모델이다. 여전히 코틀린은 OS와 선점 멀티태스킹 방식에 의존한다(앱 간의 관계 차원에서 - 프로세스, 스레드).

### 외면받던 코루틴, 다시 돌아오다
멀티스레드 프로그래밍이 각광받던 시기에 코루틴은 잠시 어둠 속으로 사라졌다. 멀티스레드 방식의 복잡성이 밝혀지고 리액티브 프로그래밍이 대두되면서 코루틴은 다시 인기를 얻게 된다.

코틀린은 자바의 "green/red" 스레딩 방식을 공유한다. 코루틴은 작업(코루틴)과 그 작업을 실행할 환경(스레드 혹은 스레드 풀)을 분리시켰다. 코루틴이 작업할 스레드 혹은 스레드 풀을 지정할 수 있다. 즉 안드로이드 main 스레드가 UI 업데이트에만 사용되게 제한하는 등의 문제를 해결할 수 있게 됐다. 

코루틴이 native thread에서 실행되든 자바 green thread에서 실행되든, 이는 코루틴 작성자(우리 같은 개발자)가 아니라 코루틴 라이브러리에서 결정한다. 코루틴은 스레드와 스레드 풀을 공유하고, 사용할 수 있는 실제 스레드보다 훨씬 많은 코루틴을 사용할 수 있다.

## Flows and Channels
### Hot and Cold Stream
GPS 위성은 지구 상에 GPS 수신기가 켜진 상태가 아니더라도 계속 송신한다. 이렇게 수신 여부와 상관없이 계속 데이터를 송신하는 것을 Hot Stream이라고 한다.

스마트폰의 GPS 라디오 기능은 배터리를 절약하기 위해 기본적으로 turn-off 상태이다. 앱이 GPS 상태를 수정해달라고 요청할 경우에만 GPS 정보를 송신한다. 수신 대상이 정보를 요청할 때만 데이터를 송신하는 것을 Cold Stream이라고 한다.

보통 Flow는 cold stream, Channel은 hot stream을 모델로 한 것이다. 물론 hot stream을 위한 Flow(SharedFlow, StateFlow)도 존재한다.

## Dispatcher
코루틴이 어느 스레드에서 돌아갈 것인가는 내가 선택한 dispatcher에 달려있다.

### 일반적으로 제공되는 Dispatchers

**Dispatchers.Default**

코루틴 빌더에 별다른 dispatchers 설정을 주지 않을 시 기본값이 된다. 일반적인 백그라운드 작업을 수행할 때 사용한다.

**Dispatchers.IO**

잠재적으로 block 될 가능성이 있는 I/O 작업에 사용한다. Dispatchers.Default와 스레드풀을 공유한다. 스레드 풀을 사용하는 로직은 Dispatchers.Default와 차이가 있음. 

즉 Dispatchers.Default와 Dispatchers.IO 모두 백그라운드 작업에 사용하나, 전자는 block될 가능성이 없는 작업, 후자는 block될 여지가 있는 작업을 대상으로 한다.

**Dispatchers.Main**

**magic thread**로 불리는 메인 스레드에서 코루틴이 돌아간다. UI 작업을 할 때 사용한다.

### 일반적이지 않은 Dispatchers

`newFixedThreadPoolContext()`

`newSingleThreadContext()`

### Dispatcher 결정
Dispatcher를 결정하는 것은 스레드 혹은 스레드풀을 선택하는 것과 비슷하다.

안드로이드 개발자 입장에서 `launch(Dispatchers.Main)`을 Handler 혹은 View의 `post()`와 비슷하다고 보면 된다. 이 메서드들은 Runnable 객체를 갖고 있으며 메인 스레드의 작업 큐에 하나씩 추가된다. 즉 `post()`를 즉시 호출해도 Runnable을 실행할 순서에 아직 도달하지 못했기 때문에 즉시 실행되지 않는다. 코루틴도 이와 마찬가지로 `launch(Dispatchers.Main)`을 호출한다고 그 즉시 실행되는 것이 아니다.

Dispatchers.Main.immediate라는 특수한 Dispatcher가 있긴 한데, 이걸 사용하면 호출하는 그 즉시 실행된다.

안드로이드 개발자 입장에서 `launch(Dispatchers.Main.immediate)`는 `runOnUiThread()`를 호출하는 것과 같다고 생각하면 된다. `post()`와 같은 역할을 하는 것처럼 보이지만 근소한 차이가 있다.

- `post()`는 항상 Runnable을 작업 큐에 넣는다.
- `runOnUiThread()`
