---
layout : single
title : Hilt 로 의존성 주입하기 
---
Hilt 는 직접 의존성 주입으로 생기는 보일러 플레이트를 줄여주는 안드로이드 의존성 주입 라이브러리다.

Hilt 는 어플리케이션의 모든 안드로이드 클래스에 컨테이너를 제공하고 생명주기를 자동으로 관리함으로 DI 의 표준적인 방법을 제공한다. Hilt 는 Dagger 기반으로 만들어 져서 컴파일 타임 정확성, 런타임 성능, 확장성, Dagger 가 제공하는 안드로이드 스튜디오 지원 등의 이점이 있다. 

### 의존성 설정
---
`hilt-android-gradle-plugin` 플러그인을 루트 `build.gradle` 파일에 추가

```gradle
buildscript {
    ...
    dependencies {
        ...
        classpath("com.google.dagger:hilt-android-gradle-plugin:2.38.1")
    }
}
```

그러고 앱의 `build.gradle` 파일에 gradle 플러그인 적용하기

```gradle
plugins {
    kotlin("kapt")
    id("dagger.hilt.android.plugin")
}

android {
    ...
}

dependencies {
    implementation("com.google.dagger:hilt-android:2.38.1")
    kapt("com.google.dagger:hilt-android-compiler:2.38.1")
}

// Allow references to generated code
kapt {
 correctErrorTypes = true
}
```

