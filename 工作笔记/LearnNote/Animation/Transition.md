#Transition
`过渡动画`
[TOC]
##Abstract
```
安卓5.0中Activity和Fragment 变换是建立在名叫Transitions的安卓新特性之上的。这个诞生于4.4的transition框架为在不同的UI状态之间产生动画效果提供了非常方便的API。该框架主要基于两个概念：场景（scenes）和变换（transitions）。
当一个场景改变的时候，transition主要负责：
（1）捕捉每个View在开始场景和结束场景时的状态。
（2）根据两个场景（开始和结束）之间的区别创建一个Animator。
```
其实Transition就是一套封装了属性动画的框架，他会计算你丢给他的View的前后状态来生成一个动画。
##Example
```java
public class ExampleActivity extends Activity implements View.OnClickListener {
    private ViewGroup mRootView;
    private View mRedBox, mGreenBox, mBlueBox, mBlackBox;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mRootView = (ViewGroup) findViewById(R.id.layout_root_view);
        mRootView.setOnClickListener(this);
        mRedBox = findViewById(R.id.red_box);
        mGreenBox = findViewById(R.id.green_box);
        mBlueBox = findViewById(R.id.blue_box);
        mBlackBox = findViewById(R.id.black_box);
    }
    @Override
    public void onClick(View v) {
        TransitionManager.beginDelayedTransition(mRootView, new Fade());
        toggleVisibility(mRedBox, mGreenBox, mBlueBox, mBlackBox);
    }
    private static void toggleVisibility(View... views) {
        for (View view : views) {
            boolean isVisible = view.getVisibility() == View.VISIBLE;
            view.setVisibility(isVisible ? View.INVISIBLE : View.VISIBLE);
        }
    }
}
```
> 就是View从Visible 到inVisible的过程

1. 当点击事件发生之后调用TransitionManager的beginDelayedTransition()方法，并且传递了mRootView和一个Fade对象最为参数。之后，framework会立即调用transition类的captureStartValues()方法为每个view保存其当前的可见状态(visibility)。
2. 当beginDelayedTransition返回之后，在上面的代码中将每个view设置为不可见。
3. 在接下来的显示中framework会调用transition类的captureEndValues()方法，记录每个view最新的可见状态。
4. 接着，framework调用transition的createAnimator()方法。transition会分析每个view的开始和结束时的数据发现view在开始时是可见的，结束时是不可见的。Fade（transition的子类）会利用这些信息创建一个用于把view的alpha属性变为0的AnimatorSet，并且将此AnimatorSet对象返回。
5. framework会运行返回的Animator，导致所有的View都渐渐消失。

