# NFIProperty

## 原作者的解释
NF当初设计的目的，是想拥有一种方法，可以统一管理对象以及数据，并使用统一的接口而又不需要随着业务增加而扩充新的接口和数据字段，这些体验源自ogre对于node的抽象，因后来参考了Bigworld的数据管理方式，设计了抽象的NFIClass，NFIObject 和 NFIElementModule，它们是NF中最重要的3个类，是支撑NF新概念面向数据编程的重要基石。

NFIClass类，顾名思义，就是NF Class。
主要用户声明数据结构体，类似程序语言（C , C# JAVA等）编程中使用class Dog, class Cat（然而实际又不一样，虽然实际上他代替了变成语言的class，struct等部分功能）。传统情况下，大家对于对象增加属性，一直都是类似结构体加入各种字段来实现，比如一个Dog先有HP，则是如下：

```c++
class Dog
{
    int nHP;
};
```


如果想要加入MP；则需要修改成如下：

```c++
class Dog
{
    int nHP;
    int nMP;
};

```

然后又还需要增加名字，配置ID，等级，则需要修改如下

```c++
class Dog
{
    int nHP;
    int nMP;
    string strName;
    string strID;
    int nLevel;
};
```

and so on................

然后随着业务的增加，还需要增加死亡是否能复活，头像，客户端模型包等。。。。那么我们又要继续修改，如果哪天因为Dog, Cat太像了，然后还需要抽象共同的属性结构体，重构接口。。。陷入无休止的基础系统维护当中！然后项目没完没了，然后项目未上线便开始处于维护状态，然后人心涣散项目跪掉。虽然话语夸张，但也不是没有公司这样死过，就看谁死的轰轰烈烈而已。
而在NF架构中，则完全没有这些烦恼，一切均统一化配置管理，不需增加额外代码。

下面就从NF的设计源头开始说说，如何改善这种局面。
正常来说，任何抽象对象，我们主要关注他俩样，属性和行为。这里先来说属性。通常来说，属性都可以抽象成通用的数据对象（想看行为如何抽象成通用的接口层，请继续看后面几章）。

我们分析，HP, MP, Name这几个其实都是成员属性，如果有一个通用的属性来容纳他们，这个属性叫 Property，则Dog可以抽象成如下：
```c++
class Dog
{
        Property nHP;
        Property nMP;
        Property strName;
        Property strID;
        Property nLevel;
};
```


此时，我们的精力就在于如何抽象一个通用的属性Property.我们知道，Property是肯定可以抽象出来的，因为在目前已知道的数据类型中，描述一个对象的基础属性, 也就int, double, string, vector2, vector3, GUID 等几个类型，而string又可以容纳一切数据类型，因此针对这些类型的数据，我们可以抽象出统一的属性接口，Property则长成如下样貌：


于是Property应该长这样:
```c++
class Property
{
    public bool SetInt(const NFINT64 value);
    public bool SetFloat(const double value);
    public bool SetString(const std::string& value);
    public bool SetObject(const NFGUID& value);
    public bool SetVector2(const NFVector2& value);
    public bool SetVector3(const NFVector3& value);

    public NFINT64 GetInt() const;
    public double GetFloat() const;
    public const std::string& GetString() const;
    public const NFGUID& GetObject() const;
    public const NFVector2& GetVector2() const;
    public const NFVector3& GetVector3() const;
}
```

到此，我们可以很方便的避免了数据操作的细节，全部抽象成合适的接口.
虽然都抽象成统一的属性了，如果每一次新业务，都要增加这些，岂不是很麻烦？ 因此当时想到了，增加一个管理器吧，其实这个很简单，重要的是各种简单的内容，组合起来就能创造出一些有趣的新内容，于是Dog类变成了如下:
```c++
class Dog
{
    map<string, Property> mPropertyMap;
}
```

然后我们又想根据名字，从容器中获取属性，然后修改属性，或者添加新的属性，这样每次新业务，我们就不会再添加新的字段，很方便，不是吗？ 但是这个远达不到我们想要的解决方案那样方便，因为有人担心，那么name哪来统一管理，各种设置接口如何设计，然后如果有多种类似的结构如何处理？不要担心，且继续看，我们为了避免后续增加新的模块，我们先把属性管理器拆分出来，则为如下:

```c++
class PropertyManager
{
    public bool SetInt(const std::string& strPropertyName, const NFINT64 value);
    public bool SetFloat(const std::string& strPropertyName, const double value);
    public bool SetString(const std::string& strPropertyName, const std::string& value);
    public bool SetObject(const std::string& strPropertyName, const NFGUID& value);
    public bool SetVector2(const std::string& strPropertyName, const NFVector2& value);
    public bool SetVector3(const std::string& strPropertyName, const NFVector3& value);

    public NFINT64 GetInt(const std::string& strPropertyName) const;
    public double GetFloat(const std::string& strPropertyName) const;
    public const std::string& GetString(const std::string& strPropertyName) const;
    public const NFGUID& GetObject(const std::string& strPropertyName) const;
    public const NFVector2& GetVector2(const std::string& strPropertyName) const;
    public const NFVector3& GetVector3(const std::string& strPropertyName) const;
}

class Dog
{
    PropertyManager mxPropertyManager;
};
```

