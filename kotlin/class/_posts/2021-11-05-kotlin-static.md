---
layout: single
title: "Kotlin에는 Static이 존재하지 않는다."
---


### Static은 어디로 갔을까? 그리고 왜 사라졌을까?

최근 현대 언어들은 static 이 `Primitive Type` 의 명시적 지원중단 같은 이유로
static을 삭제 했다. 

자바의 static 의 경우 필드와 메소드가 그들의 인스턴스가 아닌 클래스를 통해 접근이 가능하다.
static 멤버들은 instance 멤버들과 분리되어 있는것이다. 그리고 다른 규칙으로 적용된다.

||Instance methods|Static methods|
|---|---|---|
|Scope|Instance|Class
|Can override open methods in superclass|Yes|No (but can shadow them)|
|Can implement methods in interface|Yes|No|
|Dispatch|Dynamic, by actual runtime type|Static, by type known at compile-time|

클래스에서 한편으론 각 인스턴스가 동적으로 행동하고 한편으론 전역에 정적으로 있다는것.
즉 생성자가 합쳐진 두개의 다른개념을 가진다는것은 굉장히 난해하다.

그래서 코틀린에서는 클래스는 인스턴스로의 행동만 정의한다. 
정적인 값이나 전역 함수은 클래스 밖으로 분리해서 정의한다.

결국 Kotlin 은 static이 존재 하지 않는다. 대신에 이것들을 활용해야 한다.

- Top-Level Function
- Constants
- Object Instance
- Companion Objects

### Top-Level Function

자바의 경우 클래스 내부에 정의했겟지만 코틀린은 Top-Level Function 을 제공한다. 
어디든 .kt 파일에 정의하면 소스가 내부에서 자동 생성된다.

*Kotlin*

*TopLevelFunction.kt*
```java
fun topLevelTest() {}

```

*Java*
```java
public final class TopLevelFunctionKt {

   public static final void topLevelTest() {
   }
}

```

자바 내부적으로 파일명Kt 라는 클래스를 만들어 static 변수와 함수를 만들어준다.
이러한 점을 활용해 여러가지에 응용할 수 있다. 예를 들면 유틸 클래스를 대체할수 있다.
확장함수까지 사용하면 더 유용하게 사용할 수 있지만 여기서는 다루지 않는다.

*Kotlin*
```java
//Java
public class StringUtils {
  private StringUtils() { /* Forbid instantiation of utility class */ }   

  public static int lowerCaseCount(String value) {
      return value.chars().reduce(0, 
              (count, current) -> count + (Character.isLowerCase(current) ? 1 : 0));
  }
```

*Java*
```java
package topLevel

fun lowerCaseCount(value: String): Int = value.count { it.isLowerCase() }

```

Top-Level Function 과 프로퍼티는 패키지와 연관되어있다. 즉 2개의 Top-Level Function 와 프로퍼티를 
같은 이름으로 같은 패키지의 다른파일에서 생성할수는 없다. 


### Constants (상수)

프로퍼티 또한 Top-Level 에 선언이 가능하다. 하지만 자바랑 다르게 상수나 변하지 않는 값(val) 만 
사용하는걸 권장한다.

*Kotlin*

*TopLevelFunction.kt*
```java
const val CONSTANT_STRING = "CONSTANT"
val READONLY_LIST = listOf("value1", "value2")

// NOT OK: avoid public mutable top-Level properties
var mutableValue = "currentValue"
val mutableList = mutableListOf("value1", "value2")
```

*Java*
```java
public final class TopLevelFunctionKt {
   @NotNull
   public static final String CONSTANT_STRING = "CONSTANT";
   @NotNull
   private static final List READONLY_LIST = CollectionsKt.listOf(new String[]{"value1", "value2"});
   @NotNull
   private static String mutableValue = "currentValue";
   @NotNull
   private static final List mutableList = CollectionsKt.mutableListOf(new String[]{"value1", "value2"});

   @NotNull
   public static final List getREADONLY_LIST() {
      return READONLY_LIST;
   }

   @NotNull
   public static final String getMutableValue() {
      return mutableValue;
   }

   public static final void setMutableValue(@NotNull String var0) {
      Intrinsics.checkNotNullParameter(var0, "<set-?>");
      mutableValue = var0;
   }

   @NotNull
   public static final List getMutableList() {
      return mutableList;
   }
}

```

