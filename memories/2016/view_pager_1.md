Android ViewPager Tips 1: 循环滑页，自动滑页及其手势冲突
===

#### 起源
新年了，说好的计划今年写的50篇 blog 呢，到现在还没有写一篇。想起来，也不禁有些惭愧，其实老早就琢磨着写些东西的，理想状况就像陶渊明在[《五柳先生传》][1]里说的那样，「好读书，不求甚解，每有会意，便欣然忘食」。时常写些东西，不求写得很好，但是每当发现好的东西，灵感到来时，就应该写写。

话不多说，写些什么呢？算上自己在学校学习和发布 Android 应用到工作了从事 Android 开发到现在，也快有3年的时间了，在这期间，还是积累和研究了不少东西的。这些东西，随着时间的推移，不是忘了就是细节上的东西模糊了。反倒是要重新思考写一遍或者再去 Google 查阅，时间浪费了不说，在查阅的过程中，其实发现很多东西都来自别人 blog 的分享。嗯，多写些东西，利己利人。

那么第一篇，就从今天在工作上写的一个控件效果开始，[ViewPager][2] 的循环滑页及其额外的相关技巧，这是第一部分。

#### 循环滑页
ViewPager 是应用中常见的控件，经常会用在 banner 上，用来提示一些专题啦，头条等等，吸引用户的注意，具体的用法就不多说了。这个控件默认情况下已经很好用了，写好相应的 ``Adapter`` 后其余的手势滑页等效果控件都帮你做好了。但是有一个问题，当页面出于边界时``[0, count - 1]``，他就没法滑动了。如果这时候产品要求**滑动到最后一页的时候仍然能够继续向右滑到第一页**，这个时候就得自己想办法了。

本质上，ViewPager 的 Adapter 是一个线性列表结构，通过重载``Adapter.getCount()``确定一共有多少页，再通过``Adapter.instantiateItem(ViewGroup container, int position)``来加载第 position 页的布局，其中 position 位于 ``[0, count)`` 区间中。

看到这里，最初我的想法是构造一个环状的数据结构，当当前页为页尾的时候，下一页自然就是页头了，然而并没有用。因为确定了页数之后，ViewPager 滑页的机制就是判断下一页是否在 ``[0, count)`` 中，不在该区间的也就没有意义了。

换一个思路，既然页数是确定的，那么要实现一个可以循滑到下一页的效果，可以认为这个页数是无限大的，也就是滑动区间为 [0, ∞) , 然后以页数 count 来分段，即 **[0, count), [count, 2×count), ..., [n×count, (n+1)×count)**

那么问题来了，**如何在程序中表示一个无限大的数呢？极限中经常说，取一个足够大的数，然后xxx**。由于**Adapter.getCount()**返回值为 int, int 的最大值是``2ˇ31 - 1``，大概是地球人的1/3，也就是21亿左右，这个数字从实际场景来说，基本上可以认为是无限大了，如果用户真的不间断的滑这么多页，恭喜他，他可以去申请吉尼斯世界记录了 :)

把思路1和2结合起来，不难得出以下核心代码：

```java
public class InfiniteAdapter extends Adapter {
  private final int count = REAL_COUNT;

  @Override public int getCount() {
    return Integer.MAX_VALUE;
  }

  @Override public Object instantiateItem(ViewGroup container, int position) {
    final int i = position % REAL_COUNT;
    // use i to manipulate layout ...
  }
}
```

#### 自动滑页
记得刚毕业的时候，公司就要求在首页的 banner 上实现一个定时自动滑页的效果，也就是用户即使用户没有滑页，那么应用也能够自动滑页。这个效果也是非常常见的了，也不难实现，当时我是实现了这个效果，但是没有实现循环滑页的效果，以至于滑动最后一页后，突变到第一页，挺怪的，顺带说一下，玩过不少 Android 的应用类似的场景都是突变到第一页的...

