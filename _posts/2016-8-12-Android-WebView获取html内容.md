---
layout: post
title: Android-WebView获取html内容
categories: [Android]
description: 通过WebView去获取html的内容
tags: WebView
---
## 前言

有时候我们可能需要做一些模拟操作，比如爬虫，一般我们会直接采用HTTP请求实现扒取网页内容，不过，我们可以使用WebView来实现。

## 如何实现

WebView初始化，我们为其设置参数，为其设置嵌入JavaScript代码的Java对象。
```java
this.getSettings().setJavaScriptEnabled(true);
this.addJavascriptInterface(new InJavaScriptLocalObj(), "java_obj");
this.setWebViewClient(new CustomWebViewClient());
```

在CustomWebViewClient中，当访问的http请求加载完成的时候，加载JavaScript代码，在JavaScript代码中调用我们之前嵌入的java对象。
```java

    /**
     * @author linzewu
     */
    final class CustomWebViewClient extends WebViewClient {
        @Override
        public boolean shouldOverrideUrlLoading(WebView view, String url) {
            view.loadUrl(url);
            return true;
        }
        @Override
        public void onPageStarted(WebView view, String url, Bitmap favicon) {
            super.onPageStarted(view, url, favicon);
        }
        @Override
        public void onPageFinished(WebView view, String url) {
            view.loadUrl("javascript:window.java_obj.getSource('<head>'+" +
                    "document.getElementsByTagName('html')[0].innerHTML+'</head>');");
            super.onPageFinished(view, url);
          }
        @Override
        public void onReceivedError(WebView view, WebResourceRequest request, WebResourceError error) {
            super.onReceivedError(view, request, error);
        }
    }
```
再来看这个嵌入的Java对象

```java
   /**
     * 逻辑处理
     * @author linzewu
     */
    final class InJavaScriptLocalObj {
        @JavascriptInterface
        public void getSource(String html) {
			Log.d("html=", html);
        }
    }
```
这样流程完成，然后再调用loadUrl()访问我们想要抓取html代码的网页链接。

<br/>

## 关于@JavascriptInterface


上面实现了通过WebView获取html代码的流程，不过在上面定义的类我们可能发现了嵌入的Java对象类的方法添加了标注@JavascriptInterface，为什么要这样做呢？

我们可以点击addJavascriptInterface()方法的源码查看，发现，居然有这么多的注释：
```java
    /**
     * Injects the supplied Java object into this WebView. The object is
     * injected into the JavaScript context of the main frame, using the
     * supplied name. This allows the Java object's methods to be
     * accessed from JavaScript. For applications targeted to API
     * level {@link android.os.Build.VERSION_CODES#JELLY_BEAN_MR1}
     * and above, only public methods that are annotated with
     * {@link android.webkit.JavascriptInterface} can be accessed from JavaScript.
     * For applications targeted to API level {@link android.os.Build.VERSION_CODES#JELLY_BEAN} or below,
     * all public methods (including the inherited ones) can be accessed, see the
     * important security note below for implications.
     * <p> Note that injected objects will not
     * appear in JavaScript until the page is next (re)loaded. For example:
     * <pre>
     * class JsObject {
     *    {@literal @}JavascriptInterface
     *    public String toString() { return "injectedObject"; }
     * }
     * webView.addJavascriptInterface(new JsObject(), "injectedObject");
     * webView.loadData("<!DOCTYPE html><title></title>", "text/html", null);
     * webView.loadUrl("javascript:alert(injectedObject.toString())");</pre>
     * <p>
     * <strong>IMPORTANT:</strong>
     * <ul>
     * <li> This method can be used to allow JavaScript to control the host
     * application. This is a powerful feature, but also presents a security
     * risk for apps targeting {@link android.os.Build.VERSION_CODES#JELLY_BEAN} or earlier.
     * Apps that target a version later than {@link android.os.Build.VERSION_CODES#JELLY_BEAN}
     * are still vulnerable if the app runs on a device running Android earlier than 4.2.
     * The most secure way to use this method is to target {@link android.os.Build.VERSION_CODES#JELLY_BEAN_MR1}
     * and to ensure the method is called only when running on Android 4.2 or later.
     * With these older versions, JavaScript could use reflection to access an
     * injected object's public fields. Use of this method in a WebView
     * containing untrusted content could allow an attacker to manipulate the
     * host application in unintended ways, executing Java code with the
     * permissions of the host application. Use extreme care when using this
     * method in a WebView which could contain untrusted content.</li>
     * <li> JavaScript interacts with Java object on a private, background
     * thread of this WebView. Care is therefore required to maintain thread
     * safety.
     * </li>
     * <li> The Java object's fields are not accessible.</li>
     * <li> For applications targeted to API level {@link android.os.Build.VERSION_CODES#LOLLIPOP}
     * and above, methods of injected Java objects are enumerable from
     * JavaScript.</li>
     * </ul>
     *
     * @param object the Java object to inject into this WebView's JavaScript
     *               context. Null values are ignored.
     * @param name the name used to expose the object in JavaScript
     */
    public void addJavascriptInterface(Object object, String name) {
        checkThread();
        mProvider.addJavascriptInterface(object, name);
    }
```

代码注释说明，该方法为WebView注入Java对象，允许JavaScript去调用Java对象。当TargetApi为17或者以上，Java对象中只有标注了{@link android.webkit.JavascriptInterface}的公共方法能够被JavaScript调用。当TargetApi为JELLY_BEAN（api16）或者以下，Java对象所有的公共方法都能被JavaScript调用。
  
注意的安全事项：
 1）这个方法能被用于允许JavaScript去控制主应用程序。这是一个强大的功能，而且也造成了{@link android.os.Build
 .VERSION_CODES＃JELLY_BEAN}或更早版本的安全风险。TargetApi大于等17的时候也会有风险，因为应用有可能运行于4.2之前的设备。最安全可靠的方法是设置TargetApi为JELLY_BEAN_MR1或以上，同时确保该方法只有在运行于4.2以及以上版本的设备时才被调用。而在这些老版本中，JavaScript可以通过反射访问注入对象的公共变量，这样的方法会导致一些恶意攻击者去操纵我们的主机，因此，当我们的WebView访问一些可能不被信任的内容的时候，需要非常小心。
  
 2）JavaScript和Java对象的交互运行在WebView内部的后台线程，
  
 3）注入的Java对象的全局变量是不可访问的。
   
 4）注意，直到WebView再次load的时候，注入的Java对象才会出现在JavaScript代码中。因此，我们需要在http请求加载完成之后，再加载我们自己写的JavaScript代码，如下：

```java
     class JsObject {
        {@literal @}JavascriptInterface
        public String toString() { return "injectedObject"; }
     }
     webView.addJavascriptInterface(new JsObject(), "injectedObject");
     webView.loadData("<!DOCTYPE html><title></title>", "text/html", null);
     webView.loadUrl("javascript:alert(injectedObject.toString())");
```