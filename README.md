# V2Ray OpenWrt

## 安装[V2Ray](https://github.com/v2fly/v2ray-core)

- 安装依赖

```shell
opkg update
opkg install ca-certificates iptables-mod-tproxy
```

- 下载[V2Ray](https://github.com/v2fly/v2ray-core/releases/download/v4.35.0/v2ray-linux-64.zip)

```shell
scp v2ctl v2ray geoip.dat  geosite.dat root@192.168.50.254:/root

cp v2ctl v2ray geoip.dat  geosite.dat /usr/bin/
chmod +x /usr/bin/v2ctl /usr/bin/v2ray
```

## 配置V2Ray服务

- 编辑`/etc/config/v2ray.json`

```json

```

- 编辑`/etc/init.d/v2ray`

```
#!/bin/sh /etc/rc.common
USE_PROCD=1
START=95
STOP=01
NAME=v2ray
start_service() {
    procd_open_instance
    procd_set_param command /usr/bin/v2ray -c /etc/config/v2ray.json
    procd_set_param respawn
    procd_set_param pidfile /var/run/v2ray.pid
    procd_set_param stdout 1
    procd_set_param stderr 1
    procd_close_instance
}
```

- 运行V2Ray

```shell
chmod +x /etc/init.d/v2ray

/etc/init.d/v2ray enable
/etc/init.d/v2ray start
```

- [透明代理](https://github.com/shadowsocks/shadowsocks-libev#transparent-proxy)

```shell
iptables -t nat -N V2RAY
iptables -t nat -A V2RAY -d [V2RAY_SERVER_IP] -j RETURN
iptables -t nat -A V2RAY -d 0.0.0.0/8 -j RETURN
iptables -t nat -A V2RAY -d 10.0.0.0/8 -j RETURN
iptables -t nat -A V2RAY -d 127.0.0.0/8 -j RETURN
iptables -t nat -A V2RAY -d 169.254.0.0/16 -j RETURN
iptables -t nat -A V2RAY -d 172.16.0.0/12 -j RETURN
iptables -t nat -A V2RAY -d 192.168.0.0/16 -j RETURN
iptables -t nat -A V2RAY -d 224.0.0.0/4 -j RETURN
iptables -t nat -A V2RAY -d 240.0.0.0/4 -j RETURN
iptables -t nat -A V2RAY -p tcp -j REDIRECT --to-ports 1234
iptables -t nat -A PREROUTING -p tcp -j V2RAY
iptables -t nat -A OUTPUT -p tcp -j V2RAY
```

- 将ws的域名添加至本地hosts

```shell
opkg update
opkg install dnsmasq-full --download-only && opkg remove dnsmasq && opkg install dnsmasq-full --cache .

echo "address=/[V2RAY_SERVER_DOMAIN]/[V2RAY_SERVER_IP]" >> /etc/dnsmasq.conf
```

## [Kcptun](https://github.com/xtaci/kcptun/releases/)

- 下载[Kcptun](https://github.com/xtaci/kcptun/releases/download/v20210103/kcptun-linux-amd64-20210103.tar.gz)

```shell
scp client_linux_amd64 root@192.168.50.254:/root
cp client_linux_amd64 /usr/bin/kcptun
chmod +x /usr/bin/kcptun
```

- 编辑`/etc/config/kcptun.json`

```json
{
  "localaddr": ":10000",
  "remoteaddr": "[KCP_SERVER_IP]:4000",
  "key": "HelloKcptun!",
  "crypt": "aes-128",
  "mode": "fast3",
  "conn": 1,
  "autoexpire": 300,
  "mtu": 1400,
  "sndwnd": 512,
  "rcvwnd": 4096,
  "datashard": 30,
  "parityshard": 15,
  "dscp": 46,
  "nocomp": true,
  "acknodelay": false,
  "nodelay": 0,
  "interval": 20,
  "resend": 2,
  "nc": 1,
  "sockbuf": 4194304,
  "keepalive": 10,
  "log": "/var/log/kcptun.log"
}
```

- 编辑`/etc/init.d/kcptun`

```
#!/bin/sh /etc/rc.common
USE_PROCD=1
START=95
STOP=01
NAME=kcptun
start_service() {
    procd_open_instance
    procd_set_param command /usr/bin/kcptun -c /etc/config/kcptun.json
    procd_set_param respawn
    procd_set_param pidfile /var/run/kcptun.pid
    procd_set_param stdout 1
    procd_set_param stderr 1
    procd_close_instance
}
```

- 运行Kcptun

```shell
chmod +x /etc/init.d/kcptun

/etc/init.d/kcptun enable
/etc/init.d/kcptun start
```

## DNS防污

### 配置系统DNS

- [让dnsmasq-full将dns请求转发给127.0.0.1#5353](https://github.com/openwrt/packages/blob/master/net/stubby/files/README.md#dnssec-by-dnsmasq)
    1. Select the Network->DHCP and DNS menu entry.
    2. In the "General Settings" tab, enter the address `127.0.0.1#5353` as the only entry in the "DNS Forwardings" dialogue.
    3. In the "Resolv and Host files" tab tick the "Ignore resolve file" checkbox.

## OpenWrt作旁路由设置

- LAN (物理设置中取消桥接)

```shell
vi /etc/config/network

config interface 'lan'
	option ifname 'eth0'
	option proto 'static'
	option ipaddr '192.168.50.254'
	option netmask '255.255.255.0'
	option gateway '192.168.50.1'
	list dns '127.0.0.1#5353'
```

- 安装语言包

```shell
opkg update
opkg install luci-i18n-base-zh-cn luci-i18n-base-en
```

- 修改Web页面默认访问端口

```shell
vi /etc/config/uhttpd

config uhttpd main

	# HTTP listen addresses, multiple allowed
	list listen_http	192.168.2.1:8080
	# list listen_http	[::]:80

	# HTTPS listen addresses, multiple allowed
	list listen_https	192.168.2.1:8443
	# list listen_https	[::]:443
```

- 修改SSH默认访问端口

```shell
vi /etc/config/dropbear

config dropbear
	option PasswordAuth 'on'
	option Interface 'lan'
	option Port '8022'
```

- 修复日志时间时区问题

```shell
opkg install zoneinfo-asia

vi /etc/config/system

config system
        option zonename 'Asia/Shanghai'
        option timezone 'CST-8'
```

- 自动重启

```
# Reboot at 4:30am every day
# Note: To avoid infinite reboot loop, wait 70 seconds
# and touch a file in /etc so clock will be set
# properly to 4:31 on reboot before cron starts.
30 4 * * * sleep 70 && touch /etc/banner && reboot
```
