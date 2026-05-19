# 工作站配置文档(管理员版)

## 0\. 准备工作

1. 初始管理员账号：`user`，默认密码 `1\-6`

2. 新建独立管理员账号：`zxy`、`aluupy`；后续可新建普通用户用于测试隔离使用

## 1\. 软件安装

若无特殊说明，所有软件先在 `user` 用户下完成初步安装与适配。

### 1\.1 系统级安装

- 系统拼音输入法：常规安装配置即可

- 基础应用软件：QQ、微信、VS Code；安装包统一存放于 `Downloads` 目录

- Clash\-Verge 代理工具
        替代旧版 Clash，适配系统全局代理；存在双图标不影响功能。
        尝试过多用户独立安装、公共配置目录共享等方案，因 UI 单实例限制未落地，统一按后文网络配置规则使用即可。
        
补充（当日问题延伸）：需注意代理端口为 `7897`，避免误用 `7890` 端口导致连接失败，配置后可通过 `echo $http\_proxy` 验证端口是否正确。
      

- CUDA 环境
        安装路径：`/usr/local/cuda\-12\.4`、`/usr/local/cuda\-11\.8` 环境变量写入 `\~/\.bashrc`：
        `
export CUDA\_HOME=/usr/local/cuda\-12\.4
export PATH=$CUDA\_HOME/bin:$PATH
export LD\_LIBRARY\_PATH=$CUDA\_HOME/lib64:$LD\_LIBRARY\_PATH
          `

- Docker、Git：常规系统安装
        
补充（当日问题延伸）：Git 安装后需注意协议选择，SSH 协议受校园网限制，建议使用 HTTPS 协议并配置代理，同时可设置凭证缓存避免重复输入账号密码。
      

- SSH：需依托内网环境使用，后续完善配置（TODO）

### 1\.2 用户级安装

- Miniconda、Mamba：常规用户态安装配置

## 2\. 网络配置

### 2\.1 基础网络连接

1. 工作站先接入实验室有线网络，参照实验室文档完成基础联网

### 2\.2 Clash\-Verge 代理配置

1. 安装 Clash\-Verge，导入学长提供的订阅节点（Local 导入），`user` 用户可直接使用

2. 其他账号代理配置方式

    - 浏览器手动代理：Firefox 设置手动代理 `127\.0\.0\.1:7897`

    - 终端全局代理：写入 `\~/\.bashrcexport http\_proxy=\&\#34;http://127\.0\.0\.1:7897\&\#34;
    export https\_proxy=\&\#34;http://127\.0\.0\.1:7897\&\#34;
                  `
    补充（当日问题延伸）：若需访问本机/内网（如 `10\.3\.0\.30:8443`），需先执行 `unset http\_proxy https\_proxy` 关闭代理，避免访问卡死。
              

3. 进阶全局代理配置

    - 修改 Clash\-Verge `config\.yaml`，开启 `allow\-lan: true`，配合 TUN 模式实现全端口转发监听

    - 配置完成后，Docker 容器可直接复用主机代理联网

### 2\.3 WireGuard 内网穿透配置（含当日问题解决）

1. 配置文件路径（核心重点）：WireGuard 配置文件默认存放于 `/etc/wireguard/` 目录，常用配置文件名为 `wg0\.conf`（若存在多个配置，可按网段区分，如 `wg1\.conf`）。

2. 配置文件编辑命令：需使用 sudo 权限编辑，避免权限不足无法保存
        `
sudo nano /etc/wireguard/wg0\.conf
         `

3. 当日问题解决（WireGuard 穿透 Nginx 8443 访问）：
        

    - 问题现象：原本想用物理网卡 `192\.168\.x\.x` 外网穿透访问 Nginx 8443，跨网段不通；同时访问 `http://10\.3\.0\.30:8443` 提示网页解析失败。

    - 问题原因：物理网段存在隔离，WireGuard 未正确使用隧道网段；解析失败为临时网络缓存或配置未生效。

    - 解决步骤：
                

        1. 无需修改 WireGuard 配置文件（`/etc/wireguard/wg0\.conf`），确认配置文件中已包含 `10\.3\.0\.0/24` 网段放行规则（默认已配置）。

        2. 直接使用 WireGuard 隧道虚拟 IP `10\.3\.0\.30:8443` 访问 Nginx 服务，隧道内天然放行该网段，避开物理网段隔离。

        3. 若仍提示解析失败，执行 `sudo systemctl restart wireguard` 重启 WireGuard 服务，等待 1\-2 分钟后重新访问即可。

4. WireGuard 常用命令：
        `
\# 启动 WireGuard 服务（指定配置文件）
sudo wg\-quick up wg0
\# 停止 WireGuard 服务
sudo wg\-quick down wg0
\# 查看 WireGuard 运行状态
sudo wg show
\# 设置开机自启
sudo systemctl enable wg\-quick@wg0
          `

## 3\. 硬盘挂载配置

1. 硬盘现状：`f` 盘已默认挂载；`a\-e` 为 10T 机械硬盘未挂载，统一挂载至 `/mnt/disk\_sda \~ /mnt/disk\_sde`，可通过 `lsblk` 查看磁盘信息（原文档笔误 `1lsblk` 修正）。

2. 标准挂载流程（以 `/dev/sda` 为例）

