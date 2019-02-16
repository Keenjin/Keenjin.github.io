---
layout: post
title: c++ boost库使用笔记
date: 2019-01-07
tags: boost  
---

# Boost库编译步骤

参考：https://www.boost.org/doc/libs/1_55_0/doc/html/bbv2/reference.html

```txt
1、运行bootstrap.bat，生成b2.exe
2、查看需要编译的库
b2.exe --show-libraries
3、编译对应的库（范例：regex库，存放位置stage）
b2.exe --toolset=msvc-14.0 --with-regex stage

注意：
1、如果要关注xp下的支持
b2.exe -j4 toolset=msvc-14.0_xp cxxflags="/Zc:threadSafeInit- "

2、如果发现编译问题，是由于cl导致的，可以直接打开vs2015命令行工具来编译，这里包含了所有可用的编译工具
```

# 信号槽signals2和单实例

例子：有心跳通知，触发Client执行OnHeart

```C++
// 心跳
template<typename T>
class CClientHeartBind
{
public:
    typedef void (T::*FBind)(DWORD dwContext);

	void Bind(FBind fProc, T* pObj)
	{
		m_HeartBindConn = CClientHeart::get_mutable_instance().Connect(boost::bind(fProc, pObj, _1));
	}

private:
	boost::signals2::scoped_connection	m_HeartBindConn;
};

class CClientHeart : public boost::serialization::singleton<CClientHeart>
{
public:
    typedef boost::signals2::signal<void(DWORD)> signal_t;

    // 服务端在有心跳时调用此函数
    void HeartHit(DWORD dwContext = 0)
    {
        m_sig(dwContext);
    }

    boost::signals2::connection Connect(const signal_t::slot_type& BindObj)
    {
        return m_sig.connect(BindObj);
    }

private:
    signal_t	m_sig;
}
```

```C++
// 服务端
class CServer
{
public:
    void DoHeart()
    {
        CClientHeart::get_mutable_instance().HeartHit();
    }
}
```

```C++
// 客户对象
class CClient
{
public:
    void Init()
    {
        m_oBind.Bind(&CClient::OnHeart, this);
    }

    void OnHeart()
    {
        // Todo
    }

private:
    CClientHeartBind<CClient> m_oBind;
}
```

# boost::shared_ptr使用注意点

1、由于c++11支持了std::shared_ptr，项目为了能同时支持vs2015编译和vs2005编译，全部使用boost库  
2、一般我们面向接口编程，使用boost::shared_ptr<IAppInterface>。如果我们使用的类，想让外界调用方不用错（不使用std::shared_ptr来使用），一般是禁止构造函数，然后自定义static的CreateInstance导出boost::shared_ptr
3、如果我们全程都统一使用boost::shared_ptr，存在一个问题，如果我们的类之间使用了设计模式，需要传递this指针，直接传递this，对方使用boost::shared_ptr来接收，会有一个问题，对方如果析构了，会直接把我释放，那么不在我这个对象的预期之内，可能导致crash
4、为了正确使用3中所说，采用了class CApp : public boost::enable_shared_from_this<CApp>继承自boost::enable_shared_from_this，这个东西，使用装饰者模式，让CApp内部包含一个boost::weak_ptr，而this指针传递，使用它的接口this->shared_from_this()，这个东西会将本身指针包括引用计数，传递给参数的临时boost::shared_ptr，也就是说，传递出去后，本身是加引用计数的，只要对方使用boost::shared_ptr来接收，引用计数就由对方拿住（也就是此时引用计数为2）；如果对方不使用boost::shared_ptr来接收，那么引用计数随着传参的临时变量析构，保持为1
5、由于使用boost::shared_ptr来装载对象，所以析构的时候，是直接delete对象指针的。而我们又使用private 构造函数，来防止外界乱用其他指针或对象拷贝，加入析构也是private，那么boost::shared_ptr内部将无法delete对象，因此使用public 析构函数，private构造函数
6、使用了boost::enable_shared_from_this，我们可以使用this->shared_from_this()指针，但是要注意，此时无法在构造函数中使用，因为此时对象还未正式被构造出来，暂时无法使用。这个东西使用的前提是，对象已经被new出来，已经存在了weak_ptr


```c++
class IFrameFactory
	: public enable_shared_from_this<IFrameFactory>     // 让对象可以正常使用this->shared_from_this()指针，替代this
{
public:
	virtual void Create(boost::shared_ptr<ILogic> pLogic) = 0;
	virtual void Switch(EUIStateType eTo, boost::shared_ptr<IDataStream> pData) = 0;
	virtual boost::shared_ptr<IFrame> GetCurrentFrame() = 0;
};

class CFrameFactory : public IFrameFactory
{
public:
	static boost::shared_ptr<IFrameFactory> CreateInstance(boost::shared_ptr<ILogic> pLogic)
    {
        boost::shared_ptr<IFrameFactory> pFactory(new CFrameFactory);
        if (pFactory)
        {
            pFactory->Create(pLogic);
        }

        return pFactory;
    }

	virtual ~CFrameFactory() {}        // 允许boost::shared_ptr内部的delete

	// IFrameFactory
	virtual void Switch(EUIStateType eTo, boost::shared_ptr<IDataStream> pData) { ... }
	virtual boost::shared_ptr<IFrame> GetCurrentFrame() { ... }

private:
	CFrameFactory(){}             // 禁止使用构造，必须调用类自己的CreateInstance来构造对象

	virtual void Create(boost::shared_ptr<ILogic> pLogic)       // 避免了在构造函数中使用this->shared_from_this()
    {
        m_pLogic = pLogic;
        m_pStateContext = CUIStateContext::CreateInstance(this->shared_from_this());        // 传递this指针给boost::shared_ptr
    }

private:
    boost::shared_ptr<IUIStateContext>	m_pStateContext;
	boost::shared_ptr<ILogic>	m_pLogic;
}

class CUIStateContext : public IUIStateContext
{
public:
    static boost::shared_ptr<IUIStateContext> CreateInstance(boost::shared_ptr<IFrameFactory> pFactory)
    {
        boost::shared_ptr<IUIStateContext> pContext(new CUIStateContext);
        if (pContext)
        {
            pContext->Create(pFactory);
        }
        return pContext;
    }

private:
    void Create(boost::shared_ptr<IFrameFactory> pFactory)
    {
        m_pFactory = pFactory;
    }

private:
    boost::shared_ptr<IFrameFactory> m_pFactory;
}

```

