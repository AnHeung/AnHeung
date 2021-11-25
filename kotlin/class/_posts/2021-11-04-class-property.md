---
layout: single
title: "클래스 프로퍼티(property)"
---

### Getter/Setter
---

코틀린은 내부에 저장된 필드 값을 가져오거나 설정할 수 있도록 Getter 및 Setter를 내부적으로 구현하고 있고
사용자 지정 Getter/Setter를 사용시 프로퍼티에서 사용자가 원하는 방향으로 구현도 할수 있다.
기본적인 문법은 get() 및 set(value)를 사용하여 선언한다.

*Kotlin*
```kotlin
class Person(val name:String, val age:Int)
```

코틀린으로 된 소스를 자바소스로 디컴파일을 해보면 실제 내부에서 getter setter를 만들어줌을 알수 있다.

*Java*
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

*Kotlin*
```java
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

여기서 field는 자기자신값 여기서는 age 가 되겠다.
그리고 set에 들어가는 value는 set함수에 넘기는 파라미터다. 
실제 디컴파일 해보면 쉽게 알 수 있다.

*Java*
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

### Compile-Time Constants
---
읽기 전용 프로퍼티 (val)가 컴파일시 지정되려면 `const` 제어자를 사용하면 된다.  
const 를 쓰기 위해선 몇가지 수행조건이 있는데

- `Top-Level` 프로퍼티로만 사용 가능하거나 `Object` 선언 또는 `Companion Object` 에서만 쓸 수 있다. 
- String 이나 다른 primitive 타입으로만 초기화 해야한다.
- 커스텀 getter를 설정 할 수 없다.

const 를 쓰면 실제 내부에 클래스에 static final 변수명 으로 설정됨을 알 수 있다.
선언부분과 실제 자바코드로 어떻게 나오는지는  해당부분을 참조하면 되겠다. [Kotlin에는 Static이 존재하지 않는다.](/kotlin/class/kotlin-static/#constants-상수)


### Late-initialized properties and variables
---
보통의 프로퍼티는 생성자에서 non-null 타입으로 반드시 초기화가 되서 선언되어야 하지만
이것은 종종 불편할때가 많다.

예를들어 Dependency Injection(DI) 또는 추후 setter 로 넣어줄 값들의 경우 초기화를 꼭 할 필요가
없어도 생성자에 non-null 초기화가 존재하지 않으므로 꼭해야한다. 그래서 이런 경우에
`lateinit` 제어자를 kotlin 에서는 제공해준다.  `참고로 lateinit은 Primitive 타입은 사용할 수 없다.`

*Kotlin*
```java
class Test{
    lateinit var test:String
}
```

일반적인 var를 쓸경우 상태를 check 하지 않고 일반적인 `getter/setter` 가 만들어지지만
lateinit 을 사용할 경우 상태를 check하고 없을 경우 `UninitializedPropertyAccessException` 발생한다.
> <span style="color:red">kotlin.UninitializedPropertyAccessException: lateinit property adapter has not been initialized</span>

*Java*
```java
public final class Test {
   public String test;

   @NotNull
   public final String getTest() {
      String var10000 = this.test;
      if (var10000 == null) {
         Intrinsics.throwUninitializedPropertyAccessException("test");
      }

      return var10000;
   }

   public final void setTest(@NotNull String var1) {
      Intrinsics.checkNotNullParameter(var1, "<set-?>");
      this.test = var1;
   }
}
```

그래서 lateinit 의 경우 초기화가 됫는지 안됫는지 알수 있는 상태값이 있는데
`.isInitialized` 를 사용하면 알수 있다.

*Kotlin*
```java
class Test{
    lateinit var testVal :String
    fun isInitialize() : Boolean = this::testVal.isInitialized
}
```
*Java*
```java
public final boolean isInitialize() {
      return ((Test)this).testVal != null;
}
```


### 참조
---
[https://kotlinlang.org/docs/properties.html](https://kotlinlang.org/docs/properties.html)


