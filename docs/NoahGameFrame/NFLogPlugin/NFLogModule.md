# NFLogModule 解析

## NFLogModule.h
```c++

#ifndef NF_LOG_MODULE_H
#define NF_LOG_MODULE_H

#include "NFComm/NFPluginModule/NFILogModule.h"
#include "NFComm/NFCore/NFPerformance.hpp"

class NFLogModule
    : public NFILogModule
{
public:

    NFLogModule(NFIPluginManager* p);
    virtual ~NFLogModule() {}

    virtual bool Awake();
    virtual bool Init();
    virtual bool Shut();

    virtual bool BeforeShut();
    virtual bool AfterInit();

    virtual bool Execute();

    ///////////////////////////////////////////////////////////////////////
    virtual void LogStack();

    virtual bool LogElement(const NF_LOG_LEVEL nll, const NFGUID ident, const std::string& strElement, const std::string& strDesc, const char* func = "", int line = 0);
    virtual bool LogProperty(const NF_LOG_LEVEL nll, const NFGUID ident, const std::string& strProperty, const std::string& strDesc, const char* func = "", int line = 0);
    virtual bool LogRecord(const NF_LOG_LEVEL nll, const NFGUID ident, const std::string& strRecord, const std::string& strDesc, const int nRow, const int nCol, const char* func = "", int line = 0);
    virtual bool LogRecord(const NF_LOG_LEVEL nll, const NFGUID ident, const std::string& strRecord, const std::string& strDesc, const char* func = "", int line = 0);
    virtual bool LogObject(const NF_LOG_LEVEL nll, const NFGUID ident, const std::string& strDesc, const char* func = "", int line = 0);

    virtual bool LogDebug(const std::string& strLog, const char* func = "", int line = 0);
    virtual bool LogInfo(const std::string& strLog, const  char* func = "", int line = 0);
    virtual bool LogWarning(const std::string& strLog, const char* func = "", int line = 0);
    virtual bool LogError(const std::string& strLog, const char* func = "", int line = 0);
    virtual bool LogFatal(const std::string& strLog, const char* func = "", int line = 0);

    virtual bool LogDebug(const std::ostringstream& stream, const char* func = "", int line = 0);
    virtual bool LogInfo(const std::ostringstream& stream, const  char* func = "", int line = 0);
    virtual bool LogWarning(const std::ostringstream& stream, const char* func = "", int line = 0);
    virtual bool LogError(const std::ostringstream& stream, const char* func = "", int line = 0);
    virtual bool LogFatal(const std::ostringstream& stream, const char* func = "", int line = 0);

    virtual bool LogDebug(const NFGUID ident, const std::string& strLog, const char* func = "", int line = 0);
    virtual bool LogInfo(const NFGUID ident, const std::string& strLog, const  char* func = "", int line = 0);
    virtual bool LogWarning(const NFGUID ident, const std::string& strLog, const char* func = "", int line = 0);
    virtual bool LogError(const NFGUID ident, const std::string& strLog, const char* func = "", int line = 0);
    virtual bool LogFatal(const NFGUID ident, const std::string& strLog, const char* func = "", int line = 0);

    virtual bool LogDebug(const NFGUID ident, const std::ostringstream& stream, const char* func = "", int line = 0);
    virtual bool LogInfo(const NFGUID ident, const std::ostringstream& stream, const  char* func = "", int line = 0);
    virtual bool LogWarning(const NFGUID ident, const std::ostringstream& stream, const char* func = "", int line = 0);
    virtual bool LogError(const NFGUID ident, const std::ostringstream& stream, const char* func = "", int line = 0);
    virtual bool LogFatal(const NFGUID ident, const std::ostringstream& stream, const char* func = "", int line = 0);

    virtual bool LogDebugFunctionDump(const NFGUID ident, const int nMsg, const std::string& strArg, const char* func = "", const int line = 0);
    virtual bool ChangeLogLevel(const std::string& strLevel);
    
    virtual void SetHooker(LOG_HOOKER_FUNCTOR_PTR hooker);
    virtual void StackTrace();

protected:
    virtual bool Log(const NF_LOG_LEVEL nll, const char* format, ...);

    static bool CheckLogFileExist(const char* filename);
    static void rolloutHandler(const char* filename, std::size_t size);

	std::string GetConfigPath(const std::string& fileName);

private:
    std::string mstrLocalStream;
    LOG_HOOKER_FUNCTOR_PTR mLogHooker;
    static unsigned int idx;
    uint64_t mnLogCountTotal;
	std::list<NFPerformance> mxPerformanceList;
};
```

