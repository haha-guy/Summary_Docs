##TextView控件实现原理分析
1. 应用程序窗口,即Activity窗口,是由一个PhoneWindow对象,一个DecorView对象,以及一个ViewRoot对象来描述的。PhoneWindow对象用来描述窗口对象,DecorView对象用来描述窗口的顶层视图,ViewRoot对象除了用来和WindowManagerService服务通信之外,还用来接收用户输入。窗口控件本身也是一个视图,即一个View对象,它们是以树形结构组织在一起形成整个窗口的UI的。

###TextView控件的UI绘制框架
1. 测量：完成控件本身大小的确认
	为了能告诉父视图自己的所占据的空间的大小,所有控件都必须要重写父类View的成员函数onMeasure。
```Java
    public class TextView extends View implements ViewTreeObserver.OnPreDrawListener {  
        ......  
      
        @Override  
        protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {  
            int widthMode = MeasureSpec.getMode(widthMeasureSpec);  
            int heightMode = MeasureSpec.getMode(heightMeasureSpec);  
            int widthSize = MeasureSpec.getSize(widthMeasureSpec);  
            int heightSize = MeasureSpec.getSize(heightMeasureSpec);  
      
            int width;  
            int height;  
      
            //计算TextView控件的宽度和高度  
            ......  
      		/*必须要调用从父类View继承下来的成员函数setMeasuredDimension来通知父视图它所要设置的宽度和高度*/
            setMeasuredDimension(width, height);  
        }  
      
        ......
    }
```
参数widthMeasureSpec和heightMeasureSpec分别用来描述宽度测量规范和高度测量规范。测量规范使用一个int值来表法，这个int值包含了两个分量。
第一个是mode分量，使用最高2位来表示。测量模式有三种，分别是MeasureSpec.UNSPECIFIED（0）、MeasureSpec.EXACTLY（1）、和MeasureSpec.AT_MOST（2）。
第二个是size分量，使用低30位来表示。当mode分量等于MeasureSpec.EXACTLY时，size分量的值就是父视图要求当前控件要设置的宽度或者高度；当mode分量等于MeasureSpec.AT_MOST时，size分量的值就是父视图限定当前控件可以设置的最大宽度或者高度；当mode分量等于MeasureSpec.UNSPECIFIED时，父视图不限定当前控件所设置的宽度或者高度，这时候当前控件一般就按照实际需求来设置自己的宽度和高度。

2. 布局：完成控件位置的确认
控件是按照树形结构组织在一起的，其中，子控件的位置由父控件来设置，也就是说，只有容器类控件才需要执行布局操作，这是通过重写父类View的成员函数onLayout来实现的。从Activity窗口的结构可以知道，它的顶层视图是一个DecorView，这是一个容器类控件。Activity窗口的布局操作就是从其顶层视图开始执行的，每碰到一个容器类的子控件，就调用它的成员函数onLayout来让它有机会对自己的子控件的位置进行设置，依次类推。

3. 绘制：将控件在显示屏幕上显示出来
控件为了能够绘制自己的UI，必须要重写父类View的成员函数onDraw。
TextView类的成员函数onDraw的实现如下所示：
```java
    public class TextView extends View implements ViewTreeObserver.OnPreDrawListener {  
        ......

        @Override  
        protected void onDraw(Canvas canvas) {  
            //在画布canvas上绘制UI  
            ......  
        }  
      
        ......  
    }  
```
Java层的Canvas实际上是封装了C++层的SkCanvas。C++层的SkCanvas内部有一块图形缓冲区，这块图形缓冲区就是窗口的绘图表面（Surface）里面的那块图形缓冲区。窗口的绘图表面里面的那块图形缓冲区实际上是一块匿名共享内存，它是SurfaceFlinger服务负责创建的。SurfaceFlinger服务创建完成这块匿名共享内存之后，就会将其返回给窗口所运行在的进程。窗口所运行在的进程获得了这块匿名共享内存之后，就会映射到自己的进程空间来，因此，窗口的控件就可以在本进程内访问这块匿名共享内存了，实际上就是往这块匿名共享内存填入UI数据。注意，这个过程执行完成之后，控件的UI还没有反映到屏幕上来，因为这时候将控件的UI数据填入到图形缓冲区而已。窗口的UI的显示是WindowManagerService服务来控制的。因此，当窗口的所有控件都绘制完成自己的UI之后，窗口就会向WindowManagerService服务发送一个Binder进程间程通信请求。WindowManagerService服务接收到这个Binder进程间程通信请求之后，就会请求SurfaceFlinger服务刷新相应的窗口的UI。
以下两个结论：
        (1). 一个窗口的所有控件的UI都是绘制在窗口的绘图表面上的，也就是说，一个窗口的所有控件的UI数据都是填写在同一块图形缓冲区中；

        (2). 一个窗口的所有控件的UI的绘制操作是在主线程中执行的，事实上，所有与UI相关的操作都是必须是要在主线程中执行，否则的话，就会抛出一个类型为CalledFromWrongThreadException的异常来。

