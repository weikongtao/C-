# C++临时对象

## 临时对象的构造与析构

在 C++ 中，临时对象（Temporary Object）是在表达式求值过程中创建的、无名字的对象。它们通常用于存储中间结果或作为函数调用的参数或返回值，其生命周期通常仅限于表达式的求值过程中。

临时对象的构建和析构与普通对象类似，只是它们的生命周期通常比较短暂，且没有命名。在 C++ 中，临时对象的构建和析构过程如下：

* **构建（构造函数）：**当创建一个临时对象时，编译器会调用相应的构造函数来初始化该对象的成员变量。构造函数负责为对象分配内存并对其进行初始化。临时对象的构建过程与普通对象一样，只是没有显式地使用对象名字而已。
* **析构（析构函数）：**临时对象的生命周期通常是在表达式求值结束后立即结束，因此它们的析构函数会在对象销毁时被调用。析构函数负责释放对象所占用的内存空间，以及执行其他必要的清理工作。与构造函数类似，析构函数的调用也是由编译器自动完成的，无需手动介入。

## 为什么要关注临时对象

关注临时对象是有意义的，因为它们可能会对代码的性能和行为产生影响。以下是一些关于临时对象的重要注意事项：

1. **性能问题：**在频繁创建临时对象的情况下，可能会导致不必要的开销，降低程序性能。避免不必要的临时对象可以提高性能。
2. **资源管理：**如果临时对象持有资源（例如动态分配的内存），需要确保它们在不再需要时被妥善释放，以避免资源泄漏。
3. **副作用：**某些情况下，临时对象的创建可能导致意外的副作用，这需要额外的注意。

## 临时对象的使用场景

1. **函数返回值：**当函数需要向外返回一个内部对象时，编译器需要创建一个临时对象来存储返回值，因为内部对象无法返回，所以用该内部对象来初始化该临时变量。调用方使用完该临时对象后立即销毁。比如：

   ```c++
   #include <iostream>
   using namespace std;
   
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
   private:
   	T* m_arrary;
   	int m_size;
   		
   };
   
   //生产int动态数组的工厂函数
   DynamicArrary<int> arrayFactor(int size) {
   	DynamicArrary<int> arr{ size };
   	return arr;
   	//由于arr是内部对象，无法向外传递，所以返回的是一个临时对象，
   	//这个临时对象需要内部对象arr来初始化，由于这个动态数组arr即将消亡，
   	//所以是右值，那么在构建临时对象时，会调用 复制构造函数（没有右值的版本， 但是右值可以传递给const左值引用参数）
   }
   
   int main()
   {
   	DynamicArrary<int> arr = arrayFactor(10);
   	
   	return 0;
   }
   ```
   
   main函数中调用arrayFactor()函数就会导致返回临时对象。
   
2. **函数参数传递：**临时对象可以作为函数的参数（以传值的方式）传递给函数。例如：

   ```c++
   void showArraySize(DynamicArrary<int> arr){
       std::cout << arr.size() <<endl;
   }
   
   int main(){
       showArraySize(DynamicArrary<int>(10)); //传递临时对象作为参数
       
       return 0;
   }
   ```

3. **表达式求值：**在表达式求值过程中，可能会生成临时对象。例如：

   ```c++
   MyClass obj1(10);
   MyClass obj2(20);
   MyClass obj3 = obj1 + obj2; // 表达式 obj1 + obj2 会生成临时对象
   ```

4. **类型转换：** 临时对象常用于类型转换。当您执行**显式或隐式类型转换**时，临时对象用于保存转换后的值。

   * **显示类型转换：**
   
     ```c++
     double result = static_cast<double>(5); // 创建临时 double 对象进行类型转换
     ```
   
   * **隐式类型转换：**
   
     ```c++
     void printFunc(const string&){  //注意const
         cout << string <<endl;
     }
     
     char *buf = "xxxx";
     printFunc(buf);  //发生隐式类型转换
     ```
   
     在调用printFunc()函数时，编译器首先会产生一个**string类型的临时变量**，该临时变量将buf转为string类型，然后传递给函数参数，直到函数返回，该临时变量才被销毁。
   
     **注意：**只有当参数类型为**const**，或者**传值方式**时，才能进行隐式类型转换，非const的传引用方式不能进行隐式类型转换。这是因为隐式类型转换生成的临时变量是**右值**，只有const类型的参数引用即可以接收左值，也可以接收右值，而非const类型的参数引用无法接收右值。
   
     