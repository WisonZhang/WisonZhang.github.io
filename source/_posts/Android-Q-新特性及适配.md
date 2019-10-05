---
title: Android Q 新特性及适配
date: 2019-06-10 23:27:05
tags: Android
---
继Android P之后今年我们又迎来了Android Q，上次做过Android P的分享，这次顺应的也将Android Q的一些新特性整理一下，本文章是基于Android Q BETA 3来写的。文章会分两部分来介绍Android Q，分别是Android Q中的新特性和之前已有的功能在Android Q上的改动。

## Android Q新特性
在这一节中主要介绍Android Q中新推出的新特性，当然并没有全部都列出来，我从中挑取了相对来说对应用开发比较有用的来讲。分别有：

- 设置面板
- 气泡
- 暗黑主题
- 折叠屏
- 全屏手势
- WebView检测
- 温度检测
- 桌面模式

### 设置面板
Android Q 引入了“设置面板”，这是一种 让应用能够在自身环境中向用户显示设置的API。这可以避免用户转到设置更改 NFC 或移动数据等设置，以便使用此应用。
当前设置面板支持：

- 网络连接（Settings.Panel.ACTION_INTERNET_CONNECTIVITY）
- WiFi设置（Settings.Panel.ACTION_WIFI）
- NFC设置（Settings.Panel.ACTION_NFC）
- 音量设置（Settings.Panel.ACTION_VOLUME）

使用很简单，如调用网络链接的设置面板时，我们只需要使用
``` Java
Intent panelIntent = new Intent(Settings.Panel.ACTION_INTERNET_CONNECTIVITY);
startActivityForResult(panelIntent, REQUEST_CODE);
```
不同的面板只需要传入不同的值，这里展示音量设置面板，效果如下。

<img src="http://wison.carpcai.cn/6-1.png" width="330" height="589" align=center />

### 气泡
气泡是Android Q中的一个新预览功能，可让用户轻松地从设备上的任何位置进行多任务处理。它被设计为使用SYSTEM_ALERT_WINDOW的替代方案。

借助气泡，用户可以轻松地在设备上的任何位置进行多任务处理。气泡内置于“通知”系统中。它们会浮动在其他应用内容上层，并会跟随用户转到任意位置。气泡可以展开以显示应用功能和信息，并可在不使用时折叠起来。我们可以在桌面和其他应用中展示气泡，并在气泡中展示我们需要的界面和操作逻辑。比较典型的我们可以使用气泡应用在聊天中，我们先来看看气泡具体的样子。

<img src="http://wison.carpcai.cn/6-2.png" width="330" height="589" align=center />

气泡与通知关联的，因此气泡的创建也与通知紧密相关。我们先照常创建一个通知渠道。
``` Java
NotificationChannel channel = new NotificationChannel(CHANNEL_ID, CHANNEL_ID, NotificationManager.IMPORTANCE_HIGH);
```
在气泡中展示的界面其实是一个Activity，我们创建一个Activity，这里就命名为BubbleActivity，BubbleActivity在清单中的设置如下。
``` Java
<activity
    android:name=".BubbleActivity"
    android:allowEmbedded="true"
    android:documentLaunchMode="always"
    android:resizeableActivity="true"
    android:theme="@style/AppNoActionBar" />
```
之后我们就可以构建我们的Notification了，整体的代码如下。
``` Java
NotificationChannel channel = new NotificationChannel(CHANNEL_ID, CHANNEL_ID, NotificationManager.IMPORTANCE_HIGH);

Intent intent = new Intent(this, BubbleActivity.class);
intent.putExtra(BubbleActivity.BUBBLE_TYPE, notifyId);
PendingIntent bubbleIntent = PendingIntent.getActivity(this, notifyId, intent, 0);

Notification.BubbleMetadata metadata = new Notification.BubbleMetadata.Builder()
        .setDesiredHeight(300)
        .setIcon(Icon.createWithAdaptiveBitmap(BitmapFactory.decodeResource(getResources(), headRes)))
        .setIntent(bubbleIntent)
        .build();

Notification notification = new Notification.Builder(this, CHANNEL_ID)
        .setBubbleMetadata(metadata)
        .setNumber(3)
        .setSmallIcon(R.drawable.ic_launcher_foreground)
        .build();

NotificationManager notifyManager = (NotificationManager) getSystemService(NOTIFICATION_SERVICE);
if (notifyManager == null) {
    return;
}
notifyManager.createNotificationChannel(channel);
notifyManager.notify(notifyId, notification);
```

