# 前言

GPS系列——Android端，[github项目地址](https://github.com/fly7632785/Gps/tree/gps_mine) tag: gps_mine

Android移动端，主要是使用高德地图定位，后台上传定位信息，然后就是想办法尽量保活。

包括两个小功能：1、上传定位信息 2、模拟定位信息

都是练手实践，去深入了解其原理。通篇代码较多，慎入。

大家尽可以去查看源码，各取所需。



### GPS定位系统系列

>[GPS定位系统(一)——介绍](https://www.jianshu.com/p/1aa130ffbeaf)
>
>[GPS定位系统(二)——Android端](https://www.jianshu.com/p/e463b1ea6b7d)
>
>[GPS定位系统(三)——Java后端](https://www.jianshu.com/p/2e51318c56e0)
>
>[GPS定位系统(四)——Vue前端](https://www.jianshu.com/p/9df016cd65fa)
>
>[GPS定位系统(五)——Docker](https://www.jianshu.com/p/75ba5bc90988)
>
>- [Docker nginx 二级域名无端口访问多个web项目](https://www.jianshu.com/p/3378d2eacb3d)
>- [Docker nginx https二级域名无端口访问多个web项目](https://www.jianshu.com/p/3e25b24d7247)
>- [持续部署——Travis+Docker+阿里云容器镜像](https://www.jianshu.com/p/ce648e120727)



[TOC]



### 收获

学习完这篇文章你将收获：

>- 高德地图、定位使用
>- 高德坐标系转换（官方只有其他坐标系转高德，没有高德转gps）
>- 模拟定位（打卡）
>- 卸载重装也不变的uuid|imei
>- 保活策略和原理



# 一、地图

地图使用的是**高德地图**，注册申请appkey的话，请移步官网网站。

地图界面功能很简单，跟着官方文档来就行

```java
 private void initMap() {
        MyLocationStyle myLocationStyle;
        myLocationStyle = new MyLocationStyle();//初始化定位蓝点样式类myLocationStyle.myLocationType(MyLocationStyle.LOCATION_TYPE_LOCATION_ROTATE);//连续定位、且将视角移动到地图中心点，定位点依照设备方向旋转，并且会跟随设备移动。（1秒1次定位）如果不设置myLocationType，默认也会执行此种模式。
        myLocationStyle.myLocationType(MyLocationStyle.LOCATION_TYPE_FOLLOW);
        myLocationStyle.interval(10000); //设置连续定位模式下的定位间隔，只在连续定位模式下生效，单次定位模式下不会生效。单位为毫秒。
        AMap map = mMapView.getMap();
        map.setMyLocationStyle(myLocationStyle);//设置定位蓝点的Style
        map.setMyLocationEnabled(true);// 设置为true表示启动显示定位蓝点，false表示隐藏定位蓝点并不进行定位，默认是false。
        map.getUiSettings().setMyLocationButtonEnabled(true); //显示默认的定位按钮
        map.setMyLocationEnabled(true);// 可触发定位并显示当前位置
        map.moveCamera(CameraUpdateFactory.zoomTo(16));
        map.setOnMapClickListener(latLng -> {
            Log.d(TAG, "mapCLick:" + latLng.latitude + "\t" + latLng.longitude);
            mMockLat = latLng.latitude;
            mMockLng = latLng.longitude;
            if (mMarker != null) {
                mMarker.remove();
            }
            mMarker = map.addMarker(new MarkerOptions().position(latLng).title("模拟位置").snippet("default"));
        });
        map.setOnMyLocationChangeListener(location -> Log.d(TAG, "onMyLocationChange:" + location.getLatitude() + "\t" + location.getLongitude()));
    }
```

**注意**：地图选点的话，使用`map.setOnMapClickListener`来设置监听。

### gps和高德地图 经纬度 互转

**注意**：就是gps和高德的坐标体系的互转，模拟定位模拟的gps定位，需要选好模拟点之后，转成gps的定位进行模拟。这里写了一个工具类。

```java
public class ConvertUtil {
        private final static double a = 6378245.0;
        private final static double pi = 3.14159265358979324;
        private final static double ee = 0.00669342162296594323;

        // WGS-84 to GCJ-02  gps转高德
        public static LatLng toGCJ02Point(double latitude, double longitude) {
            LatLng dev = calDev(latitude, longitude);
            double retLat = latitude + dev.latitude;
            double retLon = longitude + dev.longitude;
            return new LatLng(retLat, retLon);
        }

        // GCJ-02 to WGS-84  高德转gps
        public static LatLng toWGS84Point(double latitude, double longitude) {
            LatLng dev = calDev(latitude, longitude);
            double retLat = latitude - dev.latitude;
            double retLon = longitude - dev.longitude;
            dev = calDev(retLat, retLon);
            retLat = latitude - dev.latitude;
            retLon = longitude - dev.longitude;
            return new LatLng(retLat, retLon);
        }

        private static LatLng calDev(double wgLat, double wgLon) {
            if (isOutOfChina(wgLat, wgLon)) {
                return new LatLng(0, 0);
            }
            double dLat = calLat(wgLon - 105.0, wgLat - 35.0);
            double dLon = calLon(wgLon - 105.0, wgLat - 35.0);
            double radLat = wgLat / 180.0 * pi;
            double magic = Math.sin(radLat);
            magic = 1 - ee * magic * magic;
            double sqrtMagic = Math.sqrt(magic);
            dLat = (dLat * 180.0) / ((a * (1 - ee)) / (magic * sqrtMagic) * pi);
            dLon = (dLon * 180.0) / (a / sqrtMagic * Math.cos(radLat) * pi);
            return new LatLng(dLat, dLon);
        }

        private static boolean isOutOfChina(double lat, double lon) {
            if (lon < 72.004 || lon > 137.8347)
                return true;
            if (lat < 0.8293 || lat > 55.8271)
                return true;
            return false;
        }

        private static double calLat(double x, double y) {
            double ret = -100.0 + 2.0 * x + 3.0 * y + 0.2 * y * y + 0.1 * x * y + 0.2
                    * Math.sqrt(Math.abs(x));
            ret += (20.0 * Math.sin(6.0 * x * pi) + 20.0 * Math.sin(2.0 * x * pi)) * 2.0 / 3.0;
            ret += (20.0 * Math.sin(y * pi) + 40.0 * Math.sin(y / 3.0 * pi)) * 2.0 / 3.0;
            ret += (160.0 * Math.sin(y / 12.0 * pi) + 320 * Math.sin(y * pi / 30.0)) * 2.0 / 3.0;
            return ret;
        }

        private static double calLon(double x, double y) {
            double ret = 300.0 + x + 2.0 * y + 0.1 * x * x + 0.1 * x * y + 0.1 * Math.sqrt(Math.abs(x));
            ret += (20.0 * Math.sin(6.0 * x * pi) + 20.0 * Math.sin(2.0 * x * pi)) * 2.0 / 3.0;
            ret += (20.0 * Math.sin(x * pi) + 40.0 * Math.sin(x / 3.0 * pi)) * 2.0 / 3.0;
            ret += (150.0 * Math.sin(x / 12.0 * pi) + 300.0 * Math.sin(x / 30.0 * pi)) * 2.0 / 3.0;
            return ret;
        }
 
}
```



# 二、后台保活定位

### 保活：

保活这里使用一个框架很不错，[HelloDaemon](https://github.com/xingda920813/HelloDaemon)

> 保活思路：
>
> 1. 将Service设置为前台服务而不显示通知
> 2. 在 Service 的 onStartCommand 方法里返回 START_STICKY
> 3. 覆盖 Service 的 onDestroy/onTaskRemoved 方法, 保存数据到磁盘, 然后重新拉起服务
> 4. 监听 8 种系统广播
> 5. 开启守护服务 : 定时检查服务是否在运行，如果不在运行就拉起来
>
> 6. 守护 Service 组件的启用状态, 使其不被 MAT 等工具禁用

并且，还有适配各种手机厂商rom的intent跳转【电量优化】【自启设置】【白名单】等设置界面

```
 IntentWrapper.whiteListMatters(this, "为了更好的实时定位，最好把应用加入您手机的白名单");
```

保活service继承AbsWorkService，实现其抽象方法即可

```java
/**
 * 是否 任务完成, 不再需要服务运行?
 * @return 应当停止服务, true; 应当启动服务, false; 无法判断, null.
 */
Boolean shouldStopService();

/**
 * 任务是否正在运行?
 * @return 任务正在运行, true; 任务当前不在运行, false; 无法判断, null.
 */
Boolean isWorkRunning();

void startWork();

void stopWork();

//Service.onBind(Intent intent)
@Nullable IBinder onBind(Intent intent, Void unused);

//服务被杀时调用, 可以在这里面保存数据.
void onServiceKilled();
```

#### 关于保活、安全、隐私

其实随着Android的日趋成熟，生态更健康、安全、更注重隐私、以用户为本，很多“黑科技”已经都不行了，现在的保活已经不像以前各种花里胡哨，感兴趣了解一些旧版本的保活策略的话可以这些链接学习一下：

[Android 进程常驻（2）----细数利用android系统机制的保活手段](http://blog.csdn.net/marswin89/article/details/50890708)

[D-clock / AndroidDaemonService](https://github.com/D-clock/AndroidDaemonService)

不像大厂们的做法，各种互拉，手机厂商的白名单；现在的“民间”保活思路基本都是，尽量引导用户**添加白名单、电量优化无限制、锁住应用**等。

保活没有谁能够说能100%，一直在后台存活的，就算能保活，也是在某些广播或者用户行为下，触发拉活，各大手机厂商的rom表现不一，有些还是会被杀，没有办法。不过我小米6手机，实测，白名单、电量优化设置后，能每分钟都上传gps信息，不断，但是还是挺耗电的，说实话。有些其他的手机就不行了，例如华为，华为生态和安全的确是很棒啊。



#### UploadGpsService实现：

```java
public class UploadGpsService extends AbsWorkService {
    private static final String TAG = UploadGpsService.class.getSimpleName();
    //是否 任务完成, 不再需要服务运行?
    public static boolean sShouldStopService;

    int shouldCount;
    int actualCount;


    @Override
    public void onCreate() {
        super.onCreate();
        initGps();
    }

    //声明AMapLocationClient类对象
    public AMapLocationClient mLocationClient = null;
    //声明定位回调监听器
    public AMapLocationListener mLocationListener = amapLocation -> {
        if (amapLocation != null) {
            if (amapLocation.getErrorCode() == 0) {
//可在其中解析amapLocation获取相应内容。
                Log.d("mapLocation", amapLocation.toString());
                //获取定位时间
                SimpleDateFormat df = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
                Date date = new Date();
                df.format(date);
                Log.d(TAG, String.format("经度：%s\t纬度：%s\t地址：%s\n%s\n应上传次数%d\n实上传次数%d", amapLocation.getLongitude(), amapLocation.getLatitude(), amapLocation.getAddress(), df.format(date), shouldCount, actualCount));
                upload(amapLocation.getLongitude(), amapLocation.getLatitude());
            } else {
                //定位失败时，可通过ErrCode（错误码）信息来确定失败的原因，errInfo是错误信息，详见错误码表。
                Log.e("AmapError", "location Error, ErrCode:"
                        + amapLocation.getErrorCode() + ", errInfo:"
                        + amapLocation.getErrorInfo());
            }
        }
    };

    private void upload(double longitude, double latitude) {
        String userId = PrefManager.getInstance(this).userId();
        String token = PrefManager.getInstance(this).getToken();
//        if (TextUtils.isEmpty(userId)) {
//            return;
//        }
        shouldCount++;
        RequestModel requestModel = new RequestModel();
        requestModel.setTime(System.currentTimeMillis() / 1000);
        requestModel.setLat(latitude);
        requestModel.setLng(longitude);
        RetrofitManager.getInstance()
                .mainService()
                .gps(token, requestModel)
                .compose(ReactivexCompat.singleThreadSchedule())
                .subscribe(result -> {
                    if (result.getCode() == 200) {
                        long interval = result.getData();
                        mLocationOption.setInterval(interval);
                        mLocationClient.setLocationOption(mLocationOption);
                        Log.d(TAG, "service upload success:" + new Gson().toJson(result));
                        actualCount++;
                    }
                }, e -> {
                    Log.e(TAG, "service upload err:" + e.getMessage());
                });
    }


    //声明AMapLocationClientOption对象
    public AMapLocationClientOption mLocationOption = null;


    private void initGps() {
//初始化定位
        mLocationClient = new AMapLocationClient(getApplicationContext());
//设置定位回调监听
        mLocationClient.setLocationListener(mLocationListener);

        //初始化AMapLocationClientOption对象
        mLocationOption = new AMapLocationClientOption();
        /**
         * 设置定位场景，目前支持三种场景（签到、出行、运动，默认无场景）
         */
        mLocationOption.setLocationPurpose(AMapLocationClientOption.AMapLocationPurpose.Sport);

        //设置定位模式为AMapLocationMode.Hight_Accuracy，高精度模式。
        mLocationOption.setLocationMode(AMapLocationClientOption.AMapLocationMode.Hight_Accuracy);

        //设置定位间隔,单位毫秒,默认为2000ms，最低1000ms。
        mLocationOption.setInterval(5000);

        mLocationClient.setLocationOption(mLocationOption);

        //启动定位
        mLocationClient.startLocation();
    }


    public static void stopService() {
        //我们现在不再需要服务运行了, 将标志位置为 true
        sShouldStopService = true;
        //取消 Job / Alarm / Subscription
        cancelJobAlarmSub();
    }

    @Override
    public Boolean shouldStopService(Intent intent, int flags, int startId) {
        return sShouldStopService;
    }

    @Override
    public void startWork(Intent intent, int flags, int startId) {
        Log.i(TAG, "startWork");
        String userId = PrefManager.getInstance(this).userId();
        if (!TextUtils.isEmpty(userId)) {
            if (mLocationClient != null && !mLocationClient.isStarted()) {
                Log.i(TAG, "startLocation");
                mLocationClient.startLocation();
            } else if (mLocationClient == null) {
                initGps();
            }
        } else {
            if (mLocationClient != null) {
                mLocationClient.stopLocation();
            }
        }

    }

    @Override
    public void stopWork(Intent intent, int flags, int startId) {
        Log.i(TAG, "stopWork");
        stopService();
        if (mLocationClient != null) {
            mLocationClient.stopLocation();
            mLocationClient.onDestroy();
        }
    }

    @Override
    public Boolean isWorkRunning(Intent intent, int flags, int startId) {
        //若还没有取消订阅, 就说明任务仍在运行.
        return null;
    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent, Void alwaysNull) {
        return null;
    }

    @Override
    public void onServiceKilled(Intent rootIntent) {
        Log.i(TAG, "onServiceKilled");
    }
}
```

#### 上传api

上传api设计就很简单了，上传经纬度、时间就行

```java
public interface MainService {

    @POST("/login")
    @FormUrlEncoded
    Single<LoginResult> login(@Field("username") String username, @Field("password") String password);


    @POST("/gps")
    Single<UploadResult> gps(@Header ("token")String token, @Body RequestModel model);
}

public class RequestModel
{
    private Double lat;
    private Double lng;
    private Long time;
}
```

### 关于imei

Android的生态越来越健康，也越来越安全，很多用户隐私信息都获取不到了。例如，imei、手机号、sim卡编号等等。

但是，如果要保证唯一性，ime是最优选择，其次就是uuid或者其他自行组编的code。但是，涉及到一个问题，如果本地化没有处理好，卸载重装就没有了。所以，这里有个uuid的工具类，原理是，uuid等存放到sdcard而非沙盒目录下面。

但是29(Android10)，由于文件分区的关系，也不能直接访问了。需要使用api来进行访问。`android:requestLegacyExternalStorage="true"`也可以用来兼容使用。

工具类：

```java
public final class DeviceUtil {
    private static final String TAG = DeviceUtil.class.getSimpleName();

    private static final String TEMP_DIR = "system_config";
    private static final String TEMP_FILE_NAME = "system_file";
    private static final String TEMP_FILE_NAME_MIME_TYPE = "application/octet-stream";
    private static final String SP_NAME = "device_info";
    private static final String SP_KEY_DEVICE_ID = "device_id";

    public static String getDeviceId(Context context) {
        SharedPreferences sharedPreferences = context.getSharedPreferences(SP_NAME, Context.MODE_PRIVATE);
        String deviceId = sharedPreferences.getString(SP_KEY_DEVICE_ID, null);
        if (!TextUtils.isEmpty(deviceId)) {
            return deviceId;
        }
        deviceId = getIMEI(context);
        if (TextUtils.isEmpty(deviceId)) {
            deviceId = createUUID(context);
        }
        sharedPreferences.edit()
                .putString(SP_KEY_DEVICE_ID, deviceId)
                .apply();
        return deviceId;
    }

    private static String createUUID(Context context) {
        String uuid = UUID.randomUUID().toString().replace("-", "");

        if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.Q) {
            Log.d(TAG,"Q");
            Uri externalContentUri = MediaStore.Downloads.EXTERNAL_CONTENT_URI;
            ContentResolver contentResolver = context.getContentResolver();
            String[] projection = new String[]{
                    MediaStore.Downloads._ID
            };
            String selection = MediaStore.Downloads.TITLE + "=?";
            String[] args = new String[]{
                    TEMP_FILE_NAME
            };
            Cursor query = contentResolver.query(externalContentUri, projection, selection, args, null);
            if (query != null && query.moveToFirst()) {
                Log.d(TAG,"moveToFirst");
                Uri uri = ContentUris.withAppendedId(externalContentUri, query.getLong(0));
                query.close();

                InputStream inputStream = null;
                BufferedReader bufferedReader = null;
                try {
                    inputStream = contentResolver.openInputStream(uri);
                    if (inputStream != null) {
                        bufferedReader = new BufferedReader(new InputStreamReader(inputStream));
                        uuid = bufferedReader.readLine();
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                } finally {
                    if (bufferedReader != null) {
                        try {
                            bufferedReader.close();
                        } catch (IOException e) {
                            e.printStackTrace();
                        }
                    }
                    if (inputStream != null) {
                        try {
                            inputStream.close();
                        } catch (IOException e) {
                            e.printStackTrace();
                        }
                    }
                }
            } else {
                Log.d(TAG,"ContentValues");
                ContentValues contentValues = new ContentValues();
                contentValues.put(MediaStore.Downloads.TITLE, TEMP_FILE_NAME);
                contentValues.put(MediaStore.Downloads.MIME_TYPE, TEMP_FILE_NAME_MIME_TYPE);
                contentValues.put(MediaStore.Downloads.DISPLAY_NAME, TEMP_FILE_NAME);
                contentValues.put(MediaStore.Downloads.RELATIVE_PATH, Environment.DIRECTORY_DOWNLOADS + File.separator + TEMP_DIR);

                Uri insert = contentResolver.insert(externalContentUri, contentValues);
                if (insert != null) {
                    OutputStream outputStream = null;
                    try {
                        outputStream = contentResolver.openOutputStream(insert);
                        if (outputStream == null) {
                            return uuid;
                        }
                        outputStream.write(uuid.getBytes());
                    } catch (IOException e) {
                        e.printStackTrace();
                    } finally {
                        if (outputStream != null) {
                            try {
                                outputStream.close();
                            } catch (IOException e) {
                                e.printStackTrace();
                            }
                        }
                    }
                }
            }
        } else {
            Log.d(TAG,"DIRECTORY_DOWNLOADS");
            File externalDownloadsDir = Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_DOWNLOADS);
            File applicationFileDir = new File(externalDownloadsDir, TEMP_DIR);
            if (!applicationFileDir.exists()) {
                if (!applicationFileDir.mkdirs()) {
                    Log.e(TAG, "文件夹创建失败: " + applicationFileDir.getPath());
                }
            }
            File file = new File(applicationFileDir, TEMP_FILE_NAME);
            if (!file.exists()) {
                Log.d(TAG,"mk DIRECTORY_DOWNLOADS");
                FileWriter fileWriter = null;
                try {
                    if (file.createNewFile()) {
                        fileWriter = new FileWriter(file, false);
                        fileWriter.write(uuid);
                    } else {
                        Log.e(TAG, "文件创建失败：" + file.getPath());
                    }
                } catch (IOException e) {
                    Log.e(TAG, "文件创建失败：" + file.getPath());
                    e.printStackTrace();
                } finally {
                    if (fileWriter != null) {
                        try {
                            fileWriter.close();
                        } catch (IOException e) {
                            e.printStackTrace();
                        }
                    }
                }
            } else {
                Log.d(TAG,"read DIRECTORY_DOWNLOADS");
                FileReader fileReader = null;
                BufferedReader bufferedReader = null;
                try {
                    fileReader = new FileReader(file);
                    bufferedReader = new BufferedReader(fileReader);
                    uuid = bufferedReader.readLine();

                    bufferedReader.close();
                    fileReader.close();
                } catch (IOException e) {
                    e.printStackTrace();
                } finally {
                    if (bufferedReader != null) {
                        try {
                            bufferedReader.close();
                        } catch (IOException e) {
                            e.printStackTrace();
                        }
                    }

                    if (fileReader != null) {
                        try {
                            fileReader.close();
                        } catch (IOException e) {
                            e.printStackTrace();
                        }
                    }
                }
            }
        }

        return uuid;
    }

    private static String getIMEI(Context context) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
            return null;
        }
        try {
            TelephonyManager telephonyManager = (TelephonyManager) context.getSystemService(Context.TELEPHONY_SERVICE);
            if (telephonyManager == null) {
                return null;
            }
            @SuppressLint({"MissingPermission", "HardwareIds"}) String imei = telephonyManager.getDeviceId();
            return imei;
        } catch (Exception e) {
            return null;
        }
    }
}
```





# 三、模拟定位

现在模拟gps定位，实现模拟定位、位置打开等，多数也是采用【开发者-模拟定位应用】这种方式。这种方式在某些没有做过特殊反模拟定位处理的APP上还是可以用的，比如百度地图，模拟之后，依旧能够看到自己在模拟的地方。但是，例如，钉钉打卡、微信打卡、微信定位等这些大厂APP都是做了反模拟处理，依旧是不能用的。

这里实现，也是为了练练手，了解其中原理。

参看module：mocklocationlib源码

> 实现步骤：
>
> 1、引导用户开启开发者模式，选择模拟定位应用，添加自身应用
>
> 2、利用系统api，LocationManager来添加test模拟位置信息

```java
 public boolean getUseMockPosition(Context context) {
        // Android 6.0以下，通过Setting.Secure.ALLOW_MOCK_LOCATION判断
        // Android 6.0及以上，需要【选择模拟位置信息应用】，未找到方法，因此通过addTestProvider是否可用判断
        boolean canMockPosition = (Settings.Secure.getInt(context.getContentResolver(), Settings.Secure.ALLOW_MOCK_LOCATION, 0) != 0)
                || Build.VERSION.SDK_INT > 22;
        if (canMockPosition && hasAddTestProvider == false) {
            try {
                for (String providerStr : mockProviders) {
                    //获取所有的provider 包括网络、gps、卫星等
                    LocationProvider provider = locationManager.getProvider(providerStr);
                    if (provider != null) {
                        locationManager.addTestProvider(
                                provider.getName()
                                , provider.requiresNetwork()
                                , provider.requiresSatellite()
                                , provider.requiresCell()
                                , provider.hasMonetaryCost()
                                , provider.supportsAltitude()
                                , provider.supportsSpeed()
                                , provider.supportsBearing()
                                , provider.getPowerRequirement()
                                , provider.getAccuracy());
                    } else {
                        if (providerStr.equals(LocationManager.GPS_PROVIDER)) {//模拟gps模块定位信息
                            locationManager.addTestProvider(
                                    providerStr
                                    , true, true, false, false, true, true, true
                                    , Criteria.POWER_HIGH, Criteria.ACCURACY_FINE);
                        } else if (providerStr.equals(LocationManager.NETWORK_PROVIDER)) { //模拟网络定位信息
                            locationManager.addTestProvider(
                                    providerStr
                                    , true, false, true, false, false, false, false
                                    , Criteria.POWER_LOW, Criteria.ACCURACY_FINE);
                        } else {
                            locationManager.addTestProvider(
                                    providerStr
                                    , false, false, false, false, true, true, true
                                    , Criteria.POWER_LOW, Criteria.ACCURACY_FINE);
                        }
                    }
                    locationManager.setTestProviderEnabled(providerStr, true);
                    locationManager.setTestProviderStatus(providerStr, LocationProvider.AVAILABLE, null, System.currentTimeMillis());
                }
                hasAddTestProvider = true;  // 模拟位置可用
                canMockPosition = true;
            } catch (SecurityException e) {
                canMockPosition = false;
            }
        }
        if (canMockPosition == false) {
            stopMockLocation();
        }
        return canMockPosition;
    }
```

```java
 /**
     * 模拟位置线程
     */
    private class RunnableMockLocation implements Runnable {

        @Override
        public void run() {
            while (true) {
                try {
                    Thread.sleep(3000);

                    if (hasAddTestProvider == false) {
                        continue;
                    }

                    if (bRun == false) {
                        stopMockLocation();
                        continue;
                    }
                    try {
                        // 模拟位置（addTestProvider成功的前提下）
                        for (String providerStr : mockProviders) {
                            Log.d(TAG, "providerStr：" + providerStr);
                            locationManager.setTestProviderLocation(providerStr, generateLocation(latitude, longitude));
                        }
                    } catch (Exception e) {
                        e.printStackTrace();
                        // 防止用户在软件运行过程中关闭模拟位置或选择其他应用
                        stopMockLocation();
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
    }

    public Location generateLocation(double lat, double lng) {
        Location loc = new Location("gps");
        Log.d(TAG, "mock latitude:" + lat + "\tlongitude:" + lng);

        loc.setAccuracy(2.0F);
        loc.setAltitude(55.0D);
        loc.setBearing(1.0F);
        Bundle bundle = new Bundle();
        bundle.putInt("satellites", 7);
        loc.setExtras(bundle);

        loc.setLatitude(lat);
        loc.setLongitude(lng);
        loc.setTime(System.currentTimeMillis());
        if (Build.VERSION.SDK_INT >= 17) {
            loc.setElapsedRealtimeNanos(SystemClock.elapsedRealtimeNanos());
        }
        return loc;
    }
```

代码解析：

- 先获取测试provider，包括有网络、gps卫星等模块
- 启动线程，定时往provider里面添加模拟的定位信息，进行模拟

封装好lib之后，使用起来就很简单了，开启线程，设置要模拟的位置即可

```java
    @Override
    public void startWork(Intent intent, int flags, int startId) {
        Log.i(TAG, "startWork");
        if (mMockLocationManager == null) {
            mMockLocationManager = new MockLocationManager();
            mMockLocationManager.initService(getApplicationContext());
            mMockLocationManager.startThread();
        }

        if (mMockLocationManager.getUseMockPosition(getApplicationContext())) {
            startMockLocation();
            double lat = intent.getDoubleExtra(INTENT_KEY_LAT, 0);
            double lng = intent.getDoubleExtra(INTENT_KEY_LNG, 0);
            setMangerLocationData(lat, lng);
        }
    }
```

使用：在地图上选点，然后模拟就OK



# 总结

Android端只是个小小的开始，没有后台接口的支持，数据上传了也没有用。所以，后面还需要搭建一下java服务器，写几个接口来满足我们的需求。

请移步[GPS定位系统(三)——Java后端](https://www.jianshu.com/p/2e51318c56e0)



# 关于作者

作者是一个热爱学习、开源、分享，传播正能量，喜欢打篮球、头发还很多的程序员-。-

热烈欢迎大家关注、点赞、评论交流！

简书：https://www.jianshu.com/u/d234d1569eed

github：https://github.com/fly7632785

CSDN：https://blog.csdn.net/fly7632785

掘金：https://juejin.im/user/5efd8d205188252e58582dc7/posts