---
layout: single
title: "Kotlin에서 DSL 사용하기 파트2"
---

앞에 포스트 [Kotlin에서 DSL 사용하기 파트1](/kotlin/class/kotlin-dsl-part1/) 에서 
대략적인 DSL의 정의와 DSL 을 표현하기 위한 Kotlin 문법들을 활용해 보았다.

DSL은 가독성 증가와 복잡성을 줄이기 위해서 사용되어야한다.

그리고 이 파트에서는 3가지를 사용해 더 이어나가 보겠다.

* Builder pattern
* Collections
* Narrowing scope

이전 파트에서 2개의 기본적 `data classes` 를 만들었다. 하지만 해당 클래스들은 
실제 `mutable` 변수다. 즉 쉽게 프로퍼티에 접근에 값을 바꿀수 있다. 이걸 var 가 아닌
val로 변경시 컴파일 에러가 발생할 것이다. (val는 재할당이 안된다.) 이러한것을 피하기 위해
`Builder Patter`을 사용해 빌더 클래스를 만들것이다.

### Builder pattern

```java
data class Person(val name: String,
                  val dateOfBirth: Date,
                  var address: Address?)

data class Address(val street: String,
                   val number: Int,
                   val city: String)
```

이제 이것들에 대한 Object를 만들려면 생성자에 직접 주입해야한다. 하지만 여기서 빌더 클래스를
만들어서 내부에서 생성을 할것이다.


```java
fun person(block: PersonBuilder.() -> Unit): Person = PersonBuilder().apply(block).build()

class PersonBuilder {
  
    var name: String = ""

    private var dob: Date = Date()
    var dateOfBirth: String = ""
        set(value) {
            dob = SimpleDateFormat("yyyy-MM-dd").parse(value)
        }

    private var address: Address? = null

    fun address(block: AddressBuilder.() -> Unit) {
        address = AddressBuilder().apply(block).build()
    }

    fun build(): Person = Person(name, dob, address)

}

class AddressBuilder {

    var street: String = ""
    var number: Int = 0
    var city: String = ""

    fun build() : Address = Address(street, number, city)

}
```

Person과 Address 를 직접 Person() 이런식으로 만드는게 아니라 PersonBuilder()를 통해 인스턴스를 만들고
거기에 Lambda With Receiver 를 적용한 PersonBuilder 인자를 받는 함수를 만들었다.
Custom setter 를 사용해 date 객체의 포맷도 내부에서 바꾸도록 설정했다.

PersonBuilder 내부의 `build` 메소드를 통해 내부에서 인자들이 다 만들어지고 조립되지만 직접 Person 객체값을 접근할수 없게 되어있다. person {} 스코프 내부는 실제 PersonBuilder의 스코프이다. 

```java
val person = person {
    name = "John"
    dateOfBirth = "1980-12-01"
    address {
        street = "Main Street"
        number = 12
        city = "London"
    }
}
```

### Collections

이제 여기다 Collections 를 추가해본다 가정하자. 주소가 하나가 아니라 여러주소를 받아야해서  
List<Address> 라고 가정했을 경우

```java
data class Person(val name: String,
                  val dateOfBirth: Date,
                  val addresses: List<Address>)

class PersonBuilder {
  
  // ... other properties
  
  private val addresses = mutableListOf<Address>()

  fun address(block: AddressBuilder.() -> Unit) {
      addresses.add(AddressBuilder().apply(block).build())
  }

  fun build(): Person = Person(name, dob, addresses)
  
}

val person = person {
    name = "John"
    dateOfBirth = "1980-12-01"
    address {
        street = "Main Street"
        number = 12
        city = "London"
    }
    address {
        street = "Dev Avenue"
        number = 42
        city = "Paris"
    }
}
```

이것도 깔끔하지만 여기서 더 깔끔하게 처리하려면 address 메소드를 처리하기 위한 
helper class 를 만들면 된다. 

```java
fun address(block: AddressBuilder.() -> Unit) {
      addresses.add(AddressBuilder().apply(block).build())
  }
```

원래 이렇게 된 부분을

```java
fun addresses(block: ADDRESSES.() -> Unit) {
    addresses.addAll(ADDRESSES().apply(block))
}

class ADDRESSES: ArrayList<Address>() {

    fun address(block: AddressBuilder.() -> Unit) {
        add(AddressBuilder().apply(block).build())
    }

}
```
Address List만 처리하기 위한 ArrayList<Address> 클래스를 구현한 ADDRESSES 클래스를 만들어
해서 ADDRESSES.() -> Unit를 인자로 다시 넘겨서 실제 ADDRESSES가 처리하게 만든다. 

```java
val person = person {
    name = "John"
    dateOfBirth = "1980-12-01"
    addresses {
        address {
            street = "Main Street"
            number = 12
            city = "London"
        }
        address {
            street = "Dev Avenue"
            number = 42
            city = "Paris"
        }
    }
}
```

### Narrowing scope

