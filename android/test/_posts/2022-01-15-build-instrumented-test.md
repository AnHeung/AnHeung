---
layout :  single 
title : 계측 테스트 빌드하기
---

계측 테스트는 물리 안드로이드 기기 또는 에뮬레이터에서 동작한다. 그래서 안드로이드 프레임워크 API 를 활용함에 있어 장점이 있다. 그런 이유로 계측 테스트는 로컬 테스트 보단 더욱 느리지만 충실도가 더욱 높다.

계측 테스트는 실제 기기의 동작에 대한 테스트의 경우에만 사용하길 권한다. [AndroidX Test](https://developer.android.com/training/testing/instrumented-tests/androidx-test-libraries/test-setup) 는  `JUnit4 테스트` 를 통해 액티비티들을 시작시키고 상호작용하는  `JUnit4 rules` 를 제공한다. 또한 `Espresso` 나 `UI Animator` `Roblectric` 시뮬레이터 도 포함되있다.

### 테스트 환경 구축
---
안드로이드 스튜디오 프로젝트에서 계측 테스트에 대한 소스는 `module-name/src/androidTest/java/` 에 저장 되있다. 이 디렉토리는 새로 프로젝트를 만들면 포함되어있고 예제 계측 테스트 코드까지 작성되있다.

시작하기 앞서 앱에서 빠르게 빌드하고 계측 테스트 코드를 동작 시키기 위해서 `AndroidX Test API` 를 추가해야한다. Androidx Test 는 `JUnit4 test runner`,`AndroidJUnitRunner` 와 기능적 UI 테스트를 위한 `Espresso`,`UIAnimator`, `Compose test` 가 포함 되있다.

또한 프로젝트에서 테스트 러너와 Rules APIs 를 AndroidX Test 로 부터 공급받아 사용하기위해 안드로이드 테스트 의존성을 구성하는게 필요하다. `build.gradle` 파일의 최상단에 지정한 라이브러리를 의존성으로 지정하자.

```gradle
dependencies {
    androidTestImplementation "androidx.test:runner:$androidXTestVersion"
    androidTestImplementation "androidx.test:rules:$androidXTestVersion"
    // Optional -- UI testing with Espresso
    androidTestImplementation "androidx.test.espresso:espresso-core:$espressoVersion"
    // Optional -- UI testing with UI Automator
    androidTestImplementation "androidx.test.uiautomator:uiautomator:$uiAutomatorVersion"
    // Optional -- UI testing with Compose
    androidTestImplementation "androidx.compose.ui:ui-test-junit4:$compose_version"
}
```

JUnit4 테스트 클래스를 사용하고 Test Filtering 같은 기능에 접근하기 위해서는 `AndroidJUnitRunner` 를 
기본 계측 테스트 러너 로 모듈 레벨의 `build.gradle` 에서 확실하게 지정해야 한다.

```gradle
android {
    defaultConfig {
        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }
}
```

### 계측 테스트 클래스 생성
---
계측 테스트 클래스는 JUnit4 테스트 클래스여야 한다. 이 부분은 [로컬 테스트 빌드하기](/android/test/build-local-unit-tests/) 에서 설명한 것과 유사하다. 계측된 JUnit4 테스트 클래스를 만들기 위해선 
`AndroidJUnit4` 를 기본 테스트 러너로 지정해야한다.

> 만약 테스트가 JUnit4 와 JUnit3 라이브러리를 합쳐서 의존한다면 @RunWith(AndroidJUnit4::class) 주석을 테스트 시작시 정의하면 된다.

아래의 예제는 `LogHistory` 클래스를 위한 `Parcelable` 인터페이스가 바르게 구현되는지를 확인하는 계측 테스트 작성 방법이다.

```java
import android.os.Parcel
import android.text.TextUtils.writeToParcel
import androidx.test.filters.SmallTest
import androidx.test.runner.AndroidJUnit4
import com.google.common.truth.Truth.assertThat
import org.junit.Before
import org.junit.Test
import org.junit.runner.RunWith

const val TEST_STRING = "This is a string"
const val TEST_LONG = 12345678L

// 테스트가 JUnit4 와 JUnit3 라이브러리를 합쳐서 의존한다면 @RunWith(AndroidJUnit4::class) 주석을 테스트 시작시 정의하면 된다.
@RunWith(AndroidJUnit4::class)
@SmallTest
class LogHistoryAndroidUnitTest {
    private lateinit var logHistory: LogHistory

    @Before
    fun createLogHistory() {
        logHistory = LogHistory()
    }

    @Test
    fun logHistory_ParcelableWriteRead() {
        val parcel = Parcel.obtain()
        logHistory.apply {
            // 전송하고 받을 Parcelable 객체 설정
            addEntry(TEST_STRING, TEST_LONG)

            // 데이터 쓰기
            writeToParcel(parcel, describeContents())
        }

        // 쓰기 후에는 읽기 위해 parcel 을 리셋 해야한다.
        parcel.setDataPosition(0)

        // 데이터 읽기
        val createdFromParcel: LogHistory = LogHistory.CREATOR.createFromParcel(parcel)
        createdFromParcel.getData().also { createdFromParcelData: List<Pair<String, Long>> ->

            // 받은 데이터가 맞는지 검증
            assertThat(createdFromParcelData.size).isEqualTo(1)
            assertThat(createdFromParcelData[0].first).isEqualTo(TEST_STRING)
            assertThat(createdFromParcelData[0].second).isEqualTo(TEST_LONG)
        }
    }
}
```

### 참조
---
[Build Instrumented Test](https://developer.android.com/training/testing/instrumented-tests)