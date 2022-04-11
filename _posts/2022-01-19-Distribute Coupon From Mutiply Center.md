---
layout: post
title: Distribute Coupon From Mutiply Center
tags: System Design
---
## 背景

3个中心同时发券，中心间有30ms延迟，每个中心3台2c4g机器。要求先到先得，不同中心券发完间隔不超过1s，返回无券后不能再在此中心领到券，每人领券限制2张，金额限制100元；中心隔离时，到每个中心请求的TPS低且固定

## 架构

- app（2台）：异步访问用户服务、会员服务，异步更新redis人限制返回取券状态，异步动账服务、查账服务，异步redis退券（内存中有券时，防止多发可能少发），异步写redis取券流水lpush（pipeline）
- redis（共用1台）：保存各中心券数量和TPS，人限制，取券流水（使用redis6多线程模式，需要关闭rdb）
- cr（共用1台）：根据TPS比例重分配剩余券，同步本中心剩余券和TPS到其它
- consumer（共用1台）：消费redis取券流水（list log）brpop写库（需要批量insert写库，注意不满整数处理，支持update，update前需要保证insert完成），发券时限速，{center}\_empty标记后全速消费
- mysql：保存取券流水

## app
- 校验参数、内存无券标志
- 滑动窗口记录TPS
- 校验内存无券
- 异步用户、会员服务
- 异步调用redis（lua_1），更新redis人限制，返回状态（无券，张数超限，金额超限，可以取券），无券时标记redis和app内存无券标志
- 异步动账、查账服务
- 校验内存标志有券才可以异步redis退券（lua_2）
- 异步redis取券流水

## redis
- hash coupon_summary {center}\_remains：三中心各中心剩余数量
- hash coupon_summary {center}\_velocity：三中心各中心消费速度
- hash coupon_summary {center}\_empty：三中心各中心是否为空
- hash coupon_summary {center}\_isolated：三中心各中心是否已隔离
- hash {userid} amt：已领金额
- hash {userid} num：已领数量
- list log：券流水

## cr
1. 20ms同步一次，同步本中心剩余数量和消费速度给其他中心，假设C先被隔离，后B被隔离

- 3变2，则瓜分本中心的已失败中心（C）的券。AB需要先互查对方C的记录，所以C记录不能置0，而使用c_isolated标记写入AB中心；TPS为C中心的一台机器的TPS，考虑有两台app所以乘以2
```
A += (Min(A记录的C剩余,B记录的C的剩余) - 2 * (同步时间间隔+中心延迟)*Max(A记录的C的TPS,B记录的C的TPS)) / 2
```

- 2变1，本中心（A）瓜分（B）的券。已被c_isolated标记的中心（C）用0做处理（A记录的C剩余、A记录的C的TPS），最后将BC的剩余和TPS置0。
```
A += A记录的B剩余 - 2 * (同步时间间隔+中心延迟)*A记录的B的TPS +  A记录的C剩余 - 2 * (同步时间间隔+中心延迟)*A记录的C的TPS
```

2. 200ms一次重分配
```
A200ms后券应有剩余=A速度/(A速度+A中B速度+A中C速度)*(A剩余+A中B剩余+A中C剩余) 
if (A剩余 > A200ms后券应有数量 + 2 * A速度 * 重分配时间间隔) {
    A按照速度比重分配
    toB = A中B速度 / (A中B速度+A中C速度) * (A剩余 - A200ms后券应有剩余)
    toC = A中C速度 / (A中B速度+A中C速度) * (A剩余 - A200ms后券应有剩余)
    A剩余 = A剩余 - toB - toC
}
```
在B中心的B剩余加上toB，C中心的C剩余加上toC，失败则返回给A中心的A剩余

## 结论
三中心6台机器共12c发券，TPS到达10w左右（三次netty服务，一次redis），无券到达22w左右（一次netty服务）

## 改进
停止发券后，cr停止工作，消费者不限速工作（CPU0.7），消费同机器的redis（CPU1.3），优化考虑将app接收不到请求后，转换为消费者，消费redis

