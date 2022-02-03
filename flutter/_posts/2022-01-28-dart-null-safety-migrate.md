---
layout : single
title : Dart null safety 마이그레이션
---

### 마이그레이션 기다리기
---
예를 들어 패키지 C 가 패키지 B 를 참조하고 B 는 패키지 A 를 참조하는 상황일때 가장 먼저 A 를 
null safety 를 적용하고 그 후 B , C 순으로 진행해야한다.

{% include image.html id='16WrqTXAOIq3-kt3E07q5cqyjyvg8Je8Z'%}

의존성이 null safety 를 지원하기전에 먼저 의존성에 대한 코드를 마이그레이션을 진행해야 한다.
왜냐하면 만약 함수에 nullable 한 파라미터가 있다고 가정하면 추후 null safety 적용이 되면 해당 파라미터는 기본적으로 non-null 로 변경이 되게 된다. 그로 인해 nullable 파라미터를 전달할 경우 컴파일 에러가 발생하게 된다. 

> Dart 버전을 2.12 으로 올리지 않는 한 null safety 가 적용되진 않는다. 즉 Dart 나 Flutter 코어 라이브러리는 null safety 가 적용되 있지만 버전이 해당버전보다 낮을 경우 적용이 안되서 그냥 사용할 수 있다.

### Latest Stable Dart Release 로 바꾸기
---
Dart SDK 나 Flutter SDK 를 Latest Stable Release 로 변환 하기 위해선 일단 해당 버전을 체크해 봐야 한다.
```
dart --version
```
#### 의존성 상태 확인
```
 dart pub outdated --mode=null-safety
```

위의 명령어를 실행하면 현재 프로젝트에 사용하고 있는 의존성들이 null safety 가 적용된 버전을 사용중이라면 마이그레이션을 진행할수 있다고 알려준다. 만약 null safety 가 적용된 버전이 존재한다면 녹색으로 표시해주고 없다면 빨간색으로 표시해서 알려준다.

{% include image.html id='16XGA6eRV9UDxoLqK4AU5nrMND3dFW5Le'%}

#### 의존성 업데이트

해당 부분을 다 체크했으면 아래의 명령어를 시작해서 업데이트를 진행하면 된다.
```
dart pub upgrade --null-safety
//업데이트 후 한번더 get
dart pub get.
```

### 마이그레이션
---
코드에서 null safety 가 필요한 대부분의 변경 사항은 쉽게 예측할 수 있다. 예를 들면 변수는 null 이 될수 있고 뒤에 `?` 가 붙어 null 이 될 수 있음을 표시해줄 것이다. [its type needs a ? suffix](https://dart.dev/null-safety#creating-variables)  

마이그레이션을 하기 위해서는 2가지 선택지가 있다.
1. 마이그레이션 도구 사용

2. [직접 마이그레이션 하기](https://dart.dev/null-safety/migration-guide#migrating-by-hand)

여기선 마이그레이션 도구를 사용해 보겠다.

시작하기전 준비사항이 있다.

- 최신 버전 Dart SDK 사용

- `dart pub outdated --mode=null-safety` 을 사용해 모든 의존성이 null-safety 가 되었는지 체크하고 업데이트 진행

준비가 끝나면 간단하게 명령어 한줄로 마이그레이션이 진행된다.
```
dart migrate
```

이제 마이그레이션 준비는 끝났고 아래의 주소로 들어가면  마이그레이션을 어떻게 진행할지 시각적으로 보여주는 창이 뜬다.

```
View the migration suggestions by visiting:

http://127.0.0.1:60278/Users/you/project/mypkg.console-simple?authToken=Xfz0jvpyeMI%3D
```

{% include image.html id='16c4eTlJCED3Gmp3v1KjNnklZ7iiupkyA'%}

각각 dart 파일을 보면서 어떻게 바뀔지를 보고 수정을 진행할 수 있다.  
Proposed Edits 라는 창에 각각의 변화에 대한 이유가 적혀있다. 

```dart
var ints = const <int>[0, null];
var zero = ints[0];
var one = zero + 1;
var zeroOne = <int>[zero, one];
```

만약이게 기본 코드라면 제안된 마이그레이션 코드는 아래처럼 나온다

```dart
var ints = const <int?>[0, null];
var zero = ints[0];
var one = zero! + 1;
var zeroOne = <int?>[zero, one];
```
ints 가 nullable 한 객체로 바뀌었기 때문에 zero 값도 nullable 이 되었고 one 을 null 에 대한 보장을 해주지 않으면 compile 에러가 발생하기 때문에 뒤에 ! 를 붙임으로 non-null 을 보장해 주었다.  
이 방법 외에도 hint marker 를 사용해서 마이그레이션 제안을 수정 할 수도 있다.

```dart
var ints = const <int?>[0, null];
var zero = ints[0]/*!*/;  // null 이 아님을 보장 해서 밑에 ! 표시가 다 사라졌다.
var one = zero + 1;
var zeroOne = <int>[zero, one];
```

아래는 hint marker 의 효과다.

{% include image.html id='16gTyvOjghqXAt1TZlRaDsI-B_ymrPvTr'%}

#### 파일 선택 및 마이그레이션 

이렇게 마이그레이션을 전체적으로 다 체크 하고 이상이 없다고 생각되면 마이그레이션 툴의 APPLY MIGRATION 버튼을 눌러서 마이그레이션을 진행하면 된다. 단 hint marker 로 작업을 해두었으면 hint marker 는 다 삭제되고 마이그레이션된 코드로 저장하게 된다. 

### 분석 및 테스트
---

해당 작업이 다 완료 되고 분석 명령어와 test 명령어를 수행해서 최종적으로 마이그레이션이 잘 되었는지 확인해본다. 

분석
```
dart pub get
dart analyze  # or `flutter analyze`
```

테스트
```
dart test  # or `flutter test`
```

### 참조
---
[https://dart.dev/null-safety/migration-guide](https://dart.dev/null-safety/migration-guide)