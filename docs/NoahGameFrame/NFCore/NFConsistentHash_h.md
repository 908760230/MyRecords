# NFConsistentHash


一致性哈希主要就是解决当机器减少或增加的时候，大面积的数据重新哈希的问题。

## 缺点：

一致性Hash算法在服务节点太少时，容易因为节点分部不均匀而造成数据倾斜（被缓存的对象大部分集中缓存在某一台服务器上）问题。比如只有 2 台机器，这 2 台机器离的很近，那么顺时针第一个机器节点上将存在大量的数据，第二个机器节点上数据会很少。

为了避免出现数据倾斜问题，一致性 Hash 算法引入了虚拟节点的机制，也就是每个机器节点会进行多次哈希，最终每个机器节点在哈希环上会有多个虚拟节点存在，使用这种方式来大大削弱甚至避免数据倾斜问题。同时数据定位算法不变，只是多了一步虚拟节点到实际节点的映射。


## Redis 集群分槽的实现


Redis 集群并没有直接使用一致性哈希，而是使用了哈希槽 （slot） 的概念，Redis 没有直接使用哈希算法 hash()，而是使用了crc16校验算法。槽位其实就是一个个的空间的单位。其实哈希槽的本质和一致性哈希算法非常相似，不同点就是对于哈希空间的定义。一致性哈希的空间是一个圆环，节点分布是基于圆环的，无法很好的控制数据分布，可能会产生数据倾斜问题。而 Redis 的槽位空间是自定义分配的，类似于Windows盘分区的概念。这种分区是可以自定义大小，自定义位置的。Redis 集群包含了 16384 个哈希槽，每个 Key 经过计算后会落在一个具体的槽位上，而槽位具体在哪个机器上是用户自己根据自己机器的情况配置的，机器硬盘小的可以分配少一点槽位，硬盘大的可以分配多一点。如果节点硬盘都差不多则可以平均分配。所以哈希槽这种概念很好地解决了一致性哈希的弊端。

另外在容错性和扩展性上与一致性哈希一样，都是对受影响的数据进行转移而不影响其它的数据。而哈希槽本质上是对槽位的转移，把故障节点负责的槽位转移到其他正常的节点上。扩展节点也是一样，把其他节点上的槽位转移到新的节点上。

需要注意的是，对于槽位的转移和分派，Redis集群是不会自动进行的，而是需要人工配置的。所以Redis集群的高可用是依赖于节点的主从复制与主从间的自动故障转移。



## NFVirtualNode
定义了NFIVirtualNode的接口和索引。
```c++
class NFIVirtualNode 
{
public:

    NFIVirtualNode(const int nVirID)
        :nVirtualIndex(nVirID)
    {

    }

	NFIVirtualNode()
	{
		nVirtualIndex = 0;
	}

	virtual ~NFIVirtualNode()
	{
		nVirtualIndex = 0;
	}

	virtual std::string GetDataStr() const
	{
		return "";
	}

    std::string ToStr() const 
    {
        std::ostringstream strInfo;
        strInfo << GetDataStr() << "-" << nVirtualIndex;
        return strInfo.str();
    }

private:
    int nVirtualIndex;
};



template <typename T>
class NFVirtualNode : public NFIVirtualNode
{
public:
	NFVirtualNode(const T tData, const int nVirID ) : NFIVirtualNode(nVirID)
	{
		mxData = tData;
	}
	NFVirtualNode()
	{
	}

	virtual std::string GetDataStr() const
	{
		return lexical_cast<std::string>(mxData);
	}

	T mxData;
};

```


## NFHasher

将字符串CRC32校验码当做哈希值

```c++

class NFIHasher
{
public:
	virtual ~NFIHasher(){}
    virtual uint32_t GetHashValue(const NFIVirtualNode& vNode) = 0;
};

class NFHasher : public NFIHasher
{
public:
    virtual uint32_t GetHashValue(const NFIVirtualNode& vNode)
    {
        std::string vnode = vNode.ToStr();
        return NFrame::CRC32(vnode);
    }
};

```

## NFConsistentHash

