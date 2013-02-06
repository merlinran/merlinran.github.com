---
layout: default
title: 一次DU故障处理小记
---
今天义务处理了一次故障，从10点接到电话出门，到凌晨1点回家，共16个小时。很久没有这么折腾了。

到达现场发现有两个磁盘笼子已经无法识别了。粗略看了一下历史事件，第一感觉是笼子本身坏了。但关系到客户数据安全，不敢独自做决策，于是寻求二线支持。事实证明，这次的二线工程师明显是新人，给的方案只是按固有流程走一遍，我等有现场经验的人一看就不靠谱。

还是本地技术支持给力，迅速响应，有清晰的plan一步步操作，不容易忽略任何一个环节。这次从他身上学到不少。

更换了多个部件后，最终还是定位到笼子身上。当更换完笼子，系统识别到了插上的第一块盘时，我忍不住欢呼起来。

这次处理过程也暴露了自己调度协调时的不足：

1. 没有在第一时间involve所有相关人。出门后在出租车上有足够时间，却没有通知此设备本来的负责人G同学跟进。直到收集了日志交给二线，开出CASE来之后，才想到要周知。导致本地技术支持晚了一个多小时才进来。

2. 没有在第一时间调集相关备件。虽然我第一反应就是笼子损坏，但却没有提醒G同学立即订件。控制卡、数据线都从本地库房送来了，笼子还没有订下来。任何DU case，收集完资料后的下一步，就是订所有可能备件。统筹方法在这里可以充分运用。

3. 没有整理好技术资料，有些用过多次的命令，在需要时无法立即找到，只能搜索。

要成功处理一个CASE，需要：

+ 足够的技术储备，能迅速确定排查方向。

+ 迅速调度各方资源，定位问题，然后在信息不完备的条件下，迅速判断方案是否靠谱，排定优先级并执行。

+ 随时掌握多方的进度，避免任何一方成为阻碍进度的瓶颈。

+ 一有空闲，向所有相关方通知进度。

+ 确定应用影响范围，安抚客户情绪。

其实要push任何一个项目或活动的进展，这几项都是必备技能。