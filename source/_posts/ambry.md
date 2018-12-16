---
title: ambry LinkedIn 对象存储 论文翻译
date: 2018-12-13
categories: [分布式系统,分布式存储]
tags:
    - 对象存储
    - 分布式系统
    - 存储
    - SIGMOD
    - ambry
---

# 摘要
全世界社交网络之下的基础设施，需要为数十亿的大小可变的媒体对象提供不间断的服务，例如照片，视频和音频切片。这些对象必定被存储在一个低延时高吞吐的系统中。这个系统需要支持跨地区、可扩展，并提供负载均衡能力。当存储大文件的时候，现有的文件系统和对象存储面临着一些挑战。我们推出 Ambry，一个针对大规模不可变对象（称之为blob）的生产级系统。Ambry 设计为一个分布式的系统，并且使用了如下技术：逻辑 blob 组，异步复制，再均衡机制(译注:数据再均衡)，zero-cost 失败检测机制和系统缓存。Ambry 已经在 LinkedIn 生产环境中运行了 2 年，以高达 1万次/s 请求的质量，为超过 4 亿用户提供了服务。我们的实验显示，Ambry 提供了一个高效(使用了 88% 的带宽)，低延时(1MB 文件延时小于50ms)，负载均衡(相对于未进行负载均衡提高了 8-10 倍性能)的系统。
# 关键词
对象存储，跨地区分布，可扩展，负载均衡
# 1. 介绍
在过去的十年中，社交网络已经变成了全世界最受欢迎的社交方式，数亿的用户不间断的上传和阅读数十亿不同类型，从照片和视频到文档，的媒体对象。这些我们称为 blob 的大尺寸媒体对象，上传一次，被全世界各地频繁的访问，从不改变、很少删除。LinkedIn 作为一个全球性的大规模的社交网络公司，急需一个可伸缩、高效存储和频繁召回 blob 的跨地区分布式系统。急需一个可伸缩、高效的跨地区分布系统来存储和检索这些 基本只读(read-heavy) 的 blob。

处理blob对象会面临许多独一无二的挑战。第一，由于媒体文件的多样性，blob 的大小变化非常剧烈，从数十 KB(例如头像图片)到几 GB(例如视频)。系统需要同时支持高效地存储超大文件和海量的小文件。第二，需要存储和服务的 blob 在不断增长。到目前为止，LinkedIn 每天为超过 80 亿的 put/get (总数据量超过 120TB) 操作提供服务。过去的 12 个月中，从 5000 请求/s 到 9500 请求/s，请求的频率增长几乎翻倍。请求频率的迅速增长，更加加剧了对可线性扩展系统(并且需要低的开销)的需求。第三，负载变化和当集群扩容时可能造成负载不均，造成系统延时和吞吐下降。这个导致了负载均衡的需求。最后，用户希望上传的过程很快，能够持久化，并且高可用。当用户上传了一个blob，即使是当一部分内部的基础设施失效的时候，ta 所有的朋友能够从全世界任何地方，以一个很低的延时看到 ta 上传的blob。为了提供这些功能，数据必须能够可靠的复制到全球的多个数据中心，同时为每个请求提供低延时的服务。

LinkedIn 曾经有一套自研的解决方案，称为 Media Server，由 NAS(用于存储文件)，Oracle数据库(用于存储 metadata) 和运行了 Solaris 系统的机器构成。Media Server 有着许多缺点。因为许多小文件的存在，造成了海量的元数据操作，该系统面临着 CPU 和 IO 的限制，而且还不能水平扩展、非常昂贵。考虑到 LinkedIn 扩张的非常迅速，并且未来的网页将会以媒体文件为主，我们需要找到一个代替方案。

已经有一些系统被设计用来处理大量的数据，但是没有一个能够完全符合 LinkedIn 所需要的规模和需求。已经有对分布式系统 [10, 16, 22, 24, 28] 的广泛研究。比如在 [3, 11] 中指出，当存储 blob 时，这些系统有着一些限制。例如，层级目录和丰富的元数据，对于 blob 存储是多余的，增加了许多不必要的开销。

也有许多 kv 存储 [2, 5, 8, 14] 被设计用来存储大量的对象。尽管这些系统能够处理许多小的对象，但他们没有针对大对象进行优化(数十 MB 到数 GB )。此外，这些系统为了提供一致性保障，增加了额外的开销。然而这种保障对于不可变数据来说一般是多余的。所需的开销举例来说有：使用向量时钟，冲突解决机制，日志和中间协调者。

一些系统已经被专门设计用来存储大量不可变对象，包括 facebook 的 haystack[3]，以及继任者 f4[18] 和 Twitter 的 Blob Store[27]。然而这些系统没有解决负载不均的问题，这个问题在集群扩容的时候尤其严重。
在这篇文章中， 我们将介绍 Ambry，一个被设计用来处理海量 blob 的生产级系统，blob 具有这些特性：尺寸不定、可大可小，一次写多次读(读的流量大于 95%)。Ambry 的设计主要考虑到如下四个目标：

**1) 低延时、高吞吐：** 系统每秒都需要及时处理大量的请求，并且运行在廉价的通用硬件上(例如机械盘)。为了达到这个目的，Ambry 使用了许多技术：利用系统缓存，当数据从硬盘发送到网络时使用 zero copy，分块(chunk)数据可以使用多个节点并行存储和召回，读写 replica 数量的策略可配置，zero-cost 失败探测机制。(详见 2.3, 4.2, 4.3 章节)

**2)跨地区操作：**  为了实现即使出现故障也可用和持久化，Blob必须复制到地理上分布不同的其他数据中心。为了在跨数据中心的环境中，实现低延时、高吞吐，Ambry 设计为分布式(decentralized)的多主系统，数据可以由任何一个 replica 读或写。此外它还使用了异步写：异步写入数据到最近的数据中心，异步复制数据到其他数据中心。同时为了高可用，使用 proxy 机制：当数据还没有复制到当前的数据中心时，会将请求转发到其他的数据中心。（详见 2.3, 4.2 章节）