만약 Kt가 붙는게 싫으면 package 위쪽에 
annotation `@file:JvmName:` 을 사용하면 된다.

> ex) @file:JvmName("TopLevelFunction")

### Object singleton 

코틀린은 Object 선언을 통해 싱글톤 패턴을 지원한다. 

*Kotlin*
```java
object ObjectTest{}
```

*Java*
```java
public final class ObjectTest {
   @NotNull
   public static final ObjectTest INSTANCE;

   private ObjectTest() {
   }

   static {
      ObjectTest var0 = new ObjectTest();
      INSTANCE = var0;
   }
}
```
Object를 선언한것 만으로도 내부적으로 Singleton 인스턴스를 만들어준다.
Kotlin 에선 기본적으로 클래스에 `open` 을 쓰지않으면 `final` 클래스로 생성되지만
Object는 원래 `open` 제어자를 붙일수 없게 되어있으므로 기본 `final`로 생성되고 
내부에는 `static 초기화 블럭`을 사용해 인스턴스를 만들어두고 사용한다.
Object 는 클래스나 인터페이스로 확장도 가능하다.

*Kotlin*
```java
interface Counter {
    fun count(value: String): Int
}

object LowerCaseCounter : Counter { // can implement an interface
    override fun count(value: String) = value.count { it.isLowerCase() }
}
object UpperCaseCounter : Counter { // another implementation of the same interface
    override fun count(value: String) = value.count { it.isUpperCase() }
}

fun main() {
    // Functions on singleton objects can be called like Java static methods
    println(LowerCaseCounter.count("Hello World")) // prints 8
    println(UpperCaseCounter.count("Hello World")) // prints 2
    // But singletons are values and can be assigned to variables and passed as arguments
    val someCondition = true
    val counter = if (someCondition) LowerCaseCounter else UpperCaseCounter
    println(counter.count("Hello Kotlin Everywhere")) // dynamic dispatch
}
```

*Java*
```java

// Counter.java
public interface Counter {
   int count(@NotNull String var1);
}

public final class LowerCaseCounter implements Counter {
   @NotNull
   public static final LowerCaseCounter INSTANCE;

   public int count(@NotNull String value) {
      Intrinsics.checkNotNullParameter(value, "value");
      CharSequence $this$count$iv = (CharSequence)value;
      int $i$f$count = false;
      int count$iv = 0;
      CharSequence var5 = $this$count$iv;

      for(int var6 = 0; var6 < var5.length(); ++var6) {
         char element$iv = var5.charAt(var6);
         int var9 = false;
         boolean var11 = false;
         if (Character.isLowerCase(element$iv)) {
            ++count$iv;
         }
      }

      return count$iv;
   }

   private LowerCaseCounter() {
   }

   static {
      LowerCaseCounter var0 = new LowerCaseCounter();
      INSTANCE = var0;
   }
}

public final class UpperCaseCounter implements Counter {
   @NotNull
   public static final UpperCaseCounter INSTANCE;

   public int count(@NotNull String value) {
      Intrinsics.checkNotNullParameter(value, "value");
      CharSequence $this$count$iv = (CharSequence)value;
      int $i$f$count = false;
      int count$iv = 0;
      CharSequence var5 = $this$count$iv;

      for(int var6 = 0; var6 < var5.length(); ++var6) {
         char element$iv = var5.charAt(var6);
         int var9 = false;
         boolean var11 = false;
         if (Character.isUpperCase(element$iv)) {
            ++count$iv;
         }
      }

      return count$iv;
   }

   private UpperCaseCounter() {
   }

   static {
      UpperCaseCounter var0 = new UpperCaseCounter();
      INSTANCE = var0;
   }
}
// TopLevelFunction.java
package kuma.crawler.kotlinstudy;

import kotlin.Metadata;
import kotlin.jvm.JvmName;
@JvmName(
   name = "TopLevelFunction"
)
public final class TopLevelFunction {
   public static final void main() {
      int var0 = LowerCaseCounter.INSTANCE.count("Hello World");
      boolean var1 = false;
      System.out.println(var0);
      var0 = UpperCaseCounter.INSTANCE.count("Hello World");
      var1 = false;
      System.out.println(var0);
      boolean someCondition = true;
      Counter counter = (Counter)LowerCaseCounter.INSTANCE;
      int var4 = counter.count("Hello Kotlin Everywhere");
      boolean var2 = false;
      System.out.println(var4);
   }

   // $FF: synthetic method
   public static void main(String[] var0) {
      main();
   }
}
```

