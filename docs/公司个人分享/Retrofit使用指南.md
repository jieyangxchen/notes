
# 一、概述

## 1. 什么是retrofit

retrofit是现在比较流行的网络请求框架，可以理解为okhttp的加强版，底层封装了Okhttp。**准确来说，Retrofit是一个RESTful的http网络请求框架的封装**。因为网络请求工作本质上是由okhttp来完成，而Retrofit负责网络请求接口的封装。

**本质过程**：App应用程序通过Retrofit请求网络，实质上是使用Retrofit接口层封装请求参数、Header、Url等信息，之后由okhttp来完成后续的请求工作。在服务端返回数据后，okhttp将原始数据交给Retrofit，Retrofit根据用户需求解析。

## 2. Retrofit的优点

- **超级解耦 ，接口定义、接口参数、接口回调不在耦合在一起**
- **可以配置不同的httpClient来实现网络请求，如okhttp、httpclient**
- **支持同步、异步、Rxjava**
- **可以配置不同反序列化工具类来解析不同的数据，如json、xml**
- **请求速度快，使用方便灵活简洁**

# 二、注解

Retrofit使用大量注解来简化请求，**Retrofit将okhttp请求抽象成java接口**，使用注解来配置和描述网络请求参数。大概可以分为以下几类，我们先来看看各个注解的含义，再一一去实践解释。

## 1.请求方法注解

| 请求方法注解 | 说明                                                         |
| ------------ | ------------------------------------------------------------ |
| @GET         | get请求                                                      |
| @POST        | post请求                                                     |
| @PUT         | put请求                                                      |
| @DELETE      | delete请求                                                   |
| @PATCH       | patch请求，该请求是对put请求的补充，用于更新局部资源         |
| @HEAD        | head请求                                                     |
| @OPTIONS     | options请求                                                  |
| @HTTP        | 通过注解，可以替换以上所有的注解，它拥有三个属性：method、path、hasBody |

## 2.请求头注解

| 请求头注解 | 说明                                                         |
| ---------- | ------------------------------------------------------------ |
| @Headers   | 用于添加固定请求头，可以同时添加多个，通过该注解的请求头不会相互覆盖，而是共同存在 |
| @Header    | 作为方法的参数传入，用于添加不固定的header，它会更新已有请求头 |

## 3.请求参数注解

| 请求参数注解 | 说明                                                         |
| ------------ | ------------------------------------------------------------ |
| @Body        | 多用于Post请求发送非表达数据，根据转换方式将实例对象转化为对应字符串传递参数，比如使用Post发送Json数据，添加GsonConverterFactory则是将body转化为json字符串进行传递 |
| @Filed       | 多用于Post方式传递参数，需要结合@FromUrlEncoded使用，即以表单的形式传递参数 |
| @FiledMap    | 多用于Post请求中的表单字段，需要结合@FromUrlEncoded使用      |
| @Part        | 用于表单字段，Part和PartMap与@multipart注解结合使用，适合文件上传的情况 |
| @PartMap     | 用于表单字段，默认接受类型是Map<String,RequestBody>，可用于实现多文件上传 |
| @Path        | 用于Url中的占位符                                            |
| @Query       | 用于Get请求中的参数                                          |
| @QueryMap    | 与Query类似，用于不确定表单参数                              |
| @Url         | 指定请求路径                                                 |

## 4.请求和响应格式(标记)注解

| 标记类注解    | 说明                                                         |
| ------------- | ------------------------------------------------------------ |
| @FromUrlCoded | 表示请求发送编码表单数据，每个键值对需要使用@Filed注解       |
| @Multipart    | 表示请求发送form_encoded数据(使用于有文件上传的场景)，每个键值对需要用@Part来注解键名，随后的对象需要提供值 |
| @Streaming    | 表示响应用字节流的形式返回，如果没有使用注解，默认会把数据全部载入到内存中，该注解在下载大文件时特别有用 |

# 三、Retrofit的使用

我们先来解释注解的使用和需要注意的问题(具体的使用步骤后面会给出)。上面提到注解是用来配置和描述网络请求参数的，我们来逐个讲解一下，首先创建网络接口类：

- **Retrofit将okhttp请求抽象成java接口，采用注解描述和配置网络请求参数，用动态代理将该接口的注解“翻译”成一个Http请求，最后执行Http请求。**
- **注意：接口中的每个方法的参数都要用注解标记，否则会报错。**

