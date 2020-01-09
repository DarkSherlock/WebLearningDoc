
参考：[https://www.jianshu.com/p/064d944606a7](https://www.jianshu.com/p/064d944606a7)

先看下入口：
```java
return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
    new InvocationHandler() {
      private final Platform platform = Platform.get();

      @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
          throws Throwable {
        // If the method is a method from Object then defer to normal invocation.
        if (method.getDeclaringClass() == Object.class) {
          return method.invoke(this, args);
        }
        if (platform.isDefaultMethod(method)) {
          return platform.invokeDefaultMethod(method, service, proxy, args);
        }
        ServiceMethod<Object, Object> serviceMethod =
            (ServiceMethod<Object, Object>) loadServiceMethod(method);
        OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
        return serviceMethod.adapt(okHttpCall);
      }
    });
```

用动态代理生成代理对象，然后Service调用方法时，将方法的参数信息注解类型封装成ServiceMethod：

```java
ServiceMethod<Object, Object> serviceMethod =(ServiceMethod<Object, Object>) loadServiceMethod(method);
```

然后将方法调用传进来的实参args 和serviceMethod封装成OkHttpCall

    OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
紧接着 return serviceMethod.adapt(okHttpCall);
进去方法内部看下：
```java
      T adapt(Call<R> call) {
    	return callAdapter.adapt(call);
      }
```
callAdapter 负责将网络请求的返回的Response转换为 Service中定义的方法返回对象

```java
public interface Service {
	@POST("getSongPoetry")
	@FormUrlEncoded
 	Observable<Bean> getCall(@Field("page") int page, @Field("count") int count);
}
```
比如这里会把response转化为Observable<Bean>。
那么这个callAdapter是哪里来的？ServiceMethod是由ServiceMethod.Builder 建造：
```java
public ServiceMethod build() {
      callAdapter = createCallAdapter();
      responseType = callAdapter.responseType();
      if (responseType == Response.class || responseType == okhttp3.Response.class) {
        throw methodError("'"
            + Utils.getRawType(responseType).getName()
            + "' is not a valid response body type. Did you mean ResponseBody?");
      }
      responseConverter = createResponseConverter();

      for (Annotation annotation : methodAnnotations) {
        parseMethodAnnotation(annotation);
      }

      if (httpMethod == null) {
        throw methodError("HTTP method annotation is required (e.g., @GET, @POST, etc.).");
      }

      if (!hasBody) {
        if (isMultipart) {
          throw methodError(
              "Multipart can only be specified on HTTP methods with request body (e.g., @POST).");
        }
        if (isFormEncoded) {
          throw methodError("FormUrlEncoded can only be specified on HTTP methods with "
              + "request body (e.g., @POST).");
        }
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

      if (relativeUrl == null && !gotUrl) {
        throw methodError("Missing either @%s URL or @Url parameter.", httpMethod);
      }
      if (!isFormEncoded && !isMultipart && !hasBody && gotBody) {
        throw methodError("Non-body HTTP method cannot contain @Body.");
      }
      if (isFormEncoded && !gotField) {
        throw methodError("Form-encoded method must contain at least one @Field.");
      }
      if (isMultipart && !gotPart) {
        throw methodError("Multipart method must contain at least one @Part.");
      }

      return new ServiceMethod<>(this);
    }
```
可以看到方法第一行createCallAdapter()返回了callAdapter，里面会将方法的注解和返回类型去reftofit对象中寻找匹配的callAdapter:
  

```java
return (CallAdapter<T, R>) retrofit.callAdapter(returnType, annotations);
```
retrofit中会在一个list遍历如果匹配就返回，没有就抛异常

```java
for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
      CallAdapter<?, ?> adapter = callAdapterFactories.get(i).get(returnType, annotations, this);
      if (adapter != null) {
        return adapter;
      }
    }
```
而这个callAdapterFactories是在retrofit创建时，把builder添加进来的callAdapterFactority都add进去。

先看Retrofit是怎么创建的：

```java
private Retrofit mRetrofit = new Retrofit.Builder()
    .baseUrl("http://api.apiopen.top/")
    .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
    .addConverterFactory(GsonConverterFactory.create())
    .build();
```
看下build方法：

```java
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
      List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
      callAdapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

      // Make a defensive copy of the converters.
      List<Converter.Factory> converterFactories =
          new ArrayList<>(1 + this.converterFactories.size());

      // Add the built-in converter factory first. This prevents overriding its behavior but also
      // ensures correct behavior when using converters that consume all types.
      converterFactories.add(new BuiltInConverters());
      converterFactories.addAll(this.converterFactories);

      return new Retrofit(callFactory, baseUrl, unmodifiableList(converterFactories),
          unmodifiableList(callAdapterFactories), callbackExecutor, validateEagerly);
    }
```
方法首先如果看没有设置callFactory 那么默认用的是okhttpclient。Executor callbackExecutor 这里默认用的是Android Platform的exeuctor，其实就是用主线程的Handler.post（runable）；
List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
这里就把Retrofit.Builder中addCallAdapterFactory()方法添加的CallAdapterFactory都添加进去list，然后

```java
callAdapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));
```
还默认添加了ExecutorCallAdapterFactory，负责将CallEnqueueObservable的请求成功或者失败回调post到主线程去。

 converterFactories.addAll(this.converterFactories);也是将builder.addConverterFactory添加的转换工程add进去。Converter是负责将接口返回的OkHttp3.Response解析成我们想要的对象。比如这里使用的是GsonConverterFactory，会根据service中声明的Observable<Bean>，将OkHttp3.Response解析成Bean对象。

然后回过头来重新看callAdapter.adapt(call);我们这里的例子callAdapter会匹配到RxJava2CallAdapter，那么我们来看RxJava2CallAdapter.adpt();

```java
@Override public Object adapt(Call<R> call) {
    Observable<Response<R>> responseObservable = isAsync
        ? new CallEnqueueObservable<>(call)
        : new CallExecuteObservable<>(call);

    Observable<?> observable;
    if (isResult) {
      observable = new ResultObservable<>(responseObservable);
    } else if (isBody) {
      observable = new BodyObservable<>(responseObservable);
    } else {
      observable = responseObservable;
    }

    if (scheduler != null) {
      observable = observable.subscribeOn(scheduler);
    }

    if (isFlowable) {
      return observable.toFlowable(BackpressureStrategy.LATEST);
    }
    if (isSingle) {
      return observable.singleOrError();
    }
    if (isMaybe) {
      return observable.singleElement();
    }
    if (isCompletable) {
      return observable.ignoreElements();
    }
    return observable;
  }
```

我们选一种看就行，选CallEnqueueObservable，熟悉RxJava都知道，实际方法都会走到subscribeActual（）,直接看这个方法就行。

 

```java
@Override protected void subscribeActual(Observer<? super Response<T>> observer) {
    // Since Call is a one-shot type, clone it for each new observer.
    Call<T> call = originalCall.clone();
    CallCallback<T> callback = new CallCallback<>(call, observer);
    observer.onSubscribe(callback);
    call.enqueue(callback);
  }
```
可以到看到这里是当订阅后，就是直接调用Retrofit.Call # enqueue（）方法，
而这个CallCallback中收到回调就调用rxjava的observer.onNext(response);  observer.onComplete();等方法，然后请求就结束了。
现在要再看细节的话，就继续看内部Retrofit.Call # enqueue（），
而这个call其实就是retrofit动态代理对象方法生成的那个OkHttpCall，所以到其内部看。
整体的代码：

```java
@Override public void enqueue(final Callback<T> callback) {
    checkNotNull(callback, "callback == null");

    okhttp3.Call call;
    Throwable failure;

    synchronized (this) {
      if (executed) throw new IllegalStateException("Already executed.");
      executed = true;

      call = rawCall;
      failure = creationFailure;
      if (call == null && failure == null) {
        try {
          call = rawCall = createRawCall();
        } catch (Throwable t) {
          throwIfFatal(t);
          failure = creationFailure = t;
        }
      }
    }

    if (failure != null) {
      callback.onFailure(this, failure);
      return;
    }

    if (canceled) {
      call.cancel();
    }

    call.enqueue(new okhttp3.Callback() {
      @Override public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse) {
        Response<T> response;
        try {
          response = parseResponse(rawResponse);
        } catch (Throwable e) {
          callFailure(e);
          return;
        }

        try {
          callback.onResponse(OkHttpCall.this, response);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }

      @Override public void onFailure(okhttp3.Call call, IOException e) {
        callFailure(e);
      }

      private void callFailure(Throwable e) {
        try {
          callback.onFailure(OkHttpCall.this, e);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }
    });
  }
```
这里关注这个call，
```java
if (call == null && failure == null) {
        try {
          call = rawCall = createRawCall();
        } catch (Throwable t) {
          throwIfFatal(t);
          failure = creationFailure = t;
        }
  }
```
方法内部先判断okhttp.call是否为空，空的话
 
```java
private okhttp3.Call createRawCall() throws IOException {
    okhttp3.Call call = serviceMethod.toCall(args);
    if (call == null) {
      throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
  }
```
调用serviceMethod.toCall(args)生成okhttp.call

```java
okhttp3.Call toCall(@Nullable Object... args) throws IOException {
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

    return callFactory.newCall(requestBuilder.build());
  }
```
可以看到这里就是把参数和对应的注解取出来给requestBuilder，然后生成request，然后用callFactory（就是一开始默认使用的okhttpclient）生成okhttp.call返回。
然后这下 call有了，直接调用enqueue方法：

```java
call.enqueue(new okhttp3.Callback() {
      @Override public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse) {
        Response<T> response;
        try {
          response = parseResponse(rawResponse);
        } catch (Throwable e) {
          callFailure(e);
          return;
        }

        try {
          callback.onResponse(OkHttpCall.this, response);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }

      @Override public void onFailure(okhttp3.Call call, IOException e) {
        callFailure(e);
      }

      private void callFailure(Throwable e) {
        try {
          callback.onFailure(OkHttpCall.this, e);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }
    });
```
然后在回调中调用parseResponse。

```java
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
这里做了一些状态码的判断。正常是走到

```java
 T body = serviceMethod.toResponse(catchingBody);
 return Response.success(body, rawResponse);
```
而toResponse中：

```java
R toResponse(ResponseBody body) throws IOException {
    return responseConverter.convert(body);
  }
```
这里就是用前面说到的GsonConverterFactory生成的GsonResponseBodyConverter将OkHttp3.ResponseBody转换为Bean。
然后在把结果返回，接口调用结束。

注：这里的例子其实应该返回的是BodyObservable而不是CallEnqueueObservable。
BodyObservable中：才是会将Bean调用onNext（）下发。
@Override public void onNext(Response<R> response) {
      if (response.isSuccessful()) {
        observer.onNext(response.body());
      } else {
        terminated = true;
        Throwable t = new HttpException(response);
        try {
          observer.onError(t);
        } catch (Throwable inner) {
          Exceptions.throwIfFatal(inner);
          RxJavaPlugins.onError(new CompositeException(t, inner));
        }
      }
    }
而CallEnqueueObservable下发的实际是Response< Bean>对象。
