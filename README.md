# Retrofit 2.0 사용법 정리

# 0. Gradle
```java
// java:build.gradle
// retrofit 2.0 추가
compile 'com.squareup.retrofit2:retrofit:2.0.+'
// 2.0 버전부터는 json 파서가 내장되어있지 않음.
compile 'com.squareup.retrofit2:converter-gson:2.0.1'
```

# 1. RestService - interface
```java
/**
 * Retrofit 에서는 interface 하나에 아래와 같은 식으로 GET, POST 등등의 메소드를 정의한다.
 */
public interface RestService {
    // 서버의 RootUrl
    public static final String BASE_URL = "https://api.github.com";

    // GET 방식. 아래의 메소드는 다음의 URL 을 호출한다.
    // https://api.github.com/users/{id}
    // users/{id} 중에 id 에는 파라메터 변수인 String id 의 값이 들어가는 것 같다.
    @GET("users/{id}")
    Call<User> getUser(@Path("id") String id);

    @GET("users/{id}/followers")
    Call<List<User>> getFollowers(@Path("id") String id);

    @GET("users/{id}/following")
    Call<List<User>> getFollowing(@Path("id") String id);
    
    /*
    아래의 코드는 POST 방식으로 파라메터에 데이터를 담아 보낼 때 사용
    */
    @FormUrlEncoded
    @POST("/test")
    Call<Void> createUser(@Field("email") String email,
                          @Field("password") String password,
                          @Field("name") String name,
                          @Field("term_no") int term_no,
                          @Field("agree_term_version") String term_version);
                          
    @FormUrlEncoded
    @PUT("/test")
    Call<Void> updateUser(@Field("current_password") String currentPassword,
                          @Field("password") String password, @Field("name") String name);
                          
    @Headers("Content-Type:application/json")
    @POST("/tracking")
    Call<TrackingBean> saveTracking(@Body SendingBean json);
}
```

# 인터셉터
```java
public class MyRequestInterceptor implements Interceptor {
    private String token;

    public MyRequestInterceptor(String token) {
        this.token = token;
    }

    public void setToken(String token) {
        this.token = token;
    }

    @Override
    public Response intercept(Chain chain) throws IOException {
        Request original = chain.request();

        // 매번 리퀘스트 할 때마다 헤더를 변경할 수 있음.
        // addHeader 메소드는 기본값을 건들지 않고 헤더에 뭔가 추가할때
        // header 메소드는 덮어씌울 때 사용
        Request.Builder requestBuilder = original.newBuilder()
                .addHeader("X-Auth-Token", token)
                .method(original.method(), original.body());

        Request request = requestBuilder.build();
        return chain.proceed(request);
    }
}

```

# 인터셉터 사용
```java
private void OkHttpClient getClient() {
// Http 로그를 출력하는 인터셉터
    HttpLoggingInterceptor interceptor = new HttpLoggingInterceptor();
        interceptor.setLevel(DEBUG_LEVEL);
        if (TextUtils.isEmpty(token)) {
            return new OkHttpClient.Builder().addInterceptor(interceptor).build();
        }
        
        // 여러개의 인터셉터를 OkHttpClient 에 추가할 수 있음.
        return new OkHttpClient.Builder().addInterceptor(interceptor).addInterceptor(getInterceptor(token)).build();
}
 Retrofit retrofit = new Retrofit.Builder()
                .baseUrl(RestService.BASE_URL)
                .addConverterFactory(GsonConverterFactory.create())
                .build();
```

# 2. MainActivity - 실제 사용
```java
// MainActivity.java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // 기존의 RestAdapter 가 Retrofit 이라는 이름으로 바뀐 것 같다.
        // baseUrl 메소드를 이용하여 API 서버의 Root 주소를 세팅하고
        // addConverterFactory 메소드를 이용하여 Gson 컨버터를 추가하는 내용이다.
        Retrofit retrofit = new Retrofit.Builder()
                .baseUrl(RestService.BASE_URL)
                .addConverterFactory(GsonConverterFactory.create())
                .build();

        // API 호출 메소드가 담겨있는 RestService 의 instance 를 생성하고
        RestService service = retrofit.create(RestService.class);

        // 입력한 id에 대한 값으로 GitHub 의 User 정보를 얻어오는 getUser 메소드를 호출한다.
        // 그 결과는 Call<User> 타입으로 리턴된다.
        // 하지만 call 객체에 데이터가 담겨있는 것이 아니다.
        // call 객체를 이용하여 execute 또는 enqueue 해야 response 값을 얻어올 수 있다.
        Call<User> call =  service.getUser("prChoe");

        /**
         * 이 방식은 동기 방식이다.
         * UI 스레드에서 실행하니 NetworkOnMainThreadException 이 발생했다.
         * 안드로이드에서는 UI 스레드에서 시간이 오래 걸리는 작업을 수행하면 ANR 이 발생하는데,
         * 동기 방식이기 때문에 결과값을 얻어올 때까지 UI 스레드가 잠겨있고, 
         * 해당 작업이 네트워크 관련 작업이라 예외가 발생한 것 같다.
         */
        try {
            Response<User> response = call.execute();
            System.out.println(response.body());
        } catch (IOException e) {
            e.printStackTrace();
        }

        /**
         * 이 방식은 비동기 방식이다.
         * UI 스레드에서 실행해도 NetworkOnMainThreadException 이 발생하지 않았다.
         */
        call.enqueue(new Callback<User>() {
            @Override
            public void onResponse(Call<User> call, Response<User> response) {
                System.out.println(response.body());
            }

            @Override
            public void onFailure(Call<User> call, Throwable t) {
                System.out.println("onFailure");
            }
        });

    }
}
```
