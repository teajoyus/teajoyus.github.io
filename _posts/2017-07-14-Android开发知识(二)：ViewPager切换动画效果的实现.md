---
tags: Android 
read: 543   
---
   可能读者觉得怎么第二篇就讲到ViewPager了，在这里说明一下：由于这是我重新整理的笔记，所以在 《Android重新整理笔记系列》的博文中并没有前后的顺序，我哪天觉得某个知识点需要记录下来，我就会写出来。方便之后复习之用。
      题目讲的是动画的实现，所以这里我们偏向于动画，而不是ViewPager的基础知识。对于不熟悉ViewPager的读者，请在阅读之前先了解一下什么是ViewPager。

首先，我们先做出一个有5个fragemnt页面并可以左右切换的Activity，具体实现如下：
1、先定义Activity的xml布局如下，这里我们使用了v4包的ViewPager，并且让它铺满屏幕。
```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical" android:layout_width="match_parent"
    android:layout_height="match_parent">
<android.support.v4.view.ViewPager
    android:id="@+id/pager"
    android:layout_width="match_parent"
    android:layout_height="match_parent"></android.support.v4.view.ViewPager>
</LinearLayout>
```

2、定义fragment的xml布局如下，为了方便，我们在布局里面只加入了一个TextView，用来显示当前的页面是第几个页面

```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/content"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    >
    <TextView style="?android:textAppearanceMedium"
        android:id="@+id/tv"
        android:padding="16dp"
        android:lineSpacingMultiplier="1.2"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
         />
</LinearLayout>
```

3、新建一个Fragment，在这里我们采用接收Argument的方式来接收一个索引，然后把索引显示到TextView上，并且根据索引改变背景颜色，让切换的效果更直观一点。

```
public class Fragment1 extends Fragment {
    private static final String TAG = "Fragment1";

    @Nullable
    @Override
    public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        View v = inflater.inflate(R.layout.fragment,null);
        TextView tv = (TextView) v.findViewById(R.id.tv);
        int p = getArguments()!=null?getArguments().getInt("position"):0;
        Log.i(TAG, "onCreateView: "+p);
        v.setTag(p);
        tv.setText(p+"");
        switch (p){
            case 0:v.setBackgroundColor(Color.parseColor("#ffff00f0"));break;
            case 1:v.setBackgroundColor(Color.parseColor("#fff0f000"));break;
            case 2:v.setBackgroundColor(Color.parseColor("#ffff0f00"));break;
            case 3:v.setBackgroundColor(Color.parseColor("#fff0ff00"));break;
            case 4:v.setBackgroundColor(Color.parseColor("#fff0f0f0"));break;
        }
        return v;
    }
}
```

4、在 Activity上，把我们的fragment作为ViewPager的子view适配上去，由于我们用的是Fragment，所以这里我们继承了一个
FragmentPagerAdapter，继承的实现代码如下：

```
class MyScreenSidePagerAdpter extends FragmentPagerAdapter{

        public MyScreenSidePagerAdpter(FragmentManager fm) {
            super(fm);
        }

        @Override
        public Fragment getItem(int position) {
            Fragment fragment = new Fragment1();
            Bundle bundle = new Bundle();
            bundle.putInt("position",position);
            fragment.setArguments(bundle);
            return fragment;
        }

        @Override
        public int getCount() {
            return MAX_PAGE;
        }
    }
```

可以看到上面，在继承FragmentPagerAdapter时我们最少只需要重写getItem和getCount这两个方法，由于FragmentPagerAdapter需要传入一个FragmentManager对象，所以也必须提供一个带有FragmentManager对象的构造方法。


在getItem方法中，我们创建出了Fragment并且把position塞到fragemnt中，这里的position就是上文我们写Fragment时，这句代码：
` ` ` int p = getArguments()!=null?getArguments().getInt("position"):0;` ` ` 
来获取到position。
在getCount方法中我们返回了MAX_PAGE,这个是在Activity中定义的常量，值为5，表示共有5个页面。

