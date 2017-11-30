# **系统环境准备**

$案例中采用下二台CentOS7.2的虚拟机进行操作，机器名称与IP信息如下：$

~~~
svrhqdmzha01：10.0.16.15
svrhqdmzha02：10.0.16.16
~~~

$以下的安装配置分别在这2台机器上进行。$

 1．**设置内部yum源**

$如果服务器本身可以直接访问外网，可以不配置此步骤。建议统一使用内部yum源，以保证版本一致。$

- 备份系统默认的yum源配置文件；

~~~
cd /etc/yum.repos.d/

mkdir ./bk

mv ./Ce* ./bk

~~~

- 建立内部yum源

 ~~~
vi ./Easthope-Base.repo
 ~~~

~~~
[Easthope-Base]

name=CentOS-$releasever-Easthope-Base

baseurl=http://10.0.1.44/centos/7/

#CentOS版本按实际安装替换6、7

#2017-2-14增加centos/6.8

failovermethod=priority

enabled=1

gpgcheck=0

~~~

## 2．**安装时间服务**

~~~
yum install chrony

systemctl enable chronyd

vi /etc/chrony.conf

~~~

$本例中有内部NTP服务器，需要手动增加 server 10.0.16.17$

## 3．**关闭防火墙和SELinux**

~~~
systemctl disable firewalld  

systemctl stop firewalld  

iptables -F 

~~~



**修改配置文件vi /etc/selinux/config，将SELINU置为disabled**

**也可使用命令：**

~~~
sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config
~~~

  

## 4．**分别修改**二台主机的**hostname**

~~~
hostnamectl --static --transient  set-hostname svrhqdmzha01

hostnamectl --static --transient  set-hostname svrhqdmzha02

~~~

 

## 5．配置主机名节点名称

关键，集群每个节的名称都得能互相解析。/etc/hosts中的主机名配置结果必须跟”uname -n”的解析的结果一致。

~~~
vim /etc/hosts
~~~

   ~~~
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4  

::1         localhost localhost.localdomain localhost6 localhost6.localdomain6  

10.0.16.15 svrhqdmzha01 

10.0.16.16 svrhqdmzha02

   ~~~



 6．设置双机互信

$在 节点svrhqdmzha01上操作如下：$

~~~
ssh-keygen  -t rsa -f ~/.ssh/id_rsa  -P ''  

scp /root/.ssh/id_rsa.pub root@svrhqdmzha02:/root/.ssh/authorized_keys 

~~~

 $验证设置成功，可用如下命令：$

~~~
ssh SvrHQDMZHA01 'date'
~~~

 $在 节点svrhqdmzha02上操作如下:$  

~~~
ssh-keygen -t rsa -f ~/.ssh/id_rsa -P ''  

scp /root/.ssh/id_rsa.pub root@svrhqdmzha01:/root/.ssh/authorized_keys

~~~

  $验证设置成功，可用如下命令：$

 ~~~
ssh SvrHQDMZHA01 'date'
 ~~~

