---
title: "设计模式：基于列表的发布-订阅 | Microsoft Docs"
ms.custom: ""
ms.date: "03/30/2017"
ms.prod: ".net-framework-4.6"
ms.reviewer: ""
ms.suite: ""
ms.technology: 
  - "dotnet-clr"
ms.tgt_pltfrm: ""
ms.topic: "article"
ms.assetid: f4257abc-12df-4736-a03b-0731becf0fd4
caps.latest.revision: 16
author: "Erikre"
ms.author: "erikre"
manager: "erikre"
caps.handback.revision: 16
---
# 设计模式：基于列表的发布-订阅
本示例演示作为 [!INCLUDE[indigo1](../../../../includes/indigo1-md.md)] 程序来实现的基于列表的发布\-订阅模式。  
  
> [!NOTE]
>  本主题的最后介绍了此示例的设置过程和生成说明。  
  
 Microsoft 模式和实践出版物[集成模式](http://go.microsoft.com/fwlink/?LinkId=95894)（可能为英文网页）中对基于列表的发布\-订阅设计模式进行了说明。  发布\-订阅模式可以向已经订阅某一信息主题的接收者群体传递信息。  基于列表的发布\-订阅可维护一份订户列表。  如果有要共享的信息，则会向列表上的每个订户发送一份副本。  本示例演示基于列表的动态发布\-订阅模式，在此模式下客户端可以根据需要随时订阅或取消订阅。  
  
 基于列表的发布\-订阅示例由客户端、服务和数据源程序组成。  可以有多个客户端和多个数据源程序同时运行。  客户端订阅服务、接收通知，然后取消订阅。  数据源程序向服务发送将与所有当前订户共享的信息。  
  
 在本示例中，客户端和数据源是控制台程序（.exe 文件），而服务是 Internet 信息服务 \(IIS\) 所承载的库 \(.dll\)。  客户端和数据源活动在桌面上可见。  
  
 服务使用双工通信。  `ISampleContract` 服务协定与 `ISampleClientCallback` 回调协定成对出现。  服务实现 Subscribe 和 Unsubscribe 服务操作，客户端使用这些操作加入或离开订户列表。  服务还实现 `PublishPriceChange` 服务操作，数据源程序调用该操作来向服务提供新信息。  客户端程序实现 `PriceChange` 服务操作，服务调用该操作向所有订户通知价格更改。  
  
```  
// Create a service contract and define the service operations.  
// NOTE: The service operations must be declared explicitly.  
[ServiceContract(SessionMode=SessionMode.Required,  
      CallbackContract=typeof(ISampleClientContract))]  
public interface ISampleContract  
{  
    [OperationContract(IsOneWay = false, IsInitiating=true)]  
    void Subscribe();  
    [OperationContract(IsOneWay = false, IsTerminating=true)]  
    void Unsubscribe();  
    [OperationContract(IsOneWay = true)]  
    void PublishPriceChange(string item, double price,   
                                     double change);  
}  
  
public interface ISampleClientContract  
{  
    [OperationContract(IsOneWay = true)]  
    void PriceChange(string item, double price, double change);  
}  
```  
  
 服务使用 .NET Framework 事件作为向所有订户通知新信息的机制。  如果客户端通过调用 Subscribe 加入服务，服务将提供一个事件处理程序。  如果客户端离开，服务将取消其事件处理程序对事件的订阅。  当数据源调用服务以报告价格变化时，服务将引发事件。  这将调用服务的每个实例（订阅服务的每个客户端一个实例），并导致其事件处理程序执行。  每个事件处理程序都通过其回调函数向其客户端传递信息。  
  
```  
public class PriceChangeEventArgs : EventArgs  
    {  
        public string Item;  
        public double Price;  
        public double Change;  
    }  
  
    // The Service implementation implements your service contract.  
    [ServiceBehavior(InstanceContextMode=InstanceContextMode.PerSession)]  
    public class SampleService : ISampleContract  
    {  
        public static event PriceChangeEventHandler PriceChangeEvent;  
        public delegate void PriceChangeEventHandler(object sender, PriceChangeEventArgs e);  
  
        ISampleClientContract callback = null;  
  
        PriceChangeEventHandler priceChangeHandler = null;  
  
        //Clients call this service operation to subscribe.  
        //A price change event handler is registered for this client instance.  
  
        public void Subscribe()  
        {  
            callback = OperationContext.Current.GetCallbackChannel<ISampleClientContract>();  
            priceChangeHandler = new PriceChangeEventHandler(PriceChangeHandler);  
            PriceChangeEvent += priceChangeHandler;  
        }  
  
        //Clients call this service operation to unsubscribe.  
        //The previous price change event handler is deregistered.  
  
        public void Unsubscribe()  
        {  
            PriceChangeEvent -= priceChangeHandler;  
        }  
  
        //Information source clients call this service operation to report a price change.  
        //A price change event is raised. The price change event handlers for each subscriber will execute.  
  
        public void PublishPriceChange(string item, double price, double change)  
        {  
            PriceChangeEventArgs e = new PriceChangeEventArgs();  
            e.Item = item;  
            e.Price = price;  
            e.Change = change;  
            PriceChangeEvent(this, e);  
        }  
  
        //This event handler runs when a PriceChange event is raised.  
        //The client's PriceChange service operation is invoked to provide notification about the price change.  
  
        public void PriceChangeHandler(object sender, PriceChangeEventArgs e)  
        {  
            callback.PriceChange(e.Item, e.Price, e.Change);  
        }  
  
    }  
  
```  
  
 运行示例时，将启动多个客户端。  这些客户端都将订阅服务。  然后运行数据源程序，它将信息发送到服务。  服务将信息传递给所有订户。  您可以在每个客户端控制台上查看活动并确认已收到信息。  在客户端窗口中按 Enter 可以关闭客户端。  
  
### 设置和生成示例  
  
1.  确保已经执行了[Windows Communication Foundation 示例的一次性安装过程](../../../../docs/framework/wcf/samples/one-time-setup-procedure-for-the-wcf-samples.md)。  
  
2.  若要生成 C\# 或 Visual Basic .NET 版本的解决方案，请按照[生成 Windows Communication Foundation 示例](../../../../docs/framework/wcf/samples/building-the-samples.md)中的说明进行操作。  
  
### 在同一计算机上运行示例  
  
1.  使用浏览器通过输入以下地址来测试是否可以访问该服务：http:\/\/localhost\/servicemodelsamples\/service.svc。  在响应中应显示确认页。  
  
2.  从 \\client\\bin\\（在语言特定文件夹内）中运行 Client.exe。  客户端活动将显示在客户端控制台窗口上。  启动多个客户端。  
  
3.  从 \\datasource\\bin\\（在语言特定文件夹内）中运行 Datasource.exe。  数据源活动将显示在控制台窗口中。  数据源向服务发送信息后，信息应传递到每个客户端。  
  
4.  如果客户端、数据源和服务程序无法进行通信，请参见[Troubleshooting Tips](http://msdn.microsoft.com/zh-cn/8787c877-5e96-42da-8214-fa737a38f10b)。  
  
### 跨计算机运行示例  
  
1.  安装服务计算机：  
  
    1.  在服务计算机上创建一个名为 ServiceModelSamples 的虚拟目录。  [Windows Communication Foundation 示例的一次性安装过程](../../../../docs/framework/wcf/samples/one-time-setup-procedure-for-the-wcf-samples.md)中的批处理文件 Setupvroot.bat 可用于创建磁盘目录和虚拟目录。  
  
    2.  从 %SystemDrive%\\Inetpub\\wwwroot\\servicemodelsamples 中将服务程序文件复制到服务计算机上的 ServiceModelSamples 虚拟目录中。  确保在 \\bin 目录中包括这些文件。  
  
    3.  测试是否可使用浏览器从客户端计算机访问服务。  
  
2.  安装客户端计算机：  
  
    1.  将 \\client\\bin\\ 文件夹（在语言特定文件夹内）中的客户端程序文件复制到客户端计算机上。  
  
    2.  在每个客户端配置文件中，更改终结点定义的地址值以与服务的新地址相匹配。  在地址中将任何对“localhost”的引用替换为一个完全限定的域名。  
  
3.  安装数据源计算机：  
  
    1.  将 \\datasource\\bin\\ 文件夹（在语言特定文件夹内）中的数据源程序文件复制到数据源计算机上。  
  
    2.  在数据源配置文件中，更改终结点定义的地址值以与服务的新地址相匹配。  在地址中将任何对“localhost”的引用替换为一个完全限定的域名。  
  
4.  在客户端计算机上的命令提示符下启动 Client.exe。  
  
5.  在数据源计算机上的命令提示符下启动 Datasource.exe。  
  
> [!IMPORTANT]
>  您的计算机上可能已安装这些示例。  在继续操作之前，请先检查以下（默认）目录：  
>   
>  `<安装驱动器>:\WF_WCF_Samples`  
>   
>  如果此目录不存在，请访问[针对 .NET Framework 4 的 Windows Communication Foundation \(WCF\) 和 Windows Workflow Foundation \(WF\) 示例](http://go.microsoft.com/fwlink/?LinkId=150780)（可能为英文网页），下载所有 [!INCLUDE[indigo1](../../../../includes/indigo1-md.md)] 和 [!INCLUDE[wf1](../../../../includes/wf1-md.md)] 示例。  此示例位于以下目录：  
>   
>  `<安装驱动器>:\WF_WCF_Samples\WCF\Scenario\DesignPatterns/ListBasedPublishSubscribe`  
  
## 请参阅