---
layout: single
title: "Kotlin에서 DSL 사용하기 파트1"
---

코틀린은 코드를 읽기 좋게 더불어 BoilerPlate 코드를 줄이기 위해 많은 기능들을 제공한다.
그런것들중 하나가 [도메인 특화 언어](https://ko.wikipedia.org/wiki/%EB%8F%84%EB%A9%94%EC%9D%B8_%ED%8A%B9%ED%99%94_%EC%96%B8%EC%96%B4) 이다. 

위키에 정의된 대로 특정 도메인을 적용하는데 특화된 컴퓨터 언어 로 `SQL`도 DSL중 하나이다.
가장 큰 장점으론 컴퓨터 전문가가 아니라 해도 이해하기 쉬운 언어라는 점이지만 어쨋건 새 언어를 배워야한다는 
초기비용이 많이 들고 적용 분야가 매우좁다는 단점이 있다.

그렇다면 코틀린에서 DSL 을 써서 얻는 장점이 무엇이 있을까? 코틀린에서 DSL 은 무언가 독립적인 문법을 써서
만든다는게 아니라 kotlin 자체로 명시적으로 설정 하는것이다. 여기선 밑의 3가지를 사용할 것이다.

- 메소드 괄호 외부에서 람다 사용
- [Lambda with receiver](/kotlin/function/lambda-with-receiver/)
- 확장 함수

DSL을 사용함에 있어 위의 것들은 불필요한 기호를 사용하지 않도록 깔끔한 문법을 제공해준다.

### DSL 써보기

```java
val person = person {
    name = "John"
    age = 25
    address {
        street = "Main Street"
        number = 42
        city = "London"
    }
}
```

코드에서 보이듯 코드자체가 설명적이고 비전문가도 이해하기 쉽게 써있다.  
좀더 이해하기 위해서 일단 Model을 만들어 보자

```java
data class Person(var name: String? = null,
                  var age: Int? = null,
                  var address: Address? = null)


data class Address(var street: String? = null,
                   var number: Int? = null,
                   var city: String? = null)
```

### 메소드 괄호 외부에서 람다 사용

생성자를 통해 Person 객체를 만들고  Person의 프로퍼티는 코드 내부에서 정의하고 있다.
위에서 설명한 `메소드 괄호 외부에서 람다`를 사용 하였다.

```java
fun person(block: (Person) -> Unit): Person {
    val p = Person()
    block(p)
    return p
}
```
Person 객체를 만들고 내부에서 람다를 수행한뒤 다시 Person 객체를 돌려주는 함수가 완성되었다.
이제 이걸 활용해보자

```java
val person = person {
    it.name = "John"
    it.age = 25
}
```

위에 person 함수는 람다 block을 인자로 받는데 위에 정의했듯 Person 을 인자로 받고 
어떤 동작을 수행하고 Unit 타입을 반환하는 람다를 인자로 받는다. [코틀린 스타일 가이드](https://kotlinlang.org/docs/coding-conventions.html#lambdas)를 보면 
인자로 받는 람다는 소괄호 안으로 중괄호를 넣거나 소괄호 밖으로 중괄호를 빼거나 소괄호도 쓰지말고 중괄호만 쓰는 방식 3가지로 표현할수 있는데 여기선 DSL을 사용해 간결하게 표현해야 하므로 중괄호만 표기한것이다.

```java
person(){ person -> }

person({})

person{}
```

이 방식도  충분히 깨끗하지만 DSL을 표현하려면 좀더 간결함이 필요하다 여기서 필요한 기능이 
`Lambda with receiver`를 사용하면 된다.

위에 정의한 person 함수를 리시버로써 람다로 변경해보자

### Lambda with receiver 사용

```java
fun person(block: Person.() -> Unit): Person {
    val p = Person()
    p.block()
    return p
}
```

앞에서 처럼 Person을 인자로 받는 람다함수가 아닌 Person 자체에 람다가 리시버로 추가된 모습이다.
이걸 통해 우린 인자로 Person 객체를 넣지않아도 접근할수 있게 된다. 

일반 lambda는 호출시에 객체를 Argument로 받지만 `(block : (T) -> R)`
Lambda with Receiver는 호출시에 객체를 Receiver로 받습니다  `(block : T.() -> R)` 

여기서 더 깔끔하게 줄일려면 `apply` 함수를 사용하면 된다.

```java
inline fun <T> T.apply(block: T.() -> Unit): T {this.block(); return this;}
```
apply 도 Lambda with receiver 개념이므로 호출하는 객체 T의 Receiver로 람다를 받으므로 
apply의 인자로 Person.() -> Unit 를 그대로 넣으면 더 깔끔하게 처리된다.

```java
fun person(block: Person.() -> Unit): Person = Person().apply(block)
```

Person을 인자로 넣었을 경우 it.name 이런식으로 접근했으나 위처럼 바꾸고 나면 
마치 name = "John" (this는 생략) 이런식으로 깔끔하게 표현할수 있다.

```java
val person = person {
    name = "John"
    age = 25
}
```

지금도 충분히 깔끔하지만 아직 위에 명시한 Address 클래스를 추가 하지 않았다. Person 객체에
Address 프로퍼티만 넣으면 되는데 이걸 하기 위해서 `확장 함수`를 이용한다.

`확장 함수`는 자체 소스코드의 접근없이도 함수의 기능을 추가해준다. 

### 확장 함수 사용

```java
fun person(block: Person.() -> Unit): Person = Person().apply(block)

fun Person.address(block: Address.() -> Unit) {
    address = Address().apply(block)
}
```

Address를 Lambda with Receiver 로  Pseron의 확장함수로 address를 추가했다.
이로써 Person의 프로퍼티로써 Address 객체가 set 된다.

```java
val person = person {
    name = "John"
    age = 25
    address {
        street = "Main Street"
        number = 42
        city = "London"
    }
}
```

### 참조

[https://proandroiddev.com/writing-dsls-in-kotlin-part-1-7f5d2193f277](https://proandroiddev.com/writing-dsls-in-kotlin-part-1-7f5d2193f277)

[https://developer.android.com/guide/navigation/navigation-kotlin-dsl?hl=ko](https://developer.android.com/guide/navigation/navigation-kotlin-dsl?hl=ko)

