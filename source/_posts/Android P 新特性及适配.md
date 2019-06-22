---
title: Android P 新特性及适配
date: 2018-06-12 23:27:05
tags: Android
---
### 简介
Android P 中出了很多新特性，其中以规范刘海屏的设置最为重要。另外还增加了其他功能Api，由于官方文档以及一些文章都是将官方文档翻译写的文章，并无对新特性进行实践。本文将其中一些特性实践了一下并以此作为记录，有以下几点内容：
- 刘海屏幕
- 相机变化
- Image相关Api
- 电源管理
- 指纹认证
- 非SDK接口限制
- 放大镜工具
- 隐私权变更

### 刘海屏幕
在Android P 还未出来刘海屏幕接口时，国内系统已经先一步开启了刘海屏幕的机型，也公布了各家定制系统的刘海屏幕Api。因此刘海屏幕需要分两部分来讲，一部分是原生的刘海屏适配，一部分是国内的刘海屏适配。在Android P 中，原生增加了DisplayCutout类来表示刘海屏，我们可以根据该类获取是否存在刘海屏和刘海屏的宽高。而在Android P以下我们需要判断系统，根据系统的不同去使用不同的方法来获取是否存在刘海屏和刘海屏的宽高。

#### 国内系统
这里给出oppo、vivo、huawei、xiaomi的是否拥有刘海屏和获取高度的方法，具体的文档可以在各个厂商的官方开发文档可以查阅。

- oppo

``` java
public static boolean hasNotch(Activity activity) {
    activity.getPackageManager().hasSystemFeature("com.oppo.feature.screen.heteromorphism");
}
```
``` java
public static int getNotchHeight(Activity activity) {
    String value = "";
    Class<?> cls;
    try {
        cls = Class.forName("android.os.SystemProperties");
        Method hideMethod = cls.getMethod("get", String.class);
        Object object = cls.newInstance();
        value = (String) hideMethod.invoke(object, "ro.oppo.screen.heteromorphism");
    } catch (ClassNotFoundException e) {
        LogUtils.e(TAG, "get error() " + e);
    } catch (NoSuchMethodException e) {
        LogUtils.e(TAG, "get error() " + e);
    } catch (InstantiationException e) {
    LogUtils.e(TAG, "get error() " + e);
    } catch (IllegalAccessException e) {
        LogUtils.e(TAG, "get error() " + e);
    } catch (IllegalArgumentException e) {
        LogUtils.e(TAG, "get error() " + e);
    } catch (InvocationTargetException e) {
        LogUtils.e(TAG, "get error() " + e);
    }catch (Exception e) {
        LogUtils.e(TAG, "get error() " + e);
    }

    if (TextUtils.isEmpty(value)) {
        return 0;
    }

    String[] item = value.split(":");
    if (item.length >= 2) {
        String[] lt = item[0].split(",");
        String[] rb = item[1].split(",");
        if (lt.length >= 2 && rb.length >= 2) {
            int topY = StringUtils.StringToInteger(lt[1]);
            int bottomY = StringUtils.StringToInteger(rb[1]);
            return  bottomY - topY;
        }
    }
}
```

- vivo

``` java
public static boolean hasNotch(Activity activity) {
    boolean ret = false;
    try {
        ClassLoader cl = activity.getClassLoader();
        Class FtFeature = cl.loadClass("android.util.FtFeature");
        Method get = FtFeature.getMethod("isFeatureSupport", int.class);
        ret = (boolean) get.invoke(FtFeature, NOTCH_IN_SCREEN_VOIO_MARK);
    } catch (ClassNotFoundException e) {
        LogUtils.e(TAG, "hasNotchInScreen ClassNotFoundException");
    } catch (NoSuchMethodException e) {
        LogUtils.e(TAG, "hasNotchInScreen NoSuchMethodException");
    } catch (Exception e) {
        LogUtils.e(TAG, "hasNotchInScreen Exception" + e);
    }
    return ret;
}
```

``` java
public static int getNotchHeight(Activity activity) {
    return DeviceUtils.dp2px(activity, 27);
}
```

- huawei

