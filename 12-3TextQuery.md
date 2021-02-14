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

~~~

