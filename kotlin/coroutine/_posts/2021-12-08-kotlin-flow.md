---
layout : single
title : Kotlin Flow
---

### 개요
---
우리는 어떤 연산을 수행후 한개의 값을 반환하는 중단함수를 정의하고 이를 비동기로 수행할 수 있다.
하지만 어떤 연산 후 두개이상의 값을 반환하는 중단함수를 만드는게 가능한가?  
코틀린 플로우(Flow) 를 이용해 이를 수행할 수 있다.

### 다수의 값 나타내기
---
다수의 값은 코틀린에서 컬렉션을 통해 나타낼 수 있다. 예를 들어 우리는 세개의 수를 요소로 갖는 
리스트를 반환하는 `foo()` 함수를 만들고 forEach 함수로 리스트를 하나하나 출력할 수 있다.

```java
fun foo(): List<Int> = listOf(1, 2, 3)
 
fun main() {
    foo().forEach { value -> println(value) } 
}
```

> 1  
2  
3  

### Sequence 
---
코틀린 표준 라이브러리는 Collection 과 함께 다른 컨테이너 타입의 시퀀스를 포함하고 있다.
시퀀스는 Iterable 과 동일한 기능을 제공하지만 다른 여러 콜렉션 처리에 대한 다른 접근방식을 구현한다.
그리고 Collections 확장함수의 경우 호출시마다 새로운 Collections 가 생성되는 구조라 퍼포먼스
측면에서도 차이가 난다.

```java
fun foo(): Sequence<Int> = sequence { // sequence builder
    for (i in 1..3) {
        Thread.sleep(100) // pretend we are computing it
        yield(i) // yield next value
    }
}

fun main() {
    foo().forEach { value -> println(value) } 
}
```

해당식은 위와 결과가 똑같게 나온다. `yield` 라는 함수를 통해 배출하는 흐름을 정할수 있다는 차이가 있다.
즉 eager evaluation 으로 처리하지만 sequence 는 lazy evaluation 으로 처리한다.

```java
val list = listOf(1,2,3).map { it + 1 }
val seq = listOf(1,2,3).asSequence().map {
    it+1
}
```

위와 같은 경우 list 는 결과가 들어가 있지만 sequence 의 경우 `종단 연산자 (Terminal operation)` 를 사용해 확정짓지 않으면 Sequence 타입의 객체가 들어가 있다.

### Suspending function (중단 함수)
---
위의 함수들은 공통적으로 코드 블록은 실행하고 있는 메인 스레드를 정지 시킨다. 즉 실행이 다 되고 나야
밑에 함수가 동작한다. 이러한 연산들이 비동기 코드에서 실행될때 우리는 foo() 에 `suspend` 키워드를 붙여
함수를 중단함수로 정의 할 수 있다. 그리고 이 함수를 코루틴 스코프에서 호출해서 호출 스레드의 정지 없이
실행할 수 있고 그 결과를 리스트로 반환하도록 할 수 있다.

```java
suspend fun foo(): List<Int> {
    delay(1000) 
    return listOf(1, 2, 3)
}

fun main() = runBlocking<Unit> {
    foo().forEach { value -> println(value) } 
}
```

### Flow 
---
위에 중단함수는 List\<Int> 타입을 반환 받는다. 결국 연산을 다 수행후 모든 값을 반환해야 한다는 것이다.
비동기로 처리 될 값 들의 스트림을 나타내기 위해서 우리는 비동기로 처리 되야 할 값들을 Sequence\<Int> 
타입에서 처리 한거처럼 하고 싶을때 Flow\<Int> 를 사용할 수 있다.

```java
fun foo(): Flow<Int> = flow { 
    for (i in 1..3) {
        delay(100) 
        emit(i) 
    }
}

fun main() = runBlocking<Unit> {

    launch {
     for (k in 1..3) {
         println("I'm not blocked $k")
         delay(100)
      }
    }
    
    foo().collect { value -> println("collect $value") } 
}
```

> I'm working in thread main  : I'm not blocked 1  
I'm working in thread main  : collect 1  
I'm working in thread main  : I'm not blocked 2  
I'm working in thread main  : collect 2  
I'm working in thread main  : I'm not blocked 3  
I'm working in thread main  : collect 3  

