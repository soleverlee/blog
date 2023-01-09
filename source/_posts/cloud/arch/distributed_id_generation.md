---
title: 分布式系统 ID 生成
date: 2017-09-27
categories:  
    - 云计算
    - 架构
---
ID生成一直是一个老生常谈的问题，个人不习惯使用ID自增的方式，从0开始...每次递增，原因是因为low...最近研究了一下ID生成的算法，主要用来给我们的订单系统用。首先说一下背景：我们的系统提供给很多经销商使用，每个经销商登录到我们的系统，发生业务、产生订单。
<!--more-->

# 设计原则

* **唯一性**：保证生成的ID不会出现重复，需要考虑分布式部署情况下的唯一性
* **安全性**：不会通过订单号泄露流水信息（比如别人通过订单号知道你一天大概多少交易量）
* **长度**：长度未明确要求，在满足其他需要的情况下应该尽量短
* **字符集**：字符未明确要求，按允许数字 + 英文字母（大写）设计
* **性能要求**：如果按500家商家上线，每家每天1000次订单生成，其中高峰期每分钟10单，系统能支持运行10年的指标来计算，每天的订单数为500 * 1000 = 50W，高峰期每分钟为500 * 10 = 5000单，10年为 500 * 1000 * 365 * 10 = 1825000000单
* **易读性**：如果使用英文字母，则可以考虑去除掉容易混淆的字母：O I S Z 等

# Option 1：Snowflake算法

参考twitter的snowflake 64bit ID生成算法，设计如下的ID规则：

0 00000000 00000000 00000000 00000000 00000000 0 00000000 00 00000000 0000

* 1  bit 符号位，固定为0，不使用
* 41bit时间戳，即当前时间到某一开始时间的毫秒数
* 10bit 商家标识，可根据其上线时间递增
* 12bit 序列号，每一毫秒内从0开始，如果超过最大值则等待下一毫秒

其中41bit时间戳最大可支持的时间：
(2^41) / (365 × 24 × 60 × 60 × 1000) = 2199023255552 / 31536000000 ≈ 69.7(年)

10bit 标识 最大支持的商家数为：
2^10 = 1024

12bit 序列号最大支持为每秒：
2^12 = 4096 

生成的ID大概像这样：
```
274432
...
278501
278502
278503
278504
...
109996123808346299
109996123808346300
109996123808346301
….
2647243359755833344
2647243359755833345
2647243359755833346
```
如果需要支持分布式部署，则可考虑将商家标识的10位中插入机器标识，根据最大可能的机器节点数来选取位数。比如分配3位，就可以支持2^3=8台机器。同样，时间戳可以浮动，一般的系统根本不需要考虑超过5年以上的情况，能用两年就不错了...当然我倾向于支持10年，10年后如果世界末日不到，系统还在用，那就再想办法吧。

# Option 2 : 时间戳 + 商家标识 + 随机数

## 时间（6位）+ 商家标识 （4位） + 随机数（4位）

170911 0001 8774

其中，商家标识和随机数取值范围为[0-9A-Z]即
T = 26 + 10 = 36(种)

其中，商家标识最大支持的商家数为：
T × T × T × T = T^ 4 = 1679616

随机数支持范围同商家标识。因时间精确到天，因此随机数每日不能重复，每天最大支持的订单数（每个商家）即为：
T^ 4 = 1679616

## 时间（3位）+ 商家标识 （2位） + 随机数（2位）

 79A 0A F1

其中年、月、日各占一位，其中年可以取年份-2017年的差值，超过9则用字母表示，例如2017年9月25日，可表示为：
09M

这是简短的方案，年份只取一位则最多只可以使用T = 36年，否则会出现重复

其中商家标识最大支持：
T ^ 2 = 1296

每天每一个商家标识最多只能支持T^ 2 = 1296笔订单号，若不能满足业务需求则可以将随机数扩展为3-4位

## 时间（8位）+ 商家标识 （2位） + 序列号（2位）

65DEF7BE AF 35
65DEF7BE （16进制）= 1709111230（10进制）

因时间精确到分钟，随机数按照每分钟递增，因此每分钟可以支持订单数：
T ^ 2 = 1296
每天可支持的订单数为：
24 × 60 × T^ 2 = 1866240

# Option 3 : 时间戳  + 随机数

时间戳（6位） + 随机数（4位）
170911 9527

使用4位数字则每日最大只能9999单，使用字母则可以有
T^ 4 = 1679616 单
若按500家商家算平均每个商家每天有3359单

# Option 4 : 渠道号 + 时间戳 +（流水号 +用户标识+随机数）

R 79A 0011 9527 34

其中，流水号按用户按天生成，用户标识取用户ID后四位，例如：
流水号（4位） + 用户标识（4位） + 随机数（2位）
0011 9527 34 