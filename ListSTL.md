## c++实现一个自己的迭代器和ListSTL

### 相关知识：

#### 1.模板

不用了解太深，通过传入参数的类型确认类、函数、对象的类型，提高代码的可读性和可复用性。

#### 2.重载运算符

要全部看完，重载迭代器中的各种运算符，注意不要出现二义性，哪些必须是类的成员，哪些需要const限制，何时为引用。

#### 3.类

要熟悉基本语法，了解构造函数等。



### 遇到的问题以及解决

#### 迭代器：

##### 1：

```c++
    iterator_ &operator--()
        {
            data_=data_->prev_; 
            return *this;
        }
```

情况：我最开始写的是双向链表，end是空指针无法--。

解决：接着写双向链表的话，就得维护一个尾（无有效数据）节点，另一种方法是改成双向循环链表，end（）返回head_ ，begin（）返回head_->next\_。（改了我40分钟 www）

##### 2.

```cpp
        bool operator==(const iterator_ &t)  
    {
            //return data_==t.data_;?
            if(t.data_==nullptr||data_==nullptr){
                return data_==t.data_;//*this==t错误 因为两个iterator_比较会再次 重载== 导致无限递归(点)
            } 
            else
            return data_->prev_==t.data_->prev_&&
            data_->next_==t.data_->next_&&
            data_->data_==t.data_->data_;//应该把这一段给node的重载==运算符
     }
```

​            if(t.data_==nullptr||data_==nullptr){

​                return *this==t//段错误

​            } 

情况： 因为两个iterator_比较会再次 重载== 导致无限递归。

解决：改成双向循环链表后不用判断，直接return data_==t.data\_就可以。

#### List：

##### 1.

```cpp
   class list{
	private:
        //可以添加你需要的成员变量
        node_ *head_=&head1; //头节点,哨兵（不存储有效数据） 
        node_ head1;
       ......
   }
```

在改为双向循环链表后要给头结点分配空间。初始化时head_->prev\_=head\_->next\_=head\_;

##### 2.

```cpp
        // //重载==
        // bool operator==(const list<T> &t)
        // {
        //    return t.head_==head_;
        // }
        // //重载！=
        // bool operator!=(const list<T> &t)
        // {
        //     return !(*this==t);
        // }        
		
		list<T> &operator=(const list<T> &lt)
        {
            if(&lt!=this){
            //先clear
            clear();
            node_* a=lt.head_->next_;
            while (a!=lt.head_)
            {
                push_back(a->data_);
                a=a->next_;
            }
            }
            return *this;
        }
```

情况：lt!=\*this与声明的重载！=运算符函数的类型不匹配。因为重载运算符的第一个参数（隐式）是非const的而lt是const的，所以报错

解决：1.const_cast<list\<T>&>(lt)强行转换2.将两个参数反过来*this==lt3.比较两个对象的地址，这样也不用重载运算符了(但应该还是比较两个对象更不容易出错)



注：this是一个对象的常量指针,调用成员函数应为claer()（不必用（*this）.clear();或 this->clear() ）。

（当我们调用一个成员函数时，用请求该函数的对象地址初始化 this。）

(this一般用于重名变量和返回自身对象)



#### listSTL.hpp:

```c++
#include <iostream>
//使用你的名字替换YOUR_NAME
//#include<gtest/gtest.h>
namespace YOUR_NAME
{
    template <class T>
    // list存储的节点
    // 可以根据你的需要添加适当的成员函数与成员变量
    struct node
    {
        node<T> *prev_;
        node<T> *next_;
        T data_;
        //构造函数
        node(const T &data)
            : data_(data), prev_(nullptr), next_(nullptr)
        {
        }
        node()
            : prev_(nullptr), next_(nullptr)
        {
        }
        //稀构函数
        ~node()
        {
            //delete data_;
        }
    };

    template <class T>
    //迭代器类,(类比指针)

    struct iterator
    {
        typedef node<T> node_;
        typedef iterator<T> iterator_;
        node_ *data_;
        iterator()
        {
        }
        iterator(node_ *data)
            : data_(data)
        {
        }
        ~iterator()
        {
            
        }
        //迭代到下一个节点
        //++it
        iterator_ &operator++()
        {
            data_=data_->next_;
            return *this;
        }
        //迭代到前一个节点
        //--it
        iterator_ &operator--()
        {
            data_=data_->prev_;//要不维护一个尾（空）节点，要不做双向链表(点) 情况：end是空指针无法--
            return *this;
        }
        // it++
        iterator_ operator++(int)
        {
            iterator_ ret=*this;
            ++*this;
            return ret;
        }
        //it--
        iterator_ operator--(int)
        {
            iterator_ ret=*this;
            --*this;
            return ret;
        }
        //获得迭代器的值
        T &operator*()
        {
            return this->data_->data_;
        }
        //获得迭代器对应的指针
        T *operator->()
        {
            return &(this->operator*());
        }
        //重载==
        bool operator==(const iterator_ &t)
        {
            return data_==t.data_;
            // if(t.data_==nullptr||data_==nullptr){
            //     return data_==t.data_;//*this==t错误 因为两个iterator_比较会再次 重载== 导致无限递归(点)
            // } 
            // else
            // return data_->prev_==t.data_->prev_&&
            // data_->next_==t.data_->next_&&
            // data_->data_==t.data_->data_;
        }
        //重载！=
        bool operator!=(const iterator_ &t)
        {
            return !(*this==t);
        }

        //**可以添加需要的成员变量与成员函数**
    };

    template <class T>
    class list
    {
    public:
        typedef node<T> node_;
        typedef iterator<T> Iterator;

    private:
        //可以添加你需要的成员变量
        node_ *head_=&head1; //头节点,哨兵（不存储有效数据） 
        node_ head1;//要给头结点分配空间（点）
    public:
        //构造函数
        list()
        {
            //head_->data_=0;
            head_->prev_=nullptr;//应该改为指向自己，把之后的检测为空的条件修改为(head_->next_==head_->prev_&&head_->prev_==head_)
            head_->next_=nullptr;
        }
        //拷贝构造，要求实现深拷贝
        list(const list<T> &lt)//:head_(lt.head_),tail_(lt.tail_)//不能共用同一块内存
        {
            node_* a=lt.head_->next_;
            while (a!=lt.head_)
            {
                push_back(a->data_);
                a=a->next_;
            }
        }
        template <class inputIterator>
        //迭代器构造
        list(inputIterator begin, inputIterator end)
        {
            node_* a=begin.data_;
            while (a!=end.data_)
            {
                push_back(a->data_);
                a=a->next_;
            }
        }

        ~list()
        {
        }

        // //重载==
        // bool operator==(const list<T> &t)
        // {
        //    return t.head_==head_;
        // }
        // //重载！=
        // bool operator!=(const list<T> &t)
        // {
        //     return !(*this==t);
        // }

        //拷贝赋值
        list<T> &operator=(const list<T> &lt)
        {
            if(&lt!=this){//(点)lt!=*this因为重载运算符的第一个参数（隐式）是非const的而lt是const的，所以报错//解决方法：1.const_cast<list<T>&>(lt)强行转换2.将两个参数反过来*this==lt3.比较两个对象的地址，这样也不用重载运算符了(但应该还是比较两个对象更不容易出错)
            //先clear
            clear();//(点)this是对象指针,调用成员函数应为（*this）.clear();或 this->clear(); this是一个常量指针//当我们调用一个成员函数时，用请求该函数的对象地址初始化 this。
            node_* a=lt.head_->next_;
            while (a!=lt.head_)
            {
                push_back(a->data_);
                a=a->next_;
            }
            }
            return *this;
        }
        //清空list中的数据
        void clear()
        {
            node_ *i=head_->next_;
            while(head_!=i){
                i=i->next_;
                delete i->prev_;
            }
            head_->next_=head_->prev_=nullptr;
        }
        //返回容器中存储的有效节点个数
        size_t size() const
        {
            node_ *now=head_;
            size_t a=0;
            if(head_->prev_==nullptr||head_->next_==nullptr) return 0;
            while(now->next_!=head_){
                now=now->next_;
                a++;
            }
            return a;
        }
        //判断是否为空
        bool empty() const
        {
            return head_->next_==nullptr||head_->prev_==nullptr;
        }
        //尾插
        void push_back(const T &data)
        {
            if(head_->prev_==nullptr||head_->next_==nullptr){
                head_->next_=new node(data);
                head_->prev_=head_->next_;
                head_->next_->next_=head_;
                head_->prev_->prev_=head_;
            } 
            else {
                head_->prev_->next_=new node(data);
                head_->prev_->next_->prev_=head_->prev_;
                head_->prev_=head_->prev_->next_;
                head_->prev_->next_=head_;
            }
        }
        //头插
        void push_front(const T &data)
        {
            //Iterator pos=head_;
            insert(head_,data,true);
        }
        //尾删
        void pop_back()
        {
            head_->prev_=head_->prev_->prev_;
            delete head_->prev_->next_;
            head_->prev_->next_=head_;
        }
        //头删
        void pop_front()
        {
            head_->next_=head_->next_->next_;
            delete head_->next_->prev_;
            head_->next_->prev_=head_;
        }
        //默认新数据添加到pos迭代器的后面,根据back的方向决定插入在pos的前面还是后面
        void insert(Iterator pos, const T &data, bool back = true)
        {
            node_* a=new node_(data);
            if(back==false){//前
                pos.data_->prev_->next_=a;
                a->prev_=pos.data_->prev_;
                pos.data_->prev_=a;
                a->next_=pos.data_;
            }
            else{//后
                if(pos.data_->next_==nullptr||pos.data_->prev_==nullptr){
                    head_->next_=a;
                    a->prev_=head_;
                    a->next_=head_;
                    head_->prev_=a;
                }
                else{
                    a->next_=pos.data_->next_;
                    pos.data_->next_->prev_=a;
                    pos.data_->next_=a;
                    a->prev_=pos.data_;
                }
            }
        }
        //删除pos位置的元素
        void erase(Iterator pos)
        {
            if(pos.data_==head_) return ;
            else{
                pos.data_->prev_->next_=pos.data_->next_;
                pos.data_->next_->prev_=pos.data_->prev_;
                delete pos.data_;            
            }

        }

        //获得list第一个有效节点的迭代器
        Iterator begin() const
        {
            if(head_->next_==nullptr) return head_;
            else return head_->next_;
        }

        //获得list最后一个节点的下一个位置，可以理解为nullptr
        Iterator end() const
        {
            return head_;
        }
        //查找data对应的迭代器
        Iterator find(const T &data) const
        {
            node_ *now=head_->next_;
            
            while(now!=head_){
                if(now->data_==data){
                    //Iterator res(now);
                    return now;
                }
                now=now->next_;
            }
            return nullptr;
        }
        //获得第一个有效节点
        node_ front() const
        {
            return head_->next_;
        }
        //获得最后一个有效节点
        node_ back() const
        {
            return head_->prev_;
        }

    private:
        //可以添加你需要的成员函数
    };
};
```