만약 메인스레드가 막혔다면  I'm not blocked 부분이 3번 올라오고 그 후에
flow 함수가 동작했을것이지만 메인스레드를 막지 않고 동작한것을 알수 있다.
이로써 위의 예제와 Flow 는 차이점들을 가지는데

1. `flow {}` 빌더를 사용해 Flow 타입을 생성할 수 있다.
2. `flow{...}` 블록안의 코드는 언제든 중단이 가능하다.
3. `foo()` 함수는 더이상 suspend 로 마킹할 필요가 없다.
4. 결과값들은 flow 에서 `emit()` 함수를 이용해 방출된다.
5. flow 에서 방출된 값들은 `collect` 함수를 이용해 수집된다.

위의 식에서 delay 만 Thread.sleep 으로 변경시 메인 스레드가 정지돤다.


### Flow 는 Cold Stream 이다.
---
`Cold Stream` 이란 하나의 소비자를 상대로 값을 보내는 형태로 생성된 이후 누군가 소비를 시작하면
그때 데이터를 발행하는 구조다. Rx 에서 `Cold Observable` 이란 개념이 있는데 그와 유사한데
Rx 에서도 subscribe 라는 함수를 호출하는 순간 (구독이 시작되는 순간) 부터 값들을 흘려 보냈다. 
반대로 `Hot Observable` 은 구독자와 상관없이 데이터를 배출을 한다. Hot 은 그래서 마우스 이벤트,
키보드 이벤트 등에 적합하고 Cold 의 경우 웹 요청 , 데이터 베이스 쿼리등을 사용하고 요청결과등을 
받을때 용이하다. 

다시 처음으로 돌아와서 Flow 는 Cold Stream 형태이기 때문에 `collect()` 함수 (종단 연산자) 를 통해
호출하지 않는 이상 배출되지 않는다. 

```java

fun foo(): Flow<Int> = flow { 
    println("Flow started")
    for (i in 1..3) {
        delay(100)
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
  printWithThread("foo 함수 호출")
  val flow = foo()
  printWithThread("foo 함수 아직 실행 안됨.")
  flow.collect { printWithThread("collect $it") }
  printWithThread("foo 함수 재시작")
  flow.collect { printWithThread("collect 2번째 $it") } 
}
```

> foo 함수 호출  
 foo 함수 아직 실행 안됨  
 flow Start  
 collect 1  
 collect 2  
 collect 3  
 foo 함수 재시작  
 flow Start  
 collect 2번째 1  
 collect 2번째 2  
 collect 2번째 3  

foo() 를 호출한 순간에는 이벤트가 일어나지 않다가 collect() 함수를 호출한 순간(구독) 이벤트가 실행되고
끝나고 다시 수행하니 다시 처음부터 이벤트가 시작된다.


### Flow 의 취소
---
플로우는 코루틴의 일반적인 취소 매커니즘을 준수하지만 플로우 자체가 취소지점을 제공하지 않는다.
하지만 코루틴의 경우와 동일하게 플로우도 중단함수에서 중단 되었을떄 취소가 가능하고 그렇지 않으면 할수 없다.

```java
fun foo(): Flow<Int> = flow { 
    for (i in 1..3) {
        delay(100)          
        println("Emitting... $i")
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    withTimeoutOrNull(250) { //250 밀리초 뒤에 타임아웃이 되고 Exception 대신 Null 을 반환한다.
        foo().collect { value -> println("collect $value") } 
    }
    println("완료")
}
```
> flow Start  
Emitting....1  
collect 1  
Emitting....2  
collect 2  
완료


원래대로면 300ms 를 채워 3 까지 나왔어야 했지만 그전에 Null 처리 되서 flow 가 취소가 이루어진 모습이다.

### Flow Builder (플로우 빌더)
---
이전 예제에서 본 `flow {}` 빌더는 가장 기본적인 것이다. 이거 말고도 flow 는 다양한 빌더를 제공한다.

- `flowOf {}` 빌더는 고정된 값들을 방출하는 플로우를 제공한다.
- 다양한 컬렉션들과 시퀀스들은 `.asFlow()` 확장 함수를 통해 플로우로 변환 가능하다.

