from : https://github.com/fyhertz/spydroid-ipcamera

###使用步骤 局域网用手机实现视频监控
```
1.下载运行测试 apk :https://fir.im/qnhb
2.设置里把 HTTP server 和 RTSP server 都打开
3.下载vlc (播放rtsp流) ，填入地址，地址为手机首页显示的局域网地址
4.右击 播放 就可以啦
一个问题：由于手机摄像投角度的问题，VLC 直接播放的是没有旋转处理的画面，即摄像头原始数据
```

```
1.广告(增加收入？) ：https://www.google.com/admob/
2.sdk 23以上不再支持org.apache.http，非要用高版本sdk，则 spydroid/build.gradle 添加
     android {
         useLibrary 'org.apache.http.legacy'
     }
3.电池状态监听  net.majorkernelpanic.spydroid.SpydroidApplication.mBatteryInfoReceiver
registerReceiver(mBatteryInfoReceiver, new IntentFilter(Intent.ACTION_BATTERY_CHANGED));
4.指定跳到系统桌面
    Intent setIntent = new Intent(Intent.ACTION_MAIN);//指定跳到系统桌面
    setIntent.addCategory(Intent.CATEGORY_HOME);
    setIntent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
    //setIntent.setFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP); //清除上一步缓存
    startActivity(setIntent);
5.保持屏幕常亮 新姿势
    需要电源管理的权限 <uses-permission android:name="android.permission.WAKE_LOCK"/>
    PowerManager pm = (PowerManager) getSystemService(POWER_SERVICE);
    wakeLock = pm.newWakeLock(PowerManager.SCREEN_BRIGHT_WAKE_LOCK
            | PowerManager.ON_AFTER_RELEASE, "WAKE_LOCK");
    wakeLock.acquire();
    // onStop()
    // A WakeLock should only be released when isHeld() is true !
    if (mWakeLock.isHeld()) mWakeLock.release();
    附：关于int flags 各种锁的类型对CPU 、屏幕、键盘的影响：
      PARTIAL_WAKE_LOCK:        保持CPU 运转，屏幕和键盘灯有可能是关闭的。
      SCREEN_DIM_WAKE_LOCK：   保持CPU 运转，允许保持屏幕显示但有可能是灰的，允许关闭键盘灯
      SCREEN_BRIGHT_WAKE_LOCK：保持CPU 运转，允许保持屏幕高亮显示，允许关闭键盘灯
      FULL_WAKE_LOCK：          保持CPU 运转，保持屏幕高亮显示，键盘灯也保持亮度
      ACQUIRE_CAUSES_WAKEUP：  正常唤醒锁实际上并不打开照明。相反，一旦打开他们会一直仍然保持(例如来世user的activity)。
      当获得wakelock，这个标志会使屏幕或/和键盘立即打开。一个典型的使用就是可以立即看到那些对用户重要的通知。
      ON_AFTER_RELEASE：        设置了这个标志，当wakelock释放时用户activity计时器会被重置，
      导致照明持续一段时间。如果你在wacklock条件中循环，这个可以用来减少闪烁
6. Notification & PendingIntent
    Intent notificationIntent = new Intent(this, SpydroidActivity.class);
    PendingIntent pendingIntent = PendingIntent.getActivity(this, 0, notificationIntent, PendingIntent.FLAG_CANCEL_CURRENT);

    NotificationCompat.Builder builder = new NotificationCompat.Builder(this);
    Notification notification = builder.setContentIntent(pendingIntent)
            .setWhen(System.currentTimeMillis())
            .setTicker(getText(R.string.notification_title))
            .setSmallIcon(R.drawable.icon)
            .setContentTitle(getText(R.string.notification_title))
            .setContentText(getText(R.string.notification_content)).build();
    notification.flags |= Notification.FLAG_ONGOING_EVENT;
    ((NotificationManager)getSystemService(Context.NOTIFICATION_SERVICE)).notify(0,notification);
7. PreferenceActivity 设置
    以 Notification 的打开关闭为例：
    布局 ： xml/preferences.xml:91  android:key="notification_enabled"
    改变的监听 ：net/majorkernelpanic/spydroid/SpydroidApplication.java:150   if (key.equals("notification_enabled"))
8.ViewPager 的 title  android.support.v4.view.PagerTitleStrip
    layout use in layout/spydroid.xml:27
    FragmentPagerAdapter only needs @Override  public CharSequence getPageTitle(int position)
9.bindService()  切记 需要 unbindService(conn);
    bindService(new Intent(xxx.this,Service.class),conn,flag)-->Service:onCreate()-->Service:onBind()-->conn:onServiceConnected()
    如果多次调用 bindService ，onBind()方法只会执行一次
    Google API：https://developer.android.com/guide/components/bound-services.html
10.Intent.FLAG_ACTIVITY_NO_HISTORY   net/majorkernelpanic/spydroid/ui/AboutFragment.java:62
    用这个FLAG启动的Activity，一旦退出，它不会存在于栈中，比方说！原来是A,B,C这个时候再C中以这个FLAG启动D的，D再启动E，这个时候栈中情况为A,B,C,E。
    以web 打开网页或者应用商店
    Intent intent = new Intent(Intent.ACTION_VIEW,Uri.parse("https://code.google.com/p/spydroid-ipcamera/"));
    // Intent intent = new Intent(Intent.ACTION_VIEW, Uri.parse("market://details?id="+appPackageName));
    intent.addFlags(Intent.FLAG_ACTIVITY_NO_HISTORY | Intent.FLAG_ACTIVITY_CLEAR_WHEN_TASK_RESET);
    startActivity(intent);
11.程序运行时，进入net.majorkernelpanic.spydroid.ui.SpydroidActivity，该activity 运行时候，开启http server ，rtsp server
12.rtsp server: net.majorkernelpanic.streaming.rtsp.RtspServer.start
      start 里 开启一个线程负责监听客户端的请求（VLC） mListenerThread = new RequestListener();
      当有客户端请求时，开启一个WorkerThread 线程 new WorkerThread(mServer.accept()).start();一个线程session代表一个请求
      其实就是服务器的主线程负责接收客户的连接, 每次接收到一个客户连接, 就会创建一个工作线程, 由它负责与客户的通信.
13.http server：net.majorkernelpanic.http.TinyHttpServer.start
    开启一个线程负责监听 mHttpRequestListener = new HttpRequestListener(mHttpPort);
14. 12 和 13 里面都使用的通信方式 ： ServerSocket
    这两部分的代码逻辑类似，只是使用的协议不同，都是充当服务器使用
    在客户/服务器通信模式中, 服务器端需要创建监听端口的 ServerSocket, ServerSocket 负责接收客户连接请求
    Socket socket = serverSocket.accept();//从连接队列中取出一个连接，如果没有则等待
15. 目前只测试了rtsp协议是OK的，所以先重点分享 rtsp
    目前为止已经大概搞清楚了数据传输方式，使用ServerSocket,接下来看看如何把数据传输出去
    net.majorkernelpanic.streaming.rtsp.RtspServer.Response 这个是作为结果返回回去的，有个很重要的处理接收的方法
    net.majorkernelpanic.streaming.rtsp.RtspServer.WorkerThread.processRequest，里面使用了Session类
    net.majorkernelpanic.streaming.Session 这个类用来包装音频或者视频，
    然后可以找到 H264Stream extends VideoStream ，AACStream extends AudioStream
    在 net.majorkernelpanic.streaming.rtsp.RtspServer.WorkerThread.processRequest #428
    里调用了handleRequest ＃437 调用net.majorkernelpanic.streaming.rtsp.UriParser.parse 配置session,
    Session 中含有 AudioStream VideoStream  ，在方法 net.majorkernelpanic.streaming.Session.getSessionDescription
    中获取 描述信息，而后被赋值给 response.content；
    感觉获取到的音频或者视频帧数据就在 getSessionDescription()里被处理的
16. audio ： AACStream --> AACStream.encodeWithMediaCodec --> 开启一个线程AudioRecord获取音频数据送个MediaCodec编码
           解码过程和解码数据封装为 MediaCodecInputStream ,处理数据的过程封装为 AbstractPacketizer 这样一个抽象类
           这个类里包含了一个 net.majorkernelpanic.streaming.rtp.RtpSocket 这样一个维护 一个先进先出的队列的一个线程，
           这个RtpSocket 负责将数据处理为 rtsp 数据，SenderReport 负责处理待发送的数据 SenderReport类包含着处理好的待发送出去的数据
           等待着serverSocket发送数据
           看来传播数据的部分在这里 request.method.equalsIgnoreCase("SETUP")
17. video ：H264Stream --> VideoStream.encodeWithMediaCodec --> VideoStream.encodeWithMediaCodecMethod1 与
    音频的处理类似，也是封装为 MediaCodecInputStream ，只是 AbstractPacketizer的实现类变为 H264Packetizer
    大致的过程就是这个样子，MediaCodec处理摄像头获取的数据，编码后的output数据经 MediaCodecInputStream 处理后包装成 H264Packetizer
    进一步包装为 Session ，然后就可以作为 response 数据响应客户端的请求
18. HTTP 传输
19.
20.


```