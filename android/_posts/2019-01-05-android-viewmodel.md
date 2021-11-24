---
layout : single
title :  AAC ViewModel 에 대해서
---
### Android 에 화면 회전 문제
--- 
Android 는 AAC(Android Architecture Components) 를 통해 ViewModel 을 제공한다. 
MVVM 의 ViewModel 과 같은 ViewModel 이라 같은거라 생각할 수 있는데 실제로는 완전 다르다.

MVVM 의 ViewModel 은 View 와 Model 사이의 데이터 관리와 바인딩을 목적으로 사용한다.
반면에 Android 의 ViewModel 의 경우 UI 컨트롤러의 생명 주기 관리 목적으로 사용하는 클래스다.
예를 들면 화면을 회전하는 이벤트가 있다고 가정하면 Android 생명주기상 액티비티에선
`onPause() → onSaveInstanceState() → onStop() → onDestory() → onCreate() → onStart() → onResume()`
과정을 거친다. Fragment 도 종료시 `onPause() → onSaveInstanceState()` 순서로 호출된다.
이 과정에서 인스턴스는 Bundle 이라는 객체에 Key-Value 형태로 저장된다.

```java
override fun onSaveInstanceState(outState: Bundle) {
     super.onSaveInstanceState(outState)
}
```

보통 이부분을 override 해서 회전하기 전에 저장해야 될 값들을 bundle 에 저장하고 onCreate 가 재호출시 bundle 에서 넣어둔 데이터를 가지고 View 에 뿌려주는 식으로 하지만 비트맵 같은 대용량 데이터가 아니라 직렬화 했다가 역직렬화 하기 용이한 소량의 데이터만 적합하다. 게다가 이미 수행된 호출을 다시 수행할수도 있기에 해당부분을 못하도록 처리하는 부분도 필요하게 된다.

### ViewModel 을 사용해 해결
---
AAC 의 ViewModel 은 이런 문제를 쉽게 해결해준다. ViewModel 은 액티비티와 프래그먼트에 사용되는
UI 관련 데이터를 보관하고 관리하기 위해 디자인 되었다. 액티비티가 재생성되는 과정에서 ViewModel 인스턴스를
유지함으로 데이터는 안전하게 다뤄진다. 게다가 데이터 관리를 View 가 아닌(Activity ,Fragment 등) ViewModel 이 해줌으로써 관심사 분리가 되고 UI 는 본인의 역할에 충실하게 된다. 

덕분에 UI 로직에 대한 테스트도 수월해진다. 

{% include image.html id="101L6gmG0ZE2IJLrnJdjhFWt3f1fhMyE4" %}

위 그림처럼 ViewModel 스코프는 Finshed 전까지는 계속 유지되며 마지막에 액티비티 스코프가 종료 시점에 onCleared 호출됨으로 리소스가 정리가 된다.
ViewModel 은 액티비티의 싱글톤 객체처럼 사용할 수 있어서 프래그먼트들 사이에서 ViewModel 을 이용해
데이터 공유도 용이하다. 이는 액티비티가 프래그먼트간 데이터 공유에도 역할을 덜어냄을 의미한다.

### ViewModel 내부 살펴보기
---
```java
public open class ViewModelProvider(
    private val store: ViewModelStore,
    private val factory: Factory
) {
    ...
}
```

ViewModel 의 생성은 ViewModelProvider 로만 가능하고 Provider 내부는 ViewModelStore 와 Factory 로 구성되어있다. 실제 액티비티나 프래그먼트를 내부를 보면 FragmentActivity 를 상속받고 있는데 FragmentActivity 의 부모격인 ComponentActivity 와 Fragment 클래스가 LifecycleOwner 클래스와 ViewModelStoreOwner 를 구현하고 있는 클래스라는 걸 알수 있다.

즉 액티비티나 프래그먼트는 ViewModelStoreOwner 고 ViewModelStore 는 이들을 HashMap<String, ViewModel>
형태로 보관하고 있다. 그래서 Provider 에 ViewModel 을 얻기 위해 get 을 호출 하면 클래스를 파라미터로 던져주고 내부의 get 함수를 통해 인스턴스를 얻게 된다.

