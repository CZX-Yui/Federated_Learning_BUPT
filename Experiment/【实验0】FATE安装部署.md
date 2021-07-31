# 单机部署

- 官方提供三种安装方法，推荐使用**Docker镜像**安装，好处是适用性更好，我尝试在本机linux系统下和阿里云远程服务器中安装均可安装成功。
- 本机安装linux系统是因为想保证8GB的运行内存，内存够大16GB以上且实验用需求不大的话可以使用虚拟机VMware安装linux的方案，此情况等同场景一。
- 本文主要参考CSDN：[ubuntu18.04下安装指南](https://blog.csdn.net/weixin_45131630/article/details/109998415)



## 场景一. 本地linux主机部署:

### 1. 检查硬件、系统条件

- 本机配置为2核4线程cpu，8GB运行内存，40GB存储。ubuntu18.04
- 官方建议4核8G至少，支持centos7以上，ubuntu18以上。

### 2. 安装docker

1. 卸载旧版本：
    `sudo apt-get remove docker docker-engine docker.io containerd runc`

2. 更新apt包：
    `sudo apt-get update`

3. 允许apt使用HTTPS上的仓库。
    `sudo apt-get install \
   apt-transport-https \
   ca-certificates \
   curl \
   gnupg-agent \
   software-properties-common`

4. 添加Docker官方GPG密钥。

   `curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -`

5. 用指纹 9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88 验证是否已配置好密钥，搜索指纹的最后8个字符。

   `sudo apt-key fingerprint 0EBFCD88`

6. 下面配置稳定版本。
    `sudo add-apt-repository \
    "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) \
    stable"`

7. 更新apt包源索引。
    `sudo apt-get update` 

8. 看看安装成功没有

  `docker -v`

### 3. 安装docker engine

1. 首先检查可行版本。

   `apt-cache madison docker-ce`

​	FATE官方配置文档建议docker版本为18.09，以5:18.09开头的版本都是可以选择的。
2. docker-ce-cli是客户端版本，和docker-ce的版本保持一致即可，否则可能出现server和client端docker版本不一致的情况。

   `sudo apt-get install docker-ce=5:18.09.3~3-0~ubuntu-bionic docker-ce-cli=5:18.09.3~3-0~ubuntu-bionic containerd.io`

3. 验证是否安装成功，查询docker的版本。

   `sudo docker version`

   结果如下所示。

   Client:
   Version: 18.09.3
   API version: 1.39
   Go version: go1.10.8
   Git commit: 774a1f4
   Built: Thu Feb 28 06:53:11 2019
   OS/Arch: linux/amd64
   Experimental: false
   Server: Docker Engine - Community
   Engine:
   Version: 18.09.3
   API version: 1.39 (minimum version 1.12)
   Go version: go1.10.8
   Git commit: 774a1f4
   Built: Thu Feb 28 05:59:55 2019
   OS/Arch: linux/amd64
   Experimental: false

### 4. 将普通用户加入docker组

1. 为了使非root用户也能使用docker，需要将其添加到docker组中，这一步可以方便后续的docker容器内操作，防止Got permission denied的情况出现。
```
	sudo groupadd docker
	sudo gpasswd -a $XXX docker
	newgrp docker
```
2. 完成以上步骤即完成添加。其中，XXX为你自己的用户名。然后我们可以重启docker，这里不重启也无大碍。

   `sudo systemctl restart docker`

### 5. 安装docker-compose

1. FATE官方配置文档建议docker-compose版本为1.24.0，执行如下语句获取版本资源。

   `sudo curl -L "https://github.com/docker/compose/releases/download/1.24.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose`
这一步连接github下载包会变卡，重试一次就好了，但是下载依旧很慢。

2. 然后我们添加执行许可。

   `sudo chmod +x /usr/local/bin/docker-compose`

3. 检查版本以测试是否安装成功，同时版本号是否为1.24.0。

   `docker-compose --version`

### 6. 部署FATE

1. 检查端口占用情况
获取FATE安装包之前，我们要先检查8080、9360和9380端口是否已被占用，一般是没有的，不过保险起见可以先查询一下。首先将权限切换至root根用户，再使用netstat命令查看pid号，若返回空则说明端口没有被占用。
```
	su root
	netstat -ap | grep 8080
	netstat -ap | grep 9360
	netstat -ap | grep 9380
```

2. 获取安装包
    上一步完成后，需要从root用户切换回普通用户，其中XXX为自己的用户名。

  `su $XXX`

  从微众银行的对应网址下载docker_standalone-fate压缩包并解压。

```
	wget https://webank-ai-1251170195.cos.ap-guangzhou.myqcloud.com/docker_standalone-fate-1.5.0.tar.gz
	tar -xzvf docker_standalone-fate-1.5.0.tar.gz
```

3. 执行部署
将操作路径切换到fate文件夹下，bash指令进行部署。
```
	cd docker_standalone-fate-1.5.0
	bash install_standalone_docker.sh
```

4. 测试是否安装成功
```
	CONTAINER_ID=`docker ps -aqf "name=fate_python"`
	docker exec -t -i ${CONTAINER_ID} bash
	bash ./python/federatedml/test/run_test.sh
```
​	返回以下结果证明安装成功：
​	“there are 0 failed test”

​	其中docker exec -it fate_python bash为进入容器的操作，每次启动bash的时候运行该命令进入容器。在这之前首先查看已启动的容器docker ps，安装完成后默认是启动的，有fate_fateboard和fate_python两个容器，前者是启动fateboard相关的，后者是核心fate容器。启动后cd到根目录会发现有一个fate文件夹，就是fate安装的主目录。



## 场景二. 云服务器部署:

### 1. 检查环境配置

同场景一。我选择阿里云服务器，配置：cpu&内存2核4G，ubuntu18.04

### 2. 连接服务器

几种方法：
- 进入阿里云服务器控制台，选择Workbench远程连接（第一次部署用）
- mobaXterm/Xshell连接（失败，应该可以连）
- 本地cmd通过ssh命令连接（推荐，搭配-L参数换端口转接可以在本地开fateboard，命令在后面） 

### 3. 添加普通用户

​	第一次进入是root账户。在添加普通用户后可以使用用户的用户名密码登录
​	（测试用配置：IP：116.62.205.134）

1. （可选）修改主机名和实例名，在工作台界面修改，修改完重启服务器生效

2. 添加用户USER（下面定义的USER=ganymede），在root账户下创建用户（{username}替换成自己起的名字）：
`useradd -d /home/{username} -m {username} -s /bin/bash `

3. 设置密码：
	`passwd {username} `
	（测试用配置）：用户：ganymede  密码：fate818  
	
4. 切换到ganymede：
	`su ganymede`
	可能出现的问题：
	！问题1：su切换到ganymede时默认用的sh不是bash（命令行只有一行 “$” ），解决方法是要到/etc/passwd文件中修改默认启动的命令行。执行：`sudo vi /etc/passwd`。（以后创建用户时提前修改）
	！问题2：ganymede is not in the sudoers file. 解决方法[加一个sudo权限](https://www.cnblogs.com/xym4869/p/8473646.html)。现在ganymede有了root执行权限，要注意小心使用sudo命令。（即使问题1未出现也要先添加sudo权限，后面用得上）
### 4. 安装docker、部署FATE
**同场景一**

### 5. （建议）开发范式
在windows主机下，连接虚拟机或云服务器，在**XShell**中运行（linux主机就在bash中运行）：
`ssh -L 8081:127.0.0.1:8080 ganymede@116.62.205.134`
连接到对应的环境来操作，这样在本地浏览器打开8081端口可以访问fateboard。

用XShell因为它可以使用rz命令上传数据



## 其它问题:

### 1. 容器内vim安装相关问题
1. 在fate容器中安装vim，需要提前配置国内镜像源。但是配置了豆瓣的镜像源后依然报错E: Unable to locate package vim，解决方法：[在镜像源中加入阿里镜像](https://www.dazhuanlan.com/2019/09/28/5d8eda73db86e/)
2. 安装vim后，退出容器重进后，bash出现bug，一次出现两行且删除键变成空格：原因是可能[误删了一个依赖](https://blog.csdn.net/JerryLaw_/article/details/106086602?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_title~default-8.control&spm=1001.2101.3001.4242)，在fate容器内重新安装一下即可

### 2. fateboard使用相关问题

1. 返回的url打不开可能的原因：
   - IP为0.0.0.0，则查看/fate/conf/service_conf.yaml文件里面fateboard的url是不是127.0.0.1，否则改过来。如果是虚拟机的话改成虚拟机ip
   - fateboard没有部署，查看docker ps看有没有这个容器
   - url是云服务器的localhost，本地需要开一个端口转接才能使用，同场景二/开发范式
2. XShell等工具在连接服务器时需要添加SSH隧道，选择TCP/IP转移，添加端口映射，本地拨出、侦听端口8081、目标端口8080、源主机和目标主机都是localhost

### 3. fate-client相关问题

1. 在容器内安装交互工具FATE-Client以及测试工具FATE-Test时，如果报错应该先升级pip，按照它的提示来。
2. pip install fate-client失败，但是安装fate-test的时候好像顺带安装了，后面看看会不会某一步报错可能是没安装client的原因。

### 4. rz命令使用

1. rz命令（Receive ZMODEM），使用ZMODEM协议，将本地文件批量上传到远程Linux/Unix服务器，注意不能上传文件夹。
2. 安装：`apt-get install lrzsz`
3. 卡死解决：ctrl+x 5次
4. 上传到云服务器用rz，下载用sz
5. 出现乱码，没有弹开文件选择器是shell的原因，选择支持rz的客户端连接服务器如XShell

### 5. [yum，apt-get，wget详解](https://blog.csdn.net/u014472548/article/details/90768415)

# 参考链接

[官方指南](https://github.com/FederatedAI/DOC-CHN/tree/master/%E9%83%A8%E7%BD%B2)

