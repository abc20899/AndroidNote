#### 原文链接：http://www.jianshu.com/p/3e13e5d34531
#### https://futurestud.io/tutorials/retrofit-2-how-to-upload-files-to-server

##### 2.0 使用配置
* 引入依赖

```
    compile 'com.squareup.retrofit2:retrofit:2.0.2'
    compile 'com.squareup.retrofit2:converter-gson:2.0.2'
    compile 'com.squareup.retrofit2:adapter-rxjava:2.0.2'
    compile 'com.squareup.okhttp3:logging-interceptor:3.2.0'
```

* OkHttp配置

```
   HttpLoggingInterceptor interceptor = new HttpLoggingInterceptor();
        interceptor.setLevel(HttpLoggingInterceptor.Level.BODY);
        client = new OkHttpClient.Builder()
                .addInterceptor(interceptor)
                .retryOnConnectionFailure(true)
                .connectTimeout(15, TimeUnit.SECONDS)
                .addNetworkInterceptor(authorizationInterceptor)
                .build();
```

其中 level 为 BASIC / HEADERS / BODY，BODY等同于1.9中的FULL
retryOnConnectionFailure:错误重联
addInterceptor:设置应用拦截器，可用于设置公共参数，头信息，日志拦截等
addNetworkInterceptor：网络拦截器，可以用于重试或重写，对应与1.9中的setRequestInterceptor。

* Retrofit配置

```
        Retrofit retrofit = new Retrofit.Builder()
        .baseUrl(BASE_URL)
        .client(client)
        .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
        .addConverterFactory(GsonConverterFactory.create(gson))
        .build();
        apiService = retrofit.create(ApiService.class);
```

其中baseUrl相当于1.9中的setEndPoint,
addCallAdapterFactory提供RxJava支持，如果没有提供响应的支持(RxJava,Call),则会跑出异常。
addConverterFactory提供Gson支持，可以添加多种序列化Factory，但是GsonConverterFactory必须放在最后,
否则会抛出异常。

##### 2.0使用介绍
* 注意：retrofit2.0后：BaseUrl要以/结尾；@GET 等请求不要以/开头；@Url: 可以定义完整url，不要以 / 开头。

* 基本用法：

```
//定以接口
public interface GitHubService {
  @GET("users/{user}/repos")
  Call<List<Repo>> listRepos(@Path("user") String user);
}

//获取实例
Retrofit retrofit = new Retrofit.Builder()
    //设置OKHttpClient,如果不设置会提供一个默认的
    .client(new OkHttpClient())
    //设置baseUrl
    .baseUrl("https://api.github.com/")
    //添加Gson转换器
    .addConverterFactory(GsonConverterFactory.create())
    .build();

GitHubService service = retrofit.create(GitHubService.class);

//同步请求
//https://api.github.com/users/octocat/repos
Call<List<Repo>> call = service.listRepos("octocat");
try {
     Response<List<Repo>> repos  = call.execute();
} catch (IOException e) {
     e.printStackTrace();
}

//call只能调用一次。否则会抛 IllegalStateException
Call<List<Repo>> clone = call.clone();

//异步请求
clone.enqueue(new Callback<List<Repo>>() {
        @Override
        public void onResponse(Call<List<Repo>> call, Response<List<Repo>> response) {
            // Get result bean from response.body()
            List<Repo> repos = response.body();
            // Get header item from response
            String links = response.headers().get("Link");
            /**
              * 不同于retrofit1 可以同时操作序列化数据javabean和header
              */
        }

        @Override
        public void onFailure(Call<List<Repo>> call, Throwable t) {

        }
    });

// 取消
call.cancel();

//rxjava support
public interface GitHubService {
  @GET("users/{user}/repos")
  Observable<List<Repo>> listRepos(@Path("user") String user);
}

// 获取实例
// Http request
Observable<List<Repo>> call = service.listRepos("octocat");

```

