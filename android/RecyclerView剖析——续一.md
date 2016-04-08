###前言
　　前文已经在整体上对RecyclerView的实现作出了剖析，但是有些细节上，我并没有做太过深入的解释，“续一”将针对RecyclerView的动画作更深入剖析。同样，文中所示源码版本为23.2.0。本文欢迎转载，不需要注明出处。

###pre&post layout
　　在RecyclerView中存在一个叫“预布局”的阶段，当然这个是我自己作的翻译，本来叫pre layout，与之对应的还有个叫post layout的阶段，它们分别发生在真正的子控件测量&布局的前后。其中pre layout阶段的作用是记录数据集改变前的子控件信息，post layout阶段的作用是记录数据集改变后的子控件信息及触发动画。

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
方法dispatchLayout()会在RecyclerView.onLayout()中被调用，其中dispatchLayoutStep1就是pre layout，dispatchLayoutStep3就是post layout，而dispatchLayoutStep2自然就是处理真正测量&布局的了。
　　首先来看看pre layout时都记录了什么内容：

    private void dispatchLayoutStep1() {
        ...
        if (mState.mRunSimpleAnimations) {
            // Step 0: Find out where all non-removed items are, pre-layout
            int count = mChildHelper.getChildCount();
            for (int i = 0; i < count; ++i) {
                final ViewHolder holder = ...
                ...
                final ItemHolderInfo animationInfo = mItemAnimator
                        .recordPreLayoutInformation(...);
                mViewInfoStore.addToPreLayout(holder, animationInfo);
                ...
            }
        }
        ...
    }
类ItemHolderInfo中封闭了对应ItemView的边界信息，即ItemView的left、top、right、bottom值。对象mViewInfoStore的作用正如源码注释：

    /**
     * Keeps data about views to be used for animations
     */
    final ViewInfoStore mViewInfoStore = new ViewInfoStore();
再来看看addToPreLayout()方法：

    void addToPreLayout(ViewHolder holder, ItemHolderInfo info) {
        InfoRecord record = mLayoutHolderMap.get(holder);
        if (record == null) {
            record = InfoRecord.obtain();
            mLayoutHolderMap.put(holder, record);
        }
        record.preInfo = info;
        record.flags |= FLAG_PRE;
    }
由上可已看出RecyclerView将pre layout阶段的ItemView信息存放在了ViewInfoStore中的mLayoutHolderMap集合中。
　　接下来我们看看post layout阶段：

    private void dispatchLayoutStep3() {
        ...
        if (mState.mRunSimpleAnimations) {
            // Step 3: Find out where things are now, and process change animations.
            ...
            for (int i = mChildHelper.getChildCount() - 1; i >= 0; i--) {
                ...
                final ItemHolderInfo animationInfo = mItemAnimator
                        .recordPostLayoutInformation(mState, holder);
                ...
                if (...) {
                    ...
                    animateChange(oldChangeViewHolder, holder, preInfo, postInfo,
                                    oldDisappearing, newDisappearing);
                } else {
                    mViewInfoStore.addToPostLayout(holder, animationInfo);
                }
            }

            // Step 4: Process view info lists and trigger animations
            mViewInfoStore.process(mViewInfoProcessCallback);
        }
        ...
    }
这是addToPostLayout()方法：

    void addToPostLayout(ViewHolder holder, ItemHolderInfo info) {
        InfoRecord record = mLayoutHolderMap.get(holder);
        if (record == null) {
            record = InfoRecord.obtain();
            mLayoutHolderMap.put(holder, record);
        }
        record.postInfo = info;
        record.flags |= FLAG_POST;
    }
