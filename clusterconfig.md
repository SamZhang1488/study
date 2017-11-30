# 集群配置

$因前面已经完成了集群的安装，这一部分操作只需要在任意一个节点上设置即可。$

## 1．**设置集群各节点之间认证**

~~~
pcs cluster auth svrhqdmzha01 svrhqdmzha02
~~~

*$此处需要输入的用户名必须为pcs自动创建的hacluster，其他用户不能添加成功$*

## 2．创建并启动集群

创建并启动集群，其中svrhqdmzha01 svrhqdmzha02为集群成员

 ~~~
pcs cluster setup --start --name dmz_ha_cluster1 svrhqdmzha01 svrhqdmzha02
 ~~~

## 3．设置集群自启动

~~~
pcs cluster enable --all
~~~

## 4．查看集群状态

~~~
pcs cluster status
ps aux | grep pacemaker
~~~

 5．设置资源默认粘性（防止资源回切）

~~~
pcs resource defaults resource-stickiness=100

pcs resource defaults

~~~

## 6．**设置资源超时时间**

~~~
pcs resource op defaults timeout=90s
~~~



## 7．检验Corosync的安装及当前corosync状态

~~~
corosync-cfgtool -s  

corosync-cmapctl| grep members  

pcs status corosync

~~~



## 8．检查配置是否正确

~~~
crm_verify -L -V 
~~~

$注：假若没有输出任何则配置正确$

## 9．禁用STONITH

~~~
pcs property set stonith-enabled=false
~~~



## 10．设置无法仲裁配置

无法仲裁时候，选择忽略

pcs property set no-quorum-policy=ignore

 

## 11．配置VIP资源

~~~
pcs resource create vip_dmz01 ocf:IPaddr2 ip=10.0.16.17 nic='eth0' cidr_netmask='32' op monitor interval=5s timeout=20s on-fail=restart
~~~

## 12．配置haproxy资源

~~~
pcs resource create haproxy systemd:haproxy op monitor interval="5s"
~~~



## 13．定义运行的HAProxy和VIP必须在同一节点上

~~~
pcs constraint colocation add vip_dmz01 haproxy INFINITY 
~~~



## 14．定义约束，先启动VIP之后才启动HAProxy

~~~
pcs constraint order vip_dmz01 then haproxy
~~~

$备注：配置资源多节点启动，添加--clone$



