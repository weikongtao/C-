# C++右值引用

右值引用是C++11标准的新内容，它之所以重要，是因为它是移动语义（Move semantics）与完美转发（Perfect forwarding）的基石。要理解右值引用，首先要区分**左值**和**右值**，还需要了解**表达式(expression)**的精确定义。

## 表达式

**表达式（expression）**是C++最低级别的计算。表达式将运算符作用于一个或多个运算对象，每个表达式都有对应的**求值结果**。表达式本身也可以作为运算对象，这时就得到了对多个运算符求值的**复合表达式**（含有多于一个运算符的表达式）。

## 左值（lvalue）与右值（rvalue）

左值和右值是C++中**表达式的属性**，在C++11中，每个表达式有两个属性：**类型**（**type**，除去引用特性，用于类型检查）和**值类型**（**value category**，用于语法检查，如表达式是否能被赋值等）。值类型包含三个基本类型：lvalue，prvalue与xrvalue。后两者统称为rvalue，即右值。

+ **左值（lvalue）**：求值结果为对象或函数的表达式。可以将左值看作一个**可以获取地址的量**，它可以用来标识一个对象或者函数。

+ **右值（rvalue）**：一种表达式，其结果是**值**而非**值所在的位置**。不是左值的量就是右值。右值要么是**字面值常量**，要么是在表达式求值过程中创建的**临时对象**。比如，函数中非引用类型的对象参数是右值，函数内部对象的返回值也是右值。

## 左值引用

引用必须初始化。

引用也分为**const引用**和**non-const引用**，对于non-const引用，只能用non-const左值来初始化：

```c++
int x = 20;
int &rx1 = x;  //non-const引用可以被non-const左值初始化

const int y = 10;
int &rx2 = y;  //非法：non-const引用不能被const左值初始化
int &rx3 = 10; //非法：non-const引用不能被右值初始化
```

***注：const变量虽然不可更改，但却是左值，这意味着该变量不仅有值，而且有持久的固定地址，而右值要么是字面值常量，要么是临时变量，没有持续存在的固定地址。***

const引用的限制比较少：

```C++
int x = 10;
const int cx = 20;
const int &rx1 = x;  // const 引用可以被non-const左值初始化
const int &rx2 = cx; // const 引用可以被const左值初始化
const int &rx3 = 9;  // const引用可以被右值初始化 
// ++++ 特别注意：const左值引用可以接受右值。++++
```



## 右值引用

首先，定义**右值引用**需要使用**&&**。

C++11中引入右值引用是为了支持移动操作以及完美转发，右值引用必须绑定右值。右值引用的一个重要性质是——只能绑定到一个**将要销毁的对象**，因此，我们可以自由的将一个右值引用的资源“移动”到另一个对象中。

右值引用一定不能被左值初始化，只能用右值初始化。

```c++
int x = 20;  // 左值
int && rrx1 = x;  //非法：右值引用无法被左值初始化
const int &&rrx2 = x;  //非法：右值引用无法被左值初始化
```

为什么？因为右值引用的目的是为了延长用来初始化对象的生命周期。而左值的生命周期与其作用域有关，无须延长。如何延长？

```c++
int x = 20; //左值
int &&rx = x * 2;  //x*2的结果是一个右值，rx延长其生命周期
int y = rx + 2;  //因此可以重用它：42
rx = 100;  //一旦你初始化一个右值引用变量，该变量就成了一个左值，可以被赋值
```

**注意：初始化之后的右值引用将变成一个左值，如果是non-const还可以被赋值。**

---

右值引用可以用于函数参数：

```c++
// 接收左值
void fun(int& lref)
{
    cout << "l-value reference\n";
}
// 接收右值
void fun(int&& rref)
{
    cout << "r-value reference\n";
}

int main()
{
    int x = 10;
    fun(x);   // output: l-value reference
    fun(10);  // output: r-value reference
}
```

可以看到，函数参数要区分右值引用和左值引用。

另外，如果函数参数是**const类型的引用**，则它不仅可以接收左值，还可以接收右值（如果你没有提供接收右值引用的重载版本）。

```c++
//函数不仅可以接收左值，还可以接收右值
void fun(const int& clref)
{
    cout << "l-value const reference\n";
}
```

## 移动语义

右值引用是移动语义的基础。一个对象的移动语义是通过移动构造函数和移动赋值运算符来实现的。以下通过创建一个动态数组类来说明：

