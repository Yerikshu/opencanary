OpenCanary
=================
Thinkst Applied Research

![opencanary logo](docs/logo.png)

Overview
----------

OpenCanary is a daemon that runs several canary versions of services that alerts when a service is (ab)used.

Prerequisites
----------------

* Python 2.7
* [Optional] SNMP requires the python library scapy
* [Optional] RDP requires the python library rdpy
* [Optional] Samba module needs a working installation of samba


以上皆为原作者的配置原文，后续的功能开发皆是现阶段思考开发的结果

Install
----------

通过docker进行安装，因为在实践过程中发现，现在在内网中很多主机压根不提供外网服务，而且随着系统的升级迭代python2逐渐被淘汰，大量的主机内核支持虚拟化服务，通过docker进行部署，将极大的提供蜜罐agent的推广程度和部署的敏捷度

# 环境配置
## 宿主机rsyslog配置
记录访问日志的agent服务
```
sed -i '50i kern.*                                              /var/log/kern.log' /etc/rsyslog.conf
chkconfig --level 2345 rsyslog on && service rsyslog restart
```
之后去对应的目录下`/var/log/kern.log`查看对应的文件格式，某些情况下会被配置成dir的形式，需要自己手动修改，如
```
rm -rf kern.log
touch kern.log
```

## 宿主机防火墙策略配置

