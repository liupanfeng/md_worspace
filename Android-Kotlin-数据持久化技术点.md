**Android-Kotlin-数据持久化技术点**

**1.数据持久化的方式**

Android中主要提供了三种数据持久化的方式：文件存储、SharePreferences存储以及数据库存储



**2.文件存储**

```Kotlin
fun save(inputText:String){
    val output=openFileOutput("data", Context.MODE_PRIVATE)
    val write=BufferedWriter(OutputStreamWriter(output))
    write.use { 
        it.write(inputText)
    }
}
```

借助于Context的openFileOutput方法

第一个参数是文件名，这个参数不能带路径，因为所有的文件默认保存到/data/data/<packagename>/files/目录下

第二个参数是文件的操作模式，MODE_PRIVATE:表示文件名相同内容会直接覆盖  MODE_APPEND:表示文件存在则直接追加

这里使用了一个use的扩展函数，**这个函数中的代码全部执行完之后会自动将外层的流关闭，不需要再手动写finally进行流关闭了**

另外Kotlin中没有try catch，不需要像java那些写try catch代码了



**3.文件中读取数据**

```kotlin
fun read():String{
    val content=StringBuilder()
    val input=openFileInput("data")
    val reader=BufferedReader(InputStreamReader(input))
    reader.use {
        reader.forEachLine { 
            content.append(it)
        }
    }
    return content.toString()
}
```

借助openFileInput方法

forEachLine是Kotlin的一个内置函数，它会将读到的每一行内容都回到Lambda表达式中，然后完成拼接



**4.将数据保存到SharedPreferences**

```kotlin
fun saveToSharePreference(content:String){
    getSharedPreferences("data",Context.MODE_PRIVATE).edit().apply {
        putString("name","jack")
        putInt("age",16)
        putString("desc",content)
        apply()
    }
    
}
```

SharePreference保存数据是以键值对的方式保存的

保存的路径是/data/data/<packagename>/shared_prefs/目录下



**5.从SharedPreferences读取数据**

```kotlin
fun readFromSharePreference() {
    getSharedPreferences("data", Context.MODE_PRIVATE).apply {
        val name = getString("name", "")
        val age = getInt("age", 0)
        val desc = getString("desc", "")
        Log.d(TAG, "name=$name,age=$age,desc=$desc")
    }
}
```



**6.SQLite数据库存储**

想使用数据库google提供了SQLiteOpenHelper来完成这些操作

```Kotlin
class MySqliteDataHelper(val context: Context,name:String,version:Int):SQLiteOpenHelper(context,name,null,version) {

    private val createBook="create table Book( " +
            "id integer primary key autoincrement," +
            "author text," +
            "price real," +
            "pages integer," +
            "name text)"


    private val createCategory="create table Category( id integer primary key autoincrement," +
            "category_name text," +
            "category_code integer)"

    override fun onCreate(db: SQLiteDatabase?) {
        db?.execSQL(createBook)
        db?.execSQL(createCategory)
        showToast("create books success")
    }

    override fun onUpgrade(db: SQLiteDatabase?, oldVersion: Int, newVersion: Int) {
        //根据版本号判断
        if (oldVersion<=1){
            db?.execSQL(createCategory)
        }

        if (oldVersion<=2){
            db?.execSQL("alter table Book add column category_id")
        }

        //...

    }
}
```

这个是抽象类，需要一个写一个实现类，这个抽象类构造方法有四个参数

第一个是Context上下文

第二个是数据库名字

第三个是允许查询数据库返回一个自定义的cursor

第四个是当前数据库的版本号

onCreate方法只会调用一次，当数据库存在之后，onCreate方法便不会执行了，所以如果需要有调整比如添加一个新表或者添加一个字段等需要在onUpgrade方法中执行

数据库创建之后，存在的路径是/data/data/<packagename>/目录下



初始化SQLiteOpenHelper之后，可以调用getReadableDatabase或者getWriteableDatabase

getReadableDatabase：这个是以只读的方式打开数据库

getWriteableDatabase：这个是可读可写的方式打开数据库（如果磁盘满了，会出现异常）



查看数据库中的表是否创建成功，可以借助Android studio插件 Database Navigator，安装插件之后，Android studio 左侧会出现DB Browser工具，将数据库从sd卡导出来直接导入到这个工具就可以查看了。



**7.对数据库进行增删查改**

```kotlin
fun insertData(view: View){
    val value=ContentValues().apply {
        put("name","斗破苍穹")
        put("author","tom")
        put("pages",454)
        put("price",16.68)
    }
    writableDatabase.insert("Book",null,value)
}
```

给Book表插入一条数据，insert方法，第一个参数是表名字，第二个参数是用户未指定添加的时候给某些可为空的列自动赋值NULL，第三个参数是ContentValues对象。



```kotlin
fun updateData(view: View){
    writableDatabase = mySqliteDataHelper.writableDatabase
    val value=ContentValues()
    value.put("price",12.68)
    writableDatabase.update("Book",value,"name=?", arrayOf("斗破苍穹"))
}
```

更新数据，第二个参数是更新的内容，第三个参数是where占位符，第四个参数是占位符的值



```kotlin
fun deleteData(view: View){
    writableDatabase=mySqliteDataHelper.writableDatabase
    writableDatabase.delete("Book","price>?", arrayOf("12.68"))
}
```

删除数据，第二个参数是where占位符，第三个参数是占位符的值



```kotlin
fun queryData(view: View){
    writableDatabase=mySqliteDataHelper.writableDatabase
    val cursor = writableDatabase.query(
        "Book", null, null,
        null, null, null, null
    )
    while (cursor.moveToNext()){
        val name = cursor.getString(cursor.getColumnIndex("name"))
        val page = cursor.getInt(cursor.getColumnIndex("pages"))
    }
}
```

查询数据，并对结果进行遍历，query方法参数有些多，不过好多时候不需要全部写

//参数介绍 

第一个参数：表名 

第二个参数：查询的列名字 

第三个参数：指定where的约束条件 

第四个参数：为where指定的展位符设置的值 

第五个参数：指定需要group by   

第六个参数：对group by的结果进行一步约束  

第七个参数：指定查询结果的排序



**8.数据库事务操作**

```kotlin
//使用事务
fun useTransaction(){
    writableDatabase=mySqliteDataHelper.writableDatabase
    writableDatabase.beginTransaction()
    writableDatabase.delete("Book",null,null)

    val values = ContentValues().apply {
        put("name","斗罗大陆")
        put("author","tom")
        put("pages",454)
        put("price",15.68)
    }

    writableDatabase.insert("Book",null,values)
    writableDatabase.setTransactionSuccessful()
    writableDatabase.endTransaction()
}
```

数据库的事务中的内容，必须全部完成才可以，否则已经执行的语句会直接回滚，保留原来的旧数据，银行转账等都需要用到事务操作。



**9.数据库升级**

        
      override fun onUpgrade(db: SQLiteDatabase?, oldVersion: Int, newVersion: Int) {
            //根据版本号判断
        if (oldVersion<=1){
             db?.execSQL(createCategory)
         }
        if (oldVersion<=2){
            db?.execSQL("alter table Book add column category_id")
        }
    
        //...
    
    }

数据库升级，最好是根据oldVersion进行判断，这样如果从2升到3，只会执行后边的if语句，如果是从1升级到3则会执行两个语句，这样不管版本如何进行升级都可以保证数据库的表结构是最新的，而且表中的数据也不会丢失。







