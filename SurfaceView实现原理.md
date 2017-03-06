#SurfaceView实现原理分析
1. 在Android系统中，有一种特殊的视图，称为SurfaceView，它拥有独立的绘图表面，即它不与其宿主窗口共享同一个绘图表面。
2. 一般来说，每一个窗口在SurfaceFlinger服务中都对应有一个Layer，用来描述它的绘图表面。对于那些具有SurfaceView的窗口来说，每一个SurfaceView在SurfaceFlinger服务中还对应有一个独立的Layer或者LayerBuffer，用来单独描述它的绘图表面，以区别于它的宿主窗口的绘图表面。
3. 我们假设在一个Activity窗口的视图结构中，除了有一个DecorView顶层视图之外，还有两个TextView控件，以及一个SurfaceView视图，这样该Activity窗口在SurfaceFlinger服务中就对应有两个Layer或者一个Layer的一个LayerBuffer：
![1363016714_1787.jpg](/home/xb/Desktop/ScreenShots/1363016714_1787.jpg)

4. 用来描述SurfaceView的Layer或者LayerBuffer的Z轴位置是小于用来其宿主Activity窗口的Layer的Z轴位置的，但是前者会在后者的上面挖一个“洞”出来，以便它的UI可以对用户可见。实际上，SurfaceView在其宿主Activity窗口上所挖的“洞”只不过是在其宿主Activity窗口上设置了一块透明区域。

##SurfaceView绘图表面创建过程
1. 尽管SurfaceView不与它的宿主窗口共享同一个绘图表面，但是它仍然是属于宿主窗口的视图结构的一个结点的，也就是说，SurfaceView仍然是会参与到宿主窗口的某些执行流程中去。而每当一个窗口需要刷新UI时，就会调用ViewRoot类的成员函数performTraversals。ViewRoot类的成员函数performTraversals在执行的过程中，如果发现当前窗口的绘图表面还没有创建，或者发现当前窗口的绘图表面已经失效了，那么就会请求WindowManagerService服务创建一个新的绘图表面，同时，它还会通过一系列的回调函数来让嵌入在窗口里面的SurfaceView有机会创建自己的绘图表面。
	- Step 1. ViewRoot.performTraversals
```java
public final class ViewRoot extends Handler implements ViewParent,  
        View.AttachInfo.Callbacks {  
    ......
    private void performTraversals() {  
	/*变量host与ViewRoot类的成员变量mView指向的是同一个DecorView对象，这个DecorView对象描述的当前窗口的顶层视图。变量attachInfo与ViewRoot类的成员变量mAttachInfo指向的是同一个AttachInfo对象。在Android系统中，每一个视图附加到它的宿主窗口的时候，都会获得一个AttachInfo对象，用来描述被附加的窗口的信息。变量viewVisibility描述的是当前窗口的可见性。变量viewVisibilityChanged描述的是当前窗口的可见性是否发生了变化。*/
        // cache mView since it is used so much below...  
        final View host = mView;  
        ......  
        final View.AttachInfo attachInfo = mAttachInfo;  
        final int viewVisibility = getHostVisibility();  
        boolean viewVisibilityChanged = mViewVisibility != viewVisibility  														|| mNewSurfaceNeeded;
        ......
	/*ViewRoot类的成员变量mFirst表示当前窗口是否是第一次被刷新UI。如果是的话，那么它的值就会等于true，说明当前窗口的绘图表面还未创建。在这种情况下，如果ViewRoot类的另外一个成员变量mAttached的值也等于true，那么就表示当前窗口还没有将它的各个子视图附加到它的上面来。这时候ViewRoot类的成员函数performTraversals就会从当前窗口的顶层视图开始，通知每一个子视图它要被附加到宿主窗口上去了，这是通过调用变量host所指向的一个DecorView对象的成员函数dispatchAttachedToWindow来实现的.SurfaceView的绘图表面也就是在disaptchAttachedToWindow函数被调用时创建的*/
        if (mFirst) {
            ......
            if (!mAttached) {  
                host.dispatchAttachedToWindow(attachInfo, 0);  
                mAttached = true;  
            }  
            ......  
        }  
        ......  
        if (viewVisibilityChanged) {  
            ......  
            host.dispatchWindowVisibilityChanged(viewVisibility);  
            ......  
        }  
        ......  
        mFirst = false;  
        ......  
    }
    ......  
}
```
	- Step 2. ViewGroup.dispatchAttachedToWindow