```bash

# 查看磁盘及分区信息
lsblk
fdisk -l /dev/sda

# 格式化为 ext4 文件系统
sudo mkfs.ext4 /dev/sda1

# 创建挂载点并临时挂载
sudo mkdir /mnt/disk_sda
sudo mount /dev/sda1 /mnt/disk_sda

# 查看挂载状态、获取分区 UUID
df -h /mnt/disk_sda
sudo blkid /dev/sda1

# 配置开机自启
sudo nano /etc/fstab
# 末尾添加一行（替换为实际UUID）
# UUID=xxxxxx-xxxxxx /mnt/disk_sda ext4 defaults 0 2

# 校验 fstab 配置，无输出即正常
sudo mount -a
      
```

### 机械盘数据池管理
使用mergerfs管理几块机械盘作为数据硬盘
mergerfs 会：把多个目录“叠加”成一个虚拟目录，使用户看到的是一个大目录 /mnt/dataset，实际文件仍然存储在各自的磁盘上，写入时按策略选择目标磁盘（默认最空的盘），读取时自动从对应磁盘读取

#### 开机自动挂载
```bash
sudo nano /etc/fstab

/mnt/disk_sda:/mnt/disk_sdc:/mnt/disk_sdd:/mnt/disk_sde /mnt/dataset fuse.mergerfs defaults,allow_other,use_ino,nonempty 0 0

sudo mount -a #测试是否成功

```
#### 权限管理
方案：使用“共享目录 + 用户独立子目录”权限模型

```bash
sudo chmod 1777 /mnt/dataset #用户可以创建自己的目录，但不能删除或修改别人的目录
```
用户创建自己的目录
```bash
mkdir /mnt/dataset/liupei2 #创建者自动成为所有人，其他用户：只读、不能写、不能删
```

用户允许可写
```bash
chmod 755 /mnt/dataset/用户名 #允许特定用户可写
chmod 775 /mnt/dataset/用户名 #允许同组可写
```


## 4\. 公共资源路径

OmniLRS 镜像存放目录：

- 镜像 1：`/opt/docker\-images/isaac\-sim\-omnilrs\.tar`

- 镜像 2：`/opt/docker\-images/omnilrs\-navigation\-v1\.0\.tar`

## 5\. Docker 多用户隔离与使用规范

### 5\.1 基础镜像操作

- 查看本地镜像：`docker images`

- 镜像打标签（用户隔离）：`sudo docker tag 原始镜像ID zxy/isaac\-sim\-omnilrs:v1`

- 删除镜像：`docker rmi zxy/isaac\-sim\-omnilrs:v1`

### 5\.2 多用户冲突解决方案

1. 用户权限：将普通用户加入 docker 组，免 sudo 操作
        `
sudo usermod \-aG docker zxy
newgrp docker  \# 立即刷新组权限，无需重启
          `

2. 镜像：共用公共镜像无冲突，无需重复拉取

3. 容器：**容器名称必须差异化**，建议前缀加 `$\{USER\}`区分

4. ROS2 串台解决：修改 `run\_docker\.sh`，配置独立 `ROS\_DOMAIN\_ID` 实现多用户 ROS 环境隔离

5. 端口：规避端口占用冲突，各用户尽量错开服务端口

### 5\.3 OmniLRS 镜像部署

1. 提前安装英伟达 Container Toolkit，适配 Docker 运行 Isaac Sim

2. 跳过官方 `build\_docker\.sh` 编译步骤，直接加载本地镜像：`
sudo docker load \-i /opt/docker\-images/isaac\-sim\-omnilrs\.tar
          `

3. 加载完成后按官方 Getting Started 流程启动使用

## 6\. Git 常用配置与当日问题解决汇总

### 6\.1 Git Push 报错（kex\_exchange\_identification 连接被关闭）

1. 现象：Git 用 SSH 地址 `git@xxx` push 直接报错，连接被远端关闭

2. 原因：校园网墙 Git SSH 端口，且**终端 http/https 代理对 SSH 协议无效**

3. 解决步骤：
        

    - 删除旧 SSH 远程：`git remote remove origin`

    - 新增 HTTPS 协议远程地址（以 Gitee 为例）：`git remote add origin https://gitee\.com/zxxxxxywxxxxxr/workstation\_guide\.git`

    - 终端配置代理 `7897`，HTTPS 可正常走代理推拉代码

### 6\.2 Git HTTP 每次推拉都要输账号密码

1. 问题：HTTPS 协议每次 pull/push 重复验证账号密码

2. 解决：配置 Git 全局凭证永久记住
        `git config \-\-global credential\.helper store
          `首次输入一次账号密码后，后续永久免输

### 6\.3 Git 常用命令（补充）

```bash

# 查看提交历史（简洁版）
git log --oneline
# 查看最近1条提交详情（含提交时间）
git log -1 --pretty=full
# 查看当前远程地址
git remote -v
# 配置本地 Git 用户名和邮箱（与 Gitee 一致，确保贡献统计）
git config --global user.name "zxxxxxywxxxxxr"
git config --global user.email "1341937440@qq.com"
      
```

## 7\. TODO

1. 完善 SSH 内网连接配置文档

2. 统一多用户硬盘权限与目录划分规范

3. 补充 SOCKS 代理配置、终端代理一键切换脚本

4. 整理 Docker 多用户端口分配标准、ROS\_DOMAIN\_ID 分配规则

5. 整理 Git 常用查看历史、回退、改邮箱等常用命令速查表

6. 补充 WireGuard 配置文件详细参数说明，方便后续修改调试

7. 记录 Nginx 8443 端口常见访问问题及排查流程