## 1.注解详解

### 1）@GET、@Query、@QueryMap的使用

```java
public interface Api {
    //get请求
    @GET("user")
    Call<ResponseBody> getData();
}
```

- **@GET**               请求方法注解，get请求，括号内的是请求地址，Url的一部分
- **Call<\*>**              返回类型，*表示接收数据的类，一般自定义
- **getData()**            接口方法名称，括号内可以写入参数

上面的网络接口最简单的一种形式，我们先从简单的开始，一步步深入了解。这是一个没有网络参数的**get**请求方式，需要在方法头部添加@GET注解，表示采用get方法访问网络请求，括号内的是请求的地址(Url的一部分) ，其中返回类型是**Call<\*>**，*表示接收数据的类，如果想直接获取ResponseBody中的内容，可以定义网络请求返回值为`Call<ResponseBody>`，ResponseBody是请求网络后返回的原始数据，如果网络请求没有参数，不用写。

> 这里特别说明Url的组成(下面会讲解到)，retrofit把网络请求的Url分成两部分设置：第一部分在创建Retrofit实例时通过.baseUrl()设置，第二部分在网络接口注解中设置，比如上面接口的"/user"，网络请求的完整地址Url = `Retrofit实例.baseUrl()`+`网络请求接口注解()`。

下面我们来看看有参数的get请求方法：

```java
 @GET("user")
 Call<ResponseBody> getData2(@Query("id") long idLon, @Query("name") String nameStr);
```

- **@Query**            请求参数注解，用于Get请求中的参数
- **"id"/"name"**         参数字段，与后台给的字段需要一致
- **long/String**          声明的参数类型
- **idLon/nameStr**       实际参数

添加参数在方法括号内添加@**Query**，后面是参数类型和参数字段，表示后面idLon的取值作为"id"的值，nameStr的取值作为"name"的值，其实就是键值对，Retrofit会把两个字段**拼接**到接口中，追加到"/user"后面。比如：baseUrl为 [api.github.com/](https://link.juejin.cn/?target=https%3A%2F%2Fapi.github.com%2F) ，那么拼接网络接口注解中的地址后变为：[api.github.com/user](https://link.juejin.cn/?target=https%3A%2F%2Fapi.github.com%2Fuser) ，我们需要传入的id=10006，name="刘亦菲"，那么拼接参数后就是完整的请求地址：`https://api.github.com/user?id=10006&name=刘亦菲`。

```java
 @GET("user")
 Call<ResponseBody> getData3(@QueryMap Map<String, Object> map);
```

- **@QueryMap**               请求参数注解，与@Query类似，用于不确定表单参数
- **Map<String, Object> map**    通过Map将不确定的参数传入，相当于多个Query参数

如果有不确定的把表单参数我们可以使用@**QueryMap**注解来添加参数，通过Map将不确定的参数存入，然后在方法体中带给网络接口，相当于多个Query参数，看看下面的使用：

```java
  Map<String, Object> map = new HashMap<>();
  map.put("id", 10006);
  map.put("name", "刘亦菲");

  Call<ResponseBody> call = retrofit.create(Api.class).getData3(map);
```

### 2）@POST、@FormUrlEncoded、@File、@FileMap、@Body的使用

我们来看看没有参数的post方法的请求：

```java
@POST("user/emails")
Call<ResponseBody> getPostData();
```

- **@POST**            请求方法注解，表示采用post方法访问网络请求，括号后面是部分的URL地址

post请求网络方法，需要在方法头部添加@**POST**注解，表示采用post方法访问网络请求，这里是没有参数的，那有参数的Post请求方法该怎样写呢？

```java
@FormUrlEncoded
@POST("user/emails")
Call<ResponseBody> getPostData2(@Field("name") String nameStr, @Field("sex") String sexStr);
```

- **@FormUrlEncoded**    请求格式注解，请求实体是一个From表单，每个键值对需要使用@Field注解
- **@Field**              请求参数注解，提交请求的表单字段，必须要添加，而且需要配合@FormUrlEncoded使用
- **"name"/"sex"**         参数字段，与后台给的字段需要一致
- **String**               声明的参数类型
- **nameStr/sexStr**       实际参数，表示后面nameStr的取值作为"name"的值，sexStr的取值作为"sex"的值

Post请求如果有参数需要在头部添加@**FormUrlEncoded**注解，**表示请求实体是一个From表单，每个键值对需要使用@Field注解，使用@Field添加参数，这是发送Post请求时，提交请求的表单字段，必须要添加的，而且需要配合@FormUrlEncoded使用**。

```java
 @FormUrlEncoded
 @POST("user/emails")
 Call<ResponseBody> getPsotData3(@FieldMap Map<String, Object> map);

 Map<String, Object> fieldMap = new HashMap<>();
 map.put("name", "刘亦菲");
 map.put("sex", "女");

 Call<ResponseBody> psotData3 = retrofit.create(Api.class).getPostData3(map);

```

- **@FieldMap**                请求参数注解，与@Field作用一致，用于不确定表单参数
- **Map<String, Object> map**   通过Map将不确定的参数传入，相当于多个Field参数

当有多个不确定参数时，我们可以使用@**FieldMap**注解，**@FieldMap与@Field的作用一致，可以用于添加多个不确定的参数**，类似@QueryMap，Map的key作为表单的键，Map的value作为表单的值。

适用于Post请求的还有一个注解**@Body**，**@Body可以传递自定义类型数据给服务器**，多用于post请求发送非表单数据，比如用传递Json格式数据，它可以注解很多东西，比如HashMap、实体类等，我们来看看它用法：

```java
@POST("user/emails")
Call<ResponseBody> getPsotDataBody(@Body RequestBody body);
```

- **@Body**             上传json格式数据，直接传入实体它会自动转为json，这个转化方式是GsonConverterFactory定义的

**特别注意：@Body注解不能用于表单或者支持文件上传的表单的编码，即不能与@FormUrlEncoded和@Multipart注解同时使用，否则会报错，如下图：**

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e45a2e84c52445c0801012e7b15b725b~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.image)

