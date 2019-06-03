---
layout:     post
title:      "Flutter主题切换之flutter redux"
subtitle:   "使用flutter redux实现切换主题"
date:       2019-06-01 23:32:00
author:     "Yuzo"
header-img: "img/post-bg-flutter-theme.jpg"
catalog: true
tags:
    - 移动端
    - Android
    - Flutter
    - iOS
---

# Flutter主题切换之flutter redux
> 本文详细讲述怎样在flutter中集成和使用redux，关于redux的概念、原理和实现，读者可自行百度，本文不做累述。

## flutter redux
### flutter redux组成
redux主要由`Store`、`Action`、`Reducer`三部分组成

* `Store`用于存储和管理`State`
* `Action`用于用户触发的一种行为
* `Reducer`用于根据Action产生新的`State`

### flutter redux流程

1. `Widget`通过`StoreConnector`绑定`Store`中的`State`数据
2. `Widget`通过`Action`触发一种新的行为
3. `Reducer`根据收到的`Action`更新`State`
4. 更新`Store`中的`State`绑定的`Widget`

根据以上流程，我们实现项目中的主题切换功能。

## 项目集成

### flutter redux库
> [pub.dev地址](https://pub.dev/packages/flutter_redux)
> [github地址](https://github.com/brianegan/flutter_redux)

### 集成flutter redux
修改项目根目录下`pubspec.yaml`，并添加依赖
> flutter_redux: ^0.5.3

### 初始化Store
首先看下`Store`的构造函数，如下面代码所示
```dart
 Store(
    this.reducer, {
    State initialState,
    List<Middleware<State>> middleware = const [],
    bool syncStream: false,
    bool distinct: false,
  })
    : _changeController = new StreamController.broadcast(sync: syncStream) {
    _state = initialState;
    _dispatchers = _createDispatchers(
      middleware,
      _createReduceAndNotify(distinct),
    );
  }
```
根据上面的构造函数，我们首先需要创建`State`，并且还需要完成`State`初始化；然后需要创建`Reducer`；最后需要创建`Middleware`（暂不是本文需要讲解的内容）；
#### 创建State
创建一个`State`对象`AppState`,用于储存需要共享的主题数据，并且完成`AppState`初始化工作，如下面代码所示
```dart
class AppState {
  ThemeData themeData;

  AppState({this.themeData});

  factory AppState.initial() => AppState(themeData: AppTheme.theme);
}
```
`AppTheme`类中定义了一个默认主题`theme`，如下面代码所示
```dart
class AppTheme {
  static final ThemeData _themeData = new ThemeData.light();

  static get theme {
    return _themeData.copyWith(
      primaryColor: Colors.black,
    );
  }
}
```
到此，完成了State的相关操作。
#### 创建Reducer
创建一个`Reducer`方法`appReducer`,为`AppState`类里的每一个参数创建一个`Reducer`，如下面代码所示
```dart
AppState appReducer(AppState state, action) {
  return AppState(
    themeData: themeReducer(state.themeData, action),
  );
}
```
而`themeReducer`将ThemeData和所有跟切换主题的行为绑定在一起,如下面代码所示
```dart
final themeReducer = combineReducers<ThemeData>([
  TypedReducer<ThemeData, RefreshThemeDataAction>(_refresh),
]);

ThemeData _refresh(ThemeData themeData, action) {
  themeData = action.themeData;
  return themeData;
}
```
通过`flutter redux`的`combineReducers`与`TypedReducer`将`RefreshThemeDataAction`和`_refresh`绑定在一起，当用户每次发出`RefreshThemeDataAction`时，都会触发`_refresh`,用来更新`themeData`。
#### 创建Action
创建一个`Action`对象`RefreshThemeDataAction`,如下面代码所示
```dart
class RefreshThemeDataAction{
  final ThemeData themeData;

  RefreshThemeDataAction(this.themeData);
}
```
`RefreshThemeDataAction`的参数themeData是用来接收新切换的主题。
#### 代码集成

创建`Store`所有的准备工作都已准备，下面创建`Store`，如下面代码所示
```dart
 final store = new Store<AppState>(
    appReducer,
    initialState: AppState.initial(),
 );
```
然后用`StoreProvider`加载store,`MaterialApp`通过`StoreConnector`与`Store`保持连接。到此我们已经完成了`flutter redux`的初始化工作，如下面代码所示
```dart
void main() {
  final store = new Store<AppState>(
    appReducer,
    initialState: AppState.initial(),
  );

  runApp(OpenGitApp(store));
}

class OpenGitApp extends StatelessWidget {
  final Store<AppState> store;

  OpenGitApp(this.store);

  @override
  Widget build(BuildContext context) {
    return new StoreProvider<AppState>(
      store: store,
      child: StoreConnector<AppState, _ViewModel>(
        converter: _ViewModel.fromStore,
        builder: (context, vm) {
          return new MaterialApp(
            theme: vm.themeData,
            routes: AppRoutes.getRoutes(),
          );
        },
      ),
    );
  }
}
```
`StoreConnector`通过`converter`在`_ViewModel`中转化`store.state`的数据，最后通过`builder`返回实际需要更新主题的控件，这样就完成了数据和控件的绑定。`_ViewModel`的代码如下面所示
```dart
class _ViewModel {
  final ThemeData themeData;

  _ViewModel({this.themeData});

  static _ViewModel fromStore(Store<AppState> store) {
    return _ViewModel(
      themeData: store.state.themeData,
    );
  }
}
```
#### 用户行为
最后，只需要添加切换主题部分的代码即可，这部分代码是从官方`gallery` demo里的Style/Colors copy出来的，不做过多分析，如下面代码所示
```dart  
const double kColorItemHeight = 48.0;

class Palette {
  Palette({this.name, this.primary, this.accent, this.threshold = 900});

  final String name;
  final MaterialColor primary;
  final MaterialAccentColor accent;
  final int
      threshold; // titles for indices > threshold are white, otherwise black

  bool get isValid => name != null && primary != null && threshold != null;
}

final List<Palette> allPalettes = <Palette>[
  new Palette(
      name: 'RED',
      primary: Colors.red,
      accent: Colors.redAccent,
      threshold: 300),
  new Palette(
      name: 'PINK',
      primary: Colors.pink,
      accent: Colors.pinkAccent,
      threshold: 200),
  new Palette(
      name: 'PURPLE',
      primary: Colors.purple,
      accent: Colors.purpleAccent,
      threshold: 200),
  new Palette(
      name: 'DEEP PURPLE',
      primary: Colors.deepPurple,
      accent: Colors.deepPurpleAccent,
      threshold: 200),
  new Palette(
      name: 'INDIGO',
      primary: Colors.indigo,
      accent: Colors.indigoAccent,
      threshold: 200),
  new Palette(
      name: 'BLUE',
      primary: Colors.blue,
      accent: Colors.blueAccent,
      threshold: 400),
  new Palette(
      name: 'LIGHT BLUE',
      primary: Colors.lightBlue,
      accent: Colors.lightBlueAccent,
      threshold: 500),
  new Palette(
      name: 'CYAN',
      primary: Colors.cyan,
      accent: Colors.cyanAccent,
      threshold: 600),
  new Palette(
      name: 'TEAL',
      primary: Colors.teal,
      accent: Colors.tealAccent,
      threshold: 400),
  new Palette(
      name: 'GREEN',
      primary: Colors.green,
      accent: Colors.greenAccent,
      threshold: 500),
  new Palette(
      name: 'LIGHT GREEN',
      primary: Colors.lightGreen,
      accent: Colors.lightGreenAccent,
      threshold: 600),
  new Palette(
      name: 'LIME',
      primary: Colors.lime,
      accent: Colors.limeAccent,
      threshold: 800),
  new Palette(
      name: 'YELLOW', primary: Colors.yellow, accent: Colors.yellowAccent),
  new Palette(name: 'AMBER', primary: Colors.amber, accent: Colors.amberAccent),
  new Palette(
      name: 'ORANGE',
      primary: Colors.orange,
      accent: Colors.orangeAccent,
      threshold: 700),
  new Palette(
      name: 'DEEP ORANGE',
      primary: Colors.deepOrange,
      accent: Colors.deepOrangeAccent,
      threshold: 400),
  new Palette(name: 'BROWN', primary: Colors.brown, threshold: 200),
  new Palette(name: 'GREY', primary: Colors.grey, threshold: 500),
  new Palette(name: 'BLUE GREY', primary: Colors.blueGrey, threshold: 500),
];

class ColorItem extends StatelessWidget {
  const ColorItem(
      {Key key,
      @required this.index,
      @required this.color,
      this.prefix = '',
      this.onChangeTheme})
      : assert(index != null),
        assert(color != null),
        assert(prefix != null),
        super(key: key);

  final int index;
  final Color color;
  final String prefix;
  final Function(Color) onChangeTheme;

  String colorString() =>
      "#${color.value.toRadixString(16).padLeft(8, '0').toUpperCase()}";

  @override
  Widget build(BuildContext context) {
    return new Semantics(
      container: true,
      child: new Container(
        height: kColorItemHeight,
        padding: const EdgeInsets.symmetric(horizontal: 16.0),
        color: color,
        child: new SafeArea(
          top: false,
          bottom: false,
          child: FlatButton(
            child: new Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              crossAxisAlignment: CrossAxisAlignment.center,
              children: <Widget>[
                new Text('$prefix$index'),
                new Text(colorString()),
              ],
            ),
            onPressed: () {
              onChangeTheme(color);
            },
          ),
        ),
      ),
    );
  }
}

class PaletteTabView extends StatelessWidget {
  static const List<int> primaryKeys = const <int>[
    50,
    100,
    200,
    300,
    400,
    500,
    600,
    700,
    800,
    900
  ];
  static const List<int> accentKeys = const <int>[100, 200, 400, 700];

  PaletteTabView({Key key, @required this.colors, this.onChangeTheme})
      : assert(colors != null && colors.isValid),
        super(key: key);

  final Palette colors;
  final Function(Color) onChangeTheme;

  @override
  Widget build(BuildContext context) {
    final TextTheme textTheme = Theme.of(context).textTheme;
    final TextStyle whiteTextStyle =
        textTheme.body1.copyWith(color: Colors.white);
    final TextStyle blackTextStyle =
        textTheme.body1.copyWith(color: Colors.black);
    final List<Widget> colorItems = primaryKeys.map((int index) {
      return new DefaultTextStyle(
        style: index > colors.threshold ? whiteTextStyle : blackTextStyle,
        child: new ColorItem(
            index: index,
            color: colors.primary[index],
            onChangeTheme: onChangeTheme),
      );
    }).toList();

    if (colors.accent != null) {
      colorItems.addAll(accentKeys.map((int index) {
        return new DefaultTextStyle(
          style: index > colors.threshold ? whiteTextStyle : blackTextStyle,
          child: new ColorItem(
              index: index,
              color: colors.accent[index],
              prefix: 'A',
              onChangeTheme: onChangeTheme),
        );
      }).toList());
    }

    return new ListView(
      itemExtent: kColorItemHeight,
      children: colorItems,
    );
  }
}

class ThemeSelectPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return StoreConnector<AppState, _ViewModel>(
        converter: _ViewModel.fromStore,
        builder: (context, vm) {
          return new DefaultTabController(
            length: allPalettes.length,
            child: new Scaffold(
              appBar: new AppBar(
                elevation: 0.0,
                title: const Text("主题色"),
                bottom: new TabBar(
                  isScrollable: true,
                  tabs: allPalettes
                      .map((Palette swatch) => new Tab(text: swatch.name))
                      .toList(),
                ),
              ),
              body: new TabBarView(
                children: allPalettes.map((Palette colors) {
                  return new PaletteTabView(
                    colors: colors,
                    onChangeTheme: vm.onChangeTheme,
                  );
                }).toList(),
              ),
            ),
          );
        });
  }
}

class _ViewModel {
  final Function(Color) onChangeTheme;

  _ViewModel({this.onChangeTheme});

  static _ViewModel fromStore(Store<AppState> store) {
    return _ViewModel(
      onChangeTheme: (color) {
        SharedPrfUtils.saveInt(SharedPrfKey.SP_KEY_THEME_COLOR, color.value);
        store.dispatch(RefreshThemeDataAction(AppTheme.changeTheme(color)));
      },
    );
  }
}
```
#### 运行效果
执行代码，效果如下
![enter image description here](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/flutter_redux_theme.gif)

参考文章
[https://www.jianshu.com/p/34a6224e0cf1](https://www.jianshu.com/p/34a6224e0cf1)