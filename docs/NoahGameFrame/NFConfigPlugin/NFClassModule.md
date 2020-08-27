# NFClassModule

## 头文件
```c++

#ifndef NF_LOGICCLASS_MODULE_H
#define NF_LOGICCLASS_MODULE_H

#include <string>
#include <map>
#include <iostream>
#include "NFElementModule.h"
#include "Dependencies/RapidXML/rapidxml.hpp"
#include "NFComm/NFCore/NFMap.hpp"
#include "NFComm/NFCore/NFList.hpp"
#include "NFComm/NFCore/NFDataList.hpp"
#include "NFComm/NFCore/NFRecord.h"
#include "NFComm/NFCore/NFList.hpp"
#include "NFComm/NFCore/NFPropertyManager.h"
#include "NFComm/NFCore/NFRecordManager.h"
#include "NFComm/NFPluginModule/NFIClassModule.h"
#include "NFComm/NFPluginModule/NFIElementModule.h"
#include "NFComm/NFPluginModule/NFIPluginManager.h"

class NFClass : public NFIClass
{
public:

    NFClass(const std::string& strClassName)
    {
        m_pParentClass = NULL;
        mstrClassName = strClassName;

        m_pPropertyManager = NF_SHARE_PTR<NFIPropertyManager>(NF_NEW NFPropertyManager(NFGUID()));
        m_pRecordManager = NF_SHARE_PTR<NFIRecordManager>(NF_NEW NFRecordManager(NFGUID()));
    }

    virtual ~NFClass()
    {
        ClearAll();
    }

    virtual NF_SHARE_PTR<NFIPropertyManager> GetPropertyManager()
    {
        return m_pPropertyManager;
    }

    virtual NF_SHARE_PTR<NFIRecordManager> GetRecordManager()
    {
        return m_pRecordManager;
    }

    virtual bool AddClassCallBack(const CLASS_EVENT_FUNCTOR_PTR& cb)
    {
        return mxClassEventInfo.Add(cb);
    }

    virtual bool DoEvent(const NFGUID& objectID, const CLASS_OBJECT_EVENT eClassEvent, const NFDataList& valueList)
    {
        CLASS_EVENT_FUNCTOR_PTR cb;
        bool bRet = mxClassEventInfo.First(cb);
        while (bRet)
        {
            cb->operator()(objectID, mstrClassName, eClassEvent,  valueList);

            bRet = mxClassEventInfo.Next(cb);
        }

        return true;
    }

    void SetParent(NF_SHARE_PTR<NFIClass> pClass)
    {
        m_pParentClass = pClass;
    }

    NF_SHARE_PTR<NFIClass> GetParent()
    {
        return m_pParentClass;
    }

    void SetTypeName(const char* strType)
    {
        mstrType = strType;
    }

    const std::string& GetTypeName()
    {
        return mstrType;
    }

    const std::string& GetClassName()
    {
        return mstrClassName;
    }

    const bool AddId(std::string& strId)
    {
		mIdList.push_back(strId);
        return true;
    }

	const std::vector<std::string>& GetIDList()
    {
        return mIdList;
    }

    void SetInstancePath(const std::string& strPath)
    {
        mstrClassInstancePath = strPath;
    }

    const std::string& GetInstancePath()
    {
        return mstrClassInstancePath;
    }

private:
    NF_SHARE_PTR<NFIPropertyManager> m_pPropertyManager;
    NF_SHARE_PTR<NFIRecordManager> m_pRecordManager;

    NF_SHARE_PTR<NFIClass> m_pParentClass;
    std::string mstrType;
    std::string mstrClassName;
    std::string mstrClassInstancePath;

    std::vector<std::string> mIdList;

    NFList<CLASS_EVENT_FUNCTOR_PTR> mxClassEventInfo;
};

class NFClassModule
    : public NFIClassModule
{
private:
    NFClassModule();
public:
    NFClassModule(NFIPluginManager* p);
    virtual ~NFClassModule();
	
	
	virtual bool Awake();
    virtual bool Init();
    virtual bool Shut();

    virtual bool Load();
    virtual bool Save();
    virtual bool Clear();

    virtual NFIClassModule* GetBackupClassModule() override;

    virtual bool AddClassCallBack(const std::string& strClassName, const CLASS_EVENT_FUNCTOR_PTR& cb);
    virtual bool DoEvent(const NFGUID& objectID, const std::string& strClassName, const CLASS_OBJECT_EVENT eClassEvent, const NFDataList& valueList);

    virtual NF_SHARE_PTR<NFIPropertyManager> GetClassPropertyManager(const std::string& strClassName);
    virtual NF_SHARE_PTR<NFIRecordManager> GetClassRecordManager(const std::string& strClassName);

    virtual bool AddClass(const std::string& strClassName, const std::string& strParentName);

protected:
    virtual NFDATA_TYPE ComputerType(const char* pstrTypeName, NFData& var);
    virtual bool AddPropertys(rapidxml::xml_node<>* pPropertyRootNode, NF_SHARE_PTR<NFIClass> pClass);
    virtual bool AddRecords(rapidxml::xml_node<>* pRecordRootNode, NF_SHARE_PTR<NFIClass> pClass);
    virtual bool AddComponents(rapidxml::xml_node<>* pRecordRootNode, NF_SHARE_PTR<NFIClass> pClass);
    virtual bool AddClassInclude(const char* pstrClassFilePath, NF_SHARE_PTR<NFIClass> pClass);
    virtual bool AddClass(const char* pstrClassFilePath, NF_SHARE_PTR<NFIClass> pClass);

    
    virtual bool Load(rapidxml::xml_node<>* attrNode, NF_SHARE_PTR<NFIClass> pParentClass);

protected:
    NFIElementModule* m_pElementModule;
    NFClassModule* m_pBackupClassModule;

    std::string msConfigFileName;
    bool mbBackup = false;
};

#endif

```