```java
public abstract class ViewGroup extends View implements ViewParent, ViewManager {  
    ......  
    // Child views of this ViewGroup  
/* ViewGroup类的成员变量mChildren保存的是当前正在处理的视图容器的子视图，而另外一个成员变量mChildrenCount保存的是这些子视图的数量。*/
    private View[] mChildren;  
    // Number of valid children in the mChildren array, the rest should be null or not  
    // considered as children  
    private int mChildrenCount;  
    ......  
    @Override  
    void dispatchAttachedToWindow(AttachInfo info, int visibility) {  
/*ViewGroup类的成员函数dispatchAttachedToWindow的实现很简单，它只是简单地调用当前正在处理的视图容器的每一个子视图的成员函数dispatchAttachedToWindow，以便可以通知这些子视图，它们被附加到宿主窗口上去了。
当前正在处理的视图容器即为当前正在处理的窗口的顶层视图，由于前面我们当前正在处理的窗口有一个SurfaceView，因此这一步就会调用到该SurfaceView的成员函数dispatchAttachedToWindow。由于SurfaceView类的成员函数dispatchAttachedToWindow是从父类View继承下来的*/
        super.dispatchAttachedToWindow(info, visibility);  
        visibility |= mViewFlags & VISIBILITY_MASK;  
        final int count = mChildrenCount;  
        final View[] children = mChildren;  
        for (int i = 0; i < count; i++) {  
            children[i].dispatchAttachedToWindow(info, visibility);  
        }  
    }  
    ......  
}
```
	- Step 3. View.dispatchAttachedToWindow
```java
    public class View implements Drawable.Callback, KeyEvent.Callback, AccessibilityEventSource {  
        ......  
      
        AttachInfo mAttachInfo;  
        ......  
      
        void dispatchAttachedToWindow(AttachInfo info, int visibility) {  
            //System.out.println("Attached! " + this);  
            mAttachInfo = info;  
            ......  
			/*调用SurfaceView类的onAttachedToWindow*/
            onAttachedToWindow();  
      
            ......  
        }  
      
        ......  
    }  
```
	- Step 4. SurfaceView.onAttachedToWindow
```java
    public class SurfaceView extends View {  
        ......  
      
        IWindowSession mSession;  
        ......  
      
        @Override  
        protected void onAttachedToWindow() {  
            super.onAttachedToWindow();  
			/*第一件事情是通知父视图，当前正在处理的SurfaceView需要在宿主窗口的绘图表面上挖一个洞，即需要在宿主窗口的绘图表面上设置一块透明区域。*/
            mParent.requestTransparentRegion(this);  
			/*第二件事情是调用从父类View继承下来的成员函数getWindowSession来获得一个实现了IWindowSession接口的Binder代理对象，并且将该Binder代理对象保存在SurfaceView类的成员变量mSession中。在Android系统中，每一个应用程序进程都有一个实现了IWindowSession接口的Binder代理对象，这个Binder代理对象是用来与WindowManagerService服务进行通信的，View类的成员函数getWindowSession返回的就是该Binder代理对象。*/
            mSession = getWindowSession();  
            ......  
        }  
      
        ......
		/*返回到前面的Step 1中，即ViewRoot类的成员函数performTraversals中，我们假设当前窗口的可见性发生了变化，那么接下来就会调用顶层视图的成员函数dispatchWindowVisibilityChanged，以便可以通知各个子视图，它的宿主窗口的可见性发生化了。*/
    }  
```
	- Step 5. ViewGroup.dispatchWindowVisibilityChanged