### 暗黑主题
今年Android与iOS都推出了暗黑模式，可能是潮流吧。在Android Q中，使用暗黑主题非常简单，有三种方式：

- 通过设置应用程序主题为Theme.AppCompat.DayNight
- 通过设置android:forceDarkAllowed=“true"来加入强制黑暗
- 通过调用AppCompatDelegate.setDefaultNightMode来切换主题

第三种方式是通过代码的形式来切换主题，同时也提供了我们去判断当前是什么主题的API，代码如下。

``` Java
if (AppCompatDelegate.getDefaultNightMode() == AppCompatDelegate.MODE_NIGHT_YES
                || AppCompatDelegate.getDefaultNightMode() == AppCompatDelegate.MODE_NIGHT_FOLLOW_SYSTEM) {
		// 处于暗黑模式中
}
```
需要注意的是，通过代码切换主题不会立即生效，需要在onResume之后才会生效。暗黑主题的效果图如下。

<img src="http://wison.carpcai.cn/6-3.png" width="330" height="589" align=center />

### 折叠屏
Awesome！！！Cool！！！咳咳

在Android Q上，当多个应用程序在多窗口或多显示下同时出现时，应用都处于onResume。新版本增加了新的API，应用可以监听onTopResumedActivityChanged()，在获得或失去最高恢复位置时收到回调。同时新增了新属性 android:minAspectRatio 来指示应用支持的最低屏幕比率。

这个特性没什么大多可以介绍的地方，但感觉又很重要，算是一个比较超前的特性，目前应用了折叠屏的手机比较热门的当属华为与三星，个人还是比较看好这门技术的，具体的样子不清楚的可以去搜索看看。

### 全屏手势
这个功能在Android Q未发布之前，小米在全面屏手势上已经先采用了，即从屏幕的边缘，不论左边还是右边向屏幕内滑动会返回上一级，从屏幕底部向上滑动则返回桌面，如果向上滑动过程中停住则展示多任务页。最近发现华为的手机也采用了这一套手势逻辑。不同的是Android Q还支持并木底部栏左右滑动直接切换上下一个应用，这点则是与iPhone X的手势逻辑相同了。
而当原本应用程序与滑动功能有冲突时，为了维护原本应用程序屏幕左右边缘滑动的功能，在Android Q中需要向系统指示哪些区域需要接收触摸输入。我们通过view.setSystemGestureExclusionRects()传入一个Rect来设置区域，通过该Api我们可以屏蔽系统的全屏手势而保留程序原本的滑动手势不被覆盖。代码如下，设置了屏幕左上角不被全屏手势覆盖，其他区域则依然能用全屏手势。
``` Java
List<Rect> rects = new ArrayList<>();
rects.add(new Rect(0, 0, 100, 200));
getWindow().getDecorView().setSystemGestureExclusionRects(rects);
```

### WebView检测
Android Q引入了一个新的WebViewRenderProcessClient抽象类，应用程序可以使用它来检测是否WebView已经无响应。具体步骤如下：
1. 实现其onRenderProcessResponsive()与onRenderProcessUnresponsive()方法。
2. 将WebViewRenderProcessClient设置到一个或多个WebView对象。
3. 如果WebView没有响应，系统将调用客户端的onRenderProcessUnresponsive()方法，并传递WebView和WebViewRenderProcess。（如果WebView是单进程，则WebViewRenderProcess参数为null。）应用程序可以采取适当的操作，例如向用户显示一个对话框，询问他们是否要暂停渲染过程。
4. 如果WebView仍然没有响应，系统会onRenderProcessUnresponsive()定期调用（每五秒钟不超过一次），但不采取其他操作。如果WebView再次响应，系统onRenderProcessResponsive()只调用一次。