###TextView控件获取键盘输入的过程分析
1. 每一个窗口的创建的时候，都会与系统的输入管理器建立一个用户输入接收通道。输入管理器在启动两个线程，其中一个用来监控用户输入，即监控用户是否按下或者放开了键盘按键，或者是否触摸了屏幕，另外一个用来将监控到的用户输入事件分发给当前激活的窗口来处理，而这个分发过程就是通过前面建立的通道来进行的。
2. 当前激活的窗口接收到输入管理器分发过来的用户输入事件之后，就会该事件封装成一个消息发送到当前激活的窗口所运行在的应用程序进程的主线程的消息队列中去。等到这个消息被处理的时候，就会调用与当前激活的窗口所关联的一个ViewRoot对象的成员函数deliverKeyEvent或者deliverPointerEvent来将前面接收到的用户输入分发给合适的控件。其中，ViewRoot类的成员函数deliverKeyEvent负责分发键盘输入事件，而ViewRoot类的成员函数deliverPointerEvent负责分发触摸屏输入事件。
3. TextView控件获得键盘输入的过程
	- ViewRoot.deliverKeyEvent
```java
    public final class ViewRoot extends Handler implements ViewParent,  
            View.AttachInfo.Callbacks {  
        ......  
      
        private void deliverKeyEvent(KeyEvent event, boolean sendDone) {  
/*参数event描述的是窗口接收到的键盘事件，另外一个参数sendDone表示该键盘事件处理完成后，是否需要向系统的输入管理器发送一个通知。*/
			// If mView is null, we just consume the key event because it doesn't  
            // make sense to do anything else with it.
/*ViewRoot类的成员函数deliverKeyEvent首先是调用它的成员函数dispatchKeyEventPreIme来让它优先于输入法处理参数event所描述的键盘事件。如果这个DecorView对象的成员函数dispatchKeyEventPreIme的返回值handled等于true，那么就说明参数event所描述的键盘事件已经处理完毕，即ViewRoot类的成员函数deliverKeyEvent不用往下执行了。在这种情况下，如果参数sendDone的值等于true，那么ViewRoot类的成员函数deliverKeyEvent在返回之前，还会调用成员函数finishInputEvent来通知系统的输入管理器，当前激活的窗口已经处理完成刚刚发生的键盘事件了*/
            boolean handled = mView != null  
                    ? mView.dispatchKey% s/\s\+$//gEventPreIme(event) : true;  
            if (handled) {  
                if (sendDone) {  
                    finishInputEvent();  
                }  
                return;  
            }  
            // If it is possible for this window to interact with the input  
            // method window, then we want to first dispatch our key events  
            // to the input method.
/*只有当前窗口正在显示输入法的情况下，ViewRoot类的成员函数deliverKeyEvent才会将参数event所描述的键盘事件分发给输入法处理，这是通过检查ViewRoot类的成员变量mLastWasImTarget的值是否等于true来确定的。*/
            if (mLastWasImTarget) {
/*1. 调用InputMethodManager类的静态成员函数peekInstance获得一个类型为InputMethodManager输入法管理器imm；*/
                InputMethodManager imm = InputMethodManager.peekInstance();  
                if (imm != null && mView != null) {
/*调用ViewRoot类的成员函数enqueuePendingEvent将参数event所描述的键盘事件缓存起来，等到输入法处理完成该键盘事件之后，再继续对它进行处理；*/
                    int seq = enqueuePendingEvent(event, sendDone);  
                    ......
/*3. 调用第1步获得的输入法管理器imm的成员函数dispatchKeyEvent来将参数event所描述的键盘事件分发给输入法处理。*/
                    imm.dispatchKeyEvent(mView.getContext(), seq, event,  
                            mInputMethodCallback);/*在将参数event所描述的键盘事件分发给输入法处理时，ViewRoot类的成员函数deliverKeyEvent会同时传递一个类型为InputMethodCallback的回调接口给输入法，以便输入法处理完成参数event所描述的键盘事件之后，可以调用这个回调接口的成员函数finishedEvent来向窗口发送一个键盘事件处理完成通知*/
                    return;
                }
            }  
            deliverKeyEventToViewHierarchy(event, sendDone);  
        }  
      
        ......  
    }  
```
	- Step 2. ViewGroup.dispatchKeyEventPreIme