与pre layout阶段相同RecyclerView也是将post layout阶段的ItemView信息存放在mViewInfoStore的mLayoutHolderMap集合中，并且不难看出，同一个ItemView（或者叫ViewHolder）的pre layout信息与post layout信息封装在了同一个InfoRecord中，分别叫InfoRecord.preInfo与InforRecord.postInfo，**这样InfoRecord就保存着同一个ItemView在数据集变化前后的信息，我们可以根据此信息定义动画的开始和结束状态**。
![enter image description here](https://raw.githubusercontent.com/zengzg/blog/master/android/images/path5784-3.png)
如上图所示，当我们插入A时，在完成了上文所诉过程后，以ItemView2为例，通过比较它的preInfo与postInfo——都为非空，源码中是以标志位的形式实现的，就可以知道它将执行MOVE操作；而A自然就是ADD操作。下面是ViewInfoStore.ProcessCallback实现中的其中一个方法，它会在mViewInfoStore.process()方法中被调用：

    public void processPersistent(...) {
            ...
            if (mDataSetHasChangedAfterLayout) {
                ...
            } else if (mItemAnimator.animatePersistence(viewHolder, preInfo, postInfo)) {
                postAnimationRunner();
            }
        }
我们知道，RecyclerView中ItemAnimator的默认实现是DefaultItemAnimator，这里我就只以默认实现来说明，这是animatePersistence()方法：

    public boolean animatePersistence(...) {
        if (preInfo.left != postInfo.left || preInfo.top != postInfo.top) {
            ...
            return animateMove(viewHolder,
                    preInfo.left, preInfo.top, postInfo.left, postInfo.top);
        }
        dispatchMoveFinished(viewHolder);
        return false;
    }
当然这个方法在DefaultItemAnimator的父类SimpleItemAnimator中，通过比较preInfo与postInfo的left和top属性分别确定ItemView在水平或垂直方向是否要执行MOVE操作，而上面的方法postAnimationRunner()就是用来触发动画执行的。
　　通过前文我们知道，RecyclerView中定义了4种针对数据集的操作（也可以称为针对ItemView的操作），分别是ADD、REMOVE、UPDATE、MOVE，RecyclerView就是通过比较preInfo与postInfo来确定ItemView要执行哪种操作的，上文我描述了MOVE情况，这个比较过程是在方法ViewInfoStore.process()中实现的，其它情况我就不再赘述了，各位不妨自己去看看。
	　　在DefaultItemAnimator中实现了上面4种操作下的动画。当postAnimationRunner()执行后，会触发DefaultItemAnimator.runPendingAnimations()方法的调用，这个方法过长，我这里只作下解释便可。4种操作对应的动画是有先后顺序的，remove-->move&change-->add，之所以有这样的顺序，不难看出是为了不让ItemView之间有重叠的区域，这个顺序是由ViewCompat.postOnAnimationDelayed()方法通过控制延时来实现的。在DefaultItemAnimator中，REMOVE和ADD对应的是淡入淡出动画（改变透明度），MOVE对应的是平移动画；UPDATE相对来说要复杂一些，是因为它不再是记录同一个ItemView的变化情况，而是记录2个ItemView的信息来作比较，pre layout阶段的信息来自“oldChangeViewHolder”，post layout阶段的信息来自“holder”，这两个对象在dispatchLayoutStep3方法中可以找到，而且，这2个ItemView的动画是同时执行的，所以它对应的动画是：“oldHolder”淡出且向“newHolder”平移，同时“newHolder”淡入。特别说明，前文有提过一个叫scrapped的集合，其实它除了保存REMOVE操作的ItemView，还保存着UPDATE操作中的“oldHolder”！
　　以上就是RecyclerView默认动画的具体实现逻辑了，总结下来就是：当数据集发生变化时，会导致RecyclerView重新测量&布局子控件，我们记录下这个变化前后的RecyclerView的快照（preInfo与postInfo），通过比较这2个快照，从而确定子控件要执行什么操作，最后再实现不同操作下对应的动画就好了。通常我们会调用notifyItemXXX()系列方法来通知RecyclerView数据集变化，这些方法之所以比notifyDataSetChanged()高效的原因就是它们不会让整个RecyclerView重新绘制，而是只重绘具体的子控件，并且通过动画连接子控件的前后状态，这样也就实现了在Material design中所讲的“Visual continuity”效果。

###子控件的测量与布局
　　这一节将对preInfo与postInfo是如果确定（赋值）的，作进一步描述。
　　从前文我们知道，子控件的测量与布局其实在RecyclerView的测量阶段（onMeasure）就执行完了，这样做是为了支持WRAP_CONTENT，具体的方法呢就是dispatchLayoutStep1()与dispatchLayoutStep2()，同样这两个方法也会出现在RecyclerView的布局阶段（onLayout），但并不是说它们就会被调用，这里的调用逻辑是由RecyclerView.State类控制的，它定义了RecyclerView的整个测量布局过程，分为3步STEP_START、STEP_LAYOUT、STEP_ANIMATIONS，具体流程是：初始状态是STEP_START；如果RecyclerView当前在STEP_START阶段dispatchLayoutStep1()会执行，记录下preInfo，将状态改为STEP_LAYOUT；如果RecyclerView在STEP_LAYOUT阶段dispatchLayoutStep2()会执行，测量布局子控件，将状态改为STEP_ANIMATIONS；如果RecyclerView在STEP_ANIMATIONS阶段dispatchLayoutStep3()会执行，记录下postInfo，触发动画，将状态改为STEP_START。每次数据集更改都会执行上述3步。
　　在测量布局子控件的过程中，最重要的莫过于确定布局锚点了，以LinearLayoutManager垂直布局为例，在onLayoutChildren()方法中，会调用updateAnchorInfoForLayout()方法来确定布局锚点：

    private void updateAnchorInfoForLayout(...) {
        if (updateAnchorFromPendingData(state, anchorInfo)) {
            ...
            return;
        }

        if (updateAnchorFromChildren(recycler, state, anchorInfo)) {
            ...
            return;
        }
        ...
        anchorInfo.assignCoordinateFromPadding();
        anchorInfo.mPosition = mStackFromEnd ? state.getItemCount() - 1 : 0;
    }
这里布局锚点的确定方法有3种依据。首先，如果是第一次布局（没有ItemView），这种情况已经在前文有过描述了，这里就不再说明；剩余的2种分别是“滑动位置”与“子控件”，这2种情况都是发生在已经有ItemView时的，而且这里的“滑动位置”是指由方法scrollToPosition()确认的，并赋给了mPendingScrollPosition变量。现在先来看看“滑动位置”updateAnchorFromPendingData()方法：

    private boolean updateAnchorFromPendingData(...) {
        ...
        // if child is visible, try to make it a reference child and ensure it is fully visible.
        // if child is not visible, align it depending on its virtual position.
        anchorInfo.mPosition = mPendingScrollPosition;
        ...
        if (mPendingScrollPositionOffset == INVALID_OFFSET) {
            View child = findViewByPosition(mPendingScrollPosition);
            if (child != null) {
                ...
            } else { // item is not visible.
                ...
            }
            return true;
        }
        ...
        return true;
    }
布局锚点中的mCoordinate与mPosition，在前文描述为起始绘制偏移量与索引位置，再直白点就是屏幕位置与数据集位置，就是告诉RecyclerView从屏幕的mCoordinate位置开始填充子控件，与子控件绑定的数据从数据集的mPosition位置开始取得。上面这个方法中确定“屏幕位置”分为2种情况，就是对应于mPendingScrollPosition是否存在子控件，mCoordinate值的确定我就不再讲述了，无非是一边界判断的语句。
　　下面来看看“子控件”依据的情况，这是updateAnchorFromChildren()：

    private boolean updateAnchorFromChildren(...) {
        ...
        View referenceChild = anchorInfo.mLayoutFromEnd
                ? findReferenceChildClosestToEnd(recycler, state)
                : findReferenceChildClosestToStart(recycler, state);
        if (referenceChild != null) {
            anchorInfo.assignFromView(referenceChild);
            ...
            return true;
        }
        return false;
    }
这种情况也并不复杂，就是找到最外边的一个子控件，以它的位置信息来确定布局锚点，就是方法assignFromView()，我也就不再列出来了。以上就是详细的布局锚点确认过程了。
　　
###结束语
　　本续文是对《RecyclerView剖析》一文的补充，旨在描述RecyclerView实现上更为细致的地方。
　　
> Written with [StackEdit](https://stackedit.io/).