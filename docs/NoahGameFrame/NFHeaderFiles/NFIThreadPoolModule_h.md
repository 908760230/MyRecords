# NFIThreadPoolModule
继承于 NFImodule

```c++

#ifndef NFI_THREAD_POOL_MODULE_H
#define NFI_THREAD_POOL_MODULE_H

#include "NFIModule.h"
#include <thread>

///////////////////////////////////////////////////
class NFThreadTask;

typedef std::function<void(NFThreadTask&)> TASK_PROCESS_FUNCTOR;
typedef std::shared_ptr<TASK_PROCESS_FUNCTOR> TASK_PROCESS_FUNCTOR_PTR;

//定义了数据内容和回调函数
class NFThreadTask
{
public:
	NFGUID nTaskID;
	std::string data;
	TASK_PROCESS_FUNCTOR_PTR xThreadFunc; //线程内执行的回调函数
	TASK_PROCESS_FUNCTOR_PTR xEndFunc;    //根据这个来判断是否将结果传回到主线程
};


class NFIThreadPoolModule : public NFIModule
{
public:
    //根据cpu的核数来设置线程数量
	virtual void SetCpu(const int cpuCount) = 0;

	template<typename BaseType>
	void DoAsyncTask(const NFGUID taskID, const std::string& data, BaseType* pBase, void (BaseType::*handler_begin)(NFThreadTask&&))
	{
        TASK_PROCESS_FUNCTOR functor_begin = std::bind(handler_begin, pBase, std::placeholders::_1);
		TASK_PROCESS_FUNCTOR_PTR functorPtr_begin(new TASK_PROCESS_FUNCTOR(functor_begin));

		DoAsyncTask(taskID, data, functorPtr_begin, nullptr);
	}

	template<typename BaseType>
	void DoAsyncTask(const NFGUID taskID, const std::string& data, BaseType* pBase, void (BaseType::*handler_begin)(NFThreadTask&&), void (BaseType::*handler_end)(NFThreadTask&&))
	{
        TASK_PROCESS_FUNCTOR functor_begin = std::bind(handler_begin, pBase, std::placeholders::_1);
		TASK_PROCESS_FUNCTOR_PTR functorPtr_begin(new TASK_PROCESS_FUNCTOR(functor_begin));

		TASK_PROCESS_FUNCTOR functor_end = std::bind(handler_end, pBase, std::placeholders::_1);
		TASK_PROCESS_FUNCTOR_PTR functorPtr_end(new TASK_PROCESS_FUNCTOR(functor_end));

		DoAsyncTask(taskID, data, functorPtr_begin, functorPtr_end);
	}

    void DoAsyncTask(const NFGUID taskID, const std::string& data, TASK_PROCESS_FUNCTOR asyncFunctor)
	{
		TASK_PROCESS_FUNCTOR_PTR functorPtr_begin(new TASK_PROCESS_FUNCTOR(asyncFunctor));

		DoAsyncTask(taskID, data, functorPtr_begin, nullptr);
	}

    void DoAsyncTask(const NFGUID taskID, const std::string& data, TASK_PROCESS_FUNCTOR asyncFunctor, TASK_PROCESS_FUNCTOR functor_end)
	{
		TASK_PROCESS_FUNCTOR_PTR functorPtr_begin(new TASK_PROCESS_FUNCTOR(asyncFunctor));
		TASK_PROCESS_FUNCTOR_PTR functorPtr_end(new TASK_PROCESS_FUNCTOR(functor_end));

		DoAsyncTask(taskID, data, functorPtr_begin, functorPtr_end);
	}

	virtual void DoAsyncTask(const NFGUID taskID, const std::string& data, TASK_PROCESS_FUNCTOR_PTR asyncFunctor, TASK_PROCESS_FUNCTOR_PTR functor_end) = 0;

	/////repush the result
	virtual void TaskResult(const NFThreadTask& task) = 0;
};

#endif

```