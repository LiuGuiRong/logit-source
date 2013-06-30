---
layout: post
title: "技能脚本系统实现分析"
date: 2013-05-29 16:49
comments: true
categories: game
---

**Author**: Tuz (youngtrips@mail.com)
<br>
**Date**: 05.20.2013

###Contents

* [概述](#overview)
* [服务器底层`(C++)`技能管理分析](#skill_mgr_cpp)
    * [技能C++层次类关系图](#cpp_classess)
    * [技能释放基本流程](#base_skill_proc)
    * [天赋开启流程](#enable_talent_proc)
* [脚本层技能管理分析](#skill_mgr_lua)
* [脚本中网络通信](#script_netmsg)
* [战职业技能分析](#zhan_skill)
    * [Q-连刺](#zhan_q)
    * [W-旋风](#zhan_w)
    * [E-怒吼](#zhan_e)
    * [R-冲抢](#zhan_r)
    * [战普通攻击技能](#zhan_a)
* [圣职业技能分析](#sheng_skill)
    * [Q-滑步](#sheng_q)
        * [天赋-距离](#sheng_q1)
        * [天赋-快速准备](#sheng_q2)
        * [天赋-斩杀](#sheng_q3)
    * [W-三向分身](#sheng_w)
        * [天赋-增强](#sheng_w1)
        * [天赋-鬼步](#sheng_w2)
        * [天赋-反击](#sheng_w3)
    * [E-连切](#sheng_e)
        * [天赋-影增强](#sheng_e1)
        * [天赋-嗜血](#sheng_e2)
        * [天赋-破绽](#sheng_e3)
    * [R-绝影斩](#sheng_r)
        * [天赋-追击强化](#sheng_r1)
        * [天赋-饮血](#sheng_r2)
        * [天赋-反刃](#sheng_r3)
    * [被动天赋](#passive_talent)
        * [会心一击](#sheng_b1)
        * [替身术](#sheng_b2)
        * [残影精通](#sheng_b3)

***

<h2 id="overview">概述</h2>

为了更好的优化技能脚本模块，本文主要针对职业基本属性配置、技能属性配置、技能脚本实现及与其相关联的BUFF、天赋等模块进行梳理分析。

<h2 id="skill_mgr_cpp">服务器底层(C++)技能的管理分析</h2>

<h3 id="cpp_classess">技能C++层次类关系图</h3>

![C++类关系图](/images/ability-analyze-report/classes_relation.png)

<h3 id="base_skill_proc">技能释放基本流程</h3>

技能释放由客户端输入驱动通过网络模块发送消息，服务器收到消息后查找到技能对应释放对象调用脚本系统执行对应技能的入口函数， 其执行过程如下序列图所示:
![技能释放序列图](/images/ability-analyze-report/ability_flow.png)

<h3 id="enable_talent_proc">天赋开启流程</h3>

在客户端天赋界面点选各个技能天赋确定后发消息通知服务器，服务器解析消息根据技能id及天赋id调用对应技能天赋的初始化lua函数(当玩家登陆进入服务器时也会根据天赋记录调用初始化函数)，执行序列图如下所示:

![天赋开启流程](/images/ability-analyze-report/enable_talent_proc.png)

*BTW: 当前天赋系统结构不能非常简单完美的实现重置天赋，那些根据额外的天赋属性触发的天赋可以简单的清理玩家对象的技能属性列表，但对于那些被动天赋很多都是挂触发器(Trigger)实现的，这需要删除触发器，进而涉及到触发器ID的保存问题*

技能中天赋逻辑参见[圣技能分析中天赋相关内容](#sheng_skill)

<h2 id="skill_mgr_lua">脚本层技能管理分析</h2>

当前脚本系统中根据功能大致可分为以下几个模块:

1. 职业基本属性配置模块(classAttTable): 通过lua table配置各个职业的基本数值属性、主要有基础HP、基础MP、基础护甲、基础法抗、基础攻击力，基础法术强度等与技能计算相关的属性，配置示例如下：

{% codeblock lang:lua %}
    ClassAttTable = {
	    [11] = {MaxHpBase = 550, ArmorBase = 0, SpellResistanceBase = 0, PhysicalPowerBase = 10, CriticalRating = 0,
                CriticalDamageIncrease = 1, ArmorPenetration = 0, ArmorPenetrationRatio = 0, AttackInterval = 900,
                AttackTimeBase = 1.11, SpellPowerBase = 10, SpellPenetration = 0, SpellPenetrationRatio = 0, SpellCDReduce = 0,
                MaxMpBase = 200, MpRecoverBase = 3, HpRecover = 5, SpellToughness = 0, HitProbability = 0, DodgeProbability = 0, ViewRange = 12},
    }
{% endcodeblock %}

目前该配置由策划填写的csv表格通过工具生成lua文件所得。

2. 技能数值属性(skillAttTable): 通过lua table配置各个技能的基本数值属性, 包括技能消耗、技能CD时间、技能断定距离、技能伤害类型及技能伤害，该配置与职业基本属性配置一样通过csv表格由工具生成lua文件所得；

3. 技能公共模块(serverSkillPublic): 主要包括技能流程中经常使用到的公共函数，包括施法前检查函数skillSelfOK, skillTargetOK，吟唱函数skillIncant以及BUFF公共函数等;

4. 技能属性配置模块(skill_propertyset): 配置初始化各个技能的所有属性, 其中技能伤害，技能CD等基础数值属性直接引用技能数值属性配置，技能配置中通过大量调用接口函数InitSkillProperty实现;

5. BUFF数值配置模块(buff_propertyset): 配置方式与技能属性配置相同均通过InitSkillProperty配置

6. 技能逻辑模块(skill): 各个技能的具体的执行逻辑，例如战普通攻击技能脚本逻辑/skills/skill_11001.lua

7. BUFF逻辑模块(buff)

8. 天赋系统模块(genius): 包括天赋属性配置及天赋执行逻辑函数, 目前我们有些的天赋是嵌入技能脚本中执行的;

9. 数值计算模块(math): 主要包括技能消耗计算，技能伤害计算等与数值相关的计算模块;

10. 怪物AI模块(AI): 驱动怪物行为逻辑.

这些模块通过脚本逻辑或者BUFF逻辑组织起来组成完成的技能逻辑，下面具体分析战及圣各个技能脚本实现。

<h2 id="script_netmsg">脚本中网络通信</h2>

目前项目里lua脚本与客户端的通信可以通过C++提供的功能接口进行间接通信，还可以通过调用C++提供的接口RunClientScript与客户端进行直接通信，该函数功能是通过发送客户端定义好的lua函数名及序列化好的参数调用客户端的逻辑脚本，目前脚本中大量使用RunClientScript主要:
    
1. 怪物AI中同步怪物属性及动作;
2. 副本相关逻辑脚本;
3. 技能脚本中控制技能特效、发送技能伤害及资源消耗;
        
    * 发送命中伤害显示playHitVisToClient()
            
        ![playHitVisToClient](/images/ability-analyze-report/playHitVisToClient.png)

    * 向客户端发送某目标免疫、闪避、抵抗的消息playStateInvalidToClient()

        ![playStateInvalidToClient](/images/ability-analyze-report/playStateInvalidToClient.png)

    * skillMessageToClient(主要是控制动作特效)skillMessageToClient()

        ![skillMessageToClient](/images/ability-analyze-report/skillMessageToClient.png)

    * 消耗资源，同时发送相关显示消息给客户端resCostAndMsgToClient()

        ![resCostAndMsgToClient](/images/ability-analyze-report/resCostAndMsgToClient.png)
        
4. 数值计算模块中更新角色属性(playerAttCaculate).

    ![playerAttCaculate](/images/ability-analyze-report/playerAttCaculate.png)


<h2 id="zhan_skill">战职业技能分析</h2>

战技能大都通过SkillStage技能属性控制技能的每一阶段的逻辑执行, 下面具体看看每个技能的执行序列图。

<h3 id="zhan_q">Q-连刺</h3>
    
* 需求:
    
    连续对前面扇形范围内全部敌人造成数次伤害，之后，对更大范围内敌人造成一次高伤害但自身有收招硬直的攻击

* 序列图:

    ![11003序列图](/images/ability-analyze-report/zhan_11002.png)

<h3 id="zhan_w">W-旋风</h3>

* 需求:
    
    前冲一小段距离，并连续打击周围的敌人，可再次点击热键来释放终结技

* 序列图:
    
    ![11003序列图](/images/ability-analyze-report/zhan_11003.png)

<h3 id="zhan_e">E-怒吼</h3>

* 需求:

    技能施放后，战士发出一声怒吼，以自身为中心x格（暂定6）范围内敌人减速y%（暂定40%），持续z秒（暂定3

* 时序图:

    ![11003序列图](/images/ability-analyze-report/zhan_11004.png)

<h3 id="zhan_r">R-冲枪</h3>

* 需求:

    对方向前冲，行径中对前面区域多次造成伤害并击退，击退距离和战每段冲刺距离一样，方向为被击退者对于战的方向。过程中再次热键释放终结技升龙劈

* 时序图:

    ![11005序列图](/images/ability-analyze-report/zhan_11005.png)

<h3 id="zhan_a">战普通攻击技能(11001)</h3>

* 时序图:

    ![11001序列图](/images/ability-analyze-report/zhan_11001.png)

<h2 id="sheng_skill">圣职业技能分析</h2>

<h3 id="sheng_q">Q-滑步</h3>

* 需求:

    在出现选择框后可以再次点击该技能，选择框会出现些许形状变化，此操作用于区分真身位置，默认状况是残影前冲攻击，再次点击则是真身前冲残影留在原地

* 时序图:

    ![41002序列图](/images/ability-analyze-report/sheng_41002.png)

    <h4 id="sheng_q1">天赋-距离</h4>

    * 需求: 增加流影斩最大可移动距离25%
    * 实现分析: 根据添加的技能属性Gis_ImproveDisRatio修改技能属性MaxDis值, 其过程如下时序图所示:

        ![gis_sheng_q1](/images/ability-analyze-report/gis_sheng_q1.png)

    <h4 id="sheng_q2">天赋-快速准备</h4>

    * 需求: 当流影斩成功命中敌人时，会立即缩短流影斩一半的冷却时间
    * 实现分析: 开启该天赋会添加新技能属性Gis_cutCDTime，当Q命中目标时尝试调用gis_sheng.on_cutdown_cdtime(),其过程如下图所示:

        ![gis_sheng_q2](/images/ability-analyze-report/gis_sheng_q2.png)


    <h4 id="sheng_q3">天赋-斩杀</h4>

    * 需求: 当目标的当前血量少于20%，圣会尝试斩杀对方，造成基础攻击300%的伤害，只有真身出击时才可触发这个效果
    * 实现分析: 开启该天赋会添加新技能属性Gis_lopHpRatio及Gis_lopDamRatio, 当对目标伤害结算前尝试调用gis_sheng.try_lop_target()增加伤害并伤害结算,其过程如下图所示:

        ![gis_sheng_q3](/images/ability-analyze-report/gis_sheng_q3.png)

<h3 id="sheng_w">W-三向分身</h3>

* 需求:

    出现方向的选择框并确认后，角色将朝选定的方向滑出，与角色呈120度夹角的另外2个方向会有2个残影以同样的姿势滑出

* 时序图:

    ![41003序列图](/images/ability-analyze-report/sheng_41003.png)

    <h4 id="sheng_w1">天赋-增强</h4>

    * 需求: 使用三项分身之后，获得BUFF，圣及其残影的所有伤害提升30%，持续1.5秒
    * 实现分析: 技能释放后调用gis_sheng.try_improve_all_damage()检查是否启用该天赋并添加增益BUFF, 其过程如下所示:

        ![gis_sheng_w1](/images/ability-analyze-report/gis_sheng_w1.png)

    <h4 id="sheng_w2">天赋-鬼步</h4>

    * 需求: 被圣及其分身所碰到每个目标都将被减速30%，持续2秒
    * 实现分析: 施法前调用gis_sheng.try_guibu()检查是否启用改天赋并添加碰撞触发器, 其过程如下所示:

        ![gis_sheng_w2](/images/ability-analyze-report/gis_sheng_w2.png)

    <h4 id="sheng_w3">天赋-反击</h4>

    * 需求:使用三向分身后角色会获得反击状态2秒，在此状态持续时圣受到攻击时，残影会瞬移到攻击者身旁重击他使之昏迷1秒，触发这个效果后BUFF会消失
    * 实现分析: 技能释放后调用gis_sheng.try_counterattack()检查是否启用该技能，如果启用添加反击BUFF，反击BUFF给宿主对象注册伤害触发器，当触发器触发时触发影子进行攻击，其过程如下所示:

        ![gis_sheng_w3](/images/ability-analyze-report/gis_sheng_w3.png)

<h3 id="sheng_e">E-连切</h3>

*  需求:

    按下快捷键，角色朝鼠标方向发动攻击，没按一次追加一次攻击，最多五次进入冷却时间（前面攻击长时间未按键也会进入冷却）,每次攻击会使目标定身1秒

* 时序图:

    ![41004序列图](/images/ability-analyze-report/sheng_41004.png)

    <h4 id="sheng_e1">天赋-影增强</h4>
    * 需求: 连切引发的影子联动攻击的伤害提高50%
    * 实现分析: 影子攻击前调用 gis_sheng.try_improve_blur_damage()提供技能伤害数值并返回恢复函数，攻击过程结束时执行恢复，其过程如下所示:
        
        ![gis_sheng_e1](/images/ability-analyze-report/gis_sheng_e1.png)
        
    <h4 id="sheng_e2">天赋-嗜血</h4>
    * 需求: 圣的连切会让对方暴露弱点，使对方承受的圣的伤害提升30%，这个DEBUFF持续1.5秒
    * 实现分析: 伤害结算前调用gis_sheng.try_be_bloodthirsty()给目标添加DEBUFF，其过程如下所示:
        
        ![gis_sheng_e2](/images/ability-analyze-report/gis_sheng_e2.png)
        
    <h4 id="sheng_e3">天赋-破绽</h4>
    * 需求: 3次连续被击退的敌人将被击倒(2.5秒)
    * 实现分析: 当发生3次攻击时调用gis_sheng.try_be_rip()给目标添加击倒DEBUFF，其过程如下所示:
        
        ![gis_sheng_e3](/images/ability-analyze-report/gis_sheng_e3.png)

<h3 id="sheng_r">R-绝影斩</h3>

* 需求:

    按键后出现锥形选择框, 选择方向后角色做出攻击准备姿势，在2秒后对区域发动沉重AOE攻击;同时圆形提示框内的残影会立即瞬移到箭头区域对区域内的敌人实施攻击，每个残影的攻击会使敌人昏迷0.5秒

* 时序图:

    ![41006序列图](/images/ability-analyze-report/sheng_41006.png)

    <h4 id="sheng_r1">天赋-追击强化</h4>
    * 需求: 被残影追击攻击到的敌人将被昏迷
    * 实现分析: 当影攻击到目标添加昏迷BUFF前调用gis_sheng.try_enhance_pursue()增加昏迷BUFF时间，其过程如下所示:

        ![gis_sheng_r1](/images/ability-analyze-report/gis_sheng_r1.png)

    <h4 id="sheng_r2">天赋-饮血</h4>
    * 需求: 每击中一个敌人，会给圣叠加一层饮血BUFF，在10秒内回复最大血量的10%，最多可叠加5层
    * 实现分析: 当攻击到目标伤害结算后调用gis_sheng.try_bebloodthirsty_with_healthregen()添加饮血BUFF，其过程如下所示:

        ![gis_sheng_r2](/images/ability-analyze-report/gis_sheng_r2.png)

    <h4 id="sheng_r3">天赋-反刃</h4>
    * 需求: 到达目标点之后，圣再次施放乱舞从而返回起始位置，再次造成伤害
    * 实现分析: 当技能释放后调用gis_sheng.try_anti_blade()返回lua闭包函数并执行该闭包，该闭包注册移动停止触发器，当移动停止后调整面向并调用skill_41006_cast()以达到再次施法，其过程如下图所示:

        ![gis_sheng_r3](/images/ability-analyze-report/gis_sheng_r3.png)

<h3 id="passive_talent">被动天赋</h3>

<h4 id="sheng_b1">会心一击</h4>
* 需求: 圣的普通攻击内置CD延长至2倍于目前时长，但会造成1.5倍于目前伤害的物理伤害
* 实现分析: 启用该天赋直接修改普通攻击技能CD时间, 示意图略

<h4 id="sheng_b2">替身术</h4>
* 需求: 只要还有自身残影在场，圣便不会死去，当此时圣受到致命伤害时，圣和残影位置互换，由残影来遭受此次伤害
* 实现分析: 启用该天赋时给玩家对象身上保存拦截死亡HOOK(*BTW: 由于结构的原因只能用这种取巧的方式实现*)，该HOOK函数主要功能是与周围最近的影子交换位置并取消其所受伤害减少的HP，其过程如下图所示:

    ![gis_sheng_b1](/images/ability-analyze-report/gis_sheng_b2.png)

<h4 id="sheng_b3">残影精通</h4>
* 需求: 当残影由于受到伤害消失时，将回复圣能量30点，并会使残影位置上的敌人减速30%，持续1.5秒
* 实现分析: 启用该天赋时添加一个新技能41010(用于承载资源消耗)，当影子受伤消失时执行该技能的资源消耗并给周围敌对目标添加DEBUFF，其过程如下图所示:

    ![gis_sheng_b3](/images/ability-analyze-report/gis_sheng_b3.png)

