---
layout: post
title: C++模板使用
date: 2017-12-16
tags: C++  
---

<!-- TOC -->

- [类模板声明与定义](#类模板声明与定义)
- [类模板函数指针](#类模板函数指针)
- [模板类中变量初始化](#模板类中变量初始化)
- [类模板参数当常量](#类模板参数当常量)
- [类模板特化（不定义泛型模板函数，而去定义特定类型函数）](#类模板特化不定义泛型模板函数而去定义特定类型函数)
- [类中包含友元模板类](#类中包含友元模板类)

<!-- /TOC -->
  
# 类模板声明与定义

```C++
template<typename T = char>
class CXmlParser
{
public:
    CXmlParser();
    ~CXmlParser();

    bool Load(const std::wstring& strCfgPath);
    bool LoadFromData(const std::string& strCfgData);
    bool LoadFromData(const std::wstring& strCfgData);
    bool FindElem(const T* ElemName);
    void IntoElem();
    void OutOfElem();
    template<typename TRet>
    TRet GetAttrib(const T* AttrKey);
    const std::wstring GetValue(const T* ValueKey);

private:
    rapidxml::xml_document<T>   m_xmlDoc;
};

template<typename T>
CXmlParser<T>::CXmlParser()
{

}

...

```

# 类模板函数指针

```C++

template<typename T,
    typename TObj,
    typename TContainer = CTaskContainer<TObj>,
    UINT cThreadCnt = 3>
class CTaskPool : public CMutiThread<cThreadCnt>
{
    typedef void(T::*fPolicyProc)(TObj obj);  // 模板函数指针
public:
    CTaskPool()
        : m_bQuit(FALSE)
    {}
    ~CTaskPool() {}

    virtual void ThreadProc(unsigned int nIndex)
    {
        while(!IsQuit())
        {
            WaitEvent(nIndex, INFINITE);
            if (IsQuit()) break;
            if (!m_pContainer) break;

            if (m_pObj && m_fProc)
            {
                (m_pObj->*m_fProc)(m_pContainer->PopHead());
            }
        }
    }

    HRESULT Create(T* pobj, fPolicyProc fn, TContainer* pContainer)
    {
        HRESULT hr = E_FAIL;

        do
        {
            if (!pobj || !fn || !pContainer) break;

            m_pObj = pobj;
            m_fProc = fn;
            m_pContainer = pContainer;
            hr = Start(this);

        } while(FALSE)

        return hr;
    }

    ...
};

```


# 模板类中变量初始化

```C++

template<>
template<typename TRet>
inline TRet CXmlParser<char>::GetAttrib(const char* AttrKey)
{
    TRet ret = TRet();  // 模板变量初始化

    if (m_pNodeNext && m_pNodeNext->first_attribute(AttrKey) && m_pNodeNext->first_attribute(AttrKey)->value())
    {
        std::istringstream iss(m_pNodeNext->first_attribute()->value());
        iss >> ret;
    }
}

```

# 类模板参数当常量

```C++

template<UINT cThreadCnt = 1>
class CMutiThread : public IThreadProcCallbackEx
{
public:
    CMutiThread() : m_bStop(FALSE) {}
    virtual ~CMutiThread() {}

    HRESULT Start(IThreadProcCallbackEx* pThis)
    {
        HRESULT hr = S_OK;

        for (size_t i = 0; i < cThreadCnt; i++)
        {
            unsigned int uiTID = 0;
            m_threadParam[i].event.Create(FALSE, FALSE);
            m_threadParam[i].pCallbackObj = pThis;
            m_threadParam[i].dwIndex = i;
            m_hThreads[i] = (HANDLE)_beginthreadex(NULL, 0, __MUL_THREAD_PROC, &m_threadParam[i], 0, &uiTID);
            if (m_hThreads[i] == INVALID_HANDLE_VALUE || m_hThreads[i] == NULL)
            {
                hr = E_FAIL;
                break;
            }
        }

        return hr;
    }

    ...

protected:
    HANDLE  m_hThreads[cThreadCnt];     // 类模板参数cThreadCnt当常量
    THREAD_PROC_PARAM   m_threadParam[cThreadCnt];
    CEvent  m_event[cThreadCnt];
    BOOL    m_bStop;
}

```

# 类模板特化（不定义泛型模板函数，而去定义特定类型函数）

```C++
template<typename T = char>
class CXmlParser
{
public:
    CXmlParser();
    ~CXmlParser();

    ...

    template<typename TRet>
    TRet GetAttrib(const T* AttrKey);

    ...

private:
    rapidxml::xml_document<T>   m_xmlDoc;
};

// 特化
template<>
template<typename TRet>
inline TRet CXmlParser<char>::GetAttrib(const char* AttrKey)
{
    TRet ret = TRet();  // 模板变量初始化

    if (m_pNodeNext && m_pNodeNext->first_attribute(AttrKey) && m_pNodeNext->first_attribute(AttrKey)->value())
    {
        std::istringstream iss(m_pNodeNext->first_attribute()->value());
        iss >> ret;
    }
}

```

# 类中包含友元模板类

```C++

template<typename T,
    typename TObj,
    typename TContainer = CTaskContainer<TObj>,
    UINT cThreadCnt = 3>
class CTaskPool : public CMutiThread<cThreadCnt>
{
    typedef void(T::*fPolicyProc)(TObj obj);  // 模板函数指针
public:
    CTaskPool()
        : m_bQuit(FALSE)
    {}
    ~CTaskPool() {}

    virtual void ThreadProc(unsigned int nIndex)
    {
        while(!IsQuit())
        {
            WaitEvent(nIndex, INFINITE);
            if (IsQuit()) break;
            if (!m_pContainer) break;

            if (m_pObj && m_fProc)
            {
                (m_pObj->*m_fProc)(m_pContainer->PopHead());    // 访问Container的私有成员函数PopHead。
            }
        }
    }

    ...

};

template<typename TObj>
class CTaskContainer
{
    template<typename T,typename TObj,typename TContainer,UINT cThreadCnt> friend class CTaskPool;      // 让CTaskPool可以访问私有成员。目的是防止成员函数泄露，但又提供CTaskPool此类访问方法

public:
    CTaskContainer() {}
    ~CTaskContainer() {}

    DWORD GetCount()
    {
        CAutoCriticalSection lock(m_csTaskObjs);
        return m_qTaskObjs.size();
    }

protected:
    void AddTail(TObj obj)
    {
        CAutoCriticalSection lock(m_csTaskObjs);
        m_qTaskObjs.push_back(VALUE(obj));
    }

    TObj PopHead()
    {
        TObj obj;

        CAutoCriticalSection lock(m_csTaskObjs);
        if (!m_qTaskObjs.empty())
        {
            obj = m_qTaskObjs.front().obj;
            m_qTaskObjs.erase(m_qTaskObjs.begin());
        }

        return obj;
    }

```