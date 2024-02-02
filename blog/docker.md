# 安装

`centos`

```shell
# 安装
yum update
yum install docker

# 启动
systemctl start docker

# 检查docker是否启动
docker version

# 设置开机启动
systemctl enable docker

# 停止
systemctl stop docker
```



# 配置镜像

> vim /etc/docker/daemon.json

```json
{
 "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn/"]
}
```

再执行 `systemctl restart docker`

然后执行 `docker info` 观察 `Registry Mirrors` 看是不是有配置的镜像