#软件安装

$在2台主机上分别安装相关服务，以下步骤分别在2台机器上执行。$

## 1．**安装pacemaker+corosync+HAP**

~~~
yum install pcs pacemaker corosync fence-agents-all

yum install haproxy

~~~



## 2．**启动pcsd服务**并设置**开机自启动**

~~~
systemctl start pcsd.service  

systemctl enable pcsd.service

~~~



## 3．创建集群用户

~~~
passwd hacluster
~~~

$ps:此用户在安装pcs时候会自动创建$



 