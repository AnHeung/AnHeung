---
layout: single
title: "프로퍼티(property)"
---

자바에서는 클래스 내에 자료를 저장하고 접근하기 위해 필드와 메서드를 사용했다.
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

사실 따지고보면 클래스에서 다루는 자료는 String 형 이름과 나이 두가지 뿐이다. 하지만 이 자료는 private 처리 되있고 접근하기 위해선 getter/setter 등 을 추가해야하므로
코드의 양은 불필요하게 증가했고 필드가 늘어날 수록 이에 따른 메서드가 배로 늘어나게 된다.
코틀린은 이런점은 개선하기 위해 프로퍼티(property)를 사용한다. 

```java
class Person {
    var name: String? = null
    var age: Int? = 0
}
```

언뜻 보기엔 public한 멤버변수에 접근해 값을 가져오고 바꾼것 처럼 보일수도 있지만 실제 바이트 코드상에는
다 private 하게 선언 되어있고 getter/setter 도 다 설정되어있는걸 볼 수 있다.

```java
public final class Person {
   @Nullable
   private String name;
   @Nullable
   private Integer age = 0;

   @Nullable
   public final String getName() {
      return this.name;
   }

   public final void setName(@Nullable String var1) {
      this.name = var1;
   }

   @Nullable
   public final Integer getAge() {
      return this.age;
   }

   public final void setAge(@Nullable Integer var1) {
      this.age = var1;
   }
}
```

그런데 프로퍼티로 변환된 코드를 유심히 보면 각 프로퍼티의 초기값이 `null`로 명시적 할당이 되있는걸 알수있다.
이처럼 코틀린에서 클래스의 멤버로 사용하는 프로퍼티는 반드시 초깃값을 명시적으로 지정해야하고 아닐 경우 컴파일 에러가난다.

만약 프로퍼티 초기화를 할수 없을 경우에는 lateinit 키워드를 허용하는데 
이 건 프로퍼티의 값이 나중에 할당될 것이란걸 명시하는거다.

`lateinit` 의 경우는 나중에 값이 들어오므로 값이 변하지 않는
val 대신 var만 사용이 가능하고 선언만 해놓으면 선언시점에 값을 할당하지 않아도
컴파일 에러는 뜨지 않는다.

만약 `lateinit` 키워드를 사용해 초기화 없이 사용할 경우 `UninitializedPropertyAccessException` 예외가 발생하는데 이걸 자바의 `NullpointException`과 마찬가지로 컴파일단에서 발견할 수 없고 
런타임에서 뜨는 문제다. 이 오류가 뜨면 초기화 변수부터 확인해봄이 좋다.