```java
(1..3).asFlow().collect { value -> println(value) }
```

### Intermediate Flow operators (플로우 중간 연산자)
---
플로우는 컬렉션이나 시퀀스에서 경험한것이 연산자로 변환될 수 있다. 중간 연산자는 업스트림(UpStream) 플로우에 적용되어 다운스트림 플로우를 반환한다. 이 연산자는 당연 Flow 가 콜드 타입이므로 같은 타입으로 동작한다. 연산자 자체의 호출은 중단함수는 아니므로 새롭게 변형된 플로우를 즉시 반환한다.

기본적으로 연산자들을 보면 map 이나 filter 처럼 친숙한 이름들을 가지고 시퀀스랑 가장 큰 차이는
그 블록에는 중단함수를 호출할 수 있다. 예를 들어 map 연산자를 이용해 원하는 결과값으로 변형(매핑) 할 수 있고 요청이 긴 중단함수의 경우도 문제 없이 동작한다.

```java
suspend fun requestApiCall(parameter: Int): String {
        delay(2000)
        printWithThread("requestApiCall ..........")
        return "network Ok $parameter"
}

fun main() = runBlocking<Unit> {
    (1..3).asFlow()
        .map { requestApiCall(it) }
        .collect { res-> printWithThread("요청결과 : $res") }

}

```

> requestApiCall ..........  
I'm working in thread main  : 요청결과 : network Ok 1  
I'm working in thread main  : requestApiCall ..........  
I'm working in thread main  : 요청결과 : network Ok 2  
I'm working in thread main  : requestApiCall ..........  
I'm working in thread main  : 요청결과 : network Ok 3  
I'm working in thread main  : completed  

매 2초 마다 위의 결과 값이 출력됨을 알 수 있다.


### Transform opeator (변환 연산자)
---
`transform` 연산자는 플로우 변환 연산자들중 가장 일반적인 연산자다.  그만큼 다양하게 쓰이는데
map, filter 처럼 반순 변환에도 쓰이고 복잡한 변환 구현용으로도 쓰인다.   
예를 들어 오래 걸리는 비동기 요청을 수행하기 전 기본 문자열을 먼저 방출하고 요청에 대한 응답이 도착하면
그 결과를 방출도 가능하다.

```java
(1..3).asFlow()
            .transform { req ->
                emit("요청 시작 : $req")
                emit(requestApiCall(req))
            }
            .collect { res -> printWithThread("결과 : $res") }
```

> 결과 : 요청 시작 : 1  
requestApiCall ..........  
결과 : network Ok 1  
결과 : 요청 시작 : 2  
requestApiCall ..........  
결과 : network Ok 2  
결과 : 요청 시작 : 3  
requestApiCall ..........  
결과 : network Ok 3  

통신하기 전에 emit 을 통해 요청 시작을 했다는 string 이 먼저 시작되고 그 후 통신하고 결과가 방출된다.


### Size-limiting operators (크기 제한 연산자)
---
collection 함수와 같게 take 함수를 제공하는데 제한치에 도달시 실행을 취소하게 된다. 코루틴에서
취소는 항상 예외를 발생시키는 방식으로 수행되고 이를 통해 try {...} finally {...} 같은 함수들에 걸리게 된다.

```java
fun numbers(): Flow<Int> = flow {
    try {                          
        emit(1)
        emit(2) 
        println("여기까진 도달 안됨.")
        emit(3)    
    } finally {
        println("Finally in numbers")
    }
}

fun main() = runBlocking<Unit> {
    numbers() 
        .take(2) 
        .collect { value -> println(value) }
} 
```

> 1   
2   
Finally in numbers

take 를 통해 취소가 이루어 지고 해당부분이 try {} finally {} 에 잡혀서 Finally in numbers 가
출력됨을 볼수있다. 없다고 오류가 생기진 않지만 finally 쪽에 처리할 부분이 있다면 처리할 수 있다.

### Terminal flow operators (플로우 종단 연산자)
---
플로우의 종단 연산자는 수집을 시작하는 중단 함수이다. 즉 Cold Stream 에서 구독이 시작되는 부분이라
할수 있다. 