将获得的键盘输入优先于输入法分发给窗口处理的。
```java
    public abstract class ViewGroup extends View implements ViewParent, ViewManager {  
        ......  
      
        // The view contained within this ViewGroup that has or contains focus.  
        private View mFocused;  
        ......  
      
        @Override  
        public boolean dispatchKeyEventPreIme(KeyEvent event) {  
/*ViewGroup类的成员函数dispatchKeyEventPreIme首先是检查当前正在处理的视图容器是否能够获得焦点。如果能够获得焦点的话，那么ViewGroup类的成员变量mPrivateFlags的FOCUSED位就会等于1。在当前正在处理的视图容器能够获得焦点的情况下，还要检查正在处理的视图容器是否已经计算过大小了，即检查ViewGroup类的成员变量mPrivateFlags的HAS_BOUNDS位是否等于1。只有在已经计算过大小并且能够获得焦点的情况下，那么正在处理的视图容器才有资格处理参数event所描述的键盘事件。*/
			if ((mPrivateFlags & (FOCUSED | HAS_BOUNDS)) == (FOCUSED | HAS_BOUNDS)) {  
                return super.dispatchKeyEventPreIme(event);  
            } else if (mFocused != null && (mFocused.mPrivateFlags & HAS_BOUNDS) == HAS_BOUNDS) {
/*如果当前正在处理的视图容器没有资格处理参数event所描述的键盘事件，但是它有一个能够获得焦点的子视图，并且这个子视图的大小也是已经计算好了的，那么ViewGroup类的成员函数dispatchKeyEventPreIme就会将参数event所描述的键盘事件分发给该子视图处理。当前正在处理的视图容器能够获得焦点的子视图是通过ViewGroup类的成员变量mFocused来描述的。一个视图容器是如何知道它的焦点子视图的呢？我们知道，当我们在屏幕上触摸一个窗口时，就会发生一个Pointer事件。这个Pointer事件关联有一个触摸点，通过检查这个触摸点当前是包含在窗口顶层视图的哪一个子视图里面，就可以知道哪一个子视图是焦点子视图了。*/
                return mFocused.dispatchKeyEventPreIme(event);  
            }  
            return false;  
        }  
      
        ......  
    }  
```
	- Step 3. View.dispatchKeyEventPreIme
```java
    public class View implements Drawable.Callback, KeyEvent.Callback, AccessibilityEventSource {  
        ......  
      
        public boolean dispatchKeyEventPreIme(KeyEvent event) {  
            return onKeyPreIme(event.getKeyCode(), event);  
        }  
      
        ......  
    }  
```
	- Step 4. View.onKeyPreIme
