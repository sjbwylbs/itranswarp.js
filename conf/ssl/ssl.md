# Linux通过Nginx配置SSL实现服务器/客户端双向认证（详细）

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

————————————————
版权声明：本文为CSDN博主「王绍桦」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/rexueqingchun/article/details/822515631.下载tar.gz格式

will PEM converted to CRT (.CRT file)

```shell
openssl x509 -outform der -in certificate.pem -out certificate.crt
```

will DER file(.crt .cer .der) converted to PEM

```shell
openssl x509 -inform der -in certificate.cer -out certificate.pem
```

will PEM Convert file to DER

```shell
openssl x509 -outform der -in certificate.pem -out certificate.der
```

will PEM Convert the certificate file and private key to PKCS＃12(.pfx .p12)

```shell
openssl pkcs12 -export -out certificate.pfx -inkey privateKey.key -in certificate.crt -certfile CACert.crt
```

Will contain the private key and certificate PKCS＃12 file(.pfx .p12) converted to PEM

```shell
openssl pkcs12 -in keyStore.pfx -out keyStore.pem -nodes
```

   You can add -nocerts to only output the private key or add -nokeys to only output the certificates.

Will contain the private key and certificate PKCS＃12 file(.pfx .p12) converted to PEM

```shell
openssl pkcs12 -in keyStore.pfx -out keyStore.pem -nodes
```

## OpenSSL conversion PEM

1. will PEM Convert to P7B

```shell
openssl crl2pkcs7 -nocrl -certfile certificate.cer -out certificate.p7b -certfile CACert.cer
```

1. will PEM Convert to PFX

```shell
openssl pkcs12 -export -out certificate.pfx -inkey privateKey.key -in certificate.crt -certfile CACert.crt
```

   will P7B Convert to PEM

```shell
openssl pkcs7 -print_certs -in certificate.p7b -out certificate.cer
```

   will P7B Convert to PFX

```shell
openssl pkcs7 -print_certs -in certificate.p7b -out certificate.cer
openssl pkcs12 -export -in certificate.cer -inkey privateKey.key -out certificate.pfx -certfile CACert.cer
```

   will PFX Convert to PEM

```shell
openssl pkcs12 -in certificate.pfx -out certificate.cer -nodes
```

OpenSSL generate rsa Key

by OpenSSL generate rsa Key

   Use on the command lineOpenSSLFirst, you need to generate the public and private keys, which should be used.-passoutThe parameter password protects this file. This parameter can be used in many different forms. See alsoOpenSSLDocumentation.

```shell
openssl genrsa -out private.pem 1024
```

   This will create a key file called private.pem that uses 1024 bits. The file actually has both a private key and a public key from which the public key can be extracted:

openssl rsa -in private.pem -out public.pem -outform PEM -pubout

    1

or
```shell
openssl rsa -in private.pem -pubout > public.pem

    1

or
```shell
openssl rsa -in private.pem -pubout -out public.pem

以上来自https://www.programmersought.com/article/695128881/



## 我从注册商处收到了一个.crt文件和一个.pem文件，但是我需要将其转换为密钥库（JKS）才能在服务器上使用它。

如何转换文件？

最佳答案

您无需将.crt或.pem文件转换为KeyStore，必须将它们添加到KeyStore。

您可以通过运行以下命令使用keytool添加它们：

keytool -importcert -keystore <KEYSTORE.JKS> -storepass <KEYSTORE_PASSWORD> -file <YOUR_CERT_OR_PEM_FILE> -alias <ALIAS_NAME>



如果该位置不存在该密钥库，则会创建一个密钥库，然后将证书添加到其中，或者如果密钥库存在，则将证书添加到其中。

然后，您可以通过运行以下命令查看是否实际添加了证书：

keytool -list -keystore <YOUR_KEYSTORE> -storepass <KEYSTORE_PASSWORD> -alias <ALIAS_NAME> -v

## 不同格式证书导入keystore方法

简介

Java自带的keytool工具是个密钥和证书管理工具。它使用户能够管理自己的公钥/私钥对及相关证书，用于（通过数字签名）自我认证（用户向别的用户/服务认证自己）或数据完整性以及认证服务。它还允许用户储存他们的通信对等者的公钥（以证书形式）。

keytool 将密钥和证书储存在一个所谓的密钥仓库（keystore）中。缺省的密钥仓库实现将密钥仓库实现为一个文件。它用口令来保护私钥。
Java KeyStore的类型

