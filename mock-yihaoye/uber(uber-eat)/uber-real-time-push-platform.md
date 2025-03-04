# Uber 实时推送系统设计

优步每天在全球处理数百万次出行的多边市场，并努力为所有用户构建实时体验。  
实时意味着在旅行过程中，有多个参与者可以修改和查看正在进行的旅行的状态并实时更新。这就需要让所有活跃的参与者和应用程序与实时信息保持同步，无论是通过屏幕上的接送时间、到达时间和路线，还是打开应用程序时附近的司机。  

本文描述了如何从轮询刷新应用程序到基于 gRPC 的双向流协议来构建应用程序体验。  
  
## 轮询更新
优步旅行是参与实体之间的协调，例如在现实世界中移动的乘客和司机。随着旅行的进行，这两个实体需要与后端系统和彼此保持更新。 

考虑这样一个场景，其中一名乘客请求搭车，而一名司机在线提供服务。Uber 在后端的匹配系统会识别匹配项并向司机提供行程优惠。现在每个人（骑手、司机、后端）都应该与彼此的意图同步。

司机应用程序可以每隔几秒钟轮询一次服务器，以检查是否有新的报价可用。骑手应用程序可以每隔几秒钟轮询一次服务器，以检查是否分配了司机。 

这些应用程序的轮询频率取决于它们轮询的数据的变化率。在像 Uber 应用这样的大型应用中，变化率的变化范围非常广泛，从几秒到几小时不等。


