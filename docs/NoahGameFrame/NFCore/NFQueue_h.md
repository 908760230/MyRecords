# NFQueue

```c++

#ifndef NF_QUEUE_H
#define NF_QUEUE_H

#include <list>
#include <thread>
#include <mutex>
#include <atomic>
#include "NFComm/NFPluginModule/NFPlatform.h"
#include "Dependencies/concurrentqueue/concurrentqueue.h"

//原子锁
class NFLock
{
public:
    explicit NFLock()
    {
        flag.clear();
    }

    ~NFLock()
    {
    }

    void lock()
    {
        while (flag.test_and_set(std::memory_order_acquire))
            ;
    }

    bool try_lock()
    {
        if (flag.test_and_set(std::memory_order_acquire))
        {
            return false;
        }

        return true;
    }

    void unlock()
    {
        flag.clear(std::memory_order_release);
    }

protected:
    mutable std::atomic_flag flag = ATOMIC_FLAG_INIT;

private:
    NFLock& operator=(const NFLock& src);
};

//read prior or write prior?
//it's different for these two situations

//继承第三方库写的无锁队列
template<typename T>
//class NFQueue : public NFLock
class NFQueue : public moodycamel::ConcurrentQueue<T>
{
public:
    NFQueue()
    {

    }

    virtual ~NFQueue()
    {
    }

    bool Push(const T& object)
    {
		this->enqueue(object);

        return true;
    }

	bool PushBulk(const T& object)
	{
		this->enqueue(object);

		return true;
	}

    bool TryPop(T& object)
    {
		return this->try_dequeue(object);
    }
	
	bool TryPopBulk(T& object)
	{
		return this->try_dequeue(object);
	}
};

#endif
```