##### retrofit注解：
1. 方法注解，包含@GET、@POST、@PUT、@DELETE、@PATH、@HEAD、@OPTIONS、@HTTP。
2. 标记注解，包含@FormUrlEncoded、@Multipart、@Streaming。
3. 参数注解，包含@Query,@QueryMap、@Body、@Field，@FieldMap、@Part，@PartMap。
4. 其他注解，@Path、@Header,@Headers、@Url
5. 几个特殊的注解
   @HTTP：可以替代其他方法的任意一种
   /**
     * method 表示请的方法，不区分大小写
     * path表示路径
     * hasBody表示是否有请求体
     */
    @HTTP(method = "get", path = "users/{user}", hasBody = false)
    Call<ResponseBody> getFirstBlog(@Path("user") String user);

   @Url：使用全路径复写baseUrl，适用于非统一baseUrl的场景。

   @GET
   Call<ResponseBody> v3(@Url String url);
   @Streaming:用于下载大文件
   @Streaming
   @GET
   Call<ResponseBody> downloadFileWithDynamicUrlAsync(@Url String fileUrl);
   ResponseBody body = response.body();
   long fileSize = body.contentLength();
   InputStream inputStream = body.byteStream();
6. 常用注解
   @Path：URL占位符，用于替换和动态更新,相应的参数必须使用相同的字符串被@Path进行注释
   
```
@GET("group/{id}/users")
Call<List<User>> groupList(@Path("id") int groupId);
//--> http://baseurl/group/groupId/users

//等同于：
@GET
Call<List<User>> groupListUrl(
      @Url String url);
```
  @Query,@QueryMap:查询参数，用于GET查询,需要注意的是@QueryMap可以约定是否需要encode

  @GET("group/users")
  
```
Call<List<User>> groupList(@Query("id") int groupId);
//--> http://baseurl/group/users?id=groupId
Call<List<News>> getNews((@QueryMap(encoded=true) Map<String, String> options);
```

@Body:用于POST请求体，将实例对象根据转换方式转换为对应的json字符串参数，
这个转化方式是GsonConverterFactory定义的。

@POST("add")
 Call<List<User>> addUser(@Body User user);

@Field，@FieldMap:Post方式传递简单的键值对,
需要添加@FormUrlEncoded表示表单提交
Content-Type:application/x-www-form-urlencoded

@FormUrlEncoded
```
@POST("user/edit")
Call<User> updateUser(@Field("first_name") String first, @Field("last_name") String last);
```

@Part，@PartMap：用于POST文件上传
其中@Part MultipartBody.Part代表文件，@Part("key") RequestBody代表参数
需要添加@Multipart表示支持文件上传的表单，Content-Type: multipart/form-data

@Multipart

```
    @POST("upload")
    Call<ResponseBody> upload(@Part("description") RequestBody description,
                              @Part MultipartBody.Part file);
    // https://github.com/iPaulPro/aFileChooser/blob/master/aFileChooser/src/com/ipaulpro/afilechooser/utils/FileUtils.java
    // use the FileUtils to get the actual file by uri
    File file = FileUtils.getFile(this, fileUri);
    // create RequestBody instance from file
    RequestBody requestFile =
            RequestBody.create(MediaType.parse("multipart/form-data"), file);
    // MultipartBody.Part is used to send also the actual file name
    MultipartBody.Part body =
            MultipartBody.Part.createFormData("picture", file.getName(), requestFile);
    // add another part within the multipart request
    String descriptionString = "hello, this is description speaking";
    RequestBody description =
            RequestBody.create(
                    MediaType.parse("multipart/form-data"), descriptionString);
```

@Header：header处理，不能被互相覆盖，用于修饰参数，

```
//动态设置Header值
@GET("user")
Call<User> getUser(@Header("Authorization") String authorization)
//等同于 :

//静态设置Header值
@Headers("Authorization: authorization")//这里authorization就是上面方法里传进来变量的值
@GET("widget/list")
Call<User> getUser()
@Headers 用于修饰方法,用于设置多个Header值：

@Headers({
    "Accept: application/vnd.github.v3.full+json",
    "User-Agent: Retrofit-Sample-App"
})
@GET("users/{username}")
Call<User> getUser(@Path("username") String username);
```

##### 自定义Converter
retrofit默认情况下支持的converts有Gson,Jackson,Moshi...
要自定义Converter<F, T>，需要先看一下GsonConverterFactory的实现，
GsonConverterFactory实现了内部类Converter.Factory。

其中GsonConverterFactory中的主要两个方法，主要用于解析request和response的，
在Factory中还有一个方法stringConverter，用于String的转换。

