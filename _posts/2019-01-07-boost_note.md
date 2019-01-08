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

# 信号槽signals2

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