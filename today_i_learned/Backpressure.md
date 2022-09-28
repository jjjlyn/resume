# Backpressure
코틀린 코루틴은 라이브러리 제작자가 다양한 비동기 프로그래밍 스타일을 적용할 수 있도록 언어적 특성을 제공한다. 이는 functional Rx style에 국한되지 않는다. 
코루틴을 이용하면 imperative | promise/futures | actor 등 다양한 스타일을 차용할 수 있다. Rx는 이와 달리 특정한 functional style로 그 사용성이 제한된다. 
즉 Rx는 **Kotlin coroutines를 이용한 특정 라이브러리**와 비교 대상이 되는 것이 맞다. (Kotlin coroutines 그 자체와는 비교할 수 없음.) 
예를 들어 Rx는 kotlinx.coroutines library와 비교할 수 있다. 
이 라이브러리는 다양한 원시타입을 제공하는데 async/await, channel 지시어가 그 예이다.
그렇다면 Rx와 코틀린을 이용한 라이브러리 중 하나인 **Flow**를 비교해보자. 

Backpressure는 인풋에 대한 아웃풋 처리가 ‘느린 computation speed’로 인해 제대로 되지 않은 것이다(대부분).
코루틴은 backpressure를 처리하는 기능을 빌트인으로 갖고 있는 반면, Rx는 이와 같은 기능이 기본 탑재되어 있지는 않아 Flowable를 이용하여 따로 처리한다.
Observable의 data emitting 속도가 너무 빨라서 consumer가 이를 바로 소비하지 못할 경우가 있다(속도를 따라가지 못함). 
이럴 경우 분출되었으나 미처 소비하지 못한 데이터를 어떻게 관리하고 처리할 지에 대한 전략이 바로 Backpressure strategy이다.
예를 들어 찰리 채플린이 공장 컨베이어 벨트에서 인형에 눈알을 붙이는 반복적인 행위를 한다고 가정하자. 
채플린의 작업 처리 속도가 컨베이어 벨트의 작동 속도를 따라가지 못할 경우 어떻게 될까? 크게 두 가지 방법이 있을 것이다.
- 인형을 모아두고 한꺼번에 눈알을 붙인다. (Buffering)
- 인형을 몰래 버려버린다. (Dropping)
그런데 이런 편법을 쓰게 되면 찰리 채플린은 공장에서 바로 해고다. 
대신 컨베이어 벨트의 가동 속도를 늦춰 버리면 어떨까? 이 경우 찰리 채플린은 컨베이어 벨트(Producer)의 속도를 조절할 권한이 있어야 한다.

## 예시: 파일 읽기 | 쓰기
기본적으로 파일 쓰기가 읽기보다 더 느리다. 하드 드라이브의 읽기 속도 150MB/s 쓰기 속도 100MB/s라고 가정하자. 
만약 최대한 빨리 파일을 메모리로 불러 들이는 동시에(read), 파일을 다시 디스크에 쓴다면(write)? 1초마다 50MB의 버퍼가 생길 것인데, 이건 골때리는 손실이다… 
인풋 파일을 모두 읽어들여도 파일을 디스크에 출력하는 작업은 완료되지 못할 것이다.
6GB 파일이라면 읽기 작업이 완료돼도, 여전히 2GB의 버퍼가 남아있을 것이다. 이건 심한 메모리 낭비고, 또 소프트웨어의 가용 메모리를 초과할 여지도 있다. 
또한 다수의 요청 처리를 해야 하는 웹서버인 경우 문제는 더 심각해진다.
매우 간단한 해결책이 있다. **쓰는 속도만큼만 읽어라!!! (Only read as fast as you can write)**
거의 모든 I/O 라이브러리가 이를 streams 혹은 pipes라는 개념으로 제공하고 있다. (Node.js가 그 예이다.)

- Pull based streams
consumer가 producer을 제어한다. 주로 1:1 요청 <-> 응답에 해당한다(?) 
e.g. Flowables (RxJava)

- Push based streams
User input을 다루기 위한 설계 방식이다(User의 행위를 제어할 수 없을 때). 
e.g. RxJava. Consumer가 값을 소비할 수 있는 상태일 때 데이터를 push한다. 즉 Producer는 Consumer의 availibility 여부에 따라 in control.

## 코루틴은 3개의 전략으로 backpressure을 다루며, 데이터 처리 속도를 개선한다. 
- buffer : 다운스트림에서 데이터가 처리될 때까지 source에 보관한다.
- conflate : 가장 최신의 value만을 return하여 속도를 증진한다.
- collectLatest : 느린 collectors은 취소되며, 새로운 데이터가 분출(emit)될 때마다 재시작된다.

## Rx는 기본적으로 backpressure를 제공하지 않는다.
Observable은 기본적으로 backpressure을 다루지 않는다. 만약 데이터를 분출(emit)하는 source가 있다면, 가장 최신의 것이 관찰(observe)되며, 관찰/구독 이전의 데이터는 처리되지 않는다. 
대신에 Rx는 `Flowable`(supported primitive)을 이용하여 backpressure을 처리한다.
- BackpressureStrategy.BUFFER : source로 하여금 구독자가 소비하기 전까지 계속 데이터를 들고 있을 수 있도록 한다. 이는 코루틴의 default 설정과 비슷하다.
- BackpressureStrategy.DROP : 구독자에 의해 바로 소비되지 못한 이벤트는 폐기된다…(?) 
- BackpressureStrategy.LATEST : 구독자가 있을 경우 이전 데이터를 새로운 데이터로 덮어쓰기 한다. 이는 coroutine의 conflate와 비슷한 방식이다. 
그 이유는 conflate는 중간 값을 스킵하긴 하지만 명시적으로 이를 지우거나 버리거나 하지 않기 때문이다. 
