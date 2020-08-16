# NFServer

整个程序的入口处

## rund.bat

windows 下的启动程序，可以看到NFXXXXX  其实都是NFServer根据 参数来确定启动。而每个主程序都是从Plugin.xml中加载相应的插件

```bash
cd /d %~dp0
cd Debug

echo Starting NFMasterServer...
start "NFMasterServer" "NFServer.exe" "Server=MasterServer" "ID=3" "Plugin=Plugin.xml"

choice /t 2 /d y /n >nul
echo Starting NFWorldServer...
start "NFWorldServer" "NFServer.exe" "Server=WorldServer" "ID=7" "Plugin=Plugin.xml"


choice /t 5 /d y /n >nul

echo Starting NFLoginServer...
start "NFLoginServer" "NFServer.exe" "Server=LoginServer" "ID=4" "Plugin=Plugin.xml"

choice /t 5 /d y /n >nul

echo Starting NFDBServer...
start "NFDBServer" "NFServer.exe" "Server=DBServer" "ID=8" "Plugin=Plugin.xml"


choice /t 2 /d y /n >nul

echo Starting NFGameServer...
start "NFGameServer" "NFServer.exe" "Server=GameServer" "ID=6" "Plugin=Plugin.xml"

choice /t 2 /d y /n >nul


echo Starting NFProxyServer...
start "NFProxyServer" "NFServer.exe" "Server=ProxyServer" "ID=5" "Plugin=Plugin.xml"
```



## Plugin.xml

该xml所在的目录为  _Out\NFDataCfg\Debug\Plugin.xml

```xml
<XML>
	<GameServer>
		<Plugin Name="NFNetPlugin" />
		<Plugin Name="NFKernelPlugin" />
		<Plugin Name="NFConfigPlugin" />
		<Plugin Name="NFGameServerPlugin" />
		<Plugin Name="NFGameServerNet_ServerPlugin" />
		<Plugin Name="NFGameServerNet_ClientPlugin" />
		<Plugin Name="NFLogPlugin" />	
		<Plugin Name="NFLuaScriptPlugin" />
		<!--<Plugin Name="NFRenderPlugin" />-->
		<!--<Plugin Name="NFBluePrintPlugin" />-->
		<Plugin Name="NFActorPlugin" />
		<Plugin Name="NFNavigationPlugin" />

	</GameServer>

	<LoginServer>
		<Plugin Name="NFNetPlugin" />
		<Plugin Name="NFKernelPlugin" />
		<Plugin Name="NFConfigPlugin" />	
		<Plugin Name="NFLoginNet_ServerPlugin" />
		<Plugin Name="NFLoginNet_ClientPlugin" />
		<Plugin Name="NFLoginLogicPlugin" />
		<Plugin Name="NFLoginNet_HttpServerPlugin" />
		<Plugin Name="NFLogPlugin" />	
		<Plugin Name="NFDBLogicPlugin" />
		<Plugin Name="NFNoSqlPlugin" />
		<Plugin Name="NFActorPlugin" />
	</LoginServer>

	<MasterServer>
		<Plugin Name="NFNetPlugin" />
		<Plugin Name="NFKernelPlugin" />
		<Plugin Name="NFConfigPlugin" />
		<Plugin Name="NFMasterServerPlugin" />
		<Plugin Name="NFMasterNet_ServerPlugin" />
		<Plugin Name="NFMasterNet_HttpServerPlugin" />
		<Plugin Name="NFLogPlugin" />	
		<Plugin Name="NFActorPlugin" />
	</MasterServer>

	<ProxyServer>
		<Plugin Name="NFNetPlugin" />
		<Plugin Name="NFKernelPlugin" />
		<Plugin Name="NFConfigPlugin" />
		<Plugin Name="NFProxyLogicPlugin" />
		<Plugin Name="NFProxyServerNet_ClientPlugin" />
		<Plugin Name="NFProxyServerNet_ServerPlugin" />
		<Plugin Name="NFLogPlugin" />
		<Plugin Name="NFSecurityPlugin" />
		<Plugin Name="NFActorPlugin" />		
	</ProxyServer>

	<WorldServer>
		<Plugin Name="NFNetPlugin" />
		<Plugin Name="NFKernelPlugin" />
		<Plugin Name="NFConfigPlugin" />
		<Plugin Name="NFWorldNet_ClientPlugin" />
		<Plugin Name="NFWorldNet_ServerPlugin" />
		<Plugin Name="NFDBLogicPlugin" />
		<Plugin Name="NFLogPlugin" />	
		<Plugin Name="NFSecurityPlugin" />
		<Plugin Name="NFActorPlugin" />
		<Plugin Name="NFNoSqlPlugin" />
        
        <Plugin Name="NFWorldLogicPlugin" />
        <Plugin Name="NFClanPlugin" />
        <Plugin Name="NFFriendPlugin" />
        <Plugin Name="NFMailPlugin" />
        <Plugin Name="NFRankPlugin" />
        <Plugin Name="NFTeamPlugin" />
	</WorldServer>
	<DBServer>
		<Plugin Name="NFNetPlugin" />
		<Plugin Name="NFKernelPlugin" />
		<Plugin Name="NFConfigPlugin" />
		<Plugin Name="NFDBNet_ClientPlugin" />
		<Plugin Name="NFDBNet_ServerPlugin" />
		<Plugin Name="NFLogPlugin" />
		<Plugin Name="NFDBLogicPlugin" />
		<Plugin Name="NFActorPlugin" />
		<Plugin Name="NFNoSqlPlugin" />
		<Plugin Name="NFSecurityPlugin" />
	</DBServer>
</XML>
```