- `toList` , `toSet` 같은 다양한 컬렉션으로 변환
- 첫번째 값만 방출하며 플로우는 단일 값만 방출함을 보장
- 플로우를 `reduce` 나 `fold` 함수를 이용해 값을 변환 할수 있다.

```java
val sum = (1..5).asFlow()
    .map { it * it }                          
    .reduce { a, b -> a + b } 
println(sum)
```

reduce 가 종단 연산자이므로 최종적으로 다 처리된 값 (여기선 55) 이 나오게 된다.


### 플로우는 순차적이다.
---
어떤 플로우의 독립된 각각의 수집은 다중 플로우가 사용되는 특별한 경우가 아니면 순차적으로 진행된다.
종단 연사자를 호출한 코루틴에서 수집이 수행되고 기본적으로 새로운 코루틴을 생성하지 않는다.
각각의 방출된 값은 업스트림의 모든 중간 연산자들에 의해 처리되어 다운스트림으로 전달되며 마지막으로 
종단 연산자로 전달된다. 

```java
(1..5).asFlow()
    .filter {
        printWithThread("2의 배수만 뽑아 내기 $it")
        it % 2 == 0
    }
    .map {
        printWithThread("스트링으로 변환 $it")
        "int to String $it"
    }
    .collect { printWithThread("결과 $it") } 
```

> 2의 배수만 뽑아 내기 1  
2의 배수만 뽑아 내기 2  
스트링으로 변환 2  
결과 int to String 2  
2의 배수만 뽑아 내기 3  
2의 배수만 뽑아 내기 4  
스트링으로 변환 4  
결과 int to String 4  
2의 배수만 뽑아 내기 5  

순차적으로 filter 를 거쳐 2의 배수만 뽑아 string 으로 변환 후 종단 연산자에서 값을 방출 하였다.

### Flow Context (플로우의 컨텍스트)
---
플로우의 수집은 항상 호출한 컨텍스트 안에서 수행된다. 어떤 플로우가 있을때 foo 플로우 구현내용과는 
별개로 수집은 작성자가 명시한 컨텍스트 상에서 수행되는데 이를 context preservation (컨텍스트 보존) 
이라 부른다.

기본적으로 `flow {...}` 빌더에 제공된 코드 블록은 플로우 수집을 실행한 코루틴의 컨텍스트에서 수행된다.

```java
 fun foo() = flow {
        printWithThread("flow Start")
        for (i in 1..3) {
            delay(100)
            printWithThread("emitting....$i")
            emit(i)
        }
    }.flowOn(Dispatchers.Default)

fun main() = runBlocking<Unit> {
    foo()
        .map{it.toString()}
        .collect { printWithThread("collect $it") }
}
```

main 함수는 메인스레드에서 호출되는 함수다 거기에 만약 foo () 함수를 Default 디스패처에서 호출시
foo 함수 자체는 Default 스레드 영역에서 실행되지만 결국 collect 자체는 메인스레드에서 호출된다.

> I'm working in thread DefaultDispatcher-worker-1 : flow Start  
I'm working in thread DefaultDispatcher-worker-1 : emitting....1   
I'm working in thread main : collect 1  
I'm working in thread DefaultDispatcher-worker-1 : emitting....2  
I'm working in thread main : collect 2  
I'm working in thread DefaultDispatcher-worker-1 : emitting....3  
I'm working in thread main : collect 3  
I'm working in thread main : completed  

만약 main 자체 호출 영역을 withContext 등으로 해서 Dispatcher 를 변경하면 Collect 도 호출영역이 바뀌므로 
지정한 Dispatcher 에서 호출된다.

```java
   withContext(Dispatchers.Default){
    foo()  
        .map{it.toString()}
        .collect { printWithThread("collect") }  //이러면 collect 자체가 호출되는 스레드가 달라져서 Default 영역에서 호출된다.
   }
```

> I'm working in thread DefaultDispatcher-worker-1  : collect 