我们找到@Body的源码发现，isFormEncoded为true或者isMultipart为ture时，满足其中任何一个条件都会抛出该异常，通过变量名字可以看出使用@ FormUrlEncoded 或者@Multipart 标签。

```java
 if (annotation instanceof Body) {
    if (isFormEncoded || isMultipart) {
       throw parameterError(method, p,
          "@Body parameters cannot be used with form or multi-part encoding.");
     }
  }
```

### 3）@HTTP

@HTTP注解的作用是替换@GET、@POST、@PUT、@DELETE、@HEAD以及更多拓展功能

```java
@HTTP(method = "GET", path = "user/keys", hasBody = false)
Call<ResponseBody> getHttpData();
```

- **method**       表示请求的方法，区分大小写，这里的值retrofit不会再做任何处理，必须要保证正确
- **path**          网络请求地址路径
- **hasBody**      是否有请求体，boolean类型

@HTTP注解可以通过method字段灵活设置具体请求方法，通过path设置网络请求地址，用的比较少。@HTTP替换@POST、@PUT、@DELETE、@HEAD等方法的用法类似，这里就不一一讲解了。

### 4）@Path

```java
 @GET("orgs/{id}")
 Call<ResponseBody> getPathData(@Query("name") String nameStr, @Path("id") long idLon);
```

- **@Query**          get请求方法参数的注解，上面已经解释了，这里就不重复讲
- **@Path**           请求参数注解，用于Url中的占位符{}，所有在网址中的参数

@**Path**注解用于Url中的占位符`{}`，所有在网址中的参数，如上面 `@GET("orgs/{id}")`的id，通过`{}`占位符来标记id，使用@Path注解传入idLon的值，注意有的Url既有占位符又有"?"后面的键值对，其实@Query和@Path两者是可以共用的。在发起请求时，{id}会被替换为方法中第二个参数的值idLon。

### 5）@Url

**如果需要重新地址接口地址，可以使用@Url**，将地址以参数的形式传入即可。如果有@Url注解时，GET传入的Url可以省略。

```java
@GET
Call<ResponseBody> getUrlData(@Url String nameStr, @Query("id") long idLon);
```

- **@Url**             表示指定请求路径，可以当做参数传入

### 6）@Header、@Headers

我们可以在方法参数内添加请求头，**@Header用于添加不固定的请求头**，作用于方法的参数，作为方法的参数传入，该注解会更新已有的请求头。

