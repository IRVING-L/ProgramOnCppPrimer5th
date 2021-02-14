## 《C++ Primer 5th》中值得记录的程序设计题

### 12.1 自定义一个功能类似于*vector<string>*，使用智能指针管理内存空间的类StrBlob

*talk is cheap, show me the code*
~~~cpp
//类StrBlob
class StrBlob
{
public:
	friend class StrBlobPtr;

	typedef vector<string>::size_type size_type;//取别名
	//构造函数
	StrBlob();
	StrBlob(initializer_list<string> il);//使用不定数量的string对象用于初始化
	//成员函数
	size_type size() const { return (*data).size(); }
	//添加、删除元素的函数
	void push_back(const string& t) { data->push_back(t); }
	void pop_back();
	//元素访问
	string& front();
	string& back();
	const string& front() const;
	const string& back() const;


private:
	shared_ptr<vector<string>> data;
	//如果data[i]不合法，抛出一个异常
	void check(size_type i, const string& msg) const;
};
~~~

~~~cpp
//类内函数的定义
StrBlob::StrBlob(): data(make_shared<vector<string>>()){}

StrBlob::StrBlob(initializer_list<string> il):data(make_shared<vector<string>>(il)){}

void StrBlob::check(size_type i, const string& msg) const
{
	//调用shared_ptr指针指向的vector的自带函数size（），用形参i和size比较，判断是否抛出异常
	if (i >= data->size())
		throw out_of_range(msg);
}

void StrBlob::pop_back()
{
	//用check函数保证shared_ptr指针指向的vector里有元素
	//然后调用shared_ptr指针指向的vector的自带函数pop_back（）
	check(0, "pop_back on empty StrBlob");
	data->pop_back();
}

string& StrBlob::front()
{
	//用check函数保证shared_ptr指针指向的vector里有元素
	//然后调用shared_ptr指针指向的vector的自带函数front（）
	check(0, "front on empty StrBlob");
	return data->front();
}

const string& StrBlob::front() const
{
	//用check函数保证shared_ptr指针指向的vector里有元素
	//然后调用shared_ptr指针指向的vector的自带函数front（）
	check(0, "front on empty StrBlob");
	return data->front();
}

string& StrBlob::back()
{
	//用check函数保证shared_ptr指针指向的vector里有元素
	//然后调用shared_ptr指针指向的vector的自带函数back()
	check(0, "back on empty StrBlob");
	return data->back();
}

const string& StrBlob::back() const
{
	//用check函数保证shared_ptr指针指向的vector里有元素
	//然后调用shared_ptr指针指向的vector的自带函数back（）
	check(0, "back on empty StrBlob");
	return data->back();
}
~~~

让我们复盘一下这个程序：

1. StrBlob类被定义出来，有什么功能，存在的意义？
   - StrBlob是一部分的、不完全的vector。自带了几种不同功能的成员函数：push_back()、pop_back()、size()、front()、back()
   - 在我看来，StrBlob的存在是为了C++初学者能够更好的理解如何使用智能指针进行内存管理。其实分析一下StrBlob的源码不难发现，StrBlob类所谓的那些功能，都是借花献佛，调用vector的成员函数罢了。本质上还是一个vector，只不过是将vector放置在堆区，通过智能指针进行管理。这样做有一个好处，那就是在某个程序中，如果vector容器被多个对象共享的话，其实将vector换成StrBlob更安全，避免vector被意外销毁导致非法访问
