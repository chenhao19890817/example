# 案例：docker 公共镜像代理实现方案
## 首先在局域网内确保有一台服务器可以访问外网，假设该服务器IP地址是10.10.82.224，
1. 在该服务器上安装squid代理软件
* ~]# yum install -y squid
* ~]# vi /etc/squid/squid.conf

   acl localnet src 10.10.0.0/16   # RFC1918 possible internal network
   acl SSL_ports port 443
   acl Safe_ports port 80          # http
   http_access allow  Safe_ports
   http_access allow  SSL_ports
   http_access allow localnet
   http_port 3128  transparent  加上transparent表示是透明代理
   cache_dir ufs /var/spool/squid 100 16 256
   其余配置不做改动
 * ~]# service squid start
 *  ~]# ss -tunl 查看3128端口是否监听
 *  ~]# iptables -t nat -A  PREROUTING -i eth0 -s 10.10.0.0/16  -p tcp -m multiport  --dports     80,443  -j  REDIRECT  --to-port 3128    
 *  这条命令表示在nat表的PREROUTING链上做端口转发，把对80和443端口的请求转发到3128端口

2. 在需要安装docker的服务器上正常安装好docker-ce，由于docker现在工作在代理模式下，所以需要增加如下两个配置文件
 * ~]# mkdir -p /etc/systemd/system/docker.service.d
 * ~]# vi /etc/systemd/system/docker.service.d/https-proxy.conf

       [Service]
       Environment="HTTPS_PROXY=https://10.10.82.224:3128/"  填写代理服务器的IP地址和端口
 * ~]# vi /etc/systemd/system/docker.service.d/http-proxy.conf
       [Service]
       Environment="HTTP_PROXY=http://10.10.82.224:3128/"
 * ~]# systemctl daemon-reload                   表示Reload systemd manager configuration 
 * ~]# systemctl restart docker.service
 * ~]# systemctl show --property=Environment docker  
   ​     Environment=HTTP_PROXY=http://10.10.82.224:3128/      HTTPS_PROXY=https://10.10.82.224:3128/
 + 表示查看docker服务的指定属性，如果不加--property=Environment是查看所有属性
   如果如下两条测试命令测试成功，说明docker代理配置正常
 * ~]# docker search centos
 * ~]# docker pull centos