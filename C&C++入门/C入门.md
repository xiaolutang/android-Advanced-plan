# C++中的基本数据类型

C++有整型数据：

short、int、long 和 long long 

浮点型有：

float、double、long double

字符型：

char

布尔类型：

bool

C++中类型的所占内存的字节数和运行的系统有关。我们可以通过sizeOf方法来进行获取。

# C中的内存分配

在C中没有new关键字，但它提供了一些方法来申请分配内存。

**malloc** 
	没有初始化内存的内容,一般调用函数memset来初始化这部分的内存空间.

**calloc**
	申请内存并将初始化内存数据为NULL.

​	 ` int *pn = (int*)calloc(10, sizeof(int));`

**realloc**

​	 对malloc申请的内存进行大小的调整.

```C
char *a = (char*)malloc(10);
realloc(a,20);
```

**特别的**：
**alloca**
	在栈申请内存,因此无需释放.

```C
int *p = (int *)alloca(sizeof(int) * 10);
```



# C指针

使用 *声明指针。 指针实际上就是存的一个地址

**注意**指针声明了立即赋值

野指针：没有进行赋值

悬空指针：指向的内存被释放了。

&：取址符号

解引用：解析某个指针地址的值 使用 *指针变量

通过指针修改内存值，对应引用的值也会发生改变。

指针支持自增自减操作。是对地址值进行操作。

可以通过数组遍历指针元素。因为数组保存的是连续的内存。



数组存放的是一块内存连续的数据，操作数组名就能够操作内存首地址，数组名+1，指向的是数组的下一个内存地址。

## **数组指针与指针数组**

```c = 
//数组的指针
int array[2][3] = {{11,22,33},{44,55,66}};
int (*array_p)[3]= array;
//取55这个数据  array_p+1指针数组+1到下一个长度为3的指针数组的首地址，对数组进行解引用，得到数组的首地址，+1就是55的地址；在解引用就是55
*(*(array_p+1) + 1);

//指针的数组
int *p_array[3] = {&i1,&i2,&i3}
```

## **const关键字**

const表示不可变的变量类似java的final

```C
//从右往左读
//P是一个指针 指向 const char类型
char str[] = "hello";
const char *p = str;
str[0] = 'c'; //正确
p[0] = 'c';   //错误 不能通过指针修改 const char

//可以修改指针指向的数据 
//意思就是 p 原来 指向david,
//不能修改david爱去天之道的属性，
//但是可以让p指向lance，lance不去天之道的。
p = "12345";

//性质和 const char * 一样
char const *p1;

//p2是一个const指针 指向char类型数据
char * const p2 = str;
p2[0] = 'd';  //正确
p2 = "12345"; //错误

//p3是一个const的指针变量 意味着不能修改它的指向
//同时指向一个 const char 类型 意味着不能修改它指向的字符
//集合了 const char * 与  char * const
char const* const p3 = str;
```

## **多级指针**

指向指针的指针

一个指针包含一个变量的地址。当我们定义一个指向指针的指针时，第一个指针包含了第二个指针的地址，第二个指针指向包含实际值的位置。

```C
int a = 10;
int *i = &a;
int **j = &i;
// *j 解出 i   
printf("%d\n", **j);
```

## **函数：**

可变参数

## **函数指针：**

指向函数的指针

```C
void println(char *buffer) {
	printf("%s\n", buffer);
}
//接受一个函数作为参数
void say(void(*p)(char*), char *buffer) {
	p(buffer);
}
void(*p)(char*) = println;
p("hello");
//传递参数
say(println, "hello");

//typedef 创建别名 由编译器执行解释
//typedef unsigned char u_char;
typedef void(*Fun)(char *);
Fun fun = println;
fun("hello");
say(fun, "hello");

//类似java的回调函数
typedef void(*Callback)(int);

void test(Callback callback) {
	callback("成功");
	callback("失败");
}
void callback(char *msg) {
	printf("%s\n", msg);
}

test(callback);
```

## **预处理器**

预处理器不是编译器，但是它是编译过程中一个单独的步骤。

预处理器是一个文本替换工具

所有的预处理器命令都是以井号（#）开头

**常用预处理器**

| #define  | 定义宏                                                      |
| -------- | ----------------------------------------------------------- |
| #include | 包含一个源代码文件                                          |
| #undef   | 取消已定义的宏                                              |
| #ifdef   | 如果宏已经定义，则返回真                                    |
| #ifndef  | 如果宏没有定义，则返回真                                    |
| #if      | 如果给定条件为真，则编译下面代码                            |
| #else    | #if 的替代方案                                              |
| #elif    | 如果前面的 #if 给定条件不为真，当前条件为真，则编译下面代码 |
| #endif   | 结束一个 #if……#else 条件编译块                              |
| #error   | 当遇到标准错误时，输出错误消息                              |
| #pragma  | 使用标准化方法，向编译器发布特殊的命令到编译器中            |

```C
//宏一般使用大写区分
//宏变量
//在代码中使用 A 就会被替换为1
#define A 1
//宏函数
#defind test(i) i > 10 ? 1: 0

//其他技巧
// # 连接符 连接两个符号组成新符号
#define DN_INT(arg) int dn_ ## arg
DN_INT(i) = 10;
dn_i = 100;

// \ 换行符
#define PRINT_I(arg) if(arg) { \
 printf("%d\n",arg); \
 }
PRINT_I(dn_i);

//可变宏
#define LOGE(...) __android_log_print(ANDROID_LOG_ERROR,"NDK", __VA_ARGS__);

//陷阱
#define MULTI(x,y)  x*y
//获得 4
printf("%d\n", MULTI(2, 2));
//获得 1+1*2  = 3
printf("%d\n", MULTI(1+1, 2));
```

宏函数

​	优点：

​		文本替换，每个使用到的地方都会替换为宏定义。

​		不会造成函数调用的开销（开辟栈空间，记录返回地址，将形参压栈，从函数返回还要释放堆		

​		栈。）

​	缺点：

​		生成的目标文件大，不会执行代码检查



内联函数

​	和宏函数工作模式相似，但是两个不同的概念，首先是函数，那么就会有类型检查同时也可以debug
在编译时候将内联函数插入。

不能包含复杂的控制语句，while、switch，并且内联函数本身不能直接调用自身。
如果内联函数的函数体过大，编译器会自动的把这个内联函数变成普通函数。

# 结构体



# 公用体