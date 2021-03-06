# Overload Control for Scaling WeChat Microservices

# Introduction

这是腾讯微信团队发表在SoCC'18上的一篇论文（[下载](https://www.cs.columbia.edu/~ruigu/papers/socc18-final100.pdf)。主要
介绍了微信后台已经使用5年的过载保护算法DAGOR。该实现针对大规模微服务架构，且已经在生产环境中运行5年之久，具有很强的实用价值。

企业/产品的组织结构与算法系统架构是一个相互影响的关系，这篇论文明显的体现这一点。微信是一个大规模的基于微服务架构的产品。
多个技术团队独立的开发，维护，部署数以百计的为服务。作为中国最流行的社交类产品，微信具有24/7不能停机，系统负荷波动大的
特点。

针对上述现实，这篇论文提出的过载保护算法具有以下特点：
1. 去中心化/自治 
2. 服务无关

# WeChat Backend Architecture

微信的后台由~3000个service组成。这些services被分为三类：
1. entry services - 它们直接接受来自外部世界(主要是微信用户）的请求（比如登陆，支付等）
2. leap services - 它们接受来自其它services的请求。通常，leap services会通过调用其它服务来回答这些请求。
3. basic services - 它们接受来自其它服务的请求，但只依靠自己就能回答这些请求。

# 算法描述

## 请求优先级

但用户请求到达entry services后，entry service会根据请求的类型给每个请求赋予一个优先级。比如login的请求具有最高优先级，支付相
关请求具有次高优先级。这个优先级是定义在一个hash表中，每个entry service都有这个表的一个拷贝。这个优先级被称为业务优先级。

同时，entry service会对用户id进行HASH，从而计算出一个用户优先级。这个计算保证在一个小时内，同一用户的优先级是相同的。这个
做法优点有三：
1. 优先级的计算可以由各个entry service独立进行，不需要使用一个中性化的服务
2. 在较长一个时间内（一个小时对微信用户来说算一个较长的时间段了），用户得到比较一致的服务体验
3. 公平性 - 每个用户都有得到高优先级的机会。

两个请求A和B，如果A的业务优先级高于B的业务优先级，那么A的优先级高。如果两者的业务优先级相同，则比较用户优先级。这构成了一个
两级的优先级结构。

这个优先级会被entry service传递到leap/basic services。

## 如何判断服务过载

判断服务过载可以在产品层面，服务层面，服务实例层面（比如一个container）。DAGOR的做法是各个服务实例自行判断过载，这避免了
复杂的中心化过载判断服务（瓶颈）。

判断服务过载的指标可以是响应时间，CPU利用率等。DAGOR采用了请求的排队等待时间作为指标。它把一秒钟算作一个窗口（或者2000个
请求），然后用这个窗口内的所有请求的排队时间的平均值（为什么是平均值，而非75th or 90th，文章没有解释）作为过载指标。当
这个值大于20ms时，DAGOR认为本服务实例过载。

## Request Shading Policies

当一个服务实例发现自己过载后，它会把QPS降低到当前的95%（这个过程会持续到服务实例认为自己不过载为止）。也就是说，服务实例
为提高优先级阈值，所有优先级低于这个阈值的请求都会被丢弃。

DAGOR用一个直方图来记录每个优先级的请求的数量。使用这个直方图，DAGOR能计算出新的优先级设置为多少能恰好丢弃掉5%的请求。

当服务实例发现自己不过载时，它会把自己的QPS提高1%（这会导致系统把优先级阈值调低）。

## 服务间的协作

当一个服务调整了优先级阈值，它会Piggyback这个阈值给向它发送请求的上游服务。这些上游服务会记住这个阈值，不再向下游服务
发送低于这个优先级的请求。

# 评述

整个算法/架构简洁，易懂，鲁棒性强。

系统没有中心化模块，这不仅避免了中心化模块看来带来的瓶颈，也使得各个敏捷团队可以并发独立工作，减少团队协作带来的开销。

