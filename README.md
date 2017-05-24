### Nginx  
轻量级的Web服务器/反向代理服务器及电子邮件（IMAP/POP3）代理服务器。

简单说下，就是用户的所有请求先请求到nginx服务器，然后nginx能够将用户的请求
转发到其下配置的其他容器（tomcat）中，至于具体是哪个tomcat，这对于用户来说
是不可见的。

---
#### 环境搭建
1. 下载安装需要的依赖库文件
yum install pcre 
yum install pcre-devel
yum install zlib
yum install zlib-devel

2. 解压
tar -zxvf nginx-1.13.0.tar.gz 
解压完成无需改名，因为等下安装完成后，会自动生成一个nginx文件夹

3. 检查
进行检查，看是否报错
cd /zx/nginx-1.13.0 && ./configure --prefix=/zx/nginx --conf-path=/zx/nginx/nginx.conf  
--prefix=/zx/nginx 这句指定安装后生成的文件夹名
--conf-path=/zx/nginx/nginx.conf  这句是指定conf路径，不加，下面安装的时候报了错（cp: `conf/koi-win' and `/usr/local/nginx/conf/koi-win' are the same file）
4. 安装
在nginx根目录
make && make install

5. 启动
/zx/nginx/sbin/nginx [关闭(-s stop)]  [重启(-s reload)]

查看是否启动成功
ps -ef | grep nginx
或（默认为80端口）
netstat -aon |grep 80   

浏览器访问 192.168.2.104:80 看到欢迎页面即可

---
#### 配置文件 详见 项目中的 nginx配置文件说明
根据之前的那句指令（--conf-path=/zx/nginx/nginx.conf）找到nginx.conf
vim /zx/nginx/nginx.conf

 server {
    listen       80;   #端口号
    server_name  localhost; 
    #charset koi8-r;
    #access_log  logs/host.access.log  main;
    location / {
        root   html;  #web项目路径，这个html就是相对路径，相对nginx的根目录
        index  index.html index.htm;
    }
    #error_page  404              /404.html;
    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }
 }
 
 这个server是在http{}中的，可以配置多个。
 
---
#### 日志分割
nginx/logs下有access.log 和 error.log  ，分别记录访问成功和访问失败的日志。
日志的默认记录格式是main。在配置文件中可以设置这个main的格式，或者自己新增一种格式。

简单的说下日志分割。就是写一个shell脚本，定时执行，把之前的日志备份。
http://www.cnblogs.com/benio/archive/2010/10/13/1849935.html
---
####配置文件中的location
也就是server中的
    location / {
        root   html;  #web项目路径，这个html就是相对路径，相对nginx的根目录
        index  index.html index.htm;
    }
location语法，表示uri定位.:
location = pattern {} 精准匹配
location pattern {} 一般匹配
location ~ pattern {} 正则匹配
例如location ~ test {}只使用正则test匹配

还可以写if什么的。

还可以实现动静分离，例如*.jsp请求到一个服务器，*.js/*.css/*.jpg等请求又到另一个服务器

---
### 负载均衡
* 配置反向代理
在location{}中配置:
proxy_pass http://192.168.2.105:80;

//如果是这么配置，表示使用下面配置的那个myapp负载均衡
proxy_pass http://myapp;

* 配置负载均衡
在http{}中配置：
upstream myapp{
    server 192.168.2.105:8080 weight=1 max_fails=2 fail_timeout=30s;
    server 192.168.2.106:8080 weight=1 max_fails=2 fail_timeout=30s;
}
weight是权重，
max_fails是指多少次请求失败后，认为该节点已失效
fail_timeout是请求超时时间

!!!注意，使用反向代理后获取客户端ip地址为nginx服务器地址，
这里需要nginx进行forward，设置真实的ip地址:
    #设置客户端真实ip地址
proxy_set_header X-real-ip $remote_addr;  
    #应该就是把真实ip放到http请求头的X-real-ip这个key中
---
#### nginx + keepalived 实现高可用 http://www.huangxiaobai.com/archives/1814
Keepalived是一个高性能的服务器高可用或热备解决方案。主要用来防止服务器单点故障的发生问题，
可以通过它与Nginx配合实现web服务端的高可用。
也可以通过它实现redis、mysql的高可用，也就是一个节点宕机后，自动切换另一个节点。

1. 安装软件包
yum install -y openssl openssl-devel
yum install -y libnl3.x86_64
yum install -y libnl3-devel.x86_64
yum install -y libnfnetlink.x86_64 
yum install -y libnfnetlink-devel.x86_64
2. 解压
tar -zxvf keepalived-1.3.5.tar.gz 
3. 检查 
cd /zx/keepalived-1.3.5/ && ./configure --prefix=/zx/keepealived
4. 安装 
/zx/keepalived-1.3.5 文件夹中执行
make && make install
5. 启动
/zx/keepalived -f /zx/keepalived/etc/keepalived/keepalived.conf
---
具体可以看下面这个网页（或者项目中的文档），主要就是修改了keepalived.conf,然后增加一个shell脚本，
shell执行的就是周期性检查nginx是否挂了，如果挂了就切换一台nginx
http://www.cnblogs.com/holbrook/archive/2012/10/25/2738475.html