### WithContext 를 통한 잘못된 방출
---
오랫동안 CPU 소모적인 작업들은 `Dispatchers.Default` 같이 별도의 스레드에서 수행될 필요가 있고 UI 를 업데이트 하는 코드는 `Dispatchers.MAIN` 같은 UI 전용 스레드에서 동작한다. 
보통 withContext 는 코루틴을 사용하는 코드에서 컨텍스트를 전환하기 위해 사용되는데 flow 빌더 내부 코드는 컨텍스트 보존의 특성을 가지기 때문에 내부에서 다른컨텍스트로 값을 방출하는게 허용되지 않는다.

```java
fun foo() = flow {
        withContext(Dispatchers.IO) {
            printWithThread("flow Start")
            for (i in 1..3) {
                delay(100)
                printWithThread("emitting....$i")
                emit(i)
            }
        }
}
```
이상태로 호출시 아래와 같은 Exception 이 발생하게 되있다. 

> java.lang.IllegalStateException: Flow invariant is violated:

그렇기 때문에 flowOn 연산자를 사용해 실행 컨텍스트를 바꿀 수 있다.


```java
 fun foo() = flow {
        printWithThread("flow Start")
        for (i in 1..3) {
            delay(100)
            printWithThread("emitting....$i")
            emit(i)
        }
}.flowOn(Dispatchers.Default)
```

> I'm working in thread DefaultDispatcher-worker-1 : flow Start  
I'm working in thread DefaultDispatcher-worker-1 : emitting....1  
I'm working in thread main : collect 1  
I'm working in thread DefaultDispatcher-worker-1 : emitting....2  
I'm working in thread main : collect 2  
I'm working in thread DefaultDispatcher-worker-1 : emitting....3  
I'm working in thread main : collect 3  

flowOn 연산자를 사용하면 순차성을 일부 포기하게 된다. 보이는거 처럼 수집과 방출이 서로 다른곳에서
이루어 졌다. 

### buffer 
---
오래 걸리는 비동기 연산같은 로직을 처리하다 보면 플로우의 로직을 다른 코루틴에서 수행하는 것은 
큰 도움이 된다. 예를 들어 방출 자체가 오래걸린다 가정해보면 당연 수집하는 쪽도 기다릴수 밖에 없다보니
시간이 오래 걸릴 수 밖에 없다.

```java

fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) 
        emit(i) 
    }
}

fun main() = runBlocking<Unit> { 
val time = measureTimeMillis {
           foo().collect {
                delay(300) 
                println("collect $it")
             }
           }
println("Collected in $time ms")
}
```

대략 방출때 마다 100ms 에다 수집때도 300ms 이 걸려 400ms 을 3번 하니 1200ms 이상이 걸리게 된다.

> collect 1  
collect 2  
collect 3    
Collected in 1242 ms

만약 이걸 buffer 를 사용할 경우 어떻게 될까?

```java
val time = measureTimeMillis {
                foo()
                  .buffer()
                  .collect {
                   delay(300) // pretend we are processing it for 300 ms
                   println("collect $it")
                }
           }
println("Collected in $time ms")
```

> Collected in 1138 ms

조금이나마 초가 줄었다. 이게 가능한 이유는 첫번째 수를 위해서만 100ms 를 기다리고 
각각의 수 처리를 위해서 300ms 씩만 기다리도록 파이프라인을 효율화 했기 때문이다.

`flowOn 연산자가 CoroutineDispatcher 를 변경할 경우 동일한 버퍼링 매커니즘을 사용한 것이다.
여기서 우린 buffer 연산자를 사용함으로써 실행 컨텍스트의 전환 없이 버퍼링을 수행했다.`

### conflate (병합)
---

어떤 플로우가 연산의 일부나 연산 상태의 업데이트를 방출하는 경우 방출되는 각각의 값은 
처리하는게 불필요하고 최신의 값만을 처리하는걸 원할 수 있다. 이럴경우 `conflate` 연산자를
사용해 중간에 방출된 값들에 대해 스킵을 할 수 있다.

```java
val time = measureTimeMillis {
    foo()
        .conflate() 
        .collect {
            delay(300)
            println("collect $it") 
        } 
}   
println("Collected in $time ms")
```

> collect 1    
collect 3    
Collected in 758 ms

처음값이 처리 중에 두번째와 세번째가 몰려서 스킵되고 결국 가장 최근값인 3 까지 collect 에 전달되었다.