JKS和JCEKS是Java密钥库(KeyStore)的两种比较常见类型(我所知道的共有5种，JKS, JCEKS, PKCS12, BKS，UBER)。

JKS的Provider是SUN，在每个版本的JDK中都有，JCEKS的Provider是SUNJCE，1.4后我们都能够直接使用它。

JCEKS在安全级别上要比JKS强，使用的Provider是JCEKS(推荐)，尤其在保护KeyStore中的私钥上（使用TripleDes）。

PKCS#12是公钥加密标准，它规定了可包含所有私钥、公钥和证书。其以二进制格式存储，也称为 PFX 文件，在windows中可以直接导入到密钥区，注意，PKCS#12的密钥库保护密码同时也用于保护Key。

BKS 来自BouncyCastle Provider，它使用的也是TripleDES来保护密钥库中的Key，它能够防止证书库被不小心修改（Keystore的keyentry改掉1个 bit都会产生错误），BKS能够跟JKS互操作，读者可以用Keytool去TryTry。

UBER比较特别，当密码是通过命令行提供的时候，它只能跟keytool交互。整个keystore是通过PBE/SHA1/Twofish加密，因此keystore能够防止被误改、察看以及校验。以前，Sun JDK(提供者为SUN)允许你在不提供密码的情况下直接加载一个Keystore，类似cacerts，UBER不允许这种情况。
 
证书导入

Der/Cer证书导入：

要从某个文件中导入某个证书，使用keytool工具的-import命令：

keytool -import -file mycert.der -keystore mykeystore.jks

如果在 -keystore 选项中指定了一个并不存在的密钥仓库，则该密钥仓库将被创建。

如果不指定 -keystore 选项，则缺省密钥仓库将是宿主目录中名为 .keystore 的文件。如果该文件并不存在，则它将被创建。

创建密钥仓库时会要求输入访问口令，以后需要使用此口令来访问。可使用-list命令来查看密钥仓库里的内容：

keytool -list -rfc -keystore mykeystore.jks


P12格式证书导入：

keytool无法直接导入PKCS12文件。

第一种方法是使用IE将pfx证书导入，再导出为cert格式文件。使用上面介绍的方法将其导入到密钥仓库中。这样的话仓库里面只包含了证书信息，没有私钥内容。

第二种方法是将pfx文件导入到IE浏览器中，再导出为pfx文件。
       新生成的pfx不能被导入到keystore中，报错：keytool错误： java.lang.Exception: 所输入的不是一个 X.509 认证。新生成的pfx文件可以被当作keystore使用。但会报个错误as unknown attr1.3.6.1.4.1.311.17.1,查了下资料,说IE导出的就会这样,使用Netscape就不会有这个错误.

第三种方法是将pfx文件当作一个keystore使用。但是通过微软的证书管理控制台生成的pfx文件不能直接使用。keytool不认此格式，报keytool错误： java.io.IOException: failed to decrypt safe contents entry。需要通过OpenSSL转换一下：

1）openssl pkcs12 -in mycerts.pfx -out mycerts.pem

2）openssl pkcs12 -export -in mycerts.pem -out mykeystore.p12

通过keytool的-list命令可检查下密钥仓库中的内容：

keytool -rfc -list -keystore mykeystore.p12 -storetype pkcs12

这里需要指明仓库类型为pkcs12，因为缺省的类型为jks。这样此密钥仓库就即包含证书信息也包含私钥信息。

P7B格式证书导入：

keytool无法直接导入p7b文件。

需要将证书链RootServer.p7b（包含根证书）导出为根rootca.cer和子rootcaserver.cer 。

将这两个证书导入到可信任的密钥仓库中。

keytool -import -alias rootca -trustcacerts -file rootca.cer -keystore testkeytrust.jks

遇到是否信任该证书提示时，输入y

keytool -import -alias rootcaserver -trustcacerts -file rootcaserver.cer -keystore testkeytrust.jks


总结:

1)P12格式的证书是不能使用keytool工具导入到keystore中的

2)The Sun's PKCS12 Keystore对从IE和其他的windows程序生成的pfx格式的证书支持不太好.

3)P7B证书链不能直接导入到keystore，需要将里面的证书导出成cer格式，再分别导入到keystore。
————————————————
版权声明：本文为CSDN博主「peterwanghao」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/peterwanghao/article/details/1761728
