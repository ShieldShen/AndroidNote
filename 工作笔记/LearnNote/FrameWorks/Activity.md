#Android界面的产生
##Activity
Activity作为Android显示UI的组件，它的启动可以来到ActivityThread的handleLaunchActivity方法里面看到：
在handleLaunchActivity里有performLaunchActivity 里面是Activity的创建，然后是activity的attach方法的调用，在attach里面会有PhoneWindow的创建和WindowManager的创建，而我们的onCreate里面有重要的setContentView方法 会向Activity里面添加一个DecorView
```java
final void attach(Context context, ActivityThread aThread,
6594            Instrumentation instr, IBinder token, int ident,
6595            Application application, Intent intent, ActivityInfo info,
6596            CharSequence title, Activity parent, String id,
6597            NonConfigurationInstances lastNonConfigurationInstances,
6598            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
6599            Window window) {
6600        attachBaseContext(context);
6601
6602        mFragments.attachHost(null /*parent*/);
6603
6604        mWindow = new PhoneWindow(this, window);
6605        mWindow.setWindowControllerCallback(this);
6606        mWindow.setCallback(this);
6607        mWindow.setOnWindowDismissedCallback(this);
6608        mWindow.getLayoutInflater().setPrivateFactory(this);
6609        if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
6610            mWindow.setSoftInputMode(info.softInputMode);
6611        }
6612        if (info.uiOptions != 0) {
6613            mWindow.setUiOptions(info.uiOptions);
6614        }
6615        mUiThread = Thread.currentThread();
6616
6617        mMainThread = aThread;
6618        mInstrumentation = instr;
6619        mToken = token;
6620        mIdent = ident;
6621        mApplication = application;
6622        mIntent = intent;
6623        mReferrer = referrer;
6624        mComponent = intent.getComponent();
6625        mActivityInfo = info;
6626        mTitle = title;
6627        mParent = parent;
6628        mEmbeddedID = id;
6629        mLastNonConfigurationInstances = lastNonConfigurationInstances;
6630        if (voiceInteractor != null) {
6631            if (lastNonConfigurationInstances != null) {
6632                mVoiceInteractor = lastNonConfigurationInstances.voiceInteractor;
6633            } else {
6634                mVoiceInteractor = new VoiceInteractor(voiceInteractor, this, this,
6635                        Looper.myLooper());
6636            }
6637        }
6638
6639        mWindow.setWindowManager(
6640                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
6641                mToken, mComponent.flattenToString(),
6642                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
6643        if (mParent != null) {
6644            mWindow.setContainer(mParent.getWindow());
6645        }
6646        mWindowManager = mWindow.getWindowManager();
6647        mCurrentConfig = config;
6648    }
```
然后在handlelaunchActivity里面接着往下走是handleResumeActivity方法，里面有向Activity里添加ViewRoot的过程，主要是WindowManager的addView方法以及ViewRoot的setView方法
在ViewRoot的构造方法里面可以找到WindowMangerService与ViewRoot建立连接的过程openSession
在setView方法里面有requestLayout就是UI绘制流程，后面有个IWindowSessioni.addWindow就是把当前的窗口状态等乱七八糟的东西记录到WMS里面成为一个WindowState，然后在WindowState的Attach方法里面建立第二个管道IWIndow，而这个就是输入事件的传输通道