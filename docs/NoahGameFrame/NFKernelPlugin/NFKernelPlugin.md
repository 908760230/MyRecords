# NFKernelPlugin

## 头文件
```c++
#include "NFComm/NFPluginModule/NFIPlugin.h"
#include "NFComm/NFPluginModule/NFIPluginManager.h"

//////////////////////////////////////////////////////////////////////////
class NFKernelPlugin : public NFIPlugin
{
public:
    NFKernelPlugin(NFIPluginManager* p)
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

最主要的还是在install 函数中注册的几个模块

```c++
#include "NFKernelPlugin.h"
#include "NFKernelModule.h"
#include "NFSceneModule.h"
#include "NFEventModule.h"
#include "NFScheduleModule.h"
#include "NFDataTailModule.h"
#include "NFCellModule.h"
#include "NFThreadPoolModule.h"

#ifdef NF_DYNAMIC_PLUGIN

NF_EXPORT void DllStartPlugin(NFIPluginManager* pm)
{
    CREATE_PLUGIN(pm, NFKernelPlugin)

};

NF_EXPORT void DllStopPlugin(NFIPluginManager* pm)
{
    DESTROY_PLUGIN(pm, NFKernelPlugin)
};

#endif

//////////////////////////////////////////////////////////////////////////

const int NFKernelPlugin::GetPluginVersion()
{
    return 0;
}

const std::string NFKernelPlugin::GetPluginName()
{
	return GET_CLASS_NAME(NFKernelPlugin);
}

void NFKernelPlugin::Install()
{
    REGISTER_MODULE(pPluginManager, NFISceneModule, NFSceneModule)
	REGISTER_MODULE(pPluginManager, NFIKernelModule, NFKernelModule)
	REGISTER_MODULE(pPluginManager, NFIEventModule, NFEventModule)
	REGISTER_MODULE(pPluginManager, NFIScheduleModule, NFScheduleModule)
	REGISTER_MODULE(pPluginManager, NFIDataTailModule, NFDataTailModule)
	REGISTER_MODULE(pPluginManager, NFICellModule, NFCellModule)
	REGISTER_MODULE(pPluginManager, NFIThreadPoolModule, NFThreadPoolModule)

	/*
	REGISTER_TEST_MODULE(pPluginManager, NFIKernelModule, NFKernelTestModule)
	REGISTER_TEST_MODULE(pPluginManager, NFIEventModule, NFEventTestModule)
	REGISTER_TEST_MODULE(pPluginManager, NFIScheduleModule, NFScheduleTestModule)
	*/
}

void NFKernelPlugin::Uninstall()
{
	/*
	UNREGISTER_TEST_MODULE(pPluginManager, NFIEventModule, NFEventTestModule)
	UNREGISTER_TEST_MODULE(pPluginManager, NFIKernelModule, NFKernelTestModule)
	UNREGISTER_TEST_MODULE(pPluginManager, NFIScheduleModule, NFScheduleTestModule)
*/
	UNREGISTER_MODULE(pPluginManager, NFIThreadPoolModule, NFThreadPoolModule)
	UNREGISTER_MODULE(pPluginManager, NFICellModule, NFCellModule)
	UNREGISTER_MODULE(pPluginManager, NFIDataTailModule, NFDataTailModule)
	UNREGISTER_MODULE(pPluginManager, NFIEventModule, NFEventModule)
	UNREGISTER_MODULE(pPluginManager, NFIKernelModule, NFKernelModule)
	UNREGISTER_MODULE(pPluginManager, NFISceneModule, NFSceneModule)
	UNREGISTER_MODULE(pPluginManager, NFIScheduleModule, NFScheduleModule)

}
```