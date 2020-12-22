
$apt-get update
$apt-get install apt-transport-https ca-certificates curl software-properties-common lrzsz -y

#使用阿里云的源
$ sudo curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
$ sudo add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"

$ sudo apt-get update
$ sudo apt-get install docker-ce -y
假如这一步报错：
Package 'docker-ce' has no installation candidate
说明之前用add-apt-repository 添加的docker源不好使，需要改：
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"
以此来更换一个docker源，其实作用就是等效于在/etc/apt/sources.list文件末尾增加了两行：
deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable
# deb-src [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable
之后  ： sudo apt-get update使之生效
最后再次 ： sudo apt-get install docker-ce -y 即可

#测试docker
docker version

#加速器配置
$ curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://f1361db2.m.daocloud.io
#修改配置文件
$ sudo vim /etc/docker/daemon.json
#文件内容
{"registry-mirrors": ["http://f1361db2.m.daocloud.io"], "insecure-registries": []}

假如要换成阿里云加速：
修改文件
vim  /etc/docker/daemon.json
{
    "registry-mirrors": ["https://alzgoonw.mirror.aliyuncs.com"],
    "live-restore": true
}
重起服务
systemctl daemon-reload

#修改权限
$sudo groupadd docker
$sudo gpasswd -a ${USER} docker
$sudo systemctl restart docker
$newgrp - docker
