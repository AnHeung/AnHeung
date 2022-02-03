---
layout: single
title : 안드로이드 테스트 코드 기본
---

안드로이드 스튜디오는 테스트 작업을 간단히 수행하도록 설계가 되있다. 기본적으로 JVM 단에서 동작하는
로컬테스트와 실제 에뮬레이터가 디바이스를 동작시켜 테스트 하는 계측 테스트로 나뉘는데 해당 디렉토리에
작성함으로 각자에 맞는 테스트를 진행할 수 있게된다.

### 로컬 단위 테스트(Local Test) vs 계측 테스트 (Instruments Test)

안드로이드 프로젝트를 만들고 나면 기본 main, test,  androidTest 라는 폴더로 source sets 이 구성된다.

`main` - 실제 소스를 작성하는 곳이다. 

`test` - 로컬 테스트를 작성하기 위한 곳이다. 오직 개발장비(pc, 노트북 등)의 JVM 에서만 동작하고
실제 에뮬레이터나 기기로 동작하지 않기 때문에 빠르다는 장점이 있다. 하지만 빠르다해도 충실도와 신뢰도는 다소 떨어질수 있다.

`androidTest` - 실제 기기나 에뮬레이터를 통해 동작하는 테스트다. 실제 기기로 동작하기 때문에 느리다.   
실제 테스트에 필요한 Context 에 접근할 권한을 제공받으며 이 테스트에서 개발자는 테스트 대상 앱을 테스트 코드에서 제어할 수 있다. 

신뢰도는 로컬 테스트에 비하면 아주 높고 만약 CI 서버에서 테스트 자동화가 이루어진다면 해당 테스트를 위한 기기나 에뮬레이터가 연결되있어야 한다. 


### AndroidX Test Library
---
만약 그냥 ViewModel 을 만들어 쓴다 가정하면 생성자에 어떠한 파라미터도 받지 않는다. 그런데 Application 의
`context` 를 생성자에서 필요로 하게 된다면 이럴경우 어떻게 주입하는게 맞을까? 

`AndroidX Test Library` 를 사용하면 테스트를 위한 안드로이드 컴포넌트 (Application, Activity) 들과 메서드를 제공해 준다. 로컬 테스트중 해당 컴포넌트를 호출할 부분이 필요하다면 사용해야한다.

`AndroidX Test Library` 를 사용하기 위해서는 3가지 절차가 있다.

1. AndroidX Test core , ext 라이브러리 추가
2. Robolectric Testing 라이브러리 추가
3. 사용하는 테스트 클래스에 @RunWith(AndroidJUnit::class) 추가


```gradle
testImplementation "androidx.test.ext:junit-ktx:$androidXTestExtKotlinRunnerVersion"
testImplementation "androidx.test:core-ktx:$androidXTestCoreVersion"
testImplementation "org.robolectric:robolectric:$robolectricVersion"
```

그러면 Robolectric 과 @RunWith(AndroidJUnit4::class) 는 왜 추가하는가?

#### Robolectric

AndroidX Test 는 시뮬레이트된 안드로이드 환경에서 테스트 전용 클래스와 메소드를 제공해준다.
그렇다고 실제 기기나 환경은 아니고 말 그대로 시뮬레이트 된 환경이므로 그런 환경을 실제로 만들어 주는건
Robolectric 이 할 일 이다. 예전에는 local test 에서 context 를 얻기 위해 Robolectric 을 사용했었다. 
하지만 이제 AndroidX Test 라이브러리 덕분에 직접사용하진 않게 되었다. 가장 좋은점은 예전에 따로 Robolectric 을 공부하고 사용법을 공부해야했던 부분들을 AndroidX Test 라이브러리 덕분에 많이 쉽게 적용할 수 있다는 것이다. Instrumented test 에서도 AndroidX Test Library 의 같은 메소드를 사용해서 context 를 얻는게 가능한데 그건 시뮬레이트된 환경의 context 가 아닌 실제 기기의 context 를 얻게 된다.

#### AndroidJUnit4::class

{% include image.html id="133IJVLuXDzUfg38YHxElFhPPIzCFSoqV"%}

Android4JUnit4 는 테스트 러너다. 테스트를 실행하는 주체로 junit 은 테스트 러너 없이는 테스트가 실행이 되지 않는다. runner 를 지정해주지 않으면 기본 runner 로만 실행되는데 여기서 `@RunWith` 를 함으로 러너를 교체할 수 있다. Android4JUnit 은 AndroidX Test 라이브러리가 로컬 테스트와 계측 테스트에서 서로 다르게 동작 할 수 있도록 도와준다. 위에 Robolectric 에서 언급한거 처럼 로컬에선 시뮬레이트 된 환경의 context 
계측 테스트에선 실제 기기의 context 를 제공해주는데 이것이 Android4JUnit 러너가 수행하는 역할이다. 
즉 이거 없이는 옳바르게 동작을 안할 수 있으므로 AndroidX Test 라이브러리를 쓸때면 꼭 Android4JUnit 를 사용하도록 하자.

### Rule
---

