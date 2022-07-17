### Android-Kotlin-网络

Android上发送HTTP请求，一般有两个方式：HttpURLConnection和HttpClient。HttpClient由于存在API数量多，扩展困难缺点，Android6.0开始HttpClient的功能完全移除，被废弃。

进行网络请求，请先配置权限：<uses-permission android:name="android.permission.INTERNET" />



**1.纯HttpURLConnection进行数据请求**

```kotlin
private fun doRequestWithHttpURLConnection(){
    thread {
        var connection:HttpURLConnection?=null
        val response = StringBuilder()
        val url = URL("https://www.baidu.com/")
        connection = url.openConnection() as HttpURLConnection
        connection.connectTimeout = 8000
        connection.readTimeout = 8000
        val inputStream = connection.inputStream
        val reader = BufferedReader(InputStreamReader(inputStream))

        reader.use {
            reader.forEachLine {
                response.append(it)
            }
        }

        showResponse( response.toString())

    }
}


 private fun showResponse(content: String) {
        runOnUiThread {
            responseText.text = content
        }

    }

```

注意：网络请求需要放置在子线程，刷新UI需要在UI线程



**2.使用OKHttp**

在开源盛行的今天，网络方面有很多开源的框架，其中OkHttp是比较优秀的。



```kotlin
implementation 'com.squareup.okhttp3:okhttp:4.1.0'
```

使用OkHttp需要添加这个依赖，添加之后会自动加载两个库一个是OkHttp库，一个是Okio库，后者是前者通信的基础。



```kotlin
private fun doRequestWithOkHttp(){
    thread {
        val client=OkHttpClient()
        val request=Request.Builder()
            .url("https://www.baidu.com/")
            .build()
        val response=client.newCall(request).execute()
        val string = response.body?.string()
        showResponse(string)
    }
}
```



如果向使用post的方式发送网络请求，需要构建一个FromBody