```c++
template <typename T>
class DynamicArrary {
public:
	 explicit DynamicArrary(int size):
		 m_size{ size }, m_arrary{ new T[size] }{
		 cout << "Constructor: dynamic array is created!\n";
	 }

	 virtual ~DynamicArrary() {
		 delete[] m_arrary;
		 cout << "Destructor: dynamic array is destroyed!\n";
	 }

	 //复制构造函数
	 DynamicArrary(const DynamicArrary& rhs) : m_size{ rhs.m_size } {
		 m_arrary = new T[m_size];
		 for (int i = 0; i < m_size; ++i)
			 m_arrary[i] = rhs.m_arrary[i];
		 cout << "Copy constructor: dynamic array is created!\n";
	 }

	 //复制赋值操作符
	 DynamicArrary& operator=(const DynamicArrary& rhs) {
		 cout << "Copy assignment operator is called!\n";
		 if (this == &rhs)
			 return *this;
		 delete[] m_arrary;
		 m_size = rhs.m_size;
		 m_arrary = new T[m_size];
		 for (int i = 0; i < m_size; ++i)
			 m_arrary[i] = rhs.m_arrary[i];
		 return *this;
	 }

	 //索引运算符
	 T& operator[](int index) {
		 //不进行边界检查
		 return m_arrary[index];
	 }

	 const T& operator[](int index) const {
		 return m_arrary[index];
	 }

	 int size() const { return m_size; }


	 /**

	 //移动构造函数
	 DynamicArrary(DynamicArrary&& rhs):
		 m_size{ rhs.m_size }, m_arrary{ rhs.m_arrary }{
		 rhs.m_size = 0;
		 rhs.m_arrary = nullptr;
		 cout << "Move constructor: dynamic array is moved!\n";
	 }

	 //移动赋值操作
	 DynamicArrary& operator=(DynamicArrary&& rhs) {
		 cout << "Move assignment operator is called\n";
		 if (this == &rhs) return *this;
		 delete[] m_arrary;
		 m_size = rhs.m_size;
		 m_arrary = rhs.m_arrary;
		 rhs.m_size = 0;
		 rhs.m_arrary = nullptr;

		 return *this;
	 }

	 */
private:
	T* m_arrary;
	int m_size;
		
};
```

我们通过在堆上动态分配内存来实现动态数组类，类中实现了拷贝构造函数，拷贝赋值运算符和索引操作符。以下定义一个生产动态数组的工厂函数：

```c++
//生产int动态数组的工厂函数
DynamicArrary<int> arrayFactor(int size) {
	DynamicArrary<int> f_arr{ size };
	return f_arr;
	//由于arr是内部对象，无法向外传递，所以返回的是一个临时对象，
	//这个临时对象需要内部对象arr来初始化，由于这个动态数组arr即将消亡，
	//所以是右值，那么在构建临时对象时，会调用 复制构造函数（没有右值的版本， 但是右值可以传递给const左值引用参数）
}
```

然后用以下代码来测试：

```c++
int main()
{
    {
        DynamicArray<int> arr = arrayFactor(10);
    }
    return 0;
}
```

输出结果为：

```c++
Constructor: dynamic array is created!
Copy constructor: dynamic array is created!
Destructor: dynamic array is destroyed!
Destructor: dynamic array is destroyed!
```

我们对以上过程进行一个分析：

​		首先，调用arrayFactor()函数，函数内部通过**普通构造函数**创建了一个动态数组对象f_arr，此时输出**Constructor: dynamic array is created!**。然后要返回这个对象，由于这个对象是函数内部的，函数外无法获得，所以要再生成一个**临时对象**来返回，该临时对象由内部对象f_arr来初始化（内部对象arr即将消亡，所以是右值），这里为了初始化临时对象调用了**拷贝构造函数**（传入的参数是右值，但是右值可以传递给const左值引用参数），此时输出**Copy constructor: dynamic array is created!**。

​		其次，以上函数调用所返回的临时对象被用来初始化main函数中的arr对象，此时应该要调用**拷贝赋值运算符函数**，但是**编译器**在这里做了优化：**直接拿函数返回的临时对象初始化arr**。一旦完成arr的初始化，函数内部对象f_arr就被析构，此时输出**Destructor: dynamic array is destroyed!**。**注意：临时对象的内存占用是由编译器管理的，所以不会调用析构函数。**

​		最后，main函数中的arr对象离开作用域被析构，此时输出**Destructor: dynamic array is destroyed!**。

​		我们可以看到：尽管编译器做了一次优化，但还是导致对象被创建了两次，函数内部创建的动态数组对象f_arr仅仅是个中间对象，用完就被析构了，有没有可能将其申请的内存空间直接转移到arr，从而避免多创建一次对象？问题的关键在于拷贝构造函数执行的是复制，而不是转移。此时，需要**移动构造函数**：

```c++
template <typename T>
class DynamicArray
{
public:
        // ...其它省略

    // 移动构造函数
    DynamicArray(DynamicArray&& rhs) :
        m_size{ rhs.m_size }, m_array{rhs.m_array}
    {
        rhs.m_size = 0;
        rhs.m_array = nullptr;
        cout << "Move constructor: dynamic array is moved!\n";
    }

    // 移动赋值操作符
    DynamicArray& operator=(DynamicArray&& rhs)
    {
        cout << "Move assignment operator is called\n";
        if (this == &rhs)
            return *this;
        delete[] m_array;
        m_size = rhs.m_size;
        m_array = rhs.m_array;
        rhs.m_size = 0;
        rhs.m_array = nullptr;

        return *this;
    }
};
```

