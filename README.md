# ircs
Intranet Registry Cache Server

需要频繁拉取 dockerhub、dockerhub、quay.io、nvcr.io、ghcr.io、registry.k8s.io这些镜像时

无论你的网多好，都很“慢”

在内网搭建一个镜像缓存服务器，就非常快了



> NB
> 如果你仅仅需要 dockerhub 缓存镜像
> 你可以参考 USTC 镜像站给出的教程，不必往下看了
> https://mirrors.ustc.edu.cn/help/dockerhub.html#self-host
> dockerhub的镜像比较好搭，不需要多少操作



总体思路： 

dockerhub、quay.io、nvcr.io、ghcr.io、registry.k8s.io 这些镜像需要 DNS 劫持

才能像原版一样拉取，不需要额外的加上镜像站的域名。



核心的地方有两个：

1。如何做 DNS 劫持

2。如何搭建镜像缓存服务器

你要根据实际情况做变通

在这里我只举一个 k8s 的搭建例子，其余的类似。

过程中涉及到的配置文件、脚本等，我放到了这个仓库里。

https://github.com/nealhallyoung/ircs)

## 准备

机器都安装上 docker

你需要有两个路由器

一个做主路由，一个做旁路由，并拥有其完全的控制权 

> NB 
> 你可以使用虚拟机，虚拟机全部使用 NAT 模式。
> 这样，就不需要旁路由了。

## ip 规划

整个过程涉及到机器有

主路由器、DNS 服务器、缓存服务器、旁路由、若干台需要拉取镜像的机器

主路由器网段：`192.168.3.0/24`

DNS 服务器 `192.168.3.78`

缓存服务器 `192.168.3.80`

旁路由 ip `192.168.3.79`

旁路由网段 `192.168.40.0/24`

剩下的机器，连接旁路由，全部设置 DHCP 自动获取 ip

## 搭建 DNS 服务器

```text
os debian 12.7
```

使用 coredns 来搭建

https://github.com/coredns/coredns

1。到 CoreDNS 的github项目地址那，下载二进制文件，得到可执行文件 coredns

2。编写配置文件 CoreDNS

```text
.:53 {
  # 绑定interface ip
  bind 0.0.0.0
  # 先走本机的hosts
  # https://coredns.io/plugins/hosts/
  hosts {
# 设置缓存服务器的 ip
    192.168.3.80 registry.k8s.io
    # ttl
    ttl 60
    # 重载hosts配置
    reload 1m
    # 继续执行
    fallthrough
  }

  # 最后所有的都转发到系统配置的上游dns服务器去解析
  forward . /etc/resolv.conf
  # 缓存时间ttl
  cache 120
  # 自动加载配置文件的间隔时间
  reload 6s
  # 输出日志
  log
  # 输出错误
  errors
}
```

3。放开端口 53

用你喜欢的方式放开

4。运行

```text
sudo ./coredns -conf CoreDNS
```

5。测试

```text
# 用同局域的其他的机器测试
dig registry.k8s.io @192.168.3.79
```

> NB 
> DNS 服务器的搭建，有多种方式
> 你只需要保证最后能完成 DNS 劫持就行
> 同样，我并没有给出，如何开机自启运行 coredns 的方法
> 方法太多，选择你喜欢的

## 旁路由设置

1。在 DHCP 设置中，将 DNS 服务器设置成 `192.168.3.78`

2 。测试

用旁路由下的机器测试

```text
# 不用跟上具体的 ip 地址
dig registry.k8s.io 
```

> NB
> 不同的路由器，有不同的设置方法。
> 根据实际情况操作。

## 镜像缓存服务器

1。创建docker网络

```bash
docker network create docker-registry
```

2。下载配置文件

```text
git clone --depth=1 https://github.com/nealhallyoung/ircs.git && cd ircs
```

3。生成证书

```text
./gencert.sh
```

4。启动 docker

```text
mkdir data
docker compose up -d
```

5。修改机器配置文件

在旁路由网段的机器中修改 daemon.json 文件，添加不安全镜像源

```text
vim /etc/docker/daemon.json

{
  "insecure-registries" : ["registry.k8s.io"]
}

sudo systemctl daemon-reload
sudo systemctl restart docker
```

> NB 
> 因为我们使用的是自签证书
> 也没有将证书加入到系统的信任证书中
> 所以只能将其列入不安全镜像源
> 这也是唯一不完美的地方

6。测试

```bash
docker pull registry.k8s.io/pause:3.6
docker pull registry.k8s.io/kube-apiserver:v1.23.0
```

docker-registry 是边拉取镜像，边推送镜像

再次拉取时，速度会非常快



以上有很多可以改进的地方，有时间再调整吧

一些优化方向

- nginx 换成 caddy 
- 找一些运维工具，批量地添加证书，就不用修改配置文件了
- 搭建内网 CA 证书服务器，彻底解决证书问题
- 现在有三个容器，nginx 可以和 redis 整合到一起 
- DNS服务器可以和路由器合并
- docker registry 缓存镜像的有效期最大168小时，也就是一周，是在代码里写死的，可以改一下源码，重新编译，拉长有效期

## 参考

docker registry 的文档

https://distribution.github.io/distribution/
