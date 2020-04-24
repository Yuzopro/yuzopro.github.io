---
layout:     post
title:      "Android打开通知栏并回到主页的几种方式"
subtitle:   "当应用处于后台时，默认情况下，从通知栏启动一个Activity，按返回键会回到Launcher主屏幕。类似于微信、QQ等点击通知栏，返回回到主页又是如何实现的？"
date:       2019-04-25 17:31:00
author:     "Yuzo"
header-img: "img/post-bg-android.jpg"
catalog: true
tags:
    - 移动端
    - Android
---

## 用PendingIntent.getActivity创建通知栏

在`MainActivity`中增加点击事件，用来启动`NotifyService`和延迟2秒销毁`MainActivity`，如下面代码所示

``` java
Intent intent = new Intent(MainActivity.this, NotifyService.class);
startService(intent);

tvTips.postDelayed(new Runnable() {
    @Override
    public void run() {
        finish();
    }
}, 2000L);
```

`NotifyService`类继承`IntentService`，并在`onHandleIntent()`方法类处理展示通知栏的逻辑，如下面代码所示

``` java
private void showNotification() {
    Notification notification;
    NotificationManager manager = (NotificationManager) getSystemService(Context.NOTIFICATION_SERVICE);
    //pendingIntent生成规则
    Intent notifyIntent = new Intent();
    notifyIntent.setClass(this, NotifyActivity.class);
    PendingIntent pendingIntent = PendingIntent.getActivity(this, 0, 
        notifyIntent, PendingIntent.FLAG_UPDATE_CURRENT);

    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        NotificationChannel channel = new NotificationChannel("0", "notify",
            NotificationManager.IMPORTANCE_DEFAULT);
        manager.createNotificationChannel(channel);
        Notification.Builder builder = new Notification.Builder(this, "0")
                .setAutoCancel(true)
                .setContentTitle(getString(R.string.app_name))
                .setContentText("xxx")
                .setOnlyAlertOnce(true)
                .setSmallIcon(R.mipmap.ic_launcher)
                .setContentIntent(pendingIntent);
        notification = builder.build();
    } else {
        NotificationCompat.Builder builder = new NotificationCompat.Builder(this);
        builder.setSmallIcon(R.mipmap.ic_launcher)
                .setContentText("xxx")
                .setAutoCancel(true)
                .setWhen(System.currentTimeMillis())
                .setOnlyAlertOnce(true)
                .setContentTitle(getString(R.string.app_name))
                .setContentIntent(pendingIntent);
        notification = builder.build();
    }
    manager.notify(0, notification);
}
```

运行代码，点击`启动通知栏`按钮,此时会创建一个通知栏，并且2秒后，主页自动关闭。然后在点击`通知栏`，进入到`通知栏页面`，点击返回按钮时，发下APP没有回到主页面，而是回到了Launcher主页面。如下面截图所示

