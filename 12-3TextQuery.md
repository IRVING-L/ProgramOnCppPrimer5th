## 《C++ Primer 5th》中值得记录的程序设计题

### 12.3 使用标准库：文本查询程序

*talk is cheap, show me the code*

~~~cpp 
//TextQuery类
class TextQuery
{
public:
    //构造函数
	TextQuery() = default;
	TextQuery(ifstream&);
    //成员函数
    QueryResult query(const string&) const;
private:
    //用一个指向vector<string>的智能指针，保存文件中的文本内容
	shared_ptr<vector<string>> file;
    //用一个map保存文本中每一个单词，出现的行数，因为一个单词的行数不止一个，因此是使用set容器来记录行数
	map < string, shared_ptr<set<int>> > wm;
};
~~~

~~~cpp
//类TextQuery的构造函数
//注意这里使用了构造函数的列表初始化，直接给定义的智能指针对象file绑定了具体的对象
//不过这里我有一个疑问：智能指针和new一起使用的话，只允许使用直接初始化：shaerd_ptr<T> p(new T);
//虽然列表初始化的形式看上去很像直接初始化，但是这个变量我们已经定义了啊，这样的操作只能叫做赋值啊，不能叫做初始化撒？难不成赋初值==初始化？？
TextQuery::TextQuery(ifstream& infile):file(new vector<string>)
{
	string text;
    /*
    infile是一个ifstream的引用对象，直接就可以当做一个输入流来使用
    我自己对这个infile变量的使用就很拉胯：
    infile.open("data.txt")
    string line;
    while(getline(cin,line)){}
    我犯了两个错误，一是没有理解到infile这个数据类型的本质是一个文件流，在传入构造函数之前就已经是指定了锷对应的打开文件，不需要在构造函数内部再次指定打开文件
    二是把文件读取弄成了黑框标准输入读取，严重错误
    */
	while (getline(infile,text))
	{
        /*
        file变量是什么数据类型-->shared_ptr<vector<string>>，它是一个智能指针，指向放在堆区的vector<string>对象
        file->push_back(text)等价于(*file).push_back(text)
        */
		file->push_back(text);
        //n代表当前文件的行数。为什么n是file的size-1，我觉得n应该是等于file的size呀
		int n = file->size() - 1;
        //创建一个string流，string流可以把一行文字转换成一个个单词
		istringstream line(text);
		string word;
		while (line >> word)
		{
			//lines是一个智能指针shared_ptr<set<int>>
			//这里利用了map下标访问的特殊性质，如果使用map中不存在的关键字访问map
			//map将会自动以该关键字创建一个pair元素，其中pair.second会被默认初始化为0
			//同时用引用将wm[word]的返回值和lines绑定在一起了，操作lines就是操作wm[word]
			shared_ptr<set<int>>& lines = wm[word];
			if (!lines)//如果是第一次遇到这个单词，那么智能指针为空
			{
                //给智能指针重新绑定对象。调用reset命令，可以让指针指向放在堆区的set<int>对象
				lines.reset(new set<int>);
			}
            //如果这个单词word已经存在与map中了，我们只需要在map里的set对象中，增加当前行数即可
            //如果某个单词但同一行中出现了n次，inset会被调用n次，但是insert真正执行只有第一次
			lines->insert(n);
		}
	}
}

//成员函数query
QueryResult TextQuery::query(const string& s) const
{
	//函数的功能：给定一个string对象，查找其对应的行号
	//由于该函数的返回对象是一个类，如果map中没找到该单词应该怎么办？
	//使用一个static局部对象，static的保存地址是全局变量区，要整个程序结束才会消失
	static shared_ptr<set<int>> nodata(new set<int>);
	auto loc = wm.find(s);//loc是一个map的迭代器
	if (loc == wm.end())
		return QueryResult(s, nodata, file);//没找到
	else
		return QueryResult(s, loc->second, file);//找到
}
~~~

~~~cpp
//QueryResult类
class QueryResult
{
	friend ostream& print(ostream& out, const QueryResult&);
public:
	QueryResult() = default;
	QueryResult(string s, shared_ptr<set<int>> p, shared_ptr<vector<string>> f)
		:sought(s), lines(p), file(f) {}
private:
	string sought;//查询单词
	shared_ptr<set<int>> lines;//出现的行号
	shared_ptr<vector<string>> file;//输入文件
};
~~~

~~~cpp
//QueryResult的友元函数print
ostream& print(ostream& out, const QueryResult& q)
{
	//elements occurs x time(s)
	out << q.sought << "occurs" << q.lines->size() << "time(s)" << endl;
	//打印单词的每一行
	for (auto c : *(q.lines))
	{
		out << "\t(line " << c + 1 << " )"
			<< *(q.file->begin() + c) << endl;
	}
	return out;
}
~~~

***

让我们复盘一下这个程序设计

**项目需求**

文本查询：在一个文本文档中查询某个单词出现的次数及其所在行数。如果某个单词在一行中出现多次，此行只列出一次。

**用了哪些技术**

1. STL容器：使用了vector将文本文档中的内容拷贝在程序中，方便指定内容打印；使用map关联容器记录文本文档中每一个单词，并将map的value设置为set容器，方便记录每一个单词出现的行数

2. 智能指针shared_ptr：本次程序设计中在两个地方使用了智能指针：用于保存文本内容的vector的内存空间是由智能指针管理的动态内存上的；用于记录每个单词行数的set容器的内存空间也是由一个shared_ptr管理的。

   ==为什么要把容器放在堆区？为什么要使用智能指针管理容器？==

   ​		从我的角度来看，整个程序设计，最令人觉得奇妙的就是使用智能指针来管理内存了。要回答容器放在堆区这个问题，我们首先要先知道为什么需要智能指针？这个程序设计中，打包了两个类，一个类TextQuery负责读取文本文档，记录单词出现的相关信息，另一个类QueryResult负责将记录的相关信息打印输出。我们发现，所有的数据都会保存在类TextQuery类的对象中，另一个类QueryResult的对象怎么样才能得到这些数据呢？

   ​		为了解决这个问题，该程序是将TextQuery对象的数据信息，拷贝一份给QueryResult对象。可是这里有两个问题，第一：我们不想拷贝一个vector或者map，数据量太多了，拷贝很花时间。于是乎，我们就会想到，那就把数据的指针传过去，两个类对象操作同一个地址的内存上的数据。传递指针固然方便，大大降低了时间，但是这就带来了第二个问题：如果TextQuery对象在程序运行中，被销毁了怎么办，那么我在QueryResult对象中再去操作这片内存空间不就是非法访问了？为了解决这个问题，我们很自然的就会想到使用智能指针。既然使用智能指针管理内存了，那么vector这些容器自然而然也就只能放在堆区了。

3. string流处理string对象。
