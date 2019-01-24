---
tags: Android 
read: 1056   
---

@[toc]


使用RecycleView的好处
---------------------------
从Android5.0开始，google给我们带来了一个全新的列表组件，叫做RecycleView。使得apk几乎已经抛弃了ListView，之所以RecycleView那么强大，总结的主要原因有如下几点：
1、提供ViewHolder模式，使得开发者真正操作的是ViewHolder，而不是像ListView中的getView，需要开发者自己setTag和view.getTag

2、同时支持列表布局和网格布局，而ListView只能支持列表布局，网格布局需要用GridView。

3、支持瀑布流布局。我们不在需要为实现瀑布流效果而苦恼

4、操作动画。在对列表进行增加、删除时的动画。并且Adapter提供了增加删除某个item的方法

5、性能与拓展性。RecycleView听起来像是回收的view，事实上，RecycleView本身就不关心View相关的显示、View显示什么内容（ViewHolder来管理），View怎么摆放（LayoutManager来管理），也不关心动画（ItemAmator来管理），甚至连分割线它都不管（由ItemDecoration来管理）
而它关心View的回收复用，这跟性能有关系。所以名字用Recycle也是有道理的。这样的好处是，把各个环节工作交付给了不同的类，类似“插件化”。特别方便拓展，自定义各种各样的差异化，而从这其中解耦出来


RecycleView的基本用法
---------------------------
RecycleView并没有集成在SDK中，而是放在了v7包里面，我们添加它的依赖：

```
    implementation 'com.android.support:recyclerview-v7:27.1.1'
```

在布局中添加RecycleView：
```
    <android.support.v7.widget.RecyclerView
        android:id="@+id/recycler_view"
        android:layout_width="match_parent"
        android:layout_height="match_parent">
    </android.support.v7.widget.RecyclerView>
```

```
    //LinearLayoutManager是用来做列表布局，也就是单列的列表
   RecyclerView.LayoutManager layoutManager = new LinearLayoutManager(this);
   rv.setLayoutManager(layoutManager);
    //默认就是垂直方向的
   ((LinearLayoutManager) layoutManager).setOrientation(OrientationHelper.VERTICAL);
   //谷歌提供了一个默认的item删除添加的动画
   rv.setItemAnimator(new DefaultItemAnimator());

   //谷歌提供了一个DividerItemDecoration的实现类来实现分割线
   //往往我们需要自定义分割线的效果，需要自己实现ItemDecoration接口
   DividerItemDecoration dividerItemDecoration = new DividerItemDecoration(this,DividerItemDecoration.VERTICAL);
    rv.addItemDecoration(dividerItemDecoration);

    //当item改变不会重新计算item的宽高
    //调用adapter的增删改差方法的时候就不会重新计算，但是调用nofityDataSetChange的时候还是会
    //所以往往是直接先设置这个为true，当需要布局重新计算宽高的时候才调用nofityDataSetChange
    rv.setHasFixedSize(true);

    //模拟列表数据
    newsList = new ArrayList<>();
    News news;
    for (int i = 1; i < 100; i++) {
        news = new News();
        news.title = "新闻标题内容新闻标题内容新闻标题内容新闻标题内容新闻标题内容";
        news.source = "腾讯新闻" ;
        news.time = "2019-01-17";
        newsList.add(news);
    }
    设置适配器
    NewsAdapter newsAdapter = new NewsAdapter(newsList);
    rv.setAdapter(new NormalAdapterWrapper(newsAdapter));
```

接下来编写NewsAdapter的代码，它需要实现 RecyclerView.Adapter<RecyclerView.ViewHolder>
```
public class NewsAdapter extends RecyclerView.Adapter<RecyclerView.ViewHolder> {
        private List<News> list;

        public NewsAdapter(List<News> list) {
            this.list = list;
        }

        @NonNull
        @Override
        public RecyclerView.ViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
            View v = null;
            v = getLayoutInflater().inflate(R.layout.item, null, false);
            RecyclerView.ViewHolder holder = null;
            holder = new MyViewHolder(v);
            return holder;
        }

        @Override
        public void onBindViewHolder(@NonNull RecyclerView.ViewHolder holder, int position) {
            ((MyViewHolder) holder).title.setText(list.get(position).title);
            ((MyViewHolder) holder).time.setText(list.get(position).time);
            ((MyViewHolder) holder).source.setText(list.get(position).source);
        }

        @Override
        public int getItemCount() {
            return list.size();
        }

    }

    class MyViewHolder extends RecyclerView.ViewHolder {
        public TextView title, time, source;

        public MyViewHolder(View itemView) {
            super(itemView);
            title = itemView.findViewById(R.id.title);
            source = itemView.findViewById(R.id.source);
            time = itemView.findViewById(R.id.time);
        }
    }
```

