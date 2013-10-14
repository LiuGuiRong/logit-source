---
layout: post
title: "游戏AI模块设计"
date: 2013-10-10 11:11
comments: true
categories: 
---

###概述

针对当前新的怪物AI需求，简化怪物AI脚本编写，提高怪物AI运行效率，抽象出AI框架结构，增强框架的灵活性及扩展性，同时能够让设计人员快速的开发怪物AI。

###需求

相对之前的怪物AI，当前新怪物AI相对比较简单，主要是由移动、攻击等基本行为组成的。

###AI实现方式

游戏中怪物AI的现有设计方法主要有：

1. 直接编码，或者给每个特定的怪物挂特定的脚本，当前我们游戏中很大一部分就是这样实现的；
2. 有限状态机(FSM)， 怪物在若干个状态中根据游戏世界中产生的特定事件产生特定的游戏行为，同时怪物进入新的状态；
3. 行为树(Behavior Tree)，怪物AI由若干个类型节点组成的树形结构，每个节点含有前置条件，当前置条件满足时执行该节点；

由于直接给每个怪物挂脚本的形式对于AI开发人员来说工作量繁琐，并且容易造成重复性劳动，下面主要阐述下有限状态机及行为树。

####有限状态机

有限状态机（finite-state machine， 缩写： FSM）又称有限状态自动机，简称状态机，是表示有限个状态以及在这些状态之间的转移和动作等行为的数学模型，一个有限状态机可以用一个关系式描述:

    State(S) x Event(E) -> Action(A), State(NS)

这些关系解析如下：如果我们处于状态S并且事件E发生了，那么我们需要执行动作A，并且状态转换为NS。

当前游戏中怪物AI很大部分是基于FSM实现的，例如一个巡逻小怪的AI可以表示为:

![FSM](/images/game-ai-design/fsm.png)

####行为树

行为树(Behavior Tree, [see](http://en.wikipedia.org/wiki/Behavior_tree))，顾名思义，它是由若干个子节点组成的树形结构，如下图所示:

![BehaviorTreeExample](/images/game-ai-design/behavior_tree_1.png)

其中叶子节点为行为节点，行为节点是游戏相关的，不同的游戏可以定义不同的行为节点，行为节点通常分为两种允许状态:

1. 运行中(Running)，该行为还在处理中;
2. 完成(Finished)，该行为处理完成；

其余节点是控制节点, 我们可以为行为树定义各种各样的控制节点（这也是行为树有意思的地方之一），一般来说，常用的控制节点有以下三种

1. 选择(Selector), 选择其子节点的某一个执行;
2. 序列(Sequence), 将其所有子节点依次执行, 也就是说当前一个返回"完成"状态后，再运行先一个子节点;
3. 并行(Parallel), 将其所有子节点都运行一遍;

当AI决策时从树的根节点自顶向下(depth-first search)，通过一些条件搜索这棵树，最终确定需要的行为（叶子节点），并执行相应动作。

**行为树实现**

    Task = {
        update = function(args) end,
        on_enter = function(args) end,
        on_exit = function(args) end,
    }
    BehaviorNode = {
        nodes = {},
        status = 0,
        task = nil,
        tick = function(args)
            return task:update(args)
        end,
    }
    BehaviorTree = {
        object = nil,
        root = nil,
        tick = function(args)
            if root then
                root:tick(args)
            end
        end,
    }

之前巡逻小怪AI用行为树可以表示为:

![behavior_tree_2](/images/game-ai-design/behavior_tree_2.png)

**如何选择子节点**

如何从子节点中选择呢？选择的依据是什么呢？这里就要引入另一个概念，一般称之为前置条件或者前提（Precondition），每一个节点，不管是行为节点还是控制节点，都会包含一个前提的部分，如下所示:

    -- lua code
    local behavior_node = {
        precondition = function() return true end,
        action = function()
            -- do something
        end,
    }

当自顶向下搜索行为树时，依次测试每个子节点的前提，如果满足则进入该子节点，以此继续选择下一个节点，最终返回某个行为节点，所以当前行为节点的前提可以表示为:

    Precondition[CurrNode] = Precondition[CurrNode.Parent] and Precondition[CurrNode.Parent.Parent] and ... and Precondition[RootNode]

行为树就是通过行为节点，控制节点，以及每个节点上的前提，把整个AI的决策逻辑描述了出来，对于每次的Tick，可以用如下的流程来描述：

    action = root.FindNextAction(input)
    if action is not empty then
        action.Execute(request,  input)  --request是输出的请求
    else
        print "no action is available"
    end

与有限状态机比起来，行为树结构还是比较简单的，而且当从概念上来说，行为树还是比较简单的，但对AI程序员来说，却是充满了吸引力，它的一些特性，比如可视化的决策逻辑，可复用的控制节点，逻辑和实现的低耦合等，较之传统的状态机，都是可以大大帮助我们迅速而便捷的组织我们的行为决策。根据行为树的可视化的决策逻辑的特性可以开发出类似流程图绘制的编辑器，通过拖拽的方式定制怪物AI逻辑、编写各个节点的前提脚步及行为节点的游戏逻辑，同时能够导出lua脚本。

关于行为树更加详细的解释见[这里](http://www.aisharing.com/archives/90)

**目前使用行为树的游戏**

1. Halo 3 & ODST
2. League of Legends [参考幻灯片](/assets/upload/Woo_Andrew_PuttingThePlaneTogetherMidair.pdf)

###AI模块设计

AI模块作为一个黑盒，依赖特定的输入进行决策再输出特定的请求，整体结构如下图所示:

![AI Module](/images/game-ai-design/ai_module.png)

* INPUT，AI输入数据，由游戏中产生的各种事件构成，主要有:
    1. 碰撞事件
    2. 技能事件
* AI Module，由行为树组成的AI决策模块；
* AI输出，AI决策结果，主要有:
    1. 移动: 请求移动模块移动到目标点
    2. 技能模块: 通过creature:use_skill(args)直接使用技能

###Lua脚本集成

行为树每个节点的前提(Precondition)，进入、退出、动作等是游戏相关性的，将这些与游戏逻辑紧密相关的部分使用lua脚本实现，每个行为树节点表示为：

    follow_base = {
        precondition = function()
            -- check condition
        end,
        behavior = function()
            -- do something
        end,
        on_enter = function()
            -- do something
        end,
        on_exit = function()
            -- do something
        end,
    }


###行为树编辑器

实现类似如下可视化AI行为树编辑器:

1. From "Three Ways of Cultivating Game AI"
    ![BehaviorEditor1](/images/game-ai-design/editor1.png)
2. A plugin for Unity Editor
    ![BehaviorEditor2](/images/game-ai-design/editor2.png)
3. CryENGINE 3
    ![BehaviorEditor3](/images/game-ai-design/editor3.png)

