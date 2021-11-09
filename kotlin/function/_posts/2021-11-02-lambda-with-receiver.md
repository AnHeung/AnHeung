---
layout : single
title : Lambda With Receiver
---

### 리시버가 있는 람다함수

lambda with a receiver 는 간결하고 읽기 좋게 만들어준다. 
DSL 같은 구조적 언어에 적절하고 중첩 구조를 짜기 좋은 기능들을 제공한다.

매개변수로 람다를 함수로 사용하는 구문을 예를 들면

```java
fun encloseInXMLAttribute(sb : StringBuilder, attr : String, action : (StringBuilder) -> Unit) : String{
    sb.append("<$attr>")
    action(sb)
    sb.append("</$attr>")
    return sb.toString()
}
```

위의 함수는 StringBuilder 인스턴스와 속성이름 마지막으로 람다 표현식을 인자로 받는다.
이걸 호출해서 표현하면

```java
val xml = encloseInXMLAttribute(StringBuilder(), "attr") {
    it.append("MyAttribute")
}
print(xml)

```

위의 코드의 결과는 이렇게 나온다.

> \<attr>MyAttribute</attr>

현재 함수타입으로써 람다는 `(StringBuilder) -> Unit`  즉 StringBuilder 타입의 값이 인자로 들어와
해당 람다에서 it 이나 혹은 내가 명시적으로 등록한 이름으로 표시할수 있다.
(위에는 it으로 호출되었다.)

`{ it.무언가 } {builder->...}`

지금도 가독성이 나쁜건 아니지만 조금더 가독성을 높이기 위해 이제 마지막 인자부분을 확장함수 타입으로 바꿔보자.
우리는 `Type.method` 구문을 사용해 해당 타입의 인스턴스를 첫번째 파라미터로 받을 수 있는 확장함수 타입으로 선언할 수 있다.

```java
fun encloseInXMLAttributeExtension(sb : StringBuilder, attr : String, action : StringBuilder.() -> Unit) : String{
    sb.append("<$attr>")
    sb.action()
    sb.append("</$attr>")
    return sb.toString()
}
```

`(StringBuilder) -> Unit` 이부분이 `StringBuilder.() -> Unit` 으로 변경되었다.
내부의 함수식도 action(sb) 에서 sb.action() 으로 변경되었다. action 파라미터가 StringBuilder 의 가상의 확장 함수로
사용되어져 호출되었다. 즉 action은 StringBuilder 의 인스턴스를 받는다는 것이다. 
즉 앞에서 함수타입의 람다에서 인자를 it 이나 명시한 이름으로 받지 않고 this(생략가능) 자체로 받을 수 있다.

```java
val xml = encloseInXMLAttributeExtension(StringBuilder(), "attr") {
    append("MyAttribute")
}
println(xml)
```

이 코드에서 람다가 receiver 로써 활용된 것이다. 람다 정의 구역에서(괄호안) 메소드 호출이 단조로워 졌다.
추가로 코틀린 표준 라이브러리 2개를 더 살펴보자

### with, apply
---
```java
public inline fun <T, R> with(receiver: T, block: T.() -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return receiver.block()
}

public inline fun <T> T.apply(block: T.() -> Unit): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    block()
    return this
}
```

`with` 함수는 receiver 로 인스턴스와 람다 확장함수를 인자로 받는다. 우리가 위에 예시로 만든 함수와 유사하다. 
이 함수는 인자로 받아서 동작을 수행하고 다른 타입을 반환하는 용도로 많이 쓴다.

`apply`는 반면에 확장함수 타입으로 인자를 하나만 받는다. 이 함수는 어떠한 동작을 리시버 객체에서 수행하고 
동일한 객체를 반환받는 함수다.

### 리시버가 있는 람다로 DSL 구축

해당부분은 [Kotlin에서 DSL 사용하기](/kotlin/class/kotlin-dsl-part1/) 을 참조하세요.

### 참조

[https://medium.com/tompee/idiomatic-kotlin-lambdas-with-receiver-and-dsl-3cd3348e1235](https://medium.com/tompee/idiomatic-kotlin-lambdas-with-receiver-and-dsl-3cd3348e1235)