내부에 실제 함수들이 다 구현되어있고 실제 호출은 해당 싱글톤을 통해 만든 
실제 Instance.count 로 호출되고 있다. 

### Companion Objects 

`Object` 선언 자체로도 괜찮지만 만약 자바 클래스의 static 함수가 
내부의 private 멤버들에 접근을 하려면 어떻게 해야할까? 

*Kotlin*
```java
class Rocket private constructor() {
    private fun ready() {}

    companion object {
        fun build(): Rocket {
            val rocket = Rocket()  // can call private constructor
            rocket.ready() // can call private Function
            return rocket
        }
    }

    fun main() {
        val rocket = Rocket.build() // Companion function called using the accompanied class name
    }
}
```

*Java*
```java
public final class Rocket {
   @NotNull
   public static final Rocket.Companion Companion = new Rocket.Companion((DefaultConstructorMarker)null);

   private final void ready() {
   }

   public final void main() {
      Rocket rocket = Companion.build();
   }

   private Rocket() {
   }

   public Rocket(DefaultConstructorMarker $constructor_marker) {
      this();
   }

   public static final class Companion {
      @NotNull
      public final Rocket build() {
         Rocket rocket = new Rocket((DefaultConstructorMarker)null);
         rocket.ready();
         return rocket;
      }

      private Companion() {
      }

      // $FF: synthetic method
      public Companion(DefaultConstructorMarker $constructor_marker) {
         this();
      }
   }
}
```

private 으로 생성자를 만들경우 해당 Rocket 클래스는 외부에선 생성할 수 없다.
보통 `Builder Pattern`에서 자주 쓰이는 방식인데 외부에서 직접 해당 클래스의 생성자를 만드는게 아니라
내부 클래스의 build 메소드를 통해 해당 클래스의 인스턴스를 얻는것이다. 여기선 `Companion Object`를 사용해 
내부에 static class 를 구현하고 코틀린상에선 Rocket 으로 접근해도 build 메소드에 접근할수 있다.
물론 실제로는 `Rocket.Companion.build()` 지만 코틀린자체에선 생략해서
`Rocket.build()`로 표기하거나 `Rocket.Companion.build()` 해도 상관없다 으로 붙여도 상관없다.
만약 함수에 `@JvmStatic` 을 붙일 경우 자바에서도 Rocket.Companion이 아닌 Rocket 으로 바로 붙을 수 있다.

*Kotlin*
```java
companion object {
        @JvmStatic
        fun build(): Rocket {
            val rocket = Rocket()  // can call private constructor
            rocket.ready() // can call private function
            return rocket
        }
    }
```
*Java*
```java
Rocket rocket = Rocket.build();
```



### 결론

static 을 제거함으로 Kotlin은 개념이 섞이는걸 피하고 더 강력하고(Object , Companion 객체) 
 더 적게 표기하지만 같은 기능 (Top-Level Functions And Properties 등)을 사용할 수 있다.
하지만 코틀린을 처음 사용하는 경우 static을 사용하는 유즈케이스 등을 통해 익숙해고 배울 필요가 있다.

### 참조

[https://jelmini.dev/post/from-java-to-kotlin-life-without-static/](https://jelmini.dev/post/from-java-to-kotlin-life-without-static/)

[https://kotlinlang.org/docs/functions.html#local-functions](https://kotlinlang.org/docs/functions.html#local-functions)

[https://kotlinlang.org/docs/object-declarations.html#object-declarations-overview](https://kotlinlang.org/docs/object-declarations.html#object-declarations-overview)

[https://kotlinlang.org/docs/object-declarations.html#companion-objects](https://kotlinlang.org/docs/object-declarations.html#companion-objects)