#### 测试函数 test.cpp

```c++
#include "listSTL.hpp"
using namespace YOUR_NAME;
using namespace std;
struct D
{
    int x;
    std::string y;
    float z;
    D(int x_ = 1, std::string y_ = "hello", float z_ = 3.14)
        : x(x_), y(y_), z(z_)
    {
    }

    bool operator==(const D& other) const
    {
        return x == other.x && y == other.y&& z==other.z;
    }
};
void test1()
{
    list<int> l1;

    l1.push_back(1);
    l1.push_back(2);
    l1.push_back(3);
    l1.push_front(4);
    list<int>::Iterator it = l1.begin();
    while (it != l1.end())
    {
        std::cout << (*it) << std::endl;
        ++it;
    }
    l1.clear();
    if (l1.empty())
    {
        std::cout << "empty() ok" << std::endl;
    }
}
void test2()
{
    list<D> l1;
    l1.push_back(D(1, "aaa", 1.1));
    l1.push_back(D(2, "bbb", 2.2));
    l1.pop_front();
    auto it = l1.begin();
    while (it != l1.end())
    {
        std::cout << it->x << " " << it->y << " " << it->z << std::endl;
        ++it;
    }
}
void test3()
{
    list<int> l1;
    l1.push_back(1);
    l1.push_back(2);
    l1.push_back(3);
    l1.push_back(4);
    list<int>::Iterator pos = l1.find(2);
    if (pos != l1.end())
    {
        l1.insert(pos, 5, false); //插入到前面
    }
    if (l1.size() == 5)
    {
        std::cout << "success" << std::endl;
    }
    auto it = l1.begin();
    while (it != l1.end())
    {
        std::cout << (*it) << " ";
        ++it;
    }
    //输出1 5 2 3 4
}
void test4()
{
    list<int> l1;
    l1.push_back(1);
    l1.push_back(2);
    l1.push_back(3);
    l1.push_back(4);
    list<int>::Iterator pos = l1.find(2);
    if (pos != l1.end())
    {
        l1.erase(pos);
    }
    list<int> l2(l1);
    l2.push_back(3);
    //l2此时应该为1 3 4 3
    //l1此时应该为1 3 4
    if (l1.size() == l2.size())
    {
        std::cout << "error" << std::endl;
    }
}
void test5()
{
    list<int> l1;
    l1.push_back(1);
    l1.push_back(2);
    l1.push_back(3);
    l1.push_back(4);
    auto pos=l1.find(3);
    list<int> l2(l1.begin(),pos);
    //l2中应该为1 2
    if(l2.size()!=2)
    {
        std::cout<<"error"<<std::endl;
    }

}

// 测试基本功能以及迭代器的操作
void test6()
{
    list<int> l;
    l.push_back(2);
    l.push_back(3);
    l.push_back(4);
    l.push_front(1);

    for (auto i = l.begin(); i != l.end(); ++i)
    {
        cout << *i << " ";
    }

    cout << endl;

    l.pop_front();

    for (auto i = l.begin(); i != l.end(); ++i)
    {
        cout << *i << " ";
    }

    cout << endl;

    auto it = l.find(3);
    l.insert(it, 10, false);

    for (auto i = l.begin(); i != l.end(); ++i)
    {
        cout << *i << " ";
    }

    cout << endl;

    l.erase(l.begin());//有问题
    l.erase(--l.end());

    for (auto i = l.begin(); i != l.end(); ++i)
    {
        cout << *i << " ";
    }

    cout << endl;
}

// 测试深拷贝
void test7()
{
    list<int> l1;
    l1.push_back(1);
    l1.push_back(2);
    l1.push_back(3);

    list<int> l2 = l1; // 测试拷贝构造

    l2.push_back(4);
    l2.push_back(5);

    for (auto i = l1.begin(); i != l1.end(); ++i)
    {
        cout << *i << " ";
    }

    cout << endl;

    for (auto i = l2.begin(); i != l2.end(); ++i)
    {
        cout << *i << " ";
    }

    cout << endl;

    list<int> l3;
    l3.push_back(6);
    l3.push_back(7);

    l3 = l2; // 测试拷贝赋值
//我追加了？？
    for (auto i = l2.begin(); i != l2.end(); ++i)
    {
        cout << *i << " ";
    }

    cout << endl;

    for (auto i = l3.begin(); i != l3.end(); ++i)
    {
        cout << *i << " ";
    }

    cout << endl;
}

// 测试空list
void test8()
{
    list<int> l;
    cout << l.empty() << endl; // 应输出1，即true
    l.push_back(1);
    cout << l.empty() << endl; // 应输出0，即false
}

// 测试边界条件下的操作
void test9()
{
    list<int> l;

    cout << (l.begin() == l.end()) << endl; // 应输出1，即true

    l.push_back(1);
    l.erase(l.begin());

    cout << (l.begin() == l.end()) << endl; // 应输出1，即true

    l.push_back(1);
    l.pop_front();

    cout << (l.begin() == l.end()) << endl; // 应输出1，即true

    auto it = l.begin();
    ++it;
    l.erase(it);

    cout << (l.begin() == l.end()) << endl; // 应输出1，即true
}

// 测试自定义类型
struct Person
{
    int age;
    string name;
    // Person(int a,string b):age(a),name(b)
    // {

    // }

    bool operator==(const Person &other)
    {
        return age == other.age && name == other.name;
    }
};

void test10()
{
    list<Person> l;

    l.push_back({18, "Tom"});
    l.push_back({19, "Jerry"});

    auto it = l.find({18, "Tom"});

    if (it != l.end())
    {
        cout << it->name << endl; // 应输出Tom
    }

    l.clear();

    cout << l.empty() << endl; // 应输出1，即true
}

int main()
{
    test1();
    cout<<"1完成"<<endl;
    test2();
    cout<<"2完成"<<endl;
    test3();
    cout<<"3完成"<<endl;
    test4();
    cout<<"4完成"<<endl;
    test5();
    cout<<"5完成"<<endl;
    test6();
    cout<<"6完成"<<endl;
    test7();
    cout<<"7完成"<<endl;
    test8();
    cout<<"8完成"<<endl;
    test9();
    cout<<"9完成"<<endl;
    test10();
    cout<<"10完成"<<endl;

    return 0;
}
```

