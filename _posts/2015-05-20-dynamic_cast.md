---
layout: post
title: 【C++】强制类型转换操作符 dynamic_cast
category: 技术
tags: C＋＋｜对象模型
description: 
---

**dynamic_cast** 是四个强制类型转换操作符中最特殊的一个，它支持运行时识别指针或引用。
#编译器的RTTI设置


**dynamic＿cast** 提供RTTI（Run-Time Type Information),也就是运行时类型识别。它对编译器有要求，需要编译器启动“运行时类型信息”这一选项。当编译器不开启RTTI时，运行含有dynamic_cast操作符的程序时会出现一个警告：
>warning C4541: “dynamic_cast”用在了带 /GR- 的多态类型“ANIMAL”上；可能导致不可预知的行为

VS2010在默认下是开启RTTI的，也可以自己手动去开启或者关闭，操作如下：
>视图->解决方案资源管理器

*<center><img src="/public/img/115.jpg" style="width:50%"></center>*


>在打开的解决方案管理器中，对着项目名称右击，选择属性

*<center><img src="/public/img/116.jpg" style="width:50%"></center>*
>配置属性-〉C/C++

*<center><img src="/public/img/117.png" style="width:50%"></center>*

#正文
##dynamic_cast主要用于“安全地向下转型”
**dynamic＿cast**用于类继承层次间的指针或引用转换。主要还是用于执行“安全的向下转型（safe downcasting）”，也即是基类对象的指针或引用转换为同一继承层次的其他指针或引用。至于“先上转型”（即派生类指针或引用类型转换为其基类类型），本身就是安全的，尽管可以使用dynamic＿cast进行转换，但这是没必要的， 普通的转换已经可以达到目的，毕竟使用dynamic＿cast是需要开销的。

<pre><code> 1 class Base
 2 {
 3 public:
 4     Base(){};
 5     virtual void Show(){cout<<"This is Base calss";}
 6 };
 7 class Derived:public Base
 8 {
 9 public:
10     Derived(){};
11     void Show(){cout<<"This is Derived class";}
12 };
13 int main()
14 {    
15     Base *base ;
16     Derived *der = new Derived;
17     //base = dynamic_cast<Base*>(der); //正确，但不必要。
18     base = der; //先上转换总是安全的
19     base->Show(); 
20     system("pause");
21 }
</code></pre>
##dynamic_cast与继承层次的指针

对于“向下转型”有两种情况。一种是基类指针所指对象是派生类类型的，这种转换是安全的；另一种是基类指针所指对象为基类类型，在这种情况下dynamic_cast在运行时做检查，转换失败，返回结果为0；
<pre><code>

＃include "stdafx.h"
＃include<iostream>
using namespace std;

class Base
{
public:
    Base(){};
    virtual void Show(){cout<<"This is Base calss";}
};
class Derived:public Base
{
public:
    Derived(){};
    void Show(){cout<<"This is Derived class";}
};
int main()
{    
    //这是第一种情况
    Base* base = new Derived;
    if(Derived *der= dynamic_cast<Derived*>(base))
    {
        cout<<"第一种情况转换成功"<<endl;
        der->Show();
        cout<<endl;
    }
    //这是第二种情况
    Base * base1 = new Base;
    if(Derived *der1 = dynamic_cast<Derived*>(base1))
    {
        cout<<"第二种情况转换成功"<<endl;
        der1->Show();
    }
    else 
    {
        cout<<"第二种情况转换失败"<<　endl;
    }
    delete(base);
    delete(base1);
    system("pause");
}
</code></pre>

运行结果：

*<center><img src="/public/img/118.jpg" style="width:50%"></center>*
## dynamic_cast和引用类型

在前面的例子中，使用了dynamic_cast将基类指针转换为派生类指针，也可以使用dynamic_cast将基类引用转换为派生类引用。
同样的，引用的向上转换总是安全的：
<pre><code>
Derived c;
Derived & der2= c;
Base & base2= dynamic_cast<Base&>(der2);//向上转换，安全
base2.Show();
</code></pre>
所以，在引用上，dynamic_cast依旧是常用于“安全的向下转型”。与指针一样，引用的向下转型也可以分为两种情况，与指针不同的是，并不存在空引用，所以引用的dynamic_cast检测失败时会抛出一个bad_cast异常：
<pre><code>
int main()
{    
   //第一种情况，转换成功
   Derived b ;
   Base &base1= b;
   Derived &der1 = dynamic_cast<Derived&>(base1);
   cout<<"第一种情况：";
   der1.Show();
   cout<<endl;

   //第二种情况
   Base a ;
   Base &base = a ;
   cout<<"第二种情况：";
   try{
      Derived & der = dynamic_cast<Derived&>(base);
   }
   catch(bad_cast)
   {
      cout<<"转化失败,抛出bad_cast异常"<< endl;
   }
   system("pause");
}
</code></pre>

运行结果：
*<center><img src="/public/img/119.jpg" style="width:50%"></center>*
##使用dynamic_cast转换的Base类至少带有一个虚函数
当一个类中拥有至少一个虚函数的时候，编译器会为该类构建出一个虚函数表（virtual method table），虚函数表记录了虚函数的地址。如果该类派生了其他子类，且子类定义并实现了基类的虚函数，那么虚函数表会将该函数指向新的地址。虚表是C++多态实现的一个重要手段，也是dynamic_cast操作符转换能够进行的前提条件。当类没有虚函数表的时候（也即一个虚函数都没有定义）,dynamic_cast无法使用RTTI，不能通过编译（个人猜想...有待验证）。

当然，虚函数表的建立对效率是有一定影响的，构建虚函数表、由表查询函数 都需要时间和空间上的消耗。所以，除了必须声明virtual（对于一个多态基类而言），不要轻易使用virtual函数。对于虚函数的进一步了解，可以查看《Effective C++》