```java
 @GET("user/emails")
 Call<ResponseBody> getHeaderData(@Header("token") String token);
```

- **@header**           请求头注解，用于添加不固定请求头

我们想对某个方法添加固定请求头时可以参考下面的写法，@headers用于添加固定的请求头，作用于方法，可以同时添加多个，通过该注解添加的请求头不会相互覆盖，而是共同存在。

```java
 @Headers({"phone-type:android", "version:1.1.1"})
 @GET("user/emails")
 Call<ResponseBody> getHeadersData();

```

- **@headers**           请求头注解，用于添加固定请求头，可以添加多个

### 7）@Streaming

```java
 @Streaming
 @POST("gists/public")
 Call<ResponseBody> getStreamingBig();

```

- **@Streaming**         表示响应体的数据用流的方式返回，使用于返回数据比较大，该注解在下载大文件时特别有用

我们在使用下载比较大的文件的时候需要添加@Streaming注解

### 8）@Multipart、@part、@PartMap

```java
@Multipart
@POST("user/followers")
Call<ResponseBody> getPartData(@Part("name") RequestBody name, @Part MultipartBody.Part file);

```

- **@Multipart**       表示请求实体是一个支持文件上传的表单，需要配合@Part和@PartMap使用，适用于文件上传
- **@Part**            用于表单字段，适用于文件上传的情况，@Part支持三种类型：RequestBody、MultipartBody.Part、任意类型
- **@PartMap**        用于多文件上传， 与@FieldMap和@QueryMap的使用类似

上面的使用是一个上传文字和文件的写法，在使用@Part注解时需要在头部添加@**Multipart**注解，实现支持文件上传，我们来看看上面代码怎么使用：

```java
//声明类型,这里是文字类型
 MediaType textType = MediaType.parse("text/plain");
//根据声明的类型创建RequestBody,就是转化为RequestBody对象
RequestBody name = RequestBody.create(textType, "这里是你需要写入的文本：刘亦菲");

//创建文件，这里演示图片上传
File file = new File("文件路径");
if (!file.exists()) {
   file.mkdir();
 }

//将文件转化为RequestBody对象
//需要在表单中进行文件上传时，就需要使用该格式：multipart/form-data
RequestBody imgBody = RequestBody.create(MediaType.parse("image/png"), file);
//将文件转化为MultipartBody.Part
//第一个参数：上传文件的key；第二个参数：文件名；第三个参数：RequestBody对象
MultipartBody.Part filePart = MultipartBody.Part.createFormData("file", file.getName(), imgBody);

Call<ResponseBody> partDataCall = retrofit.create(Api.class).getPartData(name, filePart);

```

首先声明类型，然后根据类型转化为**RequestBody**对象，返回RequestBody或者转化为 `MultipartBody.Part`，需要在表单中进行文件上传时，就需要使用该格式：`multipart/form-data`。

@PartMap的使用与@FieldMap和@QueryMap的使用类似，用于多文件上传，我们直接看代码：

```java
 @Multipart
 @POST("user/followers")
 Call<ResponseBody> getPartMapData(@PartMap Map<String, MultipartBody.Part> map);


    File file1 = new File("文件路径");
    File file2 = new File("文件路径");
        if (!file1.exists()) {
        file1.mkdir();
    }
        if (!file2.exists()) {
        file2.mkdir();
    }

    RequestBody requestBody1 = RequestBody.create(MediaType.parse("image/png"), file1);
    RequestBody requestBody2 = RequestBody.create(MediaType.parse("image/png"), file2);
    MultipartBody.Part filePart1 = MultipartBody.Part.createFormData("file1", file1.getName(), requestBody1);
    MultipartBody.Part filePart2 = MultipartBody.Part.createFormData("file2", file2.getName(), requestBody2);

    Map<String,MultipartBody.Part> mapPart = new HashMap<>();
        mapPart.put("file1",filePart1);
        mapPart.put("file2",filePart2);

    Call<ResponseBody> partMapDataCall = retrofit.create(Api.class).getPartMapData(mapPart);

```

上面的`(@PartMap Map<String, MultipartBody.Part> map)`方法参数中的 MultipartBody.Part 可以是RequestBody、String、MultipartBody.Part等类型，可以根据个人需要更换，这里就不一一说明了。



# 四、项目中使用retrofit-spring-boot-starter

## 0.基础配置

> 依赖

