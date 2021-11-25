---
layout : single
title : Koin 정리
---

### DI 란 무엇인가 ?
---

우리가 A ,B 두개의 클래스가 있다고 가정해보자  A 클래스가 B 클래스의 어떤 메소드를 사용한다 했을때 이 동작은 직접적으로 두 클래스간의 의존성이 생기게된다. B 클래스의 메소드를 사용하기 전에 인스턴스를 먼저 만들어야 한다.


따라서 객체를 생성하는 작업을 다른 누군가에게 양도하고 그 의존성을 직접 사용하는것을 DI라고 한다. 
특히 큰 프로젝트 일수록 객체의 인스턴스화가 자동으로 이루어져야한다.

{% include image.html id="1-bjAvGkbDpUoWmr-xaJzAXukX6Q62MLb"%}

SOLID 5원칙 중 클래스는 구체화가 아니라 추상화에 의존해야한다 명시한다.
이러한 이론을 따름으로써 코드는 테스트하기 쉬워지고 다른 클래스와의 상속도 쉬워진다.
그리고 더불어 결합성도 느슨해지게된다.


### 그래서 Koin이 뭐징?
---

{% include image.html id="1-V913lD1WN5Qb9dQFUYcLt0QInSIptMV"%}

Koin을 이해하기 앞서 위의 그림처럼 아키텍쳐를 개발을 한다 가정하자.
Activity는 ViewModel에 대한 참조를 가지고 ViewModel은 repository에 대한 참조를 가진다. 
repository 는 로컬쪽과 서버(remote)쪽에 대한 비지니스 로직을 처리해주는 인스턴스들의 참조를 가진다. 
이 예제에서 Koin 을 통해 이런 의존성을 효율적으로 관리하는 걸 알아보자.

#### 1. Gradle 에 Dependencies 추가

```gradle
// Retrofit
implementation "com.squareup.retrofit2:retrofit:$version_retrofit"
implementation "com.squareup.retrofit2:converter-gson:$version_retrofit"
implementation "com.jakewharton.retrofit:retrofit2-kotlin-coroutines-adapter: $version_retrofit_coroutines_adapter"
 
// For ViewModel
implementation "androidx.lifecycle:lifecycle-extensions:$version_viewmodel"
implementation "androidx.lifecycle:lifecycle-viewmodel-ktx:$version_viewmodel"
 
 // Koin
implementation "org.koin:koin-android:$version_koin"
implementation "org.koin:koin-androidx-viewmodel:$version_koin"
implementation "org.koin:koin-androidx-scope:$version_koin"
```

#### 2. GithubUser 객체 추가