### 최신값 처리 
---

값의 병합은 방출 수집 모두 느릴경우 처리속도를 높이기 위해 사용하는 기법이다. 즉 중간값을
삭제해서 빠르게 하는건데 또 다른 방법이 있다. 새로운 값이 방출될 때 마다 느린걸 취소하고
재시작 하는 방법이다. 연산자들마다 Latest 연산자가 존재하는데 (mapLatest, flatMapLatest,collectLatest 등) 

이 연산자들이 새로운 값이 방출되면 그들의 코드 블록이 취소가 되게 된다. 

```java
 val time = measureTimeMillis {
                foo()
                    .collectLatest {
                       printWithThread("collect start $it")
                       delay(300) // pretend we are processing it for 300 ms
                       printWithThread("collect $it")
                     }
             }
printWithThread("Collected in $time ms")
```

> collect start 1  
collect start 2    
collect start 3    
collect 3    
Collected in 749 ms  

collectLatest 의 코드 블록이 300ms 마다 소모 하고 매번 새로운 값이 100ms 마다 방출되기 때문에
우리는 이 블럭이 매 값에 대해서 실행됨을 확인할 수 있고 마지막 값에 대해서만 끝까지 수행됨을 확인할 수 있다.


### 다중 플로우 합성


#### zip

Collections , Sequence 의 확장함수 zip 과 동일하게 플로우에도 두개의 플로우를 병합하는 zip 연산자가 있다.
두개의 흐름을 하나의 흐름으로 바뀌게 된다.

```java
val nums = (1..3).asFlow().onEach { delay(300) } // numbers 1..3 every 300 ms
val strs = flowOf("one", "two", "three").onEach { delay(400) } // strings every 400 ms
val startTime = System.currentTimeMillis()
nums.combine(strs) { a, b -> "$a -> $b" }
        .collect { value -> printWithThread("$value at ${System.currentTimeMillis() - startTime} ms from start") 
```

위의 onEach 의 경우 연산자가 수행하기 전에 선행으로 수행하게 된다. Rx 를 예를 들면 `doOnSubscribe` 와 
비슷하다 할수 있다. (구독하기 전에 해야할 작업에 대한 명세)

> 1 -> one at 436 ms from start  
 2 -> two at 839 ms from start  
 3 -> three at 1247 ms from start  

#### combine

 zip 과 유사할수 있으나 combine 은 zip 과 다르게 기다리지 않는다. 이게 무슨 말이냐면 예를 들어
 이젠에는 300ms 마다 nums 가 방출되고 strs 가 400ms 마다 방출되었다. 즉 둘중 느린 Flow 에 맞게
 값이 방출된 것이다. (최종 걸린시간도 대략 1247 -> 400 * 3 이상) 그럼 combine 연산자를 사용할 경우
 어떻게 되는가?

 ```java
  nums.combine(strs) { a, b -> "$a -> $b" }
        .collect { value -> printWithThread("$value at ${System.currentTimeMillis() - startTime} ms from start") }
 ```

> 1 -> one at 490 ms from start  
2 -> one at 695 ms from start  
2 -> two at 895 ms from start  
3 -> two at 999 ms from start  
3 -> three at 1301 ms from start  

zip 연산자와 다르게 기다렸다 방출 하지 않고 각각 nums 나 strs 가 방출이 일어날 때 마다 플로우의 
최신값이 병합하여 출력됨을 알수 있다.


### Flattening flows (플로우 플래트닝)
---

플로우는 비동기로 수신되는 값들의 시퀀스를 나타낸다. 그러므로 어떤 플로우에서 수신되는 일련의 값들이
다른 값들의 시퀀스를 요청하는 플로우로 변환되는 일은 자주 일어난다.

예를 들면 500ms 간격으로 두개의 문자열을 방출하는 플로우가 있다해보자.

```java
fun requestFlow(i: Int): Flow<String> = flow {
        emit("$i: First")
        delay(500) // wait 500 ms
        emit("$i: Second")
}
```

flow 빌더에 값을 2번 방출하는 부분이 있다 했을때 여기서 우리가 3개의 정수를 방출하는 플로우를 가지고
각각 정수가 방출될 때 마다 requestFlow 를 호출한다 해보자

