---
title: Retrofit源码详细解析
date: 2017-12-30 13:24:44
tags: [开源框架, Java, Android, 网络]
---
### 概述
最近公司准备搞一个网络框架统一所有项目的网络请求，借此机会重新复习一下目前比较火的网络框架，顺便再进一步，写一写源码分析，所以这篇文章属于进阶类型，不会去介绍这个框架的用法，而是从简单的代码示例，一步步分析源码，走完整个框架的流程。之后可能会附上okio和okhttp的源码分析。  
简单说一下Retrofit，Retrofit是一个用于简化HTTP请求的库，[官方网站](http://square.github.io/retrofit/)。截止目前（2017-12-27），Retrofit的版本为2.3.0，本文分析的代码都基于该版本。

### 示例
先从示例代码开始入手

```Java
public interface GitHubService {
  @GET("users/{user}/repos")
  Call<List<Repo>> listRepos(@Path("user") String user);
}
//builder模式生成对象
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .build();

GitHubService service = retrofit.create(GitHubService.class);
Call<List<Repo>> call = service.listRepos("octocat");
call.enqueue(new Callback<List<Repo>>(){
    @Override
    public void onResponse(@NonNull Call<List<Repo>> call, @NonNull Response<List<Repo> response) {
    }

    @Override
    public void onFailure(@NonNull Call<List<Repo>> call, @NonNull Throwable t) {
    }
}
```

如上是一次简单的GET请求，结果通过异步返回。第一次看到这样的代码，可能会有如下几个简单的问题
1. 为什么这个接口没有实现类，却得到了这个接口子类的实例
2. 为什么接口内声明的方法的返回值是`Call`
3. `Call`的泛型类型能不能被随意改变

接下来将通过代码流程分析一步步解决这些问题

### 流程分析
#### 创建Retrofit实例

先看Retrofit的构建，在这里会初始化很多变量，先不用深究，也不要通过变量名进行额外的推测，随着代码的深入，每个变量的作用都会被展示出来。`Retrofit`类有一个静态内部类`Builder`，很明显的建造者模式，除了构造方法和`Retrofit.Builder.build()`方法，其他方法都是一些简单的判断赋值操作。    

变量声明：

```Java
private final Platform platform;
private @Nullable okhttp3.Call.Factory callFactory;
//okhttp中对url的封装
private HttpUrl baseUrl;
private final List<Converter.Factory> converterFactories = new ArrayList<>();
private final List<CallAdapter.Factory> adapterFactories = new ArrayList<>();
private @Nullable Executor callbackExecutor;
```

构造方法：

```Java
Builder(Platform platform) {
  this.platform = platform;
  // Add the built-in converter factory first. This prevents overriding its behavior but also
  // ensures correct behavior when using converters that consume all types.
  converterFactories.add(new BuiltInConverters());
}
```

build方法代码：  

```Java
public Retrofit build() {
  if (baseUrl == null) {
    throw new IllegalStateException("Base URL required.");
  }

  okhttp3.Call.Factory callFactory = this.callFactory;
  if (callFactory == null) {
    callFactory = new OkHttpClient();
  }

  Executor callbackExecutor = this.callbackExecutor;
  if (callbackExecutor == null) {
    callbackExecutor = platform.defaultCallbackExecutor();
  }

  // Make a defensive copy of the adapters and add the default Call adapter.
  List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
  adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

  // Make a defensive copy of the converters.
  List<Converter.Factory> converterFactories = new ArrayList<>(this.converterFactories);

  return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
      callbackExecutor, validateEagerly);
}
```

一上来就是三个代码段，但是内容都很简单。有几个重要的成员变量：
- callFactory
- callbackExecutor
- adapterFactories
- converterFactories

从build方法中可以看到，如果callFactory为空，则将其赋值为`OkHttpClient`，使用过okhttp就会知道，这个类可以通过接受一个`Request`返回一个`Call`用以进行网络请求，因此Retrofit所有请求的发送都是通过callFactory利用okhttp完成的。  
callbackExecutor类型为`Executor`，线程池，如果为空，通过`Platform.defaultCallbackExecutor()`来获取默认的Executor。  
platform用来针对每个平台进行处理，Retrofit中提供了两个平台的处理，Java8和Android。这里只分析Android。  

```Java
static class Android extends Platform {
  @Override public Executor defaultCallbackExecutor() {
    return new MainThreadExecutor();
  }

  @Override CallAdapter.Factory defaultCallAdapterFactory(@Nullable Executor callbackExecutor) {
    if (callbackExecutor == null) throw new AssertionError();
    return new ExecutorCallAdapterFactory(callbackExecutor);
  }

  static class MainThreadExecutor implements Executor {
    private final Handler handler = new Handler(Looper.getMainLooper());

    @Override public void execute(Runnable r) {
      handler.post(r);
    }
  }
}
```

`Android.defalutCallbackExecutor`返回`MainThreadExecutor`，具体的实现是，获得一个MainThread的Handle，将Runnable post到Android的主线程执行。  
回到build方法，callbackExecutor赋值完成之后，向adapterFactories的**末尾**插入了`Platform.defaultCallAdapterFactory()`的返回值，这个方法的代码在上面也贴了出来。  
`defaultCallAdapterFactory()`将传入的callbackExecutor作为参数，构建了一个`ExecutorCallAdapterFatory`的实例。这个类做什么的，先不深究，接着看converterFactories，在Retrofit.Builder构造时，向converterFactories这个空的List中插入了一个`BuiltInConverters`的实例，因此之后的add操作，都会添加到`BuiltInConverters`实例的后面。  
这两个List的泛型分别为`CallAdapter.Factory`和`Converter.Facotry`。Factory的意思显而易见，工厂模式，可见它们是用来创建`CallAdapter`和`ConverterAdapter`的实例的。  
`CallAdapter`和`ConverterAdapter`的解析放在下面，结合流程下来，会对为什么设计这两个接口有一个更深的认识。

#### 动态代理接口 retrofit#create()
在了解完Retrofit的构建之后，知道了几个用于Retrofit的成员变量，接下来看一下如何创建对应的接口的实例，创建的代码位于`Retrofit#create()`方法

```Java
public <T> T create(final Class<T> service) {
  Utils.validateServiceInterface(service);
  if (validateEagerly) {
    eagerlyValidateMethods(service);
  }
  return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
    new InvocationHandler() {
      private final Platform platform = Platform.get();
  
      @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args) throws Throwable {
        // If the method is a method from Object then defer to normal invocation.
        if (method.getDeclaringClass() == Object.class) {
          return method.invoke(this, args);
        }
        if (platform.isDefaultMethod(method)) {
          return platform.invokeDefaultMethod(method,   service, proxy, args);
        }
        ServiceMethod<Object, Object> serviceMethod = (ServiceMethod<Object, Object>) loadServiceMethod(method);
        OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
        return serviceMethod.callAdapter.adapt(okHttpCall);
      }
    });
}
```
先判断传入的class是否是接口，并且接口不能继承其他接口。为什么必须是接口？因为`Retrofit`需要动态代理完成统一的逻辑，而Java的动态代理只支持接口。  
这里问题1就有答案了，接口的实例是通过动态代理生成的。
`eagerlyValidMethods()`用来尽快验证声明的method是否合法。  
通过`Proxy.newProxyInstance()`传入一个回调来创建一个动态代理，调用接口内方法的操作都会由创建动态代理时所传入的回调来代理执行。  
`InvocationHandler.invoke()`方法中，先进行了判断，如果是属于`Object`类的方法，则不做代理，让其以原本的逻辑处理。然后根据平台判断，进行相应的处理。Retrofit中，兼容了Java8接口中声明的`Default`方法，这里不做分析。  
接下来是通过`loadMethod()`方法，取得一个`ServiceMethod`实例。这里先跳转到`Retrofit.loadServiceMethod()`方法中。

```Java
ServiceMethod<?, ?> loadServiceMethod(Method method) {
    ServiceMethod<?, ?> result = serviceMethodCache.get(method);
    if (result != null) return result;
    
    synchronized (serviceMethodCache) {
        result = serviceMethodCache.get(method);
        if (result == null) {
            result = new ServiceMethod.Builder<>(this, method).build();
            serviceMethodCache.put(method, result);
        }
    }
    return result;
}
```
这里有一个`ConcurrentHashMap`作为缓存，缓存`ServiceMethod`实例，如果取到缓存，直接返回。否则，通过`ServiceMethod.Builder`构建一个实例，并放入缓存。这个`ServiceMehod`是做什么的呢？这里暂且先放一下，之后再进行分析。  
回到`Retrofit.create()`方法中。在取得了一个`ServiceMethod`实例之后，又新建了一个OkHttpCall的实例。`OkHttpCall`实现了`Call`接口，这个`Call`接口会在后面详细讲解。  
最后，`Retrofit.create()`方法，返回了`ServiceMethod.callAdapter.adapt()`方法的返回值。


#### 调用声明的接口方法
`Retrofit.create()`调用结束后，就获得了一个声明的接口实现类的实例，通过调用方法，可以获得`Call`类型的实例，然后通过实例，调用enqueue或者execute来进行请求。从异步开始分析，跟进enqueue看一下。`Call`接口的声明如下：

```Java
public interface Call<T> extends Cloneable {
  Response<T> execute() throws IOException;

  void enqueue(Callback<T> callback);

  boolean isExecuted();

  void cancel();

  boolean isCanceled();

  Call<T> clone();

  Request request();
}
```

然后找到其真正实现类`OkHttpCall`的`enqueue`实现（其他实现都是代理，主要逻辑还是在`OkHttpCall`中：

```Java
@Override public void enqueue(final Callback<T> callback) {
    okhttp3.Call call;
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already executed.");
      executed = true;
      call = rawCall;
      if (call == null && failure == null) {
        call = rawCall = createRawCall();
      }
    }
    if (canceled) {
      call.cancel();
    }
    call.enqueue(new okhttp3.Callback() {
      @Override 
      public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse)
          throws IOException {
        Response<T> response;
        try {
          response = parseResponse(rawResponse);
        } catch (Throwable e) {
          callFailure(e);
          return;
        }
        callSuccess(response);
      }
      
      /*
       * other implementation
       */
    });
}
```
这段代码删去了错误处理和`okhttp3.Callback`的某些实现方法，以减少代码阅读的数量。  
这里首先判断了这个Call的实例是否已经被执行过了，如果被执行过，那么就抛出异常，这里和okhttp是相似的。并在调用前，判断是否已经被取消了，若已经取消，则调用`rawCall.cancel()`来完成取消操作。rawCall类型是`okhttp3.Call`，enqueue的操作实际上是通过okhttp来完成的。  
rawCall的创建，则是通过`OkHttpCall.createRawCall()`来完成的。

```Java
private okhttp3.Call createRawCall() throws IOException {
  Request request = serviceMethod.toRequest(args);
  okhttp3.Call call = serviceMethod.callFactory.newCall(request);
  if (call == null) {
    throw new NullPointerException("Call.Factory returned null.");
  }
  return call;
}
```

又调用到了ServiceMethod，那么知道了`OkHttpCall.rawCall`是通过`ServiceMethod.callFactory`创建的，而这个callFactory，默认值的类型是`OkHttpClient`。这里的代码实际上就是okhttp的请求操作。  
使用`okhttp3.Call.enqueue()`，传入回调，在回调成功时，调用`parseResponse()`。

```Java
Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
  ResponseBody rawBody = rawResponse.body();
    
  // Remove the body's source (the only stateful object) so we can pass the response along.
  rawResponse = rawResponse.newBuilder()
    .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
    .build();
    
  int code = rawResponse.code();
  if (code < 200 || code >= 300) {
    try {
      // Buffer the entire body to avoid future I/O.
      ResponseBody bufferedBody = Utils.buffer(rawBody);
      return Response.error(bufferedBody, rawResponse);
    } finally {
      rawBody.close();
    }
  }
    
  if (code == 204 || code == 205) {
    rawBody.close();
    return Response.success(null, rawResponse);
  }
    
  ExceptionCatchingRequestBody catchingBody = new ExceptionCatchingRequestBody(rawBody);
  try {
    T body = serviceMethod.toResponse(catchingBody);
    return Response.success(body, rawResponse);
  } catch (RuntimeException e) {
    // If the underlying source threw an exception, propagate that rather than indicating it was
    // a runtime exception.
    catchingBody.throwIfCaught();
    throw e;
  }
}
```
注意，这里出现了两个Response类，其中一个是okhttp中声明的Response，全类名为`okhttp3.Response`；另一个是Retrofit中的Response，代码中直接写为`Response`，这段代码的逻辑就是将okHttp的Response转换为Retrofit的Response，相当于一层额外的封装。返回值类型是`Response<T>`，返回的值，会作为`Callback`回调的参数传递，`T`为`Response.body`的类型。  
当响应码小于200且大于300时，则认为错误，调用Response#error()生成包含错误信息的`Response<T>`；当响应码为204或者205时，也认为成功，但是根据HTTP响应码的规定，204以及205的响应没有响应体。  
如果响应码是2xx，并且不为204和205，则构造一个`ExcepitonCatchingRequestBody`的实例。`ExceptionCatchingResponseBody`是对`okhttp3.ResposneBody`的一个简单的代理封装，可以把它就看做一个ResponseBody。  
调用`ServiceMethod.toResposne()`方法，将catchingBody转换为`T`类型，最后封装okhttp的reposne实例，得到Retrofit的reposne实例。  

流程走到这里，有关ServiceMethod的方法都没有分析，最主要的是下面这几个方法：
- ServiceMethod.callAdapter.adapter()
- ServiceMethod.toRequest()
- ServiceMethod.toResponse()

接下来，以这些方法为切入点，分析ServiceMethod。

#### SerivceMethod
##### serviceMehtod#callAdapter
查看`ServiceMethod`中的成员变量callAdapter，其类型为接口`CallAdapter<R,T>`，检索一下他的赋值，发现在ServiceMehtod的构造函数中，通过`ServiceMehtod.Builder`进行赋值，那么应该就是构造时传入，或者在`Builder.build()`方法中了。  
果不其然，在`ServiceMethod.Builder#build`中，发现了`callAdapter = createCallAdapter();`

```Java
private CallAdapter<T, R> createCallAdapter() {
  Type returnType = method.getGenericReturnType();
  if (Utils.hasUnresolvableType(returnType)) {
    throw methodError(
        "Method return type must not include a type variable or wildcard: %s", returnType);
  }
  if (returnType == void.class) {
    throw methodError("Service methods cannot return void.");
  }
  Annotation[] annotations = method.getAnnotations();
  try {
    //noinspection unchecked
    return (CallAdapter<T, R>) retrofit.callAdapter(returnType, annotations);
  } catch (RuntimeException e) { // Wide exception range because factories are user code.
    throw methodError(e, "Unable to create call adapter for %s", returnType);
  }
}
```
先获取成员变量method的返回值，所有注解（这个method是ServiceMethod构造时传入的），检查返回值的类型是否能够被支持，然后返回`retrofit.callAdapter()`。可以确定，这个callAdapter是通过Retrofit类取到的。
跟进调用，最后调用到`Retrofit.nextCallAdapter()`。

```Java
public CallAdapter<?, ?> nextCallAdapter(@Nullable CallAdapter.Factory skipPast, Type returnType,
      Annotation[] annotations) {
    checkNotNull(returnType, "returnType == null");
    checkNotNull(annotations, "annotations == null");
    
    int start = adapterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = adapterFactories.size(); i < count; i++) {
      CallAdapter<?, ?> adapter = adapterFactories.get(i).get(returnType, annotations, this);
      if (adapter != null) {
        return adapter;
      }
    }
    //省略后面的异常处理
}
```
返回的callAdapter是从adapterFactories中取到的，这个adapterFactories在上面提到过。Retrofit是按顺序从list中取CallAdapterFactory的，也就是说，这里CallAdapter会取，先被加入到list中，并且符合条件（调用`CallAdapter.Factory.get()`返回不为空）的CallAdapterFactory。  
说了这么久的`CallAdapter`，那么这个接口到底是做什么用的，这就来分析一下。

#### CallAdapter和CallAdapter.Factory
`CallAdapter`:
 
```Java
public interface CallAdapter<R, T> {
  Type responseType();

  T adapt(Call<R> call);

  abstract class Factory {
    public abstract @Nullable CallAdapter<?, ?> get(Type returnType, Annotation[] annotations,
        Retrofit retrofit);

    protected static Type getParameterUpperBound(int index, ParameterizedType type) {
      return Utils.getParameterUpperBound(index, type);
    }

    protected static Class<?> getRawType(Type type) {
      return Utils.getRawType(type);
    }
  }
}
```

CallAdapter有两个泛型类型，`R`和`T`，根据方法adapt的参数和返回值来看，应该是将`Call<R>`的对象转换为`T`的对象。  
先来看`CallAdapter.Factory`的实现，Retrofit中，默认为`ExecutorCallAdapterFactory`，这个类之前也提到过，向`Retrofit.adapterFactories`末尾插入的实例，就是这个类型的，列一下`CallAdapter.Factory.get()`的实现源码：

```Java
//ExecutorCallAdapterFactory#get()
@Override
public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
  if (getRawType(returnType) != Call.class) {
    return null;
  }
  final Type responseType = Utils.getCallResponseType(returnType);
  return new CallAdapter<Object, Call<?>>() {
    @Override public Type responseType() {
      return responseType;
    }
  
    @Override public Call<Object> adapt(Call<Object> call) {
      return new ExecutorCallbackCall<>(callbackExecutor, call);
    }
  };
}
```
get()方法先是判断了传入的返回类型，是否为`Call`，是的话返回一个`CallAdapter`的实例。由此可见，`CallAdapter.Factory`向外暴露了接口，通过实现这个接口，来创建CallAdapter，这是工厂模式的一个应用。  
然后看一下实现了`CallAdapter`的匿名类，这里直接返回了一个`ExecutorCallbackCall`的实例，这个类仅仅是一个简单的代理，代码就不贴了。  
`Service.callAdapter.adapt`方法，接收了OkHttpCall的实例，以默认的CallAdapter.Factory来看，返回值类型就是`ExecutorCallbackCall`了。因此，最开始的问题2得到了解答，为什么声明的方法返回值为`Call`。那么，如果改变了CallAdapter声明时的泛型类型，是不是可以改变声明方法的返回值呢？答案是可以的，Retrofit也提供了RxJava2的CallAdapter，可以在调用方法后得到`Observable`，进行数据的操作。  
通过继承`CallAdapter.Factory`，可以利用传入的returnType和annotation来构建一个CallAdapter，作为定义方法的返回值。这样提供了极高的自定义性，不用修改代码就可以定义返回的类型。而工厂模式的利用也大大减少了耦合程度。  
和calladapterFactories一起声明的还有converterFactories，下面就看一下`Converter`的声明。

#### Converter和Converter.Factory
在分析`Converter`时，先看一下上一节还没有分析的一个方法：`ServiceMethod#toResponse()`。贴一下代码：
```Java
R toResponse(ResponseBody body) throws IOException {
  return responseConverter.convert(body);
}
```

只是简单的调用了responseConverter.convert()，将ResponseBody转换为了`R`类型。responseConverter和callAdapter的赋值相同，通过`ServiceMethod.createResponseConverter()`->`Retrofit.responseBodyConverter()`->`Retrofit.nextResponseBodyConverter()`取到的，也是检索所有的Factory，并且让Factory判断类型，看是否能够生成相应的Converter。知道Converter的调用位置了，分析一下`Converter`的作用。

`Converter`:

```Java
public interface Converter<F, T> {
  T convert(F value) throws IOException;

  abstract class Factory {
    public @Nullable Converter<ResponseBody, ?> responseBodyConverter(Type type,
        Annotation[] annotations, Retrofit retrofit) {
      return null;
    }

    public @Nullable Converter<?, RequestBody> requestBodyConverter(Type type, Annotation[] parameterAnnotations, 
    Annotation[] methodAnnotations, Retrofit retrofit) {
      return null;
    }

    public @Nullable Converter<?, String> stringConverter(Type type, Annotation[] annotations,
        Retrofit retrofit) {
      return null;
    }

    protected static Type getParameterUpperBound(int index, ParameterizedType type) {
      return Utils.getParameterUpperBound(index, type);
    }

    protected static Class<?> getRawType(Type type) {
      return Utils.getRawType(type);
    }
  }
}
```

`Converter`接口也同样接受两个泛型，其中只有一个方法`convert()`，也是将`F`类型的value转换为`T`类型。  
然后看一下Factory，Factory的两个主要方法是`reponseBodyConverter`和`requestBodyConverter`，很明显,这两个方法一个是将`ResponseBody`转换为需要的类型（如JsonString，List等等），一个是将传入的对象（比如一个POJO，一个String）转换为转换`RequestBody`的。  
Retrofit最后处理的，就是Request，返回的也是Response，它做的就是提供一个转换接口，供使用者按照需求实现。  
看一下`Conveter.Factory`的实现类，在分析Retrofit的构建时，`Retrofit.Builder`的构造方法中，向converterFactories中加入了`BuiltInConverters`的实例，因此这个类就是`Converter.Factory`的默认实现类。

```Java
final class BuiltInConverters extends Converter.Factory {
  @Override
  public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations,
      Retrofit retrofit) {
    if (type == ResponseBody.class) {
      return Utils.isAnnotationPresent(annotations, Streaming.class)
          ? StreamingResponseBodyConverter.INSTANCE
          : BufferingResponseBodyConverter.INSTANCE;
    }
    if (type == Void.class) {
      return VoidResponseBodyConverter.INSTANCE;
    }
    return null;
  }

  @Override
  public Converter<?, RequestBody> requestBodyConverter(Type type,
      Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) {
    if (RequestBody.class.isAssignableFrom(Utils.getRawType(type))) {
      return RequestBodyConverter.INSTANCE;
    }
    return null;
  }
}
```

实现很简单，如果要转换的类型是`ResponseBody`或是`Void`，则该Factory可以处理，并且返回相应的Converter，同理`requestBodyConverter()`方法也是一样的。  
因此，最开始示例中接口内声明方法的返回值，默认上实际是`Call<ResponseBody>`，这个泛型类型，实际是也就是`Callback`中返回给回调的参数类型，而参数就是请求完成之后，得到的结果。  
最开始的问题3也解决了，`Call`的泛型类型可以改变，只要给Retrofit提供支持这个类型的`Converter`就可以了。  

#### ServiceMethod的构建
之前遇到ServiceMethod的地方，都没有继续深入分析，因为这一块比较复杂，在分析完大致流程，主要几个接口的功能之后，再看这个方法，会更好理解一些。做好准备，这里的代码量可能会比较大。  
那么先从ServiceMethod的构建开始，在3.2节分析`Retrofit.create()`方法中，看到了`loadServiceMethod()`方法，serviceMethod就是在这里构建的，ServiceMethod也是Builder模式构建的，看一下build方法：

```Java
public ServiceMethod build() {
  callAdapter = createCallAdapter();
  responseType = callAdapter.responseType();
  
  responseConverter = createResponseConverter();

  for (Annotation annotation : methodAnnotations) {
    parseMethodAnnotation(annotation);
  }

  int parameterCount = parameterAnnotationsArray.length;
  parameterHandlers = new ParameterHandler<?>[parameterCount];
  for (int p = 0; p < parameterCount; p++) {
    Type parameterType = parameterTypes[p];
    if (Utils.hasUnresolvableType(parameterType)) {
      throw parameterError(p, "Parameter type must not include a type variable or wildcard: %s",
          parameterType);
    }

    Annotation[] parameterAnnotations = parameterAnnotationsArray[p];
    if (parameterAnnotations == null) {
      throw parameterError(p, "No Retrofit annotation found.");
    }

    parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
  }
  /**
   * 删去一些判断代码
   */
  return new ServiceMethod<>(this);
}
```
这段代码删掉了大量的判断和错误提示，把细节放在流程主体上。  
前三行，从Retrofit的Factory中得到callAdapter和resposeConverter，用取得的callAdapter获得声明方法的类型。  
获取传入method的所有Annotation，并且针对每一个annotation进行处理。

##### parseMethodAnnotation()

```Java
private void parseMethodAnnotation(Annotation annotation) {
  if (annotation instanceof DELETE) {
    parseHttpMethodAndPath("DELETE", ((DELETE) annotation).value(), false);
  } else if (annotation instanceof GET) {
    parseHttpMethodAndPath("GET", ((GET) annotation).value(), false);
  } 
  /*
   * 删去了很多代码
   */
}
```
`parseMethodAnnotation()`方法中处理了所有`@Target(METHOD)`的注解，这里看一下`GET`的处理方式。

```Java
private void parseHttpMethodAndPath(String httpMethod, String value, boolean hasBody) {
  this.httpMethod = httpMethod;
  this.hasBody = hasBody;

  if (value.isEmpty()) {
    return;
  }

  // Get the relative URL path and existing query string, if present.
  int question = value.indexOf('?');
  if (question != -1 && question < value.length() - 1) {
    // Ensure the query string does not have any named parameters.
    String queryParams = value.substring(question + 1);
    Matcher queryParamMatcher = PARAM_URL_REGEX.matcher(queryParams);
    if (queryParamMatcher.find()) {
      throw methodError("URL query string \"%s\" must not have replace block. "
          + "For dynamic query parameters use @Query.", queryParams);
    }
  }

  this.relativeUrl = value;
  this.relativeUrlParamNames = parsePathParameters(value);
}
```

保存httpMethod和请求是否有方法体的boolean值为成员变量，注解为空则不处理。  
随后检查输入的url中，query参数中，有没有使用`{}`标记动态参数，如果有则报错，Retrofit推荐使用`@Query`来构造动态query参数的链接。  
然后，将url赋值给relativeUrl，然后通过parsePathParameters方法，将所有被`{}`包裹的字符串都放到relativeUrlParamNames中。  
##### 处理parameterAnnotationsArray

构建了一个和parameterAnnotationsArray长度相同的`ParameterHandler`数组，然后调用`parseParameter()`处理`parameterAnnotations`，顾名思义，这个是最开始创建接口中声明的方法中每个参数的注解。  

```Java
private ParameterHandler<?> parseParameter(
    int p, Type parameterType, Annotation[] annotations) {
  ParameterHandler<?> result = null;
  for (Annotation annotation : annotations) {
    ParameterHandler<?> annotationAction = parseParameterAnnotation(
        p, parameterType, annotations, annotation);
    if (annotationAction == null) {
      continue;
    }
    if (result != null) {
      throw parameterError(p, "Multiple Retrofit annotations found, only one allowed.");
    }
    result = annotationAction;
  }
  if (result == null) {
    throw parameterError(p, "No Retrofit annotation found.");
  }
  return result;
}
```

调用到`parseParameterAnnotation()`方法，这个方法处理了所有`@Traget(PARAMETER)`的注解，代码非常长，接近300行，这里用`@Path`注解走一遍流程，其他注解的处理方法都很类似。    
使用过Retrofit的都知道，在`@GET`注解的参数中，url被`{}`包裹的部分，会被`@Path`注解的参数所替代，具体怎么做的，看一下处理代码：

```Java
if (annotation instanceof Path) {
 /*
  *这里删掉了一些判断和错误处理代码
  */

  Path path = (Path) annotation;
  String name = path.value();
  validatePathName(p, name);

  Converter<?, String> converter = retrofit.stringConverter(type, annotations);
  return new ParameterHandler.Path<>(name, converter, path.encoded());
}
```

获得传入注解的值，然后验证url是否和注解匹配。随后处理转到了`ParameterHandler.Path`中：

```Java
static final class Path<T> extends ParameterHandler<T> {
  private final String name;
  private final Converter<T, String> valueConverter;
  private final boolean encoded;
    
  Path(String name, Converter<T, String> valueConverter, boolean encoded) {
    this.name = checkNotNull(name, "name == null");
    this.valueConverter = valueConverter;
    this.encoded = encoded;
  }
    
  @Override void apply(RequestBuilder builder, @Nullable T value) throws IOException {
    if (value == null) {
      throw new IllegalArgumentException(
        "Path parameter \"" + name + "\" value must not be null.");
    }
    builder.addPathParam(name, valueConverter.convert(value), encoded);
  }
}
```
简单的将value的值，valueConverter和是否需要encode传入，并构造了一个`ParameterHandler.Path`返回。  
获得了一个`ParameterHandler`的实例，那么这个实例在哪使用呢？就是在之前还剩下的一个没有分析的方法`Service.toRequest()`中。  

```Java
Request toRequest(@Nullable Object... args) throws IOException {
  RequestBuilder requestBuilder = new RequestBuilder(httpMethod, baseUrl, relativeUrl, headers,
    contentType, hasBody, isFormEncoded, isMultipart);

  @SuppressWarnings("unchecked") // It is an error to invoke a method with the wrong arg types.
  ParameterHandler<Object>[] handlers = (ParameterHandler<Object>[]) parameterHandlers;

  int argumentCount = args != null ? args.length : 0;
  if (argumentCount != handlers.length) {
    throw new IllegalArgumentException("Argument count (" + argumentCount
      + ") doesn't match expected count (" + handlers.length + ")");
  }
  for (int p = 0; p < argumentCount; p++) {
    handlers[p].apply(requestBuilder, args[p]);
  }
  return requestBuilder.build();
}
```
这里就是构建`okhttp3.Request`的逻辑了，开始分析（先分析了toResponse()再分析toRequest()是不是很奇怪。。）。  
第一行，用了解析注解时获得的参数，构建一个`RequestBuilder`。简单说一下这几个参数的作用
- httpMethod 标记请求的方法
- baseUrl 构造Retrofit时，传入的url，一般request都会以此url作为基础
- relativeUrl 在方法注解中传入的参数，通常会加到baseUrl后面，拼接成一个完整的url
- headers request的header
- contentType 请求体内容的类型
- hasBody 是否含有请求体
- isFormEncode 是否使用了FormUrlEncoded注解
- isMultipart 是否使用了Multipart注解


构建完毕之后，遍历调用`ServiceMethod.parameterHandlers.apply()`，并将获得的`RequestBuilder`实例传入。  
apply方法上面的`ParameterHandler.Path`已经贴出来了，`apply()`的实现非常简单，直接调用`RequestBuilder.addPathParam()`方法。  

```Java
void addPathParam(String name, String value, boolean encoded) {
  if (relativeUrl == null) {
    // The relative URL is cleared when the first query parameter is set.
    throw new AssertionError();
  }
  relativeUrl = relativeUrl.replace("{" + name + "}", canonicalizeForPath(value, encoded));
}
```
可见这里是直接将relativeUrl中的`{text}`字符串，直接替换为参数的值了。  
再举一个`@Query`的例子，这个注解的处理是调用了`RequestBuilder.addQueryParam()`方法：

```Java
 void addQueryParam(String name, @Nullable String value, boolean encoded) {
  if (relativeUrl != null) {
    // Do a one-time combination of the built relative URL and the base URL.
    urlBuilder = baseUrl.newBuilder(relativeUrl);
    if (urlBuilder == null) {
      throw new IllegalArgumentException(
        "Malformed URL. Base: " + baseUrl + ", Relative: " + relativeUrl);
    }
    relativeUrl = null;
  }

  if (encoded) {
    //noinspection ConstantConditions Checked to be non-null by above 'if' block.
    urlBuilder.addEncodedQueryParameter(name, value);
  } else {
    //noinspection ConstantConditions Checked to be non-null by above 'if' block.
    urlBuilder.addQueryParameter(name, value);
  }
}
```
将relativeUrl转换为HttpUrl类，并给调用其`addQueryParameter`方法为url增加query参数。  
就这样，ServiceMethod通过对声明方法的注解处理，生成了一个`okhttp3.Request`类型的实例，通过这个实例就可以进行网络请求了。

### 流程总结
上一张图，当然这张图省略了很多内容，比如线程的切换，Reqeust的构造等等。

![流程总结](/images/Retrofit流程分析.png)

用文字简单叙述一下整个流程：
- 声明网络调用的接口，给方法加上注解等信息
- 调用`Retrofit.create()`方法对接口进行动态代理
   - 针对每一个方法创建`ServiceMethod`
   - `ServiceMethod`解析方法注解和验证参数
   - 创建`OkHttpCall`，通过`CallAdapter`，转换`OkHttpCall`的实例为对应方法的返回类型
- 获取到声明接口实现类的实例之后，调用`Call.enqueue()`（或者`Call.execute()`）
   - `OkHttpCall.enqueue()`中，调用`ServiceMethod.toRequest()`
   - 使用`ParameterHandler`解析注解，并构建`okhttp3.Request`，这里也会调用`Converter.requestBodyConverter()`转换`ConverterFactory`能够处理的类型为`okhttp3.RequestBody`
   - 获取`okhttp3.Request`，接下来就是调用`OkHttpClient`处理网络请求，并得到结果`okhttp3.Response`
   - `OkHttpCall.parseResponse()`调用`Converter.responseBodyConverter()`将`ResponseBody`转换为接口方法中声明的类型，并包装`okhttp3.Response`为`Response<T>`
- 若是异步，则在回调中处理响应结果

这样，一个请求的流程就这样执行完了。源码分析完毕。