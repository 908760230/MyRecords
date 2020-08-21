# NFActorPlugin

## 头文件
```c++

#include "NFComm/NFPluginModule/NFIPlugin.h"
#include "NFComm/NFPluginModule/NFIPluginManager.h"

//////////////////////////////////////////////////////////////////////////
class NFActorPlugin : public NFIPlugin
{
public:
	NFActorPlugin(NFIPluginManager* p)
    {
        pPluginManager = p;
    }

    virtual const int GetPluginVersion();

    virtual const std::string GetPluginName();

    virtual void Install();

    virtual void Uninstall();
};

```

## cpp文件
注册NFActorModule。
如果 定义了 NF_DYNAMIC_PLUGIN宏，那么会加载动态库。

```c++

#include "NFActorPlugin.h"
#include "NFActorModule.h"

#ifdef NF_DYNAMIC_PLUGIN

NF_EXPORT void DllStartPlugin(NFIPluginManager* pm)
{
    CREATE_PLUGIN(pm, NFActorPlugin)
};

NF_EXPORT void DllStopPlugin(NFIPluginManager* pm)
{
    DESTROY_PLUGIN(pm, NFActorPlugin)
};

#endif

//////////////////////////////////////////////////////////////////////////

const int NFActorPlugin::GetPluginVersion()
{
    return 0;
}

const std::string NFActorPlugin::GetPluginName()
{
	return GET_CLASS_NAME(NFActorPlugin);
}

void NFActorPlugin::Install()
{
	REGISTER_MODULE(pPluginManager, NFIActorModule, NFActorModule)
}

void NFActorPlugin::Uninstall()
{
	UNREGISTER_MODULE(pPluginManager, NFIActorModule, NFActorModule)
}
```