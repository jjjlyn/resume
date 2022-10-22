# Android Architecture

안드로이드 앱은 익숙해지면 찰리 채플린이 컨베이어 벨트에서 나사를 조이는 것 마냥 반복적으로 찍어낼 수 있게 됩니다.</br>
그러나 전체 플랫폼의 동작 원리를 이해하는 것은 여전히 쉽지 않습니다.</br> 
시스템 관련한 비슷한 주제의 책들을 몇번이나 뒤적여도 뒤돌면 잊어버리더군요.</br>
아무래도 확실한 이해가 부족한 탓인 것 같습니다.</br>
공부한 내용을 숙지하고, 문서를 지속적으로 개선하자는 차원에서 기록합니다.</br>
제가 이해하는 선에서 최대한 정리하였으나 간혹 틀린 내용이 있을 수 있습니다.

하단은 안드로이드 아키텍처 입니다.

![Overall Architecture](/android/images/android-overall-architecture.png)

## Android Framework
어디까지가 안드로이드 프레임워크인가?
![](/android/images/android-framework.png)

### Init

Kernel Init 완료 -> Android Init (.exe 파일을 커널에서 실행해준다고 생각하면 됨) -> *init* 프로세스 실행 -> *app_process* 프로세스 실행 -> `AndroidRuntime` 객체 생성 -> Dalvik VM 생성 -> (AndroidRuntime이 Dalvik VM에 Zygote를 실행하라고 요청) -> Zygote 실행

### Zygote
**역할**

Zygote는 자기 자신을 초기화 하면서 안드로이드 프레임워크(*$AOSP/framework*) 전체를 사전에 로드합니다.(런타임에서 필요할 때 동적으로 라이브러리를 로드하는 방식이 아닙니다) 초기화가 완전히 끝난 후 루프(loop)에 들어가 소켓을 열고 안드로이드 시스템으로부터 '새로운 프로세스를 시작한다는 요청'을 기다립니다. 시스템은 새로운 프로세스를 시작할 때 Zygote 소켓에 연결한 후, 어떠한 프로세스가 생성되어야 하는지에 대한 정보가 담긴 작은 패킷을 전송합니다. 
![Zygote creates new process](/android/images/zygote-create-new-process.png)
Zygote는 전송받은 패킷을 바탕으로 자기 자신을 fork(복제)하여 새로운 프로세스를 생성합니다. 

즉, Zygote는 *(1)*안드로이드 환경에서 공통적으로 사용하는 리소스와 라이브러리를 사전에 로드하고 *(2)*새로운 프로세스를 생성할 때 자기 자신을 복제하는 역할을 합니다. 이 두 가지 기능은 프로세스 실행 시간을 단축하고 OS가 효율적으로 메모리 관리를 할 수 있도록 합니다.

보통 자바 코드로 작성된 데스크탑 어플리케이션을 실행하려면 다음 과정을 거쳐야 합니다.</br>
- OS 자원 할당
- JVM 실행
- Java 라이브러리 lazy load
- 기본 라이브러리 초기화
- 어플리케이션 로드
- 어플리케이션 로드하면서 새롭게 추가된 클래스 로드
- minor GC
- 위 단계가 완료된 후 VM이 어플리케이션의 `main()`을 실행

위에서 언급한 일련의 과정은 배터리로 전원을 공급받는 모바일 기기에서는 치명적일 수 있습니다. 앱 실행 몇 번하다 배터리가 방전되는 사태가 벌어질 수 있습니다. 새로운 프로세스가 실행될 때마다 이 과정을 반복하는 것을 방지하기 위해 나온 해결책이 바로 **Zygote**입니다.

**동작**


**Dalvik Runtime에서 Zygote (< Android 5.0)**

![Init Dalvik VM](/android/images/init-dalvik-vm.png)
Zygote는 자바 코드로 작성되어 있으므로 궁극적으로 이를 실행하려면 바이너리 코드로 변경해야 합니다. 일단 javac로 바이트코드(.class)로 바꿔주는 작업을 합니다. class파일을 jar로 묶어주고, *dx*툴을 이용하여 DEX 파일로 변환합니다. 변환된 Zygote DEX 파일은 Dalvik VM에서 한 줄 씩 해석하여 실행할 것입니다. 이를 JIT(Just In Time) 컴파일이라 합니다. 사실 따지고 보면 Zygote도 다른 자바 어플리케이션과 다를 바가 없는 것입니다. 차후 시스템으로부터 새로운 어플리케이션의 생성 요청이 들어올 때 실행된 Zygote 프로세스를 fork한 후 새로운 어플리케이션을 로드하여, 이 역시 Dalvik VM에서 돌아갈 수 있도록 합니다.

**ART Runtime에서 Zygote (>= Android 5.0)**

ART도 Dalvik VM과 마찬가지로 DEX를 매개언어로 사용합니다. ART가 도입되기 이전에 자바로 작성한 프로그램들(DEX 형식)을 Dalvik Runtime이 한 줄 씩 해석한 후 실행했다면(JIT 컴파일러), ART Runtime은 한꺼번에 네이티브 코드로 변환한 후 실행합니다. 이를 AOT(Ahead of Time) 컴파일이라 합니다. 비록 JIT에서 AOT로 컴파일 실행 방식이 바뀌었지만, **DEX 파일을 바이너리 코드로 변환·실행하는 근간**은 유지되는 것이기 때문에, 기존 프로그램들은 이에 영향받지 않고 정상적으로 동작합니다. ART에서 DEX를 바이너리 파일로 변환하는 도구를 *dex2oat*라고 합니다. 보통 매개언어(e.g. DEX)는 이식성(portability)을 높이기 위해 사용합니다. *dex2oat*는 여러 하드웨어에서 돌아갈 수 있도록 각종 버전을 제공하고 있고, 이는 안드로이드 기기에 OS의 한 부분으로서 설치됩니다. *dex2oat*가 각 하드웨어에 맞추어 바이너리 코드로 컴파일 해주기 때문에 DEX는 하드웨어 플랫폼을 추가적으로 신경 쓸 필요가 없습니다. 즉 ART는 이 도구를 사용함으로써, 매개언어의 기본 철학인 이식성을 보장합니다.

### 참고
**전문 서적**
- [Android System Programming - Porting, customizing, and debugging Android HAL](https://www.amazon.com/Android-System-Programming-customizing-debugging/dp/178712536X)
- [Inside the Android OS: Building, Customizing, Managing and Operating Android System Services (Android Deep Dive)](https://www.amazon.com/Inside-Android-OS-Customizing-Operating/dp/0134096347)
- [Embedded Android: Porting, Extending, and Customizing](https://www.amazon.com/Embedded-Android-Porting-Extending-Customizing/dp/1449308295)

**블로그**
- [Android Framework - 네이버](https://m.blog.naver.com/PostList.naver?blogId=bl2019&categoryName=an..%C2%A0framework&categoryNo=8&logCode=0)
- [안드로이드 어플리케이션이 실행되기까지 - 티스토리](https://sanseolab.tistory.com/32)