```java
@MainThread
    public open operator fun <T : ViewModel> get(modelClass: Class<T>): T {
        val canonicalName = modelClass.canonicalName
            ?: throw IllegalArgumentException("Local and anonymous classes can not be ViewModels")
        return get("$DEFAULT_KEY:$canonicalName", modelClass)
}

@Suppress("UNCHECKED_CAST")
    @MainThread
    public open operator fun <T : ViewModel> get(key: String, modelClass: Class<T>): T {
        var viewModel = store[key]
        if (modelClass.isInstance(viewModel)) {
            (factory as? OnRequeryFactory)?.onRequery(viewModel)
            return viewModel as T
        } else {
            @Suppress("ControlFlowWithEmptyBody")
            if (viewModel != null) {
                // TODO: log a warning.
            }
        }
        viewModel = if (factory is KeyedFactory) {
            factory.create(key, modelClass)
        } else {
            factory.create(modelClass)
        }
        store.put(key, viewModel)
        return viewModel
}
```
그리고 ViewModelStore 에 저장되있는 HashMap 에 해당 이름으로 된 ViewModel 을 준다.

```java
public class ViewModelStore {

    private final HashMap<String, ViewModel> mMap = new HashMap<>();

    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.put(key, viewModel);
        if (oldViewModel != null) {
            oldViewModel.onCleared();
        }
    }

    final ViewModel get(String key) {
        return mMap.get(key);
    }

    Set<String> keys() {
        return new HashSet<>(mMap.keySet());
    }

    /**
     *  Clears internal storage and notifies ViewModels that they are no longer used.
     */
    public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.clear();
        }
        mMap.clear();
    }
}
```

그래서 우리는 ViewModel 객체를 생성할 때 액티비티나 프래그먼트를 필요로 하고, 어떤 Owner 를 통해 생성하냐에 따라 ViewModel 의 Scope 가 결정된다.

단 액티비티나 프래그먼트만 있다고 해서 ViewModel 을 얻을수는 없다. 내부에 있는 interface 인
ViewModelProvider.Factory 를 통해서 얻거나 ViewModelProvider 의 get 메소드를 통해 얻을 수 있다.

### ViewModel 생성하기
---

ViewModel 은 생성자로 즉시 생성할수 없고 ViewModelProvider 를 통해서만 생성하거나 ViewModelProvider 내부의 Factory 인터페이스를 구현해서 얻을 수 있다.

ViewModelProvider 생성자는 이렇게 3가지로 나뉜다. 

- ViewModelProvider(ViewModelStoreOwner owner)
- ViewModelProvider(ViewModelStoreOwner owner, ViewModelProvider.Factory factory)
- ViewModelProvider(ViewModelStore store, ViewModelProvider.Factory factory)

#### 1. ViewModelProvider 를 사용한 경우

```java
class MainActivity : AppCompatActivity() {
private val userViewModel  by lazy { ViewModelProvider(this)[UserViewModel::class.java] }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
    }
}
```
위에 생성자중 ViewModelStoreOwner 인 액티비티를 파라미터로 넣어 생성후  get 메소드를 통해 얻는 방법이다.

#### 2. ViewModelProvider.Factory 를 사용한 경우

```java
public class ViewModelProvider {
    public interface Factory {
        @NonNull
        <T extends ViewModel> T create(@NonNull Class<T> modelClass);
    }
}
```
위처럼 Provider 내부엔 Factory 인터페이스가 있는데 이 인터페이스를 구현해서 ViewModel 을 얻을 수 있다.

```java
class UserViewModelFactory(val repository: UserRepository) : ViewModelProvider.Factory{
    override fun <T : ViewModel> create(modelClass: Class<T>): T {
        return if (modelClass.isAssignableFrom(UserViewModel::class.java)) {
            UserViewModel(repository) as T
        } else {
            throw IllegalArgumentException()
        }
    }
}
```
Factory 클래스를 구현하고 만약 ViewModel 에 파라미터를 넣고 싶으면 생성자부분에 파라미터를 받아 
ViewModel 을 만들면 된다.


### 참조
---
[https://readystory.tistory.com/176](https://readystory.tistory.com/176)
[https://developer.android.com/topic/libraries/architecture/viewmodel?hl=ko](https://developer.android.com/topic/libraries/architecture/viewmodel?hl=ko)