```
//主要用于响应体的处理，Factory中默认实现为返回null，表示不处理
 @Override
  public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations,
      Retrofit retrofit) {
    TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
    return new GsonResponseBodyConverter<>(gson, adapter);
  }

/**
  *主要用于请求体的处理，Factory中默认实现为返回null，不能处理返回null
  *作用对象Part、PartMap、Body
  */
  @Override
  public Converter<?, RequestBody> requestBodyConverter(Type type,
      Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) {
    TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
    return new GsonRequestBodyConverter<>(gson, adapter);
  }
//Converter.Factory$stringConverter
/**
  *作用对象Field、FieldMap、Header、Path、Query、QueryMap
  *默认处理是toString
  */
  public Converter<?, String> stringConverter(Type type, Annotation[] annotations,
          Retrofit retrofit) {
        return null;
      }
GsonRequestBodyConverter实现了Converter<F, T>接口，
主要实现了转化的方法

T convert(F value) throws IOException;
```


##### 自定义Interceptor
Retrofit 2.0 底层依赖于okHttp，所以需要使用okHttp的Interceptors 来对所有请求进行拦截。
我们可以通过自定义Interceptor来实现很多操作,打印日志,缓存,重试等等。

要实现自己的拦截器需要有以下步骤
(1) 需要实现Interceptor接口，并复写intercept(Chain chain)方法,返回response
(2) Request 和 Response的Builder中有header,addHeader,headers方法,需要注意的是使用header有重复的将会被覆盖,
而addHeader则不会。
标准的 Interceptor写法

```
public class OAuthInterceptor implements Interceptor {

  private final String username;
  private final String password;

  public OAuthInterceptor(String username, String password) {
    this.username = username;
    this.password = password;
  }

  @Override
  public Response intercept(Chain chain) throws IOException {
    String credentials = username + ":" + password;
    String basic = "Basic " + Base64.encodeToString(credentials.getBytes(), Base64.NO_WRAP);
    Request originalRequest = chain.request();
    String cacheControl = originalRequest.cacheControl().toString();
    Request.Builder requestBuilder = originalRequest.newBuilder()
        //Basic Authentication,也可用于token验证,OAuth验证
        .header("Authorization", basic)
        .header("Accept", "application/json")
        .method(originalRequest.method(), originalRequest.body());
    Request request = requestBuilder.build();
    Response originalResponse = chain.proceed(request);
    Response.Builder responseBuilder =
        //Cache control设置缓存
        originalResponse.newBuilder().header("Cache-Control", cacheControl);
    return responseBuilder.build();
  }
}
```
##### 缓存策略
设置缓存就需要用到OkHttp的interceptors，缓存的设置需要靠请求和响应头。
如果想要弄清楚缓存机制，则需要了解一下HTTP语义，其中控制缓存的就是Cache-Control字段
http://www.jianshu.com/p/9c3b4ea108a7
一般情况下我们需要达到的缓存效果是这样的:

1. 没有网或者网络较差的时候要使用缓存(统一设置)
2. 有网络的时候，要保证不同的需求，实时性数据不用缓存,一般请求需要缓存(单个请求的header来实现)。
OkHttp3中有一个Cache类是用来定义缓存的，此类详细介绍了几种缓存策略,具体可看此类源码。

OkHttp3中有一个Cache类是用来定义缓存的，此类详细介绍了几种缓存策略,具体可看此类源码。

```
noCache ：不使用缓存，全部走网络
noStore ： 不使用缓存，也不存储缓存
onlyIfCached ： 只使用缓存
maxAge ：设置最大失效时间，失效则不使用
maxStale ：设置最大失效时间，失效则不使用
minFresh ：设置最小有效时间，失效则不使用
FORCE_NETWORK ： 强制走网络
FORCE_CACHE ：强制走缓存
```
* 配置目录
这个是缓存文件的存放位置,okhttp默认是没有缓存,且没有缓存目录的。

```
 private static final int HTTP_RESPONSE_DISK_CACHE_MAX_SIZE = 10 * 1024 * 1024;

  private Cache cache() {
         //设置缓存路径
         final File baseDir = AppUtil.getAvailableCacheDir(sContext);
         final File cacheDir = new File(baseDir, "HttpResponseCache");
         //设置缓存 10M
         return new Cache(cacheDir, HTTP_RESPONSE_DISK_CACHE_MAX_SIZE);
     }
```
其中获取cacahe目录,我们一般采取的策略就是应用卸载,即删除。一般就使用如下两个目录:

data/$packageName/cache:Context.getCacheDir()
/storage/sdcard0/Andorid/data/$packageName/cache:Context.getExternalCacheDir()
且当sd卡空间小于data可用空间时,使用data目录。
最后来一张图看懂Android内存结构,

