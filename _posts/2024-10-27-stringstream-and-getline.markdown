---
layout: post
title:  "stringstream与getline"
date:   2024-06-24 23:14:34 +0800
categories: jekyll update
---
在读spdlog代码的时候，发现了getline和stringstream结合使用的用法，可以用来按指定字符分割字符串。

先看一下cppreference中对getline的描述：

{% highlight c++ %}
template< class CharT, class Traits, class Allocator >
std::basic_istream<CharT, Traits>&
    getline( std::basic_istream<CharT, Traits>& input,
             std::basic_string<CharT, Traits, Allocator>& str, CharT delim );

template< class CharT, class Traits, class Allocator >
std::basic_istream<CharT, Traits>&
    getline( std::basic_istream<CharT, Traits>&& input,
             std::basic_string<CharT, Traits, Allocator>& str, CharT delim );

template< class CharT, class Traits, class Allocator >
std::basic_istream<CharT, Traits>&
    getline( std::basic_istream<CharT, Traits>& input,
             std::basic_string<CharT, Traits, Allocator>& str );

template< class CharT, class Traits, class Allocator >
std::basic_istream<CharT, Traits>&
    getline( std::basic_istream<CharT, Traits>&& input,
             std::basic_string<CharT, Traits, Allocator>& str );
{% endhighlight %}

本文关注前两种用法，这两个接口的差异只有input这个参数，第一个接口是左值，第二个接口是右值。

以第一个接口为例：

这个接口会执行如下操作：
1. 调用str.erase()
2. 从input中取出字符并放到str中（append），直到遇到delim指定的分隔符或者input的结束符号

下面是一个例子：

{% highlight c++ %}
#include <iostream>
#include <sstream>
#include <string>
using namespace std;

int main()
{
    string sstr = "A=3;B=4;C=5";
    string str;
    istringstream input(sstr);

    while (getline(input, str, ';')) {
        cout << str << endl;
    }

    return 0;
}
{% endhighlight %}

执行结果如下：
<img src="https://smileshineme.github.io//images/20241017/res.png">