### 温度检测
当设备过热时，可能会限制CPU/GPU的频率，这会以意想不到的方式影响应用和游戏。在Android Q中，应用和游戏可以使用Thermal的Api来监控设备上的更改，并采取措施来维持较低的功耗以恢复正常温度。应用程序可以在PowerManager中addThermalStatusListener注册一个监听器，系统通过该监听器报告持续的热状态，分别是：

- THERMAL_STATUS_NONE = 0;
- THERMAL_STATUS_MODERATE = 2;
- THERMAL_STATUS_LIGHT = 1;
- THERMAL_STATUS_SEVERE = 3;
- THERMAL_STATUS_CRITICAL = 4;
- THERMAL_STATUS_EMERGENCY = 5;
- THERMAL_STATUS_SHUTDOWN = 6;

以上的值代表着从轻度到中度到严重，关键，紧急和关机。监听代码如下：
``` Java
PowerManager manager = (PowerManager) getSystemService(POWER_SERVICE);
if (manager != null) {
    manager.addThermalStatusListener(this);
}

// 监听回调
@Override
public void onThermalStatusChanged(int i) {
    thermal.setText("当前温度值" + i);
}

```

### 桌面模式
这也是一个比较新的技术方向，我们可以理解为跟锤子的TNT一样的技术，目前华为也是支持桌面模式。拿华为比较新的手机，通过TypeC转HDMI到屏幕上，就可以看到华为的桌面模式。不得不说体验完之后不失为一个好的发展方向，当以后CPU与GPU的能力再提升时，或许我们拿一部手机就可以实现轻量化的移动办公了。在Android Q Beta版中暂未提及如何触发桌面模式，只找到了国外开发者自己写的Launcher，如图。

<img src="http://wison.carpcai.cn/6-4.png" width="552" height="343" align=center />

## Android Q 改动
在这一节中主要介绍以前已有功能在Android Q上的改动，可以看出这次Google的改动很大目的是为了保障用户的隐私，修改了很多功能都是以保护隐私为目的。主要的改动分别有：

- SYSTEM_ALERT_WINDOW
- 全屏通知
- MAC地址
- 沙盒系统
- 图片隐私位置
- 设备标识符
- 剪贴板
- 后台Activity启动限制
- 位置权限
- 64位应用支持

### SYSTEM_ALERT_WINDOW
这个改动是针对Android Go设备的，在新版本Android Go设备上运行的应用无法获得 SYSTEM_ALERT_WINDOW 权限。这是因为绘制叠加层窗口会使用过多的内存，会影响低内存设备的性能。

如果在搭载 Android P 或更低版本的设备上运行的应用已经申请了 SYSTEM_ALERT_WINDOW 权限，即使设备升级到 Android Q 也会保留此权限。不过，尚不具有此权限的应用在设备升级后便无法获得此权限了。

### 全屏通知
针对targetSDK为Q的应用，使用fullscreen intent的通知，必须在manifest文件中声明USE_FULL_SCREEN_INTENT权限，该权限是一个普通权限，系统会自动授权。如果APP在没有申请USE_FULL_SCREEN_INTENT权限的情况下，通过fullscreen intent创建一个通知，那么系统会忽略该请求，并在控制台输出以下信息：Package [pkg]: Use of fullScreenIntent requires the USE_FULL_SCREEN_INTENT permission

也就是说在应用一下通知时，需要按照上面所说在Manifest中申请权限。
``` Java
Notification notification = new Notification.Builder(this, "full_channel")
        .setContentTitle("通知标题")
        .setContentText("通知内容")
        .setSmallIcon(R.drawable.ic_launcher_foreground)
        .setFullScreenIntent(pendingIntent, true)
        .build();
```

### MAC地址
在Android Q上运行的设备默认传输随机MAC地址。获取MAC地址的方式如下：

- 获取随机MAC地址：应用可以通过调用检索分配给特定网络的随机MAC地址getRandomizedMacAddress()。
- 获取实际MAC地址：设备所有者应用程序可以通过调用来检索设备的实际硬件MAC地址getWifiMacAddress()。

