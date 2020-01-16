A Flutter package that helps implement the BLoC Design Pattern using the power of reactive streams.

This package is built to work with [rx_bloc](https://github.com/Prime-Holding/RxBloc).

## Bloc Widgets

**RxBlocBuilder** is a Flutter widget which requires a `RxBloc`, a `builder` and a `state`  function. `RxBlocBuilder` handles building the widget in response to new states. `RxBlocBuilder` is very similar to `StreamBuilder` but has a more simple API to reduce the amount of boilerplate code needed. 

* The `builder` function will potentially be called many times and should be a [pure function](https://en.wikipedia.org/wiki/Pure_function) that returns a widget in response to the state.
* The `state` function determines which exact state of the bloc will be used. 
* If the `bloc` parameter is omitted, `RxBlocBuilder` will automatically perform a lookup using `RxBlocProvider` and the current `BuildContext`. 

See `RxBlocListener` if you want to "do" anything in response to state changes such as navigation, showing a dialog, etc...


```dart
RxBlocBuilder<NewsBloc, List<News>>( // At the first placeholder define what bloc you need, at the second define what type will be the state you want to listen. It needs to match the type of the stream in the state function below.
  state: (bloc) => bloc.states.news, // Determine which exact state of the bloc will be used for building the widget below. 
  builder: (context, state, bloc) {
    // return widget here based on BlocA's state
  }
)
```

Only specify the bloc if you wish to provide a bloc that will be scoped to a single widget and isn't accessible via a parent `RxBlocProvider` and the current `BuildContext`.

```dart
RxBlocBuilder<NewsBloc, List<News>>(
  bloc: blocA, // provide the local bloc instance
  state: (bloc) => bloc.states.news, // Determine which exact state of the bloc will be used for building the widget below.
  builder: (context, state, bloc) {
    // return widget here based on BlocA's state
  }
)
```

**RxBlocProvider** is a Flutter widget which provides a bloc to its children via `RxBlocProvider.of<T>(context)`. It is used as a dependency injection (DI) widget so that a single instance of a bloc can be provided to multiple widgets within a subtree.

In most cases, `RxBlocProvider` should be used to create new `blocs` which will be made available to the rest of the subtree. In this case, since `RxBlocProvider` is responsible for creating the bloc, it will automatically handle closing the bloc.

```dart
RxBlocProvider(
  create: (BuildContext context) => BlocA(),
  child: ChildA(),
);
```

In some cases, `RxBlocProvider` can be used to provide an existing bloc to a new portion of the widget tree. This will be most commonly used when an existing `bloc` needs to be made available to a new route. In this case, `RxBlocProvider` will not automatically close the bloc since it did not create it.

```dart
RxBlocProvider.value(
  value: RxBlocProvider.of<BlocA>(context),
  child: ScreenA(),
);
```

then from either `ChildA`, or `ScreenA` we can retrieve `BlocA` with:

```dart
RxBlocProvider.of<BlocA>(context)
```

**RxMultiBlocProvider** is a Flutter widget that merges multiple `RxBlocProvider` widgets into one.
`RxMultiBlocProvider` improves the readability and eliminates the need to nest multiple `RxBlocProviders`.
By using `RxMultiBlocProvider` we can go from:

```dart
RxBlocProvider<BlocA>(
  create: (BuildContext context) => BlocA(),
  child: RxBlocProvider<BlocB>(
    create: (BuildContext context) => BlocB(),
    child: RxBlocProvider<BlocC>(
      create: (BuildContext context) => BlocC(),
      child: ChildA(),
    )
  )
)
```

to:

```dart
RxMultiBlocProvider(
  providers: [
    RxBlocProvider<BlocA>(
      create: (BuildContext context) => BlocA(),
    ),
    RxBlocProvider<BlocB>(
      create: (BuildContext context) => BlocB(),
    ),
    RxBlocProvider<BlocC>(
      create: (BuildContext context) => BlocC(),
    ),
  ],
  child: ChildA(),
)
```

**RxBlocListener** is a Flutter widget which takes a `RxBlocWidgetListener` and an optional `RxBloc` and invokes the `listener` in response to state changes in the bloc. It should be used for functionality that needs to occur once per state change such as navigation, showing a `SnackBar`, showing a `Dialog`, etc...

`listener` is only called once for each state change (**NOT** including `initialState`) unlike `builder` in `RxBlocBuilder` and is a `void` function.

If the bloc parameter is omitted, `RxBlocListener` will automatically perform a lookup using `RxBlocProvider` and the current `BuildContext`.

```dart
RxBlocListener<NewsBloc, bool>( // Specify the type of the bloc and its state type
    state: (bloc) => bloc.states.isLoading, // pick a specific state you want to listen for
    listener: (context, state) {
      // do stuff here based on NewsBloc's state
    }
)
```

Only specify the bloc if you wish to provide a bloc that is otherwise not accessible via `BlocProvider` and the current `BuildContext`.

```dart
RxBlocListener<NewsBloc, bool>( // Specify the type of the bloc and its state type
    bloc: bloc,
    state: (bloc) => bloc.states.isLoading, // pick a specific state you want to listen for
    listener: (context, state) {
      // do stuff here based on NewsBloc's state
    }
)
```

If you want fine-grained control over when the listener function is called you can provide an optional `condition` to `RxBlocListener`. The `condition` takes the previous bloc state and current bloc state and returns a boolean. If `condition` returns true, `listener` will be called with `state`. If `condition` returns false, `listener` will not be called with `state`.

```dart
RxBlocListener<BlocA, BlocAState>(
  state: (bloc) => bloc.states.isLoading, // pick a specific state you want to listen for
  condition: (previousState, state) {
    // return true/false to determine whether or not
    // to call listener with state
  },
  listener: (context, state) {
    // do stuff here based on BlocA's state
  }
  child: Container(),
)
```

## Usage

Lets take a look at how to use `RxBlocBuilder` to hook up a `CounterPage` widget to a `CounterBloc`.

### CounterBloc
```dart
/// A class containing all incoming events to the BloC
abstract class CounterBlocEvents {
  /// Increment the count
  void increment();

  /// Decrement the count
  void decrement();
}

/// A class containing all states (outputs) of the bloc.
abstract class CounterBlocStates {
  /// The count of the Counter
  ///
  /// It can be controlled by executing [CounterBlocEvents.increment] and
  /// [CounterBlocEvents.decrement]
  ///
  Stream<String> get count;

  /// The state of the increment action control
  Stream<bool> get incrementEnabled;

  /// The state of the decrement action control
  Stream<bool> get decrementEnabled;

  /// The info message caused by changing action controls' state
  Stream<String> get infoMessage;
}

@RxBloc()
class CounterBloc extends $CounterBloc {
  /// The internal storage of the count
  final _count = BehaviorSubject.seeded(0);

  /// Acts as a container for multiple subscriptions that can be canceled at once
  final _compositeSubscription = CompositeSubscription();

  CounterBloc() {
    MergeStream([
      $incrementEvent.map((_) => ++_count.value),
      $decrementEvent.map((_) => --_count.value)
    ]).bind(_count).disposedBy(_compositeSubscription);
  }

  /// Map the count digit to presentable data
  @override
  Stream<String> mapToCountState() => _count.map((count) => count.toString());

  /// Map the count digit to a decrement enabled state.
  @override
  Stream<bool> mapToDecrementEnabledState() => _count.map((count) => count > 0);

  /// Map the count digit to a increment enabled state.
  @override
  Stream<bool> mapToIncrementEnabledState() => _count.map((count) => count < 5);

  /// Map the increment and decrement enabled state to a informational message.
  @override
  Stream<String> mapToInfoMessageState() => MergeStream([
        incrementEnabled.mapToMaximumMessage(),
        decrementEnabled.mapToMinimumMessage(),
      ]).skip(1).throttleTime(Duration(seconds: 1));

  @override
  void dispose() {
    _compositeSubscription.dispose();
    super.dispose();
  }
}

extension _InfoMessage on Stream<bool> {
  /// Map disabled state to a informational message
  Stream<String> mapToMaximumMessage() => where((enabled) => !enabled)
      .map((_) => "You have reached the maximum increment count");

  /// Map disabled state to a informational message
  Stream<String> mapToMinimumMessage() => where((enabled) => !enabled)
      .map((_) => "You have reached the minimum decrement count");
}
```

### CounterWidget
```dart
class CounterWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) => Card(
        elevation: 2,
        child: Padding(
          padding: const EdgeInsets.all(16.0),
          child: Column(
            mainAxisAlignment: MainAxisAlignment.end,
            children: <Widget>[
               RxBlocListener<CounterBlocType, String>(
                 state: (bloc) => bloc.states.infoMessage,
                 listener: (context, state) =>
                   Scaffold.of(context).showSnackBar(SnackBar(content: Text(state)))
               ),
              Expanded(
                child: Center(
                  child: RxBlocBuilder<CounterBlocType, String>(
                    state: (bloc) => bloc.states.count,
                    builder: (context, snapshot, bloc) => Text(
                      snapshot.data ?? '',
                      style: TextStyle(fontSize: 60),
                    ),
                  ),
                ),
              ),
              Row(
                children: [
                  RxBlocBuilder<CounterBlocType, bool>(
                    state: (bloc) => bloc.states.incrementEnabled,
                    builder: (context, snapshot, bloc) => Expanded(
                      child: RaisedButton(
                        child: Text('Increment'),
                        onPressed: (snapshot.data ?? false)
                            ? bloc.events.increment
                            : null,
                      ),
                    ),
                  ),
                  const SizedBox(width: 16),
                  RxBlocBuilder<CounterBlocType, bool>(
                    state: (bloc) => bloc.states.decrementEnabled,
                    builder: (context, snapshot, bloc) => Expanded(
                      child: RaisedButton(
                        child: Text('Increment'),
                        onPressed: (snapshot.data ?? false)
                            ? bloc.events.decrement
                            : null,
                      ),
                    ),
                  ),
                ],
              ),
            ],
          ),
        ),
      );
}
```
