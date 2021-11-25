---
layout : single
title : withContext 와 async 의 차이점
---

### withContext 와 async 둘은 유사하면서 다른점을 가지고 있다.
---
코루틴은 골치아프게 반응형 스타일 코딩의 번거로움 없이 비동기코드를 유창한 API를 사용해 순차적으로 다룰수 있게 해주는 도구다. 코루틴은 CoroutineContext 안에서 동작하고 Context는 몇몇 CoroutineContext.Elements로
구성되있다. 기본 Element로는 CoroutineId, CoroutineName, CoroutineDispatcher, Job 등이 있다.

CoroutineContext 는 어느 스레드 혹은 스레드들과 상호작용하며 코루틴 작업을 수행할지를 정하는 CoroutineDispatcher가 포함되있다. 

CoroutineDispatcher 는 코루틴이 특정 스레드에서만 작업을 수행하도록 한정하고 그것들을 thread pool 로 보내거나 혹은 한정하지 않고 수행하도록한다. 

CoroutineScope 는 코루틴을 위한 스코프를 정의한다. 모든 코루틴 빌더들 (async, launch 등) 은 
CoroutineScope 의 확장함수다. 그리고 그것들의 CoroutineContext 를 상속받고 자동으로 elements들과 
취소등을 전파한다. 


### async-await 란 무엇인가?
---
async 는 CoroutineScope 의 확장함수로 새로운 취소가능한 코루틴을 만든다. 
Deferred 객체를 반환하고 코드 블록의 미래의 결과를 가지고 있다. 

```java
public fun <T> CoroutineScope.async(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> T
): Deferred<T> {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyDeferredCoroutine(newContext, block) else
        DeferredCoroutine<T>(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}
```

두번째 start 라는 파라미터는 기본적으로 CoroutineStart.DEFAULT 기본적으로 생성되자마자 수행되지만 파라미터에  로 설정되있으나 CoroutineStart.LAZY를 넣을경우 즉시 실행되지 않는다.
각각 독립적 작업으로 동시적 수행이 가능하고 결과를 반환받기를 원한다면 await() 사용해 기다릴 수 있다.

### withContext 란 무엇인가?
---
withContext 는 취소가능한 새로운 코루틴을 허용해주는 scope 함수다. 

```java
public suspend fun <T> withContext(
    context: CoroutineContext,
    block: suspend CoroutineScope.() -> T
): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return suspendCoroutineUninterceptedOrReturn sc@ { uCont ->
        // compute new context
        val oldContext = uCont.context
        val newContext = oldContext + context
        // always check for cancellation of new context
        newContext.checkCompletion()
        // FAST PATH #1 -- new context is the same as the old one
        if (newContext === oldContext) {
            val coroutine = ScopeCoroutine(newContext, uCont)
            return@sc coroutine.startUndispatchedOrReturn(coroutine, block)
        }
        // FAST PATH #2 -- the new dispatcher is the same as the old one (something else changed)
        // `equals` is used by design (see equals implementation is wrapper context like ExecutorCoroutineDispatcher)
        if (newContext[ContinuationInterceptor] == oldContext[ContinuationInterceptor]) {
            val coroutine = UndispatchedCoroutine(newContext, uCont)
            // There are changes in the context, so this thread needs to be updated
            withCoroutineContext(newContext, null) {
                return@sc coroutine.startUndispatchedOrReturn(coroutine, block)
            }
        }
        // SLOW PATH -- use new dispatcher
        val coroutine = DispatchedCoroutine(newContext, uCont)
        coroutine.initParentJob()
        block.startCoroutineCancellable(coroutine, coroutine)
        coroutine.getResult()
    }
}
```
CoroutineContext 를 파라미터로 넘기면 async 같은 경우는 새 CoroutineContext 로 새로운 코루틴을 만들었다면  withContext 는 부모 Context 와 합쳐서 새로운 CoroutineContext를 만들고 그 Context 안에서 작업이 수행된다. 

그렇게 전달받은 CoroutineContext 스레드로부터 블럭이 동작하고 완료후에는 이전 CoroutineContext 로  돌아간다.

만약 부모 block 안에서 여러 withContext를 만들었다면 각각이 부모 thread를 중단시킨다. 하지만 그로인해
순차적으로 진행된다.



### 참고
---
[https://www.baeldung.com/kotlin/withcontext-vs-async-await](https://www.baeldung.com/kotlin/withcontext-vs-async-await)