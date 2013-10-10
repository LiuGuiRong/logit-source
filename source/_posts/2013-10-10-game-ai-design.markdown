---
layout: post
title: "游戏AI框架设计"
date: 2013-10-10 11:11
comments: true
categories: 
---

###概述

简化怪物AI脚本编写，提高怪物AI效率，抽象出AI框架结构，同时能够让设计人员快速的开发怪物AI。

###需求

###AI实现方式

####有限状态机

有限状态机（finite-state machine， 缩写： FSM）又称有限状态自动机，简称状态机，是表示有限个状态以及在这些状态之间的转移和动作等行为的数学模型，一个有限状态机可以用一个关系式描述:

    State(S) x Event(E) -> Action(A), State(NS)

这些关系解析如下：如果我们处于状态S并且事件E发生了，那么我们需要执行动作A，并且状态转换为NS。

当前游戏中怪物AI很大部分是基于FSM实现的，例如一个巡逻小怪的AI可以表示为:

![FSM](/images/game-ai-design/fsm.png)

####行为树


###实现


###脚本集成


###编辑器


###AI调试