```java
(1..3).asFlow()
          .map { requestFlow(it) }
          .collect { printWithThread("collect $it")}
```

> collect kotlinx.coroutines.flow.SafeFlow@ecef489  
collect kotlinx.coroutines.flow.SafeFlow@7960c8e  
collect kotlinx.coroutines.flow.SafeFlow@68accaf  

map 으로 하면 리턴타입 자체가 flow 이기 때문에 해당 리턴타입이 3번 내려오고 끝나게 된다.
하지만 원하는건 이런게 아니라 각각의 flow 에 대한 또 각각의 플로우에 대한 처리를 원할 수 있다.
Rx 에서도 flatMap 이라는 기능이 존재하는데 Observable 한 타입을 다른 Observable 한 타입으로 변환해서 
처리 할 수 있다. 컬렉션과 시퀀스에서도 flatten 과 flatMap 이라는 연산자가 있지만 플로우의 경우
비동기 특성으로 인해 다른 연산자들을 제공해준다.

#### flatMapConcat

자바에서 concat 은 문자열등을 붙일 때 사용하는 함수다. 여기서도 마찬가지로 flatMapConcat, flattenConcat 
등의 연산자들에 의해 구현되는데 이걸 사용하면 연산자가 완료되길 기다렸다가 합쳐지게 된다.

```java
 val startTime = System.currentTimeMillis() 
 (1..3).asFlow().onEach { delay(100) } 
     .flatMapConcat { requestFlow(it) }
     .collect {
         printWithThread("$it at ${System.currentTimeMillis() - startTime} ms from start")
}
```

> 1: First at 121 ms from start  
1: Second at 622 ms from start   
2: First at 727 ms from start   
2: Second at 1227 ms from start  
3: First at 1328 ms from start  
3: Second at 1829 ms from start  


#### flatMapMerge

다른 플래트닝 모드로는 모든 들어오는 플로우들을 동시에 수집하고 그 값들을 단일 플로우로 합쳐서
값들을 가능한 빠르게 방출하도록 하는 모드가 있다. 이것은 flatMapMerge 와 flattenMerge 연산자로 가능해진다.

```java
 val startTime = System.currentTimeMillis() 
 (1..3).asFlow().onEach { delay(100) } 
     .flatMapMerge { requestFlow(it) }
     .collect {
         printWithThread("$it at ${System.currentTimeMillis() - startTime} ms from start")
}
```

> 1: First at 136 ms from start   
2: First at 231 ms from start   
3: First at 333 ms from start   
1: Second at 639 ms from start   
2: Second at 732 ms from start   
3: Second at 833 ms from start  

flatMapMerge 의 경우 코드 블록은 순차적으로 호출 하지만 그 결과를 동시에 수집함을 알 수 있다.

```java
 val startTime = System.currentTimeMillis() 
 (1..3).asFlow().onEach { delay(100) } 
     .map { requestFlow(it) }
     .flattenMerge()
     .collect {
         printWithThread("$it at ${System.currentTimeMillis() - startTime} ms from start")
}
```
이렇게 처리해도 결과는 같다.

#### flatMapLatest

위에서 언급했던적 있는 최신값을 가져오는 연산자 플래트닝 버전이다. 구조는 똑같이 방출될때 마다 직전 플로우를 취소해서 새로운 값만 가져온다.

```java
 val startTime = System.currentTimeMillis() 
 (1..3).asFlow().onEach { delay(100) } 
     .flatMapLatest { requestFlow(it) }
     .collect {
         printWithThread("$it at ${System.currentTimeMillis() - startTime} ms from start")
}
```

>  1: First at 175 ms from start  
 2: First at 283 ms from start  
 3: First at 390 ms from start    
 3: Second at 893 ms from start  

 새값이 방출될 경우 그 실행블록이 ( 여기선 { requestFlow(it) } ) 이 전체를 취소한다.
 만약 requestFlow 호출이 빠를경우 중단할 틈도 없기 때문에 일반적 호출과 차이를 못느낄수도 있으나
 기다리는 시간이 길어 질수록 메리트가 있는 연산자이다.


