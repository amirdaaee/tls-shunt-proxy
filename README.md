# TLS 分流器
[Telegram](https://t.me/tls_shunt_proxy)

用于分流 TLS 流量，适用于 vmess + TLS + Web 方案实现，并可以与 trojan 共享端口。
* sni 分流
* http 和无特征流量分流
* 静态网站服务器
* 自动获取证书

## 下载安装
对于 linux-amd64 可以使用脚本安装，以 root 身份执行以下命令
```shell script
bash <(curl -L -s https://raw.githubusercontent.com/liberal-boy/tls-shunt-proxy/master/dist/install.sh)
```
* 配置文件位于 `/etc/tls-shunt-proxy/config.yaml`

* 其他平台需自行编译安装

## 使用
命令行参数：
```
  -config string
        Path to config file (default "./config.yaml")
```

<details>
  <summary>点击此处展开示例配置文件</summary>
  
```yml
# listen: 监听地址
listen: 0.0.0.0:443

# redirecthttps: 监听一个地址，发送到这个地址的 http 请求将被重定向到 https
redirecthttps: 0.0.0.0:80

# inboundbuffersize: 入站缓冲区大小，单位 KB, 默认值 4
# 相同吞吐量和连接数情况下，缓冲区越大，消耗的内存越大，消耗 CPU 时间越少。在网络吞吐量较低时，缓存过大可能增加延迟。
inboundbuffersize: 4

# outboundbuffersize: 出站缓冲区大小，单位 KB, 默认值 32
outboundbuffersize: 32

# vhosts: 按照按照 tls sni 扩展划分为多个虚拟 host
vhosts:

    # name 对应 tls sni 扩展的 server name
  - name: vmess.example.com

    # tlsoffloading: 解开 tls，true 为解开，解开后可以识别 http 流量，适用于 vmess over tls 和 http over tls (https) 分流等
    tlsoffloading: true

    # managedcert: 管理证书，开启后将自动从 LetsEncrypt 获取证书，根据 LetsEncrypt 的要求，必须监听 443 端口才能签发
    # 开启时 cert 和 key 设置的证书无效，关闭时将使用 cert 和 key 设置的证书
    managedcert: false

    # keytype: 启用 managedcert 时，生成的密钥对类型，支持的选项 ed25519、p256、p384、rsa2048、rsa4096、rsa8192
    keytype: p256

    # cert: tls 证书路径，
    cert: /etc/ssl/vmess.example.com.pem

    # key: tls 私钥路径
    key: /etc/ssl/vmess.example.com.key

    # alpn: ALPN, 多个 next protocol 之间用 "," 分隔
    alpn: h2,http/1.1

    # protocols: 指定 tls 协议版本，格式为 min,max , 可用值 tls12(默认最小), tls13(默认最大)
    # 如果最小值和最大值相同，那么你只需要写一次
    # tls12 仅支持 FS 且 AEAD 的加密套件
    protocols: tls12,tls13

    # http: 识别出的 http 流量的处理方式
    http:

      # paths: 按 http 请求的 path 分流，从上到下匹配，找不到匹配项则使用 http 的 handler
      paths:

          # path: path 以该字符串开头的请求将应用此 handler
        - path: /vmess/ws/
          handler: proxyPass
          args: 127.0.0.1:40000

          # path: http/2 请求的 path 将被识别为 *
        - path: "*"
          handler: proxyPass
          args: 127.0.0.1:40003

        - path: /static/

          # trimprefix: 修剪前缀，将 http 流量交给 handler 时，修剪 path 中的前缀
          # 如将 /static/logo.jpg 修剪为 /logo.jpg
          trimprefix: /static

          handler: fileServer
          args: /var/www/static

      # handler: fileServer 将服务一个静态网站
      # fileServer 支持 h2c, 如果使用 fileServer 处理 http, 且未设置 paths, alpn 可以开启 h2
      handler: fileServer

      # args: 静态网站的文件路径
      args: /var/www/html

    # default: 其他流量处理方式
    default:

      # handler: proxyPass 将流量转发至另一个地址
      handler: proxyPass

      # args: 转发的目标地址
      args: 127.0.0.1:40001

      # args: 也可以使用 domain socket
      # args: unix:/path/to/ds/file

  - name: trojan.example.com

    # tlsoffloading: 解开 tls，false 为不解开，直接处理 tls 流量，适用于 trojan-gfw 等
    tlsoffloading: false

    # default: 关闭 tlsoffloading 时，目前没有识别方法，均按其他流量处理
    default:
      handler: proxyPass
      args: 127.0.0.1:8443
```
</details>

## 故障排查和常见问题

1. service 启动失败，请先使用命令 `sudo setcap "cap_net_bind_service=+ep" /usr/local/bin/tls-shunt-proxy` 给 tls-shunt-proxy 赋予 Capabilities ，然后 `sudo -u tls-shunt-proxy /usr/local/bin/tls-shunt-proxy -config /etc/tls-shunt-proxy/config.yaml` 运行，获取错误信息

2. `fail to load tls key pair for xxx.xxx: open /xxx/xxx.key: permission denied` 确保用户 `tls-shunt-proxy` 有权读取证书
