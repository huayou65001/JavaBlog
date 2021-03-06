# Dubbo

<!--ts-->
- [Dubbo](#dubbo)
  - [一个网站的架构模型是怎样的？](#一个网站的架构模型是怎样的)
  - [RPC](#rpc)
  - [Dubbo的相关介绍](#dubbo的相关介绍)
  - [服务暴露](#服务暴露)
  - [服务引用](#服务引用)
  - [服务调用](#服务调用)
  - [SPI](#spi)
    - [为什么Dubbo不用java的SPI，而要自己实现？](#为什么dubbo不用java的spi而要自己实现)
    - [Adaptive注解：自适应扩展](#adaptive注解自适应扩展)
  - [如何设计一个RPC](#如何设计一个rpc)

<!-- Added by: hanzhigang, at: 2021年 8月17日 星期二 13时40分29秒 CST -->

<!--te-->

## 一个网站的架构模型是怎样的？
1. 单一应用架构：当网站流量很小的时候，将所有功能都部署在一起，以减少部署节点和成本，
2. 垂直应用架构：当访问量主键增大，单一应用增加机器带来的加速度越来越小，提升效率的方法之一是将应用拆成互不相干的几个应用，以提升效率。
3. 分布式服务架构：当垂直应用越来越多，应用之间交互不可避免，将核心业务抽取出来，作为独立的服务，逐渐形成稳定的服务中心，使前端应用能更快速的响应多变的市场需求。此时，用于提高业务复用以及整合的分布式服务器框架是关键。
4. 流动计算架构：当服务越来越多，容量的评估，小服务资源的浪费等问题逐渐显现，此时需增加一个调度中心基于访问压力实时管理集群容量，提高集群利用率。此时，用于提高机器利用率的资源调度和治理中心是关键。
## RPC
RPC(Remote Procedure Call)：远程过程调用，它是一种通过网络从远程计算机程序上请求服务，而不需要了解底层网络技术的思想。计算机 A 上的进程，调用另外一台计算机 B 上的进程，其中 A 上的调用进程被挂起，而 B 上的被调用进程开始执行，当值返回给 A 时，A 进程继续执行。调用方可以通过使用参数将信息传送给被调用方，而后可以通过传回的结果得到信息。而这一过程，对于开发人员来说是透明的。
## Dubbo的相关介绍
- Dubbo是阿里巴巴开源的一个基于Java的RPC框架。主要分为一下几个角色
- Consumer：需要调用远程服务的服务消费方
- Provider：服务提供方
- Container：服务运行的容器
- Monitor：监控中心
- Registry：注册中心

- 整体流程：服务提供方启动然后向注册中心注册自己能提供的服务。服务消费方启动向消费中心订阅自己所需的服务，然后注册中心提供元信息给服务消费方。因为服务消费方已经从注册中心获取了提供者的地址，因此可以通过负载均衡选择一个服务提供方进行直接调用。当服务消费方的元数据变更的话注册中心会把变更推送给服务消费方。服务提供方和消费者都会在内存中记录调用的次数和时间，然后定时的发送统计数据给监控中心。


## 服务暴露
服务的暴露起始于Spring IOC容器刷新完毕之后，会根据配置参数组装成URL，然后根据URL的参数进行本地调用或者远程调用。
会通过`proxtFactory.getInvoker`，利用javassist来进行动态代理，封装真正的实现类，然后再通过URL参数选择对应的协议来进行`protocol.export`，默认是Dubbo协议。
在第一次暴露的时候会调用`createServer`来创建Server，默认是NettyServer。
然后将export得到的exporter存入一个map中，供之后的远程调用查找，然后会向注册中心注册提供者的消息。

## 服务引用
服务的引入又分为了三种，第一种是本地引入、第二种是直接连接引入远程服务、第三种是通过注册中心引入远程服务。
服务的引入时机有两种，一种是饿汉式，一种是懒汉式。
饿汉式就是加载完毕就会引入，懒汉式是只有当这个服务被注入到其他类中时启动引入流程，默认是懒汉式。
会先根据配置参数组装成URL，一般而言我们都会配置注册中心，所以构建RegistryDirectory向注册中心注册消费者的消息，并且订阅提供者、配置、路由等节点。
得知提供者的信息之后就会进入Dubbo协议的引入，会创建Invoker，期间会包含NettyClient，来进行远程通信，最后通过Cluster来包装Invoker(可能一个服务有多个提供者)，默认是FailoverCluster，最后返回代理类。

## 服务调用
调用某个接口的方法会调用之前生成的代理类，然后会从cluster中经过路由的过滤、负载均衡选择一个Invoker发起远程调用，此时会记录此请求和请求的ID等待服务端的响应。
服务端接受请求之后会通过参数找到之前暴露存储的map，得到相应的exporter，然后最终调用真正的实现类，再组装好后结果返回，这个响应会带上之前请求的ID。
消费者收到这个响应之后会通过ID去找之前记录的请求，然后找到请求之后将响应塞到对应的future中，唤醒等待的线程，最后消费者得到响应，一个流程完毕。

## SPI
SPI是Service Provider Interface，主要用于框架中，框架定义好接口，不同的使用者有不同的需求，因此需要有不同的实现，而SPI就通过定义一个特定的位置，约定在ClassPath的/META-INF/services/目录下创建一个以服务接口命名的文件，然后文件里面记录此jar包提供的具体实现类的全限定类名。所以就可以通过接口找到对应的文件，获取具体的实现类然后加载即可。

### 为什么Dubbo不用java的SPI，而要自己实现？
因为 Java SPI 在查找扩展实现类的时候遍历 SPI 的配置文件并且将实现类全部实例化，假设一个实现类初始化过程比较消耗资源且耗时，但是你的代码里面又用不上它，这就产生了资源的浪费。因此 Dubbo 就自己实现了一个 SPI，给每个实现类配了个名字，通过名字去文件里面找到对应的实现类全限定名然后加载实例化，按需加载。

### Adaptive注解：自适应扩展
- 一个场景：首先我们根据配置来进行SPI扩展的加载，但是我不想在启动的时候让扩展被加载，我想根据请求时候的参数来动态选择对应的扩展。
- Dubbo通过一个代理机制实现了自适应扩展，简单地说就是为你想扩展的接口生成一个代理类，可以通过JDK或javassit编译你生成的代理类代码，然后通过反射创建实例。
- 这个实例里面的实现会根据本来方法的请求参数得知需要的扩展类，然后通过`ExtensionLoader.getExtensionLoader(type.class).getExtension(从参数得到的name)`，来获取真正的实例调用。

## 如何设计一个RPC
- 首先需要实现高性能的网络传输，可以采用 Netty 来实现，不用自己重复造轮子，然后需要自定义协议，毕竟远程交互都需要遵循一定的协议，然后还需要定义好序列化协议，网络的传输毕竟都是二进制流传输的。
- 然后可以搞一套描述服务的语言，即 IDL（Interface description language），让所有的服务都用 IDL 定义，再由框架转换为特定编程语言的接口，这样就能跨语言了。
- 此时最近基本的功能已经有了，但是只是最基础的，工业级的话首先得易用，所以框架需要把上述的细节对使用者进行屏蔽，让他们感觉不到本地调用和远程调用的区别，所以需要代理实现。
- 然后还需要实现集群功能，因此的要服务发现、注册等功能，所以需要注册中心，当然细节还是需要屏蔽的。
- 最后还需要一个完善的监控机制，埋点上报调用情况等等，便于运维。
