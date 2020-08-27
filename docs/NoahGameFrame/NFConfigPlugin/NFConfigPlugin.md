# NFConfigPlugin

## 头文件
```c++

#include "NFComm/NFPluginModule/NFIPlugin.h"
#include "NFComm/NFPluginModule/NFIPluginManager.h"

//////////////////////////////////////////////////////////////////////////
class NFConfigPlugin : public NFIPlugin
{
public:
    NFConfigPlugin(NFIPluginManager* p);

    virtual const int GetPluginVersion();

    virtual const std::string GetPluginName();

    virtual void Install();

    virtual void Uninstall();
};

```

## cpp文件
```c++

#include "NFConfigPlugin.h"
#include "NFClassModule.h"
#include "NFElementModule.h"
#include "NFCommonConfigModule.h"

#ifdef NF_DYNAMIC_PLUGIN

NF_EXPORT void DllStartPlugin(NFIPluginManager* pm)
{
    CREATE_PLUGIN(pm, NFConfigPlugin)
};

NF_EXPORT void DllStopPlugin(NFIPluginManager* pm)
{
    DESTROY_PLUGIN(pm, NFConfigPlugin)
};

#endif

//////////////////////////////////////////////////////////////////////////
NFConfigPlugin::NFConfigPlugin(NFIPluginManager* p)
{
    pPluginManager = p;
}

const int NFConfigPlugin::GetPluginVersion()
{
    return 0;
}

const std::string NFConfigPlugin::GetPluginName()
{
	return GET_CLASS_NAME(NFConfigPlugin);
}

void NFConfigPlugin::Install()
{
    REGISTER_MODULE(pPluginManager, NFIClassModule, NFClassModule)
    REGISTER_MODULE(pPluginManager, NFIElementModule, NFElementModule)
	REGISTER_MODULE(pPluginManager, NFICommonConfigModule, NFCommonConfigModule);

}

void NFConfigPlugin::Uninstall()
{
    UNREGISTER_MODULE(pPluginManager, NFIElementModule, NFElementModule)
    UNREGISTER_MODULE(pPluginManager, NFIClassModule, NFClassModule)
	UNREGISTER_MODULE(pPluginManager, NFICommonConfigModule, NFCommonConfigModule);
}

```