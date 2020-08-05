# NFMemoryCounter 解析

这个类的作用如同它的名字一样，是一个内存计数器。禁止了默认构造函数，一个私有的内部类 Data, Data内使用map 保存了 对象 和deep的映射关系。

## 内部类Data

``` c++

struct Data
{
    Data(NFMemoryCounter* p, int d)
        :deep(d)
    {
        data.insert(std::map<NFMemoryCounter*, int>::value_type(p, d));
    }

    std::map<NFMemoryCounter*, int> data;
    int deep = 0;
};


```

deep 的作用暂时未知， 可能是内存的优先级？

## 完整代码

``` c++

#ifndef NF_MEMORY_COUNTER_H
#define NF_MEMORY_COUNTER_H

#include <iostream>
#include <string>
#include <map>
#include "NFComm/NFPluginModule/NFPlatform.h"

class NFMemoryCounter
{
private:
	NFMemoryCounter() {}

    std::string mstrClassName;

    struct Data
    {
        Data(NFMemoryCounter* p, int d)
            :deep(d)
        {
            data.insert(std::map<NFMemoryCounter*, int>::value_type(p, d));
        }

        std::map<NFMemoryCounter*, int> data;
        int deep = 0;
    };


public:
	static std::map<std::string, Data>* mxCounter;

	NFMemoryCounter(const std::string& strClassName, const int deep = 0)
	{
		mstrClassName = strClassName;

        if (!mxCounter)
        {
            mxCounter = NF_NEW std::map<std::string, Data>();
        }
		
        auto it = mxCounter->find(mstrClassName);
        if (it == mxCounter->end())
        {
            mxCounter->insert(std::map<std::string, Data>::value_type(mstrClassName, Data(this, deep)));
        }
        else
        {
            it->second.data.insert(std::map<NFMemoryCounter*, int>::value_type(this, deep));
        }
	}

	virtual ~NFMemoryCounter()
	{
        auto it = mxCounter->find(mstrClassName);
        if (it != mxCounter->end())
        {
            auto it2 = it->second.data.find(this);
            if (it2 != it->second.data.end())
            {
                it->second.data.erase(it2);
            }
        }
	}

    virtual void ToMemoryCounterString(std::string& info) = 0;

    static void PrintMemoryInfo(std::string& info, const int deep = 0)
    {
        for (auto it = mxCounter->begin(); it != mxCounter->end(); ++it)
        {
            info.append(it->first);
            info.append("=>");
            info.append(std::to_string(it->second.data.size()));
            info.append("\n");

            if (deep && it->second.deep)
            {
                for  (auto data : it->second.data)
                {
                    data.first->ToMemoryCounterString(info);
                }
            }
        }
    }
};

#endif


``` 
