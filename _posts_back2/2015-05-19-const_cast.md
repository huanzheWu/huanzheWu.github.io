---
layout: post
title: 【C++】强制类型转换操作符 const_cast
category: 技术
tags: C＋＋｜对象模型
description: 
---

const_cast也是一个强制类型转换操作符。《C++ Primer》中是这样描述它的：

1.将转换掉表达式的const性质。

2.只有使用const_cast才能将const性质性质转化掉。试图使用其他三种形式的强制转换都会导致编译时的错误。（添加const还可以用其他转换符，如static＿cast）

3.除了添加const或删除const特性，使用const_cast符来执行其他任何类型的转换都会引起编译错误。（volatile限定符也包括，不过我不怎么了解，本文主要说const）

对于第一点，转换掉表达式的const性质，意思是可以改变const对象的值了吗？一开始我的确是这样子认为的，于是我敲出了如下的代码：

<pre><code>
int main()
{
    const int constant = 26;
    const int* const_p = &constant;
    int* modifier = const_cast<int*>(const_p);
     *modifier = 3;
    cout<< "constant:  "<< constant<<endl;
    cout<<"*modifier:  "<< *modifier<<endl;
    system("pause");
}
</code></pre>

然而程序并没有像预想的那样输出两个3，运行结果是这样的：
*<center><img src="/public/img/120.jpg" style="width:50%"></center>*

看来C++还是很厚道的，对声明为const的变量来说，常量就是常量，任你各种转化，常量的值就是不会变。这是C++的一个承诺。那既然const变量的值是肯定不会发生变化的，还需要这个const_cast类型转化有何用？这就引出了const_cast的最常用用法：如果有一个函数，它的形参是non-const类型变量，而且函数不会对实参的值进行改动，这时我们可以使用类型为const的变量来调用函数，此时const_cast就派上用场了。
例如：

<pre><code>
void InputInt(int * num)
{
    cout<<*num<<endl;
}
int main()
{
    const int constant = 21;
    //InputInt(constant); //error C2664: “InputInt”: 不能将参数 1 从“const int”转换为“int *”
    InputInt(const_cast<int*>(&constant));
    system("pause");
}
</code></pre>

除此之外，还有另外一种情况const指针能够派上用场。如果我们定义了一个非const的变量，却使用了一个指向const值的指针来指向它（这不是没事找事嘛），在程序的某处我们想改变这个变量的值了，但手头只持有指针，这是const_cast就可以用到了：


<pre><code>
int main()
{
    int constant = 26;
    const int* const_p = &constant;
    int* modifier = const_cast<int*>(const_p);
    *modifier = 3;
    cout<< "constant:  "<<constant<< endl;
    cout<< "*modifier:  "<< *modifier<< endl;

    system("pause");
}
</code></pre>

总结一下上文：const_cast绝对不是为了改变const变量的值而设计的！
在函数参数的传递上const_cast的作用才显现出来。

###const_cast中的未定义行为
上面的第一段程序，输出变量constant与*modefier的地址后....


<pre><code>
int main()
{
    const int constant = 26;
    const int* const_p = &constant;
    int* modifier = const_cast<int*>(const_p);
    *modifier = 3;
    cout<< "constant:  "<<constant<<"  adderss:  "<< &constant <<endl;
    cout<<"*modifier:  "<<*modifier<<"  adderss:  " << modifier<<endl;

    system("pause");
}
</code></pre>

运行结果：

*<center><img src="/public/img/121.jpg" style="width:50%"></center>*

它们的地址是一样的，值却不同。具体原因我还是不大清除。在另外一些博客中看到， *modifier = 3; 这种操作属于一种“未定义行为”，也即是说操作结果C++并没有明确地定义，结果是怎样的完全由编译器的心情决定。对于未定义的行为，我们只能避免之。

####关于const_cast是否安全的讨论
逛了一些网站，大致有如下观点：

>const_cast is safe only if you're casting a variable that was originally non-const. For example, if you have a function that takes a parameter of a const char *, and you pass in a modifiable char *, it's safe to const_cast that parameter back to a char * and modify it. However, if the original variable was in fact const, then using const_cast will result in undefined behavior.

也即是上文中所说的const_cast的二种适用情况。

>I would rather use static cast for the adding constness: static_cast<const sample*>(this). When I'm reading const_cast it means that the code is doing something potentially dangerous, so i try to avoid it's use when possible.

也有人认为const＿cast本身就给潜在危险带来可能，所以还是尽可能不用它了。
当需要给变量添加const属性时，使用更为安全的static＿cast来代替const_cast。

这里附上讨论链接。[const_cast是否安全？](http://stackoverflow.com/questions/357600/is-const-cast-safe)