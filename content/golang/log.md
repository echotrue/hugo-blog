---
title: "Log"
date: 2021年8月4日14:28:53
draft: false
tags:        [ "Go","log"]
categories :      [ "Go","log"]
---

### Write log to both console and file
```
consoleWriter := os.Stdout // os.Stderr  
w, err := os.OpenFile("./app.log", os.O_APPEND|os.O_CREATE|os.O_RDWR, os.ModeAppend|os.ModePerm)  
if err != nil {  
 log.Fatal(err)  
}  
logWriter := io.MultiWriter(consoleWriter, w)  
l := log.New(logWriter, "--->", log.LstdFlags|log.Lshortfile)  
l.Println("this is a log")
```