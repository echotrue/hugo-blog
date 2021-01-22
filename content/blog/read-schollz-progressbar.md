---
title: "Read Schollz/Progressbar"
date: 2020-11-13T14:20:35+08:00
draft: true
tags:        [ "Go","open source","progressbar"]
categories :      [ "Go","open source","progressbar"]
---

### type ProgressBar
```
type ProgressBar struct {
    state  state // 进度条状态
    config config // 进度条配置
    lock   sync.Mutex
}

type state struct {
    currentNum        int64
    currentPercent    int
    lastPercent       int
    currentSaucerSize int
    
    lastShown time.Time
    startTime time.Time
    
    counterTime         time.Time
    counterNumSinceLast int64
    counterLastTenRates []float64
    
    maxLineWidth int
    currentBytes float64
    finished     bool
}

type config struct {
    max                  int64 // max number of the counter
    width                int
    writer               io.Writer
    theme                Theme
    renderWithBlankState bool
    description          string
    ignoreLength         bool // ignoreLength if max bytes not known
    
    // whether the output is expected to contain color codes
    colorCodes bool
    
    // show rate of change in kB/sec or MB/sec
    showBytes bool
    // show the iterations per second
    showIterationsPerSecond bool
    showIterationsCount     bool
    
    // whether the progress bar should attempt to predict the finishing
    // time of the progress based on the start time and the average
    // number of seconds between  increments.
    predictTime bool
    
    // 进度条刷新最短间隔时间
    throttleDuration time.Duration
    
    // 完成后清除进度条
    clearOnFinish bool
    
    // spinnerType should be a number between 0-75
    spinnerType int
    
    // fullWidth specifies whether to measure and set the bar to a specific width
    fullWidth bool
    
    // 完成时回调函数
    onCompletion func()             
}
```

### 源码分析
可以通过以下`function`创建`ProgressBar`实例，并渲染初始进度为0的进度条
```
func New(max int) *ProgressBar {}
func NewOptions(max int, options ...Option) *ProgressBar {}
func NewOptions64(max int64, options ...Option) *ProgressBar {}
func New64(max int64) *ProgressBar {}
func Default(max int64, description ...string) *ProgressBar {}
func DefaultBytes(maxBytes int64, description ...string) *ProgressBar {}
```

渲染进度条逻辑
1. 当上一次进度条刷新距离当前的时间小于`config.throttleDuration`且当前进度数小于进度条总数时不继续执行。
2. 清理进度条。即输出空格以填充当前行。
3. 进度条状态是未完成，且当前进度数大于进度条的总数，则设置进度条的状态为完成状态。如果`clearOnFinish`选项为false，则执行`renderProgressBar`渲染。如果
设置了`onCompletion`回调函数，则执行该回调函数
4. 如果`state.finished`为true，则不继续执行。
5. 开始渲染