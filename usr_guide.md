## 网络资源

### docker的网络访问
请参考docker部分

### 代理
工作站配置了http代理，一般情况下默认启动了代理
```
-e HTTP_PROXY=http://192.168.151.57:7897 \
-e HTTPS_PROXY=http://192.168.151.57:7897 \
-e http_proxy=http://192.168.151.57:7897 \
-e https_proxy=http://192.168.151.57:7897 \
```


#### github的ssh配置
请用户先配置好ssh连接github所需要的密钥
因为校园网的DNS解析问题（似乎是因为tcp端口22被校园网拦截），我们采用**SSH overHTTPS**的方案把 SSH 流量封装在 HTTPS 端口上，从而绕过网络对 22 端口的封锁

```
Host github.com
    Hostname ssh.github.com
    Port 443 #GitHub 提供了一个备用端口：443（HTTPS 端口）
    #使用corkscrew工具用代理转发TCP流量（访问外网需要经过代理）
    ProxyCommand corkscrew 127.0.0.1 7897 %h %p 
    
    User git
    IdentityFile ~/.ssh/github_key
```

### 远程桌面
工作站安装了xfce4，推荐使用xfce4+xrdp模式的远程桌面

#### 启动远程桌面服务
使用远程桌面的用户，先于命令行下执行
```
echo "startxfce4" > ~/.xsession
chmod 644 ~/.xsession
```
确保用户登录的时候会启动XFCE4

#### 远程桌面登录认证
1. 密码登录，非常不方便
2. 建议使用公钥认证
远程桌面登录认证也可以使用ssh认证协议，安全且方便
请参考用户账户的ssh密钥部分内容，将自己使用的ssh公钥发送给管理员。



## docker
考虑到工作站的使用者大部分情况下会使用相同的环境，我们将所有人都设置为了一个docker用户组。大家都有很高的权限，来对docker进行增删改等管理操作。

### 镜像image与容器Container
**镜像是一个只读的模板**，包含了运行容器所需的文件系统和依赖。

**容器是镜像的运行实例**。它在镜像的基础上加了一层可写层，用来保存运行时的修改。如果你在容器里做了修改（比如增删库或依赖）但没有 docker commit，这些修改只存在于容器的临时可写层，容器一旦停止或删除，修改就会消失。换句话说，镜像不会被更新，你的安装就“白做了”。


### 镜像的分层机制
给现有镜像打上新的标签并docker commit，**镜像 ID 保持不变**，只是多了一个名字映射。多个标签实际上都指向同一组镜像层，因此不会增加额外的磁盘占用。

如果在容器中安装或删除库后执行 docker commit，会生成一个新的镜像。此时镜像 ID 会发生变化，因为 Docker 会在原有镜像的基础上**新增一层** 来保存这些修改。由于采用分层存储机制，硬盘只会增加这部分差异层的大小，而不是复制整个镜像。

随心所欲地打标签没问题，我们也鼓励用户给自己使用的那个镜像打上用户标签，方便管理；同时修改镜像也不用过分担心存储问题，真正消耗空间的只是你新增的那部分差异层。


### OmniLRS
#### 公共资源与用户组
- 该镜像由多人同时使用，且环境依赖的资产占用存储空间较大，不适合各用户单独存储。  
- 我们将 **`/mnt/intel_ssd/omnilrs`** 设置为 **omnilrs 用户组的共享空间**，用于存放 OmniLRS 启动所依赖的静态资源。  
- 这些资源是 **可读写的**，但为了保证环境的稳定性，建议大家在修改之前务必确认自己清楚操作的影响。  

官方教学文档：  
[OmniLRS cfg 环境创建与管理方法](https://github.com/OmniLRS/OmniLRS/wiki/environments#environments)

---

#### 用户个人资产
- 除了公共依赖部分（**最好不要随意修改**），每位用户可能需要使用自己的资产。  
- 在启动容器时，请在 **自己的工作目录** 下运行 `./run_docker.sh`。  
- 启动脚本会自动将当前目录 (`pwd`) 挂载到容器中，作为用户的工作空间。  
- 这样既能共享公共依赖，又能保持个人环境的独立性，避免互相干扰。

---

#### Omnilrs对应的ros2环境
建议搭配另外一个docker使用，我们已经搭建好了桥接器，只需要启动一个新的终端，启动该docker，就能使用ros命令与isaacsim进行交互

将ros2的文件夹放置到了/mnt/intel_ssd/omnilrs/omnilrs_ros2

进入该目录下，执行./docker/run_moveit.sh

该环境同时也配置了moveit，使用前需要执行命令：
```bash
source ./install/setup.bash
source /ros2_ws/moveit_ws/install/setup.bash
#进行初始化
```

### docker的网络访问

**host模式**，容器直接复用宿主机的网络栈，没有独立 IP，看起来和宿主机共享网络。
**bridge模式**，默认模式。容器通过虚拟网卡 (veth) 和 NAT 出网，容器有自己的虚拟 IP。

受到校园网的上网限制，要求 DHCP 分配的宿主机 IP 才能出网，容器发起的 DNS 请求可能被 NAT 或防火墙阻断，导致域名解析失败，即使使用 host 模式，容器内部的 DNS 配置也可能不符合校园网要求，从而无法解析域名。

所以容器想要访问外网，推荐通过宿主机代理转发，也就是在启动容器的时候执行
```
docker run -it \
  -e HTTP_PROXY=http://192.168.151.57:7897 \
  -e HTTPS_PROXY=http://192.168.151.57:7897 \
  -e http_proxy==http://192.168.151.57:7897 \
  -e https_proxy=http://192.168.151.57:7897 \
  xxx_image /bin/bash
```

### 构建镜像时的网络问题（管理员执行）

在校园网环境下，Docker 容器和镜像的网络访问往往受限。为了让 镜像构建过程（docker build）能够正常访问外网，需要让 Docker 守护进程 (daemon) 本身走宿主机代理。

```
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo nano /etc/systemd/system/docker.service.d/http-proxy.conf
```
将内容修改为
```
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:7897"
Environment="HTTPS_PROXY=http://127.0.0.1:7897"
Environment="NO_PROXY=localhost,127.0.0.1"
```
重新启动docker
```
sudo systemctl daemon-reload
sudo systemctl restart docker
```
验证配置是否生效
```
docker info | grep -i proxy
```

## 存储资源

### 数据池
请将大型数据集移动到/mnt/dataset/用户名下，

```bash
#用户自行创建目录
mkdir /mnt/dataset/用户名
```
该目录下所有人可读，只有用户可写

### docker以及conda资源
#### conda
```bash
#用户自行创建目录
mkdir /mnt/intel_ssd/conda_global/envs/用户名
```
配置conda路径
```bash
#./bashrc
export CONDA_ENVS_PATH=/mnt/intel_ssd/conda_global/envs/用户名
```
```bash
#./condarc
channels:
  - defaults

pkgs_dirs:
  - /mnt/intel_ssd/conda_pkgs
envs_dirs:
  - /mnt/intel_ssd/conda_global/envs/用户名
```
#### docker
现在似乎用户层不需要任何操作，制作的.tar镜像文件可以存放至/mnt/dataset/docker下

### 用户目录资源
为了保证系统盘（只有1T）的持续可用性，请大家经常检查一下自己的个人目录，观察是否有一些大型资源放置到了该目录下

建议数据放置到/mnt/dataset/下


