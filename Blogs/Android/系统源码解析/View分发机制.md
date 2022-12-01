### ViewGroup

ViewGroup 是一组 View 的组合，在其内部有可能包含多个子 View，当手指触摸屏幕上时，手指所在的区域既能在 ViewGroup 显示范围内，也可能在其内部 View 控件上。

因此它内部的事件分发的重心是处理当前 Group 和子 View 之间的逻辑关系：

1. 当前 Group 是否需要拦截 touch 事件；

2. 是否需要将 touch 事件继续分发给子 View；

3. 如何将 touch 事件分发给子 View。

### View

View是一个单纯的控件，不能再细分，所以它的事件分发重点在于当前View如何处理消费这个touch事件，并根据相应的手势逻辑进行一系列效果展示 

1. 是否存在TouchListener
2. 是否自己接收处理touch事件（主要逻辑在onTouchEvent中）

### 事件分发核心 dispatchTouchEvent

```java
public boolean dispatchTouchEvent() {
    // 步骤1 检查当前ViewGroup是否需要拦截事件 调用onInterceptTouchEvent
    
    // 步骤2 将事件分发给子View
    
    // 步骤3 根据mFirstTouchTarget，再次分发事件
   
}
```

**步骤1：**

```java
final boolean intercepted;
// 注释1
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
```

注释1：

- 如果事件为DOWN事件，则调用onInterceptTouchEvent进行判断
- 或者mFirstTouchTarget不为null，代表已经有子View捕获这个事件，子View的dispatchTouchEvent返回true就是代表捕获了touch事件

**步骤2**

```java
if (!canceled && !intercepted) {
	   // 事件前提是DOWN事件
       if (actionMasked == MotionEvent.ACTION_DOWN
                || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
           // 遍历所有子View
                   for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = getAndVerifyPreorderedIndex(
                                    childrenCount, i, customOrder);
                            final View child = getAndVerifyPreorderedView(
                                    preorderedList, children, childIndex);
                       // 判断事件坐标是否在子 View 坐标范围内，并且子 View 并没有处在动画状态；
                            if (!child.canReceivePointerEvents()
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                continue;
                    }
                	// 调用 dispatchTransformedTouchEvent 方法将事件分发给子 View，如果子 View 捕获事件成功，则将 mFirstTouchTarget 赋值给子 View。
                    if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                    }
```

**步骤3**

```java
if (mFirstTouchTarget == null) {
    // mFirstTouchTarget为null，说明上述的事件分发并没有子View对事件进行捕获，此时调用传入child为null，最终调用super.dispatchTouchEvent方法，实际上最终会调用自身的 onTouchEvent 方法，进行处理 touch 事件
    handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
} else {
    // mFirstTouchTarget 不为 null，说明在上面步骤 2 中有子 View 对 touch 事件进行了捕获，则直接将当前以及后续的事件交给 mFirstTouchTarget 指向的 View 进行处理。
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
         }
```

