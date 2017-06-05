https://github.com/Cornflower1991/RetrofitClient
###### 首先是抽象的基类
```
public abstract class BaseApi {
    public static final String API_SERVER = "服务器地址"
    private static final OkHttpClient mOkHttpClient = new OkHttpClient();
    private static Retrofit mRetrofit;

    protected static Retrofit getRetrofit() {
            if (Retrofit == null) {
                Context context = Application.getInstance().getApplicationContext();
                //设定30秒超时
                mOkHttpClient.setConnectTimeout(30, TimeUnit.SECONDS);
                //设置拦截器，以用于自定义Cookies的设置
                mOkHttpClient.networkInterceptors()
                            .add(new CookiesInterceptor(context));
                //设置缓存目录
                File cacheDirectory = new File(context.getCacheDir()
                                        .getAbsolutePath(), "HttpCache");
                Cache cache = new Cache(cacheDirectory, 20 * 1024 * 1024);
                mOkHttpClient.setCache(cache);
                //构建Retrofit
                mRetrofit = new Retrofit.Builder()
                        //配置服务器路径
                        .baseUrl(API_SERVER + "/")
                        //设置日期解析格式，这样可以直接解析Date类型
                        .setDateFormat("yyyy-MM-dd HH:mm:ss")
                        //配置转化库，默认是Gson
                        .addConverterFactory(ResponseConverterFactory.create())
                        //配置回调库，采用RxJava
                        .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
                        //设置OKHttpClient为网络客户端
                        .client(mOkHttpClient)
                        .build();
            }
            return mRetrofit;
        }
}
```
###### 然后是Cookies拦截器
```
public class CookiesInterceptor implements Interceptor{
    private Context context;

    public CookiesInterceptor(Context context) {
        this.context = context;
    }
    //重写拦截方法，处理自定义的Cookies信息
    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request();
        Request compressedRequest = request.newBuilder()
                .header("cookie", CookieUtil.getCookies(context))
                .build();
        Response response = chain.proceed(compressedRequest);
        CookieUtil.saveCookies(response.headers(), context);
        return response;
    }
}
```
CookieUtil则是一些自定义解析和生成方法以及SharedPreferences的存取，代码略

###### 然后是Api类
```
public class UserApi extends BaseApi{
    //定义接口
    private interface UserService {
        //GET注解不可用@FormUrlEncoded，要用@Query注解引入请求参数
        @GET("user/user_queryProfile")
        Observable<UserProfileResp> queryProfile(@Query("userId") int userId);

        //POST方法没有缓存，适用于更新数据
        @FormUrlEncoded
        @POST("user/user_updateUserName")
        Observable<BaseResp> updateUserName(@Field("userName") String userName);
    }
    protected static final UserService service = getRetrofit().create(UserService.class);

    //查询用户信息接口
    public static Observable<UserProfileResp> queryProfile(int userId){
        return service.queryProfile(userId);
    }

    //更新用户名接口
    public static Observable<BaseResp> updateUserName(String userName){
        return service.updateUserName(userName);
    }
}
```
再就是将Retrofit的响应消息经过Gson解析成期望的数据结构，称之为Model类
上文的BaseResp和UserProfileResp则是自定义的Model

假定服务器约定返回的Json格式为
```
{
    "result":"结果代号，0表示成功",
    "msg":"异常信息，仅在失败时返回数据",
    "userInfo":
    {
        "id":"用户id",
        "userName":"用户名名字"
    }
}
```
那么UserProfileResp可以写成
```
public class UserProfileResp {
    //@SerializedName是指定Json格式中的Key名
    //可以不写，则默认采用与变量名一样的Key名
    @SerializedName("userInfo")
    private UserProfileModel userInfo;

    public UserProfileModel getUserInfo() {
        return userInfo;
    }
}
```
UserProfileModel则是具体的数据结构
```
public class UserProfileModel {
        private int userId;
        private String userName;

        public String getUserName(){
            return userName;
        }
}
```
需要注意的是，如果没有使用@SerializedName指定Key名，当工程被混淆时，变量名会被混淆得与期望的Key名不符。因此需要将这类Model类统一放到一个工程目录，再在proguard-project文件中加入排除项
```
//不混淆Model类
-keep class com.xxx.model.xxx.** { *; }
```
最后是实际调用
```
public void getProfile(int userId){
    UserApi.queryProfile(userId)
        .subscribeOn(Schedulers.io())
        .subscribe(new Subscriber<UserProfileResp>(){
                        @Override
                        public void onCompleted() {
                        }
                        @Override
                        public void onError(Throwable e) {
                        }
                        @Override
                        public void onNext(UserProfileResp userProfileResp) {
                        }
        });
}
```