```java
    public class View implements Drawable.Callback, KeyEvent.Callback, AccessibilityEventSource {  
        ......  
      
        public boolean onKeyPreIme(int keyCode, KeyEvent event) {  
            return false;  
        }  
      
        ......  
    }  
```
View类的成员函数onKeyPreIme默认是不会在输入法之前处理参数event所描述的键盘事件的，因此，我们在实现自己的控件的时候，如果需要在输入法之前处理键盘输入，那么就必须重写父类View的成员函数onKeyPreIme。在重写父类View的成员函数onKeyPreIme来处理一个键盘事件的时候，如果不希望这个键盘事件分发给输入法处理，那么就返回一个true值，否则的话，就返回一个false值。
	- Step 5. ViewRoot.deliverKeyEventToViewHierarchy
回到前面的Step 1中，即ViewRoot类的成员函数deliverKeyEvent中，无论接下来是否需要先将一个键盘事件分发给输入法处理，最终都会调用到ViewRoot类的成员函数deliverKeyEventToViewHierarchy来继续将该键盘事件分发给当前激活的窗口处理。
```java
    public final class ViewRoot extends Handler implements ViewParent,  
            View.AttachInfo.Callbacks {  
        ......  
      
        private void deliverKeyEventToViewHierarchy(KeyEvent event, boolean sendDone) {  
            try {  
                if (mView != null && mAdded) {  
                    final int action = event.getAction();  
                    boolean isDown = (action == KeyEvent.ACTION_DOWN);  
                    ......  
/*ViewRoot类的成员函数deliverKeyEventToViewHierarchy首先将参数event所描述的键盘事件交给当前激活的窗口的顶层视图来处理，这是通过调用ViewRoot类的成员变量mView所描述的一个DecorView对象的成员函数dispatchKeyEvent来实现的。如果当前激活的窗口的顶层视图在处理完成参数event所描述的键盘事件之后，希望该键盘事件还能继续被ViewRoot类的成员函数deliverKeyEventToViewHierarchy处理，那么前面调用DecorView类的成员函数dispatchKeyEvent得到的返回值keyHandled的值就会等于false。在这种情况下，如果参数event描述的是一个按下的键盘事件，即变量isDown的值等于true，那么ViewRoot类的成员函数deliverKeyEventToViewHierarchy就会继续检查参数event描述的是否是一个DPAD事件。如果是的话，那么就可能需要改变窗口当前的焦点子视图。*/
                    boolean keyHandled = mView.dispatchKeyEvent(event);
/* 如果参数event描述的是一个DPAD事件，那么最终得到的变量direction的值就不会等于0，并且它描述的是当前按下的是哪一个方向的DPAD键。假设这时候窗口已经有一个焦点子视图，即调用ViewRoot类的成员变量mView所描述的一个DecorView对象的成员函数findFocus的返回值focused不等于null，那么接下来就要根据变量direction的值来决定下一个焦点子视图是谁。例如，假设变量direction的值等于View.FOCUS_LEFT，那么就表示在当前的焦点子视图focused的左边查找一个最靠近的子视图作为下一个焦点子视图，这是通过调用当前焦点子视图focused的成员函数focusSearch来实现的。一旦找到了下一个焦点子视图v，并且该子视图不是当前的焦点子视图focused，那么ViewRoot类的成员函数deliverKeyEventToViewHierarchy就需要将子视图v设置为焦点子视图，这是通过调用变量v所描述的一个View对象的成员函数requestFocus来实现的。*/
                    if (!keyHandled && isDown) {  
                        int direction = 0;  
                        switch (event.getKeyCode()) {  
                        case KeyEvent.KEYCODE_DPAD_LEFT:  
                            direction = View.FOCUS_LEFT;  
                            break;  
                        case KeyEvent.KEYCODE_DPAD_RIGHT:  
                            direction = View.FOCUS_RIGHT;  
                            break;  
                        case KeyEvent.KEYCODE_DPAD_UP:  
                            direction = View.FOCUS_UP;  
                            break;  
                        case KeyEvent.KEYCODE_DPAD_DOWN:  
                            direction = View.FOCUS_DOWN;  
                            break;  
                        }  
      
                        if (direction != 0) {  
      
                            View focused = mView != null ? mView.findFocus() : null;  
                            if (focused != null) {  
                                View v = focused.focusSearch(direction);  
                                ......  	
      
                                if (v != null && v != focused) {  
                                    ......  
      
                                    focusPassed = v.requestFocus(direction, mTempRect);  
                                }  
      
                                ......  
                            }  
                        }  
                    }  
                }  
      
            } finally {  
                if (sendDone) {  
                    finishInputEvent();  
                }  
                ......  
            }  
        }  
      
        ......  
    }  
```
	-  Step 6. DecorView.dispatchKeyEvent
