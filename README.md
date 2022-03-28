# how2_fuck_the_GFW_PLUS
>*这是一个基于centos7系统搭建trojan-go协议梯子的指南*

## _trojan-go_
>## 0. 准备工作
>  0.1. 检验VPS可用性——（首先更新一下各种东西yum update)  
>  0.1.1. 看有没墙ip：  
>    * 直接cmd ping ip
>    * 无比好用的网站：http://port.ping.pe/www.stanleychen.club:80
>    * 一个比较宽松的检测： https://www.vps234.com/ipchecker/  
>    * 全国ping： http://ping.chinaz.com/34.125.77.109  
>    * 节点检测：https://www.toolsdaquan.com/ipcheck/  
>  
>  0.2. 检测NF可用性：
>    * wget -O nf https://github.com/sjlleo/netflix-verify/releases/download/2.01/nf_2.01_linux_amd64 && chmod +x nf && clear && ./nf
>  
>  0.3. 然后就可以开始先去解析域名了，因为DNS解析耗时较长
>
## 1. 模块一：acme.sh  
```
1.1. 安装脚本(默认路径：/root/.acme.sh/)
curl https://get.acme.sh | sh

1.2. 用acme.sh脚本申请证书
申请证书时会占用80端口，申请时可先临时停止nginx等占用80端口的服务。(kill)

1.2.1. 使用邮箱注册账号
acme.sh --register-account -m xxx@qq.com
这一步如果acme.sh无法使用，可能是因为alias没设定，执行：alias acme.sh=~/.acme.sh/acme.sh 

1.2.2. 签发证书，your_domain.com替换成你的域名
acme.sh --issue -d your_domain.com --standalone -k ec-256
出现：Cert success. 以及 证书路径表示成功

1.2.3. 创建证书存储目录并安装证书（注意，这一步必须，因为acme.sh似乎设定不能安装在原来的issue的文件夹下）
mkdir /root/your_domain.com
acme.sh --installcert -d your_domain.com --fullchainpath /root/your_domain.com/fullchain.crt --keypath /root/your_domain.com/privkey.key
```

## 2. 模块二：nginx  
```
2.1. 引入脚本资源与安装脚本(默认路径：/etc/nginx/)
这一步是因为有可能yum没有nginx的资源
vi /etc/yum.repos.d/nginx.repo
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=0
enabled=1

yum install nginx

2.2. 配置nginx
这一部分较原网页有部分修改，暂定如下。
vi /etc/nginx/conf.d/trojan.conf
server{
    listen 1239 ssl;
    server_name your_domain.com ssl;
    root /home/www/trojan;
    index index.html;

    #ssl on;
    ssl_certificate             /root/your_domain.com/fullchain.crt;
    ssl_certificate_key         /root/your_domain.com/privkey.key;
    ssl_ciphers                 TLS13-AES-256-GCM-SHA384:TLS13-CHACHA20-POLY1305-SHA256:TLS13-AES-128-GCM-SHA256:TLS13-AES-128-CCM-8-SHA256:TLS13-AES-128-CCM-SHA256:EECDH+CHACHA20:EECDH+CHACHA20-draft:EECDH+ECDSA+AES128:EECDH+aRSA+AES128:RSA+AES128:EECDH+ECDSA+AES256:EECDH+aRSA+AES256:RSA+AES256:EECDH+ECDSA+3DES:EECDH+aRSA+3DES:RSA+3DES:!MD5;
    ssl_prefer_server_ciphers    on;
    ssl_protocols                TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_session_cache            shared:SSL:50m;
    ssl_session_timeout          1d;
    ssl_session_tickets          on;
}

server{
    listen 1234;
    return 404;
}
```

## 3. 模块三：trojan-go
```
3.1. 安装脚本(默认路径：/root/trojan-go-linux-amd64.zip)
在 https://github.com/p4gefau1t/trojan-go/releases 查看下载链接，下载解压至 /usr/local/trojan-go目录
wget https://github.com/p4gefau1t/trojan-go/releases/download/v0.10.6/trojan-go-linux-amd64.zip
unzip /root/trojan-go-linux-amd64.zip -d /usr/local/trojan-go

3.2. 配置trojan-go
将example目录下的 server.json 复制到/usr/local/trojan-go目录，修改为如下内容
cp /usr/local/trojan-go/example/server.json /usr/local/trojan-go
3.2.1. local_port为trojan-go对外端口（即客户端连接端口）
3.2.2. remote_port必须与上面配置的网站端口保持一致
3.2.3. password自行修改（客户端连接需要用到）
3.2.4. your_domain.com替换为你的域名（共2处）

这一部分较原网页有部分修改，暂定如下。
{
    "run_type": "server",
    "local_addr": "0.0.0.0",
    "local_port": 1240,
    "remote_addr": "127.0.0.1",
    "remote_port": 1239,
    "password": [
        "xxxxxxxxxxx"
    ],
    "ssl": {
        "cert": "/root/your_domain.com/fullchain.crt",
        "key": "/root/your_domain.com/privkey.key",
        "fallback_addr": "www.baidu.com",
        "fallback_port": 80
    },
    "router": {
        "enabled": true,
        "block": [
            "geoip:private"
        ]
    }
}

3.3. 创建systemctl系统启动文件
3.3.1. 复制启动文件至系统服务目录
cp /usr/local/trojan-go/example/trojan-go.service /usr/lib/systemd/system/

3.3.2. 修改启动文件，将其中启动路径，以及用户调整为如下
vi /usr/lib/systemd/system/trojan-go.service
User=root
ExecStart=/usr/local/trojan-go/trojan-go -config /usr/local/trojan-go/server.json

3.3.4. 启动服务&设置开机启动
systemctl daemon-reload
systemctl start trojan-go
systemctl enable trojan-go
```
# 参考链接
> https://www.dotatong.cn/index.php/archives/53/  
> https://dotatong.cn/index.php/archives/1/  
> https://qoant.com/2020/06/vps-with-trojan-go/  
> https://qoant.com/2019/04/vps-with-trojan/  