这个简单的例子强调了transition框架的两个主要优点。
1. Transitions抽象和封装了属性动画，Animator的概念对开发者来说是透明的，因此它极大的精简了代码量。开发者所做的所有事情只是改变一下view前后的状态数据，Transition就会自动的根据状态的区别去生成动画效果。
2. 不同场景之间变换的动画效果可以简单的通过使用不同的Transition类来改变，本例中用的是Fade。
[![](http://jcodecraeer.com/uploads/20150113/1421146073174536.gif)]
实现上图中的那两个不同的动画效果可以将Fade替换成Slide或者Explode即可。在接下来的文章中你将会发现，这些优点将使得我们只用少量代码就可以创建复杂的Activity 和Fragment切换动画。在接下来的小节中，将看到是如何使用Lollipop的Activity 和Fragment transition API来实现这种变换的。
##Prepare
Android 5.0中Transition可以被用来实现Activity或者Fragment切换时的异常复杂的动画效果。虽然在以前的版本中，已经可以使用Activity的overridePendingTransition() 和 FragmentTransaction的setCustomAnimation()来实现Activity或者Fragment的动画切换，但是他们仅仅局限与将整个视图一起动画变换。新的Lollipop api更进了一步，让单独的view也可以在进入或者退出其布局容器中时发生动画效果，甚至还可以在不同的activity/Fragment中共享一个view。
Activity transition API围绕退出（exit），进入（enter），返回（return）和再次进入（reenter）四种transition。按照上面对A和B的约定，我这样描述这一过程。
>Activity A的退出变换（exit transition）决定了在A调用B的时候，A中的View是如何播放动画的。
>Activity B的进入变换（enter transition）决定了在A调用B的时候，B中的View是如何播放动画的。
>Activity B的返回变换（return transition）决定了在B返回A的时候，B中的View是如何播放动画的。
>Activity A的再次进入变换（reenter transition）决定了在B返回A的时候，A中的View是如何播放动画的。

最后framework提供了两种Activity transition- 内容transition和共享元素的transition：
1. Content Transition: 决定非共享视图在场景切换的时候的动画
2. Shared Element Transition：决定共享元素在场景切换的时候怎样从一个场景过度到另一个场景
[![](http://jcodecraeer.com/uploads/150113/1-150113195F54J.gif)]
##API（Required 5.0Level & Beyond）
###Activity Transition API
下面的总结是实现Activity transition的步骤。
1. 在调用与被调用的activity中，通过设定Window.FEATURE_ACTIVITY_TRANSITIONS  和 Window.FEATURE_CONTENT_TRANSITIONS 来启用transition api ，可以通过代码也可以通过设置主题来启用：
>主题xml
```xml
<item name="android:windowContentTransitions">true</item>
```
>代码方式，在setContentView之前调用：
```java
getWindow().requestFeature(Window.FEATURE_CONTENT_TRANSITIONS);
```
2. 分别在调用与被调用的activity中设置exit 和enter transition。**Material主题默认会将exit的transition设置成null而enter的transition设置成Fade** .如果reenter 或者 return transition没有明确设置，则将用exit 和enter的transition替代。
3. 分别在调用与被调用的activity中设置exit 和enter 共享元素的transition。Material**主题默认会将exit的共享元素transition设置成null而enter的共享元素transition设置成@android:transition/move.**如果reenter 或者 return transition没有明确设置，则将用exit 和enter的共享元素transition替代。
开始一个activity的content transaction需要调用startActivity(Context, Bundle)方法，将下面的bundle作为第二个参数：
```java
ActivityOptions.makeSceneTransitionAnimation(activity, pairs).toBundle();
```
其中pairs参数是一个数组：Pair<View, String> ，该数组列出了你想在activity之间共享的view和view的名称。别忘了给你的共享元素加上一个唯一的名称，否则transition可能不会有正确的结果。
4. 在代码中触发通过finishAfterTransition()方法触发返回动画，而不是调用finish()方法。
5. 默认情况下，material主题的应用中enter/return的content transition会在exit/reenter的content transitions结束之前开始播放（只是稍微早于），这样会看起来更加连贯。如果你想明确屏蔽这种行为，可以调用setWindowAllowEnterTransitionOverlap() 和 setWindowAllowReturnTransitionOverlap()方法。
###Fragment Transition API
如果你想在Fragment中使用transition，除了一小部分区别之外和activity大体一致：
1. Content的exit, enter, reenter, 和return transition需要调用fragment的相应方法来设置，或者通过fragment的xml属性来设置。
2. 共享元素的enter和return transition也n需要调用fragment的相应方法来设置，或者通过fragment的xml属性来设置。
3. 虽然在activity中transition是被startActivity()和finishAfterTransition()触发的，但是Fragment的transition却是在其被FragmentTransaction执行下列动作的时候自动发生的。added, removed, attached, detached, shown, ，hidden。
4. 在Fragment commit之前，共享元素需要通过调用addSharedElement(View, String) 方法来成为FragmentTransaction的一部分。
##Content Transition
```
content transition决定了非共享view元素在activity和fragment切换期间是如何进入或者退出场景的。根据google最新的Material Design设计语言，content transition让我们毫不费力的去协调Activity/Fragment切换过程中view的进入和退出，让这个过程更流畅。在5.0之后content transition可以通过调用Window和Fragment的如下代码来设置：
```
###Detail
一个Transition主要有两个职责：捕获目标view的开始和结束时的状态、创建一个用于在两个状态之间播放动画的Animator。
在Content transition动画（animation）创建之前，framework必须通过设置transitioning view的visibility将动画需要的状态信息告诉animation。具体来说，当Activity A startsActivity B之时，发生了如下的事件：
1. Activity A 调用startActivity()**(创建A的Transition Animator)**
 * framework遍历A的View树，确定当A的exit transition运行时哪些view会退出场景（即哪些view是transitioning view）。
 * A的exit transition捕获A中transitioning view的开始状态。
 * framework将A中所有的transitioning view设置为INVISIBLE。
 * A的exit transition捕获到A中transitioning view的结束状态。
 * A的exit transition比较每个transitioning view的开始和结束状态，然后根据前后状态的区别创建一个Animator。Animator开始运行，同时transitioning view退出场景。
2. Activity B启动.**(创建B的Transition Animator)**
 * framework遍历B的View树，确定当B的enter transition运行时哪些view会进入场景，transitioning view会被初始化为INVISIBLE
 * B的enter transition捕获B中transitioning view的开始状态。
 * framework将B中所有的transitioning view设置为VISIBLE。
 * B的enter transition捕获到B中transitioning view的结束状态。
 * B的enter transition比较每个transitioning view的开始和结束状态，然后根据前后状态的区别创建一个Animator。Animator开始运行，同时transitioning view进入场景。
通过在每个transitioning view中来回**切换INVISIBLE 和VISIBLE**，framework确保content transition得到创建animation（期望的animation）所需的状态信息。显然content Transition对象需要在开始和结束场景中都能记录到transitioning view的visibility。 非常幸运的是抽象类Visibility已经为你做了这些工作：**Visibility**的子类只需要实现**onAppear()** 和 **onDisappear()** 两个工厂方法，**在这两个工厂方法中创建并返回一个进入或者退出场景的Animator对象。**在api 21中，有三个现成的Visibility的实现：Fade, Slide, 和 Explode
他们都可以用在Activity 和 Fragment中创建content transition。如果必要，还可以自定义Visibility
###Transitioning Views以及Transition Groups
到目前为止，我们假设了content transition是操作非共享元素的（即提到了很多次的transitioning view）。在本节，我们将讨论framework是如何决定transitioning view的集合以及如何使用transition groups自定义它。
在transition开始之前，framework通过递归遍历Activity（或者Fragment）的Window中的View树来决定transitioning view的集合。整个搜索过程开始在根view中调用ViewGroup的captureTransitioningView方法，captureTransitioningView的代码如下：
```java
/** @hide */
@Override
public void captureTransitioningViews(List<View> transitioningViews) {
    if (getVisibility() != View.VISIBLE) {
        return;
    }
    if (isTransitionGroup()) {
        transitioningViews.add(this);
    } else {
        int count = getChildCount();
        for (int i = 0; i < count; i++) {
            View child = getChildAt(i);
            child.captureTransitioningViews(transitioningViews);
        }
    }
}
```
这个递归的过程比较简单粗暴：framework跟踪不同级别的view树直到它找到一个VISIBLE的叶子view或者是一个transition group。transition group允许我们将整个ViewGroup作为一个整体来变换。如果一个ViewGroup的isTransitionGroup()方法返回true，则它的所有孩子都将被视为一个整体一起播放动画。否则将会继续递归该ViewGroup，其子view也会在动画的时候被单独对待。遍历的最后结果是一个完整的transitioning view的集合将在content transition的时候播放动画。
> 默认情况下，isTransitionGroup()将在ViewGroup有背景或者有transition name的时候返回true（参见documentation 中对该方法的声明）。

以下图为例，在整个过程中，用户的头像先是作为一个单独的元素渐渐的进入到下一个界面，而在返回的时候他又是和其他元素一起作为一个整体被动画。google play Games 中貌似用的是transition group来实现将屏幕分成两半的效果。
[![](http://jcodecraeer.com/uploads/20150116/1421395183295370.gif)]
有时候transition groups被用来修改一些Activity切换是出现的莫名其妙的bug。还是以上图为例，calling Activity 显示了封面图片的相册界面，而被调用activity则显示了一个header的背景图片，共享的封面图片，一个webview。这个app使用了类似与Google Play Games的transition：从中间成两半，各自滑倒上下边缘。但是，仔细观察你会发现只有上部分有滑动的动画效果，Webview没有。
那么问题来了，到底是哪里没对？上面的结果证明WebView虽然是一个ViewGroup但是没有被系统认为是transition view。因此content transition没有在它上面运行。幸运的是我们可以在return transition之前的某个地方调用
```java
webView.setTransitionGroup(true)
```
来解决这个问题。
##Shared Element Transition
###Concept
元素共享式变换（shared element transition）决定了共享的view元素从一个Activity/Fragment 到另一个Activity/Fragment t的切换中是如何动画变化的。
共享元素在被调用Activity进入和返回时播放动画，共享元素在进入和返回时的变换效果通过window和Fragment的如下方法来设置：
进入：`setSharedElementEnterTransition()`
返回：`setSharedElementReturnTransition()`
设置在B返回A时的动画，共享元素以B中的位置作为起始，A中的位置为结束来播放动画。
>注意，Activity Transition API 也可以使用 setSharedElementExitTransition() 和setSharedElementReenterTransition()方法分别设置共享元素的exit 和reenter 变换。但是一般来讲这是不必要的。如果你想看先关的例子，可以查看这篇博客[this blog post](https://halfthought.wordpress.com/2014/12/08/what-are-all-these-dang-transitions/).至于为什么Fragment中没有共享元素的exit 和reenter 变换，请查看George Mount在stackoverflow上的回答:[this StackOverflow post](http://stackoverflow.com/questions/27346020/understanding-exit-reenter-shared-element-transitions)。

[![](http://www.jcodecraeer.com/uploads/20150201/1422779465610233.gif)]

上图演示了google play music应用中的共享元素变换效果。变换包含了两个共享的view元素：一个ImageView以及他的父亲CardView。ImageView在两个activity之间无缝的动画切换，而CardView则是渐渐的扩展到界面上。
###Detail
捕获view开始和结束状态以及创建一能在两个状态间渐变的动画。共享元素变换没有什么不同。在共享元素变换开始之前，必须首先捕获每个共享元素的开始和结束状态（调用activity以及被调用activity中的位置、大小、外观），有了这些信息才能决定每个共享元素的入场动画。
当Activity A 调用 Activity B ，发生的事件流如下:
1. Activity A调用startActivity()， Activity B被创建，测量，同时初始化为半透明的窗口和透明的背景颜色。
2. framework重新分配每个共享元素在B中的位置与大小，使其跟A中一模一样。之后，B的进入变换（enter transition）捕获到共享元素在B中的初始状态。
3. framework重新分配每个共享元素在B中的位置与大小，使其跟B中的最终状态一致。之后，B的进入变换（enter transition）捕获到共享元素在B中的结束状态。
4. B的进入变换（enter transition）比较共享元素的初始和结束状态，同时基于前后状态的区别创建一个Animator(属性动画对象)。
5. framework 命令A隐藏其共享元素，动画开始运行。随着动画的进行，framework 逐渐将B的activity窗口显示出来，当动画完成，B的窗口才完全可见。
与内容变换（content transition）取决于view的可见性不同（visibility），共享元素变换取决于每个共享元素的位置、大小以及外观。在api 21中，框架层提供了几个Transition 的实现，可以用于定义共享元素在场景中的切换效果。
[ChangeBounds](https://developer.android.com/reference/android/transition/ChangeTransform.html) -捕获共享元素的layout bound，然后播放layout bound变化动画。ChangeBounds 是共享元素变换中用的最多的，因为前后两个activity中共享元素的大小和位置一般都是不同的。
[ChangeClipBounds](https://developer.android.com/reference/android/transition/ChangeClipBounds.html) -  捕获共享元素clip bounds，然后播放clip bounds变化动画。
[ChangeImageTransform](https://developer.android.com/reference/android/transition/ChangeImageTransform.html) -  捕获共享元素（ImageView）的transform matrices 属性，然后播放ImageViewtransform matrices 属性变化动画。与ChangeBounds相结合，这个变换可以让ImageView在动画中高效实现大小，形状或者[ImageView.ScaleType](https://developer.android.com/reference/android/widget/ImageView.ScaleType.html) 属性平滑过度。
[@android:transition/move](https://github.com/android/platform_frameworks_base/blob/lollipop-release/core/res/res/transition/move.xml) -  将上述所有变换同时进行的一个TransitionSet 。就如在第一章中所讲的一样，如果共享元素的进入和返回变换没有特别声明，框架将使用它作为默认的变换。
我们可以看到，共享元素变换并不是真正实现了两个activity或者Fragment之间元素的共享，实际上我们看到的几乎所有变换效果中（不管是B进入还是B返回A）,共享元素都是在B中绘制出来的。Framework没有真正试图将A中的某个元素传递给B，而是采用了不同的方法来达到相同的视觉效果。A传递给B的是共享元素的状态信息。B利用这些信息来初始化共享View元素，让它们的位置、大小、外观与在A中的时候完全一致。当变换开始的时候，B中除了共享元素之外，所有的其他元素都是不可见的。随着动画的进行，framework 逐渐将B的activity窗口显示出来，当动画完成，B的窗口才完全可见。
###使用共享元素的 Overlay
最后，我们需要讨论一下shared element overlay这个概念才算是对共享元素变换的绘制过程有了一个完整的了解。
虽然不是非常明显的可以看到，共享元素默认其实是绘制在整个view树结构的最上层，在一个叫[ViewOverlay](https://developer.android.com/reference/android/view/ViewOverlay.html)的东西上面。你可能没听说过ViewOverlay，他是4.3之后才有的一个新类，它是view的最上面的一个透明的层，添加到ViewOverlay上面的Drawable和view可以被绘制到任何东西的上面，甚至是ViewGroup的子元素。这似乎可以解释为什么framework 会选择ViewOverlay来作为共享元素变换的绘制空间了。
-关于ViewOverlay，除了官方解释还可以看看这篇文章：[ViewOverlay与animation介绍 ](http://jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0130/2384.html)
虽然共享元素默认是绘制在ViewOverlay上面，但是framework 还是提供了关闭这个选项的功能，调用Window的setSharedElementsUseOverlay(false) 方法。这样主要是为了防止万一有这样的开发需要。如果你选择了关闭overlay，那么请注意这是有一定副作用的。
[![](http://www.jcodecraeer.com/uploads/20150201/1422779979139164.gif)]


















