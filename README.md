# 探究Retrofit
## 1. Retrofit的基本使用方式
* **1.1 构建之一：定义接口.**
	<p>这个接口比较特殊，包含请求头,请求方法(@GET\@POST),请求体(参数),响应数据类型</p>
		 public interface IRetrofitService{
			@Header("cookies:123456")
			@GET("one/two")
			Call<ResponseBody> getFromServer();
			
			@POST("three/four")
			Call<ResponseBody> postFromServer(@RequestBody Request body)
		}
* **构建之二：Retrofit 闪亮登场。**
	<p>Retrofit蓄势待发</p>
	 	Retrofit retrofit = new Retrofit.Builder()
                .baseUrl("http://www.studyretrofit.com/") //指定Url,且这个baseUrl必须以 / 来结束
                .client(okHttplClient) //指定你自己的okhttp,如果你不指定的话，在调用build方法的时候，会为你默认的创建一个 okHttpClient
                .addConverterFactory(GsonConverterFactory.create())
                .build();

		public interface IService {
    	@POST("/v1/getSomeData")
    	Call<JsonObject> getSomeData(@Body Map<String,Object> map);
		}
		
		Iservice iService = retrofit.create(IService.class);
		Call<JsonObject> someDataCall = iService.getSomeData(...);	
		//异步的执行请求
        someDataCall.enqueue(new Callback<JsonObject>() {
            @Override
            public void onResponse(Call<JsonObject> call, Response<JsonObject> response) {
                //请求成功
            }

            @Override
            public void onFailure(Call<JsonObject> call, Throwable t) {
				//请求失败
            }
        });
		
## 2. Retrofit的使用带来了什么好处？
* **2.1Retrofit实质上是duiOKhttp的封装，也就是没有okhttp也就没有retrofit，可见最大的亮点在于定义了一个接口，这个接口里面，直接指名了url的后半段，请求方式，设置请求头，请求体，都可以在这里进行定义。也就是okHttp定义请求参数的RequestBody就不必出现了。**
## 3. Retrofit的细节剖析
	//传入这个接口，打开天窗说亮话。深入下去
	Iservice iService = retrofit.create(IService.class);

	public <T> T create(final Class<T> service) {
	//此类必须是接口，另外此类不能继承自其他接口
    Utils.validateServiceInterface(service);
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();
          private final Object[] emptyArgs = new Object[0];

          @Override public @Nullable Object invoke(Object proxy, Method method,
              @Nullable Object[] args) throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
			//loadServiceMethod 获取 ServiceMethod   这个invoke是反射吗？-->显然不是
            return loadServiceMethod(method).invoke(args != null ? args : emptyArgs);
          }
        });
  	}

**ServiceMethod闪亮登场**

 	ServiceMethod<?> loadServiceMethod(Method method) {
	// serviceMethodCache是一个Map 键为 Method 值是 ServiceMethod
    ServiceMethod<?> result = serviceMethodCache.get(method); //是否已经解析过这个方法的注解
    if (result != null) return result; // 取出 serviceMethod 直接开始搞事情

    synchronized (serviceMethodCache) { //加上一把 对象同步锁
      result = serviceMethodCache.get(method);
      if (result == null) { //再次检查
        result = ServiceMethod.parseAnnotations(this, method);//到ServiceMethod 中秘密操作 
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }

	abstract class ServiceMethod<T> { //抽象的类
  	static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
									//传入RequestFactory中进行烹饪,目的是解析当前方法的注解，参数那些的
    RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);
	//经过上一步的RequestFactory的操作，方法的 请求已经解析好了。
    Type returnType = method.getGenericReturnType(); //当前方法的返回Type
    if (Utils.hasUnresolvableType(returnType)) {
      throw methodError(method,
          "Method return type must not include a type variable or wildcard: %s", returnType);
    }
    if (returnType == void.class) { // 意味着你不能在接口中定义一个方法，返回的是void
      throw methodError(method, "Service methods cannot return void.");
    }
	//只见事情并没有这么简单，解析好，请求的那些参数，又解析好返回的类型，ServiceMethod这个抽象类，开始甩锅，丢给了他的小弟HttpServiceMethod，作为ServiceMethod的子类，他当然必须挺身而出。
    return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
 	 }
	// 注意这个ServiceMethod在组后定义了一个抽象方法，invoke,这个方法当然由他的小弟 HttpServiceMethod去实现了。
 	 abstract @Nullable T invoke(Object[] args);
	}