여기까지 했으면 읽기도 좋고 유지보수도 용이하고 안전하게 DSL을 구현했다 할수 있다.
하지만 아직 한가지 이슈가 남았는데 람다안에 다른 람다 스코프를 계속 겹치게 만들으므로
내부에 있는 람다 스코프에서 바깥쪽 람다의 스코프에 접근하는게 가능해진다.
위의 식에선 addresses 내부에서 person의 name 값도 수정이 가능하단 말이다.

하지만 다행히도 코틀린1.1 이후로 *@DslMarker* annotation을 제공한다 그래서 이걸통해 
커스텀 annotation 클래스를 만들어 DSL할 클래스들에게 적용하면 된다.

```java
@DslMarker
annotation class PersonDsl

@PersonDsl
class PersonBuilder {
  //...
}

@PersonDsl
class ADDRESSES: ArrayList<Address>() {
  //...
}

@PersonDsl
class AddressBuilder {
  //...
}
```

이제 위의 person 처럼 내부 람다에서 외부람다의 값을 수정시  
밑에와 같은 컴파일 오류가 발생한다.

> <span style="color:red">fun addresses(block: ADDRESSES.() -> Unit): Unit' can't be called in this context by implicit receiver. Use the explicit one if necessary</span> 


### 실사용 사례 GsonBuilder 간략하게 만들기

serializing 과 deserializing 를 위한 gson 인스턴스를 만들기를 원할때 gson 자체에서 
내부적으로 설정할수 있도록 GsonBuilder 를 제공한다. 그러나 직렬화 혹은 역직렬화 부분을
건너뛰고 싶은 경우 상당히 지저분해 진다.

```java
val gson = GsonBuilder()
          .addDeserializationExclusionStrategy(object: ExclusionStrategy {
              override fun shouldSkipClass(clazz: Class<*>?): Boolean {
                  return clazz?.equals(Address::class.java) ?: false
              }

              override fun shouldSkipField(f: FieldAttributes?): Boolean {
                  return f?.let { it.name == "internalId" } ?: false
              }

          })
          .addSerializationExclusionStrategy(object: ExclusionStrategy {
              override fun shouldSkipClass(clazz: Class<*>?): Boolean {
                  return false
              }

              override fun shouldSkipField(f: FieldAttributes?): Boolean {
                  return f?.let { it.declaringClass == Person::class.java && it.name == "address" } ?: false
              }

          })
          .serializeNulls()
          .create()
```

이걸 DSL을 활용해 더 읽기좋고 간략하게 표현한다면 이렇게 표현이 가능할 것이다.

```java
val gson = gson {
    dontDeserialize {
        whenField { name == "internalId" }
        whenClass { equals(Address::class.java)}
    }
    dontSerialize {
        whenField { declaringClass == Person::class.java && name == "address" }
    }
    serializeNulls()
}
```

밑에는 구현식이다. ExclusionStrategyBuilder 를 리시버로 받는 인자로 받는 
dontDeserialize 또는 dontSerialize 메소드를 만들어 내부적으로 ExclusionStrategyBuilder를 통해
인스턴스를 만들어 준다. 실제 함수를 가져다 쓰는 사람은 ExclusionStrategyBuilder 안의 whenField whenClass 등을 assign 하게 되어있다.

최종적으로 build 함수를 통해 ExclusionStrategy 객체를 조립을 하여 ExclusionStrategy 타입으로 반환해 완성된 strategy들을 GsonBuilder의 addSerializationExclusionStrategy , addDeserializationExclusionStrategy 등에 추가한다.

```java
fun gson(block: GsonBuilder.() -> Unit): Gson = GsonBuilder().apply(block).create()

fun GsonBuilder.dontDeserialize(block: ExclusionStrategyBuilder.() -> Unit) {
    val strategy = ExclusionStrategyBuilder().apply(block).build()
    this.addDeserializationExclusionStrategy(strategy)
}

fun GsonBuilder.dontSerialize(block: ExclusionStrategyBuilder.() -> Unit) {
    val strategy = ExclusionStrategyBuilder().apply(block).build()
    this.addSerializationExclusionStrategy(strategy)
}

class ExclusionStrategyBuilder {

    private var field: (FieldAttributes) -> Boolean = { false }
    private var clazz: (Class<*>) -> Boolean? = { false }

    fun whenField(block: FieldAttributes.() -> Boolean) {
        field = block
    }

    fun whenClass(block: Class<*>.() -> Boolean) {
        clazz = block
    }

    fun build(): ExclusionStrategy {
        return object : ExclusionStrategy {
            override fun shouldSkipClass(clazz: Class<*>?): Boolean {
                return clazz?.let { clazz(it) } ?: false
            }

            override fun shouldSkipField(f: FieldAttributes?): Boolean {
                return f?.let { field(it) } ?: false
            }

        }
    }
}
```


### 참조

[https://proandroiddev.com/writing-dsls-in-kotlin-part-2-cd9dcd0c4715](https://proandroiddev.com/writing-dsls-in-kotlin-part-2-cd9dcd0c4715)

[https://developer.android.com/guide/navigation/navigation-kotlin-dsl?hl=ko](https://developer.android.com/guide/navigation/navigation-kotlin-dsl?hl=ko)

