#AppWidget
[TOC]
##Prepare
###Declaring
#### AppWidgetProvider
 是BroadCastReceiver的子类
```xml
<receiver android:name="ExampleAppWidgetProvider" >
    <intent-filter>
        <action android:name="android.appwidget.action.APPWIDGET_UPDATE" />
    </intent-filter>
    <meta-data android:name="android.appwidget.provider"
               android:resource="@xml/example_appwidget_info" />
</receiver>
```
> <meta-data>标签：
> name ：一个标识固定写法
> resource：存放在 res/xml/ 目录下

#####AppWidgetProviderInfo
```xml
<appwidget-provider xmlns:android="http://schemas.android.com/apk/res/android"
    android:minWidth="40dp"
    android:minHeight="40dp"
    android:updatePeriodMillis="86400000"
    android:previewImage="@drawable/preview"
    android:initialLayout="@layout/example_appwidget"
    android:configure="com.example.android.ExampleAppWidgetConfigure"
    android:resizeMode="horizontal|vertical"
    android:widgetCategory="home_screen">
</appwidget-provider>
```
> minWidth/minHeight 是初始化的时候的最低大小
> minResizeWidth/Height 是调整的时候最低大小
> resizeMode 调整模式
> widgetCategory 控件显示方式，home / keyguard(锁屏) 在5.0以后 keyguard不可用
> autoAdvancedId 3.0特有， 什么意思不知道
> previewImage 预览
> configure 设置Activity 设置了之后会在添加的时候启动这个Activity
> initialLayout 要显示的界面
> updatePeriodMillis 每次系统发送通知的时间间隔（update消息），如果用其他方式来使用的话，写成0就可以


###可以使用的Layout
####RemoteView
#####Introduction
是可以在其他进程显示的View ，因为要跨进程，相比于其他View来说，多了很多限制，可以用的如下
> FrameLayout
> LinearLayout
> RelativeLayout
> GridLayout
> AnalogClock
> Button
> Chronometer
> ImageButton
> ImageView
> ProgressBar
> TextView
> ViewFlipper
> ListView
> GridView
> StackView
> AdapterViewFlipper
子类不可用

####给Widget设置Margin
在4.0以后就不用费心思，而在4.0以前需要在Layout里面自己手动设置
Example：
Layout:
```xml
<FrameLayout
  android:layout_width="match_parent"
  android:layout_height="match_parent"
  android:padding="@dimen/widget_margin">

  <LinearLayout
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="horizontal"
    android:background="@drawable/my_widget_background">
    …
  </LinearLayout>
</FrameLayout>
```
dimens:
```xml
#res/values/dimens.xml:
<dimen name="widget_margin">8dp</dimen>
#res/values-v14/dimens.xml:
<dimen name="widget_margin">0dp</dimen>
```
##AppWidgetProvider
###CallBacks
####onUpdate()
根据updatePeriodMillis周期调用，在没有使用ConfigureActivity的情况下第一次添加也会触发此回调，但是添加了ConfigureActivity添加Widget到桌面的时候就不会触发
####onAppWidgetOptionsChanged()（4.1&Later）
在空间被拖动或者改变大小都会触发
Bundle参数:
* OPTION_APPWIDGET_MIN_WIDTH—Contains the lower bound on the current width, in dp units, of a widget instance.
* OPTION_APPWIDGET_MIN_HEIGHT—Contains the lower bound on the current height, in dp units, of a widget instance.
* OPTION_APPWIDGET_MAX_WIDTH—Contains the upper bound on the current width, in dp units, of a widget instance.
* OPTION_APPWIDGET_MAX_HEIGHT—Contains the upper bound on the current width, in dp units, of a widget instance.
####onDeleted(Context, int[])
每一个控件被删除都会触发
####onEnabled(Context)
第一次添加空间会触发
####onDisabled(Context)
所有空间都被删除的时候触发
####onReceive(Context, Intent)
就是BroadCastReceive的回调，上面的回调就是这个处理了之后的结果
###Example
```java
public class ExampleAppWidgetProvider extends AppWidgetProvider {

    public void onUpdate(Context context, AppWidgetManager appWidgetManager, int[] appWidgetIds) {
        final int N = appWidgetIds.length;

        // Perform this loop procedure for each App Widget that belongs to this provider
        for (int i=0; i<N; i++) {
            int appWidgetId = appWidgetIds[i];

            // Create an Intent to launch ExampleActivity
            Intent intent = new Intent(context, ExampleActivity.class);
            PendingIntent pendingIntent = PendingIntent.getActivity(context, 0, intent, 0);

            // Get the layout for the App Widget and attach an on-click listener
            // to the button
            RemoteViews views = new RemoteViews(context.getPackageName(), R.layout.appwidget_provider_layout);
            views.setOnClickPendingIntent(R.id.button, pendingIntent);

            // Tell the AppWidgetManager to perform an update on the current app widget
            appWidgetManager.updateAppWidget(appWidgetId, views);
        }
    }
}
```
###Configuration Activity
就是用来在第一次添加控件的时候触发调起的Activity，一般做一些配置登录什么的工作
使用的话在AppWidgetProviderInfo中的那个xml文件里面配置configure
####Example
```xml
<activity android:name=".ExampleAppWidgetConfigure">
    <intent-filter>
    #这里一定要写全称，应为是从别的应用调起，省略的话会找不到
        <action android:name="android.appwidget.action.APPWIDGET_CONFIGURE"/>
    </intent-filter>
</activity>
```
####从ConfigureActivity刷新Widget
#####Ep.
1. 1st
```java
Intent intent = getIntent();
Bundle extras = intent.getExtras();
if (extras != null) {
    mAppWidgetId = extras.getInt(
            AppWidgetManager.EXTRA_APPWIDGET_ID,
            AppWidgetManager.INVALID_APPWIDGET_ID);
}
```
2. 2nd
做你的配置操作
3. 3rd
```java
AppWidgetManager appWidgetManager = AppWidgetManager.getInstance(context);
```
4. 4th
```java
RemoteViews views = new RemoteViews(context.getPackageName(),
R.layout.example_appwidget);
appWidgetManager.updateAppWidget(mAppWidgetId, views);
```
5. 5th
```java
Intent resultValue = new Intent();
resultValue.putExtra(AppWidgetManager.EXTRA_APPWIDGET_ID, mAppWidgetId);
setResult(RESULT_OK, resultValue);
finish();
```
###使用Collections展示多项元素
####可以使用的视图
1. ListView
2. GridView
3. StackView
4. AdapterViewFlipper
####使用前提
1. 一个继承RemoteViewsService的服务
2. 一个实现了RemoteViewsService.RemoteViewsFactory的Factory（其实就是个Adapter，不过它可以把生成的各个item跨进程发到Widget上）
#####Example
配置
```xml
<service android:name="MyWidgetService"
...
android:permission="android.permission.BIND_REMOTEVIEWS" />
```
代码
```java
public class StackWidgetService extends RemoteViewsService {
    @Override
    public RemoteViewsFactory onGetViewFactory(Intent intent) {
        return new StackRemoteViewsFactory(this.getApplicationContext(), intent);
    }
}

class StackRemoteViewsFactory implements RemoteViewsService.RemoteViewsFactory {

//... include adapter-like methods here. See the StackView Widget sample.

}
```
####在使用了Collections下AppWidgetProvider的示例代码
```java
public void onUpdate(Context context, AppWidgetManager appWidgetManager,
int[] appWidgetIds) {
    // update each of the app widgets with the remote adapter
    for (int i = 0; i < appWidgetIds.length; ++i) {

        // Set up the intent that starts the StackViewService, which will
        // provide the views for this collection.
        Intent intent = new Intent(context, StackWidgetService.class);
        // Add the app widget ID to the intent extras.
        intent.putExtra(AppWidgetManager.EXTRA_APPWIDGET_ID, appWidgetIds[i]);
        intent.setData(Uri.parse(intent.toUri(Intent.URI_INTENT_SCHEME)));
        // Instantiate the RemoteViews object for the app widget layout.
        RemoteViews rv = new RemoteViews(context.getPackageName(), R.layout.widget_layout);
        // Set up the RemoteViews object to use a RemoteViews adapter.
        // This adapter connects
        // to a RemoteViewsService  through the specified intent.
        // This is how you populate the data.
        rv.setRemoteAdapter(appWidgetIds[i], R.id.stack_view, intent);

        // The empty view is displayed when the collection has no items.
        // It should be in the same layout used to instantiate the RemoteViews
        // object above.
        rv.setEmptyView(R.id.stack_view, R.id.empty_view);

        //
        // Do additional processing specific to this app widget...
        //

        appWidgetManager.updateAppWidget(appWidgetIds[i], rv);
    }
    super.onUpdate(context, appWidgetManager, appWidgetIds);
}
```
####Factory的接口
1. onCreate
2. getViewAt
 * 返回的是一个RemoteViews对象
