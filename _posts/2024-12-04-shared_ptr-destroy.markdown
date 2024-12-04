---
layout: post
title:  "shared_ptr对象析构"
date:   2024-12-04 20:00:00 +0800
categories: jekyll update
---
考虑下面的代码，输出是什么
```
#include <iostream>
#include <memory>
using namespace std;

class Base {
public:
    ~Base() { cout << "Destroy Base" << endl; }
};

class Derived : public Base {
public:
    ~Derived() { cout << "Destroy Derived" << endl; }
};

int main()
{
    shared_ptr<Base> sp(new Derived);

    return 0;
}
```
众所周知，基类指针或引用指向派生类的场景中，如果基类析构函数是虚函数，那么对象析构时会先调用派生类析构函数，再调用基类析构函数。
上述代码中Base类的析构函数不是虚函数，那么理论上不会调用Derived的析构函数，可是实际上上述代码的输出
是这样的：

<img src="https://smileshineme.github.io/images/20241204/res.png">

调用了Derived的析构函数，这是为啥呢？去看一看shared_ptr的实现应该能看出原因吧。

让我们打开一份GCC源码（版本是13.2.0），找到shared_ptr的定义：
``` {.line-numbers}
  /* 展示代码时只列出与我们分析问题相关的代码 */
  template<typename _Tp>
    class shared_ptr
    : public __shared_ptr<_Tp>
    {
    public:
      template<typename _Tp1>
        explicit
        shared_ptr(_Tp1* __p)
	: __shared_ptr<_Tp>(__p) { }
    }
```
示例代码调用的是上述构造函数，根据模板参数推断的结果，显然_Tp1是Derived类型，_Tp是Base类型，这个构造函数仅仅是初始化了__shared_ptr，下面看__shared_ptr的定义：
```
  template<typename _Tp, _Lock_policy _Lp>
    class __shared_ptr
    {
    public:
      template<typename _Tp1>
        explicit
        __shared_ptr(_Tp1* __p)
	: _M_ptr(__p), _M_refcount(__p)
        {
	  __glibcxx_function_requires(_ConvertibleConcept<_Tp1*, _Tp*>)
	  typedef int _IsComplete[sizeof(_Tp1)];
	  __enable_shared_from_this_helper(_M_refcount, __p, __p);
	}
    private:
      _Tp*         	   _M_ptr;         // Contained pointer.
      __shared_count<_Lp>  _M_refcount;    // Reference counter.
    }

```
__shared_ptr调用上述构造函数，根据模板参数推断结果可知_Tp是Base类型，_Tp1是Derived类型，在构造函数初始化列表中初始化了两个成员：_M_ptr和_M_refcount。_M_ptr是一个Base类型的指针，这里的操作仅仅是让_M_ptr指向我们申请的对象的地址。而_M_refcount是__shared_count<_Lp>类型。继续在GCC源码中查找__shared_count的定义。
```
  template<_Lock_policy _Lp = __default_lock_policy>
    class __shared_count
    {
    public: 
      __shared_count()
      : _M_pi(0) // nothrow
      { }
  
      template<typename _Ptr>
        __shared_count(_Ptr __p) : _M_pi(0)
        {
	  __try
	    {
	      typedef typename std::tr1::remove_pointer<_Ptr>::type _Tp;
	      _M_pi = new _Sp_counted_base_impl<_Ptr, _Sp_deleter<_Tp>, _Lp>(
	          __p, _Sp_deleter<_Tp>());
	    }
	  __catch(...)
	    {
	      delete __p;
	      __throw_exception_again;
	    }
	}
    private:
      _Sp_counted_base<_Lp>*  _M_pi;    
    }
```
__shared_ptr构造函数中的_M_refcount(__p)对应上述构造函数。根据参数推断结果可知Ptr是Derived*类型，而_Tp的类型是Ptr去掉指针之后的类型，也就是Derived。
```
_M_pi = new _Sp_counted_base_impl<_Ptr, _Sp_deleter<_Tp>, _Lp>(__p，_Sp_deleter<_Tp>());
```
继续查看_Sp_counted_base_impl的定义：
```
  template<typename _Ptr, typename _Deleter, _Lock_policy _Lp>
    class _Sp_counted_base_impl
    : public _Sp_counted_base<_Lp>
    {
    public:
      // Precondition: __d(__p) must not throw.
      _Sp_counted_base_impl(_Ptr __p, _Deleter __d)
      : _M_ptr(__p), _M_del(__d) { }
    }
    private:
      _Ptr      _M_ptr;  // copy constructor must not throw
      _Deleter  _M_del;  // copy constructor must not throw
    }
```
据此可知_Ptr是Derived*类型，_Deleter是_Sp_deleter<_Tp>，_M_ptr是指向Derived对象的Derived类型指针，_M_del是_Sp_deleter<Derived>的一个实例，下面看一下_Sp_deleter的定义。
```
  template<typename _Tp>
    struct _Sp_deleter
    {
      typedef void result_type;
      typedef _Tp* argument_type;
      void operator()(_Tp* __p) const { delete __p; }
    };
```
根据_Sp_deleter的定义可知_M_del是一个可调用对象，对其进行调用的效果是删除指定类型的对象。

构造部分的代码到此就分析完了，下面开始分析析构部分的代码，首先会调用shared_ptr的析构函数，
查看shared_ptr的定义发现并没有定义析构函数，那么就会调用默认析构函数析构基类成员，基类__shared_ptr的成员_M_refcount是一个对象，在析构时会调用其析构函数。_Mem_refcount的析构函数如下：
```
      ~__shared_count() // nothrow
      {
	if (_M_pi != 0)
	  _M_pi->_M_release();
      }
```
而
```
// 省略无关代码
      void
      _M_release() // nothrow
      {
        // Be race-detector-friendly.  For more info see bits/c++config.
        _GLIBCXX_SYNCHRONIZATION_HAPPENS_BEFORE(&_M_use_count);
	if (__gnu_cxx::__exchange_and_add_dispatch(&_M_use_count, -1) == 1)
	  {
            _GLIBCXX_SYNCHRONIZATION_HAPPENS_AFTER(&_M_use_count);
	    _M_dispose();
      }
      }
```
以及
```
      virtual void
      _M_dispose() = 0; // nothrow
```
最终会调用_Sp_counted_base_impl的_M_dispose()接口：
```
      virtual void
      _M_dispose() // nothrow
      { _M_del(_M_ptr); }
```
根据前文可知_M_del是一个可调用对象，调用结果是执行delete操作，其类型是_Sp_deleter<Derived>，那么调用效果实例化之后是：
```
      void operator()(Derived* __p) const { delete __p; }
```
因此会调用Derived类型的析构函数。

至此，文章开头的问题已经得到解答，简单来说就是shared_ptr中保存了对象的删除器信息，在析构的时候会调用该删除器。