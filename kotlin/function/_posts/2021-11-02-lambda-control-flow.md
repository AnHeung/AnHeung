---
layout : single
title : 람다의 동작 흐름
---

### Lambda returns
---
먼저 볼 부분은 람다의 return 문이다. 람다 내부에서 어떻게 return 문이 동작하는지
예시를 먼저 보자

```java
fun higherOrderFunction(lambda : () -> Unit) {
}

fun lambdaReturnTest() {
    higherOrderFunction { 
        return // Compile error here
    }
}
```
위에 예시는 컴파일 에러가 발생한다. 

> <span style="color:red">return is not allow here</span>

람다 내부에선 return 문이 허용되지 않는다. 그러면 람다문에서 어떻게 빠져나올 수 있는가?

### Return with Labels
---
```java
data class Student(val name: String, val id: Int)

private fun findSpock(list: List<Student>) {
    list.forEach label@{ // or simply omit and use return@foreach instead
        if (it.name == "Spock") {
            println("Found Spock")
            return@label
        }
    }
    println("Did we find Spock?")
}

fun findStudentTest() {
    val studentList = listOf(Student("Kirk", 12345), Student("Spock", 54321))
    findSpock(studentList)
    println("End of findStudentTest")
}
```
위처럼 라벨을 선언하고 해당 라벨이름으로 리턴을 수행하면 된다. 
list.forEach label@ << 여기서 label 은 내가 마음대로 명시한것이고 원하면 이름을 바꿔도 된다.
만약 아무것도 라벨을 하지 않으면 여기서는 기본 return@forEach 로 빠져나갈수 있다.

**출력결과** 
> Found Spock  
Did we find Spock?  
End of findStudentTest

마치 break 문과 비슷하다. 이걸 `local return` 이라 말한다.

### Anonymous Function Returns
---
이번에는 같은 예시로 익명함수로 대체해보자

```java
private fun findSpock(list: List<Student>) {
    list.forEach(fun(student) {
        if (student.name == "Spock") {
            println("Found Spock")
            return
        }
    })
    println("Did we find Spock?")
}

fun findStudentTest() {
    val studentList = listOf(Student("Kirk", 12345), Student("Spock", 54321))
    findSpock(studentList)
    println("End of findStudentTest")
}
```

**출력결과**
> Found Spock  
Did we find Spock?  
End of findStudentTest

익명함수 내부에서 반환이 익명 함수에서 반환됬음을 알수있다. 
label 로 했던것과 비슷한 결과가 나왔다.

### Inline functions
---
알다시피 inline 함수는 호출구역으로 함수의 바이트 코드를 그대로 복사한다. 그럼 이건 return 문에 영향이 있는지 살펴보자

```java
private inline fun findSpock(list: List<Student>, func: (Student) -> Unit) {
    list.forEach { func(it) }
    println("Did we find Spock?")
}

fun findStudentTest() {
    val studentList = listOf(Student("Kirk", 12345), Student("Spock", 54321))
    findSpock(studentList) {
        if (it.name == "Spock") {
            println("Found Spock")
            return
        }
        println("Not Spock")
    }
    println("End of findStudentTest")
}
```

findSpock 함수는 원래 inline이 아닌경우 return문이 허용되지 않는다. 
[맨처음](/kotlin/function/lambda-control-flow/#lambda-returns) 에 나와있듯 람다함수에선 return 이 허용되지 않기 때문이다. 그럼 결과는 어떨까?

출력결과
> Not Spock  
Found Spock

결과에서 보이듯 둘러쌓인 함수로 부터 반환되었다. (마지막 문장을 출력안하고 빠져나감) 
마치 for문에서 return 문으로 빠져나간것과 같은 결과다. 이걸
`non-local return` 이라 부른다.

### 결론
---
우리는 여기서 패턴을 찾을 수 있다. return 은 가장 가까운 함수 선언으로 돌아간다는점을.
인라인 람다에서 둘러 쌓인 함수는 가장 가까운 함수 선언이 있는 곳이므로 둘러 싸는 함수 외부로 부터 반환되고
익명함수에서 가장 근접한 함수는 익명함수 선언문에 있어서 해당 부분에서 return 되었다.

### 참조

[https://medium.com/tompee/idiomatic-kotlin-lambdas-and-control-flows-70a7a58d7a20](https://medium.com/tompee/idiomatic-kotlin-lambdas-and-control-flows-70a7a58d7a20)