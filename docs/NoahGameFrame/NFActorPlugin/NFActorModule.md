# NFActorModule

## 头文件

```c++

#ifndef NF_ACTOR_MANAGER_H
#define NF_ACTOR_MANAGER_H

#include <map>
#include <string>
#include <queue>
#include "NFActor.h"
#include "NFComm/NFPluginModule/NFIComponent.h"
#include "NFComm/NFPluginModule/NFIActorModule.h"
#include "NFComm/NFPluginModule/NFIKernelModule.h"
#include "NFComm/NFPluginModule/NFIThreadPoolModule.h"
#include "NFComm/NFCore/NFQueue.hpp"

class NFActorModule
    : public NFIActorModule
{
public:
	NFActorModule(NFIPluginManager* p);
    virtual ~NFActorModule();

    virtual bool Init();

    virtual bool AfterInit();

    virtual bool BeforeShut();

    virtual bool Shut();

    virtual bool Execute();

	virtual NF_SHARE_PTR<NFIActor> RequireActor();
	virtual NF_SHARE_PTR<NFIActor> GetActor(const NFGUID nActorIndex);
	virtual bool ReleaseActor(const NFGUID nActorIndex);

    virtual bool SendMsgToActor(const NFGUID actorIndex, const NFGUID who, const int eventID, const std::string& data, const std::string& arg = "");

	virtual bool AddResult(const NFActorMessage& message);

protected:
    virtual bool SendMsgToActor(const NFGUID actorIndex, const NFActorMessage& message);

	virtual bool AddEndFunc(const int subMessageID, ACTOR_PROCESS_FUNCTOR_PTR functorPtr_end);

	virtual bool ExecuteEvent();
	virtual bool ExecuteResultEvent();

private:
    bool test = false;

    NFIKernelModule* m_pKernelModule;
    NFIThreadPoolModule* m_pThreadPoolModule;

	std::map<NFGUID, NF_SHARE_PTR<NFIActor>> mxActorMap;

	NFQueue<NFActorMessage> mxResultQueue;
	NFMapEx<int, ACTOR_PROCESS_FUNCTOR> mxEndFunctor;
};

#endif
```


## cpp文件

```c++

#include "NFActorModule.h"

NFActorModule::NFActorModule(NFIPluginManager* p)
{
	pPluginManager = p;
    //初始化随机函数
    srand((unsigned)time(NULL));
}

NFActorModule::~NFActorModule()
{
}

bool NFActorModule::Init()
{
	m_pKernelModule = pPluginManager->FindModule<NFIKernelModule>();
	m_pThreadPoolModule = pPluginManager->FindModule<NFIThreadPoolModule>();

    return true;
}

bool NFActorModule::AfterInit()
{

    return true;
}

bool NFActorModule::BeforeShut()
{
	mxActorMap.clear();
    return true;
}

bool NFActorModule::Shut()
{
 
    return true;
}

bool NFActorModule::Execute()
{
	ExecuteEvent();
	ExecuteResultEvent();

    return true;
}


NF_SHARE_PTR<NFIActor> NFActorModule::RequireActor()
{
	NF_SHARE_PTR<NFIActor> pActor = NF_SHARE_PTR<NFIActor>(NF_NEW NFActor(m_pKernelModule->CreateGUID(), this));
	mxActorMap.insert(std::map<NFGUID, NF_SHARE_PTR<NFIActor>>::value_type(pActor->ID(), pActor));

	return pActor;
}

NF_SHARE_PTR<NFIActor> NFActorModule::GetActor(const NFGUID nActorIndex)
{
	auto it = mxActorMap.find(nActorIndex);
	if (it != mxActorMap.end())
	{
		return it->second;
	}

	return nullptr;
}

bool NFActorModule::AddResult(const NFActorMessage & message)
{
    //只是将结果压入栈中
	return mxResultQueue.Push(message);
}

bool NFActorModule::ExecuteEvent()
{
	for (auto it : mxActorMap)
	{
		NF_SHARE_PTR<NFIActor> pActor = it.second;
		if (pActor)
		{   //测试状态下 直接执行
			if (test)
			{
				pActor->Execute();
			}
			else
			{   //在线程池内执行
				m_pThreadPoolModule->DoAsyncTask(pActor->ID(), "",
					[pActor](NFThreadTask& threadTask) -> void
					{
						pActor->Execute();
					});
			}
		}
	}
	
	return true;
}

bool NFActorModule::ExecuteResultEvent()
{
	NFActorMessage actorMessage;
	while (mxResultQueue.try_dequeue(actorMessage))
	{
		ACTOR_PROCESS_FUNCTOR_PTR functorPtr_end = mxEndFunctor.GetElement(actorMessage.msgID);
		if (functorPtr_end)
		{
			functorPtr_end->operator()(actorMessage);
		}
	}
	
	return true;
}

bool NFActorModule::SendMsgToActor(const NFGUID actorIndex, const NFGUID who, const int eventID, const std::string& data, const std::string& arg)
{
	static uint64_t index = 0;
    NF_SHARE_PTR<NFIActor> pActor = GetActor(actorIndex);
    if (nullptr != pActor)
    {
        NFActorMessage xMessage;

		xMessage.id = who;
		xMessage.index	= index++;
        xMessage.data = data;
		xMessage.msgID = eventID;
		xMessage.arg = arg;


		return this->SendMsgToActor(actorIndex, xMessage);
    }

    return false;
}

bool NFActorModule::ReleaseActor(const NFGUID nActorIndex)
{
	auto it = mxActorMap.find(nActorIndex);
	if (it != mxActorMap.end())
	{
		mxActorMap.erase(it);
	
		return true;
	}
	
	return false;
}

bool NFActorModule::AddEndFunc(const int subMessageID, ACTOR_PROCESS_FUNCTOR_PTR functorPtr_end)
{
	return mxEndFunctor.AddElement(subMessageID, functorPtr_end);
}

bool NFActorModule::SendMsgToActor(const NFGUID actorIndex, const NFActorMessage &message)
{
	auto it = mxActorMap.find(actorIndex);
	if (it != mxActorMap.end())
	{
		//std::cout << "send message " << message.msgID << " to " << actorIndex.ToString() << " and msg index is " << message.index << std::endl;
		// 将message加入MessageQueue中
		return it->second->SendMsg(message);
	}

	return false;
}

```