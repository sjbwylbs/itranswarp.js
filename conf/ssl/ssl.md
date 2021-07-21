# Linux通过Nginx配置SSL实现服务器/客户端双向认证（详细）

    修改自
    本文为CSDN博主「王绍桦」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。·
    原文链接：https://blog.csdn.net/rexueqingchun/article/details/822515631.下载tar.gz格式#

1. 下载tar.gz格式jdk并配置环境变量（已配置jdk环境的可以忽略第一步）

   1.1 解压jdk压缩包到opt目录下：

```shell
tar zxvf /opt/jdk-8u144-linux-x64.tar.gz -C /opt/
```

   1.2 配置环境变量：

   1.2.1 通过vi命令编辑profile文件

```shell
vi /etc/profile
```

   1.2.2 按键i开启编辑模式，在文件最后添加：

```shell
export JAVA_HOME="/opt/jdk1.8.0_144"
export PATH="$JAVA_HOME/bin:$PATH"
```

   备注:其中jdk1.8.0_144为jdk加压后的文件夹，修改完成后，esc键返回命令模式，输入:x保存并退出。

   1.2.3 显示jdk版本，则配置成功。

```shell
java -version
```

   备注：tomcat下载后解压即可，无需配置环境变量。

1. 安装openssl

```shell
yum install openssl
```

1. 安装nginx

3.1 配置yum源（centos不支持yum 安装 nginx，所以需要配置一下）

```shell
vi /etc/yum.repos.d/nginx.repo
```

3.2 增加如下配置：

```ini
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/7/x86_64/
gpgcheck=0
enabled=1
```

   备注： 其中"7"代表CentOS7,"x86_64"代表系统架构。

   安装nginx：

```shell
yum install nginx
```

1. 生成nginx证书

   创建证书文件夹，依次输入如下命令：

```shell
    cd /etc/nginx
    sudo mkdir ca
    cd ca
    sudo mkdir newcerts private conf server users
```

   conf目录新建openssl.conf文件：

vi /etc/nginx/ca/conf/openssl.conf

   增加如下配置：

```ini
[ca]
default_ca = myserver

[myserver]
dir = /etc/nginx/ca
database = /etc/nginx/ca/index.txt
new_certs_dir = /etc/nginx/ca/newcerts

certificate = /etc/nginx/ca/private/ca.crt
serial = /etc/nginx/ca/serial
private_key = /etc/nginx/ca/private/ca.key
RANDFILE = /etc/nginx/ca/private/.rand

default_days = 3650
default_crl_days = 3650
default_md = sha256
unique_subject = no

policy = policy_any

[policy_any]
countryName = match
stateOrProvinceName = match
organizationName = match
localityName = optional
commonName = supplied
emailAddress = optional
```

   private目录创建根证书

   生成私钥key文件：

```shell
sudo openssl genrsa -out /etc/nginx/ca/private/ca.key 2048
```

   生成根证书请求csr文件：

```shell
sudo openssl req -new -key /etc/nginx/ca/private/ca.key -out private/ca.csr
```

   生成凭证crt文件：

```shell
sudo openssl x509 -req -days 3650 -in /etc/nginx/ca/private/ca.csr -signkey /etc/nginx/ca/private/ca.key -out /etc/nginx/ca/private/ca.crt
```

   设置key起始序列号：

```shell
sudo echo FACE > /etc/nginx/ca/serial
```

   创建CA键库：

```shell
sudo touch /etc/nginx/ca/index.txt
```

   为 “用户证书” 的移除创建一个证书吊销列表：

```shell
sudo openssl ca -gencrl -out /etc/nginx/ca/private/ca.crl -crldays 7 -config "/etc/nginx/ca/conf/openssl.conf"
```

   server目录创建服务器证书

   生成私钥key文件：

```shell
sudo openssl genrsa -out /etc/nginx/ca/server/server.key 2048
```

   生成证书请求csr文件：

```shell
sudo openssl req -new -key /etc/nginx/ca/server/server.key -out /etc/nginx/ca/server/server.csr
```

   生成凭证crt文件：

```shell
sudo openssl ca -in /etc/nginx/ca/server/server.csr -cert /etc/nginx/ca/private/ca.crt -keyfile /etc/nginx/ca/private/ca.key -out /etc/nginx/ca/server/server.crt -config "/etc/nginx/ca/conf/openssl.conf"
```

   users目录创建客户端证书

   生成私钥key文件：

```shell
sudo openssl genrsa -des3 -out /etc/nginx/ca/users/client.key 2048
```

   生成证书请求csr文件：

```shell
sudo openssl req -new -key /etc/nginx/ca/users/client.key -out /etc/nginx/ca/users/client.csr
```

   生成凭证crt文件：

