# NFThreadPoolModule

## 头文件
```c++
#ifndef NF_THREAD_POOL_MODULE_H
#define NF_THREAD_POOL_MODULE_H

#include <map>
#include <string>
#include "NFComm/NFPluginModule/NFPlatform.h"
#include "NFComm/NFPluginModule/NFIThreadPoolModule.h"
#include "NFComm/NFCore/NFQueue.hpp"

//对thread 进行封装 在开始被创建时就执行Execute函数，每一个threadCell里面有一个task队列
class NFThreadCell
{
public:
	NFThreadCell(NFIThreadPoolModule* p)
	{
		m_pThreadPoolModule = p;
		mThread = NF_SHARE_PTR<std::thread>(NF_NEW std::thread(&NFThreadCell::Execute, this));
	}
    //将task 压入队列中
	void AddTask(const NFThreadTask& task)
	{
		mTaskList.Push(task);
	}

protected:

	void Execute()
	{
		while (true)
		{
			std::this_thread::sleep_for(std::chrono::milliseconds(1));
			
            //不断的pop出 task然后执行
			//pick the first task and do it
			NFThreadTask task;
			if (mTaskList.TryPop(task))
			{
				task.xThreadFunc->operator()(task);

				//repush the result to the main thread
				//and, do we must to tell the result?
				if (task.xEndFunc)
				{   //添加到ThreadPool 的rask队列里
					m_pThreadPoolModule->TaskResult(task);
				}
			}
		}
	}

private:
	NFQueue<NFThreadTask> mTaskList;    
	NF_SHARE_PTR<std::thread> mThread;
	NFIThreadPoolModule* m_pThreadPoolModule;
};

class NFThreadPoolModule
    : public NFIThreadPoolModule
{
public:
	NFThreadPoolModule(NFIPluginManager* p);
    virtual ~NFThreadPoolModule();

	virtual void SetCpu(const int cpuCount);

    virtual bool Init();

    virtual bool AfterInit();

    virtual bool BeforeShut();

    virtual bool Shut();

    virtual bool Execute();

	virtual void DoAsyncTask(const NFGUID taskID, const std::string& data, TASK_PROCESS_FUNCTOR_PTR asyncFunctor, TASK_PROCESS_FUNCTOR_PTR functor_end);

	virtual void TaskResult(const NFThreadTask& task);

protected:
	void ExecuteTaskResult();

private:
	int mCPUCount = 1;

	NFQueue<NFThreadTask> mTaskResult;
	std::vector<NF_SHARE_PTR<NFThreadCell>> mThreadPool;
};

#endif
```

## cpp代码
```c++

#include "NFThreadPoolModule.h"

NFThreadPoolModule::NFThreadPoolModule(NFIPluginManager* p)
{
	pPluginManager = p;
}

NFThreadPoolModule::~NFThreadPoolModule()
{
}

void NFThreadPoolModule::SetCpu(const int cpuCount)
{
    if (cpuCount > mCPUCount)  //只有大于1 才会往pool里面添加 thread
    {
        mCPUCount = cpuCount;
        for (int i = mThreadPool.size(); i < mCPUCount * 2; ++i)
        {
            mThreadPool.push_back(NF_SHARE_PTR<NFThreadCell>(NF_NEW NFThreadCell(this)));
        }
    }
}

bool NFThreadPoolModule::Init()
{
    if (mCPUCount > 0)
    {
        for (int i = 0; i < mCPUCount * 2; ++i)
        {
            mThreadPool.push_back(NF_SHARE_PTR<NFThreadCell>(NF_NEW NFThreadCell(this)));
        }
    }

    return true;
}

bool NFThreadPoolModule::AfterInit()
{

    return true;
}

bool NFThreadPoolModule::BeforeShut()
{
    return true;
}

bool NFThreadPoolModule::Shut()
{
 
    return true;
}

bool NFThreadPoolModule::Execute()
{
	ExecuteTaskResult();

    return true;
}

void NFThreadPoolModule::DoAsyncTask(const NFGUID taskID, const std::string & data, TASK_PROCESS_FUNCTOR_PTR asyncFunctor, TASK_PROCESS_FUNCTOR_PTR functor_end)
{   //新建一个task，保存相关参数
	NFThreadTask task;
	task.nTaskID = taskID;
	task.data = data;
	task.xThreadFunc = asyncFunctor;
	task.xEndFunc = functor_end;
	
	int index = 0;
	if (!taskID.IsNull())
	{
		index = taskID.nData64 % mThreadPool.size();
	}
	//默认为为 0号线程
	NF_SHARE_PTR<NFThreadCell> threadobject = mThreadPool[index];
    //将task 添加到threadCell里的 task队列里
	threadobject->AddTask(task);
}

void NFThreadPoolModule::ExecuteTaskResult()
{
	NFThreadTask xMsg;
	while (mTaskResult.TryPop(xMsg))
	{
		if (xMsg.xEndFunc)
		{
			xMsg.xEndFunc->operator()(xMsg);
		}
	}
}

void NFThreadPoolModule::TaskResult(const NFThreadTask& task)
{
	mTaskResult.Push(task);
}
```