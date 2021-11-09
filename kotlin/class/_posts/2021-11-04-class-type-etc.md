---
layout: single
title: "클래스의 종류"
---

자바와 코틀린의 클래스는 매우 유사하지만 자바에는 없는 몇몇 기능을 추가로 제공한다. 

### 데이터 클래스
---
자바에서 보통 데이터만 저장하고 있는 Vo 객체등을 떠올릴 수 있는데
데이터 클래스의 가장 큰 역할은 값을 쥐고 있고 인스턴스로 `getter/setter`를  제공한다는 점이다.
보통 자바였다면 

```java
class Person {

    private String name;
    private String address;

    public Person(String name, String address) {
        this.name = name;
        this.address = address;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getAddress() {
        return address;
    }

    public void setAddress(String address) {
        this.address = address;
    }
}
```

이런식의 클래스를 떠올릴거다.  문제는  코드량도 많고 여기에는 객체 비교를 할수 있는 equals() 라던가
hashCode() 메서드를 추가 구현해야한다. 물론 보통 IDE에서 이런 메서드를 자동 제공해서 어렵지 않게 추가할 수 있지만 그건 어디까지나 equals , hashCode 메서드를 생성하는 시점에 정의된 필드를 기준으로 생성 된 것이므로
차후 필드가 추가될 시 equals라던가 hashCode 메서드를 갱신해줘야하고 갱신하는 절차를 누락시 버그가 날 가능성이 높다.


필드가 늘어날 경우 또 이 로직을 수정해야 할것이다.
그래서 코틀린에선 이런 불편을 덜어주기 위해 데이터 클래스를 제공한다. 
data class로 선언하고 구성하고 있는 프로퍼티만 써주면 자동으로 toString(), hashCode(), equals() 함수를 생성해준다.

```kotlin
data class Person(val name:String, val address:String)
```

### 한정 클래스(sealed class)
---
한정 클래스는 enum 클래스를 확장한 개념을 가진 클래스로 코틀린에만 있는 개념이다. 각 종류별 
하나의 인스턴스로만 생성되있는 enum 클래스와 달리 인스턴스를 여러 개 생성이 가능하다. 
enum의 특징을 그대로 가지고 있으므로 이를 상속하는 클래스는 한정 클래스로 정의되는 여러 종류 중 하나로 취급된다

```kotlin
sealed class SealedClass (val os :String , val packageName: String){
    class Android(os :String, packageName:String) : SealedClass(os, packageName)
    class IOS(os :String, packageName:String) : SealedClass(os, packageName)
}

fun who(app :SealedClass) = when(app){
    is SealedClass.Android -> print("android")
    is SealedClass.IOS -> print("IOS")
}
```

한정 클래스를 상속하는 클래스는 일반적으로 클래스 내에 중첩하여 선언한다. 하지만 같은 파일내에 정의한다면
클래스 외부에 선언도 가능하다.
한정 클래스로 정의된 클래스의 종류에 따라 다른 작업을 처리할 때 매우 유용한데 
예를 들어 위같은  한정 클래스가 존재할때 위에 선언된 android 와 ios 클래스가 다 표현되 있기 때문에 지금은 에러가 나지 않는다. 하지만 한정클래스에서 하나더 추가할 경우

```kotlin
sealed class SealedClass (val os :String , val packageName: String){
    class Android(os :String, packageName:String) : SealedClass(os, packageName)
    class IOS(os :String, packageName:String) : SealedClass(os, packageName)
    class Windows(os:String, packageName: String) : SealedClass(os,packageName)
}
```

when 절에 빨간줄이 간다.  
만약 일반 클래스의 경우 여기서 else 절을 처리했다 하면 문제가 되지 않지만 한두개가 아닌 여러 클래스가 추가되어 모두다 else 절로 처리 되기때문에 컴파일 되지 않아 이에 대한 처리가 누락될 가능성이 높다.

하지만 한정 클래스로 선언하고 else 절을 사용하지 않도록 변경시 새로운 클래스가 추가되었을때
이를 처리하지않으면 컴파일 에러가 발생하므로 새로운 유형에 처리누락을 예방 할 수 있다.