新增一个item.xml来显示item的内容：
```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="70dp"
   >
  <TextView
      android:id="@+id/title"
      android:layout_width="match_parent"
      android:layout_height="wrap_content"
      android:textSize="18sp"
      android:textColor="#DF000000"
      />
  <TextView
      android:id="@+id/source"
      android:layout_width="wrap_content"
      android:layout_height="wrap_content"
      android:layout_below="@+id/title"
      />
  <TextView
      android:id="@+id/time"
      android:layout_width="wrap_content"
      android:layout_height="wrap_content"
      android:layout_below="@+id/title"
      android:layout_alignParentRight="true"
      />


</RelativeLayout>
```

RecycleView提供的适配器中RecyclerView.Adapter<RecyclerView.ViewHolder> 强制了绑定的ViewHolder
在Adapter中必须实现的三个方法：

//列表页需要知道有多少个条目
public int getItemCount()

//创建一个ViewHolder，我们可以根据viewType的不同而创建不同的ViewHolder实现列表页各种不一样的item
public RecyclerView.ViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType)

//在ViewHolder中绑定数据
public void onBindViewHolder(@NonNull RecyclerView.ViewHolder holder, int position)

而RecyclerView.ViewHolder本身是一个抽象类，我们往往自己继承这个抽象类，然后绑定我们的布局控件对象。继承该类时必须传入一个itemView，表示这个item显示的View