# 运行时类型识别boost::dynamic_pointer_cast

1、如果要使用boost::dynamic_pointer_cast<T>(...)，则需要保证类有虚表，否则是无法使用的。另外，任意一个类继承自任意多个接口，只要保证每一个接口有虚表，则通过其中任意一个接口，调用boost::dynamic_pointer_cast，都能找到其他接口。

```C++
class A
{
public:
	virtual void test() { MessageBox(NULL, L"A::test()", NULL, 0); }
};

class B
{
public:
	virtual void test() { MessageBox(NULL, L"B::test()", NULL, 0); }
};

class C : public A, public B
{
public:
};

int main()
{
    A* c = new C;           // 如果是以A基类作为接收指针，则A类必须有虚表，否则下面的dynamic_cast无法使用（boost::dynamic_pointer_cast同理），B类可以无需有虚表指针。但是反过来，以B类作为接收类，则B类必须有虚表，才能找到A类
	B* b = dynamic_cast<B*>(c);
	b->test();
}

```

# 多继承同名函数的二义性

如果一个接口Z同时继承多个基类A和B，每个基类又都继承自同一个类C，则Z调用包含在C中的同名接口时，会出现二义性，编译器不知道调用谁。这个时候，解决二义性有两种方式：1、直接利用Z.A::C::Func()；2、确保A和B同时虚继承自C（这个是在C对象中变量可以共享使用的前提）

```C++
class C
{
public:
    void Func() {}
}

class A : virtual public C
{
}

class B : virtual public C
{
}

class Z : public A, public B
{

}

int main()
{
    Z z;
    z.Func();
}
```

# 避免boost::enable_shared_from_this二义性问题

```C++
class virtual_enable_shared_from_this_tmp : public boost::enable_shared_from_this<virtual_enable_shared_from_this_tmp>
{
public:
	virtual ~virtual_enable_shared_from_this_tmp() {}
};

template<typename T>
class virtual_enable_shared_from_this : virtual public virtual_enable_shared_from_this_tmp
{
public:
	boost::shared_ptr<T> shared_from_this()
	{
		return boost::dynamic_pointer_cast<T>(virtual_enable_shared_from_this_tmp::shared_from_this());
	}
};

class IA : public virtual_enable_shared_from_this<IA> {}
class IB : public virtual_enable_shared_from_this<IB> {}

class C : public IA, public IB
{
    void Test()
    {
        boost::shared_ptr<IA> pThis = this->IA::shared_from_this();       // this指针
    }
}

```

# 使用boost::filesystem::recursive_directory_iterator遍历文件夹

```c++
// 递归遍历整个文件夹
boost::system::error_code dir_error, no_error;
boost::filesystem::recursive_directory_iterator iter(oPath, dir_error), end_iter;
while (iter != end_iter)
{
    try
    {
        if (dir_error != no_error)      // 经常会有一些无法访问的异常，比如权限问题等，这里要判断下，过滤下此类异常
        {
            iter.increment(dir_error);
            continue;
        }

        if (boost::filesystem::is_regular_file(*iter))      // 这里可能是子目录，也可能是文件
        {
            if (HandleFile(*iter))
            {
                break;
            }
        }

        iter.increment(dir_error);
    }
    catch (boost::filesystem::filesystem_error&)
    {

    }
}
```

# boost::asio::io_context配合boost::thread实现异步任务

```c++
class CLogic
{
private:
    // 注意，m_io_context的定义必须在m_work_guard之前
    boost::asio::io_context m_io_context;
    boost::asio::executor_work_guard<boost::asio::io_context::executor_type> m_work_guard;
    boost::thread m_thread;
public:
    CLogic()
    : m_io_context()
	, m_thread(boost::bind(&boost::asio::io_context::run, &m_io_context))		// 创建一个线程，用于m_io_context.run()
	, m_work_guard(boost::asio::make_work_guard(m_io_context))		// 设置让m_io_context.run()不返回，方便post任务
    {

    }

    ~CLogic()
    {
        m_io_context.stop();
	    m_thread.interrupt();		// 线程到中断点，抛异常
	    m_thread.join();
    }

    void DoTask()
    {
        std::wstring strParam;
        m_io_context.post(boost::bind(&CLogic::DoTaskThread, this, strParam));
    }

    void DoTaskThread(std::wstring strParam)
    {
        ...
        while(xxx)
        {
            try
            {
                ...
                boost::this_thread::interruption_point();       // 线程调用interrupt接口时，执行到这里会抛出boost::thread_interrupted异常，可以方便在异常位置处理事件

                ...
                boost::this_thread::sleep(boost::posix_time::milliseconds(100));
                boost::this_thread::interruption_point();
            }
            catch(boost::thread_interrupted&)
            {
                break;
            }
        }
    }
}

```