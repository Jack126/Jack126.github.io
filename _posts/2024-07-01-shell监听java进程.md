---
layout: post
title: "shell 监听java 进程"
category: Language
tags: shell
---

shell 监听Java应用服务进程是否正常运行，并重启

#### 编写脚本

vim /opt/listen/listen_pushdemo.sh

```
#!/bin/sh
# 必须配置，引入环境变量；不然使用crond 定时执行脚本无法启动Java应用
source /etc/profile
#当前时间
now=`date +"%Y-%m-%d %H:%M:%S"`
file_name="/opt/listen/logs/pushdemo.log"  #重启脚本的日志，保证可写入，保险一点执行 chmod 777 data.log
pid=0
proc_num() 
{
    num=`ps -ef | grep 'java -jar push-0.0.1-SNAPSHOT.jar' | grep -v grep | wc -l`  #此处'java -jar push-0.0.1-SNAPSHOT.jar'替代为实际的，尽量准确，避免误kill
    return $num 
}
proc_id()
{  
    pid=`ps -ef | grep 'java -jar push-0.0.1-SNAPSHOT.jar' | grep -v grep | awk '{print $2}'`  #此处'java -jar push-0.0.1-SNAPSHOT.jar'也替代为实际的
} 
proc_num  #执行proc_num()，获取进程数
number=$?  #获取上一函数返回值
if [ $number -eq 0 ]  #如果没有该进程，则重启
then
    cd /opt/demo/push-server
    nohup java -jar push-0.0.1-SNAPSHOT.jar  &  #启动程序的命令
    proc_id 
    echo "${now} 进程数 = ${number} 重启应用服务：push -> pid = ${pid}" >> $file_name  #把重启的进程号、时间 写入日志
#else
     #echo "${now}  应用服务：push 正常 -> ${number}" >> $file_name
fi
```

#### 加入定时任务

*/1 * * * * sh /opt/listen/listen_pushdemo.sh >/dev/null 2>&1

#### 查看crond.serivce服务的自启动状态

systemctl is-enabled crond.service

#### 加入开机启动命令

chkconfig --level 35 crond on

或

systemctl enable crond.service