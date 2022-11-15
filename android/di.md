# DI (Dependency Injection)

## DI란?

Object의 호출(invocation)과 생성(creation)을 분리하는 것을 말합니다. 보통 이를  *관심사 분리(Separation of Concerns)* 혹은 *단일 책임 원칙(Single Responsibility Principle)*이라고 합니다. 

- Service
- Interface
- Client
- Injector

## Dagger2

생명주기를 갖는 특정 컴포넌트(Component)에 모듈(Module)을 설치하여 사용합니다. 모듈은 의존성(Dependency)를 관리합니다.

## Hilt

Dagger2가 안드로이드 전용이 아닌 자바 범용 라이브러리이다 보니, 안드로이드 개발 시 추가적으로 보일러플레이트 코드를 적용해야 하는 번거로움이 있긴 합니다. 이를 개선하기 위해 안드로이드 어플리케이션에 맞게 설계된 Hilt가 출시되었습니다.

Hilt는 Dagger2 라이브러리를 감싸는 Wapper입니다. Dagger2에서 사용하는 **@Component**, **@Subcomponent** 어노테이션을 제거하고 새로운 어노테이션을 사용합니다.

- Application 클래스에는 **@HiltAndroidApp**을 사용합니다.
- Activity, Fragment, Service, Bradcast Receiver, View 등에는 **@AndroidEntryPoint**를 사용합니다.
- ViewModel에는 **@HiltViewModel**을 사용합니다.