### Flow Execption (연산자 예외)
---
플로우 수집은 방출 로직이나 연산자 안의 코드가 예외를 일으키면 예외 발생 상태로 종료가 된다.

#### try/catch 

```java

 fun exceptionTestFlow() = flow {
        for(i in 1..3){
            printWithThread("emitting...$i")
            delay(100)
            emit(i)
        }
}
fun main() = runBlocking<Unit> {
    try {
     exceptionTestFlow()
        .collect {
            printWithThread("collect $it")
            check(it <= 1) { "Collected fail $it"}
        }
    }catch (e :Exception){
      printWithThread("exception 발생 : $e")
    }
}
```

> emitting...1  
collect 1  
emitting...2  
collect 2  
exception 발생 : java.lang.IllegalStateException: Collected fail 2

check 함수는 조건을 받고 조건이 일치 하지 않는 경우 `IllegalStateException` 을 발생시키는 함수다.
위의 코드에서 collect 종단 연산자에서 성공적으로 예외를 잡고 추가 방출은 이루어 지지 않는다.

#### 모든 예외 처리(Everything is caught)

위의 예제의 경우 방출 로직 혹은 중간 , 종단 연산자 까지 발생하는 모든 예외를 잡는다. 
예를 들어 방출된 수를 문자열로 변환 하도록 코드를 변경하고 해당부분에서 예외를 발생시켜보자.

```java
fun flowExceptionTest(): Flow<String> =
        flow {
            for (i in 1..3) {
                printWithThread("Emitting... $i")
                emit(i) 
            }
        }
            .map {
                check(it <= 1) { "Crashed on $it" }
                "string $it"
            }

fun main() = runBlocking<Unit> {
    try {
        flowExceptionTest()
           .collect { printWithThread("collect $it") }
        }
    }catch (e :Exception){
      printWithThread("exception 발생 : $e")
    }
}
```

> emitting...1  
collect string 1    
emitting...2  
collect 2  
exception 발생 : java.lang.IllegalStateException: Crashed on 2  

해당 예외 또한 flowExceptionTest 함수의 Crushed on 메시지를 가지고 main 의 catch 함수로 들어오면서
수집이 중단 된다.

### 예외 투명성 (Exception transparency)
---

그럼 방출 코드의 예외 처리 로직을 캡슐화 할수 있는가? 플로우는 예외에 있어서 반드시 투명해야한다.
즉 함수가 예외를 발생하지 않는다는 것을 보장하고 항상 성공적으로 수행을 마쳐야한다. 
블록안에서 try/catch 블록으로 예외 처리를 한 후 값을 방출하는것은 예외 투명성에 어긋나는 행동이다. 

방출 로직은 이러한 투명성 보존을 위해 catch 연산자를 사용할 수 있고 이를 통해 예외 처리 로직을 캡슐화 할 수 있다. 

- throw 연산자를 통한 예외 다시 던지기
- catch 로직에서 emit 사용하여 어떤 값 타입으로 방출 (메시지 , 에러객체)
- 다른코드를 통한 예외 무시 로깅 처리 등등...

```java
     flowExceptionTest()
         .catch { e -> emit("Caught Exception $e") }
         .collect { printWithThread("collect $it") }
```

> Caught Exception java.lang.IllegalStateException: Crashed on 2

위의 함수를 catch 연산자를 붙였다. try/catch 와 같은 결과가 나왔다.


### Transparent catch (catch 예외 투명성)

예외 투명성을 지키는 catch 중간 연산자는 오직 업 스트림에서 발생하는 예외에만 대응하고
다운스트림에서 발생한 예외에 대해선 처리하지 않는다.


```java

fun foo() = flow {
        for (i in 1..3) {
            delay(100)
            emit(i)
        }
}

fun main() = runBlocking<Unit> {
foo()
   .catch { e -> printWithThread("Caught Exception $e") }
   .collect {
       check(it <= 1) { "error exception $it" }
       printWithThread("collect $it")
    }
}
```
> collect 1  
Exception : java.lang.IllegalStateException: error exception 2

다운 스트림에서 Exception 이 발생했고 위의 catch 연산자의 Caught Exception 의 문구는 출력되지 
않았다. 

