**Android ItemTouchHelper 源码分析**

当需要进行Recyclerview进行Item交换位置或者侧滑删除等操作的时候，就需要用到ItemTouchHelper，它是实现Recyclerview 拖拽效果的帮助类。

和和用户进行交互的主要环节就是：Callback类



**ItemTouchHelper源码分析**

阅读源码的目标：

* 了解ItemTouchHelper的执行流程
* 如何实现触摸跟随动画
* 如何实现的触摸结束的Item切换动画效果

带着问题去看才能有所收获，否则很容易迷失在源码里边。



**1.从attachToRecyclerView这个方法是入口**

```java
public void attachToRecyclerView(@Nullable RecyclerView recyclerView) {
    if (this.mRecyclerView != recyclerView) {
        if (this.mRecyclerView != null) {
            this.destroyCallbacks(); 解除绑定，以及清除
        }

        this.mRecyclerView = recyclerView;
        if (recyclerView != null) {
            Resources resources = recyclerView.getResources();
            this.mSwipeEscapeVelocity = resources.getDimension(dimen.item_touch_helper_swipe_escape_velocity);
            this.mMaxSwipeVelocity = resources.getDimension(dimen.item_touch_helper_swipe_escape_max_velocity);
            this.setupCallbacks(); //设置绑定
        }

    }
}
```

this.destroyCallbacks()：这里主要是解除绑定，执行一些清楚的操作。



**下面来看this.setupCallbacks()这个方法**

```java
private void setupCallbacks() {
    ViewConfiguration vc = ViewConfiguration.get(this.mRecyclerView.getContext());
    this.mSlop = vc.getScaledTouchSlop();
    this.mRecyclerView.addItemDecoration(this);
    this.mRecyclerView.addOnItemTouchListener(this.mOnItemTouchListener);
    this.mRecyclerView.addOnChildAttachStateChangeListener(this);
    this.startGestureDetection();
}
```

addItemDecoration：添加装饰者，用来画分割线的，但是这个不能说就是分割线。

addOnItemTouchListener：设置触摸监听，用于响应用户的触摸操作

startGestureDetection：开始手势检测

执行以上操作之后用户进行触摸操作之后，系统就能检测到了



**下面来看OnItemTouchListener 这个类的实现**

```java
private final OnItemTouchListener mOnItemTouchListener = new OnItemTouchListener() {
 //事件拦截
 @Override
 public boolean onInterceptTouchEvent(@NonNull RecyclerView recyclerView,
                                      @NonNull MotionEvent event) {
  mGestureDetector.onTouchEvent(event);
       ...
  return mSelected != null;
 }

 //响应触摸事件
 @Override
 public void onTouchEvent(@NonNull RecyclerView recyclerView, @NonNull MotionEvent event) {
  mGestureDetector.onTouchEvent(event);
  if (DEBUG) {
   Log.d(TAG,
           "on touch: x:" + mInitialTouchX + ",y:" + mInitialTouchY + ", :" + event);
  }
       ...
  ViewHolder viewHolder = mSelected;
  if (viewHolder == null) {
   return;
  }
  switch (action) {
   case MotionEvent.ACTION_MOVE: { //移动view  滑动的距离
    // Find the index of the active pointer and fetch its position
    if (activePointerIndex >= 0) {
     updateDxDy(event, mSelectedFlags, activePointerIndex);  //计算滑动的距离
     moveIfNecessary(viewHolder);        //计算偏移量查看是否需要移动
     mRecyclerView.removeCallbacks(mScrollRunnable);
     mScrollRunnable.run();			//递归循环触发动画
     mRecyclerView.invalidate();  //触发RecyclerView Ondraw
    }
    break;
   }
   case MotionEvent.ACTION_CANCEL:
    if (mVelocityTracker != null) {
     mVelocityTracker.clear();
    }
    // fall through
   case MotionEvent.ACTION_UP:
    select(null, ACTION_STATE_IDLE); //触摸停止
    mActivePointerId = ACTIVE_POINTER_ID_NONE;
    break;
   case MotionEvent.ACTION_POINTER_UP: {
              ...
   }
  }
 }

 @Override
 public void onRequestDisallowInterceptTouchEvent(boolean disallowIntercept) {
  if (!disallowIntercept) {
   return;
  }
  select(null, ACTION_STATE_IDLE);
 }
};
```

