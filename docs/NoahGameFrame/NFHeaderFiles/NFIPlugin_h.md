# NFIPlugin 头文件解析

NFIPlugin 是所有自定义插件的父类，而 NFIPlugin 又是继承自 NFIModule。

用 map 存储 名字和 module的 pair , 继承于NFImodule 的awake init 等函数，在继承的函数内 保存的 modules 分别调用其相应的函数。


## 六个重要的宏

```c++

#define REGISTER_MODULE(pManager, classBaseName, className)  \
	assert((TIsDerived<classBaseName, NFIModule>::Result));	\
	assert((TIsDerived<className, classBaseName>::Result));	\
	NFIModule* pRegisterModule##className= new className(pManager); \
    pRegisterModule##className->strName = (#className); \
    pManager->AddModule( #classBaseName, pRegisterModule##className );\
    this->AddElement( #classBaseName, pRegisterModule##className );

#define REGISTER_TEST_MODULE(pManager, classBaseName, className)  \
	assert((TIsDerived<classBaseName, NFIModule>::Result));	\
	assert((TIsDerived<className, NFIModule>::Result));	\
	NFIModule* pRegisterModule##className= new className(pManager); \
    pRegisterModule##className->strName = (#className); \
    pManager->AddTestModule( #classBaseName, pRegisterModule##className );

#define UNREGISTER_MODULE(pManager, classBaseName, className) \
    NFIModule* pUnRegisterModule##className = dynamic_cast<NFIModule*>( pManager->FindModule( #classBaseName )); \
	pManager->RemoveModule( #classBaseName ); \
    this->RemoveElement( #classBaseName ); \
    delete pUnRegisterModule##className;

#define UNREGISTER_TEST_MODULE(pManager, classBaseName, className) \
    NFIModule* pUnRegisterModule##className = dynamic_cast<NFIModule*>( pManager->FindtESTModule( #classBaseName )); \
	pManager->RemoveTestModule( #classBaseName ); \
    delete pUnRegisterModule##className;

#define CREATE_PLUGIN(pManager, className)  NFIPlugin* pCreatePlugin##className = new className(pManager); pManager->Registered( pCreatePlugin##className ;

#define DESTROY_PLUGIN(pManager, className) pManager->UnRegistered( pManager->FindPlugin((#className)) );


```

注册宏 REGISTER_MODULE  先判断类型 之间是否存在继承关系， 将pRegisterModule 和子类 拼接成新的变量名  然后添加到 map 里面。 其他的宏同理！


## 完整代码

