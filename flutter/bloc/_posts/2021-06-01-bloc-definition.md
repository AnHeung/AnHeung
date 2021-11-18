---
layout : single
title : Bloc 패턴의 이해
---

### Bloc (Business Logic Component)

Bloc 는 2018년 구글 개발자에 의해 디자인되었다.
Flutter 를 예로 들면  Flutter 는 상태에 따라 렌더링 발생하기 떄문에 상태관리가 매우 중요한데 UI 와  비지니스 로직 분리를 통한 코드 의존성 낮추기에 집중하고 각 UI 객체 (컴포넌트) 등은 
Bloc 를 구독하고 있다.

{% include image.html id="1-9wsI1BSop-RhohjIc0FNLmuyLLvInCe"%}

상태 구독중인 UI 객체들은 이벤트가 발생시 해당 이벤트를 UI에 알려주고
UI 는 그값을 받아 상태가 변한다.
Bloc 객체는 보통 Provider나 Repository 등으로 부터 데이터를 받고 그걸 토대로  
비지니스 로직을 수행한다.


### Bloc 특징
---

- UI 는 여러 Bloc 가 존재가능하다. 
- UI는 화면만 집중하고 로직은 Bloc에서 처리하도록 한다.
- UI 는 Bloc 내부 구현을 모르고 Bloc 는 여러 UI에 구독 가능하기 떄문에 재활용성이 좋다.  

`(A 컴포넌트가 C Bloc 를 구독하고 B 컴포넌트도 C Bloc를 구독해서 이벤트를 듣고 있으면 
C Bloc는 어느 컴포넌트가 구독하던간에 상관없이 이벤트를 방출시키고 C Bloc 가 상태가 변했음을 전달하면
A 와 B 컴포넌트 둘다 상태가 바뀌었음을 알고 값을 바꾼다.)`
이런식으로 구조를 짜게 되면 UI는 UI 대로 비지니스 로직은 비지니스 로직대로 분리가 되기 때문에 테스트 코드를 짤때 Bloc 부분만 테스트 할수 있다. 즉 의존성이 없어 좋다.


### Stream 을 활용한 Bloc
--- 
기본적으로 Flutter 에서 새 프로젝트를 만들었을때 기본적으로 만들어주는 컴포넌트가 있다.

```dart
class MyHomePage extends StatefulWidget {
  MyHomePage({Key key, this.title}) : super(key: key);

  final String title;

  @override
  _MyHomePageState createState() => _MyHomePageState();
}

class _MyHomePageState extends State<MyHomePage> {
  int _counter = 0;

  void _incrementCounter() {
    setState(() {
      _counter++;
    });
  }

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
            Text(
              '$_counter',
              style: Theme.of(context).textTheme.headline4,
            ),
          ],
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: _incrementCounter,
        tooltip: 'Increment',
        child: Icon(Icons.add),
      ), // This trailing comma makes auto-formatting nicer for build methods.
    );
  }
}
```

기본적으로 숫자버튼을 눌렀을때 값을 하나씩 올려주는 로직이 쓰여있다.
`StatefulWidget` 위젯인 MyHomePage 에서 `_counter` 라는 값을 `setState` 함수를 통해 올리고
그 값을 물고 있는 `Text` 컴포넌트에서 새로 고침이 일어나면서 값이 변하는 구조다.

이 자체로도 나쁘진 않지만 UI에 비지니스로직이 포함되있는 셈이다. 
이렇게 되면 의존성이 높아질수 밖에없다. 그래서 해당부분을 Bloc 구조로 바꿔보자.

