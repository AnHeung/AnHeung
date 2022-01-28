---
layout : single
title : Dart NullSafety 마이그레이션
---

### 마이그레이션 기다리기
---
예를 들어 패키지 C 가 패키지 B 를 참조하고 B 는 패키지 A 를 참조하는 상황일때 가장 먼저 A 를 
NullSafety 를 적용하고 그 후 B , C 순으로 진행해야한다.

{% include image.html id=''%}

의존성이 NullSafety 를 지원하기전에 먼저 의존성에 대한 코드를 마이그레이션을 진행해야 한다.
왜냐하면 만약 함수에 nullable 한 파라미터가 있다고 가정하면 추후 NullSafety 적용이 되면 해당 파라미터는 기본적으로 non-null 로 변경이 되게 된다. 그로 인해 nullable 파라미터를 전달할 경우 컴파일 에러가 발생하게 된다. 

> Dart 버전을 2.12 으로 올리지 않는 한 NullSafety 가 적용되진 않는다. 즉 Dart 나 Flutter 코어 라이브러리는 NullSafety 가 적용되 있지만 버전이 해당버전보다 낮을 경우 적용이 안되서 그냥 사용할 수 있다.

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

위의 명령어를 실행하면 현재 프로젝트에 사용하고 있는 의존성들이 NullSafety 가 적용된 버전을 사용중이라면 마이그레이션을 진행할수 있다고 알려준다. 만약 그렇지 못한 의존성들이 있다면 NullSafety 가 적용된 버전이 존재한다면 녹색으로 표시해주고 없다면 빨간색으로 표시해서 알려준다.


### 참조
---
[https://dart.dev/null-safety/migration-guide](https://dart.dev/null-safety/migration-guide)