```
/**
     * |   ($rootDir)
     * +- /data                    -> Environment.getDataDirectory()
     * |   |
     * |   |   ($appDataDir)
     * |   +- data/$packageName
     * |       |
     * |       |   ($filesDir)
     * |       +- files            -> Context.getFilesDir() / Context.getFileStreamPath("")
     * |       |      |
     * |       |      +- file1     -> Context.getFileStreamPath("file1")
     * |       |
     * |       |   ($cacheDir)
     * |       +- cache            -> Context.getCacheDir()
     * |       |
     * |       +- app_$name        ->(Context.getDir(String name, int mode)
     * |
     * |   ($rootDir)
     * +- /storage/sdcard0         -> Environment.getExternalStorageDirectory()/ Environment.getExternalStoragePublicDirectory("")
     * |                 |
     * |                 +- dir1   -> Environment.getExternalStoragePublicDirectory("dir1")
     * |                 |
     * |                 |   ($appDataDir)
     * |                 +- Andorid/data/$packageName
     * |                                         |
     * |                                         | ($filesDir)
     * |                                         +- files                  -> Context.getExternalFilesDir("")
     * |                                         |    |
     * |                                         |    +- file1             -> Context.getExternalFilesDir("file1")
     * |                                         |    +- Music             -> Context.getExternalFilesDir(Environment.Music);
     * |                                         |    +- Picture           -> Context.getExternalFilesDir(Environment.Picture);
     * |                                         |    +- ...               -> Context.getExternalFilesDir(String type)
     * |                                         |
     * |                                         |  ($cacheDir)
     * |                                         +- cache                  -> Context.getExternalCacheDir()
     * |                                         |
     * |                                         +- ???
     * <p/>
     * <p/>
     * 1.  其中$appDataDir中的数据，在app卸载之后，会被系统删除。
     * <p/>
     * 2.  $appDataDir下的$cacheDir：
     * Context.getCacheDir()：机身内存不足时，文件会被删除
     * Context.getExternalCacheDir()：空间不足时，文件不会实时被删除，可能返回空对象,Context.getExternalFilesDir("")亦同
     * <p/>
     * 3. 内部存储中的$appDataDir是安全的，只有本应用可访问
     * 外部存储中的$appDataDir其他应用也可访问，但是$filesDir中的媒体文件，不会被当做媒体扫描出来，加到媒体库中。
     * <p/>
     * 4. 在内部存储中：通过  Context.getDir(String name, int mode) 可获取和  $filesDir  /  $cacheDir 同级的目录
     * 命名规则：app_ + name，通过Mode控制目录是私有还是共享
     * <p/>
     * <code>
     * Context.getDir("dir1", MODE_PRIVATE):
     * Context.getDir: /data/data/$packageName/app_dir1
     * </code>
     */
```

* 缓存第一种类型
配置单个请求的@Headers，设置此请求的缓存策略,不影响其他请求的缓存策略,不设置则没有缓存。

// 设置 单个请求的 缓存时间
@Headers("Cache-Control: max-age=640000")
@GET("widget/list")
Call<List<Widget>> widgetList();

* 缓存第二种类型
有网和没网都先读缓存，统一缓存策略，降低服务器压力。

```
private Interceptor cacheInterceptor() {
      Interceptor cacheInterceptor = new Interceptor() {
            @Override
            public Response intercept(Chain chain) throws IOException {
                Request request = chain.request();
                Response response = chain.proceed(request);

                String cacheControl = request.cacheControl().toString();
                if (TextUtils.isEmpty(cacheControl)) {
                    cacheControl = "public, max-age=60";
                }
                return response.newBuilder()
                        .header("Cache-Control", cacheControl)
                        .removeHeader("Pragma")
                        .build();
            }
        };
      }
```
此中方式的缓存Interceptor实现：ForceCachedInterceptor.java