``` java
public static boolean hasNotch(Activity activity) {
    boolean ret = false;
    try {
        ClassLoader cl = activity.getClassLoader();
        Class HwNotchSizeUtil = cl.loadClass("com.huawei.android.util.HwNotchSizeUtil");
        Method get = HwNotchSizeUtil.getMethod("hasNotchInScreen");
        ret = (boolean) get.invoke(HwNotchSizeUtil);
    } catch (ClassNotFoundException e) {
        LogUtils.e(TAG, "hasNotchInScreen ClassNotFoundException");
    } catch (NoSuchMethodException e) {
        LogUtils.e(TAG, "hasNotchInScreen NoSuchMethodException");
    } catch (Exception e) {
        LogUtils.e(TAG, "hasNotchInScreen Exception" + e);
    }
    return ret;
}
```

``` java
public static int getNotchHeight(Activity activity) {
    int[] ret = new int[]{0, 0};
    try {
        ClassLoader cl = activity.getClassLoader();
        Class HwNotchSizeUtil = cl.loadClass("com.huawei.android.util.HwNotchSizeUtil");
        Method get = HwNotchSizeUtil.getMethod("getNotchSize");
        ret = (int[]) get.invoke(HwNotchSizeUtil);
    } catch (ClassNotFoundException e) {
        LogUtils.e(TAG, "getNotchSize ClassNotFoundException");
    } catch (NoSuchMethodException e) {
        LogUtils.e(TAG, "getNotchSize NoSuchMethodException");
    } catch (Exception e) {
        LogUtils.e(TAG, "getNotchSize Exception" + e);
    }
    return ret[1];
}
```

- xiaomi

``` java
public static boolean hasNotch(Activity activity) {
    String value;
    Class<?> cls;
    try {
        cls = Class.forName("android.os.SystemProperties");
        Method hideMethod = cls.getMethod("get", String.class);
        Object object = cls.newInstance();
        value = (String) hideMethod.invoke(object, "ro.miui.notch");
        return StringUtils.StringToInteger(value) == 1;
    } catch (ClassNotFoundException e) {
        LogUtils.e(TAG, "get error() " + e);
    } catch (NoSuchMethodException e) {
        LogUtils.e(TAG, "get error() " + e);
    } catch (InstantiationException e) {
        LogUtils.e(TAG, "get error() " + e);
    } catch (IllegalAccessException e) {
        LogUtils.e(TAG, "get error() " + e);
    } catch (IllegalArgumentException e) {
        LogUtils.e(TAG, "get error() " + e);
    } catch (InvocationTargetException e) {
        LogUtils.e(TAG, "get error() " + e);
    } catch (Exception e) {
        LogUtils.e(TAG, "get error() " + e);
    }
    return false;
}
```

``` java
public static int getNotchHeight(Activity activity) {
    int resourceId = activity.getResources().getIdentifier("notch_height", "dimen", "android");
    if (resourceId > 0) {
        return activity.getResources().getDimensionPixelSize(resourceId);
    }
}
```
#### Android P
``` java
public static boolean hasNotch(Activity activity) {
    View decorView = activity.getWindow().getDecorView();
    if (decorView != null && android.os.Build.VERSION.SDK_INT >= 28) {
        WindowInsets windowInsets = decorView.getRootWindowInsets();
        if (windowInsets != null && windowInsets.getDisplayCutout() != null) {
            DisplayCutout cutout = windowInsets.getDisplayCutout();
            return cutout.getSafeInsetTop();
        }
    }
}
```
``` java
public static int getNotchHeight(Activity activity) {
    View decorView = activity.getWindow().getDecorView();
    if (decorView != null && android.os.Build.VERSION.SDK_INT >= 28) {
        WindowInsets windowInsets = decorView.getRootWindowInsets();
        if (windowInsets != null && windowInsets.getDisplayCutout() != null) {
            return true;
        }
    }
}
```

### 相机变化
在相机方面，Android P新增了以下功能，由于用到很少而且无法实践，所以这个权当一个知识点。

- 支持外置闪光灯
- 支持外部USB/UVC相机
- 支持同时获取多个相机的数据流

