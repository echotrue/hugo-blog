---
title: "Struct Interface"
date: 2022年11月21日15:06:40
draft: false
---

### interface struct 能否相互嵌套

1.  struct struct //继承(不能多态), 如果内部struct实现了接口, 它也相当于实现了接口
2.  struct interface //可以多态
3.  interface interface //单纯的导入
4.  interface struct //不允许