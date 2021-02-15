### 13.5 动态内存管理类StrVec

 *talk is cheap, show me the code*  
 
~~~cpp
//头文件中的动态内存管理类StrVec
class StrVec
{
public:
	//构造函数
	StrVec() :elements(nullptr), first_free(nullptr), cap(nullptr) {}
	//拷贝构造函数
	StrVec(const StrVec&);
	//拷贝赋值运算符
	StrVec& operator=(const StrVec&);
	//析构函数
	~StrVec();
	//成员函数
	void push_back(const string&);
	void reserve(const size_t n);
	void resize(const size_t n);
	void resize(const size_t n, string& t);
	//下面四个成员函数单独一组，因为在头文件中就已经被定义了
	size_t size() const { return first_free - elements; }
	size_t capacity() const{ return cap - elements; }
	string* begin() const { return elements; }
	string* end() const { return first_free; }

private:
	static allocator<string> alloc;//分配元素
	//私有成员函数
	void chk_n_alloc();
	pair<string*, string*> alloc_n_copy(const string*, const string*);
	void free();//销毁元素并释放内存
	void reallocate();//获得更多内存并拷贝已有元素
	//私有成员变量
	string* elements;
	string* first_free;
	string* cap;
};
~~~  
~~~cpp  

  //源文件中成员函数的定义
#include "StrVec.h"
//拷贝构造函数
StrVec::StrVec(const StrVec& s)
{
	auto newdata = alloc_n_copy(s.begin(), s.end());
	elements = newdata.first;
	first_free = cap = newdata.second;
}
//拷贝赋值运算符
StrVec& StrVec::operator=(const StrVec& rhs)
{
	auto data = alloc_n_copy(rhs.begin(), rhs.end());
	free();//这一步很关键：因为调用拷贝赋值运算符有可能发生在初始化阶段也有可能发生在赋值阶段（要清空原有内存）
	elements = data.first;
	first_free = cap = data.second;
	return* this;
}
//析构函数
StrVec::~StrVec()
{
	free();
}
//成员函数
void StrVec::push_back(const string& s)
{
	chk_n_alloc();//检查空间是否已满
	alloc.construct(first_free++, s);
}

void StrVec::reserve(const size_t n)
{
    //分配至少能容纳n个元素的空间，如果n小于等于当前容量，那么reserve什么都不会做
	if (capacity() < n)
	{
		auto data = alloc.allocate(n);
		auto dest = data;
		auto elem = elements;
		for (size_t i = 0; i != size(); ++i)
		{
			alloc.construct(dest++, std::move(*elem++));
		}
		free();
		elements = data;
		first_free = dest;
		cap = data + n;
	}
}

void StrVec::resize(const size_t n)
{
	//调整容器的大小为n个元素。如果当前容器元素大于n，那多出来的元素将被丢弃
	auto data = alloc.allocate(n);
	auto dest = data;
	auto elem = elements;
	for (size_t i = 0; i < n; ++i)
	{
		if (i < size())
		{
			alloc.construct(dest++, std::move(*elem++));
		}
		else
		{
			alloc.construct(dest++);
		}
	}
	free();
	elements = data;
	first_free = dest;
	cap = data + n;
}

void StrVec::resize(const size_t n, const string& t)
{
	//调整容器的大小为n个元素，每个元素的值都为t
	auto data = alloc.allocate(n);
	auto dest = data;
	auto elem = elements;
	for (size_t i = 0; i < n; ++i)
	{
		alloc.construct(dest++, t);
	}
	free();
	elements = data;
	first_free = dest;
	cap = data + n;
}
//私有成员函数
void StrVec::chk_n_alloc()
{
	if (size() == capacity())
		reallocate();
}

pair<string*, string*> StrVec::alloc_n_copy(const string* b, const string* e)
{
    /*
    alloc_n_copy函数的功能：
    接受两个指向string对象的指针，一前一后，把这两个指针所圈定的范围内的元素拷贝一份
    返回拷贝后新内存空间的首地址和尾地址
    */
	auto data = alloc.allocate(e - b);
	return make_pair(data, uninitialized_copy(b, e, data));
}

void StrVec::free()
{
	if (!elements)
	{
		for (auto it = first_free; it != elements;)
			alloc.destroy(it--);
		alloc.deallocate(elements, cap - elements);
	}
}

void StrVec::reallocate()
{
	//定义一个新空间大小，如果原来空间大小size()大于0，则新空间大小是原来的两倍大
	auto newcapacity = (size() ? 2 * size() : 1);
	//使用allocator新开辟一个内存空间，大小为原来内存空间的两倍大
	auto newdata = alloc.allocate(newcapacity);
	//设置指针，方便后序遍历内存空间
	auto dest = newdata;
	auto elem = elements;
	//使用std::move使用string的移动构造函数，这样的原内存空间的元素是被移动到新空间的，而不是拷贝了新的一份过去
	for (size_t i = 0; i != size(); ++i)
		alloc.construct(dest++, std::move(*elem++));
	//原空间已经没用了，调用free成员函数将原空间的对象和内存都销毁（应该没有对象可销毁把，毕竟都移动走了）
	free();
	//原空间被销毁了，让私有成员指针指向新内存的首元素等位置
	elements = newdata;
	first_free = dest;
	cap = newdata + newcapacity;
}
~~~  

让我们复盘一下这个程序设计：

**项目需求**

用allocator实现一个简易版的vector

**用到的技术**

1. 自定义的拷贝函数、拷贝赋值运算符、析构函数。第13章整章内容都是围绕类的拷贝和析构函数来讲的。这里再次强调一下为什么需要自定义拷贝构造函数和析构函数：对于功能比较复杂，内存管理上比较麻烦的类，如果不自定义拷贝和析构函数，编译器会自己帮我们生成合成拷贝构造函数和析构函数，而往往这种自动生成的函数，所实现的功能并不能满足我们的需求，不仅如此，涉及到内存管理的问题，还会造成内存泄露或者非法访问等BUG。
2. 用地址指针实现简易版的迭代器。虽然我不知道STL的迭代器到底和指针有什么区别，不过可以明确一点，迭代器不是简单的指针。但是，既然这个程序是简易版vector，那么用指针实现迭代器还是能够满足要求，毕竟迭代器和指针都具有几个相同的功能：解引用、相加相减运算
3. 使用了std::move函数。使用该函数的作用，是为了将旧内存的元素“搬运”到新内存中去，避免拷贝复制造成的效率损失
4. 定义了一个私有的静态成员变量。该程序将alloc成员设置为static修饰的静态成员，为什么这个我要单独拉出来说呢？因为这个操作有点细节。大家都知道static静态成员是属于类的，不属于类的对象。换句话说，类的多个对象使用的都是同一个static的静态成员。这样做有什么好处？我说不清楚，这也许和allocator类有关，也许会为了避免资源浪费吧。毕竟alloc成员是一个功能组件，创造它的目的只是为了用它来申请内存，如果每一个对象都有一个自己的alloc，好像没必要？

