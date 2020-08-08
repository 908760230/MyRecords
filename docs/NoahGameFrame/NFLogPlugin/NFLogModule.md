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