### Image相关Api
新增ImageDecoder，在Android P中它可以替代BitmapFactory和BitmapFactory.Options相关类。ImageDecoder支持PNG、JPEG、WEBP、
GIF、HEIF等格式的图片解码。它相比BitmapFactory有以下几个优势：
支持精确缩放，支持单步解码至硬件存储器，支持解码后处理，以及动画
图像解码。

简单来讲，就是原生支持GIF的展示，在操作图像方面会比以前更加容易。在新SDK中一共提供了5种解析方法:

- ImageDecoder.createSource(File file)
- ImageDecoder.createSource(ByteBuffer buffer)
- ImageDecoder.createSource(Resources res, int resId)
- ImageDecoder.createSource(ContentResolver cr, Uri uri)
- ImageDecoder.createSource(AssetManager assets, String fileName)

以上方法均返回Source 对象，通过ImageDecoder.decodeDrawable解析成Drawable
ImageDecoder.decodeBitmap解析成Bitmap。我们可以将解析后的图片在ImageView上展示代码如下：
``` java
ImageDecoder.Source source = ImageDecoder.createSource(getResources(), R.drawable.bitmap_example);
Drawable drawable = ImageDecoder.decodeDrawable(source, onHeaderDecodedListener);
imageIV.setImageDrawable(drawable);
```

对于GIF来说，新增了一个AnimatedImageDrawable供开发者调用，我们可以使用以下代码来播放一个GIF图片：
``` java
ImageDecoder.Source source = ImageDecoder.createSource(getResources(), R.drawable.gif_example);
Drawable drawable = ImageDecoder.decodeDrawable(source);
imageIV.setImageDrawable(drawable);
if (drawable instanceof AnimatedImageDrawable) {
    ((AnimatedImageDrawable) drawable).start();
}
```
在新Api中，还提供了一个解析监听器，通过监听器我们可以对图像进行处理，获取获取图像的信息。在SmallFlag中，对图像的采样信息采样了一半。在SizeFlag中，我们可以直接设置图像的解析像素值。
```java
ImageDecoder.OnHeaderDecodedListener listener = new ImageDecoder.OnHeaderDecodedListener() {
    @Override
    public void onHeaderDecoded(ImageDecoder decoder, ImageDecoder.ImageInfo info, ImageDecoder.Source source) {
        if (isSmallFlag) {
            decoder.setTargetSampleSize(2);
        }
        if (isSizeFlag) {
            decoder.setTargetSize(500, 500);
        }
    }
};
Drawable drawable = ImageDecoder.decodeDrawable(source, listener);
```
在该监听中，通过 ImageDecoder 还可以为圆角或圆形遮罩之类的图像添加复杂的定制效果。 我们可以使用PostProcessor类，再通过decoder调用setPostProcessor()执行所需的绘图。在AnimatedImageDrawable上，回调只会被调用一次，但绘图命令将应用于每个帧。
``` java
final PostProcessor postProcessor = new PostProcessor() {
    @Override
    public int onPostProcess(@androidx.annotation.NonNull @NonNull Canvas canvas) {
        Path path = new Path();
        path.setFillType(Path.FillType.INVERSE_EVEN_ODD);
        int width = canvas.getWidth();
        int height = canvas.getHeight();
        path.addRoundRect(0, 0, width, height, 20, 20, Path.Direction.CW);
        Paint paint = new Paint();
        paint.setAntiAlias(true);
        paint.setColor(Color.TRANSPARENT);
        paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.SRC));
        canvas.drawPath(path, paint);
        return PixelFormat.TRANSLUCENT;
        return PixelFormat.TRANSLUCENT;
    }
};
ImageDecoder.OnHeaderDecodedListener listener = new ImageDecoder.OnHeaderDecodedListener() {
    @Override
    public void onHeaderDecoded(ImageDecoder decoder, ImageDecoder.ImageInfo info, ImageDecoder.Source source) {
        decoder.setPostProcessor(postProcessor);
    }
};
```

