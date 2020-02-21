---
layout: post
title: hystrix-go源码分析
date: 2020-02-19 19:43:30
categories: go
tags: hystrix
---

在微服务中基础服务延迟高，服务间调用超时时间设置不合理等原因导致服务整体不可用，这称为雪崩效应。为了解决雪崩，需要设置合理的服务间超时时间，采用熔断、限流等措施，避免服务整体不可用。
[hystrix-go](https://github.com/afex/hystrix-go)是个有熔断、限流功能的开源的库，这个库来源于java版的hystrix库。  

### 怎样使用   
hystrix-go的官方文档对如何使用作了简单介绍。 介绍了如何执行自定义的方法、定义自定义的失败执行方法等。
定义失败处理示例代码：
```
hystrix.Go("my_command", func() error {
	// talk to other services
	return nil
}, func(err error) error {
	// do this when services are down
	return nil
})
```
完整的例子可以参考hystrix-go源码目录下loadtest服务。  

### 源码类图   
利用类图可以帮助我们很好的理解代码。下面是hystrix-go源码的部分代码类图。
![](/images/hystrix-go/hystrix-go.png)

### 源码分析  
1、配置  
hystrix-go支持按Command进行个性化配置。在程序启动时，可以调用hystrix.ConfigureCommand()对每个Command对象进行配置。
```
hystrix.ConfigureCommand("my_command", hystrix.CommandConfig{
	Timeout:                1000,  
	MaxConcurrentRequests:  100,
	SleepWindow:            int(time.Second * 5),
	RequestVolumeThreshold: 30,
	ErrorPercentThreshold:  25,
})
```  
CommandConfig字段的意义如下表： 

| 字段| 含义| 默认值|   
| ---- | --- | --- |      
| Timeout |执行的超时时间 | 1000毫秒|   
| MaxConcurrentRequests | 最大并发量 | 10 |    
| SleepWindow | 熔断器打开，控制下次单次测试是否恢复 | 5000毫秒|    
| RequestVolumeThreshold | 一个统计窗口的容量值，超过后判断是否开启熔断| 20|    
| ErrorPercentThreshold | 请求错误百分比，请求超过RequestVolumeThreshold且错误量达到此值，则开启熔断| 50|    

2、统计收集器   
MetricCollector表示收集熔断器的所有统计信息，比如调用次数、失败次数、被拒绝次数等，DefaultMetricCollector代表默认的统计收集器。
```
type DefaultMetricCollector struct {
	mutex *sync.RWMutex

	numRequests *rolling.Number
	errors      *rolling.Number

	successes               *rolling.Number
	failures                *rolling.Number
	rejects                 *rolling.Number
	shortCircuits           *rolling.Number
	timeouts                *rolling.Number
	contextCanceled         *rolling.Number
	contextDeadlineExceeded *rolling.Number

	fallbackSuccesses *rolling.Number
	fallbackFailures  *rolling.Number
	totalDuration     *rolling.Timing
	runDuration       *rolling.Timing
}
```
关键的rolling.Number保存每个变量的值，每个变量保存最近10s的值，统计间隔是1s。 rolling.Number成员Buckets是个map，key为最近10s内的时间戳，值为64位浮点数。这是怎么实现的呢？
```
func (r *Number) getCurrentBucket() *numberBucket {
	now := time.Now().Unix()
	var bucket *numberBucket
	var ok bool

	if bucket, ok = r.Buckets[now]; !ok {
		bucket = &numberBucket{}
		r.Buckets[now] = bucket
	}

	return bucket
}

func (r *Number) Increment(i float64) {
	if i == 0 {
		return
	}

	r.Mutex.Lock()
	defer r.Mutex.Unlock()

	b := r.getCurrentBucket()
	b.Value += i
	r.removeOldBuckets()
}

```
添加值时先查询是否有当前1s时间戳的数据，否则新建Bucket，将新值放在新的Bucket中。最后调用removeOldBuckets方法。
```
func (r *Number) removeOldBuckets() {
	now := time.Now().Unix() - 10

	for timestamp := range r.Buckets {
		// TODO: configurable rolling window
		if timestamp <= now {
			delete(r.Buckets, timestamp)
		}
	}
}
```
removeOldBuckets方法主要是遍历Buckets判断key的时间戳是否小于最近10s，来删除数据。   

3、状态上报   
CircuitBreaker（熔断器)->metricExchange(metric交换器)->MetricCollector(metric收集器)