```xml
				<!-- retrofit start -->
        <dependency>
            <groupId>com.github.lianjiatech</groupId>
            <artifactId>retrofit-spring-boot-starter</artifactId>
            <version>2.2.22</version>
        </dependency>
        <!-- retrofit end -->
```

> 配置文件

```yml
retrofit:

  # 连接池配置
  pool:
    # test1连接池配置
    test1:
      # 最大空闲连接数
      max-idle-connections: 3
      # 连接保活时间(秒)
      keep-alive-second: 100
  test2:
    # 最大空闲连接数
    max-idle-connections: 10
    # 连接保活时间(秒)
    keep-alive-second: 100
  # 是否禁用void返回值类型
  disable-void-return-type: false


  # 全局转换器工厂
  global-converter-factories:
    - com.github.lianjiatech.retrofit.spring.boot.core.BasicTypeConverterFactory
    - retrofit2.converter.jackson.JacksonConverterFactory
  # 全局调用适配器工厂
  global-call-adapter-factories:
    - com.github.lianjiatech.retrofit.spring.boot.core.BodyCallAdapterFactory
    - com.github.lianjiatech.retrofit.spring.boot.core.ResponseCallAdapterFactory

  # 日志打印配置
  log:
    # 启用日志打印
    enable: true
    # 日志打印拦截器
    logging-interceptor: com.github.lianjiatech.retrofit.spring.boot.interceptor.DefaultLoggingInterceptor
    # 全局日志打印级别
    global-log-level: info
    # 全局日志打印策略
    global-log-strategy: body


  # 重试配置
  retry:
    # 是否启用全局重试
    enable-global-retry: true
    # 全局重试间隔时间
    global-interval-ms: 1
    # 全局最大重试次数
    global-max-retries: 1
    # 全局重试规则
    global-retry-rules:
      - response_status_not_2xx
      - occur_io_exception
    # 重试拦截器
    retry-interceptor: com.github.lianjiatech.retrofit.spring.boot.retry.DefaultRetryInterceptor

  # 熔断降级配置
  degrade:
    # 是否启用熔断降级
    enable: false
    # 熔断降级实现方式
    degrade-type: sentinel
    # 熔断资源名称解析器
    resource-name-parser: com.github.lianjiatech.retrofit.spring.boot.degrade.DefaultResourceNameParser

  # 全局连接超时时间
  global-connect-timeout-ms: 5000
  # 全局读取超时时间
  global-read-timeout-ms: 5000
  # 全局写入超时时间
  global-write-timeout-ms: 5000
  # 全局完整调用超时时间
  global-call-timeout-ms: 0
```

## 1.自定义注入OkHttpClient

通常情况下，通过`@RetrofitClient`注解属性动态创建`OkHttpClient`对象能够满足大部分使用场景。但是在某些情况下，用户可能需要自定义`OkHttpClient` ，这个时候，可以在接口上定义返回类型是`OkHttpClient.Builder`的静态方法来实现。代码示例如下：

```java
@RetrofitClient(baseUrl = "http://ke.com")
public interface HttpApi3 {

   @OkHttpClientBuilder
   static OkHttpClient.Builder okhttpClientBuilder() {
      return new OkHttpClient.Builder()
              .connectTimeout(1, TimeUnit.SECONDS)
              .readTimeout(1, TimeUnit.SECONDS)
              .writeTimeout(1, TimeUnit.SECONDS);
   }

   @GET
   Result<Person> getPerson(@Url String url, @Query("id") Long id);
}
```

> 方法必须使用`@OkHttpClientBuilder`注解标记！

## 2.调用适配器和数据转码器

### 1）调用适配器

`Retrofit`可以通过调用适配器`CallAdapterFactory`将`Call<T>`对象适配成接口方法的返回值类型。`retrofit-spring-boot-starter`扩展2种`CallAdapterFactory` 实现：

1. ```
   BodyCallAdapterFactory
   ```

    - 默认启用，可通过配置`retrofit.enable-body-call-adapter=false`关闭
    - 同步执行http请求，将响应体内容适配成接口方法的返回值类型实例。
    - 除了`Retrofit.Call<T>`、`Retrofit.Response<T>`、`java.util.concurrent.CompletableFuture<T>`之外，其它返回类型都可以使用该适配器。

