---
title: Android网易云信视频踩坑
date: 2018-01-17 09:29:56
tags: Android
---

最近在项目中集成了网易云信的视频通话功能，期间遇到了几个问题，然而网上关于网易云信的问题实在有些少，官方社区也是一片冷落的迹象，很多问的问题没有人解答，因此在此记录一下自己最近踩过的坑。（基于SDK 3.8）

### 悬浮窗

该功能是将通话界面缩小成悬浮窗在界面右上角，具体效果参考微信。
在网易云信中音视频通话是在AVChatActivity中，首先要先实现将Activity放到后台的功能，这里需要在AndroidManifest中将AVChatActivity的启动模式设为SingleInstance，之后在相应的点击事件中调用movetoback(true)即可。这里有个需要注意的地方，在onPause和onResume中将

```java
avChatUI.pauseVideo();
avChatUI.resumeVideo();
```

这两个方法去掉，否则Activity退到后台会暂停视频聊天。
之后我们新建一个service用来显示悬浮窗，这里贴出service的完整代码

```java
public class FloatChattingWindowService extends Service implements View.OnClickListener {

    private WindowManager windowManager;
    private View floatView;
    private LinearLayout previewLayout;
    private LinearLayout audioLayout;
    private Chronometer timeTV;
    private String account;

    @Override
    public void onCreate() {
        super.onCreate();
        initWindow();
        initEvent();
    }

    private void initWindow() {
        windowManager = (WindowManager) getSystemService(Context.WINDOW_SERVICE);
        WindowManager.LayoutParams params = getParams();
        floatView = LayoutInflater.from(getApplicationContext())
          .inflate(R.layout.layout_floating_chatting, null);
        previewLayout = (LinearLayout) floatView.findViewById(R.id.previewLayout);
        audioLayout = (LinearLayout) floatView.findViewById(R.id.audioLayout);
        timeTV = (Chronometer) floatView.findViewById(R.id.timeTV);
        windowManager.addView(floatView, params);
    }

    private WindowManager.LayoutParams getParams() {
        WindowManager.LayoutParams params = new WindowManager.LayoutParams();
        params.type = WindowManager.LayoutParams.TYPE_TOAST;
        params.flags = WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE;
        params.width = WindowManager.LayoutParams.WRAP_CONTENT;
        params.height = WindowManager.LayoutParams.WRAP_CONTENT;
        params.gravity = Gravity.END | Gravity.TOP;
        params.x = ConvertUtils.dp2px(15);
        params.y = ConvertUtils.dp2px(15);
        return params;
    }

    private void initEvent() {
        previewLayout.setOnClickListener(this);
        audioLayout.setOnClickListener(this);
    }

    @Override
    public void onClick(View v) {
        Intent intent = new Intent();
        intent.setClass(this, AVChatActivity.class);
        intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        startActivity(intent);
    }

    /**
     * 初始化悬浮窗视频
     * @param account 对方账号
     */
    public void initSurface(String account) {
        this.account = account;
        AVChatVideoRender render = new AVChatVideoRender(getApplicationContext());
        AVChatManager.getInstance().setupRemoteVideoRender(account, render, false, AVChatVideoScalingType.SCALE_ASPECT_FILL);
        if (render.getParent() != null) {
            ((ViewGroup) render.getParent()).removeView(render);
        }
        previewLayout.addView(render);
        render.setZOrderMediaOverlay(false);
    }

    /**
     * 展示语音通话
     * @param base 当前通话时间
     */
    public void initAudioLayout(long base) {
        previewLayout.setVisibility(View.GONE);
        audioLayout.setVisibility(View.VISIBLE);
        timeTV.setBase(base);
        timeTV.start();
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        if (floatView != null) {
            windowManager.removeView(floatView);
        }
        timeTV.stop();
        if (!StringUtils.isSpace(account)) {
            AVChatManager.getInstance().setupRemoteVideoRender(account, null, false, 0);
        }
    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return new MyBinder();
    }

    public class MyBinder extends Binder {
        public FloatChattingWindowService getService() {
            return FloatChattingWindowService.this;
        }
    }
  
}
```

布局

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content">

    <FrameLayout
        android:layout_width="64dp"
        android:layout_height="98dp"
        android:background="@color/white">

        <LinearLayout
            android:id="@+id/previewLayout"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:background="@color/transparent"
            android:orientation="vertical" />

        <LinearLayout
            android:id="@+id/audioLayout"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:background="@color/transparent"
            android:gravity="center"
            android:orientation="vertical"
            android:visibility="gone">

            <ImageView
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:background="@drawable/content_icon_tel"/>

            <Chronometer
                android:id="@+id/timeTV"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginTop="5dp"
                android:textColor="@color/text_item_title"
                android:textSize="12sp"
                android:visibility="visible" />

        </LinearLayout>
      
    </FrameLayout>
  