包含在当前窗口里面的一个SurfaceView的绘图表面的创建过程。
```java
    public abstract class ViewGroup extends View implements ViewParent, ViewManager {   
        ......  
      
        @Override  
        public void dispatchWindowVisibilityChanged(int visibility) {  
            super.dispatchWindowVisibilityChanged(visibility);  
            final int count = mChildrenCount;  
            final View[] children = mChildren;  
            for (int i = 0; i < count; i++) {  
                children[i].dispatchWindowVisibilityChanged(visibility);  
            }  
        }  
      
        ......  
    }  
```
	- Step 6. View.dispatchWindowVisibilityChanged
```java
    public class View implements Drawable.Callback, KeyEvent.Callback, AccessibilityEventSource {  
        ......  
      
        public void dispatchWindowVisibilityChanged(int visibility) {  
			/*调用SurfaceView对应的成员函数*/
            onWindowVisibilityChanged(visibility);  
        }  
        ......  
    }  
```
	- Step 7. SurfaceView.onWindowVisibilityChanged
```java
    public class SurfaceView extends View {  
        ......  
		/*mWindowVisibility表示SurfaceView的宿主窗口的可见性，mViewVisibility表示SurfaceView自身的可见性。只有当mWindowVisibility和mViewVisibility的值均等于true的时候，mRequestedVisible的值才为true，表示SurfaceView是可见的。*/
        boolean mRequestedVisible = false;  
        boolean mWindowVisibility = false;  
        boolean mViewVisibility = false;  
        .....  
      
        @Override  
        protected void onWindowVisibilityChanged(int visibility) {  
            super.onWindowVisibilityChanged(visibility);  
            mWindowVisibility = visibility == VISIBLE;  
            mRequestedVisible = mWindowVisibility && mViewVisibility;  
            updateWindow(false, false);  
        }  
      
        ......  
    }  
```
	- Step 8. SurfaceView.updateWindow