上面是Touch事件的代码逻辑，在 MotionEvent.ACTION_MOVE：

 updateDxDy(event, mSelectedFlags, activePointerIndex);  //计算滑动的距离  

 moveIfNecessary(viewHolder);        //计算偏移量查看是否需要移动

 mScrollRunnable.run();			//递归循环触发动画

mRecyclerView.invalidate();  //触发RecyclerView Ondraw



**updateDxDy由于这个在自定义View的时候会经常遇到，所以把这部分粘贴出来**

```java
private void updateDxDy(MotionEvent ev, int directionFlags, int pointerIndex) {
 final float x = ev.getX(pointerIndex);
 final float y = ev.getY(pointerIndex);

 // Calculate the distance moved
 mDx = x - mInitialTouchX;
 mDy = y - mInitialTouchY;
 if ((directionFlags & LEFT) == 0) {
  mDx = Math.max(0, mDx);
 }
 if ((directionFlags & RIGHT) == 0) {
  mDx = Math.min(0, mDx);
 }
 if ((directionFlags & UP) == 0) {
  mDy = Math.max(0, mDy);
 }
 if ((directionFlags & DOWN) == 0) {
  mDy = Math.min(0, mDy);
 }
}
```

这个是经典的处理方式



```java
/**
 *当用户将视图拖到边缘时，只要视图部分超出边界，我们就会开始滚动 LayoutManager。
 *  When user drags a view to the edge, we start scrolling the LayoutManager as long as View
 *  is partially out of bounds.
 */
private final Runnable mScrollRunnable = new Runnable() {
 @Override
 public void run() {
  if (mSelected != null && scrollIfNecessary()) {
   if (mSelected != null) { //it might be lost during scrolling
    moveIfNecessary(mSelected);
   }
   mRecyclerView.removeCallbacks(mScrollRunnable);
   ViewCompat.postOnAnimation(mRecyclerView, this);  //动画的递归调用
  }
 }
};
```

这个是move的时候执行的Runnable，这个主要是递归动画的



**下面来看RecyclerView 的onDraw方法**

```java
@Override
public void onDraw(Canvas c) {
 super.onDraw(c);

 final int count = mItemDecorations.size();
 for (int i = 0; i < count; i++) {
  mItemDecorations.get(i).onDraw(c, this, mState);
 }
}
```

这个调用了item装饰的onDraw方法



ItemDecoration 条目装饰着，这里使用到了装饰者设计模式，不仅仅是用来画分割线，可以绘制任何

```java
 public abstract static class ItemDecoration {
  /**
   *   在提供给 RecyclerView 的 Canvas 中绘制任何适当的装饰。通过此方法绘制的任何内容都将在绘制项目		视图之前绘制，因此将出现在视图下方。
   * Draw any appropriate decorations into the Canvas supplied to the RecyclerView.
   * Any content drawn by this method will be drawn before the item views are drawn,
   * and will thus appear underneath the views.
   *
   */
  public void onDraw(@NonNull Canvas c, @NonNull RecyclerView parent, @NonNull State state) {
   onDraw(c, parent);
  }
     
 ...
     
}
```

这里是抽象类，实现类是OnItemHelper



**上面的是抽象类，来看实现类ItemTouchHelper的onDraw方法**

```java
@Override
public void onDraw(Canvas c, RecyclerView parent, RecyclerView.State state) {
 // we don't know if RV changed something so we should invalidate this index.
 mOverdrawChildPosition = -1;
 float dx = 0, dy = 0;
 if (mSelected != null) {
  getSelectedDxDy(mTmpPosition);
  dx = mTmpPosition[0];
  dy = mTmpPosition[1];
 }
 mCallback.onDraw(c, parent, mSelected,
         mRecoverAnimations, mActionState, dx, dy);
}
```

