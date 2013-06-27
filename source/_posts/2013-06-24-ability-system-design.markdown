---
layout: post
title: "技能脚本系统设计"
date: 2013-06-24 08:59
comments: true
categories: 
---

###概述

针对现有技能系统发送消息过大, C++和lua交互频繁, 客户端功能服务器化, 触发器抽象粒度太小, 数据更新同步等问题进行脚本系统重构优化. 通过分析现有技能实现抽象出技能框架(技能引擎), 实现控制基本技能流程, 同时加入事件进制, 如攻击, 受伤等事件, 进而希望能够丰富技能同时使天赋的实现比较优雅.

###技能系统文件组织结构

* data (配置文件所在)
	+ skill (技能实例配置)
	+ buff (BUFF实例配置)
	+ talent (天赋实例配置)
	+ metadata (基本数据/元数据配置)
		- chapion.lua (各个职业基本数值配置)
		- monster.lua (各个类别怪物基本数值配置)
		- level.lua  (职业等级基本数值配置)
		- skill.lua  (各个技能基本数值配置)
* engine (技能框架,控制技能实例的运行及包含技能的公共函数等)
* lib (与具体游戏逻辑无关的公共库模块,包括基本计算几何计算等)
* skill (各个技能实例执行逻辑文件)
* buff (各个BUFF实例执行逻辑文件)
* talent (各个天赋实例执行逻辑)
* ai (怪物Ai逻辑)
* test (模块测试例程)

###模块组织结构

以lua table的形式组织各个模块组织结构

* world 整个脚本层最外层的模块，组织管理其他模块/数据
    + creatures 管理脚本中玩家/怪物对象table
    + settings 存储配置表，包括技能属性配置表，BUFF属性配置表，天赋属性配置表，元数据配置列表
        - skills 存储各个技能属性配置表
        - buffs 存储各个BUFF属性配置表
        - talents 存储各个天赋属性配置表
        - metadata 存储各类元数据配备表
            * chapions 各个职业数值配置表
            * monsters 各类型怪物数值配置表
            * levels 升级数值配置
            * skills 各个技能基本数值配置表
    + engine 服务器启动脚本文件统一装载, 技能引擎模块(流程管理, 封装技能公共函数), 玩家上下线处理过程

Example:
{% codeblock lang:lua %}
world = {
    creatures = {},
    settings = {
        skill = {},
        buff = {},
        talent = {},
        metadata = {
            chapion = {},
            monster = {},
            level = {},
            skill = {},
        }
    },
    engine = {
        skill = {}
    },
    talent = {},
    skill = {},
    buff = {},
    ai = {},
}
{% endcodeblock %}

###对象数据管理

C++中玩家或者怪物在脚本层中对应有管理其基本属性(需要在lua中管理的属性)，技能列表，BUFF列表的对象(lua table)，对象结构如下所示:

* 类型值 区分玩家和怪物
* uid 对象ID，与C++层uid对象，用于索引对象
* props 基本属性列表
* skills 技能列表
* buffs BUFF列表
* variables 保存对象在脚本执行过程中产生的临时变量
* env       技能执行相关环境变量(包括技能执行子阶段)

Example:

{% codeblock lang:lua %}
creature = {
    type = 0x0,
    uid = 123,
    props = {},
    skills = {},
    buffs = {},
    variables = {},
    env = {},
}
{% endcodeblock %}

操作接口(封装成PT使用方式), 例如:

1. AttachInPT(uid)
2. GetSkillProperty(skill_id, uid)

**Question**

1. 对象ID使用GUID还是uid
2. 目前怪物没有uid

###技能框架(engine.skill)

技能框架划分为Enter, Fire, End三个大的通用阶段, 在Fire阶段中不同的技能实例又可划分为多个阶段:

1. Enter阶段: 每个技能实例的唯一入口, 如果技能正在执行中则调用技能实例事件处理, 否则进行施法前判定并进行吟唱;
2. Fire阶段: 吟唱正常结束后进入的技能执行阶段, 该阶段控制技能实例在不同执行阶段的切换;
3. End阶段: 技能结束阶段, 执行技能清理工作, 该阶段回调技能实例onEnd执行技能实例结束处理;

技能框架示例如下:

{% codeblock lang:lua %}
module("world.engine.skill", package.seeall)

function Enter(skill_id, uid, args)
    local skill = _G["skill"..skill_id]
    local env = {skill, uid, args}
    local obj = GetObj(uid)
    if obj.stage then
        obj.stage.handleEvent(ON_INPUT, args)
    else
        if not selfOK() then
            End(env)
        else
            local on_enter = skill.onEnter
            if not on_enter(env) then
                End(env)
            else
                local onTimeout = function Fire(env) end
                local onInterrupt = function End(env) end
                Incant(env, skill.props.incantTime, onTimeout, onInterrupt)
                SyncToCLT() -- sync to client
            end
        end
    end
end

function Fire(env, cnt)
    cnt = cnt or 0
    stage = env.skill.onFire["stage_"..cnt]
    if not stage then
        End(env)
    else
        stage.start()
        local ontick = function()
            local ret = stage.update()
            if ret >= 0 then -- 转到下一个阶段
                SyncToCLT()
                Fire(env, ret)
                return false
            else            -- 转到结束阶段
                SyncToCLT()
                End(env)
                return false
            end
            return true
        end
        Schedule(ontick) -- 执行阶段循环
    end
end

function End(env)
    local skill = env.skill
    skill.onEnd(env)
    SyncToCLT()
end

{% endcodeblock %}

####技能实例部分

技能实例组成划分:

* onEnter: 吟唱前技能实例自定义过程, 返回false终止技能的执行;
* onEnd: 技能实例结束处理阶段;
* onFire: 技能执行阶段表, 该表中配置技能执行中的各个子阶段; 每个子阶段由begin, update, handleEvent三个函数组成, 分别表示子阶段开始的处理, 子阶段循环处理过程及子阶段事件处理过程; 子阶段配置格式为:

{% codeblock lang:lua %}
onFire = {
    stage_0 = {
        begin = function() end,
        update = function() end,
        handleEvent = function() end,
    }
}
{% endcodeblock %}


技能实例示例:

{% codeblock lang:lua %}
module("world.skill.1000", package.seeall)

onEnter = function()
end

onEnd = function()
end

onFire = {
    stage_0 = {
        begin = function() end,
        update = function() return 1 end,
        handleEvent = function() end,
    },
    stage_1 = {
        begin = function() end,
        update = function() return -1 end,
        handleEvent = function() end,
    },
}
{% endcodeblock %}

###属性更新机制

C++层实现一套脏属性更新机制, 提供给lua层添加更新属性请求的接口函数(该函数主要由脚本底层调用, 上层逻辑不直接调用), 属性更新系统定时或即时的编码变化的属性值给客户端.

###消息同步机制

在技能阶段过程中按消息产生的时间先后顺序将消息缓存起来, 在阶段结束时进行统一消息发送.

{% codeblock lange:c %}

// 缓存消息脚本接口
void CacheMessage(const CGUID &dst, void *args);

// 开始发送消息接口
void SyncToCLT()

{% endcodeblock %}

###事件机制

将游戏逻辑中产生的游戏事件进行统一管理注册分发, 事件主要包括:

1. 定时器事件
2. 对象濒死事件
3. 对象死亡事件
4. 碰撞事件
5. 开始移动事件
6. 停止移动事件

####C++层事件管理结构

{% codeblock lang:c %}
class EventBox {
public:
    void Connect(EventType ev); // 注册所关心的事件
    void Signal(EventType ev, void *args);    // 产生事件
    void Dispatch(); //事件分发
};

{% endcodeblock %}

####Lua层事件管理结构

脚本层中事件采用分级管理, 技能模块中管理技能中产生及所关注的事件, creature模块中管理对象产生及所关注的事件, 管理结构及事件注册分发过程如下图所示:

![EventBox](/images/ability-design/eventbox2.png)

代码示例:

{% codeblock lang:lua %}

-- event object
event = {}
function event:new(type, cb, args)
    local ev = {type = type, cb = cb, args = args}
    return ev
end

-- 事件映射
engine.events = {}

-- 事件处理函数
engine.connect = function(ev)
end

engine.schedule = function(ev)
end

engine.signal = function(ev)
end

engine.dispatch = function(ev)
end

{% endcodeblock %}

