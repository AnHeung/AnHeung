---
layout : single
title :  MVVM ViewModel 과 AAC ViewModel 의 차이점 정리
---

Android 는 AAC(Android Architecture Components) 를 통해 ViewModel 을 제공한다. 
MVVM 의 ViewModel 과 같은 ViewModel 이라 같은거라 생각할 수 있는데 실제로는 완전 다르다.

MVVM 의 ViewModel 은 View 와 Model 사이의 데이터 관리와 바인딩을 목적으로 사용한다.
반면에 Android 의 ViewModel 의 경우 UI 컨트롤러의 생명 주기 관리 목적으로 사용하는 DataHolder 클래스다.
예를 들면 화면을 회전하는 이벤트가 있다고 가정하면 Android 생명주기상 액티비티에선
`onPause() → onSaveInstanceState() → onStop() → onDestory() → onCreate() → onStart() → onResume()`
과정을 거친다. Fragment 도 종료시 `onPause() → onSaveInstanceState()` 순서로 호출된다.
이 과정에서 인스턴스는 Bundle 이라는 객체에 Key-Value 형태로 저장된다.

```java
override fun onSaveInstanceState(outState: Bundle) {
     super.onSaveInstanceState(outState)
}
```

보통 이부분을 override 해서 저장해야 될 값들을 bundle 에 저장하고 onCreate 시 다시 저장하는식으로 하지만
비트맵 같은 대용량 데이터가 아니라 직렬화 했다가 역직렬화 하기 용이한 소량의 데이터만 적합하다.
게다가 이미 수행된 호출을 다시 수행할수도 있기에 해당부분을 못하도록 처리하는 부분도 필요하게 된다.

### ViewModel 을 사용해 해결
---
AAC 의 ViewModel 은 이런 문제를 쉽게 해결해준다. ViewModel 은 액티비티와 프래그먼트에 사용되는
UI 관련 데이터를 보관하고 관리하기 위해 디자인 되었다. 액티비티가 재생성되는 과정에서 ViewModel 인스턴스를
유지함으로 데이터는 안전하게 다뤄진다. 게다가 데이터 관리를 View 가 아닌(Activity ,Fragment 등) ViewModel 이 해줌으로써 관심사 분리가 되고 UI 는 본인의 역할에 충실하게 된다. 

덕분에 UI 로직에 대한 테스트도 수월해진다. 

위 그림처럼 ViewModel 스코프는 Finshed 전까지는 계속 유지되며 마지막에 액티비티 스코프가 종료 시점에 onCleared 호출됨으로 리소스가 정리가 된다.
ViewModel 은 액티비티의 싱글톤 객체처럼 사용할 수 있어서 프래그먼트들 사이에서 ViewModel 을 이용해
데이터 공유도 용이하다. 이는 액티비티가 프래그먼트간 데이터 공유에도 역할을 덜어냄을 의미한다.

### ViewModel 좀더 살펴보기
---
실제 ViewModel 의 내부를 보면 내부적으로 프래그먼트를 사용한다. ViewModel 의 생성은 ViewModelProvider 로만 가능하고 Provider 내부는 ViewModelStore 와 Factory 로 구성되어있다. 실제 액티비티나 프래그먼트를
들어가보면 FragmentActivity 의 부모격인 ComponentActivity 와 Fragment 클래스가 LifecycleOwner 클래스와 ViewModelStoreOwner 를 구현하고 있는 클래스라는 걸 알수 있는데 ViewModelProvider 생성자는 이렇게 3가지로 나뉜다. 

- ViewModelProvider(ViewModelStoreOwner owner)
- ViewModelProvider(ViewModelStoreOwner owner, ViewModelProvider.Factory factory)
- ViewModelProvider(ViewModelStore store, ViewModelProvider.Factory factory)

즉 액티비티나 프래그먼트는 ViewModelStoreOwner 고 ViewModelStore 는 이들을 HashMap<String, ViewModel>
형태로 보관하고 있다. ViewModel 을 선언하는 클래스가 다르더라도 생성자로 넘긴 ViewModelStoreOwner 객체가 같다면 ViewModeProvider는 동일한 ViewModel 을 전달해준다.


### 참조
---
[https://ko.wikipedia.org/wiki/%EB%AA%A8%EB%8D%B8-%EB%B7%B0-%EB%B7%B0%EB%AA%A8%EB%8D%B8](https://ko.wikipedia.org/wiki/%EB%AA%A8%EB%8D%B8-%EB%B7%B0-%EB%B7%B0%EB%AA%A8%EB%8D%B8)