> [databinding](https://developer.android.com/topic/libraries/data-binding) 과 Hilt 둘다 사용하는 프로젝트의 경우 안드로이드 스튜디오 4.0 이상이 필요하다.

Hilt 는 Java8 기능을 사용하므로 app 의 `build.gradle` 파일에 Java8 이 가능하게 추가해야한다.

```gradle
android {
    ...
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_1_8
        targetCompatibility = JavaVersion.VERSION_1_8
    }
}
```

### Hilt Application 클래스
---
Hilt 를 사용하는 모든 앱들은 `@HiltAndroidApp` 주석을 Application 클래스에 포함시켜야 한다.
Application 클래스는 앱이 실행되면서 종료되기전까지 계속 유지되는 클래스이기 때문에 그 동안에 의존성을 
제공하는 역할을 해야하기 때문이다.

`@HiltAndroidApp` 는 Hilt 의 코드생성의 시작점이고 Application 레벨에서의 의존성 컨테이너를 Application 에 제공하는 기본클래스를 포함하고 있다.

```java
@HiltAndroidApp
class ExampleApplication : Application() { ... }
```

이렇게 생성되는 Hilt 컴포넌트는 Application 객체의 생명주기에 따르고 의존성을 제공한다.

### 각 안드로이드 클래스에 의존성 주입하기
---
한번 Application 클래스에 Hilt 가 설정하고 Application 단계의 컴포넌트를 사용가능해지면 Hilt는 다른 `@AndroidEntryPoint` 주석을 가진 안드로이드 클래스에 의존성을 제공한다.

```java
@AndroidEntryPoint
class ExampleActivity : AppCompatActivity() { ... }
```

아래의 클래스에 @AndroidEntryPoint 을 붙이면 Hilt 해당 클래스에 의존성을 제공해 줄 수 있는 컴포넌트를 생성해준다.

- Application (@HiltAndroidApp 사용)
- ViewModel (@HiltViewModel 사용)
- Activity
- Fragment
- View
- Service
- BroadcastReceiver 

만약 @AndroidEntryPoint 를 안드로이드 클래스에 적용했으면 이 안드로이드 클래스를 의존하는 다른 안드로이드 클래스 또한 @AndroidEntryPoint 주석을 붙여야한다. 예를 들면 fragment 에 붙였으면 해당 프래그먼트를 사용하는 다른 activity 들에도 적용해야 하는것이다.

@AndroidEntryPoint 를 붙이면 안드로이드 클래스에 개별적으로 Hilt 컴포넌트가 생긴다. 이 컴포넌트는 그들의
각각의 부모부터 의존성을 받을 수 있다. 즉 [Component Hierarchy](/android/hilt/dependency-injection-with-hilt/#컴포넌트-계층-component-hierarchy) 를 따른다. 즉 Component Hierarchy 에 따라 자식은 부모로 컴포넌트로 부터 의존성을 주입받을 수 있다.

### Injection (주입)
---
Hilt 는 의존성 그래프를 만들어서 필요한 곳에 적절히 의존성을 제공해주는 라이브러리다. 그래서 의존성이 필요한 곳이 있다면 Hilt 에게 주입받고 싶다라고 알려야 Hilt 가 인지해서 해당부분에 의존성을 주입해 줄 수 있다. 

##### 1. Field Injection (필드 주입)

필드에 의존성을 주입받고 싶으면 @Inject 주석을 사용하면 Hilt 컴포넌트로 부터 의존성을 얻을 수 있다.

```java
@AndroidEntryPoint
class ExampleActivity : AppCompatActivity() {

  @Inject lateinit var analytics: AnalyticsAdapter
  ...
}
```
> Hilt 로 부터 의존성 주입을 받는 필드 객체들은 private 으로 선언할 수 없다. private 으로 선언시 컴파일에서 에러가 발생한다. 내부적으로 set 을 해줘야 하는데 private 으로 선언하면 해당 변수에 접근할 수 없기 때문이다.

필드 주입을 수행하기 위해 Hilt 는 거기에 해당하는 컴포넌트로 부터 필요한 의존성 인스턴스를 제공하는 방법을 
알 필요가 있다. 바인딩은 의존성 Type 의 인스턴스로써 제공하는데 필요한 정보를 포함하고 있다.  


##### 2. Constructor Injection (생성자 주입)

Hilt 에 바인딩 정보를 제공하는 한가지 방법으로 생성자 주입이 있다. `@Inject` 주석을 생성자에 쓰고 
거기에 제공할 인스턴스를 Hilt 에게 알려준다.

```java

interface AnalyticsService {
  fun analyticsMethods()
}

class AnalyticsAdapter @Inject constructor(
  private val service: AnalyticsService
) { ... }
```

해당 주석이 붙은 클래스의 생성자 파라미터는 클래스의 의존성들이다. 예를 들어 AnalyicsAdapter 가 
AnalyticsService 라는 의존성을 가진다 할때 Hilt 는 AnalyticsService 인스턴스를 제공하는 방법을 반드시 알아야 한다는 것이다. 

> 빌드할때 Hilt 는 Android 클래스에서 Dagger 컴포넌트를 만든다. 그리고 Dagger 는 코드에 따라 아래의 단계를 수행한다.

- 빌드하고 의존성 그래프를 검증하여 충족되지 않은 종속성과 종속성 주기(lifecycle) 이 없는지 확인한다. 
- 실제 객체 및 의존성을 생성하기 위해 런타임에서 사용하는 클래스를 생성한다.

생성자 주입시 주의할 점이 있는데 생성자 주입시 안되는 타입이 2가지가 있다는 점이다. 
- Interface
- 외부 라이브러리 

인터페이스를 예시로 들자면 만약 A,B 클래스 2개의 클래스가 있고 A 클래스는 B 클래스에 대한 의존성을 가지고 특정 Activity 에서 필드로 A 클래스를 참조한다 가정해보자 

```java
class AClass @Inject constructor(private val bClass : BClass){
  fun doSomethingForA(): String = bClass.test()
}

class BClass @Inject constructor(){
  fun test():String = "test"
}

@AndroidEntryPoint
class SomeActivity : AppCompatActivity(){

  @Inject
  val aClass : AClass 

  override fun onCreate(savedInstanceState :Bundle ?){
    aClass.doSomethingForA()
  }
}

```

이럴 경우 필드 주입을 받아 사용할 수 있지만 여기서 조금 수정해서 어떤 인터페이스가 있고 위에 B 클래스가 
인터페이스를 구현하는 클래스라고 가정해보자.

```java
interface TestInterface{
  fun doTest():String
}

class AClass @Inject constructor(private val bClass : TestInterface){
  fun doSomethingForA(): String = bClass.test()
}

class BClass @Inject constructor(): TestInterface{
  override fun doTest():String = "test"
}

```

이럴경우 AClass 에서는 정상적으로 주입되지 않고 에러가 나게 된다. interface 가 구현화된 Type 의 객체를 어떻게 생성해야 하는지 알 수 없기 때문이다.

### Hilt Modules
---
Hilt 모듈은 `@Module` 주석이 붙은 클래스로 Dagger 모듈과 유사하게 Hilt 에게 확실한 Type의 인스턴스를 제공하는 방법을 알린다. Dagger 모듈과 다른점이라면 `@InstallIn` 주석을 Hilt 모듈에 붙여서 어떤 안드로이드 클래스에 각 모듈을 사용하거나 설치 할걸지 Hilt 에 알려야 한다는것이다. 

Hilt 모듈을 통해 제공하는 의존성은 Hilt 모듈이 설치된 안드로이드 클래스와 관련된 모든 생성된 컴포넌트에서 이용가능하다. 

특히 interface 나 외부 라이브러리 처럼 Hilt 가 객체를 어떻게 생성해야할지 모를 경우 꼭 필요하다.

##### 1. @Binds 로 interface 인스턴스 주입

AnalyticsService 예제를 생각해보면  AnalyticsService 는 인터페이스이기 때문에 생성자 주입이 안된다.  

대신 Hilt 모듈안의 @Binds 주석이 붙은 추상클래스를 만듬으로써 바인딩 정보를 Hilt 에게 제공할 수 있다. @Binds 주석은 인터페이스 인스턴스 제공이 필요할때 어느 구현체를 사용할 건지 Hilt 에게 알려준다.  

@Binds 주석 함수는 Hilt 에게 아래의 정보를 제공한다.

- 함수의 반환 Type 은 함수의 인터페이스 Type 을 Hilt 에게 전달한다.
- 함수의 파라미터는 실제 제공할 구현체를 Hilt 에게 전달한다.

```java
interface AnalyticsService {
  fun analyticsMethods()
}

//Hilt 가 AnalyticsServiceImpl 의 인스턴스의 제공법을 알아야 하므로 생성자 주입을 한다.
class AnalyticsServiceImpl @Inject constructor(
  ...
) : AnalyticsService { ... }

@Module
@InstallIn(ActivityComponent::class)
abstract class AnalyticsModule {

  @Binds
  abstract fun bindAnalyticsService(
    analyticsServiceImpl: AnalyticsServiceImpl
  ): AnalyticsService
}
```

Hilt 모듈인 AnalyticsModule 은 `@InstallIn(ActivityComponent::class)` 주석을 붙인다. ExampleActivity 에 의존성 주입을 하기를 원하기 때문이다. 이 주석은 `AnalyticsModule` 안의 모든 의존성은 앱의 모든 액티비티에서 사용가능하다라는 뜻이다. 

##### 2. @Provide 로 인스턴스 주입하기

위에서 설명하듯 인터페이스만이 유일한 생성자 의존성 주입이 안되는건 아니다. 내가 가지고 있지 않는 클래스(retrofit,OkHttpClient , Room 같은 외부 라이브러리) 또는 builder 패턴을 사용해 만드는 인스턴스 또한 생성자 주입이 안된다.  

이전의 예제를 생각하면 `AnalyticsService` 클래스를 직접 가지고 있진 않앗다. 
Hilt 모듈안에 `@Provides` 주석이 붙은 함수를 만듬으로써 Hilt 에게 인스턴스를 제공하는 방법을 알릴 수 있다.  

@Provides 주석이 붙은 함수는 Hilt 에게 아래의 정보를 제공한다.

- 함수의 반환 Type 은 함수가 제공하는 인스턴스가 어떤 Type 인지 알려준다.
- 함수의 파라미터는 해당하는 Type 의 의존성을 Hilt 에게 알려준다.  
  (간단하게 함수 내부에서 제작할때 인스턴스가 필요로 하는 값들을 파라미터에 넣으면 된다.)  
- 함수의 바디는 Hilt 에게 해당하는 Type 의 인스턴스를 제공하는 방법을 알려준다. Hilt 는 해당 Type 의 인스턴스가 제공받기를 원하면 매번 함수를 실행한다.  
```java
@Module
@InstallIn(ActivityComponent::class)
object AnalyticsModule {

  @Provides
  fun provideAnalyticsService(
    // 반환 Type 을 만들시 필요한 값들을 위한 파라미터들 
  ): AnalyticsService {
      return Retrofit.Builder()
               .baseUrl("https://example.com")
               .build()
               .create(AnalyticsService::class.java)
    }
}
```

##### 3. 같은 Type 에 대한 멀티 바인딩 제공하기

같은 Type 의 의존성을 다른 구현체로 (예를 들어 같은 인터페이스에서 나온 각기 다른 구현체를 주입하는 경우나 반환 Type 이 String 인 함수가 2개인데 각각 분리해서 주입해야하는 경우 등) 주입하는 경우 한정자 (qualifier) 를  정의해서 Hilt 에게 멀티 바인딩 을 제공할 수 있다.  

한정자는 멀티 바인딩 정의를 가질때 지정된 바인딩을 식별하기 위해 사용하는 주석이다. 즉 한정자 주석이 붙어있으면 같은 반환 Type 이라 할지라도 구별을 할 수 있다.

예를 들어 AnalyticsService 가 통신 중간에 가로채는 처리가 필요할 때 보통 OkHttpClient 객체를 [interceptor](https://square.github.io/okhttp/interceptors/) 를 적용해서 사용하곤 한다. 만약 AnalyticsService 외에 다른 서비스를 구현해서 다른 interceptor 처리가 필요할 경우가 있다고 가정하면 Hilt 에게 두 다른 OkHttpClient 구현체를 제공할 방법을 알려줘야 한다.  

일단 @Binds 나 @Provides 를 사용할 메소드에서 쓸 한정자를 정의하자 

```java
// 인증 Interceptor 한정자
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class AuthInterceptorOkHttpClient
// 기타 Interceptor 한정자
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class OtherInterceptorOkHttpClient


@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

  @AuthInterceptorOkHttpClient
  @Provides
  fun provideAuthInterceptorOkHttpClient(
    authInterceptor: AuthInterceptor
  ): OkHttpClient {
      return OkHttpClient.Builder()
               .addInterceptor(authInterceptor)
               .build()
  }

  @OtherInterceptorOkHttpClient
  @Provides
  fun provideOtherInterceptorOkHttpClient(
    otherInterceptor: OtherInterceptor
  ): OkHttpClient {
      return OkHttpClient.Builder()
               .addInterceptor(otherInterceptor)
               .build()
  }
}
```

두 함수다 같은 Type 을 반환하지만 한정자는 두개의 바인딩으로 라벨링을 했기때문에 Hilt 는 각각 다르게 주입할 수 있다.

```java
// 다른 클래스의 의존성으로 사용
@Module
@InstallIn(ActivityComponent::class)
object AnalyticsModule {

  @Provides
  fun provideAnalyticsService(
    @AuthInterceptorOkHttpClient okHttpClient: OkHttpClient
  ): AnalyticsService {
      return Retrofit.Builder()
               .baseUrl("https://example.com")
               .client(okHttpClient)
               .build()
               .create(AnalyticsService::class.java)
  }
}

// 생성자 의존성으로써 사용
class ExampleServiceImpl @Inject constructor(
  @AuthInterceptorOkHttpClient private val okHttpClient: OkHttpClient
) : ...

// 필드 주입으로 사용
@AndroidEntryPoint
class ExampleActivity: AppCompatActivity() {

  @AuthInterceptorOkHttpClient
  @Inject lateinit var okHttpClient: OkHttpClient
}
```

> 한정자를 사용하지 않을 경우 Hilt 가 잘못된 객체를 주입하게 되서 에러가 발생할 수 있다.

##### 4. Hilt 에 있는 사전정의된 한정자

Hilt 는 사전 정의된 한정자를 제공한다. 예를 들어 Application 이나 Activity 로 부터 Context 클래스가 필요 할 수 있다. 이럴 경우 Hilt 는 @ApplicationContext 와 @ActivityContext 한정자를 제공해준다.

`AnalyticsAdapter` 클래스가 Activity 의 context 가 필요하다고 가정하면 아래 코드처럼 주입해 줄 수 있다.

```java
class AnalyticsAdapter @Inject constructor(
    @ActivityContext private val context: Context,
    private val service: AnalyticsService
) { ... }
```

다른 Hilt 에서 사용 가능한 사전정의된 바인딩들을 보려면 [컴포넌트 기본 바인딩](/android/hilt/dependency-injection-with-hilt/#컴포넌트-기본-바인딩) 를 참조하세요.


### 안드로이드 클래스를 위해 생성된 클래스
--- 
필드 주입을 수행할 수 있는 각각의 안드로이드 클래스는 `@InstallIn` 주석 안의 참조할 수 있는 Hilt 컴포넌트와 관련 있다. 각 Hilt 컴포넌트는 해당하는 안드로이드 클래스의 바인딩 주입하는 역할을 가진다.  

위의 예제에서는 Hilt 모듈안에서 `ActivityComponent` 사용을 설명햇다. Hilt 는 아래의 컴포넌트들을 제공한다. 

{% include image.html id="16Ci0yPt2xfh_9zucltdV9dugaKaluUZ8" %}

### 컴포넌트 생명주기
---
Hilt 는 생성된 클래스의 인스턴스를 아래의 해당하는 안드로이드 클래스의 생명주기에 맞게 자동적으로 만들고 없앤다. 

{% include image.html id="16DnOWaBIURa2WVowdMQ6E3D4TMux2AnM" %}

### 컴포넌트 Scope
---
기본적으로 Hilt 에서 바인딩 된 값들은 모두 범위가 지정되있지 않다. 즉 앱이 바인딩을 요청하고 Hilt 는 필요한 Type 의 인스턴스를 매순간 만든다는 뜻이다.  

예를 들어 `AnalyticsAdapter` 의 인스턴스를 필드 주입이나 생성자 주입 등의 방법으로 제공할때 매번 새 인스턴스를 제공한다는 뜻이다.  

그러나 Hilt 는 특정 컴포넌트에 바인딩 범위도 지정 할 수 있다. Hilt 가 지정된 범위의 바인딩을 일단 한번 만들면 해당 바인딩에 대한 모든 요청에 같은 인스턴스를 공유한다.  

아래의 표는 각 생성된 컴포넌트의 scope 주석 리스트다.  

{% include image.html id="16EZ4YS0oAqDfg36glcrX1coXiBIEvms0" %}

위의 예제에서 `AnalyticsAdapter` 를 `@ActivityScoped` 를 사용해 `ActivityComponent`로 범위를 지정했다면 Hilt 는 해당하는 Activity 생명주기동안에 같은 `AnalyticsAdapter` 인스턴스를 제공하게 된다.
만약 Activity 에서 주입될 객체를 생성하는 클래스에 @FragmentScoped 를 붙이면 에러가 발생하게 된다.

```java
@ActivityScoped
class AnalyticsAdapter @Inject constructor(
  private val service: AnalyticsService
) { ... }
```
> 컴포넌트에 범위지정 바인딩은 컴포넌트가 사라질때까지 메모리에 상주하기 때문에 꽤 비용이 많이 들 수 있다. 그러므로 앱에서 범위지정 바인딩 은 최소한으로 사용해야 한다.   

`AnalyticsService` 가 Activity 뿐만 아니라 앱의 모든 부분에서 매번 동일한 인스턴스를 필요로 하는 내부상태를 가진다 가정해 보자 이럴 경우 AnalyticsService 의 범위는 SingletonComponent 로 하는게 적절하다. 그러면 매번 인스턴스가 필요할 때 마다 동일한 AnalyticsService 인스턴스를 제공할 것이다.

아래의 예제는 Hilt 모듈에서 컴포넌트 바인딩 범위를 지정하는지 설명한다. 바인딩 된 범위는 설치된 컴포넌트의 범위와도 일치해야 한다.  

```java
// If AnalyticsService is an interface.
@Module
@InstallIn(SingletonComponent::class)
abstract class AnalyticsModule {

  @Singleton
  @Binds
  abstract fun bindAnalyticsService(
    analyticsServiceImpl: AnalyticsServiceImpl
  ): AnalyticsService
}

// If you don't own AnalyticsService.
@Module
@InstallIn(SingletonComponent::class)
object AnalyticsModule {

  @Singleton
  @Provides
  fun provideAnalyticsService(): AnalyticsService {
      return Retrofit.Builder()
               .baseUrl("https://example.com")
               .build()
               .create(AnalyticsService::class.java)
  }
}
```

예제 대로면 AnalyticsService 에 Singleton 을 선언했기때문에 InstallIn 에 `ActivityComponent` 대신  `SingletonComponent` 를 선언해야 한다.  

### 컴포넌트 계층 (Component Hierarchy)
---
컴포넌트에 모듈 설치를 하면 해당 바인딩이 해당 컴포넌트 또는 컴포넌트 요소 계층 구조에서 자식 구성 요소의 다른 바인딩 의 종속성으로 접근이 가능하다. 자식들이 위쪽 부모쪽의 의존성에 접근이 가능하단 뜻이다. 

{% include image.html id="16JZ9rVbQct_QBDpA5XeXBmkVKxnL-UOX" %}

### 컴포넌트 기본 바인딩 
---
각 Hilt 컴포넌트에는 Hilt가 사용자 정의 바인딩에 종속성으로 주입할 수 있는 기본 바인딩 세트가 함께 제공됩니다. 주의할 점은 이런 바인딩은 일반적인 Activity 와 Fragment Type 에만 해당하고 특정 하위 클래스 등에는 적용되지 않는다. Hilt 는 모든 Activity 들에 주입하기 위해 단일 Activity 컴포넌트를 정의하여 사용한다. 각 Activity 는 이 컴포넌트의 다른 인스턴스를 가진다. 

{% include image.html id="16LaM8PlEkF-XnXAZXThYfOE7zlqb8AJC" %}


### Hilt 와 Dagger
---
Hilt 는 Dagger 위에서 만들어진 의존성 주입 라이브러리로 안드로이드 앱안의 Dagger 를 통합시키는 표준적인 방법을 제공한다. Dagger 와 관련해 Hilt 는 다음의 목표를 지닌다.  

- Dagger 관련 인프라를 쉽게 제공.

- 앱 간의 설정, 가독성 및 코드 공유를 용이하게 하기 위해 표준 컴포넌트 및 Scope 세트를 생성합니다.

- 테스트, 디버그 또는 릴리스와 같은 다양한 빌드 유형에 각각 다른 바인딩 객체를 쉽게 공급해줍니다.  

안드로이드 OS 는 많은 자체 프레임워크 클래스를 시작하고 안드로이드 App 에서 Dagger 를 사용하면 상당한 양의 보일러 플레이트를 작성해야 하기 때문에 Hilt 는 안드로이드 앱안의 Dagger 코드량을 줄여준다. Hilt 는 자동적으로 생성되며 아래의 것들을 공급해준다.  

- Dagger 의 안드로이드 프레임워크 통합 컴포넌트를 제공한다. 이걸 제공해주지 않을경우 직접 작성해야 한다. 

- Hilt 가 자동 생성한 컴포넌트에 사용할 Scope 주석을 제공한다.

- Activity 나 Fragment 를 나타내기 위한 사전정의된 바인딩

- @ApplicationContext 나 @ActivityContext 같은 사전정의된 한정자

Dagger 나 Hilt 는 같은 코드베이스에서 공존한다. 하지만 가장 좋은 방법은 안드로이드에서 사용되는 Dagger  를  Hilt 를 통해 관리하는 것이다. 

### 출저
---
[Dependency injection with Hilt](https://developer.android.com/training/dependency-injection/hilt-android#component-hierarchy)