Flttur에선 [Stream](https://dart.dev/tutorials/language/streams)을 활용해 Bloc를 구현한다.

[StreamController](https://api.dart.dev/stable/2.14.4/dart-async/StreamController-class.html) 로 Observable 객체를 생성하고 Sink 를 통해 값을 전달한다
StreamController 는 Stream 을 통해 상태를 전달할수도 구독할수도 있는 객체이다.

```dart
class CounterBloc {
  int _currentCount = 0;

  final StreamController<int> counterEventController = StreamController();

  Stream<int> get counter => counterEventController.stream;

  void addNumber() {
    counterEventController.add(_currentCount++);
  }

  void dispose() {
    counterEventController.close();
  }
}
```

일단 counter 값을 처리해줄 CounterBloc 클래스를 만들고 생성됫을때 Controller를 생성해준다.
Controller는 .add 메소드를 통해(.sink.add 와 동일) Stream 을 가지고 있는 (구독) 쪽으로 값을 전달할 수도 있고 반대로 listen 메소드를 통해 이벤트가 발생하는것을 구독할수도 있다.
단 기본적으로 스트림은  한곳에서만 listen 할수 있고 stream 이 두번 발생하면 `Bad state: Stream has already been listened to.` Exception 이 발생하니 주의하자.

```dart
void initial(){
final StreamController ctrl = StreamController());
final Stream<int> stream = Stream.periodic(Duration(milliseconds: 1000) , (x)=>x).take(5);
stream.listen((event)=>print(event));
stream.listen((event)=>print(event));
}
```

이제 Bloc 를 구독할 컴포넌트에 인스턴스를 만들어서 구독을 해보자 기본적으로 `StreamBuilder`를 통해 
구독할 Stream을 정하고 Stream에서 이벤트가 발생시 알림을 줘서 뷰에 이벤트를 던져줄수 있다.


```dart
class MyHomePage extends StatefulWidget {
  MyHomePage({Key key, this.title}) : super(key: key);

  final CounterBloc bloc = CounterBloc();
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
            StreamBuilder<int>(
              builder: (context, snapshot) => Text(
                "${snapshot.data}",
                style: Theme.of(context).textTheme.headline4,
              ),
              stream: widget.bloc.counter,
              initialData: 0,
            )
          ],
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: widget.bloc.addNumber,
        tooltip: 'Increment',
        child: Icon(Icons.add),
      ), // This trailing comma makes auto-formatting nicer for build methods.
    );
  }

  @override
  void dispose() {
    super.dispose();
    widget.bloc?.dispose();
  }
}
```

컴포넌트를 StreamBuilder로 둘러싸고 거기에 Bloc 객체의 stream을 구독한다. 이제 snatshot 이란 객체의 data라는 값을 통해 Bloc에서 정의한 int 타입의 counter 값을 받을 준비가 다 되었다.
floatingActionButton 버튼에 Bloc의 addNumber라는 메소드를 통해 Controller의 `add` 메소드를 통해 이벤트가 발생하면 이제 StreamBuilder 안에 포함되있는 여기선 MyHomePage 안의 Text 컴포넌트로 값이 넘어온다.

RxDart로도 비슷한 구현이 가능한데 [PublishSubject](https://pub.dev/documentation/rxdart/latest/rx/PublishSubject-class.html) 라는 구독 및 관찰자를 통해 쉽게 구현할수 있다. PublichSubject는 Controller와 유사한 기능들을 가지고 있다.

Bloc 를 씀으로 인해 StatelessWidget 에서 setState 등을 사용해 UI를 다시 그릴 필요가 없다.
그리고 UI에 처리하는 로직자체가 들어있지 않기 때문에 비지니스 로직이 분리가 잘 되었다.

### 결론
---
Bloc는 Flutter에만 국한된게 아닌 React , Augular 등 상태를 관리하기 위해 디자인 되었다.
즉 Observer 패턴이 구현된곳 어디든 사용이 가능하다. Bloc 패턴으로 설계함으로써
책임의 분리와 테스트에 유연한 코드를 구성할수 있다.


### 참조
---
[https://pks2974.medium.com/bloc-%EC%9D%B4%ED%95%B4-%ED%95%98%EA%B8%B0-%EB%B0%8F-%EA%B0%84%EB%8B%A8-%EC%A0%95%EB%A6%AC-%ED%95%98%EA%B8%B0-7dc705e4c640](https://pks2974.medium.com/bloc-%EC%9D%B4%ED%95%B4-%ED%95%98%EA%B8%B0-%EB%B0%8F-%EA%B0%84%EB%8B%A8-%EC%A0%95%EB%A6%AC-%ED%95%98%EA%B8%B0-7dc705e4c640)