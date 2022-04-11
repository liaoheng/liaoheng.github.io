---
title: 传统蓝牙协议栈初探
date: 
tags:
---

# 概述

此文主要是简单的探索一下蓝牙电话和蓝牙音乐相关协议栈，所以很多细节并未涉及。

一般而言，我们把某个协议的实现代码称为协议栈(protocol stack)，要理解协议栈需要先了解蓝牙的核心系统架构，如下图：

![](https://s1.ax1x.com/2022/04/11/LV6Ybd.png)

也可简单的**抽象为3层**：![](https://s1.ax1x.com/2022/04/11/LV6JDH.png)

- **User Application(Host)**：User Application即应用层，也被称为Host，我们调用Bluetooth API就属于应用层，例如，[BluetoothAdapter](https://developer.android.com/reference/android/bluetooth/BluetoothAdapter.html)中提供的接口。
- **HCI (Host controller Interface)**：上层在调用蓝牙API时，**不会直接操作蓝牙底层**(Controller)相关接口，而是**通过HCI下发对应操作的Command给Controller**，然后底层执行命令后返回执行结果，**即Controller发送Event给HCI，HCI再通知给应用层**，HCI起到了一个**中间层**的作用。
- **Controller**：Controller是在最底层，硬件驱动。

按照上面的定义，HCI与Host层中的协议实现就是**蓝牙协议栈**。

## HCI

>  BLUETOOTH SPECIFICATION Version 4.2 [Vol 2, Part E] page 400  
>
>  HCI（HOST CONTROLLER INTERFACE）：主机控制层接口，主要负责透过transport把协议栈的数据发送给蓝牙芯片，并且接受来自蓝牙芯片的数据，数据主要分为HCI COMMAND(HOST->CONTROLLER),HCI EVENT(HOST<-CONTROLLER),HCI ACL(HOST<->CONTROLLER),HCI SCO(这个有点些微差异，因为部分芯片的SCO数据不是透过TRANSPORT直接跟HOST沟通，而是通过特殊的引脚，PCM IN/OUT/SYNC/CLK脚来传输数据)

HOST层的几乎所有数据最后都会通过HCI，所以我们通过HCI的日志就可以了解蓝牙协议栈中每个协议，手机中可以通过开发者模式打开HCI日志开关，推荐使用**wireshark**或者**Frontline ComProbe Protocol Analysis System**来分析HCI日志，下面给出来的的日志都是**wireshark**的结果。

HCI日志由各种数据包组成，我们需要了解其中的两种数据包如下：

### Command 数据包

HCI Command数据包格式如下，开头的**Opcode是区分不同类型的命令的唯一标识**，Opcode由**OpCode Group Field (OGF)** 和 **OpCode Command Field (OCF)**组成。根据OGF的值，可以将HCI commands进行分类。OpCode 的计算公式为：`OpCode = OGF << 6 + OCF` **。有了OpCode计算的方式，我们就可以**通过OpCode过滤HCI log里面的指定类型的HCI Command。

![](https://s1.ax1x.com/2022/04/11/LV608f.png)

一次HCI Command指令发送完成，通常两条数据包，一条是host给controller的指令，一条是controller给host的指令反馈，可以确定指令发送是成功的。

### Event 数据包

HCI Event 数据包由controller发送给Host，其中可以是蓝牙设备的消息也可以是连接的蓝牙设备的消息。

![](https://s1.ax1x.com/2022/04/11/LV6B28.png)

# 设备连接

## 蓝牙设备状态

> BLUETOOTH SPECIFICATION Version 4.2 [Vol 2, Part B] page 159  

扫描设备然后与之建立连接是蓝牙工作流程的第一步，所以先要了解一下蓝牙的状态如下图：

![](https://s1.ax1x.com/2022/04/11/LV6wPP.png)

从<u>蓝牙状态转换图</u>中可以看出 STANDBY状体是蓝牙设备的默认状态。

**Page**：这个子状态就是我们通常称为的连接(寻呼)，进行连接/激活对应的slave的操作我们就称为page。

**Page scan**：这个子状态是和page对应的，它就是等待被page的slave所处的状态，换句话说，若想被page到，我们就要处于page scan的状态。

**inquiry**：这就是我们通常所说的扫描状态，这个状态的设备就是去扫描周围的设备。

**inquiry scan**：这就是我们通常看到的可被发现的设备。体现在上层就是我们在android系统中点击设备可被周围什么发现，那设备就处于这样的状态。

## 逻辑链路传输协议

### ACL

> BLUETOOTH SPECIFICATION Version 4.2 [Vol 1, Part A]  page  63  
>
> The asynchronous connection-oriented (ACL) logical transport is used to carry
> LMP and L2CAP control signaling and best effort asynchronous user data. The
> ACL logical transport uses a 1-bit ARQN/SEQN scheme to provide simple
> channel reliability. Every active slave device within a piconet has one ACL
> logical transport to the piconet master, known as the default ACL.  

ACL全称**BR/EDR Asynchronous Connection-oriented**，定向连接，它既支持对称连接，也支持不对称连接（既可以一对一，也可以一对多）。主设备负责控制链路带宽，并决定微微网中的每个从设备可以占用多少带宽和连接的对称性。从设备只有被选中时才能传送数据。ACL链路也支持接收主设备发给微微网中所有从设备的广播消息。

建立ACL需要设备HCI之间进行的下面的操作，这个也与**蓝牙状态**息息相关。

#### 开始扫描

- **Host**:

  ```java
  LocalBluetoothManager.getInstance().getBluetoothAdapter().startScanning(true);
  ```

- **Controller**:

  ```dtd
  #发送扫描指令
  > 103	4.351128	host	controller	HCI_CMD	9	Sent Inquiry
  ---------------------------------------------------------------------------
  Bluetooth HCI Command - Inquiry
      Command Opcode: Inquiry (0x0401)
          0000 01.. .... .... = Opcode Group Field: Link Control Commands (0x01)
          .... ..00 0000 0001 = Opcode Command Field: Inquiry (0x001)
      Parameter Total Length: 5
      LAP: 0x9e8b33
      Inquiry Length: 10 (12.8 sec)
      Num Responses: 0
      [Pending in frame: 104]
      [Command-Pending Delta: 0.846ms]
      [Response in frame: 429]
      [Command-Response Delta: 12811.07ms]
  
  #HCI反馈扫描指令状态
  > 104	4.351974	controller	host	HCI_EVT	7	Rcvd Command Status (Inquiry)
  ---------------------------------------------------------------------------
  Bluetooth HCI Event - Command Status
      Event Code: Command Status (0x0f)
      Parameter Total Length: 4
      Status: Pending (0x00)
      Number of Allowed Command Packets: 1
      Command Opcode: Inquiry (0x0401)
      [Command in frame: 103]
      [Response in frame: 429]
      [Command-Pending Delta: 0.846ms]
      [Pending-Response Delta: 12810.224ms]
  
  #HCI发送的扫描到的设备信息
  > 114	4.505582	controller	host	HCI_EVT	258	Rcvd Extended Inquiry Result
  ---------------------------------------------------------------------------
  Bluetooth HCI Event - Extended Inquiry Result
      Event Code: Extended Inquiry Result (0x2f)
      Parameter Total Length: 255
      Number of responses: 1
      BD_ADDR: XiaomiCo_77:c2:35 (00:00:00:00:00:00)
      Page Scan Repetition Mode: R1 (0x01)
      Reserved: 0x02
      Class of Device: 0x5a020c (Phone:Smartphone - services: Networking Capturing ObjectTransfer Telephony)
      .000 0000 0011 0100 = Clock Offset: 0x0034
      RSSI: -88 dBm
      Extended Inquiry Response Data
          Device Name: +My Phone1234567890123456789012
          16-bit Service Class UUIDs
          32-bit Service Class UUIDs
          128-bit Service Class UUIDs
          Unused
  
  ```

#### 停止扫描

- **Host**:

    ```java
    LocalBluetoothManager.getInstance().getBluetoothAdapter().stopScanning();
    ```

- **Controller**:

  ```dtd
  > 85	2.202129	host	controller	HCI_CMD	4	Sent Inquiry Cancel
  ---------------------------------------------------------------------------
  Bluetooth HCI Command - Inquiry Cancel
      Command Opcode: Inquiry Cancel (0x0402)
          0000 01.. .... .... = Opcode Group Field: Link Control Commands (0x01)
          .... ..00 0000 0010 = Opcode Command Field: Inquiry Cancel (0x002)
      Parameter Total Length: 0
      [Response in frame: 86]
      [Command-Response Delta: 2.051ms]
  
  > 86	2.204180	controller	host	HCI_EVT	7	Rcvd Command Complete (Inquiry Cancel)
  ---------------------------------------------------------------------------
  Bluetooth HCI Event - Command Complete
      Event Code: Command Complete (0x0e)
      Parameter Total Length: 4
      Number of Allowed Command Packets: 1
      Command Opcode: Inquiry Cancel (0x0402)
      Status: Success (0x00)
      [Command in frame: 85]
      [Command-Response Delta: 2.051ms]
  ```

#### 完成扫描

- **Host**:

  ```java
  LocalBluetoothManager.getInstance().getEventManager().registerCallback(new BluetoothCallback() {
     		@Override
  		public void onScanningStateChanged(boolean started) {
  		} 
  });
  ```

- **Controller**:

  ```dtd
  > 429	17.162198	controller	host	HCI_EVT	4	Rcvd Inquiry Complete
  ---------------------------------------------------------------------------
  Bluetooth HCI Event - Inquiry Complete
      Event Code: Inquiry Complete (0x01)
      Parameter Total Length: 1
      Status: Success (0x00)
      [Command in frame: 103]
      [Pending in frame: 104]
      [Pending-Response Delta: 12810.224ms]
      [Command-Response Delta: 12811.07ms]
  ```

#### 设备配对

![](https://s1.ax1x.com/2022/04/11/LV6GKe.png)

##### 配对

- **Host**:

  ```java
  CachedBluetoothDevice.startPairing();
  ```
  
- **Controller**:

  ```dtd
  #删除需要连接设备本地记忆的Link Key
  > 685	33.812149	host	controller	HCI_CMD	11	Sent Delete Stored Link Key
  ---------------------------------------------------------------------------
  Bluetooth HCI Command - Delete Stored Link Key
      Command Opcode: Delete Stored Link Key (0x0c12)
          0000 11.. .... .... = Opcode Group Field: Host Controller & Baseband Commands (0x03)
          .... ..00 0001 0010 = Opcode Command Field: Delete Stored Link Key (0x012)
      Parameter Total Length: 7
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
      Delete All Flag: Delete only Link Key for specified BD_ADDR (0x00)
      [Response in frame: 686]
      [Command-Response Delta: 2.147ms]
  
  #建立ACL连接
  > 687	33.814531	host	controller	HCI_CMD	17	Sent Create Connection
  ---------------------------------------------------------------------------
  Bluetooth HCI Command - Create Connection
      Command Opcode: Create Connection (0x0405)
      Parameter Total Length: 13
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
      Packet Type: 0xcc18, DH5, DM5, DH3, DM3, DH1, DM1
      Page Scan Repetition Mode: R1 (0x01)
      Page Scan Mode: Mandatory Page Scan Mode (0x00)
      .000 0010 0000 1000 = Clock Offset: 0x0208 (650 msec)
      1... .... .... .... = Clock_Offset_Valid_Flag: true (1)
      Allow Role Switch: Local device may be master, or may become slave after accepting a master slave switch. (0x01)
      [Pending in frame: 688]
      [Command-Pending Delta: 0.965ms]
      [Response in frame: 690]
      [Command-Response Delta: 1118.609ms]
  
  #建立ACL连接成功
  > 690	34.933140	controller	host	HCI_EVT	14	Rcvd Connect Complete
  ---------------------------------------------------------------------------
  Bluetooth HCI Event - Connect Complete
      Event Code: Connect Complete (0x03)
      Parameter Total Length: 11
      Status: Success (0x00)
      Connection Handle: 0x0033
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
      Link Type: ACL connection (Data Channels) (0x01)
      Encryption Mode: Encryption Disabled (0x00)
      [Command in frame: 687]
      [Pending in frame: 688]
      [Pending-Response Delta: 1117.644ms]
      [Command-Response Delta: 1118.609ms]
  
  #改变当前设备的角色，这里当前设备为从(Slave)，连接设备为主(Master)
  > 689	34.929964	controller	host	HCI_EVT	11	Rcvd Role Change
  ---------------------------------------------------------------------------
  Bluetooth HCI Event - Role Change
      Event Code: Role Change (0x12)
      Parameter Total Length: 8
      Status: Success (0x00)
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
      Role: Currently the Slave for specified BD_ADDR (0x01)
  
  #与设备建立连接之后，准备开始验证
  > 697	34.935615	host	controller	HCI_CMD	6	Sent Authentication Requested
  ---------------------------------------------------------------------------
  Bluetooth HCI Command - Authentication Requested
      Command Opcode: Authentication Requested (0x0411)
      Parameter Total Length: 2
      Connection Handle: 0x0033
      [Pending in frame: 699]
      [Command-Pending Delta: 0.917ms]
      [Response in frame: 743]
      [Command-Response Delta: 4291.694ms]
  
  #连接设备询问Link Key
  > 701	34.936976	controller	host	HCI_EVT	9	Rcvd Link Key Request
  ---------------------------------------------------------------------------
  Bluetooth HCI Event - Link Key Request
      Event Code: Link Key Request (0x17)
      Parameter Total Length: 6
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
  
  #当前设备无Link Key，也就是未于连接设备配对过
  > 705	34.938603	host	controller	HCI_CMD	10	Sent Link Key Request Negative Reply
  ---------------------------------------------------------------------------
  Bluetooth HCI Command - Link Key Request Negative Reply
      Command Opcode: Link Key Request Negative Reply (0x040c)
          0000 01.. .... .... = Opcode Group Field: Link Control Commands (0x01)
          .... ..00 0000 1100 = Opcode Command Field: Link Key Request Negative Reply (0x00c)
      Parameter Total Length: 6
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
      [Response in frame: 706]
      [Command-Response Delta: 0.87ms]
  
  #使用的配对方式是`简单安全配对`SSP(Simple Secure Pairng), 就是直接弹配对码
  
  #连接设备询问当前设备的IO能力
  > 707	34.939729	controller	host	HCI_EVT	9	Rcvd IO Capability Request
  ---------------------------------------------------------------------------
  Bluetooth HCI Event - IO Capability Request
      Event Code: IO Capability Request (0x31)
      Parameter Total Length: 6
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
  
  #回复当前设备的IO能力
  >708	34.939897	host	controller	HCI_CMD	13	Sent IO Capability Request Reply
  ---------------------------------------------------------------------------
  Bluetooth HCI Command - IO Capability Request Reply
      Command Opcode: IO Capability Request Reply (0x042b)
          0000 01.. .... .... = Opcode Group Field: Link Control Commands (0x01)
          .... ..00 0010 1011 = Opcode Command Field: IO Capability Request Reply (0x02b)
      Parameter Total Length: 9
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
      IO Capability: Display Yes/No (1)
      OOB Data Present: OOB Authentication Data Not Present (0)
      Authentication Requirements: MITM Protection Required - Dedicated Bonding. Use IO Capability To Determine Procedure, No Secure Connection (3)
      [Response in frame: 709]
      [Command-Response Delta: 0.811ms]
  
  ######其它参数同步
  
  #连接设备的IO能力，io能力参数为Display Yes/No (0x01)，需要用户确认
  > 737	36.532369	controller	host	HCI_EVT	12	Rcvd IO Capability Response
  ---------------------------------------------------------------------------
  Bluetooth HCI Event - IO Capability Response
      Event Code: IO Capability Response (0x32)
      Parameter Total Length: 9
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
      IO Capability: Display Yes/No (0x01)
      OOB Data Present: OOB Authentication Data Not Present (0)
      Authentication Requirements: MITM Protection Required - Dedicated Bonding. Use IO Capability To Determine Procedure, No Secure Connection (3)
  
  #当前设备接收到配对码信息，Numeric Value就是显示的配对码
  > 738	37.066181	controller	host	HCI_EVT	13	Rcvd User Confirmation Request
  ---------------------------------------------------------------------------
  Bluetooth HCI Event - User Confirmation Request
      Event Code: User Confirmation Request (0x33)
      Parameter Total Length: 10
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
      Numeric Value: 446514
  
  #回复连接设备已接收到配对码信息
  > 739	37.096306	host	controller	HCI_CMD	10	Sent User Confirmation Request Reply
  ---------------------------------------------------------------------------
  Bluetooth HCI Command - User Confirmation Request Reply
      Command Opcode: User Confirmation Request Reply (0x042c)
          0000 01.. .... .... = Opcode Group Field: Link Control Commands (0x01)
          .... ..00 0010 1100 = Opcode Command Field: User Confirmation Request Reply (0x02c)
      Parameter Total Length: 6
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
      [Response in frame: 740]
      [Command-Response Delta: 1.46ms]
  
  #连接设备SSP验证成功
  > 741	39.215747	controller	host	HCI_EVT	10	Rcvd Simple Pairing Complete
  ---------------------------------------------------------------------------
  Bluetooth HCI Event - Simple Pairing Complete
      Event Code: Simple Pairing Complete (0x36)
      Parameter Total Length: 7
      Status: Success (0x00)
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
  
  #连接设备的Link Key，需要保存，用于下次连接
  > 742	39.226970	controller	host	HCI_EVT	26	Rcvd Link Key Notification
  ---------------------------------------------------------------------------
  Bluetooth HCI Event - Link Key Notification
      Event Code: Link Key Notification (0x18)
      Parameter Total Length: 23
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
      Link Key: bc31fb070e91eec2f2296fbf9bc8b518
      Key Type: Unknown (0x08)
  
  #验证完成
  > 743	39.227309	controller	host	HCI_EVT	6	Rcvd Authentication Complete
  ---------------------------------------------------------------------------
  Bluetooth HCI Event - Authentication Complete
      Event Code: Authentication Complete (0x06)
      Parameter Total Length: 3
      Status: Success (0x00)
      Connection Handle: 0x0033
      [Command in frame: 697]
      [Pending in frame: 699]
      [Pending-Response Delta: 4290.777ms]
      [Command-Response Delta: 4291.694ms]
  ```

##### 被配对

- **Host**:

  ```java
  LocalBluetoothManager.getInstance().getEventManager().registerCallback(new BluetoothCallback() {
     		@Override
  		public void onDeviceBondStateChanged(CachedBluetoothDevice cachedDevice, int bondState) {
  		}
  });
  ```

- **Controller**:

  ```dtd
  #请求ACL连接
  > 326	21.946869	controller	host	HCI_EVT	13	Rcvd Connect Request
  ---------------------------------------------------------------------------
  Bluetooth HCI Event - Connect Request
      Event Code: Connect Request (0x04)
      Parameter Total Length: 10
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
      Class of Device: 0x5a020c (Phone:Smartphone - services: Networking Capturing ObjectTransfer Telephony)
      Link Type: ACL connection (Data Channels) (0x01)
  
  #允许ACL连接
  > 327	21.947178	host	controller	HCI_CMD	11	Sent Accept Connection Request
  ---------------------------------------------------------------------------
  Bluetooth HCI Command - Accept Connection Request
      Command Opcode: Accept Connection Request (0x0409)
      Parameter Total Length: 7
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
      Role: Remain the Slave for this connection. The LM will NOT perform the role switch. (0x01)
      [Pending in frame: 328]
      [Command-Pending Delta: 0.685ms]
      [Response in frame: 329]
      [Command-Response Delta: 4.432ms]
  
  #ACL连接成功
  > 329	21.951610	controller	host	HCI_EVT	14	Rcvd Connect Complete
  ---------------------------------------------------------------------------
  Bluetooth HCI Event - Connect Complete
      Event Code: Connect Complete (0x03)
      Parameter Total Length: 11
      Status: Success (0x00)
      Connection Handle: 0x0033
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
      Link Type: ACL connection (Data Channels) (0x01)
      Encryption Mode: Encryption Disabled (0x00)
      [Command in frame: 327]
      [Pending in frame: 328]
      [Pending-Response Delta: 3.747ms]
      [Command-Response Delta: 4.432ms]
  
  
  > 370	22.259461	controller	host	HCI_EVT	12	Rcvd IO Capability Response
  ---------------------------------------------------------------------------
  Bluetooth HCI Event - IO Capability Response
      Event Code: IO Capability Response (0x32)
      Parameter Total Length: 9
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
      IO Capability: Display Yes/No (0x01)
      OOB Data Present: OOB Authentication Data Not Present (0)
      Authentication Requirements: MITM Protection Required - Dedicated Bonding. Use IO Capability To Determine Procedure, No Secure Connection (3)
  
  > 371	22.259809	controller	host	HCI_EVT	9	Rcvd IO Capability Request
  ---------------------------------------------------------------------------
  Bluetooth HCI Event - IO Capability Request
      Event Code: IO Capability Request (0x31)
      Parameter Total Length: 6
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
  
  > 372	22.259982	host	controller	HCI_CMD	13	Sent IO Capability Request Reply
  ---------------------------------------------------------------------------
  Bluetooth HCI Command - IO Capability Request Reply
      Command Opcode: IO Capability Request Reply (0x042b)
      Parameter Total Length: 9
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
      IO Capability: Display Yes/No (1)
      OOB Data Present: OOB Authentication Data Not Present (0)
      Authentication Requirements: MITM Protection Required - Dedicated Bonding. Use IO Capability To Determine Procedure, No Secure Connection (3)
      [Response in frame: 373]
      [Command-Response Delta: 0.619ms]
  
  > 374	22.958223	controller	host	HCI_EVT	13	Rcvd User Confirmation Request
  ---------------------------------------------------------------------------
  Bluetooth HCI Event - User Confirmation Request
      Event Code: User Confirmation Request (0x33)
      Parameter Total Length: 10
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
      Numeric Value: 804568
  
  > 375	23.069385	host	controller	HCI_CMD	10	Sent User Confirmation Request Reply
  ---------------------------------------------------------------------------
  Bluetooth HCI Command - User Confirmation Request Reply
      Command Opcode: User Confirmation Request Reply (0x042c)
      Parameter Total Length: 6
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
      [Response in frame: 376]
      [Command-Response Delta: 1.81ms]
  
  #连接设备SSP验证成功
  > 377	26.919540	controller	host	HCI_EVT	10	Rcvd Simple Pairing Complete
  ---------------------------------------------------------------------------
  Bluetooth HCI Event - Simple Pairing Complete
      Event Code: Simple Pairing Complete (0x36)
      Parameter Total Length: 7
      Status: Success (0x00)
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
  
  #连接设备的Link Key，需要保存，用于下次连接
  > 378	26.926691	controller	host	HCI_EVT	26	Rcvd Link Key Notification
  ---------------------------------------------------------------------------
  Bluetooth HCI Event - Link Key Notification
      Event Code: Link Key Notification (0x18)
      Parameter Total Length: 23
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
      Link Key: 2160acf6b7403a97050f64de72570b1d
      Key Type: Unknown (0x08)
  ```
  
  

#### 设备连接

![](https://s1.ax1x.com/2022/04/11/LV63vD.png)

##### 连接

- **Host**:

  ```java
  CachedBluetoothDevice.conntect(true);
  ```

- **Controller**:

  ```dtd
  > 1214	47.023224	host	controller	HCI_CMD	17	Sent Create Connection
  ---------------------------------------------------------------------------
  Bluetooth HCI Command - Create Connection
      Command Opcode: Create Connection (0x0405)
      Parameter Total Length: 13
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
      Packet Type: 0xcc18, DH5, DM5, DH3, DM3, DH1, DM1
      Page Scan Repetition Mode: R1 (0x01)
      Page Scan Mode: Mandatory Page Scan Mode (0x00)
      .011 1001 0100 0100 = Clock Offset: 0x3944 (18325 msec)
      1... .... .... .... = Clock_Offset_Valid_Flag: true (1)
      Allow Role Switch: Local device may be master, or may become slave after accepting a master slave switch. (0x01)
      [Pending in frame: 1215]
      [Command-Pending Delta: 1.458ms]
      [Response in frame: 1217]
      [Command-Response Delta: 1630.81ms]
  
  > 1216	48.646874	controller	host	HCI_EVT	11	Rcvd Role Change
  ---------------------------------------------------------------------------
  Bluetooth HCI Event - Role Change
      Event Code: Role Change (0x12)
      Parameter Total Length: 8
      Status: Success (0x00)
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
      Role: Currently the Slave for specified BD_ADDR (0x01)
  
  > 1217	48.654034	controller	host	HCI_EVT	14	Rcvd Connect Complete
  ---------------------------------------------------------------------------
  Bluetooth HCI Event - Connect Complete
      Event Code: Connect Complete (0x03)
      Parameter Total Length: 11
      Status: Success (0x00)
      Connection Handle: 0x0033
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
      Link Type: ACL connection (Data Channels) (0x01)
      Encryption Mode: Encryption Disabled (0x00)
      [Command in frame: 1214]
      [Pending in frame: 1215]
      [Pending-Response Delta: 1629.352ms]
      [Command-Response Delta: 1630.81ms]
  
  > 1278	48.812485	host	controller	HCI_CMD	6	Sent Authentication Requested
  ---------------------------------------------------------------------------
  Bluetooth HCI Command - Authentication Requested
      Command Opcode: Authentication Requested (0x0411)
      Parameter Total Length: 2
      Connection Handle: 0x0033
      [Pending in frame: 1279]
      [Command-Pending Delta: 0.945ms]
      [Response in frame: 1283]
      [Command-Response Delta: 16.693ms]
  
  > 1280	48.813892	controller	host	HCI_EVT	9	Rcvd Link Key Request
  ---------------------------------------------------------------------------
  Bluetooth HCI Event - Link Key Request
      Event Code: Link Key Request (0x17)
      Parameter Total Length: 6
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
  
  > 1281	48.814039	host	controller	HCI_CMD	26	Sent Link Key Request Reply
  ---------------------------------------------------------------------------
  Bluetooth HCI Command - Link Key Request Reply
      Command Opcode: Link Key Request Reply (0x040b)
      Parameter Total Length: 22
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
      Link Key: 2160acf6b7403a97050f64de72570b1d
      [Response in frame: 1282]
      [Command-Response Delta: 1.336ms]
  
  > 1282	48.815375	controller	host	HCI_EVT	13	Rcvd Command Complete (Link Key Request Reply)
  ---------------------------------------------------------------------------
  Bluetooth HCI Event - Command Complete
      Event Code: Command Complete (0x0e)
      Parameter Total Length: 10
      Number of Allowed Command Packets: 1
      Command Opcode: Link Key Request Reply (0x040b)
      Status: Success (0x00)
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
      [Command in frame: 1281]
      [Command-Response Delta: 1.336ms]
  
  > 1283	48.829178	controller	host	HCI_EVT	6	Rcvd Authentication Complete
  ---------------------------------------------------------------------------
  Bluetooth HCI Event - Authentication Complete
      Event Code: Authentication Complete (0x06)
      Parameter Total Length: 3
      Status: Success (0x00)
      Connection Handle: 0x0033
      [Command in frame: 1278]
      [Pending in frame: 1279]
      [Pending-Response Delta: 15.748ms]
      [Command-Response Delta: 16.693ms]
  
  ```
  

##### 被连接

- **Host**:

  ```java
  LocalBluetoothManager.getInstance().getEventManager().registerCallback(new BluetoothCallback() {
     		@Override
  		public void onConnectionStateChanged(CachedBluetoothDevice cachedDevice, int bondState) {
  		}
  });
  ```

- **Controller**:

  ```dtd
  > 1900	62.151696	controller	host	HCI_EVT	13	Rcvd Connect Request
  ---------------------------------------------------------------------------
  Bluetooth HCI Event - Connect Request
      Event Code: Connect Request (0x04)
      Parameter Total Length: 10
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
      Class of Device: 0x5a020c (Phone:Smartphone - services: Networking Capturing ObjectTransfer Telephony)
      Link Type: ACL connection (Data Channels) (0x01)
  
  > 1901	62.152042	host	controller	HCI_CMD	11	Sent Accept Connection Request
  ---------------------------------------------------------------------------
  Bluetooth HCI Command - Accept Connection Request
      Command Opcode: Accept Connection Request (0x0409)
      Parameter Total Length: 7
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
      Role: Remain the Slave for this connection. The LM will NOT perform the role switch. (0x01)
      [Pending in frame: 1902]
      [Command-Pending Delta: 0.748ms]
      [Response in frame: 1903]
      [Command-Response Delta: 4.455ms]
  
  > 1903	62.156497	controller	host	HCI_EVT	14	Rcvd Connect Complete
  ---------------------------------------------------------------------------
  Bluetooth HCI Event - Connect Complete
      Event Code: Connect Complete (0x03)
      Parameter Total Length: 11
      Status: Success (0x00)
      Connection Handle: 0x0032
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
      Link Type: ACL connection (Data Channels) (0x01)
      Encryption Mode: Encryption Disabled (0x00)
      [Command in frame: 1901]
      [Pending in frame: 1902]
      [Pending-Response Delta: 3.707ms]
      [Command-Response Delta: 4.455ms]
  
  > 1968	62.415419	controller	host	HCI_EVT	9	Rcvd Link Key Request
  ---------------------------------------------------------------------------
  Bluetooth HCI Event - Link Key Request
      Event Code: Link Key Request (0x17)
      Parameter Total Length: 6
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
  
  > 1969	62.415629	host	controller	HCI_CMD	26	Sent Link Key Request Reply
  ---------------------------------------------------------------------------
  Bluetooth HCI Command - Link Key Request Reply
      Command Opcode: Link Key Request Reply (0x040b)
      Parameter Total Length: 22
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
      Link Key: 2160acf6b7403a97050f64de72570b1d
      [Response in frame: 1971]
      [Command-Response Delta: 0.934ms]
  
  > 1971	62.416563	controller	host	HCI_EVT	13	Rcvd Command Complete (Link Key Request Reply)
  ---------------------------------------------------------------------------
  Bluetooth HCI Event - Command Complete
      Event Code: Command Complete (0x0e)
      Parameter Total Length: 10
      Number of Allowed Command Packets: 1
      Command Opcode: Link Key Request Reply (0x040b)
      Status: Success (0x00)
      BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
      [Command in frame: 1969]
      [Command-Response Delta: 0.934ms]
  
  ```

### SCO

>BLUETOOTH SPECIFICATION Version 4.2 [Vol 1, Part A] page 64  
>
>The synchronous connection-oriented (SCO) logical transport is a symmetric,
>point-to-point transport between the master and a specific slave. The SCO
>logical transport reserves slots on the physical channel and can therefore be
>considered as a circuit-switched connection between the master and the slave.
>SCO logical transports carry 64 kb/s of information synchronized with the
>piconet clock. Typically this information is an encoded voice stream. Three
>different SCO configurations exist, offering a balance between robustness,
>delay and bandwidth consumption.  

SCO全称**BR/EDR Synchronous Connection-Oriented**，利用保留时隙传送数据包。连接建立后，主设备和从设备可以不被选中就发送SCO数据包。SCO数据包既可以传送话音，也可以传送数据，但在传送数据时，只用于重发被损坏的那部分的数据。

### L2CAP

> BLUETOOTH SPECIFICATION Version 4.2 [Vol 3, Part A] page 30  
>
> L2CAP（Logical Link Control and Adaptation Protocol）：逻辑链路控制与适配协议，将ACL数据分组交换为便于高层应用的数据分组格式，并提供协议复用和服务质量交换等功能。
>
> 通过协议多路复用、分段重组操作和组概念,向高层提供面向连接的和无连接的数据服务,L2CAP还屏蔽了低层传输协议中的很多特性，使得高层协议应用开发人员可以不必了解基层协议而进行开发。

![](https://s1.ax1x.com/2022/04/11/LV6sKg.png)

从架构图可以看出，L2CAP位于HCI与上层业务之间，在建立物理连接(ACL)之后，上层业务的通信和控制都由L2CAP协议与微微网(piconet )中其它蓝牙设备进行交互，类似HTTP协议的存在。

下图概括了两个设备L2CAP之间的连接过程：

![](https://s1.ax1x.com/2022/04/11/LV6UUI.png)

### SDP

> BLUETOOTH SPECIFICATION Version 4.2 [Vol 3, Part B] page 222  
>
> The service discovery mechanism provides the means for client applications to
> discover the existence of services provided by server applications as well as
> the attributes of those services. The attributes of a service include the type or
> class of service offered and the mechanism or protocol information needed to
> utilize the service.  

全称是**Service Discovery Protocol**，服务发现协议，服务发现协议(SDP)为应用程序提供了一种方法来发现哪些服务可用，并确定这些可用服务的特征。

![](https://s1.ax1x.com/2022/04/11/LV66bj.png)

从上图可知蓝牙设备中同时存在一个Client和Server，通过L2CAP协议与别的设备时行交互，通过`Service Search Attribute Request   `命令查询对方设备支持的蓝牙服务，其工作流程如图：

![](https://s1.ax1x.com/2022/04/11/LV6a5t.png)

通过HCI可以看到发现蓝牙服务过过程中的数据：

```dtd
> 455	22.824346	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	SDP	29	Sent Service Search Attribute Request : L2CAP: Attribute Range (0x0000 - 0xffff) 
---------------------------------------------------------------------------
Frame 455: 29 bytes on wire (232 bits), 29 bytes captured (232 bits)
Bluetooth
Bluetooth HCI H4
Bluetooth HCI ACL Packet
Bluetooth L2CAP Protocol
Bluetooth SDP Protocol
    PDU: Service Search Attribute Request (0x06)
    Transaction Id: 0x0000
    Parameter Length: 15
    Service Search Pattern: L2CAP
    Maximum Attribute Byte Count: 1008
    Attribute ID List
        Data Element: Sequence uint8 5 bytes
            0011 0... = Data Element Type: Sequence (6)
            .... .101 = Data Element Size: uint8 (5)
            Data Element Var Size: 5
            Data Value
                Data Element: Unsigned Integer 4 bytes
                    0000 1... = Data Element Type: Unsigned Integer (1)
                    .... .010 = Data Element Size: 4 bytes (2)
                    Data Value
                        Attribute Range: 0x0000ffff
                            Attribute Range From: 0x0000
                            Attribute Range To: 0xffff
    Continuation State: no (00)

> 471	22.896191	XiaomiCo_6e:38:c5 (PHONE)	localhost ()	SDP	281	Rcvd Service Search Attribute Response
---------------------------------------------------------------------------
Frame 471: 281 bytes on wire (2248 bits), 281 bytes captured (2248 bits)
Bluetooth
Bluetooth HCI H4
Bluetooth HCI ACL Packet
Bluetooth L2CAP Protocol
Bluetooth SDP Protocol
    PDU: Service Search Attribute Response (0x07)
    Transaction Id: 0x0001
    Parameter Length: 267
    Attribute List Byte Count: 264
    Data Fragment
    Continuation State: no (00)
    [Reassembled Attribute List]
        Attribute Lists [count = 14]
            Data Element: Sequence uint16 1269 bytes
                0011 0... = Data Element Type: Sequence (6)
                .... .110 = Data Element Size: uint16 (6)
                Data Element Var Size: 1269
                Data Value
                    Attribute List [count =  4] (Generic Attribute Profile)
                    Attribute List [count =  4] (Generic Access Profile)
                    Attribute List [count =  6] (Headset Audio Gateway)
                    Attribute List [count =  8] (Handsfree Audio Gateway)
                    Attribute List [count =  8] (A/V Remote Control Target)
                    Attribute List [count =  7] (Audio Source)
                    Attribute List [count = 11] (PAN NAP)
                    Attribute List [count =  9] (PAN PANU)
                    Attribute List [count =  7] (Phonebook Access Server)
                    Attribute List [count = 10] (Message Access Server)
                    Attribute List [count =  4] (Xiaomi Inc.)
                    Attribute List [count =  5] (CustomUUID: Unknown)
                    Attribute List [count =  8] (OBEX Object Push)
                    Attribute List [count =  4] (Unknown)
```



# 蓝牙电话


## PBAP(RFCOMM/OBEX)

> RFCOMM协议提供了对L2CAP协议上的串行端口的模拟。该协议基于ETSI标准GSM 07.10。
>
> OBEX：Object Exchange的简称，对象交换协议，来源于红外通讯协议，但又不局限于具体的传输方式，后来被蓝牙组织SIG吸纳其中部分并进行优化处理作为蓝牙协议中的OBEX层用于蓝牙设备间的文件数据传输，如蓝牙传输文件（OPP）、同步电话簿（PBAP）和同步短信（MAP）等场景下都是以OBEX协议组织相关数据进行传输的。

PBAP: Phone Book Access Profile的简称，**电话本访问协议**，用于同步手机这些具有电话本功能设备上的通讯录和通话记录等。

RFCOMM/OBEX和协议不属于蓝牙定义的规范 ，只是被蓝牙采纳，而PBAP协议包在OBEX协议之上封装而成。

### PBAP虚拟文件夹架构

PBAP中通话数据文件是在虚拟文件夹架构下，访问通话数据时需要指定路径，下图为官方架构图：

![](https://s1.ax1x.com/2022/04/11/LV6yrQ.png)

其中vcf的文件名称意义如下：

![](https://s1.ax1x.com/2022/04/11/LV6gVs.png)

### 建立连接/断开连接

只使用OBEX协议

#### 连接

```dtd
> 798	23.907353	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	OBEX	40	Sent Connect - Phone Book Access Profile
---------------------------------------------------------------------------
Frame 798: 40 bytes on wire (320 bits), 40 bytes captured (320 bits)
Bluetooth
Bluetooth HCI H4
Bluetooth HCI ACL Packet
Bluetooth L2CAP Protocol
Bluetooth RFCOMM Protocol
OBEX Protocol
    [Profile: PBAP (4)]
    [Current Path: ?]
    .000 0000 = Opcode: Connect (0x00)
    1... .... = Final Flag: True
    Packet Length: 26
    [Response in Frame: 803]
    Version: 1.0 (0x10)
    Flags: 0x00
    Max. Packet Length: 65534
    Headers
        Target: Phone Book Access Profile
            Header Id: Target (0x46)
            Length: 19
            Value: 796135f0f0c511d809660800200c9a66: Phone Book Access Profile


> 803	23.917550	XiaomiCo_6e:38:c5 (PHONE)	localhost ()	OBEX	45	Rcvd Success - Phone Book Access Profile
---------------------------------------------------------------------------
Frame 803: 45 bytes on wire (360 bits), 45 bytes captured (360 bits)
Bluetooth
Bluetooth HCI H4
Bluetooth HCI ACL Packet
Bluetooth L2CAP Protocol
Bluetooth RFCOMM Protocol
OBEX Protocol
    [Profile: PBAP (4)]
    [Current Path: /]
    .010 0000 = Response Code: Success (0x20)
    1... .... = Final Flag: True
    Packet Length: 31
    [Request in Frame: 798]
    Version: 1.0 (0x10)
    Flags: 0x00
    Max. Packet Length: 65534
    Headers
        Connection Id: 1
            Header Id: Connection Id (0xcb)
            Connection ID: 1
        Who: Phone Book Access Profile
            Header Id: Who (0x4a)
            Length: 19
            Value: 796135f0f0c511d809660800200c9a66: Phone Book Access Profile

```

#### 断开

```dtd
> 1142	58.752933	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	OBEX	22	Sent Disconnect
---------------------------------------------------------------------------
OBEX Protocol
    [Profile: PBAP (4)]
    [Current Path: /]
    .000 0001 = Opcode: Disconnect (0x01)
    1... .... = Final Flag: True
    Packet Length: 8
    [Response in Frame: 1164]
    Headers
        Connection Id: 1
            Header Id: Connection Id (0xcb)
            Connection ID: 1

> 1164	59.162598	XiaomiCo_6e:38:c5 (PHONE)	localhost ()	OBEX	22	Rcvd Success
---------------------------------------------------------------------------
OBEX Protocol
    [Profile: PBAP (4)]
    [Current Path: /]
    .010 0000 = Response Code: Success (0x20)
    1... .... = Final Flag: True
    Packet Length: 8
    [Request in Frame: 1142]
    Headers
        Connection Id: 1
            Header Id: Connection Id (0xcb)
            Connection ID: 1
```

### 设置访问虚拟文件夹路径

在[PBAP协议规范]的`5.2 SetPhoneBook Function`章节，使用OBEX协议SetPath命令，通过obex.opcode=**0x05**判断。

```dtd
# $/
> 808	24.023528	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	OBEX	27	Sent Set Path ""
---------------------------------------------------------------------------
OBEX Protocol
    [Profile: PBAP (4)]
    [Current Path: /]
    .000 0101 = Opcode: Set Path (0x05)
    1... .... = Final Flag: True
    Packet Length: 13
    [Response in Frame: 810]
    Flags: 0x02
    .... ...0 = Go back one folder (../) first: False
    .... ..1. = Do not create folder, if not existing: True
    Constants: 0x00
    Headers
        Connection Id: 1
            Header Id: Connection Id (0xcb)
            Connection ID: 1
        Name: ""
            Header Id: Name (0x01)
            Length: 3
            Name: 

> 810	24.065127	XiaomiCo_6e:38:c5 (PHONE)	localhost ()	OBEX	22	Rcvd Success
---------------------------------------------------------------------------
OBEX Protocol
    [Profile: PBAP (4)]
    [Current Path: /]
    .010 0000 = Response Code: Success (0x20)
    1... .... = Final Flag: True
    Packet Length: 8
    [Request in Frame: 808]
    Headers
        Connection Id: 1
            Header Id: Connection Id (0xcb)
            Connection ID: 1

# $/telecom/
> 811	24.078180	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	OBEX	43	Sent Set Path "telecom"
```



### 获取通讯录数量

在[PBAP协议规范]的`5.3 PullvCardListing Function`章节，通过obex.header=**<x-bt/vcard-listing>**判断。

#### 手机

```dtd
> 808	24.023528	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	OBEX	27	Sent Set Path ""
> 811	24.078180	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	OBEX	43	Sent Set Path "telecom"
# $/telecom/pb/
> 816	24.118267	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	OBEX	70	Sent Get final "pb"
---------------------------------------------------------------------------
OBEX Protocol
    [Profile: PBAP (4)]
    [Current Path: /telecom]
    .000 0011 = Opcode: Get (0x03)
    1... .... = Final Flag: True
    Packet Length: 56
    [Response in Frame: 818]
    Headers
        Connection Id: 1
            Header Id: Connection Id (0xcb)
            Connection ID: 1
        Name: "pb"
            Header Id: Name (0x01)
            Length: 9
            Name: pb
        Type: "x-bt/vcard-listing"
            Header Id: Type (0x42)
            Length: 22
            Type: x-bt/vcard-listing
        Application Parameters
            Header Id: Application Parameters (0x4c)
            Length: 17
            Parameter: Order #排序规则: { Alphabetical | Indexed | Phonetical }
                Parameter Id: Order (0x01)
                Parameter Length: 1
                Max List Count: Indexed (0x00)
            Parameter: Search Attribute #搜索:  {Name | Number | Sound } 与 SearchValue配合，默认为Name
                Parameter Id: Search Attribute (0x03)
                Parameter Length: 1
                Search Attribute: Name (0x00)
            Parameter: Max List Count #最大条目数
                Parameter Id: Max List Count (0x04)
                Parameter Length: 2
                Max List Count: 0 (0x0000)
            Parameter: List Start Offset #偏移量
                Parameter Id: List Start Offset (0x05)
                Parameter Length: 2
                List Start Offset: 0 (0x0000)

> 818	24.332995	XiaomiCo_6e:38:c5 (PHONE)	localhost ()	OBEX	29	Rcvd Success
---------------------------------------------------------------------------
OBEX Protocol
    [Profile: PBAP (4)]
    [Current Path: /telecom]
    .010 0000 = Response Code: Success (0x20)
    1... .... = Final Flag: True
    Packet Length: 15
    [Request in Frame: 816]
    Headers
        Connection Id: 1
            Header Id: Connection Id (0xcb)
            Connection ID: 1
        Application Parameters
            Header Id: Application Parameters (0x4c)
            Length: 7
            Parameter: Phonebook Size
                Parameter Id: Phonebook Size (0x08)
                Parameter Length: 2
                Phonebook Size: 71 (0x0047)
```

#### SIM1

```dtd
> 819	24.336294	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	OBEX	27	Sent Set Path ""
> 822	24.379640	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	OBEX	27	Sent Set Path ""
> 825	24.430836	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	OBEX	37	Sent Set Path "SIM1"
> 828	24.481559	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	OBEX	43	Sent Set Path "telecom"
# $/SIM1/telecom/pb/
> 831	24.532312	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	OBEX	70	Sent Get final "pb"

> 833	24.578831	XiaomiCo_6e:38:c5 (PHONE)	localhost ()	OBEX	29	Rcvd Success
---------------------------------------------------------------------------
OBEX Protocol
    [Profile: PBAP (4)]
    [Current Path: /SIM1/telecom]
    .010 0000 = Response Code: Success (0x20)
    1... .... = Final Flag: True
    Packet Length: 15
    [Request in Frame: 831]
    Headers
        Connection Id: 1
            Header Id: Connection Id (0xcb)
            Connection ID: 1
        Application Parameters
            Header Id: Application Parameters (0x4c)
            Length: 7
            Parameter: Phonebook Size
                Parameter Id: Phonebook Size (0x08)
                Parameter Length: 2
                Phonebook Size: 1 (0x0001)
```

### 获取通讯录

在[PBAP协议规范]的`5.3 PullvCardListing Function`章节，通过obex.header=**<x-bt/phonebook>** 判断，分页获取vCard格式。

#### 手机

```dtd
> 834	24.580643	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	OBEX	27	Sent Set Path ""
#分页请求获取通讯录，从第1条数据开始，每页最多50条
> 837	24.632264	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	OBEX	97	Sent Get final "telecom/pb.vcf"
---------------------------------------------------------------------------
OBEX Protocol
    [Profile: PBAP (4)]
    [Current Path: /]
    .000 0011 = Opcode: Get (0x03)
    1... .... = Final Flag: True
    Packet Length: 83
    [Response in Frame: 844]
    Headers
        Connection Id: 1
            Header Id: Connection Id (0xcb)
            Connection ID: 1
        Name: "telecom/pb.vcf"
            Header Id: Name (0x01)
            Length: 33
            Name: telecom/pb.vcf
        Type: "x-bt/phonebook"
            Header Id: Type (0x42)
            Length: 18
            Type: x-bt/phonebook
        Application Parameters
            Header Id: Application Parameters (0x4c)
            Length: 24
            Parameter: Max List Count #最大条目数,这里只读取50条
                Parameter Id: Max List Count (0x04)
                Parameter Length: 2
                Max List Count: 50 (0x0032)
            Parameter: List Start Offset #偏移量，这是从第1条开始读取
                Parameter Id: List Start Offset (0x05)
                Parameter Length: 2
                List Start Offset: 1 (0x0001)
            Parameter: Filter #用于指示所请求的vCard中包含的属性
                Parameter Id: Filter (0x06)
                Parameter Length: 8
                Filter: 0x00000000
                Filter: 0x008001af, vCard Version, Formatted Name, Structured Presentation of Name, Associated Image or Photo, Delivery Address, Telephone Number, Electronic Mail Address, Nickname
            Parameter: Format #vCard文件格式版本，这是3.0
                Parameter Id: Format (0x07)
                Parameter Length: 1
                Format: 3.0 (0x01)

#传输数据片段
> 839	24.764415	XiaomiCo_6e:38:c5 (PHONE)	localhost ()	OBEX	1004	Rcvd OBEX fragment
---------------------------------------------------------------------------
Frame 839: 1004 bytes on wire (8032 bits), 1004 bytes captured (8032 bits)
Bluetooth
Bluetooth HCI H4
Bluetooth HCI ACL Packet
Bluetooth L2CAP Protocol
Bluetooth RFCOMM Protocol
OBEX Protocol
    [Profile: PBAP (4)]
    [Current Path: /]
    [Reassembled OBEX in frame: 844]
    Data (990 bytes)

...
#传输成功
> 844	24.794969	XiaomiCo_6e:38:c5 (PHONE)	localhost ()	OBEX	182	Rcvd Success (x-bt/phonebook)
---------------------------------------------------------------------------
OBEX Protocol
    [Profile: PBAP (4)]
    [Current Path: /]
    [6 OBEX Fragments (5117 bytes): #839(990), #840(990), #841(990), #842(990), #843(990), #844(167)] #6个数据片段，需要拼接对应Frame
    .010 0000 = Response Code: Success (0x20)
    1... .... = Final Flag: True
    Packet Length: 5117
    [Request in Frame: 837]
    Headers
        Connection Id: 1
            Header Id: Connection Id (0xcb)
            Connection ID: 1
        End Of Body
            Header Id: End Of Body (0x49)
            Length: 5109
            Value: 424547494e3a56434152440d0a56455253494f4e3a332e300d0a4e3ae591a83be4baa6e6…
    Line-based text data: x-bt/phonebook (320 lines)
        BEGIN:VCARD\r\n
        VERSION:3.0\r\n
        N:张;三;;;\r\n
        FN:张三\r\n
        TEL;TYPE=CELL:22222222\r\n
        END:VCARD\r\n
        BEGIN:VCARD\r\n
        VERSION:3.0\r\n
        N:李四;;;;\r\n
        FN:李四\r\n
        TEL;TYPE=CELL:11111111\r\n
        END:VCARD\r\n

#请求获取vCard 3.0格式，并从第51条数据开始，最多50条
> 847	24.875858	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	OBEX	96	Sent Get final "telecom/pb.vcf"
---------------------------------------------------------------------------
OBEX Protocol
    [Profile: PBAP (4)]
    [Current Path: /]
    .000 0011 = Opcode: Get (0x03)
    1... .... = Final Flag: True
    Packet Length: 83
    [Response in Frame: 850]
    Headers
        Connection Id: 1
            Header Id: Connection Id (0xcb)
            Connection ID: 1
        Name: "telecom/pb.vcf"
            Header Id: Name (0x01)
            Length: 33
            Name: telecom/pb.vcf
        Type: "x-bt/phonebook"
            Header Id: Type (0x42)
            Length: 18
            Type: x-bt/phonebook
        Application Parameters
            Header Id: Application Parameters (0x4c)
            Length: 24
            Parameter: Max List Count
                Parameter Id: Max List Count (0x04)
                Parameter Length: 2
                Max List Count: 50 (0x0032)
            Parameter: List Start Offset
                Parameter Id: List Start Offset (0x05)
                Parameter Length: 2
                List Start Offset: 51 (0x0033)
            Parameter: Filter
                Parameter Id: Filter (0x06)
                Parameter Length: 8
                Filter: 0x00000000
                Filter: 0x008001af, vCard Version, Formatted Name, Structured Presentation of Name, Associated Image or Photo, Delivery Address, Telephone Number, Electronic Mail Address, Nickname
            Parameter: Format
                Parameter Id: Format (0x07)
                Parameter Length: 1
                Format: 3.0 (0x01)

> 849	24.953425	XiaomiCo_6e:38:c5 (PHONE)	localhost ()	OBEX	1004	Rcvd OBEX fragment

# 刚刚通过`Sent Get final "pb"`获取手机通讯录只有71条，现在只剩下21，这次就可以传输完成。
> 850	24.960363	XiaomiCo_6e:38:c5 (PHONE)	localhost ()	OBEX	940	Rcvd Success (x-bt/phonebook)
---------------------------------------------------------------------------
OBEX Protocol
    [Profile: PBAP (4)]
    [Current Path: /]
    [2 OBEX Fragments (1915 bytes): #849(990), #850(925)]
    .010 0000 = Response Code: Success (0x20)
    1... .... = Final Flag: True
    Packet Length: 1915
    [Request in Frame: 847]
    Headers
        Connection Id: 1
            Header Id: Connection Id (0xcb)
            Connection ID: 1
        End Of Body
            Header Id: End Of Body (0x49)
            Length: 1907
            Value: 424547494e3a56434152440d0a56455253494f4e3a332e300d0a4e3ae4bd993be4bf8ae7…
    Line-based text data: x-bt/phonebook (122 lines)
        BEGIN:VCARD\r\n
        VERSION:3.0\r\n
        N:张;三;;;\r\n
        FN:张三\r\n
        TEL;TYPE=CELL:22222222\r\n
        END:VCARD\r\n
        BEGIN:VCARD\r\n
        VERSION:3.0\r\n
        N:李四;;;;\r\n
        FN:李四\r\n
        TEL;TYPE=CELL:11111111\r\n
        END:VCARD\r\n
```

#### SIM1

```dtd
> 866	25.021131	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	OBEX	107	Sent Get final "SIM1/telecom/pb.vcf"
#这里手机直接拒绝了
> 868	25.032728	XiaomiCo_6e:38:c5 (PHONE)	localhost ()	OBEX	25	Rcvd Success
---------------------------------------------------------------------------
OBEX Protocol
    [Profile: PBAP (4)]
    [Current Path: /]
    .010 0000 = Response Code: Success (0x20)
    1... .... = Final Flag: True
    Packet Length: 11
    [Request in Frame: 866]
    Headers
        Connection Id: 1
            Header Id: Connection Id (0xcb)
            Connection ID: 1
        End Of Body
            Header Id: End Of Body (0x49)
            Length: 3
            Value: <MISSING>
```



### 获取通话记录

在[PBAP协议规范]的`5.3 PullvCardListing Function`章节，通过obex.header=**<x-bt/phonebook>** 判断，分页获取vCard格式。

```dtd
> 869	25.071027	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	OBEX	27	Sent Set Path ""
> 882	25.125323	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	OBEX	43	Sent Set Path "telecom"
# $/telecom/cch/
> 887	25.138796	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	OBEX	72	Sent Get final "cch"
> 895	25.222650	XiaomiCo_6e:38:c5 (PHONE)	localhost ()	OBEX	29	Rcvd Success
---------------------------------------------------------------------------
OBEX Protocol
    [Profile: PBAP (4)]
    [Current Path: /telecom]
    .010 0000 = Response Code: Success (0x20)
    1... .... = Final Flag: True
    Packet Length: 15
    [Request in Frame: 887]
    Headers
        Connection Id: 1
            Header Id: Connection Id (0xcb)
            Connection ID: 1
        Application Parameters
            Header Id: Application Parameters (0x4c)
            Length: 7
            Parameter: Phonebook Size
                Parameter Id: Phonebook Size (0x08)
                Parameter Length: 2
                Phonebook Size: 2177 (0x0881)

> 897	25.225643	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	OBEX	27	Sent Set Path ""
#分页获取通话记录，从第0条开始，每页最多50条
> 919	25.279427	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	OBEX	85	Sent Get final "telecom/cch.vcf"
---------------------------------------------------------------------------
OBEX Protocol
    [Profile: PBAP (4)]
    [Current Path: /]
    .000 0011 = Opcode: Get (0x03)
    1... .... = Final Flag: True
    Packet Length: 71
    [Response in Frame: 935]
    Headers
        Connection Id: 1
            Header Id: Connection Id (0xcb)
            Connection ID: 1
        Name: "telecom/cch.vcf"
            Header Id: Name (0x01)
            Length: 35
            Name: telecom/cch.vcf
        Type: "x-bt/phonebook"
            Header Id: Type (0x42)
            Length: 18
            Type: x-bt/phonebook
        Application Parameters
            Header Id: Application Parameters (0x4c)
            Length: 10
            Parameter: Max List Count
                Parameter Id: Max List Count (0x04)
                Parameter Length: 2
                Max List Count: 50 (0x0032)
            Parameter: Format
                Parameter Id: Format (0x07)
                Parameter Length: 1
                Format: 3.0 (0x01)
> 935	25.381127	XiaomiCo_6e:38:c5 (PHONE)	localhost ()	OBEX	132	Rcvd Success (x-bt/phonebook)
---------------------------------------------------------------------------
OBEX Protocol
    [Profile: PBAP (4)]
    [Current Path: /]
    [8 OBEX Fragments (7048 bytes): #926(990), #927(990), #928(990), #929(990), #930(990), #931(990), #934(990), #935(118)]
    .010 0000 = Response Code: Success (0x20)
    1... .... = Final Flag: True
    Packet Length: 7048
    [Request in Frame: 919]
    Headers
        Connection Id: 1
            Header Id: Connection Id (0xcb)
            Connection ID: 1
        End Of Body
            Header Id: End Of Body (0x49)
            Length: 7040
            Value: 424547494e3a56434152440d0a56455253494f4e3a332e300d0a464e3a0d0a4e3a0d0a54…
    Line-based text data: x-bt/phonebook (350 lines)
        BEGIN:VCARD\r\n
        VERSION:3.0\r\n
        FN:\r\n
        N:\r\n
        TEL;TYPE=0:10086\r\n
        X-IRMC-CALL-DATETIME;TYPE=DIALED:20220317T161710\r\n
        END:VCARD\r\n
        BEGIN:VCARD\r\n
        VERSION:3.0\r\n
        FN:\r\n
        N:\r\n
        TEL;TYPE=0:02310050\r\n
        X-IRMC-CALL-DATETIME;TYPE=MISSED:20220316T160142\r\n
        END:VCARD\r\n

#分页获取通话记录，从第50条开始，每页最多50条
936	25.423573	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	OBEX	89	Sent Get final "telecom/cch.vcf"
---------------------------------------------------------------------------
OBEX Protocol
    [Profile: PBAP (4)]
    [Current Path: /]
    .000 0011 = Opcode: Get (0x03)
    1... .... = Final Flag: True
    Packet Length: 75
    [Response in Frame: 947]
    Headers
        Connection Id: 1
            Header Id: Connection Id (0xcb)
            Connection ID: 1
        Name: "telecom/cch.vcf"
            Header Id: Name (0x01)
            Length: 35
            Name: telecom/cch.vcf
        Type: "x-bt/phonebook"
            Header Id: Type (0x42)
            Length: 18
            Type: x-bt/phonebook
        Application Parameters
            Header Id: Application Parameters (0x4c)
            Length: 14
            Parameter: Max List Count
                Parameter Id: Max List Count (0x04)
                Parameter Length: 2
                Max List Count: 50 (0x0032)
            Parameter: List Start Offset
                Parameter Id: List Start Offset (0x05)
                Parameter Length: 2
                List Start Offset: 50 (0x0032)
            Parameter: Format
                Parameter Id: Format (0x07)
                Parameter Length: 1
                Format: 3.0 (0x01)
> 947	25.516478	XiaomiCo_6e:38:c5 (PHONE)	localhost ()	OBEX	810	Rcvd Success (x-bt/phonebook)
```

## HFP(RFCOMM)

> HFP(Hands-free Profile)，让蓝牙设备可以控制电话，如接听、挂断、拒接、语音拨号等，拒接、语音拨号要视蓝牙耳机及电话是否支持。
>
> 目前HFP的使用场景有车载蓝牙，耳机和PDA，定义了AG和HFP两种角色。
> AG（Audio Gate）音频网关—音频设备输入输出网关
> HF（Hands Free）免提—该设备作为音频网关的远程音频输入/输出机制，并可提供若干遥控功能。

在车载蓝牙中，**手机侧是AG**，**车载蓝牙侧是HF**，在android源代码中，将AG侧称为HFP/AG，将HF侧称为HFPClient/HF。

### AT命令

在HFP使用AT命令控制电话，其参考3GPP 27.007标准，命令规范为：

- 每个命令行只有一个命令

- AG侧默认不回显命令

- AG使用冗长的格式返回结果

- 以下字符将被用于AT命令和返回结果格式中

  - `<cr>` 表示回车，也就是`\r`字符

  - ` <lf>`表示换行，也就是`\n`字符

- 从HF发送到AG的命令格式是：`<AT command> <cr>`

- 从AG返回给HF的OK命令格式是：`<cr><lf>OK<cr><lf>`

- 从AG到HF的ERROR命令是：`<cr><lf>ERROR<cr><lf>`

- 从AG到HF的结果命令格式是：`<cr><lf><result code><cr><lf>`

HFP协议复用的AT命令：

- ATA：标准电话应答AT命令
- ATDdd...dd;：用电话号码打电话
- ATD>nnn...;：ATD扩展命令，记忆拨号
- ERROR：错误指示符，语法，格式或者通信过程出错。
- OK：命令的成功应答。
- NO CARRIER, BUSY, NO ANSWER, DELAYED, BLACKLISTED：AT扩展命令，AG返回给HF。
- RING：来电
- AT+CCWA：calling waiting notification AT命令。AT+CCWA=`[<n>[,<mode>[,<class>]]]`，
- +CCWA：Call Waiting notification返回结果码。只有`<number>`和`<type>`参数对HFP有意义，`<number>`是由双引号及其中的文本串组成。`<type>`是支持的电话格式，有如下值：
  - 128~143：国家或国际格式，
  - 144~159：国际电话，包括国家码前缀。
  - 160-175:国家码
  - AT+CHLD：通话保持，多方处理。AT+CHLD=`<n>`中`<n>`值覆盖0, 1, 1`<idx>`, 2, 2`<idx>`, 3 and 4,说明：
    - 0：释放所有保持电话或者设置用户的忙等待
    - 1：释放正在通话的电话，接听保持或等待的电话
    - 1`<idx>`:释放`<idx>`标识的电话
    - 2：将所有活跃电话设置成保持并且接受其它电话。
    - 2`<idx>`:请求接受`<idx>`标识电话，让其它电话保持。
    - 3：增加一个保持电话到对话中
    - 4：连接连个电话并且断开两个电话的订阅。HF侧可选。
  
  - AT+CHLD=?：查询AG侧保持和多方会话。
- AT+CHUP：标准的挂断命令。AG会结束通话，也可用于拒接来电。
- AT+CIND：`AT+CIND?`获取AG indicators当前的状态,`AT+CIND=?获取AG支持的indicator以及它们的次序。
  - service: Service availability indication:
    - `<value>=0` implies no service. No Home/Roam network available.
    - `<value>=1` implies presence of service. Home/Roam network available.  
  - call: Standard call status indicator:  
    - `<value>=0 `means there are no calls in progress  
    - `<value>=1` means at least one call is in progress  
  - callsetup: Bluetooth proprietary call set up status indicator4. Support for this indicator is optional
    for the HF. When supported, this indicator shall be used in conjunction with, and as an extension
    of the standard call indicator. Possible values are as follows:  
    - `<value>=0` means not currently in call set up.  
    - `<value>=1` means an incoming call process ongoing.
    - `<value>=2` means an outgoing call set up is ongoing.  
    - `<value>=3` means remote party being alerted in an outgoing call.  
  - callheld: Bluetooth proprietary call hold status indicator. Support for this indicator is mandatory for
    the AG, optional for the HF. Possible values are as follows:  
    - 0= No calls held  
    - 1= Call is placed on hold or active/held calls swapped
      (The AG has both an active AND a held call)  
    - Call on hold, no active call  
  - signal: Signal Strength indicator:  
    - `<value>= `ranges from 0 to 5  
  - roam: Roaming status indicator:  
    - `<value>=0` means roaming is not active  
    - `<value>=1` means a roaming is active  
  - battchg: Battery Charge indicator of AG:  
    - `<value>=`ranges from 0 to 5  
- +CIND：当前indicator的列表
- AT+CLCC：列出当前电话命令，
- +CLCC：当前call结果码，支持参数是

  - idx：表示建立连接顺序或者接听电话的数字（从1开始）。
  - dir：0（outgoing），1（incoming）
  - status：

    - 0=Active

    - 1=Held

    - 2=Dialing（outgoing calls only）

    - 3=Alerting(outgoing calls only)

    - 4=Incoming(incoming calls only)

    - 5=Waiting(incoming calls only)

    - 6 = Call held by Response and Hold
  - mode= 0 (Voice), 1 (Data), 2 (FAX)
  - mpty=
    - 0 - this call is NOT a member of a multi-party (conference) call
    - 1 - this call IS a member of a multi-party (conference) call
  - number (optional)
  - type (optional)
- AT+COPS: AT+COPS=3,0将被HF发送给AG
- AT+CMEE: 使能+CME ERROR: <err>结果码
- +CME ERROR
  - +CME ERROR: 0 – AG failure
- AT+CLIP: Calling Line Identification notification 使能命令，It enables/disables the Calling Line Identification notification unsolicited result code 。
- +CLIP : Standard “Calling Line Identification notification” unsolicited result code.  
- AT+CMER :Standard event reporting activation/deactivation AT command.  
- +CIEV: “indicator events reporting”结果码。`<ind>`,`<value>` result code ，对应AT+CIND 返回的列表中的参数和值。
- AT+VTS: DTMF生成命令。
- AT+CNUM: HF相应`+CNUM`命令
  - AT+CNUM (Retrieve Subscriber Number Information)
  - AT+CNUM=? (Test Subscriber Number Information – Not Implemented)
- +CNUM:  用于将“ Subscriber Number Information ”从AG发送到HF的标准响应。

HFP协议扩展的AT命令：

- AT+BIA (Bluetooth Indicators Activation)  
- AT+BINP (Bluetooth INPut)  
- AT+BLDN (Bluetooth Last Dialed Number)  
- AT+BVRA (Bluetooth Voice Recognition Activation)  
- +BVRA (Bluetooth Voice Recognition Activation)  
- AT+BRSF (Bluetooth Retrieve Supported Features)  
- +BRSF (Bluetooth Retrieve Supported Features)   
- AT+NREC (Noise Reduction and Echo Canceling)  
- AT+VGM (Gain of Microphone)  
- AT+VGS (Gain of Speaker)  
- +VGM (Gain of Microphone)  
- +VGS (Gain of Speaker)  
- +BSIR (Bluetooth Setting of In-band Ring tone)  
- AT+BTRH (Bluetooth Response and Hold Feature)  
- +BTRH (Bluetooth Response and Hold Feature)  
- AT+BCC (Bluetooth Codec Connection)  
- AT+BCS (Bluetooth Codec Selection)  
- +BCS (Bluetooth Codec Selection)  
- AT+BAC (Bluetooth Available Codecs)  
- AT+BIND (Bluetooth HF Indicators Feature)  
- +BIND (Bluetooth HF Indicators Feature)  
- AT+BIEV (Bluetooth HF Indicators Feature)

### 建立服务连接

> Upon a user action or an internal event, either the HF or the AG may initiate a **Service Level Connection**
> **establishment** procedure.
> A Service Level Connection establishment requires the existence of a RFCOMM connection, that is, a
> RFCOMM data link channel between the HF and the AG.
> Both the HF and the AG may initiate the RFCOMM connection establishment. If there is no RFCOMM
> session between the AG and the HF, the initiating device shall first initialize RFCOMM.  

连接过程如下：

![](https://s1.ax1x.com/2022/04/11/LV6DxS.png)

1. 支持能力交换
   首先HF发送AT+BRSF=< HF supported features >给AG，目的是首先通知AG其具有的能力，其次接收AG返回的其自身的BRSF能力。
   
2. Codec协商
   如果HF支持Codec Negotiation特征，其会查看AG返回的BRSF中是否也支持该特性，如果都支持该特性，则HF将发送AT+BAC=< HF available codecs >命令给AG以告知其可用的codec。
   
3. AG Indicator
   HF从AG接收到的BRSF，可以知道AG支持的Indicator，并按顺序拍好，这是因为根据3GPP 27.007规范，AG可以支持Hands-Free不支持的profile。HF使用AT+CIND=?测试命令接收AG支持的indicator以及它们的次序。
   当HF获得必须的Indicator和它们的次序，它将通过AT+CIND?命令取得AG端正在使用indicator的状态。
   当HF取得AG的indicator后，HF会使用AT+CMER使能AG的indicator状态跟新功能，AG会返回OK作为应答。当service，call或者call建立状态发生时，AG将发送和indicator相关的+CIEV结果码给HF。HF根据收到的+CIEV码来跟新其自身内部的indicator。
   AG侧会一直保持indicator状态跟新功能使能直到收到AT+CMER指示其关闭或者HF和AG端的Service Level Connection连接断开。
   当HF使能AG的indicator状态跟新，如果AG和HF都支持呼叫等待（Call waiting）和3方通信（3-way calling）。HF将发送AT+CHLD=?测试命令取得AG是如何支持这种功能的。如果HF或者AG其中之一不支持三方通信，AT+CHLD=?命令不会被发送。
   
4. HF Indicator
   如果HF支持HF indicator，其会查看AG是否支持HF indicator。
   如果HF和AG支持HF indicator特性，HF将发送AT+BIND=< HF supported HF indicators >通知HF侧支持的indicator，AG以OK应答。
   当AG接收到HF告知的HF indicator特性，HF将发送AT+BIND=?请求AG侧支持的HF indicator。AG将会以+BIND和以OK结尾的应答。
   当HF接收到AG支持的HF indicator，HF将会发送AT+BIND?命令确定HF目前使能的HF indicator。AG将会一次或多次以+BIND应答和以OK结尾的应答。
   至此HF可能发送AT+BIEV命令告知AG其使能的HF indicator发生变化。
   AG可以使用+BIND使能或者禁止任何HF indicator。
   
5. End of Service Level Connection
   - HF需要知道Service Level Connection被完全建立，这可以通过以下几个方式：
     - 当且仅当AG通过+BRSF命令告知HF其支持的`"HF indicator"`，在HF收到AG通过**AT+BIND？**命令发来的其支持的`"HF indicator"`可认为完全建立。
     - 当且仅当SDP服务发现AH和AG双方均支持`"Call waiting and 3-way calling"`，在HFAG通过**AT+CHLD**命令发来的其对呼叫等待和多方电话的支持，对这种情况，`"HF indicator"`不要设置该比特位，AG也不要在+BRSF命令中设置该比特位。
     - 在HF使用**AT+CMER**命令成功启用`“Indicator status update”`功能，对这种情况SDP服务不应该设置`“Call waiting and 3-way calling”`比特位。
       如果HF收到AG通过indicator指示当前有电话时，HF查询AG的接听和保持状态来判断是否是未接听电话。
     
   - 同样AG侧Service Level Connection完全建立也有几种情况：
     - 当且仅当HF indicator在HF被设置且AG侧支持的indicator已经通过+BRSF命令应答，则AG以**+BIND加OK结尾**的命令应答其使能的HF indicator时可认为Service Level Connection完全建立。
     
     - 当且仅当`“Call waiting and 3-way calling”`比特在HF和AG的SDP服务中被置位，在AG通过**+CHLD加OK结尾**命令成功响应其对呼叫保持和多方电话支持时SLC会被完全建立。对这种情况，+BRSF不应该设置该HF indicator比特位。
     
     - AG已成功**响应OK到AT + CMER**命令（启用`“Indicator status update”`功能。）当`“Call waiting and 3-way calling”`位时，应适用这种情况未在支持的功能位图中设置为HF或AG，以及`“HF Indicators ”`对于通过+ BRSF交换的HF或AG的支持功能位图中未设置位的位图命令。
     

```dtd
#判断是否支持"Call waiting and 3-way calling"
#AG发现HF, hfP支持的协议,HF支持"Call waiting and 3-way calling"
> 596	17.850811	XiaomiCo_6e:38:c5 (PHONE)	SANYOEle_62:35:bb (DEVICE)	SDP	36	Rcvd Service Search Attribute Request : Handsfree: [Service Class ID List 0x0001] [Protocol Descriptor List 0x0004] [Bluetooth Profile Descriptor List 0x0009] [(HFP HS) Supported Features 0x0311] 
> 597	17.851437	SANYOEle_62:35:bb (DEVICE)	XiaomiCo_6e:38:c5 (PHONE)	SDP	69	Sent Service Search Attribute Response 
---------------------------------------------------------------------------
Bluetooth SDP Protocol
    PDU: Service Search Attribute Response (0x07)
    Transaction Id: 0x0000
    Parameter Length: 55
    Attribute List Byte Count: 52
    Attribute Lists [count =  1]
        Data Element: Sequence uint8 50 bytes
            0011 0... = Data Element Type: Sequence (6)
            .... .101 = Data Element Size: uint8 (5)
            Data Element Var Size: 50
            Data Value
                Attribute List [count =  4] (Handsfree)
                    Data Element: Sequence uint16 47 bytes
                        0011 0... = Data Element Type: Sequence (6)
                        .... .110 = Data Element Size: uint16 (6)
                        Data Element Var Size: 47
                        Data Value
                            Service Attribute: Service Class ID List (0x1), value = Handsfree -> Generic Audio
                            Service Attribute: Protocol Descriptor List (0x4), value = L2CAP -> RFCOMM:2
                            Service Attribute: Bluetooth Profile Descriptor List (0x9), value = Handsfree 1.6
                            Service Attribute: (HFP HS) Supported Features (0x311), value = (EC and/or Nr Function) (Call Waiting or Three Way Calling) (CLI Presentation Capability) (Voice Recognition Activation) (Remote Volume Control) (Wide Band Speech) 
    Continuation State: no (00)

#HF发现AG, hfP支持的协议,AG不支持"Call waiting and 3-way calling"，只支持"3-way calling"
> 637	18.017460	SANYOEle_62:35:bb (DEVICE)	XiaomiCo_6e:38:c5 (PHONE)	SDP	33	Sent Service Search Attribute Request : Handsfree Audio Gateway: [Service Class ID List 0x0001] [Bluetooth Profile Descriptor List 0x0009] [(HFP AG) Supported Features 0x0311] 
> 644	18.024689	XiaomiCo_6e:38:c5 (PHONE)	SANYOEle_62:35:bb (DEVICE)	SDP	52	Rcvd Service Search Attribute Response 
---------------------------------------------------------------------------
Bluetooth SDP Protocol
    PDU: Service Search Attribute Response (0x07)
    Transaction Id: 0x0000
    Parameter Length: 38
    Attribute List Byte Count: 35
    Attribute Lists [count =  1]
        Data Element: Sequence uint8 33 bytes
            0011 0... = Data Element Type: Sequence (6)
            .... .101 = Data Element Size: uint8 (5)
            Data Element Var Size: 33
            Data Value
                Attribute List [count =  3] (Handsfree Audio Gateway)
                    Data Element: Sequence uint16 30 bytes
                        0011 0... = Data Element Type: Sequence (6)
                        .... .110 = Data Element Size: uint16 (6)
                        Data Element Var Size: 30
                        Data Value
                            Service Attribute: Service Class ID List (0x1), value = Handsfree Audio Gateway -> Generic Audio
                            Service Attribute: Bluetooth Profile Descriptor List (0x9), value = Handsfree 1.6
                            Service Attribute: (HFP AG) Supported Features (0x311), value = (Three Way Calling) (EC and/or Nr Function) (Voice Recognition Function) (Wide Band Speech) 
    Continuation State: no (00)

#HFP建立连接
#交换支持功能，AG和HF都不支持"HF indicator"
> 634	18.012482	SANYOEle_62:35:bb (DEVICE)	XiaomiCo_6e:38:c5 (PHONE)	HFP	26	Sent AT+BRSF=255
---------------------------------------------------------------------------
Bluetooth HFP Profile
    [Role: HS - Headset (2)]
    AT Stream: AT+BRSF=255\r
    Command 0: +BRSF
        Command Line Prefix: AT
        Command: +BRSF (Bluetooth Retrieve Supported Features)
        Type: Action Command (0x003d)
        Parameters
            HS supported features bitmask: 255
                .... .... .... .... .... .... .... ...1 = EC and/or NR function: True
                .... .... .... .... .... .... .... ..1. = Call waiting or 3-way calling: True
                .... .... .... .... .... .... .... .1.. = CLI Presentation: True
                .... .... .... .... .... .... .... 1... = Voice Recognition Activation: True
                .... .... .... .... .... .... ...1 .... = Remote Volume Control: True
                .... .... .... .... .... .... ..1. .... = Enhanced Call Status: True
                .... .... .... .... .... .... .1.. .... = Enhanced Call Control: True
                .... .... .... .... .... .... 1... .... = Codec Negotiation: True
                .... .... .... .... .... ...0 .... .... = HF Indicators: False
                .... .... .... .... .... ..0. .... .... = eSCO S4 (and T2) Settings Support: False
                0000 0000 0000 0000 0000 00.. .... .... = Reserved: 0x000000
> 641	18.022446	XiaomiCo_6e:38:c5 (PHONE)	SANYOEle_62:35:bb (DEVICE)	HFP	28	Rcvd   +BRSF: 871
---------------------------------------------------------------------------
Bluetooth HFP Profile
    [Role: AG - Audio Gate (1)]
    AT Stream: \r\n+BRSF: 871\r\n
    Command 0: +BRSF
        Command: +BRSF (Bluetooth Retrieve Supported Features)
        Type: Response (0x003a)
        Parameters
            AG supported features bitmask: 871
                .... .... .... .... .... .... .... ...1 = Three Way Calling: True
                .... .... .... .... .... .... .... ..1. = EC and/or NR function: True
                .... .... .... .... .... .... .... .1.. = Voice Recognition Function: True
                .... .... .... .... .... .... .... 0... = In-band Ring Tone: False
                .... .... .... .... .... .... ...0 .... = Attach Number to Voice Tag: False
                .... .... .... .... .... .... ..1. .... = Ability to Reject a Call: True
                .... .... .... .... .... .... .1.. .... = Enhanced Call Status: True
                .... .... .... .... .... .... 0... .... = Enhanced Call Control: False
                .... .... .... .... .... ...1 .... .... = Extended Error Result Codes: True
                .... .... .... .... .... ..1. .... .... = Codec Negotiation: True
                .... .... .... .... .... .0.. .... .... = HF Indicators: False
                .... .... .... .... .... 0... .... .... = eSCO S4 (and T2) Settings Support: False
                0000 0000 0000 0000 0000 .... .... .... = Reserved: 0x00000

#AG和HF都支持"Codec Negotiation"，HF告知AG其可用的Codec编码(CVSD)
> 643	18.023696	SANYOEle_62:35:bb (DEVICE)	XiaomiCo_6e:38:c5 (PHONE)	HFP	23	Sent AT+BAC=1
---------------------------------------------------------------------------
Bluetooth HFP Profile
    [Role: HS - Headset (2)]
    AT Stream: AT+BAC=1\r
    Command 0: +BAC
        Command Line Prefix: AT
        Command: +BAC (Bluetooth Available Codecs)
        Type: Action Command (0x003d)
        Parameters
            Codec: CVSD (1)

#获取AG支持的indicator以及它们的次序
> 649	18.029340	SANYOEle_62:35:bb (DEVICE)	XiaomiCo_6e:38:c5 (PHONE)	HFP	24	Sent AT+CIND=? 
> 652	18.035680	XiaomiCo_6e:38:c5 (PHONE)	SANYOEle_62:35:bb (DEVICE)	HFP	147	Rcvd   +CIND: ("CALL",(0,1)),("CALLSETUP",(0-3)),("SERVICE",(0-1)),("SIGNAL",(0-5)),("ROAM",(0,1)),("BATTCHG",(0-5)),("CALLHELD",(0-2)) 

#获取AG indicators当前的状态
> 654	18.037547	SANYOEle_62:35:bb (DEVICE)	XiaomiCo_6e:38:c5 (PHONE)	HFP	23	Sent AT+CIND? 
> 656	18.092460	XiaomiCo_6e:38:c5 (PHONE)	SANYOEle_62:35:bb (DEVICE)	HFP	38	Rcvd   +CIND: 0,0,1,5,0,2,0

#标志HFP连接建立
#启用"Indicator status update"
> 658	18.093591	SANYOEle_62:35:bb (DEVICE)	XiaomiCo_6e:38:c5 (PHONE)	HFP	30	Sent AT+CMER=3,0,0,1
> 660	18.117156	XiaomiCo_6e:38:c5 (PHONE)	SANYOEle_62:35:bb (DEVICE)	HFP	20	Rcvd   OK  
```

### HF拨打\挂断电话

```dtd
#HF拨打电话
> 921	29.433324	SANYOEle_62:35:bb (DEVICE)	XiaomiCo_6e:38:c5 (PHONE)	HFP	24	Sent ATD10086; 
---------------------------------------------------------------------------
Bluetooth HFP Profile
    [Role: HS - Headset (2)]
    AT Stream: ATD10086;\r
    Command 0: D
        Command Line Prefix: AT
        Command: D (Dial)
        [Expert Info (Warning/Protocol): Non mandatory type or command in this role]
        Parameters: No
> 924	30.778650	XiaomiCo_6e:38:c5 (PHONE)	SANYOEle_62:35:bb (DEVICE)	HFP	20	Rcvd   OK

#挂断电话
> 965	37.723080	SANYOEle_62:35:bb (DEVICE)	XiaomiCo_6e:38:c5 (PHONE)	HFP	22	Sent AT+CHUP
> 966	37.752000	XiaomiCo_6e:38:c5 (PHONE)	SANYOEle_62:35:bb (DEVICE)	HFP	20	Rcvd   OK
```

### HF获取通话电话列表

会HF会轮询此命令

```dtd
> 925	30.779082	SANYOEle_62:35:bb (DEVICE)	XiaomiCo_6e:38:c5 (PHONE)	HFP	22	Sent AT+CLCC 
> 929	30.820871	XiaomiCo_6e:38:c5 (PHONE)	SANYOEle_62:35:bb (DEVICE)	HFP	46	Rcvd   +CLCC: 1,0,2,0,0,"10086",129 
---------------------------------------------------------------------------
Bluetooth HFP Profile
    [Role: AG - Audio Gate (1)]
    AT Stream: \r\n+CLCC: 1,0,2,0,0,"10086",129\r\n
    Command 0: +CLCC
        Command: +CLCC (Current Calls)
        Type: Response (0x003a)
        Parameters
            ID: 1
            Direction: Mobile Originated (0)
            State: Dialing (2)
            Mode: Voice (0)
            Mpty: Call is not one of multiparty (conference) call parties (0)
            Number: "10086"
            Type: The phone number format may be a national or international format, and may contain prefix and/or escape digits. No changes on the number presentation are required. (129)
```

### AG下发通话状态

在开启`"Indicator status update"`(AT+CMER)之后，AG会主动下发当前的通话状态，`Indicator Index`对应+CIND返回的参数列表，`Indicator <idx>`对应其值。

#### 拨打电话

```dtd
#CALLSETUP=2,拨号中
> 1056	66.689969	XiaomiCo_6e:38:c5 (PHONE)	SANYOEle_62:35:bb (DEVICE)	HFP	27	Rcvd   +CIEV: 2,2  
```

#### 无通话或者挂断

```dtd
#CALL=0,无通话
1106	74.975552	XiaomiCo_6e:38:c5 (PHONE)	SANYOEle_62:35:bb (DEVICE)	HFP	27	Rcvd   +CIEV: 1,0  
```

#### 来电

```dtd
#CALLSETUP=1,来电中
> 1056	66.689969	XiaomiCo_6e:38:c5 (PHONE)	SANYOEle_62:35:bb (DEVICE)	HFP	27	Rcvd   +CIEV: 2,1 
```
### 建立SCO/eSCO连接

蓝牙电话的音频数据由SCO/eSCO协议传输，在HCI中无数据相关的日志。

```dtd
#HF选择编解码格式
> 900	387.320548	XiaomiCo_6e:38:c5 (PHONE)	localhost ()	HFP	24	Rcvd   +BCS: 1  
---------------------------------------------------------------------------
Bluetooth HFP Profile
    [Role: AG - Audio Gate (1)]
    AT Stream: \r\n+BCS: 1\r\n
    Command 0: +BCS
        Command: +BCS (Bluetooth Codec Selection)
        Type: Response (0x003a)
        Parameters
            Codec: CVSD (1)
> 903	387.380459	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	HFP	23	Sent AT+BCS=1 
---------------------------------------------------------------------------
Bluetooth HFP Profile
    [Role: HS - Headset (2)]
    AT Stream: AT+BCS=1\r
    Command 0: +BCS
        Command Line Prefix: AT
        Command: +BCS (Bluetooth Codec Selection)
        Type: Action Command (0x003d)
        Parameters
            Codec: CVSD (1)

#AG请求建立eSCO连接
> 906	387.530834	controller	host	HCI_EVT	13	Rcvd Connect Request
---------------------------------------------------------------------------
Bluetooth HCI Event - Connect Request
    Event Code: Connect Request (0x04)
    Parameter Total Length: 10
    BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
    Class of Device: 0x001f00 (Uncategorized: device code not specified:Unknown - services:)
    Link Type: eSCO connection (Voice Channels) (0x02)

#HF允许建立eSCO连接
> 907	387.531078	host	controller	HCI_CMD	67	Sent Enhanced Accept Synchronous Connection Request
---------------------------------------------------------------------------
Bluetooth HCI Command - Enhanced Accept Synchronous Connection Request
    Command Opcode: Enhanced Accept Synchronous Connection Request (0x043e)
        0000 01.. .... .... = Opcode Group Field: Link Control Commands (0x01)
        .... ..00 0011 1110 = Opcode Command Field: Enhanced Accept Synchronous Connection Request (0x03e)
    Parameter Total Length: 63
    BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
    Tx Bandwidth (bytes/s): 8000
    Rx Bandwidth (bytes/s): 8000
    Transmit Coding Format
    Receive Coding Format
    Transmit Codec Frame Size: 60
    Receive Codec Frame Size: 60
    Input Bandwidth: 16000
    Output Bandwidth: 16000
    Input Coding Format
    Output Coding Format
    Input Coded Data Size: 16
    Output Coded Data Size: 16
    Input PCM Data Format: 2's complement (0x02)
    Output PCM Data Format: 2's complement (0x02)
    Input PCM Sample Payload MSB Position: 0
    Output PCM Sample Payload MSB Position: 0
    Input Data Path: Vendor Specific
    Output Data Path: Vendor Specific
    Input Transport Unit Size: 16
    Output Transport Unit Size: 16
    Max. Latency (ms): 12
    Packet Type: 0x03bf, 3-EV5 may NOT be used, 2-EV5 may NOT be used, 3-EV3 may NOT be used, EV5 may be used, EV4 may be used, EV3 may be used, HV3 may be used, HV2 may be used, HV1 may be used
    Retransmission Effort: At least 1 retransmission, optimize for link quality (2)
    [Pending in frame: 909]
    [Command-Pending Delta: 1.38ms]
    [Response in frame: 910]
    [Command-Response Delta: 3.862ms]

#eSCO连接成功
> 910	387.534940	controller	host	HCI_EVT	20	Rcvd Synchronous Connection Complete
---------------------------------------------------------------------------
Bluetooth HCI Event - Synchronous Connection Complete
    Event Code: Synchronous Connection Complete (0x2c)
    Parameter Total Length: 17
    Status: Success (0x00)
    Connection Handle: 0x0e00
    BD_ADDR: XiaomiCo_6e:38:c5 (00:00:00:00:00:00)
    Link Type: eSCO connection (0x02)
    Transmit Interval: 12 slots (7.5 msec)
    Retransmit Window: 2 slots (1.25 msec)
    Rx Packet Length: 60
    Tx Packet Length: 60
    Air Mode: CVSD (2)
    [Command in frame: 907]
    [Pending in frame: 909]
    [Pending-Response Delta: 2.482ms]
    [Command-Response Delta: 3.862ms]
```

# 蓝牙音乐

## AVDTP

**AVDTP：AUDIO/VIDEO DISTRIBUTION TRANSPORT PROTOCOL(音视频分配传输协议)** ，用于音乐数据流的建立与传输控制。

### 建立连接

```dtd
> 458	11.574257	SANYOEle_62:35:bb (DEVICE)	XiaomiCo_6e:38:c5 (PHONE)	L2CAP	17	Sent Connection Request (AVDTP, SCID: 0x0042)
```

### 数据流传输

数据流的生命状态：

![](https://s1.ax1x.com/2022/04/11/LV6NVA.png)

#### 查找(Idle)

```dtd
#查询AG的所有音源
> 468	11.652772	SANYOEle_62:35:bb (DEVICE)	XiaomiCo_6e:38:c5 (PHONE)	AVDTP	11	Sent Command - Discover
> 469	11.722467	XiaomiCo_6e:38:c5 (PHONE)	SANYOEle_62:35:bb (DEVICE)	AVDTP	15	Rcvd ResponseAccept - Discover - items: 2
---------------------------------------------------------------------------
Bluetooth AVDTP Protocol
    Signal: Discover (ResponseAccept)
    ACP SEP [1 - Audio Source] item 1/2
        0000 01.. = SEID: 1
        .... ..0. = In Use: False (0x0)
        .... ...0 = RFA0: 0x0
        0000 .... = Media Type: Audio (0x0)
        .... 0... = Type: Source (0x0)
        .... .000 = RFA1: 0x0
    ACP SEP [2 - Audio Source] item 2/2
        0000 10.. = SEID: 2
        .... ..0. = In Use: False (0x0)
        .... ...0 = RFA0: 0x0
        0000 .... = Media Type: Audio (0x0)
        .... 0... = Type: Source (0x0)
        .... .000 = RFA1: 0x0

#获取音源详细信息
> 470	11.722896	SANYOEle_62:35:bb (DEVICE)	XiaomiCo_6e:38:c5 (PHONE)	AVDTP	12	Sent Command - GetAllCapabilities - ACP SEID [1 - Audio Source]
> 473	11.728705	XiaomiCo_6e:38:c5 (PHONE)	SANYOEle_62:35:bb (DEVICE)	AVDTP	23	Rcvd ResponseAccept - GetAllCapabilities - Audio SBC (44100 | Mono JointStereo | block: 4 8 12 16 | subbands: 8 | allocation: Loudness | bitpool: 2..53)
> 474	11.728960	SANYOEle_62:35:bb (DEVICE)	XiaomiCo_6e:38:c5 (PHONE)	AVDTP	12	Sent Command - GetAllCapabilities - ACP SEID [2 - Audio Source]
> 476	11.734811	XiaomiCo_6e:38:c5 (PHONE)	SANYOEle_62:35:bb (DEVICE)	AVDTP	25	Rcvd ResponseAccept - GetAllCapabilities - Audio MPEG-2,4 AAC
---------------------------------------------------------------------------
Bluetooth AVDTP Protocol
    Signal: GetAllCapabilities (ResponseAccept)
        0010 .... = Transaction: 0x2
        .... 00.. = Packet Type: Single (0x0)
        .... ..10 = Message Type: ResponseAccept (0x2)
        00.. .... = RFA: 0x0
        ..00 1100 = Signal: GetAllCapabilities (0x0c)
    Capabilities
        Service: Media Transport
        Service: Media Codec - Audio MPEG-2,4 AAC
        Service: Delay Reporting
```

#### 设置/打开(Configured/Open)

```dtd
#选择对应音源
> 477	11.735508	SANYOEle_62:35:bb (DEVICE)	XiaomiCo_6e:38:c5 (PHONE)	AVDTP	25	Sent Command - SetConfiguration - ACP SEID [2 - Audio Source] - INT SEID [2 - Audio Source] - Audio MPEG-2,4 AAC
> 479	11.742210	XiaomiCo_6e:38:c5 (PHONE)	SANYOEle_62:35:bb (DEVICE)	AVDTP	11	Rcvd ResponseAccept - SetConfiguration

#打开音源
> 480	11.742538	SANYOEle_62:35:bb (DEVICE)	XiaomiCo_6e:38:c5 (PHONE)	AVDTP	12	Sent Command - Open - ACP SEID [2 - Audio Source]
---------------------------------------------------------------------------
Bluetooth AVDTP Protocol
    Signal: Open (Command)
    ACP SEID [2 - Audio Source]
> 485	11.747312	XiaomiCo_6e:38:c5 (PHONE)	SANYOEle_62:35:bb (DEVICE)	AVDTP	11	Rcvd ResponseAccept - Open
```

#### 开始传输(Streaming)

```dtd
> 601	28.945727	XiaomiCo_f2:31:c7 (PHONE)	SANYOEle_71:7b:5c (DEVICE)	AVDTP	12	Rcvd Command - Start - ACP SEID [2 - Audio Sink]
> 602	28.946025	SANYOEle_71:7b:5c (DEVICE)	XiaomiCo_f2:31:c7 (PHONE)	AVDTP	11	Sent ResponseAccept - Start
```

#### 传输(Streaming)

使用RTP协议传输数据，AVDTP协议本身不处理数据流。

```dtd
> 612	29.166322	XiaomiCo_f2:31:c7 (PHONE)	SANYOEle_71:7b:5c (DEVICE)	RTP	37	PT=Unknown, SSRC=0x2, Seq=1, Time=0
---------------------------------------------------------------------------
Frame 612: 37 bytes on wire (296 bits), 37 bytes captured (296 bits)
Bluetooth
Bluetooth HCI H4
Bluetooth HCI ACL Packet
Bluetooth L2CAP Protocol
Bluetooth A2DP Profile
    [ACP SEID: 2]
    [INT SEID: 2]
    [Codec: MPEG-2,4 AAC (0x02)]
    [Stream Start in Frame: 612]
    [Stream End in Frame: 22786]
    [Stream Number: 1]
Real-Time Transport Protocol
    [Stream setup by BT A2DP (frame 612)]
    10.. .... = Version: RFC 1889 Version (2)
    ..0. .... = Padding: False
    ...0 .... = Extension: False
    .... 0000 = Contributing source identifiers count: 0
    0... .... = Marker: False
    Payload type: Unknown (98)
    Sequence number: 1
    [Extended sequence number: 65537]
    Timestamp: 0
    Synchronization Source identifier: 0x00000002 (2)
Data (16 bytes)
    Data: 47fc0000b0908003000621100520a41c
    [Length: 16]

```

#### 暂停(Streaming)

```dtd
> 773	23.510436	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	AVDTP	11	Sent ResponseAccept - Suspend
```

#### 释放(Closing)

```dtd
> 5692	96.414176	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	AVDTP	12	Sent Command - Close - ACP SEID [2 - Audio Source]
```

## AVCTP

**AVCTP： Audio/Video Control Transport Protocol(音频/视频控制传输协议)**，用于蓝牙设备间音乐数据流功能的发现与控制，但是控制的具体功能逻辑不由AVCTP处理，其只发送命令。

发现由SDP协议完成，然后发起L2CAP通道连接蓝牙设备之间的AVCTP服务，如下:

```dtd
#发现
> 134	0.964749	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	SDP	33	Sent Service Search Attribute Request : Audio Source: [Service Class ID List 0x0001] [Protocol Descriptor List 0x0004] [Bluetooth Profile Descriptor List 0x0009] 
> 139	0.981837	XiaomiCo_6e:38:c5 (PHONE)	localhost ()	SDP	64	Rcvd Service Search Attribute Response 
---------------------------------------------------------------------------
Bluetooth SDP Protocol
    PDU: Service Search Attribute Response (0x07)
    Transaction Id: 0x0000
    Parameter Length: 50
    Attribute List Byte Count: 47
    Attribute Lists [count =  1]
        Data Element: Sequence uint8 45 bytes
            0011 0... = Data Element Type: Sequence (6)
            .... .101 = Data Element Size: uint8 (5)
            Data Element Var Size: 45
            Data Value
                Attribute List [count =  3] (Audio Source)
                    Data Element: Sequence uint16 42 bytes
                        0011 0... = Data Element Type: Sequence (6)
                        .... .110 = Data Element Size: uint16 (6)
                        Data Element Var Size: 42
                        Data Value
                            Service Attribute: Service Class ID List (0x1), value = Audio Source
                                Attribute ID: Service Class ID List
                                Value
                            Service Attribute: Protocol Descriptor List (0x4), value = L2CAP:25 -> AVDTP (1.3)
                                Attribute ID: Protocol Descriptor List
                                Value
                            Service Attribute: Bluetooth Profile Descriptor List (0x9), value = Advanced Audio Distribution 1.3
                                Attribute ID: Bluetooth Profile Descriptor List
                                Value
    Continuation State: no (00)


#连接
> 147	1.006617	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	L2CAP	17	Sent Connection Request (AVDTP, SCID: 0x0043)
> 155	1.017317	XiaomiCo_6e:38:c5 (PHONE)	localhost ()	L2CAP	21	Rcvd Connection Response - Success (SCID: 0x0043, DCID: 0x0042)
```

## AVRCP

**AVRCP：Audio/Video Remote Control Profile(音频/视频远程控制配置文件)**，用于控制蓝牙音乐的标准接口。

在AVRCP中，**TG**表示播放设备，**CT**表示控制设备。

### 命令格式

命令分为两个类型：VENDOR DEPENDENT 与PASS THROUGH，

- VENDOR DEPENDENT：通过`Ctype`与`PDU ID`来区分功能，`Ctype`有**Control, Status , Notify**,ACCEPTED,REJECTED, CHANGED,INTERIM,IMPLEMENTED,STABLE等。

- PASS THROUGH：是播放控制命令，`Ctype`只有**Control**。

其中请求命令`Ctype`为**Notify**，回复事件`Ctype`为**Interim**，事件通知`Ctype`为**Changed**。

### 获取TG播放器支持功能

`PDU ID`：GetFolderItems，通过`Scope`来区分功能，用于**TG**可以播放器列表与每个播放器支持的功能。

```dtd
> 307	1.551666	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	AVRCP	29	Sent Browsing: Command - GetFolderItems - Scope: MediaPlayerList, StartItem: 0x0000, EndItem: 0x0013
---------------------------------------------------------------------------
Bluetooth AVRCP Profile
    PDU ID: GetFolderItems (0x71)
    Parameter Length: 10
    Scope: MediaPlayerList (0x00)
    StartItem: 0
    EndItem: 19
    Attribute Count: All attributes are requested (0)
    Attribute List

> 326	1.651062	XiaomiCo_6e:38:c5 (PHONE)	localhost ()	AVRCP	67	Rcvd Browsing: OK - GetFolderItems - UidCounter: 0x0000, NumberOfItems: 1
---------------------------------------------------------------------------
Bluetooth AVRCP Profile
    PDU ID: GetFolderItems (0x71)
    Parameter Length: 48
    Status: OK (0x04)
    UID Counter: 0x0000
    Number Of Items: 1
    Player: Music Player
        Item Type: Media Player Item (0x01)
        Item Length: 40
        Player ID: 0
        Major Player Type: Unknown (0x01)
        Player SubType: Audio Book (0x00000001)
        Play Status: Stopped (0x00)
        Features
            Not Used Features
            .... ...1 = PASSTHROUGH Play: 0x1
            .... ..1. = PASSTHROUGH Stop: 0x1
            .... .1.. = PASSTHROUGH Pause: 0x1
            ...1 .... = PASSTHROUGH Rewind: 0x1
            ..1. .... = PASSTHROUGH FastForward: 0x1
            1... .... = PASSTHROUGH Forward: 0x1
            .... ...1 = PASSTHROUGH Backward: 0x1
            .... .1.. = Advanced Control Player: 0x1
        Character Set: UTF-8 (106)
        Displayable Name Length: 12
        Displayable Name: Music Player
```

### 获取TG支持事件

`PDU ID`：GetCapabilities，用于**TG**可以支持的事件。

```dtd
> 288	1.233497	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	AVRCP	23	Sent Vendor dependent: Status - GetCapabilities(Events Supported)
> 299	1.252688	XiaomiCo_6e:38:c5 (PHONE)	localhost ()	AVRCP	30	Rcvd Vendor dependent: Stable - GetCapabilities(Events Supported) - Count: 6
---------------------------------------------------------------------------
Bluetooth AVRCP Profile
    0000 .... = Reserved: 0x0
    .... 1100 = Ctype: Stable (0xc)
    0100 1... = Subunit Type: Panel (0x09)
    .... .000 = Subunit ID: 0x0
    Opcode: Vendor dependent (0x00)
    Company ID: 00:19:58 (Bluetooth SIG, Inc.)
    PDU ID: GetCapabilities (0x10)
    0000 00.. = RFA: 0x00
    .... ..00 = Packet Type: Single (0x0)
    Parameter Length: 8
    Capability: Events Supported (0x03)
    Capability Count: 0x06
    Event ID: PlaybackStatusChanged (0x01)
    Event ID: TrackChanged (0x02)
    Event ID: PlaybackPositionChanged (0x05)
    Event ID: AvailablePlayersChanged (0x0a)
    Event ID: AddressedPlayerChanged (0x0b)
    Event ID: NowPlayingContentChanged (0x09)
    [Response Time: 19/1000ms]
    [Command in frame: 288]
```

### 播放器状态

**CT**端获取**TC**端播放状态，`Play Status`的参数由`GetFolderItems`原语获取，通过`Ctype`来区分操作。

#### 注册事件

`PDU ID`：RegisterNotification，`Event ID`为: PlaybackStatusChanged(`GetCapabilities`原语获取)。

```dtd
> 300	1.252942	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	AVRCP	27	Sent Vendor dependent: Notify - RegisterNotification - PlaybackStatusChanged
---------------------------------------------------------------------------
Bluetooth AVRCP Profile
    0000 .... = Reserved: 0x0
    .... 0011 = Ctype: Notify (0x3)
    0100 1... = Subunit Type: Panel (0x09)
    .... .000 = Subunit ID: 0x0
    Opcode: Vendor dependent (0x00)
    Company ID: 00:19:58 (Bluetooth SIG, Inc.)
    PDU ID: RegisterNotification (0x31)
    0000 00.. = RFA: 0x00
    .... ..00 = Packet Type: Single (0x0)
    Parameter Length: 5
    Event ID: PlaybackStatusChanged (0x01)
    Interval: 0
    [Response Time: 107/1000ms]
    [Response in frame: 324]

> 324	1.649874	XiaomiCo_6e:38:c5 (PHONE)	localhost ()	AVRCP	24	Rcvd Vendor dependent: Interim - RegisterNotification - PlaybackStatusChanged - PlayStatus: Paused
---------------------------------------------------------------------------
Bluetooth AVRCP Profile
    0000 .... = Reserved: 0x0
    .... 1111 = Ctype: Interim (0xf)
    0100 1... = Subunit Type: Panel (0x09)
    .... .000 = Subunit ID: 0x0
    Opcode: Vendor dependent (0x00)
    Company ID: 00:19:58 (Bluetooth SIG, Inc.)
    PDU ID: RegisterNotification (0x31)
    0000 00.. = RFA: 0x00
    .... ..00 = Packet Type: Single (0x0)
    Parameter Length: 2
    Event ID: PlaybackStatusChanged (0x01)
    Play Status: Paused (0x02)
    [Response Time: 107/1000ms]
    [Command in frame: 300]
```

#### 事件通知

`PDU ID`：RegisterNotification，`Event ID`为: PlaybackStatusChanged，通过`Ctype`来区分功能。

```dtd
> 372	2.461590	XiaomiCo_6e:38:c5 (PHONE)	localhost ()	AVRCP	24	Rcvd Vendor dependent: Changed - RegisterNotification - PlaybackStatusChanged - PlayStatus: Stopped
---------------------------------------------------------------------------
Bluetooth AVRCP Profile
    0000 .... = Reserved: 0x0
    .... 1101 = Ctype: Changed (0xd)
    0100 1... = Subunit Type: Panel (0x09)
    .... .000 = Subunit ID: 0x0
    Opcode: Vendor dependent (0x00)
    Company ID: 00:19:58 (Bluetooth SIG, Inc.)
    PDU ID: RegisterNotification (0x31)
    0000 00.. = RFA: 0x00
    .... ..00 = Packet Type: Single (0x0)
    Parameter Length: 2
    Event ID: PlaybackStatusChanged (0x01)
    Play Status: Stopped (0x00)
    [Response Time: 107/1000ms]
    [Command in frame: 300]
```

#### 主动获取

`PDU ID`：GetPlayStatus，往往与`RegisterNotification-TrackChanged`命令配合使用。

```dtd
> 466	18.051244	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	AVRCP	22	Sent Vendor dependent: Status - GetPlayStatus
---------------------------------------------------------------------------
Bluetooth AVRCP Profile
    0000 .... = Reserved: 0x0
    .... 0001 = Ctype: Status (0x1)
    0100 1... = Subunit Type: Panel (0x09)
    .... .000 = Subunit ID: 0x0
    Opcode: Vendor dependent (0x00)
    Company ID: 00:19:58 (Bluetooth SIG, Inc.)
    PDU ID: GetPlayStatus (0x30)
    0000 00.. = RFA: 0x00
    .... ..00 = Packet Type: Single (0x0)
    Parameter Length: 0
    [Response Time: 40/1000ms]
    [Response in frame: 478]

> 478	18.091970	XiaomiCo_6e:38:c5 (PHONE)	localhost ()	AVRCP	31	Rcvd Vendor dependent: Stable - GetPlayStatus PlayStatus: Playing, SongPosition: 6118ms, SongLength: 175000ms
---------------------------------------------------------------------------
Bluetooth AVRCP Profile
    0000 .... = Reserved: 0x0
    .... 1100 = Ctype: Stable (0xc)
    0100 1... = Subunit Type: Panel (0x09)
    .... .000 = Subunit ID: 0x0
    Opcode: Vendor dependent (0x00)
    Company ID: 00:19:58 (Bluetooth SIG, Inc.)
    PDU ID: GetPlayStatus (0x30)
    0000 00.. = RFA: 0x00
    .... ..00 = Packet Type: Single (0x0)
    Parameter Length: 9
    Song Length: 175000
    Song Position: 6118
    Play Status: Playing (0x01)
    [Response Time: 40/1000ms]
    [Command in frame: 466]
```

### 播放进度

使用`PDU ID`：GetPlayStatus，获取**TG**端播放进度，**CT**端轮循调用此命令。其中`Song Length`参数为音乐长度，`Song Position`为当前播放进度。日志参考`主动获取播放器`章节。

### 音轨状态

**CT**端获取**TC**端播放器音轨状态，状态有下面两种：

- 无音轨：`Identifier: 0xffffffffffffffff (NOT SELECTED)`
- 有音轨：`Identifier: 0x0000000000000000 (SELECTED)`

**音轨状态事件**会在每次开始播放都会有事件，歌词就是通过发送**音轨状态事件**来完成的。

#### 注册事件

`PDU ID`：RegisterNotification，`Event ID`为: TrackChanged(`GetCapabilities`原语获取)。

```dtd
> 323	1.331254	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	AVRCP	27	Sent Vendor dependent: Notify - RegisterNotification - TrackChanged
---------------------------------------------------------------------------
Bluetooth AVRCP Profile
    0000 .... = Reserved: 0x0
    .... 0011 = Ctype: Notify (0x3)
    0100 1... = Subunit Type: Panel (0x09)
    .... .000 = Subunit ID: 0x0
    Opcode: Vendor dependent (0x00)
    Company ID: 00:19:58 (Bluetooth SIG, Inc.)
    PDU ID: RegisterNotification (0x31)
    0000 00.. = RFA: 0x00
    .... ..00 = Packet Type: Single (0x0)
    Parameter Length: 5
    Event ID: TrackChanged (0x02)
    Interval: 0
    [Response Time: 35/1000ms]
    [Response in frame: 330]

> 330	1.366518	XiaomiCo_6e:38:c5 (PHONE)	localhost ()	AVRCP	31	Rcvd Vendor dependent: Interim - RegisterNotification - TrackChanged - 0xFFFFFFFFFFFFFFFF (NOT SELECTED)
---------------------------------------------------------------------------
Bluetooth AVRCP Profile
    0000 .... = Reserved: 0x0
    .... 1111 = Ctype: Interim (0xf)
    0100 1... = Subunit Type: Panel (0x09)
    .... .000 = Subunit ID: 0x0
    Opcode: Vendor dependent (0x00)
    Company ID: 00:19:58 (Bluetooth SIG, Inc.)
    PDU ID: RegisterNotification (0x31)
    0000 00.. = RFA: 0x00
    .... ..00 = Packet Type: Single (0x0)
    Parameter Length: 9
    Event ID: TrackChanged (0x02)
    Identifier: 0xffffffffffffffff (NOT SELECTED)
    [Response Time: 35/1000ms]
    [Command in frame: 323]
```

#### 事件通知

`PDU ID`：RegisterNotification，`Event ID`为: TrackChanged，通过`Ctype`来区分功能。

```dtd
> 451	18.026715	XiaomiCo_6e:38:c5 (PHONE)	localhost ()	AVRCP	31	Rcvd Vendor dependent: Changed - RegisterNotification - TrackChanged - 0x0000000000000000 (SELECTED)
```

### 获取播放音轨的属性

获取当前音轨，播放音乐的属性，如标题、歌手等信息。`Attribute List`请查看[AVRCP协议规范](参考)第26章节，往往与`RegisterNotification-TrackChanged`命令配合使用。

```dtd
> 465	18.050915	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	AVRCP	59	Sent Vendor dependent: Status - GetElementAttributes - 0x0000000000000000 (PLAYING)
---------------------------------------------------------------------------
Bluetooth AVRCP Profile
    0000 .... = Reserved: 0x0
    .... 0001 = Ctype: Status (0x1)
    0100 1... = Subunit Type: Panel (0x09)
    .... .000 = Subunit ID: 0x0
    Opcode: Vendor dependent (0x00)
    Company ID: 00:19:58 (Bluetooth SIG, Inc.)
    PDU ID: GetElementAttributes (0x20)
    0000 00.. = RFA: 0x00
    .... ..00 = Packet Type: Single (0x0)
    Parameter Length: 37
    Identifier: 0x0000000000000000
    Number of Attributes: 7
    Attribute List
        Attribute ID: Title (0x00000001)
        Attribute ID: Artist (0x00000002)
        Attribute ID: Album (0x00000003)
        Attribute ID: Media Number (0x00000004)
        Attribute ID: Total Number of Media (0x00000005)
        Attribute ID: Genre (0x00000006)
        Attribute ID: Playing Time (0x00000007)
    [Response Time: 3/1000ms]
    [Response in frame: 469]

> 469	18.054345	XiaomiCo_6e:38:c5 (PHONE)	localhost ()	AVRCP	127	Rcvd Vendor dependent: Stable - GetElementAttributes - Title: "Demons"
---------------------------------------------------------------------------
Bluetooth AVRCP Profile
    0000 .... = Reserved: 0x0
    .... 1100 = Ctype: Stable (0xc)
    0100 1... = Subunit Type: Panel (0x09)
    .... .000 = Subunit ID: 0x0
    Opcode: Vendor dependent (0x00)
    Company ID: 00:19:58 (Bluetooth SIG, Inc.)
    PDU ID: GetElementAttributes (0x20)
    0000 00.. = RFA: 0x00
    .... ..00 = Packet Type: Single (0x0)
    Parameter Length: 105
    Number of Attributes: 7
    Attribute Entries
        Attribute [                Title]: Demons
        Attribute [               Artist]: Imagine Dragons
        Attribute [                Album]: Continued Silence
        Attribute [         Media Number]: 10
        Attribute [Total Number of Media]: 18
        Attribute [                Genre]: 
        Attribute [         Playing Time]: 175000
    [Response Time: 3/1000ms]
    [Command in frame: 465]
```

### 播放器控制

**CT**控制**TG**播放器的相关命令，都是PASS THROUGH类型，通过`Operation ID`来区分功能，通过`Ctype`来区分操作。

#### 播放

```dtd
> 298	1.252592	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	AVRCP	17	Sent Pass Through: Control - PLAY (Pushed)
---------------------------------------------------------------------------
Bluetooth AVRCP Profile
    0000 .... = Reserved: 0x0
    .... 0000 = Ctype: Control (0x0)
    0100 1... = Subunit Type: Panel (0x09)
    .... .000 = Subunit ID: 0x0
    Opcode: Pass Through (0x7c)
    0... .... = State: Pushed (0x0)
    .100 0100 = Operation ID: PLAY (0x44)
    Data Length: 0x00
    [Response Time: 14/200ms]
    [Response in frame: 308]

>308	1.266992	XiaomiCo_6e:38:c5 (PHONE)	localhost ()	AVRCP	17	Rcvd Pass Through: Accepted - PLAY (Pushed)
```

#### 暂停

```dtd
> 430	7.424245	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	AVRCP	17	Sent Pass Through: Control - PAUSE (Pushed)
---------------------------------------------------------------------------
Bluetooth AVRCP Profile
    0000 .... = Reserved: 0x0
    .... 0000 = Ctype: Control (0x0)
    0100 1... = Subunit Type: Panel (0x09)
    .... .000 = Subunit ID: 0x0
    Opcode: Pass Through (0x7c)
    0... .... = State: Pushed (0x0)
    .100 0110 = Operation ID: PAUSE (0x46)
    Data Length: 0x00
    [Response Time: 70/200ms]
    [Response in frame: 434]

> 434	7.494377	XiaomiCo_6e:38:c5 (PHONE)	localhost ()	AVRCP	17	Rcvd Pass Through: Accepted - PAUSE (Pushed)
```

#### 下一曲

```dtd
> 1163	40.755874	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	AVRCP	17	Sent Pass Through: Control - BACKWARD (Pushed)
---------------------------------------------------------------------------
Bluetooth AVRCP Profile
    0000 .... = Reserved: 0x0
    .... 0000 = Ctype: Control (0x0)
    0100 1... = Subunit Type: Panel (0x09)
    .... .000 = Subunit ID: 0x0
    Opcode: Pass Through (0x7c)
    0... .... = State: Pushed (0x0)
    .100 1100 = Operation ID: BACKWARD (0x4c)
    Data Length: 0x00
    [Response Time: 15/200ms]
    [Response in frame: 1168]

> 1168	40.771498	XiaomiCo_6e:38:c5 (PHONE)	localhost ()	AVRCP	17	Rcvd Pass Through: Accepted - BACKWARD (Pushed)
```

#### 上一曲

```dtd
> 969	37.736799	localhost ()	XiaomiCo_6e:38:c5 (PHONE)	AVRCP	17	Sent Pass Through: Control - FORWARD (Pushed)
---------------------------------------------------------------------------
Bluetooth AVRCP Profile
    0000 .... = Reserved: 0x0
    .... 0000 = Ctype: Control (0x0)
    0100 1... = Subunit Type: Panel (0x09)
    .... .000 = Subunit ID: 0x0
    Opcode: Pass Through (0x7c)
    0... .... = State: Pushed (0x0)
    .100 1011 = Operation ID: FORWARD (0x4b)
    Data Length: 0x00
    [Response Time: 25/200ms]
    [Response in frame: 976]

> 976	37.762691	XiaomiCo_6e:38:c5 (PHONE)	localhost ()	AVRCP	17	Rcvd Pass Through: Accepted - FORWARD (Pushed)
```

# 参考
- 蓝牙核心协议规范[Core_v4.2.pdf](https://www.bluetooth.org/DocMan/handlers/DownloadDoc.ashx?doc_id=286439)
- PBAP协议规范[PBAP_v1.2.3.pdf](https://www.bluetooth.com/specifications/specs/phone-book-access-profile-1-2-3/)
- HFP协议规范[HFP_v1.8.pdf](https://www.bluetooth.com/specifications/specs/hands-free-profile-1-8/)
- AVDTP协议规范[AVDTP_SPEC_V13.pdf](https://www.bluetooth.com/specifications/specs/a-v-distribution-transport-protocol-1-3/)
- AVCTP协议规范[AVCTP_SPEC_V14.pdf](https://www.bluetooth.com/specifications/specs/a-v-%e2%80%8b%e2%80%8bcontrol-transport-protocol-1-4/)
- AVRCP协议规范[AVRCP_v1.6.2.pdf](https://www.bluetooth.com/specifications/specs/a-v-remote-control-profile-1-6-2/)
- [蓝牙物理链路类型：SCO和ACL链路与A2DP](https://blog.csdn.net/wenzongliang/article/details/84689377)
- [安全简单配对](https://blog.csdn.net/XiaoXiaoPengBo/article/details/107792407)
- [在HCI层看蓝牙的连接过程](https://blog.csdn.net/u010657219/article/details/42192481)
- [蓝牙SDP协议概述 ](https://www.cnblogs.com/libs-liu/p/9498952.html)
- [RFCOMM协议概念介绍](https://blog.csdn.net/XiaoXiaoPengBo/article/details/108125755)
- [蓝牙的OBEX协议](https://blog.51cto.com/u_2982693/3521456)
- [蓝牙电话之PBAP协议分析](https://blog.csdn.net/weixin_44260005/article/details/105223139)
- [蓝牙之八-HFP](https://blog.csdn.net/shichaog/article/details/52123439)
- [蓝牙之九-AT命令](https://blog.csdn.net/shichaog/article/details/52126415)