测试结果如下：

```c++
Constructor: dynamic array is created!
Move constructor: dynamic array is moved!
Destructor: dynamic array is destroyed!
Destructor: dynamic array is destroyed!
```

可以看到，调用的是移动构造函数，函数内部对象f_arr申请的内存空间直接被转移到arr。这减少了一份相同内存的申请与释放。注意：析构函数调用了两次，这是因为尽管内部进行了内存转移，但是内部对象f_arr依然存在，多以第一次析构的只是一个空指针**nullptr**，并不会影响程序。

通过该例也可以看到：一旦你自己创建了**拷贝构造**和**拷贝赋值运算符**函数，编译器便不会默认创建**移动构造**和**移动赋值运算符**函数。所以最好养成这四个函数一旦自己实现一个，就应该自己实现另外三个的习惯。

`C++11`中，`STL`的容器都实现了移动构造函数与移动赋值运算符。

## std::move

对象的移动语义是依靠**移动构造函数**和**移动赋值操作符**来实现的，但是前提是你传入的必须是**右值**。

有时候你需要将一个**左值进行移动语义**（因为你已经知道该左值以后不在使用），那么就需要将左值先转化为右值。在C++中，`std::move`就是为此而生。

```c++
vector<int> v1{1, 2, 3, 4};
vector<int> v2 = v1;             // 此时调用复制构造函数，v2是v1的副本
vector<int> v3 = std::move(v1);  // 此时调用移动构造函数，v3与v1交换：v1为空，v3为{1, 2, 3, 4}
```

可以看到，我们通过`std::move`将v1转化为**右值**，从而激发v3的移动构造函数，实现移动语义。

`C++`中利用`std::move`实现移动语义的一个典型函数是`std::swap`：实现两个对象的交换。`C++11`之前，`std::swap`的实现如下：

```c++
template <typename T>
void swap(T& a, T& b)
{
    T tmp{a};  // 调用复制构造函数
    a = b;     // 复制赋值运算符
    b = tmp;     // 复制赋值运算符
}
```

从上面的实现可以看到：共进行了3次复制。如果类型`T`比较占内存，那么交换的代价是非常昂贵的。但是利用移动语义，我们可以更加高效地交换两个对象：

```c++
template <typename T>
void swap(T& a, T& b)
{
    T temp{std::move(a)};   // 调用移动构造函数
    a = std::move(b);       // 调用移动赋值运算符
    b = std::move(tmp);     // 调用移动赋值运算符
}
```

仅通过三次移动，实现两个对象的交换，由于没有复制，效率更高！

你可能会想，`std::move`函数内部到底是怎么实现的。其实`std::move`函数并不“移动”，它仅仅进行了类型转换。下面给出一个简化版本的`std::move`:

```c++
template <typename T>
typename remove_reference<T>::type&& move(T&& param)
{
    using ReturnType = typename remove_reference<T>::type&&;

    return static_cast<ReturnType>(param);
}
```

首先看一下函数的返回类型，`remove_reference`在头文件中，`remove_reference<T>`有一个成员`type`，是`T`去除引用后的类型，所以`remove_reference<T>::type&&`一定是右值引用，对于返回类型为右值的函数其返回值是一个右值（准确地说是`xvalue`）。所以，知道了`std::move`函数的返回值是一个右值。

然后，我们看一下函数的参数，使用的是**通用引用类型（`&&`）**，**意味着其可以接收左值，也可以接收右值**。其推导规则如下：如果实参是左值，推导后的形参是左值引用，如果是右值，推导出来的是右值引用（感兴趣的话可以看看reference collapsing）。但是不管怎么推导，`ReturnType`的类型一定是右值引用，最后`std::move`函数只是简单地调用`static_cast`将参数转化为右值引用。

所以，`std::move`什么也没有做，只是告诉编译器将传入的参数无条件地转化为一个右值。所以，当你使用`std::move`作用于一个对象时，你只是告诉编译器这个对象要转化为右值，然后就有资格进行移动语义了！



下面举一个由于误用`std::move`而无效的例子。假如你在设计一个标注类，其构造函数接收一个`string`类型参数作为标注文本，你不希望它被修改，所以标注为const，然后将其复制给其的一个数据成员，你可能会使用移动语义：

```c++
class Annotation
{
public:
    explicit Annotation(const string& text):
        m_text {std::move(text)}
    { }

    const string& getText() const { return m_text; }
private:
    string m_text;
};
```

