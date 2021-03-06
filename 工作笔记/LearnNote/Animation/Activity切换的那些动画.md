#Activity切换的那些动画
##过渡
Android已经发展到7.0。android app数不胜数，仅仅功能切中用户痛点已经不能满足市场的需求了。与功能并重的UI效果被各家App所重视，它们实现的各种各样交互方式、UI效果来给用户带来更强的吸引力。
其中，如何让各个界面的跳转显得更加流畅也是非常值得研究的问题。而Android原本已经很好的支持了一套转场动画的框架来提供给开发者实现自己的过渡动画。
在Android4.4的时候，出现了Transition框架，这套框架为属性动画的实现提供极大便利，而在5.0之后，这套框架又被用于转场动画，可以实现很多酷炫的效果。
##5.0时代前后的实现
Activity与Fragment都可以实现，而且方法类似，这里仅讨论Activity的跳转
###5.0以前
转场动画的实现一般有两种方式
1. 在Activity startActivity以及finish的后面，紧接调用overridePenddingTransition
```java
 public void overridePendingTransition(int enterAnim, int exitAnim)
```
2. 在Theme中定义windowAnimationStyle
```xml
<style name="AppTheme" parent="Theme.AppCompat.Light.DarkActionBar">
        <!-- Customize your theme here. -->
        <item name="colorPrimary">@color/colorPrimary</item>
        <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
        <item name="colorAccent">@color/colorAccent</item>
        <item name="windowContentTransitions">true</item>
        <item name="android:windowAnimationStyle">@style/RightInOut</item>
    </style>

    <style name="RightInOut">
        <item name="android:activityCloseExitAnimation">@anim/slide_right_out</item>
        <item name="android:activityCloseEnterAnimation">@anim/steady</item>
        <item name="android:activityOpenExitAnimation">@anim/steady</item>
        <item name="android:activityOpenEnterAnimation">@anim/slide_right_in</item>
    </style>
```
这两种方式，其中直接使用api的方式会覆盖通过主题设置的动画，而在主题中的四个标签可以这样理解
```text
有 A B 两个Activity
1. A使用startActivity跳转B：
    这个时候B的Open***生效，相当于在A的startActivity后面使用overridePendingTransition
2. B调用finish回退到A：
    这个时候A的Close***生效，相当于在B的finish后面使用overridePendingTransition
```
而除了这四个标签以外，还有两个标签，不过一般不太会用windowExitAnimation 以及 windowEnterAnimation，这两个标签的动画是作用于window的，如果设置了，会让设置的activity的其他过场动画失效。相比于activityOpenEnterAnimation这四个标签，这两个标签不够灵活，但是够暴力。（在上述的例子中，如果A设置了windowExitAnimation,那么在B进入的时候，A只会响应自己的windowExitAnimation）
贴一个动画xml
slide_right_in.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android">
    <translate android:fromXDelta="0" android:toXDelta="100%p" android:duration="300"/>
</set>
```
###5.0以后
5.0之前实现过场动画有一定局限性，它仅仅是把整个视图一起进行简单的过渡变幻。而在Lolipop之后，我们可以使用api让过渡动画细分到每一个View，还可以实现一个view从一个界面过渡另一个界面的视觉效果。
5.0之后的场景切换动画是建立在Transitions上，这个框架为不同UI间切换产生的动画效果提供了便捷的API。Transitions的实现主要基于两个概念，场景（Scenes）和变换（Transition）。Scenes会记录UI状态，而Transitions会根据开始场景以及结束场景中对应的UI状态创建一个动画。
####使用Transitions做一个动画
```java
    @Override
    protected void initView() {

        final RelativeLayout container = (RelativeLayout) findViewById(R.id.content_kitkat);
        final TextView iv1 = (TextView) findViewById(R.id.tv1);
        final TextView iv2 = (TextView) findViewById(R.id.tv2);
        if (TextUtils.equals(getMode(), MODE_3)) {

            setFab(new View.OnClickListener() {
                @Override
                public void onClick(View view) {
                    TransitionManager.beginDelayedTransition(container, new Fade());
                    toggle(iv1);
                    toggle(iv2);
                }
            });
        } else {
            iv1.setVisibility(View.GONE);
            iv2.setVisibility(View.GONE);
        }
    }

    public void toggle(View view) {
        view.setVisibility(view.getVisibility() == View.VISIBLE ? View.INVISIBLE : View.VISIBLE);
    }
```
![](./_image/20170213-1.gif)
这个框架是不是很方便啊，接下来就是主题了，使用Transitions做场景切换动画。
###使用Transitions做场景切换动画
与5.0之前版本一样，A进入B B返回A这两个过程由四个动画来决定效果：
> A进入B时，A执行exit transition B执行enter transition
> B返回A时，B执行return transition A执行reenter transition

而在使用Transitions做场景切换时，框架允许我们可以让场景整体变换，也可以有一个元素从一个场景过渡到另一个场景的效果（shared elements transition）
这个被到处用的例子
![](./_image/1-150113195F54J.gif)
下面就详细介绍Transitions的相关Api










