书籍：《深入解析Mac OS X & iOS操作系统》

# OS X 和 iOS的架构

##### OS X 和 iOS的层次结构：

* Cocoa Touch Layer用户体验层：Aqua、Dashboard、Spotlight 和辅助功能\(Accessibility\)，iOS SpringBoard 同时支持Spotllight；
* 应用框架层：Cocoa、Carbon 和 Java，iOS 中只有 Cocoa（Cocoa Touch）；
* 核心框架层：称为图形和媒体层。包括核心框架、Open GL 和 QuickTime；
* Darwin\(操作系统内核\)：操作系统核心，包括内核 和 UNIX shell 环境；

![](/assets/iOS 操作系统及内核结构层次.png)

### iOS 操作系统层次架构:

* ###### Cocoa Touch Layer \(触摸UI层\)；
* ###### MediaLayer（媒体层）；
* ###### Core Services Layer（核心服务层）；
* ###### Core OS Layer（核心OS层）；

##### 核心操作系统层（Core OS Layer）

```
   Core OS 是用 FreeBSD 和 Mach 所写的 Darwin，是开源且符合 POSIX 标注的一个 Unix 核心。这一层提供了整个 iOS 系统的一些基础功能 ，如：硬件驱动、内存管理、程序管理、线程管理（POSIX）、文件系统、网络（BSD Socket）以及标注的输入输出等，这些功能通过 C语言 的 API 来提供。

   核心OS层的驱动也提供了硬件和系统框架之间的接口。然而，由于安全的考虑，只有有限的系统框架类能访问内核和驱动。可以调用私有 API ;
```

###### Core OS 包含的框架：

* Accelerate 加速框架，Accelerate框架 \(Accelerate.framework\)包含执行数字信号处理、线性代数、图像处理计算的接口。使用该框架的优点是它们针对所有的 iOS 设备上存在的硬件配置做了优化，因此你能写一次代码确保在所有设备上有效运行。

* Core Bluetooth Framework（核心蓝牙框架）， CoreBluetooth 框架 \(CoreBluetooth.framework\)允许开发者与蓝牙低耗电外设（LE）交互。使用该框架的 [Objective-C](http://lib.csdn.net/base/objective-c) 接口能够完成 蓝牙通信；
* External Accessory Framework（外部附件框架），ExternalAccessory 框架\(ExternalAccessory.framework\)提供与连接到 iOS设备的硬件附件通讯的支持。附件能通过30-pin连接器或使用蓝牙无线与 iOS 设备进行连接。该框架给你提供了获得关于每一个可获得的附件信息和启动通讯会话的方式。然后，你可自由的使用附件支持的命令直接操作附件。

* Generic Security Services Framework（通用安全服务框架），GenericSecurity Services 框架 \(GSS.framework\)给 iOS 应用提供一组标准安全相关的服务。该框架的基本接口规定在IETF[RFC2743](http://www.ietf.org/rfc/rfc2743.txt) and [RFC4401](http://tools.ietf.org/html/rfc4401)。除了提供标准的接口，iOS 还包括一些没有在标准中规定但被许多应用需要的一些管理证书需要的额外东西。

* Security Framework（安全框架），除了内建的安全功能，iOS 也提供了一个明确的安全框架（Security.framework\)，能用它来保证应用管理的数据的安全。该框架提供管理证书、公有和私有 key 和信任策略的接口。支持产生加密安全伪随机码。它也支持在 keychain（保存敏感用户数据的安全仓库）中保存证书和加密 key。公共加密库提供对称加密、hash认证编码（HMACs）、数字签名等额外支持，数字签名功能本质上与 iOS 上没有的 OpenSSL 库兼容。在你创建的多个应用之间共享keychain是可能的。共享使它容易在相同的一套应用之间更平滑的协作。例如，你能使用该功能来共享用户口令或其它元素，否则可能使每个应用都需要提示用户。为了在应用之间共享数据，必须为每个应用的Xcode工程配置适当的权限。

* System，System级包含 kernel 环境、驱动以及操作系统级别的 unix 接口。kernel本身负责操作系统的每一个方面：如虚拟内存管理、线程、文件系统、网络和互联通信。在该层的驱动也提供在可获得的硬件与系统框架之间的接口。为了安全，对kernel和驱动的存取被限制到一组有限的系统框架和应用。

  

                     





















































