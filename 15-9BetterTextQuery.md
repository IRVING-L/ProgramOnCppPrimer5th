### 15.9 改进的文本查询程序

 *talk is cheap, show me the code*

~~~cpp
//抽象第一基类Query_Base、派生类WordQuery、派生类NotQuery、抽象第二基类BinaryQuery及其它的派生类AndQuery、派生类OrQuery、对外接口类Query的声明定义
//Query_Base类，这是一个抽象基类
class Query_Base
{
	friend class Query;
protected:
	//using line_no = TextQuery::line_no;
	virtual ~Query_Base() = default;
private:
	//eval返回与当前Query匹配的QUeryResult
	virtual QueryResult eval(const TextQuery&) const = 0;
	//rep是表示查询的一个string
	virtual string rep() const = 0;
};
//对外接口Query
class Query
{
	friend Query operator~(const Query&);
	friend Query operator&(const Query&, const Query&);
	friend Query operator|(const Query&, const Query&);
public:
	//构造函数
	Query() = default;
	Query(const string& s);
	//成员函数:做接口用，调用对应的Query_Base操作
	QueryResult eval(const TextQuery& t) const
	{
		return q->eval(t);
	}
	string rep() const { return q->rep(); }
private:
	shared_ptr<Query_Base> q;//存储
	Query(shared_ptr<Query_Base> query) :q(query) 
	{ cout << "Query(shared_ptr>)" << endl; }//私有的构造函数
};
//派生类WordQuery
class WordQuery :public Query_Base
{
	friend class Query;
	string query_word;//成员变量，私有的
	WordQuery(const string& s) : query_word(s) 
	{ cout << "WordQuery(stinrg)" << endl; }//私有构造函数
	//成员函数
	QueryResult eval(const TextQuery& t) const override
	{
		return t.query(query_word);
	}
	string rep() const override { return query_word; }
};
//派生类NotQuery
class NotQuery :public Query_Base
{
	friend Query operator~(const Query&);//友元函数
	Query query;//成员变量query

	NotQuery(const Query& q) :query(q) {}
	QueryResult eval(const TextQuery& t) const override;
	string rep() const override { return "~(" + query.rep() + ")"; }
};
//抽象基类BinaryQuery
class BinaryQuery :public Query_Base
{
protected:
	BinaryQuery(const Query& l, const Query& r, string s) :
		lhs(l), rhs(r), opSym(s) { cout << "BianryQuery()" << endl; }
	string rep() const override
	{
		return "(" + lhs.rep() + " " + opSym + " " + rhs.rep() + ")";
	}
	Query lhs, rhs;//左侧和右侧运算对象
	string opSym;//运算符名称
};
//抽象基类BinaryQuery的派生类AndQuery
class AndQuery:public BinaryQuery
{
	friend Query operator&(const Query&, const Query&);
	//构造函数
	AndQuery(const Query& left, const Query& right) :
		BinaryQuery(left, right, "&") { cout << "AndQuery()" << endl; }
	//具体的类：AndQuery继承了rep并且定义了其他纯虚函数
	QueryResult eval(const TextQuery&) const override;
};
//抽象基类BinaryQuery的派生类OrQuery
class OrQuery :public BinaryQuery
{
	friend Query operator|(const Query&, const Query&);
	OrQuery(const Query& left, const Query& right) :
		BinaryQuery(left, right, "|") { cout << "OrQuery()" << endl; }
	QueryResult eval(const TextQuery&) const override;
};
~~~

~~~cpp
#include "BetterTextQuery.h"

