---
layout : single
title : Flutter_Bloc 를 사용한 Bloc 이해와 사용
---

개요
---

Bloc 에 대한 참조는 [Bloc 패턴의 이해](/flutter/bloc/bloc-definition/)

Flutter에서 Bloc 패턴을 편한하게 쓰기 위한 [flutter_bloc](https://pub.dev/packages/flutter_bloc)  라이브러리가 있다. 보통 Bloc 패턴을 구현하기 위해 Cubit 이라던가 StreamController 등을 통해 구현하지만 flutter_bloc는 그런 부분을 쉽게 구현할수 있게 도와준다.

기본적으로 `Event` , `State` , `Bloc` 3가지로 구성되며 각 클래스에서 역할이 나뉜다.
- Event : 실제 수행해야할 Bloc에 호출할 함수에 대한 명시 
- State: Bloc에 비지니스 로직이 처리된 후에 결과에 대한 상태에 대해 명시 
- Bloc : Event 와 State 를 가지고 실제 비지니스 로직을 처리하고 그걸 구독하고 있는 컴포넌트에 상태를 전달한다.

위에 언급한 StreamController 와 비교하면 조금 더 클래스도 많아지고 귀찮아 보이긴 하지만 실제 익숙해지면 
굉장히 편해짐을 알수있다.

[Stream 을 활용한 Bloc](/flutter/bloc/bloc-definition/#stream-을-활용한-bloc) 과 똑같이 Counter 를 만든다 가정하면 

Event 

```dart
@immutable
abstract class CounterEvent extends Equatable{

  @override
  List<Object> get props =>[];

  const CounterEvent();
}

class AddNumber extends CounterEvent {

  const AddNumber();

}

```
Event 는 Bloc 의 .add 라는 메소드를 통해 Bloc에 해당 동작을 호출하게 된다. 실제 처리하는 클래스는 아니지만 어떤 동작을 수행할지에 대한 명시하는 클래스라 생각하면 된다.
클래스 내부에 변수를 추가해서 마치 함수에 파라미터를 던지듯 구성할수도 있다.



State 
```dart
enum CounterStatus {Initial , Success , Failure}

class CounterState extends Equatable{

  final CounterStatus status;
  final int currentCount;
  final String msg;

  const CounterState({this.status, this.currentCount = 0, this.msg = ""});

  @override
  List<Object> get props =>[status, currentCount , msg];
}
```
State 는 Bloc 클래스가 실제 처리하고 나온 결과값에 대한 상태에대한 명시를 하는 클래스다.
Bloc 클래스에서 로직을 다 처리하고 처리한 값등을 보내면 실제 컴포넌트에서 Builder 의 state를 통해
값을 전달받게 된다.
위에 상속받은 `Equatable` 는 두 인스턴스를 비교해준다 기본적으로 `==` 와 hashCode를 내부적으로 
override 해줌으로 쓸데없는 코드량을 줄일 수 있다. 그리고 Bloc 에서 해당 부분을 구현함으로 
OldState 와 NewState Date를 비교해서 바뀐점이 있다면 State 를 컴포넌트에 전달하고 아닐경우 전달하지 않아
불필요한 호출을 막는다. 

Bloc 
```dart
class CounterBloc extends Bloc<CounterEvent, CounterState> {
  CounterBloc() : super(CounterState(status: CounterStatus.Initial));

 @override
  Stream<CounterState> mapEventToState(
    CounterEvent event,
  ) async* {
    if(event is AddNumber){
      yield* _mapEventAddNumber();  
    }
  }

  Stream<CounterState> _mapEventAddNumber() async*{
    //실제 비지니스 로직 처리
    int currentCount = state.currentCount;
    yield CounterState(status: CounterStatus.Success, currentCount: ++currentCount , msg: "update");
  }
}
```
실제 비지니스 로직을 처리하고 전달하는 클래스다. 위에 명시한 Event 클래스가 add 메소드를 통해 호출이 이루어지면 `mapEventToState` 이 부분을 통해 Event 가 전달되게 되고 여기서 어떻게 처리할지에 대한걸 정의할수 있다. 비지니스 로직을 다 처리하고 나면 `yield` 를 통해 State를 전달하면 해당 Bloc 를 참조하고 있는 `BlocBuilder` 컴포넌트에 전달되게 된다.


### BlocBuilder
---
[BlocBuilder](https://pub.dev/documentation/flutter_bloc/latest/flutter_bloc/BlocBuilder-class.html) 는 `StreamBuilder`와 유사하지만 BoilerPlate 코드를 줄여주고 Bloc 성능 향상을 위해 API가 단순화 되어있다.
기본적으로 Bloc 와 State 를 타입 매개변수로 받고 state가 새로 업데이트 될때 새로 그려지는 위젯을 반환한다.

```dart
class MyHomePage extends StatefulWidget {
  MyHomePage({Key key, this.title}) : super(key: key);
  final String title;

  @override
  _MyHomePageState createState() => _MyHomePageState();
}

class _MyHomePageState extends State<MyHomePage> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(widget.title),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            Text(
              'You have pushed the button this many times:',
            ),
            BlocBuilder<CounterBloc, CounterState>(
                builder: (context, state) => Text("${state.currentCount}"))
          ],
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => BlocProvider.of<CounterBloc>(context).add(AddNumber()),
        tooltip: 'Increment',
        child: Icon(Icons.add),
      ), 
    );
  }
}
```


### BlocProvider
---
위에 처리하는 부분에서 이상한점이 하나 있었을 것이다. 보통 비지니스 로직을 처리하기 위한 Bloc 인스턴스를
어느 부분이든 생성해서 인스턴스를 통해 메소드를 처리해야하는데 여기선 `BlocProvider.of<CounterBloc>(context)` 로 호출했다. 이건 기본적으로 Provider에 대한 이해가 있어야된다. 

{% include image.html id="1-SA1ahIJIqbJDxlZW_X27P3ce2BGBvxh"%}

여러 위젯이 있고 값에 대해서 공유를 하고 싶을 경우 상태를 공유할 위젯의 부모위젯을 먼저 만들고
트리구조로 해당 상태를 자식위젯으로 전달하게 되면 부모 자식 모두 동일한 상태를 받을수 있게된다. 하지만
이것을 전달하기 위해 생성자에 인스턴스를 담아 전달하는것도 번잡하기 때문에 Provider를 통해 전역적으로 다른위젯들과 상태등을 공유한다.

{% include image.html id="1-ReqtUJ5hM7fV3Z8okqgww73Xht6uTo9"%}

공통 부모 위젯이 Provider를 제공하고 값을 사용하는 곳에서 of 메소드를 통해 `BlocProvider.of<T>(context)` 해당 Bloc의 인스턴스를 전달받게 된다. 즉 `Dependency Injection (종속성 주입)` 기술을 사용해 해당객체를 클래스마다 생성하는 대신 클래스에 종속객체를 주입하는것이다.

```dart
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    BlocProvider(
      create: (context) => CounterBloc(),
      child: MyHomePage(),
    );

    return MaterialApp(
      title: 'Flutter Demo',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: BlocProvider(
          create: (context) => CounterBloc(),
          child: MyHomePage(title: 'Flutter Demo Home Page')),
    );
  }
}
```

BlocProvider는 기본적으로 느린초기화로 Bloc 객체를 생성해준다. 즉 실체 해당 인스턴스를 호출하기 전까지는
인스턴스가 생성되지 않는다. 만약 강제로 즉시 생성하고 싶으면 옵션에 lazy 만 false 로 주면 된다.

```dart
BlocProvider(
  lazy: false,
  create: (BuildContext context) => CounterBloc(),
  child: MyHomePage(),
);
```

`BlocProvider.value` 는 이미 만들어진 Bloc 를 다른 Widget에서 사용하고 싶을때 쓴다. 그리고 이걸로 만든경우 BlocProvider는 먼저 만들어진 BlocProvider 에 의해 다루어진다.

```dart
BlocProvider.value(
  value: BlocProvider.of<CounterBloc>(context),
  child: MyHomePage(),
);
```

여기선 MyHomePage의 부모위젯인 App에 BlocProvider 를 통해 인스턴스를 생성했고
이제 자식위젯인 MyHomePage에서는 of 메소드로 인스턴스를 획득하고 사용할수 있게된다.

```dart
BlocProvider.of<CounterBloc>(context).add(AddNumber()) 
```

이제 이런식으로 컴포넌트 등에서 Event를 호출하면 Bloc 에서 Event 를 받아처리해주고
처리한 State 값을 다시 BlocBuilder 에 전달한다.

### MultiBlocProvider 
---
만약 하나뿐 아니라 여러 Bloc 를 참조하고 싶다면 MultiBlocProvider 를 사용하면 된다.

```dart
MultiBlocProvider(
  providers: [
    BlocProvider<BlocA>(
      create: (BuildContext context) => BlocA(),
    ),
    BlocProvider<BlocB>(
      create: (BuildContext context) => BlocB(),
    ),
    BlocProvider<BlocC>(
      create: (BuildContext context) => BlocC(),
    ),
  ],
  child: ChildA(),
)
```



### 참조
--- 
[https://pub.dev/packages/flutter_bloc](https://pub.dev/packages/flutter_bloc)