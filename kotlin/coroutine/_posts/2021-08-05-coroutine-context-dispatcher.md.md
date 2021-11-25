---
layout : single
title : CoroutineContext 와 Dispatcher
---

### 코루틴이란?
---
코루틴은 협력적 멀티태스킹을 하기 위한 서브루틴 혹은 프로그램이다. 
코루틴은 중단과 재개를 할 수 있고 다른 코루틴에게 양보도 가능하다. 여기서 중단의 뜻은
호출자를 중단하고 코루틴내에서만 호출할 수 있다는 뜻이다.


### Coroutine Context
---

```java
@SinceKotlin("1.3")
public interface CoroutineContext {
    /**
     * Returns the element with the given [key] from this context or `null`.
     */
    public operator fun <E : Element> get(key: Key<E>): E?

    /**
     * Accumulates entries of this context starting with [initial] value and applying [operation]
     * from left to right to current accumulator value and each element of this context.
     */
    public fun <R> fold(initial: R, operation: (R, Element) -> R): R

    /**
     * Returns a context containing elements from this context and elements from  other [context].
     * The elements from this context with the same key as in the other one are dropped.
     */
    public operator fun plus(context: CoroutineContext): CoroutineContext =
        if (context === EmptyCoroutineContext) this else // fast path -- avoid lambda creation
            context.fold(this) { acc, element ->
                val removed = acc.minusKey(element.key)
                if (removed === EmptyCoroutineContext) element else {
                    // make sure interceptor is always last in the context (and thus is fast to get when present)
                    val interceptor = removed[ContinuationInterceptor]
                    if (interceptor == null) CombinedContext(removed, element) else {
                        val left = removed.minusKey(ContinuationInterceptor)
                        if (left === EmptyCoroutineContext) CombinedContext(element, interceptor) else
                            CombinedContext(CombinedContext(left, element), interceptor)
                    }
                }
            }

    /**
     * Returns a context containing elements from this context, but without an element with
     * the specified [key].
     */
    public fun minusKey(key: Key<*>): CoroutineContext

    /**
     * Key for the elements of [CoroutineContext]. [E] is a type of element with this key.
     */
    public interface Key<E : Element>

    /**
     * An element of the [CoroutineContext]. An element of the coroutine context is a singleton context by itself.
     */
    public interface Element : CoroutineContext {
        /**
         * A key of this coroutine context element.
         */
        public val key: Key<*>

        public override operator fun <E : Element> get(key: Key<E>): E? =
            @Suppress("UNCHECKED_CAST")
            if (this.key == key) this as E else null

        public override fun <R> fold(initial: R, operation: (R, Element) -> R): R =
            operation(initial, this)

        public override fun minusKey(key: Key<*>): CoroutineContext =
            if (this.key == key) EmptyCoroutineContext else this
    }
}

```

모든 코루틴은 CoroutineContext 와 관련있다. CoroutineContext 는 Elements 모음인데 각
요소들의 고유키와 함께 지정된다. 
모든 코루틴 클래스는 CoroutineScope를 구현하고 있고 CoroutineContext를 프로퍼티로 가진다.
그래서 우리는 코루틴 블록안에서 CoroutineContext에 접근할 수 있는것이다.

### CoroutineContext 는 어떻게 조정할 수 있는가?
---
CoroutineContext 는 불변이다. 하지만 element를 추가하거나 지우거나 이미 존재하는 context에 합치거나 하는식으로 새 Context 를 얻을 수 있다. 또한 만약 어떠한 요소도 없는 context 면 EmptyCoroutineContext 로 
새로 만든다.
plus operator 가 구현되있어서 기존 Context 에 새로 들어오는 context도 merge 할 수 있다. 
여기서 눈에 띄는 요소는 CoroutineContext 자체가 싱글톤인 Element 인스턴스라는 거다. 
그래서 우리는 context 를 추가함으로 쉽게 새로운 context를 만들 수 있다.


```java
val context = EmptyCoroutineContext
val newContext = context + CoroutineName("baeldung")
```

이런식으로 컨텍스트를 더할수 있고

```java
val context = CoroutineName("baeldung")
val newContext = context.minusKey(CoroutineName)
```
이런식으로 뺄수도 있다.

```java
public operator fun <E : Element> get(key: Key<E>): E?
```
이 부분을 통해 coroutineContext[Job]로 Element 도 읽을 수 있다.

### Coroutine Context Elements
---
코틀린은 CoroutineContext.Element 가 코루틴의 다양한 측면을 유지하고 관리하기 위한 여러 구현이 있습니다.

- 디버깅 : CoroutineName , CoroutineId
- 생명주기 관리 : Job, 작업 계층을 저장하고 생명주기를 관리하기 위해 사용
- Exception Handling : CoroutineExceptionHandler 는 launch 와 같은 코루틴 빌더안에서 exception 이 발생할 경우 전파되지 않도록 다룬다.
- 스레드 관리 :  ContinuationInterceptor 는 코루틴 안의 Continuation 를 감시하고 그것을 재개하기 위해
중간에 가로챈다.  

### Dispatchers
---
CoroutineDispatcher 는 ContinuationInterceptor Element 의 서브 타입이다. 

```java
public abstract class CoroutineDispatcher :
    AbstractCoroutineContextElement(ContinuationInterceptor), ContinuationInterceptor {
        ...
    }
```

그리고 ContinuationInterceptor 는  CoroutineContext.Element 의 서브 타입이다.
```java
public interface ContinuationInterceptor : CoroutineContext.Element {
    ...
}
```

그러므로 코루틴을 어느 스레드에서 수행할지 결정하는데 책임이 있다. 
코틀린이 코루틴을 수행할때 처음 체크하는부분이 CoroutineDispatcher의 isDispatchNeeded 부분이다.

```java
public open fun isDispatchNeeded(context: CoroutineContext): Boolean = true
```

true 일경우 dispatch 메소드가 실행할 스레드를 할당한다. 만약 false 면 제한없이 코루틴을 수행한다.

- Dispatchers.Default : CPU 사용량이 많은 작업에 사용합니다. 주 스레드에서 작업하기에는 너무 긴 작업 들에게 알맞다.
- Dispatchers.Main : 안드로이드의 경우 UI 스레드를 사용합니다.
- Dispatchers.IO : 네트워크, 디스크 사용 할때 사용합니다. 파일 읽고, 쓰고, 소켓을 읽고, 쓰고 작업을 멈추는것에 최적화되어 있습니다.
- Dispatchers.Unconfined : 다른 Dispatcher 와 는 달리 특정 스레드 또는 특정 스레드 풀을 지정하지 않습니다. 특정목적시에만 사용된다.

```java
runBlocking{
     launch(Dispatchers.Main) {
          println("Main : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(Dispatchers.IO) {
        println("IO : I'm working in thread ${Thread.currentThread().name}")
     }
    launch(Dispatchers.Unconfined) {
        println("Unconfined : I'm working in thread ${Thread.currentThread().name}")
     }
    launch(Dispatchers.Default) {
        println("Default : I'm working in thread ${Thread.currentThread().name}")
    }
}
```

출력결과
> Unconfined : I'm working in thread main  
Default : I'm working in thread DefaultDispatcher-worker-2  
IO : I'm working in thread DefaultDispatcher-worker-1  
Main : I'm working in thread main

### 참조
---
[https://kotlinlang.org/docs/coroutines-basics.html](https://kotlinlang.org/docs/coroutines-basics.html)