```java
public class SurfaceView extends View {  
    ......  
  	/*SurfaceView类的成员变量mSurface指向的是一个Surface对象，这个Surface对象描述的便是SurfaceView专有的绘图表面。*/
    final Surface mSurface = new Surface();  
    ......  
	/*每一个Activity窗口都关联有一个W对象。这个W对象是一个实现了IWindow接口的Binder本地对象，它是用来传递给WindowManagerService服务的，以便WindowManagerService服务可以通过它来和它所关联的Activity窗口通信。同时，这个W对象还用来在WindowManagerService服务这一侧唯一地标志一个窗口，也就是说，WindowManagerService服务会为这个W对象创建一个WindowState对象。*/
	/*SurfaceView类的成员变量mWindow指向的是一个MyWindow对象。MyWindow类是从BaseIWindow类继承下来的，后者与W类一样，实现了IWindow接口。也就是说，每一个SurfaceView都关联有一个实现了IWindow接口的Binder本地对象，就如第一个Activity窗口都关联有一个实现了IWindow接口的W对象一样。从这里我们就可以推断出，每一个SurfaceView在WindowManagerService服务这一侧都对应有一个WindowState对象。从这一点来看，WindowManagerService服务认为Activity窗口和SurfaceView的地位是一样的，即认为它们都是一个窗口，并且具有绘图表面。*/
    MyWindow mWindow;  
    .....  
  	/* SurfaceView类的成员变量mWindowType描述的是SurfaceView的窗口类型，它的默认值等于TYPE_APPLICATION_MEDIA。也就是说，我们在创建一个SurfaceView的时候，默认是用来显示多媒体的.如果一个WindowState对象所描述的窗口的类型为TYPE_APPLICATION_MEDIA或者TYPE_APPLICATION_MEDIA_OVERLAY，那么它就会位于它所附加在的窗口的下面。*/
    int mWindowType = WindowManager.LayoutParams.TYPE_APPLICATION_MEDIA;  
    ......  
  	/*SurfaceView类的成员变量mRequestedType描述的是SurfaceView的绘图表面的类型，一般来说，它的值可能等于SURFACE_TYPE_NORMAL，也可能等于SURFACE_TYPE_PUSH_BUFFERS,当一个SurfaceView的绘图表面的类型等于SURFACE_TYPE_NORMAL的时候，就表示该SurfaceView的绘图表面所使用的内存是一块普通的内存。一般来说，这块内存是由SurfaceFlinger服务来分配的，我们可以在应用程序内部自由地访问它.当一个SurfaceView的绘图表面的类型等于SURFACE_TYPE_PUSH_BUFFERS的时候，就表示该SurfaceView的绘图表面所使用的内存不是由SurfaceFlinger服务分配的，因而我们不能够在应用程序内部对它进行操作。在创建了一个SurfaceView之后，可以调用它的成员函数getHolder获得一个SurfaceHolder对象，然后再调用该SurfaceHolder对象的成员函数setType来修改该SurfaceView的绘图表面的类型，即修改该SurfaceView的成员变量mRequestedType的值。*/
    int mRequestedType = -1;  
    ......  
  
    private void updateWindow(boolean force, boolean redrawNeeded) {  
        if (!mHaveFrame) {  /*mHaveFrame，用来描述SurfaceView的宿主窗口的大小是否已经计算好了。只有当宿主窗口的大小计算之后，SurfaceView才可以更新自己的窗口。*/
            return;  
        }  
        ......  
  
        int myWidth = mRequestedWidth; /*mRequestedWidth，用来描述SurfaceView最后一次被请求的宽度。*/
        if (myWidth <= 0) myWidth = getWidth();  
        int myHeight = mRequestedHeight;  
        if (myHeight <= 0) myHeight = getHeight();  
  
        getLocationInWindow(mLocation);  
        final boolean creating = mWindow == null;  
        final boolean formatChanged = mFormat != mRequestedFormat;  
        final boolean sizeChanged = mWidth != myWidth || mHeight != myHeight;  
        final boolean visibleChanged = mVisible != mRequestedVisible  
                || mNewSurfaceNeeded;  
        final boolean typeChanged = mType != mRequestedType;  
        if (force || creating || formatChanged || sizeChanged || visibleChanged  
            || typeChanged || mLeft != mLocation[0] || mTop != mLocation[1]  
            || mUpdateWindowNeeded || mReportDrawNeeded || redrawNeeded) {  
            ......  
  
            try {  
                final boolean visible = mVisible = mRequestedVisible;  
                mLeft = mLocation[0];  
                mTop = mLocation[1];  
                mWidth = myWidth;  
                mHeight = myHeight;  
                mFormat = mRequestedFormat;  
                mType = mRequestedType;  
                ......  
  
                // Places the window relative  
                mLayout.x = mLeft;  
                mLayout.y = mTop;  
                mLayout.width = getWidth();  
                mLayout.height = getHeight();  
                ......  
  
                mLayout.memoryType = mRequestedType;  
  
                if (mWindow == null) {  
					/* 检查成员变量mWindow的值是否等于null。如果等于null的话，那么就说明该SurfaceView还没有增加到WindowManagerService服务中去。*/
                    mWindow = new MyWindow(this);  
                    mLayout.type = mWindowType;  
                    ......  
					/*Session类的成员函数addWithoutInputChannel与另外一个成员函数add的实现是类似的，它们都是用来在WindowManagerService服务内部为指定的窗口增加一个WindowState对象*/
                    mSession.addWithoutInputChannel(mWindow, mLayout,  
                            mVisible ? VISIBLE : GONE, mContentInsets);  
                }  
                ......  
  
                mSurfaceLock.lock();  
                try {  
                    ......  
  					/*调用成员变量mSession所描述的一个Binder代理对象的成员函数relayout来请求WindowManagerService服务对SurfaceView的UI进行布局。从前面Android应用程序窗口（Activity）的绘图表面（Surface）的创建过程分析一文可以知道，WindowManagerService服务在对一个窗口进行布局的时候，如果发现该窗口的绘制表面还未创建，或者需要需要重新创建，那么就会为请求SurfaceFlinger服务为该窗口创建一个新的绘图表面，并且将该绘图表面返回来给调用者。注意，这一步由于可能会修改SurfaceView的绘图表面，即修改成员变量mSurface的指向的一个Surface对象的内容，因此，就需要在获得成员变量mSurfaceLock所描述的一个锁的情况下执行，避免其它线程同时修改该绘图表面的内容*/
                    final int relayoutResult = mSession.relayout(  
                        mWindow, mLayout, mWidth, mHeight,  
                            visible ? VISIBLE : GONE, false, mWinFrame, mContentInsets,  
                            mVisibleInsets, mConfiguration, mSurface);  
                    ......  
  
                } finally {  
                    mSurfaceLock.unlock();  
                }  
  
                ......  
            } catch (RemoteException ex) {  
            }  
  
            .....  
        }  
    }  
  
    ......  
}
```