2. ```
   ResponseCallAdapterFactory
   ```

    - 默认启用，可通过配置`retrofit.enable-response-call-adapter=false`关闭
    - 同步执行http请求，将响应体内容适配成`Retrofit.Response<T>`返回。
    - 如果方法的返回值类型为`Retrofit.Response<T>`，则可以使用该适配器。

**Retrofit自动根据方法返回值类型选用对应的`CallAdapterFactory`执行适配处理！加上Retrofit默认的`CallAdapterFactory`，可支持多种形式的方法返回值类型：**

- 基础类型(`String`/`Long`/`Integer`/`Boolean`/`Float`/`Double`)：直接将响应内容转换为上述基础类型。
- 其它任意POJO类型： 将响应体内容适配成一个对应的POJO类型对象返回，如果http状态码不是2xx，直接抛错！（推荐）
- `CompletableFuture<T>`: 将响应体内容适配成`CompletableFuture<T>`对象返回！（异步调用推荐）
- `Void`: 不关注返回类型可以使用`Void`。如果http状态码不是2xx，直接抛错！（不关注返回值）
- `Response<T>`: 将响应内容适配成`Response<T>`对象返回！（不推荐）
- `Call<T>`: 不执行适配处理，直接返回`Call<T>`对象！（不推荐）

```java
    /**
     * 其他任意Java类型
     * 将响应体内容适配成一个对应的Java类型对象返回，如果http状态码不是2xx，直接抛错！
     * @param id
     * @return
     */
    @GET("person")
    Result<Person> getPerson(@Query("id") Long id);

    /**
     *  CompletableFuture<T>
     *  将响应体内容适配成CompletableFuture<T>对象返回
     * @param id
     * @return
     */
    @GET("person")
    CompletableFuture<Result<Person>> getPersonCompletableFuture(@Query("id") Long id);

    /**
     * Void
     * 不关注返回类型可以使用Void。如果http状态码不是2xx，直接抛错！
     * @param id
     * @return
     */
    @GET("person")
    Void getPersonVoid(@Query("id") Long id);

    /**
     *  Response<T>
     *  将响应内容适配成Response<T>对象返回
     * @param id
     * @return
     */
    @GET("person")
    Response<Result<Person>> getPersonResponse(@Query("id") Long id);

    /**
     * Call<T>
     * 不执行适配处理，直接返回Call<T>对象
     * @param id
     * @return
     */
    @GET("person")
    Call<Result<Person>> getPersonCall(@Query("id") Long id);
```

**我们也可以通过继承`CallAdapter.Factory`扩展实现自己的`CallAdapter`**！

`retrofit-spring-boot-starter`支持通过`retrofit.global-call-adapter-factories`配置全局调用适配器工厂，工厂实例优先从Spring容器获取，如果没有获取到，则反射创建。默认的全局调用适配器工厂是`[BodyCallAdapterFactory, ResponseCallAdapterFactory]`！

```yaml
retrofit:
  # 全局调用适配器工厂
  global-call-adapter-factories:
    - com.github.lianjiatech.retrofit.spring.boot.core.BodyCallAdapterFactory
    - com.github.lianjiatech.retrofit.spring.boot.core.ResponseCallAdapterFactory
```

针对每个Java接口，还可以通过`@RetrofitClient`注解的`callAdapterFactories()`指定当前接口采用的`CallAdapter.Factory`，指定的工厂实例依然优先从Spring容器获取。

**注意：如果`CallAdapter.Factory`没有`public`的无参构造器，请手动将其配置成`Spring`容器的`Bean`对象**！

### 2）数据转码器

`Retrofit`使用`Converter`将`@Body`注解标注的对象转换成请求体，将响应体数据转换成一个`Java`对象，可以选用以下几种`Converter`：

