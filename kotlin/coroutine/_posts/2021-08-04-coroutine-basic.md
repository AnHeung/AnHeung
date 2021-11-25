---
layout : single
title : 코루틴 기초
---

코루틴은 스레드와 유사한 개념이다. 그러나 코루틴은 어떤 특정 스레드와 묶이지 않고 
하나의 스레드에서 동작을 중단하고 재개한다. 

코루틴은 가벼운 스레드 정도로 생각할 수 있지만 실제 사용에 있어선 
스레드와 중요한 차이점이 많다.

```java
runBlocking { // this: CoroutineScope
    launch { // launch a new coroutine and continue
        delay(1000L) // non-blocking delay for 1 second (default time unit is ms)
        println("World!") // print after delay
    }
    println("Hello") // main coroutine continues while a previous one is delayed
}
```
> Hello  
World!

launch 는 코루틴 빌더다. 독립적으로 계속 해야할 작업이 남아있는 코드와 동시에 새 코루틴을 띄운다. 
그래서 선행으로 남아있는 작업이 수행되서 Hello가 먼저 뜨고 World가 나중에 떳다.

delay 는 중단(suspend) 함수다. 중단함수는 특정 시간 코루틴을 중단하고 동작하는 스레드를 차단 하지
않고 다른 코루틴이 해당스레드에서 코드를 동작하도록 한다. 즉 내 코루틴을 임시중단하고 다른 코루틴에게
스레드 우선권을 넘겨주는것이다.

runBlocking도 코루틴 빌더로 새 코루틴을 만들고 현재 스레드를 runBlocking 안에 코드가 다 완료 될때까지
차단 한다. 보통 코루틴과 다르게 main 함수나 테스트 상황에서 주로 사용된다. 

### Structured concurrency
---
코루틴은 구조적 동시성의 원칙을 따른다. 이 말은 새 코루틴은 특정 생명주기가 제한된 CoroutineScope에서만 띄울수 있다라는 것이다.  CoroutineScope는 서로 다른 코루틴간에 부모 자식관계를 담당한다. 우리는 항상
스코프 내에서 새 코루틴을 시작한다. CoroutineContext 는 코루틴 사용자가 지정한 이름 , 코루틴이 예약되어야 하는 스레드를 지정하는 Dispather 등의 추가 기술정보를 저장한다.

launch, async , runBlocking 은 새 코루틴에서 사용되고 자동으로 거기에 맞는 스코프를 만든다.
이 함수들은 모두 리시버가 있는 람다함수로 암시적으로 CoroutineScope를 전달하고 있다.
launch { /* this: CoroutineScope */ } 이런식으로 실제 내부는 해당 코루틴 스코프를 전달받는다.

새 코루틴은 항상 스코프 안에서만 동작한다. launch 나 async 둘다 CoroutineScope 의 확장함수로 
암시적 혹은 명시적으로 리시버가 CoroutineScope를 전달해준다. 

코루틴 안에 중첩된 코루틴이 있다면 우리는 안쪽의 코루틴을 자식 바깥쪽 코루틴은 부모라 지정한다.
이런 부모 자식 관계에서 자식 코루틴은 부모 코루틴에 상응해서 동작한다.


### 참조
---
[https://kotlinlang.org/docs/coroutines-basics.html](https://kotlinlang.org/docs/coroutines-basics.html)