每个Command执行后将状态码(错误码)转成事件码，调用熔断器的reportEvent方法，写入metric交换器的update管道中，metric交换器开启协程调用IncrementMetrics方法写入到统计收集器中。
```
func (m *metricExchange) IncrementMetrics(wg *sync.WaitGroup, collector metricCollector.MetricCollector, update *commandExecution, totalDuration time.Duration) {
	// granular metrics
	r := metricCollector.MetricResult{
		Attempts:         1,
		TotalDuration:    totalDuration,
		RunDuration:      update.RunDuration,
		ConcurrencyInUse: update.ConcurrencyInUse,
	}

	switch update.Types[0] {
	case "success":
		r.Successes = 1
	case "failure":
		r.Failures = 1
		r.Errors = 1
	case "rejected":
		r.Rejects = 1
		r.Errors = 1
	case "short-circuit":
		r.ShortCircuits = 1
		r.Errors = 1
	case "timeout":
		r.Timeouts = 1
		r.Errors = 1
	case "context_canceled":
		r.ContextCanceled = 1
	case "context_deadline_exceeded":
		r.ContextDeadlineExceeded = 1
	}

	if len(update.Types) > 1 {
		// fallback metrics
		if update.Types[1] == "fallback-success" {
			r.FallbackSuccesses = 1
		}
		if update.Types[1] == "fallback-failure" {
			r.FallbackFailures = 1
		}
	}

	collector.Update(r)

	wg.Done()
}

```
collector是统计收集器，最终事件码转换metric结果保存在统计收集器中。

4、流量控制  
hystrix-go实现了对流量控制，采用的是令牌算法，请求方法执行前，先请求令牌，拿到令牌后执行请求，请求执行完成后返还令牌，得不到令牌拒绝执行请求方法，如果设置了callback方法，接着执行callback方法。如果没设置callback方法，则不执行。
每个CircuitBreaker熔断器都有个executorPool变量指向一个executorPool结构体。executorPool结构表示流控器，其中的变量max表示最大的令牌量，可以通过MaxConcurrentRequests配置，Tickets变量表示令牌管道，存放令牌。
```
type executorPool struct {
	Name    string
	Metrics *poolMetrics
	Max     int
	Tickets chan *struct{}  // 存放令牌
}
```  
executorPool的Return方法表示请求完成后返还令牌的方法。

