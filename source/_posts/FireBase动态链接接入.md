---
layout: firebase
title: FireBase动态链接接入
date: 2019-02-12 23:16:26
tags: Android
---
最近接入了Firebase的动态链接用于用户分享，在国内很常见的邀请机制并没有很好的解决方案，通常都需要用户主动输入邀请人的邀请码来完成邀请条件。而在国外谷歌推出了Firebase的动态链接，当打开链接时App能够接收到相关的信息回调。这个很好的解决邀请同时不需要用户手动填写邀请码的问题。
接入之前需要在Firebase后台创建相应项目，iOS需要设置包ID，App store ID，团队ID，Android需要设置报名与SHA-1，SHA-256签名。

#### 创建网域
要接入链接首先需要在Firebase上的DynamicLink栏目中选择开通，然后注册自己的网域。
![](http://pm8jyj0w6.bkt.clouddn.com/%233-01.png)
网域会在之后创建链接展示，如你的网域是example，创建后的短链接就会是https://example.page.link/skjk

#### 相关概念
这里创建完后可以选择在控制台进行创建链接，也可以通过谷歌提供的Api自己创建链接。在邀请机制中我们使用的是自己创建链接，但在控制台这里我们可以看到相应的选项。在SDk中也会有相应的Api提供配置，创建前需要了解相应的概念。
![](http://pm8jyj0w6.bkt.clouddn.com/%233-02.png)

* 动态链接
动态链接分为长链接与短链接，顾名思义，长连接就是长的动态链接，短链接就是短的动态链接。长连接会把深度链接的内容拼接到网域中，例如深度链接为www.xxx.com，生成的长动态链接则为https://example.page.link?www.xxx.com ，如果深度链接很长的话，生成的长连接也会很长。短链接则是将深度链接的内容进行加密，形式为https://example.page.link/skjk ，短链接在长度与保密性上都是更好的选择，但是在低版本的动态链接库上是不支持短链接的生成。使用时务必用最新版的动态链接库。

* 深度链接
深度链接是隐藏在动态链接下的我们所设置的指向地址，在配置动态链接的时候我们可以配置是否点击动态跳转到深度链接，方便落地页的跳转，如果没有配置直接跳转到深度链接，且在没有Google play或者App store的情况下，也会直接跳转到深度链接。

* iOS链接行为
可设置应用包名，设置App Store ID，设置最低应用版本（当本地的版本号小于动态链接中的最小版本号时，会自动跳转到应用市场，APi才能设置），设置跳转App Store还是链接（Firebase后台设置）。

* Android链接行为
可设置应用包名，设置最低应用版本（当本地的版本号小于动态链接中的最小版本号时，会自动跳转到应用市场），设置跳转App Store还是链接。

* 设置社交属性
可以设置社交分享标题，社交分享图片（图片的尺寸应至少为 300x200 像素，且小于 300 KB。），社交分享描述。设置完之后生成的链接在社交平台会显示相应的UI展示。

#### 创建链接
创建链接可以在Firebase后台创建，也可以使用SDK的Api创建链接，其中区别是Api生成的链接是不能统计点击量，而后台生成的链接可以看到点击次数、安装次数、打开次数等数据。这里给出Android生成短链接的代码。
```Java
FirebaseDynamicLinks.getInstance()
                .createDynamicLink()
                .setDomainUriPrefix("https://example.page.link")
                .setLink(Uri.parse("深度链接"))
                .setAndroidParameters(DynamicLink.AndroidParameters.Builder("packageName")
                        .setMinimumVersion("MinVersion")
                        .build())
                .setIosParameters(DynamicLink.IosParameters.Builder("packageName")
                        .setAppStoreId("appleStoreId")
                        .setMinimumVersion("MinVersion")
                        .build())
                .setSocialMetaTagParameters(DynamicLink.SocialMetaTagParameters.Builder()
                        .setTitle("分享标题")
                        .setDescription("分享描述")
                        .setImageUrl(Uri.parse("分享图片"))
                        .build())
                .buildShortDynamicLink(ShortDynamicLink.Suffix.SHORT)
                .addOnCompleteListener{ it ->
                    if (it.isSuccessful && it.result != null) {
                        it.result!!.shortLink.toString()
                    }
                }
```

#### 遇到的问题
在接入过程中有遇到一些问题，项目中之前所用的Firebase都是比较旧的版本，如果用新版本的动态链接库就需要将其他库同时升到最新版本，但是上线后发现Firebase引起的奔溃出现了很多，而如果用旧版本的动态链接库不会出现奔溃，但是生成的链接长度非常长，在使用上与体验上都不是很好。
最后排查问题发现应该是Google play service的版本不一致导致的奔溃。而Firebase在应用启动时会自动初始化，解决办法就是取消Firebase的自动初始化，在应用启动的时候判断谷歌服务是否可用，当可用时再对Firebase进行初始化。
首先需要去除项目build.gradle中com.google.gms:google-services的依赖。在MainActivity的onCreate()与onResume()中进行服务是否可用的判断，当判断可用且未初始化时对Firebase进行初始化，从而完成了可以生成短链接，且解决了新版本奔溃的问题，下面是检查版本库是否可用的代码。
```Java
fun checkServiceAvailable(context: Context): Boolean {
        val status = GoogleApiAvailability.getInstance().isGooglePlayServicesAvailable(context)
        return status == ConnectionResult.SUCCESS
    }
```

