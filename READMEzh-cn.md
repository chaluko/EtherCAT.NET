# EtherCAT.NET

[![AppVeyor](https://ci.appveyor.com/api/projects/status/github/apollo3zehn/ethercat.net?svg=true&branch=master)](https://ci.appveyor.com/project/Apollo3zehn/ethercat-net)
[![NuGet](https://img.shields.io/nuget/vpre/ethercat.net.svg)](https://www.nuget.org/packages/EtherCAT.NET)

EtherCAT。NET提供了底层原生*简单开源EtherCAT主站*（[SOEM]）的高级抽象(https://github.com/OpenEtherCATsociety/soem)).为了实现这一点，该解决方案包含另一个项目：SOEM。PInvoke。它包括Windows和Linux的实际本机库，并允许简单地将P/Invoke转换为本机SOEM方法。其目的是提供一种访问本地SOEM主机的管理方式。EtherCAT。NET依赖于SOEM。PInvoke并添加用于高级抽象的类。



在其当前状态下，许多但并非所有计划中的功能都已实现。因此，只有alpha版本可用（[NuGet](https://www.nuget.org/packages/EtherCAT.NET))到目前为止。这主要意味着可以配置和启动任何EtherCAT网络，但尚未实现SDO的简单配置等高级功能。



此主机已经支持通过ESI文件进行从机配置。事实上，这些文件是主设备工作所必需的。如[示例]（示例/SampleMaster）所示，您需要将配置指向ESI文件所在的文件夹。



另一个特点是复杂从属设备的配置。例如，该主机已成功通过Profibus终端（“EL6731-0010”）进行了测试。TwinCAT允许通过特殊的配置页面配置此终端。由于为每个复杂的从属设备创建高级配置接口是一项艰巨的工作，因此EtherCAT的优先级很高。NET提供了一个简单的界面来定制SDO（如TwinCAT），这样最终用户就可以以通用的方式为任何从属设备调整所需的设置。

## 运行应用程序

> \>**Linux**：以root权限运行应用程序，如[此处]所示(https://github.com/OpenEtherCATsociety/SOEM/blob/8832ef0f2fa09fb0b06acaca239b51263ca1878c/doc/soem.dox#L20).
>
> \>**Windows**：安装[WWinPcap](https://www.winpcap.org/).

## 如何使用 EtherCAT.NET

如果你从[示例]开始(https://github.com/Apollo3zehn/EtherCAT.NET/tree/master/sample/SampleMaster)，请确保将“Program.cs”中的接口名称与网络适配器的名称相匹配，并用所需的ESI文件填充ESI目录。运行示例应用程序时，输出将类似于以下内容：

![sample-master](https://user-images.githubusercontent.com/20972129/57144734-01a22f80-6dc2-11e9-9b8a-f32d5a8d7b2f.png)

### 生成从属列表

主设备可以在没有从属设备列表的情况下进行操作。在这种情况下，它会在启动过程中扫描可用的从属设备。但缺点是无法提前配置从属设备，也没有可用的变量引用。因此，有两种方法可以生成从列表，如下所示：:

1. 手动创建列表（它必须与实际连接的从属设备匹配）
    ```
    --> 还不可能
    ```
2. 扫描已连接的从属设备列表
    ```cs
    var rootSlave = EcUtilities.ScanDevices(<network interface name>);
    ```

归还的“rootSlave”是主人本身，主人在其“子女”/“后代”财产中拥有儿童奴隶）
之后，找到的奴隶应该填充ESI信息：

```cs
rootSlave.Descendants().ToList().ForEach(slave =>
{
    ExtensibilityHelper.CreateDynamicData(settings.EsiDirectoryPath, slave);
});
    
```

### Accessing the slaves

该主站与TwinCAT的工作方式不同，因为从站使用EEPROM中的*配置从站别名*（CSA）字段进行标识（见[硬件数据表第二节]第2.3.1节(https://download.beckhoff.com/download/Document/io/ethercat-development-products/ethercat_esc_datasheet_sec2_registers_2i7.pdf)).每当主节点发现一个CSA=0的从节点时，它都会分配一个新的随机数。该编号可以在第一次运行后通过打印每个从属设备的CSA来获取：

```cs
var message = new StringBuilder();
var slaves = rootSlave.Descendants().ToList();

message.AppendLine($"Found {slaves.Count()} slaves:");

slaves.ForEach(current =>
{
    message.AppendLine($"{current.DynamicData.Name} (PDOs: {current.DynamicData.Pdos.Count} - CSA: { current.Csa })");
});

logger.LogInformation(message.ToString().TrimEnd());
```

现在，如果硬件从属顺序发生了变化，则可以通过以下方式识别各个从属：:

```cs
var slaves = rootSlave.Descendants().ToList();
var EL1008 = slaves.FirstOrDefault(current => current.Csa == 3);
```

当然，只要硬件设置不变，您总是可以通过简单的索引获得对从属设备的引用：

```cs
var EL1008 = slaves[1];
```

### 访问过程数据对象（PDO）和变量

当您引用从属设备时，可以通过“DynamicData”属性访问PDO：

```cs
var pdos = slaves[0].DynamicData.Pdos;
var channel0 = pdo[0];
```

由于PDO是一组变量，因此可以在PDO下方找到这些变量：

```cs
var variables = pdo.Variables;
var variable0 = variables[0];
```

变量保存对RAM中某个地址的引用。该地址位于属性“variable0.Datatr”中。在运行时（主设备配置后），此地址设置为实际RAM地址。因此，可以使用“unsafe”关键字来操纵数据。这里我们有一个布尔变量，它是EtherCAT中的一个位，可以[切换](https://stackoverflow.com/questions/47981/how-do-you-set-clear-and-toggle-a-single-bit)使用以下代码：

```cs
unsafe
{
    var myVariableSpan = new Span<int>(variable0.DataPtr.ToPointer(), 1);
    myVariableSpan[0] ^= 1UL << variable0.BitOffset;
}
```

使用原始指针时要小心，这样就不会修改数组边界之外的数据。

### Running the master


首先，必须创建一个“EcSettings”对象。构造函数接受参数“cycleFrequency”、“esiDirectoryPath”和“interfaceName”。第一个指定了主时钟的循环时间，对于分布式时钟配置很重要。最后一个“interfaceName”是您的网络适配器的名称。



`esiDirectoryPath`参数包含一个包含ESI文件的文件夹的路径。对于Beckhoff奴隶，可以在[此处]下载(https://www.beckhoff.de/default.asp?download/elconfg.htm).

第一次启动可能需要一段时间，因为ESI缓存是为了加速后续启动而构建的。每当添加新的未知从属服务器时，都会重建此缓存。



使用“EcSettings”对象和其他一些类型（如“ILogger”，请参阅示例），可以使用以下命令对master进行操作：

```cs
using (var master = new EcMaster(rootSlave, settings, extensionFactory, logger))
{
    master.Configure();

    while (true)
    {
        /* Here you can update your inputs and outputs. */
        master.UpdateIO(DateTime.UtcNow);
        /* Here you should let your master pause for a while, e.g. using Thread.Sleep or by simple spinning. */
    }
}

```

如果你需要一个更复杂的定时器实现，看看[这个](https://github.com/OneDAS-Group/OneDAS-Core/blob/master/src/OneDas.Core/Engine/RtTimer.cs).它可以按如下方式使用：

```cs
var interval = TimeSpan.FromMilliseconds(100);
var timeShift = TimeSpan.Zero;
var timer = new RtTimer();

using (var master = new EcMaster(settings, extensionFactory, logger))
{
    master.Configure(rootSlave);
    timer.Start(interval, timeShift, UpdateIO);

    void UpdateIO()
    {
        /* Here you can update your inputs and outputs. */
        master.UpdateIO(DateTime.UtcNow);
    }

    Console.ReadKey();
    timer.Stop();
}

```

## 编译应用程序

所有平台都使用单个Powershell*Core*脚本来初始化解决方案。这简化了CI构建，但要求Powershell Core在目标系统上可用。如果您不想安装它，可以提取脚本中的信息并手动执行步骤。

### Windows

You need the following tools:

* [.NET Core 3.1](https://dotnet.microsoft.com/download/dotnet-core/3.1)
* [PowerShell Core](https://docs.microsoft.com/en-us/powershell/scripting/install/installing-powershell-core-on-windows?view=powershell-6)
* [Visual Studio 2026](https://visualstudio.microsoft.com
* [CMake](https://cmake.org)

然后，可以按如下方式构建解决方案：

1. 在根文件夹中执行一次powershell脚本。如果你不想安装Powershell Core，你可以调整脚本，将“$IsWindows”替换为“$true”。
    ```
    ./init_solution.ps1
    ```
2. 每次对本机代码进行更改时，运行“msbuild”*：
    ```
    msbuild ./artifacts/bin32/SOEM_wrapper/soem_wrapper.vcxproj /p:Configuration=Release
    msbuild ./artifacts/bin64/SOEM_wrapper/soem_wrapper.vcxproj /p:Configuration=Release
    ```
    （*如果“PATH”变量中没有“msbuild”，请使用Visual Studio开发人员命令提示符，或直接使用Visual Studio编译）。
3. 每次对托管代码进行更改时，都运行``dotbuild``：
    ```
    dotnet build ./src/EtherCAT.NET/EtherCAT.NET.csproj
    ```
4. 在“”中查找结果包。/工件/包/*“”。
