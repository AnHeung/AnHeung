---
title: 로컬 단위 테스트 빌드하기
layout: single
---
로컬 테스트는 안드로이드 에뮬레이터나 실제 기기말고 JVM 을 사용하여 개발머신에서 동작한다. 
로컬 테스트는 당신의 앱의 로직을 빠르게 평가해준다. 하지만 안드로이드 프레임워크와 상호작용을 할 수 없기 
때문에 너가 할수 있는 테스트 유형에 한계가 있다. 

단위 테스트는 코드의 작은 부분의 동작을 확인한다. 그렇게 함으로 코드를 수행하고 결과를 확인한다.  
단위테스트는 보통 단순하지만 테스트가능하게 설계하지 않는다면 그것의 설정함에 있어 문제를 일으킬수 있다.

- 확인하고 싶은 코드는 테스트로 부터 접근이 필요하다. 예를 들어 private 메소드는 직접 테스트 할 순 없지만 public API 를 사용해 대신 테스트 할 수 있다.

- 단위테스트를 분리(고립) 해서 하기 위해서는 단위테스트 의존성들이 fake 객체나 다른 테스트 더블등 제어 가능한 컴포넌트로 교체 되어야한다. 코드가 안드로이드 프레임워크에 의존한다면 이건 더욱 문제를 
일으킬 수 있기 때문이다.

### 테스트 의존성들 주입
---
기본적으로 로컬 단위테스트 소스 파일은 `module-name/src/test/` 에 위치한다. 안드로이드 스튜디오에서 새 프로젝트 작성시 기본적으로 존재하는 폴더다.

이제 JUnit 테스트 프레임워크에 의해 공급되는 표준 API 를 프로젝트의 의존성에 주입하자.

```gradle
dependencies {
  // Required -- JUnit 4 framework
  testImplementation 'junit:junit:$jUnitVersion'
  // Optional -- Robolectric environment
  testImplementation 'androidx.test:core:$androidXTestVersion'
  // Optional -- Mockito framework
  testImplementation 'org.mockito:mockito-core:$mockitoVersion
}
```

### 로컬 단위 테스트 클래스 만들기
---
JUnit4 클래스를 통해 로컬 단위테스트를 작성할 수 있다.   
`module-name/src/test/` 폴더안에 하나 또는 그 이상의 테스트 메소드를 포함한 클래스를 만들고  
 `@Test` 주석을 테스트 메소드에 붙인다. 해당 주석이 붙은 코드가 동작하고 검증한다.  

아래 따라오는 예시는 어떻게 로컬 단위 테스트가 동작하는지 설명한다. 아래 `emailValidator_correctEmailSimple_returnsTrue()` 는 `isValidEmail()` 를 검증한다. 만약 `isValidEmail()` 가 항상 true 를 반환한다면 테스트 함수도 true 를  반환할 것이다.

```java
import org.junit.Assert.assertFalse
import org.junit.Assert.assertTrue
import org.junit.Test

class EmailValidatorTest {
  @Test fun emailValidator_CorrectEmailSimple_ReturnsTrue() {
    assertTrue(EmailValidator.isValidEmail("name@email.com"))
  }

}
```