```
func (p *executorPool) Return(ticket *struct{}) { //返还令牌
	if ticket == nil {
		return
	}

	p.Metrics.Updates <- poolMetricsUpdate{
		activeCount: p.ActiveCount(),
	}
	p.Tickets <- ticket
}
```   
5、流量状态上报   
在executorPool结构体中还有个Metrics变量，看名字我们知道这个用来收集thread Pool统计信息的。 返还令牌时，调用executorPool的Return方法，向poolMetrics的Updates管道写入数据，开启协程消费poolMetrics的Updates管道的数据更新MaxActiveRequests和Executed值。
```
func (m *poolMetrics) Monitor() {
	for u := range m.Updates {
		m.Mutex.RLock()

		m.Executed.Increment(1)
		m.MaxActiveRequests.UpdateMax(float64(u.activeCount))

		m.Mutex.RUnlock()
	}
}
```    
6、GoC整个执行过程   
GoC是个非常重要的基础函数，Go、Do、Doc都会调用此函数。
```
func GoC(ctx context.Context, name string, run runFuncC, fallback fallbackFuncC) chan error {
	cmd := &command{
		run:      run,
		fallback: fallback,
		start:    time.Now(),
		errChan:  make(chan error, 1),
		finished: make(chan bool, 1),
	}

	circuit, _, err := GetCircuit(name)
	if err != nil {
		cmd.errChan <- err
		return cmd.errChan
	}
	cmd.circuit = circuit
	ticketCond := sync.NewCond(cmd)
	ticketChecked := false

	returnTicket := func() { // 还回令牌环函数
		cmd.Lock()
		// Avoid releasing before a ticket is acquired.
		for !ticketChecked {
			ticketCond.Wait()
		}
		cmd.circuit.executorPool.Return(cmd.ticket)
		cmd.Unlock()
	}
	returnOnce := &sync.Once{}
	reportAllEvent := func() { // 调用熔断器的函数上报状态
		err := cmd.circuit.ReportEvent(cmd.events, cmd.start, cmd.runDuration)
		if err != nil {
			log.Printf(err.Error())
		}
	}

	go func() {
		defer func() { cmd.finished <- true }()
		if !cmd.circuit.AllowRequest() { // 判断熔断器的状态
			cmd.Lock()
			// It's safe for another goroutine to go ahead releasing a nil ticket.
			ticketChecked = true
			ticketCond.Signal()
			cmd.Unlock()
			returnOnce.Do(func() {
				returnTicket()
				cmd.errorWithFallback(ctx, ErrCircuitOpen)
				reportAllEvent()
			})
			return
		}
		cmd.Lock()
		select {
		case cmd.ticket = <-circuit.executorPool.Tickets: // 分配令牌
			ticketChecked = true
			ticketCond.Signal()
			cmd.Unlock()
		default:
			ticketChecked = true
			ticketCond.Signal()
			cmd.Unlock()
			returnOnce.Do(func() {
				returnTicket() // 返还令牌
				cmd.errorWithFallback(ctx, ErrMaxConcurrency)
				reportAllEvent()
			})
			return
		}

		runStart := time.Now()
		runErr := run(ctx) // run 表示真正执行的函数
		returnOnce.Do(func() {
			defer reportAllEvent()
			cmd.runDuration = time.Since(runStart)
			returnTicket()
			if runErr != nil {
				cmd.errorWithFallback(ctx, runErr)
				return
			}
			cmd.reportEvent("success")
		})
	}()

	go func() {
		timer := time.NewTimer(getSettings(name).Timeout)
		defer timer.Stop()

		select {
		case <-cmd.finished:
			// returnOnce has been executed in another goroutine
		case <-ctx.Done():
			returnOnce.Do(func() {
				returnTicket()
				cmd.errorWithFallback(ctx, ctx.Err())
				reportAllEvent()
			})
			return
		case <-timer.C:
			returnOnce.Do(func() {
				returnTicket()
				cmd.errorWithFallback(ctx, ErrTimeout)
				reportAllEvent()
			})
			return
		}
	}()

	return cmd.errChan
}

```   
![](/images/hystrix-go/command.png)
7、监控     
服务的metrics收集方式有两种：一种是监控程序主动来服务拉取metrics数据。使用hystrix-go的服务需要在main.go中，在一个特定的端口注册http handler事件并开启一个goroutine监听端口。
```
hystrixStreamHandler := hystrix.NewStreamHandler()
hystrixStreamHandler.Start()
go http.ListenAndServe(net.JoinHostPort("", "81"), hystrixStreamHandler)

```  
接着需要在hystrix dashboard上配置拉取metrics的路径。为了快速测试，hystrix dashboard采用docker镜像的方式来部署。
```
docker run -d -p 8888:9002 --name hystrix-dashboard mlabouardy/hystrix-dashboard:latest
```  
hystrix-go源码目录下有个loadtest服务，加上主动拉取监控metrics的代码来使用。
![](/images/hystrix-go/1582273115.png)
![](/images/hystrix-go/1582273210.png)

第二种方式是服务主动推送metrics到收集metircs的服务中。比如服务主动推送metrics信息到statsd中，使用hystrix-go的示例代码如下。
```
c, err := plugins.InitializeStatsdCollector(&plugins.StatsdCollectorConfig{
	StatsdAddr: "localhost:8125",
	Prefix:     "myapp.hystrix",
})
if err != nil {
	log.Fatalf("could not initialize statsd client: %v", err)
}

metricCollector.Registry.Register(c.NewStatsdCollector)
```  
statsd是个收集metrics的守护程序。它收集的metircs数据可以保存在类似InfluxDB的程序中，通过Grafana等前端程序拉取展示。这里采用docker镜像的方式[samuelebistoletti/docker-statsd-influxdb-grafana](https://github.com/samuelebistoletti/docker-statsd-influxdb-grafana)来快速部署整个程序。
![](/images/hystrix-go/1582276570.png)
