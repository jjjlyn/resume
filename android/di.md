# DI (Dependency Injection)

> Dagger, Hilt, Dagger-Hilt 등  DI 프레임워크를 사용하면서도 무엇이 어떻게 다른지 모르는 상태였습니다. 구글에서 제공하는 기본 도큐먼트로 사용법 정도만 익힌지라 정확한 이해가 부족했습니다. 그래서 이번 기회에 Hilt에 대해 자세히 알아보고자 합니다.

## DI란?

### Types of dependencies

**DI fundamentals**
This states that we should depend on abstractions rather than concretions. The idea here is to depend as much as possible on abstract classes and interfaces. This can be very difficult to achieve considering that we rely on concretions a lot of the time. Here, we should identify parts of the code that are constantly developed and subject to change and introduce layers of abstractions between our code and these classes. A good way to protect against
this is through dependency injection frameworks such as Dagger and Hilt, which generate factories to create volatile components.


You'll understand what dependencies are and why you need to control them to create successful apps. 

**Implementation inheritance**

B 클래스가 A 클래스에 의존한다. B는 A의 모든 특성을 갖고, A가 하는 모든 행위를 한다. A에 어떠한 행위를 추가하면 B도 그 행위를 해야하기 때문에 B에도 변화가 일어난다. 

그러나 B는 자기만의 특성이 있기 때문에 일반적인 A와는 차이가 있다. 모든 A가 B만의 특징을 갖고 있는 것도 아니고, 네가 B에 아예 관심이 없을 수도 있다(B만이 가진 특성에 대해서 관심이 없을 수도 있다는 뜻). 그래서 이럴 때 B는 A이다 라고 하는 표현보다, A는 B의 일반화 혹은 추상이다 라는 표현이 더 정확하다.

예시 코드
```kt
open class Person(val name: String){
    fun think(){
        println("$name is thinking...")
    }
}

class Student(name: String): Person(name){
    fun study(topic: String){
        println("$name is studying $topic")
    }
}
```

추상화를 할 줄 안다는 것은, 내 관심사가 아닌 관점을 제거하는 방법을 아는 것과 같다. 여기서 Person은 abstraction type이기 때문에 instance로 만드는 것을 방지해준다. 
```kt
abstract class Person(val name: String){
    fun think(){
        println("$name is thinking...")
    }
}
```
추상화의 장점은, 여기서 Person에 다른 특성을 추가해도 이를 의존하고 있는 다른 코드에 영향을 미치지 않는다는 것이다. (an abstraction can lead to a reduction of dependency and, therefore, to changes having a smaller impact on the existing code)

Person은 추상체(abstraction)이다. 즉 specific object가 아니라 description of an abstraction이다. 
Student, Musician, Teacher은 Person 추상 클래스를 실현(?)(realization)한 것이다. 이들은 인스턴스로 만들 수 있는 구체적인 클래스이다.

SOLID 개방 폐쇄 원칙
어떠한 기능을 추가하고자 할 때, 기존 코드를 수정하지 않고 새로운 기능을 추가할 수 있어야 한다.

**Composition over inheritance**

```kt
data class Data(val value: Int)

class Repository {
    fun save(data: Data){

    }
}

class Server {
    private val repository = Repository()

    fun receive(data: Data){
        repository.save(data)
    }
}