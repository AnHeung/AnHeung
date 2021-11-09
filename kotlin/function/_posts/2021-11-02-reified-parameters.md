---
layout : single
title : Reified Parameters
---

### 제네릭과 런타임
---
자바에는 제네릭이 지원 안됬을때가 있었다. (Java 5 버전부터 추가되었다.)
그러기 때문에 하위 버전과의 호환성 유지때문에 Type Erasure 를 사용했다.
Type Erasure는 원소 타입을 컴파일 타입에만 검사하고 런타임에는 해당 타입 정보를 알 수 없다.
즉 컴파일 타임에만 타입 제약 조건을 정의하고, 런타임에는 타입을 제거한다는 뜻입니다.

코틀린도 자바와 마찬가지도 Type Erasure를 구현했다. 예제를 보면

*Kotlin*
```java
fun <T> getCount(list : List<T>) : Int = list.size
```

*Java*
```java
public static final int getCount(@NotNull List list) {
  Intrinsics.checkParameterIsNotNull(list, "list");
  return list.size();
}
```

위에서 언급했듯 컴파일 타임에 Type은 지워지고 타입이 기본 타입로 대체된다.
List가 제네릭 List에서 어떠한 정보도 없는 기본 리스트로 바뀌었다.

### Type erasure 결과
---
타입 정보가 런타임에 삭제되었기 때문에 매개변수화 된 클래스에 `is` 를 사용해 check 하는것은 안된다.
이미 런타임에서 기본Type 으로 변경되었기 때문이다. 밑에 예제는 컴파일 에러가 난다.

*Kotlin*
```java
fun <T> isListOfString(arg : T) : Boolean{
    return arg is List<String>
}
```

*Java*
```java
public static final boolean isListOfString(Object arg) {
      return arg instanceof List;
}
```
> Cannot check for instance of erased type: List\<String>

기본타입이 무엇인지는 확인할 수 있지만 그것의 타입 매개변수는 알수가 없다.
즉 List 인지는 알수 있지만 무슨 List 인지는 알수 없단 말이다.
그럼 `star projection (*)` 을 사용해 체크해보자

```java
fun <T> isList(arg: T): Boolean {
    return arg is List<*>
}
```

이렇게 하면 컴파일 에러는 나지 않지만 무조건 true가 나온다.하지만 타입 매개변수가 일치하는지 까지는
보장할 수 없다.

### Reification(구체화)

구체화는 런타임에서 제네릭 타입을 예방해준다. 코틀린은 인라인 함수라는 특정 조건에서 
타입 매개변수의 구체화를 지원한다.

*Kotlin*
```java
inline fun <reified T> doSomethingWithType(obj: T) {
    val typeName = T::class.java
    println(typeName)
}

fun main(args: Array<String>) {
    doSomethingWithType(String())
}
```
반드시 인라인 함수로 선언하고 타입 매개변수를 refied로 해야한다.
이제 해당 코드를 디컴파일 해서 보자.

*Java*
```java
private static final void doSomethingWithType(Object obj) {
  Intrinsics.reifiedOperationMarker(4, "T");
  Class typeName = Object.class;
  System.out.println(typeName);
}

public static final void main(@NotNull String[] args) {
  Intrinsics.checkParameterIsNotNull(args, "args");
  new String();
  Class typeName$iv = String.class;
  System.out.println(typeName$iv);
}
```
`doSomethingWithType` 함수는 우리가 기대한대로 타입을 삭제했다.
위의 함수는 인라인 함수고 Object 타입 대신 정확한 타입 대체가 이루어 졌다.
이게 가능한 이유는 호출된 구역에서 컴파일러가 인라인 함수로 부터 넘어온 객체 타입을 추론했기 때문이다.

### 현실적인 사용
---
현실적으로 reification 은  `reflection`,`타입 체크`,`형변환` 등에 쓰인다.
reification 을 사용해 타입을 함수의 인자로 받는게 아니라 `<reified T>` 같이 함수에 선언해서 
타입을 유추하는등에 쓸 수 있다.

### 참조

[https://medium.com/tompee/idiomatic-kotlin-reified-parameters-e89f665ab026](https://medium.com/tompee/idiomatic-kotlin-reified-parameters-e89f665ab026)