## 完整源码

```c++

#include "../NFPluginLoader/NFPluginServer.h"


////////////EXTERNAL PLUGIN LISTED BELOW////////////////////////////////////////////

#if NF_PLATFORM == NF_PLATFORM_WIN
#pragma comment( lib, "NFPluginLoader.lib" )

#include "Tutorial/Tutorial1/Tutorial1.h"
#include "Tutorial/Tutorial2/Tutorial2.h"
#include "Tutorial/Tutorial3/Tutorial3Plugin.h"
#include "Tutorial/Tutorial4/Tutorial4Plugin.h"
#include "Tutorial/Tutorial5/Tutorial5.h"
#include "Tutorial/Tutorial6/Tutorial6.h"
#include "Tutorial/Tutorial7/Tutorial7.h"

#pragma comment( lib, "Tutorial1.lib" )
#pragma comment( lib, "Tutorial2.lib" )
#pragma comment( lib, "Tutorial3.lib" )
#pragma comment( lib, "Tutorial4.lib" )
#pragma comment( lib, "Tutorial5.lib" )
#pragma comment( lib, "Tutorial6.lib" )
#pragma comment( lib, "Tutorial7.lib" )
#endif

void MidWareLoader(NFIPluginManager* pPluginManager)
{
#if NF_PLATFORM == NF_PLATFORM_WIN
	//TUTORIAL
	CREATE_PLUGIN(pPluginManager, Tutorial1)
	CREATE_PLUGIN(pPluginManager, Tutorial2)
	CREATE_PLUGIN(pPluginManager, Tutorial3Plugin)
	CREATE_PLUGIN(pPluginManager, Tutorial4Plugin)
	CREATE_PLUGIN(pPluginManager, Tutorial5)
	CREATE_PLUGIN(pPluginManager, Tutorial6)
	CREATE_PLUGIN(pPluginManager, Tutorial7)
#endif
}
////////////////////////////////////////////////////////

int main(int argc, char* argv[])
{
	std::cout << "__cplusplus:" << __cplusplus << std::endl;

	std::vector<NF_SHARE_PTR<NFPluginServer>> serverList;
	
    // 将 命令行的那些参数保存到string中用空格分开，注意首字符是空格
	std::string strArgvList;
	for (int i = 0; i < argc; i++)
	{
		strArgvList += " ";
		strArgvList += argv[i];
	}
	
    // 根据argc的参数判断，程序是不带参数直接启动的，所以第一个参数默认为程序的启动路径
	if (argc == 1)
	{
        //将参数保存到string中，然后将每个NFPluginServer对象保存到vector中
		//IDE
		serverList.push_back(NF_SHARE_PTR<NFPluginServer>(NF_NEW NFPluginServer(strArgvList + " Server=MasterServer ID=3 Plugin=Plugin.xml")));
        serverList.push_back(NF_SHARE_PTR<NFPluginServer>(NF_NEW NFPluginServer(strArgvList + " Server=WorldServer ID=7 Plugin=Plugin.xml")));
        serverList.push_back(NF_SHARE_PTR<NFPluginServer>(NF_NEW NFPluginServer(strArgvList + " Server=LoginServer ID=4 Plugin=Plugin.xml")));
        serverList.push_back(NF_SHARE_PTR<NFPluginServer>(NF_NEW NFPluginServer(strArgvList + " Server=DBServer ID=8 Plugin=Plugin.xml")));
        serverList.push_back(NF_SHARE_PTR<NFPluginServer>(NF_NEW NFPluginServer(strArgvList + " Server=ProxyServer ID=5 Plugin=Plugin.xml")));
		serverList.push_back(NF_SHARE_PTR<NFPluginServer>(NF_NEW NFPluginServer(strArgvList + " Server=GameServer ID=6 Plugin=Plugin.xml")));
	}
	else
	{
		serverList.push_back(NF_SHARE_PTR<NFPluginServer>(NF_NEW NFPluginServer(strArgvList)));
	}
	
    // 创建完各个server的实体之后，接下来就是执行初始的操作
	for (auto item : serverList)
	{
        //保存的是以pPluginManager* 为参数的函数对象，方便在函数里面扩展加载新的插件
		item->SeMidWareLoader(MidWareLoader);
        //这个函数就庞大了，不仅包含了各个插件对象的构建，还包括每个plugin中module的创建和注册，同时还有资源文建的加载，
		item->Init();
	}

	////////////////
	uint64_t nIndex = 0;
	while (true)
	{
		nIndex++;

		std::this_thread::sleep_for(std::chrono::milliseconds(1));
		for (auto item : serverList)
		{
			item->Execute();
		}
	}

	////////////////

	for (auto item : serverList)
	{
		item->Final();
	}

	serverList.clear();

    return 0;
}
```