```java
public class PhoneWindow extends Window implements MenuBuilder.Callback {  
    ......  
  
    private final class DecorView extends FrameLayout implements RootViewSurfaceTaker {  
        ......  
        @Override  
        public boolean dispatchKeyEvent(KeyEvent event) {  
            final int keyCode = event.getKeyCode();  
            final boolean isDown = event.getAction() == KeyEvent.ACTION_DOWN;  
            ......  
 /* PhoneWindow类的成员函数getCallback是从父类Window继承下来的，它返回的是一个Window.Callback接口。每一个Activity组件都会实现一个Window.Callback接口，并且将这个Window.Callback接口设置到与它所关联的一个PhoneWindow对象的内部去，这样当该PhoneWindow对象接收到键盘事件的时候，就可以将该键盘事件分发给与它所关联的Activity组件处理。 DecorView类的成员变量mFeatureId用来描述当前正在处理的DecorView对象的特征，当它的值小于0的时候，就表示当前正在处理的一个DecorView对象是用来描述一个Activity组件窗口的顶层视图的。*/ 
            final Callback cb = getCallback();  
            final boolean handled = cb != null && mFeatureId < 0 ? cb.dispatchKeyEvent(event)  
                    : super.dispatchKeyEvent(event);  
            if (handled) {  
                return true;  
            }  
            return isDown ? PhoneWindow.this.onKeyDown(mFeatureId, event.getKeyCode(), event)  
                    : PhoneWindow.this.onKeyUp(mFeatureId, event.getKeyCode(), event);  
        }  
  
        ......  
  
    }  
  
    ......  
}
```
	- Activity.dispatchKeyEvent
```java
public class Activity extends ContextThemeWrapper  
        implements LayoutInflater.Factory,  
        Window.Callback, KeyEvent.Callback,  
        OnCreateContextMenuListener, ComponentCallbacks {  
    ......  
  
    public boolean dispatchKeyEvent(KeyEvent event) {  
        ......  
/*Activity类的成员函数getWindow返回的是与当前正处理的Activity组件所关联的一个PhoneWindow对象*/
        Window win = getWindow();  
        if (win.superDispatchKeyEvent(event)) {  
            return true;  
        }
        View decor = mDecor;  
        if (decor == null) decor = win.getDecorView();
/*在调用event所描述的一个KeyEvent对象的成员函数dispatch的时候，第一个参数指定为当前正在处理的Activity组件所实现的一个KeyEvent.Callback接口。参数event所指向的一个KeyEvent对象的成员函数dispatch的执行的过程中，就会相应地调用这个KeyEvent.Callback接口的成员函数onKeyDown、onKeyUp或者onKeyMultiple来处理它所描述的键盘事件，实际上就是调用Activity类的成员函数onKeyDown、onKeyUp或者onKeyMultiple来处理参数event所描述的键盘事件。因此，我们在自定义一个Activity组件时，如果需要处理分发给该Activity组件的键盘事件，那么就需要重写父类Activity的成员函数onKeyDown、onKeyUp或者onKeyMultiple。*/
        return event.dispatch(this, decor != null  
                ? decor.getKeyDispatcherState() : null, this);  
    }  
  
    ......  
}
```
	- Step 8. PhoneWindow.superDispatchKeyEvent