```c++

#ifndef NFI_PLUGIN_H
#define NFI_PLUGIN_H

#include <iostream>
#include <assert.h>
#include "NFComm/NFCore/NFMap.hpp"
#include "NFComm/NFPluginModule/NFIModule.h"
#include "NFComm/NFPluginModule/NFIPluginManager.h"

#define REGISTER_MODULE(pManager, classBaseName, className)  \
	assert((TIsDerived<classBaseName, NFIModule>::Result));	\
	assert((TIsDerived<className, classBaseName>::Result));	\
	NFIModule* pRegisterModule##className= new className(pManager); \
    pRegisterModule##className->strName = (#className); \
    pManager->AddModule( #classBaseName, pRegisterModule##className );\
    this->AddElement( #classBaseName, pRegisterModule##className );

#define REGISTER_TEST_MODULE(pManager, classBaseName, className)  \
	assert((TIsDerived<classBaseName, NFIModule>::Result));	\
	assert((TIsDerived<className, NFIModule>::Result));	\
	NFIModule* pRegisterModule##className= new className(pManager); \
    pRegisterModule##className->strName = (#className); \
    pManager->AddTestModule( #classBaseName, pRegisterModule##className );

#define UNREGISTER_MODULE(pManager, classBaseName, className) \
    NFIModule* pUnRegisterModule##className = dynamic_cast<NFIModule*>( pManager->FindModule( #classBaseName )); \
	pManager->RemoveModule( #classBaseName ); \
    this->RemoveElement( #classBaseName ); \
    delete pUnRegisterModule##className;

#define UNREGISTER_TEST_MODULE(pManager, classBaseName, className) \
    NFIModule* pUnRegisterModule##className = dynamic_cast<NFIModule*>( pManager->FindtESTModule( #classBaseName )); \
	pManager->RemoveTestModule( #classBaseName ); \
    delete pUnRegisterModule##className;

#define CREATE_PLUGIN(pManager, className)  NFIPlugin* pCreatePlugin##className = new className(pManager); pManager->Registered( pCreatePlugin##className );

#define DESTROY_PLUGIN(pManager, className) pManager->UnRegistered( pManager->FindPlugin((#className)) );

/*
#define REGISTER_COMPONENT(pManager, className)  NFIComponent* pRegisterComponent##className= new className(pManager); \
    pRegisterComponent##className->strName = (#className); \
    pManager->AddComponent( (#className), pRegisterComponent##className );

#define UNREGISTER_COMPONENT(pManager, className) NFIComponent* pRegisterComponent##className =  \
        dynamic_cast<NFIComponent*>( pManager->FindComponent( (#className) ) ); pManager->RemoveComponent( (#className) ); delete pRegisterComponent##className;
*/

class NFIPluginManager;

class NFIPlugin : public NFIModule
{

public:
	NFIPlugin()
	{
	}
    virtual ~NFIPlugin()
	{
	}
    virtual const int GetPluginVersion() = 0;
    virtual const std::string GetPluginName() = 0;

    virtual void Install() = 0;

	virtual void Uninstall() = 0;


	void AddElement(const std::string& name, NFIModule* module)
	{
		mModules[name] = module;
	}

	NFIModule* GetElement(const std::string& name)
	{
		auto it = mModules.find(name);
		if (it != mModules.end())
		{
			return it->second;
		}

		return nullptr;
	}

	void RemoveElement(const std::string& name)
	{
		auto it = mModules.find(name);
		if (it != mModules.end())
		{
			mModules.erase(it);
		}
	}

	virtual bool Awake()
	{
		for (auto it : mModules)
		{
			NFIModule* pModule = it.second;

			pPluginManager->SetCurrentModule(pModule);

			bool bRet = pModule->Awake();
			if (!bRet)
			{
				std::cout << pModule->strName << std::endl;
				assert(0);
			}
		}

		return true;
	}

    virtual bool Init()
	{
		for (auto it : mModules)
		{
			NFIModule *pModule = it.second;

			pPluginManager->SetCurrentModule(pModule);
			bool bRet = pModule->Init();
			if (!bRet)
			{
				std::cout << pModule->strName << std::endl;
				assert(0);
			}
		}

        return true;
    }

    virtual bool AfterInit()
    {
		for (auto it : mModules)
		{
			NFIModule *pModule = it.second;

			pPluginManager->SetCurrentModule(pModule);
            bool bRet = pModule->AfterInit();
            if (!bRet)
            {
				std::cout << pModule->strName << std::endl;
                assert(0);
            }
        }
        return true;
    }

    virtual bool CheckConfig()
    {
		for (auto it : mModules)
		{
			NFIModule *pModule = it.second;

			pPluginManager->SetCurrentModule(pModule);
            pModule->CheckConfig();

        }

        return true;
    }

	virtual bool ReadyExecute()
	{
		for (auto it : mModules)
		{
			NFIModule *pModule = it.second;

			pPluginManager->SetCurrentModule(pModule);
			pModule->ReadyExecute();
		}

		return true;
	}

    virtual bool Execute()
    {
		for (auto it : mModules)
		{
			NFIModule *pModule = it.second;

			pPluginManager->SetCurrentModule(pModule);
            pModule->Execute();

        }

        return true;
    }

    virtual bool BeforeShut()
    {
		for (auto it : mModules)
		{
			NFIModule *pModule = it.second;

			pPluginManager->SetCurrentModule(pModule);
            pModule->BeforeShut();
        }

        return true;
    }

    virtual bool Shut()
    {
		for (auto it : mModules)
		{
			NFIModule *pModule = it.second;

			pPluginManager->SetCurrentModule(pModule);
            pModule->Shut();
        }

        return true;
    }

    virtual bool Finalize()
    {
		for (auto it : mModules)
		{
			NFIModule *pModule = it.second;

			pPluginManager->SetCurrentModule(pModule);
            pModule->Finalize();

        }

        return true;
    }

	virtual bool OnReloadPlugin()
	{
		for (auto it : mModules)
		{
			NFIModule *pModule = it.second;

			pPluginManager->SetCurrentModule(pModule);
			pModule->OnReloadPlugin();
		}

		return true;
	}

private:
	std::map<std::string, NFIModule*> mModules;
};

#endif


```