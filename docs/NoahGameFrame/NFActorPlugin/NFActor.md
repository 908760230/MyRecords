# NFActor

## 头文件
```c++
#ifndef NF_ACTOR_H
#define NF_ACTOR_H

#include <map>
#include <string>
#include "NFComm/NFCore/NFQueue.hpp"
#include "NFComm/NFPluginModule/NFGUID.h"
#include "NFComm/NFPluginModule/NFPlatform.h"
#include "NFComm/NFPluginModule/NFIComponent.h"
#include "NFComm/NFPluginModule/NFIActorModule.h"

class NFActor
    : public NFIActor
{
public:
	NFActor(const NFGUID id, NFIActorModule* pModule);
	virtual ~NFActor();

	const NFGUID ID();
	
    virtual bool Execute();

    virtual bool AddComponent(NF_SHARE_PTR<NFIComponent> pComponent);
	virtual bool RemoveComponent(const std::string& strComponentName);
	virtual NF_SHARE_PTR<NFIComponent> FindComponent(const std::string& strComponentName);

    virtual bool SendMsg(const NFActorMessage& message);
    virtual bool SendMsg(const int eventID, const std::string& data, const std::string& arg);
    virtual bool BackMsgToMainThread(const NFActorMessage& message);

    virtual bool AddMessageHandler(const int nSubMsgID, ACTOR_PROCESS_FUNCTOR_PTR xBeginFunctor);

    virtual void ToMemoryCounterString(std::string& info);
protected:
	NFGUID id;

	NFIActorModule* m_pActorModule;
    // 无锁队列
    NFQueue<NFActorMessage> mMessageQueue;

	NFMapEx<std::string, NFIComponent> mComponent;

	NFMapEx<int, ACTOR_PROCESS_FUNCTOR> mxProcessFunctor;
};
#endif
```


## cpp文件

```c++

#include "NFActor.h"
#include "NFComm/NFPluginModule/NFIPluginManager.h"

NFActor::NFActor(const NFGUID id, NFIActorModule* pModule)
{
	this->id = id;
	m_pActorModule = pModule;
}

NFActor::~NFActor()
{
}

const NFGUID NFActor::ID()
{
	return this->id;
}

bool NFActor::Execute()
{
	//bulk
	NFActorMessage messageObject;
    //将消息队列的消息全部弹出
	while (mMessageQueue.TryPop(messageObject))
	{
		//must make sure that only one thread running this function at the same time
		//mxProcessFunctor is not thread-safe
		NFLock lock;

		lock.lock();
        //根据消息ID 获取相应的处理函数
		ACTOR_PROCESS_FUNCTOR_PTR xBeginFunctor = mxProcessFunctor.GetElement(messageObject.msgID);
		lock.unlock();

		if (xBeginFunctor != nullptr)
		{
			//std::cout << ID().ToString() << " received message " << messageObject.msgID << " and msg index is " << messageObject.index << " totaly msg count: " << mMessageQueue.size_approx() << std::endl;

			xBeginFunctor->operator()(messageObject);

			//return the result to the main thread
			m_pActorModule->AddResult(messageObject);
		}
	}

	return true;
}

bool NFActor::AddComponent(NF_SHARE_PTR<NFIComponent> pComponent)
{
	//if you want to add more components for the actor, please don't clear the component
	//mComponent.ClearAll();
	if (!mComponent.ExistElement(pComponent->GetComponentName()))
	{
        // Actor 里面有 Components 组成的map 
		mComponent.AddElement(pComponent->GetComponentName(), pComponent);
        // Actor 和 每个component 相互映射
		pComponent->SetActor(NF_SHARE_PTR<NFIActor>(this));

		pComponent->Awake();
		pComponent->Init();
		pComponent->AfterInit();
		pComponent->ReadyExecute();

		return true;
	}

	return false;
}

bool NFActor::RemoveComponent(const std::string& strComponentName)
{
	return false;
}

NF_SHARE_PTR<NFIComponent> NFActor::FindComponent(const std::string & strComponentName)
{
	return mComponent.GetElement(strComponentName);
}

bool NFActor::AddMessageHandler(const int nSubMsgID, ACTOR_PROCESS_FUNCTOR_PTR xBeginFunctor)
{
	return mxProcessFunctor.AddElement(nSubMsgID, xBeginFunctor);
}

bool NFActor::SendMsg(const NFActorMessage& message)
{
	return mMessageQueue.Push(message);
}

bool NFActor::SendMsg(const int eventID, const std::string& data, const std::string& arg)
{
	static NFActorMessage xMessage;

	xMessage.id = this->id;
	xMessage.msgID = eventID;
	xMessage.data = data;
	xMessage.arg = arg;

	return SendMsg(xMessage);
}

bool NFActor::BackMsgToMainThread(const NFActorMessage& message)
{
	return m_pActorModule->AddResult(message);
}

void NFActor::ToMemoryCounterString(std::string& info)
{
	info.append(id.ToString());
	info.append(":NFActor:");
	info.append(std::to_string(mMessageQueue.size_approx()));
}
```