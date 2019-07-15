---
layout:     post
title:      "MVC、MVP、BloC、Redux四种架构在Flutter上的尝试"
subtitle:   "花了将近一个月的时间实践，总算找到适合OpenGit项目的架构了"
date:       2019-07-13 22:16:00
author:     "Yuzo"
header-img: "img/post-bg-flutter-architecture.jpeg"
catalog: true
tags:
    - 移动端
    - Android
    - Flutter
    - iOS
---

# MVC、MVP、BloC、Redux四种架构在Flutter上的尝试

---
## 前言
从进行开发[OpenGit_Flutter](https://github.com/Yuzopro/OpenGit_Flutter)项目以来，在项目中选择哪种架构困扰了很久。近段时间，分别在项目中尝试了`BloC`、`Redux`这两种架构，通过开发中遇到的问题，已经找到了合适的方案。为了演示方便，我选择了该项目的登录流程来为大家做演示，下面对登录流程做下拆解。

 1. 登录首先需要输入账号和密码，只有在账号和密码都有输入的时候，底部登录按钮才能点击，所以需要监听账号和密码输入框的输入状态，用来控制登录按钮的点击状态；
 2. 账号输入框需要支持一键删除的功能；
 3. 密码输入框需要支持对密码可见的功能；
 4. 点击登录按钮触发登录逻辑，在登录过程中需要展示loading界面，当登录失败后，取消loading界面，并进行toast提示；当登录成功之后，跳转的主界面，展示用户的基本信息；
 5. 用户资料和token等信息的保存，在本文中不会提到，如需查看该部分代码，点击[OpenGit_Flutter](https://github.com/Yuzopro/OpenGit_Flutter)；

最终的演示效果如下所示
![](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/flutter_architecture_demo.gif)
![](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/flutter_architecture_demo.png)

登录界面的布局代码，不做过多的介绍，如果需要了解更多，可以查看相关源码，地址会在本文的最后贴出。

## 工程结构
[flutter_architecture](https://github.com/Yuzopro/flutter_architecture)根目录是一个`Flutter Package`，其下面分别创建了`bloc`、`mvc`、`mvp`、`redux`四个工程，`lib`目录分别是四个工程的公用模块，例如网络请求、日志打印、toast提示、主页信息展示等。如下图所示
![](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/flutter_architecture_project.png)

## MVC
该架构是在写[flutter_architecture](https://github.com/Yuzopro/flutter_architecture)例子时最后加上的，因为在进行`Android`开发的过程中，经常用它来与`MVP`做对比。

### 架构视图
![](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/flutter_architecture_mvc.png)

### 程序入口
`main.dart`是程序的入口，完成登录界面的启动，相关代码如下所示
```dart
void main() => runApp(MVCApp());

class MVCApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      theme: ThemeData(
        primaryColor: Colors.black,
      ),
      home: LoginPage(),
    );
  }
}
```
### 登录流程
由于登录状态涉及到界面相关控件的刷新，所以继承的是`StatefulWidget`。

#### 文本监听
文本监听需要监听账号和密码输入框的输入状态，需声明两个`TextEditingController`对象，相关代码如下所示
```dart
final TextEditingController _nameController = new TextEditingController();
final TextEditingController _passwordController = new TextEditingController();
```
在`initState`处理输入框的监听事件，当输入状态改变时，并刷新页面，更新登录按钮状态，相关代码如下所示
```dart
@override
void initState() {
    super.initState();

    _nameController.addListener(() {
      setState(() {});
    });
    _passwordController.addListener(() {
      setState(() {});
    });
}
```
登录按钮状态判断需要通过账号和密码输入字符串长短来判断，当长度都大于0的时候，按钮才能点击，逻辑层相关代码如下所示
```dart
_isValidLogin() {
    String name = _nameController.text;
    String password = _passwordController.text;

    return name.length > 0 && password.length > 0;
}
```
登录按钮UI层代码如下所示
```dart
Align _buildLoginButton(BuildContext context) {
    return Align(
      child: SizedBox(
        height: 45.0,
        width: 270.0,
        child: RaisedButton(
          child: Text(
            '登录',
            style: Theme.of(context).primaryTextTheme.headline,
          ),
          color: Colors.black,
          onPressed: _isValidLogin()
              ? () {
                  _login();
                }
              : null,
          shape: StadiumBorder(side: BorderSide()),
        ),
      ),
    );
}
```

#### 清空账号输入框
清空输入框只需调用`TextEditingController` clear方法，如下面代码所示
```dart
  TextFormField _buildNameTextField() {
    return new TextFormField(
      controller: _nameController,
      decoration: new InputDecoration(
        labelText: 'Github账号:',
        suffixIcon: new GestureDetector(
          onTap: () {
            _nameController.clear();
          },
          child: new Icon(_nameController.text.length > 0 ? Icons.clear : null),
        ),
      ),
      maxLines: 1,
    );
  }
```

#### 密码是否可见
密码是否可见主要是通过更新变量`_obscureText`实现，点击事件处理逻辑很简单，只是对`_obscureText`做下取反操作，并刷新页面，代码如下所示
```dart
TextFormField _buildPasswordTextField(BuildContext context) {
    return new TextFormField(
      controller: _passwordController,
      decoration: new InputDecoration(
        labelText: 'Github密码:',
        suffixIcon: new GestureDetector(
          onTap: () {
            setState(() {
              _obscureText = !_obscureText;
            });
          },
          child:
              new Icon(_obscureText ? Icons.visibility_off : Icons.visibility),
        ),
      ),
      maxLines: 1,
      obscureText: _obscureText,
    );
}
```

#### 触发登录
`View`层点击登录按钮，触发`Control`层登录逻辑，在`Control`层通过state控制loading界面的展示和隐藏，而loading的最终状态是由`Model`层的loading状态决定，loading UI相关代码如下所示：
```dart
Offstage(
    offstage: !Con.isLoading,
    child: new Container(
        width: MediaQuery.of(context).size.width,
        height: MediaQuery.of(context).size.height,
        color: Colors.black54,
            child: new Center(
                child: SpinKitCircle(
                  color: Theme.of(context).primaryColor,
                  size: 25.0,
                ),
            ),
    ),
),
```

##### 定义Control层
首先创建单例对象，并初始化`Model`层数据，向`View`层提供登录、加载状态、用户资料等状态的接口，相关代码如下所示
```dart
class Con {
  factory Con() => _getInstance();

  static Con get instance => _getInstance();
  static Con _instance;

  Con._internal();

  static Con _getInstance() {
    if (_instance == null) {
      _instance = new Con._internal();
    }
    return _instance;
  }

  static final model = Model();

  static bool get isLoading => model.isLoading;

  static UserBean get userBean => model.userBean;

  Future login(State state, String name, String password) async {
    state.setState(() {
      _showLoading();
    });

    await model.login(name, password);

    state.setState(() {
      _hideLoading();
    });
  }

  void _showLoading() {
    model.showLoading();
  }

  void _hideLoading() {
    model.hideLoading();
  }
}
```

##### 定义Model层
`Model`层主要进行登录、获取用户资料的网络请求，并保存loading状态以及用户资料，相关代码如下所示
```dart
class Model {
  bool get isLoading => _isLoading;
  bool _isLoading = false;

  UserBean get userBean => _userBean;
  UserBean _userBean;

  Future login(String name, String password) async {
    final login = await LoginManager.instance.login(name, password);
    //授权成功
    if (login != null) {
      final user = await LoginManager.instance.getMyUserInfo();
      _userBean = user;
    }
    return;
  }

  void showLoading() {
    _isLoading = true;
  }

  void hideLoading() {
    _isLoading = false;
  }
}
```
网络层的相关代码就不再贴出，感兴趣的可以在本文末尾下载源码进行查看。

由上面代码可知，当`View`层触发登录时，调用了`Control`层`login`接口，在该接口内，实现了展示loading状态，并等待登录的网络请求，当请求完成后，则取消loading状态，最终交给`View`层进行数据处理，相关处理的代码如下所示
```dart
_login() async {
    String name = _nameController.text;
    String password = _passwordController.text;

    await Con.instance.login(this, name, password);

    if (Con.userBean != null) {
      NavigatorUtil.goHome(context, Con.userBean);
    } else {
      ToastUtil.showToast('登录失败，请重新登录');
    }
}
```
到此，MVC整个框架的登录流程已进行完成。

## MVP
该架构是在进行Android开发时，是一种比较常用的架构。

### 架构视图
![](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/flutter_architecture_mvp.png)

### 程序入口
`main.dart`是程序的入口，完成登录界面的启动，相关代码如下所示
```dart
void main() => runApp(MVPApp());

class MVPApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      theme: ThemeData(
        primaryColor: Colors.black,
      ),
      home: LoginPage(),
    );
  }
}
```
### 登录流程
与`MVC`一致，可以参考`MVC`。

#### 文本监听
与`MVC`一致，可以参考`MVC`。

#### 清空账号输入框
与`MVC`一致，可以参考`MVC`。

#### 密码是否可见
与`MVC`一致，可以参考`MVC`。

#### 触发登录
`View`层点击登录按钮，触发`Presenter`层登录逻辑，在`Presenter`层通过`View`层提供的接口来控制loading界面的展示和隐藏，loading UI相关代码如下所示
```dart
Offstage(
    offstage: !isLoading,
    child: new Container(
        width: MediaQuery.of(context).size.width,
        height: MediaQuery.of(context).size.height,
        color: Colors.black54,
        child: new Center(
            child: SpinKitCircle(
                color: Theme.of(context).primaryColor,
                size: 25.0,
            ),
        ),
    ),
),
```
上面代码中的`isLoading`状态已在基类中做了统一封装，下面对`MVP`做定义封装。

##### 封装View层
触发网络请求，需要满足展示和隐藏loading界面，所以`View`的对外需要提供这两个最基本的接口，如下面代码所示
```dart
abstract class IBaseView {
  showLoading();

  hideLoading();
}
```

##### 封装Presenter层
`Presenter`层的公共接口只需提供对`View`层的注册以及反注册，如下面代码所示
```dart
abstract class IBasePresenter<V extends IBaseView> {
  void onAttachView(V view);

  void onDetachView();
}
```
下面对`Presenter`层的代码实现上面所提供的接口，如下面代码所示
```dart
abstract class BasePresenter<V extends IBaseView> extends IBasePresenter<V> {
  V view;

  @override
  void onAttachView(IBaseView view) {
    this.view = view;
  }

  @override
  void onDetachView() {
    this.view = null;
  }
}
```

##### 封装State基类
在`State`基类中，需要提供`Presenter`的初始化的方法、loading状态、数据初始化以及视图的构建等，如下面代码所示
```dart
abstract class BaseState<T extends StatefulWidget, P extends BasePresenter<V>,
    V extends IBaseView> extends State<T> implements IBaseView {
  P presenter;

  bool isLoading = false;

  P initPresenter();

  Widget buildBody(BuildContext context);

  void initData() {
  }

  @override
  void initState() {
    super.initState();
    presenter = initPresenter();
    if (presenter != null) {
      presenter.onAttachView(this);
    }
    initData();
  }

  @override
  void dispose() {
    super.dispose();
    if (presenter != null) {
      presenter.onDetachView();
      presenter = null;
    }
  }

  @override
  @mustCallSuper
  Widget build(BuildContext context) {
    return new Scaffold(
      body: buildBody(context),
    );
  }

  @override
  void showLoading() {
    setState(() {
      isLoading = true;
    });
  }

  @override
  void hideLoading() {
    setState(() {
      isLoading = false;
    });
  }
}
```
到此，`MVP`框架已经封装完，下面只需对登录界面做相应的实现即可。

##### 实现登录逻辑
在进行登录时，`Presenter`层需要向`View`层提供登录接口，当进行登录完毕后，需要向`View`层进行登录状态的反馈，所以`View`需要提供登录成功、失败两个接口，如下面代码所示
```dart
abstract class ILoginPresenter<V extends ILoginView> extends BasePresenter<V> {
  void login(String name, String password);
}

abstract class ILoginView extends IBaseView {
  void onLoginSuccess(UserBean userBean);

  void onLoginFailed();
}
```
当相关接口定义完毕后，首先实现登录的`Presenter`层的代码，如下面代码所示
```dart
class LoginPresenter extends ILoginPresenter {
  @override
  void login(String name, String password) async {
    if (view != null) {
      view.showLoading();
    }
    final login = await LoginManager.instance.login(name, password);
    //授权成功
    if (login != null) {
      final user = await LoginManager.instance.getMyUserInfo();
      if (user != null) {
        if (view != null) {
          view.hideLoading();
          view.onLoginSuccess(user);
        } else {
          view.hideLoading();
          view.onLoginFailed();
        }
      }
    } else {
      if (view != null) {
        view.hideLoading();
        view.onLoginFailed();
      }
    }
  }
}
```
然后对登录`State`的代码进行实现，如下面代码所示
```dart
class _LoginPageState extends BaseState<LoginPage, LoginPresenter, ILoginView>
    implements ILoginView {

  @override
  void initData() {
    super.initData();
  }

  @override
  Widget buildBody(BuildContext context) {
   return null;
  }

  @override
  LoginPresenter initPresenter() {
    return LoginPresenter();
  }

  @override
  void onLoginSuccess(UserBean userBean) {
    NavigatorUtil.goHome(context, userBean);
  }

  @override
  void onLoginFailed() {
    ToastUtil.showToast('登录失败，请重新登录');
  }
}
```
相关代码已经封装完毕，最后只需调用登录相关逻辑，如下面代码所示
```dart
 _login() {
    if (presenter != null) {
      String name = _nameController.text;
      String password = _passwordController.text;
      presenter.login(name, password);
    }
}
```

## BloC
关于什么是BloC，可以参考[[Flutter Package]状态管理之BLoC的封装](https://juejin.im/post/5be42e9e5188256ccc192c68)和[Flutter | 状态管理探索篇——BLoC(三)](https://juejin.im/post/5bb6f344f265da0aa664d68a)。

### 架构视图
![](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/flutter_architecture_bloc.png)

### 程序入口
`main.dart`是程序的入口，完成登录界面的启动，相关代码如下所示
```dart
void main() => runApp(BlocApp());

class BlocApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      theme: ThemeData(
        primaryColor: Colors.black,
      ),
      home: BlocProvider<LoginBloc>(
        child: LoginPage(),
        bloc: LoginBloc(),
      ),
    );
  }
}
```
上面代码跟`MVC`和`MVP`有不同之处，传入`home`的对象是`BlocProvider`，且其包含了`child`和`bloc`实例。如下面代码所示
```dart
class BlocProvider<T extends BaseBloc> extends StatefulWidget {
  final T bloc;
  final Widget child;

  BlocProvider({
    Key key,
    @required this.child,
    @required this.bloc,
  }) : super(key: key);

  @override
  _BlocProviderState<T> createState() {
    return _BlocProviderState<T>();
  }

  static T of<T extends BaseBloc>(BuildContext context) {
    final type = _typeOf<BlocProvider<T>>();
    BlocProvider<T> provider = context.ancestorWidgetOfExactType(type);
    return provider.bloc;
  }

  static Type _typeOf<T>() => T;
}

class _BlocProviderState<T> extends State<BlocProvider<BaseBloc>> {
  static final String TAG = "_BlocProviderState";

  @override
  void initState() {
    super.initState();
    LogUtil.v('initState ' + T.toString(), tag: TAG);
  }

  @override
  Widget build(BuildContext context) {
    LogUtil.v('build ' + T.toString(), tag: TAG);
    return widget.child;
  }

  @override
  void dispose() {
    super.dispose();
    LogUtil.v('dispose ' + T.toString(), tag: TAG);
    widget.bloc.dispose();
  }
}
```

### 登录流程
BLoC能够允许我们分离业务逻辑，不用考虑什么时候需要刷新屏幕，一切交给StreamBuilder和BLoC就可以完成，所以登录页面继承`StatelessWidget`即可。如下面代码所示
```dart
class LoginPage extends StatelessWidget {
    @override
    Widget build(BuildContext context) {
        return StreamBuilder(
            stream: bloc.stream,
            initialData: initialData(),
            builder: (BuildContext context,
            AsyncSnapshot<LoadingBean<LoginBlocBean>> snapshot) {
            }
        );
    }
}
```

 - `stream`代表了这个stream builder监听的流，这里监听的是`LoginBloc`的stream；
 - `initData`代表初始的值，因为在首次渲染的时候，还未与用户产生交互，也就不会有事件从流中流出，所以需要给首次渲染一个初始值；
 - `builder`函数接收一个位置参数BuildContext和一个snapshot，snapshot就是这个流输出的数据的一个快照，我们可以通过snapshot.data访问快照中的数据，StreamBuilder中的builder是一个AsyncWidgetBuilder，它能够异步构建widget，当检测到有数据从流中流出时，将会重新构建。

#### 创建BloC
首先完成`BloC`基类的封装，基类需要只需要满足登录状态，如下面代码所示
```dart
class LoadingBean<T> {
  bool isLoading;
  T data;

  LoadingBean({this.isLoading, this.data});

  @override
  String toString() {
    return 'LoadingBean{isLoading: $isLoading, data: $data}';
  }
}

abstract class BaseBloc<T extends LoadingBean> {
  static final String TAG = "BaseBloc";

  BehaviorSubject<T> _subject = BehaviorSubject<T>();

  Sink<T> get sink => _subject.sink;

  Stream<T> get stream => _subject.stream;

  void dispose() {
    _subject.close();
    sink.close();
  }
}
```

#### 创建BloC实例
在登录的`BloC`实例中，完成整个登录过程，我们需要监听账号、密码的输入状态，密码的是否可见状态，以及登录状态，如下面代码所示
```dart
class LoginBloc extends BaseBloc<LoadingBean<LoginBlocBean>> {
  LoadingBean<LoginBlocBean> bean;

  LoginBloc() {
    bean = LoadingBean<LoginBlocBean>(
      isLoading: false,
      data: LoginBlocBean(
        name: '',
        password: '',
        obscure: true,
      ),
    );
  }

  changeObscure() {
  }

  changeName(String name) {
  }

  changePassword(String password) {
  }

  login(BuildContext context) async {
  }

  void _showLoading() {
    bean.isLoading = true;
    sink.add(bean);
  }

  void _hideLoading() {
    bean.isLoading = false;
    sink.add(bean);
  }
}
```

#### 文本监听
创建账号和密码两个`TextEditingController`实例，并完成其事件监听，如下面代码所示
```dart
final TextEditingController _nameController = new TextEditingController();
final TextEditingController _passwordController = new TextEditingController();

LoginBloc bloc = BlocProvider.of<LoginBloc>(context);

_nameController.addListener(() {
    bloc.changeName(_nameController.text);
});
_passwordController.addListener(() {
    bloc.changePassword(_passwordController.text);
});
```
当文本发生改变时，会调用`LoginBloc`里相应的改变方法，并对相应的文本进行重新复杂，在通过`sink.add()`更新界面，如下面代码所示
```dart
changeName(String name) {
    bean.data.name = name;
    sink.add(bean);
}

changePassword(String password) {
    bean.data.password = password;
    sink.add(bean);
}
```

#### 清空账号输入框
与`MVC`一致，可以参考`MVC`。

#### 密码是否可见
需要改变可见状态，调用`LoginBloc`中的`changeObscure`方法，如下面代码所示
```dart
changeObscure() {
    bean.data.obscure = !bean.data.obscure;
    sink.add(bean);
}
```

#### 触发登录
需要进行网络请求，控制loading的展示和隐藏，这里需要调用`LoginBloc`中的`login`方法，当登录成功后，则跳转主页展示基本信息，不成功则toast提示，如下面代码所示
```dart
login(BuildContext context) async {
    _showLoading();

    final login =
        await LoginManager.instance.login(bean.data.name, bean.data.password);
    //授权成功
    if (login != null) {
      final user = await LoginManager.instance.getMyUserInfo();
      if (user != null) {
        NavigatorUtil.goHome(context, user);
      } else {
        ToastUtil.showToast('登录失败，请重新登录');
      }
    } else {
      ToastUtil.showToast('登录失败，请重新登录');
    }

    _hideLoading();
}
```

## Redux
`Redux`是网页开发着广泛使用的设计模式，比如用在React.js中。关于它的介绍可以参考文章[Flutter主题切换之flutter redux](https://yuzopro.github.io/2019/06/01/Flutter%E4%B8%BB%E9%A2%98%E5%88%87%E6%8D%A2%E4%B9%8Bflutter-redux/)。

### 架构视图
![](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/flutter_architecture_redux.png)

### 程序入口
`main.dart`是程序的入口，完成登录界面的启动，相关代码如下所示
```dart
void main() {
  final store = new Store<AppState>(
    appReducer,
    initialState: AppState.initial(),
    middleware: [
      LoginMiddleware(),
    ],
  );

  runApp(
    ReduxApp(
      store: store,
    ),
  );
}

class ReduxApp extends StatelessWidget {
  final Store<AppState> store;

  const ReduxApp({Key key, this.store}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return StoreProvider<AppState>(
      store: store,
      child: StoreConnector<AppState, _ViewModel>(
        converter: _ViewModel.fromStore,
        builder: (context, vm) {
          return MaterialApp(
            theme: ThemeData(
              primaryColor: Colors.black,
            ),
            home: LoginPage(),
          );
        },
      ),
    );
  }
}

class _ViewModel {
  _ViewModel();

  static _ViewModel fromStore(Store<AppState> store) {
    return _ViewModel();
  }
}
```
在程序的入口处，对`Store`进行了初始化工作，完成了对`reducer`、`state`、`middleware`初始化工作。

#### 定义action
完成登录需要有请求登录、请求加载中、请求错误、请求成功等几个状态，如下面代码所示
```dart
class FetchLoginAction {
  final BuildContext context;
  final String userName;
  final String password;

  FetchLoginAction(this.context, this.userName, this.password);
}

class ReceivedLoginAction {
  ReceivedLoginAction(
    this.token,
    this.userBean,
  );

  final String token;
  final UserBean userBean;
}

class RequestingLoginAction {}

class ErrorLoadingLoginAction {}
```

#### 初始化state
目前只有一个登录功能，所以只需一个登录的state，如下面代码所示
```dart
class AppState {
  final LoginState loginState;

  AppState({
    this.loginState,
  });

  factory AppState.initial() => AppState(
        loginState: LoginState.initial(),
      );
}

class LoginState {
  final bool isLoading;
  final String token;

  LoginState({this.isLoading, this.token});

  factory LoginState.initial() {
    return LoginState(
      isLoading: false,
      token: '',
    );
  }

  LoginState copyWith({bool isLoading, String token}) {
    return LoginState(
      isLoading: isLoading ?? this.isLoading,
      token: token ?? this.token,
    );
  }
}
```

#### 初始化reducer
目前只有一个登录功能，所以只需一个登录的reducer，如下面代码所示
```dart
AppState appReducer(AppState state, action) {
  return AppState(
    loginState: loginReducer(state.loginState, action),
  );
}

final loginReducer = combineReducers<LoginState>([
  TypedReducer<LoginState, RequestingLoginAction>(_requestingLogin),
  TypedReducer<LoginState, ReceivedLoginAction>(_receivedLogin),
  TypedReducer<LoginState, ErrorLoadingLoginAction>(_errorLoadingLogin),
]);
```

#### 初始化middleware
登录的中间件暂时只对其做个简单的初始化过程，如下面代码所示
```dart
class LoginMiddleware extends MiddlewareClass<AppState> {
  static final String TAG = "LoginMiddleware";

  @override
  void call(Store store, action, NextDispatcher next) {
  }
}
```

### 登录流程
Redux能够允许我们分离业务逻辑，不用考虑什么时候需要刷新屏幕，一切交给StoreConnector可以完成，所以登录页面继承`StatelessWidget`即可。如下面代码所示
```dart
class LoginPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return StoreConnector<AppState, LoginPageViewModel>(
      distinct: true,
      converter: (store) => LoginPageViewModel.fromStore(store, context),
      builder: (_, viewModel) => LoginPageContent(viewModel),
    );
  }
}
```
`LoginPageViewModel`只负责登录状态以及登录行为，如下面代码所示
```dart
typedef OnLogin = void Function(String name, String password);

class LoginPageViewModel {
  static final String TAG = "LoginPageViewModel";

  final OnLogin onLogin;
  final bool isLoading;

  LoginPageViewModel({this.onLogin, this.isLoading});

  static LoginPageViewModel fromStore(
      Store<AppState> store, BuildContext context) {
    return LoginPageViewModel(
      isLoading: store.state.loginState.isLoading,
      onLogin: (String name, String password) {
        LogUtil.v('name is $name, password is $password', tag: TAG);
        store.dispatch(FetchLoginAction(context, name, password));
      },
    );
  }
}
```

#### 文本监听
与`MVC`一致，可以参考`MVC`。

#### 清空账号输入框
与`MVC`一致，可以参考`MVC`。

#### 密码是否可见
与`MVC`一致，可以参考`MVC`。

#### 触发登录
在进行登录时，我们只需调用`LoginPageViewModel`内的`onLogin`方法，该方法会通过`store`分发出`FetchLoginAction`，此时中间件`LoginMiddleware`会收到该行为，并对其进行处理。
```dart
@override
void call(Store store, action, NextDispatcher next) {
    next(action);
    if (action is FetchLoginAction) {
      _doLogin(next, action.context, action.userName, action.password);
    }
}
```
上面对收到的行为继续分发出去，如果是自己感兴趣的行为，就自己进行操作，处理`FetchLoginAction`行为，相关代码如下所示
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
          next(ReceivedLoginAction(token, userBean));
          NavigatorUtil.goHome(context, userBean);
        } else {
          ToastUtil.showToast('登录失败请重新登录');
          LoginManager.instance.setToken(null, true);
        }
      } else {
        ToastUtil.showToast('登录失败请重新登录');
        next(ErrorLoadingLoginAction());
      }
    } catch (e) {
      LogUtil.v(e, tag: TAG);
      ToastUtil.showToast('登录失败请重新登录');
      next(ErrorLoadingLoginAction());
    }
}
```
在进行登录的过程中，最初会发出正在请求的行为`RequestingLoginAction`，当登录成功后也会发出行为`ReceivedLoginAction`，登录失败后发出行为`ErrorLoadingLoginAction`，而这些发出的行为都会被`reducer`收到，并对数据进行处理，在通知UI刷新。`loginReducer`相关处理逻辑如下面代码所示
```dart
LoginState _requestingLogin(LoginState state, action) {
  LogUtil.v('_requestingLogin', tag: TAG);
  return state.copyWith(isLoading: true);
}

LoginState _receivedLogin(LoginState state, action) {
  LogUtil.v('_receivedLogin', tag: TAG);
  return state.copyWith(isLoading: false, token: action.token);
}

LoginState _errorLoadingLogin(LoginState state, action) {
  LogUtil.v('_errorLoadingLogin', tag: TAG);
  return state.copyWith(isLoading: false);
}
```

## 总结

### 局部状态和全局状态
上面的登录例子中，登录表单的任何验证类型，都可以考虑为局部状态，因为这些规则仅适用于这个组件，而App的其他部分不需要知道这个类型。但是从后台获取的token和用户资料，就需要考虑成全局状态，因为它影响整个app的作用域（未登录和已登陆），而且可能别的组件会依赖它。

### 选择
对比上面四种架构的好坏，最终还是的回归到状态管理上来。`MVC`、`MVP`的状态管理都是采用`setState`方式，而`BloC`和`Redux`都有自己的一套状态管理。

当项目最初不是很复杂的时候，采用`setState`方式更新数据是可以的。但是随着功能的增加，你的项目将会有几十个甚至上百个状态，`setState`出现的次数便会显著增加，每次`setState`都会重新调用build方法，这势必对于性能以及代码的可阅读性带来一定的影响。所以就放弃了`MVC`、`MVP`这两种架构。

最初对[OpenGit_Flutter](https://github.com/Yuzopro/OpenGit_Flutter)进行架构重构的时候，用到的是`Redux`，到涉及到多个页面复用时，例如项目中的`项目页`，每涉及到一个复用页面就需要在`state`内定义一些列的变量，这是个很痛苦的过程，所以后面就放弃了用`Redux`，但是`Redux`在保存全局状态有优势，例如主题、语言、用户资料等。后面又尝试了`BloC`，该架构在多页面复用时，就没存在`Redux`的问题。

所以最后我采用的架构是`Bloc+Redux`，用`BloC`控制局部状态，用`Redux`控制全局状态。同时大家也可以参考文章[[译]让我来帮你理解和选择Flutter状态管理方案](https://juejin.im/post/5bac54c45188255c681589d3#heading-5)

## 项目地址

 - 架构Sample：[flutter_architecture](https://github.com/Yuzopro/flutter_architecture)
 - OpenGit_Flutter项目：[OpenGit_Flutter](https://github.com/Yuzopro/OpenGit_Flutter/tree/master)
 - OpenGit_Flutter项目BloC尝试：[OpenGit_Flutter](https://github.com/Yuzopro/OpenGit_Flutter/tree/dev_bloc)
 - OpenGit_Flutter项目Redux尝试：[OpenGit_Flutter](https://github.com/Yuzopro/OpenGit_Flutter/tree/dev_redux)