</LinearLayout>
```

接下来只要在AVChatActivity创建一个ServiceConnection，并在其中调用service的方法

```java
private ServiceConnection chattingConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            FloatChattingWindowService.MyBinder binder = 
              (FloatChattingWindowService.MyBinder) service;
            FloatChattingWindowService chattingService = binder.getService();
            if (chattingService != null) {
                if (state == AVChatType.AUDIO.getValue()) {
                    chattingService.initAudioLayout(avChatUI.getTimeBase());
                }
                if (state == AVChatType.VIDEO.getValue()) {
                    chattingService.initSurface(avChatUI.getAccount());
                }
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
        }
    };
```

在onPause与onRestart进行绑定与解绑的操作

```java
	@Override
    protected void onPause() {
        super.onPause();
        Intent intent = new Intent(this, FloatChattingWindowService.class);
        bindService(intent, chattingConnection, Context.BIND_AUTO_CREATE);
    }
    
    @Override
    protected void onRestart() {
        super.onRestart();
        unbindService(chattingConnection);
        if (state == AVChatType.VIDEO.getValue()) {
            if (isCallEstablished) {
                new Handler().postDelayed(new Runnable() {
                    @Override
                    public void run() {
                        avChatUI.initAllSurfaceView(avChatUI.getAccount());
                    }
                }, 800);
            } else {
                avChatUI.initLargeSurfaceView(avChatUI.getAccount());
            }
        }
    }
```

至此，就完成了语音视频缩小到后台并且有小窗口的功能啦。



### 美颜

网易云信提供了基础的美颜、磨皮、水印的滤镜功能，开发者也可以接入第三方进行美颜或者特效的优化，这里提供一个第三方的使用案例

> Faceunity
> https://github.com/Faceunity/FUNimDemoDroid

而要使用网易自带的滤镜，得先在项目中引用video_effect.jar和libvideoeffect.so，这里不得不吐槽一下官方文档实在是没什么用，而且在官方音视频的demo里面是没有相关美颜的使用，使用在互动直播的demo里面。由于在项目中我只使用了美颜和磨皮，因此需要水印的功能的话可以参考互动直播的demo代码。
首先按照文档，相关的滤镜应该在onVideoFrameFilter的回调中使用，然而打了Log后发现该回调并没有被调用，因此往上面方法setParameters中寻找

```java
AVChatManager.getInstance().setParameters(avChatParameters);
```

查看设置avChatParameters的方法updateAVChatOptionalConfig()，发现并没有跟滤镜相关的设置，于是再往AVChatParameters类中查找，发现有一个KEY为AVChatParameters.KEY_VIDEO_FRAME_FILTER，设置

```java
avChatParameters.setBoolean(AVChatParameters.KEY_VIDEO_FRAME_FILTER, true);
```

再打Log，发现回调正常调用了，接下来在该方法中初始化VideoEffect并对视频进行滤镜处理，完整代码如下

```java
@Override
public boolean onVideoFrameFilter(AVChatVideoFrame frame) {
	if (frame == null || (Build.VERSION.SDK_INT < 18)) {
    	return true;
    }

    if (videoEffect == null) {
    	mVideoEffectHandler = new Handler();
        videoEffect = VideoEffectFactory.getVCloudEffect();
        videoEffect.init(getApplicationContext(), true, false);
        videoEffect.setBeautyLevel(5);
        videoEffect.setFilterLevel(0.5f);
        videoEffect.setFilterType(VideoEffect.FilterType.nature);
        videoEffect.addWaterMark(null, 0, 0);
        videoEffect.closeDynamicWaterMark(true);
    }

    if (videoEffect == null) {
    	return true;
    }

    VideoEffect.YUVData[] result;
    VideoEffect.DataFormat format = frame.format == 
      VideoFrame.ImageFormat.I420 ? VideoEffect.DataFormat.YUV420 : VideoEffect.DataFormat.NV21;
    byte[] intermediate = videoEffect.filterBufferToRGBA(format, frame.data.array(), 
                                                         frame.width, frame.height);
    result = videoEffect.TOYUV420(intermediate, VideoEffect.DataFormat.RGBA, 
                                  frame.width, frame.height,
    frame.rotation, 90, frame.width, frame.height, false, true);

    System.arraycopy(result[0].data, 0, frame.data.array(), 0, result[0].data.length);
    frame.width = result[0].width;
  	frame.height = result[0].height;
    frame.dataLen = result[0].data.length;
    frame.rotation = 0;
    frame.format = VideoFrame.ImageFormat.I420;
    return true;
}
```

最后在onDestroy中销毁VideoEffect

```java
if (videoEffect != null) {
	mVideoEffectHandler.post(new Runnable() {
        @Override
        public void run() {
            videoEffect.unInit();
            videoEffect = null;
    	}
    });
}
```

运行项目，美颜成功了~