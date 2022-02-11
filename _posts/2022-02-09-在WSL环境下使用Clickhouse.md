---
layout: post
title: 在WSL环境下使用Clickhouse
uuid: dd9a6cee-06f9-49ed-afdf-c7a2415cbb18
---

{{ page.title }}
================

在 windows 上跑 clickhouse 比较好和省力的方法是使用 WSL - Windows Subsystem For Linux\\
WSL 安装方法在 微软官方的文档上就有，直接可以查到，这里不赘述。\\
下面主要讲如何安装 clickhouse \\


首先打开已经装好的 WSL Ubuntu，在终端中开始我们的安装
```bash
sudo apt update
sudo apt upgrade -y
sudo apt-get install apt-transport-https ca-certificates dirmngr -y
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv E0C56BD4

# 如果你的网络有被墙掉，可以使用 清华大学的镜像源   
# "deb https://mirrors.tuna.tsinghua.edu.cn/clickhouse/deb/stable/ main/" 
echo "deb https://repo.clickhouse.com/deb/stable/ main/" | sudo tee \
    /etc/apt/sources.list.d/clickhouse.list

sudo apt-get update
sudo apt-get install -y clickhouse-server clickhouse-client
``` 

接着你可以启动 clickhouse，不过一般来说可能你的 9000 端口已经被占用，你可以修改下 clickhouse-server 的配置

```bash
# 比如把 9000 端口修改成 29000 
sudo sed -i 's/<tcp_port>9000<\/tcp_port>/<tcp_port>29000<\/tcp_port>/g' /etc/clickhouse-server/config.xml
```

其次你可能也需要修改下你的密码
```bash
# 比如我们把密码修改成 10241024
sudo sed -i 's/            <password><\/password>/            <password>10241024<\/password>/g' /etc/clickhouse-server/user.xml
```

下面你就可以启动 clickhouse-server
```bash
sudo service clickhouse-server start
```

然后你可以使用 cli 工具连接开始你的 ck 游戏之旅
```bash
clcikhosue-client -m --port 29000 --passowrd 10241024
```
