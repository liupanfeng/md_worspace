Android-Kotlin 高级语法

**1.扩展函数**

定义：扩展函数是指在不修改某个类的源码的情况下，仍然可以打开某个类，向这个类添加新方法。

如何定义扩展函数，语法如下

```kotlin
fun ClassName.methodName(param1:Int,param2:Int):Int{
    return 0
}
```

只需要在函数名前面加上一个ClassName.语法结构，就表示该函数添加都某个类上面了。



案例：String类想必都很熟悉了，现在往这个类增加一个lettersCount方法，用于统计字符的个数，目前这个类是没有这个方法的。

```kotlin
//这样就在String类上面定义了一个函数
fun String.letterCount():Int{
    var count=0
    for(char in this){
        if (char.isLetter()){
            count++
        }
    }
    return count
}
```

这样就定义了一个扩展函数，这个函数一般建议新建一个String.kt的顶层函数，放置在这个函数里边。这个特性使Kotlin变的非常灵活。



**2.运算符重载**





