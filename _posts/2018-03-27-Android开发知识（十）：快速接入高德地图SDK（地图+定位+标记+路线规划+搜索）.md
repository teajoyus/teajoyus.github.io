---
tags: Android 
read: 578   
---

@[toc]
# 申请接入流程
-------------------
1、首先到<a href="http://lbs.amap.com/" target="_blank"> [ 高德地图API官网] </a>申请注册帐号
2、进入控制台，点击应用管理，我们创建一个新的应用：

3、为刚才创建的应用添加key：
![添加key](https://img-blog.csdn.net/20180327112700142?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2xjX21pYW8=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

随便输入一个key的名称，这里我们只讨论Android平台，所以服务平台选Android平台，选了Android后则需要填写SHA1，一个发布版和一个调试版。在这里如果我们的应用还没有正式签名的话，就先用自己随便生成的签名来填写进去，（有读者不了解如何生成签名或者不知道如何获取SHA1的话，请看官网教程<a href="http://lbs.amap.com/faq/top/hot-questions/249"> 常见问题 | 高德地图API </a>）。后面正式发布的时候再修改回来即可，修改SHA1也无需审核。
最后我们填写下应用的包名 勾选同意后，即可生成一个key。
是不是很简单。
申请接入流程非常的简单。



# 显示高德地图
-------
申请好了一个key后，我们需要把key填写到里面AndroidManifests.xml里面：

```
       <meta-data
            android:name="com.amap.api.v2.apikey"
            android:value="你申请的key"/>
```
需要特别注意的是，需要包含在**application节点**里面才有效。


然后我们添加如下权限（地图SDK（包含其搜索功能）需要的基础权限）：

```
<!--允许程序打开网络套接字-->
<uses-permission android:name="android.permission.INTERNET" />
<!--允许程序设置内置sd卡的写权限-->
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />   
<!--允许程序获取网络状态-->
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" /> 
<!--允许程序访问WiFi网络信息-->
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE" /> 
<!--允许程序读写手机状态和身份-->
<uses-permission android:name="android.permission.READ_PHONE_STATE" />     
<!--允许程序访问CellID或WiFi热点来获取粗略的位置-->
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" /> 
```


然后添加依赖包和so库平台：

```
android {
    defaultConfig {
        ndk {
            //设置支持的SO库架构（开发者可以根据需要，选择一个或多个平台的so）
            abiFilters "armeabi", "armeabi-v7a", "arm64-v8a", "x86","arm64-v8a","x86_64"
        }
    }
}

dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    //3D地图so及jar
    compile 'com.amap.api:3dmap:latest.integration'
    //定位功能
    compile 'com.amap.api:location:latest.integration'
    //搜索功能
    compile 'com.amap.api:search:latest.integration'
    //2D地图
    compile 'com.amap.api:map2d:latest.integration'
}
```
根据你的业务需求，这里我们不讨论3D地图的方式，基本上和2D地图是一样的。


OK，到这里基本的配置步骤就结束了，我们可以开始来写代码了。

最先提及的是MapView这个控件，它是地图容器。 用于在布局中中放置地图。我们简单来演示一下如何利用MapView显示地图。
首先在布局xml文件中添加地图控件，而官方文档会这么教你:

```
<com.amap.api.maps.MapView

    android:id="@+id/map"

    android:layout_width="match_parent"

    android:layout_height="match_parent"/>
```
而运行时却报错：`Caused by: android.view.InflateException: Binary XML file line #10: Error inflating class com.amap.api.maps.MapView`
```

```
这是大坑，正确的做法是，如果你是2D地图的话，则类的全名是：com.amap.api.maps2d.MapView。可以在代码里面输入MapView后查看自动导包的包名
然后在Activity的onCreate方法中添加代码即可：

```
  //获取地图控件引用
    mMapView = (MapView) findViewById(R.id.map);
    //在activity执行onCreate时执行mMapView.onCreate(savedInstanceState)，创建地图
    mMapView.onCreate(savedInstanceState);
```

这里特别需要说明的是需要注意MapView的生命周期，最好是要跟Activity的生命周期保持一致，避免内存泄漏。为此官方也给我们封装好了MapView对应的生命周期：

```
public class MainActivity extends Activity {
  MapView mMapView = null;
  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState); 
    setContentView(R.layout.activity_main);
    //获取地图控件引用
    mMapView = (MapView) findViewById(R.id.map);
    //在activity执行onCreate时执行mMapView.onCreate(savedInstanceState)，创建地图
    mMapView.onCreate(savedInstanceState);
  }
  @Override
  protected void onDestroy() {
    super.onDestroy();
    //在activity执行onDestroy时执行mMapView.onDestroy()，销毁地图
    mMapView.onDestroy();
  }
 @Override
 protected void onResume() {
    super.onResume();
    //在activity执行onResume时执行mMapView.onResume ()，重新绘制加载地图
    mMapView.onResume();
    }
 @Override
 protected void onPause() {
    super.onPause();
    //在activity执行onPause时执行mMapView.onPause ()，暂停地图的绘制
    mMapView.onPause();
    }
 @Override
 protected void onSaveInstanceState(Bundle outState) {
    super.onSaveInstanceState(outState);
    //在activity执行onSaveInstanceState时执行mMapView.onSaveInstanceState (outState)，保存地图当前的状态
    mMapView.onSaveInstanceState(outState);
  } 
}
```

到这一步我们就可以显示出来地图了

![显示地图](https://img-blog.csdn.net/20180327133748223?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2xjX21pYW8=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
如果需要地图显示其他什么元素，比如放大缩小，指南针 ，定位图标，logo等等，都可以通过获取UISettings这个类来设置，通过：

```
mMapView.getMap().getUiSettings()
```






# 显示定位
---------------------

首先我们需要先注册一个定位服务：

```
 <!--定位-->
        <service android:name="com.amap.api.location.APSService"></service>
```
蓝点图标的样式主要是这个类：MyLocationStyle ，我们通过对这个类的自定义设置，最后调用mMapView.getMap().setMyLocationStyle(myLocationStyle);即可

设置蓝点图标，也可以自定义 方法如下：

```
    /**
     * 显示定位蓝点图标
     */
    private void initLocationStyle() {
        BitmapDescriptor descriptor = BitmapDescriptorFactory.fromResource(R.drawable.main_location_icon);//自定义蓝点图标
        MyLocationStyle myLocationStyle = new MyLocationStyle();//初始化定位蓝点样式类myLocationStyle.myLocationType(MyLocationStyle.LOCATION_TYPE_LOCATION_ROTATE);//连续定位、且将视角移动到地图中心点，定位点依照设备方向旋转，并且会跟随设备移动。（1秒1次定位）如果不设置myLocationType，默认也会执行此种模式。
        myLocationStyle.myLocationIcon(descriptor);
//        myLocationStyle.interval(2000); //设置连续定位模式下的定位间隔，只在连续定位模式下生效，单次定位模式下不会生效。单位为毫秒。
//        myLocationStyle.myLocationType(MyLocationStyle.LOCATION_TYPE_FOLLOW);//连续定位、且将视角移动到地图中心点，定位蓝点跟随设备移动。（1秒1次定位）
        myLocationStyle.myLocationType(MyLocationStyle.LOCATION_TYPE_LOCATE);//定位一次，且将视角移动到地图中心点。
        myLocationStyle.strokeColor(getResources().getColor(R.color.colorPrimary));// 设置圆形的边框颜色
        myLocationStyle.radiusFillColor(getResources().getColor(R.color.colorPrimary_50));// 设置圆形的填充颜色
        mMapView.getMap().setMyLocationStyle(myLocationStyle);//设置定位蓝点的Style
//        mMapView.getMap().getUiSettings().setMyLocationButtonEnabled(true);//设置默认定位按钮是否显示，非必需设置。
        mMapView.getMap().setMyLocationEnabled(true);// 设置为true表示启动显示定位蓝点，false表示隐藏定位蓝点并不进行定位，默认是false。
    }

```

调用上面的方法即可实现定位。
如果需要实时监听位置的变化，我们可以加个监听：

```
/**
     * 定位的监听回调
     */
    private void initLocationListener() {
//初始化定位
        AMapLocationClient mLocationClient = new AMapLocationClient(getApplicationContext());
        //声明定位回调监听器
        AMapLocationListener mLocationListener = new AMapLocationListener() {
            @Override
            public void onLocationChanged(AMapLocation aMapLocation) {
                LogUtils.i(TAG, "aMapLocation:" + aMapLocation.getAddress());
                //获取纬度
                LogUtils.i(TAG, "aMapLocation:" + aMapLocation.getLatitude());
                LogUtils.i(TAG, "aMapLocation:" + aMapLocation.getLongitude());
                LogUtils.i(TAG, "aMapLocation==null:" + (aMapLocation == null));
                if (aMapLocation != null) {
                    if (aMapLocation.getErrorCode() == 0) {
//可在其中解析amapLocation获取相应内容。
                        LogUtils.i(TAG, "aMapLocation:" + aMapLocation.getPoiName());
                        current_latitude = aMapLocation.getLatitude();
                        current_longitude = aMapLocation.getLongitude();
                    } else {
                        //定位失败时，可通过ErrCode（错误码）信息来确定失败的原因，errInfo是错误信息，详见错误码表。
                        LogUtils.i(TAG, "location Error, ErrCode:"
                                + aMapLocation.getErrorCode() + ", errInfo:"
                                + aMapLocation.getErrorInfo());
                    }
                }
            }
        };
//设置定位回调监听
        mLocationClient.setLocationListener(mLocationListener);
//启动定位
        mLocationClient.startLocation();
    }

```

到这里就实现了定位以及定位的监听。如下效果图：
![定位](https://img-blog.csdn.net/20180327135149595?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2xjX21pYW8=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

默认的情况下，其实并不是这样子的，这是笔者在运行后放大了地图后的结果。
而通常我们是需要显示定位，并且放大到某个级别的，那么可以通过如下设置轻松设置：

```
  //设置希望展示的地图缩放级别。 缩放等级为1-19级
   mMapView.getMap().moveCamera(CameraUpdateFactory.zoomTo(18));
```

# Marker 显示地图标记
----------------
可能我们需要标记出地图上的某些东西，比如我们做了一个奶茶店的app，需要显示附近的奶茶店。那么我们可以通过后台返回来经度和纬度来标记出来，标记经纬度我们用到一个类：LatLng，该类构造方法时传入一个经度一个纬度，都是float型。
然后我们通过创建一个MarkerOptions类，这个MarkerOptions可以让我们构建标记的显示，我们也是可以自定义的view传进去的，后面再说。当我们构建好MarkerOptions后就可以通过mMapView.getMap().addMarker（）方法，把我们的MarkerOptions传递进去，即可形成一个标记。
代码如下：
```
 LatLng latLng = new LatLng(22.56686, 114.170988);
        MarkerOptions markerOption = new MarkerOptions();
        markerOption.title("我是Title").snippet("market desc market desc");
        markerOption.draggable(true);//设置Marker可拖动
        markerOption.position(latLng);
        markerOption.icon(BitmapDescriptorFactory.fromResource(R.drawable.main_marker_icon));
        //设置覆盖物比例
        markerOption.anchor(0.5f, 0.5f);
        Marker marker = mMapView.getMap().addMarker(markerOption);
```

这样子还不能在地图上显示出来，我们还需要调用：

```
 marker.showInfoWindow();
```
这样就可以显示出来了：
![marker](https://img-blog.csdn.net/20180327140548332?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2xjX21pYW8=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

而我们旺旺需要自定义浮现的布局样式，那么可以通过AMap.InfoWindowAdapter类来适配一个布局
代码如下：

```
 mMapView.getMap().setInfoWindowAdapter(new () {
            @Override
            public View getInfoWindow(Marker marker) {
                return LayoutInflater.from(getContext()).inflate(R.layout.marker,null);
            }

            @Override
            public View getInfoContents(Marker marker) {
                return LayoutInflater.from(getContext()).inflate(R.layout.marker,null);
            }
        });
```
然后设置监听：

```
 //点击标记弹出框
        mMapView.getMap().setOnMarkerClickListener(new AMap.OnMarkerClickListener() {
            @Override
            public boolean onMarkerClick(Marker marker) {
                if (!marker.isInfoWindowShown()) {
                    marker.showInfoWindow();
                } else {
                    marker.hideInfoWindow();
                }
              
```

比较简单。

# Route 路线规划
-----------------
当我们把店铺标记出来后，想要规划去这个店铺的路线怎么做呢
思路就是根据这个店铺的经纬度，我们利用RouteSearch这个类来实现
用LatLonPoint构造出一个经纬度的坐标点，我们构造出两个，分别是出发点的经纬度和目的地的经纬度，出发点的经纬度可以根据上面我们提到的定位监听回调里面获取当前的经纬度。然后利用这两个坐标构造出RouteSearch.FromAndTo对象

```
//当前经纬度
    LatLonPoint pointFrom = new LatLonPoint(current_latitude, current_longitude);
 //目的地的经纬度
LatLonPoint pointTo = new LatLonPoint(latitude, longitude);

RouteSearch.FromAndTo fromAndTo = new RouteSearch.FromAndTo(pointFrom, pointTo);
```
有了这个后，我们构造出一个查询，并且开始算出路线：

```
 //初始化query对象，fromAndTo是包含起终点信息，walkMode是步行路径规划的模式 mode，计算路径的模式。SDK提供两种模式：RouteSearch.WALK_DEFAULT 和 RouteSearch.WALK_MULTI_PATH。
        RouteSearch.WalkRouteQuery query = new RouteSearch.WalkRouteQuery(fromAndTo, RouteSearch.WALK_DEFAULT);
 RouteSearch routeSearch = new RouteSearch(this);
 routeSearch.calculateWalkRouteAsyn(query);//开始算路       
```
当然我们是需要路线规划出来的回调的，才能把它显示出来，不然就没用了：

```
  routeSearch.setRouteSearchListener(new RouteSearch.OnRouteSearchListener() {
            @Override
            public void onBusRouteSearched(BusRouteResult busRouteResult, int i) {
                
            }

            @Override
            public void onDriveRouteSearched(DriveRouteResult driveRouteResult, int i) {

            }

            @Override
            public void onWalkRouteSearched(WalkRouteResult walkRouteResult, int i) {

            }

            @Override
            public void onRideRouteSearched(RideRouteResult rideRouteResult, int i) {

            }
        });
```
通过实现RouteSearch.OnRouteSearchListener接口，该接口有几个需要实现的方法，不难对应就是不同交通工具的规划：公交、自驾、步行等。
这里我们来画出一个步行的路线，通过回调方法：onWalkRouteSearched(WalkRouteResult walkRouteResult, int i)

实现该接口方法如下：

```
  @Override
            public void onWalkRouteSearched(WalkRouteResult walkRouteResult, int i) {
                if (i == 1000) {//1000代表成功
                    //在地图上绘制路径：
                    MyWalkRouteOverlay walkRouteOverlay = new MyWalkRouteOverlay(getBaseContext(), mMapView.getMap(), walkRouteResult.getPaths().get(0), walkRouteResult.getStartPos(), walkRouteResult.getTargetPos());
                    walkRouteOverlay.setNodeIconVisibility(false);
//                    mMapView.getMap().clear();
                    walkRouteOverlay.removeFromMap();
                    walkRouteOverlay.addToMap();//将Overlay添加到地图上显示
                    walkRouteOverlay.zoomToSpan();//调整地图能看到起点和终点
                    lastWalkRouteOverlay = walkRouteOverlay;
                }
            }
```
这样就轻松的画出一条路线了，我们还可以通过继承WalkRouteOverlay这个类，来得到起点、终点、中间的图标以及路线的颜色、宽度等等。

# Search 搜索
-------------------------
地图搜索的个人感觉官方说的是比较详细的，这处不必要累赘什么。
建议读者到官方网站上了解： <a href="http://lbs.amap.com/api/android-sdk/guide/map-data/poi" target="_blank"> 搜索 | 高德地图API</a>
