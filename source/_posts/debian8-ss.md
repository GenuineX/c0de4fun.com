---
title: 在Debian8上部署shadowsocks服务及其优化
date: 2017-08-09 22:58:30
toc: true
tags:
- Shadowsocks
- 翻墙
- Python
- Debian
---
本人作为一个后端开发程序员，虽然公司云主机+物理机一堆，但是我还是一直都想有一个属于自己的服务器，这应该是每个后端开发人员的梦想。只是一直都没决定好买来干嘛，就一拖再拖，这次正巧同事介绍再加基友合资，于是就有了今天这么一出。先买来，买来就知道干嘛了。
<!-- more -->我买的时[vultr](http://vultr.com)的Vultr Cloud Compute (VC2)，买的是5$一个月的配置，1核1G1000G流量，机器在它的硅谷机房(翻墙肯定要选海外机房，这个vultr本来也就是美国的服务商，试了东京机房，丢包丢的实在没法用)，这对于我和基友俩人平常翻个墙啊、没事儿干搭个博客啊、做点试验啊之类的足够用了(因为大一点的应用肯定都用公司的机器跑了)，选择系统的时候其实没有什么特别的原因选的[Debian](https://www.debian.org/)，只是因为平常[CentOS](https://www.centos.org/)用的太多了，Debian又那么有名，于是就装上玩玩咯，所以才选的它。  
好了，闲话就说这么多吧，无视的看官请直接忽略，以下操作请自行更新到最新版，update一下库总是要的嘛。接下来进入正文。
## 一、安装SS服务
### 1. 安装pip
``` bash  
$ apt-get install -y python-pip
```
### 2. 安装shadowsocks 
``` bash
$ pip install shadowsocks
```
### 3. 新建ss配置文件 /etc/shadowsocks.json
``` json
{
    "server":"0.0.0.0",
    "server_port":8388,
    "local_address": "127.0.0.1",
    "local_port":1080,
    "password":"your_password",
    "timeout":300,
    "method":"rc4-md4"
}
```
### 4. 启动服务
``` bash
$ ssserver -c /etc/shadowsocks.json
```
完成。是的，没错，搭建完了。就这么简单。启动完成之后检查一下进程，如果正常启动那这个服务就已经能够使用了。接下来就是下个[shadowsocks-client](https://github.com/shadowsocks)把你的地址、端口、账号、密码、加密方式五个信息一填就可以愉快的科学上网了。  
失败的话就看下启动失败的信息，没权限的加权限、有防火墙的开端口、端口占用的换端口、配置写错的自行改好，这里就不一一列举了，自己debug去吧。  
这里没有使用守护进程模式直接放入后台启动是为了你检查是否正常启动，启动后访问ss服务你的命令行会刷日志，接下来再介绍怎么放入后台执行并保活。
## 二、SS服务的优化
### 1. 安装gevent
安装gevent可以提高Shadowsocks的性能。

``` bash
$ apt-get install -y libevent
$ pip install greenlet
$ pip install gevent
```
### 2. 使用 supervisor 保活
前面提到的启动指令其实是前台启动ss服务，你能看到有请求在访问时的日志，但是实际使用我发现不管你是用tmux挂起还是别的什么办法，这种服务都是不太稳定的，总是会有一些莫名其妙的错误时你的程序退出，这时候你就需用使用ss的守护进程启动方式，并给他加上[supervisor](http://supervisord.org/)保活，这样保证你的服务稳定正常。  
* 开启守护进程模式:

``` bash
$ ssserver -c /etc/shadowsocks.json -d start
```
* 安装supervisor(apt安装默认开机启动):
    
``` bash
$ apt-get install supervisor
```
* 新建文件 /etc/supervisor/conf.d/s-server.conf，写入以下内容：
    
```
[program:ssserver]
command=/usr/local/bin/ssserver -c /etc/shadowsocks.json
directory=/tmp
user=root
autostart=true
autorestart=true
stdout_logfile=/dev/null
stderr_logfile=/dev/null
```
* 重启supervisor使配置生效：
    
``` bash
$ supervisorctl reload
```
    
### 3. 安装锐速TCP加速引擎加速你的服务  
这里就感谢大神吧，锐速官网已经打不开了，但是github仍然还有大神帮大家完成的[一键安装版](https://github.com/91yun/serverspeeder)，照着操作就行了，不过我也没觉得装好之后速度有快多少……你想装就装咯~看你自己了。
    
到这里一个完善的SS服务就应该算是搭建好了，本人及几个好基友用的都是我搭建的这个服务，服务稳定质量也还行，特此分享给大家，因为我写这篇博客记点东西主要是想面向开发人员的，所里里面涉及的具体环节，如shadowsocks的配置文件里字段含义和workers等其他字段啦、supervisor的配置文件说明及使用啦等等都统一略过，也不是本文重点，有兴趣就大家自己去看吧~ 
以上，祝好~