以上是文档中对于MAC地址的推荐做法，但在我尝试的时候发现通过getRandomizedMacAddress()方法获取的值总是02:00:00:00:00:00，不知道在正式版发布时会不会有改动。而在使用7.0获取MAC地址的方式还依然有效。
``` Java
public static String getNMacAddress() {
    try {
        List<NetworkInterface> all = Collections.list(NetworkInterface.getNetworkInterfaces());
        for (NetworkInterface nif : all) {
            if (!nif.getName().equalsIgnoreCase("wlan0")) continue;
            byte[] macBytes = nif.getHardwareAddress();
            if (macBytes == null) {
                return "";
            }
            StringBuilder res1 = new StringBuilder();
            for (byte b : macBytes) {
                res1.append(String.format("%02X:", b));
            }
            if (res1.length() > 0) {
                res1.deleteCharAt(res1.length() - 1);
            }
            return res1.toString();
        }
    } catch (Exception e) {
        e.printStackTrace();
    }
    return "02:00:00:00:00:00";
}
```

### 沙盒系统
为了保护隐私同时为了让用户更好地控制自己的文件，并限制文件混乱情况，Android Q 更改了应用访问设备外部存储空间中文件的方式。Android Q 在外部存储设备中为每个应用提供了一个隔离存储沙盒。任何其他应用都无法直接访问不属于自己应用的沙盒文件。由于文件是各个应用的私有文件，因此不再需要任何权限即可在外部存储设备中访问和保存自己的文件。

在Android Q之前的生态对于文件存储是没有限制，因此很多应用都会将自己的应用放在自己想要的路劲下，且在应用卸载后还能保留信息以便用户第二次安装时能够获取上一次应用的遗留信息。这次对文件的规范可以说影响了很多应用，如果不及时适配可能会导致应用不可用甚至奔溃。我们可以先来看一张官方发布的图片，这是官方推荐的存储文件的方式：

