---
layout : single
title : Kotlin Flow
---

### 개요

우리는 어떤 연산을 수행후 한개의 값을 반환하는 중단함수를 정의하고 이를 비동기로 수행할 수 있다.
하지만 어떤 연산 후 두개이상의 값을 반환하는 중단함수를 만드는게 가능한가?  
코틀린 플로우(Flow) 를 이용해 이를 수행할 수 있다.

### 다수의 값 나타내기

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

코틀린 표준 라이브러리는 Collection 과 함께 다른 컨테이너 타입의 시퀀스를 포함하고 있다.
시퀀스는 Iterable 과 동일한 기능을 제공하지만 다른 여러 콜렉션 처리에 대한 다른 접근방식을 구현한다.

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