## cpp文件
```c++

#include <time.h>
#include <algorithm>
#include "NFConfigPlugin.h"
#include "NFClassModule.h"
#include "Dependencies/RapidXML/rapidxml.hpp"
#include "Dependencies/RapidXML/rapidxml_print.hpp"

NFClassModule::NFClassModule()
{
    m_pBackupClassModule = nullptr;
    msConfigFileName = "NFDataCfg/Struct/LogicClass.xml";
}

NFClassModule::NFClassModule(NFIPluginManager* p)
{
    m_pBackupClassModule = nullptr;
    pPluginManager = p;
    msConfigFileName = "NFDataCfg/Struct/LogicClass.xml";

    std::cout << "Using [" << pPluginManager->GetConfigPath() + msConfigFileName << "]" << std::endl;

    if (!this->mbBackup)
    {
        m_pBackupClassModule = new NFClassModule();
        m_pBackupClassModule->pPluginManager = pPluginManager;
        m_pBackupClassModule->mbBackup = true;
    }
}

NFClassModule::~NFClassModule()
{
    ClearAll();

    if (!this->mbBackup)
    {
        delete m_pBackupClassModule;
        m_pBackupClassModule = nullptr;
    }
}

bool NFClassModule::Awake()
{
    m_pElementModule = pPluginManager->FindModule<NFIElementModule>();
    if (this->mbBackup)
    {
        m_pElementModule = m_pElementModule->GetBackupElementModule();
    }

    if (m_pBackupClassModule) m_pBackupClassModule->Awake();
    // 加载NFDataCfg/Struct/LogicClass.xml 里的内容
    Load();

	return true;
	
}

bool NFClassModule::Init()
{
    if (m_pBackupClassModule) m_pBackupClassModule->Init();
    return true;
}

bool NFClassModule::Shut()
{
    ClearAll();

    if (m_pBackupClassModule) m_pBackupClassModule->Shut();
    return true;
}

NFDATA_TYPE NFClassModule::ComputerType(const char* pstrTypeName, NFData& var)
{
    //根据名字设置相应类型，并设立初始值
    if (0 == strcmp(pstrTypeName, "int"))
    {
        var.SetInt(NULL_INT);
        return var.GetType();
    }
    else if (0 == strcmp(pstrTypeName, "string"))
    {
        var.SetString(NULL_STR);
        return var.GetType();
    }
    else if (0 == strcmp(pstrTypeName, "float"))
    {
        var.SetFloat(NULL_FLOAT);
        return var.GetType();
    }
    else if (0 == strcmp(pstrTypeName, "object"))
    {
        var.SetObject(NULL_OBJECT);
        return var.GetType();
    }
	else if (0 == strcmp(pstrTypeName, "vector2"))
	{
		var.SetVector2(NULL_VECTOR2);
		return var.GetType();
	}
	else if (0 == strcmp(pstrTypeName, "vector3"))
	{
		var.SetVector3(NULL_VECTOR3);
		return var.GetType();
	}

    return TDATA_UNKNOWN;
}

bool NFClassModule::AddPropertys(rapidxml::xml_node<>* pPropertyRootNode, NF_SHARE_PTR<NFIClass> pClass)
{
    for (rapidxml::xml_node<>* pPropertyNode = pPropertyRootNode->first_node(); pPropertyNode; pPropertyNode = pPropertyNode->next_sibling())
    {
        if (pPropertyNode)
        {
            const char* strPropertyName = pPropertyNode->first_attribute("Id")->value();
            //如果已保存Id属性就会报错
            if (pClass->GetPropertyManager()->GetElement(strPropertyName))
            {
                //error
                NFASSERT(0, strPropertyName, __FILE__, __FUNCTION__);
                continue;
            }

            const char* pstrType = pPropertyNode->first_attribute("Type")->value();
            const char* pstrPublic = pPropertyNode->first_attribute("Public")->value();
            const char* pstrPrivate = pPropertyNode->first_attribute("Private")->value();
            const char* pstrSave = pPropertyNode->first_attribute("Save")->value();
            const char* pstrCache = pPropertyNode->first_attribute("Cache")->value();
			const char* pstrRef = pPropertyNode->first_attribute("Ref")->value();
			const char* pstrForce = pPropertyNode->first_attribute("Force")->value();
			const char* pstrUpload = pPropertyNode->first_attribute("Upload")->value();
            // 将属性转化为bool
            bool bPublic = lexical_cast<bool>(pstrPublic);
            bool bPrivate = lexical_cast<bool>(pstrPrivate);
            bool bSave = lexical_cast<bool>(pstrSave);
            bool bCache = lexical_cast<bool>(pstrCache);
			bool bRef = lexical_cast<bool>(pstrRef);
			bool bForce = lexical_cast<bool>(pstrForce);
			bool bUpload = lexical_cast<bool>(pstrUpload);

            NFData varProperty;
            //得到相应的类型，并保存到 varProperty 中
            if (TDATA_UNKNOWN == ComputerType(pstrType, varProperty))
            {
                //std::cout << "error:" << pClass->GetTypeName() << "  " << pClass->GetInstancePath() << ": " << strPropertyName << " type error!!!" << std::endl;

                NFASSERT(0, strPropertyName, __FILE__, __FUNCTION__);
            }

            //printf( " Property:%s[%s]\n", pstrPropertyName, pstrType );
            //为pClass 添加新的 property
            NF_SHARE_PTR<NFIProperty> xProperty = pClass->GetPropertyManager()->AddProperty(NFGUID(), strPropertyName, varProperty.GetType());
            xProperty->SetPublic(bPublic);
            xProperty->SetPrivate(bPrivate);
            xProperty->SetSave(bSave);
            xProperty->SetCache(bCache);
			xProperty->SetRef(bRef);
			xProperty->SetForce(bForce);
			xProperty->SetUpload(bUpload);

        }
    }

    return true;
}
// Record属性 可以查看 NFDataCfg\Struct\Comm\EffectData.xml
bool NFClassModule::AddRecords(rapidxml::xml_node<>* pRecordRootNode, NF_SHARE_PTR<NFIClass> pClass)
{
    for (rapidxml::xml_node<>* pRecordNode = pRecordRootNode->first_node(); pRecordNode;  pRecordNode = pRecordNode->next_sibling())
    {
        if (pRecordNode)
        {
            const char* pstrRecordName = pRecordNode->first_attribute("Id")->value();

            if (pClass->GetRecordManager()->GetElement(pstrRecordName))
            {
                //error
                //file << pClass->mstrType << ":" << pstrRecordName << std::endl;
                //assert(0);
                NFASSERT(0, pstrRecordName, __FILE__, __FUNCTION__);
                continue;
            }

            const char* pstrRow = pRecordNode->first_attribute("Row")->value();
            const char* pstrCol = pRecordNode->first_attribute("Col")->value();

            const char* pstrPublic = pRecordNode->first_attribute("Public")->value();
            const char* pstrPrivate = pRecordNode->first_attribute("Private")->value();
            const char* pstrSave = pRecordNode->first_attribute("Save")->value();
			const char* pstrCache = pRecordNode->first_attribute("Cache")->value();
			const char* pstrRef = pRecordNode->first_attribute("Ref")->value();
			const char* pstrForce = pRecordNode->first_attribute("Force")->value();
			const char* pstrUpload = pRecordNode->first_attribute("Upload")->value();

            std::string strView;
            if (pRecordNode->first_attribute("View") != NULL)
            {
                strView = pRecordNode->first_attribute("View")->value();
            }

            bool bPublic = lexical_cast<bool>(pstrPublic);
            bool bPrivate = lexical_cast<bool>(pstrPrivate);
            bool bSave = lexical_cast<bool>(pstrSave);
			bool bCache = lexical_cast<bool>(pstrCache);
			bool bRef = lexical_cast<bool>(pstrCache);
			bool bForce = lexical_cast<bool>(pstrCache);
			bool bUpload = lexical_cast<bool>(pstrUpload);

			NF_SHARE_PTR<NFDataList> recordVar(NF_NEW NFDataList());
			NF_SHARE_PTR<NFDataList> recordTag(NF_NEW NFDataList());

            for (rapidxml::xml_node<>* recordColNode = pRecordNode->first_node(); recordColNode;  recordColNode = recordColNode->next_sibling())
            {
                //const char* pstrColName = recordColNode->first_attribute( "Id" )->value();
                NFData TData;
                const char* pstrColType = recordColNode->first_attribute("Type")->value();
                if (TDATA_UNKNOWN == ComputerType(pstrColType, TData))
                {
                    //assert(0);
                    NFASSERT(0, pstrRecordName, __FILE__, __FUNCTION__);
                }
                // 将 每个列的 Type 的值加到 list 里面
                recordVar->Append(TData);
                //////////////////////////////////////////////////////////////////////////
                if (recordColNode->first_attribute("Tag") != NULL)
                {
                    const char* pstrTag = recordColNode->first_attribute("Tag")->value();
                    // 将 每个列的 Tag 的值加到 list 里面
                    recordTag->Add(pstrTag);
                }
                else
                {
                    // 添加空值
                    recordTag->Add("");
                }
            }

            NF_SHARE_PTR<NFIRecord> xRecord = pClass->GetRecordManager()->AddRecord(NFGUID(), pstrRecordName, recordVar, recordTag, atoi(pstrRow));

			xRecord->SetPublic(bPublic);
            xRecord->SetPrivate(bPrivate);
            xRecord->SetSave(bSave);
            xRecord->SetCache(bCache);
			xRecord->SetRef(bRef);
			xRecord->SetForce(bForce);
			xRecord->SetUpload(bUpload);
        }
    }

    return true;
}

bool NFClassModule::AddComponents(rapidxml::xml_node<>* pComponentRootNode, NF_SHARE_PTR<NFIClass> pClass)
{
	/*
    for (rapidxml::xml_node<>* pComponentNode = pComponentRootNode->first_node(); pComponentNode; pComponentNode = pComponentNode->next_sibling())
    {
        if (pComponentNode)
        {
            const char* strComponentName = pComponentNode->first_attribute("Name")->value();
            const char* strLanguage = pComponentNode->first_attribute("Language")->value();
            const char* strEnable = pComponentNode->first_attribute("Enable")->value();
            bool bEnable = lexical_cast<bool>(strEnable);
            if (bEnable)
            {
                if (pClass->GetComponentManager()->GetElement(strComponentName))
                {
                    //error
                    NFASSERT(0, strComponentName, __FILE__, __FUNCTION__);
                    continue;
                }
                NF_SHARE_PTR<NFIComponent> xComponent(NF_NEW NFIComponent(NFGUID(), strComponentName));
                pClass->GetComponentManager()->AddComponent(strComponentName, xComponent);
            }
        }
    }
	*/
    return true;
}

bool NFClassModule::AddClassInclude(const char* pstrClassFilePath, NF_SHARE_PTR<NFIClass> pClass)
{   // 如果已经添加了 返回false
    if (pClass->Find(pstrClassFilePath))
    {
        return false;
    }

    //////////////////////////////////////////////////////////////////////////
    // 设置 路径为 ../FDataCfg/Struct/... 
    std::string strFile = pPluginManager->GetConfigPath() + pstrClassFilePath;
	std::string strContent;
	pPluginManager->GetFileContent(strFile, strContent);

	rapidxml::xml_document<> xDoc;
    xDoc.parse<0>((char*)strContent.c_str());
    //////////////////////////////////////////////////////////////////////////

    //support for unlimited layer class inherits
    rapidxml::xml_node<>* root = xDoc.first_node();

    // 获取 Propertys 属性
    rapidxml::xml_node<>* pRropertyRootNode = root->first_node("Propertys");
    if (pRropertyRootNode)
    {
        AddPropertys(pRropertyRootNode, pClass);
    }

    //////////////////////////////////////////////////////////////////////////
    //and record
    rapidxml::xml_node<>* pRecordRootNode = root->first_node("Records");
    if (pRecordRootNode)
    {
        AddRecords(pRecordRootNode, pClass);
    }

    rapidxml::xml_node<>* pComponentRootNode = root->first_node("Components");
    if (pComponentRootNode)
    {
        AddComponents(pComponentRootNode, pClass);
    }

    //pClass->mvIncludeFile.push_back( pstrClassFilePath );
    //and include file
    rapidxml::xml_node<>* pIncludeRootNode = root->first_node("Includes");
    if (pIncludeRootNode)
    {
        for (rapidxml::xml_node<>* includeNode = pIncludeRootNode->first_node(); includeNode; includeNode = includeNode->next_sibling())
        {   // 通过id 属性得到 文件的路径
            const char* pstrIncludeFile = includeNode->first_attribute("Id")->value();
            //std::vector<std::string>::iterator it = std::find( pClass->mvIncludeFile.begin(), pClass->mvIncludeFile..end(), pstrIncludeFile );
            //添加 include文件里面的 property  和 record
            if (AddClassInclude(pstrIncludeFile, pClass))
            {
                pClass->Add(pstrIncludeFile);
            }
        }
    }

    return true;
}

bool NFClassModule::AddClass(const char* pstrClassFilePath, NF_SHARE_PTR<NFIClass> pClass)
{
    NF_SHARE_PTR<NFIClass> pParent = pClass->GetParent();
    //首次 parent为null，直接跳过循环
    while (pParent)
    {
        //inherited some properties form class of parent
        std::string strFileName = "";
        pParent->First(strFileName);
        while (!strFileName.empty())
        {
            if (pClass->Find(strFileName))
            {
                strFileName.clear();
                continue;
            }

            if (AddClassInclude(strFileName.c_str(), pClass))
            {
                pClass->Add(strFileName);
            }

            strFileName.clear();
            pParent->Next(strFileName);
        }

        pParent = pParent->GetParent();
    }

    //////////////////////////////////////////////////////////////////////////
    //添加文件里面的property 和 record
    if (AddClassInclude(pstrClassFilePath, pClass))
    {
        pClass->Add(pstrClassFilePath);
    }

    //file.close();

    return true;
}
// 设置 父子类关系
bool NFClassModule::AddClass(const std::string& strClassName, const std::string& strParentName)
{
    NF_SHARE_PTR<NFIClass> pParentClass = GetElement(strParentName);
    NF_SHARE_PTR<NFIClass> pChildClass = GetElement(strClassName);
    if (!pChildClass)
    {
        pChildClass = NF_SHARE_PTR<NFIClass>(NF_NEW NFClass(strClassName));
        AddElement(strClassName, pChildClass);
        //pChildClass = CreateElement( strClassName );

        pChildClass->SetTypeName("");
        pChildClass->SetInstancePath("");

        if (pParentClass)
        {
            pChildClass->SetParent(pParentClass);
        }
    }

    return true;
}

bool NFClassModule::Load(rapidxml::xml_node<>* attrNode, NF_SHARE_PTR<NFIClass> pParentClass)
{
    // 获取class 对象的三个属性，第一次传进来的是 null
    const char* pstrLogicClassName = attrNode->first_attribute("Id")->value();
    const char* pstrPath = attrNode->first_attribute("Path")->value();
    const char* pstrInstancePath = attrNode->first_attribute("InstancePath")->value();

    //printf( "-----------------------------------------------------\n");
    //printf( "%s:\n", pstrLogicClassName );

    NF_SHARE_PTR<NFIClass> pClass(NF_NEW NFClass(pstrLogicClassName));
    // 新建 ID属性值的对象，将其存入到当前module的map中。
    AddElement(pstrLogicClassName, pClass);
    pClass->SetParent(pParentClass);
    pClass->SetInstancePath(pstrInstancePath);
    // 从 NFDataCfg/Struct 加载文件内容，将内容保存到pclass中
    AddClass(pstrPath, pClass);

    for (rapidxml::xml_node<>* pDataNode = attrNode->first_node(); pDataNode; pDataNode = pDataNode->next_sibling())
    {
        //her children
        Load(pDataNode, pClass);
    }
    //printf( "-----------------------------------------------------\n");
    return true;
}

bool NFClassModule::Load()
{
    //////////////////////////////////////////////////////////////////////////
	std::string strFile = pPluginManager->GetConfigPath() + msConfigFileName;
	std::string strContent;
	pPluginManager->GetFileContent(strFile, strContent);

	rapidxml::xml_document<> xDoc;
	xDoc.parse<0>((char*)strContent.c_str());
    //////////////////////////////////////////////////////////////////////////
    //support for unlimited layer class inherits
    rapidxml::xml_node<>* root = xDoc.first_node();
    // 递归每个class对象
    for (rapidxml::xml_node<>* attrNode = root->first_node(); attrNode; attrNode = attrNode->next_sibling())
    {
        Load(attrNode, NULL);
    }

    if (m_pBackupClassModule) m_pBackupClassModule->Load();

    return true;
}

bool NFClassModule::Save()
{
    if (m_pBackupClassModule) m_pBackupClassModule->Save();
    return true;
}

NF_SHARE_PTR<NFIPropertyManager> NFClassModule::GetClassPropertyManager(const std::string& strClassName)
{
    NF_SHARE_PTR<NFIClass> pClass = GetElement(strClassName);
    if (pClass)
    {
        return pClass->GetPropertyManager();
    }

    return NULL;
}

NF_SHARE_PTR<NFIRecordManager> NFClassModule::GetClassRecordManager(const std::string& strClassName)
{
    NF_SHARE_PTR<NFIClass> pClass = GetElement(strClassName);
    if (pClass)
    {
        return pClass->GetRecordManager();
    }

    return NULL;
}

bool NFClassModule::Clear()
{
    if (m_pBackupClassModule) m_pBackupClassModule->Clear();
    return true;
}

NFIClassModule* NFClassModule::GetBackupClassModule()
{
	return m_pBackupClassModule;
}

bool NFClassModule::AddClassCallBack(const std::string& strClassName, const CLASS_EVENT_FUNCTOR_PTR& cb)
{
    NF_SHARE_PTR<NFIClass> pClass = GetElement(strClassName);
    if (nullptr == pClass)
    {
        return false;
    }
    // 将回调函数 添加到 List 
    return pClass->AddClassCallBack(cb);
}
// 不断执行 list 里面的回调函数
bool NFClassModule::DoEvent(const NFGUID& objectID, const std::string& strClassName, const CLASS_OBJECT_EVENT eClassEvent, const NFDataList& valueList)
{
    NF_SHARE_PTR<NFIClass> pClass = GetElement(strClassName);
    if (nullptr == pClass)
    {
        return false;
    }

    return pClass->DoEvent(objectID, eClassEvent, valueList);
}

```