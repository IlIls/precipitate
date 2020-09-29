# wrk
https://github.com/wg/wrk

## 简介
    wrk是一个HTTP压力测试工具，能在使用单个多核CPU的情况下产生很高的负载。原因是它使用了多线程技术和可扩展的事件通知系统, 比如 select, epoll, kqueue等。

## 安装(centos 7)
    1. 安装openssl-devel
       1. sudo yum install openssl-devel -y 
    2. 安装wrk
       1. git clone https://github.com/wg/wrk.git
       2. cd wrk
       3. make

## 入门

### 命令
```
wrk -t12 -c400 -d30s http://127.0.0.1:8080/index.html
```
使用12个线程，开启400个连接请求地址 http://127.0.0.1:8080/index.html 30秒
### 输出
```
Running 30s test @ http://127.0.0.1:8080/index.html
  12 threads and 400 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   635.91us    0.89ms  12.92ms   93.69%
    Req/Sec    56.20k     8.07k   62.00k    86.54%
  22464657 requests in 30.00s, 17.76GB read
Requests/sec: 748868.53
Transfer/sec:    606.33MB
```
### 输入参数说明
```
-c，--connections： 总HTTP连接数，每个线程的连接数 = 总连接数 / 线程数
-d，--duration：    测试持续时间，例如：2s，2m，2h
-t，--threads：     总线程数
-s，--script：      lua脚本
-H，--header：      HTTP请求头
    --latency：     打印详细的响应信息
    --timeout：     设置一个时间，在该事件内未接受到返回则视为超时
```
### 输出参数说明
```
- 12 threads and 100 connections:
    12个线程,100个连接
- latency和Req/Sec:
    单个线程的统计数据,latency代表响应时间,Req/Sec代表单个线程每秒完成的请求数，分别计算平均值, 标准差, 最大值, 正负一个标准差占比。
- 23725 requests in 30.05s, 347.47MB read
    在30秒之内完成23725个请求,读取347.47MB的数据
- Socket errors: connect 0, read 48, write 0, timeout 50
    总共有48个读错误,50个超时.
- Requests/sec和Transfer/sec
    所有线程平均每秒钟完成了789.57个请求,每秒钟读取11.56MB数据量
```

## 进阶

wrk可以结合lua脚本，修改请求以及响应行为。以下为wrk提供的lua函数。也可参考官网提供的函数示例：https://github.com/wg/wrk/tree/master/scripts。
- setup函数： 在IP解析以及所有线程创建完成，且HTTP请求调用还未开始时，每个线程执行一次该函数。可以通过thread:get(name),thread:set(name,value)设置线程级别的变量。
- init函数：每次请求前被调用，可以接受wrk命令行的额外参数，通过 -- 指定。
- delay函数： 在一次请求完成后，延迟指定时间执行下一个请求。
- request函数： 在每次请求前修改请求的属性，有可能会影响测试性能。
- response函数： 返回后被调用，可以对响应做特殊处理，比如将结果输出到控制台。
- done函数： 所有请求完成后调用，可用于自定义统计结果。

命令：wrk -t 1 -c 40 -d 30s --latency --timeout 60s -s sample.lua http://api。sample.lua文件示例如下：

### 设置请求头、请求方法、请求体并打印返回状态码
```
wrk.method = "POST"
wrk.body  = '{"id":"123","name":"zhangsan"}'
wrk.headers["Content-Type"] = "application/json"
wrk.headers["apikey"] = "12345678"
wrk.headers["appkey"] = "appKey"
function request()
  return wrk.format('POST', nil, nil, body)
end

function response(status, headers, body)
    print(status)
end
```

### 从文件中读取参数
```
注：需要先创建/data/wrk/lua/ids.txt文件
idArr = {}
flag = 0
function init(args)
    for line in io.lines("/data/wrk/lua/ids.txt") do
       idArr[flag] = line
       flag = flag + 1
   end
   flag = 0
end

wrk.method = "GET"
wrk.headers["Content-Type"] = "application/json"
function request()
  rawPath="/api/v1/id/%s?x_tenant_id=1"
  parms = idArr[flag%(table.getn(idArr) + 1)]
  path=string.format(rawPath,parms)
  flag = flag+1
  return wrk.format(nil,path)
end


function response(status, headers, body)
end
```

### 自增参数
```
wrk.method = "GET"
wrk.headers["Content-Type"] = "application/json"

page=1
function request()
  page=page+1
  rawPath="/api/v1/customers?page=%s&rows=100"
  path=string.format(rawPath,page)
  return wrk.format(nil,path)
end

function response(status, headers, body)
end
```

## 注意
1. 执行wrk的机器需要有充足的临时端口号，并且能快速回收关闭的socket。需要保证listen(2)的backlog参数大于并发连接数量。（注：listen(2)的backlog参数，https://m.newsmth.net/article/LinuxDev/27088?p=1）
2. 通过脚本只改变请求的方法、路径、请求头、请求体，对性能没有影响。但是其他预处理行为，尤其是创建新的HTTP请求或使用response()方法，将必然影响到产生的负载。
3. 线程数不宜过多，建议为核数的2到4倍. 多了反而因为线程切换过多造成效率降低。因为wrk不是使用每个连接一个线程的模型,而是通过异步网络IO提升并发量.所以网络通信不会阻塞线程执行。