- [Gson](https://github.com/google/gson): com.squareup.Retrofit:converter-gson
- [Jackson](https://github.com/FasterXML/jackson): com.squareup.Retrofit:converter-jackson
- [Moshi](https://github.com/square/moshi/): com.squareup.Retrofit:converter-moshi
- [Protobuf](https://developers.google.com/protocol-buffers/): com.squareup.Retrofit:converter-protobuf
- [Wire](https://github.com/square/wire): com.squareup.Retrofit:converter-wire
- [Simple XML](http://simple.sourceforge.net/): com.squareup.Retrofit:converter-simplexml
- [JAXB](https://docs.oracle.com/javase/tutorial/jaxb/intro/index.html): com.squareup.retrofit2:converter-jaxb
- fastJson：com.alibaba.fastjson.support.retrofit.Retrofit2ConverterFactory

`retrofit-spring-boot-starter`支持通过`retrofit.global-converter-factories`配置全局数据转换器工厂，转换器工厂实例优先从Spring容器获取，如果没有获取到，则反射创建。默认的全局数据转换器工厂是`retrofit2.converter.jackson.JacksonConverterFactory`，你可以直接通过`spring.jackson.*`配置`jackson`序列化规则，配置可参考[Customize the Jackson ObjectMapper](https://docs.spring.io/spring-boot/docs/2.1.5.RELEASE/reference/htmlsingle/#howto-customize-the-jackson-objectmapper)！

```
retrofit:
  # 全局转换器工厂
   global-converter-factories:
      - retrofit2.converter.jackson.JacksonConverterFactory
```

针对每个Java接口，还可以通过`@RetrofitClient`注解的`converterFactories()`指定当前接口采用的`Converter.Factory`，指定的转换器工厂实例依然优先从Spring容器获取。

**注意：如果`Converter.Factory`没有`public`的无参构造器，请手动将其配置成`Spring`容器的`Bean`对象**！

## 3.其他使用参考

### 1）form参数接口调用

```java
@FormUrlEncoded
@POST("token/verify")
Object tokenVerify(@Field("source") String source,@Field("signature") String signature,@Field("token") String token);


@FormUrlEncoded
@POST("message")
CompletableFuture<Object> sendMessage(@FieldMap Map<String, Object> param);
```

### 2）上传文件

#### 构建MultipartBody.Part

```java
// 对文件名使用URLEncoder进行编码
public ResponseEntity importTerminology(MultipartFile file){
        String fileName=URLEncoder.encode(Objects.requireNonNull(file.getOriginalFilename()),"utf-8");
        okhttp3.RequestBody requestBody=okhttp3.RequestBody.create(MediaType.parse("multipart/form-data"),file.getBytes());
        MultipartBody.Part part=MultipartBody.Part.createFormData("file",fileName,requestBody);
        apiService.upload(part);
        return ok().build();
        }
```

#### http上传接口

```java
@POST("upload")
@Multipart
Void upload(@Part MultipartBody.Part file);
```

### 3）下载文件

#### http下载接口

```java
@RetrofitClient(baseUrl = "https://img.ljcdn.com/hc-picture/")
public interface DownloadApi {

    @GET("{fileKey}")
    Response<ResponseBody> download(@Path("fileKey") String fileKey);
}
```

#### http下载使用

```java
@SpringBootTest(classes = RetrofitTestApplication.class)
@RunWith(SpringRunner.class)
public class DownloadTest {
    @Autowired
    DownloadApi downLoadApi;

    @Test
    public void download() throws Exception {
        String fileKey = "6302d742-ebc8-4649-95cf-62ccf57a1add";
        Response<ResponseBody> response = downLoadApi.download(fileKey);
        ResponseBody responseBody = response.body();
        // 二进制流
        InputStream is = responseBody.byteStream();

        // 具体如何处理二进制流，由业务自行控制。这里以写入文件为例
        File tempDirectory = new File("temp");
        if (!tempDirectory.exists()) {
            tempDirectory.mkdir();
        }
        File file = new File(tempDirectory, UUID.randomUUID().toString());
        if (!file.exists()) {
            file.createNewFile();
        }
        FileOutputStream fos = new FileOutputStream(file);
        byte[] b = new byte[1024];
        int length;
        while ((length = is.read(b)) > 0) {
            fos.write(b, 0, length);
        }
        is.close();
        fos.close();
    }
}
```

### 4）动态URL

使用`@url`注解可实现动态URL。

**注意：`@url`必须放在方法参数的第一个位置。原有定义`@GET`、`@POST`等注解上，不需要定义端点路径**！

```java
 @GET
 Map<String, Object> test3(@Url String url,@Query("name") String name);
```

### 5）DELETE请求传请求体

```java
@HTTP(method = "DELETE", path = "/user/delete", hasBody = true)
```

> 参考文档：
>
> https://github.com/LianjiaTech/retrofit-spring-boot-starter
> https://juejin.cn/post/6978777076073660429