## NFLogModule.cpp

### 构造函数 
保存了 NFIPluginManager的指针，设置日志库。

**StrictLogFileSizeCheck** ： 确保每次记录的时候都会检查日志文件的大小。

**DisableApplicationAbortOnFatalLog** ： 防止Fatal级别日志中断程序。
```c++
NFLogModule::NFLogModule(NFIPluginManager* p)
{
    pPluginManager = p;

	el::Loggers::addFlag(el::LoggingFlag::StrictLogFileSizeCheck);
	el::Loggers::addFlag(el::LoggingFlag::DisableApplicationAbortOnFatalLog);
}
```
### awake

```c++

bool NFLogModule::Awake()
{
	mnLogCountTotal = 0;

    //这个会是空串
	std::string strLogConfigName = pPluginManager->GetLogConfigName();
	if (strLogConfigName.empty())
	{
		strLogConfigName = pPluginManager->GetAppName();
	}

    // 获取log配置文件
	string strAppLogName = GetConfigPath(strLogConfigName);

	el::Configurations conf(strAppLogName);

	el::Configuration* pConfiguration = conf.get(el::Level::Debug, el::ConfigurationType::Filename);
    // 如果没有获取到 DBServer 或者 GamerServer 等 的配置文件 就加载默认的配置文件
	if (pConfiguration == nullptr)
	{
		conf = el::Configurations(GetConfigPath("Default"));
		pConfiguration = conf.get(el::Level::Debug, el::ConfigurationType::Filename);
	}

	const std::string& strFileName = pConfiguration->value();
	pConfiguration->setValue(pPluginManager->GetConfigPath() + strFileName);

	std::cout << "LogConfig: " << strAppLogName << std::endl;

	el::Loggers::reconfigureAllLoggers(conf);
	el::Helpers::installPreRollOutCallback(rolloutHandler);

	return true;
}
```
目前 我看到的是GetLogConfigName() 返回的是空串。

setValue 重新设置日志文件为 “logs\\master_server_debug_%datetime{%Y%M%d}.log”（假如当前是MasterServer）
```c++

std::string NFLogModule::GetConfigPath(const std::string & fileName)
{
	std::string strAppLogName;
#if NF_PLATFORM == NF_PLATFORM_WIN
#ifdef NF_DEBUG_MODE
	strAppLogName = pPluginManager->GetConfigPath() + "NFDataCfg/Debug/logconfig/" + fileName + "_win.conf";
#else
	strAppLogName = pPluginManager->GetConfigPath() + "NFDataCfg/Release/logconfig/" + fileName + "_win.conf";
#endif

#else
#ifdef NF_DEBUG_MODE
	strAppLogName = pPluginManager->GetConfigPath() + "NFDataCfg/Debug/logconfig/" + fileName + ".conf";
#else
	strAppLogName = pPluginManager->GetConfigPath() + "NFDataCfg/Release/logconfig/" + fileName + ".conf";
#endif
#endif

	return strAppLogName;
}

```

假如 fileName 是 MasterServe 那么返回的字符串是 “../NFDataCfg/Debug/logconfig/MasterServer_win.conf”

### Log函数

```c++

bool NFLogModule::Log(const NF_LOG_LEVEL nll, const char* format, ...)
{
    //日志记录的总数加一
    mnLogCountTotal++;

    char szBuffer[1024 * 10] = {0};

    va_list args;
    va_start(args, format);
    vsnprintf(szBuffer, sizeof(szBuffer) - 1, format, args);
    va_end(args);

    mstrLocalStream.clear();

    mstrLocalStream.append(std::to_string(mnLogCountTotal));
    mstrLocalStream.append(" | ");
    mstrLocalStream.append(std::to_string(pPluginManager->GetAppID()));
    mstrLocalStream.append(" | ");
    mstrLocalStream.append(szBuffer);

    if (mLogHooker)
    {
        mLogHooker.get()->operator()(nll, mstrLocalStream);
    }

    switch (nll)
    {
        case NFILogModule::NLL_DEBUG_NORMAL:
			{
				std::cout << termcolor::green;
				LOG(DEBUG) << mstrLocalStream;
			}
			break;
        case NFILogModule::NLL_INFO_NORMAL:
			{
				std::cout << termcolor::green;
				LOG(INFO) << mstrLocalStream;
			}	
			break;
        case NFILogModule::NLL_WARING_NORMAL:
			{
				std::cout << termcolor::yellow;
				LOG(WARNING) << mstrLocalStream;
			}
			break;
        case NFILogModule::NLL_ERROR_NORMAL:
			{
				std::cout << termcolor::red;
				LOG(ERROR) << mstrLocalStream;
				//LogStack();
			}
			break;
        case NFILogModule::NLL_FATAL_NORMAL:
			{
				std::cout << termcolor::red;
				LOG(FATAL) << mstrLocalStream;
			}
			break;
        default:
			{
				std::cout << termcolor::green;
				LOG(INFO) << mstrLocalStream;
			}
			break;
    }

	std::cout<<termcolor::reset;

    return true;
}
```
将参数保存到mstrLocalStream字符串中 根据log的级别 改变字体的颜色进行log输出。