##SurfaceView的挖洞过程
1. urfaceView的窗口类型一般都是TYPE_APPLICATION_MEDIA或者TYPE_APPLICATION_MEDIA_OVERLAY，也就是说，它的Z轴位置是小于其宿主窗口的Z位置的。为了保证SurfaceView的UI是可见的，SurfaceView就需要在其宿主窗口的上面挖一个洞出来，实际上就是在其宿主窗口的绘图表面上设置一块透明区域，以便可以将自己显示出来。
	- Step 1. SurfaceView.onAttachedToWindow
```java
    public class SurfaceView extends View {  
        ......  
      
        @Override  
        protected void onAttachedToWindow() {  
            super.onAttachedToWindow();  
            mParent.requestTransparentRegion(this);  
            ......  
        }  
      
        ......  
    }  
```
	-  Step 2. ViewGroup.requestTransparentRegion
```java
    public abstract class ViewGroup extends View implements ViewParent, ViewManager {  
        ......  
      
        public void requestTransparentRegion(View child) {  
			/*ViewGroup类的成员函数requestTransparentRegion首先将它的成员变量mPrivateFlags的值的View.REQUEST_TRANSPARENT_REGIONS位设置为1，表示它要在宿主窗口上设置透明区域，接着再调用从父类View继承下来的成员变量mParent所指向的一个视图容器的成员函数requestTransparentRegion来继续向上请求设置透明区域，这个过程会一直持续到当前正在处理的视图容器为窗口的顶层视图为止。*/
            if (child != null) {  
                child.mPrivateFlags |= View.REQUEST_TRANSPARENT_REGIONS;  
                if (mParent != null) {  
                    mParent.requestTransparentRegion(this);  
                }  
            }  
        }  
      
        ......  
    }  
```
	- Step 3. ViewRoot.requestTransparentRegion
```java
public final class ViewRoot extends Handler implements ViewParent,  
        View.AttachInfo.Callbacks {  
    ......  
  
    public void requestTransparentRegion(View child) {  
/* 当一个窗口被请求设置了一块透明区域之后，它的窗口属性就发生变化了，因此，这时候除了要将与它所关联的一个ViewRoot对象的成员变量mWindowAttributesChanged的值设置为true之外，还要调用该ViewRoot对象的成员函数requestLayout来请求刷新一下窗口的UI，即请求对窗口的UI进行重新布局和绘制。*/
        // the test below should not fail unless someone is messing with us  
        checkThread();  
        if (mView == child) {  
            mView.mPrivateFlags |= View.REQUEST_TRANSPARENT_REGIONS;  
            // Need to make sure we re-evaluate the window attributes next  
            // time around, to ensure the window has the correct format.  
            mWindowAttributesChanged = true;  
            requestLayout();/*最终调用performTraversals来实际执行刷新窗口UI的操作,ViewRoot类的成员函数performTraversals在刷新窗口UI的过程中，就会将嵌入在它里面的SurfaceView所要设置的透明区域收集起来，以便可以请求WindowManagerService将这块透明区域设置到它的绘图表面上去*/
        }  
    }  
  
    ......  
}
```
	- Step 4. ViewRoot.performTraversals
