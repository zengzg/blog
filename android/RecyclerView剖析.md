####简介
　　本文将从RecyclerView实现原理并结合源码详细分析这个强大的控件。阅读本文要求：1、熟悉android控件绘制，2、了解动画，3、了解Scroller，4、You`re a fucking kind person。本文所示源码版本是23.2.0。本文欢迎转载，不需要注明出处。

####基本使用
　　RecyclerView的基本使用并不复杂，只需要提供一个RecyclerView.Apdater的实现用于处理数据集与ItemView的绑定关系，和一个RecyclerView.LayoutManager的实现用于 **测量并布局** ItemView。

####绘制流程
　　众所周知，android控件的绘制可以分为3个步骤：measure、layout、draw。RecyclerView的绘制自然也经这3个步骤。但是，RecyclerView将它的measure与layout过程委托给了RecyclerView.LayoutManager来处理，并且，它对子控件的measure及layout过程是逐个处理的，也就是说，执行完成一个子控件的measure及layout过程再去执行下一个。下面看下这段代码：

    protected void onMeasure(int widthSpec, int heightSpec) {
        ...
        if (mLayout.mAutoMeasure) {
            final int widthMode = MeasureSpec.getMode(widthSpec);
            final int heightMode = MeasureSpec.getMode(heightSpec);
            final boolean skipMeasure = widthMode == MeasureSpec.EXACTLY
                    && heightMode == MeasureSpec.EXACTLY;
            mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
            if (skipMeasure || mAdapter == null) {
                return;
            }
            ...
            dispatchLayoutStep2();
            
            mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);
            ...
        } else {
            ...
        }
    }

这是RecyclerView的测量方法，再看下dispatchLayoutStep2()方法：

    private void dispatchLayoutStep2() {
        ...
        mLayout.onLayoutChildren(mRecycler, mState);
        ...
    }
上面的mLayout就是一个RecyclerView.LayoutManager实例。通过以上代码（和方法名称），不难推断出，RecyclerView的measure及layout过程委托给了RecyclerView.LayoutManager。接着看onLayoutChildren方法，在兼容包中提供了3个RecyclerView.LayoutManager的实现，这里我就只以LinearLayoutManager来举例说明：

    public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
        // layout algorithm:
        // 1) by checking children and other variables, find an anchor coordinate and an anchor
        //  item position.
        // 2) fill towards start, stacking from bottom
        // 3) fill towards end, stacking from top
        // 4) scroll to fulfill requirements like stack from bottom.
        ...
        mAnchorInfo.mLayoutFromEnd = mShouldReverseLayout ^ mStackFromEnd;
        // calculate anchor position and coordinate
        updateAnchorInfoForLayout(recycler, state, mAnchorInfo);
        ...
        if (mAnchorInfo.mLayoutFromEnd) {
            ...
        } else {
            // fill towards end
            updateLayoutStateToFillEnd(mAnchorInfo);
            mLayoutState.mExtra = extraForEnd;
            fill(recycler, mLayoutState, state, false);
            endOffset = mLayoutState.mOffset;
            final int lastElement = mLayoutState.mCurrentPosition;
            if (mLayoutState.mAvailable > 0) {
                extraForStart += mLayoutState.mAvailable;
            }
            // fill towards start
            updateLayoutStateToFillStart(mAnchorInfo);
            mLayoutState.mExtra = extraForStart;
            mLayoutState.mCurrentPosition += mLayoutState.mItemDirection;
            fill(recycler, mLayoutState, state, false);
            startOffset = mLayoutState.mOffset;
            ...
        }
        ...
    }
源码中的注释部分我并没有略去，它已经解释了此处的逻辑了。这里我以垂直布局来说明，mAnchorInfo为布局锚点信息，包含了子控件在Y轴上起始绘制偏移量（coordinate），ItemView在Adapter中的索引位置（position）和布局方向（mLayoutFromEnd）——这里是指start、end方向。这部分代码的功能就是：确定布局锚点，以此为起点向开始和结束方向填充ItemView，如图所示：
![图一](https://raw.githubusercontent.com/zengzg/blog/master/android/images/path5784-0.png)
在上一段代码中，fill()方法的作用就是填充ItemView，而图（3）说明了，在上段代码中fill()方法调用2次的原因。虽然图（3）是更为普遍的情况，而且在实现填充ItemView算法时，也是按图（3）所示来实现的，但是mAnchorInfo在赋值过程(updateAnchorInfoForLayout)中，只会出现图（1）、图（2）所示情况。现在来看下fill()方法：

    int fill(RecyclerView.Recycler recycler, LayoutState layoutState,
            RecyclerView.State state, boolean stopOnFocusable) {
        ...
        int remainingSpace = layoutState.mAvailable + layoutState.mExtra;
        LayoutChunkResult layoutChunkResult = new LayoutChunkResult();
        while (...&&layoutState.hasMore(state)) {
            ...
            layoutChunk(recycler, state, layoutState, layoutChunkResult);
            
            ...
            if (...) {
                layoutState.mAvailable -= layoutChunkResult.mConsumed;
                remainingSpace -= layoutChunkResult.mConsumed;
            }
            if (layoutState.mScrollingOffset != LayoutState.SCOLLING_OFFSET_NaN) {
                layoutState.mScrollingOffset += layoutChunkResult.mConsumed;
                if (layoutState.mAvailable < 0) {
                    layoutState.mScrollingOffset += layoutState.mAvailable;
                }
                recycleByLayoutState(recycler, layoutState);
            }
        }
        ...
    }
下面是layoutChunk()方法：

    void layoutChunk(RecyclerView.Recycler recycler, RecyclerView.State state,
            LayoutState layoutState, LayoutChunkResult result) {
        View view = layoutState.next(recycler);
        ...
        if (layoutState.mScrapList == null) {
            if (mShouldReverseLayout == (layoutState.mLayoutDirection
                    == LayoutState.LAYOUT_START)) {
                addView(view);
            } else {
                addView(view, 0);
            }
        }
        ...
        measureChildWithMargins(view, 0, 0);
        ...
        // We calculate everything with View's bounding box (which includes decor and margins)
        // To calculate correct layout position, we subtract margins.
        layoutDecorated(view, left + params.leftMargin, top + params.topMargin,
                right - params.rightMargin, bottom - params.bottomMargin);
        ...
    }
这里的addView()方法，其实就是ViewGroup的addView()方法；measureChildWithMargins()方法看名字就知道是用于测量子控件大小的，这里我先跳过这个方法的解释，放在后面来做，目前就简单地理解为测量子控件大小就好了。下面是layoutDecoreated()方法：

    public void layoutDecorated(...) {
            ...
            child.layout(...);
    }
总结上面代码，在RecyclerView的measure及layout阶段，填充ItemView的算法为：向父容器增加子控件，测量子控件大小，布局子控件，布局锚点向当前布局方向平移子控件大小，重复上诉步骤至RecyclerView可绘制空间消耗完毕或子控件已全部填充。
　　这样所有的子控件的measure及layout过程就完成了。回到RecyclerView的onMeasure方法，执行` mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec)`这行代码的作用就是根据子控件的大小，设置RecyclerView的大小。至此，RecyclerView的measure和layout实际上已经完成了。
　　但是，你有可能已经发现上面过程中的问题了：如何确定RecyclerView的可绘制空间？不过，如果你熟悉android控件的绘制机制的话，这就不是问题。其实，这里的可绘制空间，可以简单地理解为父容器的大小；更准确的描述是，父容器对RecyclerView的布局大小的要求，可以通过`MeasureSpec.getSize()`方法获得——这里不包括滑动情况，滑动情况会在后文描述。需要特别说明的是在23.2.0版本之前，RecyclerView是不支持WRAP_CONTENT的。先看下RecyclerView的onLayout()方法：

    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        ...
        dispatchLayout();
        ...
    }

这是dispatchLayout()方法：

    void dispatchLayout() {
        ...
        if (mState.mLayoutStep == State.STEP_START) {
            dispatchLayoutStep1();
            ...
            dispatchLayoutStep2();
        }
        dispatchLayoutStep3();
        ...
    }
可以看出，这里也会执行子控件的measure及layout过程。结合onMeasure方法对skipMeasure的判断可以看出，如果要支持WRAP_CONTENT，那么子控件的measure及layout就会提前在RecyclerView的测量方法中执行完成，这步在源码注释中被称为预布局(pre-layout)，当然，这一过程和之后在onLayout中执行的过程并没有什么不同；如果是其它情况(测量模式为EXACTLY)，子控件的measure及layout过程就会延迟至RecyclerView的layout过程（RecyclerView.onLayout()）中执行。再看onMeasure方法中的mLayout.mAutoMeasure，它表示，RecyclerView的measure及layout过程是否要委托给RecyclerView.LayoutManager，在兼容包中提供的３种RecyclerView.LayoutManager的这个属性默认都是为true的。好了，以上就是RecyclerView的measure及layout过程，下面来看下它的draw过程。
　　RecyclerView的draw过程可以分为２部分来看：RecyclerView负责绘制所有decoration；ItemView的绘制由ViewGroup处理，这里的绘制是android常规绘制逻辑，本文就不再阐述了。下面来看看RecyclerView的draw()和onDraw()方法：

    @Override
    public void draw(Canvas c) {
        super.draw(c);

        final int count = mItemDecorations.size();
        for (int i = 0; i < count; i++) {
            mItemDecorations.get(i).onDrawOver(c, this, mState);
        }
        ...
    }

    @Override
    public void onDraw(Canvas c) {
        super.onDraw(c);

        final int count = mItemDecorations.size();
        for (int i = 0; i < count; i++) {
            mItemDecorations.get(i).onDraw(c, this, mState);
        }
    }
可以看出对于decoration的绘制代码上十分简单。但是这里，我必须要抱怨一下RecyclerView.ItemDecoration的设计，它实在是太过于灵活了，虽然理论上我们可以使用它在RecyclerView内的任何地方绘制你想要的任何东西——到这一步，RecyclerView的大小位置已经确定的哦。但是过于灵活，太难使用，以至往往使我们无从下手。
　　好了，题外话就不多说了，来看看decoration的绘制吧。还记得上面提到过的measureChildWithMargins()方法吗？先来看看它：

     public void measureChildWithMargins(View child, int widthUsed, int heightUsed) {
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();

            final Rect insets = mRecyclerView.getItemDecorInsetsForChild(child);
            widthUsed += insets.left + insets.right;
            heightUsed += insets.top + insets.bottom;

            final int widthSpec = ...
            final int heightSpec = ...
            if (shouldMeasureChild(child, widthSpec, heightSpec, lp)) {
                child.measure(widthSpec, heightSpec);
            }
        }
这里是getItemDecorInsetsForChild()方法：

     Rect getItemDecorInsetsForChild(View child) {
        ...
        final Rect insets = lp.mDecorInsets;
        insets.set(0, 0, 0, 0);
        final int decorCount = mItemDecorations.size();
        for (int i = 0; i < decorCount; i++) {
            mTempRect.set(0, 0, 0, 0);
            mItemDecorations.get(i).getItemOffsets(mTempRect, child, this, mState);
            insets.left += mTempRect.left;
            insets.top += mTempRect.top;
            insets.right += mTempRect.right;
            insets.bottom += mTempRect.bottom;
        }
        lp.mInsetsDirty = false;
        return insets;
    }
方法getItemOffsets()就是我们在实现一个RecyclerView.ItemDecoration时可以重写的方法，通过mTempRect的大小，可以为每个ItemView设置位置偏移量，这个偏移量最终会参与计算ItemView的大小，也就是说ItemView的大小是包含这个位置偏移量的。我们在重写getItemOffsets()时，可以指定任意数值的偏移量：
![itemView offset](https://raw.githubusercontent.com/zengzg/blog/master/android/images/text9449-1.png)
4个方向的位置偏移量对应mTempRect的4个属性(left,top,right,bottom)，我以top offset的值在垂直线性布局中的应用来举例说明下。如果top offset等于0，那么ItemView之间就没有空隙；如果top offset大于0，那么ItemView之前就会有一个间隙；如果top offset小于0，那么ItemView之间就会有重叠的区域。
　　当然，我们在实现RecyclerView.ItemDecoration时，并不一定要重写getItemOffsets()，同样的对于RecyclerView.ItemDecoration.onDraw()或RecyclerView.ItemDecoration.onDrawOver()方法也不是一定要重写，而且，这个绘制方法和我们所设置的位置偏移量**没有任何联系**。下面我来实现一个RecyclerView.ItemDecoration来加深下这里的理解：我将在垂直线性布局下，在ItemView间绘制一条5个像素宽、只有ItemView一半长、与ItemView居中对齐的红色分割线，这条分割线在ItemView内部top位置。

    @Override
    public void onDraw(Canvas c, RecyclerView parent, RecyclerView.State state) {
      Paint paint = new Paint();
      paint.setColor(Color.RED);

      for (int i = 0; i < parent.getLayoutManager().getChildCount(); i++) {
        final View child = parent.getChildAt(i);

        float left = child.getLeft() + (child.getRight() - child.getLeft()) / 4;
        float top = child.getTop();
        float right = left + (child.getRight() - child.getLeft()) / 2;
        float bottom = top + 5;
        
        c.drawRect(left,top,right,bottom,paint);
      }
    }

    @Override
    public void getItemOffsets(Rect outRect, View view, RecyclerView parent, RecyclerView.State state) {
      outRect.set(0, 0, 0, 0);
    }
代码不是很严谨，大家姑且一看吧，当然这里getItemOffsets()方法可以省略的。
　　以上就是RecyclerView的整个绘制流程了，值得注意的地方也就是在23.2.0中RecyclerView支持WRAP_CONTENT属性了；还有就是ItemView的填充算法fill()算是一个亮点吧。接下来，我将分析ReyclerView的滑动流程。

####滑动
　　RecyclerView的滑动过程可以分为2个阶段：手指在屏幕上移动，使RecyclerView滑动的过程，可以称为scroll；手指离开屏幕，RecyclerView继续滑动一段距离的过程，可以称为fling。现在先看看RecyclerView的触屏事件处理onTouchEvent()方法：

    public boolean onTouchEvent(MotionEvent e) {
        ...
        if (mVelocityTracker == null) {
            mVelocityTracker = VelocityTracker.obtain();
        }
        ...
        switch (action) {
            ...
            case MotionEvent.ACTION_MOVE: {
                ...
                final int x = (int) (MotionEventCompat.getX(e, index) + 0.5f);
                final int y = (int) (MotionEventCompat.getY(e, index) + 0.5f);
                int dx = mLastTouchX - x;
                int dy = mLastTouchY - y;
                ...
                if (mScrollState != SCROLL_STATE_DRAGGING) {
                    ...
                    if (canScrollVertically && Math.abs(dy) > mTouchSlop) {
                        if (dy > 0) {
                            dy -= mTouchSlop;
                        } else {
                            dy += mTouchSlop;
                        }
                        startScroll = true;
                    }
                    if (startScroll) {
                        setScrollState(SCROLL_STATE_DRAGGING);
                    }
                }

                if (mScrollState == SCROLL_STATE_DRAGGING) {
                    mLastTouchX = x - mScrollOffset[0];
                    mLastTouchY = y - mScrollOffset[1];

                    if (scrollByInternal(
                            canScrollHorizontally ? dx : 0,
                            canScrollVertically ? dy : 0,
                            vtev)) {
                        getParent().requestDisallowInterceptTouchEvent(true);
                    }
                }
            } break;
            ...
            case MotionEvent.ACTION_UP: {
                ...
                final float yvel = canScrollVertically ?
                        -VelocityTrackerCompat.getYVelocity(mVelocityTracker, mScrollPointerId) : 0;
                if (!((xvel != 0 || yvel != 0) && fling((int) xvel, (int) yvel))) {
                    setScrollState(SCROLL_STATE_IDLE);
                }
                resetTouch();
            } break;
            ...
        }
        ...
    }
这里我以垂直方向的滑动来说明。当RecyclerView接收到ACTION_MOVE事件后，会先计算出手指移动距离（dy），并与滑动阀值（mTouchSlop）比较，当大于此阀值时将滑动状态设置为SCROLL_STATE_DRAGGING，而后调用scrollByInternal()方法，使RecyclerView滑动，这样RecyclerView的滑动的第一阶段scroll就完成了；当接收到ACTION_UP事件时，会根据之前的滑动距离与时间计算出一个初速度yvel，这步计算是由VelocityTracker实现的，然后再以此初速度，调用方法fling()，完成RecyclerView滑动的第二阶段fling。显然滑动过程中关键的方法就2个：scrollByInternal()与fling()。接下来同样以垂直线性布局来说明。先来说明scrollByInternal()，跟踪进入后，会发现它最终会调用到LinearLayoutManager.scrollBy()方法，这个过程很简单，我就不列出源码了，但是分析到这里先暂停下，去看看fling()方法：

    public boolean fling(int velocityX, int velocityY) {
        ...
        mViewFlinger.fling(velocityX, velocityY);
        ...
    }
有用的就这一行，其它乱七八糟的不看也罢。mViewFlinger是一个Runnable的实现ViewFlinger的对象，就是它来控件着ReyclerView的fling过程的算法的。下面来看下类ViewFlinger的一段代码：

    void postOnAnimation() {
        if (mEatRunOnAnimationRequest) {
            mReSchedulePostAnimationCallback = true;
        } else {
            removeCallbacks(this);
            ViewCompat.postOnAnimation(RecyclerView.this, this);
        }
    }

    public void fling(int velocityX, int velocityY) {
        setScrollState(SCROLL_STATE_SETTLING);
        mLastFlingX = mLastFlingY = 0;
        mScroller.fling(0, 0, velocityX, velocityY,
                Integer.MIN_VALUE, Integer.MAX_VALUE, Integer.MIN_VALUE, Integer.MAX_VALUE);
        postOnAnimation();
    }
可以看到，其实RecyclerView的fling是借助Scroller实现的；然后postOnAnimation()方法的作用就是在将来的某个时刻会执行我们给定的一个Runnable对象，在这里就是这个mViewFlinger对象，这部分原理我就不再深入分析了，它已经不属于本文的范围了。并且，关于Scroller的作用及原理，本文也不会作过多解释。对于这两点各位可以自行查阅，有很多文章对于作过详细阐述的。接下来看看ViewFlinger.run()方法：

    public void run() {
        ...
        if (scroller.computeScrollOffset()) {
            final int x = scroller.getCurrX();
            final int y = scroller.getCurrY();
            final int dx = x - mLastFlingX;
            final int dy = y - mLastFlingY;
            ...
            if (mAdapter != null) {
                ...
                if (dy != 0) {
                    vresult = mLayout.scrollVerticallyBy(dy, mRecycler, mState);
                    overscrollY = dy - vresult;
                }
                ...
            }
            ...
            if (!awakenScrollBars()) {
                invalidate();//刷新界面
            }
            ...
            if (scroller.isFinished() || !fullyConsumedAny) {
                setScrollState(SCROLL_STATE_IDLE);
            } else {
                postOnAnimation();
            }
        }
        ...
    }
本段代码中有个方法mLayout.scrollVerticallyBy()，跟踪进入你会发现它最终也会走到LinearLayoutManager.scrollBy()，这样虽说RecyclerView的滑动可以分为两阶段，但是它们的实现最终其实是一样的。这里我先解释下上段代码。第一，dy表示滑动偏移量，它是由Scroller根据时间偏移量（Scroller.fling()开始时间到当前时刻）计算出的，当然如果是RecyclerView的scroll阶段，这个偏移量也就是手指滑动距离。第二，上段代码会多次执行，至到Scroller判断滑动结束或已经滑动到边界。再多说一下，postOnAnimation()保证了RecyclerView的滑动是流畅，这里涉及到著名的“android 16ms”机制，简单来说理想状态下，上段代码会以16毫秒一次的速度执行，这样其实，Scroller每次计算的滑动偏移量是很小的一部分，而RecyclerView就会根据这个偏移量，确定是平移ItemView，还是除了平移还需要再创建新ItemView。
![scroll offset](https://raw.githubusercontent.com/zengzg/blog/master/android/images/path5784-1.png)
　　现在就来看看LinearLayoutManager.scrollBy()方法：

    int scrollBy(int dy, RecyclerView.Recycler recycler, RecyclerView.State state) {
        ...
        final int absDy = Math.abs(dy);
        updateLayoutState(layoutDirection, absDy, true, state);
        final int consumed = mLayoutState.mScrollingOffset
                + fill(recycler, mLayoutState, state, false);
        ...
        final int scrolled = absDy > consumed ? layoutDirection * consumed : dy;
        mOrientationHelper.offsetChildren(-scrolled);
        ...
    }
如上文所讲到的fill()方法，作用就是向可绘制区间填充ItemView，那么在这里，可绘制区间就是滑动偏移量！再看方法` mOrientationHelper.offsetChildren()`作用就是平移ItemView。好了整个滑动过程就分析完成了，当然RecyclerView的滑动还有个特性叫平滑滑动（smooth scroll），其实它的实现就是一个fling滑动，所以就不再赘述了。

####Recycler
　　Recycler的作用就是重用ItemView。在填充ItemView的时候，ItemView是从它获取的；滑出屏幕的ItemView是由它回收的。对于不同状态的ItemView存储在了不同的集合中，比如有scrapped、cached、exCached、recycled，当然这些集合并不是都定义在同一个类里。
　　回到之前的layoutChunk方法中，有行代码`layoutState.next(recycler)`，它的作用自然就是获取ItemView，我们进入这个方法查看，最终它会调用到RecyclerView.Recycler.getViewForPosition()方法：

    View getViewForPosition(int position, boolean dryRun) {
        ...
        // 0) If there is a changed scrap, try to find from there
        if (mState.isPreLayout()) {
            holder = getChangedScrapViewForPosition(position);
            fromScrap = holder != null;
        }
        // 1) Find from scrap by position
        if (holder == null) {
            holder = getScrapViewForPosition(position, INVALID_TYPE, dryRun);
            ...
        }
        if (holder == null) {
            ...
            // 2) Find from scrap via stable ids, if exists
            if (mAdapter.hasStableIds()) {
                holder = getScrapViewForId(mAdapter.getItemId(offsetPosition), type, dryRun);
                ...
            }
            if (holder == null && mViewCacheExtension != null) {
                final View view = mViewCacheExtension
                        .getViewForPositionAndType(this, position, type);
                if (view != null) {
                    holder = getChildViewHolder(view);
                    ...
                }
            }
            if (holder == null) {
                ...
                holder = getRecycledViewPool().getRecycledView(type);
                ...
            }
            if (holder == null) {
                holder = mAdapter.createViewHolder(RecyclerView.this, type);
                ...
            }
        }
        ...
        boolean bound = false;
        if (mState.isPreLayout() && holder.isBound()) {
            // do not update unless we absolutely have to.
            holder.mPreLayoutPosition = position;
        } else if (!holder.isBound() || holder.needsUpdate() || holder.isInvalid()) {
            ...
            mAdapter.bindViewHolder(holder, offsetPosition);
            ...
        }
        ...
    }

这个方法比较长，我先解释下它的逻辑吧。根据列表位置获取ItemView，先后从scrapped、cached、exCached、recycled集合中查找相应的ItemView，如果没有找到，就创建（Adapter.createViewHolder()），最后与数据集绑定。其中scrapped、cached和exCached集合定义在RecyclerView.Recycler中，分别表示将要在RecyclerView中删除的ItemView、一级缓存ItemView和二级缓存ItemView，cached集合的大小默认为２，exCached是需要我们通过RecyclerView.ViewCacheExtension自己实现的，默认没有；recycled集合其实是一个Map，定义在RecyclerView.RecycledViewPool中，将ItemView以ItemType分类保存了下来，这里算是RecyclerView设计上的亮点，通过RecyclerView.RecycledViewPool可以实现在不同的RecyclerView之前共享ItemView，只要为这些不同RecyclerView设置同一个RecyclerView.RecycledViewPool就可以了。
　　上面解释了ItemView从不同集合中获取的方式，那么RecyclerView又是在什么时候向这些集合中添加ItemView的呢？下面我逐个介绍下。
　　scrapped集合中存储的其实是正在执行REMOVE操作的ItemView，这部分会在后文进一步描述。
　　在fill()方法的循环体中有行代码`recycleByLayoutState(recycler, layoutState);`，最终这个方法会执行到RecyclerView.Recycler.recycleViewHolderInternal()方法：

    void recycleViewHolderInternal(ViewHolder holder) {
            ...
            if (forceRecycle || holder.isRecyclable()) {
                if (!holder.hasAnyOfTheFlags(ViewHolder.FLAG_INVALID | ViewHolder.FLAG_REMOVED
                        | ViewHolder.FLAG_UPDATE)) {
                    // Retire oldest cached view
                    final int cachedViewSize = mCachedViews.size();
                    if (cachedViewSize == mViewCacheMax && cachedViewSize > 0) {
                        recycleCachedViewAt(0);
                    }
                    if (cachedViewSize < mViewCacheMax) {
                        mCachedViews.add(holder);
                        cached = true;
                    }
                }
                if (!cached) {
                    addViewHolderToRecycledViewPool(holder);
                    recycled = true;
                }
            }
            ...
        }

这个方法的逻辑是这样的：首先判断集合cached是否満了，如果已満就从cached集合中移出一个到recycled集合中去，再把新的ItemView添加到cached集合；如果不満就将ItemView直接添加到cached集合。
　　最后exCached集合是我们自己创建的，所以添加删除元素也要我们自己实现。

####数据集、动画
　　RecyclerView定义了4种针对数据集的操作，分别是ADD、REMOVE、UPDATE、MOVE，封装在了AdapterHelper.UpdateOp类中，并且所有操作由一个大小为30的对象池管理着。当我们要对数据集作任何操作时，都会从这个对象池中取出一个UpdateOp对象，放入一个等待队列中，最后调用RecyclerView.RecyclerViewDataObserver.triggerUpdateProcessor()方法，根据这个等待队列中的信息，对所有子控件重新测量、布局并绘制且执行动画，也就是说对于数据集的操作同时只能有一个。以上就是我们调用Adapter.notifyItemXXX()系列方法后发生的事。
　　显然当我们对某个ItemView做操作时，它很有可以会影响到其它ItemView。下面我以REMOVE为例来梳理下这个流程。
![remove example](https://raw.githubusercontent.com/zengzg/blog/master/android/images/path5784-2.png)
　　首先调用Adapter.notifyItemRemove()，追溯到方法RecyclerView.RecyclerViewDataObserver.onItemRangeRemoved()：

    public void onItemRangeRemoved(int positionStart, int itemCount) {
        assertNotInLayoutOrScroll(null);
        if (mAdapterHelper.onItemRangeRemoved(positionStart, itemCount)) {
            triggerUpdateProcessor();
        }
    }
这里的mAdapterHelper.onItemRangeRemoved()就是向之前提及的等待队列添加一个类型为REMOVE的UpdateOp对象， triggerUpdateProcessor()方法就是调用View.requestLayout()方法，这会导致界面重新布局，也就是说方法RecyclerView.onLayout()会随后调用，这之后的流程就和在**绘制流程**一节中所描述的一致了。但是动画在哪是执行的呢？查看之前所列出的onLayout()方法发现dispatchLayoutStepX方法共有3个，前文只解释了dispatchLayoutStep2()的作用，这里就其它2个方法作进一步说明。不过dispatchLayoutStep1()没有过多要说明的东西，它的作用只是初始化数据，需要详细说明的是dispatchLayoutStep3()方法：

    private void dispatchLayoutStep3() {
        ...
        if (mState.mRunSimpleAnimations) {
            // Step 3: Find out where things are now, and process change animations.
            ...
            // Step 4: Process view info lists and trigger animations
            mViewInfoStore.process(mViewInfoProcessCallback);
        }
        ...
    }
代码注释已经说明得很清楚了，这里我没有列出step 3相关的代码是因为这部分只是初始化或赋值一些执行动画需要的中间数据，process()方法最终会执行到RecyclerView.animateDisappearance()方法：

    private void animateDisappearance(...) {
        addAnimatingView(holder);
        holder.setIsRecyclable(false);
        if (mItemAnimator.animateDisappearance(holder, preLayoutInfo, postLayoutInfo)) {
            postAnimationRunner();
        }
    }
这里的animateDisappearance()会把一个动画与ItemView绑定，并添加到待执行队列中， postAnimationRunner()调用后就会执行这个队列中的动画，注意方法addAnimatingView()：

    private void addAnimatingView(ViewHolder viewHolder) {
        final View view = viewHolder.itemView;
        ...
        mChildHelper.addView(view, true);
        ...
    }
这里最终会向ChildHelper中的一个名为mHiddenViews的集合添加给定的ItemView，那么这个mHiddenViews又是什么东西？上节中的getViewForPosition()方法中有个getScrapViewForPosition()，作用是从scrapped集合中获取ItemView：

    ViewHolder getScrapViewForPosition(int position, int type, boolean dryRun) {
        ...
        View view = mChildHelper.findHiddenNonRemovedView(position, type);
        ...
    }
接下来是findHiddenNonRemovedView()方法：

    View findHiddenNonRemovedView(int position, int type) {
        final int count = mHiddenViews.size();
        for (int i = 0; i < count; i++) {
            final View view = mHiddenViews.get(i);
            RecyclerView.ViewHolder holder = mCallback.getChildViewHolder(view);
            if (holder.getLayoutPosition() == position && !holder.isInvalid() && !holder.isRemoved()
                    && (type == RecyclerView.INVALID_TYPE || holder.getItemViewType() == type)) {
                return view;
            }
        }
        return null;
    }
Oops！看到这里就我之前所讲的scrapped集合联系起来了，虽然绕了个圈。所以这里就论证我之前对于scrapped集合的理解。
　　文章到这里也快结束了，最后关于动画，本节提到的对数据集的4种操作，在DefalutItemAnimator中给出了对应的默认实现，就是改变透明度，实现淡入淡出效果。如果要自定义ItemView的动画可以参考这里的实现来做。好了，以上就是我对于RecyclerView的全部剖析了，也许还有我没有提及的方面，或是我讲错的地方，欢迎指正。

####结束语
　　之所以写这篇文章，是因为之前一直没有找到关于RecyclerView实现原理上分析的文章，找到的都是怎么使用的，所有写下本文，希望能对此感兴趣的同学有些许帮助。特别注意，本文还没有让其他人校验过，所以，希望哪位大神可以帮我做下校验，不胜感激！




> Written with [StackEdit](https://stackedit.io/).