---
title: Android启动模式分析
date: 2017-12-20 08:33:53
tags: Android
---
我们知道，Android有四种启动模式，分别是stander、singleTop、singleTask、singleInstance。
设置Activity的启动模式很简单，只要在AndroidManifest里面设置launchMode就可以。
四种模式的出现是为了解决各种应用场景，下面我们来分析一下各种应用场景，以及其内部任务栈发生了什么样的变化。
首先我们来了解一下什么是栈，什么是任务栈。栈是一种后进先出的数据结构，有两种操作，push和pop，你可以把它想象成是在盒子里装衣服，拿衣服时最上面那件是最先拿出来的，最底下那件肯定是最后才拿出来的。而在Android里的Activity就是以这样的栈结构来进行管理的，我们称之为任务栈。我们可以通过`adb shell sysdump`命令来查看当前设备的任务栈（需要配置环境变量）。输入我们可以看到：
``` Java
TaskRecord{7f67d30 #10439 A=com.wison.demo U=0 sz=2}
Run #3: ActivityRecord{
813695e u0 com.wison.demo/.activity.MainActivity t10439}
```
这里可以知道当前栈栈名为com.wison.demo，该栈有一个Activity，名字是MainActivity。

四种模式都跟任务栈有关联，我们一个个来讲。

