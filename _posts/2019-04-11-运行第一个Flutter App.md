---
layout:     post
title:      "运行第一个Flutter App"
subtitle:   "聊聊官方Flutter Demo"
date:       2019-04-11 17:16:00
author:     "Yuzo"
header-img: "img/post-bg-flutter.jpeg"
catalog: true
tags:
    - Flutter
    - Dart
    - Android
    - iOS
---

> 文本是在 AndroidStudio 开发工具中开发 Flutter 的。


## 运行第一个Flutter App

1：启动AndroidStudio，选择Start a new Flutter project。

![enter image description here](https://github.com/Yuzopro/image/blob/master/flutter/first_flutter_app_1.png?raw=true)

2：选择Flutter Application。

![enter image description here](https://github.com/Yuzopro/image/blob/master/flutter/first_flutter_app_2.png?raw=true)

3：配置信息。

![enter image description here](https://github.com/Yuzopro/image/blob/master/flutter/first_flutter_app_3.png?raw=true)

4：设置包名。

![enter image description here](https://github.com/Yuzopro/image/blob/master/flutter/first_flutter_app_4.png?raw=true)

5：运行flutter_hello_world App。

![enter image description here](https://github.com/Yuzopro/image/blob/master/flutter/first_flutter_app_5.png?raw=true)

## 分析lib/main.dart

1： Flutter程序入口。

Flutter程序入口是一个`main()`函数：

```dart
void main() => runApp(MyApp());
```

在`main()`函数中调用`runApp()`函数，传入一个`MyApp()`widget的参数。

2：创建一个无状态的部件（Stateless widget）

有状态的部件和无状态部件的区别主要在于状态的改变：

- Stateless widgets 是不可变的, 这意味着它们的属性不能改变 - 所有的值都是最终的。

- Stateful widgets 持有的状态可能在widget生命周期中发生变化. 实现一个 stateful widget 至少需要两个类:

	1. 一个 StatefulWidget类。

	2. 一个 State类。 StatefulWidget类本身是不变的，但是 State类在widget生命周期中始终存在。


`MyApp()`是一个无状态的部件，所有的界面UI都是在`build()`函数中处理，如下面代码所示

```dart
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: MyHomePage(title: 'Flutter Demo Home Page'),
    );
  }
}
```

在`build()`函数中引用了`MaterialApp()`widget，主要实现了Material风格的相关部件，包括标题、主题、主界面。

2：创建一个有状态的部件（Stateful widget）

`MyHomePage()`是一个有状态的部件，除了创建State类之外几乎没有其他任何东西，如下面代码所示

```dart
class MyHomePage extends StatefulWidget {
  MyHomePage({Key key, this.title}) : super(key: key);

  final String title;

  @override
  _MyHomePageState createState() => _MyHomePageState();
}
```

3：创建`_MyHomePageState()`类

该类持有`MyHomePage()` widget的状态，并且该应用程序的大部分代码都在该类中。界面主要展示标题栏、居中展示两行文本以及悬浮按钮，如下面代码所示

```dart
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
            style: Theme.of(context).textTheme.display1,
          ),
        ],
      ),
    ),
    floatingActionButton: FloatingActionButton(
      onPressed: _incrementCounter,
      tooltip: 'Increment',
      child: Icon(Icons.add),
    ),
  );
}
```

4：添加交互

与有状态的部件进行交互，主要是通过`setState()`函数，当调用该函数时，会触发`build()`函数刷新，如下面代码所示

```dart
setState(() {
});
```

用户可以通过点击悬浮按钮，来刷新点击的次数，如下面代码所示

```dart
void _incrementCounter() {
  setState(() {
    _counter++;
  });
}
```

这部分内容就不深究了，简单了解一下就行