###设置事件监听
####一般事件监听
```java
 // Create an Intent to launch ExampleActivity
            Intent intent = new Intent(context, ExampleActivity.class);
            PendingIntent pendingIntent = PendingIntent.getActivity(context, 0, intent, 0);

            // Get the layout for the App Widget and attach an on-click listener
            // to the button
            RemoteViews views = new RemoteViews(context.getPackageName(), R.layout.appwidget_provider_layout);
            views.setOnClickPendingIntent(R.id.button, pendingIntent);
```
####Collections给每个Item设置
这里面的rv变量就是RemoteViews
1. 在Provider onUpdate中设置Template
```java
 // This section makes it possible for items to have individualized behavior.
            // It does this by setting up a pending intent template. Individuals items of a collection
            // cannot set up their own pending intents. Instead, the collection as a whole sets
            // up a pending intent template, and the individual items set a fillInIntent
            // to create unique behavior on an item-by-item basis.
            Intent toastIntent = new Intent(context, StackWidgetProvider.class);
            // Set the action for the intent.
            // When the user touches a particular view, it will have the effect of
            // broadcasting TOAST_ACTION.
            toastIntent.setAction(StackWidgetProvider.TOAST_ACTION);
            toastIntent.putExtra(AppWidgetManager.EXTRA_APPWIDGET_ID, appWidgetIds[i]);
            intent.setData(Uri.parse(intent.toUri(Intent.URI_INTENT_SCHEME)));
            PendingIntent toastPendingIntent = PendingIntent.getBroadcast(context, 0, toastIntent,
                PendingIntent.FLAG_UPDATE_CURRENT);
                //这里这里这里
            rv.setPendingIntentTemplate(R.id.stack_view, toastPendingIntent);
```
2. 给每个Item设置FillInIntent
```java
 Bundle extras = new Bundle();
            extras.putInt(StackWidgetProvider.EXTRA_ITEM, position);
            Intent fillInIntent = new Intent();
            fillInIntent.putExtras(extras);
            // Make it possible to distinguish the individual on-click
            // action of a given item
            rv.setOnClickFillInIntent(R.id.widget_item, fillInIntent);
```
###保持Collection的数据刷新的方式
当然有好多方法，我们的Calendar就是一个定时去Query的
[![](https://developer.android.com/images/appwidgets/appwidget_collections.png)]