最后，我们在Activity的onCreate方法中，把我们继承FragmentPagerAdapter的适配器，适配给ViewPager,相关代码如下：

```
 protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main1);
         pager = (ViewPager) findViewById(R.id.pager);
        FragmentPagerAdapter adapter = new MyScreenSidePagerAdpter(getSupportFragmentManager());
        pager.setAdapter(adapter);

    }
```

好了，运行一下看看效果： 

![页面切换](http://img.blog.csdn.net/20170714094935226?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGNfbWlhbw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

关于ViewPager实现页面的滑动我们已经做完了，接下来让我们来给ViewPager添加一下切换时候的动画效果吧。
要做出有区别于默认的滑动效果的切换动画，我们需要实现ViewPager.PageTransformer接口。

PageTransformer接口只有一个transformPage方法，该方法有两个参数：第一个参数是当前页面的View，第二个是当前页面在滑动中所处的一个位置position。
我们先说说这个position。

在我们看到的当前ViewPager显示的页面，它的position从左到右的区间是[0，1]，也就说如果当我们从左到右滑动当前页面的过程中，页面刚刚要向右移动时position为0，然后逐渐增加，当划到最右边切换掉了当前页面时，position增加到为1.

在这个过程中，会不断的调用transformPage方法。也就是因为如此，我们才可以在transformPage方法内，根绝position的变化来改变页面的透明度、偏移量、缩放量等等。

而这个过程并不是只有当前页面在调用transformPage方法，默认的情况下，当前页面的左右两个页面也会调用这个方法，当然如果当前页面是第一个或者最后一个，那就只有右(左)页面在调用。

那么，假如当前页面不是第一个也不是最后一个，也就是说它有左和右两个页面的情况下，左右页面的position变化如下：
在刚要从左到右滑动的过程中，左页面的position的区间是[-1,0]，刚到0的时候意味着刚好切换到了左页面，同理右页面的position的区间是[1,2]，在切换完成的时候，右页面的position刚好变成了2。

以上关于position的说法，我们通过运行查看log为证。
首先实现PageTransformer接口，相关代码如下：

```
public class ZoomOutPageTranformer implements ViewPager.PageTransformer {
    private static final String TAG = "ZoomOutPageTranformer";

    @Override
    public void transformPage(View page, float position) {
       		Log.i(TAG, "transformPage current page:"+page.getTag()+" , position: "+position);	｝
｝
```

修改下我们的Activity，在原来的写法中，我们增加一句setPageTranformer的方法来添加我们刚才实现的切换效果

```
protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main1);
         pager = (ViewPager) findViewById(R.id.pager);
        FragmentPagerAdapter adapter = new MyScreenSidePagerAdpter(getSupportFragmentManager());
        pager.setAdapter(adapter);
        pager.setPageTransformer(true,new ZoomOutPageTranformer());//设置自定义的切换效果

    }
```

看代码而知，我们在重写的过程中，只是打印出position和当前页面索引的log，其中的page.getTag()，tag由Fragment1中的setTag从而得到索引。

然后运行下，第一个是页面0，我们先切换到页面1，然后再在页面1中从做到右滑动，查看log ，截图如下：
![刚滑动时的log](http://img.blog.csdn.net/20170714095026375?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGNfbWlhbw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![滑动完成的log](http://img.blog.csdn.net/20170714095056241?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGNfbWlhbw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
  由于这个过程中，该方法执行了太多次，所以我们只抓取最先的log和最后的log截图，
      第一张截图是刚滑动的时候各页面的position，第二张截取圈出来的是滑动结束后的各页面的position：
  当前页面（page：1）position是从0.009722223变成1.0
  而左页面（page：0）position是从-99027777变成了0.0
  而右页面（page：2） position是从1.0069444，到最后变成了2.0

  这里论证了我们前面提到的当前页面从0变化到1，左页面从-1变化到0，右页面从1变化到2.
当然，在这里只是为了说明左右页面，才事先切换到第二个页面去，不然的话启动时加载第一个页面，并没有左界面，此时就只有当前页面和右页面在调用这个方法，
当到了最后一个页面时，也就只有当前页面和左页面在调用。
 
到此，我们已经说明了position变化的相关过程，正是因为如此，我们才能通过position的变化来动态改变页面的透明度、偏移量、缩放量这些参数达到动画的效果
 
首先我们先放以下几张截图，这是从页面0滑到页面1的截图。再来看看这个效果的实现：
![滑动效果](http://img.blog.csdn.net/20170714095112743?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGNfbWlhbw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


 由于录制工具的不足，看上去影响效果，不过这并不影响我要说的，这个效果就是当切换的时候，当前页面会渐变缩放并且渐变透明，另外一个页面则从半透明渐变到透明状态，并且放大到屏幕大小。那，这是怎么做到的呢。
我先把实现代码贴上来：

```
package com.example.study.study1;

import android.support.v4.view.ViewPager;
import android.util.Log;
import android.view.View;

/**
 * Created by linmh on 2017/7/13.
 */

public class ZoomOutPageTranformer implements ViewPager.PageTransformer {
    private static final String TAG = "ZoomOutPageTranformer";
    private static final float MIN_SCALE = 0.75f;
    private static final float MIN_ALPHA = 0.5f;
    @Override
    public void transformPage(View page, float position) {
        Log.i(TAG, "transformPage position: "+position);
        Log.i(TAG, "transformPage page: "+page.getTag());
        int pageWidth = page.getWidth();
        int pageHeight = page.getHeight();

        if(position<-1){
            page.setAlpha(0f);
        }else if(position<=1){
            //得到一个scale因子
            float scaleFactor = Math.max(MIN_SCALE,1-Math.abs(position));
            // 把页面缩放到因子大小
            page.setScaleX(scaleFactor);
            page.setScaleY(scaleFactor);
            // 改变透明度
            page.setAlpha(Math.max(MIN_ALPHA ,1-Math.abs(position)));
            //增加一个偏移量，使得更易滑动，提升用户体验
            float vertMargin = pageHeight * (1 - scaleFactor) / 2;
            float horzMargin = pageWidth * (1 - scaleFactor) / 2;
            if(position<0){
                page.setTranslationX(horzMargin - vertMargin / 2);
            }else{
                page.setTranslationX(horzMargin + vertMargin / 2);

            }
        }else {
           page.setAlpha(0);
        }
    }
}
```

从上面代码分析，首先我们先获取页面的宽度和高度
 

```
 int pageWidth = page.getWidth();
 int pageHeight = page.getHeight();
```

然后对position进行判断，当position小于-1，这是什么情况呢，通过我们上面的position分析可以得知，是当前页面的左页面被向左滑动了，这里因为我们只是在屏幕上显示出了ViewPager的一个view 所以也看不到他左右两个view。如果是设置成能看到当前、左、右三个view的话，那么这里就起作用了，也就是把左页面给完全透明化。最后面的position大于1也是这个做法，也就是说随着滑动的过程，消失掉左右原来的页面。

我们重点是看position在这个[-1,1]这个变化区间的过程，当我们把当前页面向左移动过程中，position的变化区间是从0变化到-1，也就是执行这一段代码：

```
        }else if(position<=1){
            //得到一个scale因子
            float scaleFactor = Math.max(MIN_SCALE,1-Math.abs(position));
            // 把页面缩放到因子大小
            page.setScaleX(scaleFactor);
            page.setScaleY(scaleFactor);
            // 改变透明度
            page.setAlpha(Math.max(MIN_ALPHA ,1-Math.abs(position)));
            //增加一个偏移量，使得更易滑动，提升用户体验
            float vertMargin = pageHeight * (1 - scaleFactor) / 2;
            float horzMargin = pageWidth * (1 - scaleFactor) / 2;
            if(position<0){
                page.setTranslationX(horzMargin - vertMargin / 2);
            }else{
                page.setTranslationX(horzMargin + vertMargin / 2);

            }
        }
```

从上面得知，首先获取一个尺寸的比例因子，他最小是MIN_SCALE，被定义为0.75f。

   他随着一开始的position从0递减到-1，那么它的变化区间就为1到0.75，有了这个变化因子，我们直接设置scaleX和scaleY，在这个过程的效果是当前屏幕在切换过程中，当前页面大小从width变化到0.75*width，被缩放了一些。 

   然后，我们根据同样的方法求得一个透明度的变化因子，从Math.max(MIN_ALPHA ,1-Math.abs(position)这句代码可以看出是从1变化到MIN_ALPHA的过程，MIN_ALPHA我们定义为了0.5f，也就是透明度从1递减到0.5，我们看到的过程效果是切换过程中，当前页面的透明度从不透明变化到了半透明。

接下来我们再来求一个偏移量，其实这个偏移量在这个实现动画的过程中并不是必要的，之所以加进来是因为交互体验上。

  默认情况下，用户需要左右滑动大半个屏幕才能气到切换的效果，如果滑动距离不够页面就切换不过去，这样导致用户体验极差。所以我们加进了setTranslationX，让用户在向左滑动的过程中页面随着移动，然后通过translationX的偏移，页面会向左移动的更明显。同理向右也是如此。

```
float vertMargin = pageHeight * (1 - scaleFactor) / 2;
float horzMargin = pageWidth * (1 - scaleFactor) / 2;
```

  在这两句代码中，scaleFactor我们前面已经提到了他的区间是1到0.75，那么(1 - scaleFactor) / 2得到的区间则是[0,0.125]，求出的vertMargin 的区间则是[0，0.125*pageHeight]，同理horzMargin 则是[0,0.125*pageWidth]。

然后horzMargin - vertMargin / 2后的区间则是[0，0.125pageWidth-0.0625*pageHeight]
为什么这么写呢，这是参考了官方的写法。也就是最终的margin是通过宽度的margin减去一半的高度margin，

这样子做的效果是：如果只设置horzMargin ，那么当我们像左滑动的时候，越到左，水平上每次偏移量就越加多，加多到最后趋近于1/8的宽度，而加进了horzMargin -vertMargin /2，那么在水平上就加快左滑的过程中加快偏移的速度被高度所影响，而保持了一个相对缓和的速度。这样子在手机上面的交互效果就好了很多。

当然，通过设置水平的偏移量来加快左滑，这并不是唯一或者说固定的做法，你也可以通过你想要的方式来定义偏移量。
 
说到这里，也差不多到尾了。你有没有发现，代码中是position变化区间是[-1，1],我们才讨论了[-1,0]呢，还没讨论[0,1]呢。
其实这个[0，1] 区间的过程，正是右界面在左移到替换了当前界面的位置。
 
其实道理也一样，读完了上面的分析之后，你是否也能把[0,-1]这个过程分析出来呢。

其实这两个过程，变化的是factor，也就是变化因子，比如说尺寸的因子，左滑时当前页面是从1变化到0.75，导致当前页面被缩放了一些。透明度也是从1变化到0.5f，导致当前页面被渐变到半透明。而与此同时，右界面也在左移到当前界面来，他的因子则是反过来，变成尺寸因子是0.75到1，透明度是0.5到1，所以我们看到的效果才是左滑的时候，当前页面缩小变半透明，右页面从缩小和半透明变化到全屏幕和不透明。
 
当然，在右滑的过程中，分析也是一样的。只是相对于某个页面来说，他的变化因子比如尺寸因子，是1变化到0.75 ，还是0.75变化到1.当前页面和切换到的页面他们的效果是想法，才给我们造成了一种缩放淡入淡出的效果。

讲到这里就结束啦。由于知识水平有限，以上若有写错或者理解错的地方，还烦请读者在评论中勘误出来，让我们共同学习。







