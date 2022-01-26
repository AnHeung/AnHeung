---
title: 안드로이드에서 테스트 더블(Test Doubles) 사용하기
layout: single
---
시스템이나 어떤 요소에 대한 테스트 전략을 기획할때는 3가지 고려해야할 점이 있다.

- 범위 (Scope): 얼마나 많은 코드를 테스트 할 것인가? 테스트는 단일 메소드에서 어플리케이션 전체 
또는 그 사이 까지 다양하게 검증할 수 있다. 테스트 범위 (scope) 는 테스트 중인 대상 (Subject Under Test), 
테스트 중인 시스템 (System Under Test) , 테스트 중인 단위 (Unit Under Test) 등으로 불린다.

- 속도 (Speed) : 얼마나 빨리 테스트가 동작 하는가? 테스트 속도는 밀리초 부터 수분 까지 다를 수 있다.

- 충실함 (Fidelity) : 얼마나 실제에 충실했는가? 예를 들어 테스트 코드의 일부가 네트워크 요청을 필요로 한다면 실제 테스트 코드가 네트워크 통신을 요청을 해야하는가 ? 아니면 거짓 결과로 해야하는가? 만약 실제 네트워크 통신을 한다는 것은 충실도가 높다는것이다. 대신 그로인해 테스트는 오래걸리고 비용이 많이 들고 네트워크가 다운될 경우 에러가 날 수도 있다.

### 고립성(Isolation) 과 의존성들(Dependencies)
---
시스템이나 시스템 요소들을 테스트 할때 분리하여 테스트를 한다. 예를 들어 ViewModel 테스트를 한다면 실제 안드로이드 기기나 에뮬레이터를 사용할 필요는 없다. 왜냐하면 안드로이드 프레임워크를 의존하지 않기 때문이다.

그러나 테스트 중인 대상은 작동하기 위해서 의존성이 필요할 수도 있다. ViewModel 생성자에 Data Repository 가
필요하다고 가정해보자. 일반적인 방법으로는 테스트중인 대상에 의존성이 필요하다면 테스트 더블 (Test Double) 또는 테스트 객체 를 제공한다. 

테스트 더블은 앱에서 구성요소 처럼 보이고 동작하는 객체이지만 테스트에서 특정 동작이나 데이터를 공급 해주기 위해 만들어진다. 이를 사용함으로 가장 큰 이점은 테스트가 더 빨리지고 간단해 진다는 것이다.

### 테스트 더블 유형
---

| Fake | 동작에 대한 구현이 있어서 테스트에는 적합하지만 프로덕션에선 적합하지 않는 방식으로 구현된 테스트 더블로 가볍고 Mock 프레임워크가 필요하지 않는다.|
|---|---|
| Mock | 내가 프로그램 한데로 동작하고 상호작용에 대해 기대값을 가진다. 정의한 대로 상호작용 하지 않으면 테스트가 실패한다. Mock 은 보통  Mock 프레임워크로 생성한다. ex) 함수에서 DB 호출이 정확하게 한번만 불렸는지 검증|
| Stub | 내가 프로그램 한대로 행동하지만 상호작용에 대한 기대값은 없는 테스트 더블. 보통 Mock 프레임워크로 생성한다.|
| Dummy |  실제 사용하진 않지만 파라미터 전달용등으로 만드는 테스트 더블 ex) 온 클릭 함수에 전달하기 위한 비어있는 함수  |
| Spy |  실제 객체에 대한 래퍼로 Mock 과 유사하며 추가적인 정보를 기록하기 위한 테스트 더블이다. |
| Shadow | Robolectric 에서 사용하는 Fake 테스트 더블 |


### Fake 사용하기 예제
---
UserRepository 라는 인터페이스를 의존하는 ViewModel 로 최초 유저 이름을 UI 에 노출 시키는 단위 테스트 한다고 가정해보자. 인터페이스를 구현하고 정해둔 데이터를 반환함으로써 Fake 테스트 더블을 만들 수 있다.

```java
object FakeUserRepository : UserRepository {
    fun getUsers() = listOf(UserAlice, UserBob)
}

val const UserAlice = User("Alice")
val const UserBob = User("Bob")
```

이 FakeUserRepository 객체는 프로덕션 버전에서 사용할 실제 Local 또는 Remote Data Source 에 의존할 필요가 없다. 해당 파일은 테스트 소스안에 존재하며 프로덕션 앱에는 있지 않는다.  

{% include image.html id="15ipw7E-dKjrV3-kLdPnJiu4zIhwtVoOj" %}

다음 테스트는 뷰모델의 최초 유저 이름이 뷰에 정확히 노출되는지 확인할 수 있다.

```java
@Test
fun viewModelA_loadUsers_showsFirstUser(){
    //Given : fake 를 사용한 ViewModel
    val viewModel =  ViewModel(FakeUserRepository) //초기화시 데이터 로드 시작
    // 노출된 데이터가 정확한지 검증
    assertEqual(viewModel.firstUserName , UserAlice.name)
}
```
ViewModel 은 테스터에 의해 만들어 졌기 때문에 단위테스트에서 UserRepository 를 Fake 로 교체하기 쉽다. 하지만 더 큰 테스트에서 임의의 요소를 대체한다는건 어려운 일이다.


### 컴포넌트 대체 및 의존성 주입(Dependency Injection)
---
테스트가 테스트 중인 시스템 생성에서 제어가 안된다면 테스트 더블로 컴포넌트를 대체하기 더 복잡해지고 앱의 아키텍쳐가 테스트 가능한 설계를 따르는것이 필요하다.  

심지어 유저의 흐름을 탐색하는 계측 UI 테스트같은 큰 e2e 테스트에서도 테스트 더블을 사용하는건 이점이 있다. 이러한 유연함을 직접 달성하도록 앱을 설계할 수 있지만 테스트 시 앱의 구성 요소를 교체하기 위해 [Hilt](/android/hilt/dependency-injection-with-hilt/) 와 같은 종속성 주입 프레임워크를 사용하는 것이 좋다.

{% include image.html id="15nArPy3qiGXUIW6gdn13IUE_X97tGSru" %}

### [Robolectric](http://robolectric.org/)
---
안드로이드에선 Robolectric 프레임워크를 통해 특별한 타입의 테스트 더블을 제공받을 수 있다.  

Robolectric 는 끊임없는 통합 환경이나 개발머신에서 JVM 을 통해 테스트를 하게 해준다. Robolectric 는  `Shadow` 라 불리는 테스트 더블로 뷰의 생성, 리소스 로딩, 안드로이드 프레임워크의 다른 일부분을 시뮬레이션 하게 해준다.

Robolectric 시뮬레이터이므로 단순 단위 테스트를 대체하거나 호환성 테스트를 하는데 사용되어서는 안된다.
Robolectric 는 속도와 경우에 따라 충실도가 낮아 비용이 절감된다. 가장 좋은 UI 테스트 접근 방식은 Robelectric 및 계측 테스트 둘다 호환되도록 만들고 기능 또는 호환성을 테스트해야 하는 필요에 따라 테스트 실행 시기를 결정하는 것이다. Robelectric에서 Espresso 와 Compose 테스트를 모두 실행할 수 있다.


### 참조
---
[Use test doubles in Android](https://developer.android.com/training/testing/fundamentals/test-doubles)