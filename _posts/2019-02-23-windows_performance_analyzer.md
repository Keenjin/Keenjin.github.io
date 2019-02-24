---
layout: post
title: Windows Performance Analyzer使用
date: 2019-02-23
tags: 调试 性能
---

### 概述

基本过程：WPA是辅助WPR使用，WPR（Windows Performance Record）用来录制ETL（包括CPU使用率、IO、文件、网络、GPU、堆等），借助ETW技术框架实现，WPA用来可视化分析ETL文件。  
基本使用方法：拖拽、右键选择  
基本展示内容：各种维度的统计数据，可选择总和、平均、最大、最小等各种统计方法，也可选择UI延迟、CPU占用、文件读写耗时等各种维度数据  

一个特点：灵活，丰富的内容需要根据各种不同场景灵活选择处理，以便尽可能快的定位问题  

![jpg](/images/post/wpt/1.jpg)

### ETW技术架构

![png](/images/post/wpt/etdiag2.png)

```txt
Controllers：启动和停止Events发送，以及log路径和大小设置，以及Provider的选择允许。WPR（Windows Performance Record）就是一个Controller。它会使用StartTrace等API

Providers：不同类型的Event产生者。例如线程、网络、IO、内存等各种Provider。它会使用WriteEvent

Consumers：使用Event的使用者。WPA就是使用者，用来做可视化分析。它会解析log file
```

### Data Table

灵活处理用各种不同维度做主key，来统计数据，需要了解两根线条（黄线蓝线）的含义

![png](/images/post/wpt/3VD_2DataTable.png)  

```txt
Key Area：用这一组维度数据作为统计关键字，通常需要正确搭配，例如Process可以搭配Thread

Data Area：数据统计区，这里可以是消耗时间cost time、次数等

Graphing element Area：绘制区，这里可以是时间间隔during time、权重占比Weight等
```

### WPA使用步骤

```txt
Step1：打开ETL文件
Step2：选择要分析的图形属性，例如是要分析CPU占用？磁盘存储？界面卡顿UI Delay？网络？内存使用？
Step3：选择一个时间区间（通常是肉眼观察到的数据波形有问题的区间段）
Step4：放大Zoom
Step5：定制要分析观测的数据Data Table维度
```

### 各种Event Provider一览

通过System Configuration - Trace Statistic，可以看到有非常多的各种不同的Provider

<details>
<summary>Providers列表</summary>

