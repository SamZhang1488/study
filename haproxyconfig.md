# 配置HAPROXY

$安装好HAProxy后，系统已经生成了默认的配置文件，可以直接在的默认配置上进行修改。$

~~~
vi /etc/haproxy/haproxy.cfg
~~~

~~~
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

~~~

$haprxy服务不用单独启动，会随着corosync一起启动$