ViewRoot类的成员函数performTraversals是在窗口的UI布局完成之后，并且在窗口的UI绘制之前，收集嵌入在它里面的SurfaceView所设置的透明区域的，这是因为窗口的UI布局完成之后，各个子视图的大小和位置才能确定下来，这样SurfaceView才知道自己要设置的透明区域的位置和大小。
```java
public final class ViewRoot extends Handler implements ViewParent,  
        View.AttachInfo.Callbacks {  
    ......  
  
    private void performTraversals() {  
        ......  
  
        // cache mView since it is used so much below...  
        final View host = mView;  
        ......  
  
        final boolean didLayout = mLayoutRequested;  
        ......  
  
        if (didLayout) {  
            ......  
  
            host.layout(0, 0, host.mMeasuredWidth, host.mMeasuredHeight);  
  
            ......  
/*这块透明区域的收集过程如下所示：
        (1). 计算顶层视图的位置和大小，即计算顶层视图所占据的区域。
        (2). 将顶层视图所占据的区域作为窗口的初始化透明区域，保存在ViewRoot类的成员变量mTransparentRegion中。
        (3). 从顶层视图开始，从上到下收集每一个子视图所要设置的区域，最终收集到的总透明区域也是保存在ViewRoot类的成员变量mTransparentRegion中。
        (4). 检查ViewRoot类的成员变量mTransparentRegion和mPreviousTransparentRegion所描述的区域是否相等。如果不相等的话，那么就说明窗口的透明区域发生了变化，这时候就需要调用ViewRoot类的的静态成员变量sWindowSession所描述的一个Binder代理对象的成员函数setTransparentRegion通知WindowManagerService为窗口设置由成员变量mTransparentRegion所指定的透明区域。*/
            if ((host.mPrivateFlags & View.REQUEST_TRANSPARENT_REGIONS) != 0) {  
                // start out transparent  
                // TODO: AVOID THAT CALL BY CACHING THE RESULT?  
                host.getLocationInWindow(mTmpLocation);  
                mTransparentRegion.set(mTmpLocation[0], mTmpLocation[1],  
                        mTmpLocation[0] + host.mRight - host.mLeft,  
                        mTmpLocation[1] + host.mBottom - host.mTop);  
  
                host.gatherTransparentRegion(mTransparentRegion);  
                ......  
                if (!mTransparentRegion.equals(mPreviousTransparentRegion)) {  
                    mPreviousTransparentRegion.set(mTransparentRegion);  
                    // reconfigure window manager  
                    try {  
                        sWindowSession.setTransparentRegion(mWindow, mTransparentRegion);  
                    } catch (RemoteException e) {  
                    }  
                }  
            }  
  
            ......  
        }  
   
        ......  
  
        boolean cancelDraw = attachInfo.mTreeObserver.dispatchOnPreDraw();  
  
        if (!cancelDraw && !newSurface) {  
            ......  
  
            draw(fullRedrawNeeded);  
  
            ......  
        }   
  
        ......  
    }  
  
    ......  
}
```
	- Step 5. ViewGroup.gatherTransparentRegion
```java
public abstract class ViewGroup extends View implements ViewParent, ViewManager {  
    ......  
  
    @Override  
    public boolean gatherTransparentRegion(Region region) {  
        // If no transparent regions requested, we are always opaque.  
        final boolean meOpaque = (mPrivateFlags & View.REQUEST_TRANSPARENT_REGIONS) == 0;  
        if (meOpaque && region == null) {  
            // The caller doesn't care about the region, so stop now.  
            return true;  
        }  
/* (1). 调用父类View的成员函数gatherTransparentRegion来检查当前正在处理的视图容器是否需要绘制。如果需要绘制的话，那么就会将它所占据的区域从参数region所占据的区域移除，这是因为参数region所描述的区域开始的时候是等于窗口的顶层视图的大小的，也就是等于窗口的整个大小的。
        (2). 调用当前正在处理的视图容器的每一个子视图的成员函数gatherTransparentRegion来继续往下收集透明区域。*/
        super.gatherTransparentRegion(region);  
        final View[] children = mChildren;  
        final int count = mChildrenCount;  
        boolean noneOfTheChildrenAreTransparent = true;  
        for (int i = 0; i < count; i++) {  
            final View child = children[i];  
            if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {  
                if (!child.gatherTransparentRegion(region)) {  
                    noneOfTheChildrenAreTransparent = false;  
                }  
            }  
        }  
        return meOpaque || noneOfTheChildrenAreTransparent;  
    }  
  
    ......  
}
```
	- Step 6. SurfaceView.gatherTransparentRegion