执行了Callback的onDraw方法



**这里是Callback的onDraw方法**

```java
void onDraw(Canvas c, RecyclerView parent, ViewHolder selected,
            List<ItemTouchHelper.RecoverAnimation> recoverAnimationList,
            int actionState, float dX, float dY) {
 final int recoverAnimSize = recoverAnimationList.size();
 for (int i = 0; i < recoverAnimSize; i++) {
  final ItemTouchHelper.RecoverAnimation anim = recoverAnimationList.get(i);
  anim.update();
  final int count = c.save();
  onChildDraw(c, parent, anim.mViewHolder, anim.mX, anim.mY, anim.mActionState,
          false);
  c.restoreToCount(count);
 }
 if (selected != null) {
  final int count = c.save();
  onChildDraw(c, parent, selected, dX, dY, actionState, true);
  c.restoreToCount(count);
 }
}
```

主要调用了onChildDraw这个方法



**ItemTouchHelper的onChildDraw方法**

```java
/**
 * Called by ItemTouchHelper on RecyclerView's onDraw callback.
 * 由 ItemTouchHelper 在 RecyclerView 的 onDraw 回调中调用。

 */
public void onChildDraw(@NonNull Canvas c, @NonNull RecyclerView recyclerView,
                        @NonNull ViewHolder viewHolder,
                        float dX, float dY, int actionState, boolean isCurrentlyActive) {
 ItemTouchUIUtilImpl.INSTANCE.onDraw(c, recyclerView, viewHolder.itemView, dX, dY,
         actionState, isCurrentlyActive);
}
```

重点来了，ItemTouchUIUtilImpl的onDraw这个才是真正处理滑动跟随操作的逻辑。





**ItemTouchUIUtilImpl的onDraw**

```java
class ItemTouchUIUtilImpl implements ItemTouchUIUtil {
 static final ItemTouchUIUtil INSTANCE =  new ItemTouchUIUtilImpl();

 @Override
 public void onDraw(Canvas c, RecyclerView recyclerView, View view, float dX, float dY,
                    int actionState, boolean isCurrentlyActive) {
  if (Build.VERSION.SDK_INT >= 21) {
   if (isCurrentlyActive) {
    Object originalElevation = view.getTag(R.id.item_touch_helper_previous_elevation);
    if (originalElevation == null) {
     originalElevation = ViewCompat.getElevation(view);
     float newElevation = 1f + findMaxElevation(recyclerView, view);
     ViewCompat.setElevation(view, newElevation);
     view.setTag(R.id.item_touch_helper_previous_elevation, originalElevation);
    }
   }
  }
  //这里是跟随手指移动的实现
  view.setTranslationX(dX);
  view.setTranslationY(dY);
 }
```

那么上面提到的手指按下Item，并滑动的实现就在这里处理的，就是在onDraw里边执行了setTranslation操作。



**2.那么手指抬起的动画又是在哪里处理的呢？继续回到onTouchEvent的MotionEvent.ACTION_UP**

```java
  case MotionEvent.ACTION_UP:
	select(null, ACTION_STATE_IDLE); //触摸停止
	mActivePointerId = ACTIVE_POINTER_ID_NONE;
  break;
```

触摸停止只调用了select方法



**ItemTouchUIUtilImpl的select方法**

