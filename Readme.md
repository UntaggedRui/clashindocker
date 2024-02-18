# 使用clash作为旁路由
提供了两种方式,一种是使用docker+macvlan+iptables,另一种是使用clash的tun模式.两种方式各有优缺点,可以根据自己的需求选择.
| 方式 | 优点 | 缺点 |
| --- | --- | --- |
| docker+macvlan+iptables | 1. 对系统入侵性小 <br>2. 便于迁移 | 1. 根据网友的热心提示,只能代理TCP <br>2. 宿主机的流量没有经过代理 |
| clash tun模式 | 代理所有流量(TCP+UDP)  | 在本地会生成一些配置文件但是在指定位置 |
# 方式一: docker+macvlan+iptables
## 前言
软路由，openwrt，是老生常谈的内容了。但是我更加喜欢all in one，而且不喜欢用虚拟机。每次装openwrt的主要目的也只是使用其中的clash。所以我就干脆直接用docker+clash来充当软路由的功能了。其中使用到的主要工具是docker,macvlan,clash(mihomo),iptables.

## 创建macvlan网络
为了能够让docker启动的容器作为家庭网络中的旁路由，因此需要创建macvlan网络。
其中`192.168.3.1`为你局域网的网关，`em1`为你机器的网卡名称，这两个请根据实际情况修改。

1. （可选）让docker监听ipv6。
    编辑etc/docker/daemon.json文件
```json
{  
      "ipv6": true,  
      "fixed-cidr-v6": "2409:DA8:8001:7B22:200::/80"  
}
```

重启docker
```bash
sudo systemctl restart docker
```
    
2. 创建macvlan
    没有ipv6的版本
```bash
   docker network create -d macvlan \  
        --subnet=192.168.3.0/24 \  
        --gateway=192.168.3.1 \  
         -o parent=em1 \  
         -o macvlan_mode=bridge macnet
    
```
有ipv6的版本
```bash
    docker network create -d macvlan --ipv6 \  
        --subnet=192.168.3.0/24 \  
        --gateway=192.168.3.1 \  
        --subnet=2409:DA8:8001:7B22:200::/80 
        --gateway=2409:DA8:8001:7B22:200::1 \  
         -o parent=em1 \  
         -o macvlan_mode=bridge macnet
```    
注意看含义，有的值需要变
## 制作docker镜像并创建容器
1. 获取代码
```bash
git clone https://github.com/UntaggedRui/clashindocker
cd clashindocker
cp example.yml config.yml
```

2. 更改地址`docker-compose.yml`中的`ipv4_address`为你的ip地址.

3. 更改`config.yml`中的`proxy-provider`的`url`为你的机场订阅地址.

4. 启动容器
```bash
docker compose up -d 
```

5. 假设你的docker容器ip地址为`192.168.3.23`. 通过`http://192.168.3.23:9090/ui/`可以管理clash,进行切换节点等.后端地址为`http://192.168.3.23:9090/`,密码为`yourpassword`.

6. 在同一个局域网下,将其他机器的网关设置为`192.168.3.23`就可以实现该机器的所有流量都经过clash,并且根据clash的规则进行分流.

7. 可以不看的说明. [example.yml](./example.yml)中使用`proxy-provider`和`rule-providers`来实现远端配置. 示例中是两个机场的情况.如果只有一个机场,删除其中一个和下面`proxy-groups`对应的部分即可.这个配置文件中需要注意以下几点.

+ 配置`redir-port`来让clash能够处理请求.
```yaml
redir-port: 7892 
```

+ 配置web管理配合metacubexd来进行网页管理.
```yaml
external-controller: '0.0.0.0:9090'
external-ui: ui
# RESTful API 的口令
secret: 'yourpassword'
```

如果有无法使用的欢迎在issue中讨论.

# 方式二: clash tun模式
## 前言
根据[热心网友的提示](https://v2ex.com/t/1015815#r_14321395:~:text=1%20%E5%A4%A9%E5%89%8D-,%E8%BF%87%E6%97%B6%E7%9A%84%E6%93%8D%E4%BD%9C,-%EF%BC%8C%E5%8F%AA%E8%83%BD%E4%BB%A3%E7%90%86%20TCP),方式一的操作只能代理 TCP.并且clash本来就是单独的二进制文件,没有依赖,产生的额外文件也可控,所以可以不用使用docker进行隔离.因此这里介绍一种使用clash tun模式来做旁路由的模式.

## 准备工作
1. 获取代码
```bash
git clone https://github.com/UntaggedRui/clashindocker
cd clashindocker
cp example.yml config.yml
```

2. 更改`config.yml`中的配置. 开启tun模式,将tun的`enable`设置为`true`(第95行). 设置`proxy-provider`的`url`为你的机场订阅地址.


3. 将clash注册为系统服务方便管理. 修改`clash.service`中的`WorkingDirectory`和`ExecStart`中的路径,指向正确的位置.然后将`clash.service`放到`/lib/systemd/system/`中.然后执行如下指定,设置clash开机自启,并立即起动服务.
```bash
sudo systemctl enable clash --now
```

4. 查看clash的运行状态,此时本机流量应该是通过clash代理的.
```bash
sudo journalctl -u clash -f
```

5. 在同一个局域网下,将其他机器的网关设置为你的机器的ip地址就可以实现该机器的所有流量都经过clash,并且根据clash的规则进行分流. 为了让DNS劫持生效，设备的DNS不能设置为局域网地址，需要是公网地址或者198.18.0.1，详见官方的描述：
> Since tun.auto-route does not intercept LAN traffic, if your system DNS is set to servers in private subnets, DNS hijack will not work. You can either:
> + Use a non-private DNS server as your system DNS like 1.1.1.1
> + Or manually set up your system DNS to the Clash DNS (by default, 198.18.0.1)

关于DNS的配置参考了[这篇文章](https://www.arloor.com/posts/clash-tun-gateway/).

6. 如果其他机器能够正常上网就万事大吉不用管下面的了.如果无法上网,可能是由于iptables的原因,可以执行如下命令,开启转发.
```bash
sudo iptables -P FORWARD ACCEPT
```
如果能够正常上网就万事大吉不用管下面的了.如果还不行再加上这句试试.
```bash
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```
我不会`iptables`这个东西,很难受.