```shell
sudo openssl ca -in /etc/nginx/ca/users/client.csr -cert /etc/nginx/ca/private/ca.crt -keyfile /etc/nginx/ca/private/ca.key -out /etc/nginx/ca/users/client.crt -config "/etc/nginx/ca/conf/openssl.conf"
```

5.修改Nginx配置文件nginx.conf

```nginx
user  root;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
#pid        /var/run/nginx.pid;


events {
   worker_connections  1024;
}


http {
   include       /etc/nginx/mime.types;
   default_type  application/octet-stream;

   log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                     '$status $body_bytes_sent "$http_referer" '
                     '"$http_user_agent" "$http_x_forwarded_for"';

   access_log  /var/log/nginx/access.log  main;

   sendfile        on;
   #tcp_nopush     on;

   keepalive_timeout  120;

   #gzip  on;

   client_max_body_size 120m;
   client_body_buffer_size 128k;
   server_names_hash_bucket_size 128;
   large_client_header_buffers 4 4k;
   open_file_cache max=8192 inactive=20s;
   open_file_cache_min_uses 1;
   open_file_cache_valid 30s;
   
   upstream tomcat_server {
      server 192.168.1.220:8080 fail_timeout=0; 
   }

   server {
      listen 443;
      server_name 192.168.1.220;
      ssi on;
      ssi_silent_errors on;
      ssi_types text/shtml;
   
      ssl on;
      ssl_certificate           /etc/nginx/ca/server/server.crt;
      ssl_certificate_key       /etc/nginx/ca/server/server.key;
      ssl_client_certificate    /etc/nginx/ca/private/ca.crt;
      
      ssl_session_timeout 5m;
      ssl_verify_client on;
      
      ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
      ssl_ciphers ALL:!ADH:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP;
      ssl_prefer_server_ciphers on;

      charset utf-8;
      access_log logs/host.access.log main;
      error_page 500 502 503 504 /50x.html;
      location = /50x.html {
         root html;
      }

      location = /favicon.ico {
         log_not_found off;
         access_log off;
         expires 90d;
      }

      location / {
         proxy_pass http://tomcat_server;
         include proxy.conf;
      }
   }    


   #include /etc/nginx/conf.d/*.conf;
}
```

   备注：upstream tomcat_server中的server修改为nginx转发服务器的ip地址及端口号，server中server_name为nginx所在服务器ip，此处配置ssl安全认证，故采用https默认端口号443，另外安装nginx后，若根目录没有logs文件夹，可手动创建，否则启动nginx会提示找不到logs文件夹错误。

1. Nginx代理文件配置

   编辑nginx的proxy.conf文件：

vi /etc/nginx/proxy.conf

   添加如下配置：

```nginx
proxy_redirect 			off;
proxy_connect_timeout 	60;
proxy_read_timeout 		600;
proxy_set_header 		Host 				$host;
proxy_set_header 		X-Real-IP 			$remote_addr;
proxy_set_header 		X-Forwarded-For 	$proxy_add_x_forwarded_for;
proxy_set_header 		X-Forwarded-Proto 	$scheme;
proxy_set_header 		X-SSL-Client-Cert 	$ssl_client_cert;
proxy_set_header 		X-SSL-DN 			$ssl_client_s_dn;
```

1. 修改Tomcat配置文件server.xml

```xml
<!-- proxyName:双向认证服务器地址，如映射外网地址，则为外网地址 -->
<!-- proxyPort:双向认证服务端口，与Nginx配置文件中server节listen端口相同，如映射外网地址，则为外网端口 -->
<!-- scheme:双向认证服务器协议类型，此处为https -->
<Connector port="8080" protocol="HTTP/1.1" 
         connectionTimeout="20000" 
         redirectPort="8443"
         scheme="https"
         proxyName="192.168.1.220"
         proxyPort="443" />
```

   至此已配置结束，接下来我们测试一下：

1. 启动Tomcat和Nginx服务

   启动tomcat服务：

```shell
opt/apache-tomcat-8.5.20/bin/startup.sh
```

   打印启动日志：

```shell
tail -f opt/apache-tomcat-8.5.20/logs/catalina.out
```

   启动nginx服务：

```shell
nginx -c /etc/nginx/nginx.conf
```

1. 测试https请求

   把客户端证书.crt转化为Windows可安装的.p12格式

```shell
sudo openssl pkcs12 -export -clcerts -in /etc/nginx/ca/users/client.crt -inkey /etc/nginx/ca/users/client.key -out /etc/nginx/ca/users/client.p12
```

  生成后把.p12格式的证书拷贝到windows系统上安装，重启浏览器访问https请求，例如：https://192.168.1.220:443，选择刚安装的证书，能显示tomcat或者应用首页则安装成功。
  
