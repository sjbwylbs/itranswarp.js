# key tool gen ssl key

1、为服务器生成证书
keytool -genkey -v -alias tomcat -keyalg RSA -keystore tomcat.keystore -validity 36500
输入密码

2、为客户端生成证书
keytool -genkey -v -alias mykey -keyalg RSA -storetype PKCS12 -keystore mykey.p12

3、让服务器信任客户端证书
keytool -export -alias mykey -keystore mykey.p12 -storetype PKCS12 -storepass 密码 -rfc -file mykey.cer
导入证书
keytool -import -v -file mykey.cer -keystore tomcat.keystore

4、让客户端信任服务器证书
keytool -keystore tomcat.keystore -export -alias tomcat -file tomcat.cer

5、配置Tomcat服务器
<Connector port="8080" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="443" />

<Connector port="443" protocol="HTTP/1.1" SSLEnabled="true"
               maxThreads="150" scheme="https" secure="true"
               clientAuth="false" sslProtocol="TLS" keystoreFile="tomcat.keystore" keystorePass="密码"/>
