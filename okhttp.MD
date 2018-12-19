#### okhttp

- 基本流程 （get） (先不管同步还是异步的流程)

  - 调用`RealCall`的`execut()`

    ```java
    @Override protected void execute() {
          boolean signalledCallback = false;
          try {
              // 这里是请求发生server和处理reponse的地方
            Response response = getResponseWithInterceptorChain();
            if (retryAndFollowUpInterceptor.isCanceled()) {
              signalledCallback = true;
              responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
            } else {
              signalledCallback = true;
              responseCallback.onResponse(RealCall.this, response);
            }
          } catch (IOException e) {
          }
        }
    }
    //======================================================
     /**
       * 真正请求的地方
       */
      Response getResponseWithInterceptorChain() throws IOException {
        // Build a full stack of interceptors.
        List<Interceptor> interceptors = new ArrayList<>();
        interceptors.addAll(client.interceptors());
        interceptors.add(retryAndFollowUpInterceptor);
        interceptors.add(new BridgeInterceptor(client.cookieJar()));
        interceptors.add(new CacheInterceptor(client.internalCache()));
        interceptors.add(new ConnectInterceptor(client));
        if (!forWebSocket) {
          interceptors.addAll(client.networkInterceptors());
        }
        interceptors.add(new CallServerInterceptor(forWebSocket));

        Interceptor.Chain chain = new RealInterceptorChain(
            interceptors, null, null, null, 0, originalRequest);
        return chain.proceed(originalRequest);
      }
    ```


- `RealInterceptorChain proceed()` 接下来就是进入链式调用的过程，盗用网上的图

  ![fe](https://segmentfault.com/img/remote/1460000012656609?w=171&h=648)

- 我们先从连接开始 `ConnectInterceptor`

  ````java
  // HTTP是基于tcp的 java的TCP socket实现

  package okhttp3.internal.connection;

  /** Opens a connection to the target server and proceeds to the next interceptor. */
  public final class ConnectInterceptor implements Interceptor {
    public final OkHttpClient client;

    public ConnectInterceptor(OkHttpClient client) {
      this.client = client;
    }

    @Override public Response intercept(Chain chain) throws IOException {
      RealInterceptorChain realChain = (RealInterceptorChain) chain;
      Request request = realChain.request();
      StreamAllocation streamAllocation = realChain.streamAllocation();

      // We need the network to satisfy this request. Possibly for validating a conditional GET.
      boolean doExtensiveHealthChecks = !request.method().equals("GET");
      HttpCodec httpCodec = streamAllocation.newStream(client, doExtensiveHealthChecks);
      RealConnection connection = streamAllocation.connection();

      return realChain.proceed(request, streamAllocation, httpCodec, connection);
    }
  }
  // 
  //HttpCodec httpCodec = streamAllocation.newStream(client, doExtensiveHealthChecks);
    ///  RealConnection connection = streamAllocation.connection();

  /**
    

  **/
   // Plattform
   connectSocket(Socket socket ,InetSocketAddress address){
      socket
    public void connectSocket(Socket socket, InetSocketAddress address,
        int connectTimeout) throws IOException {
      socket.connect(address, connectTimeout);
    }
  }



  ````

  ​


- e

- e

- ​

  ​