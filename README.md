**前言：**

​    在生产接到监控人员告警，有一台服务器的网卡来回切换，询问是否异常。原因是这台服务器之前在测试区时双网卡的模式为双活，上到生产环境后就出现了来回切换现象。本文在esxi环境模拟双网卡配置和测试。



**环境说明：**

| 主机名 |  操作系统版本   |      ip      | ESXi 版本 |      备注      |
| :----: | :-------------: | :----------: | :-------: | :------------: |
| client | Centos 7.6.1810 | 172.27.34.85 |   6.5.0   | 双网卡测试主机 |

## 一、构造双网卡测试环境

系统目前只有一张网卡，需构造双网卡环境。

### 1.现有环境查看

![image-20201117105005446](https://i.loli.net/2020/11/19/LlAcSjKzUed862F.png)

网卡ifcfg-ens160配置：

```bash
[root@client network-scripts]# more ifcfg-ens160
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
NAME=ens160
UUID=003981c1-76e4-4a67-9f84-f42cb033bbba
DEVICE=ens160
ONBOOT=yes
IPADDR=172.27.34.85
PREFIX=24
GATEWAY=172.27.34.1
IPV6_PRIVACY=no
DNS1=202.xxx.xxx.xxx
```

DNS根据实际情况填写

### 2.新增一个网卡

![image-20201117102209500](https://i.loli.net/2020/11/19/1sDBmioCw3b9HYy.png)

现有环境只有一张网卡

![image-20201117102242714](https://i.loli.net/2020/11/19/gtIUkwaxizNYm9L.png)

新增一张网卡

### 3.查看新增网卡

![image-20201117105132102](https://i.loli.net/2020/11/19/i1rOKhPUqcgExzJ.png)

## 二、双网络卡配置

### 1.新建ifcfg-bond0

```bash
[root@client network-scripts]# touch ifcfg-bond0
[root@client network-scripts]# more ifcfg-bond0 
TYPE=Bond
BOOTPROTO=static
DEFROUTE=yes
DEVICE=bond0
USERCTL=no
ONBOOT=yes
IPADDR=172.27.34.85
PREFIX=24
GATEWAY=172.27.34.1
DNS1=202.xxx.xxx.xxx
BONDING_OPTS="miimon=100 mode=1"
```

新建网卡文件ifcfg-bond0并配置。mode=1：主备模式，只有一张网卡工作，当主网卡失效时会切换到备网卡；mode=0：负载均衡模式，两块网卡都工作，提供两倍带宽。网卡模式可根据生产实际情况选择。

### 2.配置网卡ifcfg-ens160

```bash
[root@client network-scripts]# more ifcfg-ens160
TYPE=Ethernet
BOOTPROTO=static
NAME=eno2
HWADDR=00:0c:29:c8:de:24
DEVICE=ens160
USERCTL=no
ONBOOT=yes
MASTER=bond0
SLAVE=yes

```

### 3.配置ifcfg-ens190

```bash
[root@client network-scripts]# touch ifcfg-ens190
[root@client network-scripts]# more ifcfg-ens190 
TYPE=Ethernet
BOOTPROTO=static
NAME=eno2
HWADDR=00:0c:29:c8:de:2e
DEVICE=ens190
USERCTL=no
ONBOOT=yes
MASTER=bond0
SLAVE=yes
```

新建网卡文件ifcfg-ens190并配置

### 4.重启网络

```bash
[root@client ~]# systemctl restart network
```

重启网络或主机

### 5.查看网络

![image-20201119162219208](https://i.loli.net/2020/11/19/zmtX7jyQDwfplO9.png)

网卡bond0已经绑上了ip 172.27.34.85

## 三、双网卡切换测试

### 1.主网口查看

```bash
[root@client ~]#  cat /proc/net/bonding/bond0 
```

![image-20201119162447591](https://i.loli.net/2020/11/19/UuOnGXAoB53FK2k.png)

双网卡模式为主备，主网卡为ens160

### 2.关闭ens160

```bash
[root@client ~]# ifdown ifcfg-ens160
成功断开设备 'ens160'。
[root@client ~]#  cat /proc/net/bonding/bond0    
```

![image-20201119162731004](https://i.loli.net/2020/11/19/eJCvQxh7EwRrb2t.png)

此时主网卡为ens190，网络连接正常

### 3.启动ens160

```bash
[root@client ~]# ifup ifcfg-ens160
连接已成功激活（D-Bus 活动路径：/org/freedesktop/NetworkManager/ActiveConnection/11）
[root@client ~]#  cat /proc/net/bonding/bond0      
```

![image-20201119162842163](https://i.loli.net/2020/11/19/zJCsvZA79gT4qdk.png)

启动ens160，此时主网卡还是ens190，网络连接正常

测试完成，双网卡主备模式有效。
