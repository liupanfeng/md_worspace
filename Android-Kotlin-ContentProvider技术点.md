**Android-Kotlin-ContentProvider技术点**

ContentProvider内容提供者，主要用于再不同的应用程序之前实现数据共享的功能，它提供了一套完整的机制，允许一个程序访问另外一个程序的数据，同时还能保证数据的安全性。通讯录的联系人信息，短信信息，媒体库信息等都通过这个方式把数据共享出来，第三方可以获取到进行二次开发。



**1.ContentProvider访问程序的数据**

contentProvider的用法有两种

* 使用现用的ContentProvider访问相应程序的数据。
* 创建自己的contentProvider给程序的数据提供外部访问接口。

如果向访问共享的数据需要借助ContentResolver，这个类提供了一系列的方法对数据进行增删查改操作：insert、update、delete、query

ContentResolver通过URI来访问数据，**URI字符串的格式 content：//程序包名.provider/表名**



**2.ContentResolver 使用**

* 查询数据

```kotlin
public final @Nullable Cursor query(@RequiresPermission.Read @NonNull Uri uri,
        @Nullable String[] projection, @Nullable String selection,
        @Nullable String[] selectionArgs, @Nullable String sortOrder) {
    return query(uri, projection, selection, selectionArgs, sortOrder, null);
}
```

Uri：确定了访问数据的唯一标识，指定查询某个应用程序下的某一张表

projection:指定查询的列名字，是一个字符串，可以同时指定多个列名字。

selection：指定where的约束条件。

selectionArgs：为where中的占位符提供具体的值。

sortOrder：指定结果的排序方式。

这个方法返回一个Cursor对象，使用Cursor遍历得到的数据

```kotlin
while (cursor?.moveToNext()!!){
    val column1 = cursor.getString(cursor.getColumnIndex("column1"))
    val column2 = cursor.getString(cursor.getColumnIndex("column2"))
}
cursor.close()
```

* 插入数据

```kotlin
val values= contentValuesOf("columu1" to "content","column2" to 1)
contentResolver.insert(Uri.parse(uri),values)
```

* 更新数据

```kotlin
val  valuesUpdate= contentValuesOf("column1" to "")
contentResolver.update(Uri.parse(uri),values,"column1 = ? and column2 = ?", arrayOf("text","1"))
```

使用selection selectionArgs 参数对想要更新的数据进行约束，防止所有的数据都会收到影响

* 删除数据

```kotlin
contentResolver.delete(Uri.parse(uri),"column2 = ?", arrayOf("1"))
```



**3.使用ContentProvider读取联系人的数据**

```kotlin
if(ContextCompat.checkSelfPermission(this,Manifest.permission.READ_CONTACTS)!=PackageManager.PERMISSION_GRANTED){
    ActivityCompat.requestPermissions(this, arrayOf(Manifest.permission.READ_CONTACTS),1)
}else{
    readContacts()
}
```

读取系统联系人需要动态申请权限，先确认程序是否具备READ_CONTACTS权限，如果存在权限，读取联系人数据，不存在就申请权限。



```kotlin
override fun onRequestPermissionsResult(requestCode: Int, permissions: Array<out String>, grantResults: IntArray) {
    super.onRequestPermissionsResult(requestCode, permissions, grantResults)
    when(requestCode){
        1->{
            if (grantResults.isNotEmpty()&&grantResults[0]==PackageManager.PERMISSION_GRANTED){
                readContacts()
            }else{
                showToast("you denied the permission")
            }
        }
    }
}
```

请求权限之后，不管用户是否给权限都会回调onRequestPermissionsResult方法，如果用户点击允许执行读取联系人的逻辑，否则给Toast提示



```
private fun readContacts() {
    mDatas.clear()
    contentResolver.query(ContactsContract.CommonDataKinds.Phone.CONTENT_URI,null,
        null,null,null)?.apply {
         while (moveToNext()){
             val displayName=getString(getColumnIndex(ContactsContract.CommonDataKinds.Phone.DISPLAY_NAME))
             val number=getString(getColumnIndex(ContactsContract.CommonDataKinds.Phone.NUMBER))
             val contacts=Contacts(name = displayName,number =number )
             mDatas.add(contacts)
         }
    }
    contactAdapter.setData(mDatas)

}
```

读取联系人信息，并遍历cursor数据，获取到名字和手机号码

ContactsContract.CommonDataKinds.Phone.CONTENT_URI：这个是数据库Uri

ContactsContract.CommonDataKinds.Phone.DISPLAY_NAME：这个是指名称的列名字

ContactsContract.CommonDataKinds.Phone.NUMBER：这个是手机号的列名字



**4.通过ContentProvider暴露自己程序的数据接口**