이렇게 함수가 기대한 결과값을 반환하는지 평가하기 위해서 우리는 읽기쉬운 테스트를 작성해야한다. assertion 라이브러리로는 [junit.Assert](https://junit.org/junit4/javadoc/latest/org/junit/Assert.html) , [Hamcrest](https://github.com/hamcrest) , [Truth](https://truth.dev/) 등을 추천하고 위의 예제는 junit.Assert 를 사용했다.

### 모의 실행 가능한 안드로이드 라이브러리
---
단위 테스트를 실행할때 [Android Gradle Plug-in](https://developer.android.com/studio/releases/gradle-plugin) 는 당신의 프로젝트에서 사용되는 정확한 버전의 안드로이드 프레임워크의 모든 API를 포함한 라이브러리를 가지고 있다. 해당 라이브러리는 public 메소드들이나 클래스들을 가지고 있지만 실제 내부는 삭제되있다. 즉 실제 메소드에 접근하면 exception 을 던진다. 

이런 라이브러리는 로컬테스트에서 안드로이드 프레임워크 내부에 Context 같은 클래스에 참조하는것을 허용해준다.
뿐만 아니라 안드로이드 클래스와 함께 모의 클래스를 사용하는것을 허용해준다.   
(테스트 유형인 mock 은 그것의 상호작용에 대한 기대값을 가지고 너가 정의한대로 행동한다. [안드로이드에서 테스트 더블(test doubles) 사용하기.](/android/test/use-test-doubles-in-android/))

#### 안드로이드 mock 의존성

클래스가 string 리소스를 사용하는것을 알아내는것은 일반적인 문제다. `getString()` 이란 Context 내부 함수를 통해 string 리소스를 얻을 수 있지만 로컬 테스트에서는 안드로이드 프레임워크에 속하는 
함수는 (Context 같은) 쓸 수 없다. 이상적인 방법으로 getString 함수를 클래스 밖으로 빼내는 방법도 있지만
현실적이지 못하다. 해결책으로는 Context 에 대한 mock 이나 stub 을 만들어 항상 getString 함수가 호출되면 같은 값을 반환하게 하는것이다.

프레임워크를 사용해 로컬 단위테스트에 mock 객체를 추가하기 위해선 아래의 방법을 따른다.

1. `build.gradle` 파일에 Mockito 라이브러리 의존성을 넣어라. 자세한건 [테스트 환경 구축하기](https://developer.android.com/training/testing/instrumented-tests/androidx-test-libraries/test-setup#add-gradle) 참조

2. 단위테스트 클래스를 시작하기 앞서 @RunWith(MockitoJUnitRunner.class) 주석을 넣어라.   
이 주석은 Mockito test runner 에게 프레임워크가 올바르게 사용되고 있는지 확인하고 mock 객체의 
초기화를 단순화 하도록 지시한다. 

3. 안드로이드 의존성에 mock 객체를 만들기 위해 필드 선언전에 `@Mock` 주석 추가 

4. 의존성의 동작을 stub 으로 만들기 위해 `when()` 과 `thenReturn()` 함수를 사용해 조건이 충족하면 
조건이나 반환값을 지정할 수 있다. 

아래의 따라오는 예제는 mock `Context` 를 사용해 단위테스트를 만드지는 보여준다.

```java
import android.content.Context
import org.junit.Assert.assertEquals
import org.junit.Rule
import org.junit.Test
import org.mockito.Mock
import org.mockito.Mockito.`when`
import org.mockito.junit.MockitoJUnit

private const val FAKE_STRING = "HELLO WORLD"

@RunWith(MockitoJUnitRunner::class)
class MockedContextTest {

    @Mock
    private lateinit var mockContext : Context

    @Test
    fun readStringFromContext_LocalizedString(){
        // given : 테스트중인 객체로 mock Context 가 주입됨.
        `when`(mockContext.getString(R.String.name_label))
            .thenReturn(FAKE_STRING)

         val myObjectUnderTest = ClassUnderTest(mockContext)            

        // when : 테스트 중인 객체를 통해 string 반환
         val result : String = myObjectUnderTest.getName() 

         // then : 결과는 기대한 값대로 나와야 한다. 
         assertEquals(result, FAKE_STRING)
    }

}
```

### 에러 : Method ... not mocked 
---
모의 실행 가능한 안드 라이브러리는 자체 메소드에 접근시 Error: "Method ... not mocked 메시지를 띄우며 
exception 을 던진다. 

만약 테스트에서 해당 exception 이 문제를 일으킬 수 있다면 해당 반환값을 null 이나 0 으로 바꿀 수 있다. 
아래 처럼 `build.gradle` 의 상단에서 설정값만 넣어주면 된다.
```gradle

android {
  ...
  testOptions {
    unitTests.returnDefaultValues = true
  }
}
```


### 참조
--- 
[build local unit tests](https://developer.android.com/training/testing/local-tests)