* 缓存第三种类型
结合前两种，离线读取本地缓存，在线获取最新数据(读取单个请求的请求头，亦可统一设置)。
```
private Interceptor cacheInterceptor() {
        return new Interceptor() {
            @Override
            public Response intercept(Chain chain) throws IOException {
                Request request = chain.request();

                if (!AppUtil.isNetworkReachable(sContext)) {
                    request = request.newBuilder()
                            //强制使用缓存
                            .cacheControl(CacheControl.FORCE_CACHE)
                            .build();
                }

                Response response = chain.proceed(request);

                if (AppUtil.isNetworkReachable(sContext)) {
                    //有网的时候读接口上的@Headers里的配置，你可以在这里进行统一的设置
                    String cacheControl = request.cacheControl().toString();
                    Logger.i("has network ,cacheControl=" + cacheControl);
                    return response.newBuilder()
                            .header("Cache-Control", cacheControl)
                            .removeHeader("Pragma")
                            .build();
                } else {
                    int maxStale = 60 * 60 * 24 * 28; // tolerate 4-weeks stale
                    Logger.i("network error ,maxStale="+maxStale);
                    return response.newBuilder()
                            .header("Cache-Control", "public, only-if-cached, max-stale="+maxStale)
                            .removeHeader("Pragma")
                            .build();
                }

            }
        };
    }
```
此中方式的缓存Interceptor实现：OfflineCacheControlInterceptor.java

* 错误处理
在请求网络的时候,我们不止会得到HttpException,还有我们和服务器约定的errorCode和errorMessage,为了统一处理,
我们可以预处理以下上面两个字段,定义BaseModel,在ConverterFactory中进行处理,
可参照:
Retrofit+RxJava实战日志(3)-网络异常处理
http://blog.csdn.net/efan006/article/details/50544204
retrofit-2-simple-error-handling

* 网络状态监听
一般在没有网络的时候使用缓存数据,有网络的时候及时重试获取最新数据,其中获取是否有网络，我们采用广播的形式：

```
 public class NetWorkReceiver extends BroadcastReceiver {

     @Override
     public void onReceive(Context context, Intent intent) {
         HttpNetUtil.INSTANCE.setConnected(context);
     }
 }
```

HttpNetUtil实时获取网络连接状态,关键代码

```
   /**
     * 获取是否连接
     */
    public boolean isConnected() {
        return isConnected;
    }
   /**
     * 判断网络连接是否存在
     *
     * @param context
     */
    public void setConnected(Context context) {
        ConnectivityManager manager = (ConnectivityManager) context.getSystemService(Context.CONNECTIVITY_SERVICE);
        if (manager == null) {
            setConnected(false);


            if (networkreceivers != null) {
                for (int i = 0, z = networkreceivers.size(); i < z; i++) {
                    Networkreceiver listener = networkreceivers.get(i);
                    if (listener != null) {
                        listener.onConnected(false);
                    }
                }
            }

        }

        NetworkInfo info = manager.getActiveNetworkInfo();

        boolean connected = info != null && info.isConnected();
        setConnected(connected);

        if (networkreceivers != null) {
            for (int i = 0, z = networkreceivers.size(); i < z; i++) {
                Networkreceiver listener = networkreceivers.get(i);
                if (listener != null) {
                    listener.onConnected(connected);
                }
            }
        }

    }
```
在需要监听网络的界面或者base(需要判断当前activity是否在栈顶)实现Networkreceiver。


##### Retrofit封装
全局单利的OkHttpClient：

```
okHttp() {
        HttpLoggingInterceptor interceptor = new HttpLoggingInterceptor();
        interceptor.setLevel(HttpLoggingInterceptor.Level.BODY);

        okHttpClient = new OkHttpClient.Builder()
                //打印日志
                .addInterceptor(interceptor)

                //设置Cache目录
                .cache(CacheUtil.getCache(UIUtil.getContext()))

                //设置缓存
                .addInterceptor(cacheInterceptor)
                .addNetworkInterceptor(cacheInterceptor)

                //失败重连
                .retryOnConnectionFailure(true)

                //time out
                .readTimeout(TIMEOUT_READ, TimeUnit.SECONDS)
                .connectTimeout(TIMEOUT_CONNECTION, TimeUnit.SECONDS)

                .build();
    }
```
全局单利的Retrofit.Builder,这里返回builder是为了方便我们设置baseUrl的,我们可以动态创建多个api接口,
当然也可以用@Url注解
```
Retrofit2Client() {
        retrofitBuilder = new Retrofit.Builder()
                //设置OKHttpClient
                .client(okHttp.INSTANCE.getOkHttpClient())

                //Rx
                .addCallAdapterFactory(RxJavaCallAdapterFactory.create())

                //String转换器
                .addConverterFactory(StringConverterFactory.create())

                //gson转化器
                .addConverterFactory(GsonConverterFactory.create())
        ;
    }
```

demo：
https://github.com/BoBoMEe/AndroidDev/























