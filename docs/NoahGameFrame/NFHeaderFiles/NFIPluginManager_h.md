# NFIPluginManager 头文件解析

是所有 NFPluginManager 的父类。


## TIsDerived

模板元编程，利用编译期多态的性质来判断两个类是否具备继承关系。


```c++

template<typename DerivedType, typename BaseType>
class TIsDerived
{
public:
    static int AnyFunction(BaseType* base)
    {
        return 1;
    }

    static  char AnyFunction(void* t2)
    {
        return 0;
    }

    enum
    {
        Result = (sizeof(int) == sizeof(AnyFunction((DerivedType*)NULL))),
    };
};

```

重载了 AnyFunction， 传入 DerivedType* 类型的空指针，根据 AnyFunction 返回值类型的大小 来判断两个类型之间的关系。 如果有继承关系，Result的值为1， 否则为0；


## NFReplaceContent

```c++

class NFReplaceContent
{
public:
	NFReplaceContent(const std::string content, const std::string newValue)
	{
		this->content = content;
		this->newValue = newValue;
	}
	std::string content;
	std::string newValue;
};

```

暂时不清楚 这么写的用意是什么


## NFIPluginManager

用虚函数定义了基本的接口。

```c++
template <typename T>
	T* FindModule()
	{
		NFIModule* pLogicModule = FindModule(typeid(T).name());
		if (pLogicModule)
		{
			if (!TIsDerived<T, NFIModule>::Result)
			{
				return NULL;
			}
            //TODO OSX上dynamic_cast返回了NULL
#if NF_PLATFORM == NF_PLATFORM_APPLE
            T* pT = (T*)pLogicModule;
#else
			T* pT = dynamic_cast<T*>(pLogicModule);
#endif
			assert(NULL != pT);

			return pT;
		}
		assert(NULL);
		return NULL;
	}

	template <typename T>
	void ReplaceModule(NFIModule* pModule)
	{
		NFIModule* pLogicModule = FindModule(typeid(T).name());
		if (pLogicModule)

		{
			RemoveModule(typeid(T).name());
		}

        // name() 返回值具有多余字符， 已提交issue
		AddModule(typeid(T).name(), pModule);
	};


```

两个模板函数，

## 完整代码
```c++


#ifndef NFI_PLUGIN_MANAGER_H
#define NFI_PLUGIN_MANAGER_H

#include <functional>
#include <list>
#include <vector>
#include "NFPlatform.h"

class NFIPlugin;
class NFIModule;
class NFIPluginManager;

typedef std::function<bool (NFIPluginManager* p, const std::string& strFileName, std::string& strContent)> GET_FILECONTENT_FUNCTOR;

template<typename DerivedType, typename BaseType>
class TIsDerived
{
public:
    static int AnyFunction(BaseType* base)
    {
        return 1;
    }

    static  char AnyFunction(void* t2)
    {
        return 0;
    }

    enum
    {
        Result = (sizeof(int) == sizeof(AnyFunction((DerivedType*)NULL))),
    };
};

class NFReplaceContent
{
public:
	NFReplaceContent(const std::string content, const std::string newValue)
	{
		this->content = content;
		this->newValue = newValue;
	}
	std::string content;
	std::string newValue;
};

#define FIND_MODULE(classBaseName, className)  \
	assert((TIsDerived<classBaseName, NFIModule>::Result));

class NFIPluginManager
{
public:
    NFIPluginManager()
    {

    }

	/////////////////////

	virtual bool LoadPluginConfig()
	{
		return true;
	}

	virtual bool LoadPlugin()
	{
		return true;
	}

	virtual bool Awake()
	{
		return true;
	}

	virtual bool Init()
	{

		return true;
	}

	virtual bool AfterInit()
	{
		return true;
	}

	virtual bool CheckConfig()
	{
		return true;
	}

	virtual bool ReadyExecute()
	{
		return true;
	}

	virtual bool Execute()
	{
		return true;
	}

	virtual bool BeforeShut()
	{
		return true;
	}

	virtual bool Shut()
	{
		return true;
	}

	virtual bool Finalize()
	{
		return true;
	}

	virtual bool OnReloadPlugin()
	{
		return true;
	}

	/////////////////////

	template <typename T>
	T* FindModule()
	{
		NFIModule* pLogicModule = FindModule(typeid(T).name());
		if (pLogicModule)
		{
			if (!TIsDerived<T, NFIModule>::Result)
			{
				return NULL;
			}
            //TODO OSX上dynamic_cast返回了NULL
#if NF_PLATFORM == NF_PLATFORM_APPLE
            T* pT = (T*)pLogicModule;
#else
			T* pT = dynamic_cast<T*>(pLogicModule);
#endif
			assert(NULL != pT);

			return pT;
		}
		assert(NULL);
		return NULL;
	}

	template <typename T>
	void ReplaceModule(NFIModule* pModule)
	{
		NFIModule* pLogicModule = FindModule(typeid(T).name());
		if (pLogicModule)
		{
			RemoveModule(typeid(T).name());
		}


		AddModule(typeid(T).name(), pModule);
	};

	virtual bool ReLoadPlugin(const std::string& strPluginDLLName) = 0;

    virtual void Registered(NFIPlugin* plugin) = 0;

    virtual void UnRegistered(NFIPlugin* plugin) = 0;

    virtual NFIPlugin* FindPlugin(const std::string& strPluginName) = 0;

	virtual void AddModule(const std::string& strModuleName, NFIModule* pModule) = 0;

	virtual void AddTestModule(const std::string& strModuleName, NFIModule* pModule) = 0;

    virtual void RemoveModule(const std::string& strModuleName) = 0;

    virtual NFIModule* FindModule(const std::string& strModuleName) = 0;

    virtual NFIModule* FindTestModule(const std::string& strModuleName) = 0;

	virtual std::list<NFIModule*> Modules() = 0;
	virtual std::list<NFIModule*> TestModules() = 0;

    virtual int GetAppID() const = 0;
    virtual void SetAppID(const int nAppID) = 0;

    virtual bool IsRunningDocker() const = 0;
    virtual void SetRunningDocker(bool bDocker) = 0;

    virtual bool IsStaticPlugin() const = 0;

    virtual NFINT64 GetInitTime() const = 0;
    virtual NFINT64 GetNowTime() const = 0;

	virtual const std::string& GetConfigPath() const = 0;
	virtual void SetConfigPath(const std::string & strPath) = 0;

	virtual void SetConfigName(const std::string& strFileName) = 0;	
	virtual const std::string& GetConfigName() const = 0;

	virtual const std::string& GetAppName() const = 0;
	virtual void SetAppName(const std::string& strAppName) = 0;

	virtual const std::string& GetLogConfigName() const = 0;
	virtual void SetLogConfigName(const std::string& strName) = 0;

	virtual NFIPlugin* GetCurrentPlugin() = 0;
	virtual NFIModule* GetCurrentModule() = 0;

	virtual void SetCurrentPlugin(NFIPlugin* pPlugin) = 0;
	virtual void SetCurrentModule(NFIModule* pModule) = 0;

	virtual int GetAppCPUCount() const = 0;
	virtual void SetAppCPUCount(const int count) = 0;

	virtual void SetGetFileContentFunctor(GET_FILECONTENT_FUNCTOR fun) = 0;
	virtual bool GetFileContent(const std::string &strFileName, std::string &strContent) = 0;

	virtual void AddFileReplaceContent(const std::string& fileName, const std::string& content, const std::string& newValue) = 0;
	virtual std::vector<NFReplaceContent> GetFileReplaceContents(const std::string& fileName) = 0;
};

#endif


```