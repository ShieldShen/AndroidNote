#MotionEvent
##一个事件的产生到我们可以处理的历程
在Linux内核有一个用来处理Input的子系统InputManager管理，Linux子系统会在/dev/input/这个路径下创建硬件输入设备节点（触摸屏）。当手指触动触摸屏，硬件设备通过设备节点向InputManger报告事件，InputManger会把这个动作处理包装后传递给Android系统里面的WindowMangerService。
[![](http://img.voidcn.com/vcimg/000/000/886/580_c66_819.gif)]
WindowMangerService负责找到应该处理触摸事件的WindowState，通过IWindow代理将下次发送到IWindow服务端，这个IWindow.Stub属于ViewRoot（这个类继承Handler，用于连接PhoneWindow与WindowMangerService）。再由ViewRoot#dispatchPointer()（它就是把MotionEvent发送到handleMessage处理），而handleMessage会调用mView.dispatchTouchEvent方法，这个mView就是一个PhoneWindow.DecorView对象（DecorView是FrameLayout的子类），而DecorView#dispatchTouchEvent方法会有个Callback对象cb，调用到cb的dispatchToucheEvent方法，而这个cb可以在Activity#attach()方法中找到设置
```java:n
 //in file PhoneWindow.java
 public boolean dispatchTouchEvent(MotionEvent ev) {
          final Callback cb = getCallback();
             if (mEnableFaceDetection) {
                int pointCount = ev.getPointerCount();

                switch (ev.getAction() & MotionEvent.ACTION_MASK) {
                case MotionEvent.ACTION_POINTER_DOWN:
       
         .......

             return cb != null && !isDestroyed() && mFeatureId < 0 ? cb.dispatchTouchEvent(ev)
                     : super.dispatchTouchEvent(ev);
         }
 //in file Activity.java -> attach()
         mFragments.attachActivity(this, mContainer, null);
         mWindow = PolicyManager.makeNewWindow(this);
         mWindow.setCallback(this);//设置回调
```
也就是这个Callback其实就是Activity。
在Activity#dispatchTouchEvent()方法：
```java:n
 public boolean dispatchTouchEvent(MotionEvent ev) {
         if (ev.getAction() == MotionEvent.ACTION_DOWN) {
             onUserInteraction();
         }
         if (getWindow().superDispatchTouchEvent(ev)) {
             return true;
         }
         return onTouchEvent(ev);
     }
```
（onUserInteraction()是一个空方法，这个方法在一个Touch事件的周期肯定会被调用）
它会调用PhoneWindow#superDispatchTouchEvent(ev)，而这个方法使用的又是PhoneWindow中mDecor的superDispatchTouchEvent(ev)，这个方法就是super.dispatchTouchEvent()方法，也就是最终还是使用到了ViewGroup的dispatchTouchEvent方法。
小结一下。Event事件是首先到了 PhoneWindow 的 DecorView 的 dispatchTouchEvent 方法，此方法通过 CallBack 调用了 Activity 的 dispatchTouchEvent 方法，在 Activity 这里，我们可以重写 Activity 的dispatchTouchEvent 方法阻断 touch事件的传播。接着在Activity里的dispatchTouchEvent 方法里，事件又再次传递到DecorView，DecorView通过调用父类(ViewGroup)的dispatchTouchEvent 将事件传给父类处理。
（之所以DecorView会把事件溜一圈给Activity再回到自己的superDispatchTouchEvent是为了让Activity能够在事件传递插一腿）
##我们接到MotionEvent（一般是从Activity里面拿到）
如上所述的Activity中dispatchTouchEvent最终还是发到DecorView的superDispatchTouchEvent中，也就是ViewGroup的dispatchTouchEvent。
```java:n
 public boolean dispatchTouchEvent(MotionEvent ev) {
        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onTouchEvent(ev, 1);
        }

        // If the event targets the accessibility focused view and this is it, start
        // normal event dispatch. Maybe a descendant is what will handle the click.
        if (ev.isTargetAccessibilityFocus() && isAccessibilityFocusedViewOrHost()) {
            ev.setTargetAccessibilityFocus(false);
        }

        boolean handled = false;
        if (onFilterTouchEventForSecurity(ev)) {
            final int action = ev.getAction();
            final int actionMasked = action & MotionEvent.ACTION_MASK;

            // Handle an initial down.
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                // Throw away all previous state when starting a new touch gesture.
                // The framework may have dropped the up or cancel event for the previous gesture
                // due to an app switch, ANR, or some other state change.
                cancelAndClearTouchTargets(ev);
                resetTouchState();
            }

            // Check for interception.
            final boolean intercepted;
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }
            } else {
                // There are no touch targets and this action is not an initial down
                // so this view group continues to intercept touches.
                intercepted = true;
            }

            // If intercepted, start normal event dispatch. Also if there is already
            // a view that is handling the gesture, do normal event dispatch.
            if (intercepted || mFirstTouchTarget != null) {
                ev.setTargetAccessibilityFocus(false);
            }

            // Check for cancelation.
            final boolean canceled = resetCancelNextUpFlag(this)
                    || actionMasked == MotionEvent.ACTION_CANCEL;

            // Update list of touch targets for pointer down, if needed.
            final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
            TouchTarget newTouchTarget = null;
            boolean alreadyDispatchedToNewTouchTarget = false;
            if (!canceled && !intercepted) {

                // If the event is targeting accessiiblity focus we give it to the
                // view that has accessibility focus and if it does not handle it
                // we clear the flag and dispatch the event to all children as usual.
                // We are looking up the accessibility focused host to avoid keeping
                // state since these events are very rare.
                View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                        ? findChildWithAccessibilityFocus() : null;

                if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                    final int actionIndex = ev.getActionIndex(); // always 0 for down
                    final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                            : TouchTarget.ALL_POINTER_IDS;

                    // Clean up earlier touch targets for this pointer id in case they
                    // have become out of sync.
                    removePointersFromTouchTargets(idBitsToAssign);

                    final int childrenCount = mChildrenCount;
                    if (newTouchTarget == null && childrenCount != 0) {
                        final float x = ev.getX(actionIndex);
                        final float y = ev.getY(actionIndex);
                        // Find a child that can receive the event.
                        // Scan children from front to back.
                        final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                        final boolean customOrder = preorderedList == null
                                && isChildrenDrawingOrderEnabled();
                        final View[] children = mChildren;
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = getAndVerifyPreorderedIndex(
                                    childrenCount, i, customOrder);
                            final View child = getAndVerifyPreorderedView(
                                    preorderedList, children, childIndex);

                            // If there is a view that has accessibility focus we want it
                            // to get the event first and if not handled we will perform a
                            // normal dispatch. We may do a double iteration but this is
                            // safer given the timeframe.
                            if (childWithAccessibilityFocus != null) {
                                if (childWithAccessibilityFocus != child) {
                                    continue;
                                }
                                childWithAccessibilityFocus = null;
                                i = childrenCount - 1;
                            }

                            if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
                                continue;
                            }

                            newTouchTarget = getTouchTarget(child);
                            if (newTouchTarget != null) {
                                // Child is already receiving touch within its bounds.
                                // Give it the new pointer in addition to the ones it is handling.
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }

                            resetCancelNextUpFlag(child);
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                // Child wants to receive touch within its bounds.
                                mLastTouchDownTime = ev.getDownTime();
                                if (preorderedList != null) {
                                    // childIndex points into presorted list, find original index
                                    for (int j = 0; j < childrenCount; j++) {
                                        if (children[childIndex] == mChildren[j]) {
                                            mLastTouchDownIndex = j;
                                            break;
                                        }
                                    }
                                } else {
                                    mLastTouchDownIndex = childIndex;
                                }
                                mLastTouchDownX = ev.getX();
                                mLastTouchDownY = ev.getY();
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }

                            // The accessibility focus didn't handle the event, so clear
                            // the flag and do a normal dispatch to all children.
                            ev.setTargetAccessibilityFocus(false);
                        }
                        if (preorderedList != null) preorderedList.clear();
                    }

                    if (newTouchTarget == null && mFirstTouchTarget != null) {
                        // Did not find a child to receive the event.
                        // Assign the pointer to the least recently added target.
                        newTouchTarget = mFirstTouchTarget;
                        while (newTouchTarget.next != null) {
                            newTouchTarget = newTouchTarget.next;
                        }
                        newTouchTarget.pointerIdBits |= idBitsToAssign;
                    }
                }
            }

            // Dispatch to touch targets.
            if (mFirstTouchTarget == null) {
                // No touch targets so treat this as an ordinary view.
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
                // Dispatch to touch targets, excluding the new touch target if we already
                // dispatched to it.  Cancel touch targets if necessary.
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
                while (target != null) {
                    final TouchTarget next = target.next;
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        handled = true;
                    } else {
                        final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true;
                        }
                        if (cancelChild) {
                            if (predecessor == null) {
                                mFirstTouchTarget = next;
                            } else {
                                predecessor.next = next;
                            }
                            target.recycle();
                            target = next;
                            continue;
                        }
                    }
                    predecessor = target;
                    target = next;
                }
            }

            // Update list of touch targets for pointer up or cancel, if needed.
            if (canceled
                    || actionMasked == MotionEvent.ACTION_UP
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                resetTouchState();
            } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
                final int actionIndex = ev.getActionIndex();
                final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
                removePointersFromTouchTargets(idBitsToRemove);
            }
        }

        if (!handled && mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
        }
        return handled;
    }
```
又多了一个dispatchTransformedTouchEvent方法，除了down事件的所有事件传递代码都在这里
dispatchTouchEvent主要的判断是以Down事件为基础，在down事件的判断里面，会在ViewGroup中寻找符合当前位置的子View，如果子View消费了这个事件就会记录这个子View作为一个TouchTarget对象，然后后续的事件都会传送给这个Target#child对象去dispatch。
而interceptTouchEvent基本上只会在down的时候去判断，所以如果你down事件没有拦下来，那么也就没啥拦的意思了。
而具体处理事件的方法丢出来给发送给onTouchEvent来处理，而在给onTouchEvent之前会先给onTouch()（当然如果你设置了onTouchListener），而onTouch的返回true的话就不会吧事件发到onTouchEvent里面，而View的onTouchEvent里面就有一些基本的设置View点击状态的还有长短点击的处理。


















