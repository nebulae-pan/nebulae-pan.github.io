---
title: WebView SSL双向认证
tags: [webview, 网络, SSL]
---

### Android 5.0 （API 21）以上

提供双向认证接口：`WebViewClient#onReceivedClientCertRequest(WebView, ClientCerRequest)`.
其中`ClientCertRequest`提供了双向认证的方法:
`ClientCerRequest#proceed(PrivateKey,X509Certificate)`.

1. 继承WebViewClient，并override方法onReceivedClientCertRequest
2. override方法onReceivedSslError(WebView,SsLErrorHandler,SsLError)，调用handler.proceed()

`ClientCerRequest#proceed(PrivateKey,X509Certificate)`两个参数的获取方法

```Java
PrivateKey privateKey = null;
X509Certificate[] certificates = null;
//读取证书
InputStream certificateFileStream = getClass().getResourceAsStream("fileName");
KeyStore keyStore = KeyStore.getInstance("PKCS12");
String password = "password";
//将证书写入keyStore
keyStore.load(certificateFileStream, password.toCharArray());

Enumeration<String> aliases = keyStore.aliases();
String alias = aliases.nextElement();

Key key = keyStore.getKey(alias, password.toCharArray());
if (key instanceof PrivateKey) {
    privateKey = (PrivateKey) key;
    Certificate cert = keyStore.getCertificate(alias);
    certificates = new X509Certificate[1];
    certificates[0] = (X509Certificate) cert;
}

certificateFileStream.close();
```


原因：SSL可能在证书验证前就产生错误，如果不调用handler.proceed，那么这次请求会失败。[链接](https://stackoverflow.com/questions/35871350/not-getting-callback-for-onreceivedclientcertrequest-in-webview)

### Android 5.0 以下
没有提供能够进行直接双向认证的接口。
- 4.0以下WebViewClient有`onReceivedClientCerRequest`方法，但被标记为@hide；
- 4.0至4.3版本`onReceivedClientCerRequest`被移动至`WebViewClientClassicExt`类中，该类仍为@hide标记。
- 4.4版本将所有webView双向认证相关的代码移除，暂时没有找到办法做4.4版本的双向认证。 

目前有三种妥协的方法可以进行双向认证。[链接](https://stackoverflow.com/questions/15588851/android-webview-with-client-certificate)

1. 编译去掉@hide标记后的类文件，使用得到的Jar。
    缺点：4.0-4.3，每个版本都需要编译，判断版本加载不同jar包，并且jar加入工程会影响到包的大小。4.4仍然无法双向验证。
2. 4.0以上加载证书
    缺点：加载证书后有很多机型的WebView仍然无法验证
3. 通过`WebViewClient#shouldInterceptRequest(WebView,String)`拦截所有webView发出的请求，并使用得到的url，通过能够完成认证的链接去获取数据，然后将结果交给WebView进行渲染。
    缺点：如果webView loadUrl的过程中，含有POST请求，或是需要在请求体携带参数，这种方法都无法取得需要发送的参数，因为低版本能够拦截到的只有url而已。

```Java
//拦截示例
private WebResourceResponse intercept(Uri uri) {
    Request request;
    request = new Request.Builder()
            .url(new URL(uri.toString()))
            .removeHeader("User-Agent")
            .addHeader("User-Agent", mUserAgent)
            .build();
    //能够进行双向认证的链接,具体见下一部分：普通链接双向认证
    Response response = mClient.newCall(request).execute();
    ResponseBody responseBody = response.body();

    if (responseBody == null) {
        throw new Exception();
    }
    InputStream is = responseBody.byteStream();
    String contentType = response.header("content-type");
    String encoding = response.header("content-encoding");
    //获取响应的相应参数

    if (contentType != null) {
        String mimeType = contentType;

        if (contentType.contains(";")) {
            mimeType = contentType.split(";")[0].trim();
        }
        //返回WebResourceResponse交由WebView渲染
        return new WebResourceResponse(mimeType, encoding, is);
    }
}
```


### 普通链接双向认证
为链接设置SSLContext

```Java
SSLContext sslContext = SSLContext.getInstance("TLS");
sslContext.init(keyManager, trustManagers, new SecureRandom());
```

- KeyManager,使用客户端证书
- TrushManager,使用服务端证书
将SSLContext初始化完毕，设置给发出请求的链接，此处以Okhttp为示例

```Java
private OkHttpClient getSSLClient() {
    X509TrustManager manager = prepareSslPinning();
    return new OkHttpClient.Builder()
            .cookieJar(new LoginCookieJar())
            .sslSocketFactory(mSSLContext.getSocketFactory(), manager)
            .addNetworkInterceptor(logging)
            .build();
}
private X509TrustManager prepareSslPinning() throws NoSuchAlgorithmException, KeyManagementException, IOException {
    InputStream[] certificates = new InputStream[]{mContext.getResources().getAssets().open("server.cer")};
    InputStream inputStream = mContext.getResources().getAssets().open("client.p12");

    TrustManager[] trustManagers = prepareTrustManager(certificates);
    KeyManager[] keyManagers = prepareKeyManager(inputStream, "password");

    mSSLContext = SSLContext.getInstance("TLS");
    mSSLContext.init(keyManagers, trustManagers, new SecureRandom());
    return chooseTrustManager(trustManagers);
}
```

- 获取keyManager和trustManager

```Java
//省去了try-catch以及判空代码
private TrustManager[] prepareTrustManager(InputStream... certificates) {
    CertificateFactory certificateFactory = CertificateFactory.getInstance("X.509");
    KeyStore keyStore = KeyStore.getInstance(KeyStore.getDefaultType());
    keyStore.load(null);
    int index = 0;
    for (InputStream certificate : certificates) {
        String certificateAlias = Integer.toString(index++);
        keyStore.setCertificateEntry(certificateAlias, certificateFactory.generateCertificate(certificate));
        try {
            if (certificate != null)
                certificate.close();
        } catch (IOException e) {
            LogUtils.e("Certificate Exception", e.getMessage());
        }
    }
    TrustManagerFactory trustManagerFactory = TrustManagerFactory.
            getInstance(TrustManagerFactory.getDefaultAlgorithm());
    trustManagerFactory.init(keyStore);
    return trustManagerFactory.getTrustManagers();
}

private KeyManager[] prepareKeyManager(InputStream bksFile, String password) {
    KeyStore clientKeyStore = KeyStore.getInstance("PKCS12");
    clientKeyStore.load(bksFile, password.toCharArray());
    KeyManagerFactory keyManagerFactory = KeyManagerFactory.getInstance(KeyManagerFactory.getDefaultAlgorithm());
    keyManagerFactory.init(clientKeyStore, password.toCharArray());
    return keyManagerFactory.getKeyManagers();
}

private X509TrustManager chooseTrustManager(TrustManager[] trustManagers) {
    for (TrustManager trustManager : trustManagers) {
        if (trustManager instanceof X509TrustManager) {
            return (X509TrustManager) trustManager;
        }
    }
    return null;
}
```