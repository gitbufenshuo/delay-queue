# delay-queue
[![Go Report Card](https://goreportcard.com/badge/github.com/ouqiang/delay-queue)](https://goreportcard.com/report/github.com/ouqiang/delay-queue)
[![Downloads](https://img.shields.io/github/downloads/ouqiang/delay-queue/total.svg)](https://github.com/ouqiang/delay-queue/releases)
[![license](https://img.shields.io/github/license/mashape/apistatus.svg?maxAge=2592000)](https://github.com/ouqiang/delay-queue/blob/master/LICENSE)
[![Release](https://img.shields.io/github/release/ouqiang/delay-queue.svg?label=Release)](https://github.com/ouqiang/delay-queue/releases)

基于Redis实现的延迟队列, 参考[有赞延迟队列设计](http://tech.youzan.com/queuing_delay)实现

## 应用场景
* 订单超过30分钟未支付，自动关闭
* 订单完成后, 如果用户一直未评价, 5天后自动好评
* 会员到期前15天, 到期前3天分别发送短信提醒

## 支付宝异步通知实现
支付宝异步通知时间间隔是如何实现的(通知的间隔频率一般是：2m,10m,10m,1h,2h,6h,15h)  
 
订单支付成功后, 生成通知任务, 放入消息队列中.    
任务内容包含Array{0,0,2m,10m,10m,1h,2h,6h,15h}和通知到第几次N(这里N=1, 即第1次).    
消费者从队列中取出任务, 根据N取得对应的时间间隔为0, 立即发送通知.   

第1次通知失败, N += 1 => 2  
从Array中取得间隔时间为2m, 添加一个延迟时间为2m的任务到延迟队列, 任务内容仍包含Array和N     

第2次通知失败, N += 1 => 3, 取出对应的间隔时间10m, 添加一个任务到延迟队列, 同上   
......    
第7次通知失败, N += 1 => 8, 取出对应的间隔时间15h, 添加一个任务到延迟队列, 同上  
第8次通知失败, N += 1 => 9, 取不到间隔时间, 结束通知    


## 实现原理
> 利用Redis的有序集合，member为JobId, score为任务执行的时间戳,    
每秒扫描一次集合，取出执行时间小于等于当前时间的任务.   

## 依赖
* Redis



## 源码安装
* `go`语言版本1.7+
* `go get -d github.com/gitbufenshuo/delay-queue`
* `go build`


## 运行
`./delay-queue -c delay-queue.conf`  
> HTTP Server监听`0.0.0.0:9277`, Redis连接地址`127.0.0.1:6379`, 数据库编号`1`

## 客户端
[PHP](https://github.com/ouqiang/delayqueue-php)

## HTTP接口

* 请求方法 `POST`   
* 请求Body及返回值均为`json`

### 返回值
```json
{
  "code": 0,
  "message": "添加成功",
  "data": null
}
```

|  参数名 |     类型    |     含义     |        备注       |
|:-------:|:-----------:|:------------:|:-----------------:|
|   code  |     int     |    状态码    | 0: 成功 非0: 失败 |
| message |    string   | 状态描述信息 |                   |
|   data  | object, null |   附加信息   |                   |

### 添加任务   
URL地址 `/push`   
```json
{
  "topic": "order",
  "id": "15702398321",
  "delay": 3600,
  "ttr": 120,
  "body": "{\"uid\": 10829378,\"created\": 1498657365 }"
}
```
|  参数名 |     类型    |     含义     |        备注       |
|:-------:|:-----------:|:------------:|:-----------------:|
|   topic  | string     |    Job类型                   |                     |
|   id     | string     |    Job唯一标识                   | 需确保JobID唯一                  |
|   delay  | int        |    Job需要延迟的时间, 单位：秒    |                   |
|   ttr  | int        |    Job执行超时时间, 单位：秒   |                   |
|   body   | string     |    Job的内容，供消费者做具体的业务处理，如果是json格式需转义 |                   |

### 轮询队列获取任务
服务端会Hold住连接, 直到队列中有任务或180秒后超时返回,   
任务执行完成后需调用`finish`接口删除任务, 否则任务会重复投递, 消费端需能处理同一任务的多次投递   

 
URL地址 `/pop`    
```json
{
  "topic": "order"
}
```
|  参数名 |     类型    |     含义     |        备注       |
|:-------:|:-----------:|:------------:|:-----------------:|
|   topic  | string     |    Job类型                   |                     |


队列中有任务返回值
```json
{
  "code": 0,
  "message": "操作成功",
  "data": {
    "id": "15702398321",
    "body": "{\"uid\": 10829378,\"created\": 1498657365 }"
  }
}
```
队列为空返回值   
```json
{
  "code": 0,
  "message": "操作成功",
  "data": null
}
```


### 删除任务  
URL地址 `/delete`   

```json
{
  "id": "15702398321"
}
```

|  参数名 |     类型    |     含义     |        备注       |
|:-------:|:-----------:|:------------:|:-----------------:|
|   id  | string     |    Job唯一标识       |            |


### 完成任务   
URL地址 `/finish`   

```json
{
  "id": "15702398321"
}
```

|  参数名 |     类型    |     含义     |        备注       |
|:-------:|:-----------:|:------------:|:-----------------:|
|   id  | string     |    Job唯一标识    |                     |


### 查询任务  
URL地址 `/get`   

```json
{
  "id": "15702398321"
}
```

|  参数名 |     类型    |     含义     |        备注       |
|:-------:|:-----------:|:------------:|:-----------------:|
|   id  | string     |    Job唯一标识       |            |


返回值
```json
{
    "code": 0,
    "message": "操作成功",
    "data": {
        "topic": "order",
        "id": "15702398321",
        "delay": 1506787453,
        "ttr": 60,
        "body": "{\"uid\": 10829378,\"created\": 1498657365 }"
    
    }
}
```

|  参数名 |     类型    |     含义     |        备注       |
|:-------:|:-----------:|:------------:|:-----------------:|
|   topic  | string     |    Job类型                   |                     |
|   id     | string     |    Job唯一标识           |                   |
|   delay  | int        |    Job延迟执行的时间戳    |                   |
|   ttr  | int        |    Job执行超时时间, 单位：秒   |                   |
|   body   | string     |    Job内容，供消费者做具体的业务处理 | 


Job不存在返回值  
```json
{
  "code": 0,
  "message": "操作成功",
  "data": null
}
```
  
