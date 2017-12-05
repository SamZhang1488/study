# CentOS7+PCS+HAProxy实现高可用集群



- **系统环境准备**

- **安装配置pacemaker、corosync、HAProxy**

- **配置集群**

- **配置HAProxy**

  ​

  ## 一、系统环境准备

  $案例中采用下二台CentOS7.2的虚拟机进行操作，机器名称与IP信息如下：$

  ```
  svrhqdmzha01：10.0.16.15
  svrhqdmzha02：10.0.16.16
  ```

  $以下的安装配置分别在这2台机器上进行。$

  ### 1、设置内部yum源

  $如果服务器本身可以直接访问外网，可以不配置此步骤。建议统一使用内部yum源，以保证版本一致。$

  - 备份系统默认的yum源配置文件；

  ```
  cd /etc/yum.repos.d/

  mkdir ./bk

  mv ./Ce* ./bk

  ```

  - 建立内部yum源

  ```
  vi ./Easthope-Base.repo
  ```

  ```
  [Easthope-Base]

  name=CentOS-$releasever-Easthope-Base

  baseurl=http://10.0.1.44/centos/7/

  #CentOS版本按实际安装替换6、7

  #2017-2-14增加centos/6.8

  failovermethod=priority

  enabled=1

  gpgcheck=0

  ```

  ### 2、安装时间服务

  ```
  yum install chrony

  systemctl enable chronyd

  vi /etc/chrony.conf

  ```

  $本例中有内部NTP服务器，需要手动增加 server 10.0.16.17$

  ### 3．**关闭防火墙和SELinux**

  ```
  systemctl disable firewalld  

  systemctl stop firewalld  

  iptables -F 

  ```

  **修改配置文件vi /etc/selinux/config，将SELINU置为disabled**

  **也可使用命令：**

  ```
  sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config
  ```

    ### 4．**分别修改**二台主机的**hostname**

  ```
  hostnamectl --static --transient  set-hostname svrhqdmzha01

  hostnamectl --static --transient  set-hostname svrhqdmzha02

  ```

   ### 5．配置主机名节点名称

  关键，集群每个节的名称都得能互相解析。/etc/hosts中的主机名配置结果必须跟”uname -n”的解析的结果一致。

  ```
  vim /etc/hosts
  ```

  ```
  127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4  

  ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6  

  10.0.16.15 svrhqdmzha01 

  10.0.16.16 svrhqdmzha02

  ```

  ### 6．设置双机互信

  $在 节点svrhqdmzha01上操作如下：$

  ```
  ssh-keygen  -t rsa -f ~/.ssh/id_rsa  -P ''  

  scp /root/.ssh/id_rsa.pub root@svrhqdmzha02:/root/.ssh/authorized_keys 

  ```

   $验证设置成功，可用如下命令：$

  ```
  ssh SvrHQDMZHA01 'date'
  ```

   $在 节点svrhqdmzha02上操作如下:$  

  ```
  ssh-keygen -t rsa -f ~/.ssh/id_rsa -P ''  

  scp /root/.ssh/id_rsa.pub root@svrhqdmzha01:/root/.ssh/authorized_keys

  ```

    $验证设置成功，可用如下命令：$

  ```
  ssh SvrHQDMZHA01 'date'
  ```

  ​

  # **二、软件安装**

  $在2台主机上分别安装相关服务，以下步骤分别在2台机器上执行。$

  ## 1．**安装pacemaker+corosync+HAP**

  ```
  yum install pcs pacemaker corosync fence-agents-all

  yum install haproxy

  ```

  ​

  ## 2．**启动pcsd服务**并设置**开机自启动**

  ```
  systemctl start pcsd.service  

  systemctl enable pcsd.service

  ```

  ​

  ## 3．创建集群用户

  ```
  passwd hacluster
  ```

  $ps:此用户在安装pcs时候会自动创建$

  ​


# 三、集群配置

$因前面已经完成了集群的安装，这一部分操作只需要在任意一个节点上设置即可。$

## 1．**设置集群各节点之间认证**

```
pcs cluster auth svrhqdmzha01 svrhqdmzha02
```

*$此处需要输入的用户名必须为pcs自动创建的hacluster，其他用户不能添加成功$*

## 2．创建并启动集群

创建并启动集群，其中svrhqdmzha01 svrhqdmzha02为集群成员

```
pcs cluster setup --start --name dmz_ha_cluster1 svrhqdmzha01 svrhqdmzha02
```

## 3．设置集群自启动

```
pcs cluster enable --all
```

## 4．查看集群状态

```
pcs cluster status
ps aux | grep pacemaker
```

 5．设置资源默认粘性（防止资源回切）

```
pcs resource defaults resource-stickiness=100

pcs resource defaults

```

## 6．**设置资源超时时间**

```
pcs resource op defaults timeout=90s
```



## 7．检验Corosync的安装及当前corosync状态

```
corosync-cfgtool -s  

corosync-cmapctl| grep members  

pcs status corosync

```



## 8．检查配置是否正确

```
crm_verify -L -V 
```

$注：假若没有输出任何则配置正确$

## 9．禁用STONITH

```
pcs property set stonith-enabled=false
```



## 10．设置无法仲裁配置

无法仲裁时候，选择忽略

pcs property set no-quorum-policy=ignore

 

## 11．配置VIP资源

```
pcs resource create vip_dmz01 ocf:IPaddr2 ip=10.0.16.17 nic='eth0' cidr_netmask='32' op monitor interval=5s timeout=20s on-fail=restart
```

## 12．配置haproxy资源

```
pcs resource create haproxy systemd:haproxy op monitor interval="5s"
```



## 13．定义运行的HAProxy和VIP必须在同一节点上

```
pcs constraint colocation add vip_dmz01 haproxy INFINITY 
```



## 14．定义约束，先启动VIP之后才启动HAProxy

```
pcs constraint order vip_dmz01 then haproxy
```

$备注：配置资源多节点启动，添加--clone$



# 四、配置HAPROXY

$安装好HAProxy后，系统已经生成了默认的配置文件，可以直接在的默认配置上进行修改。$

```
vi /etc/haproxy/haproxy.cfg
```

```
#--------------------------------------------------------------------
# Monitor the Admin Status
#--------------------------------------------------------------------
listen admin_stat                   #status
    bind 0.0.0.0:81                #监听端口
    mode http                       #http的7层模式
    stats enable
    option httplog
    log global
    stats refresh 30s               #统计页面自动刷新时间
    stats uri /stats                #统计页面URL
    stats realm Haproxy\ Statistics #统计页面密码框上提示文本
    stats auth haproxy:haproxy          #统计页面用户名和密码设置
    stats hide-version              #隐藏统计页面上HAProxy的版本信息
    stats admin if TRUE             #手工启用/禁用,后端服务器
#---------------------------------------------------------------------
# 日常办公监听80端口设置
#---------------------------------------------------------------------
frontend  main *:80
    default_backend        mes_web

    acl mes_zone hdr_beg(host) -i mes.abc.com
    use_backend   mes_web if mes_zone

#---------------------------------------------------------------------
# 日常办公应用后端服务器设置
#---------------------------------------------------------------------
backend mes_web
    balance  roundrobin
    server   MES-1 10.98.0.253:8080 check port 8080 inter 2000 fall 3 weight 40
    server   MES-2 10.98.0.252:80 check port 80 inter 2000 fall 3 weight 30

```

$haprxy服务不用单独启动，会随着corosync一起启动$






