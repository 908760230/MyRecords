# NFPluginManager

可以说pluginserver其实只是在pluginmanager上面在套一层，其实核心是pluginmanager，他完成了主要的工作。
NFPluginManager 继承于NFIPluginManager，里面大多是对接口的重写

## 头文件
```c++

#ifndef NF_PLUGIN_MANAGER_H
#define NF_PLUGIN_MANAGER_H

#include <map>
#include <string>
#include <time.h>
#include <thread>
#include "NFDynLib.h"
#include "NFComm/NFPluginModule/NFIModule.h"
#include "NFComm/NFPluginModule/NFIPluginManager.h"

class NFPluginManager
    : public NFIPluginManager
{
public:
    NFPluginManager();
    virtual ~NFPluginManager();

    virtual bool LoadPluginConfig() override;

    virtual bool LoadPlugin() override;

    virtual bool Awake() override;

    virtual bool Init() override;

    virtual bool AfterInit() override;

    virtual bool CheckConfig() override;

    virtual bool ReadyExecute() override;

    virtual bool BeforeShut() override;

    virtual bool Shut() override;

    virtual bool Finalize() override;


    //////////////////////////////////////////////////////////////////////////

    virtual void Registered(NFIPlugin* pPlugin) override;

    virtual void UnRegistered(NFIPlugin* pPlugin) override;

    //////////////////////////////////////////////////////////////////////////

    virtual bool ReLoadPlugin(const std::string& strPluginDLLName) override;

    virtual NFIPlugin* FindPlugin(const std::string& strPluginName) override;

    virtual void AddModule(const std::string& strModuleName, NFIModule* pModule) override;

    virtual void AddTestModule(const std::string& strModuleName, NFIModule* pModule) override;
 
    virtual void RemoveModule(const std::string& strModuleName) override;

    virtual NFIModule* FindModule(const std::string& strModuleName) override;

    virtual NFIModule* FindTestModule(const std::string& strModuleName) override;

    virtual std::list<NFIModule*> Modules() override;

    virtual bool Execute() override;

    virtual int GetAppID() const override;

    virtual void SetAppID(const int nAppID) override;

    virtual bool IsRunningDocker() const override;

    virtual void SetRunningDocker(bool bDocker) override;

    virtual bool IsStaticPlugin() const override;

    virtual NFINT64 GetInitTime() const override;

    virtual NFINT64 GetNowTime() const override;

    virtual const std::string& GetConfigPath() const override;
    virtual void SetConfigPath(const std::string & strPath) override;

    virtual void SetConfigName(const std::string& strFileName) override;
    virtual const std::string& GetConfigName() const override;

    virtual const std::string& GetAppName() const override;

    virtual void SetAppName(const std::string& strAppName) override;

    virtual const std::string& GetLogConfigName() const override;

    virtual void SetLogConfigName(const std::string& strName) override;

    virtual NFIPlugin* GetCurrentPlugin() override;
    virtual NFIModule* GetCurrenModule() override;

    virtual void SetCurrentPlugin(NFIPlugin* pPlugin) override;
    virtual void SetCurrenModule(NFIModule* pModule) override;

    virtual void SetGetFileContentFunctor(GET_FILECONTENT_FUNCTOR fun) override;

    virtual bool GetFileContent(const std::string &strFileName, std::string &strContent) override;

	virtual void AddFileReplaceContent(const std::string& fileName, const std::string& content, const std::string& newValue);
	virtual std::vector<NFReplaceContent> GetFileReplaceContents(const std::string& fileName);

protected:

    bool LoadStaticPlugin();
    bool CheckStaticPlugin();

    bool LoadStaticPlugin(const std::string& strPluginDLLName);
    bool LoadPluginLibrary(const std::string& strPluginDLLName);
    bool UnLoadPluginLibrary(const std::string& strPluginDLLName);
    bool UnLoadStaticPlugin(const std::string& strPluginDLLName);

private:
    int mnAppID;
    bool mbIsDocker;
    bool mbStaticPlugin;
    NFINT64 mnInitTime;
    NFINT64 mnNowTime;
    std::string mstrConfigPath;
    std::string mstrConfigName;
    std::string mstrAppName;
    std::string mstrLogConfigName;

    NFIPlugin* mCurrentPlugin;
    NFIModule* mCurrenModule;

    typedef std::map<std::string, bool> PluginNameMap;
    typedef std::map<std::string, NFDynLib*> PluginLibMap;
    typedef std::map<std::string, NFIPlugin*> PluginInstanceMap;
    typedef std::map<std::string, NFIModule*> ModuleInstanceMap;
    typedef std::map<std::string, NFIModule*> TestModuleInstanceMap;

    typedef void(* DLL_START_PLUGIN_FUNC)(NFIPluginManager* pm);
    typedef void(* DLL_STOP_PLUGIN_FUNC)(NFIPluginManager* pm);

    std::vector<std::string> mStaticPlugin;
	std::map<std::string, std::vector<NFReplaceContent>> mReplaceContent;

    PluginNameMap mPluginNameMap;
    PluginLibMap mPluginLibMap;
    PluginInstanceMap mPluginInstanceMap;
    ModuleInstanceMap mModuleInstanceMap;
    TestModuleInstanceMap mTestModuleInstanceMap;

    GET_FILECONTENT_FUNCTOR mGetFileContentFunctor;
};
```