![enter image description here](https://user-gold-cdn.xitu.io/2019/4/30/16a6d2908264057b?w=360&h=640&f=gif&s=462601)

所以用`PendingIntent.getActivity`方式打开通知栏，就会出现上面所描述的问题。为了解决这问题，提供了一下几种方式。

## 用PendingIntent.getActivities创建通知栏

处理逻辑基本上跟上面一致，只需替换`pendingIntent生成规则`那部分代码，需替换的代码如下面所示

``` java
Intent notifyIntent = new Intent();
Intent mainIntent = new Intent();
notifyIntent.setClass(this, NotifyActivity.class);
mainIntent.setClass(this, MainActivity.class);
//需要注意这里的顺序
Intent[] intents = new Intent[]{mainIntent, notifyIntent};
PendingIntent pendingIntent = PendingIntent.
        getActivities(this, 0, intents, PendingIntent.FLAG_UPDATE_CURRENT);
```

运行代码，如下面截图所示

![enter image description here](https://user-gold-cdn.xitu.io/2019/4/30/16a6d29082b69e5c?w=360&h=640&f=gif&s=397744)

此方法适用于`MainActivity`和`NotifyActivity`在同一个moudle的情况。如果不在同一个moudle又该如何处理呢？接着往下看。

## 用TaskStackBuilder创建通知栏

替换`pendingIntent生成规则`那部分代码，需替换的代码如下面所示

``` java
Intent notifyIntent = new Intent();
notifyIntent.setClass(this, NotifyActivity.class);

TaskStackBuilder stackBuilder = TaskStackBuilder.create(this);
stackBuilder.addParentStack(NotifyActivity.class);
stackBuilder.addNextIntent(notifyIntent);

PendingIntent pendingIntent = stackBuilder.getPendingIntent(0, PendingIntent.FLAG_UPDATE_CURRENT);
```

除了替换`pendingIntent生成规则`之外，还需要修改`AndroidManifest.xml`内的代码，为`NotifyActivity`指定`parentActivityName`属性，如下面代码所示

```xml
<activity android:name=".NotifyActivity"
    android:parentActivityName=".MainActivity"/>
```

该属性是在Android 4.1（API level 16）引入的，此处的名称必须与为相应`<activity>`元素的`android:name`属性指定的类名称一致，以确定当用户按下返回按钮时应该启动哪一个`Activity`。

运行代码，效果如图2所示

此方法可以适用于`Activity`在不同moudle的情况。

但是，当主页`MainActivity`（这里用A代表，方便后面描述）的`launchMode`设置为`singleTask`时，当主页`A`存在时，并且还打开了其他页面'OtherActivity'（B），目前Activity的栈的顺序是`A、B`。当打开用`PendingIntent.getActivities`和`TaskStackBuilder`两种方式创建的通知栏，页面跳转到`NotifyActivity`（C），并且一直按返回键，退栈的顺序是`C、A、Launcher`，`B`却没在栈内了，见`图3`。具体原因是，当打开通知栏是，栈的顺序是`A、B、A`，由于`A`的`launchMode`是`singleTask`，此时会删除`B`,当整个通知栏操作全部完成时，Activity的栈的顺序是`A、C`，所以会出现上面描述的现象。如果要满足退栈顺序是`C、B、A、Launcher`该怎么实现？

![enter image description here](https://user-gold-cdn.xitu.io/2019/4/30/16a6d290823bfeba?w=360&h=640&f=gif&s=249354)

## 用PendingIntent.getActivity创建通知栏,本地维护Activity栈

 1. 首先需要创建一个Activity管理类`ActivityManager`，来维护Activity栈，如下面代码所示

``` java
public class ActivityManager {
    private static final byte[] sLock = new byte[0];

    private final Stack<Activity> mActivityStack = new Stack<>();

    private static ActivityManager sInstance;

    public static ActivityManager getInstance() {
        if (sInstance == null) {
            synchronized (sLock) {
                if (sInstance == null) {
                    sInstance = new ActivityManager();
                }
            }
        }
        return sInstance;
    }

    private ActivityManager() {
    }

    /**
     *  activity入栈
     */
    public void addActivity(Activity activity) {
        mActivityStack.add(activity);
    }

    /**
     *  activity出栈
     */
    public void removeActivity(Activity activity) {
        mActivityStack.remove(activity);
    }

    /**
     *  当栈的个数为1的时候，判断cls是否在栈内
     */
    public boolean currentActivity(Class<?> cls) {
        if (mActivityStack.size() != 1) {
            return true;
        }
        for (Activity activity : mActivityStack) {
            if (activity.getClass().equals(cls)) {
                return true;
            }
        }
        return false;
    }
}
```

 2. 其次创建一个`Activity`的基类`BaseActivity`，所有`Activity`页面需要继承这个基类，并且分别在`onCreate`和`onDestroy`方法中分别实现`Activity`的入栈和出栈操作，并且重写返回事件，如下面代码所示
 
 ``` java
 public abstract class BaseActivity extends AppCompatActivity {

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ActivityManager.getInstance().addActivity(this);
    }

    @Override
    public void onBackPressed() {
        super.onBackPressed();
        if (!ActivityManager.getInstance().currentActivity(MainActivity.class)) {
            Intent intent = new Intent(BaseActivity.this, MainActivity.class);
            startActivity(intent);
        }
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        ActivityManager.getInstance().removeActivity(this);
    }
}
 ```
运行代码，如下面截图所示

![enter image description here](https://user-gold-cdn.xitu.io/2019/4/30/16a6d29083543c54?w=360&h=640&f=gif&s=391265)