到现在，我们增加通用的管理机器，方便各种添加，删除，和操作接口，正是这些基础接口，可以节省开发中50%以上的时间，而关键在于，稳定，不出错，也省却了大部分调试时间。

那么对于现在这个NFClass类来说，他可以叫NPC，可以叫Dog，可以叫Cat，都没有任何关系，因为他已经支持动态的各种属性添加等事项.那么抽象的数据容器有了，接下来，我们如何方便的，可以在excel，或者xml中，把数据能自动的导入呢，虽然中途需要各种代码要码，但是终究是可以实现的。

首先，我们去掉了在程序语言中增加各种字段，那么当然避免不了的，我们的结构描述文本，肯定会有类似的机制来保证可以添加新字段，暂且用XML来表达吧，如下：

```xml
<XML>
     <Propertys>
          <Property Id="HP" Type="int"  />
          <Property Id="MP" Type="string" />
          <Property Id="Name" Type="int" />
          <Property Id="ID" Type="string" />
          <Property Id="Level" Type="int" />
     </Propertys>
</XML>
```
还记得之前我们说的 PropertyManager类中的map<string, Property> mPropertyMap 的Key哪来吗？就是上面的Id字段，HP,MP,NAME等。


## 完整代码
```c++

#ifndef NFI_PROPERTY_H
#define NFI_PROPERTY_H

#include "NFDataList.hpp"
#include "NFList.hpp"
#include "NFComm/NFPluginModule/NFPlatform.h"

typedef std::function<int(const NFGUID&, const std::string&, const NFData&, const NFData&)> PROPERTY_EVENT_FUNCTOR;
typedef NF_SHARE_PTR<PROPERTY_EVENT_FUNCTOR> PROPERTY_EVENT_FUNCTOR_PTR;

class _NFExport NFIProperty : public NFMemoryCounter
{
public:
	NFIProperty() : NFMemoryCounter(GET_CLASS_NAME(NFIProperty), 1)
	{
	}

	virtual ~NFIProperty() {}

	virtual void SetValue(const NFData& TData) = 0;
	virtual void SetValue(const NFIProperty* pProperty) = 0;

	virtual bool SetInt(const NFINT64 value) = 0;
	virtual bool SetFloat(const double value) = 0;
	virtual bool SetString(const std::string& value) = 0;
	virtual bool SetObject(const NFGUID& value) = 0;
	virtual bool SetVector2(const NFVector2& value) = 0;
	virtual bool SetVector3(const NFVector3& value) = 0;

	virtual const NFDATA_TYPE GetType() const = 0;
	virtual const bool GeUsed() const = 0;
	virtual const std::string& GetKey() const = 0;
	virtual const bool GetSave() const = 0;
	virtual const bool GetPublic() const = 0;
	virtual const bool GetPrivate() const = 0;
	virtual const bool GetCache() const = 0;
	virtual const bool GetRef() const = 0;
	virtual const bool GetForce() const = 0;
	virtual const bool GetUpload() const = 0;

	virtual void SetSave(bool bSave) = 0;
	virtual void SetPublic(bool bPublic) = 0;
	virtual void SetPrivate(bool bPrivate) = 0;
	virtual void SetCache(bool bCache) = 0;
	virtual void SetRef(bool bRef) = 0;
	virtual void SetForce(bool bRef) = 0;
	virtual void SetUpload(bool bUpload) = 0;

	virtual NFINT64 GetInt() const = 0;
	virtual int GetInt32() const = 0;
	virtual double GetFloat() const = 0;
	virtual const std::string& GetString() const = 0;
	virtual const NFGUID& GetObject() const = 0;
	virtual const NFVector2& GetVector2() const = 0;
	virtual const NFVector3& GetVector3() const = 0;

	virtual const NFData& GetValue() const = 0;
	virtual const NF_SHARE_PTR<NFList<std::string>> GetEmbeddedList() const = 0;
	virtual const NF_SHARE_PTR<NFMapEx<std::string, std::string>> GetEmbeddedMap() const = 0;

	virtual bool Changed() const = 0;

	virtual std::string ToString() = 0;
	virtual bool FromString(const std::string& strData) = 0;
	virtual bool DeSerialization() = 0;

	virtual void RegisterCallback(const PROPERTY_EVENT_FUNCTOR_PTR& cb) = 0;
};

#endif

```