**3)可扩展性：** 随着日益增长的数据量，系统必须拥有横向扩展能力，同时开销(译注:系统扩展开销)要低。为了实现这个目标，Ambry主要做了三个设计方面的权衡，第一，Ambry 将 blob 的逻辑存储与物理存储分离，允许更改物理位置时对逻辑位置透明。第二，Ambry 设计为完全分布式系统，没有 manager/master。第三，为了索引 blob 的可扩展性和效率，Ambry 使用了带有布隆过滤的分段索引，索引维护在硬盘中，最新的索引缓存在内存中。（详见 4.3 章节）

**4)负载均衡：** 尽管数据在增长，但是系统仍然需要保持平衡。在静态集群中 Ambry 通过将大 blob 分块并随机选择存储的方式保证数据平衡。再均衡(re-balancing)机制则保证当集群扩容后会恢复到均衡状态。（详见 3 章节）
Ambry 已经在生产环境中成功的运行 24 个月，跨 4 个数据中心，服务超过 4 亿用户。我们的实验结果显示，Ambry 已经达到高吞吐（接近网络带宽的 88%）和低延时(50ms 内处理完成 1MB 的 blob)的目标，高效运行在多个跨地区的数据中心，并在尽可能少移动数据的前提下，解决数据不均的问题，将效率提高了约 8-10 倍

# 2. 系统概览
在这个章节我们讨论了Ambry的总体设计，包含了系统的高层架构(2.1)，partition的概念(2.2)和支持的操作(2.3)