JUnit 에는 JUnit Rule 이라는것이 있다. 테스트를 작성하다가 보면 여러 테스트 클래스에서 테스트 사전작업,
직 후 작업이 동일 할 때가 있다. 예를 들면 코루틴에서는 `@Before` 에서 디스패처를 Main 으로 바꾸고
`@After` 에서 되돌려 놓는다. 이런 작업을 하나의 Rule 로 만들면 `@Before`, `@After` 때마다 자동으로 수행되어 보일러 플레이트 코드를 줄일 수 있게된다.  즉 동작 방식을 재정의 하거나 쉽게 추가하는것이다. 
`@get:Rule` annotation 을 붙여 사용한다. 
AndroidX test 라이브러리엔 `ActivityTestRule`, `ServiceTestRule` 등과 같은 유용한 rule 도 제공한다.

```java
 // ActivityTestRule
 @get:Rule
 val activityRule = ActivityTestRule(MyClass::class.java)

//InstantTaskExecutorRule
 @get:Rule
    var instantExecutorRule = InstantTaskExecutorRule()
```


위처럼 다양한 Rule 이 있으니 필요에 따라 적절한 Rule 인스턴스를 얻어 작업하면 된다.


### LiveData Test
---
LiveData 를 사용해 ViewModel 테스트를 한다고 가정해보자. 

```java
class TestViewModel: ViewModel() {
    private val _msg = MutableLiveData<String>()

    val msg: LiveData<String> = _myLiveData

    fun sendMsg() {
        _msg.postValue("value")
    }
}
```

viewModel 을 구성하다 보면 위처럼 Activity 나 fragment 에 어떤 이벤트를 날리기 위해 livedata 로 구성해서 
알림을 날릴 것이다. 근데 위의 코드를 보면 postValue 를 사용했는데 해당 메소드는 백그라운드 Thread 에서 일을 수행한다. 그 말은 해당 부분을 테스트 코드로 작성해 msg 객체가 sendMsg 함수후 값이 변했나 안변했나를 
체크하면 값이 비동기로 처리되기 때문에 테스트가 이미 끝나서 오류가 발생할 것이다. 이럴경우 사용하는 Rule 이
InstantTaskExecutorRule 이 있다. 해당 룰은 모든 아키텍처 구성요소 관련 백그라운드 작업을 동일한 Thread 에서 돌게한다. 즉 동기적으로 처리가 된다. 여기서 재밋는건 로컬 테스트에서 postValue 가 아닌 setValue 를 사용한다 한들 InstantTaskExecutorRule 을 적용하지 않으면 테스트는 실패한다. 

### Coroutine Test
---
코루틴 테스트를 한다고 가정해보자

```java
class TestViewModel : ViewModel (){
    var name : String = "hello"
    private set

    fun changeName(newName:String) = viewModelScope.launch{
        name = newName
    }
}
```

위의 코드는 activity 나 fragment 등에서 호출할 경우 Dispatcher.main 의 스코프를 그대로 받아
코루틴이 실행된다. 즉 해당 코드를 테스트 하기 위해 메인 스레드에서 호출하는 테스트를 하면 아래와 같다.

```java
class MyViewModelTest: ViewModel() {
   @Test
   fun changeNameTest() = runBlocking {
       val myViewModel = MyViewModel()
       val newName = "world"
       myViewModel.changeName(newName)
       Assert.assertEquals(myViewModel.name, newName)
   }
}
```

로컬 테스트에서는 에서는 실제 환경에서의  Android main looper 를 사용해선 안된다. 대신 테스트 환경에 맞는 TestCoroutineDispatcher 를 호출해서 사용해야 한다. `kotlinx-coroutines-test` 는 테스트 전용으로 만들어진 라이브러리다. gradle 에 추가해서 사용하도록 하자.

```gradle
testImplementation "org.jetbrains.kotlinx:kotlinx-coroutines-test:$version"
```

```java
@ExperimentalCoroutinesApi
class TestViewModelTest {
    private val testDispatcher = TestCoroutineDispatcher()

    @Before
    fun setup() {
        Dispatchers.setMain(testDispatcher)
    }

    @After
    fun tearDown() {
        //  원래의 상태로 되돌려 놓는다.
        Dispatchers.resetMain()
        // 테스트가 끝났으니 혹시 모를 실행중인 작업을 clean up 시켜준다.
        testDispatcher.cleanupTestCoroutines()
    }

    @Test
    fun testSomething() = runBlockingTest {
        ...
    }
}
```

만약 모든 테스트마다 before after 를 만들기 귀찮다면 Rule을 직접만들어서 활용할 수 있다.

```java
@ExperimentalCoroutinesApi
class MainCoroutineRule(
    val testDispatcher: TestCoroutineDispatcher = TestCoroutineDispatcher()
) : TestWatcher() {

    override fun starting(description: Description?) {
        super.starting(description)
        Dispatchers.setMain(testDispatcher)
    }

    override fun finished(description: Description?) {
        super.finished(description)
        Dispatchers.resetMain()
        testDispatcher.cleanupTestCoroutines()
    }
}

@ExperimentalCoroutinesApi
fun MainCoroutineRule.runBlockingTest(block: suspend () -> Unit) =
    runTest { 
        block()
    }
```

위처럼 작성하고 필요한 부분마다 호출해서 사용하면 된다.

```java
@get:Rule
val mainCoroutineRule = MainCoroutineRule()
```

위의 코드에서 runTest 함수를 호출했는데 실제 runBlocking 과  큰 차이점은  delay 같은 함수가 있을경우 즉시 실행이 된다는점이다. 덕분에 테스트 시간을 절약할 수 있다는 장점이 있다.

