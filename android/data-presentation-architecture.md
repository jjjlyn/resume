# Android Data Presentation Architecture - MVP, MVVM, MVI

MVC, MVVM, MVI 등 `MV` 접두어가 달린 친구들은 **Data Presentation** (Clean Architecture 관점에서 Presentation Layer)을 위한 소프트웨어 아키텍처 패턴입니다. 안드로이드 앱은 UI로 사용자에게 데이터를 보여주는 것이 주 목적입니다. 사용자의 관점에서 앱 화면은 어떻게 개발을 했든(코드 퀄리티가 좋든 나쁘든 ㅋㅋ) 동일하게 보입니다. 그러나 개발자 관점에서는 이야기가 달라집니다. 하나의 프래그먼트에 뷰 업데이트·비지니스·데이터 관리(CRUD) 로직 등이 모두 들어 있다고 생각해봅시다. 제 개인적으로는 유지보수가 매우 끔찍할 것 같습니다. 즉 `Model-View-Whatever` 소프트웨어 아키텍처 패턴은 개발자에게 필요해서 나온 패러다임 입니다. 협업, 유지보수 등이 수월해진다면 이는 궁극적으로 사용자 경험에도 좋은 영향을 주겠죠 :)

## MVP (Model-View-Presenter)
![MVP](/android/images/mvp.png)

비록 실무에서 사용하지는 않았으나 아키텍처 패턴의 변천기를 아는 것은 여전히 중요하기에 간단히 다룹니다.

안드로이드 초창기에는 MVC와 유사한 아키텍처 패턴을 주로 사용했습니다. Activity가 Controller, View는 android.widget.View 계층, Model은 어플리케이션의 데이터를 관리하는 역할. 
Activity를 최대한 많이 만들어서 다른 앱들과 통신하기 위함. 이러한 이유로 액티비티는 인스턴스화 할 수 없고 오로지 인텐트를 사용하여 시작할 수 있음. 액티비티를 직접적으로 인스턴스화 하지 못하기 때문에 액티비티 생성자에 특정 의존성을 주입할 수가 없다. 또한 액티비티는 생명주기를 갖고 있어 이를 모두 상속받아야 한다. 그리하여 액티비티는 특별한 테스트 도구가 없는 한 유닛 테스팅이 어려워 실제 기기나 AVD 에뮬레이터를 통해 통합 테스트에 의존하는 수밖에 없다. 특별한 테스트 도구(e.g. Roboletric)나 통합 테스트를 사용하는 것은 속도가 느리기도 하거니와, firebase test lab과 같은 테스팅 클라우드를 이용하는 경우 비용적인 측면에서도 부담이 있다.
이러한 문제를 해결하기 위해 액티비티와 차후 나타난 프래그먼트를 위한 Humble Object 패턴이 도입된다. 이는 액티비티의 복잡다단한 로직을 최대한 추출해서 분리하는 패턴이다. 그 중 유명한 방법이 MVP 아키텍처 패턴이다. 액티비티와 뷰계층(android.widget.View)은 View가 되고, presenter는 모델로부터 데이터를 가져와 비지니스 로직을 수행하며, 모델은 mvc의 m의 역할을 수행한다. 즉 어플리케이션의 데이터를 관리하는 역할을 한다. Presenter는 데이터를 가공하여 View에 업데이트한다. View 또한 user interaction이 발생했을 때 Present를 호출한다. 두 컴포넌트의 관계로 인해 둘 간의 관계(contract)를 인터페이스로 정의해야 한다.

```kt
interface Presenter {
    fun loadUsers()

    fun validateInput(text: String)
}

interface View {
    fun showUsers(users: List<User>)

    fun showInputError(error: String)
}
```

뷰와 유즈케이스(모델)을 Presenter에서 모두 관리한다. PresenterImpl은 View와 GetUsersUseCase 객체에 의존성을 갖는다. GetUsersUseCase를 통해 사용자 목록을 받은 후 View의 showUsers()가 호출된다. 

```kt
class PresenterImpl(
    private val view: View,
    private val getUsersUseCase: GetUsersUseCase
): Presenter {

    private val scope = CoroutineScope(Dispatchers.Main)

    override fun loadUsers(){
        scope.launch {
            getUsersUseCase.execute().collect { users ->
                view.showUsers(users)
            }
        }
    }

    override fun validateInput(text: String){
        if(text.isEmpty()){
            view.showInputError("Invalid input")
        }
    }
}
```

