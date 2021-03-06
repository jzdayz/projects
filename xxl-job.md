# 版本
2.2.0

# 白话流程
## 点点细节
- 如果客户端没有设置IP，则直接从本机网卡获取IP
- 如果没有设置端口，则从9999开始累加获取可用IP，最后再从9999累减获取
- 客户端与服务器是用的基于netty的httpServer
- GLUE java 是使用

## 客户端启动
1. 将beanfactory中的所有对象拿出来，然后挨个解析方法，解析含有XxlJob注解的方法
2. 初始化日志文件夹
3. 启动一个日志清理后台线程，每天监测遍历一次日志文件，如果达到指定的清楚时间，则清理日志文件
   - 日志文件是根据时间创建的文件夹，然后文件以数据库中的xxllog的ID为名称，eg: 2020-11-01/1223.log
4. 启动一个callback线程，专门去处理任务执行的返回值
5. 启动本机的nettyHttpServer，用来接受admin的调度
6. 启动一个后台线程，不断进行注册，也算是心跳

## 服务器
1. 启动处理注册信息的线程(只处理自动注册的执行器)，溢出过期的执行器，更新数据库中的ip地址
2. 启动处理任务重试的线程，用来处理调度失败的重试，并且进行告警
3. 启动处理任务丢失的线程，如果有任务，调度时间大于10分钟，并且调度的地址不存在于已知的执行器，则标记任务为失败
4. 启动调度线程池，调度线程池分为fast和slow线程池，默认是fast线程池，如果一个任务在一分钟内调度时间大于500ms的次数大于10次，则进入slow线程池进行调度
5. 启动一个清理过期日志的线程，根据指定的配置，进行日志清除(只清除数据库的日志)
6. 启动一个后台轮训调度线程

- 先给数据库上排他锁
- 读取接下来5s需要执行的任务
- 遍历所有任务
- 如果当前时间大于job的需要执行时间+5s，则将改任务执行时间重置为下个周期(这次不会执行调度)
- 如果当前时间可以执行job(执行时间大于当前时间)，则扔进调度池，进行任务调度，然后将任务赋予新的执行时间

