---
layout: post
title: 资损监控系统搭建（二）: 基于消息的实时旁路资损监控系统设计
description: >
    简单低廉的内部方案，快速发现并及时止损，有优有劣。

image: 
  path: /assets/img/blog/bypass-logic.jpg
  srcset:
    1060w: /assets/img/blog/bypass-logic.jpg
    530w:  /assets/img/blog/bypass-logic@0,5x.jpg
    265w:  /assets/img/blog/bypass-logic@0,25x.jpg
sitemap: true
comments: true
---

资损问题应该是每个商业系统的头等大事，如何发现资损和止血止损的速度是衡量业务团队的重要指标，

整个系列应该会分为五章吧：
    - 规则引擎调研: 为系统引入实时性，快速迭代的功能
    - 基于消息的实时旁路资损监控系统设计：简单低廉的内部方案，快速发现并及时止损，有优有劣。
    - 小时级对账系统：牺牲一部分实时性，从而实现跨端校验。
    - 上下游资金离线核对系统设计：业务代码无入侵，构建成本高，但是可靠性与准确性强！实时对账了属于是。
    - 反思：一个资损问题反映出来的，是千百个业务逻辑的耦合和期望通过人工测试来覆盖所有业务场景的错误想法。
  
本文是第二章，介绍了一种简单低廉的**内部**方案，快速发现并及时止损，**基于消息的实时旁路资损监控系统**, 优点是开发成本低，所需资源少，缺点是校验准确性基于消息的“真实性”，且无法跨业务校验。

## 背景

为了**降低线上系统的资损风险**，**快速发现异常用户的探测行为**，可以基于业务消息topic，搭建一套基于消息的旁路监控系统。

基本原理是取出业务消息topic中的消息，针对消息体中的**敏感字段**，使用配置中心内的规则进行校验，然后告警，dev/qa确认风险后，
配置拦截规则，在主流程中将符合拦截规则的异常创单请求拦截下来，及时止损。

为了满足使用需求，该系统需要具备以下几个特性：
    - 迭代速度快，可以支持校验规则热更新
    - 确认风险后可以立马发布止损措施
    - 不影响业务主流程

此外，该系统还可以部署在开发环境，发现需求中的高危bug，如：是否更改了“价格字段”含义，金额恒等式改动。


## 调研

为了满足特性1，2点，只能基于规则引擎实现，进过[调研](https://www.zeonll.com/code/work/2022-03-27-go-rule-eval-bi-lan/)，
决定使用billbill开源的gengine库。

大多电商业务系统都会将下单消息投递到消息引擎中，给下游系统使用，用来实现系统解耦。通过消费引擎中的消息，我们可以获取到用户下单时生成的“快照”，
针对该快照，抽象出一套规则，对所有消息进行校验，如果消息体不符合当前规则，则认为具有危险性，再通过配置拦截规则，下单前将类似的逻辑拦截下来。

## 逻辑架构
如下图：
[!img](/assets/img/blog/bypass-logic.jpg)


## 物理架构
如果没有配置中心的话，可以把规则存储在db内，定时拉取即可。

注意：消费者服务可以集成在现有的消费者服务内，但是最好隔离。因为如果其他消费服务逻辑panic了，会导致该服务内的消费逻辑集体拉闸。
对于这类的资金监控的服务，最好还是做好资源隔离吧。

## 存储设计
如果是配置在apollo上的话，namespace的格式我推荐使用txt，原因有以下两点：
1. 规则内不可避免的会用到“< >"，大于小于号，所以直接排除XML格式
2. gengine是支持一条消息校验多条规则的，所以规则集一点是“条块”状的，KV格式在展示时只展示一行，不太友好
3. txt格式可以存储的长度远大于kv格式，可以自行编排结构

gengine的所有规则都有自己的`rule_name`，比如如下的规则：
`rule_name`为`cate1-cate2-cate3`,
```
rule "cate1-cate2-cate3"
        begin
            displayTotal(req.GetTotalAmount())
             if req.GetTotalAmount() < 0 && version < 2{
                return "cate1-cate2-cate3 total amount < 0"
            }
            return ""
        end
```

我推荐的配置中心配置规则结构如下：
全品类校验规则单独一个namespace，存放通用校验规则，如总价格字段不为负数等。
一级品类校验规则各自一个namespace，如果某个二级品类所需规则比较多，也可以将二级品类的规则再剥离出来。

[!img](/assets/img/blog/bypass-msg.jpg)



## 详细设计
本节介绍架构之外的具体设计，比如版本号兼容逻辑，规则编写，消息与规则的对应等。

### 版本号兼容
这一点十分容易忽略，也是我当时的leader，一个老开发提醒我的，感谢。

由于旁路监控系统是从消息引擎中取的数据，所以规则和数据之间存在时效性问题，
具体的讲就是：在某次发版，在消息体内新增了字段A，当前的规则中包含A字段，但是一小时前的消息并没有A字段。

解决方案是，在业务代码内添加一个版本号(version), 每次改动消息体时版本号加一递增。规则感知版本号。

### 规则编写
规则编写首先是准确性，这需要开发人员熟悉业务逻辑已经试验；其次是返回的错误信息能让dev快速定位问题。
`gengine`返回的结果可以是任何格式，我推荐的约定使用string类型的返回结构，如果返回`""`则认为校验通过，
否则，就将返回的string结果拼接到报警消息中上报。

比如上一节中的规则，在异常情况下会告警“cate1-cate2-cate3 total amount < 0” + order_id，这样一来，对排查问题十分友好。

### 消息与规则的对应

消息通过品类来获取需要校验的规则，比如某条消息的品类为`cate1-cate2-cate3`，则它会被以下四条规则校验：
- 全品类规则
- cate1规则
- cate1-cate2规则
- cate1-cate2-cate3规则

gengine支持空规则，所以你没必要把4条规则都配全。

## 缺点与意外的收获
本系统设计有两个缺点：

### 投递的消息失真
系统是基于消息的监控，如果投递的消息失真了，则无法发现风险，有能力的话可以基于mysql的bin_log去做。这样的话规则复杂度会指数增长。

### 无法跨系统
自己监控自己的消息体，通过自己的逻辑监控自己的下单字段，总有点“玩家家”的感觉，准确性肯定比不上直接和上游系统实时“核对”。

## 意外的收获
如果把该系统部署在线下系统，可以发现开发中的需求是否有资损风险，防范于未然，很意外的一个收获。

## 总结
反正我是挺喜欢这个轻量化监控系统，在下一章，我会介绍一个稍微有点”臃肿“，但是实时性高，支持跨系统的plus版监控系统。
实际上已经不算是监控系统了，而是实时核对系统。