```java
public class SurfaceView extends View {  
    ......  
  
    @Override  
    public boolean gatherTransparentRegion(Region region) {  
/*首先是检查当前正在处理的SurfaceView是否是用作窗口面板的，即它的成员变量mWindowType的值是否等于WindowManager.LayoutParams.TYPE_APPLICATION_PANEL。如果等于的话，那么就会调用父类View的成员函数gatherTransparentRegion来检查该面板是否需要绘制。如果需要绘制，那么就会将它所占据的区域从参数region所描述的区域移除。*/
        if (mWindowType == WindowManager.LayoutParams.TYPE_APPLICATION_PANEL) {  
            return super.gatherTransparentRegion(region);  
        }  
  
        boolean opaque = true;  
/*直接检查当前正在处理的SurfaceView是否是需要在宿主窗口的绘图表面上进行绘制，即检查成员变量mPrivateFlags的值的SKIP_DRAW位是否等于1。如果需要的话，那么也会调用父类View的成员函数gatherTransparentRegion来将它所占据的区域从参数region所描述的区域移除。*/
        if ((mPrivateFlags & SKIP_DRAW) == 0) {  
            // this view draws, remove it from the transparent region  
            opaque = super.gatherTransparentRegion(region);  
        } else if (region != null) {  
/*计算好当前正在处理的SurfaceView所占据的区域，然后再将该区域添加到参数region所描述的区域中去，这样就可以得到窗口的一个新的透明区域。*/
            int w = getWidth();  
            int h = getHeight();  
            if (w>0 && h>0) {
                getLocationInWindow(mLocation);
                // otherwise, punch a hole in the whole hierarchy  
                int l = mLocation[0];  
                int t = mLocation[1];  
                region.op(l, t, l+w, t+h, Region.Op.UNION);  
            }  
        }  
        if (PixelFormat.formatHasAlpha(mRequestedFormat)) {  
            opaque = false;  
        }  
        return opaque;  
    }  
  
    ......  
}
```

##SurfaceView的绘制过程
1. SurfaceView虽然具有独立的绘图表面，不过它仍然是宿主窗口的视图结构中的一个结点，因此，它仍然是可以参与到宿主窗口的绘制流程中去的.
```java
public class SurfaceView extends View {  
    ......  
  
    @Override  
    public void draw(Canvas canvas) {  
        if (mWindowType != WindowManager.LayoutParams.TYPE_APPLICATION_PANEL) {  
            // draw() is not called when SKIP_DRAW is set  
            if ((mPrivateFlags & SKIP_DRAW) == 0) {  
                // punch a whole in the view-hierarchy below us  
                canvas.drawColor(0, PorterDuff.Mode.CLEAR);  
            }  
        }  
        super.draw(canvas);  
    }  
  
    @Override  
    protected void dispatchDraw(Canvas canvas) {  
        if (mWindowType != WindowManager.LayoutParams.TYPE_APPLICATION_PANEL) {  
            // if SKIP_DRAW is cleared, draw() has already punched a hole  
            if ((mPrivateFlags & SKIP_DRAW) == SKIP_DRAW) {  
                // punch a whole in the view-hierarchy below us  
                canvas.drawColor(0, PorterDuff.Mode.CLEAR);  
            }  
        }  
        // reposition ourselves where the surface is   
        mHaveFrame = true;  
        updateWindow(false, false);  
        super.dispatchDraw(canvas);  
    }  
  
    ......  
}
/*SurfaceView类的成员函数draw和dispatchDraw的参数canvas所描述的都是建立在宿主窗口的绘图表面上的画布，因此，在这块画布上绘制的任何UI都是出现在宿主窗口的绘图表面上的。从SurfaceView类的成员函数draw和dispatchDraw的实现就可以看出，SurfaceView在其宿主窗口的绘图表面上面所做的操作就是将自己所占据的区域绘为黑色，除此之外，就没有其它更多的操作了，这是因为SurfaceView的UI是要展现在它自己的绘图表面上面的。接下来我们就分析如何在SurfaceView的绘图表面上面进行UI绘制。*/
```