```java
/**
 * 开始拖动或滑动给定的视图。如果要清除它，请使用 null 调用。
 * Starts dragging or swiping the given View. Call with null if you want to clear it.
 */
private void select(@Nullable ViewHolder selected, int actionState) {
 if (selected == mSelected && actionState == mActionState) {
  return;
 }
 mDragScrollStartTimeInMs = Long.MIN_VALUE;
 final int prevActionState = mActionState;
  ...
 if (mSelected != null) {
  final ViewHolder prevSelected = mSelected;
  if (prevSelected.itemView.getParent() != null) {
   final int swipeDir = prevActionState == ACTION_STATE_DRAG ? 0
           : swipeIfNecessary(prevSelected);
   releaseVelocityTracker();
   // find where we should animate to
   final float targetTranslateX, targetTranslateY;
   int animationType;
   switch (swipeDir) {
    case LEFT:
    case RIGHT:
    case START:
    case END:
     targetTranslateY = 0;
     targetTranslateX = Math.signum(mDx) * mRecyclerView.getWidth();
     break;
    case UP:
    case DOWN:
     targetTranslateX = 0;
     targetTranslateY = Math.signum(mDy) * mRecyclerView.getHeight();
     break;
    default:
     targetTranslateX = 0;
     targetTranslateY = 0;
   }
   if (prevActionState == ACTION_STATE_DRAG) {
    animationType = ANIMATION_TYPE_DRAG;
   } else if (swipeDir > 0) {
    animationType = ANIMATION_TYPE_SWIPE_SUCCESS;
   } else {
    animationType = ANIMATION_TYPE_SWIPE_CANCEL;
   }
   getSelectedDxDy(mTmpPosition);
   final float currentTranslateX = mTmpPosition[0];
   final float currentTranslateY = mTmpPosition[1];
   final RecoverAnimation rv = new RecoverAnimation(prevSelected, animationType,
           prevActionState, currentTranslateX, currentTranslateY,
           targetTranslateX, targetTranslateY) {
    @Override
    public void onAnimationEnd(Animator animation) {
     super.onAnimationEnd(animation);
     if (this.mOverridden) {
      return;
     }
     if (swipeDir <= 0) {
      // this is a drag or failed swipe. recover immediately
      mCallback.clearView(mRecyclerView, prevSelected); //清除
      // full cleanup will happen on onDrawOver
     } else {
      // wait until remove animation is complete.
      mPendingCleanup.add(prevSelected.itemView);
      mIsPendingCleanup = true;
      if (swipeDir > 0) {
       // Animation might be ended by other animators during a layout.
       // We defer callback to avoid editing adapter during a layout.
       postDispatchSwipe(this, swipeDir);
      }
     }
     // removed from the list after it is drawn for the last time
     if (mOverdrawChild == prevSelected.itemView) {
      removeChildDrawingOrderCallbackIfNecessary(prevSelected.itemView);
     }
    }
   };
   //得到动画的时间  同时暴露出去，用户可以设置时间来控制动画的时间
   final long duration = mCallback.getAnimationDuration(mRecyclerView, animationType,
           targetTranslateX - currentTranslateX, targetTranslateY - currentTranslateY);
   rv.setDuration(duration);
   mRecoverAnimations.add(rv);
   rv.start();
   preventLayout = true;
  } else {
   removeChildDrawingOrderCallbackIfNecessary(prevSelected.itemView);
   mCallback.clearView(mRecyclerView, prevSelected);
  }
  mSelected = null;
 }
 ...
 mCallback.onSelectedChanged(mSelected, mActionState);
 mRecyclerView.invalidate();
}
```

在这个里边看到了RecoverAnimation，系统就是通过这个方法来执行动画的

* getAnimationDuration：这个方法用来获取动画执行的时间，我们可以重写这个方法，并返回一个时间值，那么动画的执行时间就被改变了，当然不建议这样做。

* postDispatchSwipe(this, swipeDir) 执行动画结束回调了这个方法，这个里边又执行了

* mCallback.onSwiped(anim.mViewHolder, swipeDir);   //删除动画做完了，需要做数据的删除了

* mCallback.onSelectedChanged(mSelected, mActionState);  //这个是选择的发生变化的回调方法，同样这个方法也回调给了用户。



**总结通过分析源码：**

**1.了解了ItemTouchHelper的整个执行的流程。**

**2.明白了Item跟随手指滑动的显示方式是：在onDraw方法中setTranslation来实现的**

**3.滑动距离的计算方法**

**4.学到了装饰者的应用ItemDecoration**

**5.通过getAnimationDuration这个方法可以修改抬手的动画执行时间**

**6.了解到了onSwiped() 、onSelectedChanged()方法的调用时间点**