| Line # | Provider Name                                  | Event Name                                                                                     | Count   |
|--------|------------------------------------------------|------------------------------------------------------------------------------------------------|---------|
| 1      | d00792da-07b7-40f5-97eb-5d974e054740           | <Unknown>                                                                                      | 27      |
| 2      | a688ee40-d8d9-4736-b6f9-6b74935ba3b1           | <Unknown>                                                                                      | 652     |
| 3      | WinSATAssessment                               |                                                                                                | 3       |
| 4      |                                                | WinSAT: WinSPR Compressed Info                                                                 | 1       |
| 5      |                                                | WinSAT: Metrics Compressed Info                                                                | 1       |
| 6      |                                                | WinSAT: SystemConfig Compressed Info                                                           | 1       |
| 7      | Thread                                         |                                                                                                | 143991  |
| 8      |                                                | Thread: Create                                                                                 | 95      |
| 9      |                                                | Thread: Delete                                                                                 | 159     |
| 10     |                                                | Thread: Start Rundown                                                                          | 2490    |
| 11     |                                                | Thread: End Rundown                                                                            | 2428    |
| 12     |                                                | Thread: CSwitch                                                                                | 85467   |
| 13     |                                                | Thread: SetPriority                                                                            | 860     |
| 14     |                                                | Thread: SetBasePriority                                                                        | 5362    |
| 15     |                                                | Thread: ReadyThread                                                                            | 46270   |
| 16     |                                                | Thread: Set Page Priority                                                                      | 167     |
| 17     |                                                | Thread: Set I/O Priority                                                                       | 100     |
| 18     |                                                | Thread [Provider]                                                                              | 401     |
| 19     |                                                | Thread: Set Ideal Processor                                                                    | 97      |
| 20     |                                                | Thread: Set User Ideal Processor                                                               | 95      |
| 21     | SysConfigEx                                    |                                                                                                | 63      |
| 22     |                                                | SysConfigEx: BuildInfo                                                                         | 1       |
| 23     |                                                | SysConfigEx: SystemPaths                                                                       | 1       |
| 24     |                                                | SysConfigEx: UnknownVolume                                                                     | 1       |
| 25     |                                                | SysConfigEx: VolumeMapping                                                                     | 13      |
| 26     |                                                | SysConfigEx: NetworkInterface                                                                  | 46      |
| 27     |                                                | SysConfigEx [Provider]                                                                         | 1       |
| 28     | SysConfig                                      |                                                                                                | 455     |
| 29     |                                                | SysConfig: CPUs                                                                                | 1       |
| 30     |                                                | SysConfig: Physical Disks                                                                      | 2       |
| 31     |                                                | SysConfig: Logical Disks                                                                       | 7       |
| 32     |                                                | SysConfig: Network Cards                                                                       | 5       |
| 33     |                                                | SysConfig: Video Adapters                                                                      | 2       |
| 34     |                                                | SysConfig: Services                                                                            | 283     |
| 35     |                                                | SysConfig: Power Management                                                                    | 1       |
| 36     |                                                | SysConfig: IRQs                                                                                | 16      |
| 37     |                                                | SysConfig: PnP Devices                                                                         | 117     |
| 38     |                                                | SysConfig: NUMA Nodes                                                                          | 1       |
| 39     |                                                | SysConfig: Platform                                                                            | 1       |
| 40     |                                                | SysConfig: Processor Group Configuration                                                       | 1       |
| 41     |                                                | SysConfig: Processor Mapping                                                                   | 1       |
| 42     |                                                | SysConfig: Display DPI                                                                         | 1       |
| 43     |                                                | SysConfig: Code Integrity                                                                      | 1       |
| 44     |                                                | SysConfig: Machine Id                                                                          | 1       |
| 45     |                                                | SysConfig [Provider]                                                                           | 14      |
| 46     | StackWalk                                      | Stack Walk                                                                                     | 1260641 |
| 47     | Process                                        |                                                                                                | 594     |
| 48     |                                                | Process: Create                                                                                | 1       |
| 49     |                                                | Process: Delete                                                                                | 6       |
| 50     |                                                | Process: Start Rundown                                                                         | 154     |
| 51     |                                                | Process: End Rundown                                                                           | 149     |
| 52     |                                                | Process [Provider]                                                                             | 6       |
| 53     |                                                | Process: PerfCounters: End                                                                     | 6       |
| 54     |                                                | Process: PerfCounters: Rundown                                                                 | 149     |
| 55     |                                                | Process: Zombie                                                                                | 123     |
| 56     | Power                                          |                                                                                                | 113     |
| 57     |                                                | Power: Perf State Change                                                                       | 4       |
| 58     |                                                | Power: Idle State Change                                                                       | 109     |
| 59     | Pool                                           |                                                                                                | 626357  |
| 60     |                                                | Pool: Allocate                                                                                 | 241999  |
| 61     |                                                | Pool: Allocate Session                                                                         | 7365    |
| 62     |                                                | Pool: Free                                                                                     | 368975  |
| 63     |                                                | Pool: Free Session                                                                             | 7643    |
| 64     |                                                | Pool: PoolSnap Start Rundown                                                                   | 24      |
| 65     |                                                | Pool: PoolSnap End Rundown                                                                     | 24      |
| 66     |                                                | Pool: BigPoolSnap Start Rundown                                                                | 149     |
| 67     |                                                | Pool: BigPoolSnap End Rundown                                                                  | 146     |
| 68     |                                                | Pool: PoolSnap Session Start Rundown                                                           | 5       |
| 69     |                                                | Pool: PoolSnap Session End Rundown                                                             | 5       |
| 70     |                                                | Pool: BigPoolSnap Session Start Rundown                                                        | 11      |
| 71     |                                                | Pool: BigPoolSnap Session End Rundown                                                          | 11      |
| 72     | Perfinfo                                       |                                                                                                | 1568074 |
| 73     |                                                | Mark                                                                                           | 3       |
| 74     |                                                | Sampled Profile                                                                                | 13853   |
| 75     |                                                | Message Signaled Interrupt                                                                     | 5468    |
| 76     |                                                | SysCall: Enter                                                                                 | 769129  |
| 77     |                                                | SysCall: Exit                                                                                  | 767373  |
| 78     |                                                | Interrupt                                                                                      | 190     |
| 79     |                                                | Dpc                                                                                            | 12056   |
| 80     |                                                | Sampled Profile Freq: Start Rundown                                                            | 1       |
| 81     |                                                | Sampled Profile Freq: End Rundown                                                              | 1       |
| 82     | PageFault                                      |                                                                                                | 135319  |
| 83     |                                                | PageFault: Transition                                                                          | 38820   |
| 84     |                                                | PageFault: Demand Zero                                                                         | 47925   |
| 85     |                                                | PageFault: Copy on Write                                                                       | 76      |
| 86     |                                                | PageFault: Guard Page                                                                          | 60      |
| 87     |                                                | PageFault: Hard Page Fault                                                                     | 16401   |
| 88     |                                                | PageFault [Provider]                                                                           | 34      |
| 89     |                                                | Hardfault                                                                                      | 5150    |
| 90     |                                                | Memory: VirtualAlloc                                                                           | 1269    |
| 91     |                                                | Memory: VirtualFree                                                                            | 855     |
| 92     |                                                | Memory: MemInfo                                                                                | 27      |
| 93     |                                                | Memory: MMStat                                                                                 | 1       |
| 94     |                                                | Memory: MemInfoExWS                                                                            | 27      |
| 95     |                                                | Memory: MemInfoExSessionWS                                                                     | 27      |
| 96     |                                                | Memory: VirtualAlloc Start Rundown                                                             | 12434   |
| 97     |                                                | Memory: VirtualAlloc End Rundown                                                               | 12213   |
| 98     | Microsoft-Windows-Win32k                       |                                                                                                | 13925   |
| 99     |                                                | Microsoft-Windows-Win32k/ThreadInfoRundown/win:Info                                            | 778     |
| 100    |                                                | Microsoft-Windows-Win32k/QueuePostMessage/win:Info                                             | 4994    |
| 101    |                                                | Microsoft-Windows-Win32k/SendMessage/win:Start                                                 | 31      |
| 102    |                                                | Microsoft-Windows-Win32k/RetrievePostMessage/win:Info                                          | 7296    |
| 103    |                                                | Microsoft-Windows-Win32k/RetrieveSendMessage/win:Start                                         | 31      |
| 104    |                                                | Microsoft-Windows-Win32k/RetrieveInputMessage/win:Info                                         | 201     |
| 105    |                                                | Microsoft-Windows-Win32k/RetrievePseudoMessage/win:Info                                        | 37      |
| 106    |                                                | Microsoft-Windows-Win32k/WakePump/win:Info                                                     | 228     |
| 107    |                                                | Microsoft-Windows-Win32k/SendMessage/win:Stop                                                  | 31      |
| 108    |                                                | Microsoft-Windows-Win32k/RetrieveSendMessage/win:Stop                                          | 31      |
| 109    |                                                | Microsoft-Windows-Win32k/QueueInputMessage/win:Info                                            | 196     |
| 110    |                                                | Microsoft-Windows-Win32k/DispatchMessage/win:Start                                             | 28      |
| 111    |                                                | Microsoft-Windows-Win32k/DispatchMessage/win:Stop                                              | 28      |
| 112    |                                                | Microsoft-Windows-Win32k/QueueNullPostMessage/win:Info                                         | 15      |
| 113    | Microsoft-Windows-UserModePowerService         |                                                                                                | 310     |
| 114    |                                                | Microsoft-Windows-UserModePowerService/RundownPlatformRole/win:Info                            | 1       |
| 115    |                                                | Microsoft-Windows-UserModePowerService/RundownPowerScheme/win:Info                             | 1       |
| 116    |                                                | Microsoft-Windows-UserModePowerService/RundownAcPowerSetting/win:Info                          | 140     |
| 117    |                                                | Microsoft-Windows-UserModePowerService/RundownDcPowerSetting/win:Info                          | 140     |
| 118    |                                                | Microsoft-Windows-UserModePowerService/RundownBatteryInformation/win:Info                      | 1       |
| 119    |                                                | Microsoft-Windows-UserModePowerService/RundownBatteryStatus/win:Info                           | 1       |
| 120    |                                                | Microsoft-Windows-UserModePowerService/RundownBrightnessCapability/win:Info                    | 1       |
| 121    |                                                | Microsoft-Windows-UserModePowerService/RundownPowerSource/win:Info                             | 1       |
| 122    |                                                | Microsoft-Windows-UserModePowerService/RundownOverrideConfiguration/win:Info                   | 1       |
| 123    |                                                | Microsoft-Windows-UserModePowerService/RundownPowerProfileSetting/win:Info                     | 7       |
| 124    |                                                | Microsoft-Windows-UserModePowerService/RundownSmartUserPresenceState/win:Info                  | 1       |
| 125    |                                                | Microsoft-Windows-UserModePowerService/RundownOverlaySchemePowerSetting/win:Info               | 11      |
| 126    |                                                | Microsoft-Windows-UserModePowerService/RundownActualOverlayPowerScheme/win:Info                | 2       |
| 127    |                                                | Microsoft-Windows-UserModePowerService/RundownEffectiveOverlayPowerScheme/win:Info             | 1       |
| 128    |                                                | Microsoft-Windows-UserModePowerService/RundownOverlaySuspendReason/win:Info                    | 1       |
| 129    | Microsoft-Windows-TCPIP                        |                                                                                                | 1930    |
| 130    |                                                | Microsoft-Windows-TCPIP/TcpEndpointCreation/win:Info                                           | 12      |
| 131    |                                                | Microsoft-Windows-TCPIP/TcpRequestConnect/win:Info                                             | 3       |
| 132    |                                                | Microsoft-Windows-TCPIP/TcpInspectConnectComplete/win:Info                                     | 3       |
| 133    |                                                | Microsoft-Windows-TCPIP/TcpTcbSynSend/win:Info                                                 | 3       |
| 134    |                                                | Microsoft-Windows-TCPIP/TcpBindEndpointComplete/win:Info                                       | 3       |
| 135    |                                                | Microsoft-Windows-TCPIP/TcpCloseEndpoint/win:Info                                              | 16      |
| 136    |                                                | Microsoft-Windows-TCPIP/TcpCreateEndpointComplete/win:Info                                     | 12      |
| 137    |                                                | Microsoft-Windows-TCPIP/TcpConnectTcbProceeding/win:Info                                       | 3       |
| 138    |                                                | Microsoft-Windows-TCPIP/TcpConnectTcbComplete/win:Info                                         | 1       |
| 139    |                                                | Microsoft-Windows-TCPIP/TcpConnectTcbFailure/win:Info                                          | 2       |
| 140    |                                                | Microsoft-Windows-TCPIP/TcpCloseTcbRequest/win:Info                                            | 5       |
| 141    |                                                | Microsoft-Windows-TCPIP/TcpAbortTcbRequest/win:Info                                            | 4       |
| 142    |                                                | Microsoft-Windows-TCPIP/TcpAbortTcbComplete/win:Info                                           | 4       |
| 143    |                                                | Microsoft-Windows-TCPIP/TcpShutdownTcb/win:Info                                                | 6       |
| 144    |                                                | Microsoft-Windows-TCPIP/TcpDisconnectTcbRtoTimeout/win:Info                                    | 2       |
| 145    |                                                | Microsoft-Windows-TCPIP/TcpTcbStateChange/win:Info                                             | 10      |
| 146    |                                                | Microsoft-Windows-TCPIP/TcpTcbStartTimer/win:Info                                              | 68      |
| 147    |                                                | Microsoft-Windows-TCPIP/TcpTcbStopTimer/win:Info                                               | 124     |
| 148    |                                                | Microsoft-Windows-TCPIP/TcpTcbExpireTimer/win:Info                                             | 7       |
| 149    |                                                | Microsoft-Windows-TCPIP/TcpDataTransferReceive/win:Info                                        | 118     |
| 150    |                                                | Microsoft-Windows-TCPIP/TcpSetTcpOption/win:Info                                               | 1       |
| 151    |                                                | Microsoft-Windows-TCPIP/TcpReceiveRequest/win:Info                                             | 6       |
| 152    |                                                | Microsoft-Windows-TCPIP/TcpDeliveryIndicated/win:Info                                          | 54      |
| 153    |                                                | Microsoft-Windows-TCPIP/TcpDeliverySatisfied/win:Info                                          | 5       |
| 154    |                                                | Microsoft-Windows-TCPIP/TcpSendPosted/win:Info                                                 | 57      |
| 155    |                                                | Microsoft-Windows-TCPIP/TcpSendTransmitted/win:Info                                            | 57      |
| 156    |                                                | Microsoft-Windows-TCPIP/TcpSendAdvance/win:Info                                                | 58      |
| 157    |                                                | Microsoft-Windows-TCPIP/TcpSrttMeasurementStarted/win:Info                                     | 60      |
| 158    |                                                | Microsoft-Windows-TCPIP/TcpSrttMeasurementComplete/win:Info                                    | 58      |
| 159    |                                                | Microsoft-Windows-TCPIP/TcpSrttMeasurementCancelled/win:Info                                   | 2       |
| 160    |                                                | Microsoft-Windows-TCPIP/UdpEndpointSendMessages/win:Info                                       | 1       |
| 161    |                                                | Microsoft-Windows-TCPIP/UdpEndpointReceiveMessages/win:Info                                    | 1       |
| 162    |                                                | Microsoft-Windows-TCPIP/TcpDeliveryFlush/win:Info                                              | 4       |
| 163    |                                                | Microsoft-Windows-TCPIP/TcpConnectRestransmit/win:Info                                         | 2       |
| 164    |                                                | Microsoft-Windows-TCPIP/TcpAcquirePort/win:Info                                                | 3       |
| 165    |                                                | Microsoft-Windows-TCPIP/TcpAcquireWeakRefPort/win:Info                                         | 3       |
| 166    |                                                | Microsoft-Windows-TCPIP/TcpReleasePort/win:Info                                                | 14      |
| 167    |                                                | Microsoft-Windows-TCPIP/TcpFlushSack/win:Info                                                  | 2       |
| 168    |                                                | Microsoft-Windows-TCPIP/IpInterfaceRundown/win:Info                                            | 10      |
| 169    |                                                | Microsoft-Windows-TCPIP/TcpipReceiveSlowPath/win:Info                                          | 16      |
| 170    |                                                | Microsoft-Windows-TCPIP/TcpipSendSlowPath/win:Info                                             | 141     |
| 171    |                                                | Microsoft-Windows-TCPIP/TcpTemplateParameters/win:Info                                         | 1       |
| 172    |                                                | Microsoft-Windows-TCPIP/TcpTemplateChanged/win:Info                                            | 3       |
| 173    |                                                | Microsoft-Windows-TCPIP/TcpCwndRestart/win:Info                                                | 58      |
| 174    |                                                | Microsoft-Windows-TCPIP/RssBindingRundown/win:Info                                             | 1       |
| 175    |                                                | Microsoft-Windows-TCPIP/RssPortRundown/win:Info                                                | 1       |
| 176    |                                                | Microsoft-Windows-TCPIP/TcpConnectionRundown/win:Info                                          | 32      |
| 177    |                                                | Microsoft-Windows-TCPIP//win:Info                                                              | 136     |
| 178    |                                                | Microsoft-Windows-TCPIP/IpNeighborState/win:Info                                               | 1       |
| 179    |                                                | Microsoft-Windows-TCPIP/IpNeighborDiscovery/win:Info                                           | 2       |
| 180    |                                                | Microsoft-Windows-TCPIP/IpSourceAddressSelection/win:Info                                      | 16      |
| 181    |                                                | Microsoft-Windows-TCPIP/IpSortedAddressPairs/win:Info                                          | 26      |
| 182    |                                                | Microsoft-Windows-TCPIP/TcpDataTransferCumAck/win:Info                                         | 57      |
| 183    |                                                | Microsoft-Windows-TCPIP/TcpDataTransferSend/win:Info                                           | 123     |
| 184    |                                                | Microsoft-Windows-TCPIP/TcpDataTransferRttSample/win:Info                                      | 58      |
| 185    |                                                | Microsoft-Windows-TCPIP/TcpDataTransferRetransmitRound/win:Info                                | 2       |
| 186    |                                                | Microsoft-Windows-TCPIP/TcpipNblOob/win:Info                                                   | 39      |
| 187    |                                                | Microsoft-Windows-TCPIP/TcpipRouteLookup/win:Info                                              | 56      |
| 188    |                                                | Microsoft-Windows-TCPIP/TcpipSrcAddrLookup/win:Info                                            | 8       |
| 189    |                                                | Microsoft-Windows-TCPIP/Memory/win:Info                                                        | 6       |
| 190    |                                                | Microsoft-Windows-TCPIP/TcpAssociateNameResContext/win:Info                                    | 2       |
| 191    |                                                | Microsoft-Windows-TCPIP/TcpInspectConnectWithNameResContext/win:Info                           | 1       |
| 192    |                                                | Microsoft-Windows-TCPIP/IpRouteBlocked/win:Info                                                | 1       |
| 193    |                                                | Microsoft-Windows-TCPIP/TcpTailLossProbe/win:Info                                              | 3       |
| 194    |                                                | Microsoft-Windows-TCPIP/TcpRack/win:Info                                                       | 11      |
| 195    |                                                | Microsoft-Windows-TCPIP/UdpCreateEndpointComplete/win:Info                                     | 10      |
| 196    |                                                | Microsoft-Windows-TCPIP/UdpBindEndpointComplete/win:Info                                       | 3       |
| 197    |                                                | Microsoft-Windows-TCPIP/UdpCloseEndpointBound/win:Info                                         | 3       |
| 198    |                                                | Microsoft-Windows-TCPIP/UdpCloseEndpointUnBound/win:Info                                       | 7       |
| 199    |                                                | Microsoft-Windows-TCPIP/IcmpSendRecv/win:Info                                                  | 8       |
| 200    |                                                | Microsoft-Windows-TCPIP/TcpSendComplete/win:Info                                               | 57      |
| 201    |                                                | Microsoft-Windows-TCPIP/TcpCubicDataTransferCumAck/win:Info                                    | 3       |
| 202    |                                                | Microsoft-Windows-TCPIP/IpRouteDGDStateChange/win:Info                                         | 1       |
| 203    |                                                | Microsoft-Windows-TCPIP/IpRouteRundown/win:Info                                                | 34      |
| 204    |                                                | Microsoft-Windows-TCPIP/InetInspect/win:Info                                                   | 171     |
| 205    |                                                | Microsoft-Windows-TCPIP/TcpipSourceConstraint/win:Info                                         | 2       |
| 206    |                                                | Microsoft-Windows-TCPIP/RemoteEndpoint/win:Info                                                | 26      |
| 207    | Microsoft-Windows-StorPort                     |                                                                                                | 15747   |
| 208    |                                                | Microsoft-Windows-StorPort/Port/win:Info                                                       | 2632    |
| 209    |                                                | Microsoft-Windows-StorPort/Port/Dispatch                                                       | 2615    |
| 210    |                                                | Microsoft-Windows-StorPort/Port/Completion                                                     | 2615    |
| 211    |                                                | Microsoft-Windows-StorPort/Port/Queue                                                          | 5265    |
| 212    |                                                | Microsoft-Windows-StorPort/Port/win:Start                                                      | 8       |
| 213    |                                                | Microsoft-Windows-StorPort/Port/win:Stop                                                       | 8       |
| 214    |                                                | Microsoft-Windows-StorPort/Isr/Completion                                                      | 2604    |
| 215    | Microsoft-Windows-Search-Core                  |                                                                                                | 39      |
| 216    |                                                | Microsoft-Windows-Search-Core/USN_Notify/win:Info                                              | 8       |
| 217    |                                                | Microsoft-Windows-Search-Core/Gatherer_OnDataChange_Track_Url/win:Info                         | 8       |
| 218    |                                                | Microsoft-Windows-Search-Core/ETWLogging/win:Info                                              | 9       |
| 219    |                                                | Microsoft-Windows-Search-Core/FileChangeTracker_ProcessUSN/win:Start                           | 7       |
| 220    |                                                | Microsoft-Windows-Search-Core/FileChangeTracker_ProcessUSN/win:Stop                            | 7       |
| 221    | Microsoft-Windows-ReadyBoostDriver             | Microsoft-Windows-ReadyBoostDriver/GlobalStats/win:Info                                        | 2       |
| 222    | Microsoft-Windows-RPC                          |                                                                                                | 2473    |
| 223    |                                                | Microsoft-Windows-RPC/RpcClientCall/win:Start                                                  | 201     |
| 224    |                                                | Microsoft-Windows-RPC/RpcServerCall/win:Start                                                  | 1036    |
| 225    |                                                | Microsoft-Windows-RPC/RpcClientCall/win:Stop                                                   | 199     |
| 226    |                                                | Microsoft-Windows-RPC/RpcServerCall/win:Stop                                                   | 1037    |
| 227    | Microsoft-Windows-ProcessStateManager          | Microsoft-Windows-ProcessStateManager/StateChange/win:Info                                     | 124     |
| 228    | Microsoft-Windows-Performance-Recorder-Control |                                                                                                | 96      |
| 229    |                                                | Microsoft-Windows-Performance-Recorder-Control/Perf_LoadProfileFromString/win:Start            | 3       |
| 230    |                                                | Microsoft-Windows-Performance-Recorder-Control/Perf_LoadProfileFromString/win:Stop             | 3       |
| 231    |                                                | Microsoft-Windows-Performance-Recorder-Control/Perf_AddProfileToCollection/win:Start           | 3       |
| 232    |                                                | Microsoft-Windows-Performance-Recorder-Control/Perf_AddProfileToCollection/win:Stop            | 3       |
| 233    |                                                | Microsoft-Windows-Performance-Recorder-Control/Perf_LoadTraceMergePropertiesFromFile/win:Start | 3       |
| 234    |                                                | Microsoft-Windows-Performance-Recorder-Control/Perf_LoadTraceMergePropertiesFromFile/win:Stop  | 3       |
| 235    |                                                | Microsoft-Windows-Performance-Recorder-Control/Perf_StartProfiles/win:Stop                     | 1       |
| 236    |                                                | Microsoft-Windows-Performance-Recorder-Control/Perf_StopProfiles/win:Start                     | 1       |
| 237    |                                                | Microsoft-Windows-Performance-Recorder-Control/Perf_QueryProfiles/win:Start                    | 2       |
| 238    |                                                | Microsoft-Windows-Performance-Recorder-Control/Perf_QueryProfiles/win:Stop                     | 2       |
| 239    |                                                | Microsoft-Windows-Performance-Recorder-Control/Perf_ControlProgressHandlerBegin/win:Start      | 1       |
| 240    |                                                | Microsoft-Windows-Performance-Recorder-Control/Perf_ControlProgressHandlerBegin/win:Stop       | 1       |
| 241    |                                                | Microsoft-Windows-Performance-Recorder-Control/Perf_ControlProgressHandlerUpdate/win:Start     | 34      |
| 242    |                                                | Microsoft-Windows-Performance-Recorder-Control/Perf_ControlProgressHandlerUpdate/win:Stop      | 34      |
| 243    |                                                | Microsoft-Windows-Performance-Recorder-Control/Perf_ControlProgressHandlerEnd/win:Start        | 1       |
| 244    |                                                | Microsoft-Windows-Performance-Recorder-Control/Perf_ControlProgressHandlerEnd/win:Stop         | 1       |
| 245    | Microsoft-Windows-Networking-Correlation       |                                                                                                | 3381    |
| 246    |                                                | Microsoft-Windows-Networking-Correlation//win:Start                                            | 423     |
| 247    |                                                | Microsoft-Windows-Networking-Correlation//win:Stop                                             | 376     |
| 248    |                                                | Microsoft-Windows-Networking-Correlation//win:Send                                             | 2582    |
| 249    | Microsoft-Windows-Kernel-StoreMgr              | Microsoft-Windows-Kernel-StoreMgr/StoreRundown/win:Info                                        | 30      |
| 250    | Microsoft-Windows-Kernel-Processor-Power       |                                                                                                | 190     |
| 251    |                                                | Microsoft-Windows-Kernel-Processor-Power/IdleAccountingRundown/win:Info                        | 4       |
| 252    |                                                | Microsoft-Windows-Kernel-Processor-Power/ProcessorFirmwareRundown/win:Info                     | 4       |
| 253    |                                                | Microsoft-Windows-Kernel-Processor-Power/PTStateDomainFirmwareRundown/win:Info                 | 1       |
| 254    |                                                | Microsoft-Windows-Kernel-Processor-Power/Summary/win:Info                                      | 4       |
| 255    |                                                | Microsoft-Windows-Kernel-Processor-Power/PerfStatesRundown/win:Info                            | 4       |
| 256    |                                                | Microsoft-Windows-Kernel-Processor-Power/BiosPStatesRundown/win:Info                           | 4       |
| 257    |                                                | Microsoft-Windows-Kernel-Processor-Power/BiosCStatesRundown/win:Info                           | 4       |
| 258    |                                                | Microsoft-Windows-Kernel-Processor-Power/BiosTStatesRundown/win:Info                           | 4       |
| 259    |                                                | Microsoft-Windows-Kernel-Processor-Power/LogicalProcessorIdlingRundown/win:Info                | 1       |
| 260    |                                                | Microsoft-Windows-Kernel-Processor-Power/Summary2/win:Info                                     | 4       |
| 261    |                                                | Microsoft-Windows-Kernel-Processor-Power/PepQueryCapabilities/win:Info                         | 4       |
| 262    |                                                | Microsoft-Windows-Kernel-Processor-Power/ProcessorPerformanceRundown/win:Info                  | 4       |
| 263    |                                                | Microsoft-Windows-Kernel-Processor-Power/ParkNodeRundown/win:Info                              | 1       |
| 264    |                                                | Microsoft-Windows-Kernel-Processor-Power/ProcessorIdleRundown/win:Info                         | 4       |
| 265    |                                                | Microsoft-Windows-Kernel-Processor-Power/ProcessorIdRundown/win:Info                           | 4       |
| 266    |                                                | Microsoft-Windows-Kernel-Processor-Power/PepGetPlatformIdleStates/win:Info                     | 1       |
| 267    |                                                | Microsoft-Windows-Kernel-Processor-Power/PlatformAccountingBucketIntervalsRundown/win:Info     | 1       |
| 268    |                                                | Microsoft-Windows-Kernel-Processor-Power/StaticPolicyRundown/win:Info                          | 1       |
| 269    |                                                | Microsoft-Windows-Kernel-Processor-Power/CoordinatedIdleRundown/win:Info                       | 1       |
| 270    |                                                | Microsoft-Windows-Kernel-Processor-Power/ProfileRundown/win:Info                               | 10      |
| 271    |                                                | Microsoft-Windows-Kernel-Processor-Power/ProfileSettingRundown/win:Info                        | 122     |
| 272    |                                                | Microsoft-Windows-Kernel-Processor-Power/ProfileStatusRundown/win:Info                         | 1       |
| 273    |                                                | Microsoft-Windows-Kernel-Processor-Power/HeterogeneousPoliciesRundown/win:Info                 | 1       |
| 274    |                                                | Microsoft-Windows-Kernel-Processor-Power/QosSupportRundown/win:Info                            | 1       |
| 275    | Microsoft-Windows-Kernel-Power                 |                                                                                                | 1344    |
| 276    |                                                | Microsoft-Windows-Kernel-Power/SystemTimeResolutionChange/win:Info                             | 1046    |
| 277    |                                                | Microsoft-Windows-Kernel-Power/SystemTimeResolutionRundown/win:Info                            | 1       |
| 278    |                                                | Microsoft-Windows-Kernel-Power/SystemTimeResolutionRequestRundown/win:Info                     | 1       |
| 279    |                                                | Microsoft-Windows-Kernel-Power/SystemTimeResolutionKernelChange/win:Info                       | 103     |
| 280    |                                                | Microsoft-Windows-Kernel-Power/PowerRequestRundown/win:Info                                    | 8       |
| 281    |                                                | Microsoft-Windows-Kernel-Power/SleepDisableReasonRundown/win:Info                              | 2       |
| 282    |                                                | Microsoft-Windows-Kernel-Power/AcDcStateRundown/win:Info                                       | 1       |
| 283    |                                                | Microsoft-Windows-Kernel-Power/SystemTimerResolutionStackRundown/win:Info                      | 14      |
| 284    |                                                | Microsoft-Windows-Kernel-Power/FirmwarePlatformRoleRundown/win:Info                            | 1       |
| 285    |                                                | Microsoft-Windows-Kernel-Power/DeviceRundown/win:Info                                          | 116     |
| 286    |                                                | Microsoft-Windows-Kernel-Power/StandbyConnectivityRundown/win:Info                             | 1       |
| 287    |                                                | Microsoft-Windows-Kernel-Power/CsComplianceRundown/win:Info                                    | 5       |
| 288    |                                                | Microsoft-Windows-Kernel-Power/DeepSleepConstraintRundown/win:Info                             | 1       |
| 289    |                                                | Microsoft-Windows-Kernel-Power/SystemLatencyRundown/win:Info                                   | 1       |
| 290    |                                                | Microsoft-Windows-Kernel-Power/DynamicTickStatusRundown/win:Info                               | 1       |
| 291    |                                                | Microsoft-Windows-Kernel-Power/PowerStateEventRundown/win:Info                                 | 42      |
| 292    | Microsoft-Windows-Kernel-EventTracing          |                                                                                                | 1949    |
| 293    |                                                | Microsoft-Windows-Kernel-EventTracing/ETW_TASK_STACK_TRACE/ETW_OPCODE_USER_MODE_STACK_TRACE    | 1109    |
| 294    |                                                | Microsoft-Windows-Kernel-EventTracing/ETW_TASK_LOST_EVENT/win:Info                             | 840     |
| 295    | Microsoft-Windows-DxgKrnl                      |                                                                                                | 23024   |
| 296    |                                                | Microsoft-Windows-DxgKrnl/VSyncDPC/win:Info                                                    | 425     |
| 297    |                                                | Microsoft-Windows-DxgKrnl/WorkerThread/win:Start                                               | 178     |
| 298    |                                                | Microsoft-Windows-DxgKrnl/WorkerThread/win:Stop                                                | 179     |
| 299    |                                                | Microsoft-Windows-DxgKrnl/ChangePriority/win:Info                                              | 51      |
| 300    |                                                | Microsoft-Windows-DxgKrnl/AttemptPreemption/win:Info                                           | 75      |
| 301    |                                                | Microsoft-Windows-DxgKrnl/Adapter/win:DC_Start                                                 | 2       |
| 302    |                                                | Microsoft-Windows-DxgKrnl/Device/win:DC_Start                                                  | 38      |
| 303    |                                                | Microsoft-Windows-DxgKrnl/Context/win:DC_Start                                                 | 51      |
| 304    |                                                | Microsoft-Windows-DxgKrnl/AdapterAllocation/win:Start                                          | 1       |
| 305    |                                                | Microsoft-Windows-DxgKrnl/AdapterAllocation/win:Stop                                           | 1       |
| 306    |                                                | Microsoft-Windows-DxgKrnl/AdapterAllocation/win:DC_Start                                       | 979     |
| 307    |                                                | Microsoft-Windows-DxgKrnl/DeviceAllocation/win:Start                                           | 2       |
| 308    |                                                | Microsoft-Windows-DxgKrnl/DeviceAllocation/win:Stop                                            | 2       |
| 309    |                                                | Microsoft-Windows-DxgKrnl/DeviceAllocation/win:DC_Start                                        | 1068    |
| 310    |                                                | Microsoft-Windows-DxgKrnl/TerminateAllocation/win:Info                                         | 1       |
| 311    |                                                | Microsoft-Windows-DxgKrnl/ProcessTerminateAllocation/win:Info                                  | 1       |
| 312    |                                                | Microsoft-Windows-DxgKrnl/Lock/win:Info                                                        | 40      |
| 313    |                                                | Microsoft-Windows-DxgKrnl/Unlock/win:Info                                                      | 40      |
| 314    |                                                | Microsoft-Windows-DxgKrnl/ReferenceAllocations/win:Info                                        | 74      |
| 315    |                                                | Microsoft-Windows-DxgKrnl/PatchLocationList/win:Info                                           | 87      |
| 316    |                                                | Microsoft-Windows-DxgKrnl/ApertureMapping/win:Info                                             | 11      |
| 317    |                                                | Microsoft-Windows-DxgKrnl/ApertureUnmapping/win:Info                                           | 1       |
| 318    |                                                | Microsoft-Windows-DxgKrnl/PagingOpMapApertureSegment/win:Info                                  | 22      |
| 319    |                                                | Microsoft-Windows-DxgKrnl/PagingOpUnmapApertureSegment/win:Info                                | 2       |
| 320    |                                                | Microsoft-Windows-DxgKrnl/Preparation/win:Start                                                | 74      |
| 321    |                                                | Microsoft-Windows-DxgKrnl/Preparation/win:Info                                                 | 13      |
| 322    |                                                | Microsoft-Windows-DxgKrnl/Preparation/win:Stop                                                 | 74      |
| 323    |                                                | Microsoft-Windows-DxgKrnl/ReserveResource/win:Start                                            | 11      |
| 324    |                                                | Microsoft-Windows-DxgKrnl/ReserveResource/win:Stop                                             | 11      |
| 325    |                                                | Microsoft-Windows-DxgKrnl/InnerIteration/win:Start                                             | 22      |
| 326    |                                                | Microsoft-Windows-DxgKrnl/InnerIteration/win:Stop                                              | 22      |
| 327    |                                                | Microsoft-Windows-DxgKrnl/AllocationFault/win:Info                                             | 32      |
| 328    |                                                | Microsoft-Windows-DxgKrnl/MarkAllocation/win:Info                                              | 1       |
| 329    |                                                | Microsoft-Windows-DxgKrnl/PageInAllocation/win:Info                                            | 11      |
| 330    |                                                | Microsoft-Windows-DxgKrnl/AddDmaBuffer/win:Start                                               | 51      |
| 331    |                                                | Microsoft-Windows-DxgKrnl/ReportSegment/win:Info                                               | 9       |
| 332    |                                                | Microsoft-Windows-DxgKrnl/ReportCommittedAllocation/win:Info                                   | 43      |
| 333    |                                                | Microsoft-Windows-DxgKrnl/Semaphore/win:DC_Start                                               | 45      |
| 334    |                                                | Microsoft-Windows-DxgKrnl/Fence/win:Start                                                      | 1       |
| 335    |                                                | Microsoft-Windows-DxgKrnl/Fence/win:Stop                                                       | 1       |
| 336    |                                                | Microsoft-Windows-DxgKrnl/Fence/win:DC_Start                                                   | 57      |
| 337    |                                                | Microsoft-Windows-DxgKrnl/SetDisplayMode/win:Info                                              | 2       |
| 338    |                                                | Microsoft-Windows-DxgKrnl/BlockThread/win:Info                                                 | 3       |
| 339    |                                                | Microsoft-Windows-DxgKrnl/Profiler/win:Start                                                   | 7151    |
| 340    |                                                | Microsoft-Windows-DxgKrnl/Profiler/win:Stop                                                    | 7150    |
| 341    |                                                | Microsoft-Windows-DxgKrnl/ExtendedProfiler/win:Start                                           | 189     |
| 342    |                                                | Microsoft-Windows-DxgKrnl/ExtendedProfiler/win:Stop                                            | 189     |
| 343    |                                                | Microsoft-Windows-DxgKrnl/SetPointerPosition/win:Info                                          | 190     |
| 344    |                                                | Microsoft-Windows-DxgKrnl/DpiReportAdapter/win:Info                                            | 3       |
| 345    |                                                | Microsoft-Windows-DxgKrnl/MMIOFlip/win:Info                                                    | 36      |
| 346    |                                                | Microsoft-Windows-DxgKrnl/EtwVersion/win:Stop                                                  | 1       |
| 347    |                                                | Microsoft-Windows-DxgKrnl/Flip/win:Info                                                        | 36      |
| 348    |                                                | Microsoft-Windows-DxgKrnl/Render/win:Info                                                      | 36      |
| 349    |                                                | Microsoft-Windows-DxgKrnl/RenderKm/win:Info                                                    | 38      |
| 350    |                                                | Microsoft-Windows-DxgKrnl/PresentHistory/win:Info                                              | 37      |
| 351    |                                                | Microsoft-Windows-DxgKrnl/PresentHistory/win:Stop                                              | 37      |
| 352    |                                                | Microsoft-Windows-DxgKrnl/DmaPacket/win:Start                                                  | 74      |
| 353    |                                                | Microsoft-Windows-DxgKrnl/DmaPacket/win:Stop                                                   | 73      |
| 354    |                                                | Microsoft-Windows-DxgKrnl/DmaPacket/win:Info                                                   | 73      |
| 355    |                                                | Microsoft-Windows-DxgKrnl/QueuePacket/win:Start                                                | 269     |
| 356    |                                                | Microsoft-Windows-DxgKrnl/QueuePacket/win:Info                                                 | 316     |
| 357    |                                                | Microsoft-Windows-DxgKrnl/QueuePacket/win:Stop                                                 | 267     |
| 358    |                                                | Microsoft-Windows-DxgKrnl/VSyncInterrupt/win:Info                                              | 425     |
| 359    |                                                | Microsoft-Windows-DxgKrnl/GetDeviceState/win:Info                                              | 216     |
| 360    |                                                | Microsoft-Windows-DxgKrnl/Present/win:Info                                                     | 36      |
| 361    |                                                | Microsoft-Windows-DxgKrnl/OfferAllocation/win:Start                                            | 40      |
| 362    |                                                | Microsoft-Windows-DxgKrnl/OfferAllocation/win:Info                                             | 40      |
| 363    |                                                | Microsoft-Windows-DxgKrnl/OfferAllocation/win:Stop                                             | 11      |
| 364    |                                                | Microsoft-Windows-DxgKrnl/ReportOfferAllocation/win:Info                                       | 83      |
| 365    |                                                | Microsoft-Windows-DxgKrnl/ReclaimAllocation/win:Info                                           | 40      |
| 366    |                                                | Microsoft-Windows-DxgKrnl/PresentHistoryDetailed/win:Start                                     | 38      |
| 367    |                                                | Microsoft-Windows-DxgKrnl/ReportCommittedGlobalAllocation/win:DC_Start                         | 6       |
| 368    |                                                | Microsoft-Windows-DxgKrnl/SignalSynchronizationObject2/win:Info                                | 72      |
| 369    |                                                | Microsoft-Windows-DxgKrnl/NodeMetadata/win:Info                                                | 5       |
| 370    |                                                | Microsoft-Windows-DxgKrnl/VSyncDPCMultiPlane/win:Info                                          | 425     |
| 371    |                                                | Microsoft-Windows-DxgKrnl/TotalBytesResidentInSegment/win:Info                                 | 12      |
| 372    |                                                | Microsoft-Windows-DxgKrnl/Brightness/win:Info                                                  | 2       |
| 373    |                                                | Microsoft-Windows-DxgKrnl/BacklightOptimizationLevel/win:Info                                  | 2       |
| 374    |                                                | Microsoft-Windows-DxgKrnl/VidMmDereferenceObjectAsync/win:Start                                | 1       |
| 375    |                                                | Microsoft-Windows-DxgKrnl/VidMmDereferenceObjectAsync/win:Stop                                 | 1       |
| 376    |                                                | Microsoft-Windows-DxgKrnl/VidMmUnmapViewAsync/win:Start                                        | 2       |
| 377    |                                                | Microsoft-Windows-DxgKrnl/VidMmUnmapViewAsync/win:Stop                                         | 2       |
| 378    |                                                | Microsoft-Windows-DxgKrnl/PagingPreparation/win:Start                                          | 279     |
| 379    |                                                | Microsoft-Windows-DxgKrnl/PagingPreparation/win:Stop                                           | 279     |
| 380    |                                                | Microsoft-Windows-DxgKrnl/CddStandardAllocation/win:Info                                       | 1       |
| 381    |                                                | Microsoft-Windows-DxgKrnl/MonitoredFence/win:DC_Start                                          | 63      |
| 382    |                                                | Microsoft-Windows-DxgKrnl/SignalSynchronizationObjectFromGpu/win:Info                          | 72      |
| 383    |                                                | Microsoft-Windows-DxgKrnl/UnwaitCpuWaiter/win:Info                                             | 19      |
| 384    |                                                | Microsoft-Windows-DxgKrnl/DWMVsyncCountWait/win:Info                                           | 71      |
| 385    |                                                | Microsoft-Windows-DxgKrnl/DWMVsyncSignal/win:Info                                              | 425     |
| 386    |                                                | Microsoft-Windows-DxgKrnl/PagingQueuePacket/win:Start                                          | 21      |
| 387    |                                                | Microsoft-Windows-DxgKrnl/PagingQueuePacket/win:Info                                           | 21      |
| 388    |                                                | Microsoft-Windows-DxgKrnl/PagingQueuePacket/win:Stop                                           | 21      |
| 389    |                                                | Microsoft-Windows-DxgKrnl/ClearFlipDevice/win:Info                                             | 1       |
| 390    |                                                | Microsoft-Windows-DxgKrnl/ExtendedProfiler/win:Info                                            | 177     |
| 391    |                                                | Microsoft-Windows-DxgKrnl/FlushScheduler/win:Info                                              | 4       |
| 392    |                                                | Microsoft-Windows-DxgKrnl/LockAllocationBackingStore/win:Info                                  | 2       |
| 393    |                                                | Microsoft-Windows-DxgKrnl/VidMmProcessBudgetChange/win:Info                                    | 8       |
| 394    |                                                | Microsoft-Windows-DxgKrnl/VidMmProcessUsageChange/win:Info                                     | 8       |
| 395    |                                                | Microsoft-Windows-DxgKrnl/VidMmProcessCommitmentChange/win:Info                                | 8       |
| 396    |                                                | Microsoft-Windows-DxgKrnl/AssociateDxgSchedulerObject/win:Info                                 | 33      |
| 397    |                                                | Microsoft-Windows-DxgKrnl/ReportSyncObject/win:Info                                            | 1       |
| 398    |                                                | Microsoft-Windows-DxgKrnl/ReportSyncObject/win:Start                                           | 72      |
| 399    | Microsoft-Windows-DotNETRuntimeRundown         |                                                                                                | 4662    |
| 400    |                                                | Microsoft-Windows-DotNETRuntimeRundown/CLRMethodRundown/MethodDCEndVerbose                     | 1796    |
| 401    |                                                | Microsoft-Windows-DotNETRuntimeRundown/CLRMethodRundown/DCEndComplete                          | 8       |
| 402    |                                                | Microsoft-Windows-DotNETRuntimeRundown/CLRMethodRundown/DCEndInit                              | 8       |
| 403    |                                                | Microsoft-Windows-DotNETRuntimeRundown/CLRMethodRundown/MethodDCEndILToNativeMap               | 1676    |
| 404    |                                                | Microsoft-Windows-DotNETRuntimeRundown/CLRLoaderRundown/DomainModuleDCEnd                      | 378     |
| 405    |                                                | Microsoft-Windows-DotNETRuntimeRundown/CLRLoaderRundown/ModuleDCEnd                            | 386     |
| 406    |                                                | Microsoft-Windows-DotNETRuntimeRundown/CLRLoaderRundown/AssemblyDCEnd                          | 386     |
| 407    |                                                | Microsoft-Windows-DotNETRuntimeRundown/CLRLoaderRundown/AppDomainDCEnd                         | 16      |
| 408    |                                                | Microsoft-Windows-DotNETRuntimeRundown/CLRRuntimeInformationRundown/win:Start                  | 8       |
| 409    | Microsoft-Windows-DotNETRuntime                |                                                                                                | 2584    |
| 410    |                                                | Microsoft-Windows-DotNETRuntime/CLRMethod/MethodUnloadVerbose                                  | 1796    |
| 411    |                                                | Microsoft-Windows-DotNETRuntime/CLRLoader/ModuleUnload                                         | 386     |
| 412    |                                                | Microsoft-Windows-DotNETRuntime/CLRLoader/AssemblyUnload                                       | 386     |
| 413    |                                                | Microsoft-Windows-DotNETRuntime/CLRLoader/AppDomainUnload                                      | 16      |
| 414    | Microsoft-Windows-Direct3D11                   |                                                                                                | 1002    |
| 415    |                                                | Microsoft-Windows-Direct3D11/Name/win:DC_Start                                                 | 237     |
| 416    |                                                | Microsoft-Windows-Direct3D11/Device/win:DC_Start                                               | 29      |
| 417    |                                                | Microsoft-Windows-Direct3D11/Buffer/win:DC_Start                                               | 250     |
| 418    |                                                | Microsoft-Windows-Direct3D11/Texture2D/win:Start                                               | 1       |
| 419    |                                                | Microsoft-Windows-Direct3D11/Texture2D/win:Stop                                                | 1       |
| 420    |                                                | Microsoft-Windows-Direct3D11/Texture2D/win:DC_Start                                            | 431     |
| 421    |                                                | Microsoft-Windows-Direct3D11/Texture2D/win:Info                                                | 31      |
| 422    |                                                | Microsoft-Windows-Direct3D11/JournalEntry/win:Info                                             | 22      |
| 423    | Microsoft-Windows-DXGI                         |                                                                                                | 384     |
| 424    |                                                | Microsoft-Windows-DXGI/Factory/win:DC_Start                                                    | 40      |
| 425    |                                                | Microsoft-Windows-DXGI/Adapter/win:DC_Start                                                    | 51      |
| 426    |                                                | Microsoft-Windows-DXGI/Output/win:DC_Start                                                     | 29      |
| 427    |                                                | Microsoft-Windows-DXGI/SwapChain/win:DC_Start                                                  | 2       |
| 428    |                                                | Microsoft-Windows-DXGI/Present/win:Start                                                       | 64      |
| 429    |                                                | Microsoft-Windows-DXGI/Present/win:Stop                                                        | 64      |
| 430    |                                                | Microsoft-Windows-DXGI/GetFrameStatistics/win:Info                                             | 64      |
| 431    |                                                | Microsoft-Windows-DXGI/JournalEntry/win:Info                                                   | 70      |
| 432    | Microsoft-JScript                              |                                                                                                | 83376   |
| 433    |                                                | Microsoft-JScript/MethodRundown/DCEndInit                                                      | 16      |
| 434    |                                                | Microsoft-JScript/MethodRundown/DCEndComplete                                                  | 16      |
| 435    |                                                | Microsoft-JScript/MethodRundown/MethodDCEnd                                                    | 16156   |
| 436    |                                                | Microsoft-JScript/ScriptContextRundown/ScriptContextDCEnd                                      | 40      |
| 437    |                                                | Microsoft-JScript/MethodRuntime/MethodLoad                                                     | 67070   |
| 438    |                                                | Microsoft-JScript/ScriptContextRundown/SourceDCEnd                                             | 78      |
| 439    | ImageId                                        |                                                                                                | 88017   |
| 440    |                                                | ImageId: Info                                                                                  | 22513   |
| 441    |                                                | DbgId: None                                                                                    | 294     |
| 442    |                                                | DbgId: BIN                                                                                     | 6       |
| 443    |                                                | DbgId: DBG                                                                                     | 6       |
| 444    |                                                | DbgId: RSDS                                                                                    | 22207   |
| 445    |                                                | DbgId: ILRSDS                                                                                  | 258     |
| 446    |                                                | ImageId [Provider]                                                                             | 40920   |
| 447    |                                                | ImageId: FileVersion                                                                           | 1813    |
| 448    | Image                                          |                                                                                                | 22520   |
| 449    |                                                | Image: Unload                                                                                  | 579     |
| 450    |                                                | Image: Start Rundown                                                                           | 11227   |
| 451    |                                                | Image: End Rundown                                                                             | 10681   |
| 452    |                                                | Image: Load                                                                                    | 32      |
| 453    |                                                | Image: Kernel Base                                                                             | 1       |
| 454    | FileIo                                         |                                                                                                | 81561   |
| 455    |                                                | Filename: Create                                                                               | 152     |
| 456    |                                                | Filename: Delete                                                                               | 1680    |
| 457    |                                                | Filename: Rundown                                                                              | 9839    |
| 458    |                                                | FileIo: Create                                                                                 | 1108    |
| 459    |                                                | FileIo: Cleanup                                                                                | 900     |
| 460    |                                                | FileIo: Close                                                                                  | 3113    |
| 461    |                                                | FileIo: Read                                                                                   | 5286    |
| 462    |                                                | FileIo: Write                                                                                  | 475     |
| 463    |                                                | FileIo: SetInfo                                                                                | 43      |
| 464    |                                                | FileIo: Rename                                                                                 | 1       |
| 465    |                                                | FileIo: DirEnum                                                                                | 222     |
| 466    |                                                | FileIo: Flush                                                                                  | 7       |
| 467    |                                                | FileIo: QueryInfo                                                                              | 23633   |
| 468    |                                                | FileIo: FSCTL                                                                                  | 155     |
| 469    |                                                | FileIo: OperationEnd                                                                           | 34941   |
| 470    |                                                | FileIo: DirNotify                                                                              | 5       |
| 471    |                                                | FileIo: RenamePath                                                                             | 1       |
| 472    | EventTrace                                     |                                                                                                | 21      |
| 473    |                                                | EventTrace: Header                                                                             | 1       |
| 474    |                                                | EventTrace: Group Masks                                                                        | 4       |
| 475    |                                                | EventTrace: Rundown Complete                                                                   | 3       |
| 476    |                                                | EventTrace: Group Masks End                                                                    | 3       |
| 477    |                                                | EventTrace: DbgId (RSDS)                                                                       | 2       |
| 478    |                                                | EventTrace: Build Lab                                                                          | 1       |
| 479    |                                                | EventTrace: Binary Path                                                                        | 2       |
| 480    |                                                | EventTrace [Provider]                                                                          | 5       |
| 481    | EventMetadata                                  |                                                                                                | 468     |
| 482    |                                                | Event Metadata: Event Info                                                                     | 333     |
| 483    |                                                | Event Metadata: Map Info                                                                       | 135     |
| 484    | DiskIo                                         |                                                                                                | 8941    |
| 485    |                                                | DiskIo: Read                                                                                   | 5329    |
| 486    |                                                | DiskIo: Write                                                                                  | 438     |
| 487    |                                                | DiskIo: Read Init                                                                              | 2994    |
| 488    |                                                | DiskIo: Write Init                                                                             | 158     |
| 489    |                                                | DiskIo: Flush                                                                                  | 18      |
| 490    |                                                | DiskIo: Flush Init                                                                             | 4       |
| 491    | 3044f61a-99b0-4c21-b203-d39423c73b00           | <Unknown>                                                                                      | 30      |

</details>

### 生成ETL

除了使用WPR来记录生成ETL，可以直接使用系统的性能计数功能  