2. 如果要在一个绘图表面进行UI绘制，那么就顺序执行以下的操作：
        (1). 在绘图表面的基础上建立一块画布，即获得一个Canvas对象。
        (2). 利用Canvas类提供的绘图接口在前面获得的画布上绘制任意的UI。
        (3). 将已经填充好了UI数据的画布缓冲区提交给SurfaceFlinger服务，以便SurfaceFlinger服务可以将它合成到屏幕上去。
3. SurfaceView的绘制过程
	- Step 1. SurfaceView.getHolder
```java
public class SurfaceView extends View {  
    ......  
  
    public SurfaceHolder getHolder() {  
        return mSurfaceHolder;  
    }  
  
    ......  
  
    private SurfaceHolder mSurfaceHolder = new SurfaceHolder() {  
        ......  
    }  
  
    ......  
}
```
	- Step 2. SurfaceHolder.lockCanvas
```java
public class SurfaceView extends View {  
    ......  
  
    final ReentrantLock mSurfaceLock = new ReentrantLock();  
/*SurfaceView类的成员变量mSurface描述的是就是SurfaceView的专有绘图表面*/
    final Surface mSurface = new Surface();  
    ......  
  
    private SurfaceHolder mSurfaceHolder = new SurfaceHolder() {  
        ......  
  
        public Canvas lockCanvas() {  
            return internalLockCanvas(null);  
        }  
  
        ......  
  
        private final Canvas internalLockCanvas(Rect dirty) {  
            if (mType == SURFACE_TYPE_PUSH_BUFFERS) {  
                throw new BadSurfaceTypeException(  
                        "Surface type is SURFACE_TYPE_PUSH_BUFFERS");  
            }  
            mSurfaceLock.lock();  
            ......  
  
            Canvas c = null;  
            if (!mDrawingStopped && mWindow != null) {  
                Rect frame = dirty != null ? dirty : mSurfaceFrame;  
                try {  
                    c = mSurface.lockCanvas(frame);  
                } catch (Exception e) {  
                    Log.e(LOG_TAG, "Exception locking surface", e);  
                }  
            }  
            ......  
  
            if (c != null) {  
                mLastLockTime = SystemClock.uptimeMillis();  
                return c;  
            }  
            ......  
  
            mSurfaceLock.unlock();  
  
            return null;    
        }   
  
        ......           
    }  
  
    ......  
}
```
	- Step 3. Surface.lockCanvas
Surface类的成员函数lockCanvas它大致就是通过JNI方法来在当前正在处理的绘图表面上获得一个图形缓冲区，并且将这个图形绘冲区封装在一块类型为Canvas的画布中返回给调用者使用。
	- Step 4. SurfaceHolder.unlockCanvasAndPost
调用者在画布上绘制完成所需要的UI之后，就可以将这块画布的图形绘冲区的UI数据提交给SurfaceFlinger服务来处理了，这是通过调用SurfaceHolder类的成员函数unlockCanvasAndPost来实现的。
```java
public class SurfaceView extends View {  
    ......  
  
    final ReentrantLock mSurfaceLock = new ReentrantLock();  
    final Surface mSurface = new Surface();  
    ......  
  
    private SurfaceHolder mSurfaceHolder = new SurfaceHolder() {  
        ......  
  
        public void unlockCanvasAndPost(Canvas canvas) {  
            mSurface.unlockCanvasAndPost(canvas);  
            mSurfaceLock.unlock();  
        }   
  
        ......           
    }  
  
    ......  
}
```