测试结果如下：

```c++
int main()
{
    string text{ "hello" };
    Annotation ant{ text };

    cout << ant.getText() << endl;  // output: hello
    cout << text << endl;           // output: hello 不是空，移动语义没有实现

    return 0;
}
```

你发现移动语义并没有被实现，这是为什么呢？首先，从直观上看，假如你移动语义成功了，那么`text`会发生改变，这会违反其const属性。所以，你不大可能成功！其实，`std::move`函数会在推导形参时会保持形参的const属性，所以其最终返回的是一个const右值引用类型，那么`m_text{std::move(text)}`到底会调用什么构造函数呢？我们知道`string`的内部有两个构造函数可能会匹配：

```c++
class string
{
    // ...
    string(const string& rhs);   // 复制构造函数
    string(string&& rhs);    // 移动构造函数
}
```

那么到底会匹配哪个呢？肯定的是移动构造函数不会被匹配，因为不接受const对象，复制构造函数会匹配吗？答案是可以，因为前面我们讲过const左值引用可以接收右值，const右值更可以！所以，你其实调用了复制构造函数，那么移动语义当然无法实现。

**所以，如果你想接下来进行移动，那不要把`std::move`引用在const对象上！**

## std::fast_forward与完美转发

完美转发就是创建一个函数，该函数可以接收任意类型的参数，然后将这些参数按原来的类型转发给目标函数，完美转发的实现要依靠`std::forward`函数。下面就定义了这样一个函数：

```c++
// 目标函数
void foo(const string& str);   // 接收左值
void foo(string&& str);        // 接收右值

template <typename T>
void wrapper(T&& param)
{
    foo(std::forward<T>(param));  // 完美转发
}
```

首先要有一点要明确，不论传入`wrapper`的参数是左值还是右值，一旦传入之后，`param`一定是**左值**(右值一旦被初始化即成为左值，大概)。

* 当一个类型为`string`类型的右值传递给`wrapper`时，`T`被推导为`string`，`param`为右值引用类型，但是一旦传入后，`param`就变成了左值，所以你直接转发给`foo`函数，将丢失`param`的右值属性，那么`std::forward`就确保传入`foo`的值还是一个右值；
* 当类型为`const string`的左值传递给`wrapper`时，`T`被推导为`const string&`，`param`为const左值引用类型，传入后，`param`仍为const左值类型，所以你直接转发给`foo`函数，没有问题，此时应用`std::forward`函数可以看成什么也没有做；
* 当类型为`string`的左值传递给`wrapper`时，`T`被推导为`string&`，`param`为左值引用类型，传入后，`param`仍为左值类型，所以你直接转发给`foo`函数，没有问题，此时应用`std::forward`函数可以看成什么也没有做；

所以`wrapper`函数可以实现完美转发，其关键点在于使用了`std::forward`函数确保传入的右值依然转发为右值，而对左值传入不做处理。

`std::forward`实现如下：

```c++
template<typename T> 
T&& forward(typename remove_reference<T>::type& param) 
{
    return static_cast<T&&>(param);
}
```

代码依然与`std::move`一样简洁，我们结合`wrapper`来看，如果传入`wrapper`函数中的是`string`左值，那么推导出`T`是`string&`，那么将调用`std::foward<string&>`，根据`std::foward`的实现，其实例化为：

```c++
string& && forward(typename remove_reference<string&>::type& param)
{
    return static_cast<string& &&>(param);
}
```

可以看到这里连续出现了3个`&`符号，我们知道`C++`不允许引用的引用，那么其实编译器这里进行是**引用折叠**（reference collapsing，大致就是后面的引用消掉），因此，变成：

```c++
string& forward(string& param)
{
    return static_cast<string&>(param);
}
```

上面的代码就很清晰了，一个左值引用的参数，然后还是返回左值引用，此时的`std::foward`就是什么也没有做，因为传入与返回完全一样。

那么如果传入`wrapper`函数中的是**`string`右值**，那么推导出`T`是`string`，那么将调用`std::foward<string>`，根据`std::foward`的实现，其实例化为：

```c++
string && forward(typename remove_reference<string>::type& param)
{
    return static_cast<string&&>(param);
}
```

继续简化，变成：

```c++
string&& forward(string& param)
{
    return static_cast<string&&>(param);
}
```

参数依然是左值引用（这点是一致的，因为前面说过传入`std:;forward`中的实参一直是左值），但是返回的是右值引用，此时的`std::foward`就是将一个左值转化了右值，这样保证传入目标函数的实参是右值！

总结：`std::foward`函数是**有条件地**将传入的参数转化为右值，而`std::move`**无条件地**将参数转化为右值，这是两者的区别。但是本质上，两者什么没有做，做多就是进行了一次类型转换。

## Reference

https://www.cnblogs.com/tomato-haha/p/17429268.html
