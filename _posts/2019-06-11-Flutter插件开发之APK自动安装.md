---
layout:     post
title:      "Flutter插件开发之APK自动安装"
subtitle:   "实现Flutter Plugin开发"
date:       2019-06-10 18:48:00
author:     "Yuzo"
header-img: "img/post-bg-flutter-plugin.jpg"
catalog: true
tags:
    - 移动端
    - Android
    - Flutter
---

# Flutter插件开发之APK自动安装
> 本文适用于Android开发人员

## 什么是Flutter Plugin
Flutter Plugin是一种特殊的包，包含一个用Dart编写的API定义，结合Android和iOS的平台特定实现，从而达到二者兼容。
* 应用的Flutter部分通过平台通道（platform channel）将消息发送到其应用程序的所在的宿主（iOS或Android）
* 宿主监听的平台通道，并接收该消息。然后它会调用特定于该平台的API（使用原生编程语言） - 并将响应发送回客户端，即应用程序的Flutter部分
使用平台通道在客户端（Flutter UI）和宿主（平台）之间传递消息，如下图所示
![enter image description here](https://flutterchina.club/images/PlatformChannels.png)
## 创建Flutter App
相关代码见[运行第一个Flutter App](https://yuzopro.github.io/2019/04/11/%E8%BF%90%E8%A1%8C%E7%AC%AC%E4%B8%80%E4%B8%AAFlutter-App/)
## 创建Flutter Plugin
右键工程->New->Module，如下图所示
![enter image description here](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/flutter_plugin_1.png)
选择Flutter Plugin,点击Next，如下图所示
![enter image description here](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/flutter_plugin_2.png)
输入工程名（Project name），点击Next，如下图所示
![enter image description here](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/flutter_plugin_3.png)
输入包名（Package name），点击Finish，入下图所示
![enter image description here](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/flutter_plugin_4.png)
到此Flutter plugin创建完成。
## 引入插件
在工程目录下找到`pubspec.yaml`文件，在`dev_dependencies`添加如下依赖，如下图所示
![enter image description here](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/flutter_plugin_5.png)
相关代码如下
```dart
dev_dependencies:
  flutter_test:
    sdk: flutter
  install_apk_plugin:
    path: install_apk_plugin
```
## 获取版本号demo
打开插件lib下的dart文件，会有平台自动生成的代码，具体是实现获取APP版本号，如下面代码所示

```dart
class InstallApkPlugin {
  static const MethodChannel _channel =
      const MethodChannel('install_apk_plugin');

  static Future<String> get platformVersion async {
    final String version = await _channel.invokeMethod('getPlatformVersion');
    return version;
  }

}
```

java部分的代码如下面所示

```java
public class InstallApkPlugin implements MethodCallHandler {
    private static final String TAG = "InstallApkPlugin";

    private final Registrar registrar;

    /**
     * Plugin registration.
     */
    public static void registerWith(Registrar registrar) {
        final MethodChannel channel = new MethodChannel(registrar.messenger(), "install_apk_plugin");
        channel.setMethodCallHandler(new InstallApkPlugin(registrar));
    }

    private InstallApkPlugin(Registrar registrar) {
        this.registrar = registrar;
    }

    @Override
    public void onMethodCall(MethodCall call, Result result) {
        if (call.method.equals("getPlatformVersion")) {
            result.success("Android " + android.os.Build.VERSION.RELEASE);
        } else {
            result.notImplemented();
        }
    }

}
```
## 实现自动安装APK
实现自动安装APK，需要从Flutter应用层传入一个APK安装包的地址到host层，dart代码如下所示：
```dart
class InstallApkPlugin {
  static const MethodChannel _channel =
      const MethodChannel('install_apk_plugin');

  static Future<String> get platformVersion async {
    final String version = await _channel.invokeMethod('getPlatformVersion');
    return version;
  }

  static Future<bool> installApk(String path) async {
    final bool isSuccess = await _channel.invokeMethod('installApk', path);
    return isSuccess;
  }
}
```
java部分的代码如下所示
```java
public class InstallApkPlugin implements MethodCallHandler {
    private static final String TAG = "InstallApkPlugin";

    private final Registrar registrar;

    /**
     * Plugin registration.
     */
    public static void registerWith(Registrar registrar) {
        final MethodChannel channel = new MethodChannel(registrar.messenger(), "install_apk_plugin");
        channel.setMethodCallHandler(new InstallApkPlugin(registrar));
    }

    private InstallApkPlugin(Registrar registrar) {
        this.registrar = registrar;
    }

    @Override
    public void onMethodCall(MethodCall call, Result result) {
        if (call.method.equals("getPlatformVersion")) {
            result.success("Android " + android.os.Build.VERSION.RELEASE);
        } else if (call.method.equals("installApk")) {
            final String path = (String) call.arguments;
            Log.d(TAG, "installApk path is " + path);
        } else {
            result.notImplemented();
        }
    }
}
```
到此，host层就能获取到APK安装包的路径了，后面只需实现Android安装APK的代码逻辑即可，在日志下面添加如下代码
```java
    File file = new File(path);
    installApk(file, registrar.context());
```
`installApk`代码实现如下所示
```java
    private void installApk(File apkFile, Context context) {
        Intent installApkIntent = new Intent();
        installApkIntent.setAction(Intent.ACTION_VIEW);
        installApkIntent.addCategory(Intent.CATEGORY_DEFAULT);
        installApkIntent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);

        Uri apkUri = null;
        if (Build.VERSION.SDK_INT > Build.VERSION_CODES.M) {
            apkUri = FileProvider.getUriForFile(context, context.getPackageName() + ".fileprovider", apkFile);
            installApkIntent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
        } else {
            apkUri = Uri.fromFile(apkFile);
        }
        installApkIntent.setDataAndType(apkUri, "application/vnd.android.package-archive");

        if (context.getPackageManager().queryIntentActivities(installApkIntent, 0).size() > 0) {
            context.startActivity(installApkIntent);
        }
    }
```
除此之外，还需修改`AndroidManifest.xml`内的代码，如下面代码所示
```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    package="com.yuzo.install_apk_plugin">

    <uses-permission android:name="android.permission.REQUEST_INSTALL_PACKAGES" />

    <application>
        <!--provider start-->
        <provider
            android:name="androidx.core.content.FileProvider"
            tools:replace="android:authorities"
            android:authorities="com.yuzo.opengit.fileprovider"
            android:exported="false"
            android:grantUriPermissions="true">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                tools:replace="android:resource"
                android:resource="@xml/file_path" />
        </provider>
        <!--provider end-->
    </application>
</manifest>
```
`file_path.xml`放在res->xml文件夹下面，如下面代码所示
```xml
<?xml version="1.0" encoding="utf-8"?>
<paths xmlns:tools="http://schemas.android.com/tools"
    tools:ignore="ResourceName">

    <root-path
        name="root_path"
        path="." />

</paths>
```
运行代码如下图所示
![enter image description here](https://raw.githubusercontent.com/Yuzopro/image/master/flutter/flutter_plugin_6.gif)