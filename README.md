# OMS

## TOC

- [What is OMS](#what-is-oms)
   - [Functionalities](#functionalities)
   - [Characteriscs](#characteriscs)
      - [variety](#variety多样性)
      - [正向、逆向、盘点和搬仓混在一起，有race condition](#正向逆向盘点和搬仓混在一起有race-condition)
      - [模型复杂](#模型复杂)
      - [依赖多](#依赖多)
- [Core tech building blocks](#core-tech-building-blocks)
- [Design](#design)
   - [带着问题思考](#带着问题思考)
   - [订单形态](#订单形态)
   - [Landscape](#Landscape)
   - [核心流程](#核心流程)

## What is OMS

订单作为电商系统的纽带，贯穿了电商系统的关键流程，其他模块都是围绕订单构建的。 

订单可以认为是一次交易的生命周期，订单是交易的载体，反应的是履约的内容，即：一份订单就是一份合同。

```
       Partener --+         
                  |         
          Admin --+         
                  |         
         Seller --+         +-- BMS
                  |         |
           User --+         +-- TMS
                  |-- OMS --|
           API ---+    |    +-- WMS
                       |
    +------------------------------------------------------+
    |		 |       |                                 |
orderMark       FSM   storage                         dependencies
                         |                                 |
                    +----+--------+                        |- 台账
                    |             |                        |- 计费
              lookup & search  modeling                    |- BigData
                                  |                        |- 商品
                                facts                      |- 库存
                                  |                        |- 产品模型
                           BI & data mining                |- 时效
                                                           |- 地址和GIS
                                                           |- 支付
                                                           |- 预约
                                                           |- 主数据
                                                           |- 预分拣
                                                           |- 风控
                                                           +- ...
```

### Functionalities

- FSM
- Information Collector and Passthru(model)
   - Products
      - 商品信息主要影响库存、金额和WMS生产
   - Stakeholders
   - Addresses
   - Promotions
      - 涉及风控、黑白名单等
      - 可能涉及到人工审核
   - Payment
      - 记录交易金额，也要记录过程金额
      - 例如，商品分摊的优惠金额、积分、支付金额、应付等
         - 积分互换
      - 退换货、风控、财务等会用到
   - Time
      - 全生命周期的状态节点触发时间
      - 与状态一起表示它发生的时间语义
- Interact with stock
   - 避免超卖和少卖
   - 超卖对用户体验差
   - 少卖对商家体验差
- Execution(translation or orchestrator)
   - transport to WMS
   - transport to TMS
   - 履约内容
      - 时效
      - 发票
      - 优惠
      - COD收款
      - 预约
      - 运费
- Push
   - to whom
      - 商家
      - 用户
      - 仓储人员
      - 配送人员
   - how 
      - 手机push
      - 站内信
      - callback API
      - SMS
      - wechat

### Characteriscs

#### Variety(多样性)

例如
- 地址信息，O2O订单，会多出自提点地址
- 支付信息，如果COD
- 缺货处理
   - 部分缺货
- 预售，团购，拼单，拆单，合单，集单，憋单
- 来源
   - 导入
   - API
   - 页面手工建单
   - 内部测试UAT
   - 自动生成订单
   - 无界(新)零售
      - 意味着到处都可能是入口

#### 正向、逆向、盘点和搬仓混在一起，有race condition

取消可能包括如下场景，同时包括整单取消和部分取消：
- 买家取消
- 卖家取消
- WMS取消
- TMS取消
- 风控取消
- 客服取消
- 超时未支付取消
- 换货报缺
- 仲裁取消

#### 模型复杂

- 父子单(拆单)
- 主赠关系
- 团单与子单

#### 依赖多

- 内部依赖
   - 强依赖
      - 商品
      - 主数据
      - 客户中心
      - 库存
      - 产品中心
      - 时效
         - 控制下发WMS的时机
         - 控制下发TMS的时机
      - 预分拣
      - 台账
      - 风控
      - VMI
      - GIS
      - 优惠券
      - 账户中心
   - 弱依赖
      - 清关
      - 大件预约
      - 发票系统
      - 计费系统
      - 售后系统
      - 订单异常中心
      - 保价系统
      - TMS
      - WMS
         - 仓间调拨
- 外部依赖
   - 第三方承运商
   - 第三方WMS
   - 第三方支付

## Core tech building blocks

- 各种语义的分布式锁
   - 幂等性
   - 唯一性
   - 防并发锁
   - 防重锁
   - 时间区间锁
   - 分布式Latch
- 定时任务平台
- 可靠的MQ发送机制
- FSM
   - 订单状态是交易进展的反馈，是订单流程的一个个连接点
   - 不同类型订单的状态机不同
- 统一异常中心
   - 异常处理平台化、流程化
   - 异常的自我解释，自我定位
   - 主动和被动的异常监控
- 异步接单框架 task
- 接单与落库解耦
   - 提高接单能力
- 流程引擎，流程编排
- 业务扩展点机制
   - 微内核
   - 不能让订单系统成为瓶颈
   - 打破Conway law
- 查询
   - 读写分离
   - 按订单号查询
      - 动静分离
   - 搜索引擎
- FlexDB
   - 解决个性化扩展字段问题
- TCC
   - 解决复杂场景下数据一致性问题
- Graceful shutdown
- Custom metrics reporter

## Design

### 带着问题思考

- 2B和2C的订单是否统一存放和处理

### 订单形态

- 交易单(用户下达的指令)
- 计划单(生产计划单)
   - 订单拆分
   - 订单转移
- 生产单(生产运营单)

### Landscape

### 核心流程
