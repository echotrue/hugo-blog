---
title: "Binary an Bit Operation"
date: 2021-08-27T10:34:44+08:00
draft: false
---

```Go
fmt.Printf("%d（%08b）左移1位：%d ,左移2位: %d \n", 0, 0, 0<<1, 0<<2)  
fmt.Printf("%d（%08b）左移1位：%d ,左移2位: %d \n", 1, 1, 1<<1, 1<<2)  
fmt.Printf("%d（%08b）左移1位：%d ,左移2位: %d \n", 2, 2, 2<<1, 2<<2)  
fmt.Printf("%d（%08b）左移1位：%d ,左移2位: %d \n", 3, 3, 3<<1, 3<<2)  
  
fmt.Printf("%d右移1位：%d ,右移2位: %d \n", 0, 0>>1, 0>>2)  
fmt.Printf("%d右移1位：%d ,右移2位: %d \n", 1, 1>>1, 1>>2)  
fmt.Printf("%d右移1位：%d ,右移2位: %d \n", 2, 2>>1, 2>>2)  
fmt.Printf("%d右移1位：%d ,右移2位: %d \n", 3, 3>>1, 3>>2)
```