定时自动播放，定时，看到这词，最直接的做法就是使用 Java 标准库的 [Timeri][3] 类来定时播放了。我最开始也是这样做的，it works。然而，这个类有诸多缺陷。首先 UI 线程同步问题，接着初始化后不能 reset 状态，又得重新 new 一个，代码不易维护的同时也带来了另一个更严重的问题，内存泄露。``Timer``这个类我记得是在文档中说过，不会及时地释放内存。这里先引出下一个主题，**用户主动滑页和自动轮播的冲突问题：当用户手势滑页的时候需要屏蔽或者说暂停自动播放，否则就自动播放**。这个时候如果仍然使用``Timer``的话，那么就得不间断的 new Timer，内存泄露的问题就越来越严重。

幸运的是，Android 自带了解决问题的办法，这个方法每个开发者都应当知道，``Handler.postDelay(Runnable, long)``方法，我最喜欢用这个方法去处理闪屏页的定时跳转问题。并且，任意的 ``View`` 都有一个 ``Handler`` 用来处理 UI 线程同步的问题。所以我们不需要额外的构造 ``Handle`` 而直接调用 ``View.postDealy()`` 以及 ``View.removeCallbacks()`` 方法来实现。具体来说，就是在 ``postDelay()`` 的 ``Runnable`` 中继续调用 ``postDelay()``，从某种意义上说这也算是一种递归吧，最后**记得在 Activity 或者 View 中的相关回调调用 ``removeCallbacks()``，否则可能会存在内存泄露或者无意义的 UI 绘制**。

用代码来说：

```java
private Runnable mTimer = new Runnable {
  @Override public void run() {
    mViewPager.setCurrentItem(mViewPager.getCurrentItem() + 1);
    mViewPager.postDelay(this, DELAY_MILLIS);
  }
}

// on populating layout or other cases, start timer
private void populateLayout() {
  // ...
  mViewPager.postDelay(mTimer, DELAY_MILLIS);
}

// remove timer in some cases, such as on View.onDetachFromWindow()...
@Override protected void onDetachedFromWindow() {
  super.onDetachedFromWindow();
  // ...
  mViewPager.removeCallbacks(mTimer);
}
```

#### 手势和自动滑页的冲突
这个问题前面已经提及了，解决办法其实也很简单，就是**当用户在于 ViewPager 交互的时候停止播放，否则自动播放**。好像，当时并没有很好的解决这个问题...今天刚好遇到这个问题，就写了一个，直接上代码吧 :)

```java
public class InteractingViewPager extends ViewPager {
  public InteractingViewPager(Context context) {
    super(context);
  }

  public InteractingViewPager(Context context, AttributeSet attrs) {
    super(context, attrs);
  }

  @Override public boolean onTouchEvent(MotionEvent ev) {
    switch (ev.getAction()) {
      case MotionEvent.ACTION_DOWN:
      case MotionEvent.ACTION_MOVE:
        if (onPageGestureListener != null) {
          onPageGestureListener.onAttach();
        }
        break;
      case MotionEvent.ACTION_UP:
      case MotionEvent.ACTION_CANCEL:
        if (onPageGestureListener != null) {
          onPageGestureListener.onDetach();
        }
        break;
    }
    return super.onTouchEvent(ev);
  }

  private OnPageGestureListener onPageGestureListener;

  public void setOnPageGestureListener(OnPageGestureListener onPageGestureListener) {
    this.onPageGestureListener = onPageGestureListener;
  }

  public interface OnPageGestureListener {
    void onAttach();

    void onDetach();
  }
}
```

使用方式也很简单:

```java
mViewPager.setOnPageGestureListener(new InteractingViewPager.OnPageGestureListener() {
  @Override public void onAttach() {
    mViewPager.emoveCallbacks(mTimer);
  }

  @Override public void onDetach() {
    mViewPager.ostDelayed(mTimer, PLAY_DELAY);
  }
});
```

#### One More Thing
欢迎阅读并留言，敬请期待下一篇关于 ViewPager 的技巧 :)

#### EOF
\#Tech, \#Android   
2016-01-06, Shenzhen, a little warm


[1]: http://baike.baidu.com/link?url=QMeakKDuiC946nRTN0ZnQXhoin0a8-nLeoqEG1zy6T8NAjIV3Hx6o2neNxtVcJ3M4pm6TkMd7RNtB8V6riUI-K
[2]: http://developer.android.com/reference/android/support/v4/view/ViewPager.html
[3]: http://developer.android.com/reference/android/support/v4/view/ViewPager.html