![image](https://raw.githubusercontent.com/phantooom/blog/master/image/ambry/01.jpg)

## 2.1 架构
Ambry被设计为一个全分布式的系统，可以跨地区数据中心。整体的架构展示在了图1中。系统主要有三部分组成：Frontends: 用来接收和路由请求。Datanodes:存储真实数据。 Cluster Managers：维护集群状态信息。每个数据中心拥有并且分布式的运行自己的一套组件。Frontends 与 Datanodes 之间是完全独立的。Cluster Managers之间是同步的，同步通过zk实现。我们在下边提供了各组件的概览(详细的介绍在4章)。

**Cluster Manager**: Ambry 组织了数据的虚拟单元称为 partition（2.2 章）一个partition是由若干blob组成的逻辑分组。使用大的replicated文件实现。起初，partition是可读可写的。例如 blob可以读同时也可以添加。当一个逻辑的partition接近他的容量时，将会转变为只读状态，Cluster Manager持续跟踪状态(可读可写/只读) 同时 跟踪每个partition replica的位置。还跟踪了器群的物理布局（节点和磁盘位置）。

**Frontend**: 在多租户环境中Frontends 负责接收和路由请求。系统处理三种类型的请求：put get delete。 热数据会由Ambry之上的CDN处理。Frontends直接从客户端接收请求，或者通过CDN接收。Frontends转发相应的请求到相应的Datanode 并且返回相应的数据给客户端，或者返回给CDN。

**Datanode**: Datanode存储和召回真实的数据。每个Datanode管理一组磁盘。为了获得更好的性能，Datanode维护了许多额外的数据结构，包括：blob索引，journals 和 布隆过滤器（4.3章）。

## 2.2 Partition
为了避免直接将blob 映射到物理机，例如 Chord [26] and CRUSH [29],Ambry 随机的将blob分组到一个虚拟的单元称之为 Partition。partition的物理安置是由一个独立的程序处理。这将逻辑位置和物理位置进行了解耦，对数据迁移透明（对数据再均衡很必须要）同时可以避免在集群扩展时立刻再哈希(rehashing)。

一个partition的实现是使用一个预分配的大的append-only的日志文件实现的。目前，在系统的生命周期中，partition是固定大小的。partition的大小需要足够大，来抵消partition的额外开销。例如，会为每个partition维护额外的数据结构，像索引, journal，布隆过滤器(4.3章),当partition足够大时这些开销是微不足道的。另一方面，故障恢复和重建的时候应该很快。我们在我们的集群中使用100GB 的partition。因为重建可以并行的从多个replica中进行，我们发现即使是100GB大小的partition可以在几分钟内重建完毕。

![image](https://raw.githubusercontent.com/phantooom/blog/master/image/ambry/02.jpg)

Blob是顺序的写入到partition中用作put和delete 的 entry(图2)。所有的entry都包含一个header(存储entry中field的偏移量)和blob id。 blob id是唯一标识，由Frontend 在put/delete 生成。在get/delete 操作时候用来定位blob。这个id包含partition id，代表blob的位置(8Bytes),后面接着是32Bytes的UUID，代表blob的id。id冲突可能存在，但是概率非常低(< 2^230)。当冲突发生时两个put操作必须生成同样的id并且写到同一个partition。冲突会在Datanodes中解决，将后put的失败即可。

put entries 也包含了预定义的属性，包括blob size， ttl，创建时间，content type。同时也支持用户定义的属性。

为了提供该可用性和故障容错，每个partition 被复制到多个Datanodes。每个replica使用贪心算法安置在磁盘上。这个算法选择一个，未使用空间最大的磁盘，同时要遵守如下限制条件：1）每个Datanode(相同partition)不能安置超过一个replica。2）replica需要存在于多个数据中心，目前replica数量可以由系统管理员配置。未来一部分的工作是，我们打算支持在热的partition中自动的增加replica数。同时对于冷的数据我们使用纠删码来进一步减少副本数量。

起初，partition是可读可写，能处理所有的操作（put get delete）。当partition 到达他的极限大小时(容量极限)，会变为只读的，此后只处理 get 和 delete 请求。

容量极限应该略微的比最大容量小(80%~90%),主要由于两个原因，第一，在变为只读之后，replica可能没有完全的同步，需要空闲的空间去追赶落后的部分（因为是异步的写)。第二删除请求仍然需要添加delete entry。

delete 与 put 操作类似，但是delete作用于已经存在blob上，默认的情况下，删除的结果是为一个blob 添加一个delete的entry(带有delete 标记，软删除)。被删除的blob会周期性的清理，使用一个内部的compaction机制。在compaction之后，只读的partition如果存在足够的空闲空间，则会转变为可读可写的。文章的剩余部分我们主要关注put，因为它与delete类似。

![image](https://raw.githubusercontent.com/phantooom/blog/master/image/ambry/03.jpg)
## 2.3 操作
Ambry拥有轻量级的API，仅支持三个操作: put ,get ,delete。API请求的处理过程在图三中进行了展示。当收到请求时，Frontend有选择的对请求进行一些安全检查，然后使用Router库(包含了最核心的处理逻辑)，Router库随机选择一个partition，负责与Datanode进行通信，同时处理请求。在put操作中，partition是随机选择的(主要目的是为了数据的均衡)，并且在get/delete 操作中,partition是从blob id中获取的。

操作被设计为可以被多个master处理，操作可以由任何一个replica处理。具体要与多少replica进行通信是由用户策略定义的。这些策略类似于Cassandra中的一致性级别，用来控制一个操作要涉及多少个(1,k, 大多数，所有)replica参与。对于put(或者delete)请求会转发到所有的replica，并且策略定义了返回成功(为了权衡持久性和延时)需要确认(写入)的数量。对于get，策略决定了需要随机选择多少个replica进行通讯（为了权衡负载和延时)，实践中我们发现对于所有的操作 k=2时的策略 我们能够获得我们所期望的平衡点(负载，延时，持久性的平衡)。更严格的策略(包含所有replica)将会提供一个更强的一致性保障。

此外，在跨数据中心的场景下，执行写操作时候，同步写所有的replica，将会影响延时和吞吐。为了能够缓解这个问题，Ambry使用了异步写操作，put操作只同步写本地数据中心，例如一个数据中心的Frontend接收到了一个本地的请求，处理后这个请求被认为已经成功了。稍后，blob会使用一个轻量级的复制算法，被复制到其他的数据中心。(5 章)

为了在一个尚未blob尚未被复制到的数据中(在数据中心A写在数据中心B读)，提供read-after-write(写完后立刻可读)一致性。Ambry使用了代理请求的方式，如果Frontend不能从本地数据中召回blob，他会将请求代理到另外一个数据中心，将那边的结果返回。尽管代理请求消耗非常大，但是在实践当中我们发现代理请求发生的频率很低(小于0.001%)。

# 3 负载均衡
负载倾斜，大量的大blob，和集群扩容，造成了负载不均，并且对系统的延时和吞吐造成了影响。Ambry在动态和静态集群中都实现了负载均衡功能(根据磁盘使用率和请求速率)。

**Static Cluster**：将大的blob分割为多个小的块（4.2.1章）同样通过Router库执行put操作，随机选择一个parition，来实现partition 大小的负载均衡。此外使用大小相同的partition，依赖CDN处理非常热门的内容，可以显著降低产生热点partition可能。使用这些技术后，在生产环境中，负载不均的请求和大小不均的partition占所有Datanodes 5%。

**Dynamic Cluster**：在实践中，可读可写partition 接收所有的写流量，同时承担了主要的读流量。因为partition是以一种半平衡(semibalanced)的方式增长.可读可写的partition数量成为了负载不均的主要因素。在集群扩展之后，新的Datanode 包含了绝大部分可读可写的partition，然而老的Datanode包含了绝大部分只读的partition。这种可读可写partition倾斜的分布造成了系统的负载不均。在我们起初的版本中，新的Datanode得请求比例比老的高100倍，比平均值高10倍。

为了解决这个问题，Ambry引入了一个再均衡机制，移动最少的数据使集群变为半平衡（semi-balanced）状态（针对磁盘使用率和请求而言）。再均衡实现了减少请求比例和硬盘使用率，分别达到了6-10倍和9-10倍。

Ambry 定义了 ideal 状态 (已经负载均衡的状态)，ideal状态包含三个(idealRW=totalNumRW / numDisks, idealRO=totalNumRO / numDisks, idealUsed=totalUsed / numDisks）可以由 只读/可读可写的总量 /硬盘使用的总空间 除以集群中硬盘的总数得到。如果一个磁盘有更多的(或更少的)只读/可读可写的partition/磁盘使用率，会被认定为高于(或低于)ideal。

再均衡算法尝试到达ideal状态，通过使用将高于ideal状态磁盘的partition移动到低于 ideal状态的磁盘来实现。主要分为两个阶段如下代码所示：
![image](https://raw.githubusercontent.com/phantooom/blog/master/image/ambry/04.jpg)

阶段1-移动到partition池：在这个阶段，Ambry将高于ideal状态磁盘的partition移动到一个池子里，这个池子称为 partition pool (6-13行)，这个阶段的结尾，不会有磁盘高于ideal状态，除非移动了partition会造成磁盘低于ideal状态。

Ambry从可读可写的partition开始(最主要的影响因素)，基于idealRW极限，移动额外的partition。同样的过程也在只读的partition中重复。但是通过考察idealRO 和 idealUsed指标，然后进行移动。选择移动那个partition的策略是基于最小数据移动确定的。对于可读可写的partition，就是选择一个容量使用最小的partition。对于只读的partition，就是进行一个随机的选择，因为所有的只读的partition都是写满了的。

阶段2- 安置partition到磁盘：在这个阶段,Ambry将partition从partition池移动到低于ideal(14-16行)的磁盘，先移动可读可写的partition，然后移动只读的partition。partition安置采用了round-robin的方式(17-22)。Ambry找出所有的磁盘都在ideal之下，将他们进行打乱，然后使用round-robin的方式将partition安置到他们上面，这个过程一直进行直到partition 池为空。

在找到新的安置处后，replica 将会无缝的删除1)创建印个新的replica 在目的地。2）通过同步协议，将从旧的replica同步到新的replica，这时所有的replica都能提供写服务。3）当同步完成时删除旧的replica。

# 4. 组件详解
这章我们将更详细的讨论ambry的主要组件。我们将详细的描述Cluster Manager(4.1章)存储的状态，此外还讨论了Frontends的职责包括分块和失败检测（4.2章），还有Datanodes（4.3章）维护的数据结构。
## 4.1 Cluster Manager
Cluster Manager 负责维护集群的状态，每个数据中心有一个本地的Cluster Manager实例，使用zk与其他实例保持同步。Cluster Manager存储的，由硬件和逻辑布局组成的状态非常小(总和小于几MB)。
![image](https://raw.githubusercontent.com/phantooom/blog/master/image/ambry/05.jpg)
### 4.1.1 硬件布局
硬件布局包含了集群的物理结构信息。例如数据中心，Datanodes, 磁盘的分布。同样也为每个磁盘维护了原始容量和状态，例如健康(UP) 或者故障(DOWN)。表1是一个硬件布局的例子，如表所示，Ambry工作在一个异构的环境中，不同的硬件不同的配置不同的数据中心。
![image](https://raw.githubusercontent.com/phantooom/blog/master/image/ambry/06.jpg)
### 4.1.2逻辑布局
逻辑布局维护了partition的replica的物理位置。也维护了每个partition的状态(可读可写/只读)。为了检测partition的状态，manager周期行了与Datanode进行通信，请求他们partition的状态。逻辑布局用于选择一个partition安置新创建的blob(put 操作)，也用于定位一个给定的replica的Datanode(所有操作)。一个示例布局如表2所示，partition的replica可以被安置在一个数据中心的多个Datanode，也可以在多个数据中心。此外一块硬盘(例如 DC 1：Datanode 1: disk 1) 可以包括不同partition的replica，有一些只读，有一些可读可写。添加一个partition是通过更新Cluster Manager实例存储的逻辑布局实现的。
## 4.2 Frontend 层
Frontend 是外部请求的实体，每个数据中心拥有一组自己的Frontend。Frontend是完全分布式的没有master或者coordination，是无状态的，完全一样的处理任务,所有的状态存储在Cluster Manager中（周期性的拉取）。这种设计对Frontend来说能够，提高扩展能力(新的Frontend加入不会有太多的性能代价)，和故障容忍能力(请求可以被转发到任意一个Frontend)，和故障恢复能力（故障的Frontend可以很快的被取代）。Frontend主要有三个职责：

1. 请求处理：这个包含了接收请求和使用Router库(4.2.1章)将他们路由到相应的Datanode.并且将返回response。

2. 安全检查：可以选择性的执行检查，例如病毒扫描和请求认证。

3. 捕获操作：由于Ambry缺乏离线分析的功能，推送事件到变更捕获系统，例如发现系统的请求模式，我们使用kafka作为我们的变更捕获系统，主要由于他提供了高持久性，高吞吐，和低开销的能力。

### 4.2.1 Router 库
Router库有处理请求和与Datanode通信的核心逻辑。Frontend简单的嵌入了并使用了这个库。客户端可以植入这个库直接处理请求 绕开(bypass)Frontend。这个库包括了四个主要部分.1)策略路由。2）大blob分块。3）失败检测。4)代理请求。

**策略路由**：当接收到一个请求，库会决定选择哪个partition(put操作时候随机选择，get/delete 操作从blob id中提取partition)。然后基于2.3章中讨论的策略({one, k, majority, all}，与相应的replica进行通讯，直到请求成功或失败。

**分块**：非常大的blob(例如 视频) 造成了负载不均，小块的blob，造成了固有的高延时。为了减缓这些问题，Ambry将大的blob分为大小相等的单元称之为分块(chunk)，大的分块不能完全解决大blob的挑战，小的分块将会带来过多的额外开销。基于我们目前的大blob分布我们发现最有效的分块大小时4-8MB。
![image](https://raw.githubusercontent.com/phantooom/blog/master/image/ambry/07.jpg)

在put操作期间，一个blob b 被分为k 块{c1,c2,c3....ck},每个分块被当做一个独立的blob，每个分块使用相同的步骤，作为一个普通的blob被执行put操作(2.3章)，大多数会被分散到不同的partition。也会被分配一个无二的分块id 与 blob id格式相同。为了为了能够召回b, Ambry为blob b 创建了一个元数据(metadata)blob b-metadata，b-metadata存储了分块的数量，和顺序排列的分块id。如图4所示。这个元数据blob最后作为一个普通的blob，put到了Ambry中，然后返回b-metadata 的blob id 作为b的 blob id。如果put操作在写完所有分块之前失败，那么将会将写入的分块标记为删除，然后重做操作。

在get时，元数据blob会被召回，然后从中提取到分块id，Ambry使用大小为s的滑动缓冲区召回blob，然后Ambry并行(因为大部分的分块独立的存储在不同的Datanode中)的查找blob 前s个分块，当滑动缓冲区的第一个分块已经被召回了，Ambry将缓冲区滑动到下一个分块，如此往复。整个blob开始返回开始于第一个分块被召回。

尽管，在这种机制下需要额外的put/get，但总的来说我们改善了延时，因为多个分块写和召回都是并行的。
![image](https://raw.githubusercontent.com/phantooom/blog/master/image/ambry/08.jpg)

**Zero-cost 故障检测**: 在大型系统中故障频繁的发生。从无响应，连接超时，到磁盘io问题。因为Ambry需要一个故障检测机制，像心跳或者ping那样利用请求消息。在实践中我们发现我们的检测机制是简单，有效，使用很少的带宽。这个机制如图5所示，在这种方法中，Ambry在最近的检查周期，不停的追踪请求连续失败的特定Datanode(或者磁盘) 。如果失败的数量达到了最大失败的上限(在我们的例子中是2)，Datanode被标记为 临时下线一段时间，Datanode在这种状态下排队的所有请求最终都会超时，并且需要用户重试。在一段时间之后，Datanode会转变为临时可用状态，当Datanode在临时可用的状态，可以接收请求，当收到的请求仍然失败会转为临时不可用状态，如此往复，直到接收请求不失败，Datanode会被标记为可用，再次恢复到正常状态。

**代理请求**：如2.3章描述的，Ambry使用代理请求实现高可用，和在远程数据中心时 read-after-write一致性。当一个blob没有被复制到本地的数据中心时，那个blob的请求将会被转发到其他的数据中心，并且处理(代理请求)。然而，数据中心的partition可能由于，尚未复制数据造成不可用，直到partition追回数据 并且replica恢复时。
代理请求是由Router库处理，对于用户请求透明，实践中我们发现代理请求发生的比例小于0.001%，因此对用户体验的影响很小。

## 4.3 Datanode层
Datanode 负责维护真实的数据，每个Datanode管理一些磁盘，并且响应安置partition的replica到磁盘的请求。put请求通过写到partition文件的末尾的方式处理。Get可能需要花费更多的时间，特别是blob在partition中的位置未知时，为了最小化读写的延时，Datanode使用了一些技术：
* **索引blob**： Ambry为每个partition的replica存储了一个blob偏移的索引，用来减少线性查找blob。(4.3.1章)
* **利用系统缓存**：Ambry利用系统缓存，在内存中处理大部分请求，同时限制其他组件的内存使用率(4.3.2)章
* **批量写,单次seek**：对于某个partition Ambry批量的写，并且周期性的将写操作flush到磁盘，因此，对于一个批量的连续写最多引发一次磁盘seek。flush的周期是可以配置的，取决于可用性和一致性的权衡。尽管批量可以引入开销，如，flush，dirty buffer和tunning，但是收益远大于这些开销。
* **保持所有的fd打开**：因为partition非常大(我们设置为100GB)，安置在Datanode上的partition的replica数量非常少(数百)，因此Ambry所有时间保持fd打开。
* **zero copy get**：当读取一个blob是，Ambry利用 zero copy 机制，例如内核直接从磁盘拷贝数据到网络缓冲区不经过程序，这个是可行的，因为Datanode在get操作中不会对数据进行任何操作。

![image](https://raw.githubusercontent.com/phantooom/blog/master/image/ambry/09.jpg)
### 4.3.1 索引
为了以一个低的延时找到blob在partition replica中为位置，Datanode为每一个replica维护了一个轻量级的内存索引，如图6所示。索引根据blob id进行排序。构建了一个blob id 到 blob实体起始偏移量的映射。当blob被 put(如blob 60)或者delete(blob 20) 时这个索引是实时更新的。

与SStables类似，Ambry限制了索引的大小，将索引分段，将老的分段存放在磁盘中，同时为每个磁盘上的分段维护了一个布隆过滤器。（4.3.2章）

索引也存储了标识，如果一个blob已经被删除了，或者指定了一个ttl，在get操作期间如果发现blob 已经过期或者被删除了，在读取真实数据之前会返回一个错误。

注意索引文件不包含任何额外影响正确性的信息，仅为了提高性能，如果一个Datanode故障了，整个索引能够通过partition进行重建。
### 4.3.2 利用系统缓存
最近写入的数据，通常也是热门数据，会被自动的缓存，不需要任何额外的开销(通过操作系统).通过利用这个功能，许多读的操作可以由内存提供服务，能够显著的提高性能。因此Ambry限制了Datanode中其他的数据结构的内存使用。Ambry限制了索引通过分段的方式，仅仅保留最新的分段在内存中(图6所示)。新的实体被添加到内存的索引分段中，当内存中的分段超过最大限制的大小，会被flush到磁盘，变为只读的分段，这个设计也帮助了故障恢复，因为当从故障中恢复时，只需要从partition中重建内存中的索引即可。查找blob的顺序是按照时间反向查找，从最新的分段开始(内存中的分段)。因此删除的实体将会比put的实体先查找到，这个保证了删除的blob不会被召回。

布隆过滤：为了减少查找磁盘索引分段延时，Ambry为每个分段维护了一个内存的布隆过滤器，包含了对应分段的blob id。使用布隆过滤器Ambry可以快速的找到读取哪个内存索引分段，因此有很大的概率只发生一次磁盘seek，然而由于我们主要的读都会命中内存中的分段，不需要任何seek操作。
# 5 复制
属于同一个partition的replica，可能由任何失败或者异步写的原因造成不同步。为了修复这种不一致的replica，Ambry使用异步复制算法，定期的同步这些replica。这个算法是完全分布式的。在这个过程中，每个replica单独扮演master角色，从其他replica同步数据，是一种all-to-all的方式。同步使用一种两阶段异步复制协议，如下所示，这个协议是采用pull的方式，每个replica独立的从其他replica中请求获取当前replica中缺失的blob。

* **第一阶段**：这个阶段从上次同步点之后，查找缺少的blob。这个是通过获取上次同步完成时offset之后所有blob id，然后与过滤出本地缺失的blob。

* **第二阶段**：这个阶段replica缺少blob，发送一个请求，包含所有缺失的blob id，然后缺失的blob会被发送并添加到replica。

![image](https://raw.githubusercontent.com/phantooom/blog/master/image/ambry/10.jpg)
为了快速的找出最近写入的blob，复制算法为每个replica维护了一个数据结构，称之为journal。journal是一个内存的缓存，其中维护了最近blob，按照offset进行排序，图7展示了两个replica(r1,r2)的两个journal的例子，两阶段复制过程从r1向r2发起，offset为600。在第一阶段，r1请求offset在600之后的所有最近添加到r2的blob id，r2 使用journal 返回了一个blob id的列表 B={55,40,70,90}，然后r1 过滤出缺少的blob{55,90}.在第二阶段，r1接收到缺少的blob，添加blob到replica的末尾，同时更新journal，索引和lastoffset。
为了改善系统的效率和扩展能力，复制算法使用了如下的优化：

* 数据中心内部和外部线程池分离，使用不同的同步周期
* 两个Datanodes之间相同的partition replica 使用批量请求，同时批量传输blob在数据中心之间。
* 落后的replica优先，为了能更快的赶上。

# 6. 实验结果
我们构建了三个实验环境，小的集群(6.1章)，生产集群(6.2章)和仿真集群(6.3章)。
## 6.1 吞吐和延时
在这个章节我们测量了系统的延时与吞吐。使用了一个微型的benchmark工具,对系统在只读，只写，和可读可写的负载下进行压测(6.1.1章)。

### 6.1.1 微型-benchmark
我们首先测量了吞吐峰值，我们设计了一个微型benchmark工具，工具线性的增加系统负载（通过添加更多client的方式），每个client在上个请求刚返回后就继续发送请求,直到系统达到饱和，不能处理更多请求。

benchmark有三种模式： 写，读，读写。在写模式下，许多clinent put随机大小的字节数组blob。在读模式下，首先在写周期内进行饱和的写(最快的速率写),然后从写入的blob随机选择进行读。在大多数实验中，我们设置写的周期足够长，可以让大多数的读请求(>80%)都是由硬盘进行处理的，而不是从内存。读写模式与读相似，只是在写周期之后读和写的请求各占50%。
因为延时和吞吐与blob大小相关，我们使写如了总大小固定大小的blob在每次运行时，但是会改变单个blob的大小。

### 6.1.2 实验设置
我们部署了单节点Datanode的Ambry。这个Datanode运行在一个24cpu ，64g内存，14 块1T HDD，和一个全双工的1Gb/s的以太网卡的服务器上。4G内存被用作Datanode服务内部使用，剩余的全部内存作为系统缓存。我们在每个磁盘上创建个8个replica。一共122个partition。14个磁盘使用了1Gb/s的网卡看起来有些多余，但是对于磁盘延时的主要影响因素是小文件的seek操作。因为partition中大部分blob是小的(<50kb)，我们需要使用多个磁盘进行并行(更多详情在6.1.4章)。值得注意的是我们将Ambry设计为具有性价比的系统，所以使用了廉价的机械盘。

client从相同的数据中心的机器上发送请求，这些client充当Frontends，直接向Datanode发送请求，之前讨论的微型benchmark工具被用来产大小不同的blob{25KB,50KB,100KB,250KB,500KB,1MB,5MB},我们没有继续产生超过5MB的blob，因为5MB已经超过单个分块大小了。
![image](https://raw.githubusercontent.com/phantooom/blog/master/image/ambry/11.jpg)

### 6.1.3 client数量的影响
我们运行了微型benchmark生成变化的blob大小，当线性增加client数量时。对于读模式，第一个写的周期我们设置了写6倍于内存的大小的数据。图8a展示了以MB/s表示的系统提供的吞吐。适当添加client吞吐会增长，直到饱和。吞吐饱和发生在网络带宽的75%~85%。唯一的例外就是读取小的blob，主要是因为频繁的seek(6.1.4章讨论)。饱和发生的很快(通常client<=6)因为client在benchmark中会尽可能的发送请求。
图8b展示了经过blob大小标准化后的延时(例如平均延时/blob大小)。在抵达饱和点钱延时几乎是一个常数，然后线性增长，线性增长的原因是因为，系统不能处理额外的请求了。

![image](https://raw.githubusercontent.com/phantooom/blog/master/image/ambry/12.jpg)

### 6.1.4 blob 大小的影响
在图9a和9b中我们分析了最大的吞吐(20 clinets)和负载， 在不同blob大小下的情况。对于大的对象(>200KB)，最大吞吐put(MB/s)一直恒定，同时接近带宽的最大负载。类似的吞吐每秒请求的吞吐也成比例的增加。然而对于读和读写来说，小文件的读的吞吐打破线性规律。规律被打破的原因是因为微型benchmark工具是随机的读取blob，导致了频繁的磁盘seek。对于小文件存在seek放大影响。为了进一步分析磁盘，使用Bonnie++(一个io benchmark工具，用来测量磁盘性能)，我们确定磁盘seek是小文件延时的主要原因。例如当读取一个50KB的blob，超过94%的延时是由硬盘seek产生的(6.49ms用于磁盘seek，0.4ms用于读取数据)。

Datanode读和写工作分别利用了流入，流出链路。但是读写模式下会有个一个更大的吞吐，因为两个链路同时被利用到了。因此我们的全双工模式的网络设施，可以在读写模式下获得接近两倍的吞吐(1.7Gb/s)。对于小文件，读写吞吐大致为只读吞吐的2倍，因为读写各占50%的工作，读成了主要的限制因素。

图9c展示了延时根据blob大小变化的趋势。这些结果未到达饱和点，只启用了两个client。与吞吐相似延时线性增长，小文件除外。读的延时更高，因为大多数的读都会引发磁盘seek，然而写请求是批量写入的。读写延时位于 读和写延时之间，因为读写是由读和写混合而成的。

![image](https://raw.githubusercontent.com/phantooom/blog/master/image/ambry/13.jpg)

### 6.1.5 延时的变化
对于请求延时来说，尾部和变化都很重要。图10展示了2个client的读，写，读写累积分布图。这个累积分布图中的读和写都非常接近一个垂直线，并且带有一个短的尾部，并且大多数值都接近中位数。读模式下(50KB 大小的blob)，会跳到0.15附近，主要是因为小文件中的一小部分能够直接由Linux cache提供服务，cache会比硬盘快几个数量级(6.1.6章)。读写模式累积分布图是由读和写混合组成，在0.5处发生改变。主要是因为50%读和写的工作模式造成的。

![image](https://raw.githubusercontent.com/phantooom/blog/master/image/ambry/14.jpg)
### 6.1.6 Linux Cache的影响
当以50KB大小的blob 和 2个 client的配置下运行微型benchmark工具时，提供两种配置。1）在读之前写6倍于内存大小的数据，所以大多数的请求(83%)是由硬盘提供服务。2)写数据大小等于内存的数据，保证所有数据都在内存当中(读缓存)。表3比较了这两种模式。

缓存读的性能，超过2100请求/s (104MB/s接近79%的网络带宽)与最大的写带宽相匹配(6.1.3章)，与之相对从磁盘读有540/s请求。我们也可以测量 平均，最大，最小 延时的标准差，表3所示。在所有的case中，最小值相等，然而从缓存读能够提高平均延时和最大延时，分别提升了5.5倍和13倍。这能够说明利用linux cache 是很有效的(4.3.2章)。

### 6.2 跨数据中心优化
我们分析了我们的复制算法在3个分布在全美国的LinkedIn数据中心{DC1,DC2,DC3},本章所有的实验都来自于生产环境。

![image](https://raw.githubusercontent.com/phantooom/blog/master/image/ambry/15.jpg)

### 6.2.1 复制落后
我们定义了一对replica(r1,r2)之间的复制落后，用r2使用的最高的offset与r1的上次同步的offset之前的不同表示。需要主要不是所有的落后的数据都要被复制到r1，因为他也可以从别的replica获取到缺失的blob。

我们测量了给定Datanode上所有的replica与集群中其他replica之间的落后。我们发现了超过85%的值为0.图11展示了不同数据中心的不为0值的累积分布。对于100GB的partition（所有的数据中）95%的落后小于1KB，数据中心3有更多的落后，因为他距离其他的数据中心相对远。

![image](https://raw.githubusercontent.com/phantooom/blog/master/image/ambry/16.jpg)
### 6.2.2复制带宽
Ambry写数据到其他数据中心依赖后台复制，我们测量了24小时内用于数据中心间复制的网络带宽。如图12所示。带宽总计小于10MB/s，所有的数据中心都类似。并且与请求比率相关，每天的带宽使用模式都类似。这个是非常小，因为相同的replica我们采用了批量复制，也因为主要是读的流量比重大。

![image](https://raw.githubusercontent.com/phantooom/blog/master/image/ambry/17.jpg)

图13展示了每个Datanode上的平均复制带宽，包含了数据中心内和数据中心之间的复制。数据中心内的带宽很小(200B/s 95%分位)，与之相对的数据中心间的带宽有150KB/s 在95%分位 (相差1000倍)。数据中心之间带宽高的原因主要是因为异步写。然而数据中心之间的带宽仍然很小(仅占1Gb/s 链路的0.2%)。三个数据中心有点小的不同，主要是因为不同的请求和不同的接收。

![image](https://raw.githubusercontent.com/phantooom/blog/master/image/ambry/18.jpg)

图14展示了一个放大的图片，这个图片只包含数据中心之间的带宽。数据中心间的复制有一个短的尾部，在95%和5%分位可能存在3倍的差异。这个尾部产生的原因是因为Ambry中负载均衡的设计。数据中心内复制也会有更长尾部。也存在一些为零的值(图中忽略掉了)。因此复制可能不消耗带宽(数据中心内)，或者消耗几乎所有的带宽(数据中心间)。

![image](https://raw.githubusercontent.com/phantooom/blog/master/image/ambry/19.jpg)

### 6.2.3 复制延时
我们定义了复制延时，使用一次复制协议花费时间。例如 T接收到缺失的blob -T发起同步时间。图15展示了我们生产环境中数据中心间与数据中心内平均复制延时的累积分布图。

数据中心间复制延时中位数小于150ms，有一个非常短的尾部。尽管这个延时可能变高，代理请求的比例接近0(<0.001%).这是因为用户通常读取数据从他们最近写数据相同的数据中心。因此复制复制对用户体验的影响非常小。

出人意料的，数据中心内的复制的延时相当高(比数据中心间高6倍)同时带有一点变化。出现这种模式的原因是人为添加了1秒的延时。为了防止blob冲突检测，如果一个blob被复制比Datanode接收到请求还快，那么当Datanode接收到这个put请求时发现本地已经存在了这个blob会人为这个blob已经存在了，认为id冲突，会将请求失败。人为添加了一个延时是为了防止这个问题发生在数据中心内复制时。这个小的延时并没有影响到可用性或者之久性。因为数据中心内复制只用于修复失效的数据或者慢的Datanode。

## 6.3 负载均衡
因为集群扩容发生的不是很频繁(最多几个月发生一次)。我们实现了一个模拟器用来展示Ambry经过长时间(通常是数年)运行后和大规模集群的行为(数百个Datanode)。我们使用的负载时基于生产环境的。这章所有的结果都是模拟器的。
### 6.3.1 模拟器设计
模拟器的设计与Ambry的架构类似，请求和服务使用相同的路径。然而不存在真实的物理硬盘。为了处理请求只有元数据会被存储(blob id 和 大小)和召回，同时用于反应请求的倾向(磁盘大小增长)。

**Workload**：我们使用合成Workload，这个Workload非常接近LinkedIn真实的流量。这个Workload记录了每种类型请求的比例(read write delete)，大小，和blob访问模式（基于年龄）。

**集群扩展**：模拟器起初拥有一组同一个数据中心的 Datanodes 磁盘 和partition 。随着时间的流逝，当partition接近容量上限时，一批新的partition使用2.2章的策略被添加。如果partition不能够被添加(磁盘没有空间了)，一批Datanodes将会被添加。

### 6.3.2 实验设置
模拟器运行在单个数据中心，拥有十个Frontend节点。这个实验起初有15个Datanodes，每个都带有10个4TB 硬盘 和1200个100GB大小的partition，每个partition有三副本。当达到partition和Datanodes的需要增加的节点时，600个partition和Datanodes会被添加。模拟器运行了超过400周(几乎8年)并且拥有240个Datanodes。测量请求速率，磁盘使用率，和数据迁移时，模拟器运行了两种，一种是带有负载匀衡的，另外一个不带负载均衡，其他配置均相同。

![image](https://raw.githubusercontent.com/phantooom/blog/master/image/ambry/20.jpg)

### 6.3.3 请求速率
我们测量了每个磁盘的各个时间的读取速率(KB/s)，图16展示了系统带有负载均衡和没负载均衡的平均，标准差，最大，最小值。写的结果相似，因为空间原因被移除了。
平均值，是比较理想的，是一个逐步下降的函数。下降的点是因为新的Datanodes被添加到了集群中。在没有rebalancing的case中大部分老的磁盘都是只读的，几乎没有流量，然而新添加的磁盘收到了绝大部分的请求。因此最小值接近0。所以标准差和最大值有重要的意义(最大值比平均值大3-7倍，标准差比平均值大1-2倍)。当rebalancing机制添加后，标准差下降到接近为0.我们推断，Ambry的负载均衡是有效的。

![image](https://raw.githubusercontent.com/phantooom/blog/master/image/ambry/21.jpg)

### 6.3.4 磁盘使用情况
我们分析了带负载均衡和不带负载均衡的磁盘使用比例。例如使用的空间除以磁盘的总空间，如图17所示，没有负载均衡的最大值一直处于容量上限。因为一些磁盘已经满了，然而当新的Datanodes被添加进来时最小值接近于0.带有负载均衡的最小值和最大值接近平均值，最小值会临时下降，直到负载均衡完成。此外标准差下降明显，接近于0，偶尔跳起因为DataNodes被添加到集群中。

![image](https://raw.githubusercontent.com/phantooom/blog/master/image/ambry/22.jpg)

### 6.3.5  这段时间的评估
我们评估了过去这段时间的改变，通过测量400周内的请求速率，磁盘使用的区间(最大值减最小值)和标准差。如表4所示。负载均衡有显著的效果，改善区间6-9倍和8-10倍标准差。

![image](https://raw.githubusercontent.com/phantooom/blog/master/image/ambry/24.jpg)
### 6.3.6 数据迁移

当负载均衡被触发时，我们测量了由于负载均衡需要时造成的数据迁移，为了达到ideal状态的最小数据迁移。当添加使用率在ideal状态与当前状态之间且使用在ideal状态之上的磁盘时，我们计算了最最小数据迁移量。这个值比最小值低一点点。因为数据移动是按照partition移动的。如图18所示，数据建议非常接近最小迁移，有时还比那个值低，说明我们的算法是非常优秀的。需要特别注意的，负载均衡算法当添加或者删除磁盘使用率会高于或者低于ideal状态时，通常不会删除(或者添加)partition。因为有时这么做会打破平衡。

# 7 相关工作

**文件系统**：Ambry的设计受到了日志文件系统的启发[LFS].这些系统利用顺序写类日志数据结构优化了写的吞吐，使用系统缓存优化读。尽管这些这些系统承受着碎片问题和清理开销，但是核心的思想是非常有意义的，尤其是当blob是不可变对象时。主要的区别在于数据倾斜访问模式和额外的优化像使用分片索引和布隆过滤。

已经有处理元数据和小文件的有效方法了。其中的一些技术包括 减少磁盘seek，联合使用日志结构系统（用于meta和小数据）和 快速文件系统(大文件)，使用索引块存储数据的起始段。我们的系统解决了这些问题，通过使用内存的数据分段索引和布隆过滤以及批量写技术。

**分布式文件系统**：由于大量数据和数据共享的徐牛，许多分布式文件系统，像NFS，AFS，同时出现了一些更可靠的能够处理故障的系统，像GFS，HDFS，和ceph。然而这些系统元数据开销太大，此外对于blob存储一些额外属性(目录，权限等等)也是不需要的。这些系统中的许多(例如HDFS，GFS，NFS)，由于单一的元数据服务器造成了，元数据的天花板。并且这个服务器（元数据服务器）,成了每个请求的中心，同时也成为了单点，由于单点的限制，扩展性也成了问题。最近的研究中，这个问题已经被分布式元数据或者缓存解决。尽管这些系统减少了元数据的访问，但是每个小对象仍然有很大的元数据(通常存储于硬盘)，降低了系统的吞吐。

**分布式数据存储**：许多kv存储，像 [2, 5, 8, 14],已经被设计为每秒可以处理大量请求。然而这些系统不能高效的处理大对象(数十MB到GB)，并且添加了额外的开销，为了维护异质性。也有一些系统[2, 8,14]使用哈希将数据分布到机器上，当机器添加时会发生大量的数据迁移。

PNUTS [6] and Spanner [7]  是可扩展跨地区的分布式系统，PNUTS也保持了输在均衡。然而这些系统提供了更多的功能，和数据保障，但是对简单不可变的blob存储是不需要的。

**blob存储**: 与Ambry中的partition相似的概念已经在其他系统中被使用，Haystack使用了 logic volumes，Twitter's blob store 使用virtual bucket，Petal file system 引入了虚拟磁盘。Ambry 从这些系统中引入了一些优化，例如Haystack中的添加内部缓存，然而，不管是Haystack还是Twitter‘blob store 都没有处理负载均衡的问题。此外Haystack使用同步写副本影响了跨机房的性能。
Facebook 已经设计了F4，一个blob 存储，对老数据使用纠删码来减少副本因子，尽管这些新思想可以引入Ambry，但是我们主要点是既需要新数据也需要老数据。Oracle’s Database [19] 和Windows Azure Storage (WAS) [4] 也能存储可变blob，并且WAS 甚至能够对跨机房环境进行优化。然而他们都提供了许多额外的数据类型和一致性保障还有可变blob，这是我们需求中不需要的。

# 8 结论
这篇论文描述了Ambry，一个专门为存储不可变对象的分布式存储系统。我们将Ambry设计为一个可以在跨机房环境下提供低延时和高吞吐的系统。使用了分布式设计，再均衡机制，分块，和逻辑blob组(partition)，我们提供了一个负载均衡和水平扩展能力，来面对LinkedIn迅速增长。我们未来的一部分工作是改变热数据的副本机制，和使用纠删码机制来存储冷数据。我们也打算引入压缩机制，但是要权衡开销。此外我们正在改进Ambry的安全性，特别是跨数据中心流量。

# 9 致谢
略
# 10 参考
![image](https://raw.githubusercontent.com/phantooom/blog/master/image/ambry/23.jpg)
