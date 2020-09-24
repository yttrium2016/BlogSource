---
title: v2ay Cloudflare CDN 加速访问 科学
tags:
  - 梯子
  - vps
  - google
  - 服务器
categories: 技术
abbrlink: 61662
date: 2020-09-24 15:00:00
---

## v2ay Cloudflare CDN 加速访问 科学

本质原理是将v2ray伪装成web服务，然后利用CDN进行流量转发，从而隐藏真实VPS地址。

### 0.准备工作

1. 自有域名，可配置解析
1. cloudflare帐号
1. vps

### 1.cloudflare 设置

登录 Cloudflare 进去 添加域名 设置解析ip地址到 VPS的ip地址 

更换原DNS为Cloudflare的DNS地址

	christina.ns.cloudflare.com
	sullivan.ns.cloudflare.com

>先在dns那边配置为 DNS ONLY 


### 2.vps安装 v2ay

	bash <(curl -s -L https://git.io/v2ray.sh)

一键安装脚本

#### 注意

1. 传输协议 WebSocket + TLS，要选择4
2. 端口不能为80,443
3. 域名写上面Cloudflare的填写的
4. 最好伪装一下
5. 啥环境没有安装啥


	# centos安装环境
	yum install unzip -y
	yum install curl -y


安装完后v2ray常用命令

	v2ray url 查看 V2Ray 链接
	v2ray info 查看 V2Ray 配置信息
	v2ray config 修改 V2Ray 配置
	v2ray link 生成 V2Ray 配置文件链接
	v2ray infolink 生成 V2Ray 配置信息链接
	v2ray qr 生成 V2Ray 配置二维码链接
	v2ray ss 修改 Shadowsocks 配置
	v2ray ssinfo 查看 Shadowsocks 配置信息
	v2ray ssqr 生成 Shadowsocks 配置二维码链接
	v2ray status 查看 V2Ray 运行状态
	v2ray start 启动 V2Ray
	v2ray stop 停止 V2Ray
	v2ray restart 重启 V2Ray
	v2ray log 查看 V2Ray 运行日志
	v2ray update 更新 V2Ray
	v2ray update.sh 更新 V2Ray 管理脚本
	v2ray uninstall 卸载 V2Ray


检查 v2ray url 正常 同时 caddy 正常 域名访问可以出现页面后这步OK

### 安装BBR Plus

	# 升级内核安装
	wget "https://github.com/cx9208/bbrplus/raw/master/ok_bbrplus_centos.sh" && chmod +x ok_bbrplus_centos.sh && ./ok_bbrplus_centos.sh

	# 使用该脚本切换到bbr_plus以及配置优化
	wget -N --no-check-certificate "https://raw.githubusercontent.com/chiakge/Linux-NetSpeed/master/tcp.sh" && ./tcp.sh

	# 查看状态
	lsmod | grep bbr

	# 查看当前已经使用的TCP拥塞控制配置
	cat /proc/sys/net/ipv4/tcp_congestion_control

	# 查看当前配置
	cat /etc/sysctl.conf
	
	#如果没开启,则使用下面命令
	sudo modprobe tcp_bbrplus
 
以上来自[网络教程](https://mrdear.cn/posts/tools-v2ray-cloudflare.html) centos7 安装

### 关闭防火墙

	# 查看防火墙状态
	firewall-cmd --state
	
	# 关闭防火墙
	systemctl stop firewalld.service
	
	# 关闭防火墙开机启动
	systemctl disable firewalld.service
	
### 设置Cloudflare开启CDN

在Cloudflare 后台 dns设置 灰色的小云点击一下 设置为已代理


### 偷偷使用吧

	