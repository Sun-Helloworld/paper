# Accelerated serverless computing based on GPU virtualization

### 背景知识

**Serverless**的全称是Serverless computing无服务器运算，又被称为函数即服务（Function-as-a-Service，缩写为 FaaS），是云计算的一种模型。以平台即服务（PaaS）为基础，无服务器运算提供一个微型的架构，终端客户不需要部署、配置或管理服务器服务，代码运行所需要的服务器服务皆由云端平台来提供。

rCUDA在runtime上封装；rCUDA中可以使用不同的通信协议，如Infiniband、RoCE (RDMA over Converged Ethernet)等，以优化数据交换以及TCP/IP协议。

### 面临的问题及需求

(1)限制了最大执行时间，对长时间运行的科学应用不可行;   

(2)资源有限，因为资源密集型的科学应用程序可能需要超过3008 MB的RAM(目前AWS Lambda的最大内存大小);       

(3)受限的执行环境，因为科学应用程序通常需要各种各样的库;

(4)有限的临时存储，因为512 MB的磁盘空间不足以承载大依赖的应用程序的执行;

(5)在高负载的函数调用中无法访问GPU资源

需要的是可以使用GPU加速的执行长时间资源敏感型的任务

### 本文的解决方案

介绍了一种支持无服务器计算的可扩展事件驱动数据处理平台；该平台基于Docker容器，使用k8s进行集群管理，评估了几种涉及到本地或者远程GPU虚拟化的访问方法。（使用rCUDA对OSCAR平台进行了改进，从而允许函数之间有效地共享gpu，以支持多租户环境。）

![img](https://api2.mubu.com/v3/document_image/fa045159-3fd1-4d45-8f9b-6dae9f2466fc-15661181.jpg)

### 基于gpu的超声心动图分类

#### 设计了四个场景

- **Python console**
  代码是在一个Docker容器中的Python控制台中执行的,主要缺点是对代码的每个输入和输出进行手工配置，远远不是一个自动化的过程。主要是确定一个basline
- **OSCAR+CPU**
  为了确定从CPU到GPU的改进
- **OSCAR+Remote GPU (rCUDA)**
- **OSCAR+Native GPU**

#### 效果

**（采用10G网卡）；加载训练模型以及Tensorflow和Keras库所需的平均时间**

![img](https://api2.mubu.com/v3/document_image/11686d34-a166-453a-829a-55a15eb50ef0-15661181.jpg)

**显示了整个处理时间，包括视频分割成图像的时间，加载库的时间(如图5所示)，以及图像分类的时间**![img](https://api2.mubu.com/v3/document_image/8296fc2c-c259-4d75-9714-954468f7e2e6-15661181.jpg)

处理多个视频的场景

![img](https://api2.mubu.com/v3/document_image/56325468-e58d-4802-b19b-47dd9f4fd0b1-15661181.jpg)

### 结论

虽然GPU虚拟化和无服务框架的结合有些缺陷，但将rCUDA集成到OSCAR架构中，在无服务器计算和Kubernetes集群中多个应用程序共享GPU访问方面取得了重要进展。