![image](http://wison.carpcai.cn/6-5.jpeg)

通过Context.getExternalFilesDir可以获取到属于 App 自身的文件路径，通常是~/Android/data/<package-name>/**/。在该目录中读写文件均不需要申请权限，当 App 被卸载时，该文件夹及内容也会全部删除。
当应用获取不属于沙盒范围的文件时，即以前根据路径获取文件的方式将会得到FileNotFoundException。

MediaStore这个类是android系统提供的一个多媒体数据库，可以从MediaStore中获取音频，视频和图像。这个数据库存放在/data/data/com.android.providers.media/databases当中，里面有两个数据库：internal.db和external.db，internal.db存放的是系统分区的文件信息，开发者是没法通过接口获得其中的信息的，而external.db存放的则是我们用户能看到的存储区的文件信息。这里我贴上通过MediaStore获取图片的代码：
``` Java
String[] projection = new String[]{MediaStore.Images.Media.DISPLAY_NAME
            , MediaStore.Images.Media.DATA
            , MediaStore.Images.Media._ID};
List<ImageInfo> imgList = new ArrayList<>();

Cursor cursor = getApplicationContext().getContentResolver().query(MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
        projection, null, null, MediaStore.Images.Media.DATE_ADDED);
File file;
if (cursor != null) {
    if (cursor.moveToLast()) {
        imgList.clear();
        do {
            ImageInfo info = new ImageInfo();
            info.fileName = cursor.getString(cursor.getColumnIndex(projection[0]));
            info.filePath = cursor.getString(cursor.getColumnIndex(projection[1]));
            file = new File(info.filePath);
            if (file.exists()) {
                imgList.add(info);
            }
        } while (cursor.moveToPrevious());
        handler.post(new Runnable() {
            @Override
            public void run() {
                adapter.setData(imgList);
            }
        });
    }
    cursor.close();
}

class ImageInfo {
    String fileName;
    String filePath;
}

```

从Android 4.4 引入了存储访问框架 (SAF)。SAF 让用户能够在其所有首选文档存储提供程序中方便地浏览并打开文档、图像以及其他文件。 用户可以通过易用的标准 UI，以统一方式在所有应用和提供程序中浏览文件和访问最近使用的文件。当然不排除国产系统会阉割掉 Documents 应用导致无法请求访问。SAF
使用方式非常简单，具体效果大家可以跑代码试一下，这里贴出通过SAF打开文件以及存储文件的代码。
``` Java
// 打开图片文件
Intent intent = new Intent(Intent.ACTION_OPEN_DOCUMENT
intent.addCategory(Intent.CATEGORY_OPENABLE);
intent.setType("image/*");
startActivityForResult(intent, FOLD_REQUEST_CODE);

// 存储文件
Intent intent = new Intent(Intent.ACTION_CREATE_DOCUMENT);
intent.addCategory(Intent.CATEGORY_OPENABLE);
intent.setType("text/*");
intent.putExtra(Intent.EXTRA_TITLE, "androidQTest.txt");
startActivityForResult(intent, CREATE_DOCUMENT_CODE);
```

可以明白，我们需要做的事有：

- 迁移之前已有文件
- 修改文件存储方式
- 修改通过遍历获取文件列表行为
- 文件共享改用FileProvider

可能Google也明白这个改动一下发出很多应用来不及适配，可能会导致大面积的应用不可使用，因此Google也推迟了该特性的推迟，当目标SDK Android P或更低时。或者目标SDK为Android Q，且在清单中设置”allowExternalStorageSandbox”为false(默认值为true)，都可以退出该沙盒系统的限制。而在明年的新系统中，所有应用程序都会强制要求沙盒限制，与目标SDK级别无关。

### 图片隐私位置
一些照片在其 Exif 元数据中包含位置信息，以便用户查看照片的拍摄地点。由于此位置信息很敏感，因此默认情况下 Android Q 会对该信息进行遮盖。
如果应用需要访问照片的位置信息，需要完成以下步骤：

- 将新ACCESS_MEDIA_LOCATION权限添加到应用的清单中。
- 从MediaStore对象，调用setRequireOriginal()，传递照片的URI。当未添加权限而获取位置信息时，会有UnsupportedOperationException的错误。

``` Java
Cursor cursor = getApplicationContext().getContentResolver().query(MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
        null, null, null, MediaStore.Images.Media.DATE_ADDED);
if (cursor == null) {
    return;
}
Uri uri = Uri.withAppendedPath(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, pathSegment);
uri = MediaStore.setRequireOriginal(uri);
InputStream stream;
try {
    stream = getContentResolver().openInputStream(uri);
    if (stream != null) {
        ExifInterface exifInterface = new ExifInterface(stream);
        double[] returnedLatLong = exifInterface.getLatLong();
        if (returnedLatLong == null) {
            Log.i(TAG, "经纬度为空");
        } else {
            Log.i(TAG, "经纬度: " + returnedLatLong[0] + ";" + returnedLatLong[1]);
        }
        stream.close();
    }
} catch (IOException e) {
    e.printStackTrace();
}
cursor.close();
```

### 设备标识符
从Android Q开始，应用必须具有READ_PRIVILEGED_PHONE_STATE特权权限才能访问设备的不可重置标识符，包括IMEI和序列号。如果应用程序没有权限，并且尝试询问有关标识符的信息，则响应会因目标SDK版本而异：

- 如果应用针对Android Q，则会出现SecurityException。
- 如果应用针对Android P或更低版本，且具有READ_PHONE_STATE权限，则该方法会返回null或占位符。否则，会出现SecurityException。

在国内的App中我们经常会使用IMEI或者IMSI号来标志设备的唯一性，现在已经无法拿到，而READ_PRIVILEGED_PHONE_STATE这个权限需要电信类的应用才能获取得到，因此如何去更换一个新的唯一标识将会成为Android Q正式版出来之前一个重要的课题。

官方推荐替代方案：

- Android ID
- Instance ID
- 广告ID
- GUID

其中Android ID在恢复出厂设置时会重置且不能保证唯一，Instance ID需要谷歌服务，广告ID可以被广告服务商重置而且只推荐用在广告服务中，GUID则是自定义ID
，需要自己定义算法且只能存在文件中，当卸载应用时也会变得不可靠。

### 剪贴板
在Android Q中，除了以下标明的应用程序外，其他程序都不能访问剪贴板数据，不管应用的目标SDK多少都会生效。

- 默认输入法编辑器
- 前台应用程序

也就说当程序使用以下代码监听剪贴板的数据时，在Android Q中是无法获取的，像有些浏览器会监听剪贴板的内容，然后根据剪贴板内容去弹出通知这种功能将会失效。
``` Java
ClipboardManager manager = (ClipboardManager) getSystemService(Context.CLIPBOARD_SERVICE);
ClipboardManager.OnPrimaryClipChangedListener changedListener = new ClipboardManager.OnPrimaryClipChangedListener() {
    @Override
    public void onPrimaryClipChanged() {
        if (manager.getPrimaryClip() != null && manager.getPrimaryClip().getItemCount() > 0) {
            CharSequence content = manager.getPrimaryClip().getItemAt(0).getText();
            Log.d("Clipboard", "复制、剪切的内容为：" + content);
        }
    }
};
manager.addPrimaryClipChangedListener(changedListener);
```

### 后台Activity启动限制
Android Q禁止了处于后台的应用启动Activity，此行为针对Android Q设备上的所有程序，当应用尝试从后台启动活动时，会在Logcat收到警告并且会收到Toast提醒。（Toast提醒不会出现在Android Q的正式版中）在开发的时候我们可以取消这个限制通过下面两种方式：

- 在开发者选项中，开启允许后台活动启动
- 在终端中运行adb shell settings put global background_activity_starts_enabled 1

而正常运行时，在Android Q上运行的应用只有在满足以下一个或多个条件时才能启动后台Activity：

- 该应用具有可见窗口，例如在前台运行的 Activity。
- 该应用程序在前台任务的后台堆栈中有一个活动。
- 该应用程序具有系统服务，如：AccessibilityService， AutofillService， CallRedirectionService， HostApduService， InCallService， TileService， VoiceInteractionService和 VrListenerService。
- 接收到前台运行的另一个应用发送属于该应用的 PendingIntent。
- 接收到系统发送属于该应用的 PendingIntent，例如点按通知。
- 应用接收系统发送的广播，如ACTION_NEW_OUTGOING_CALL 和 SECRET_CODE_ACTION。
- 该应用程序通过CompanionDeviceManager API 与配套硬件设备相关联 。此API允许应用程序启动活动以响应用户在配对设备上执行的操作。
- 该应用是在设备所有者模式下运行 的设备策略控制器。

可以想到的，我们日常使用桌面微信登录时会手机会自动弹出页面让我们确认登录，像这种透传然后弹出页面的功能是不能再使用了。而对于日常普通开发来讲，最受影响的便是应用的闪屏页，闪屏页一般都会倒数几秒展示广告，当还未倒计时完返回桌面，再回到应用且已经倒数完后，这时本应该倒数完跳转页面的逻辑便不会生效，页面会继续停留在闪屏页。

### 位置权限
在iOS中我们经常可以看到仅在应用运行期间使用定位权限的对话框，Android Q为了让用户更好地控制应用对位置信息的访问权限，引入了新的位置权限 ACCESS_BACKGROUND_LOCATION。与现有的 ACCESS_FINE_LOCATION 和 ACCESS_COARSE_LOCATION 权限不同，新权限仅会影响应用在后台运行时对位置信息的访问权。

如果应用在 Android Q 上运行但目标平台是 Android P或更低版本：
- 如果应用已经在清单申请了ACCESS_FINE_LOCATION或ACCESS_COARSE_LOCATION，则系统会在安装期间自动添加ACCESS_BACKGROUND_LOCATION。
- 如果应用请求了ACCESS_FINE_LOCATION或ACCESS_COARSE_LOCATION，系统会自动将ACCESS_BACKGROUND_LOCATION添加到请求中。

### 64位应用支持
从2019年8月1日开始，在 Google Play 上发布的应用必须支持64位架构。
- 对于ARM 架构，32 位对应armeabi-v7a，64 位对应arm64-v8a。
- 对于x86 架构，32 位对应x86，64位对应x86_64。

