---
layout : single
title : Hilt 테스트 가이드
---
Hilt 같은 DI 프레임워크를 사용함으로 얻는 장점은 테스트 코드를 작성하기 쉽다는 점이다.

### 단위 테스트
---
생성자 주입을 사용하는 클래스를 테스트할때는 클래스 생성을 위해 Hilt 를 단위테스트에서 사용할 필요는 없다. 대신 생성자에 주석을 달지 않는 경우 처럼 직접 Fake 나 Mock 같은 의존성을 전달하여 생성자를 호출할 수 있다.

```java
@ActivityScoped
class AnalyticsAdapter @Inject constructor(
  private val service: AnalyticsService
) { ... }

class AnalyticsAdapterTest {

  @Test
  fun `Happy path`() {
    // You don't need Hilt to create an instance of AnalyticsAdapter.
    // You can pass a fake or mock AnalyticsService.
    val adapter = AnalyticsAdapter(fakeAnalyticsService)
    assertEquals(...)
  }
}
```

### End-To-End 테스트
---
통합 테스트를 위해 Hilt 는 프로덕션 코드처럼 의존성을 주입해주어야 한다. Hilt 가 자동으로 각각의 테스트를 위한 컴포넌트 세트를 만들어주기 때문에 따로 유지관리는 필요없다.

#### 테스트 의존성 추가
---
Hilt 를 테스트에 쓰기 위해 `hilt-android-testing` 의존성을 프로젝트에 주입하자.

```gradle
dependencies {
    // For Robolectric tests.
    testImplementation("com.google.dagger:hilt-android-testing:2.38.1")
    // ...with Kotlin.
    kaptTest("com.google.dagger:hilt-android-compiler:2.38.1")
    // ...with Java.
    testAnnotationProcessor("com.google.dagger:hilt-android-compiler:2.38.1")


    // For instrumented tests.
    androidTestImplementation("com.google.dagger:hilt-android-testing:2.38.1")
    // ...with Kotlin.
    kaptAndroidTest("com.google.dagger:hilt-android-compiler:2.38.1")
    // ...with Java.
    androidTestAnnotationProcessor("com.google.dagger:hilt-android-compiler:2.38.1")
}
```

### UI 테스트 설정
---
UI 테스트를 진행하려면 `@HiltAndroidTest` 주석을 사용해야한다. 해당 주석은 각각의 테스트를 위한 Hilt 컴포넌트를 생성해준다.  

또한 HiltAndroidRule 을 추가해야한다. 이 주석은 컴포넌트의 상태관리와 테스트의 주입을 수행한다.

```java
@HiltAndroidTest
class SettingsActivityTest {

  @get:Rule
  var hiltRule = HiltAndroidRule(this)

  // UI tests here.
}
```

