---
layout:     post
title:      "Flutter可滚动Widgets-ListView"
subtitle:   "flutter可滚动Widgets之ListView的基本用法"
date:       2019-05-13 19:32:00
author:     "Yuzo"
header-img: "img/post-bg-flutter-listview.jpg"
catalog: true
tags:
    - 移动端
    - Android
    - Flutter
    - iOS
---

## ListView
先看下如下截图
![enter image description here](https://user-gold-cdn.xitu.io/2019/5/13/16aaf47a1c6315cc?w=360&h=640&f=png&s=109793)
以上效果图的代码，是从`flutter`官方demo`flutter_gallery`内copy的部分代码。
首先，首先定义一个列表，代码如下
```dart
List<String> items = <String>[
    'A', 'B', 'C', 'D', 'E', 'F', 'G', 'H', 'I', 'J', 'K', 'L', 'M', 'N',
];
```
然后，通过上面的定义的列表数据，现在构建`ListView`的子Widget数据，代码如下
```dary
Iterable<Widget> listTiles = items
    .map<Widget>((String item) => buildListTile(context, item));

Widget buildListTile(BuildContext context, String item) {
    Widget secondary = const Text(
      'Even more additional list item information appears on line three.',
    );
    return ListTile(
      isThreeLine: true,
      leading: ExcludeSemantics(child: CircleAvatar(child: Text(item))),
      title: Text('This item represents $item.'),
      subtitle: secondary,
      trailing: Icon(Icons.info, color: Theme.of(context).disabledColor),
    );
}
```
最后，将生成的子Widget数据填充到`ListView`内，代码如下
```dart
ListView(
    children: listTiles.toList(),
)
```
以上代码，就能完成最上面截图的效果。下面主要对`ListTile`做一下介绍
## ListTile
`ListTile`是`Flutter`给我们准备好的widget提供非常常见的构造和定义方式，包括文字，icon，点击事件，一般是能够满足基本列表需求。
### 构造函数
```dart
ListTile({
    Key key,
    this.leading,
    this.title,
    this.subtitle,
    this.trailing,
    this.isThreeLine = false,
    this.dense,
    this.contentPadding,
    this.enabled = true,
    this.onTap,
    this.onLongPress,
    this.selected = false,
 })
```
### 属性
![enter image description here](https://user-gold-cdn.xitu.io/2019/5/13/16aaf47a0b8cee71?w=1087&h=577&f=png&s=30551)
### 使用
```dart
ListTile(
    //展示三行
    isThreeLine: true,
    //前置图标
    leading: ExcludeSemantics(child: CircleAvatar(child: Text(item))),
    //标题
    title: Text('This item represents $item.'),
    //副标题
    subtitle: secondary,
    //后置图标
    trailing: Icon(Icons.info, color: Theme.of(context).disabledColor),
)
```
### 效果
![enter image description here](https://user-gold-cdn.xitu.io/2019/5/13/16aaf47a13dc0876?w=476&h=102&f=png&s=13040)

## ListView.builder
ListView.builder适合列表项比较多（或者无限）的情况，因为只有当子Widget真正显示的时候才会被创建。
将上面列表填充的代码修改为`ListView.builder`，代码如下所示
```dart
ListView.builder(
    itemCount: items.length,
    itemBuilder: (BuildContext context, int index) {
        return buildListTile(context, items[index]);
})
```
运行结果如下图所示
![enter image description here](https://user-gold-cdn.xitu.io/2019/5/13/16aaf47a1c6315cc?w=360&h=640&f=png&s=109793)
## ListView.separated
`ListView.separated`可以生成列表项之间的分割器，它比ListView.builder多了一个separatorBuilder参数，该参数是一个分割器生成器。
将上面列表填充的代码修改为`ListView.separated`，代码如下所示
```dart
ListView.separated(
    itemBuilder: (BuildContext context, int index) {
        return buildListTile(context, items[index]);
    },
    separatorBuilder: (BuildContext context, int index) {
        return index % 2 == 0 ? divider1 : divider2;
    },
    itemCount: items.length
)
```
运行结果如下图所示
![enter image description here](https://user-gold-cdn.xitu.io/2019/5/13/16aaf47a0ac39bef?w=360&h=640&f=png&s=96048)
## 实例：Listview下拉刷新 上拉加载更多
下面实现首次进入页面，加载数据，下拉能刷新页面数据，上拉能加载更多数据。
### 下拉刷新
下拉刷新，用到的是`Flutter`自带的`RefreshIndicator`Widget,`ListView`主要用`ListView.builder`进行实现。代码如下所示
```dart
RefreshIndicator(
    key: refreshIndicatorKey,
    child: ListView.builder(
        itemCount: list.length,
        itemBuilder: (context, index) {
            return buildListTile(context, list[index]);
        },
    ),
    onRefresh: onRefresh)
```
实现下拉刷新，主要需要实现`RefreshIndicator`的`onRefresh`属性，代码如下所示
```dart
Future<Null> onRefresh() async {
    return Future.delayed(Duration(seconds: 2)).then((e) {
      list.addAll(items);
      setState(() {
        //重新构建列表
      });
    });
}
```
主要实现延迟2s加载数据，在重新刷新列表。
首次进入页面，`Loading`状态的实现实现如下面代码所示
```dart
void showRefreshLoading() async {
    await Future.delayed(const Duration(seconds: 0), () {
      refreshIndicatorKey.currentState.show().then((e) {});
      return true;
    });
}
```
当`Loading`完之后会触发`RefreshIndicator`的`onRefresh`属性，到此，下拉刷新已经实现完毕。
运行效果如下图所示
![enter image description here](https://user-gold-cdn.xitu.io/2019/5/13/16aaf47a334c88db?w=320&h=569&f=gif&s=71085)
### 上拉加载更多
上拉加载需要监听`ListView`的滚动事件，当滚动事件与底部小于50并且有更多数据加载时，才会触发加载更多的逻辑，如下面代码所示
```dart
scrollController.addListener(() {
    var position = scrollController.position;
    // 小于50px时，触发上拉加载；
    if (position.maxScrollExtent - position.pixels < 50 &&
        !isNoMore) {
        loadMore();
    }
});

void loadMore() async {
    if (!isLoading) {
      //刷新加载状态
      isLoading = true;
      setState(() {});

      Future.delayed(Duration(seconds: 2)).then((e) {
        list.addAll(items);
        //取消加载状态，并提示暂无更多数据
        isLoading = false;
        isNoMore = true;
        setState(() {
          //重新构建列表
        });
      });
    }
}
```
视图层的代码，当需要处理加载更多的逻辑时，`ListView`的`itemCount`属性需要进行加1，用来填充加载更多的视图。如下面代码所示
```dart
int getListItemCount() {
    int count = list.length;
    if (isLoading || isNoMore) {
      count += 1;
    }
    return count;
}
```
`ListView`的`itemBuilder`属性，加载更多的视图代码如下所示
```dart
Widget builderMore() {
    return Center(
      child: Padding(
        padding: EdgeInsets.all(10.0),
        child: Row(
          mainAxisAlignment: MainAxisAlignment.center,
          crossAxisAlignment: CrossAxisAlignment.center,
          children: <Widget>[
            isNoMore
                ? Text("")
                : SizedBox(
                    width: 20.0,
                    height: 20.0,
                    child: CircularProgressIndicator(
                        strokeWidth: 4.0,
                        valueColor: AlwaysStoppedAnimation(Colors.black)),
                  ),
            Padding(
              padding: EdgeInsets.symmetric(vertical: 5.0, horizontal: 15.0),
              child: Text(
                isNoMore ? "没有更多数据" : "加载中...",
                style: TextStyle(fontSize: 16.0),
              ),
            ),
          ],
        ),
      ),
    );
  }
```
对`RefreshIndicator`代码做如下修改
```dart
RefreshIndicator(
    key: refreshIndicatorKey,
    child: ListView.builder(
        controller: scrollController,
        itemCount: getListItemCount(),
        itemBuilder: (context, index) {
              return builderItem(context, index);
        },
    ),
    onRefresh: onRefresh)
          
Widget builderItem(BuildContext context, int index) {
    if (index < list.length) {
      return buildListTile(context, list[index]);
    }
    return builderMore();
}
```
运行代码
加载中的效果如下图所示
![enter image description here](https://user-gold-cdn.xitu.io/2019/5/13/16aaf47a1b5fccff?w=360&h=640&f=png&s=98117)
没有更多数据的效果如下图所示
![enter image description here](https://user-gold-cdn.xitu.io/2019/5/13/16aaf47b30641e18?w=360&h=640&f=png&s=99259)