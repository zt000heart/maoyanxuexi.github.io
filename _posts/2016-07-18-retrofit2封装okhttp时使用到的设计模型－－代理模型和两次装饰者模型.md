---
layout: post  
title:  retrofit2封装okhttp时使用到的设计模型
categories: [blog ]  
tags: [Tech, ]     
---
## retrofit2封装okhttp时使用到的设计模型－－代理模型和两次装饰者模型
retrofit2和okhttp的具体使用网上有很多，这里不具体阐述，只对retrofit2封装okhttp时使用到的几个设计模型学习一下。  
****
### okhttp基础知识
看下http最基本的使用方式：
![15:24:51.jpg](http://ww4.sinaimg.cn/large/006tNbRwjw1f5tgswca4fj30k40d2gmx.jpg)
先创建一个request，并利用该request，使用okhttpclient生成一个call。
call的enqueue会降该网络请求加入队列，进行数据的异步请求。我们只需要传入一个实现了的callback回调，即可对异步请求的结果进行处理。**问题1: 注意onResponse()方法也是在异步线程进行的，所以需要一个huandler调用post（runnable）方法返回到ui线程进行ui处理。**  
### retrofit2对该问题的封装
要了解retrofit2对该问题的封装，我们需要了解retrofit的大致工作情况：
#### retrofit2基础知识
我们需要先定义一个接口对象，演示一个最简单的get请求，接口如下所示：：

![15:44:27.jpg](http://ww3.sinaimg.cn/large/006tNbRwjw1f5thd6eht7j30ah0323yj.jpg)

可以看到有一个getUsers(）方法，通过@GET注解标识为get请求，@GET中所填写的这个“users”值和baseUrl组成完整的路径，baseUrl在构造retrofit对象时给出。
然后我们需要入如下步骤：
![15:42:50.jpg](http://ww1.sinaimg.cn/large/006tNbRwjw1f5thbhncqbj30ma0bfabg.jpg)
#### 代理模式
通过

	IUserBiz userBiz = retrofit.create(IUserBiz.class);
来创建该接口的一个代理，如下面代码所示：
![14:57:13.jpg](http://ww2.sinaimg.cn/large/006tNbRwjw1f5ulmbyy3oj30mb0813zm.jpg)
然后当调用该接口代理的方法时，会调用invoke的那三条语句。下面逐句分析这三条语句。
先看下![15:02:49.jpg](http://ww4.sinaimg.cn/large/006tNbRwjw1f5uls5pdznj30df00n748.jpg)
这个语句为生成servicemethod实例，具体为servicemethod的build（）方法所做的事情：

	//Retrofit class
	ServiceMethod loadServiceMethod(Method method) {
	    ServiceMethod result;
	    synchronized (serviceMethodCache) {
	      result = serviceMethodCache.get(method);
	      if (result == null) {
	        result = new ServiceMethod.Builder(this, method).build();
	        serviceMethodCache.put(method, result);
	      }
	    }
	    return result;
	  }
	
	//ServiceMethod
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
	
	      return new ServiceMethod<>(this);
	    }

我们看到在生成servicemethod的过程中，会初始化calladapter和responseConverter两个变量，这两个变量的初始化的实例是之前在生成retrofit实例时，提供给它的adapter工厂和converter工厂进行生产的，如果adapter工厂没有提供，那么就会采用默认的adapter工厂，如代码：
![15:16:04.jpg](http://ww2.sinaimg.cn/large/006tNbRwjw1f5um5yby42j30lp0gf0vd.jpg)
我们看到在retrofit的build中，默认会使用defaultCallAdapterFactory，同时callfactroy也会采用默认的OkHttpClient。如果不采用默认的，我们可以传入自己的factroy进行替换。这里使用到了**工厂模式**。converterFactory并没有传入默认的值，所以需要自己传入。
#### 装饰者模式
回到servicemethod的build（）方法中，除了初始化calladapter和responseConverter两个变量，还会采用反射等方式得到生成request的一切信息。这时候并没有生成对应的request。
然后是执行![15:28:33.jpg](http://ww1.sinaimg.cn/large/006tNbRwjw1f5umixrzcsj30en00nq2v.jpg)  
即生成retrofit框架中的okhttpcall。这里使用了装饰者模型，该okhttpcall继承了okhttp框架的那个call。然后重写了enqueue方法。看下该方法代码：

```
 @Override
public void enqueue(final Callback<T> callback)
{
    okhttp3.Call call;
    Throwable failure;

    synchronized (this)
    {
        if (executed) throw new IllegalStateException("Already executed.");
        executed = true;

        try
        {
            call = rawCall = createRawCall();
        } catch (Throwable t)
        {
            failure = creationFailure = t;
        }
    }

    if (failure != null)
    {
        callback.onFailure(this, failure);
        return;
    }

    if (canceled)
    {
        call.cancel();
    }

    call.enqueue(new okhttp3.Callback()
    {
        @Override
        public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse)
                throws IOException
        {
            Response<T> response;
            try
            {
                response = parseResponse(rawResponse);
            } catch (Throwable e)
            {
                callFailure(e);
                return;
            }
            callSuccess(response);
        }

        @Override
        public void onFailure(okhttp3.Call call, IOException e)
        {
            try
            {
                callback.onFailure(OkHttpCall.this, e);
            } catch (Throwable t)
            {
                t.printStackTrace();
            }
        }

        private void callFailure(Throwable e)
        {
            try
            {
                callback.onFailure(OkHttpCall.this, e);
            } catch (Throwable t)
            {
                t.printStackTrace();
            }
        }

        private void callSuccess(Response<T> response)
        {
            try
            {
                callback.onResponse(OkHttpCall.this, response);
            } catch (Throwable t)
            {
                t.printStackTrace();
            }
        }
    });
}
```

会发现在调用okhttpcall.enqueue时，会执行createRawCall()方法，即：
```
private okhttp3.Call createRawCall() throws IOException
{
    Request request = serviceMethod.toRequest(args);
    okhttp3.Call call = serviceMethod.callFactory.newCall(request);
    if (call == null)
    {
        throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
}
```
即此时会将之前servicemethod的到的信息生成request，并调用callfactory生成call，这个call就是okhttp框架的那个call。所以在生成okhttpcall实例时，只是传入了servicemethod参数等，并没有生成request和okhttp框架的的那个call，在调用其enqueue时，才生成的，并调用了okhttp框架的call的enqueue方法。不过应该这里才用了装饰者模式，外部并不会看到不同。那采用装饰者模式对原call进行的封装的目的是什么呢？我们看到okhttpcall的enqueue方法中：
![15:54:33.jpg](http://ww1.sinaimg.cn/large/006tNbRwjw1f5un9zpklvj30jd0fjmy3.jpg)  
其中parseResponse（）方法为
![15:55:34.jpg](http://ww3.sinaimg.cn/large/006tNbRwjw1f5unb1jga4j30md0473z5.jpg)  
serviceMethod.toResponse（）方法为
![15:56:57.jpg](http://ww3.sinaimg.cn/large/006tNbRwjw1f5unchhe14j30di02iglq.jpg)
所以知道了，这里使用装饰者模式对原call进行的封装，主要是使用之前传入convertfatory生成的responseConverter对异步加载的到的数据进行转型。所以我们传入的convertfatory生成的convert实例需要实现convert方法。
下面看下invoke中的第三句

	serviceMethod.callAdapter.adapt(okHttpCall);
	
如果在生成retrofit实例时，传入了adapterfactory，那么这里的serviceMethod.callAdapter就是自己传入的factory生成的calladapter。如果没有，那么就使用默认的calladapter。如果想传入adapterfactory时，我们需要注意fatory生成的adapter需要实现两个方法，首先因为ServiceMethod进行build（）时，执行了

	responseType = callAdapter.responseType();
	
所以需要实现responseType()方法来的到responseType，还要实现的是这里的adapt（）方法。因为框架中生成calladapter是调用的Factory.get(returntype，..)生成的，所以我们要生成的adpterfactory继承CallAdapter.Factory，实现其get方法，然后在内部生成的adapter中实现responsetype和adapter（）方法即可。
我们看下框架默认的ExecutorCallAdapterFactory：

	final class ExecutorCallAdapterFactory extends CallAdapter.Factory {
	  final Executor callbackExecutor;
	
	  ExecutorCallAdapterFactory(Executor callbackExecutor) {
	    this.callbackExecutor = callbackExecutor;
	  }
	
	  @Override
	  public CallAdapter<Call<?>> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
	    if (getRawType(returnType) != Call.class) {
	      return null;
	    }
	    final Type responseType = Utils.getCallResponseType(returnType);
	    return new CallAdapter<Call<?>>() {
	      @Override public Type responseType() {
	        return responseType;
	      }
	
	      @Override public <R> Call<R> adapt(Call<R> call) {
	        return new ExecutorCallbackCall<>(callbackExecutor, call);
	      }
	    };
	  }
  

#### 二次封装
默认的calladapter的adapter（）方法是生成了ExecutorCallbackCall。看看它的作用。和上一个装饰者模式一样，ExecutorCallbackCall对okhttpcall的封装主要也是继承那个call，并重写了enqueue方法：
该ExecutorCallbackCall的成员变量为：  
![16:15:28.jpg](http://ww2.sinaimg.cn/large/006tNbRwjw1f5ti9g44bqj307t0173yg.jpg)  
其中变量delegate是okhttpcall。  
我们看下重写的enqueue方法，我们发现调用了delegate中的enqueue方法，也就是说这里使用了**装饰者模式**，在delegate中的enqueue方法传入实现了“huandler调用post（runnable）方法返回到ui线程进行ui处理”逻辑的callback。
![16:30:28.jpg](http://ww3.sinaimg.cn/large/006tNbRwjw1f5tip1vfzkj30ml0gbgo3.jpg)
如果是Android平台，callbackExecutor为一个Executor对象，并且利用Looper.getMainLooper()实例化一个handler对象，在Executor内部通过handler.post(runnable)来执行callbackExecutor.execute（）。这样一来，ExecutorCallbackCall的作用就很明显了：**使用装饰者模式将okhttp中的那个call的异步回调转发至UI线程**，因此我们调用ExecutorCallbackCall.enqueue()传入的callback，只需要写ui的逻辑即可。这样也就解决了我们开头的那个问题。


 



