# **KubeGPU: efficient sharing and isolation mechanisms for GPU resource management in container cloud**

### 背景知识

#### **目前的几种在K8S使用GPU资源的方法：**

1、**为申请GPU资源的容器分配独占的GPU资源**。（提供本地GPU资源，对容器有良好的适配性）

2、**在容器之间共享一个物理GPU**，可以提高GPU利用率和整体系统吞吐量。（有些程序直接将整个物理GPU分配给一个或多个容器，而有些程序则对物理GPU进行分区将GPU划分为多个vgpu (virtual GPU)，并将一个或多个vgpu分配给申请GPU资源的容器。）

文章中提到已经有完成了扩展Kubernetes以支持远程GPU虚拟化的工作，使运行在非GPU节点上的容器共享GPU节点的GPU资源以实现任务加速。（Accelerated serverless computing based on GPU virtualization）

#### **现有方法的局限性：**

1、第一类方法将独占的GPU分配给容器，**这通常会导致资源利用不足和系统吞吐量低**；第二类方法通过在容器之间共享GPU来提高总体系统吞吐量。（但因为GPU产品大多没有开源的缘故，所以**很难实现细粒度分配**；API转发在driver级别的虚拟化也受到**性能开销和功能不完备**的限制）

2、以前Kubernetes中关于远程GPU虚拟化的工作主要集中在GPU加速上。因此，**通信开销和共享资源干扰的问题仍然没有得到解决**。

**背景知识：**

|          方法           | GPU共享 | 细粒度  |  远程  | 灵活资源分配 | 网络自适应 | 兼容性  |
| :-------------------: | :---: | :--: | :--: | :----: | :---: | :--: |
|   k8s-device-plugin   |   ❌   |  ❌   |  ❌   |   ❌    |   ❌   |  ⭕   |
|      Deepomatic       |   ⭕   |  ❌   |  ❌   |   ❌    |   ❌   |  ⭕   |
| GPU sharing Scheduler |   ⭕   |  ❌   |  ❌   |   ❌    |   ❌   |  ⭕   |
|        ConVGPU        |   ⭕   |  ❗   |  ❌   |   ❌    |   ❌   |  ❗   |
|       Gaia GPU        |   ⭕   |  ⭕   |  ❌   |   ❌    |   ❌   |  ❗   |
|       KubeShare       |   ⭕   |  ⭕   |  ❌   |   ❌    |   ❌   |  ❗   |
|       DynamoML        |   ⭕   |  ⭕   |  ❌   |   ❌    |   ❌   |  ❗   |
|        Satzke         |   ⭕   |  ⭕   |  ❌   |   ❌    |   ❌   |  ❗   |
|         Oscar         |   ⭕   |  ⭕   |  ⭕   |   ❌    |   ❌   |  ❗   |
|        KubeGPU        |   ⭕   |  ⭕   |  ⭕   |   ⭕    |   ⭕   |  ⭕   |

### 面临的问题

​	虽然GPU共享在VM上已经得到了广泛的研究，但在容器上的研究却非常有限。现有的工作只使用一种特定的GPU虚拟化技术来部署容器，如GPU直通或API转发，缺乏远程GPU虚拟化优化。

### 设计的解决方案

1. **（GPU虚拟化的动态选择）******自适应共享策略****，该策略可以根据可用的GPU资源和容器的配置参数(如GPU资源需求)
2. **（最小化Kubernetes中远程GPU虚拟化的通信开销）**提出了一种**网络感知调度方法**，通过分析底层云网络和分配可用的、更好的网络性能的容器和网络设备来最小化通信开销
3. **（消除远程GPU资源竞争）*****采用细粒度分配的方式**，允许用户指定对远程图形处理器的资源需求，根据用户对GPU计算时间和内存空间的资源需求，将远程图形处理器的使用在各个容器之间隔离开来。

### **实现的效果**

使用HPC和深度学习的典型现实工作负载，展示了KubeGPU与其他现有工作相比的优势，以及KubeGPU在**最小化通信开销**和**消除远程GPU资源竞争**方面的有效性。



### 架构设计

#### 整体架构：

