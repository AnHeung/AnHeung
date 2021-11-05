---
layout: single
title: "클래스 프로퍼티(property)"
---

### Getter/Setter
---

코틀린은 내부에 저장된 필드 값을 가져오거나 설정할 수 있도록 Getter 및 Setter를 내부적으로 구현하고 있고
사용자 지정 Getter/Setter를 사용시 프로퍼티에서 사용자가 원하는 방향으로 구현도 할수 있다.
기본적인 문법은 get() 및 set(value)를 사용하여 선언한다.

```kotlin
class Person(val name:String, val age:Int)
```

코틀린으로 된 소스를 자바소스로 디컴파일을 해보면 실제 내부에서 getter setter를 만들어줌을 알수 있다.

```java
public final class Person {
   @NotNull
   private final String name;
   private final int age;

   @NotNull
   public final String getName() {
      return this.name;
   }

   public final int getAge() {
      return this.age;
   }

   public Person(@NotNull String name, int age) {
      Intrinsics.checkNotNullParameter(name, "name");
      super();
      this.name = name;
      this.age = age;
   }
}
```

만약 여기서 자체 getter/setter 를 만들고 싶다면 `get(),  set()` 을 사용해서 커스텀을 하면 된다.

```kotlin
class Person(val name: String) {

    var age = 0
        set(value) {
            field = value
            field = if (field > 19) age + 1
            else age - 1
        }
        get() {
            println("호출됨")
            return field
        }
}
```

setter도 이와 비슷한데 직접 커스텀한다면 val 대신 변수인 var 를 사용해 
값을 받아야 한다는 차이가 있다.
(setter 를 쓴다는 개념 자체가 값이 변할 수 있다는 개념이라 val 로 할순없다.)

이런식으로 선언도 되는데 여기서 field는 자기자신값 여기서는 age 가 되겠다.
그리고 set에 들어가는 value는 set함수에 넘기는 파라미터다. 
실제 디컴파일 해보면 쉽게 알 수 있다.

```java
public final class Person {
   private int age;
   @NotNull
   private final String name;

   public final int getAge() {
      String var1 = "호출됨";
      boolean var2 = false;
      System.out.println(var1);
      return this.age;
   }

   public final void setAge(int value) {
      this.age = value;
      this.age = this.age > 19 ? this.getAge() + 1 : this.getAge() - 1;
   }

   @NotNull
   public final String getName() {
      return this.name;
   }

   public Person(@NotNull String name) {
      Intrinsics.checkNotNullParameter(name, "name");
      super();
      this.name = name;
   }
}
```

### Compile-time constants

읽기 전용 프로퍼티 (val)가 컴파일시 지정되려면 `const ` 제어자를 사용하면 된다.  
const 를 쓰기 위해선 몇가지 수행조건이 있는데

- top-level 프로퍼티로만 사용 가능하거나 `object` 선언 또는 `Companion Object` 에서만 쓸 수 있다. 
- String 이나 다른 primitive 타입으로만 초기화 해야한다.
- 커스텀 getter를 설정 할 수 없다.

Kotlin 은 static이 존재 하지 않기때문에 
- Top-level function
- Top-level constants
- Companion Object functions
- Object Instance

이것들을 활용해야 한다. 그리고 실제 해당 코드들은 디컴파일 해보면 내부 클래스 안에 static final 로 값이
설정 되어있다.


### 링크

[https://kotlinlang.org/docs/properties.html](https://kotlinlang.org/docs/properties.html)
[https://jelmini.dev/post/from-java-to-kotlin-life-without-static/](https://jelmini.dev/post/from-java-to-kotlin-life-without-static/)

