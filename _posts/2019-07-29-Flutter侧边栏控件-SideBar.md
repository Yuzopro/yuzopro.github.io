---
layout:     post
title:      "Flutter侧边栏控件-SideBar"
subtitle:   "Flutter第一个自定义控件SideBar"
date:       2019-07-29 20:00:00
author:     "Yuzo"
header-img: "img/post-bg-flutter-sidebar.jpeg"
catalog: true
tags:
    - 移动端
    - Android
    - Flutter
    - iOS
---

# Flutter侧边栏控件-SideBar

---

## 前言

SideBar是APP开发当中常见的功能之一，多用于索引列表，如城市选择，分类等。在优化[OpenGit](https://github.com/Yuzopro/opengit_flutter)趋势列表时，由于在选择语言时需要用到这样的控件，尝试开发了这个控件，效果如下图所示

![](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/flutter_side_bar_demo.gif)

## 准备

完成SideBar需要向外提供以下参数

 1. SideBar宽以及每个letter的高度；
 2. 默认背景色和文本颜色；
 3. 按下时的背景色和文本颜色；
 4. 当前选中letter的回调；
 5. 索引列表；

选中letter的回掉函数如下所示

```dart
typedef OnTouchingLetterChanged = void Function(String letter);
```

索引列表数据如下面代码所示

```dart
const List<String> A_Z_LIST = const [
  "A",
  "B",
  "C",
  "D",
  "E",
  "F",
  "G",
  "H",
  "I",
  "J",
  "K",
  "L",
  "M",
  "N",
  "O",
  "P",
  "Q",
  "R",
  "S",
  "T",
  "U",
  "V",
  "W",
  "X",
  "Y",
  "Z",
  "#"
];
```
 
当按下SideBar时需要刷新UI，所以SideBar需要继承`StatefulWidget`，构造函数如下所示

```dart
class SideBar extends StatefulWidget {
  SideBar({
    Key key,
    @required this.onTouch,
    this.width = 30,
    this.letterHeight = 16,
    this.color = Colors.transparent,
    this.textStyle = const TextStyle(
      fontSize: 12.0,
      color: Color(YZColors.subTextColor),
    ),
    this.touchDownColor = const Color(0x40E0E0E0),
    this.touchDownTextStyle = const TextStyle(
      fontSize: 12.0,
      color: Color(YZColors.mainTextColor),
    ),
  });

  final int width;

  final int letterHeight;

  final Color color;

  final Color touchDownColor;

  final TextStyle textStyle;

  final TextStyle touchDownTextStyle;

  final OnTouchingLetterChanged onTouch;
}
```

## 封装SideBar

在`_SideBarState`中，需要通过touch的状态来判断背景色的展示，相关代码如下所示
```dart
class _SideBarState extends State<SideBar> {
  bool _isTouchDown = false;

  @override
  Widget build(BuildContext context) {
    return Container(
      alignment: Alignment.center,
      color: _isTouchDown ? widget.touchDownColor : widget.color,
      width: widget.width.toDouble(),
      child: _SlideItemBar(
        letterWidth: widget.width,
        letterHeight: widget.letterHeight,
        textStyle: _isTouchDown ? widget.touchDownTextStyle : widget.textStyle,
        onTouch: (letter) {
          if (widget.onTouch != null) {
            setState(() {
              _isTouchDown = !TextUtil.isEmpty(letter);
            });
            widget.onTouch(letter);
          }
        },
      ),
    );
  }
}
```
上面代码，主要部分通过`_SlideItemBar`的letter状态的改变来刷新`Container`的color，下面看下`_SlideItemBar`的实现，相关代码如下所示
```dart
class _SlideItemBar extends StatefulWidget {
  final int letterWidth;

  final int letterHeight;

  final TextStyle textStyle;

  final OnTouchingLetterChanged onTouch;

  _SlideItemBar(
      {Key key,
      @required this.onTouch,
      this.letterWidth = 30,
      this.letterHeight = 16,
      this.textStyle})
      : assert(onTouch != null),
        super(key: key);

  @override
  _SlideItemBarState createState() {
    return _SlideItemBarState();
  }
}
```
上文代码，没有做过多的操作，只是定义了几个变量，详细的操作在`_SlideItemBarState`。
在`_SlideItemBarState`中，需要知道每个`letter`在垂直方向上的偏移高度，如下面代码所示
```dart
void _init() {
    _letterPositionList.clear();
    _letterPositionList.add(0);
    int tempHeight = 0;
    A_Z_LIST?.forEach((value) {
      tempHeight = tempHeight + widget.letterHeight;
      _letterPositionList.add(tempHeight);
    });
}
```
填充每个letter widget，并设置固定宽高，代码如下所示
```dart
List<Widget> children = List();
A_Z_LIST.forEach((v) {
    children.add(SizedBox(
        width: widget.letterWidth.toDouble(),
        height: widget.letterHeight.toDouble(),
        child: Text(v, textAlign: TextAlign.center, style: _style),
    ));
});
```
在滑动`SideBar`过程中，需要检测手势事件，代码如下所示
```dart
GestureDetector(
      onVerticalDragDown: (DragDownDetails details) {
        //计算索引列表距离顶部的距离
        if (_widgetTop == -1) {
          RenderBox box = context.findRenderObject();
          Offset topLeftPosition = box.localToGlobal(Offset.zero);
          _widgetTop = topLeftPosition.dy.toInt();
        }
        //获取touch点在索引列表的偏移值
        int offset = details.globalPosition.dy.toInt() - _widgetTop;
        int index = _getIndex(offset);
        //判断索引是否在列表中，如果存在，则通知上层更新数据
        if (index != -1) {
          _lastIndex = index;
          _triggerTouchEvent(A_Z_LIST[index]);
        }
      },
      onVerticalDragUpdate: (DragUpdateDetails details) {
        //获取touch点在索引列表的偏移值
        int offset = details.globalPosition.dy.toInt() - _widgetTop;
        int index = _getIndex(offset);
        //并且前后两次的是否一致，如果不一致，则通知上层更新数据
        if (index != -1 && _lastIndex != index) {
          _lastIndex = index;
          _triggerTouchEvent(A_Z_LIST[index]);
        }
      },
      onVerticalDragEnd: (DragEndDetails details) {
        _lastIndex = -1;
        _triggerTouchEvent('');
      },
      onTapUp: (TapUpDetails details) {
        _lastIndex = -1;
        _triggerTouchEvent('');
      },
      //填充UI
      child: Column(
        mainAxisSize: MainAxisSize.min,
        children: children,
      ),
    )
```
上文代码，在`onVerticalDragDown`事件时，首次获取索引距离顶部的高度，并通过touch点的y坐标获取到touch点在偏移值，并通过该值找到目前touch的索引，记录该状态，并通知上层ui；在`onVerticalDragUpdate`事件时，获取索引跟`onVerticalDragDown`一致，只是多了出重的操作。而`onVerticalDragEnd`和`onTapUp`代表touch事件的结束。到此，可以获取到触摸SideBar时，回掉的`letter`数据。

## 展示letter

`SideBar`通常展示在`ListView`的上面，父容器我们采用`Stack`，如下面代码所示
```dart
@override
Widget build(BuildContext context) {
    super.build(context);

    return Scaffold(
      body: Stack(
        children: <Widget>[
          _buildSideBar(context),
          _buildLetterTips(),
        ],
      ),
    );
}

Widget _buildSideBar(BuildContext context) {
    return Offstage(
      offstage: widget.offsetBuilder == null,
      child: Align(
        alignment: Alignment.centerRight,
        child: SideBar(
          onTouch: (letter) {
            setState(() {
              _letter = letter;
            });
          },
        ),
      ),
    );
}

Widget _buildLetterTips() {
    return Offstage(
      offstage: TextUtil.isEmpty(_letter),
      child: Align(
        alignment: Alignment.center,
        child: Container(
          alignment: Alignment.center,
          width: 65.0,
          height: 65.0,
          color: Color(0x40000000),
          child: Text(
            TextUtil.isEmpty(_letter) ? '' : _letter,
            style: YZConstant.largeLargeTextWhite,
          ),
        ),
      ),
    );
}
```
当接收到`letter`发生改变时，会通过`setState`刷新ui，当`_letter`不为空时，就会展示当前letter的提示。

## 滚动ListView

滚动ListView目前只发现两种方法，如下面代码所示
```dart
//带动画的滚动
scrollController.animateTo(double offset);
//不带动画的滚动
scrollController.jumpTo(double offset);
```
由于上述两种方法都需要知道滚动的具体位置，所以需要知道ListView列表的每个item相对于屏幕顶部的偏移量，所以高度必须是固定的。

### 获取语言列表数据

调用接口[https://github-trending-api.now.sh/languages](https://github-trending-api.now.sh/languages)，并封装bean对象，如下面代码所示
```dart
List<TrendingLanguageBean> getTrendingLanguageBeanList(List<dynamic> list) {
  List<TrendingLanguageBean> result = [];
  list.forEach((item) {
    result.add(TrendingLanguageBean.fromJson(item));
  });
  return result;
}

@JsonSerializable()
class TrendingLanguageBean extends Object {
  @JsonKey(name: 'id')
  String id;

  @JsonKey(name: 'name')
  String name;

  String letter;

  bool isShowLetter;

  TrendingLanguageBean(this.id, this.name, {this.letter});

  factory TrendingLanguageBean.fromJson(Map<String, dynamic> srcJson) =>
      _$TrendingLanguageBeanFromJson(srcJson);

  Map<String, dynamic> toJson() => _$TrendingLanguageBeanToJson(this);
}
```
对获取到的数据进行排序，如下面代码所示
```dart
void _sortListByLetter(List<TrendingLanguageBean> list) {
    if (list == null || list.isEmpty) return;
    list.sort(
      (a, b) {
        if (a.letter == "@" || b.letter == "#") {
          return -1;
        } else if (a.letter == "#" || b.letter == "@") {
          return 1;
        } else {
          return a.letter.compareTo(b.letter);
        }
      },
    );
}
```
通过语言对应的首字母，设置其展示状态，如下面代码所示
```dart
void _setShowLetter(List<TrendingLanguageBean> list) {
    if (list != null && list.isNotEmpty) {
      String tempLetter;
      for (int i = 0, length = list.length; i < length; i++) {
        TrendingLanguageBean bean = list[i];
        String letter = bean.letter;
        if (tempLetter != letter) {
          tempLetter = letter;
          bean.isShowLetter = true;
        } else {
          bean.isShowLetter = false;
        }
      }
    }
}
```
列表数据已经准备完毕，初始化单个item的高度，如下面代码所示
```dart
double getLetterHeight() => 48.0;

double getItemHeight() => 56.0;
```
然后进一步计算每个letter在`ListView`中所处的高度，如下面代码所示
```dart
void _initListOffset(List<TrendingLanguageBean> list) {
    _letterOffsetMap.clear();
    double offset = 0;
    String letter;
    list?.forEach((v) {
      if (letter != v.letter) {
        letter = v.letter;
        _letterOffsetMap.putIfAbsent(letter, () => offset);
        offset = offset + getLetterHeight() + getItemHeight();
      } else {
        offset = offset + getItemHeight();
      }
    });
}
```
通过`letter`获取滚动的指定高度，如下面代码所示
```dart
double getOffset(String letter) => _letterOffsetMap[letter];
```
当获取到高度后，完成`ListView`的滚动，如下面代码所示
```dart
if (offset != null) {
    _scrollController.jumpTo(offset.clamp(
            .0, _scrollController.position.maxScrollExtent));
}
```

## 项目源码
[OpenGit_Fultter](https://github.com/Yuzopro/opengit_flutter)






