---
layout: post
title: Android下载功能与进度监听
tags: Android   
---

## 下载功能

### 导入依赖和获取权限



> 安卓有个很好用的网络连接的库，叫okhttp3。当然okhttp3还有其他已经封装好的库，不过我们现在只用原生的okhttp3来实现我们的需求。

有关okhttp3的使用可以看这里  [Okhttp3的基本使用](./okhttp3)  ，这里就不在重复赘述了。

**首先导入库**

~~~java
implementation 'com.squareup.okhttp3:okhttp:3.10.0'
~~~

**然后在`AndroidManifest.xml` 中申请权限：**

~~~xml
	<uses-permission android:name="android.permission.INTERNET"/>	//网络权限
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>	//读取文件权限
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>	//写入文件权限
~~~

如果你是安卓6.0及以上的系统，还需要在代码中手动获取，详细点这里 [安卓6.0动态获取权限举例](./android6.0)

---



### 下载功能的实现

**以下代码并不完整，只有核心功能**

实现下载功能之前我们先来了解一些知识：



> 1、我们创建OkHttpClient、Request是为了给访问网络设置属性（包括网络连接地址、超时时间等），而访问网络的最终方法是下面这个：



~~~java
call.enqueue(new callback(){
         	@Override
            public void onFailure(Call call, IOException e) {
                DebugLog.i("联网失败调用")；
            }
            @Override
            public void onResponse(Call call, Response response) {
                 DebugLog.i("联网成功调用")；
		} )
~~~

> 2、如果联网成功，我们就能通过`InputStream is = response.body().byteStream(); `方法，把联网获得的数据放入一个流，然后通过循环，将读取到的数据依次写入本地。

#### **核心代码**

```java
//核心代码，用来理解原理，实际使用可能要更加复杂

public void download(){

     //调用okhttp3的方法完成网络连接，这里使用的是异步get请求
	OkHttpClient httpClient = new OkHttpClient;
    Request request = new Request.Builder().url(url).build();
    Call call = httpClient.newCall(request);

     //开始连接
    call.enqueue(new Callback() {

         	@Override
            public void onFailure(Call call, IOException e) {
                DebugLog.i("联网失败调用")；
            }
            @Override
            public void onResponse(Call call, Response response) {
                 DebugLog.i("联网成功调用")；

                  //在这里可以操作数据了，下载功能其实很简单
                   InputStream is = null;
                   FileOutputStream fos = null;
                   BufferedOutputStream bos=null ;
                   byte [] buf = new byte[1024*64];
                   int leg ;
                   File saveFile = new File(getFile,fileName);

                   //把获得的数据放入字节流,从网络取得的数据就放在这里
                   is = response.body().byteStream();

                   fos = new FileOutputStream(saveFile);   //为输出流设置路径(saveFile)
                   bos = new BufferedOutputStream(fos);    //为输出流添加缓冲流（加速）
                   long total = response.body().contentLength(); //获得文件长度
                   long sum = 0;

                    //使用一个循环读取并把数据写入本地
               while ((leg = is.read(buf))!=-1){
                    bos.write(buf,0,leg);
                    sum +=leg;
                    progress = (int) (sum * 1.0f / total * 100);//进度
                }
				   DebugLog.i("循环完成，下载结束");
                    fos.flush();         
               		//关闭流的操作我就不写了
    		}
        }
 }                 
```

---

#### 创建文件目录

~~~java
 private final String SAVE_PATH = "/Download/";  //自定义下载目录
/**
     * 判断并创建文件目录，这个方法会返回一个包含下载目录的File对象
     * @return file目录对象
     */
    private File getFile (){
        //获得sd卡路径
        String savePath = Environment.getExternalStorageDirectory().getPath()+SAVE_PATH;
        File file= new File(savePath);
        //判断路径是否存在
        if (file.exists()) {
            if (file.isDirectory()) {
                DebugLog.i("文件路径已经存在");
            } else {
                DebugLog.i("同一名称文件存在，不能创建dir");
            }
        } else {
            DebugLog.i("文件路径不存在，正在创建。。");
            file.mkdir();
        }
        return file;
    }
~~~



---

#### 增加抽象方法，使代码更灵活

> 我们不可能每次联网都写这么一大串代码，所以将代码封装成一个方法是很有必要的。但是每次下载失败或者下载完成怎么做却不能写死，增加虚拟方法就很有必要了。

---



~~~java
 	/**
     * 下载完成调用
     */
    abstract void downDone();

    /**
     * 下载出错调用
     */
    abstract void downFailed();

    /**
     * 下载进度
     * @param progress 进度0-100
     */
    abstract void getProgress(int progress) ;


//现在我们有三个抽象方法，每次调用方法都需要指明这三个方法该怎么做
//联网请求失败
  	        @Override
            public void onFailure(Call call, IOException e) {
                DebugLog.i("联网失败调用")；
                    downFailed(); //下载失败方法放这里
            }

 //循环写入
 			while ((leg = is.read(buf))!=-1){
                    bos.write(buf,0,leg);
                    sum +=leg;
                    progress = (int) (sum * 1.0f / total * 100);//进度
                	getProgress(progress); //下载进度放这里
                }
//循环完成
			downDone()；//循环结束代表下载完成。
~~~



---

### 注意事项：

1. `call.enqueue()` 方法发起的是异步请求，然而在java中，非主线程是不能操作UI的。我们通常使用Handle

   ~~~java
   //在异步线程中，我们可以使用这个方法发送信息
   		Message message = new Message();
                    message.what = 1;
                    mHandler.sendMessage(message);
   ~~~

   ~~~java
   //然后再主线程中，创建Handler接收信息，然后就大胆操作UI吧
         mHandler = new Handler(){
               @Override
               public void handleMessage(Message msg) {
                   super.handleMessage(msg);
                   switch (msg.what){
                       case 0:
                        //Ui操作或其他操作
                           break;
                       case 1:
                        //Ui操作或其他操作
                           break;
                           ....
                   }
               }
     };
   ~~~

   2.  推荐让 OkHttpClient 保持单例，用同一个 OkHttpClient 实例来执行你的所有请求；

      因为每一个 OkHttpClient 实例都拥有自己的连接池和线程池，重用这些资源可以减少延时和节省资源；

      可以统一管理。