![img](https://api2.mubu.com/v3/document_image/e23a99c7-b051-4339-a56b-8ac5e228e1d8-15661181.jpg)

#### 各个部件功能：

KubeGPU Scheduler ：一个由一系列处理程序组成的行为设计模式；GPU资源处理器和网络模式处理器。

Device Manager：根据KubeGPU Scheduler的分配请求从资源池中获取设备资源。然后，设备管理器创建容器，根据设备资源为容器分配设备，并更新对应设备资源的剩余容量。（设备管理器支持计算资源弹性分配，即当容器使用的GPU计算资源超过其请求的资源需求时，对应的物理GPU有空闲的计算资源，容器仍然可以获得GPU计算资源。）

Device Agent ：实现设备代理来通告所有设备资源到设备管理器

Resource Pool ：是“GPU设备资源”和“NET设备资源”两种设备资源的集合。

Device Resource ：是对集群内物理设备进行灵活管理的逻辑抽象，包含了物理设备的剩余容量、物理位置、设备类型等信息。

#### 工作流程：

1. **Device Agent**向**Device Manager**注册自己并发布可用的**Device Resource**。**Device Manager**将新的可用**Device Resource**添加到**Resource Pool** 。


2. **KubeGPU Scheduler** 接收**client**发送的请求。
3. 在删除请求的情况下，**KubeGPU Scheduler** 发送一个释放请求到**Device Manager**。**Device Manager**将删除相应的容器，释放所有关联的设备，并更新相应资源的剩余容量“资源池中的设备”。


4. 在创建请求的情况下，**KubeGPU Scheduler**从**Device Manager**查询可用的**Device Resource**，并根据可用的设备制定调度策略资源和容器配置。**Device Manager**通过查询设备代理获取可用的**Device Resource**，设备代理将物理设备的监控数据返回给**Device Manager**。**Device Manager**还根据查询结果维护“资源池”。


5. **KubeGPU Scheduler**要求**Device Manager**通过发送分配请求来创建一个容器。
6. **Device Manager**接收**KubeGPU Scheduler**发送的分配请求，**Device Manager**创建容器，为容器分配设备，并根据分配请求更新“资源池”中对应的“资源设备”的剩余容量。


7. **Device Manager**监视分配设备的运行容器，以确保这些容器的设备使用可以限制在它们请求的资源需求之下。

​    GPU计算资源按**时间片共享**，GPU内存资源按**空间共享**。GPU计算资源对容器性能影响较大，GPU计算资源在执行后立即释放，而不是等待容器完成。但是GPU的内存资源对容器的性能影响很小，直到完成，容器才会释放内存资源。因此,KubeGPU仅采用弹性资源分配机制对GPU计算资源进行分配。

### 弹性计算资源分配

通过令牌来实现此流程，此处的优先级由两个因素决定：

![img](https://api2.mubu.com/v3/document_image/c203453a-f983-40e5-9331-1ccf3588180b-15661181.jpg)

1. GU_BIAS 容器的计算资源需求与容器实际GPU利用率之间的差值，即计算资源需求(gucore)减去实际GPU利用率。
2. GU表示最大GPU利用率(100%)与容器实际GPU利用率的差值，即最大GPU利用率减去容器实际GPU利用率。GU值越高，表示容器的GPU利用率越低。


### 适应性资源共享策略

两个分支第一是用户设置参数，第二是容器的计算资源需求动态选择GPU虚拟化。

![img](https://api2.mubu.com/v3/document_image/2cf1c4c6-4a1c-4bb5-bc21-e00f4861b86e-15661181.jpg)

### 网络自感知调度

为了适应GPU远程虚拟化调用而设计的算法，网络感知调度方法背后的逻辑是通过分析底层云网络尽可能地使用更高性能的网络资源和网络模式

![img](https://api2.mubu.com/v3/document_image/f68d6735-cff3-4d81-96d9-70f715cec833-15661181.jpg)

### 细粒度分配

这里所谓的分配方式是硬限制方式，即消耗的资源超过其请求时将不会分配资源，然后调用上文中提到的弹性计算资源分配进行分配

![img](https://api2.mubu.com/v3/document_image/5840b00a-f811-4566-b4d7-17b63bfbbddd-15661181.jpg)

### 实验部分

![img](https://api2.mubu.com/v3/document_image/630470cd-d68f-44d8-8d64-80d8119985a7-15661181.jpg)

不同架构的对比

![img](https://api2.mubu.com/v3/document_image/52048b77-2df4-464d-83bc-f42425fbfbd8-15661181.jpg)

随着容器数量增长发生的变化

![img](https://api2.mubu.com/v3/document_image/90f6ac58-d3ef-4b40-b137-92c88db7adce-15661181.jpg)

x轴为可用容器数量，y轴为执行时间

![img](https://api2.mubu.com/v3/document_image/67092a16-4d3d-414a-9e15-53a160157fb6-15661181.jpg)