**stander**
经典模式，也是默认的一个启动模式，在Android中我们用普通的方式启动一个Activity，就是这种模式。在这种模式下，每次启动一个Activity，都会将创建一个新的实例并放入启动这个Activity的任务栈中。
``` Java
TaskRecord{7f67d30 #10439 A=com.wison.demo U=0 sz=2}
Run #5: ActivityRecord
{c513ekd u0 com.wison.demo/.activity.AActivity t10439}
Run #4: ActivityRecord
{d2164bd u0 com.wison.demo/.activity.AActivity t10439}
Run #3: ActivityRecord
{813695e u0 com.wison.demo/.activity.MainActivity t10439}
```
可以看到，MainActivity启动了AActivity并放入它所在的任务栈当中，而AActivity在启动一个AActivity会重新创建并放入该栈中。
而当我们尝试用Application去启动Activity时会发现报出以下错误
![这里写图片描述](http://img.blog.csdn.net/20170319170631356?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWRyb2d5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
如刚才所说，每次启动一个Activity，都会将创建一个新的实例并放入启动这个Activity的任务栈中。而非Activity类型的Context没有任务栈一说，因此会报错。解决方法是启动的时候为其加上一个FLAG_ACTIVITY_NEW_TASK的标志，该标志会为该启动的Activity创建一个新的任务栈并将其放入其中。

**singleTop**
栈顶复用模式，当Activity设置这个模式后，每次启动这个Activity系统都会判断该Activity是否在栈顶存在，如果存在则不创建，如果不在栈顶则创建一个新的实例。
假设当前任务栈有ABC三个Activity，如果C的启动模式为singleTop，则当C启动C时，由于在栈顶C已经存在，这时候不会创建一个新的C实例，而是会复用栈顶的C。即任务栈还是ABC，这时候会调用C的onNewIntent，打Log我们可以看到

``` Java
com.wison.demo I/tag: onPause
com.wison.demo I/tag: onNewIntent
com.wison.demo I/tag: onResume
```

**singleTask**
栈内复用模式，在该模式下，只要该Activity在任何一个栈内存在，就不会创建新的实例，同样的，会调用它的onNewIntent。如果这时该Activity不在栈顶，则将其之上的Activity全部弹出栈外，并将该Activity置于栈顶。
这里分几种情况分析。
一、
任务栈内有AB两个Activity，C的启动模式为singleTask，则启动C后的栈内情况如下
``` Java
TaskRecord{7f67d30 #10439 A=com.wison.demo U=0 sz=2}
Run #5: ActivityRecord
{c513ekd u0 com.wison.demo/.activity.CActivity t10439}
Run #4: ActivityRecord
{d2164bd u0 com.wison.demo/.activity.BActivity t10439}
Run #3: ActivityRecord
{813695e u0 com.wison.demo/.activity.AActivity  t10439}
```
当C再启动C时，由于C存在于栈顶，因此C会调用
``` Java
com.wison.demo I/tag: onPause
com.wison.demo I/tag: onNewIntent
com.wison.demo I/tag: onResume
```

二、
任务栈内有ABCD，C的启动模式为singleTask，这时候D再启动C，任务栈的情况为
``` Java
TaskRecord{7f67d30 #10439 A=com.wison.demo U=0 sz=2}
Run #5: ActivityRecord
{c513ekd u0 com.wison.demo/.activity.CActivity t10439}
Run #4: ActivityRecord
{d2164bd u0 com.wison.demo/.activity.BActivity t10439}
Run #3: ActivityRecord
{813695e u0 com.wison.demo/.activity.AActivity  t10439}
```
从任务栈我们可以看到D已经不在任务栈，我们再看看Log的情况

``` Java
com.wison.demo I/DActivity: onPause
com.wison.demo I/CActivity: onNewIntent
com.wison.demo I/CActivity: onStart
com.wison.demo I/CActivity: onResume
com.wison.demo I/DActivity: onStop
com.wison.demo I/DActivity: onDestroy
```
可以看出，D已经执行onDestroy了。这也印证了如果这时该Activity不在栈顶，则将其之上的Activity全部弹出栈外，并将该Activity置于栈顶。

三、
这里我们介绍一个新的概念，TaskAffinity，该属性能够Activity所需的任务栈名，默认为包名。同样的我们可以在AndroidManifest中进行设置。
这里我们设置CActivity的TaskAffinity为com.wison.test，接着我们按顺序启动ABC，再来看看任务栈的情况
``` Java
 TaskRecord{b8815da #10599 A=com.wison.test U=0 sz=1}
 Run #4: ActivityRecord{
 988e526 u0 com.wison.demo/.activity.CActivity t10599}
 
 TaskRecord{35ea0b #10597 A=com.wison.demo U=0 sz=2}
 Run #3: ActivityRecord{
 7efda4e u0 com.wison.demo/.activity.BActivity t10597}
 Run #2: ActivityRecord{
 9dbb6a1 u0 com.wison.demo/.activity.AActivity t10597}
```
可以看到，CActivity是运行在com.wison.test这个任务栈里的，这时候C是在前台的，所以com.wison.test叫做前台任务栈，而com.wison.demo则叫做后台任务栈。

**singleInstance**
这个模式可以称为singleTask的加强版，我们可以称它为单例模式，在这个模式中，如果在任何一个任务栈中没有该实例，则单独创建一个任务栈并放入该实例，如果在某个任务栈中有该实例，则将该实例任务栈调到前台任务栈来。
设置CActivity的LaunchMode为singleInstance，再来启动ABC，我们看看任务栈的情况。
``` Java
TaskRecord{1ed9db9 #10604 A=com.wison.demo U=0 sz=1}
Run #4: ActivityRecord{
4c9334b u0 com.wison.demo/.activity.CActivity t10604}

TaskRecord{72d87ac #10603 A=com.wison.demo U=0 sz=2}
Run #3: ActivityRecord{
f28fcd3 u0 com.wison.demo/.activity.BActivity t10603}
Run #2: ActivityRecord{
3fc4983 u0 com.wison.demo/.activity.AActivity t10603}
```
可以看到虽然没有指定TaskAffinity，但是CActivity还是单独在一个任务栈里。看看任务栈的情况。
``` Java
TaskRecord{1ed9db9 #10604 A=com.wison.demo U=0 sz=1}
Run #4: ActivityRecord{
4c9334b u0 com.wison.demo/.activity.CActivity t10604}

TaskRecord{72d87ac #10603 A=com.wison.demo U=0 sz=2}
Run #3: ActivityRecord{
f28fcd3 u0 com.wison.demo/.activity.BActivity t10603}
Run #2: ActivityRecord{
3fc4983 u0 com.wison.demo/.activity.AActivity t10603}
```
可以看到虽然没有指定TaskAffinity，但是CActivity还是单独在一个任务栈里。我们接着尝试在C中启动D，看看会是什么情况。
``` Java
TaskRecord{891d010 #10794 A=com.wison.demo U=0 sz=2}
Run #3: ActivityRecord{
cf7d657 u0 com.wison.demo/.activity.DActivity t10794}

TaskRecord{367ec1a #10795 A=com.wison.demo U=0 sz=1}
Run #2: ActivityRecord{
12cbb50 u0 com.wison.demo/.activity.CActivity t10795}

TaskRecord{891d010 #10794 A=com.wison.demo U=0 sz=2}
Run #1: ActivityRecord{
3738523 u0 com.wison.demo/.activity.BActivity t10794}
Run #0: ActivityRecord{
6bf361b u0 com.wison.demo/.activity.AActivity t10794}
```
我们可以看到，ABD在一个任务栈，而C则还是单独在一个任务栈里，而当我们从D再次启动C时，会出现跟singleTask一样的情况。
``` Java
com.wison.demo I/DActivity: onPause
com.wison.demo I/CActivity: onNewIntent
com.wison.demo I/CActivity: onStart
com.wison.demo I/CActivity: onResume
com.wison.demo I/DActivity: onStop
com.wison.demo I/DActivity: onDestroy
```
而C的任务栈当然也被放置到前台。
