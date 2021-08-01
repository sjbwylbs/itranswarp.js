# key tool gen ssl key

1. 为服务器生成证书

```shell
keytool -genkey -v -alias tomcat -keyalg RSA -keystore tomcat.keystore -validity 36500
keytool -importkeystore -srckeystore tomcat.keystore -destkeystore tomcat.p12 -deststoretype pkcs12
# can replace with on line command is below:
keytool -genkey -v -storetype PKCS12 -keystore tomcat.p12 -alias tomcat -keyalg RSA -keysize 2048 -validity 36500

```

2. 为客户端生成证书

```shell
keytool -genkey -v -alias client -keyalg RSA -storetype PKCS12 -keystore client.p12
```

3. 让服务器信任客户端证书

```shell
keytool -export -alias client -keystore client.p12 -storetype PKCS12 -storepass 密码 -rfc -file client.cer
```

    导入证书

```shell
keytool -import -v -file client.cer -keystore tomcat.p12
```

4. 让客户端信任服务器证书

```shell
keytool -keystore tomcat.p12 -export -alias tomcat -file tomcat.cer
```

5. 配置Tomcat服务器


```xml
<Connector port="8080" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="443" />


<Connector port="443" protocol="HTTP/1.1" SSLEnabled="true"
               maxThreads="150" scheme="https" secure="true"
               clientAuth="false" sslProtocol="TLS" keystoreFile="tomcat.keystore" keystorePass="密码"/>
```

```xml
# spring boot config
server.port=8443
server.ssl.key-store=classpath:ssl/tomcat.p12
server.ssl.key-store-password=PASSWORD
server.ssl.key-store-type=PKCS12
server.ssl.key-alias=tomcat
# if not need client auth please comment below.
server.ssl.client-auth=need
server.ssl.trust-store=classpath:ssl/tomcat.p12
server.ssl.trust-store-password=PASSWORD
server.ssl.trust-store-type=PKCS12
server.ssl.trust-store-provider=SUN

```
