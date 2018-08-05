---
title: Magical Codes in C++ 11/14
date: 2018-08-05
categories:
- C++
tags:
- C++
- C++ 11/14
---

本文通过阅读gcc源码，分析C++ 11/14中一些神奇的操作是如何实现的

---



# std::move
[std::move](https://en.cppreference.com/w/cpp/utility/move "std::move")用于显式的触发对象间的“移动”
例如在以下代码中，可以把s“移动”进入vector，减少一次内存拷贝（如果s不再需要了）

```cpp
std::string s("Hello, world !!!!!");
std::vector<std::string> vss;
vss.push_back( std::move(s) );
```

std::move是如何做到让对象“移动”的？答案很简单：强制类型转换。强制类型转换为右值，然后调用接收对象的“移动构造函数”或“移动赋值函数”

std::move实现（详见[bits/move.h](https://gcc.gnu.org/onlinedocs/gcc-7.3.0/libstdc++/api/a00413_source.html)，91行）：
```cpp
template<typename T>
constexpr typename std::remove_reference<T>::type&& move(T&& t) noexcept
{ return static_cast<typename std::remove_reference<T>::type&&>(t); }
```

有点难看懂？没关系。以下列代码为例
```cpp
std::string str1("Hello, world !!!!!");
std::string str2 = std::move(str1);
```
第二句str2的赋值，相当于` std::string str2 = static_cast<std::string &&>(str1);  `
然后函数重载，匹配 str2的移动赋值`operator=(string &&)` ，然后`operator=(string &&)`将str1的内容“移动”到str2

回过头看std::move的实现
入参 T&& 叫做“universal reference”，可以同时接受左值引用和右值引用
`std::remove_reference<T>::type`用于去掉T的引用，来确保 `std::remove_reference<T>::type&&` 一定是右值引用。关于std::remove_reference，请继续看第2节
那这里为何不直接返回`T&&`？原因刚才也讲了，在这里`T&&`其实是“universal reference”，而不是右值引用。具体请参考《Effective Modern C++》Item 24。

---



# std::remove_reference
如上边std::move实现中展示的例子，std::remove_reference用于得到去掉引用后的类型。这种操作在一般的业务逻辑开发中应该很少遇到吧，除非是写类/模板库

std::move_reference利用C++模板的类型推导实现，典型的[实现](https://en.cppreference.com/w/cpp/types/remove_reference "实现")：
```cpp
template< class T > struct remove_reference      {typedef T type;};
template< class T > struct remove_reference<T&>  {typedef T type;};
template< class T > struct remove_reference<T&&> {typedef T type;};
```

类似的，还有`std::remove_const`这样的东西，详见[这里](https://en.cppreference.com/w/cpp/types/remove_cv "这里")。

---



# emplace / emplace_back
从C++ 11开始，很多容器都增加了emplace / emplace_back等操作。相比insert、push_back，特定场景下性能有提升。《Effective Modern C++》Item 24对比了push_back与emplace_back的性能差别，并给出了性能提升的三个条件：
1. the value being added is constructed into the container, not assigned;
2. the argument type(s) passed differ from the type held by the container;
3. the container won’t reject the value being added due to it being a duplicate.

以`std::vector<std::string> vss;`为例。`vss.push_back("Hello, world!");`相当于`vss.push_back(std::string("Hello, world!"));`，需要构造和析构临时std::string变量。而emplace_back直接在std::vector内部进行元素构造，因此节省了这部分开销。这也是`emplace`这个单词的含义吧。
具体push_back和emplace_back的实现：
* push_back（完整源码见[stl_vector.h](https://gcc.gnu.org/onlinedocs/gcc-7.3.0/libstdc++/api/a00590_source.html) ，939行）
```cpp
void push_back(const value_type& __x)
{
	if (this->_M_impl._M_finish != this->_M_impl._M_end_of_storage)
	  {
	    _Alloc_traits::construct(this->_M_impl, this->_M_impl._M_finish, __x);
	    ++this->_M_impl._M_finish;
	  }
	else
	  _M_realloc_insert(end(), __x);
}
```

* emplace_back (完整源码见[vector.tcc](https://gcc.gnu.org/onlinedocs/gcc-7.3.0/libstdc++/api/a00635_source.html)，96行)
```cpp
template<typename... _Args>
void emplace_back(_Args&&... __args)
{
	if (this->_M_impl._M_finish != this->_M_impl._M_end_of_storage)
	  {
	    _Alloc_traits::construct(this->_M_impl, this->_M_impl._M_finish,
				     std::forward<_Args>(__args)...);
	    ++this->_M_impl._M_finish;
	  }
	else
	  _M_realloc_insert(end(), std::forward<_Args>(__args)...);
}
```

其中，`_Alloc_traits::construct`最终会调用placement new
对于push_back：
```cpp
void construct(pointer __p, const _Tp& __val)
{ ::new((void *)__p) _Tp(__val); }
```
对于emplace_back：
```cpp
template<typename _Up, typename... _Args>
void construct(_Up* __p, _Args&&... __args)
{ ::new((void *)__p) _Up(std::forward<_Args>(__args)...); }
```
所以，emplace系列方法还得益C++ 11的新特性[template parameter pack](https://en.cppreference.com/w/cpp/language/parameter_pack)（也有文档叫variadic parameter），来适应不同的构造函数。

---




# std::shared_ptr
`std::shared_ptr`是可共享的智能指针，通过引用计数实现。详见[这里](https://gcc.gnu.org/onlinedocs/gcc-7.3.0/libstdc++/api/a15774_source.html)，86行。

```cpp
  template<typename _Tp>
    class shared_ptr : public __shared_ptr<_Tp>
    {
      ...
      
      shared_ptr(const shared_ptr&) noexcept = default;
      
      shared_ptr& operator=(const shared_ptr&) noexcept = default;
      
      template<typename _Yp, typename = _Constructible<_Yp*>>
      explicit shared_ptr(_Yp* __p) : __shared_ptr<_Tp>(__p) { }
      
      ...
    };
```

可以看出，实际的实现代码都在`__shared_ptr`中，`shared_ptr`只是一层皮。详见 [shared_ptr_base.h](https://gcc.gnu.org/onlinedocs/gcc-7.3.0/libstdc++/api/a00497_source.html)，1033行。
#### __shared_ptr
```cpp
  template<typename _Tp, _Lock_policy _Lp = __default_lock_policy>
    class __shared_ptr
    : public __shared_ptr_access<_Tp, _Lp>
    {
      using element_type = typename remove_extent<_Tp>::type;
      
      ...
      
      template<typename _Yp, typename = _SafeConv<_Yp>>
        explicit __shared_ptr(_Yp* __p)
        : _M_ptr(__p), _M_refcount(__p, typename is_array<_Tp>::type())
        {
          ...
        }

      template<typename _Yp, typename = _Compatible<_Yp>>
        __shared_ptr(const __shared_ptr<_Yp, _Lp>& __r) noexcept
        : _M_ptr(__r._M_ptr), _M_refcount(__r._M_refcount)
        { }

      template<typename _Yp>
      _Assignable<_Yp> operator=(const __shared_ptr<_Yp, _Lp>& __r) noexcept
        {
          _M_ptr = __r._M_ptr;
          _M_refcount = __r._M_refcount; // __shared_count::op= doesn't throw
          return *this;
        }
      
      ~__shared_ptr() = default;

      ...

      element_type*	   _M_ptr;         // Contained pointer.
      __shared_count<_Lp>  _M_refcount;    // Reference counter.
    };
```

可以看到，引用计数的秘密都在`__shared_count`，详见[shared_ptr_base.h](https://gcc.gnu.org/onlinedocs/gcc-7.3.0/libstdc++/api/a00497_source.html)，571行。
#### __shared_count
```cpp
  template<_Lock_policy _Lp = __default_lock_policy>
    class __shared_count
    {
    public:
      constexpr __shared_count() noexcept : _M_pi(0)
      { }

      template<typename _Ptr>
        explicit __shared_count(_Ptr __p) : _M_pi(0)
        {
          _M_pi = new _Sp_counted_ptr<_Ptr, _Lp>(__p);
          ...
        }

      __shared_count(const __shared_count& __r) noexcept
      : _M_pi(__r._M_pi)
      {
        if (_M_pi != 0)
        _M_pi->_M_add_ref_copy();
      }

      __shared_count& operator=(const __shared_count& __r) noexcept
      {
        _Sp_counted_base<_Lp>* __tmp = __r._M_pi;
        if (__tmp != _M_pi)
        {
          if (__tmp != 0)
            __tmp->_M_add_ref_copy();
          if (_M_pi != 0)
            _M_pi->_M_release();
          _M_pi = __tmp;
        }
        return *this;
      }
      
      ~__shared_count() noexcept
      {
        if (_M_pi != nullptr)
          _M_pi->_M_release();
      }
      
      ...
      
    private:
      ...
      _Sp_counted_base<_Lp>*  _M_pi;
    };
```
`__shared_count`的核心是 `_M_pi` 成员变量，类型是 `_Sp_counted_ptr`。详见 [shared_ptr_base.h](https://gcc.gnu.org/onlinedocs/gcc-7.3.0/libstdc++/api/a00497_source.html)，366行。

#### _Sp_counted_ptr & _Sp_counted_base
```cpp
  template<typename _Ptr, _Lock_policy _Lp>
    class _Sp_counted_ptr final : public _Sp_counted_base<_Lp>
    {
      virtual void _M_dispose() noexcept
      { delete _M_ptr; }
      
      ...
    };

  template<_Lock_policy _Lp = __default_lock_policy>
    class _Sp_counted_base
    : public _Mutex_base<_Lp>
    {
    public:
      _Sp_counted_base() noexcept
      : _M_use_count(1), ... { }
    
      ...
    
      void _M_release() noexcept
      {
        ...
        if (__gnu_cxx::__exchange_and_add_dispatch(&_M_use_count, -1) == 1)
          {
            ...
            _M_dispose();
            ...
      }

    private:
      _Atomic_word  _M_use_count;     // #shared
      ...
    };

  template<>
    inline void
    _Sp_counted_base<_S_atomic>::
    _M_add_ref_lock()
    {
      // Perform lock-free add-if-not-zero operation.
      _Atomic_word __count = _M_get_use_count();
      do
      {
        ...
      }
      while (!__atomic_compare_exchange_n(&_M_use_count, &__count, __count + 1,
					  true, __ATOMIC_ACQ_REL,
					  __ATOMIC_RELAXED));
    }
```

_`std::shared_ptr`的实现看起来有点过于复杂，经过了多层继承/组合。如此实现的原因目前从代码中看不出来，后续找相关文档研究研究）_

---



# std::begin & std::end

`std::begin`与`std::end`只是一个简单的工具，使__原生数组__（即`int a[10]`这类东西）和STL容器（包括支持begin和end方法的其他容器），可以使用相同的方式进行遍历，更“泛型”。
实现上比较简单，对于原生数组（完整源码见[range_access.h](https://gcc.gnu.org/onlinedocs/gcc-7.3.0/libstdc++/api/a00449_source.html) 85行和95行）：
```cpp
template<typename _Tp, size_t _Nm>
  inline constexpr _Tp* begin(_Tp (&__arr)[_Nm])
  { return __arr; }

template<typename _Tp, size_t _Nm>
  inline constexpr _Tp* end(_Tp (&__arr)[_Nm])
  { return __arr + _Nm; }
```
注意入参模板是`_Tp (&__arr)[_Nm]`，需要传入在编译期已知大小的数组。这点其实也好理解，不知道大小的数组是无法实现end方法的。

对于容器，直接调用容器的begin和end（完整源码见[range_access.h](https://gcc.gnu.org/onlinedocs/gcc-7.3.0/libstdc++/api/a00449_source.html) 46行和66行）
```cpp
template<typename _Container>
  inline constexpr auto
  begin(_Container& __cont) -> decltype(__cont.begin())
  { return __cont.begin(); }
```
```cpp
template<typename _Container>
  inline constexpr auto
  end(_Container& __cont) -> decltype(__cont.end())
  { return __cont.end(); }
```

---



# std::array

[std::array](https://en.cppreference.com/w/cpp/container/array)可以替代传统的C-style数组，提供更好的封装以及更便捷的方法(例如，两个std::array对象可以直接进行==、<、>等逻辑比较)
实现上，std::array对象在内部持有一个C-style数组，再进行二次封装。详见[array](https://gcc.gnu.org/onlinedocs/gcc-7.3.0/libstdc++/api/a00041_source.html)。

先定义什么是array：
```cpp
  template<typename _Tp, std::size_t _Nm>
    struct __array_traits
    {
      typedef _Tp _Type[_Nm];

      static constexpr _Tp&
      _S_ref(const _Type& __t, std::size_t __n) noexcept
      { return const_cast<_Tp&>(__t[__n]); }

      static constexpr _Tp*
      _S_ptr(const _Type& __t) noexcept
      { return const_cast<_Tp*>(__t); }
    };
```

`__array_traits::_Type`就是数组类型，`__array_traits::_S_ref`访问某一个元素，`__array_traits::__S_ptr`是首元素地址。

然后，定义std::array，以及成员方法

```cpp
  template<typename _Tp, std::size_t _Nm>
    struct array
    {
      ...
      typedef value_type*                     pointer;
      typedef value_type*          		      iterator;
      ...
      typedef std::__array_traits<_Tp, _Nm>   _AT_Type;
      typename _AT_Type::_Type                _M_elems;
      
      ...

      constexpr reference operator[](size_type __n) noexcept
      { return _AT_Type::_S_ref(_M_elems, __n); }

      constexpr const_reference operator[](size_type __n) const noexcept
      { return _AT_Type::_S_ref(_M_elems, __n); }
        
      ...
      
      pointer data() noexcept
      { return _AT_Type::_S_ptr(_M_elems); }
      iterator begin() noexcept
      { return iterator(data()); }
      iterator end() noexcept
      { return iterator(data() + _Nm); }

      ...
    };
```

然后，再定义其他方法，例如__逻辑相等__
```cpp
  template<typename _Tp, std::size_t _Nm>
    inline bool
    operator==(const array<_Tp, _Nm>& __one, const array<_Tp, _Nm>& __two)
    { return std::equal(__one.begin(), __one.end(), __two.begin()); }
```

---



# std::future & std::promise

Future/Promise是一种异步编程模型，参见[Futures_and_promises](https://en.wikipedia.org/wiki/Futures_and_promises)。

### std::async

Future的实现，先以std::async的使用场景为例：

```cpp
  #include <future>
  #include <iostream>

  int doTask()
  {
    int result = 0;
    ...
  
    return result;
  }

  int main()
  {
    auto fut = std::async(doTask);

    std::cout << "doTask result: " << fut.get() << "\n";
    return 0;
  }
```

##### std::async的实现

做了适当精简。完整代码详见[future](https://gcc.gnu.org/onlinedocs/gcc-7.3.0/libstdc++/api/a00074_source.html)，1709行：

```cpp
  template<typename _Fn, typename... _Args>
    future<__async_result_of<_Fn, _Args...>>
    async(launch __policy, _Fn&& __fn, _Args&&... __args)
    {
      std::shared_ptr<__future_base::_State_base> __state;
      if ((__policy & launch::async) == launch::async)
	  {
	      __state = __future_base::_S_make_async_state(
		    std::thread::__make_invoker(std::forward<_Fn>(__fn),
					      std::forward<_Args>(__args)...)
		  );
	  }
      if (!__state)
	  {
	    __state = __future_base::_S_make_deferred_state(
	        std::thread::__make_invoker(std::forward<_Fn>(__fn),
					  std::forward<_Args>(__args)...));
	  }
      return future<__async_result_of<_Fn, _Args...>>(__state);
    }
```
##### std::future的实现

std::future::get()的实现（做了适当精简。完整代码详见[future](https://gcc.gnu.org/onlinedocs/gcc-7.3.0/libstdc++/api/a00074_source.html)，791行）如下：

```cpp
  template<typename _Res>
    class future : public __basic_future<_Res>
    {
      ...
      
      typedef __basic_future<_Res> _Base_type;
      typedef typename _Base_type::__state_type __state_type;

      explicit future(const __state_type& __state) : _Base_type(__state) { }

      ... 
      
      /// Retrieving the value
      _Res get()
      {
        ...
        return std::move(this->_M_get_result()._M_value());
      }

      ...
    };
```
其中，```__basic_future::_M_get_result()```获得一个```__future_base::_Result```对象，并调用```__state```的```wait()```方法：
```cpp
  template<typename _Res>
    class __basic_future : public __future_base
    {
      ...

    protected:
      typedef shared_ptr<_State_base>		__state_type;
      typedef __future_base::_Result<_Res>&	__result_type;

    private:
      __state_type 		_M_state;

    public:
      __result_type _M_get_result() const
      {
        ...
        _Result_base& __res = _M_state->wait();
        ...
        return static_cast<__result_type>(__res);
      }
      
      explicit
      __basic_future(const __state_type& __state) : _M_state(__state)
      {
        ...
      }
      
      ...
    };
```
而```__future_base::_Result```对函数返回结果做了一个封装。底层通过```__gnu_cxx::__aligned_buffer```实现，详见[aligned_buffer.h](https://gcc.gnu.org/onlinedocs/gcc-7.3.0/libstdc++/api/a01019_source.html)

```cpp
    template<typename _Res>
      struct _Result : _Result_base
      {
      private:
        __gnu_cxx::__aligned_buffer<_Res>	_M_storage;
        ...

      public:
        typedef _Res result_type;

        ...

        _Res& _M_value() noexcept { return *_M_storage._M_ptr(); }
```
```_M_state```的基类为```_State_baseV2```，详见[future](https://gcc.gnu.org/onlinedocs/gcc-7.3.0/libstdc++/api/a00074_source.html)，306行：
```cpp
    class _State_baseV2
    {
     ...

      _Result_base& wait()
      {
        // Run any deferred function or join any asynchronous thread:
        _M_complete_async();
        ...
        return *_M_result;
      }
      
      ...
      virtual void _M_complete_async() { }
      ...
    };
```

##### 回头看std::async的实现

```__future_base::_S_make_deferred_state```与```__future_base::_S_make_async_state```分别生成```_Deferred_state```与```_Async_state_impl```对象：

```cpp
  template<typename _BoundFn>
    inline std::shared_ptr<__future_base::_State_base>
    __future_base::_S_make_deferred_state(_BoundFn&& __fn)
    {
      typedef typename remove_reference<_BoundFn>::type __fn_type;
      typedef _Deferred_state<__fn_type> __state_type;
      return std::make_shared<__state_type>(std::move(__fn));
    }

  template<typename _BoundFn>
    inline std::shared_ptr<__future_base::_State_base>
    __future_base::_S_make_async_state(_BoundFn&& __fn)
    {
      typedef typename remove_reference<_BoundFn>::type __fn_type;
      typedef _Async_state_impl<__fn_type> __state_type;
      return std::make_shared<__state_type>(std::move(__fn));
    }
```

```_Deferred_state```不生成线程，在```future::get()```调用的时候才真正执行：
```cpp
  template<typename _BoundFn, typename _Res>
    class __future_base::_Deferred_state final
    : public __future_base::_State_base
    {
      ...
      
      // Run the deferred function.
      virtual void
      _M_complete_async()
      {
        // Multiple threads can call a waiting function on the future and
        // reach this point at the same time. The call_once in _M_set_result
        // ensures only the first one run the deferred function, stores the
        // result in _M_result, swaps that with the base _M_result and makes
        // the state ready. Tell _M_set_result to ignore failure so all later
        // calls do nothing.
        _M_set_result(_S_task_setter(_M_result, _M_fn), true);
      }
      
      ...
    };
```
而```_Async_state_impl```在构造时生成新的线程，并且在```_M_complete_async```等待线程结束

```cpp
  template<typename _BoundFn, typename _Res>
    class __future_base::_Async_state_impl final
    : public __future_base::_Async_state_commonV2
    {
    public:
      explicit
      _Async_state_impl(_BoundFn&& __fn)
      : _M_result(new _Result<_Res>()), _M_fn(std::move(__fn))
      {
        _M_thread = std::thread{ [this] {
          ...
          _M_set_result(_S_task_setter(_M_result, _M_fn));
          ...
        } };
      }
      
      ...
    };

  class __future_base::_Async_state_commonV2
    : public __future_base::_State_base
  {
    ...
    
    virtual void _M_complete_async() { _M_join(); }
    void _M_join() { std::call_once(_M_once, &thread::join, &_M_thread); }

    thread _M_thread;
    once_flag _M_once;
  };
```



### std::promise

以下列场景为例：

```cpp
#include <future>
#include <iostream>
#include <thread>

int main()
{
    std::promise<int> p;
    std::future<int> fut = p.get_future();
    std::thread( [&p]{ p.set_value(100); }).detach();
 
    std::cout << "Waiting..." << std::flush;

    fut.wait();
    std::cout << "Done!\nResults are: " << fut.get() << '\n';
    return 0;
}
```

```std::promise```的实现如下（做了适当精简。完整源代码见[future](https://gcc.gnu.org/onlinedocs/gcc-7.3.0/libstdc++/api/a00074_source.html)，1041行）：
```cpp
  template<typename _Res>
    class promise
    {
      typedef __future_base::_State_base 	_State;
      ...

      shared_ptr<_State>                        _M_future;

    public:
      promise()
      : _M_future(std::make_shared<_State>()), ...
      { }
      
      ...

      // Retrieving the result
      future<_Res> get_future()
      { return future<_Res>(_M_future); }

      // Setting the result
      void set_value(const _Res& __r)
      { _M_future->_M_set_result(_State::__setter(this, __r)); }

      ...
    };
```
可以看到future与promise之间通过```std::shared_ptr<__future_base::_State_base>```来实现数据传递。

---