```kotlin
class DatabaseProvider : ContentProvider() {

    private val bookDir = 0
    private val bookItem = 1
    private val authority = "com.test.kotlin_test.provider"
    private var dbHelper : MySqliteDataHelper ?= null

    
    private val uriMatcher by lazy {
         val mathcher=UriMatcher(UriMatcher.NO_MATCH)
        mathcher.addURI(authority,"book",bookDir)
        mathcher.addURI(authority,"book/#",bookItem)
        mathcher
    }


    override fun insert(uri: Uri, values: ContentValues?)=dbHelper?.let {

        val db =it.writableDatabase
        val uriReturn=when(uriMatcher.match(uri)){
            bookDir ->{
                val newBookId=db.insert("Book",null,values)
                Uri.parse("content://$authority/book/$newBookId")
            }
            else ->null
        }
        uriReturn
    }

    override fun query(
        uri: Uri,
        projection: Array<out String>?,
        selection: String?,
        selectionArgs: Array<out String>?,
        sortOrder: String?
    ) = dbHelper?.let {
        val db=it.readableDatabase
        val cursor = when(uriMatcher.match(uri)){
            bookDir ->db.query("Book",projection,selection,selectionArgs,null,null,sortOrder)
            bookItem -> {
                val bookId=uri.pathSegments[1]
                db.query("Book",projection,"id=?", arrayOf(bookId),null,null,sortOrder)
            }
            else->null
        }
        cursor
    }

    override fun onCreate() = context?.let {
        dbHelper=MySqliteDataHelper(it,"BookStore.db",2)
        true
    } ?:false

    override fun update(uri: Uri, values: ContentValues?, selection: String?, selectionArgs: Array<out String>?)= dbHelper?.let {
        val db=it.writableDatabase
        val updateRows=when(uriMatcher.match(uri)){
            bookDir -> db.update("Book",values,selection,selectionArgs)
            bookItem ->{
                val bookId=uri.pathSegments[1]
                db.update("Book",values,"id=?", arrayOf(bookId))
            }
            else -> 0
        }
        updateRows
    } ?: 0

    override fun delete(uri: Uri, selection: String?, selectionArgs: Array<out String>?)=dbHelper?.let {
        val db=it.writableDatabase
        val deleteRows=when(uriMatcher.match(uri)){
            bookDir -> db.delete("Book",selection,selectionArgs)
            bookItem -> {
                val bookId=uri.pathSegments[1]
                db.delete("Book","id=?", arrayOf(bookId))
            }
            else->0
        }
        deleteRows
    }?: 0

    override fun getType(uri: Uri)=when(uriMatcher.match(uri)){
        bookDir -> "vnd.android.cursor.dir/vnd.com.test.kotlin_test.provider.book"
        bookItem -> "vnd.android.cursor.item/vnd.com.test.kotlin_test.provider.book"
        else -> null
    }
   }
```

* 在by lazy 块里边对uriMatcher进行初始化，开始的时候并不会进行加载，只有当uriMatcher首次进行调用的时候才会执行





```kotlin
<provider android:authorities="com.test.kotlin_test.provider"
          android:enabled="true"
          android:exported="true"
          android:name=".content_provider.DatabaseProvider"/>
```

在清单文件进行注册，exported是否允许外部程序访问，enabled是否启动这个提供者



```kotlin
val uri=Uri.parse("content://com.test.kotlin_test.provider/book")
contentResolver.query(uri,null,null,null,null)?.apply {
    while (moveToNext()){
        val name=getString(getColumnIndex("name"))
        val author=getString(getColumnIndex("author"))
        val pages=getInt(getColumnIndex("pages"))
        val price=getDouble(getColumnIndex("price"))
        Log.d(TAG,"anme=$name,author=$author,price=$price,pages=$pages")
    }
}
```

通过我们自己的提供者进行数据查询，并将结果打印出来



```kotlin
 val uri=Uri.parse("content://com.test.kotlin_test.provider/book")
 val contentValuesOf =
     contentValuesOf("name" to "Kings of the world", "author" to "kings", "pages" to 1001, "price" to 22.68)
 val newUri = contentResolver.insert(uri, contentValuesOf)
bookId = newUri?.pathSegments?.get(1)
```

通过我们自己的内容提供者插入一条数据



```kotlin
val uri=Uri.parse("content://com.test.kotlin_test.provider/book")
val contentValue=contentValuesOf("price" to 14)
contentResolver.update(uri,contentValue,"name=?", arrayOf("Kings of the world"))
```

通过我们自己的内容提供者插入更新一条数据



```kotlin
val uri=Uri.parse("content://com.test.kotlin_test.provider/book")
contentResolver.delete(uri,"name=?", arrayOf("Kings of the world"))
```

通过我们自己的内容提供者删除一条数据

上面的几个部分可以已经测试均正确没有