**RequestFactory.java**

	final class RequestFactory {
	  static RequestFactory parseAnnotations(Retrofit retrofit, Method method) {
	    return new Builder(retrofit, method).build();
	  }
	 //一看下面这些属性，刹那明白原来，解析接口中的那些注解，参数就是这里了~~
	  private final Method method;
	  private final HttpUrl baseUrl;
	  final String httpMethod;
	  private final @Nullable String relativeUrl;
	  private final @Nullable Headers headers;
	  private final @Nullable MediaType contentType;
	  private final boolean hasBody;
	  private final boolean isFormEncoded;
	  private final boolean isMultipart;
	  private final ParameterHandler<?>[] parameterHandlers;
	.....
	// 这个create方法，干的漂亮，最终解析了注解，然后生成okHttp的Request.
  	okhttp3.Request create(Object[] args) throws IOException {
    @SuppressWarnings("unchecked") // It is an error to invoke a method with the wrong arg types.
    ParameterHandler<Object>[] handlers = (ParameterHandler<Object>[]) parameterHandlers;

    int argumentCount = args.length;
    if (argumentCount != handlers.length) {
      throw new IllegalArgumentException("Argument count (" + argumentCount
          + ") doesn't match expected count (" + handlers.length + ")");
    }

    RequestBuilder requestBuilder = new RequestBuilder(httpMethod, baseUrl, relativeUrl,
        headers, contentType, hasBody, isFormEncoded, isMultipart);

    if (isKotlinSuspendFunction) {
      // The Continuation is the last parameter and the handlers array contains null at that index.
      argumentCount--;
    }

    List<Object> argumentList = new ArrayList<>(argumentCount);
    for (int p = 0; p < argumentCount; p++) {
      argumentList.add(args[p]);
      handlers[p].apply(requestBuilder, args[p]);
    }

    return requestBuilder.get()
        .tag(Invocation.class, new Invocation(method, argumentList))
        .build();
  }

**HttpServiceMethod.java**

	//生成CallAdapter ？？？？？？？？  留下疑问 -->CallAdapter是什么
	//答： CallAdaoter是一个接口
	CallAdapter<ResponseT, ReturnT> callAdapter =
    createCallAdapter(retrofit, method, adapterType, annotations);
    Type responseType = callAdapter.responseType();
	
    if (responseType == okhttp3.Response.class) {
      throw methodError(method, "'"
          + getRawType(responseType).getName()
          + "' is not a valid response body type. Did you mean ResponseBody?");
    }
    if (responseType == Response.class) {
      throw methodError(method, "Response must include generic type (e.g., Response<String>)");
    }
	// 解析完参数，最终调用了 invoke 实在这里
 	@Override final @Nullable ReturnT invoke(Object[] args) {
					//创建 OkHttpCall,然后返回了回去，那么故事的重点突然来到了OkHttpCall中
    Call<ResponseT> call = new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);
    return adapt(call, args);
 	 }

**OkHttpCall.java**

	// 调用enqueue(CallBack<T> callBack)开始请求
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
          call = rawCall = createRawCall(); //--注意 ,创建了OkHttp的真正的Call
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
 
    call.enqueue(new okhttp3.Callback() {  //真正执行enqueue请求
      @Override public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse) {
        Response<T> response;
        try {
          response = parseResponse(rawResponse);
        } catch (Throwable e) {
          throwIfFatal(e);
          callFailure(e);
          return;
        }

        try {
          callback.onResponse(OkHttpCall.this, response);
        } catch (Throwable t) {
          throwIfFatal(t);
          t.printStackTrace(); // TODO this is not great
        }
      }

      @Override public void onFailure(okhttp3.Call call, IOException e) {
        callFailure(e);
      }

      private void callFailure(Throwable e) {
        try {
          callback.onFailure(OkHttpCall.this, e);
        } catch (Throwable t) {
          throwIfFatal(t);
          t.printStackTrace(); // TODO this is not great
        }
      }
    });
  }

	//createRawCall() 创建okHttp使用的真正的Call
	private okhttp3.Call createRawCall() throws IOException {
    okhttp3.Call call = callFactory.newCall(requestFactory.create(args));
    if (call == null) {
      throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
  }

## 4. 总结
	Retrofit总的来说，使用了一个动态代理，让一个背负着请求方法、参数的 接口方法，经过内部的处理直接和OKhttp暗中交互。请求接口统一定义在一个方法里面，修改Url、参数这种小的需求，能够更加快速、准确的定位。可说是棒极了。