Query::Query(const std::string& s) : q(new WordQuery(s)) { cout << "Query(string)" << endl; }
//重载操作运算符：非
Query operator~(const Query& operand)
{
	//return 之后发生一个隐式转换，把下面的临时变量转换成Query类型的临时变量
	return shared_ptr<Query_Base>(new NotQuery(operand));
}
//重载操作运算符：与
Query operator&(const Query& lhs, const Query& rhs)
{
	return shared_ptr<Query_Base>(new AndQuery(lhs, rhs));
}
//重载操作运算符：或
Query operator|(const Query& lhs, const Query& rhs)
{
	return shared_ptr<Query_Base>(new OrQuery(lhs, rhs));
}
//OrQuery的eval函数定义
QueryResult OrQuery::eval(const TextQuery& t) const
{
	//right和left 的数据类型为自定义的QueryResult，
	//是用数据类型为TextQuery的t中的文本文档中查找单词s出现的行数
	auto right = rhs.eval(t), left = lhs.eval(t);
	//ret_lines是一个指针shared_ptr<set<int>>，它指向一片保存有left中的set容器拷贝元素的新空间
	auto ret_lines = make_shared<set<int>>(left.begin(), left.end());
	//由于OrQuery是求并集，直接将两个单词求得的set相加即可
	ret_lines->insert(right.begin(), right.end());
	//还记得QueryResult的构造函数接受的形参是什么嘛？
	//QueryResult（const string& s, shared_ptr<set<int>> p, shared_ptr<vector<string>> f)
	return QueryResult(rep(), ret_lines, left.get_file());
}
//AndQuery的eval函数定义
QueryResult AndQuery::eval(const TextQuery& t) const
{
	//right和left 的数据类型为自定义的QueryResult，
	//是用数据类型为TextQuery的t中的文本文档中查找单词s出现的行数
	auto right = rhs.eval(t), left = lhs.eval(t);
	//ret_lines是一个指针shared_ptr<set<int>>，它指向一片保存有left中的set容器拷贝元素的新空间
	auto ret_lines = make_shared<set<int>>();
	//调用标准库函数将两个查询结果求交集
	set_intersection(left.begin(), left.end(), right.begin(), right.end(),
		inserter(*ret_lines, ret_lines->begin()));
	return QueryResult(rep(), ret_lines, left.get_file());
}
//NotQuery的eval函数定义
QueryResult NotQuery::eval(const TextQuery& t) const
{
	//result是查询结果，这个没问题
	auto result = query.eval(t);
	//ret_lines是一个智能指针shared_ptr，指向一片空的内存空间
	auto ret_lines = make_shared<set<int>>();
	//beg和end是迭代器，指向查询单词出现行数的set容器的迭代器
	auto beg = result.begin();
	auto end = result.end();
	//调用了保存文本文档vector容器的自带函数size(), 求出这个文本总共有多少行
	auto sz = result.get_file()->size();
	for (int i = 0; i < sz; ++i)
	{
		/*
		这个循环做了什么？
		循环了sz次，把查询单词未出现的行数i保存在ret_lines指向的set容器中
		*/
		if (beg == end || *beg != i)
			ret_lines->insert(i);
		else if (beg != end)
			++beg;
	}
	return QueryResult(rep(), ret_lines, result.get_file());
}
~~~

让我们复盘一下这个程序设计：

**项目需求**

对第12章设计的文本查询程序进行改进：原来的程序只能输入一个单词，查询其出现的行数并打印该行。改进的程序要能实现对多个单词进行查询。多个单词的查询结果可以是并集或者交集，甚至能对单词的查询结果进行取反。

**用到的技术**

1. 面向对象的三大特性。
   - 封装：这个早在第一版的文本查询程序中就已经使用了，QueryResult类就是一个对变量的封装，第二版的对外接口Query也是一个封装。
   - 继承：不多说，自己看。继承关系一下子就看出来了。
   - 多态：什么是多态？用基类的引用或者指针指向派生类的对象，这个引用或者指针是动态绑定在基类或者派生类对象上的，其实际指向在函数运行时才能决定，这就是多态。第二版的文本查询程序中，Query对外接口类中就使用到了多态。Query类中定义了一个私有成员变量```q```，q是一个指向Query_Base基类对象的智能指针，在Query类的成员函数eval中，调用了```q->eval(args);```，eval函数到底是基类Query_Base对象还是它的几个派生类对象的eval函数，由程序运行时动态决定。
2. 动态内存管理（使用智能指针）：第一版的程序用智能指针管理了set和vector容器，第二版的程序增加了一个指向基类Query_Base的智能指针，从而实现多态。
3. 运算符重载：关于运算符重载的知识我不是很了解，《C++Primer》14章没看。
4. 标准库函数：在派生类AndQuery的eval函数中，调用了set_insection函数求两个set容器的交集，简化了程序设计。

这个程序设计应该是目前为止最为复杂的了，它把第12章内存管理（智能指针）、第14章运算符重载、第15章OOP设计的重要知识点都涉及到了。不仅知识点多，由于类的数量较多，这个逻辑关系也比较复杂，尤其是函数的调用，你调用它，它调用其他。后面有时间我会绘制一个图来说明这个程序设计中各个类相互之间的操作，便于读者阅读这个程序。