```c++
template <typename T>
class NFIConsistentHash
{
public:
	virtual std::size_t Size() const = 0;
	virtual bool Empty() const = 0;

	virtual void ClearAll() = 0;
	virtual void Insert(const T& name) = 0;
	virtual void Insert(const NFVirtualNode<T>& xNode) = 0;

	virtual bool Exist(const NFVirtualNode<T>& xInNode) = 0;
	virtual void Erase(const T& name) = 0;
	virtual std::size_t Erase(const NFVirtualNode<T>& xNode)  = 0;

	virtual bool GetSuitNodeRandom(NFVirtualNode<T>& node) = 0;
	virtual bool GetSuitNodeConsistent(NFVirtualNode<T>& node) = 0;
	virtual bool GetSuitNode(const T& name, NFVirtualNode<T>& node) = 0;
	//virtual bool GetSuitNode(const std::string& str, NFVirtualNode<T>& node) = 0;
	virtual bool GetSuitNode(uint32_t hashValue, NFVirtualNode<T>& node) = 0;

	virtual bool GetNodeList(std::list<NFVirtualNode<T>>& nodeList) = 0;
};

template <typename T>
class NFConsistentHash : public NFIConsistentHash<T>
{
public:
    NFConsistentHash()
    {
        m_pHasher = new NFHasher();
    }

    virtual ~NFConsistentHash()
    {
        delete m_pHasher;
        m_pHasher = NULL;
    }

public:
	virtual std::size_t Size() const
    {
        return mxNodes.size();
    }

	virtual bool Empty() const
    {
        return mxNodes.empty();
    }

	virtual void ClearAll()
	{
		mxNodes.clear();
	}

	virtual void Insert(const T& name)
	{
        // 一次性插入五百个 没看懂是为什么
		for (int i = 0; i < mnNodeCount; ++i)
		{
			NFVirtualNode<T> vNode(name, i);
			Insert(vNode);
		}
	}

	virtual void Insert(const NFVirtualNode<T>& xNode)
    {
        uint32_t hash = m_pHasher->GetHashValue(xNode);
        auto it = mxNodes.find(hash);
        if (it == mxNodes.end())
        {
            mxNodes.insert(typename std::map<uint32_t, NFVirtualNode<T>>::value_type(hash, xNode));
        }
    }

	virtual bool Exist(const NFVirtualNode<T>& xInNode)
	{
		uint32_t hash = m_pHasher->GetHashValue(xInNode);
		typename std::map<uint32_t, NFVirtualNode<T>>::iterator it = mxNodes.find(hash);
		if (it != mxNodes.end())
		{
			return true;
		}

		return false;
	}

	virtual void Erase(const T& name)
	{
		for (int i = 0; i < mnNodeCount; ++i)
		{
			NFVirtualNode<T> vNode(name, i);
			Erase(vNode);
		}
	}

	virtual std::size_t Erase(const NFVirtualNode<T>& xNode)
    {
        uint32_t hash = m_pHasher->GetHashValue(xNode);
        return mxNodes.erase(hash);
    }

	virtual bool GetSuitNodeRandom(NFVirtualNode<T>& node)
	{
		int nID = (int) std::chrono::duration_cast<std::chrono::milliseconds>(std::chrono::system_clock::now().time_since_epoch()).count();
		return GetSuitNode(nID, node);
	}

	virtual bool GetSuitNodeConsistent(NFVirtualNode<T>& node)
	{
		return GetSuitNode(0, node);
	}

	virtual bool GetSuitNode(const T& name, NFVirtualNode<T>& node)
	{
		std::string str = lexical_cast<std::string>(name);
		uint32_t nCRC32 = NFrame::CRC32(str);
		return GetSuitNode(nCRC32, node);
	}
	virtual bool GetSuitNode(uint32_t hashValue, NFVirtualNode<T>& node)
	{
		if(mxNodes.empty())
		{
			return false;
		}

		typename std::map<uint32_t, NFVirtualNode<T>>::iterator it = mxNodes.lower_bound(hashValue);

		if (it == mxNodes.end())
		{
			it = mxNodes.begin();
		}

		node = it->second;

		return true;
	}

	virtual bool GetNodeList(std::list<NFVirtualNode<T>>& nodeList)
	{
		for (typename std::map<uint32_t, NFVirtualNode<T>>::iterator it = mxNodes.begin(); it != mxNodes.end(); ++it)
		{
			nodeList.push_back(it->second);
		}

		return true;
	}

private:
	int mnNodeCount = 500;
	typename std::map<uint32_t, NFVirtualNode<T>> mxNodes;
    NFIHasher* m_pHasher;
};
```