View 구현체는 이런 식으로 표현된다. 
```kt
class MainActivity : ComponentActivity(), View {
    @Inject
    private lateinit val presenter: Presenter
    private lateinit val usersAdapter: UsersAdapter
    private lateinint val editText: EditText
    private lateinit val errorView: TextView

    override fun onCreate(savedInstanceState: Bundle?){
        super.onCreate(savedInstanceState)

        editText.addTextChangedListener(object : 
            TextWatcher {
                override fun afterTextChanded(s: Editable?){
                    presenter.validateInput(s?.toString().orEmpty())
                }
            }
        )

        presenter.loadUsers()
    }

    override fun showUsers(users:List<User>){
        usersAdapter.add(users)
    }

    override fun showInputError(error: String){
        errorView.text = error
    }
}
```

Presenter가 백그라운드 작업 수행을 종료할 때 Context 누수가 발생할 수 있다. 액티비티 생명주기에 Presenter 객체의 메모리를 release하는 작업이 필요하다.

```kt
interface Presenter {
    fun close()
}
```

```kt
override fun onDestroy(){
    presenter.close()
    super.onDestroy()
}
```

```kt
class PresenterImpl(
    private val view: View,
    private val getUsersUseCase: GetUsersUseCase
): Presenter {
    private val scope = CoroutineScope(Dispatchers.Main)

    override fun close(){
        scope.cancel()
    }
}
```

Flow 객체에 대한 구독을 취소했으니 Activity가 destroy되었을 때 더이상 업데이트를 받지 않게 된다.

비록 이 MVC를 개선한 MVP가 초기에는 많이 쓰였고 아직도 많은 상용앱에서 사용하고 있지만, 이 또한 여러 문제를 야기한다. MVVM 패턴을 장착한 AAC 출시 이후, 추가적으로 Jetpack Compose까지 더해서 더 좋은 data flow를 제공한다.

## MVVM (Model-View-ViewModel)

![MVVI](/android/images/mvvi.png)
AAC(Android Architecture Components)를 통해 뷰 업데이트 로직과 비지니스 로직을 분리합니다.

MVVM은 액티비티나 프래그먼트로부터 로직을 추출하는 Humble Object 패턴과는 다른 접근방식이다. View는 액티비티 혹은 프래그먼트, Model은 data management, ViewModel은 View가 요청할 때 Model로부터 데이터를 요청한다. 세 컴포넌트는 단방향의 흐름을 갖는다. View는 ViewModel에 의존성을 갖고, ViewModel은 Model에 의존성을 갖는다. 많은 View가 동일한 ViewModel을 사용할 수 있기 때문에 더 유연하다. View에서 데이터를 업데이트 하기 위해 ViewModel은 Observer 패턴으로 구현해야 한다. ViewModel은 Observable을 이용하여, View가 이를 구독하고 데이터를 실시간으로 변경할 수 있도록 해야 한다. 

AAC 라이브러리는 ViewModel 클래스를 제공하는데, 이는 액티비티와 프래그먼트의 생명주기에 맞춰 데이터 흐름을 관리해준다. Coroutine extensions와 결합된 AAC ViewModel은 액티비티나 프래그먼트가 더이상 데이터를 받으면 안되는 생명주기 상태에 있을 때 flow 혹은 coroutine의 구독을 중지하여 컨텍스트 누수를 막는다.

Clean Architecture 관점에서 ViewModel은 UseCase로부터 데이터를 가져온 후 entity를 Framework 계층이 필요로 하는 객체로 변환하는 역할을 한다(데이터 가공처리). 또한 사용자가 넘긴 데이터를 entity로 가공하여 UseCase로 넘기는 반대의 역할도 수행한다. 

```kt
class MyViewModel(
    private val getUsersUseCase: GetUsersUseCase
): ViewModel(){

    private val _usersFlow = MutableStateFlow<List<UiUser>>(listOf<UiUser>())
    val usersFlow: StateFlow<List<UiUser>> = _usersFlow

    fun load(){
        viewModelScope.launch {
            getUsersUseCase.execute().map {
                // Convert List<User> to List<UiUser>
            }.collect {
                _usersFlow.value = it
            }
        }
    }
}
```

## MVI (Model-View-Intent)
어플리케이션의 상태 관리 차원에서 이점이 있는 아키텍처이다. 

## 그렇다면 작성자 본인은 무슨 패턴을 선택하였는가

## 참고

**전문 서적**

- [Clean Android Architecture: Take a layered approach to writing clean, testable, and decoupled Android applications](https://www.amazon.com/Clean-Android-Architecture-decoupled-applications/dp/180323458X)