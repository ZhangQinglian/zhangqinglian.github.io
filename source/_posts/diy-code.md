---
title: 聊聊给DiyCode写第三方App的事
date: 2017-6-12 19:11:44
tags: android
cover: https://blog-1256162814.cos.ap-nanjing.myqcloud.com/normal/2702499-6b41628dadbf72b1.webp
---

首先介绍一下`DiyCode`，它的地址是[https://www.diycode.cc/](https://www.diycode.cc/)，是一个**致力于构建开发工程师高端交流分享社区。**它的后台 API 是开放出来的，恰好有段时间我也想用 Kotlin 写一个 App 练练手，所以就有了接下来的事。
<!-- more -->
## 这事我到底行不行？

在上家公司，Leader 总挂在嘴边的一句话是“是男人就别说不行”。话虽如此，但做一件事之前最好还是有一个自我评估，在给 DiyCode 写 App 这件事上需要评估的就是我所掌握的能否将所有需求都实现？但再一想，现在社区类的 App 比比皆是，技术上应该没啥问题，不懂就 Google 呗。

## 我能做啥？

我能做啥？这个还需要看 API 给我们提供了哪些接口，API 地址如下 [https://www.diycode.cc/api](https://www.diycode.cc/api)。大体能做的我整理了下：

- 用户登录
- 话题的创建、查询、点赞、收藏、关注、回复
- 通知的查询、读取
- 对用户的相关操作如：关注用户、查看我关注的用户和被关注的用户等
- 项目、News的创建、查询、回复等

可以说绝大部分功能对应的 API 都有了，这时候心里就要有点数了，我的 App 要去实现哪些功能

## 我想做啥？

在知道了 API 给的接口后，就需要选定一些在 APP 中需要实现的功能，以下是我的选择：

- 登录、退出
- 主题的查看、回复、收藏、点赞和关注
- 查看用户个人资料
- 查看通知
- 个人中心、我关注的人、关注我的人、我收藏的文章

## 步步为营，有坑填坑

### 私有信息对公屏蔽

由于用户的登录需要 client\_id 和 client\_secret来申请token，如果把这两个信息直接放入源码中，那会引来熊孩子的搞破坏，所以这里选择一个比较大众的放发，就是将这两个信息放在 local.properties 文件中，.gitignore 通常会忽略这个文件。接着在 build.gradle 的 defaultConfig 块中加入如下配置：

``` groovy
buildConfigField "String","CLIENT_ID","\"" + properties.get("client_id","null") + "\""
buildConfigField "String","CLIENT_SECRET","\"" + properties.get("client_secret","null") + "\""
```

然后就可以在项目中使用 BuildConfig 这个动态生成的类来访问这两个变量了，不用担心提交 Github 的时候会把一些私有信息提交上去。

### UI 该如何设计？

作为一名 Android 程序员，如果想自己画交互设计出 UI 设计文档，那此人也算神人了。根据我的经验，作为一名程序员在 UI 设计上别太有自己的想法，80%肯定都是不符合设计规范的。所以呢，还是要参考一些其他设计规范。好在 Google 自 Android 5.0 就推出了 Material Design 的设计规范并给出了相应的 Android 组件，我们大可从简，遵循少即是多的原则来设计 App 的 UI。

我个人比较欣赏 Bilibili 客户端和知乎等客户端，打开 App，浓浓的 Material Design 设计味道那是深得我心。不废话，说白了就是程序猿好好写代码，UI 参考一些现成的规范即可。

### App 架构

此处说架构有些装逼了，其实就是怎么分层。目前流行的分层方法无外乎那几种，只要挑选自己觉得可以的就行。在这个 App 中我的选择是 MVP。

### 开源框架的选择

下面我简单列举一下在这个 App 中用到的开源开源框架：

- [CircleImageView](https://github.com/hdodenhof/CircleImageView) : 圆形 ImageView
- [Picasso](https://github.com/square/picasso) : 图片下载及缓存
- [OkHttp](https://github.com/square/okhttp) : Http Client
- [MarkdownView](https://github.com/tiagohm/MarkdownView) : 用于显示 MarkDown 的视图
- [MaterialTextField](https://github.com/florent37/MaterialTextField) : 登录用的输入框
- [PhotoView](https://github.com/chrisbanes/PhotoView) : 显示图片，可手势放大缩小
- [Dexter](https://github.com/Karumi/Dexter) : Runtime Permission 相关
- [Retrofit](https://github.com/square/retrofit) : Http Client

大概就是这些个，如有新的后续补充。

### 其他技术

这个 App 代码部分用 Kotlin 完成，视图的数据填充使用 Databinding 。

### 坑

#### 1.
由于使用了 Kotlin，直接使用 Databinding 会出错，需要在 build.gradle 中添加依赖：

```groovy
kapt "com.android.databinding:compiler:2.3.3"
```

以及插件：

```groovy
apply plugin: 'kotlin-kapt'
```

只有这样 Databinding 才能正常工作。

#### 2.
在使用 Retrofit 和 converter-gson 配合获取数据时，对应的实体类应该定义成如下样子：

```kotlin
data class Token
(
        @SerializedName("access_token") val accessToken: String,
        @SerializedName("token_type") val tokenType: String,
        @SerializedName("expires_in") val expiresIn: String,
        @SerializedName("refresh_token") val refreshToken: String,
        @SerializedName("created_at") val createdAt: String
)
```

使用 kotlin 的数据类保存数据应该是十分明智的选择，应为就只凭它重写的 toString 方法来输出所有字段就大大降低了我在调试时候的工作量。

#### 3.
由于此 App 涉及到用户登录的问题，那就肯定会涉及到请求中添加 token 信息的问题，这里应为用的是 Retrofit ，由于它底下用的也是 Okhttp，所以就自然可以选择 OkHttp 的拦截器 Interceptor。

```kotlin
var interceptor: Interceptor = object : Interceptor {
    override fun intercept(chain: Interceptor.Chain?): Response {
        var originRequest = chain!!.request()
        if (originRequest.url().toString().contains(DiyCodeContract.kOAuthUrl)) {
              return chain.proceed(originRequest)
        }

        if (originRequest.headers()["Authorization"] != null) {
            return chain.proceed(originRequest)
        }

        if (mCallback!!.getToken() == null || mCallback!!.getToken()?.length == 0) {
            return chain.proceed(originRequest)
        }

        var newRequest = originRequest.
                newBuilder().
                addHeader("Authorization", "Bearer " + mCallback!!.getToken()).
                build()
        return chain.proceed(newRequest)
    }

}

var okHttpClient: OkHttpClient = OkHttpClient.Builder().addInterceptor(interceptor).build()

val retrofit = Retrofit.Builder()
        .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
        .addConverterFactory(GsonConverterFactory.create())
        .baseUrl(DiyCodeContract.kDiyCodeApi)
        .client(okHttpClient)
        .build()
```

#### 4.
Markdown 正文中的图片，连接的点击问题，这个问题其实比较简单，应为在使用的开源框架 MarkdownView 中就提供了如下接口：

```kotlin
    public interface OnElementListener {
        void onButtonTap(String var1);

        void onCodeTap(String var1, String var2);

        void onHeadingTap(int var1, String var2);

        void onImageTap(String var1, int var2, int var3);

        void onLinkTap(String var1, String var2);

        void onKeystrokeTap(String var1);

        void onMarkTap(String var1);
    }
```

图片的单独显示，连接的跳转都可以在此处做处理。

#### 5.
这里有个略微复杂的问题，由于在话题详情页下方会有评论列表，返回的评论数据有两种格式，markdown 和 html，本来我想有 markdown 不就足够了吗？评论同样采用 MarkdownView ，分分钟搞定。但万万没想到，这个 MarkdownView 是继承自 WebView，试想一下，一个列表里全是 WebView 在那滑动，界面会卡成啥样。
所以只能退而求其次选择使用 TextView 来显示 html。此方法本来也不难，一句话就搞定了，

```kotlin
binding.markdownView.text = Html.fromHtml(topicReply.bodyHtml)
```
但问题来了，有些评论里带链接，有些是@他人的，有些是带图片的，这里的三个元素都需要做处理。链接如果是指向某一个话题的，应该直接在应用内跳转到该话题而不是用浏览器打开对应页面。点@他人的文字，应该跳转到被@人的个人资料页。点击图片可以进入图片查看页，进行方法和缩小。

这里的前两个问题都可以使用 ClickableSpan 进行处理，而 TextView 显示图片本身就是一个问题，上面的 Html.fromHtml 其实还提供了一个接口：

```kotlin
 public static Spanned fromHtml(String source, int flags, Html.ImageGetter imageGetter, Html.TagHandler tagHandler) 
```

注意这里的 ImageGetter，使用它就能 html 中的 <img> 标签进行处理。

```kotlin
   public interface ImageGetter {
        Drawable getDrawable(String var1);
    }
```

但这个接口怎么看也是一个同步的方法，而加载网络图片大家都知道这是一个异步的操作，所以我们还要做一下进一步的继承和封装处理：

```kotlin
class URLDrawable : BitmapDrawable() {
    var drawable: Drawable? = null

    override fun draw(canvas: Canvas) {
        if (drawable != null) {
            drawable!!.draw(canvas)
        }
    }
}

class URLImageParser(internal var container: View, internal var c: Context) : ImageGetter {

    override fun getDrawable(source: String): Drawable {
        val urlDrawable = URLDrawable()

        val asyncTask = ImageGetterAsyncTask(urlDrawable)

        asyncTask.execute(source)

        return urlDrawable
    }

    inner class ImageGetterAsyncTask(internal var urlDrawable: URLDrawable) : AsyncTask<String, Void, Drawable>() {

        override fun doInBackground(vararg params: String): Drawable? {
            val source = params[0]
            return fetchDrawable(source)
        }

        override fun onPostExecute(result: Drawable?) {
            if(result == null) return
            urlDrawable.setBounds(0, 0, 0 + result.intrinsicWidth, 0 + result.intrinsicHeight)

            urlDrawable.drawable = result

            val textview = this@URLImageParser.container as TextView
            textview.text = textview.text
        }

        fun fetchDrawable(urlString: String): Drawable? {
            try {
                val `is` = fetch(urlString)
                val drawable = Drawable.createFromStream(`is`, "src")
                drawable.setBounds(0, 0, 0 + drawable.intrinsicWidth, 0 + drawable.intrinsicHeight)
                return drawable
            } catch (e: Exception) {
                return null
            }

        }

        @Throws(MalformedURLException::class, IOException::class)
        private fun fetch(urlString: String): InputStream {
            val url = URL(urlString)
            val connection = url.openConnection()
            val inputStream = BufferedInputStream(connection.getInputStream())
            return inputStream
        }
    }
}
```
这里的思路比较清晰，首先实现 ImageGetter 类，返回一个我们自定义的 URLDrawable 对象。然后使用 AsyncTask 加载网络图片，加载完成后将图片设置到 URLDrawable 内部，并对 TextView 做一次重新赋值的操作，让其进行一次刷新来显示我们异步加载的图片。

#### 6.
在使用 Kotlin 的过程中如果还遵循 Java 的那套编码习惯，恐怕写出来的代码不比 Java 的简单到哪里去，既然使用了 Kotlin，就要将其特性都用上。

首先要说的就是它自带的 lambda 表达式，用起来确实省事。就拿设置 Button 的响应事件来讲，java 的如下：

```java
button.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        // ...
    }
});
```

在 Kotlin 中：

```kotlin
 button.setOnClickListener { 
        // ...
}
```

是不是简单了很多，还有另外值得一提的就是我们要利用好 Kotlin 的 Extensions 这个特性，这样可以是代码更具可读性，下面举个简单的例子，比如我们通常使用的 SharedPreference 的时候有时候会忘记最后的 commit 或者 apply 操作。传统的代码写法如下：

```java
SharedPreferences sharedPreferences = getContext().getSharedPreferences("text",Context.MODE_PRIVATE);
sharedPreferences.edit().putBoolean("b1",true).putString("str","foo").putLong("l1",1L).apply();
```

但如果在 Kotlin 中结合了 Extensions 特性，则写法相当风骚。

```kotlin
val sharedPreference  = context.getSharedPreferences("test",Context.MODE_PRIVATE)
sharedPreference.save {
    putBoolean("b1",true)
    putString("str1","foo")
    putLong("l1",1L)
}
```
首先看这里，apply没有了，并且 putBoolean 这些操作前也没有了相应的对象。更神奇的是这里的 save 方法，SharedPreferences 应该没有这个方法的。其实这一切都是 Extensions 的功劳，我没看一下隐藏在上面代码背后的几行代码：

```kotlin
    fun SharedPreferences.save(func: SharedPreferences.Editor.()->Unit){
        val edit = edit()
        edit.func()
        edit.apply()
    }
```

上述代码首先给 SharedPreferences 扩展了一个 save 方法，然后在扩展方法里做了 Editor 的初始化和最后的 apply 工作。只要在项目中的一处地方给出定义，其他地方都能使用。是不是很方便？

## 完成

下面是 App 的部分截图，

![image](https://blog-1256162814.cos.ap-nanjing.myqcloud.com/normal/2702499-6b41628dadbf72b1.webp)

源码在我的github上，分别是 dclib 和 dcapp，我的 github 地址：[https://github.com/ZhangQinglian](https://github.com/ZhangQinglian)