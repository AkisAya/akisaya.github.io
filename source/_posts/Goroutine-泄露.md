---
title: Goroutine 泄露案例分享
date: 2022-07-24 10:21:17
updated: 2024-08-20 01:08:17
categories: golang
tags: [golang]
---
golang 可以很方便的通过 go 关键字开一个协程，但是不正确的使用，也容易造成协程泄漏，下面分享我碰到的案例
<!--more-->
# 泄漏案例
## 1 channel 容量不当，导致发送但不接收
通常我们在主线程有一个耗时的执行逻辑，希望控制超时时间，就会对耗时逻辑开一个 goroutine，同时主线程等待 timeout
```go
ch := make(chan int)
go func() {
    time.Sleep(1 * time.Second)
    fmt.Println("")
    ch <- 1
}()

timeout := time.After(200 * time.Millisecond)
select {
    case <-ch:
    fmt.Println("run fast")
    case <-timeout:
    fmt.Println("run timeout")
}
fmt.Println("done")
```
乍一看，这个逻辑好像没有问题，但是如果一旦超时发生了我们的 goroutine 却会被阻塞在 `ch <- 1`这一行，因为超时一旦发生，select 逻辑执行完毕，最终 ch 就失去了接收者，那么发送方就会发生阻塞，导致 goroutine 无法运行结束。这里关键就在于声明 ch 的时候，没有给定容量。所以最简单的解决方法就是生命一个容量为 1 的 ch。
```go
ch := make(chan int, 1)
...
```
## 2 忘记关 channel 导致泄漏
大部分情况下，channel 不需要我们手动关闭，使用完之后会被垃圾回收掉。不过如果我们在做一个资源清理，重置之类的逻辑的时候就需要注意 channel 的使用是否被正确关闭了。
这里举一个 hystrix-go 开源库的一个忘记关闭 channel 导致泄漏的 bug，[Fix goroutines leakage when flushing hystrix configuration](https://github.com/afex/hystrix-go/pull/64), hystrix-go 有个 flush 操作，清除限流的配置，但是原来在配置的时候，开了一个 goroutine，对请求采样监控，里面有一段逻辑是
```go
// m.Updates 是一个 channel
func (m *poolMetrics) Monitor() {
	for u := range m.Updates {
		m.Mutex.RLock()

		m.Executed.Increment(1)
		m.MaxActiveRequests.UpdateMax(float64(u.activeCount))

		m.Mutex.RUnlock()
	}
}
```
也就是 channel 有接收者(此处为 m.Updates)，但是由于 flush 之后，channel 没有 close，导致这个 monitor 的逻辑不会退出。最简单的修复方式自然是 flush 时加上 close channel 的方法，具体可以参考上文中的 pr。不过 hystrix-go 这个项目基本没在维护了，这个 pr 是没有被 merge。

## 3 timer 阻塞
```go
func (l *MysqlLocker) keepAlive() {
	tick := 10
	timer := time.NewTimer(time.Duration(tick) * time.Second)
	defer timer.Stop()
	for {
		select {
		case <-timer.C:
			// do something
            doSomething()
			timer.Reset(time.Duration(tick) * time.Second)
		case <-l.donec:
			// This "if" block is important.
			// the if code block is used to discard/drain a possible timer notification
			if !timer.Stop() {
				<-timer.C
			}

			break
		}
	}
}
```
我们有个 mysql 锁，要定期给锁续期，用到了 timer 做定时任务，如果锁释放了，`l.donec`这个 channel 会受到消息，最后退出续期的逻辑
这段代码里，很容易忽略掉`if !timer.Stop() { <-timer.C }`这段 if 判断。如果没有这个判断去消费可能存在 channel 中的消息，那发送方就会阻塞，最终造成 timer 泄露。如果我们查看 timer 的 stop 方法，就会发现这个 if 判断也是推荐的做法
```go
// To ensure the channel is empty after a call to Stop, check the
// return value and drain the channel.
// For example, assuming the program has not received from t.C already:
//
// 	if !t.Stop() {
// 		<-t.C
// 	}
```
# 如何发现并排查 goroutine 泄漏
## 1 监控
首先我们可以对应用的 goroutine 进行监控，prometheus 本身就有开箱即用的监控指标，引入 prometheus sdk，在 grafana 这样子的可视化监控 dashboard 上配置好图标就行，发现不正常的增长就需要立刻排查。
## 2 pprof 查看 goroutine
go 内置的 pprof 工具可以很方便的查看 goroutine 的数量分布

# 其他
别人总结的
[Goroutine Leaks - The Forgotten Sender](https://www.ardanlabs.com/blog/2018/11/goroutine-leaks-the-forgotten-sender.html)
[如何防止 goroutine 泄露 - 掘金](https://juejin.cn/post/6844903891935461383)

