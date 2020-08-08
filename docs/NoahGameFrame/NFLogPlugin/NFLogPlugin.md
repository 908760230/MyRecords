# NFLogPlugin 解析

## NFLogPlugin.h 

```c++
#include "NFComm/NFPluginModule/NFIPlugin.h"
#include "NFComm/NFPluginModule/NFIPluginManager.h"

//////////////////////////////////////////////////////////////////////////
class NFLogPlugin : public NFIPlugin
{
public:
    NFLogPlugin(NFIPluginManager* p)
    {
        pPluginManager = p;
    }

    virtual const int GetPluginVersion();

    virtual const std::string GetPluginName();

    virtual void Install();

    virtual void Uninstall();
};

```
对 继承自NFIPlugin的接口进行重写。

## NFLogPlugin.cpp

```c++


#include "NFLogPlugin.h"
#include "NFLogModule.h"

#ifdef NF_DYNAMIC_PLUGIN

NF_EXPORT void DllStartPlugin(NFIPluginManager* pm)
{
    CREATE_PLUGIN(pm, NFLogPlugin)
};

NF_EXPORT void DllStopPlugin(NFIPluginManager* pm)
{
    DESTROY_PLUGIN(pm, NFLogPlugin)
};

#endif

//////////////////////////////////////////////////////////////////////////

const int NFLogPlugin::GetPluginVersion()
{
    return 0;
}

const std::string NFLogPlugin::GetPluginName()
{
	return GET_CLASS_NAME(NFLogPlugin);
}

void NFLogPlugin::Install()
{
    REGISTER_MODULE(pPluginManager, NFILogModule, NFLogModule)
}

void NFLogPlugin::Uninstall()
{
    UNREGISTER_MODULE(pPluginManager, NFILogModule, NFLogModule)
}

```

### GET_CLASS_NAME

可以在 NFPlatform 头文件 看到 GET_CLASS_NAME 宏定义。
```c++

#define GET_CLASS_NAME(className) (#className)

```
通过 宏 将className 转化成字符串


### REGISTER_MODULE 和 UNREGISTER_MODULE 

```c++

#define REGISTER_MODULE(pManager, classBaseName, className)  \
	assert((TIsDerived<classBaseName, NFIModule>::Result));	\
	assert((TIsDerived<className, classBaseName>::Result));	\
	NFIModule* pRegisterModule##className= new className(pManager); \
    pRegisterModule##className->strName = (#className); \
    pManager->AddModule( #classBaseName, pRegisterModule##className );\
    this->AddElement( #classBaseName, pRegisterModule##className );

    
#define UNREGISTER_MODULE(pManager, classBaseName, className) \
    NFIModule* pUnRegisterModule##className = dynamic_cast<NFIModule*>( pManager->FindModule( #classBaseName )); \
	pManager->RemoveModule( #classBaseName ); \
    this->RemoveElement( #classBaseName ); \
    delete pUnRegisterModule##className;
```

在NFPluginManager 里对 mModuleInstanceMap 进行 添加 or 删除 module。
因为 NFLogModule 继承与 NFIModule  所以 会将 NFILogModule字符串为key  NFLogModule对象指针为value，存入到 NFIModule 的map 里面。