```java
    public class PhoneWindow extends Window implements MenuBuilder.Callback {  
        ......  
      
        // This is the top-level view of the window, containing the window decor.  
        private DecorView mDecor;  
        ......  
      
        @Override  
        public boolean superDispatchKeyEvent(KeyEvent event) {  
            return mDecor.superDispatchKeyEvent(event);  
        }  
      
        ......  
    }  
```
	-  Step 9. DecorView.superDispatchKeyEvent
```java
public class PhoneWindow extends Window implements MenuBuilder.Callback {  
    ......  
  
    private final class DecorView extends FrameLayout implements RootViewSurfaceTaker {  
        ......  
  
        public boolean superDispatchKeyEvent(KeyEvent event) {  
            return super.dispatchKeyEvent(event);  
        }  
       
        ......
    }
    ......
}
```
	- Step 10. ViewGroup.dispatchKeyEvent
```java
public abstract class ViewGroup extends View implements ViewParent, ViewManager {  
    ......  
  
    @Override  
    public boolean dispatchKeyEvent(KeyEvent event) {  
        if ((mPrivateFlags & (FOCUSED | HAS_BOUNDS)) == (FOCUSED | HAS_BOUNDS)) {  
            return super.dispatchKeyEvent(event);  
        } else if (mFocused != null && (mFocused.mPrivateFlags & HAS_BOUNDS) == HAS_BOUNDS) {  
            return mFocused.dispatchKeyEvent(event);  
        }  
        return false;  
    }  
  
    ......  
}
```
	-  Step 11. View.dispatchKeyEvent
```java
public class View implements Drawable.Callback, KeyEvent.Callback, AccessibilityEventSource {  
    ......  
  
    private OnKeyListener mOnKeyListener;  
    ......  
  
    public boolean dispatchKeyEvent(KeyEvent event) {  
        // If any attached key listener a first crack at the event.  
        //noinspection SimplifiableIfStatement  
  
        ......  
/*当View类的成员变量mOnKeyListener的值不等于null时，它所指向的一个OnKeyListener对象描述的是注册到当前正在处理的视图的一个键盘事件监听器,监听器在处理完成参数event所描述的键盘事件之后，如果希望该键盘事件还能继续往下处理，那么View类的成员函数dispatchKeyEvent就会继续调用参数event所指向的一个KeyEvent对象的成员函数dispatch来处理该键盘事件。*/
        if (mOnKeyListener != null && (mViewFlags & ENABLED_MASK) == ENABLED  
                && mOnKeyListener.onKey(this, event.getKeyCode(), event)) {  
            return true;  
        }  
  
        return event.dispatch(this, mAttachInfo != null  
                ? mAttachInfo.mKeyDispatchState : null, this);  
    }  
  
    ......  
}
```
	- Step 12. KeyEvent.dispatch
```java
public class KeyEvent extends InputEvent implements Parcelable {    
    ......    
/* 由于前面我们已经假设了当前获得焦点的是一个TextView控件，因此，参数receiver指向的实际上是一个TextView对象。TextView类重写了父类View的成员函数onKeyDown，因此，接下来KeyEvent类的成员函数dispatch就会调用TextView类的成员函数onKeyDown来接收当前正在处理的键盘事件。*/
    public final boolean dispatch(Callback receiver, DispatcherState state,    
            Object target) {    
        switch (mAction) {    
        case ACTION_DOWN: {    
            ......
/*TextView类的成员函数onKeyDown调用另外一个成员函数doKeyDown来处理参数event所描述的键盘事件，以便可以相应地改变当前获得焦点的TextView控件的UI。*/
            boolean res = receiver.onKeyDown(mKeyCode, this);    
            ......    
            return res;    
        }    
        case ACTION_UP:    
            ......    
            return receiver.onKeyUp(mKeyCode, this);    
        case ACTION_MULTIPLE:    
            final int count = mRepeatCount;    
            final int code = mKeyCode;    
            if (receiver.onKeyMultiple(code, count, this)) {    
                return true;    
            }    
            ......    
            return false;    
        }    
        return false;    
    }    
    
    ......    
}
```
