### LogStack 函数

```c++
void NFLogModule::LogStack()
{

    //To Add
#if NF_PLATFORM == NF_PLATFORM_WIN
    time_t t = time(0);
    char szDmupName[MAX_PATH];
    tm* ptm = localtime(&t);

    sprintf(szDmupName, "%d_%d_%d_%d_%d_%d.dmp",  ptm->tm_year + 1900, ptm->tm_mon, ptm->tm_mday, ptm->tm_hour, ptm->tm_min, ptm->tm_sec);
    
    HANDLE hDumpFile = CreateFileA(szDmupName, GENERIC_WRITE, 0, NULL, CREATE_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL);

    
    MINIDUMP_EXCEPTION_INFORMATION dumpInfo;
    //dumpInfo.ExceptionPointers = pException;
    dumpInfo.ThreadId = GetCurrentThreadId();
    dumpInfo.ClientPointers = TRUE;

    
    MiniDumpWriteDump(GetCurrentProcess(), GetCurrentProcessId(), hDumpFile, MiniDumpNormal, &dumpInfo, NULL, NULL);

    CloseHandle(hDumpFile);
#else
	int size = 16;
	void * array[16];
	int stack_num = backtrace(array, size);
	char ** stacktrace = backtrace_symbols(array, stack_num);
	for (int i = 0; i < stack_num; ++i)
	{
		//printf("%s\n", stacktrace[i]);
        
        Log(NLL_FATAL_NORMAL, "%s", stacktrace[i]);
	}

	free(stacktrace);
#endif

}
```
将时间 和线程信息 保存到Dump文件中


### 完整代码