### 电源管理
下面这张图是摘自Google官方，介绍了5.0到8.0版本中电源管理的变化。
![image](http://wison.carpcai.cn/5-1.jpeg)

在Android P中则对电源进行进一步的限制。主要可以分为两个类别：应用待机分组，省电模式改进。

#### 应用待机分组
基于应用最近使用时间和使用频率，帮助系统排定应用请求资源的优先级。 根据使用模式，每个应用都会归类到五个优先级群组之一中。 系统将根据应用所属的群组限制每个应用可以访问的设备资源。系统的分组方式可以变化，每个设备制造商都可以选择使用自己的算法编写分组应用。（处于低耗电白名单的应用不适用于此限制中） 5个分组如下：
- 活跃：用户当前正在使用应用
- 工作：应用经常运行，当前未处于活跃状态
- 常用：应用会定期使用，但不是每天都必须使用
- 极少：应用不经常使用
- 从不：安装但是从未运行过的应用

处于不同分组的应用可使用的服务都会有不同的限制。我们可以使用ADB命令来设置和查看应用的分组：
设置应用分组：adb shell am set-standby-bucket packagename active|working_set|frequent|rare
查询应用分组：adb shell am get-standby-bucket [packagename]（不传递 packagename 参数，则将列出所有应用的群组）

#### 省电模式改进
从上图我们可以看出，在之前几个版本Google已经开始使用省电模式，在Android p 中省电模式得到了更进一步的限制，具体有以下几个方面：
- 系统会更积极地将应用置于应用待机模式，而不是等待应用空闲。
- 后台执行限制适用于所有应用，无论它们的目标 API 级别如何。
- 当屏幕关闭时，位置服务可能会被停用。
- 后台应用没有网络访问权限。

### 指纹认证
在之前的机型中，我们很早之前就已经用上了指纹解锁、指纹支付等相关操作。而在Android P中，Google为应用提供生物识别身份验证对话框。 该功能创建标准化的对话框外观、风格和位置，让用户更加确信，他们在使用可信的生物识别凭据检查程序进行身份验证。简单来说便是Google统一了指纹认证的对话框，提供了一套指纹相关的Api供调用，使得对话框的样式统一，更可信。
我们可以使用BiometricPrompt来代替FingerprintManager 向用户显示指纹身份验证对话框。 BiometricPrompt 依赖系统来显示身份验证对话框。 它还会改变其行为，以适应用户所选择的生物识别身份验证类型。
在使用 BiometricPrompt 之前，应该先使用 hasSystemFeature()函数以确保设备支持 FEATURE_FINGERPRINT、FEATURE_IRIS 或 FEATURE_FACE。
如果设备不支持生物识别身份验证，可以回退为使用 createConfirmDeviceCredentialIntent() 函数验证用户的 PIN 码、图案或密码。createConfirmDeviceCredentialIntent在手机没有密码或者图案密码时将返回空Intent，记得做非空判断。
用代码来说话，查询设备是否支持生物识别，但在尝试使用的时候，发现SDK包中并无提供FEATURE_IRIS与FEATURE_FACE的常量，因此我们需要自己定义。
``` java
private static final String FEATURE_IRIS = "android.hardware.iris";
private static final String FEATURE_FACE = "android.hardware.face";

boolean isSupport = getPackageManager().hasSystemFeature(FEATURE_FINGERPRINT)
                || getPackageManager().hasSystemFeature(FEATURE_IRIS)
                || getPackageManager().hasSystemFeature(FEATURE_FACE);
```

再来我们看一下Google提供的统一认证对话框的样式。
<img src="http://wison.carpcai.cn/5-2.jpeg" width="200" height="200" align=center />
接下来是生成对话框的代码，对照代码应该很清楚对应上图所设置的地方，这里就不再赘述了。
``` java
BiometricPrompt prompt = new BiometricPrompt
        .Builder(this)
        .setTitle("Title")
        .setSubtitle("SubTitle")
        .setDescription("Description")
        .setNegativeButton("取消", getMainExecutor(), 
            new DialogInterface.OnClickListener() {
                @Override
                public void onClick(DialogInterface dialog, int which) {
                    Toast.makeText(BiometricActivity.this, "点击取消了", Toast.LENGTH_SHORT).show();
                }
            })
        .build();

CancellationSignal cancelSignal = new CancellationSignal();
cancelSignal.setOnCancelListener(new CancellationSignal.OnCancelListener() {
    @Override
    public void onCancel() {
        Toast.makeText(BiometricActivity.this, "取消了", Toast.LENGTH_SHORT).show();
    }
});

BiometricPrompt.AuthenticationCallback callback = new BiometricPrompt.AuthenticationCallback() {
    @Override
    public void onAuthenticationSucceeded(BiometricPrompt.AuthenticationResult result) {
        super.onAuthenticationSucceeded(result);
        Toast.makeText(BiometricActivity.this, "成功了", Toast.LENGTH_SHORT).show();
    }

    @Override
    public void onAuthenticationError(int errorCode, CharSequence errString) {
        super.onAuthenticationError(errorCode, errString);
        Toast.makeText(BiometricActivity.this, "错误了，errorCode : " + errorCode + ";errString : " + errString, Toast.LENGTH_SHORT).show();
    }

    @Override
    public void onAuthenticationHelp(int helpCode, CharSequence helpString) {
        super.onAuthenticationHelp(helpCode, helpString);
        Toast.makeText(BiometricActivity.this, "帮助，helpCode : " + helpCode + ";helpString : " + helpString, Toast.LENGTH_SHORT).show();
    }

    @Override
    public void onAuthenticationFailed() {
        super.onAuthenticationFailed();
        Toast.makeText(BiometricActivity.this, "失败了", Toast.LENGTH_SHORT).show();
    }
};

prompt.authenticate(cancelSignal, getMainExecutor(), callback);
```

### 非SDK接口限制
Android P 引入了针对非SDK接口的使用限制，无论是直接使用还是通过反射或JNI间接使用。其中SDK的分类可以区分为以下四种：
- 白名单：SDK
- 浅灰名单：仍可以访问的非 SDK 函数/字段。
- 深灰名单：对于目标 SDK 低于 API 级别 28 的应用，允许使用深灰名单接口。对于目标 SDK 为 API 28 或更高级别的应用：行为与黑名单相同
- 黑名单：受限，无论目标 SDK 如何。 平台将表现为似乎接口并不存在。 例如，无论应用何时尝试使用接口，平台都会引发 NoSuchMethodError/NoSuchFieldException。

#### 检测使用
对于我们是否使用了限制SDK，官方提供了3种方式供我们检测。

1. 在开发中
LogCat日志，格式为Accessing hidden field|method…
弹警告Toast（DP版本）
弹对话框警告（Debuggable应用）

2. 使用veridex检测工具(Mac，Linux)
谷歌提供的一个静态检测工具，可以帮助我们检测apk中是否使用了非SDK接口。
./appcompat.sh --dex-file=文件名.apk --imprecise
下载地址：
https://android.googlesource.com/platform/prebuilts/runtime/+/master/appcompat

3. 使用StrictMode的detectNonSdkApiUsage来启动检测功能

#### 非SDK限制原理
在Android的源码中存在三个文件，分别是hiddenapi-light-greylist.txt、hiddenapi-dark-greylist.txt、hiddenapi-blacklist.txt。从文件名我们可以知道分别记载着浅灰、深灰、黑名单的SDK，我们也可以找到这三个文件从而知道官方限制的SDK分别是哪些。而在系统编译阶段会从这三个文件生成HiddenApi，HiddenApi之后会在dex编译是对其中的函数和字段进行修改，改变其access_flag字段，再生成新的dex文件。在程序运行反射的期间，系统会判断access_flag是否合法或者classloader是否为BootStrapClassLoader，从而实现进行SDK的限制。
参考文章：https://mp.weixin.qq.com/s/sktB0x5yBexkn4ORQ1YofA

#### 绕过SDK限制
1. 直接调用
举例，查阅代码，我们知道ActivityThread的currentPackageName方法属于public方法，但是我们直接调用是无法通过编译的。因此我们可以在项目中创建一个library，在其中创建一个ActivityThread类，注意包名需要跟系统的包名一样，这里我们的包名是android.app，类的代码如下。
之后需要将该library以compileOnly project的方式引入，这样该library只会在编译的时候引入，不会打到最终包里面，同时我们可以发现刚刚的编译错误已经没了。这个方法不使用反射，但是只适用public与default的方法与字段。注意：如果需要调用的隐藏API所在的类已经位于android.jar中，Provided方式不再适用，此时需要自定义android.jar，将需要的Method或Field添加到android.jar中。
``` java
public class ActivityThread {
    public static String currentPackageName() {
        return null;
    }
}
```
2. 修改access_flag
修改函数和字段对应的access_flag，去掉其隐藏属性。

3. 修改ClassLoader
只要将我们apk中定义的类的ClassLoader改为BootStrapClassLoader就可以绕过SDK限制。在art/runtime/mirror/class.h可知SetClassLoader函数可以为一个类指定ClassLoader，
在art/runtime/well_known_classes.h中ToClass函数能够为我们设置所需的ClassLoader。
通过这两个我们便可以不用hook便调用系统的限制SDK。

参考文章：https://mp.weixin.qq.com/s/4k3DBlxlSO2xNNKqjqUdaQ

### 放大镜工具
在Android P 中新增了放大镜工具，以提升文本选择方面的用户体验。由于该放大器提供了可以在文本上方拖拽的文本放大面板，所以有助于用户精准地定位光标或文本选择手柄。同时该功能还支持文本之外的放大，目前我尝试了文本与图片都是可以放大的。生成一个Magnifier对象
``` java
TextView textTV;
Magnifier textMagnifier = new Magnifier(textTV);
```
之后我们可以监听textTV的onTouchListener，在该回调中实现放大镜功能。
``` java
switch (event.getAction()) {
    case MotionEvent.ACTION_DOWN:
        textMagnifier.show(event.getX(), event.getY());
        break;
    case MotionEvent.ACTION_MOVE:
        textMagnifier.show(event.getX(), event.getY());
        break;
    case MotionEvent.ACTION_UP:
        textMagnifier.dismiss();
        break;
}
```

### 隐私权变更
Android P中对隐私进行了进一步的限制，具体表现有以下几点
#### 后台应用传感器的访问受限
后台应用被限制访问用户输入和传感器数据。 当应用在设备的后台运行，系统将对应用采取以下限制：

- 应用不能访问麦克风或摄像头。
- 使用连续报告模式的传感器（例如加速度计和陀螺仪）不会接收事件。
使用变化或一次性报告模式的传感器不会接收事件。
如果应用需要检测传感器事件，需要使用前台服务。

#### 限制访问通话记录
引入 CALL_LOG 权限组并将 READ_CALL_LOG、WRITE_CALL_LOG 和 PROCESS_OUTGOING_CALLS 权限移入该组。 在之前的 Android 版本中，这些权限位于 PHONE 权限组。
对于需要访问通话敏感信息（如读取通话记录和识别电话号码）的应用，该 CALL_LOG 权限组为用户提供了更好的控制和可见性。如果应用需要访问通话记录或者需要处理来去电，则必须向 CALL_LOG 权限组明确请求这些权限。 否则会发生 SecurityException。
#### 限制访问电话号码
在未获得 READ_CALL_LOG 权限的情况下，除了应用的用例需要的其他权限之外，应用无法读取电话号码或手机状态。
与来电和去电关联的电话号码可在手机状态广播中看到，并可通过 PhoneStateListener 类访问。 但是，如果没有 READ_CALL_LOG 权限，则 PHONE_STATE_CHANGED 广播和 PhoneStateListener 提供的电话号码字段为空。
要通过 PHONE_STATE Intent 操作读取电话号码，需要同时获取 READ_CALL_LOG 权限和 READ_PHONE_STATE 权限。
要从 onCallStateChanged() 中读取电话号码，只需要 READ_CALL_LOG 权限，不需要 READ_PHONE_STATE 权限。
#### 限制访问 Wi-Fi 位置和连接信息
应用进行 Wi-Fi 扫描的权限要求比之前的版本更严格。 
类似的限制也适用于 getConnectionInfo() 函数，该函数返回描述当前 Wi-Fi 连接的 WifiInfo 对象。如果应用具有以下权限，则只能使用该对象的函数来检索 SSID 和 BSSID 值：ACCESS_FINE_LOCATION、ACCESS_COARSE_LOCATION、
ACCESS_WIFI_STATE，检索SSID或BSSID还需要在设备上启用位置服务。


