###前言
　　本文将介绍股票K线图的实现方案，项目名为KLineChart，github地址https://github.com/zengzg/KLineChart。

###需求分析
　　K线图一般分为日K、周K、月K，显示的内容有开/收盘价、最高/低价、成交量，额外信息为均线(ma5/10/20)。例如，日K图中就为当日开/收盘价、最高/低价、成交量和5/10/20日均线。K线图支持滑动，滑动过程中，动态改变最高最低价（和成交量）；放大缩小可以由两根手指触发，双击也可以放大K线图；节点应该由右向左按时间倒序分布，支持“加载更多”。

###概要设计
　　无论是日K还是周K、月K显示的内容的模型其实是一样，只是采样周期不同，所以节点可以封装为同一个，名为Entry，包含属性：open、close、high、low、volume、ma5、ma10、ma20。显然K线图会同时显示多个节点，那就封装为EntryData，除了封装有一个节点集合外，还封装了对集合的所有相关操作，比如计算最大最小价、添加节点等。为了方便使用，可以将K线图做成一个自定义控件，并且可以将所有绘制工作封装在一起，名为Renderer，这个控件名当然就叫KLineChart了，由于滑动相关的实现与Android触摸事件处理息息相关，所以就写在KLineChart类中了。

###详细设计
　　标题只是个噱头，我也不打算将每行代码都在这篇文章中一一细说，本小说就实现中的2个重点作讲解，具体代码大家可以去查看源码。
　　首先要说的第一点就是绘制Entry。对于Entry的绘制，难点在于坐标映射，也就是说要将Entry集合一一计算出最终在canvas上绘制的坐标。这种映射逻辑可以用3个Matrix表示，下面我一一说明这3个Matrix的作用。
　　value matrix：坐标映射的第一步是将所有节点均匀分布在整个canvas绘制区域内。对于x轴，我们要计算的只是拉伸量，可以用width/entrise.size表示，entrise表示的就是Entry集合；对于y轴，除了拉伸量外还有个平移量，这是因为所有节点的Y值并不是从0开始的，可以用height/yValueRange表示，yValueRange就是所有节点的Y值区间（最大值-最小值）；最后由于要实现从右到左分布节点，而且要y值越小的y坐标越大，可以将拉伸量设置为负数，并平移width(height)距离：

    
    mMatrixValue.postTranslate(0, -yMin);
    mMatrixValue.postScale(-scaleX, -scaleY);
    mMatrixValue.postTranslate(rect.width(),rect.height());
