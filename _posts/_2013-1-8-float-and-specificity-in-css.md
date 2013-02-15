---
layout: post
title: CSS中的浮动和优先级(draft)
---
CSS选择器的优先级，可能会造成很多混乱，下面是基本规则
+ ID selectors are more specific than (and will override)
+ Class selectors, which are more specific than (and will override)
+ Contextual selectors, which are more specific than (and will override)
+ Individual element selectors

###浮动
利用浮动实现双栏布局
+ 两个div，按比较设置好各自的width。
+ 两个div都float: left，第二个会自动紧贴右边摆放。
+ 后续内容clear: left。
