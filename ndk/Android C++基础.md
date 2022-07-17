### Android C++基础

#### 1.命名空间的使用

```c++
using namespace std;//使用命名空间std标准的命名空间
```

##### 1.1自己定义命名空间

```c++
namespace namespaceB {
    int a = 20;
    namespace namespaceC {
        struct Teacher {
            char name[20];
            int age;
        };
    }
}
//虽然有嵌套，使用的时候可以直接使用
using namespace namespaceC;
Teacher t2;
```



#### 2.bool 类型

```
true代表真值，编译器内部用1表示
false代表假值，编译器内部用0表示
c++编译器会在赋值的时候将非0转换为true,0转化给false
bool b1 = true;//c++编译器

如果给b1赋除了0或者1别的值，b1的值不会变化
```



#### 3.const 使用

```c++
const int a = 10;
int const b = 20;//一样的

const int *c;//常量指针 const修饰的是指针所指向的变量   输入参数
//代表指针所指向的内存空间，不能被修改
 c = &a;
 c = &b;
// *c = 30;//常量指针不允许修改指针所执向的内存空间  这里不允许

int *const d = &a1;//指针常量，const修饰的是指针本身，指向不可变  必须赋初值


const int *const e = &a1;//常量指针常量
 //指针的指向不能改变，所指向的内存空间也不能改变


```



#### 4.引用

```c++
int a = 10;
// type & name = var;
int &b = a;//b就是一个引用,请不要用C的语法是思考


int myswap(int a,int b){//值传递
    int tmp = a;
    a = b;
    b = tmp;
}//完成不了交换的功能

//传指针可以完成位置交换的
int myswap1(int *a,int *b){
    int tmp = *a;
    *a = *b;
    *b = tmp;
}

//使用引用了就可以交换了  相当于取地址了  
//将a he b指向的地址更改位置  那么值肯定已经改变了位置
int myswap2(int &a,int &b){
    int tmp = a;
    a = b;
    b = tmp;
}


//引用就是取别名
//引用在C++的内部实现 就是一个常量指针


```

