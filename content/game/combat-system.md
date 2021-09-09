---
title: "Combat System"
date: 2021-09-08T14:24:16+08:00
draft: false
---

加载英雄,怪物,宠物等等到状态机的`soldiers`属性
```php
$this->loader->load();
```


##### 逐个初始化`soldiers`.挂载主动,被动技能效果并添加监听事件到`observers`属性.

保留原始属性

触发`EVENT_CREATED`事件.事件id为1的技能效果将会被加载.该类型的技能效果是永久加成效果.`observers`的数据格式为:
```json
{"3":[53102,53103,312],"9":[53104],"4":[53105],"5":[53106]}
```

计算英雄被动属性差值`diff_addition`

计算队伍加成属性,并赋值到`soldier`的`addition`属性. 

附加效果加成计算

累加加成效果数据到`soldier`

检查并削弱基础属性

处理残血模式下血量

处理蓝量

保存属性值到英雄基础属性对象`base`