## cpp文件



```c++

//基本把所有的插件头文件都包含了
#include "NFPluginManager.h"
#include "Dependencies/RapidXML/rapidxml.hpp"
#include "Dependencies/RapidXML/rapidxml_iterators.hpp"
#include "Dependencies/RapidXML/rapidxml_print.hpp"
#include "Dependencies/RapidXML/rapidxml_utils.hpp"
#include "NFComm/NFPluginModule/NFIPlugin.h"
#include "NFComm/NFPluginModule/NFPlatform.h"

#if NF_PLATFORM == NF_PLATFORM_WIN
#pragma comment( lib, "opengl32.lib" )
#pragma comment( lib, "ws2_32.lib" )
#pragma comment( lib, "version.lib" )
//#pragma comment( lib, "SDL2maind.lib" )
#pragma comment( lib, "msimg32.lib" )
#pragma comment( lib, "winmm.lib" )
#pragma comment( lib, "imm32.lib" )
//#pragma comment( lib, "msvcrt.lib" )
#pragma comment (lib, "Setupapi.lib")

#ifndef NF_DYNAMIC_PLUGIN
#ifdef NF_DEBUG_MODE
#pragma comment( lib, "libprotobufd.lib" )
#pragma comment( lib, "SDL2d.lib" )
#else
#pragma comment( lib, "libprotobuf.lib" )
#pragma comment( lib, "SDL2.lib" )
#endif

#pragma comment( lib, "NFCore.lib" )
#pragma comment( lib, "NFMessageDefine.lib" )
#pragma comment( lib, "event.lib" )
#pragma comment( lib, "event_core.lib" )
#pragma comment( lib, "lua.lib" )
#pragma comment( lib, "navigation.lib" )
#pragma comment( lib, "hiredis.lib" )


#pragma comment( lib, "NFCore.lib" )
#pragma comment( lib, "NFActorPlugin.lib" )
#pragma comment( lib, "NFConfigPlugin.lib" )
#pragma comment( lib, "NFKernelPlugin.lib" )
#pragma comment( lib, "NFLogPlugin.lib" )
#pragma comment( lib, "NFLuaScriptPlugin.lib" )
#pragma comment( lib, "NFNavigationPlugin.lib" )
#pragma comment( lib, "NFNetPlugin.lib" )
#pragma comment( lib, "NFNoSqlPlugin.lib" )
#pragma comment( lib, "NFSecurityPlugin.lib" )
#pragma comment( lib, "NFTestPlugin.lib" )
#pragma comment( lib, "NFRenderPlugin.lib" )
#pragma comment( lib, "NFBluePrintPlugin.lib" )

#pragma comment( lib, "NFDBLogicPlugin.lib" )
#pragma comment( lib, "NFDBNet_ClientPlugin.lib" )
#pragma comment( lib, "NFDBNet_ServerPlugin.lib" )

#pragma comment( lib, "NFGameServerPlugin.lib" )
#pragma comment( lib, "NFGameServerNet_ClientPlugin.lib" )
#pragma comment( lib, "NFGameServerNet_ServerPlugin.lib" )

#pragma comment( lib, "NFLoginLogicPlugin.lib" )
#pragma comment( lib, "NFLoginNet_ClientPlugin.lib" )
#pragma comment( lib, "NFLoginNet_ServerPlugin.lib" )
#pragma comment( lib, "NFLoginNet_HttpServerPlugin.lib" )

#pragma comment( lib, "NFMasterServerPlugin.lib" )
#pragma comment( lib, "NFMasterNet_ServerPlugin.lib" )
#pragma comment( lib, "NFMasterNet_HttpServerPlugin.lib" )

#pragma comment( lib, "NFProxyLogicPlugin.lib" )
#pragma comment( lib, "NFProxyServerNet_ClientPlugin.lib" )
#pragma comment( lib, "NFProxyServerNet_ServerPlugin.lib" )

#pragma comment( lib, "NFWorldNet_ClientPlugin.lib" )
#pragma comment( lib, "NFWorldNet_ServerPlugin.lib" )

#endif

#endif

#ifdef NF_DEBUG_MODE

#if NF_PLATFORM == NF_PLATFORM_WIN

#elif NF_PLATFORM == NF_PLATFORM_LINUX || NF_PLATFORM == NF_PLATFORM_ANDROID
#elif NF_PLATFORM == NF_PLATFORM_APPLE || NF_PLATFORM == NF_PLATFORM_APPLE_IOS
#endif

#else

#if NF_PLATFORM == NF_PLATFORM_WIN

#elif NF_PLATFORM == NF_PLATFORM_LINUX || NF_PLATFORM == NF_PLATFORM_ANDROID
#elif NF_PLATFORM == NF_PLATFORM_APPLE || NF_PLATFORM == NF_PLATFORM_APPLE_IOS
#endif

#endif


#ifndef NF_DYNAMIC_PLUGIN
//for nf-sdk plugins
#include "NFComm/NFActorPlugin/NFActorPlugin.h"
#include "NFComm/NFConfigPlugin/NFConfigPlugin.h"
#include "NFComm/NFKernelPlugin/NFKernelPlugin.h"
#include "NFComm/NFLogPlugin/NFLogPlugin.h"
#include "NFComm/NFLuaScriptPlugin/NFLuaScriptPlugin.h"
#include "NFComm/NFNavigationPlugin/NFNavigationPlugin.h"
#include "NFComm/NFNetPlugin/NFNetPlugin.h"
#include "NFComm/NFNoSqlPlugin/NFNoSqlPlugin.h"
#include "NFComm/NFSecurityPlugin/NFSecurityPlugin.h"
#include "NFComm/NFTestPlugin/NFTestPlugin.h"

#if NF_PLATFORM != NF_PLATFORM_LINUX
#include "NFComm/NFRenderPlugin/NFRenderPlugin.h"
#include "NFComm/NFBluePrintPlugin/NFBluePrintPlugin.h"
#endif

//DB
#include "NFServer/NFDBLogicPlugin/NFDBLogicPlugin.h"
#include "NFServer/NFDBNet_ClientPlugin/NFDBNet_ClientPlugin.h"
#include "NFServer/NFDBNet_ServerPlugin/NFDBNet_ServerPlugin.h"
//GAME
#include "NFServer/NFGameServerNet_ClientPlugin/NFGameServerNet_ClientPlugin.h"
#include "NFServer/NFGameServerNet_ServerPlugin/NFGameServerNet_ServerPlugin.h"
#include "NFServer/NFGameServerPlugin/NFGameServerPlugin.h"
//LOGIN
#include "NFServer/NFLoginLogicPlugin/NFLoginLogicPlugin.h"
#include "NFServer/NFLoginNet_ClientPlugin/NFLoginNet_ClientPlugin.h"
#include "NFServer/NFLoginNet_ServerPlugin/NFLoginNet_ServerPlugin.h"
#include "NFServer/NFLoginNet_HttpServerPlugin/NFLoginNet_HttpServerPlugin.h"
//MASTER
#include "NFServer/NFMasterNet_HttpServerPlugin/NFMasterNet_HttpServerPlugin.h"
#include "NFServer/NFMasterNet_ServerPlugin/NFMasterNet_ServerPlugin.h"
//PROXY
#include "NFServer/NFProxyLogicPlugin/NFProxyLogicPlugin.h"
#include "NFServer/NFProxyServerNet_ClientPlugin/NFProxyServerNet_ClientPlugin.h"
#include "NFServer/NFProxyServerNet_ServerPlugin/NFProxyServerNet_ServerPlugin.h"
//WORLD
#include "NFServer/NFWorldNet_ClientPlugin/NFWorldNet_ClientPlugin.h"
#include "NFServer/NFWorldNet_ServerPlugin/NFWorldNet_ServerPlugin.h"


#endif

NFPluginManager::NFPluginManager() : NFIPluginManager()
{
	mnAppID = 0;
    mbIsDocker = false;

	mCurrentPlugin = nullptr;
	mCurrenModule = nullptr;

//用宏判定插件是否为静态库
#ifdef NF_DYNAMIC_PLUGIN
	mbStaticPlugin = false;
#else
	mbStaticPlugin = true;
#endif
    //获得启动的初始时间
	mnInitTime = time(NULL);
	mnNowTime = mnInitTime;

	mGetFileContentFunctor = nullptr;

	mstrConfigPath = "../";


   mstrConfigName = "NFDataCfg/Debug/Plugin.xml";

}

NFPluginManager::~NFPluginManager()
{

}

//在LoadPlugin之前会执行LoadPluginConfig()
bool NFPluginManager::LoadPlugin()
{
	std::cout << "----LoadPlugin----" << std::endl;

#ifndef NF_DYNAMIC_PLUGIN
	//创建注册所有的plugin
	LoadStaticPlugin();
#endif
	//根据xml配置,将该server下的plugin名字存入到mStaticPlugin 这个string数组中
	PluginNameMap::iterator it = mPluginNameMap.begin();
	for (; it != mPluginNameMap.end(); ++it)
	{
#ifndef NF_DYNAMIC_PLUGIN
		//根据xml配置,将该server下的plugin存入到mStaticPlugin 这个string数组中
		LoadStaticPlugin(it->first);
#else
		//加载名字对应的动态库
		LoadPluginLibrary(it->first);
#endif
	}

#ifndef NF_DYNAMIC_PLUGIN
	//判断mStaticPlugin和mPluginInstanceMap中内容是否一致 等等。
	CheckStaticPlugin();
#endif

	return true;
}

bool NFPluginManager::Awake()
{
	std::cout << "----Awake----" << std::endl;

	PluginInstanceMap::iterator itAfterInstance = mPluginInstanceMap.begin();
	for (; itAfterInstance != mPluginInstanceMap.end(); itAfterInstance++)
	{
		SetCurrentPlugin(itAfterInstance->second);
		//这个直接调用的是NFIPlugin的awake(),而它的awake则是遍历NFMap中已添加的module，顺便调用modul的awake()
		itAfterInstance->second->Awake();
	}

	return true;
}


inline bool NFPluginManager::Init()
{
	std::cout << "----Init----" << std::endl;

	PluginInstanceMap::iterator itInstance = mPluginInstanceMap.begin();
	for (; itInstance != mPluginInstanceMap.end(); itInstance++)
	{
		SetCurrentPlugin(itInstance->second);
		std::cout << itInstance->first << std::endl;
		//这个直接调用的是NFIPlugin的Init(),而它的Init则是遍历NFMap中已添加的module，顺便调用modul的Init()
		itInstance->second->Init();
	}

	return true;
}

bool NFPluginManager::LoadPluginConfig()
{
	std::string strContent;
    // strFilePath的值为 ../NFDataCfg/Debug/Plugin.xml
	std::string strFilePath = GetConfigPath() + mstrConfigName;
    //文本内容保存在 strContent中
	GetFileContent(strFilePath, strContent);

	rapidxml::xml_document<> xDoc;
	xDoc.parse<0>((char*)strContent.c_str());

    rapidxml::xml_node<>* pRoot = xDoc.first_node();
    //看xml文件可以知道,这获取的是程序的节点，比如GameServer节点
    rapidxml::xml_node<>* pAppNameNode = pRoot->first_node(mstrAppName.c_str());
    if (pAppNameNode)
    {
        // 加载每个Server下的plugin
		for (rapidxml::xml_node<>* pPluginNode = pAppNameNode->first_node("Plugin"); pPluginNode; pPluginNode = pPluginNode->next_sibling("Plugin"))
		{
			const char* strPluginName = pPluginNode->first_attribute("Name")->value();
            // 将plugin名字保存到Map中
			mPluginNameMap.insert(PluginNameMap::value_type(strPluginName, true));

			//std::cout << strPluginName << std::endl;
		}
    }
	else
	{
        //如果没有找到相对应的Server节点，则往下继续查找xml中的Server节点，这里有个问题是：它会把所有server的所有plugin全部加进去
		for (rapidxml::xml_node<>* pServerNode = pRoot->first_node(); pServerNode; pServerNode = pServerNode->next_sibling())
		{
			for (rapidxml::xml_node<>* pPluginNode = pServerNode->first_node("Plugin"); pPluginNode; pPluginNode = pPluginNode->next_sibling("Plugin"))
			{
				const char* strPluginName = pPluginNode->first_attribute("Name")->value();
				if (mPluginNameMap.find(strPluginName) == mPluginNameMap.end())
				{
					mPluginNameMap.insert(PluginNameMap::value_type(strPluginName, true));
				}
			}
		}
	}

    return true;
}

//静态库模式下，会把公共的plugin和各个server的plugin注册到NFPluginManager里面
bool NFPluginManager::LoadStaticPlugin()
{

#ifndef NF_DYNAMIC_PLUGIN

//for nf-sdk plugins
	CREATE_PLUGIN(this, NFActorPlugin)
	CREATE_PLUGIN(this, NFConfigPlugin)
	CREATE_PLUGIN(this, NFKernelPlugin)
	CREATE_PLUGIN(this, NFLogPlugin)
	CREATE_PLUGIN(this, NFLuaScriptPlugin)
	CREATE_PLUGIN(this, NFNavigationPlugin)
	CREATE_PLUGIN(this, NFNetPlugin)
	CREATE_PLUGIN(this, NFNoSqlPlugin)
	CREATE_PLUGIN(this, NFSecurityPlugin)
	CREATE_PLUGIN(this, NFTestPlugin)

#if NF_PLATFORM == NF_PLATFORM_APPLE || NF_PLATFORM == NF_PLATFORM_WIN
	CREATE_PLUGIN(this, NFRenderPlugin)
	CREATE_PLUGIN(this, NFBluePrintPlugin)
#endif
		
//DB
	CREATE_PLUGIN(this, NFDBLogicPlugin)
	CREATE_PLUGIN(this, NFDBNet_ClientPlugin)
	CREATE_PLUGIN(this, NFDBNet_ServerPlugin)

//GAME
	CREATE_PLUGIN(this, NFGameServerNet_ClientPlugin)
	CREATE_PLUGIN(this, NFGameServerNet_ServerPlugin)
	CREATE_PLUGIN(this, NFGameServerPlugin)

//LOGIN
	CREATE_PLUGIN(this, NFLoginLogicPlugin)
	CREATE_PLUGIN(this, NFLoginNet_ClientPlugin)
	CREATE_PLUGIN(this, NFLoginNet_ServerPlugin)
	CREATE_PLUGIN(this, NFLoginNet_HttpServerPlugin)

//MASTER
	CREATE_PLUGIN(this, NFMasterNet_HttpServerPlugin)
	CREATE_PLUGIN(this, NFMasterNet_ServerPlugin)

//PROXY
	CREATE_PLUGIN(this, NFProxyLogicPlugin)
	CREATE_PLUGIN(this, NFProxyServerNet_ClientPlugin)
	CREATE_PLUGIN(this, NFProxyServerNet_ServerPlugin)

//WORLD
	CREATE_PLUGIN(this, NFWorldNet_ClientPlugin)
	CREATE_PLUGIN(this, NFWorldNet_ServerPlugin)



#endif

    return true;
}

bool NFPluginManager::CheckStaticPlugin()
{
#ifndef NF_DYNAMIC_PLUGIN
	//plugin
	for (auto it = mPluginInstanceMap.begin(); it != mPluginInstanceMap.end();)
	{
		bool bFind = false;
		const std::string& strPluginName = it->first;
		for (int i = 0; i < mStaticPlugin.size(); ++i)
		{
			const std::string& tempPluginName = mStaticPlugin[i];
			if (tempPluginName == strPluginName)
			{
				bFind = true;
				break;
			}
		}

		if (!bFind)
		{
			it = mPluginInstanceMap.erase(it);  
		}
		else
		{
			it++;
		}
	}

	for (auto it = mPluginInstanceMap.begin(); it != mPluginInstanceMap.end(); ++it)
	{
		std::cout << it->first << std::endl;
	}

	std::cout << "-------------" << std::endl;

	//////module
	for (auto it = mModuleInstanceMap.begin(); it != mModuleInstanceMap.end();)
	{
		bool bFind = false;
		const std::string& strModuleName = it->first;

		for (int i = 0; i < mStaticPlugin.size(); ++i)
		{
			const std::string& strPluginName = mStaticPlugin[i];
				
			NFIPlugin* pPlugin = this->FindPlugin(strPluginName);
			if (pPlugin)
			{
				NFIModule* pModule = pPlugin->GetElement(strModuleName);
				if (pModule)
				{
					bFind = true;
					break;
				}
			}
		}

		if (!bFind)
		{
			it = mModuleInstanceMap.erase(it);  
		}
		else
		{
			it++;
		}
	}

	std::cout << "-------------" << std::endl;

	for (auto it = mModuleInstanceMap.begin(); it != mModuleInstanceMap.end(); ++it)
	{
		std::cout << it->first << std::endl;
	}
#endif

    return true;
}

bool NFPluginManager::LoadStaticPlugin(const std::string& strPluginDLLName)
{
	mStaticPlugin.push_back(strPluginDLLName);

	return true;
}

//此函数将会被在CREATE_PLUGIN宏中调用，用以注册new 出来的plugin对象
void NFPluginManager::Registered(NFIPlugin* plugin)
{
    const std::string& strPluginName = plugin->GetPluginName();
    //如果在mPluginInstanceMap没有找到，就保存这个插件
    if (!FindPlugin(strPluginName))
	{
		mPluginInstanceMap.insert(PluginInstanceMap::value_type(strPluginName, plugin));
		//会调用addmodule将module和其名字存入到mModuleInstanceMap中，而且plugin继承于NFMap(内部有个私有map成员变量，用于存储module)
		//addElement()将module存入到NFMap中
        plugin->Install();
    }
	else
	{
        //显示重复保存plugin的名字
		std::cout << strPluginName << std::endl;
		assert(0);
	}
}

void NFPluginManager::UnRegistered(NFIPlugin* plugin)
{
    PluginInstanceMap::iterator it = mPluginInstanceMap.find(plugin->GetPluginName());
    if (it != mPluginInstanceMap.end())
    {
        it->second->Uninstall();
        delete it->second;
        it->second = NULL;
        mPluginInstanceMap.erase(it);
    }
}

bool NFPluginManager::ReLoadPlugin(const std::string & strPluginDLLName)
{
	//1.shut all module of this plugin
	//2.unload this plugin
	//3.load new plugin
	//4.init new module
	//5.tell others who has been reloaded
	PluginInstanceMap::iterator itInstance = mPluginInstanceMap.find(strPluginDLLName);
	if (itInstance == mPluginInstanceMap.end())
	{
		return false;
	}
	//1
	NFIPlugin* pPlugin = itInstance->second;
	NFIModule* pModule = pPlugin->First();
	while (pModule)
	{
		pModule->BeforeShut();
		pModule->Shut();
		pModule->Finalize();

		pModule = pPlugin->Next();
	}

	//2
	PluginLibMap::iterator it = mPluginLibMap.find(strPluginDLLName);
	if (it != mPluginLibMap.end())
	{
		NFDynLib* pLib = it->second;
		DLL_STOP_PLUGIN_FUNC pFunc = (DLL_STOP_PLUGIN_FUNC)pLib->GetSymbol("DllStopPlugin");

		if (pFunc)
		{
			pFunc(this);
		}

		pLib->UnLoad();

		delete pLib;
		pLib = NULL;
		mPluginLibMap.erase(it);
	}

	//3
	NFDynLib* pLib = new NFDynLib(strPluginDLLName);
	bool bLoad = pLib->Load();
	if (bLoad)
	{
		mPluginLibMap.insert(PluginLibMap::value_type(strPluginDLLName, pLib));

		DLL_START_PLUGIN_FUNC pFunc = (DLL_START_PLUGIN_FUNC)pLib->GetSymbol("DllStartPlugin");
		if (!pFunc)
		{
			std::cout << "Reload Find function DllStartPlugin Failed in [" << pLib->GetName() << "]" << std::endl;
			assert(0);
			return false;
		}

		pFunc(this);
	}
	else
	{
#if NF_PLATFORM == NF_PLATFORM_LINUX
		char* error = dlerror();
		if (error)
		{
			std::cout << stderr << " Reload shared lib[" << pLib->GetName() << "] failed, ErrorNo. = [" << error << "]" << std::endl;
			std::cout << "Reload [" << pLib->GetName() << "] failed" << std::endl;
			assert(0);
			return false;
		}
#elif NF_PLATFORM == NF_PLATFORM_WIN
		std::cout << stderr << " Reload DLL[" << pLib->GetName() << "] failed, ErrorNo. = [" << GetLastError() << "]" << std::endl;
		std::cout << "Reload [" << pLib->GetName() << "] failed" << std::endl;
		assert(0);
		return false;
#endif // NF_PLATFORM
	}

	//4
	PluginInstanceMap::iterator itReloadInstance = mPluginInstanceMap.begin();
	for (; itReloadInstance != mPluginInstanceMap.end(); itReloadInstance++)
	{
		if (strPluginDLLName != itReloadInstance->first)
		{
			itReloadInstance->second->OnReloadPlugin();
		}
	}
	return true;
}

NFIPlugin* NFPluginManager::FindPlugin(const std::string& strPluginName)
{
    PluginInstanceMap::iterator it = mPluginInstanceMap.find(strPluginName);
    if (it != mPluginInstanceMap.end())
    {
        return it->second;
    }

    return NULL;
}

bool NFPluginManager::Execute()
{
    mnNowTime = time(NULL);

    bool bRet = true;

    PluginInstanceMap::iterator it = mPluginInstanceMap.begin();
    for (; it != mPluginInstanceMap.end(); ++it)
    {
        bool tembRet = it->second->Execute();
        bRet = bRet && tembRet;
    }

    return bRet;
}

inline int NFPluginManager::GetAppID() const
{
	return mnAppID;
}

inline void NFPluginManager::SetAppID(const int nAppID)
{
    mnAppID = nAppID;
}

bool NFPluginManager::IsRunningDocker() const
{
	return mbIsDocker;
}

void NFPluginManager::SetRunningDocker(bool bDocker)
{
	mbIsDocker = bDocker;
}

bool NFPluginManager::IsStaticPlugin() const
{
	return mbStaticPlugin;
}

inline NFINT64 NFPluginManager::GetInitTime() const
{
	return mnInitTime;
}

inline NFINT64 NFPluginManager::GetNowTime() const
{
	return mnNowTime;
}

inline const std::string & NFPluginManager::GetConfigPath() const
{
	return mstrConfigPath;
}

inline void NFPluginManager::SetConfigPath(const std::string & strPath)
{
	mstrConfigPath = strPath;
}

void NFPluginManager::SetConfigName(const std::string & strFileName)
{
	if (strFileName.empty())
	{
		return;
	}

	if (strFileName.find(".xml") == string::npos)
	{
		return;
	}

#ifdef NF_DEBUG_MODE
	mstrConfigName = "NFDataCfg/Debug/" + strFileName;
#else
	mstrConfigName = "NFDataCfg/Release/" + strFileName;
#endif
}

const std::string& NFPluginManager::GetConfigName() const
{
	return mstrConfigName;
}

const std::string& NFPluginManager::GetAppName() const
{
	return mstrAppName;
}

void NFPluginManager::SetAppName(const std::string& strAppName)
{
	if (!mstrAppName.empty())
	{
		return;
	}

	mstrAppName = strAppName;
}

const std::string & NFPluginManager::GetLogConfigName() const
{
	return mstrLogConfigName;
}

void NFPluginManager::SetLogConfigName(const std::string & strName)
{
	mstrLogConfigName = strName;
}

NFIPlugin * NFPluginManager::GetCurrentPlugin()
{
	return mCurrentPlugin;
}

NFIModule * NFPluginManager::GetCurrenModule()
{
	return mCurrenModule;
}

void NFPluginManager::SetCurrentPlugin(NFIPlugin * pPlugin)
{
	 mCurrentPlugin = pPlugin;
}

void NFPluginManager::SetCurrenModule(NFIModule * pModule)
{
	mCurrenModule = pModule;
}

void NFPluginManager::SetGetFileContentFunctor(GET_FILECONTENT_FUNCTOR fun)
{
	mGetFileContentFunctor = fun;
}

//先执行SetGetFileContentFunctor 设置好需要执行读取文件内容的函数，就可以根据自定义的函数来读取文件的内容
bool NFPluginManager::GetFileContent(const std::string &strFileName, std::string &strContent)
{
	if (mGetFileContentFunctor)
	{
		return mGetFileContentFunctor(this, strFileName, strContent);
	}

	FILE *fp = fopen(strFileName.c_str(), "rb");
	if (!fp)
	{
		return false;
	}

	fseek(fp, 0, SEEK_END);
	const long filelength = ftell(fp);
	fseek(fp, 0, SEEK_SET);
	strContent.resize(filelength);
	fread((void*)strContent.data(), filelength, 1, fp);
	fclose(fp);

	return true;
}

void NFPluginManager::AddModule(const std::string& strModuleName, NFIModule* pModule)
{
    if (!FindModule(strModuleName))
    {
        mModuleInstanceMap.insert(ModuleInstanceMap::value_type(strModuleName, pModule));
    }
}

void NFPluginManager::AddTestModule(const std::string& strModuleName, NFIModule* pModule)
{
	if (!FindTestModule(strModuleName))
	{
		mTestModuleInstanceMap.insert(TestModuleInstanceMap::value_type(strModuleName, pModule));
	}
}

void NFPluginManager::RemoveModule(const std::string& strModuleName)
{
    ModuleInstanceMap::iterator it = mModuleInstanceMap.find(strModuleName);
    if (it != mModuleInstanceMap.end())
    {
        mModuleInstanceMap.erase(it);
    }
}


NFIModule* NFPluginManager::FindModule(const std::string& strModuleName)
{
	std::string strSubModuleName = strModuleName;

#if NF_PLATFORM == NF_PLATFORM_WIN
	std::size_t position = strSubModuleName.find(" ");
	if (string::npos != position)
	{
		strSubModuleName = strSubModuleName.substr(position + 1, strSubModuleName.length());
	}
#else
	for (int i = 0; i < strSubModuleName.length(); i++)
	{
		std::string s = strSubModuleName.substr(0, i + 1);
		int n = atof(s.c_str());
		if (strSubModuleName.length() == i + 1 + n)
		{
			strSubModuleName = strSubModuleName.substr(i + 1, strSubModuleName.length());
			break;
		}
	}
#endif

	ModuleInstanceMap::iterator it = mModuleInstanceMap.find(strSubModuleName);
	if (it != mModuleInstanceMap.end())
	{
		return it->second;
	}
	
	if (this->GetCurrenModule())
	{
		std::cout << this->GetCurrenModule()->strName << " want to find module: " << strModuleName << " but got null_ptr!!!" << std::endl;
	}

    return NULL;
}

NFIModule* NFPluginManager::FindTestModule(const std::string& strModuleName)
{
	std::string strSubModuleName = strModuleName;

#if NF_PLATFORM == NF_PLATFORM_WIN
	std::size_t position = strSubModuleName.find(" ");
	if (string::npos != position)
	{
		strSubModuleName = strSubModuleName.substr(position + 1, strSubModuleName.length());
	}
#else
	for (int i = 0; i < strSubModuleName.length(); i++)
	{
		std::string s = strSubModuleName.substr(0, i + 1);
		int n = atof(s.c_str());
		if (strSubModuleName.length() == i + 1 + n)
		{
			strSubModuleName = strSubModuleName.substr(i + 1, strSubModuleName.length());
			break;
		}
	}
#endif

    TestModuleInstanceMap::iterator it = mTestModuleInstanceMap.find(strSubModuleName);
	if (it != mTestModuleInstanceMap.end())
	{
		return it->second;
	}

	return NULL;
}

std::list<NFIModule*> NFPluginManager::Modules()
{
	std::list<NFIModule*> xModules;

	ModuleInstanceMap::iterator itCheckInstance = mModuleInstanceMap.begin();
	for (; itCheckInstance != mModuleInstanceMap.end(); itCheckInstance++)
	{
		xModules.push_back(itCheckInstance->second);
	}

	return xModules;
}

bool NFPluginManager::AfterInit()
{
	std::cout << "----AfterInit----" << std::endl;

    PluginInstanceMap::iterator itAfterInstance = mPluginInstanceMap.begin();
    for (; itAfterInstance != mPluginInstanceMap.end(); itAfterInstance++)
    {
		SetCurrentPlugin(itAfterInstance->second);
        itAfterInstance->second->AfterInit();
    }

    return true;
}

bool NFPluginManager::CheckConfig()
{
	std::cout << "----CheckConfig----" << std::endl;

    PluginInstanceMap::iterator itCheckInstance = mPluginInstanceMap.begin();
    for (; itCheckInstance != mPluginInstanceMap.end(); itCheckInstance++)
    {
		SetCurrentPlugin(itCheckInstance->second);
        itCheckInstance->second->CheckConfig();
    }

    return true;
}

bool NFPluginManager::ReadyExecute()
{
	std::cout << "----ReadyExecute----" << std::endl;

    PluginInstanceMap::iterator itCheckInstance = mPluginInstanceMap.begin();
    for (; itCheckInstance != mPluginInstanceMap.end(); itCheckInstance++)
    {
		SetCurrentPlugin(itCheckInstance->second);
        itCheckInstance->second->ReadyExecute();
    }

    return true;
}

bool NFPluginManager::BeforeShut()
{
    PluginInstanceMap::iterator itBeforeInstance = mPluginInstanceMap.begin();
    for (; itBeforeInstance != mPluginInstanceMap.end(); itBeforeInstance++)
    {
		SetCurrentPlugin(itBeforeInstance->second);
        itBeforeInstance->second->BeforeShut();
    }

    return true;
}

bool NFPluginManager::Shut()
{
    PluginInstanceMap::iterator itInstance = mPluginInstanceMap.begin();
    for (; itInstance != mPluginInstanceMap.end(); ++itInstance)
    {
		SetCurrentPlugin(itInstance->second);
        itInstance->second->Shut();
    }

    return true;
}

bool NFPluginManager::Finalize()
{
	PluginInstanceMap::iterator itInstance = mPluginInstanceMap.begin();
	for (; itInstance != mPluginInstanceMap.end(); itInstance++)
	{
		SetCurrentPlugin(itInstance->second);
		itInstance->second->Finalize();
	}

	////////////////////////////////////////////////

	PluginNameMap::iterator it = mPluginNameMap.begin();
	for (; it != mPluginNameMap.end(); it++)
	{
#ifdef NF_DYNAMIC_PLUGIN
		UnLoadPluginLibrary(it->first);
#else
		UnLoadStaticPlugin(it->first);
#endif
	}

	mPluginInstanceMap.clear();
	mPluginNameMap.clear();

	return true;
}

bool NFPluginManager::LoadPluginLibrary(const std::string& strPluginDLLName)
{
    PluginLibMap::iterator it = mPluginLibMap.find(strPluginDLLName);
    if (it == mPluginLibMap.end())
    {
        NFDynLib* pLib = new NFDynLib(strPluginDLLName);
        bool bLoad = pLib->Load();

        if (bLoad)
        {
            mPluginLibMap.insert(PluginLibMap::value_type(strPluginDLLName, pLib));

            DLL_START_PLUGIN_FUNC pFunc = (DLL_START_PLUGIN_FUNC)pLib->GetSymbol("DllStartPlugin");
            if (!pFunc)
            {
                std::cout << "Find function DllStartPlugin Failed in [" << pLib->GetName() << "]" << std::endl;
                assert(0);
                return false;
            }

            pFunc(this);

            return true;
        }
        else
        {
#if NF_PLATFORM == NF_PLATFORM_LINUX || NF_PLATFORM == NF_PLATFORM_APPLE
            char* error = dlerror();
            if (error)
            {
                std::cout << stderr << " Load shared lib[" << pLib->GetName() << "] failed, ErrorNo. = [" << error << "]" << std::endl;
                std::cout << "Load [" << pLib->GetName() << "] failed" << std::endl;
                assert(0);
                return false;
            }
#elif NF_PLATFORM == NF_PLATFORM_WIN
            std::cout << stderr << " Load DLL[" << pLib->GetName() << "] failed, ErrorNo. = [" << GetLastError() << "]" << std::endl;
            std::cout << "Load [" << pLib->GetName() << "] failed" << std::endl;
            assert(0);
            return false;
#endif // NF_PLATFORM
        }
    }

    return false;
}

bool NFPluginManager::UnLoadPluginLibrary(const std::string& strPluginDLLName)
{
    PluginLibMap::iterator it = mPluginLibMap.find(strPluginDLLName);
    if (it != mPluginLibMap.end())
    {
        NFDynLib* pLib = it->second;
        DLL_STOP_PLUGIN_FUNC pFunc = (DLL_STOP_PLUGIN_FUNC)pLib->GetSymbol("DllStopPlugin");

        if (pFunc)
        {
            pFunc(this);
        }

        pLib->UnLoad();

        delete pLib;
        pLib = NULL;
        mPluginLibMap.erase(it);

        return true;
    }

    return false;
}

bool NFPluginManager::UnLoadStaticPlugin(const std::string & strPluginDLLName)
{
	//     DESTROY_PLUGIN(this, NFConfigPlugin)
	//     DESTROY_PLUGIN(this, NFEventProcessPlugin)
	//     DESTROY_PLUGIN(this, NFKernelPlugin)
	return false;
}

void NFPluginManager::AddFileReplaceContent(const std::string& fileName, const std::string& content, const std::string& newValue)
{
	auto it = mReplaceContent.find(fileName);
	if (it == mReplaceContent.end())
	{
		std::vector<NFReplaceContent> v;
		v.push_back(NFReplaceContent(content, newValue));

		mReplaceContent.insert(std::pair<std::string, std::vector<NFReplaceContent> >(fileName, v));
	}
	else
	{
		it->second.push_back(NFReplaceContent(content, newValue));
	}
}

std::vector<NFReplaceContent> NFPluginManager::GetFileReplaceContents(const std::string& fileName)
{
	auto it = mReplaceContent.find(fileName);
	if (it != mReplaceContent.end())
	{
		return it->second;
	}

	return std::vector<NFReplaceContent>();
}

```