---
layout:     post
title:      "Flutter基础Widgets-Text"
subtitle:   "flutter基础Widgets之Text的基本用法"
date:       2019-05-10 15:04:00
author:     "Yuzo"
header-img: "img/post-bg-android.jpg"
catalog: true
tags:
    - 移动端
    - Android
    - Flutter
    - iOS
---

## Text
### 构造函数
Text构造函数的源码如下面代码所示
``` dart
const Text(this.data, {
    Key key,
    this.style,
    this.strutStyle,
    this.textAlign,
    this.textDirection,
    this.locale,
    this.softWrap,
    this.overflow,
    this.textScaleFactor,
    this.maxLines,
    this.semanticsLabel,
}) : assert(
    data != null,
    'A non-null String must be provided to a Text widget.',
    ),
    textSpan = null,
    super(key: key);
```
### 属性

 - data
 >需要显示的字符串

``` dart
Text(
    'Hello World',
)
```
运行结果如下图所示
![enter image description here](https://github.com/Yuzopro/image/blob/master/flutter/flutter_widgets_text_1.png?raw=true)

- style
 >文本样式，样式属性如下表所示

![enter image description here](https://github.com/Yuzopro/image/blob/master/flutter/flutter_widgets_text_11.png?raw=true)
```dart
Text(
    'Hello World',
    style: TextStyle(
        color: Colors.green,
        fontSize: 16.0,
        fontWeight: FontWeight.bold,
        fontStyle: FontStyle.italic,
        letterSpacing: 2.0,
        wordSpacing: 1.5,
        textBaseline: TextBaseline.alphabetic,
        decoration: TextDecoration.lineThrough,
        decorationStyle: TextDecorationStyle.dashed,
        decorationColor: Colors.blue
    ),
)
```
运行结果如下图所示
![enter image description here](https://github.com/Yuzopro/image/blob/master/flutter/flutter_widgets_text_2.png?raw=true)

 - textAlign
>文本对齐方式，参数如下面表格所示

![enter image description here](https://github.com/Yuzopro/image/blob/master/flutter/flutter_widgets_text_12.png?raw=true)
```dart
Column(
    children: <Widget>[
        Text(
            text,
            textAlign: TextAlign.left,
            style: TextStyle(
              color: Colors.green,
            ),
        ),
        Text(
            text,
            textAlign: TextAlign.center,
            style: TextStyle(
              color: Colors.black,
            ),
        ),
        Text(
            text,
            textAlign: TextAlign.right,
            style: TextStyle(
              color: Colors.yellow,
            ),
        ),
        Text(
            text,
            textAlign: TextAlign.start,
            style: TextStyle(
              color: Colors.red,
            ),
        ),
        Text(
            text,
            textAlign: TextAlign.end,
            style: TextStyle(
              color: Colors.orange,
            ),
        ),
        Text(
            text,
            textAlign: TextAlign.justify,
            style: TextStyle(
              color: Colors.blue,
            ),
        ),
    ],
)
```
运行结果如下图所示
![enter image description here](https://github.com/Yuzopro/image/blob/master/flutter/flutter_widgets_text_3.png?raw=true)
 
 - textDirection
>`TextDirection.ltr`，文本从左向右流动；
`TextDirection.rtl`，文本从右向左流动。
相对TextAlign中的start、end而言有用（当start使用了ltr相当于end使用了rtl，也相当于TextAlign使用了left）

```dart
Column(
    children: <Widget>[
        Text(
            text,
            textDirection: TextDirection.ltr,
            style: TextStyle(
                color: Colors.green,
            ),
        ),
        Text(
            text,
            textDirection: TextDirection.rtl,
            style: TextStyle(
                color: Colors.blue,
            ),
        ),
    ],
)
```
运行结果如下图所示
![enter image description here](https://github.com/Yuzopro/image/blob/master/flutter/flutter_widgets_text_4.png?raw=true)

 - locale
>此属性很少设置，用于选择区域特定字形的语言环境

 - softWrap
>是否自动换行，若为false，文字将不考虑容器大小，单行显示，超出屏幕部分将默认截断处理


```dart
Column(
    children: <Widget>[
        Text(
            text,
            softWrap: false,
            style: TextStyle(
                color: Colors.green,
            ),
        ),
        Text(
            text,
            softWrap: true,
            style: TextStyle(
                color: Colors.blue,
            ),
        ),
    ],
)
```
运行结果如下图所示
![enter image description here](https://github.com/Yuzopro/image/blob/master/flutter/flutter_widgets_text_5.png?raw=true)

 - overflow
> 处理溢出文本，参数如下面表格所示

![enter image description here](https://github.com/Yuzopro/image/blob/master/flutter/flutter_widgets_text_13.png?raw=true)
```dart
Column(
    children: <Widget>[
        Text(
            text,
            overflow: TextOverflow.clip,
            maxLines: 3,
            style: TextStyle(
                color: Colors.green,
            ),
        ),
        Text(
            text,
            overflow: TextOverflow.fade,
            maxLines: 3,
            style: TextStyle(
                color: Colors.blue,
            ),
        ),
        Text(
            text,
            overflow: TextOverflow.ellipsis,
            maxLines: 3,
            style: TextStyle(
                color: Colors.yellow,
            ),
        ),
    ],
)
```
运行结果如下图所示
![enter image description here](https://github.com/Yuzopro/image/blob/master/flutter/flutter_widgets_text_6.png?raw=true)


 - textScaleFactor
>字体显示倍率

```dart
Column(
    children: <Widget>[
        Text(
            text,
            style: TextStyle(
                color: Colors.green,
            ),
        ),
        Text(
            text,
            textScaleFactor: 2,
            style: TextStyle(
                color: Colors.blue,
            ),
        ),
          ],
        )
```
运行结果如下图所示
![enter image description here](https://github.com/Yuzopro/image/blob/master/flutter/flutter_widgets_text_7.png?raw=true)

 - maxLines
>最大行数

```dart
Column(
    children: <Widget>[
        Text(
            text,
            style: TextStyle(
                color: Colors.green,
            ),
        ),
        Text(
            text,
            maxLines: 3,
            style: TextStyle(
                color: Colors.blue,
            ),
        ),
    ],
)
```
运行结果如下图所示
![enter image description here](https://github.com/Yuzopro/image/blob/master/flutter/flutter_widgets_text_8.png?raw=true)
 
## Text.rich
在上面的例子中，Text的所有文本内容只能按同一种样式进行展示，如果我们需要对一个Text内容的不同部分按照不同的样式显示，那又该怎么处理？此时需要用到`Text.rich`。
### 构造函数
```dart
const Text.rich(this.textSpan, {
    Key key,
    this.style,
    this.strutStyle,
    this.textAlign,
    this.textDirection,
    this.locale,
    this.softWrap,
    this.overflow,
    this.textScaleFactor,
    this.maxLines,
    this.semanticsLabel,
}) : assert(
        textSpan != null,
        'A non-null TextSpan must be provided to a Text.rich widget.',
    ),
    data = null,
    super(key: key);
```
### 属性
`Text.rich`中除了`textSpan`属性跟`Text`不一样，其他都一样。`textSpan`属性如下所示

#### TextSpan
`TextSpan`定义如下
```dart
const TextSpan({
    this.style,
    this.text,
    this.children,
    this.recognizer,
});
```
其中`style`和`text`属性代表该文本的样式和内容；`children`是一个TextSpan的数组，也就是说`TextSpan`可以包括其他`TextSpan`；而`recognizer`用于对该文本片段上用于手势进行识别处理。下面看一个例子
```dart
Text.rich(TextSpan(
    children: [
        TextSpan(
            text: "百度: "
        ),
        TextSpan(
            text: "https://www.baidu.com",
            style: TextStyle(
                color: Colors.blue
            ),
        ),
    ]
))
```
运行结果如下图所示
![enter image description here](https://github.com/Yuzopro/image/blob/master/flutter/flutter_widgets_text_9.png?raw=true)

## DefaultTextStyle
在widget树中，文本的样式默认是可以被继承的，父节点的文本样式子节点默认会继承，如果子节点中重新设置了默认样式的某些属性，那么则以子节点设置的为准。我们也可以通过设置`inherit: false`不继承父节点的默认样式。下面我们看一个例子
```dart
DefaultTextStyle(
    textAlign: TextAlign.left,
    style: TextStyle(
            fontSize: 20.0,
            color: Colors.green,
            fontWeight: FontWeight.bold,
            fontStyle: FontStyle.italic),
            child: Column(
                children: <Widget>[
                  Text(
                    text,
                  ),
                  Text(
                    text,
                    style: TextStyle(
                      color: Colors.yellow,
                    ),
                  ),
                  Text(
                    text,
                    style: TextStyle(
                      inherit: false,
                      color: Colors.blue,
                    ),
                  ),
                ],
        ))
```
运行结果如下图所示
![enter image description here](https://github.com/Yuzopro/image/blob/master/flutter/flutter_widgets_text_10.png?raw=true)
 
好了，`Text`相关的内容大概就这么多。更详细的请看[官方文档](https://docs.flutter.io/flutter/widgets/Text-class.html)