这样value matrix就设置完成了，下面给张图帮助理解：
![enter image description here](https://raw.githubusercontent.com/zengzg/blog/master/android/images/IMG_20160413_163700899.jpg)

　　touch matrix：通过上面的value matrix变换后，我们就可以把当前集合中的所有节点映射到canvas上了，下一步拉伸（并平移）上面映射出的图形，因为往往一屏并不能显示出所有节点，为了支持滑动或者说当集合元素个数大于我们设置可绘制节点数时，就要对上一步变换后的图形作拉伸，x轴拉伸量可以用entries.size/drawCount表示，drawCount就是我们设置的可绘制节点数量：

     mMatrixTouch.postScale(scaleX, scaleY);
     minTouchOffset = 0;
     maxTouchOffset = candleRect.width() * (scaleX - 1f);
     mMatrixTouch.postTranslate(-maxTouchOffset, 0);
其中y轴不需要拉伸，所以scaleY取1。有了这个x轴的拉伸量后，也就可以计算出x轴的最大最小平移量了，如上代码所示。下面给张图帮助理解：
![enter image description here](https://raw.githubusercontent.com/zengzg/blog/master/android/images/IMG_20160413_170932712.jpg)

 　　offset matrix：这个matrix相对来说较为简单，它的作用是上面变换后的图形作平移，空出绘制y轴和x轴的区间。

    mMatrixOffset.postTranslate(offsetX, offsetY);
这个平移大小y轴并没有特殊限制了，而x轴的平移大小应该和y值的绘制区间相关。下面给张图帮助理解：
![enter image description here](https://github.com/zengzg/blog/raw/master/android/images/IMG_20160413_172129583.jpg)

　　通过上面3个矩阵变换后就可以得到最终的绘制点了，其实这本可以用1个矩阵来表示的，之所以分成3个最主要的原因是滑动过程中，我们只需要更新touch matrix的x轴的平移量便可，而平移的变化量自然就是由手指滑动距离来计算的了。
　　下面就开始介绍实现中的另一个难点了——滑动。与RecyclerView相同，定义滑动分为2部分scroll与fling，其中scroll是由触屏事件ACTION_MOVE触发的，fling是由触屏事件ACTION_UP触发的。
　　我们先通过ViewConfiguration获得**启动滑动的最小位移量**，名为touch slop，这个常量的用法是：当滑动偏移量首次大于这个值时，之后的ACTION_MOVE事件，才能识别为用户滑屏事件。当将ACTION_MOVE视为滑屏事件时，计算滑动偏移量dx，然后将这个偏移量作为x轴的平移增量更新touch matrix：

    mMatrixTouch.getValues(matrixValues);

    matrixValues[Matrix.MTRANS_X] += -dx;
    matrixValues[Matrix.MTRANS_Y] += dy;

    if (matrixValues[Matrix.MTRANS_X] < -maxTouchOffset) {
      matrixValues[Matrix.MTRANS_X] = -maxTouchOffset;
    }
    if (matrixValues[Matrix.MTRANS_X] > 0) {
      matrixValues[Matrix.MTRANS_X] = 0;
    }

    mMatrixTouch.setValues(matrixValues);
这样再通过3个matrix变换后得到的Entry坐标就是我们滑动过后的新坐标，最后重绘UI。
　　fling的滑动是借助Scroller类实现的。捕获ACTION_UP事件，通过VelocityTracker类计算得到事件触发时的滑动初速度，在K线图中只用处理x轴的即可，这个初速度我命名为velocityX，对velocityX作边界检测，这个很好理解，速度不能太快或太慢，边界值或叫阀值同样可以通过ViewConfiguration获得；因为fling是一个自发的连续性动画，方法View.postOnAnimation(Runnable)可以使给定的Runnable对象在下一帧绘制时执行，我们可以利用这个机制，来实现一个“类递归”功能，以驱动fling的“自发及连续性”，将这个功能封装在类ViewFlinger中：

    class ViewFlinger implements Runnable{
      public void run{
        final int x = scroller.getCurrX();
        final int y = scroller.getCurrY();
        final int dx = x - mLastFlingX;
        final int dy = y - mLastFlingY;
        mLastFlingX = x;
        mLastFlingY = y;
        scroll(dx, 0);

        if (!scroller.isFinished()) {
        postOnAnimation();
        }
      }
      public void fling(velocityX，velocityY){
        scroller.fling(0,0,velocityX,velocityY,...);
        postOnAnimation();
      }
    }
上面就是ViewFling中的关键代码了，可以看出方法fling()就是触发这个“类递归”执行的地方，然后在这个Runnable执行过程中，调用的scroll()方法其实就是上文所讲的scroll过程中更新touch matrix并重绘UI的方法，只是给定的x轴平移增量是由Scroller计算得到的。至此K线图实现的2个难点就讲完了，如有不对，欢迎指正。最后再给张相对完整的演示图：
![enter image description here](https://raw.githubusercontent.com/zengzg/blog/master/android/images/device-2016-04-13-204038.gif)

###结束语
　　在实现基本的K线图功能中，对于坐标映射与滑动这2个难点的实现方案，其实并不是我想出来的。其中，坐标映射的方案来自开源项目MPAndroidChart，而滑动的方案取自RecyclerView。这就叫学以致用:)


> Written with [StackEdit](https://stackedit.io/).