```c++

#define GLOG_NO_ABBREVIATED_SEVERITIES
#include <stdarg.h>
#include "NFLogModule.h"
#include "NFLogPlugin.h"
#include "termcolor.hpp"
#include "NFComm/NFCore/easylogging++.h"

#if NF_PLATFORM != NF_PLATFORM_WIN
#include <execinfo.h>
#endif


unsigned int NFLogModule::idx = 0;

bool NFLogModule::CheckLogFileExist(const char* filename)
{
    std::stringstream stream;
    stream << filename << "." << ++idx;
    std::fstream file;
    file.open(stream.str(), std::ios::in);
    if (file)
    {
        return CheckLogFileExist(filename);
    }

    return false;
}

void NFLogModule::rolloutHandler(const char* filename, std::size_t size)
{
    std::stringstream stream;
    if (!CheckLogFileExist(filename))
    {
        stream << filename << "." << idx;
        rename(filename, stream.str().c_str());
    }
}

std::string NFLogModule::GetConfigPath(const std::string & fileName)
{
	std::string strAppLogName;
#if NF_PLATFORM == NF_PLATFORM_WIN
#ifdef NF_DEBUG_MODE
	strAppLogName = pPluginManager->GetConfigPath() + "NFDataCfg/Debug/logconfig/" + fileName + "_win.conf";
#else
	strAppLogName = pPluginManager->GetConfigPath() + "NFDataCfg/Release/logconfig/" + fileName + "_win.conf";
#endif

#else
#ifdef NF_DEBUG_MODE
	strAppLogName = pPluginManager->GetConfigPath() + "NFDataCfg/Debug/logconfig/" + fileName + ".conf";
#else
	strAppLogName = pPluginManager->GetConfigPath() + "NFDataCfg/Release/logconfig/" + fileName + ".conf";
#endif
#endif

	return strAppLogName;
}

NFLogModule::NFLogModule(NFIPluginManager* p)
{
    pPluginManager = p;

	el::Loggers::addFlag(el::LoggingFlag::StrictLogFileSizeCheck);
	el::Loggers::addFlag(el::LoggingFlag::DisableApplicationAbortOnFatalLog);
}

bool NFLogModule::Awake()
{
	mnLogCountTotal = 0;

	std::string strLogConfigName = pPluginManager->GetLogConfigName();
	if (strLogConfigName.empty())
	{
		strLogConfigName = pPluginManager->GetAppName();
	}

	string strAppLogName = GetConfigPath(strLogConfigName);

	el::Configurations conf(strAppLogName);

	el::Configuration* pConfiguration = conf.get(el::Level::Debug, el::ConfigurationType::Filename);
	if (pConfiguration == nullptr)
	{
		conf = el::Configurations(GetConfigPath("Default"));
		pConfiguration = conf.get(el::Level::Debug, el::ConfigurationType::Filename);
	}

	const std::string& strFileName = pConfiguration->value();
	pConfiguration->setValue(pPluginManager->GetConfigPath() + strFileName);

	std::cout << "LogConfig: " << strAppLogName << std::endl;

	el::Loggers::reconfigureAllLoggers(conf);
	el::Helpers::installPreRollOutCallback(rolloutHandler);

	return true;
}

bool NFLogModule::Init()
{
   
    return true;
}

bool NFLogModule::Shut()
{
    el::Helpers::uninstallPreRollOutCallback();

    return true;
}

bool NFLogModule::BeforeShut()
{
    return true;

}

bool NFLogModule::AfterInit()
{

    return true;
}

bool NFLogModule::Execute()
{
    return true;

}

bool NFLogModule::Log(const NF_LOG_LEVEL nll, const char* format, ...)
{
    mnLogCountTotal++;

    char szBuffer[1024 * 10] = {0};

    va_list args;
    va_start(args, format);
    vsnprintf(szBuffer, sizeof(szBuffer) - 1, format, args);
    va_end(args);

    mstrLocalStream.clear();

    mstrLocalStream.append(std::to_string(mnLogCountTotal));
    mstrLocalStream.append(" | ");
    mstrLocalStream.append(std::to_string(pPluginManager->GetAppID()));
    mstrLocalStream.append(" | ");
    mstrLocalStream.append(szBuffer);

    if (mLogHooker)
    {
        mLogHooker.get()->operator()(nll, mstrLocalStream);
    }

    switch (nll)
    {
        case NFILogModule::NLL_DEBUG_NORMAL:
			{
				std::cout << termcolor::green;
				LOG(DEBUG) << mstrLocalStream;
			}
			break;
        case NFILogModule::NLL_INFO_NORMAL:
			{
				std::cout << termcolor::green;
				LOG(INFO) << mstrLocalStream;
			}	
			break;
        case NFILogModule::NLL_WARING_NORMAL:
			{
				std::cout << termcolor::yellow;
				LOG(WARNING) << mstrLocalStream;
			}
			break;
        case NFILogModule::NLL_ERROR_NORMAL:
			{
				std::cout << termcolor::red;
				LOG(ERROR) << mstrLocalStream;
				//LogStack();
			}
			break;
        case NFILogModule::NLL_FATAL_NORMAL:
			{
				std::cout << termcolor::red;
				LOG(FATAL) << mstrLocalStream;
			}
			break;
        default:
			{
				std::cout << termcolor::green;
				LOG(INFO) << mstrLocalStream;
			}
			break;
    }

	std::cout<<termcolor::reset;

    return true;
}

bool NFLogModule::LogElement(const NF_LOG_LEVEL nll, const NFGUID ident, const std::string& strElement, const std::string& strDesc, const char* func, int line)
{
    if (line > 0)
    {
        Log(nll, "[ELEMENT] Indent[%s] Element[%s] %s %s %d", ident.ToString().c_str(), strElement.c_str(), strDesc.c_str(), func, line);
    }
    else
    {
        Log(nll, "[ELEMENT] Indent[%s] Element[%s] %s", ident.ToString().c_str(), strElement.c_str(), strDesc.c_str());
    }

    return true;
}

bool NFLogModule::LogProperty(const NF_LOG_LEVEL nll, const NFGUID ident, const std::string& strProperty, const std::string& strDesc, const char* func, int line)
{
    if (line > 0)
    {
        Log(nll, "[PROPERTY] Indent[%s] Property[%s] %s %s %d", ident.ToString().c_str(), strProperty.c_str(), strDesc.c_str(), func, line);
    }
    else
    {
        Log(nll, "[PROPERTY] Indent[%s] Property[%s] %s", ident.ToString().c_str(), strProperty.c_str(), strDesc.c_str());
    }

    return true;

}

bool NFLogModule::LogRecord(const NF_LOG_LEVEL nll, const NFGUID ident, const std::string& strRecord, const std::string& strDesc, const int nRow, const int nCol, const char* func, int line)
{
    if (line > 0)
    {
        Log(nll, "[RECORD] Indent[%s] Record[%s] Row[%d] Col[%d] %s %s %d", ident.ToString().c_str(), strRecord.c_str(), nRow, nCol, strDesc.c_str(), func, line);
    }
    else
    {
        Log(nll, "[RECORD] Indent[%s] Record[%s] Row[%d] Col[%d] %s", ident.ToString().c_str(), strRecord.c_str(), nRow, nCol, strDesc.c_str());
    }

    return true;

}

bool NFLogModule::LogRecord(const NF_LOG_LEVEL nll, const NFGUID ident, const std::string& strRecord, const std::string& strDesc, const char* func, int line)
{
    if (line > 0)
    {
        Log(nll, "[RECORD] Indent[%s] Record[%s] %s %s %d", ident.ToString().c_str(), strRecord.c_str(), strDesc.c_str(), func, line);
    }
    else
    {
        Log(nll, "[RECORD] Indent[%s] Record[%s] %s", ident.ToString().c_str(), strRecord.c_str(), strDesc.c_str());
    }

    return true;
}

bool NFLogModule::LogObject(const NF_LOG_LEVEL nll, const NFGUID ident, const std::string& strDesc, const char* func, int line)
{
    if (line > 0)
    {
        Log(nll, "[OBJECT] Indent[%s] %s %s %d", ident.ToString().c_str(), strDesc.c_str(), func, line);
    }
    else
    {
        Log(nll, "[OBJECT] Indent[%s] %s", ident.ToString().c_str(), strDesc.c_str());
    }

    return true;

}

void NFLogModule::LogStack()
{

    //To Add
#if NF_PLATFORM == NF_PLATFORM_WIN
    time_t t = time(0);
    char szDmupName[MAX_PATH];
    tm* ptm = localtime(&t);

    sprintf(szDmupName, "%d_%d_%d_%d_%d_%d.dmp",  ptm->tm_year + 1900, ptm->tm_mon, ptm->tm_mday, ptm->tm_hour, ptm->tm_min, ptm->tm_sec);
    
    HANDLE hDumpFile = CreateFileA(szDmupName, GENERIC_WRITE, 0, NULL, CREATE_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL);

    
    MINIDUMP_EXCEPTION_INFORMATION dumpInfo;
    //dumpInfo.ExceptionPointers = pException;
    dumpInfo.ThreadId = GetCurrentThreadId();
    dumpInfo.ClientPointers = TRUE;

    
    MiniDumpWriteDump(GetCurrentProcess(), GetCurrentProcessId(), hDumpFile, MiniDumpNormal, &dumpInfo, NULL, NULL);

    CloseHandle(hDumpFile);
#else
	int size = 16;
	void * array[16];
	int stack_num = backtrace(array, size);
	char ** stacktrace = backtrace_symbols(array, stack_num);
	for (int i = 0; i < stack_num; ++i)
	{
		//printf("%s\n", stacktrace[i]);
        
        Log(NLL_FATAL_NORMAL, "%s", stacktrace[i]);
	}

	free(stacktrace);
#endif
 
    
}
bool NFLogModule::LogDebugFunctionDump(const NFGUID ident, const int nMsg, const std::string& strArg,  const char* func /*= ""*/, const int line /*= 0*/)
{
    //#ifdef NF_DEBUG_MODE
    LogDebug(ident, strArg + "MsgID:" + std::to_string(nMsg), func, line);
    //#endif
    return true;
}

bool NFLogModule::ChangeLogLevel(const std::string& strLevel)
{
    el::Level logLevel = el::LevelHelper::convertFromString(strLevel.c_str());
    el::Logger* pLogger = el::Loggers::getLogger("default");
    if (NULL == pLogger)
    {
        return false;
    }

    el::Configurations* pConfigurations = pLogger->configurations();
    if (NULL == pConfigurations)
    {
        return false;
    }

    switch (logLevel)
    {
        case el::Level::Fatal:
        {
            el::Configuration errorConfiguration(el::Level::Error, el::ConfigurationType::Enabled, "false");
            pConfigurations->set(&errorConfiguration);
        }
        case el::Level::Error:
        {
            el::Configuration warnConfiguration(el::Level::Warning, el::ConfigurationType::Enabled, "false");
            pConfigurations->set(&warnConfiguration);
        }
        case el::Level::Warning:
        {
            el::Configuration infoConfiguration(el::Level::Info, el::ConfigurationType::Enabled, "false");
            pConfigurations->set(&infoConfiguration);
        }
        case el::Level::Info:
        {
            el::Configuration debugConfiguration(el::Level::Debug, el::ConfigurationType::Enabled, "false");
            pConfigurations->set(&debugConfiguration);

        }
        case el::Level::Debug:
            break;
        default:
            break;
    }

    el::Loggers::reconfigureAllLoggers(*pConfigurations);
    LogInfo("[Log] Change log level as " + strLevel, __FUNCTION__, __LINE__);
    return true;
}

bool NFLogModule::LogDebug(const std::string& strLog, const char* func , int line)
{
    if (line > 0)
     {
         Log(NF_LOG_LEVEL::NLL_DEBUG_NORMAL, "%s %s %d", strLog.c_str(), func, line);
     }
     else
     {
         Log(NF_LOG_LEVEL::NLL_DEBUG_NORMAL, "%s", strLog.c_str());
     }

     return true;
}

bool NFLogModule::LogInfo(const std::string& strLog, const  char* func, int line)
{
    if (line > 0)
     {
         Log(NF_LOG_LEVEL::NLL_INFO_NORMAL, "%s %s %d", strLog.c_str(), func, line);
     }
     else
     {
         Log(NF_LOG_LEVEL::NLL_INFO_NORMAL, "%s", strLog.c_str());
     }

     return true;
}

bool NFLogModule::LogWarning(const std::string& strLog, const char* func , int line)
{
    if (line > 0)
     {
         Log(NF_LOG_LEVEL::NLL_WARING_NORMAL, "%s %s %d", strLog.c_str(), func, line);
     }
     else
     {
         Log(NF_LOG_LEVEL::NLL_WARING_NORMAL, "%s", strLog.c_str());
     }

     return true;
}

bool NFLogModule::LogError(const std::string& strLog, const char* func , int line)
{
    if (line > 0)
     {
         Log(NF_LOG_LEVEL::NLL_ERROR_NORMAL, "%s %s %d", strLog.c_str(), func, line);
     }
     else
     {
         Log(NF_LOG_LEVEL::NLL_ERROR_NORMAL, "%s", strLog.c_str());
     }

     return true;
}

bool NFLogModule::LogFatal(const std::string& strLog, const char* func , int line)
{
    if (line > 0)
     {
         Log(NF_LOG_LEVEL::NLL_FATAL_NORMAL, "%s %s %d", strLog.c_str(), func, line);
     }
     else
     {
         Log(NF_LOG_LEVEL::NLL_FATAL_NORMAL, "%s", strLog.c_str());
     }

     return true;
}


bool NFLogModule::LogDebug(const std::ostringstream& stream, const char* func , int line)
{
    if (line > 0)
     {
         Log(NF_LOG_LEVEL::NLL_DEBUG_NORMAL, "%s %s %d", stream.str().c_str(), func, line);
     }
     else
     {
         Log(NF_LOG_LEVEL::NLL_DEBUG_NORMAL, "%s", stream.str().c_str());
     }

     return true;
}

bool NFLogModule::LogInfo(const std::ostringstream& stream, const  char* func, int line)
{
    if (line > 0)
     {
         Log(NF_LOG_LEVEL::NLL_INFO_NORMAL, "%s %s %d", stream.str().c_str(), func, line);
     }
     else
     {
         Log(NF_LOG_LEVEL::NLL_INFO_NORMAL, "%s", stream.str().c_str());
     }

     return true;
}

bool NFLogModule::LogWarning(const std::ostringstream& stream, const char* func , int line)
{
    if (line > 0)
     {
         Log(NF_LOG_LEVEL::NLL_WARING_NORMAL, "%s %s %d", stream.str().c_str(), func, line);
     }
     else
     {
         Log(NF_LOG_LEVEL::NLL_WARING_NORMAL, "%s", stream.str().c_str());
     }

     return true;
}

bool NFLogModule::LogError(const std::ostringstream& stream, const char* func , int line)
{
    if (line > 0)
     {
         Log(NF_LOG_LEVEL::NLL_ERROR_NORMAL, "%s %s %d", stream.str().c_str(), func, line);
     }
     else
     {
         Log(NF_LOG_LEVEL::NLL_ERROR_NORMAL, "%s", stream.str().c_str());
     }

     return true;
}

bool NFLogModule::LogFatal(const std::ostringstream& stream, const char* func , int line)
{
    if (line > 0)
     {
         Log(NF_LOG_LEVEL::NLL_FATAL_NORMAL, "%s %s %d", stream.str().c_str(), func, line);
     }
     else
     {
         Log(NF_LOG_LEVEL::NLL_FATAL_NORMAL, "%s", stream.str().c_str());
     }

     return true;
}


bool NFLogModule::LogDebug(const NFGUID ident, const std::string& strLog, const char* func , int line)
{
    if (line > 0)
     {
         Log(NF_LOG_LEVEL::NLL_DEBUG_NORMAL, "Indent[%s] %s %s %d", ident.ToString().c_str(), strLog.c_str(), func, line);
     }
     else
     {
         Log(NF_LOG_LEVEL::NLL_DEBUG_NORMAL, "Indent[%s] %s", ident.ToString().c_str(), strLog.c_str());
     }

     return true;
}

bool NFLogModule::LogInfo(const NFGUID ident, const std::string& strLog, const  char* func, int line)
{
    if (line > 0)
     {
         Log(NF_LOG_LEVEL::NLL_INFO_NORMAL, "Indent[%s] %s %s %d", ident.ToString().c_str(), strLog.c_str(), func, line);
     }
     else
     {
         Log(NF_LOG_LEVEL::NLL_INFO_NORMAL, "Indent[%s] %s", ident.ToString().c_str(), strLog.c_str());
     }

     return true;
}

bool NFLogModule::LogWarning(const NFGUID ident, const std::string& strLog, const char* func , int line)
{
    if (line > 0)
     {
         Log(NF_LOG_LEVEL::NLL_WARING_NORMAL, "Indent[%s] %s %s %d", ident.ToString().c_str(), strLog.c_str(), func, line);
     }
     else
     {
         Log(NF_LOG_LEVEL::NLL_WARING_NORMAL, "Indent[%s] %s", ident.ToString().c_str(), strLog.c_str());
     }

     return true;
}

bool NFLogModule::LogError(const NFGUID ident, const std::string& strLog, const char* func , int line)
{
     if (line > 0)
     {
         Log(NF_LOG_LEVEL::NLL_ERROR_NORMAL, "Indent[%s] %s %s %d", ident.ToString().c_str(), strLog.c_str(), func, line);
     }
     else
     {
         Log(NF_LOG_LEVEL::NLL_ERROR_NORMAL, "Indent[%s] %s", ident.ToString().c_str(), strLog.c_str());
     }

     return true;
}

bool NFLogModule::LogFatal(const NFGUID ident, const std::string& strLog, const char* func , int line)
{
     if (line > 0)
     {
         Log(NF_LOG_LEVEL::NLL_FATAL_NORMAL, "Indent[%s] %s %s %d", ident.ToString().c_str(), strLog.c_str(), func, line);
     }
     else
     {
         Log(NF_LOG_LEVEL::NLL_FATAL_NORMAL, "Indent[%s] %s", ident.ToString().c_str(), strLog.c_str());
     }

     return true;
}


bool NFLogModule::LogDebug(const NFGUID ident, const std::ostringstream& stream, const char* func , int line)
{
    if (line > 0)
    {
        Log(NF_LOG_LEVEL::NLL_DEBUG_NORMAL, "Indent[%s] %s %s %d", ident.ToString().c_str(), stream.str().c_str(), func, line);
    }
    else
    {
        Log(NF_LOG_LEVEL::NLL_DEBUG_NORMAL, "Indent[%s] %s", ident.ToString().c_str(), stream.str().c_str());
    }

    return true;
}

bool NFLogModule::LogInfo(const NFGUID ident, const std::ostringstream& stream, const  char* func, int line)
{
    if (line > 0)
    {
        Log(NF_LOG_LEVEL::NLL_INFO_NORMAL, "Indent[%s] %s %s %d", ident.ToString().c_str(), stream.str().c_str(), func, line);
    }
    else
    {
        Log(NF_LOG_LEVEL::NLL_INFO_NORMAL, "Indent[%s] %s", ident.ToString().c_str(), stream.str().c_str());
    }

    return true;
}

bool NFLogModule::LogWarning(const NFGUID ident, const std::ostringstream& stream, const char* func , int line)
{
    if (line > 0)
    {
        Log(NF_LOG_LEVEL::NLL_WARING_NORMAL, "Indent[%s] %s %s %d", ident.ToString().c_str(), stream.str().c_str(), func, line);
    }
    else
    {
        Log(NF_LOG_LEVEL::NLL_WARING_NORMAL, "Indent[%s] %s", ident.ToString().c_str(), stream.str().c_str());
    }

    return true;
}

bool NFLogModule::LogError(const NFGUID ident, const std::ostringstream& stream, const char* func , int line)
{
    if (line > 0)
    {
        Log(NF_LOG_LEVEL::NLL_ERROR_NORMAL, "Indent[%s] %s %s %d", ident.ToString().c_str(), stream.str().c_str(), func, line);
    }
    else
    {
        Log(NF_LOG_LEVEL::NLL_ERROR_NORMAL, "Indent[%s] %s", ident.ToString().c_str(), stream.str().c_str());
    }

    return true;
}

bool NFLogModule::LogFatal(const NFGUID ident, const std::ostringstream& stream, const char* func , int line)
{
    if (line > 0)
    {
        Log(NF_LOG_LEVEL::NLL_FATAL_NORMAL, "Indent[%s] %s %s %d", ident.ToString().c_str(), stream.str().c_str(), func, line);
    }
    else
    {
        Log(NF_LOG_LEVEL::NLL_FATAL_NORMAL, "Indent[%s] %s", ident.ToString().c_str(), stream.str().c_str());
    }

    return true;
}

void NFLogModule::StackTrace(/*const NF_LOG_LEVEL nll = NFILogModule::NLL_FATAL_NORMAL*/)
{
#if NF_PLATFORM != NF_PLATFORM_WIN

    int size = 8;
    void * array[8];
    int stack_num = backtrace(array, size);
    char ** stacktrace = backtrace_symbols(array, stack_num);
    for (int i = 0; i < stack_num; ++i)
    {
    	//printf("%s\n", stacktrace[i]);
        Log(NLL_FATAL_NORMAL, "%s", stacktrace[i]);
    }
    

    free(stacktrace);
#else

	static const int MAX_STACK_FRAMES = 8;
	
	void *pStack[MAX_STACK_FRAMES];
 
	HANDLE process = GetCurrentProcess();
	SymInitialize(process, NULL, TRUE);
	WORD frames = CaptureStackBackTrace(0, MAX_STACK_FRAMES, pStack, NULL);
 
    Log(NLL_FATAL_NORMAL, "stack traceback: ");
	//LOG(FATAL) << "stack traceback: " << std::endl;
	for (WORD i = 0; i < frames; ++i) {
		DWORD64 address = (DWORD64)(pStack[i]);
 
		DWORD64 displacementSym = 0;
		char buffer[sizeof(SYMBOL_INFO) + MAX_SYM_NAME * sizeof(TCHAR)];
		PSYMBOL_INFO pSymbol = (PSYMBOL_INFO)buffer;
		pSymbol->SizeOfStruct = sizeof(SYMBOL_INFO);
		pSymbol->MaxNameLen = MAX_SYM_NAME;
 
		DWORD displacementLine = 0;
		IMAGEHLP_LINE64 line;
		//SymSetOptions(SYMOPT_LOAD_LINES);
		line.SizeOfStruct = sizeof(IMAGEHLP_LINE64);
 
		if (SymFromAddr(process, address, &displacementSym, pSymbol)
		 && SymGetLineFromAddr64(process, address, &displacementLine, &line))
        {
            Log(NLL_FATAL_NORMAL, "\t %s at %s : %d (0x%16d)", pSymbol->Name, line.FileName, line.LineNumber, pSymbol->Address);
			//LOG(FATAL) << "\t" << pSymbol->Name << " at " << line.FileName << ":" << line.LineNumber << "(0x" << std::hex << pSymbol->Address << std::dec << ")" << std::endl;
		}
		else
        {
            Log(NLL_FATAL_NORMAL, "\terror %d", GetLastError());
			//LOG(FATAL) << "\terror: " << GetLastError() << std::endl;
		}
	}
#endif
}

void NFLogModule::SetHooker(LOG_HOOKER_FUNCTOR_PTR hooker)
{
    mLogHooker = hooker;
}
```