### 安装iptables service
cat /etc/redhat-release   ## 查看redhat小版本
我这里暂时提供redhat7.6所需要的iptable的[安装包](https://github.com/Yerikshu/opencanary/releases/download/7.6/iptables-services-1.4.21-28.el7.x86_64.rpm)，后续根据需要在逐步添上
```
rpm -ivh iptables-services-1.4.21-28.el7.x86_64.rpm 
service iptables start
```

### 添加iptables策略
直接在命令行添加以下命令
```
iptables -t mangle -A PREROUTING -p tcp -i lo -j LOG --log-level=warning --log-prefix="canaryfw: " -m limit --limit="3/hour"
iptables -t mangle -A PREROUTING -p tcp --syn -j LOG --log-level=warning --log-prefix="canaryfw: " -m limit --limit="5/second" !  -i lo
iptables -t mangle -A PREROUTING -p tcp --tcp-flags ALL URG,PSH,SYN,FIN -m u32 --u32 "40=0x03030A01 && 44=0x02040109 && 48=0x080Affff && 52=0xffff0000 && 56=0x00000402" -j LOG --log-level=warning --log-prefix="canarynmap: " -m limit --limit="5/second"
iptables -t mangle -A PREROUTING -p tcp -m u32 --u32 "6&0xFF=0x6 && 0>>22&0x3C@12=0x50000400" -j LOG --log-level=warning --log-prefix="canarynmapNULL: " -m limit --limit="5/second"
iptables -t mangle -A PREROUTING -p tcp -m u32 --u32 "6&0xFF=0x6 && 0>>22&0x3C@12=0x50290400" -j LOG --log-level=warning --log-prefix="canarynmapXMAS: " -m limit --limit="5/second"
iptables -t mangle -A PREROUTING -p tcp -m u32 --u32 "6&0xFF=0x6 && 0>>22&0x3C@12=0x50010400" -j LOG --log-level=warning --log-prefix="canarynmapFIN: " -m limit --limit="5/second"
```
```
iptables -t mangle  -L  ## 确认策略已添加
service iptables save  ## 保存策略
```
### 测试扫描日志

```
curl http://127.0.0.1:443
```
查看是否出现源地址为本机，目的端口为443的日志

```
tail -f /var/log/kern.log 
canaryfw: IN=lo OUT= MAC=00:00:00:00:00:00:00:00:00:00:00:00:08:00 SRC=127.0.0.1 DST=127.0.0.1 LEN=40 TOS=0x00 PREC=0x00 TTL=64 ID=54161 DF PROTO=TCP SPT=443 DPT=39104 WINDOW=0 RES=0x00 ACK RST URGP=0
```

## docker环境配置

通过sftp上传docker二进制文件及蜜罐镜像之后解压缩
```
tar -zxvf docker-19.03.9.tgz && cp docker/* /usr/bin/ && tar -zxvf honey-agent-2-0.tar.gz
```
启动docker服务   
```
dockerd &
```
导入镜像
```
docker load < honey-agent-2-0
```
创建容器并运行镜像，映射kern.log文件
```
setenforce 0
docker run -itd --name honey  --network=host -v /var/log/kern.log:/var/log/kern.log honeypot-agent:2.0
```
进入容器
```
docker exec -it honey bash
vi /root/.opencanary.conf  --->>>修改节点名称以及配置对应的master地址，其他不用改
opencanaryd --start
```


# 自动化安装
如果当前主机支持访问外网，以下指令就可以解决安装问题
```
docker pull ccr.ccs.tencentyun.com/otherproject/honeypot-agent:2.1
```



# agen版本介绍
自己个性化部署后续陆续加上，还没来得及整理好

## 2.0
基础版本，由于之前在测试的时候已经做了一系列的迭代，所以这个公开的时候就直接上升到2.0，后续都在这个基础上升级迭代，需要的话自行在release取用下载使用

## 2.1
- 增加crontab功能
- 对agent状态进行监测
- 宕机自动恢复

## 2.2
- 增加网络服务检测控件

# 后续规划
- 引入消息队列来处理日志服务

# 后面有时间的话再提供物理机独立部署的攻略，也是挺麻烦的

# 注意：超级坑爹的bug  
这个是有时候在部署的过程中会出现这个错误：
```
2020-06-29T15:42:30+0800 [-] Unhandled Error
        Traceback (most recent call last):
          File "/usr/local/lib/python2.7/site-packages/twisted/python/log.py", line 103, in callWithLogger
            return callWithContext({"system": lp}, func, *args, **kw)
          File "/usr/local/lib/python2.7/site-packages/twisted/python/log.py", line 86, in callWithContext
            return context.call({ILogContext: newCtx}, func, *args, **kw)
          File "/usr/local/lib/python2.7/site-packages/twisted/python/context.py", line 122, in callWithContext
            return self.currentContext().callWithContext(ctx, func, *args, **kw)
          File "/usr/local/lib/python2.7/site-packages/twisted/python/context.py", line 85, in callWithContext
            return func(*args,**kw)
        --- <exception caught here> ---
          File "/usr/local/lib/python2.7/site-packages/twisted/internet/posixbase.py", line 614, in _doReadOrWrite
            why = selectable.doRead()
          File "/usr/local/lib/python2.7/site-packages/twisted/internet/inotify.py", line 249, in doRead
            fdesc.readFromFD(self._fd, self._doRead)
          File "/usr/local/lib/python2.7/site-packages/twisted/internet/fdesc.py", line 94, in readFromFD
            callback(output)
          File "/usr/local/lib/python2.7/site-packages/twisted/internet/inotify.py", line 276, in _doRead
            iwp._notify(path, mask)
          File "/usr/local/lib/python2.7/site-packages/twisted/internet/inotify.py", line 150, in _notify
            callback(self, filepath, events)
          File "/usr/local/lib/python2.7/site-packages/opencanary/modules/__init__.py", line 169, in onChange
            self.processAuditLines()
          File "/usr/local/lib/python2.7/site-packages/opencanary/modules/__init__.py", line 161, in processAuditLines
            self.handleLines(lines=lines)
          File "/usr/local/lib/python2.7/site-packages/opencanary/modules/portscan.py", line 57, in handleLines
            self.logger.log(data)
          File "/usr/local/lib/python2.7/site-packages/opencanary/logger.py", line 176, in log
            scheduler = TwistedScheduler()
          File "/usr/local/lib/python2.7/site-packages/apscheduler/schedulers/base.py", line 83, in __init__
            self.configure(gconfig, **options)
          File "/usr/local/lib/python2.7/site-packages/apscheduler/schedulers/base.py", line 122, in configure
            self._configure(config)
          File "/usr/local/lib/python2.7/site-packages/apscheduler/schedulers/twisted.py", line 37, in _configure
            super(TwistedScheduler, self)._configure(config)
          File "/usr/local/lib/python2.7/site-packages/apscheduler/schedulers/base.py", line 694, in _configure
            self.timezone = astimezone(config.pop('timezone', None)) or get_localzone()
          File "/usr/local/lib/python2.7/site-packages/tzlocal/unix.py", line 165, in get_localzone
            _cache_tz = _get_localzone()
          File "/usr/local/lib/python2.7/site-packages/tzlocal/unix.py", line 128, in _get_localzone
            utils.assert_tz_offset(tz)
          File "/usr/local/lib/python2.7/site-packages/tzlocal/utils.py", line 46, in assert_tz_offset
            raise ValueError(msg)
        exceptions.ValueError: Timezone offset does not match system offset: 39600 != 28800. Please, check your config files.
```
原因是时区问题，解决办法如下：
修改 /usr/local/lib/python2.7/site-packages/opencanary/logger.py
在176行，scheduler = TwistedScheduler()，修改为scheduler = TwistedScheduler(timezone="Asia/Shanghai")  ，强行设置为与操作系统相同的东八区即可