按照上面的步骤，我们就完成了一个列表页。效果如下:


 ![列表页效果](https://img-blog.csdnimg.cn/20190117165716295.png)


使用RecycleView的网格布局
---------------------------
我们可以实现2列的效果，方便起见我们就不修改布局了
我们只需要修改即可：

```
 rv.setLayoutManager(new GridLayoutManager(this,2));
```


效果如下：（虽然两列的新闻看起来好奇怪，将就吧）


 ![网格效果](https://img-blog.csdnimg.cn/20190117170150826.png)



RecycleView的定位和查找
---------------------------
#RecycleView
在文中一开始就说了RecycleView并不负责View的显示，只负责回收。
在滚动定位中，我们要使用的自然是它的LayoutManager
有时候我们想去滚动到某个位置可用：
```
//滚动到第50个item
   rv.getLayoutManager().scrollToPosition(0);
```
但是请注意，这会马上定位到该位置。并没有滚动效果，如果你想平滑滚动的话，可以用：
```
  //平滑滚动到第50个item
  rv.getLayoutManager().smoothScrollToPosition(rv, null, 50);
```

上面的滚动定位并不保证你想要的item会完全显示，也不保证它滚动的位置

当要滚动到的item出现在屏幕后，即使不能显示完全，那么滚动也会停止

如果你想滚动到某个位置，并且那个位置自动到屏幕顶部。那么我们需要自己做出滚动逻辑
```
int lastVisiablePostion = ((LinearLayoutManager) rv.getLayoutManager()).findLastVisibleItemPosition();
//第一个参数是要在哪个位置上开始滑（固定位置），第二个参数是要把该位置的view滑到屏幕的哪个位置（相对位置）
 ((LinearLayoutManager) rv.getLayoutManager()).scrollToPositionWithOffset(lastVisiablePostion, 0);
```

如果我们想找到某个位置的view，可以用：
```
//得到第20个View
rv.getLayoutManager().findViewByPosition(20);
```

如果我们想知道当前屏幕最顶部的item位置，可以用：
```
int firstVisiablePostion = ((LinearLayoutManager) rv.getLayoutManager()).findFirstVisibleItemPosition();
```

注意以上只是屏幕顶部第一个出现的view位置，并不保证这个view是完全可见的，
如果我们想知道当前屏幕最顶部第一个完全可见的的item位置，可以用：
```
 //第一个完全可见的位置
 int firstCompletelyVisiablePostion = ((LinearLayoutManager) rv.getLayoutManager()).findFirstCompletelyVisibleItemPosition();
```

有了找屏幕顶部的方法，自然也有找屏幕底部的方法：
```
 int lastVisiablePostion = ((LinearLayoutManager) rv.getLayoutManager()).findLastVisibleItemPosition();
//屏幕最后一个完全可见的位置
int lastCompletelyVisiablePostion = ((LinearLayoutManager) rv.getLayoutManager()).findLastCompletelyVisibleItemPosition();
```


RecycleView的Item增加删除
---------------------------
如果我们想要删除掉其中某个Item（如果你使用了 rv.setItemAnimator(new DefaultItemAnimator());则会有删除动画效果）：

```
    public void deleteItem() {
        if(newsList == null || newsList.isEmpty()) {
            return;
        }
        newsList.remove(5);
        rv.getAdapter().notifyItemRemoved(5);
    }
```

当我们想要添加某个Item进来：

```
    public void addNewItem() {
        News news = new News();
        news.title = "新添加的新添加的新添加的新添加的新添加的新添加的";
        news.source = "腾讯新闻" ;
        news.time = "2019-9-17";
        newsList.add(3, news);
        rv.getAdapter().notifyItemInserted(3);
    }
```

如果你想更新多个条目，那么可以用跟ListView的Adapter一样的方法：notifyDataSetChange()方法来通知布局刷新

RecycleView实现不同item布局
---------------------------
在RecycleView的Adapter中提供了这个方法，我们重写它：
```
        @Override
        public int getItemViewType(int position) {
//            return super.getItemViewType(position);
           return position % 3 + 1;
        }
```
这里我们让轮流返回3个不同的viewtype，然后我们再随便编写另外item2.xml,item3.xml布局。
之后的NewsAdapter就变成这样：
```
 public class NewsAdapter extends RecyclerView.Adapter<RecyclerView.ViewHolder> {
        private List<News> list;

        public NewsAdapter(List<News> list) {
            this.list = list;
        }

        @Override
        public int getItemViewType(int position) {
//            return super.getItemViewType(position);
            return position % 3 + 1;
        }

        @NonNull
        @Override
        public RecyclerView.ViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
            View v = null;
            RecyclerView.ViewHolder holder = null;
            if (viewType == 2) {
                v = getLayoutInflater().inflate(R.layout.item2, null, false);
                    holder = new MyViewHolder2(v);
            } else if (viewType == 3) {
                v = getLayoutInflater().inflate(R.layout.item3, null, false);
                holder = new MyViewHolder3(v);
            } else {
                v = getLayoutInflater().inflate(R.layout.item, null, false);
    			holder = new MyViewHolder(v);
            }
      
            return holder;
        }

        @Override
        public void onBindViewHolder(@NonNull RecyclerView.ViewHolder holder, int position) {
        //根据postion得到不同的viewType，根据不同的viewType得到对应的ViewHolder
        //我这里为了简单 就三个ViewHolder都是一样这几个控件和id，只是每个布局的几个控件的位置调整一下以表示差异
        	if(getItemViewType(position)==3){
		            ((MyViewHolder3) holder).title.setText(list.get(position).title);
           			 ((MyViewHolder3) holder).time.setText(list.get(position).time);
           			 ((MyViewHolder3) holder).source.setText(list.get(position).source);
			}else if(getItemViewType(position)==2){
					((MyViewHolder2) holder).title.setText(list.get(position).title);
           			 ((MyViewHolder2) holder).time.setText(list.get(position).time);
           			 ((MyViewHolder2) holder).source.setText(list.get(position).source);
			}else{
		            ((MyViewHolder) holder).title.setText(list.get(position).title);
		            ((MyViewHolder) holder).time.setText(list.get(position).time);
		            ((MyViewHolder) holder).source.setText(list.get(position).source);
            }
        }

        @Override
        public int getItemCount() {
            return list.size();
        }

    }

    class MyViewHolder extends RecyclerView.ViewHolder {
        public TextView title, time, source;
		//这个itemView就是我们传入的itemView，往往是从XML中加载进来的布局
        public MyViewHolder(View itemView) {
            super(itemView);
            title = itemView.findViewById(R.id.title);
            source = itemView.findViewById(R.id.source);
            time = itemView.findViewById(R.id.time);
        }
    }
        class MyViewHolder2 extends RecyclerView.ViewHolder {
        public TextView title, time, source;

        public MyViewHolder2(View itemView) {
            super(itemView);
            title = itemView.findViewById(R.id.title);
            source = itemView.findViewById(R.id.source);
            time = itemView.findViewById(R.id.time);
        }
    }
        class MyViewHolder3 extends RecyclerView.ViewHolder {
        public TextView title, time, source;

        public MyViewHolder3(View itemView) {
            super(itemView);
            title = itemView.findViewById(R.id.title);
            source = itemView.findViewById(R.id.source);
            time = itemView.findViewById(R.id.time);
        }
    }
```

我们改回用LinearLayoutManager

效果图如下（我把另外两个布局改了下来源和时间显示的位置，可以看到每个连续的item之间是不同的布局方式）：

 ![网格效果](https://img-blog.csdnimg.cn/20190117172946469.png)


既然可以定义多种ViewType，那我们可以利用这个来实现头部（HeaderView）和尾部（FootView），因为RecycleView并没有为我们明显提供HeaderView和FootView结构，需要我们自己实现。
我们创建另外一个装饰者适配器，让原有的适配器不需要改动
```
class NewsAdapterDecorator extends RecyclerView.Adapter{
        //在不破坏原有类的情况下，对原有的适配器拓展heeder和foot
        private NewsAdapter adapter;
        private int type;//-1：header -2：foot  other:content
        private boolean hasHeader,hasFoot;
        public NewsAdapterDecorator(NewsAdapter adapter) {
            this.adapter = adapter;
        }

        @Override
        public int getItemViewType(int position) {
        	//新增了个viewType=-1的布局来表示头部
            if(position==0){
                return -1;
                //新增了个viewType=-2的布局来表示尾部，
            }else if(position==getItemCount()-1){
                return -2;
            }
            //列表页还是原先的adapter
            return adapter.getItemViewType(position);
        }

        @NonNull
        @Override
        public RecyclerView.ViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        //头部
            if(viewType==-1){
                RelativeLayout layout = new RelativeLayout(MainActivity.this);
                return new HeaderViewHolder(layout);
                //尾部
            }else if(viewType==-2){
                RelativeLayout layout = new RelativeLayout(MainActivity.this);
                return new FootViewHolder(layout);
            }
            return adapter.onCreateViewHolder(parent,viewType);
        }

        @Override
        public void onBindViewHolder(@NonNull RecyclerView.ViewHolder holder, int position) {
            if(position==0 && holder instanceof HeaderViewHolder){
              ((HeaderViewHolder)holder).tv.setText("我是头部");
                hasHeader = true;
            }else if(position==adapter.getItemCount()+1&& holder instanceof FootViewHolder){
                ((FootViewHolder )holder). tv.setText("我是尾部");
                tv.setGravity(Gravity.CENTER);
                hasFoot = true;
            }else{
            //注意传入给原先的adapter的postion要-1，因为有了头部。
                adapter.onBindViewHolder(holder,position-1);
            }
        }

        @Override
        public long getItemId(int position) {
        //注意，得到的item id 是在原有的adapter上-1的，因为多了个头部
            return adapter.getItemId(position-1);
        }
        @Override
        public int getItemCount() {
        	//多了头部尾部，所以在原有的长度要+2
            return adapter.getItemCount()+2;
        }
        class HeaderViewHolder extends RecyclerView.ViewHolder{
            public TextView tv;
            public HeaderViewHolder(View itemView) {
                super(itemView);
                tv = new TextView(MainActivity.this);
                tv.setLayoutParams(new RelativeLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT,ViewGroup.LayoutParams.MATCH_PARENT));
                tv.setGravity(Gravity.CENTER);
                ((ViewGroup)itemView).addView(tv);
                itemView.setBackgroundColor(0xff00ffff);
            }
        }
        class FootViewHolder extends RecyclerView.ViewHolder{
            public  TextView tv;
            public FootViewHolder(View itemView) {
                super(itemView);
                tv = new TextView(MainActivity.this);
                tv.setLayoutParams(new RelativeLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT,ViewGroup.LayoutParams.MATCH_PARENT));
                ((ViewGroup)itemView).addView(tv);
                itemView.setBackgroundColor(0xff00ff00);
            }
        }
    }
```

效果图如下：

 ![HeadView](https://img-blog.csdnimg.cn/20190117173735228.png)



RecycleView实现瀑布流
---------------------------
我们还是用这个布局吧，懒得改了。
只需要使用谷歌给我们提供的StaggeredGridLayoutManager即可实现：
```
 rv.setLayoutManager(new StaggeredGridLayoutManager(3,OrientationHelper.VERTICAL));
```
我们把item.xml 、 item2.xml 、 item3.xml里面的字体大小也设置不一样。不然默认高度一样则看不出瀑布流效果 跟网格布局一样了

设置后效果如下：（难看了点 凑合着吧，勉勉强强能看出内容参差不齐吧？）


![HeadView](https://img-blog.csdnimg.cn/20190117174823498.png)