```java
data class GithubUser(
  val id: Long,
  val login: String,
  val avatar_url: String
)
```
[https://api.github.com/users](https://api.github.com/users) 와 통신해서 가져온 json 객체를 나타낸다. 이중에서 id, login , avartar_url 값만 추출할 것이다.

#### 3. Retrofit 을 위한 interface 추가
```java
interface UserApi {
    @GET("users")
    fun getAllUsersAsync(): Deferred<List<GithubUser>>
}
```
GitHub에서 user 정보를 가져올 Api 클래스이다. 코루틴의 async를 사용하기 위해 Deffered 타입으로 가져올것이다.

```java
class UserRepository(private val userApi: UserApi, private val userDao: UserDao) {

    val data = userDao.findAll()

    suspend fun refresh() {
        withContext(Dispatchers.IO) {
            val users = userApi.getAllUsersAsync().await()
            userDao.add(users)
        }
    }
}
```
UserRepository 클래스다. 전체적으로 remote(Github API) 쪽 처리와 로컬 DB 쪽 처리에 대한 대행자 역할을 한다. repository 는 실제 수행을 하는쪽이 아니라 대행을 해주는쪽이라 서버쪽과 디비쪽 처리를 해주는 인스턴스들의 참조들을 가진다.

```java
class UserViewModel(private val userRepository: UserRepository) : ViewModel() {

    private val _loadingState = MutableLiveData<LoadingState>()
    val loadingState: LiveData<LoadingState>
        get() = _loadingState

    val data = userRepository.data

    init {
        fetchData()
    }

    private fun fetchData() {
        viewModelScope.launch {
            try {
                _loadingState.value = LoadingState.LOADING
                userRepository.refresh()
                _loadingState.value = LoadingState.LOADED
            } catch (e: Exception) {
                _loadingState.value = LoadingState.error(e.message)
            }
        }
    }
}
```
UserViewModel 클래스다. 실제 View 를 대신해서 비지니스 로직쪽을 수행을 해주는 대리자 역할을 한다.
처리후에 View 에선 ViewModel 의 프로퍼티를 통해 View 는 상태의 변화를 받아서 렌더링한다.

```java
data class LoadingState private constructor(val status: Status, val msg: String? = null) {
    companion object {
        val LOADED = LoadingState(Status.SUCCESS)
        val LOADING = LoadingState(Status.RUNNING)
        fun error(msg: String?) = LoadingState(Status.FAILED, msg)
    }

    enum class Status {
        RUNNING,
        SUCCESS,
        FAILED
    }
}
```  
loading 에 대한 상태를 표시할 LoadingState 로딩중, 성공 ,실패 3가지 상태를 가진다.

#### 4. koin 모듈 정의하기

koin 에서 모듈은 다른 구성요소에 주입하기 위한 모든 의존성을 선언하는 곳이다.
위에 내용처럼 repository는 ViewModel이 ViewModel은 View에서 참조가 이루어진다. 그러므로 우리는 미리 모듈을 통해 사용할 인스턴스들에 대한 정의를 하고 이렇게 함으로 `Boiler Plate Code` 가 줄고 기본적으로 우리가 원할때 인스턴스를 선언할수 있게된다.

```java
val viewModelModule = module {
    single { UserViewModel(get()) }
}

val apiModule: Module = module {

    fun provideUserApi(retrofit: Retrofit): UserApi = retrofit.create(UserApi::class.java)

    single { provideUserApi(get()) }
}

val netModule = module {

    fun provideCache(application: Application): Cache = Cache(application.cacheDir, (10 * 1024 * 1024).toLong())

    fun provideHttpClient(cache: Cache): OkHttpClient =
        OkHttpClient.Builder().run {
            cache(cache)
            build() }

    fun provideGson(): Gson = GsonBuilder().run {
        setFieldNamingPolicy(FieldNamingPolicy.IDENTITY)
        create()
    }

    fun provideRetrofit(factory: Gson, client: OkHttpClient): Retrofit = Retrofit.Builder()
            .run {
                baseUrl("https://api.github.com/")
                addConverterFactory(GsonConverterFactory.create(factory))
                addCallAdapterFactory(CoroutineCallAdapterFactory())
                client(client)
                build()
            }

    single { provideCache(androidApplication()) }
    single { provideHttpClient(get()) }
    single { provideGson() }
    single { provideRetrofit(get(), get()) }

}

val databaseModule = module {

    fun provideDatabase(application: Application): AppDatabase =
         Room.databaseBuilder(application, AppDatabase::class.java, "eds.database").run {
             fallbackToDestructiveMigration()
             allowMainThreadQueries()
             build()
         }

    fun provideDao(database: AppDatabase): UserDao =  database.userDao

    single { provideDatabase(androidApplication()) }
    single { provideDao(get()) }
}

val repositoryModule = module {

    fun provideUserRepository(api: UserApi, dao: UserDao): UserRepository =  UserRepository(api,dao)

    single { provideUserRepository(get(), get()) }
}

val moduleList = listOf(viewModelModule,apiModule , netModule, databaseModule , repositoryModule)
```

`viewmodel` : ViewModel 을 선언하기 위한 컴포넌트다. 반드시 `android.arch.lifecycle.ViewModel`를 확장한 ViewModel 에만 사용해야 한다. inject 시 by viewModel 를 사용해 늦은 초기화를 할수도 있고 getViewModel 를 사용해 즉시 받을수도 있다.

`get()` : 여기서는 UserViewModel 클래스는 UserRepository가 파라미터로 인스턴스가 필요하다. 즉 우리가 get() 을 선언하고 Koin은 해당 인스턴스를 검색하고 공급해준다. 반면 Koin 이 해당 인스턴스를 못찾으면 에러가 발생한다.

`single` : Koin 에게 인스턴스를 Singleton 으로 만들라 지정한다. 어플리케이션 수행동안 딱 한번만 만들어진다.

`factory` :  single 과 반대로 매 순간 새로운 인스턴스를 만들어준다. 위의 예시에선 사용하지 않았지만 매번
새로운 인스턴스가 필요한 부분은 factory 로 만들자.

#### 5. Koin 시작하기
---

이제 우리 앱에서 Koin을 시작해보자 . Application 클래스의 onCreate에  startKoin() 함수를 호출하자.
그리고 위에 선언한 모듈들을 리스트에 담자.

```java
class App : Application() {
    override fun onCreate() {
        super.onCreate()
        startKoin {
            androidLogger(Level.DEBUG)
            androidContext(this@App)
            modules(moduleList)
        }
    }
}
```

#### 6. Dependencies 주입

UserViewModel 컴포넌트는 UserRepository 파라미터와 같이 만들어진다. 
해당 인스턴스를 MainActivity 에서 `by viewModel()` 를 선언하면 Delegate Injector 를 통해 주입이 된다.

```java
class MainActivity : AppCompatActivity() {

    private val userViewModel by viewModel<UserViewModel>()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        userViewModel.data.observe(this , {
            it.forEach { githubUser ->  showToast(githubUser.login) }
        })

        userViewModel.loadingState.observe(this, Observer {
            when(it.status){
                LoadingState.Status.FAILED -> showToast(it.msg)
                LoadingState.Status.RUNNING -> showToast("Loading")
                LoadingState.Status.SUCCESS -> showToast("Success")
            }
        })
    }
    
    private fun showToast(msg:String?) = Toast.makeText(applicationContext , msg ?:"메시지", Toast.LENGTH_LONG).show()
}
```

### 결론
---
Koin 은 순수 코틀린으로 만든 Dagger 와 다르게 컴파일이 아닌 런타임에서 리플렉션을 사용한 DI 라이브러리로 DSL 를 사용해 읽고 이해하기 쉽게 구성할수 있다. 

### 참조
---
[https://insert-koin.io/docs/reference/introduction](https://insert-koin.io/docs/reference/introduction)

[https://medium.com/swlh/dependency-injection-with-koin-6b6364dc8dba](https://medium.com/swlh/dependency-injection-with-koin-6b6364dc8dba)

