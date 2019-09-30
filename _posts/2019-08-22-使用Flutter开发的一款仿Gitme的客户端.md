---
layout:     post
title:      "使用Flutter开发的一款仿Gitme的客户端"
subtitle:   "OpenGit 1.3.0版本UI风格主要采用卡片式风格，实现了闪屏、引导、Github相关信息查看和修改等功能"
date:       2019-07-30 19:12:00
author:     "Yuzo"
header-img: "img/post-bg-opengit-introduce.jpeg"
catalog: true
tags:
    - 移动端
    - Android
    - Flutter
    - iOS
---

# 使用Flutter开发的一款仿Gitme的客户端

## 前言
离[上篇文章](https://yuzopro.github.io/2019/06/13/%E4%B8%80%E6%AC%BEFlutter%E5%AE%9E%E7%8E%B0%E7%9A%84Github%E5%AE%A2%E6%88%B7%E7%AB%AF%E4%B9%8BOpenGit/)介绍[OpenGit_Flutter](https://github.com/Yuzopro/opengit_flutter)已经过了两个月，在两个月期间完成了`v1.1.0`、`v1.2.0`以及下文马上介绍的`v1.3.0`版本，点击见[版本更新记录](https://github.com/Yuzopro/opengit_flutter/releases)。在`v1.3.0`版本中，对整体UI做了修改，采用卡片式风格；对登录界面做了改版，UI主要参考[flutter-ui-nice](https://github.com/nb312/flutter-ui-nice)；优化了编辑issue、评论相关逻辑，并增加标签功能；改版了个人资料页面，并增加`组织`相关逻辑，UI主要参考[flutter-ui-nice](https://github.com/nb312/flutter-ui-nice)；增加了分享功能等。`v1.3.0`版本相比较以前的版本，体验上做了较大的改动，下面一一介绍该客户端涉及到的相关内容。

该项目涉及到的主要架构，可以参考[MVC、MVP、BloC、Redux四种架构在Flutter上的尝试](https://yuzopro.github.io/2019/07/13/MVC-MVP-BloC-Redux%E5%9B%9B%E7%A7%8D%E6%9E%B6%E6%9E%84%E5%9C%A8Flutter%E4%B8%8A%E7%9A%84%E5%B0%9D%E8%AF%95/)。

项目中卡片式风格的主要代码如下所示

```dart
InkWell(
      child: Padding(
        padding: const EdgeInsets.all(8.0),
        child: _postCard(context, item),
      ),
      onTap: () {
        NavigatorUtil.goWebView(context, item.title, item.originalUrl);
      },
)

Widget _postCard(BuildContext context) {
    return Card(
      elevation: 2.0,
      child: ......
    );
}
```

## 程序入口

```dart
void main() {
  final store = Store<AppState>(
    appReducer,
    initialState: AppState.initial(),
    middleware: [
      LoginMiddleware(),
      UserMiddleware(),
      AboutMiddleware(),
    ],
  );

  runZoned(() {
    runApp(OpenGitApp(store));
  }, onError: (Object obj, StackTrace trace) {
    print(obj);
    print(trace);
  });
}
```

程序入口`main`方法内，进行了`redux`相关初始化操作，并启动了`OpenGitApp`页面。而`runZoned`是为了在运行环境内捕获全局异常等信息，便于分析问题。

下面看下`OpenGitApp`页面的相关代码，具体代码如下所示

```dart
class OpenGitApp extends StatefulWidget {
  final Store<AppState> store;

  OpenGitApp(this.store) {
    final router = Router();

    AppRoutes.configureRoutes(router);

    Application.router = router;
  }

  @override
  State<StatefulWidget> createState() {
    return _OpenGitAppState();
  }
}
```

在`OpenGitApp`构造函数内，完成了`Fluro`路由的相关初始化操作，关于`Fluro`后续会补充文章介绍。而闪屏页的定义如下面代码所示

```dart
static final splash = '/';

router.define(
    splash,
    handler: splashHandler,
    transitionType: TransitionType.cupertino,
);

var splashHandler = Handler(
    handlerFunc: (BuildContext context, Map<String, List<String>> params) {
  return SplashPage();
});
```

当`OpenGitApp`相关页面初始化功能加载完成后，默认会启动`SplashPage`页面，同时`_OpenGitAppState`类中，会进行相关数据的初始化功能，如下面代码所示

```dart
class _OpenGitAppState extends State<OpenGitApp> {
  static final String TAG = "OpenGitApp";

  @override
  void initState() {
    super.initState();
    widget.store.dispatch(InitAction());
  }
 }
```

在`initState`中发起`redux`初始化数据指令`InitAction`，当指令发出后，`UserMiddleware`会收到该指令，并对该指令做相应的处理，如下面代码所示

```dart
Future<Null> _init(Store<AppState> store, NextDispatcher next) async {
    //完成sp的初始化
    await SpUtil.instance.init();

    //初始化数据库，并进行删除操作
    CacheProvider provider = CacheProvider();
    await provider.delete();

    //主题
    int theme = SpUtil.instance.getInt(SP_KEY_THEME_COLOR);
    if (theme != 0) {
      Color color = Color(theme);
      next(RefreshThemeDataAction(AppTheme.changeTheme(color)));
    }
    //语言
    int locale = SpUtil.instance.getInt(SP_KEY_LANGUAGE_COLOR);
    if (locale != 0) {
      next(RefreshLocalAction(LocaleUtil.changeLocale(store.state, locale)));
    }
    //用户信息
    String token = SpUtil.instance.getString(SP_KEY_TOKEN);
    UserBean userBean = null;
    var user = SpUtil.instance.getObject(SP_KEY_USER_INFO);
    if (user != null) {
      LoginManager.instance.setUserBean(user, false);
      userBean = UserBean.fromJson(user);
    }
    LoginManager.instance.setToken(token, false);
    //引导页
    String version =
        SpUtil.instance.getString(SP_KEY_SHOW_GUIDE_VERSION);
    String currentVersion = Config.SHOW_GUIDE_VERSION;
    next(InitCompleteAction(token, userBean, currentVersion != version));
    //初始化本地数据
    ReposManager.instance.initLanguageColors();
}
```

## 闪屏页

![闪屏页](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/introduce_opengit/opengit_splash.png)

当进入到闪屏页后，通过`redux`启动页面的倒计时操作，如下面代码所示

```dart
store.dispatch(StartCountdownAction(context));
```

当发出倒计时指令后，`UserMiddleware`会收到该指令，并对该指令做相应的处理，如下面代码所示

```dart
void startCountdown(
      Store<AppState> store, NextDispatcher next, BuildContext context) {
    TimerUtil.startCountdown(5, (int count) {
      next(CountdownAction(count));

      if (count == 0) {
        _jump(context, store.state.userState.status,
            store.state.userState.isGuide);
      }
    });
}
```

通过`TimerUtil`启动一个5s的倒计时，并将倒计时的时间点同步给`SplashPage`页面，用来刷新倒计时时间，`TimerUtil`工具类不做过多介绍，细节可以参考[OpenGit_Flutter项目常用公共库总结](https://yuzopro.github.io/2019/07/30/OpenGit_Flutter%E9%A1%B9%E7%9B%AE%E5%B8%B8%E7%94%A8%E5%85%AC%E5%85%B1%E5%BA%93%E6%80%BB%E7%BB%93/)。当倒计时跑完之后，会通过用户初始化数据状态，进行页面跳转操作，如下面代码所示

```dart
void _jump(BuildContext context, LoginStatus status, bool isShowGuide) {
    if (isShowGuide) {
      NavigatorUtil.goGuide(context);
    } else if (status == LoginStatus.success) {
      NavigatorUtil.goMain(context);
    } else if (status == LoginStatus.error) {
      NavigatorUtil.goLogin(context);
    }
}
```

当用户是首次操作应用时，则跳转到`引导页`；如果已登录，则跳转`主页`，如果未登录；则跳转`登录页`。

## 引导页

![引导页](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/introduce_opengit/opengit_guide.png)  
![引导页](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/introduce_opengit/opengit_guide_1.png)
![引导页](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/introduce_opengit/opengit_guide_2.png)
![引导页](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/introduce_opengit/opengit_guide_3.png) 
![引导页](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/introduce_opengit/opengit_guide_4.png)

`引导页`相关代码参考[flutter_gallery](https://github.com/flutter/flutter/tree/master/examples/flutter_gallery)里的`animation`，这里不做过大介绍。当点击`立即体验`时，相关代码如下所示

```dart
void _onExperience(BuildContext context) {
    Store<AppState> store = StoreProvider.of(context);
    LoginStatus status = store.state.userState.status;
    if (status == LoginStatus.success) {
      NavigatorUtil.goMain(context);
    } else if (status == LoginStatus.error) {
      NavigatorUtil.goLogin(context);
    }
}
```

首先通过`redux`查询用户的登录状态，如果已登录，则跳转`主页`，如果未登录；则跳转`登录页`。

## 登录页

![登录页](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/introduce_opengit/opengit_login.png)

登录过程分为授权和获取用户资料，涉及到的api如下所示

* 授权api
    > POST /authorizations  
* 获取用户资料api
    > GET /user

当用户没有账号时，可以进行账号的注册，如下面代码所示

```dart
NavigatorUtil.goWebView(
    context,
    AppLocalizations.of(context).currentlocal.sign_up,
    'https://github.com/');
```

当用户存在账号，完成账号和密码的输入，点击登录，如下面代码所示

```dart
store.dispatch(FetchLoginAction(context, name, password));
```

当`LoginMiddleware`收到该指令，触发登录，如下面代码所示

```dart
Future<void> _doLogin(NextDispatcher next, BuildContext context,
      String userName, String password) async {
    next(RequestingLoginAction());

    try {
      LoginBean loginBean =
          await LoginManager.instance.login(userName, password);
      if (loginBean != null) {
        String token = loginBean.token;
        LoginManager.instance.setToken(loginBean.token, true);
        UserBean userBean = await LoginManager.instance.getMyUserInfo();
        if (userBean != null) {
          next(InitCompleteAction(token, userBean, false));
          next(ReceivedLoginAction(token, userBean));
          NavigatorUtil.goMain(context);
        } else {
          ToastUtil.showMessgae('登录失败请重新登录');
          LoginManager.instance.setToken(null, true);
        }
      } else {
        ToastUtil.showMessgae('登录失败请重新登录');
        next(ErrorLoadingLoginAction());
      }
    } catch (e) {
      LogUtil.v(e, tag: TAG);
      ToastUtil.showMessgae('登录失败请重新登录');
      next(ErrorLoadingLoginAction());
    }
}
```

当开始登录时，`redux`发出指令`RequestingLoginAction`加载`loading`界面，当登录成功后，会对`token`信息进行缓存，然后在获取用户资料，当用户资料获取成功后，则判断登录成功，跳转到主页面。

## 主页

主页的页面加载是采用`TabBar`+`PageView`(TabBarView慎用)组合加载`home`、`repo`、`event`、`issue`四个页面，关键代码如下所示

```dart
TabBar(
    controller: _tabController,
    labelPadding: EdgeInsets.all(8.0),
    indicatorColor: Colors.white,
    tabs: choices.map((Choice choice) {
         return Tab(
                    text: choice.title,
                );
         }).toList(),
    onTap: (index) {
        _pageController.jumpTo(ScreenUtil.getScreenWidth(context) * index);
    },
)

//慎用TabBarView，假如现在有四个tab，如果首次进入app之后，
//点击issue tab，动态 tab也会触发加载数据，并且立即销毁
PageView(
    controller: _pageController,
    physics: NeverScrollableScrollPhysics(),
    children: <Widget>[
        BlocProvider<HomeBloc>(
            child: HomePage(),
            bloc: _homeBloc,
        ),
        BlocProvider<ReposBloc>(
            child: ReposPage(PageType.repos),
            bloc: _reposBloc,
        ),
        BlocProvider<EventBloc>(
            child: EventPage(PageType.received_event),
            bloc: _eventBloc,
        ),
        BlocProvider<IssueBloc>(
            child: IssuePage(),
            bloc: _issueBloc,
        ),
     ],
     onPageChanged: (index) {
         _tabController.animateTo(index);
     },
)
```

## 首页

![首页](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/introduce_opengit/opengit_home.png)

首页展示的数据是获取掘金flutter列表，相关api如下所示

> GET https://timeline-merger-ms.juejin.im/v1/get_tag_entry?'
        'src=web&tagId=5a96291f6fb9a0535b535438&page=$page&pageSize=20&sort=rankIndex
        
涉及到的相关代码如下所示

```dart
Future _fetchHomeList() async {
    LogUtil.v('_fetchHomeList', tag: TAG);
    try {
      var result = await JueJinManager.instance.getJueJinList(page);
      if (bean.data == null) {
        bean.data = List();
      }
      if (page == 1) {
        bean.data.clear();
      }

      noMore = true;
      if (result != null) {
        bean.isError = false;
        noMore = result.length != Config.PAGE_SIZE;
        bean.data.addAll(result);
      } else {
        bean.isError = true;
      }

      sink.add(bean);
    } catch (_) {
      if (page != 1) {
        page--;
      }
    }
}
```
        
点击item，跳转到相应的h5页面，如下面代码所示

```dart
NavigatorUtil.goWebView(context, item.title, item.originalUrl)
```

## 项目页

![项目页](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/introduce_opengit/opengit_repo.png)

项目页展示的数据是自己已公开的项目列表，相关api如下所示

> GET /users/:username/repos

涉及到的相关代码如下所示

```dart
///repo_bloc.dart
Future _fetchReposList() async {
    LogUtil.v('_fetchReposList', tag: TAG);
    try {
      var result = await fetchRepos(page);
      if (bean.data == null) {
        bean.data = List();
      }
      if (page == 1) {
        bean.data.clear();
      }

      noMore = true;
      if (result != null) {
        bean.isError = false;
        noMore = result.length != Config.PAGE_SIZE;
        bean.data.addAll(result);
      } else {
        bean.isError = true;
      }

      sink.add(bean);
    } catch (_) {
      if (page != 1) {
        page--;
      }
    }
}

///repo_main_bloc.dart
@override
fetchRepos(int page) async {
    return await ReposManager.instance
        .getUserRepos(userName, page, null, false);
}
```
上面代码对请求项目相关接口进行下封装，主要逻辑在`repo_bloc.dart`中已经进行了处理，子类只需实现`fetchRepos`方法即可。

点击item，跳转至项目详情页，如下面代码所示

```dart
NavigatorUtil.goReposDetail(context, item.owner.login, item.name);
```

## 动态页

![动态页](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/introduce_opengit/opengit_event.png)

动态页展示的数据是已收到的动态列表，相关api如下所示

> GET /users/:username/received_events

涉及到的相关代码如下所示

```dart
///event_bloc.dart
Future _fetchEventList() async {
    LogUtil.v('_fetchEventList', tag: TAG);
    try {
      var result = await fetchEvent(page);
      if (bean.data == null) {
        bean.data = List();
      }
      if (page == 1) {
        bean.data.clear();
      }

      noMore = true;
      if (result != null) {
        bean.isError = false;
        noMore = result.length != Config.PAGE_SIZE;
        bean.data.addAll(result);
      } else {
        bean.isError = true;
      }

      sink.add(bean);
    } catch (_) {
      if (page != 1) {
        page--;
      }
    }
}

///received_event_Bloc
@override
fetchEvent(int page) async {
    return await EventManager.instance.getEventReceived(userName, page);
}
```

点击item，会区分不同事件，如果和issue相关事件，则跳转问题详情页，如果和项目相关事件，则跳转项目详情页，如下面代码所示

```dart
if (item.payload != null && item.payload.issue != null) {
    NavigatorUtil.goIssueDetail(context, item.payload.issue);
} else if (item.repo != null && item.repo.name != null) {
    String repoUser, repoName;
    if (item.repo.name.isNotEmpty && item.repo.name.contains("/")) {
        List<String> repos = TextUtil.split(item.repo.name, '/');
        repoUser = repos[0];
        repoName = repos[1];
    }
    NavigatorUtil.goReposDetail(context, repoUser, repoName);
}
```


## 问题页

![问题页](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/introduce_opengit/opengit_issue.png)

问题页展示的数据是已收到的问题列表，相关api如下所示

> GET /issues?filter=:filter&state=:state&sort=:sort&direction=:direction

涉及到的相关代码如下所示

```dart
Future _fetchIssueList() async {
    LogUtil.v('_fetchIssueList', tag: TAG);
    try {
      var result = await IssueManager.instance
          .getIssue(filter, state, sort, direction, page);
      if (bean.data == null) {
        bean.data = List();
      }
      if (page == 1) {
        bean.data.clear();
      }

      noMore = true;
      if (result != null) {
        bean.isError = false;
        noMore = result.length != Config.PAGE_SIZE;
        bean.data.addAll(result);
      } else {
        bean.isError = true;
      }

      sink.add(bean);
    } catch (_) {
      if (page != 1) {
        page--;
      }
    }
}
```

点击item，跳转至问题详情页，如下面代码所示

```dart
NavigatorUtil.goIssueDetail(context, item);
```

## 项目详情页

![项目详情页](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/introduce_opengit/opengit_repo_detail.png)

首次进入项目详情页时，会查询该项目的详情以及star和watch状态，相关api如下所示

* 项目详情
    > GET /repos/:owner/:repo
* star状态
    > GET /user/starred/:owner/:repo
* watch状态
    > GET /user/subscriptions/:owner/:repo

涉及到的相关代码如下所示

```dart
Future _fetchReposDetail() async {
    final repos =
        await ReposManager.instance.getReposDetail(reposOwner, reposName);
    bean.data.repos = repos;

    if (repos == null) {
      bean.isError = true;
    } else {
      bean.isError = false;
    }

    sink.add(bean);

    _fetchStarStatus();
    _fetchWatchStatus();
}

Future _fetchStarStatus() async {
    final response =
        await ReposManager.instance.getReposStar(reposOwner, reposName);
    bean.data.starStatus =
        response.result ? ReposStatus.active : ReposStatus.inactive;

    sink.add(bean);
}

Future _fetchWatchStatus() async {
    final response =
        await ReposManager.instance.getReposWatcher(reposOwner, reposName);
    bean.data.watchStatus =
        response.result ? ReposStatus.active : ReposStatus.inactive;

    sink.add(bean);
}
```

改变star和watch状态，相关api如下
添加

* star状态
    > PUT /user/starred/:owner/:repo
* watch状态
    > PUT /user/subscriptions/:owner/:repo


删除

* star状态
    > DELETE /user/starred/:owner/:repo
* watch状态
    > DELETE /user/subscriptions/:owner/:repo

涉及到的相关代码如下所示

```dart
void changeStarStatus() async {
    bool isEnable = bean.data.starStatus == ReposStatus.active;

    bean.data.starStatus = ReposStatus.loading;
    sink.add(bean);

    final response = await ReposManager.instance
        .doReposStarAction(reposOwner, reposName, isEnable);
    if (response.result) {
      if (isEnable) {
        bean.data.starStatus = ReposStatus.inactive;
      } else {
        bean.data.starStatus = ReposStatus.active;
      }
    }
    sink.add(bean);
}

void changeWatchStatus() async {
    bool isEnable = bean.data.watchStatus == ReposStatus.active;

    bean.data.watchStatus = ReposStatus.loading;
    sink.add(bean);

    final response = await ReposManager.instance
        .doReposWatcherAction(reposOwner, reposName, isEnable);
    if (response.result) {
      if (isEnable) {
        bean.data.watchStatus = ReposStatus.inactive;
      } else {
        bean.data.watchStatus = ReposStatus.active;
      }
    }
    sink.add(bean);
}
```

上述代码如果`isEnable`状态为`true`，则请求`DELETE`，反之是`PUT`

### 项目stars用户列表

相关api如下所示
> GET /repos/:owner/:repo/stargazers

涉及到的相关代码如下所示

```dart
///user_bloc.dart
Future _fetchUserList() async {
    LogUtil.v('_fetchUserList', tag: TAG);
    try {
      var result = await fetchList(page);
      if (bean.data == null) {
        bean.data = List();
      }
      if (page == 1) {
        bean.data.clear();
      }

      noMore = true;
      if (result != null) {
        bean.isError = false;
        noMore = result.length != Config.PAGE_SIZE;
        bean.data.addAll(result);
      } else {
        bean.isError = true;
      }

      sink.add(bean);
    } catch (_) {
      if (page != 1) {
        page--;
      }
    }
}
 
///stargazer_bloc.dart
@override
fetchList(int page) async {
    return await UserManager.instance.getStargazers(url, page);
}
```

上面代码对请求用户相关接口进行下封装，主要逻辑在`user_bloc.dart`中已经进行了处理，子类`stargazer_bloc`继承了`user_bloc`，只需实现`fetchList`方法即可

### 项目issues列表

相关api如下所示

> GET repos/:owner/:repo/issues

涉及到的相关代码如下所示

```dart
Future _fetchIssueList() async {
    LogUtil.v('_fetchIssueList', tag: TAG);
    try {
      var result = await IssueManager.instance.getRepoIssues(owner, repo, page);
      if (bean.data == null) {
        bean.data = List();
      }
      if (page == 1) {
        bean.data.clear();
      }

      noMore = true;
      if (result != null) {
        bean.isError = false;
        noMore = result.length != Config.PAGE_SIZE;
        bean.data.addAll(result);
      } else {
        bean.isError = true;
      }

      sink.add(bean);
    } catch (_) {
      if (page != 1) {
        page--;
      }
    }
}
```

### 项目forks用户列表

相关api如下所示

> GET repos/:owner/:repo/forks

涉及到的相关代码如下所示

```dart
@override
fetchList(int page) async {
    return await ReposManager.instance.getRepoForks(owner, repo, page);
}
```

由于`repo_fork_bloc`继承`user_bloc`所以只需实现`fetchList`即可，具体细节可以在上文查看

### 项目watchers用列表

相关api如下所示

> GET repos/:owner/:repo/subscribers

涉及到的相关代码如下所示

```dart
@override
fetchList(int page) async {
    return await UserManager.instance.getSubscribers(url, page);
}
```

由于`subscriber_bloc`继承`user_bloc`所以只需实现`fetchList`即可，具体细节可以在上文查看

### 项目语言趋势列表

相关api如下所示

> GET search/repositories?q=language:$language&sort=stars

涉及到的相关代码如下所示

```dart
Future _fetchTrendList() async {
    try {
      var result = await ReposManager.instance.getLanguages(language, page);
      if (bean.data == null) {
        bean.data = List();
      }
      if (page == 1) {
        bean.data.clear();
      }

      noMore = true;
      if (result != null) {
        bean.isError = false;
        noMore = result.length != Config.PAGE_SIZE;
        bean.data.addAll(result);
      } else {
        bean.isError = true;
      }

      sink.add(bean);
    } catch (_) {
      if (page != 1) {
        page--;
      }
    }
}
```

### 项目动态列表

相关api如下所示

> GET networks/:owner/:repo/events

涉及到的相关代码如下所示

```dart
Future _fetchEventList() async {
    try {
      var result = await ReposManager.instance
          .getReposEvents(reposOwner, reposName, page);
      if (bean.data == null) {
        bean.data = List();
      }
      if (page == 1) {
        bean.data.clear();
      }

      noMore = true;
      if (result != null) {
        bean.isError = false;
        noMore = result.length != Config.PAGE_SIZE;
        bean.data.addAll(result);
      } else {
        bean.isError = true;
      }

      sink.add(bean);
    } catch (_) {
      if (page != 1) {
        page--;
      }
    }
}
```

### 项目贡献者用户列表

相关api如下所示

> GET repos/:owner/:repo/contributors

涉及到的相关代码如下所示

```dart
@override
fetchList(int page) async {
    return await UserManager.instance.getContributors(url, page);
}
```

由于`contributor_bloc`继承`user_bloc`所以只需实现`fetchList`即可，具体细节可以在上文查看

### 项目分支列表

相关api如下所示

> GET repos/$owner/$repo/branches

涉及到的相关代码如下所示

```dart
void fetchBranches() async {
    final response =
        await ReposManager.instance.getBranches(reposOwner, reposName);
    bean.data.branchs = response;
    sink.add(bean);
}
```

### 项目详情列表

相关api如下所示

> GET repos/:owner/:repo/contents:path

涉及到的相关代码如下所示

```dart
Future _fetchSourceFile() async {
    String path = _getPath();
    final result = await ReposManager.instance
        .getReposFileDir(reposOwner, reposName, path: path, branch: branch);

    if (bean.data == null) {
      bean.data = List();
    }

    bean.data.clear();

    if (result != null) {
      bean.isError = false;
      bean.data.addAll(result);
    } else {
      bean.isError = true;
    }

    sink.add(bean);
}
```

点击详情列表，会区分文件夹、图片、文件详情三种三种场景，如下面代码所示

```dart
void _onItemClick(BuildContext context, SourceFileBean item) {
    bool isImage = ImageUtil.isImage(item.name);
    if (item.type == "dir") {
      RepoFileBloc bloc = BlocProvider.of<RepoFileBloc>(context);
      bloc.fetchNextDir(item.name);
    } else if (isImage) {
      NavigatorUtil.goPhotoView(context, item.name, item.htmlUrl + "?raw=true");
    } else {
      NavigatorUtil.goReposSourceCode(context, item.name,
          ImageUtil.isImage(item.url) ? item.downloadUrl : item.url);
    }
}
```
如果是文件夹状态则刷新该目录文件列表；如果是图片状态则调用图片加载页面，详见[OpenGit_Flutter项目常用公共库总结](https://yuzopro.github.io/2019/07/30/OpenGit_Flutter%E9%A1%B9%E7%9B%AE%E5%B8%B8%E7%94%A8%E5%85%AC%E5%85%B1%E5%BA%93%E6%80%BB%E7%BB%93/)；如果是详情状态则跳转详情页进行处理，具体如下所示

#### 项目详情页

涉及到的相关代码如下所示

```dart
getCodeDetail(url) async {
    final response =
        await _getFileAsStream(url, {"Accept": 'application/vnd.github.html'});
    String data = CodeDetailUtil.resolveHtmlFile(response, "java");
    String result = Uri.dataFromString(data,
            mimeType: 'text/html', encoding: Encoding.getByName("utf-8"))
        .toString();
    return result;
}

Widget build(BuildContext context) {
    if (data == null) {
      return Scaffold(
        appBar:CommonUtil.getAppBar(widget.title),
        body: Container(
          alignment: Alignment.center,
          child: Center(
            child: SpinKitCircle(
              color: Theme.of(context).primaryColor,
              size: 25.0,
            ),
          ),
        ),
      );
    }

    return Scaffold(
      appBar: CommonUtil.getAppBar(widget.title),
      body: WebView(
        initialUrl: data,
        javascriptMode: JavascriptMode.unrestricted,
      ),
    );
}
```

### 项目README

相关api如下所示

> GET repos/:owner/:repo/readme

涉及到的相关代码如下所示

```dart
void fetchReadme() async {
    final response =
        await ReposManager.instance.getReadme("$reposOwner/$reposName", null);
    bean.data.readme = response.data;
    sink.add(bean);
}
```

### 浏览器打开

涉及到的相关代码如下所示

```dart
@override
void openWebView(BuildContext context) {
    RepoDetailBloc bloc = BlocProvider.of<RepoDetailBloc>(context);
    NavigatorUtil.goWebView(
        context, bloc.reposName, bloc.bean.data.repos.htmlUrl);
}
```

### 分享

涉及到的相关代码如下所示

```dart
void _share(BuildContext context) {
    ShareUtil.share(getShareText(context));
}
  
@override
String getShareText(BuildContext context) {
    RepoDetailBloc bloc = BlocProvider.of<RepoDetailBloc>(context);
    return bloc.bean.data.repos.htmlUrl;
}
```

## 问题详情页

![问题详情页](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/introduce_opengit/opengit_issue_detail.png)

首次进入问题详情页时，会查询该问题的详情以及评论列表，相关api如下所示

* 问题详情
    > GET /repos/:owner/:repo/issues/:issue_number
* 评论列表
    > GET /repos/:owner/:repo/issues/:issue_number/comments

涉及到的相关代码如下所示

```dart
void _fetchIssueComment() async {
    IssueBean result =
        await IssueManager.instance.getSingleIssue(url, num);
    bean.data.issueBean = result;
}
  
Future _fetchIssueComments() async {
    try {
      var result = await IssueManager.instance
          .getIssueComment(url, num, page);
      if (bean.data == null) {
        bean.data.comments = List();
      }
      if (page == 1) {
        bean.data.comments.clear();
      }

      noMore = true;
      if (result != null) {
        noMore = result.length != Config.PAGE_SIZE;
        bean.data.comments.addAll(result);
      } else {
        bean.isError = true;
      }

      sink.add(bean);
    } catch (_) {
      if (page != 1) {
        page--;
      }
    }
}
```

### 添加评论

相关api如下所示

> POST /repos/:owner/:repo/issues/:issue_number/comments

涉及到的相关代码如下所示

```dart
 _editIssueComment() async {
    IssueBean result = null;
    _showLoading();
    if (!widget.isAdd) {
      result = await IssueManager.instance.editIssueComment(
          widget.repoUrl, widget.id, _controller.text.toString());
    } else {
      result = await IssueManager.instance.addIssueComment(
          widget.repoUrl, widget.id, _controller.text.toString());
    }
    _hideLoading();
    if (result != null) {
      Navigator.pop(context, result);
    }
 }
```

添加评论时`widget.isAdd = true`

### 编辑评论

相关api如下所示

> PATCH /repos/:owner/:repo/issues/:issue_number/comments

涉及到的相关代码如下所示

```dart
_editIssueComment() async {
    IssueBean result = null;
    _showLoading();
    if (!widget.isAdd) {
      result = await IssueManager.instance.editIssueComment(
          widget.repoUrl, widget.id, _controller.text.toString());
    } else {
      result = await IssueManager.instance.addIssueComment(
          widget.repoUrl, widget.id, _controller.text.toString());
    }
    _hideLoading();
    if (result != null) {
      Navigator.pop(context, result);
    }
}
```

编辑评论时`widget.isAdd = false`

### 删除评论

相关api如下所示

> DELETE /repos/:owner/:repo/issues/:issue_number/comments

涉及到的相关代码如下所示

```dart
void deleteIssueComment(IssueBean item) async {
    showLoading();
    int comment_id = item.id;
    final response =
        await IssueManager.instance.deleteIssueComment(url, comment_id);
    if (response != null && response.result) {
      bean.data.comments.remove(item);
      sink.add(bean);
    }
    hideLoading();
}
```

### 编辑问题

相关api如下所示

> PATCH /repos/:owner/:repo/issues/:issue_number

涉及到的相关代码如下所示

```dart
_editIssue() async {
    _showLoading();
    final result = await IssueManager.instance.editIssue(widget.url, widget.num,
        _titleController.text.toString(), _bodyController.text.toString());
    _hideLoading();
    if (result != null) {
      Navigator.pop(context, result);
    }
}
```

### Reactions

#### 问题Reactions列表

相关api如下所示

> GET /repos/:owner/:repo/issues/:issue_number/reactions

涉及到的相关代码如下所示

```dart
_queryIssueCommentReaction(IssueBean item, comment, isIssue) async {
    int id;
    if (isIssue) {
      id = item.number;
    } else {
      id = item.id;
    }
    final response = await IssueManager.instance
        .getCommentReactions(url, id, comment, 1, isIssue);
    ReactionDetailBean findReaction = null;
    if (response != null) {
      UserBean userBean = LoginManager.instance.getUserBean();
      for (int i = 0; i < response.length; i++) {
        ReactionDetailBean reactionDetailBean = response[i];
        if (reactionDetailBean != null &&
            reactionDetailBean.content == comment &&
            userBean != null &&
            reactionDetailBean.user != null &&
            userBean.login == reactionDetailBean.user.login) {
          findReaction = reactionDetailBean;
          break;
        }
      }
    }
    if (findReaction != null) {
      return await _deleteIssueCommentReaction(item, findReaction, comment);
    } else {
      return await _createIssueCommentReaction(item, comment, isIssue);
    }
}
```

#### 添加问题Reaction

相关api如下所示

> POST /repos/:owner/:repo/issues/:issue_number/reactions

涉及到的相关代码如下所示

```dart
_createIssueCommentReaction(IssueBean item, comment, isIssue) async {
    int id;
    if (isIssue) {
      id = item.number;
    } else {
      id = item.id;
    }
    final response =
        await IssueManager.instance.editReactions(url, id, comment, isIssue);
    if (response != null && response.result) {
      _addIssueBean(item, comment);
      sink.add(bean);
    }
    return response;
}
  
IssueBean _addIssueBean(IssueBean issueBean, String comment) {
    if (issueBean.reaction == null) {
      issueBean.reaction = ReactionBean('', 0, 0, 0, 0, 0, 0, 0, 0, 0);
    }
    if ("+1" == comment) {
      issueBean.reaction.like++;
    } else if ("-1" == comment) {
      issueBean.reaction.noLike++;
    } else if ("hooray" == comment) {
      issueBean.reaction.hooray++;
    } else if ("eyes" == comment) {
      issueBean.reaction.eyes++;
    } else if ("laugh" == comment) {
      issueBean.reaction.laugh++;
    } else if ("confused" == comment) {
      issueBean.reaction.confused++;
    } else if ("rocket" == comment) {
      issueBean.reaction.rocket++;
    } else if ("heart" == comment) {
      issueBean.reaction.heart++;
    }
    return issueBean;
}
```

#### 评论Reactions列表

相关api如下所示

> GET /repos/:owner/:repo/issues/comments/:comment_id/reactions

涉及到的相关代码如下所示

```dart
_queryIssueCommentReaction(IssueBean item, comment, isIssue) async {
    int id;
    if (isIssue) {
      id = item.number;
    } else {
      id = item.id;
    }
    final response = await IssueManager.instance
        .getCommentReactions(url, id, comment, 1, isIssue);
    ReactionDetailBean findReaction = null;
    if (response != null) {
      UserBean userBean = LoginManager.instance.getUserBean();
      for (int i = 0; i < response.length; i++) {
        ReactionDetailBean reactionDetailBean = response[i];
        if (reactionDetailBean != null &&
            reactionDetailBean.content == comment &&
            userBean != null &&
            reactionDetailBean.user != null &&
            userBean.login == reactionDetailBean.user.login) {
          findReaction = reactionDetailBean;
          break;
        }
      }
    }
    if (findReaction != null) {
      return await _deleteIssueCommentReaction(item, findReaction, comment);
    } else {
      return await _createIssueCommentReaction(item, comment, isIssue);
    }
}
```

#### 添加评论Reaction

相关api如下所示

> POST /repos/:owner/:repo/issues/comments/:comment_id/reactions

涉及到的相关代码如下所示

```dart
_createIssueCommentReaction(IssueBean item, comment, isIssue) async {
    int id;
    if (isIssue) {
      id = item.number;
    } else {
      id = item.id;
    }
    final response =
        await IssueManager.instance.editReactions(url, id, comment, isIssue);
    if (response != null && response.result) {
      _addIssueBean(item, comment);
      sink.add(bean);
    }
    return response;
}
  
IssueBean _addIssueBean(IssueBean issueBean, String comment) {
    if (issueBean.reaction == null) {
      issueBean.reaction = ReactionBean('', 0, 0, 0, 0, 0, 0, 0, 0, 0);
    }
    if ("+1" == comment) {
      issueBean.reaction.like++;
    } else if ("-1" == comment) {
      issueBean.reaction.noLike++;
    } else if ("hooray" == comment) {
      issueBean.reaction.hooray++;
    } else if ("eyes" == comment) {
      issueBean.reaction.eyes++;
    } else if ("laugh" == comment) {
      issueBean.reaction.laugh++;
    } else if ("confused" == comment) {
      issueBean.reaction.confused++;
    } else if ("rocket" == comment) {
      issueBean.reaction.rocket++;
    } else if ("heart" == comment) {
      issueBean.reaction.heart++;
    }
    return issueBean;
}
```

#### 删除Reaction

相关api如下所示

> DELETE /reactions/:reaction_id

涉及到的相关代码如下所示

```dart
  _deleteIssueCommentReaction(
      IssueBean issueBean, ReactionDetailBean item, content) async {
    final response = await IssueManager.instance.deleteReactions(item.id);
    _subtractionIssueBean(issueBean, content);
    sink.add(bean);
    return response;
  }
  
    IssueBean _subtractionIssueBean(IssueBean issueBean, String comment) {
    if ("+1" == comment) {
      issueBean.reaction.like--;
    } else if ("-1" == comment) {
      issueBean.reaction.noLike--;
    } else if ("hooray" == comment) {
      issueBean.reaction.hooray--;
    } else if ("eyes" == comment) {
      issueBean.reaction.eyes--;
    } else if ("laugh" == comment) {
      issueBean.reaction.laugh--;
    } else if ("confused" == comment) {
      issueBean.reaction.confused--;
    } else if ("rocket" == comment) {
      issueBean.reaction.rocket--;
    } else if ("heart" == comment) {
      issueBean.reaction.heart--;
    }
    return issueBean;
  }
```

### 浏览器打开

涉及到的相关代码如下所示

```dart
@override
void openWebView(BuildContext context) {
    IssueDetailBloc bloc = BlocProvider.of<IssueDetailBloc>(context);
    NavigatorUtil.goWebView(
        context, bloc.getTitle(), bloc.bean.data?.issueBean?.htmlUrl);
}
```

### 分享

涉及到的相关代码如下所示

```dart
void _share(BuildContext context) {
    ShareUtil.share(getShareText(context));
}
  
@override
void openWebView(BuildContext context) {
    IssueDetailBloc bloc = BlocProvider.of<IssueDetailBloc>(context);
    NavigatorUtil.goWebView(
        context, bloc.getTitle(), bloc.bean.data?.issueBean?.htmlUrl);
}
```

### 标签

#### 查询项目标签列表

相关api如下所示

> GET /repos/:owner/:repo/labels

涉及到的相关代码如下所示

```dart
Future _fetchLabelList() async {
    LogUtil.v('_fetchLabelList', tag: TAG);
    try {
      var result = await IssueManager.instance.getLabel(owner, repo, page);
      if (bean.data == null) {
        bean.data = List();
      }
      if (page == 1) {
        bean.data.clear();
      }

      noMore = true;
      if (result != null) {
        bean.isError = false;
        noMore = result.length != Config.PAGE_SIZE;
        bean.data.addAll(result);
      } else {
        bean.isError = true;
      }

      sink.add(bean);
    } catch (_) {
      if (page != 1) {
        page--;
      }
    }
}
```

#### 添加标签

相关api如下所示

> POST /repos/:owner/:repo/labels

涉及到的相关代码如下所示

```dart
_editOrCreateLabel() async {
    String name = _nameController.text.toString();
    if (TextUtil.isEmpty(name)) {
      ToastUtil.showMessgae('名称不能为空');
      return;
    }

    String desc = _descController.text.toString() ?? '';

    UserBean userBean = LoginManager.instance.getUserBean();
    String owner = userBean?.login;

    String color = ColorUtil.color2RGB(_currentColor);

    _showLoading();

    var response;
    if (_isCreate) {
      response = await IssueManager.instance
          .createLabel(owner, widget.repo, name, color, desc);
    } else {
      response = await IssueManager.instance
          .updateLabel(owner, widget.repo, widget.item.name, name, color, desc);
    }
    if (response != null && response.result) {
      Labels labels = Labels(widget.item?.id, widget.item?.nodeId,
          widget.item?.url, name, desc, color, widget.item?.default_);
      Navigator.pop(context, labels);
    } else {
      ToastUtil.showMessgae('操作失败，请重试');
    }
    _hideLoading();
}
```

#### 编辑标签

相关api如下所示

> PATCH /repos/:owner/:repo/labels/:current_name

涉及到的相关代码如下所示

```dart
_editOrCreateLabel() async {
    String name = _nameController.text.toString();
    if (TextUtil.isEmpty(name)) {
      ToastUtil.showMessgae('名称不能为空');
      return;
    }

    String desc = _descController.text.toString() ?? '';

    UserBean userBean = LoginManager.instance.getUserBean();
    String owner = userBean?.login;

    String color = ColorUtil.color2RGB(_currentColor);

    _showLoading();

    var response;
    if (_isCreate) {
      response = await IssueManager.instance
          .createLabel(owner, widget.repo, name, color, desc);
    } else {
      response = await IssueManager.instance
          .updateLabel(owner, widget.repo, widget.item.name, name, color, desc);
    }
    if (response != null && response.result) {
      Labels labels = Labels(widget.item?.id, widget.item?.nodeId,
          widget.item?.url, name, desc, color, widget.item?.default_);
      Navigator.pop(context, labels);
    } else {
      ToastUtil.showMessgae('操作失败，请重试');
    }
    _hideLoading();
}
```

#### 删除标签

相关api如下所示

> DELETE /repos/:owner/:repo/labels/:name

涉及到的相关代码如下所示

```dart
void _deleteLabel() async {
    UserBean userBean = LoginManager.instance.getUserBean();
    String owner = userBean?.login;

    _showLoading();

    var response = await IssueManager.instance
        .deleteLabel(owner, widget.repo, widget.item.name);
    if (response != null && response.result) {
      widget.item.id = -1;
      Navigator.pop(context, widget.item);
    } else {
      ToastUtil.showMessgae('操作失败，请重试');
    }

    _hideLoading();
}
```

#### 添加某个问题的标签

相关api如下所示

> POST /repos/:owner/:repo/issues/:issue_number/labels

涉及到的相关代码如下所示

```dart
void addIssueLabel(Labels label) async {
    showLoading();

    var result = await IssueManager.instance
        .addIssueLabel(owner, repo, issueNum, label.name);
    if (result != null && result.result) {
      if (labels == null) {
        labels = [];
      }
      labels.add(label);
    } else {
      ToastUtil.showMessgae('操作失败，请重试');
    }

    hideLoading();
}
```

#### 删除某个问题的标签

相关api如下所示

> DELETE /repos/:owner/:repo/issues/:issue_number/labels/:name

涉及到的相关代码如下所示

```dart
void deleteIssueLabel(String name) async {
    showLoading();
    var result = await IssueManager.instance
        .deleteIssueLabel(owner, repo, issueNum, name);
    if (result != null && result.result) {
      if (labels != null) {
        int deleteIndex = -1;
        for (int i = 0; i < labels.length; i++) {
          Labels item = labels[i];
          if (TextUtil.equals(item.name, name)) {
            deleteIndex = i;
          }
        }
        if (deleteIndex != null) {
          labels.removeAt(deleteIndex);
        }
      }
    } else {
      ToastUtil.showMessgae('操作失败，请重试');
    }
    hideLoading();
}
```

## 用户资料页

![个人资料页](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/introduce_opengit/opengit_my_profile.png) ![follow页](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/introduce_opengit/opengit_user_profile_follow.png) 
![unfollow页](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/introduce_opengit/opengit_user_profile_unfollow.png)

用户资料页展示了用户的昵称、简介、项目列表、star项目列表、关注列表、被关注列表、动态、所在组织、公司、地址、邮箱、博客等信息。首次进入用户资料页时，会查询该用户的详情以及关注状态，相关api如下所示

* 用户资料
    > GET /users/:name
* 关注状态
    > GET /user/following/:name

涉及到的相关代码如下所示

```dart
Future _fetchProfile() async {
    final result = await UserManager.instance.getUserInfo(name);
    bean.data = result;

    if (result == null) {
      bean.isError = true;
    } else {
      bean.isError = false;
    }
}

Future _fetchFollow() async {
    if (!UserManager.instance.isYou(name) && bean.data != null) {
      final response = await UserManager.instance.isFollow(name);
      bool isFollow = false;
      if (response != null && response.result) {
        isFollow = true;
      }
      bean.data.isFollow = isFollow;
    }
}
```

在查询关注状态时，需要判断该用户是否是自己，如果是，则不进行查询操作

### 关注用户

相关api如下所示

> PUT /user/following/:username

涉及到的相关代码如下所示

```dart
Future _follow() async {
    final response = await UserManager.instance.follow(name);
    if (response != null && response.result) {
      bean.data.isFollow = true;
      sink.add(bean);
    } else {
      ToastUtil.showMessgae('操作失败请重试');
    }
}
```

### 取消关注用户

相关api如下所示

> DELETE /user/following/:username

涉及到的相关代码如下所示

```dart
Future _follow() async {
    final response = await UserManager.instance.follow(name);
    if (response != null && response.result) {
      bean.data.isFollow = true;
      sink.add(bean);
    } else {
      ToastUtil.showMessgae('操作失败请重试');
    }
}
```

### 项目列表

见上文的`项目页`

### star项目列表

相关api如下所示

> GET /users/:username/starred

涉及到的相关代码如下所示

```dart
@override
fetchRepos(int page) async {
    return await ReposManager.instance.getUserRepos(userName, page, null, true);
}
```

由于`repo_user_star_bloc.dart`继承`repo_bloc.dart`所以只需实现`fetchRepos`即可，具体细节可以在上文查看

### 关注列表

相关api如下所示

> GET /users/:username/following

涉及到的相关代码如下所示

```dart
@override
fetchList(int page) async {
    return await UserManager.instance.getUserFollower(userName, page);
}
```

由于`following_bloc`继承`user_bloc`所以只需实现`fetchList`即可，具体细节可以在上文查看

### 被关注列表

相关api如下所示

> GET /users/:username/followers

涉及到的相关代码如下所示

```dart
@override
fetchList(int page) async {
    return await UserManager.instance.getUserFollowing(userName, page);
}
```

由于`followers_bloc`继承`user_bloc`所以只需实现`fetchList`即可，具体细节可以在上文查看

### 动态

相关api如下所示

> GET users/:userName/events

涉及到的相关代码如下所示

```dart
@override
fetchEvent(int page) async {
    return await EventManager.instance.getEvent(userName, page);
}
```

由于`user_event_bloc`继承`event_bloc`所以只需实现`fetchEvent`即可，具体细节可以在上文查看

### 组织

相关api如下所示

> GET users/:userName/orgs

涉及到的相关代码如下所示

```dart
Future _fetchProfile() async {
    final result = await UserManager.instance.getOrgs(name, page);
    if (bean.data == null) {
      bean.data = List();
    }
    if (page == 1) {
      bean.data.clear();
    }

    noMore = true;
    if (result != null) {
      bean.isError = false;
      noMore = result.length != Config.PAGE_SIZE;
      bean.data.addAll(result);
    } else {
      bean.isError = true;
    }

    sink.add(bean);
}
```

## 编辑资料

![编辑资料页](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/introduce_opengit/opengit_edit_profile.png)

编辑资料页支持昵称、邮箱、博客、公司、所在地、简介的编辑，相关api如下所示

> PATCH /user

涉及到的相关代码如下所示

```dart
void _editProfile() async {
    String name = _name.text;
    String email = _email.text;
    String blog = _blog.text;
    String company = _company.text;
    String location = _location.text;
    String bio = _bio.text;

    if (TextUtil.equals(name, _userBean.name) &&
        TextUtil.equals(email, _userBean.email) &&
        TextUtil.equals(blog, _userBean.blog) &&
        TextUtil.equals(company, _userBean.company) &&
        TextUtil.equals(location, _userBean.location) &&
        TextUtil.equals(bio, _userBean.bio)) {
      ToastUtil.showMessgae('没有进行任何修改，请重新操作');

      return;
    }

    _showLoading();
    var response = await UserManager.instance
        .updateProfile(name, email, blog, company, location, bio);
    if (response != null && response.result) {
      _userBean.name = name;
      _userBean.email = email;
      _userBean.blog = blog;
      _userBean.company = company;
      _userBean.location = location;
      _userBean.bio = bio;

      LoginManager.instance.setUserBean(_userBean.toJson, true);
      Navigator.pop(context);
    } else {
      ToastUtil.showMessgae('操作失败，请重试');
    }
    _hideLoading();
}
```

## 搜索页面

![搜索项目页](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/introduce_opengit/opengit_search_repo.png) ![搜索用户页](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/introduce_opengit/opengit_search_user.png) ![搜索问题页](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/introduce_opengit/opengit_search_issue.png)

### 搜索项目页

相关api如下所示

> GET search/repositories?q=:query

涉及到的相关代码如下所示

```dart
void startSearch(String text) async {
    searchText = text;
    showLoading();
    await _searchText();
    hideLoading();

    refreshStatusEvent();
}

Future _searchText() async {
    final response =
        await SearchManager.instance.getIssue(type, searchText, page);
    if (response != null && response.result) {
      dealResult(response.data);
    }
}

@override
void dealResult(result) {
    if (bean.data == null) {
      bean.data = List();
    }
    if (page == 1) {
      bean.data.clear();
    }

    noMore = true;
    if (result != null && result.length > 0) {
      var items = result["items"];
      noMore = items.length != Config.PAGE_SIZE;
      for (int i = 0; i < items.length; i++) {
        var dataItem = items[i];
        Repository repository = Repository.fromJson(dataItem);
        repository.description =
            ReposUtil.getGitHubEmojHtml(repository.description ?? "暂无描述");
        bean.data.add(repository);
      }
    } else {
      bean.isError = true;
    }

    sink.add(bean);
}
```

### 搜索用户页

相关api如下所示

> GET search/users?q=:query

涉及到的相关代码如下所示

```dart
void startSearch(String text) async {
    searchText = text;
    showLoading();
    await _searchText();
    hideLoading();

    refreshStatusEvent();
}

Future _searchText() async {
    final response =
        await SearchManager.instance.getIssue(type, searchText, page);
    if (response != null && response.result) {
      dealResult(response.data);
    }
}

@override
void dealResult(result) {
    if (bean.data == null) {
      bean.data = List();
    }
    if (page == 1) {
      bean.data.clear();
    }

    noMore = true;
    if (result != null && result.length > 0) {
      var items = result["items"];
      noMore = items.length != Config.PAGE_SIZE;
      for (int i = 0; i < items.length; i++) {
        var dataItem = items[i];
        UserBean user = UserBean.fromJson(dataItem);
        bean.data.add(user);
      }
    } else {
      bean.isError = true;
    }

    sink.add(bean);
}
```

### 搜索问题页

相关api如下所示

> GET search/issues?q=:query

涉及到的相关代码如下所示

```dart
void startSearch(String text) async {
    searchText = text;
    showLoading();
    await _searchText();
    hideLoading();

    refreshStatusEvent();
}

Future _searchText() async {
    final response =
        await SearchManager.instance.getIssue(type, searchText, page);
    if (response != null && response.result) {
      dealResult(response.data);
    }
}

@override
void dealResult(result) {
    if (bean.data == null) {
      bean.data = List();
    }
    if (page == 1) {
      bean.data.clear();
    }

    noMore = true;
    if (result != null && result.length > 0) {
      var items = result["items"];
      noMore = items.length != Config.PAGE_SIZE;
      for (int i = 0; i < items.length; i++) {
        var dataItem = items[i];
        IssueBean issue = IssueBean.fromJson(dataItem);
        bean.data.add(issue);
      }
    } else {
      bean.isError = true;
    }

    sink.add(bean);
}
```

## 趋势页

![搜索项目页](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/introduce_opengit/opengit_trending.png)
![搜索用户页](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/introduce_opengit/opengit_trending_sort.png)

趋势页分为项目和用户两种趋势，支持按照时间和语言种类的筛选，api主要参考[Github-trending-api](https://github.com/huchenme/github-trending-api)

### 项目页

相关api如下所示

> GET https://github-trending-api.now.sh/repositories?language=:language&since=:since

涉及到的相关代码如下所示

```dart
Future _fetchTrendList() async {
    LogUtil.v('_fetchTrendList', tag: TAG);
    try {
      var result = await TrendingManager.instance.getRepos(language, since);
      if (bean.data == null) {
        bean.data = List();
      }
      bean.data.clear();
      if (result != null) {
        bean.isError = false;
        bean.data.addAll(result);
      } else {
        bean.isError = true;
      }
      sink.add(bean);
    } catch (_) {}
}
```

### 用户页

相关api如下所示

> GET https://github-trending-api.now.sh/developers?language=:language&since=:since

涉及到的相关代码如下所示

```dart
Future _fetchTrendList() async {
    LogUtil.v('_fetchTrendList', tag: TAG);
    try {
      var result = await TrendingManager.instance.getUser(language, since);
      if (bean.data == null) {
        bean.data = List();
      }
      bean.data.clear();
      if (result != null) {
        bean.isError = false;
        bean.data.addAll(result);
      } else {
        bean.isError = true;
      }
      sink.add(bean);
    } catch (_) {}
}
```

### 语言列表页

语言页面可以参考文章[Flutter侧边栏控件-SideBar](https://yuzopro.github.io/2019/07/29/Flutter%E4%BE%A7%E8%BE%B9%E6%A0%8F%E6%8E%A7%E4%BB%B6-SideBar/)

相关api如下所示

> GET https://github-trending-api.now.sh/languages

涉及到的相关代码如下所示

```dart
Future _fetchTrendList() async {
    LogUtil.v('_fetchTrendList', tag: TAG);
    try {
      var result = await TrendingManager.instance.getUser(language, since);
      if (bean.data == null) {
        bean.data = List();
      }
      bean.data.clear();
      if (result != null) {
        bean.isError = false;
        bean.data.addAll(result);
      } else {
        bean.isError = true;
      }
      sink.add(bean);
    } catch (_) {}
}
```

## Android版安装包：
[点击下载](https://github.com/Yuzopro/opengit_flutter/releases/download/1.3.0/opengit-release-1.3.0.apk)

扫码下载

![](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/flutter_opengit_3.png)

## 项目地址

[OpenGit客户端](https://github.com/Yuzopro/OpenGit_Flutter)

[flutter_common_lib](https://github.com/Yuzopro/flutter_common_lib)

## 关于作者

- [个人博客](https://yuzopro.github.io/)

- [Github](https://github.com/yuzopro)

- [掘金](https://juejin.im/user/56ea9d7ca341310054a57b7c)

- [简书](https://www.jianshu.com/u/ef3cb65219d4)

- [CSDN](https://blog.csdn.net/Yuzopro)