# NFIModule 头文件解析

NFIModule 是基类，定义了公共的借口。 protected 的 抽象基类 NFIPluginManager，保存了 插件的管理对象指针。


## 完整代码

```c++



#ifndef NFI_MODULE_H
#define NFI_MODULE_H

#include <string>
#include "NFIPluginManager.h"
#include "NFComm/NFCore/NFMap.hpp"
#include "NFComm/NFCore/NFList.hpp"
#include "NFComm/NFCore/NFDataList.hpp"
#include "NFComm/NFCore/NFSmartEnum.hpp"

class NFIModule
{

public:
    NFIModule() : m_bIsExecute(false), pPluginManager(NULL)
    {
    }

    virtual ~NFIModule() {}

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

    virtual NFIPluginManager* GetPluginManager() const
    {
        return pPluginManager;
    }

    std::string strName;
    bool m_bIsExecute;
protected:
	NFIPluginManager* pPluginManager;
};
#endif



```