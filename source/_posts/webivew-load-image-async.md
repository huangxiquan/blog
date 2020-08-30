---
layout: layout
title: webview异步加载图片
date: 2017-02-13 09:55:03
tags:
---
# 概述
webview的加载进程虽然是在子线程，但是文字加载和图片加载是在同一线程中，如果图片很大，会严重影响文字的加载效果，影响阅读体验。所以把图片放到另一个线程中加载也是很有必要的。这样用户至少可以先看到文字，而且我们还可以对图片进行相应的处理-压缩，缓存等等。本文粘贴代码都是从真实项目中抠出来的。一些细节需要自己处理。主要讲思路。
<!--more-->
# 异步加载图片
webview作为android的一个加载html的框架。如果我们要自定义图片加载就需要重写接口。我们可以通过重写WebViewClient中的方法来拦截webview请求图片。

```
public WebResourceResponse shouldInterceptRequest(WebView view, String url)
```

我们可以看到方法有两个参数，需要一个WebResourceResponse的返回。我们重写该方法需要区别出图片的URL,然后下载图片（图片压缩，图片缓存）并将图片流封装成WebResourceResponse返回。在这里我封装了一个WebViewImageLoad这样的一个类。图片下载用的okhttp,图片缓存用的是DiskLruCache.下面是代码。
重写WebViewClient中的方法。该方法中主要是判断url是否是图片。（不同业务判断方式不同，大家自行判断）具体加载逻辑在WebViewImageLoad这个类中。


```
     public WebResourceResponse shouldInterceptRequest(WebView view, String url) { 
        if(mImageUrls.contains(url)) {
         return WebViewImageLoad.getInstance().loadImage(url);
     }
     return super.shouldInterceptRequest(view, url);
 }
```
WebViewImageLoad。这个类中主要负责下载图片，压缩图片，缓存图片，返回一个WebResourceResponse的对象，代码如下


```
 public class WebViewImageLoad {
    public static WebViewImageLoad sLoad;
    private OkHttpClient mClient;
    private static DiskLruCache mCache;

    private WebViewImageLoad(){
        mClient = new OkHttpClient();
    }

    public static WebViewImageLoad getInstance() {
        if(sLoad == null) {
            sLoad = new WebViewImageLoad();

        }
        return sLoad;
    }

    public WebResourceResponse loadImage(String url) {
        WebResourceResponse response = null;
        try {
            if(mCache == null) {
                mCache = DiskLruCache.open(new File(BaseApp.getInstance().getCacheDir(),"webviewImage/"),1,1,1024 * 1024 * 200);
            }
            PipedOutputStream out = new PipedOutputStream();
            InputStream in = new PipedInputStream(out);
            String key = EncryptUtils.encryptMD5ToString(url).toLowerCase();
            DiskLruCache.Snapshot snapshot = mCache.get(key);
            if(snapshot != null) {
                in = snapshot.getInputStream(0);

            }else {
                Request request = new Request.Builder().url(url).build();
                Call call = mClient.newCall(request);
                call.enqueue(new Callback() {
                    @Override
                    public void onFailure(Call call, IOException e) {

                    }

                    @Override
                    public void onResponse(Call call, Response response) throws IOException {
                        byte[] originBytes = response.body().bytes();

                        Bitmap bitmap = ImageUtils.bytes2Bitmap(originBytes);
                        Bitmap newBitmap = ImageUtils.compressByScale(bitmap, ScreenUtils.getScreenWidth(), (int) ((ScreenUtils.getScreenWidth() * 1.0 / bitmap.getWidth()) * bitmap.getHeight()));
                        byte[] bytes = null;
                        if (originBytes[0] == (byte) 'G' && originBytes[1] == (byte) 'I' && originBytes[2] == (byte) 'F'){
                            //gif图片
                            bytes = originBytes;
                        }else {
                            bytes = ImageUtils.bitmap2Bytes(newBitmap, Bitmap.CompressFormat.WEBP);
                        }

                        out.write(bytes);
                        out.close();
                        DiskLruCache.Editor editor = mCache.edit(key);
                        OutputStream outputStream = editor.newOutputStream(0);
                        outputStream.write(bytes);
                        editor.commit();
                    }
                });
            }

            response = new WebResourceResponse("image/png","UTF-8",in);
        } catch (IOException e) {
            e.printStackTrace();
        }

        return response;
    }

}
```
# 占位图问题
图片异步加载后，必然是文字先加载完成，然后图片加载完成后重文字中间突然出来，体验不是很好所以需要给img标签加占位图。那么就需要给img设置一个宽高和一个背景图。所以我们要在加载html之前给img标签添加一些属性。这时候引入一个框架Jsoup.该框架可以很方便的解析和修改html。图片的宽高是从图片URL中解析出来的。代码如下：


```
Document doc = Jsoup.parse(data);
        Elements elements = doc.select("img");
        for(Element element : elements) {
            String src = element.attr("src");
            mImageUrls.add(src);
            int height = SizeUtils.px2dp(ImageUtils.getImageScaleHeight(getContext(),src));
            element.attr("width","100%");
            element.attr("height",height + "");
            element.attr("style","background:#d8d8d8");
        }
```

data就是html文本。此处设置的背景是颜色，也可以是图片。