> 만약 위의 Rule 말고도 다른 Rule 도 가지고 있다면 [계측 테스트의 여러 Test Rule 객체](/android/hilt/hilt-test-guide/#계측-테스트의-여러-test-rule-객체) 문서를 확인하세요.


### 테스트 Application 
---
계측 테스트를 수행한다면 Hilt 가 지원하는 Application 을 사용해서 해야한다. Hilt 라이브러리는 테스트를 위한 `HiltTestApplication` 클래스를 제공한다. 만약 다른 Application 클래스를 쓰고 싶다면 [Custom application for tests](https://developer.android.com/training/dependency-injection/hilt-testing#custom-application) 해당 문서를 참조하세요.

계측테스트나 Rolectric 테스트를 위해서 테스트 Application 를 세팅해둬야한다. 아래의 지시사항은 Hilt 만을 특정한것은 아니지만 일반적으로 테스트를 위한 사용자 정의 Application 에 대한 가이드 라인이다.

#### 계측테스트에서 테스트 Application 설정

계측 테스트에서 Hilt Test Application 을 사용하고 싶으면 새로운 테스트 러너를 구성할 필요가 있다. 아래의 단계를 수행하시오.  

1. AndroidJUnitRunner 클래스를 확장한 사용자 정의 클래스 생성
2. `newApplication` 함수 override 해서 생성할 Hilt test application 이름 전달 

```java
// A custom runner to set up the instrumented application class for tests.
class CustomTestRunner : AndroidJUnitRunner() {

    override fun newApplication(cl: ClassLoader?, name: String?, context: Context?): Application {
        return super.newApplication(cl, HiltTestApplication::class.java.name, context)
    }
}
```

그러고 난후 Gradle Config 에 기본 테스트 러너를 해당 사용자 정의 클래스로 수정

```gradle
android {
    defaultConfig {
        // Replace com.example.android.dagger with your class path.
        testInstrumentationRunner = "com.example.android.dagger.CustomTestRunner"
    }
}
```

#### Robolectric 테스트에서 테스트 Application 설정

Robolectric 을 사용해 UI 영역을 테스트중이면 `robolectric.properties` 파일에 어느 Application 클래스를 쓸건지 명시해야 한다. 
```
application = dagger.hilt.android.testing.HiltTestApplication
```
이 방법이 싫다면 각각 개별의 테스트에 Robolectric 의 @Config 주석에 사용할 Application 클래스를 명시해야 한다.

```java
@HiltAndroidTest
@Config(application = HiltTestApplication::class)
class SettingsActivityTest {

  @get:Rule
  var hiltRule = HiltAndroidRule(this)

  // Robolectric tests here.
}
```

### 테스트 기능들
---
일단 Hilt 테스트에서 세팅되면 몇몇 기능들을 사용해 테스트 프로세스를 사용자 정의 할 수 있다. 테스트에 Type 을 주입하려면 필드에 @Inject 를 사용해야 한다. 필드값을 채우도록 Hilt 에 지시하려면 hiltRule.inject() 함수를 호출하세요.

```java
@HiltAndroidTest
class SettingsActivityTest {

  @get:Rule
  var hiltRule = HiltAndroidRule(this)

  @Inject
  lateinit var analyticsAdapter: AnalyticsAdapter

  @Before
  fun init() {
    hiltRule.inject()
  }

  @Test
  fun `happy path`() {
    // Can already use analyticsAdapter here.
  }
}
```

#### 바인딩 교체

Fake 나 Mock 인스턴스의 의존성이 필요하다면 Hilt 에게 프로덕션에서 사용하는 바인딩 코드 말고 다른걸 써야한다고 알려야 한다. 바인딩 교체를 위해서 테스트에서 사용할 바인딩을 포함할 테스트 모듈로 교체가 필요하다. (app 모듈 -> 테스트 모듈) 예를 들어 테스트에서 AnalyticsService 를 프로덕션 코드에서 바인딩을 선언했다고 가정해보자. 

```java
@Module
@InstallIn(SingletonComponent::class)
abstract class AnalyticsModule {

  @Singleton
  @Binds
  abstract fun bindAnalyticsService(
    analyticsServiceImpl: AnalyticsServiceImpl
  ): AnalyticsService
}
```
테스트의 AnalyticsService 바인딩을 교체하려면 `test` 나 `androidTest` 폴더에 새 Hilt 모듈을 생성하고 
Fake 의존성을 만들어 `@TestInstallIn` 주석을 붙여야한다. 해당 폴더안의 테스트들은 이제 Fake 의존성이 주입된다. 

```java
@Module
@TestInstallIn(
    components = [SingletonComponent::class],
    replaces = [AnalyticsModule::class]
)
abstract class FakeAnalyticsModule {

  @Singleton
  @Binds
  abstract fun bindAnalyticsService(
    fakeAnalyticsService: FakeAnalyticsService
  ): AnalyticsService
}
```

#### 단일 테스트의 바인딩 교체

모든 테스트의 바인딩 대신 단일 테스트의 바인딩을 교체하려면 `@UninstallModules` 주석을 사용해 Hilt 모듈을 제거하고 테스트안에 새 테스트 모듈을 만들어야한다.  

```java
@UninstallModules(AnalyticsModule::class)
@HiltAndroidTest
class SettingsActivityTest { ... }
```

아래의 AnalyticsService 예제는 Hilt 에게 `@UninstallModules` 를 사용해 프로덕션 모듈을 무시하라고 말한다. 그 뒤에 테스트 바인딩을 정의해서 새 모듈을 만든다.

```java
@UninstallModules(AnalyticsModule::class)
@HiltAndroidTest
class SettingsActivityTest {

  @Module
  @InstallIn(SingletonComponent::class)
  abstract class TestModule {

    @Singleton
    @Binds
    abstract fun bindAnalyticsService(
      fakeAnalyticsService: FakeAnalyticsService
    ): AnalyticsService
  }

  ...
}
```
이것은 단일 테스트에 대한 바인딩만 대체한다. 모든 테스트 클래스에 대한 바인딩 교체를 원하면 위에서 설명한
`@TestInstallIn` 주석을 사용해야한다. 가능하면 `@TestInstallIn` 을 사용하는게 좋다.

> `@InstallIn` 주석이 없는 모듈은 삭제할 수 없다. 해당 작업 수행시 컴파일 에러가 발생한다. 또한 `@UninstallModules` 모듈은 `@InstallIn` 모듈만 삭제할 수 있지 `@TestInstallIn` 모듈은 해당사항이 아니다. 이 또한 컴파일 에러가 발생한다. `@UninstallModules` 을 사용해 Hilt 가 새 컴포넌트를 만드는건 
단위테스트 빌드 시간이 상당히 걸린다. 왠만하면 필요시에만 사용하고 `@TestInstallIn` 를 테스트시 사용하길 권한다. 

### 새로운 값 바인딩
---
`@BindValue` 주석을 사용해 테스트에서 Hilt 의존성 그래프에 쉽게 필드 바인딩을 할 수 있다. 
`@BindValue` 로 필드에 주석을 추가하면 한정자와 함께 사용한 필드 전체를 바인딩 할 수 있다.

```java
@UninstallModules(AnalyticsModule::class)
@HiltAndroidTest
class SettingsActivityTest {

  @BindValue @JvmField
  val analyticsService: AnalyticsService = FakeAnalyticsService()

  ...
}
```

이걸 통해 테스트에서 바인딩 교체 및 바인딩 참조를 동시에 할 수 있다.  
`@BindValue` 는 한정자와 다른 테스트 주석과 함께 동작한다. 예를 들어 테스트 라이브러리로 Mockito 를 
사용한다고 하면 Robolectric 테스트를 아래 처럼 진행할 것이다. 

```java
class SettingsActivityTest {
  ...

  @BindValue @ExampleQualifier @Mock
  lateinit var qualifiedVariable: ExampleCustomType

  // Robolectric tests here
}
```

만약 [다중 결합](https://dagger.dev/dev-guide/multibindings) 을 사용하고 싶으면 `@BindValueIntoSet` 와 `@BindValueIntoMap` 주석을 `@BindValue` 자리에 넣으면 된다. 

### 특수 케이스
---
Hilt 는 일반적이지 않은 경우에 대해서도 기능을 지원한다.  

#### 테스트를 위한 사용자 정의 Application

만약 HiltTestApplication 를 다른 Application 의 확장이 필요해서 사용하지 않는 다면 새 클래스나 인터페이스에 `@CustomTestApplication` 주석을 붙여 사용할 수 있다.   
`@CustomTestApplication` 는 매개변수로 전달한 Application 을 확장하는 Hilt 로 테스트할 준비가 된 Application 클래스를 생성한다. 

```java
@CustomTestApplication(BaseApplication::class)
interface HiltTestApplication
```
위의 예제에서 Hilt 는 BaseApplication 클래스를 확장하는 `HiltTestApplication_Application` 라는 Application 클래스를 생성한다. 생성된 이름은 클래스명_Application 이다. 

> HiltTestApplication_Application 는 런타임 시 생성하는 코드이므로 IDE 는 테스트가 실행될때 까지 빨간색으로 표시할 수 있다.

#### 계측 테스트의 여러 Test Rule 객체

만약 다른 `TestRule` 객체를 가지고 있다면 함께 동작을 보장하게 하는 다양한 방법이 있다.
아래와 처럼 rule 들을 체인으로 연결해서 함께 Wrap 할수 있다. 

```java
@HiltAndroidTest
class SettingsActivityTest {

  @get:Rule
  var rule = RuleChain.outerRule(HiltAndroidRule(this)).
        around(SettingsActivityTestRule(...))

  // UI tests here.
}
```

그밖에도 `@Rule` 주석의 order 속성을 사용해서 `HiltAndroidRule` 이 먼저 실행되게 하면서 동시에 두가지 룰을 동시에 사용할 수도 있다. 단 JUnit 4.13 이상 버전에서만 동작한다.

```java
@HiltAndroidTest
class SettingsActivityTest {

  @get:Rule(order = 0)
  var HiltAndroidRule = HiltAndroidRule(this)

  @get:Rule(order = 1)
  var settingsActivityTestRule = SettingsActivityTestRule()

  // UI tests here.
}
```



### 출저
---
[https://developer.android.com/training/dependency-injection/hilt-testing](https://developer.android.com/training/dependency-injection/hilt-testing)