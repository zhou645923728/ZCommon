# ZCommon

[![](https://jitpack.io/v/yuyuyu123/ZCommon.svg)](https://jitpack.io/#yuyuyu123/ZCommon)
# Dependency
Step1:在根目录.build文件下添加 
```gradle
repositories {
  maven { url "https://jitpack.io" }
}
```
Step2：在具体项目.build目录下添加
```gradle
implemention 'com.github.yuyuyu123:ZCommon:1.2.0'
```
# Data Requests   
1.约定：数据请求一律采用RxJava+Retrofit  
2.配置   
   1.请求单个base url
```java
ConfigurationHelper.setBaseUrl(URLHelper.BASE_URL);
ConfigurationHelper.setOkhttpClient(okHttpClient);
``` 
   2.请求多个base url
```java
Map<String , String> baseUrlMap = new HashMap<>();
String url1 = "";
String url2 = "";
baseUrlMap.put(key1, url1);
baseUrlMap.put(key2, url2);
ConfigurationHelper.setBaseUrlMap(baseUrlMap);

Map<String,OkHttpClient> clientMap = new HashMap<>();
clientMap.put(key1, okHttpClient1);
clientMap.put(key2, okHttpClient2);
ConfigurationHelper.setOkhttpClientMap(clientMap);
``` 
  说明:    
  1.虽然多个base url可以共用一个OkHttpClient对象，但是为了适应不同的配置，即使是相同的OkHttpClient也要放进Map中；   
  2.设置base url的map和设置client的key必须一一对应   
  3.使用   
  1.请求单个base url
```java
BaiduService service = ManagerFactory.getFactory().getManager(BaiduService.class);
```
  2.请求多个base url
```java
OneService service1 = ManagerFactory.getFactory().getManager(OneService.class, key1);
TwoService service2 = ManagerFactory.getFactory().getManager(TwoService.class, key2);
```
  说明:     
  1.单个base url的使用可以不传key，对应只设置base url;   
  2.多个base url的使用必须传key区分，对应设置base url map;   
  3.上面只是罗列了要点，具体使用可以参考内部app。   
# Request Logging
如果需要打印网络请求，建议用OkHttp3提供的拦截器HttpLoggingInterceptor,只需要在请求前配置即可，例：
```java
HttpLoggingInterceptor logging = new HttpLoggingInterceptor(new HttpLoggingInterceptor.Logger() {
            @Override
            public void log(String message) {
                Log.d("okhttp", message);
            }
        });
logging.setLevel(HttpLoggingInterceptor.Level.BASIC);
new OkHttpClient
  .Builder()
  .addInterceptor(logging);
```

# Https
OkHttp支持Http和Https两种协议的请求，只需要在请求前用ZCommon内的HttpsUtils配置一下即可，例：
```java
HttpsUtils.SSLParams sslParams = HttpsUtils.getSslSocketFactory(null, null, null);
new OkHttpClient
  .Builder()
  .sslSocketFactory(sslParams.sSLSocketFactory, sslParams.trustManager);
```

# Crash Handler
为了防止NullPointerException等Bug导致程序崩溃，可以在自定义Application中使用CrashHandler类，例：
```java
public class App extends BaseApplication {

    @Override
    public void onCreate() {
        super.onCreate();
        initCrashHandler();
        
    }
    
    private void initCrashHandler() {
        CrashHandler.init((thread, throwable) -> {
            new Handler(Looper.getMainLooper()).post(() -> {
                try {
                    //处理错误，可以上报，也可以不处理
                } catch (Throwable e) {

                }
            });
        });
    }
} 
```
# Status Bar
ZCommon中提供了状态栏适配的Util叫StatusBarUtil，建议在基类里面作统一适配，例：
```java
public abstract class DkBaseActivity<V extends RxBaseView, P extends RxBasePresenter<V>>
    extends RxBasePermissionActivity<V, P> {


  @Override
  protected void init(Bundle savedInstanceState) {
    super.init(savedInstanceState);
    StatusBarUtil.showStatusBarWithLightMode(this, com.ccclubs.common.R.color.material_light_white);
  }
}
```
# Activity
建议在业务baseLib中可以根据自己的需求写一到多个基类Activity，继承RxBaseActivity或者RxBasePermissionActivity（封装了权限请求）,例新建一个处理Umeng统计的基类:
```java
public abstract class DkBaseActivity<V extends RxBaseView, P extends RxBasePresenter<V>>
    extends RxBaseActivity<V, P> {

  @Override
  protected void onResume() {
    super.onResume();
    MobclickAgent.onPageStart(TAG);
    MobclickAgent.onResume(this);
  }

  @Override
  protected void onPause() {
    super.onPause();
    MobclickAgent.onPageEnd(TAG);
    MobclickAgent.onPause(this);
  }
}
```
例写一个处理滑动返回的基类：
```java
public abstract class DkBaseSwipeBackActivity<V extends RxBaseView, P extends RxBasePresenter<V>> extends DkBaseActivity<V,P> {

    @Override
    protected void init(Bundle savedInstanceState) {
        super.init(savedInstanceState);

        if(isSwipeBackEnabled()) {
            SwipeBackManager.onCreate(this);
            SwipeBackManager.getCurrentPage(this)//get current instance
                    .setSwipeBackEnable(true)//on-off
                    .setSwipeEdge(mScreenWidth / 5)
                    .setSwipeEdgePercent(0.2f)
                    .setSwipeSensitivity(0.6f)
                    .setClosePercent(0.5f)
                    .setSwipeRelateEnable(false)
                    .setSwipeRelateOffset(500);
        }
    }

    @Override
    protected void onPostCreate(Bundle savedInstanceState) {
        super.onPostCreate(savedInstanceState);
        if(isSwipeBackEnabled()) {
            SwipeBackManager.onPostCreate(this);
        }
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        if(isSwipeBackEnabled()) {
            SwipeBackManager.onDestroy(this);
        }
    }
}
```

1.setContentView
当Activity需要setContentView时，可以复写下面两个方法：
```java
@Override
protected abstract int getLayoutId() {
   return R.layout.activity_home_layout;
};

@Override
protected View getLayoutView() {
   return new FrameLayout(this);
}
```
如果复写了设置了getLayoutView方法，getLayoutId方法将失效。

2.初始化数据
```java
@Override
protected void init(Bundle savedInstanceState) {
  super.init(savedInstanceState);
  //初始化，包括findViewById
}
```
3.Activity内创建Presenter  
Activity或Fragment内需要创建Presenter需要复写createPresenter方法
```java
@Override
protected SplashPresenter createPresenter() {
  return new SplashPresenter();
}
```
4.加载列表
参考内部App
# Activity's Swipe Back
Activity需要滑动返回效果时，可以创建一个基类用SwipeBackManager来设置相应的滑动返回参数。  
注：  
1.有地图的界面不允许设置滑动返回效果；  
2.支持滑动切换多个Tab的界面不允许设置滑动返回。

# Fragment  
1.基类和Activity类似；  
2.初始化提供两个方法：  
   1）初始化View
```java
@Override
protected void init(Bundle savedInstanceState, View view) {
  super.init(savedInstanceState, view);
  AppCompatImageView imgClose =  view.findViewById(R.id.id_img_close);
  imgClose.setOnClickListener(v -> titleLeftClicked());
}
```
    
   2）初始化数据
```java
@Override
protected void initData() {
   super.initData();
   final Bundle bundle = getArguments();
   mOrderId = bundle.getLong("orderId", -1);
   mCarLat = bundle.getDouble("carLat", 0d);
   mCarLng = bundle.getDouble("carLng", 0d);
   mIsLongOrder = bundle.getBoolean("isLongOrder", false);
}
```
3.设置布局文件和创建Presenter和Fragment类似

4.处理back键  
需要处理back键的Fragment首先需要实现IFragmentBackPressed接口，并复写其中的方法，例：
```java
public class PhotoGalleryFragment extends Fragment implements IFragmentBackPressed {
    @Override
    public void onFragmentBackPressed() {
        //处理返回键
    }
}
```
然后在Fragment的宿主Activity中复写onBackPressed方法，如下：
```java
@Override
public void onBackPressed() {
  if (!FragmentBackManager.handleBackPress(this)) {
      super.onBackPressed();
  }
}
```
# Permissions for Android M
1.Activity申请权限   
1）申请单个权限
Activity若需要申请权限，需要继承RxBasePermissionActivity，例：
```java
public class SplashActivity extends RxBasePermissionActivityimplements RxBasePermissionActivity.OnSinglePermissionRequestCallBack {
  @Override
  protected void onStart() {
    super.onStart();
    if (dealWithPermission(android.Manifest.permission.ACCESS_FINE_LOCATION, “被拒绝后的提示”)){
        //如果权限已经通过dealWithPermission方法会返回true
    }
  }
    
  @Override
  public void onPermissionAllowed(String permission) {//权限申请通过回调
    if (permission.equals(android.Manifest.permission.ACCESS_FINE_LOCATION)) {
       //定位权限被打开
    }
  }

    @Override
    public void onPermissionDenied(String permission) {//权限被拒绝回调
        if( !shouldShowPermissionRationale(permission)) {
            String permissionTip = "您拒绝了定位权限，将无法正常使用App，请前往\"权限管理\"界面手动允许定位权限。";
        } else {
            finish();
        }
    }
}
```
2）申请多个权限     
申请多个权限和单个权限类似，参考内部App   
2.Fragment申请权限   
Fragment申请权限和Activity类似，只要继承RxBasePermissionFragment并实现OnSinglePermissionRequestCallBack或OnMultiPermissionRequestCallBack接口并实现其中的方法即可。
# Event Bus
```java
Activity中默认注册EventBus，Fragment中想要用EventBus,需要复写该方法：
  @Override
    protected boolean isBindEventBusHere() {
        return true;
    }
```
发送EventBus消息：
```java
EventBusHelper.post(new AppUpdateEvent());
```
# Net
1.获取网络类型、判断网络是否可用等集合在NetworkUtils工具类中。  
2.网络状态变更时获取网络连接状态  
  step1.让Activity实现NetChangeObserver接口，复写其中的方法  
  step2.在onResume和onPause方法中分别调用相应的注册方法和反注册方法：  
```java
private void registerNetReceiver() {
  NetStateReceiver.registerObserver(this);
  NetStateReceiver.registerNetworkStateReceiver(this);
}

private void unRegisterNetReceiver() {
  NetStateReceiver.removeRegisterObserver(this);
  NetStateReceiver.unRegisterNetworkStateReceiver(this);
}
```

# Image
1.加载普通图片   
无论是加载网络图片、File中的图片还是Asset资源中的图片，均可调用
```java 
displayImg(ImageView imageView, String path)
```
当然也可以调用
```java
displayImg(ImageView imageView, File file)
```
加载文件中的图片，调用
```java
displayAssetImg(ImageView imageView, String assetName)
```
加载assets中的图片，示例：
```java
ZImageLoader.get(imageView.getContext()).displayImg(imageView, path);
```
2.加载图片时需要设置相应的error和holder图
```java
ZImageLoader loader =  ZImageLoader.get(imageView.getContext());
loader.setResPlaceHolderId(R.mipmap.bg_img_load_failed);//设置加载中的占位图
loader.setResErrorId(R.mipmap.bg_img_load_failed);//设置加载错误时的占位图
loader.displayImg(imageView, path);
```   
3.加载圆形或圆角图片  
  1）加载圆形图片：
```java
ZImageLoader loader =  ZImageLoader.get(imageView.getContext());
loader.setType(ZImageLoader.IMG_TYPE.CIRCLE);
loader.displayImg(imageView, path);
```   
  2）加载圆角图片
```java
ZImageLoader loader =  ZImageLoader.get(imageView.getContext());
loader.setType(ZImageLoader.IMG_TYPE.ROUND_CORNER);
loader.setRoundingRadius(20);
loader.displayImg(imageView, path);
```
     
4.加载图片支持设置回调
```java
ZImageLoader loader =  ZImageLoader.get(imageView.getContext());
loader.setType(ZImageLoader.IMG_TYPE.ROUND_CORNER);
loader.setRoundingRadius(20);
loader.displayImg(imageView, path, new OnImgLoadListener() {
   @Override
   public void onLoadComplete() {
     //图片加载完成
   }

   @Override
   public void onLoadFailure(Exception e) {
     //图片加载失败
   }
 });
 ```
# Fast Click  
快速点击处理  
1.用ViewClickUtils处理快速点击，间隔默认500ms，支持自定义时间，例：
```java
@Override
public void onClick(View v) {
  if (ViewClickUtils.isFastClick(v)) {//默认500ms，建议用这个
      return;
  }
}

@Override
public void onClick(View v) {
   if(ViewClickUtils.isSpecificTimeClick(v, 600)) {//指定600ms
      return;
   }
}
```  
2.也可以用FastClickAgent来处理快速点击，实际上他就是一个OnClickListener的扩展类，例：
```java
private FastClickAgent mFastClickListener = new FastClickAgent(view -> {
  final long id = view.getId();
  if(id == R.id.id_item_2) {
     //具体的点击事件
  }
});
view.setOnClickListener(mFastClickListener);
```  

# SQLite
ZCommon中提供了SQLite的帮助类来实现数据的持久化存储，支持直接以Model的方式存储和获取。   
1.使用前提：用于存储和读取的Model类必须继续ZCommon中的BaseModel。   
2.使用步骤：  
  1）调用ZDbConfigHelper类来对数据库名称和数据库版本号，以及相应的表和对应的列进行初始化，示例：   
 ```java
 //初始化数据库名
ZDbConfigHelper.getInstance(this).setDatabaseName(TablesHelper.DATABASE_NAME);
//初始化数据库版本号
ZDbConfigHelper.getInstance(this).setDatabaseVersion(TablesHelper.DATABASE_VERSION);
//用户表
List<String> userColumns = new ArrayList<>();
//表的列名+列名对应的类型
userColumns.add(TablesHelper.USER_CSM_ID + ZDbConfigHelper.TEXT_TYPE);
userColumns.add(TablesHelper.USER_TOKEN + ZDbConfigHelper.TEXT_TYPE);
userColumns.add(TablesHelper.USER_MOBILE + ZDbConfigHelper.TEXT_TYPE);
userColumns.add(TablesHelper.USER_HEADER + ZDbConfigHelper.TEXT_TYPE);
userColumns.add(TablesHelper.USER_NICKNAME + ZDbConfigHelper.TEXT_TYPE);
userColumns.add(TablesHelper.USER_BALANCE + ZDbConfigHelper.TEXT_TYPE);
//添加表，表名+所有的列
ZDbConfigHelper.getInstance(this).addTable(TablesHelper.USER_INFO_TABLE, userColumns);
//地址搜索表
List<String> searchColumns = new ArrayList<>();
searchColumns.add(TablesHelper.USER_SEARCH_DATE + ZDbConfigHelper.LONG_TYPE);
searchColumns.add(TablesHelper.USER_SEARCH_LAT + ZDbConfigHelper.LONG_TYPE);
searchColumns.add(TablesHelper.USER_SEARCH_LON + ZDbConfigHelper.DOUBLE_TYPE);
searchColumns.add(TablesHelper.USER_SEARCH_ADDRESS + ZDbConfigHelper.TEXT_TYPE);
searchColumns.add(TablesHelper.USER_SEARCH_NAME + ZDbConfigHelper.TEXT_TYPE);
ZDbConfigHelper.getInstance(this).addTable(TablesHelper.USER_SEARCH_TABLE, searchColumns);
```  
  2）编写具体的Dao类来对数据库进行操作，师例如下：   
 ```java
public class SearchDao implements ISearchDao{

   @Override
    public long insertData(UserSearchModel userSearchModel) {
        if(userSearchModel == null || TextUtils.isEmpty(userSearchModel.search_addr)
                || userSearchModel.search_lat == 0 || userSearchModel.search_lon == 0) {
            Log.d(TAG, "the data to insert is invalid.");
            return  -1;
        }
        if(isDataExisted(userSearchModel)) {
            return ZDbDataHelper.getInstance().update(TablesHelper.USER_SEARCH_TABLE, TablesHelper.USER_SEARCH_ADDRESS + "=?", new String[]{userSearchModel.search_addr}, userSearchModel);
        } else {
           return ZDbDataHelper.getInstance().insert(TablesHelper.USER_SEARCH_TABLE, userSearchModel);
        }
    }
  }
```  
具体用法可以参考内部App

# Download
1.下载apk文件，示例：
```java

private void downloadApk(String url) {
  File savingFile = new File(getExternalFilesDir(null) + File.separator + "rainbow.apk");
  RetrofitDownloadConfig retrofitDownloadConfig = new RetrofitDownloadConfig.Builder(this)
     .setSavingFile(savingFile)//设置保存的文件
     .setNotification(null)//通知栏显示下载进度，没有设置为null
     .setPackageName(getPackageName())//下载apk文件需要安装时建议手动设置包名，当然也可以不设置
     .setAutoInstallApk(true)//是否自动安装apk
     .setRetrofitDownloadAdapter(new RetrofitDownloadAdapter() {//下载进度监听
          @Override
          public void onDownloading(int code, String message, long fileTotalSize, long fileSizeDownloaded) {
             super.onDownloading(code, message, fileTotalSize, fileSizeDownloaded);
             if (fileTotalSize <= 0) return;
               final int percent = (int) ((fileSizeDownloaded * 1.f / fileTotalSize) * 100);
               showDownloadProgressDialog(percent);
             }

           @Override
           public void onDownloadSuccess(int code, String message) {
              //下载成功
              super.onDownloadSuccess(code, message);
              dismissDownloadProgressDialog();
           }

            @Override
            public void onDownloadFailure(int code, String message) {
                //下载失败
                super.onDownloadFailure(code, message);
                dismissDownloadProgressDialog();
                T.showShort(HomeActivity.this, getStringResource(R.string.download_failure) + ":" + message);
            }

            @Override
            public void onDownloadError(Throwable t) {
                //下载出错
                super.onDownloadError(t);
                dismissDownloadProgressDialog();
             }
            }).build();
        RetrofitDownloadManager retrofitDownloadManager = new RetrofitDownloadManager(retrofitDownloadConfig);
        retrofitDownloadManager.downloadFile(url);
    }
    
```
注意，在Android7.0以上的手机，需要提供相应的provider文件，并在AndroidManifest文件中注册，具体可以参考我们的项目。  
2.下载普通文件  
   下载普通文件和apk类似  
# Upload
支持上传单文件和多文件，例：   
```java
private void uploadFiles(File[] files) {//上传单文件只需要写一个file即可
  Map<String, Object> map = new HashMap<>();
  map.put("app", URLHelper.IMG_APP_VALUE);
  map.put("key", URLHelper.IMG_KEY_VALUE);
  Map<String, Object> paramMap = new HashMap<>();
  paramMap.put("json", new Gson().toJson(map).toString());//上传的参数
  RetrofitUploadConfig retrofitUploadConfig = new RetrofitUploadConfig.Builder(context)
    .setUploadUrl(URLHelper.IMG_SERVER).setParamsMap(paramMap).setFileKey("file")
    .setDescriptionString("image for getting evidence")
    .setRetrofitUploadAdapter(new RetrofitUploadAdapter<PhotoModel>() {
         @Override
         public void onUploadSuccess(int code, PhotoModel bean, boolean allFinished) {
                //上传成功
         }

         @Override
         public void onUploadFailure(int code, String message, boolean allFinished) {
               //上传失败
         }

         @Override
         public void onUploadError(Throwable t, boolean allFinished) {
               //上传发生错误
         }
                }).build();
        List<File> list = new ArrayList<>();
        for (File file : files) {
            if (null == file) {
                continue;
            }
            list.add(file);
        }
        RetrofitUploadManager retrofitUploadManager = new RetrofitUploadManager
                (retrofitUploadConfig);
        retrofitUploadManager.uploadFiles(list);
    }
```
# Camera And Choose Local Picture  
1.拍照  
```java
 startActivityForResult(CameraAndPictureActivity.newIntent(this, CameraAndPictureActivity.TYPE_TAKING_PHOTO, "hello"), 1);
 ```
2.拍照并裁剪   
```java
 startActivityForResult(CameraAndPictureActivity.newIntent(this, CameraAndPictureActivity.TYPE_TAKING_PHOTO_AND_CROP, "hello"), 1);
 ```
3.选择图片  
```
 startActivityForResult(CameraAndPictureActivity.newIntent(this, CameraAndPictureActivity.TYPE_CHOOSING_IMG, "hello"), 1);
```
4.选择图片并裁剪  
```
 startActivityForResult(CameraAndPictureActivity.newIntent(this, CameraAndPictureActivity.TYPE_CHOOSING_IMG_AND_CROP, "hello"), 1);
```
5.接收数据  
```
  @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        if(data == null) {
            return;
        }
        if(requestCode == 1) {
            final String path = data.getStringExtra("path");
            Bitmap bitmap = BitmapFactory.decodeFile(path);
            img.setImageBitmap(bitmap);
        }
    }
```
注意:  
1.需要在AndroidManifest中注册CameraAndPictureActivity，并设置为透明主题；  
2.CameraAndPictureActivity内已经对6.0权限作了适配，但推荐在外部申请好相应的权限之后在调用；   
3.拍摄单个图片可以不传图片名，但拍摄多张图片时必须传入图片名以作区分；    
4.对于Android7.0需要提供相应的provider文件，具体参考Simple或内部App。 
# Cache  
暂时只提供ACache这个类作轻量级的缓存，后续会引入接口缓存的方案。  
# Animation
暂时忽略
# Utils
ZCommon下面提供了许多有用的Utils，比如跟Log相关的L类，跟toast相关的T类等等，待进一步完善。

# Thanks
